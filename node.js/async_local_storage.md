# Node.js AsyncLocalStorage 완전 분석: Request Context가 섞이지 않는 이유

## 목차
1. [서론](#서론)
2. [C++ 코드 읽기 가이드](#c-코드-읽기-가이드)
3. [AsyncLocalStorage란?](#asynclocalstorage란)
4. [핵심 구조: AsyncContextFrame](#핵심-구조-asynccontextframe)
5. [V8 엔진의 Continuation Preserved Embedder Data](#v8-엔진의-continuation-preserved-embedder-data)
6. [Node.js 아키텍처: Single Thread vs Multi Thread](#nodejs-아키텍처-single-thread-vs-multi-thread)
7. [Context 격리 메커니즘](#context-격리-메커니즘)
8. [동기 vs 비동기: Context 처리 방식의 차이](#동기-vs-비동기-context-처리-방식의-차이)
9. [als 중첩과 다중 als 사용](#als-중첩과-다중-als-사용)
10. [I/O 작업과 Context 전파](#io-작업과-context-전파)
11. [Worker Thread와 메모리 격리](#worker-thread와-메모리-격리)
12. [결론](#결론)

---

## 서론

Node.js에서 HTTP 요청마다 독립적인 컨텍스트를 유지하는 것은 오랫동안 어려운 문제였습니다. 특히 비동기 작업이 많은 환경에서 "이 로그는 어떤 요청에서 발생한 것인가?"를 추적하는 것은 매우 중요합니다.

AsyncLocalStorage는 이 문제를 해결하기 위해 등장했으며, Node.js v13.10.0부터 stable API가 되었습니다. 이 글에서는 AsyncLocalStorage가 어떻게 각 요청의 context를 격리하는지, 그 내부 동작 원리를 코드 레벨에서 깊이 분석합니다.

---

## C++ 코드 읽기 가이드

이 문서에는 V8과 Node.js의 C++ 코드가 포함되어 있습니다. JavaScript 개발자를 위해 핵심 문법을 간단히 설명합니다.

### 포인터 멤버 접근: `->`

```cpp
// JavaScript로 비유하면:
obj.property      // 일반 객체의 멤버 접근
ptr->property     // 포인터가 가리키는 객체의 멤버 접근

// ptr->property는 (*ptr).property와 같음
// 즉, 포인터를 역참조하고 멤버에 접근
```

예시:

```cpp
// C++
isolate->GetContinuationPreservedEmbedderDataV2()

// JavaScript로 표현하면
isolate.getContinuationPreservedEmbedderDataV2()
```

### 기타 자주 나오는 문법

```cpp
Local<Value>      // 템플릿 문법, JavaScript의 제네릭과 유사
auto              // 타입 자동 추론 (JavaScript의 let과 유사)
::                // 네임스페이스/스코프 접근 (v8::Isolate)
```

---

## AsyncLocalStorage란?

AsyncLocalStorage는 비동기 작업 간에 데이터를 전파하는 메커니즘을 제공합니다.

### 기본 사용법

```javascript
const { AsyncLocalStorage } = require('async_hooks');
const als = new AsyncLocalStorage();

// HTTP 서버 예시
server.on('request', (req, res) => {
  const requestId = generateId();

  als.run({ requestId }, async () => {
    // 이 블록 내부의 모든 비동기 작업에서
    // als.getStore()로 { requestId } 접근 가능

    await database.query('SELECT ...');
    logger.info('Query completed', als.getStore()); // { requestId }

    await fetch('https://api.example.com');
    logger.info('API called', als.getStore()); // 여전히 { requestId }
  });
});
```

### 핵심 질문

여러 요청이 동시에 처리될 때, 각 요청의 context가 어떻게 섞이지 않을까?

```javascript
// Request 1
als.run({ id: 'req-1' }, async () => {
  await someAsyncWork();
  console.log(als.getStore()); // { id: 'req-1' } ✅
});

// Request 2 (Request 1이 완료되기 전에 시작)
als.run({ id: 'req-2' }, async () => {
  await someAsyncWork();
  console.log(als.getStore()); // { id: 'req-2' } ✅
});
```

이 글에서는 이 질문에 대한 답을 찾아갑니다.

---

## 핵심 구조: AsyncContextFrame

Node.js v24.0.0부터 AsyncLocalStorage는 `AsyncContextFrame`이라는 새로운 구현을 사용합니다. 이는 이전의 `async_hooks` 기반 구현보다 성능이 크게 향상되었습니다.

### AsyncContextFrame 클래스

**`lib/internal/async_context_frame.js`**

```javascript
class AsyncContextFrame extends SafeMap {
  constructor(store, data) {
    super(AsyncContextFrame.current());  // 현재 frame을 복사
    this.set(store, data);               // 새 store-data 쌍 추가
  }

  disable(store) {
    this.delete(store);
  }
}
```

핵심 포인트:
- `AsyncContextFrame`은 **Map을 상속**합니다
- **AsyncLocalStorage 인스턴스가 키**, 데이터가 값입니다
- 생성 시 현재 frame을 복사하므로 **Copy-on-Write** 방식으로 동작합니다

### 동적 프로토타입 교체

성능 최적화를 위해 Node.js는 동적으로 프로토타입을 교체합니다:

**`lib/internal/async_context_frame.js:40-52`**

```javascript
function checkEnabled() {
  const enabled = require('internal/options')
    .getOptionValue('--async-context-frame');

  // If enabled, swap to active prototype so we don't need to check status
  // on every interaction with the async context frame.
  if (enabled) {
    // eslint-disable-next-line no-use-before-define
    ObjectSetPrototypeOf(AsyncContextFrame, ActiveAsyncContextFrame);
  }

  return enabled;
}
```

기본적으로 `--async-context-frame` 옵션은 **true**입니다 (`src/node_options.h:151`):

```cpp
bool async_context_frame = true;
```

---

## V8 엔진의 Continuation Preserved Embedder Data

AsyncContextFrame이 비동기 경계를 넘어 전파되는 핵심은 V8 엔진의 **Continuation Preserved Embedder Data** API입니다.

### 슬롯(Slot)과 Context의 관계

먼저 중요한 개념을 명확히 해야 합니다:

```
슬롯(Slot) = 저장 공간 (변수와 같음)
Context = 슬롯에 저장되는 값 (Map 객체)
```

```
┌─────────────────────────────────────┐
│  V8 Isolate                         │
│                                     │
│  continuation_preserved_embedder_data_ (슬롯)
│  ┌─────────────────────────────┐   │
│  │                             │   │
│  │  ← 여기에 context가 들어감   │   │
│  │    (Map 객체 또는 undefined) │   │
│  │                             │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘
```

JavaScript로 비유하면:

```javascript
// 의사 코드
class Isolate {
  currentContextSlot = undefined  // ← 이게 "슬롯"
}

// als.run() 진입 시
isolate.currentContextSlot = Map([[als, data]])  // Map이 "context"

// als.run() 종료 시
isolate.currentContextSlot = undefined  // 슬롯은 있지만 비어있음
```

**핵심**: "슬롯 = context"가 아니라 "슬롯에 context가 저장된다"입니다.

### V8 API 정의

**`deps/v8/include/v8-isolate.h:985-997`**

```cpp
/**
 * Returns the value set by `SetContinuationPreservedEmbedderDataV2()` or
 * restored during microtask execution for the currently running continuation,
 * if any. Returns undefined if no continuation preserved embedder data was
 * set.
 */
Local<Data> GetContinuationPreservedEmbedderDataV2();

/**
 * Sets a value that will be stored on continuations and restored while the
 * continuation runs. If `data` is empty, the continuation preserved embedder
 * data is set to undefined.
 */
void SetContinuationPreservedEmbedderDataV2(Local<Data> data);
```

### Node.js의 바인딩

**`src/async_context_frame.cc:36-46`**

```cpp
Local<Value> current(Isolate* isolate) {
  return isolate->GetContinuationPreservedEmbedderDataV2().As<Value>();
}

void set(Isolate* isolate, Local<Value> value) {
  auto env = Environment::GetCurrent(isolate);
  if (!env->options()->async_context_frame) {
    return;
  }

  isolate->SetContinuationPreservedEmbedderDataV2(value);
}
```

### IsolateData에 저장

V8의 각 Isolate는 단일 context 슬롯을 가집니다:

**`deps/v8/src/execution/isolate-data.h:350-355, 580`**

```cpp
class IsolateData {
  Tagged<Object> continuation_preserved_embedder_data() const {
    return continuation_preserved_embedder_data_;
  }

  void set_continuation_preserved_embedder_data(Tagged<Object> data) {
    continuation_preserved_embedder_data_ = data;
  }

private:
  // This is data that should be preserved on newly created continuations.
  Tagged<Object> continuation_preserved_embedder_data_ = Smi::zero();
};
```

**핵심**: Isolate는 단 하나의 "현재 활성 컨텍스트" 슬롯만 가집니다.

---

## Node.js 아키텍처: Single Thread vs Multi Thread

Node.js의 아키텍처를 이해하는 것이 context 격리를 이해하는 핵심입니다.

### 1 Request = 1 Isolate가 아니다!

흔한 오해 중 하나는 "각 HTTP 요청마다 새로운 Isolate가 생성된다"는 것입니다. **이는 사실이 아닙니다.**

```
┌─────────────────────────────────────────────────────┐
│              Node.js 프로세스                        │
│                                                     │
│   ┌─────────────────────────────────────────────┐   │
│   │           V8 Isolate (딱 1개)               │   │
│   │                                             │   │
│   │   Request 1 ──┐                             │   │
│   │   Request 2 ──┼── 모두 같은 Isolate 공유     │   │
│   │   Request 3 ──┘                             │   │
│   │                                             │   │
│   │   현재 context 슬롯: [____]  ← 1개만 존재    │   │
│   │                                             │   │
│   └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

- **1 프로세스 = 1 Isolate** (Worker Thread 제외)
- 모든 Request가 **같은 Isolate를 시분할**로 사용
- 그래서 context 격리 메커니즘이 중요한 것!

### V8 Engine: Single Thread

V8 공식 문서 (`deps/v8/include/v8-isolate.h:283-288`):

```cpp
/**
 * Isolate represents an isolated instance of the V8 engine. V8 isolates have
 * completely separate states. Objects from one isolate must not be used in
 * other isolates. The embedder can create multiple isolates and use them in
 * parallel in multiple threads. An isolate can be entered by at most one
 * thread at any given time.
 */
```

핵심:
- **하나의 Isolate는 동시에 하나의 thread만 접근 가능**
- JavaScript 코드는 절대 동시에 실행되지 않음
- Event loop가 순차적으로 처리

### libuv: Multi Thread I/O

libuv 공식 문서 (`deps/uv/docs/src/threadpool.rst:22-26`):

```
The threadpool is global and shared across all event loops. When a particular
function makes use of the threadpool (e.g. when using uv_queue_work)
libuv preallocates and initializes the maximum number of threads allowed by
UV_THREADPOOL_SIZE. More threads usually means more throughput but a higher
memory footprint.
```

Thread pool 구현 (`deps/uv/src/threadpool.c:33-44`):

```c
static uv_once_t once = UV_ONCE_INIT;
static uv_cond_t cond;
static uv_mutex_t mutex;
static unsigned int idle_threads;
static unsigned int slow_io_work_running;
static unsigned int nthreads;
static uv_thread_t* threads;
static uv_thread_t default_threads[4];  // 기본 4개
static struct uv__queue exit_message;
static struct uv__queue wq;             // work queue (공유)
```

핵심:
- **Thread pool threads끼리는 메모리 공유** (mutex로 동기화)
- **하지만 V8 JavaScript Heap에는 접근 불가**
- 순수 I/O 작업만 수행

### Thread Pool이 V8 Heap에 접근할 수 없는 이유

V8 Heap은 Main Thread 전용입니다. 그 이유는:

```
┌─────────────────────────────────────────┐
│           프로세스 메모리                │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │   V8 JavaScript Heap            │   │
│  │   (Main Thread만 접근 가능)      │   │
│  │                                 │   │
│  │   - JS 객체들                    │   │
│  │   - AsyncContextFrame           │   │
│  │   - 가비지 컬렉터 관리           │   │
│  └─────────────────────────────────┘   │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │   Native Memory (C/C++)         │   │
│  │   (모든 Thread 접근 가능)        │   │
│  │                                 │   │
│  │   - libuv 구조체                 │   │
│  │   - 파일 버퍼                    │   │
│  │   - 네트워크 버퍼                │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

**접근 불가 이유:**

1. **Single-threaded GC**: V8은 단일 스레드 가비지 컬렉터를 사용합니다. 다른 스레드가 동시에 접근하면 메모리 corruption이 발생합니다.

2. **객체 이동**: GC 과정에서 JS 객체의 메모리 주소가 변경될 수 있습니다. 다른 스레드가 들고 있던 포인터가 갑자기 무효화됩니다.

3. **설계 철학**: V8 API 자체가 Isolate당 하나의 스레드만 허용하도록 설계되었습니다.

### 아키텍처 다이어그램

```
┌─────────────────────────────────────────────────┐
│         V8 Engine (Single Thread)               │
│  ───────────────────────────────────           │
│  • JavaScript 실행                              │
│  • Event Loop                                   │
│  • AsyncContextFrame                            │
│  • 한 번에 하나의 코드만 실행                    │
└────────────────┬────────────────────────────────┘
                 │
                 │ Binding
                 ▼
┌─────────────────────────────────────────────────┐
│         libuv (Multi Thread I/O)                │
│  ───────────────────────────────────           │
│  Thread Pool (기본 4개):                        │
│  [Thread 1] [Thread 2] [Thread 3] [Thread 4]   │
│   File I/O    DNS       crypto     ...          │
│                                                 │
│  • 병렬 실행 ✅                                  │
│  • V8 접근 불가 ❌                               │
│  • JavaScript 객체 접근 불가 ❌                  │
└─────────────────────────────────────────────────┘
```

---

## Context 격리 메커니즘

### 1. Microtask 생성 시: 스냅샷 저장

Promise나 async/await을 사용할 때, V8은 microtask를 생성하고 현재 context를 저장합니다.

**Microtask 구조** (`deps/v8/src/objects/microtask.tq:6-9`):

```javascript
@abstract
extern class Microtask extends Struct {
  @if(V8_ENABLE_CONTINUATION_PRESERVED_EMBEDDER_DATA)
  continuation_preserved_embedder_data: Object|Undefined;  // ← 저장!
}
```

**Microtask 생성** (`deps/v8/src/heap/factory.cc:1642-1644`):

```cpp
microtask->set_continuation_preserved_embedder_data(
    isolate()->isolate_data()->continuation_preserved_embedder_data(),
    SKIP_WRITE_BARRIER);
```

핵심: **각 microtask는 생성 시점의 context 스냅샷을 보관**합니다.

### Microtask에 저장되는 것: Map 전체 (Key 포함!)

중요한 점은 **Map 전체가 스냅샷으로 저장**된다는 것입니다. Key(als 인스턴스)와 Value(데이터) 모두 포함됩니다.

```javascript
// als.run() 내부에서 Promise 생성 시
Promise.resolve().then(callback)
```

```
┌─────────────────────────────────────────────────────┐
│  Microtask 객체                                     │
│                                                     │
│  {                                                  │
│    callback: callback,                              │
│    continuation_preserved_embedder_data:            │
│      Map([                                          │
│        [als인스턴스, {id:'req-1'}],  ← key도 포함!  │
│        [다른als, {other:'data'}],                   │
│      ])                                             │
│  }                                                  │
└─────────────────────────────────────────────────────┘
```

이렇게 Map 전체가 저장되기 때문에, 나중에 microtask가 실행될 때 `als.getStore()`가 올바른 값을 반환할 수 있습니다.

### 2. Microtask 실행 시: Context 복원

**Microtask 실행** (`deps/v8/src/builtins/builtins-microtask-queue-gen.cc:125-138`):

```cpp
void SetupContinuationPreservedEmbedderData(TNode<Microtask> microtask) {
  // Microtask 객체에서 저장된 context를 로드
  TNode<Object> continuation_preserved_embedder_data = LoadObjectField(
      microtask, Microtask::kContinuationPreservedEmbedderDataOffset);

  // undefined가 아니면 Isolate의 현재 context로 설정
  if (!IsUndefined(continuation_preserved_embedder_data)) {
    SetContinuationPreservedEmbedderData(continuation_preserved_embedder_data);
  }
}
```

**실행 흐름** (`deps/v8/src/builtins/builtins-microtask-queue-gen.cc:174-196`):

```cpp
BIND(&is_callable);
{
  // 1. Microtask의 context 준비
  TNode<Context> microtask_context =
      LoadObjectField<Context>(microtask, CallableTask::kContextOffset);

  // 2. Context 복원
  #ifdef V8_ENABLE_CONTINUATION_PRESERVED_EMBEDDER_DATA
    SetupContinuationPreservedEmbedderData(microtask);  // ← 복원!
  #endif

  // 3. 실제 함수 호출 (복원된 context에서 실행)
  Call(microtask_context, callable, UndefinedConstant());

  // 4. 실행 후 정리
  #ifdef V8_ENABLE_CONTINUATION_PRESERVED_EMBEDDER_DATA
    ClearContinuationPreservedEmbedderData();  // ← undefined로 리셋
  #endif
}
```

### Context 복원 과정 상세

```
┌─────────────────────────────────────────────────────────────┐
│  Microtask 실행 시                                          │
│                                                             │
│  1. microtask.continuation_preserved_embedder_data          │
│     → Map([[als, {id:'req-1'}]]) 가져옴                     │
│                                                             │
│  2. Isolate 슬롯에 이 Map 통째로 설정                       │
│     isolate.슬롯 = Map([[als, {id:'req-1'}]])               │
│                                                             │
│  3. callback() 실행                                         │
│     → als.getStore() 호출                                   │
│     → AsyncContextFrame.current() = 현재 슬롯의 Map         │
│     → map.get(als) = {id:'req-1'}                           │
│                                                             │
│  4. 실행 완료 후                                            │
│     isolate.슬롯 = undefined                                │
└─────────────────────────────────────────────────────────────┘
```

### 3. als.run()의 복원 메커니즘

**`lib/internal/async_local_storage/async_context_frame.js:56-67`**

```javascript
run(data, fn, ...args) {
  const prior = this.getStore();  // 1. 현재 상태 백업
  if (ObjectIs(prior, data)) {
    return ReflectApply(fn, null, args);
  }
  this.enterWith(data);           // 2. 새 context 설정
  try {
    return ReflectApply(fn, null, args);  // 3. 함수 실행
  } finally {
    this.enterWith(prior);        // 4. 항상 이전 상태로 복원!
  }
}
```

**핵심**: `finally` 블록이 항상 이전 상태를 복원하므로, `als.run()` 종료 후 Isolate의 현재 context는 undefined로 돌아갑니다.

### 4. getStore()의 동작

**`lib/internal/async_local_storage/async_context_frame.js:73-79`**

```javascript
getStore() {
  const frame = AsyncContextFrame.current();  // Isolate에서 현재 frame 가져옴
  if (!frame?.has(this)) {  // this = AsyncLocalStorage 인스턴스 (키!)
    return this.#defaultValue;
  }
  return frame?.get(this);  // Map.get()으로 조회
}
```

핵심:
- `AsyncContextFrame`은 **Map 객체**
- **AsyncLocalStorage 인스턴스 자체가 키**
- 같은 frame에서 다른 AsyncLocalStorage는 다른 값을 가짐

### 5. Microtask 생명주기와 가비지 컬렉션

Microtask는 실행 완료 후 가비지 컬렉션 대상이 됩니다:

```
┌─────────────────────────────────────────────────────────────┐
│  1. 생성                                                    │
│                                                             │
│  Promise.resolve().then(callback)                           │
│        │                                                    │
│        ▼                                                    │
│  Microtask 객체 생성 → Microtask Queue에 추가               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  2. 대기                                                    │
│                                                             │
│  Microtask Queue: [task1, task2, task3, ...]               │
│  (현재 동기 코드가 끝날 때까지 대기)                          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  3. 실행                                                    │
│                                                             │
│  while (microtaskQueue.length > 0) {                       │
│    task = microtaskQueue.shift()  // Queue에서 제거         │
│    isolate.슬롯 = task.context    // context 복원           │
│    task.callback()                // 실행                   │
│    isolate.슬롯 = undefined       // 정리                   │
│    // task 객체는 이제 아무도 참조 안 함                      │
│  }                                                          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  4. 가비지 컬렉션                                            │
│                                                             │
│  Microtask 객체:                                            │
│    - Queue에서 제거됨 ✓                                     │
│    - 더 이상 참조하는 곳 없음 ✓                              │
│    → GC 대상 → 메모리 해제                                  │
│                                                             │
│  저장된 context Map도 함께 해제 (다른 참조 없으면)           │
└─────────────────────────────────────────────────────────────┘
```

### 타임라인 분석

```
Time | Isolate 슬롯 값           | 이벤트
-----|-------------------------|---------------------------
  1  | undefined               | Request 1 도착
  2  | Map([als, req-1])       | als.run() 진입
  3  | Map([als, req-1])       | Promise.then() 생성
     |                         | → microtask1에 스냅샷 저장
  4  | undefined               | als.run() 종료 (finally)
  5  | undefined               | Request 2 도착
  6  | Map([als, req-2])       | als.run() 진입
  7  | Map([als, req-2])       | Promise.then() 생성
     |                         | → microtask2에 스냅샷 저장
  8  | undefined               | als.run() 종료 (finally)
  9  | Map([als, req-1])       | microtask1 실행 (스냅샷 복원)
 10  | undefined               | microtask1 실행 완료 (정리)
 11  | Map([als, req-2])       | microtask2 실행 (스냅샷 복원)
 12  | undefined               | microtask2 실행 완료 (정리)
```

**결론**:
- Request 1과 2의 **동기 코드는 순차적**으로 실행
- 각 microtask는 **생성 시점의 독립적인 스냅샷** 보관
- 실행 시 **Isolate의 슬롯에 복원**, 실행 후 정리
- **슬롯이 undefined라는 것은 "context가 없다"가 아니라 "현재 활성화된 context가 없다"는 의미**

---

## 동기 vs 비동기: Context 처리 방식의 차이

동기 작업과 비동기 작업은 context를 다르게 처리합니다.

### 동기 작업: 별도 저장 필요 없음

```javascript
als.run({ id: 'req-1' }, () => {
  // 동기 코드는 als.run() 안에서 바로 실행
  // Isolate 슬롯이 계속 유지됨
  console.log(als.getStore())  // 그냥 현재 슬롯 읽기

  syncFunction()  // 콜 스택만 쌓임, context 그대로

  console.log(als.getStore())  // 여전히 같은 context
})
```

동기 코드에서는:
- `als.run()` 블록 안에서 실행되는 동안 Isolate 슬롯이 유지됨
- 별도의 저장/복원 과정 없이 현재 슬롯을 직접 읽음
- 콜 스택이 쌓여도 context는 변하지 않음

### 비동기 작업: 저장/복원 필요

```javascript
als.run({ id: 'req-1' }, () => {
  console.log(als.getStore())  // { id: 'req-1' }

  setTimeout(() => {
    // 이 콜백은 나중에 실행됨
    // 그때 Isolate 슬롯은 이미 undefined
    // → 미리 저장해둔 context 복원 필요
    console.log(als.getStore())  // { id: 'req-1' } (복원됨)
  }, 100)
})
// als.run() 종료 → Isolate 슬롯 = undefined
// ... 100ms 후 ...
// setTimeout 콜백 실행 시 저장된 context 복원
```

비동기 코드에서는:
- 비동기 작업 생성 시 현재 context를 스냅샷으로 저장
- `als.run()` 종료 후 슬롯은 undefined가 됨
- 콜백 실행 시 저장된 스냅샷을 슬롯에 복원

### 비교 요약

| 구분 | 동기 | 비동기 |
|------|------|--------|
| context 위치 | Isolate 슬롯 (현재) | Microtask/AsyncResource에 저장 |
| 저장 시점 | 저장 안 함 | 비동기 작업 생성 시 |
| 복원 시점 | 복원 안 함 | 콜백 실행 직전 |
| getStore() 동작 | 현재 슬롯 직접 읽기 | 복원된 슬롯에서 읽기 |

---

## als 중첩과 다중 als 사용

### Case A: 같은 als 중첩 (als 안에서 als)

```javascript
const als = new AsyncLocalStorage()

als.run({ id: 'outer' }, () => {
  console.log(als.getStore())  // { id: 'outer' }

  als.run({ id: 'inner' }, () => {
    console.log(als.getStore())  // { id: 'inner' }
  })

  console.log(als.getStore())  // { id: 'outer' } ← 복원됨!
})
```

**Map 변화:**

```
시점 1: 외부 als.run() 진입
┌─────────────────────────────┐
│ Isolate 슬롯:               │
│   Map([[als, {id:'outer'}]])│
└─────────────────────────────┘

시점 2: 내부 als.run() 진입
┌─────────────────────────────┐
│ Isolate 슬롯:               │
│   Map([[als, {id:'inner'}]])│  ← 같은 key, 값만 덮어씀
└─────────────────────────────┘

시점 3: 내부 als.run() 종료 (finally)
┌─────────────────────────────┐
│ Isolate 슬롯:               │
│   Map([[als, {id:'outer'}]])│  ← prior 값으로 복원!
└─────────────────────────────┘
```

**als.run() 내부 동작:**

```javascript
run(data, fn) {
  const prior = this.getStore()  // 'outer' 저장
  this.enterWith(data)           // 'inner'로 설정
  try {
    return fn()
  } finally {
    this.enterWith(prior)        // 'outer'로 복원!
  }
}
```

### Case B: 다른 als1, als2 순차 사용

```javascript
const als1 = new AsyncLocalStorage()
const als2 = new AsyncLocalStorage()

als1.run({ a: 1 }, () => {
  console.log(als1.getStore())  // { a: 1 }
  console.log(als2.getStore())  // undefined

  als2.run({ b: 2 }, () => {
    console.log(als1.getStore())  // { a: 1 } ← 유지!
    console.log(als2.getStore())  // { b: 2 }
  })

  console.log(als1.getStore())  // { a: 1 }
  console.log(als2.getStore())  // undefined ← 복원됨
})
```

**Map 변화:**

```
시점 1: als1.run() 진입
┌─────────────────────────────┐
│ Isolate 슬롯:               │
│   Map([                     │
│     [als1, {a:1}]           │
│   ])                        │
└─────────────────────────────┘

시점 2: als2.run() 진입
┌─────────────────────────────┐
│ Isolate 슬롯:               │
│   Map([                     │
│     [als1, {a:1}],          │  ← 유지!
│     [als2, {b:2}]           │  ← 추가!
│   ])                        │
└─────────────────────────────┘

시점 3: als2.run() 종료
┌─────────────────────────────┐
│ Isolate 슬롯:               │
│   Map([                     │
│     [als1, {a:1}]           │  ← 유지
│   ])                        │  ← als2 제거됨
└─────────────────────────────┘
```

### 비교 요약

| 상황 | Map 동작 | 결과 |
|------|----------|------|
| 같은 als 중첩 | 같은 key의 value 덮어쓰기 → finally에서 이전 value 복원 | 외부 context로 돌아감 |
| 다른 als 순차 | 다른 key 추가 → finally에서 해당 key 제거 | 각자 독립적 |

```
Case A: als 중첩
Map: [als → outer] → [als → inner] → [als → outer]
      같은 key, value만 교체

Case B: als1, als2 순차
Map: [als1 → {a:1}] → [als1 → {a:1}, als2 → {b:2}] → [als1 → {a:1}]
      다른 key, 독립적으로 추가/제거
```

---

## I/O 작업과 Context 전파

Thread pool에서 I/O 작업이 실행될 때는 어떻게 context가 유지될까요?

### Thread Pool이 Context를 모르는 이유

Thread Pool은 "순수 계산기"일 뿐입니다. JavaScript 객체에 접근할 수 없으므로 context를 알 수도, 알 필요도 없습니다.

```
┌─────────────────────────────────────────────────────┐
│  Main Thread                                        │
│                                                     │
│  crypto.pbkdf2(password, salt, ..., callback) {    │
│    // 1. 요청 데이터를 Native 메모리에 복사         │
│    uv_work_t* req = {                              │
│      password: "abc",     // 복사된 문자열          │
│      salt: "xyz",         // 복사된 문자열          │
│      iterations: 100000,  // 숫자 값                │
│    }                                                │
│    // context는 복사 안 함! (JS 객체니까)           │
│                                                     │
│    // 2. Thread Pool에 작업 요청                    │
│    uv_queue_work(req)                              │
│  }                                                  │
└──────────────────┬──────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────┐
│  Thread Pool (Worker Thread)                        │
│                                                     │
│  void DoWork(uv_work_t* req) {                     │
│    // 받은 것: password, salt, iterations          │
│    // 모르는 것: 이게 어떤 HTTP 요청인지            │
│    //           AsyncLocalStorage가 뭔지            │
│    //           JavaScript가 뭔지 (!)               │
│                                                     │
│    // 그냥 수학 계산만 수행                          │
│    result = PBKDF2_derive(password, salt, ...)     │
│  }                                                  │
└─────────────────────────────────────────────────────┘
```

**왜 context를 안 넘기나?**
- **넘길 수 없음**: JS 객체는 V8 Heap에만 존재
- **넘길 필요 없음**: 순수 계산에 context가 왜 필요?
- **설계 철학**: Thread Pool은 stateless 계산기

### AsyncResource의 역할

**`lib/async_hooks.js:184, 206-219`**

```javascript
constructor(type, opts = kEmptyObject) {
  // ...
  // 생성 시점의 context 저장!
  this[contextFrameSymbol] = AsyncContextFrame.current();
  // ...
}

runInAsyncScope(fn, thisArg, ...args) {
  const asyncId = this[async_id_symbol];
  emitBefore(asyncId, this[trigger_async_id_symbol], this);

  // 저장된 context 복원!
  const contextFrame = this[contextFrameSymbol];
  const prior = AsyncContextFrame.exchange(contextFrame);
  try {
    return ReflectApply(fn, thisArg, args);  // 함수 실행
  } finally {
    AsyncContextFrame.set(prior);  // 원상복구
    if (hasAsyncIdStack())
      emitAfter(asyncId);
  }
}
```

### Main Thread로 복귀하는 방법

libuv의 이벤트 기반 알림 시스템을 통해 Main Thread로 복귀합니다:

```
┌─────────────────────────────────────────────────────┐
│  Main Thread                                        │
│                                                     │
│  ┌─────────────────────────────────────┐           │
│  │  Event Loop                         │           │
│  │                                     │           │
│  │  while (true) {                     │           │
│  │    1. 타이머 체크                    │           │
│  │    2. I/O 폴링 (epoll/kqueue)       │  ←────┐   │
│  │    3. 완료된 작업 콜백 실행          │       │   │
│  │  }                                  │       │   │
│  └─────────────────────────────────────┘       │   │
└────────────────────────────────────────────────│───┘
                                                 │
        완료 알림 (pipe/eventfd)                  │
                                                 │
┌────────────────────────────────────────────────│───┐
│  Thread Pool                                   │   │
│                                                │   │
│  void DoWork() {                               │   │
│    result = 계산()                              │   │
│  }                                             │   │
│                                                │   │
│  void AfterWork() {  // 계산 완료 후            │   │
│    req->result = result                        │   │
│    uv_async_send(...)  ─────────────────────────┘   │
│    // Main Thread에게 "끝났어!" 알림              │
│  }                                             │   │
└────────────────────────────────────────────────────┘
```

**핵심 메커니즘:**

```c
// libuv 내부 (단순화)
void worker_thread() {
  while (true) {
    work = queue.pop()        // 작업 가져오기
    work->work_cb()           // 계산 수행 (DoWork)

    // 완료 알림을 위해 완료 큐에 추가
    completed_queue.push(work)
    uv_async_send(loop)       // Main Thread 깨우기!
  }
}

// Main Thread의 Event Loop
void event_loop() {
  while (true) {
    uv_run()  // I/O 이벤트 대기

    // Thread Pool 완료 알림이 오면:
    while (completed = completed_queue.pop()) {
      completed->after_work_cb()  // 콜백 실행 (Main Thread!)
    }
  }
}
```

### Crypto 예시: PBKDF2

**JavaScript 레벨** (`lib/internal/crypto/pbkdf2.js:45-60`):

```javascript
function pbkdf2(password, salt, iterations, keylen, digest, callback) {
  // ...
  const job = new PBKDF2Job(  // ← AsyncWrap 상속 (context 저장)
    kCryptoJobAsync,
    password,
    salt,
    iterations,
    keylen,
    digest);

  job.ondone = (err, result) => {
    if (err !== undefined)
      return FunctionPrototypeCall(callback, job, err);
    const buf = Buffer.from(result);
    return FunctionPrototypeCall(callback, job, null, buf);
  };

  job.run();  // Thread pool로 전송
}
```

**C++ 레벨** (`src/crypto/crypto_util.h:268-302`):

```cpp
void AfterThreadPoolWork(int status) override {
  Environment* env = AsyncWrap::env();  // ✅ Main Thread!

  // ✅ V8 Isolate 접근 가능
  v8::HandleScope handle_scope(env->isolate());
  v8::Context::Scope context_scope(env->context());

  // ... 결과 처리 ...

  // ✅ JavaScript 콜백 실행 (AsyncWrap이 context 복원)
  ptr->MakeCallback(env->ondone_string(), arraysize(args), args);
}
```

### 전체 흐름

```
┌────────────────────────────────────────────────┐
│         Main Thread (JavaScript)               │
│                                                │
│  als.run({ id: 'REQ-1' }, () => {             │
│    crypto.pbkdf2(...)                         │
│         │                                     │
│         └─ new PBKDF2Job()                    │
│            job[contextFrameSymbol] =          │
│              Map([[als, {id: 'REQ-1'}]])      │ ← 저장!
│  })                                            │
└────────────┬───────────────────────────────────┘
             │
             │ uv_queue_work()
             ▼
┌────────────────────────────────────────────────┐
│         libuv Thread Pool                      │
│                                                │
│  void DoWork(uv_work_t* req) {                │
│    ❌ V8 접근 불가                              │
│    ❌ JavaScript 객체 접근 불가                 │
│    ✅ 순수 계산만: PBKDF2_derive(...)          │
│  }                                             │
└────────────┬───────────────────────────────────┘
             │ 작업 완료, uv_async_send()
             ▼
┌────────────────────────────────────────────────┐
│    Main Thread (AfterThreadPoolWork)          │
│                                                │
│  void AfterWork() {                           │
│    PBKDF2Job* job = ...;                      │
│                                                │
│    // ✅ JavaScript 객체 접근!                 │
│    Local<Object> js_obj = job->object();      │
│                                                │
│    // ✅ 저장된 context 읽기!                  │
│    Local<Value> context =                     │
│      js_obj->Get(contextFrameSymbol);         │
│    // = Map([[als, {id: 'REQ-1'}]])           │
│                                                │
│    // ✅ Context 복원하고 콜백 실행!           │
│    job->MakeCallback(...);                    │
│    // → runInAsyncScope()                     │
│    //   → AsyncContextFrame.set(context)      │
│    //   → callback()                          │
│  }                                             │
└────────────────────────────────────────────────┘
```

**핵심**:
1. JavaScript 객체는 **Main Thread Heap**에 저장
2. Thread pool은 context를 **전혀 모름** (순수 계산만)
3. Main Thread로 복귀 시 JavaScript 객체에서 context **복원**

---

## Worker Thread와 메모리 격리

Node.js의 Worker Thread는 완전히 독립적인 Isolate를 가집니다.

### Worker Thread 예시

```javascript
const { Worker, isMainThread } = require('worker_threads');
const { AsyncLocalStorage } = require('async_hooks');

const als = new AsyncLocalStorage();

if (isMainThread) {
  // Main thread
  als.run({ thread: 'main', value: 100 }, () => {
    console.log('Main:', als.getStore()); // { thread: 'main', value: 100 }

    const worker = new Worker(__filename);
  });
} else {
  // Worker thread
  console.log('Worker:', als.getStore()); // undefined! (독립적인 Isolate)

  als.run({ thread: 'worker', value: 200 }, () => {
    console.log('Worker:', als.getStore()); // { thread: 'worker', value: 200 }
  });
}
```

### 메모리 구조

```
프로세스 메모리 공간
├─ Main Thread (Isolate #1)
│  └─ JavaScript Heap
│     └─ AsyncContextFrame: Map([[als, {thread: 'main'}]])
│
├─ Worker Thread (Isolate #2)
│  └─ JavaScript Heap (완전히 독립적!)
│     └─ AsyncContextFrame: Map([[als, {thread: 'worker'}]])
│
└─ libuv Native Memory (C++)
   └─ work_queue, mutex, cond (threads끼리 공유, mutex로 동기화)
```

### V8 Isolate 격리

V8 공식 문서 확인:

```cpp
/**
 * Isolate represents an isolated instance of the V8 engine. V8 isolates have
 * completely separate states. Objects from one isolate must not be used in
 * other isolates.
 */
```

**결론**:
- ✅ 각 Worker Thread는 별도의 V8 Isolate
- ✅ Isolate끼리 메모리 격리
- ✅ AsyncLocalStorage도 완전히 독립적

---

## 결론

AsyncLocalStorage가 request context를 안전하게 격리하는 이유를 정리하면:

### 1. V8 Engine: Single Thread
- JavaScript 코드는 절대 동시에 실행되지 않음
- Event loop가 순차적으로 처리
- Race condition 원천 차단

### 2. 1 Process = 1 Isolate
- 모든 Request가 같은 Isolate를 시분할로 사용
- 그래서 context 격리 메커니즘이 필수적

### 3. 슬롯과 Context의 분리
- 슬롯 = 저장 공간 (하나만 존재)
- Context = 슬롯에 저장되는 Map 객체
- undefined = "활성화된 context 없음" (context 자체가 없는 게 아님)

### 4. AsyncContextFrame: Map 기반 저장소
- AsyncLocalStorage 인스턴스가 키
- Copy-on-Write로 중첩 context 지원
- Request별로 독립적인 Map 인스턴스

### 5. Microtask 스냅샷
- 각 microtask는 생성 시점의 context(Map 전체) 저장
- Key와 Value 모두 포함
- 실행 시 Isolate의 슬롯에 복원
- 실행 완료 후 정리 및 GC 대상

### 6. 동기 vs 비동기
- 동기: 슬롯에서 직접 읽기
- 비동기: 저장 → 복원 → 정리 사이클

### 7. als.run()의 finally 블록
- 항상 이전 상태로 복원
- Request의 동기 코드 완료 후 context는 undefined

### 8. AsyncResource: I/O Context 바인딩
- JavaScript 객체에 context 저장
- Thread pool은 context 접근 불가 (V8 Heap 접근 불가)
- Main Thread로 복귀 시 복원

### 9. Worker Thread: 완전한 격리
- 각 Worker는 독립적인 Isolate
- 메모리 공유 없음 (SharedArrayBuffer 제외)

### 핵심 메커니즘 요약

```
[생성]
als.run() → AsyncContextFrame 생성 → Map([[als, data]])

[전파]
Promise/async → microtask 생성 → context 스냅샷 저장 (Map 전체, key 포함)

[실행]
microtask 실행 → Isolate 슬롯에 복원 → 코드 실행 → 정리 → GC

[I/O]
AsyncResource 생성 → context 저장 → Thread pool 작업 (context 모름)
→ uv_async_send() → Main Thread 복귀 → context 복원 → 콜백 실행

[격리]
Single Thread + Event Loop + Microtask Snapshot + finally 복원
= Race Condition 없음 + Context 혼선 없음
```

### 참고 자료

- [V8 API Documentation](https://v8.dev/docs)
- [libuv Documentation](https://docs.libuv.org/)
- [Node.js Async Hooks](https://nodejs.org/api/async_hooks.html)
- [Node.js Source Code](https://github.com/nodejs/node)

---

## 부록: 실험 코드

### 실험 1: Request 격리 확인

```javascript
const { AsyncLocalStorage } = require('async_hooks');
const als = new AsyncLocalStorage();

// Request 1
als.run({ id: 'req-1' }, () => {
  console.log('Request 1 sync:', als.getStore());  // { id: 'req-1' }

  Promise.resolve().then(() => {
    console.log('Request 1 microtask:', als.getStore());  // { id: 'req-1' }
  });
});

// Request 2 (즉시 실행)
als.run({ id: 'req-2' }, () => {
  console.log('Request 2 sync:', als.getStore());  // { id: 'req-2' }

  Promise.resolve().then(() => {
    console.log('Request 2 microtask:', als.getStore());  // { id: 'req-2' }
  });
});

// 출력:
// Request 1 sync: { id: 'req-1' }
// Request 2 sync: { id: 'req-2' }
// Request 1 microtask: { id: 'req-1' }  ← 섞이지 않음!
// Request 2 microtask: { id: 'req-2' }  ← 섞이지 않음!
```

### 실험 2: Thread Pool 동시 실행

```javascript
const crypto = require('crypto');

const start = Date.now();
const tasks = [];

// 4개 작업 동시 시작
for (let i = 1; i <= 4; i++) {
  tasks.push(new Promise((resolve) => {
    crypto.pbkdf2('password', 'salt', 100000, 64, 'sha512', () => {
      resolve(Date.now() - start);
    });
  }));
}

Promise.all(tasks).then((durations) => {
  console.log('소요 시간:', durations);
  // 출력: [65ms, 77ms, 78ms, 80ms] ← 거의 동시!
  // 결론: libuv thread pool이 병렬 실행
});
```

### 실험 3: Worker Thread 격리

```javascript
const { Worker, isMainThread } = require('worker_threads');
const { AsyncLocalStorage } = require('async_hooks');

const als = new AsyncLocalStorage();

if (isMainThread) {
  als.run({ value: 'main' }, () => {
    console.log('Main:', als.getStore());  // { value: 'main' }
    new Worker(__filename);
  });
} else {
  console.log('Worker:', als.getStore());  // undefined
  als.run({ value: 'worker' }, () => {
    console.log('Worker:', als.getStore());  // { value: 'worker' }
  });
}

// 출력:
// Main: { value: 'main' }
// Worker: undefined  ← 완전히 독립적!
// Worker: { value: 'worker' }
```

### 실험 4: als 중첩 테스트

```javascript
const { AsyncLocalStorage } = require('async_hooks');
const als = new AsyncLocalStorage();

als.run({ level: 'outer', value: 1 }, () => {
  console.log('Outer:', als.getStore());  // { level: 'outer', value: 1 }

  als.run({ level: 'inner', value: 2 }, () => {
    console.log('Inner:', als.getStore());  // { level: 'inner', value: 2 }
  });

  console.log('Back to outer:', als.getStore());  // { level: 'outer', value: 1 }
});

// 출력:
// Outer: { level: 'outer', value: 1 }
// Inner: { level: 'inner', value: 2 }
// Back to outer: { level: 'outer', value: 1 }  ← 복원됨!
```

### 실험 5: 다중 als 테스트

```javascript
const { AsyncLocalStorage } = require('async_hooks');
const als1 = new AsyncLocalStorage();
const als2 = new AsyncLocalStorage();

als1.run({ name: 'als1', value: 100 }, () => {
  console.log('als1:', als1.getStore());  // { name: 'als1', value: 100 }
  console.log('als2:', als2.getStore());  // undefined

  als2.run({ name: 'als2', value: 200 }, () => {
    console.log('als1:', als1.getStore());  // { name: 'als1', value: 100 } ← 유지!
    console.log('als2:', als2.getStore());  // { name: 'als2', value: 200 }
  });

  console.log('als1:', als1.getStore());  // { name: 'als1', value: 100 }
  console.log('als2:', als2.getStore());  // undefined ← 복원됨
});

// 출력:
// als1: { name: 'als1', value: 100 }
// als2: undefined
// als1: { name: 'als1', value: 100 }
// als2: { name: 'als2', value: 200 }
// als1: { name: 'als1', value: 100 }
// als2: undefined
```

---

이 글을 통해 AsyncLocalStorage의 내부 동작 원리와 context 격리 메커니즘을 깊이 이해하셨기를 바랍니다. 질문이나 피드백은 언제든 환영합니다!
