# Node.js AsyncLocalStorage 완전 분석: Request Context가 섞이지 않는 이유

## 목차
1. [서론](#서론)
2. [AsyncLocalStorage란?](#asynclocalstorage란)
3. [핵심 구조: AsyncContextFrame](#핵심-구조-asynccontextframe)
4. [V8 엔진의 Continuation Preserved Embedder Data](#v8-엔진의-continuation-preserved-embedder-data)
5. [Node.js 아키텍처: Single Thread vs Multi Thread](#nodejs-아키텍처-single-thread-vs-multi-thread)
6. [Context 격리 메커니즘](#context-격리-메커니즘)
7. [I/O 작업과 Context 전파](#io-작업과-context-전파)
8. [Worker Thread와 메모리 격리](#worker-thread와-메모리-격리)
9. [결론](#결론)

---

## 서론

Node.js에서 HTTP 요청마다 독립적인 컨텍스트를 유지하는 것은 오랫동안 어려운 문제였습니다. 특히 비동기 작업이 많은 환경에서 "이 로그는 어떤 요청에서 발생한 것인가?"를 추적하는 것은 매우 중요합니다.

AsyncLocalStorage는 이 문제를 해결하기 위해 등장했으며, Node.js v13.10.0부터 stable API가 되었습니다. 이 글에서는 AsyncLocalStorage가 어떻게 각 요청의 context를 격리하는지, 그 내부 동작 원리를 코드 레벨에서 깊이 분석합니다.

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

### 타임라인 분석

```
Time | Isolate 현재 값          | 이벤트
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
- 실행 시 **Isolate의 임시 슬롯에 복원**, 실행 후 정리

---

## I/O 작업과 Context 전파

Thread pool에서 I/O 작업이 실행될 때는 어떻게 context가 유지될까요?

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
             │ 작업 완료
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

### 2. AsyncContextFrame: Map 기반 저장소
- AsyncLocalStorage 인스턴스가 키
- Copy-on-Write로 중첩 context 지원
- Request별로 독립적인 Map 인스턴스

### 3. Microtask 스냅샷
- 각 microtask는 생성 시점의 context 저장
- 실행 시 Isolate의 슬롯에 복원
- 실행 완료 후 정리

### 4. als.run()의 finally 블록
- 항상 이전 상태로 복원
- Request의 동기 코드 완료 후 context는 undefined

### 5. AsyncResource: I/O Context 바인딩
- JavaScript 객체에 context 저장
- Thread pool은 context 접근 불가
- Main Thread로 복귀 시 복원

### 6. Worker Thread: 완전한 격리
- 각 Worker는 독립적인 Isolate
- 메모리 공유 없음 (SharedArrayBuffer 제외)

### 핵심 메커니즘 요약

```
[생성]
als.run() → AsyncContextFrame 생성 → Map([[als, data]])

[전파]
Promise/async → microtask 생성 → context 스냅샷 저장

[실행]
microtask 실행 → Isolate 슬롯에 복원 → 코드 실행 → 정리

[I/O]
AsyncResource 생성 → context 저장 → Thread pool 작업
→ Main Thread 복귀 → context 복원 → 콜백 실행

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

---

이 글을 통해 AsyncLocalStorage의 내부 동작 원리와 context 격리 메커니즘을 깊이 이해하셨기를 바랍니다. 질문이나 피드백은 언제든 환영합니다!
