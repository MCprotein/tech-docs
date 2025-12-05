# Node.js 스레드 심화: Worker Threads와 메모리 모델

## 목차

1. [Worker Threads vs Java(Spring) 멀티스레드](#1-worker-threads-vs-javaspring-멀티스레드)
2. [Worker Thread가 OS 스레드인데 왜 메모리 격리?](#2-worker-thread가-os-스레드인데-왜-메모리-격리)
3. [V8 인스턴스 생성 비용이 무거운 이유](#3-v8-인스턴스-생성-비용이-무거운-이유)
4. [JavaScript 객체는 왜 공유가 안 되나?](#4-javascript-객체는-왜-공유가-안-되나)
5. [Worker 간 통신: MessagePort와 Structured Clone](#5-worker-간-통신-messageport와-structured-clone)
6. [SharedArrayBuffer 심화](#6-sharedarraybuffer-심화)
7. [V8 Locker: 스레드 안전성 보장](#7-v8-locker-스레드-안전성-보장)
8. [면접 한 줄 정리](#8-면접-한-줄-정리)

---

## 1. Worker Threads vs Java(Spring) 멀티스레드

### Java (Spring)

```java
@Service
public class MyService {
    private int sharedCounter = 0; // 모든 스레드가 접근 가능

    public synchronized void increment() {
        sharedCounter++; // 락 걸고 수정
    }
}
```

- JVM 위에서 OS 스레드 생성
- 모든 스레드가 같은 JVM 힙 메모리 공유
- `synchronized`, `Lock` 등으로 동기화 필수
- 동기화 안 하면 race condition

### Node.js Worker Threads

```javascript
// main.js
import { Worker } from 'worker_threads';

const worker = new Worker('./worker.js');
worker.postMessage({ data: 'heavy task' });
worker.on('message', (result) => console.log(result));

// worker.js
import { parentPort } from 'worker_threads';

parentPort?.on('message', (data) => {
  parentPort?.postMessage('done');
});
```

- OS 스레드 맞음
- 하지만 각 Worker가 **독립된 V8 인스턴스** 생성
- 각 V8이 **자기만의 힙** 관리 → 기본적으로 격리
- 메시지 패싱으로 통신

### 비교

| 구분 | Java (Spring) | Node.js Worker |
|------|---------------|----------------|
| 스레드 종류 | OS 스레드 | OS 스레드 |
| 메모리 | JVM 힙 공유 | V8 힙 격리 |
| 통신 방식 | 직접 메모리 접근 | 메시지 패싱 (Structured Clone) |
| 동기화 | synchronized, Lock 필수 | 불필요 (격리됨) |
| 생성 비용 | 상대적으로 가벼움 | 무거움 (V8 인스턴스) |

---

## 2. Worker Thread가 OS 스레드인데 왜 메모리 격리?

### OS 스레드의 기본 특성

- 같은 프로세스 내 스레드들은 메모리 **공유 가능**

### Node.js Worker Thread의 실제 동작

- OS 스레드 맞음
- 같은 프로세스 안에 있음
- **하지만** 각 Worker가 독립된 V8 인스턴스 생성
- V8이 자기만의 힙을 따로 관리 → **V8 레벨에서 격리**

```
┌─────────────────── Node.js 프로세스 ───────────────────┐
│                    (OS 메모리 공간)                     │
│                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │ V8 Instance  │  │ V8 Instance  │  │ V8 Instance  │ │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │ │
│  │ │ V8 Heap  │ │  │ │ V8 Heap  │ │  │ │ V8 Heap  │ │ │
│  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │ │
│  │  Main Thread │  │   Worker 1   │  │   Worker 2   │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
│         ↑                 ↑                 ↑         │
│         └─────── OS 레벨에선 같은 공간 ──────┘         │
│                  V8이 각자 힙을 격리해서 관리            │
└────────────────────────────────────────────────────────┘
```

### Node.js 내부 코드로 검증

**`src/node_worker.cc:184-194`** - Worker가 새로운 Isolate를 생성하는 코드:

```cpp
// 새로운 ArrayBuffer 할당자 생성
std::shared_ptr<ArrayBufferAllocator> allocator =
    ArrayBufferAllocator::Create();

Isolate::CreateParams params;
SetIsolateCreateParamsForNode(&params);
params.array_buffer_allocator_shared = allocator;

// ⭐ 핵심: 새로운 V8 Isolate 생성
Isolate* isolate =
    NewIsolate(&params, &loop_, w->platform_, w->snapshot_data());

SetIsolateUpForNode(isolate);  // Node.js 설정 적용
```

**`src/api/environment.cc:326-375`** - NewIsolate 함수:

```cpp
Isolate* NewIsolate(Isolate::CreateParams* params,
                    uv_loop_t* event_loop,
                    MultiIsolatePlatform* platform,
                    const SnapshotData* snapshot_data,
                    const IsolateSettings& settings) {
  IsolateGroup group = GetOrCreateIsolateGroup();

  // V8 Isolate 메모리 할당
  Isolate* isolate = Isolate::Allocate(group);
  if (isolate == nullptr) return nullptr;

  // 플랫폼에 Isolate 등록
  platform->RegisterIsolate(isolate, event_loop);

  // Isolate 초기화
  Isolate::Initialize(isolate, *params);

  return isolate;
}
```

### 왜 이렇게 설계했나?

1. **JavaScript 객체에 락 메커니즘 없음**
2. **V8 인스턴스 자체가 스레드 안전하지 않음** (하나의 Isolate를 여러 스레드가 동시 접근하면 크래시)
3. **안전한 기본값 (격리) + 필요할 때만 공유 옵션 (SharedArrayBuffer)**

---

## 3. V8 인스턴스 생성 비용이 무거운 이유

Worker Thread 하나 만들 때마다 **완전히 새로운 V8 엔진**이 초기화됨.

### 초기화 순서 (Node.js 내부 코드 기준)

**`src/node_worker.cc:164-390`** - Worker::Run() 메서드에서 초기화 순서:

```
1. Event Loop 초기화           // uv_loop_init(&loop_)
         ↓
2. ArrayBufferAllocator 생성   // ArrayBufferAllocator::Create()
         ↓
3. V8 Isolate 생성             // NewIsolate(&params, ...)
         ↓
4. SetIsolateUpForNode         // 콜백, 핸들러 설정
         ↓
5. IsolateData 생성            // IsolateData::CreateIsolateData()
   (내장 객체, 심볼, 문자열 등)
         ↓
6. Context 생성                // Context::FromSnapshot() 또는 NewContext()
         ↓
7. Environment 생성            // CreateEnvironment()
   (EventLoop, Timer 등)
         ↓
8. 모듈 시스템 초기화           // LoadEnvironment()
```

### 초기화되는 것들

| 구성 요소 | 코드 위치 | 설명 |
|-----------|-----------|------|
| Event Loop | `node_worker.cc:167` | `uv_loop_init(&loop_)` |
| Allocator | `node_worker.cc:178-179` | `ArrayBufferAllocator::Create()` |
| Isolate | `node_worker.cc:184-185` | `NewIsolate()` |
| IsolateData | `node_worker.cc:209-215` | 내장 객체, 심볼, 상수 |
| Context | `node_worker.cc:351/361` | V8 실행 컨텍스트 |
| Environment | `node_worker.cc:379-390` | Node.js 환경 |

### 메모리 할당

**`src/node_worker.cc:176-183`**:

```cpp
// 스택 크기 제한 설정 (기본값 4MB)
size_t stack_size_ = 4 * 1024 * 1024;

// 새로운 ArrayBuffer 할당자 생성
std::shared_ptr<ArrayBufferAllocator> allocator =
    ArrayBufferAllocator::Create();

Isolate::CreateParams params;
SetIsolateCreateParamsForNode(&params);
```

**`src/api/environment.cc:214-232`** - 힙 크기 설정:

```cpp
void SetIsolateCreateParamsForNode(Isolate::CreateParams* params) {
  const uint64_t total_memory = uv_get_total_memory();

  if (total_memory > 0 &&
      params->constraints.max_old_generation_size_in_bytes() == 0) {
    // V8이 시스템 메모리 기반으로 힙 크기 자동 설정
    params->constraints.ConfigureDefaults(total_memory, 0);
  }
}
```

### Java 스레드와 비교

```
Java 새 스레드 생성:
┌─────────────────────────────────────┐
│ new Thread()                        │
│  • 스택 메모리 할당 (~1MB)           │
│  • OS 스레드 생성                   │
│  • JVM 힙? 이미 있음 (공유)          │
└─────────────────────────────────────┘
→ ~0.5ms

Node.js 새 Worker 생성:
┌─────────────────────────────────────┐
│ new Worker()                        │
│  • Event Loop 초기화                │  ← 추가
│  • ArrayBufferAllocator 생성        │  ← 추가
│  • V8 Isolate 생성 및 초기화        │  ← 추가 (가장 무거움)
│  • IsolateData 생성                 │  ← 추가
│  • Context 생성                     │  ← 추가
│  • Environment 생성                 │  ← 추가
│  • 모듈 시스템 초기화               │  ← 추가
└─────────────────────────────────────┘
→ ~30-50ms
```

### Worker Thread 사용 가이드

그래서 Worker Thread는:

- 풀링해서 재사용하거나
- 정말 무거운 작업에만 사용
- 짧은 작업에 매번 생성하면 오히려 손해

```javascript
// ❌ 나쁜 예: 요청마다 Worker 생성
app.post('/process', (req, res) => {
  const worker = new Worker('./task.js'); // 매번 30ms 손해
  // ...
});

// ✅ 좋은 예: Worker Pool 사용
import { Pool } from 'workerpool';
const pool = Pool('./worker.js', { maxWorkers: 4 });

app.post('/process', async (req, res) => {
  const result = await pool.exec('task', [req.body]);
  res.json(result);
});
```

---

## 4. JavaScript 객체는 왜 공유가 안 되나?

### Java 객체: 락 메커니즘 내장

```java
// Java 객체 내부 구조
class MyObject {
    Object header;      // ← 락 정보, GC 정보 포함 (Mark Word)
    int field1;
    String field2;
}
```

- 모든 Java 객체는 헤더에 락 메타데이터 있음
- JVM이 처음부터 멀티스레드 고려해서 설계

### JavaScript 객체: 락 없음

```javascript
// JS 객체 내부 구조 (V8)
{
    hidden_class: 0x1234,  // 타입 정보
    properties: { ... },
    elements: [ ... ],
    // 락? 없음!
}
```

### 공유 시 발생하는 문제들

#### 문제 1: Race Condition

```javascript
// 스레드 1                    // 스레드 2 (동시에)
obj.count = obj.count + 1;    obj.count = obj.count + 1;

// 내부 동작:
// 스레드1: 읽기(10) → 더하기(11) → 쓰기
// 스레드2: 읽기(10) → 더하기(11) → 쓰기
//          ↑ 스레드1이 아직 안 씀!

// 결과: 12여야 하는데 11 → Race Condition!
```

#### 문제 2: 포인터 주소 체계

```javascript
const obj = { name: "kim", age: 30 };
```

메모리 구조:

```
┌─────────────────────────────────────────┐
│ HiddenClass 포인터 → (타입 정보)        │
│ Properties 포인터 → (다른 메모리 주소)   │
│   ├→ "name" → 문자열 포인터 → "kim"     │
│   └→ "age" → 30                         │
└─────────────────────────────────────────┘
```

- 포인터들이 복잡하게 얽혀있음
- 각 V8 인스턴스마다 메모리 주소 체계가 다름
- 공유하면 포인터가 엉뚱한 곳 가리킴 → **크래시**

---

## 5. Worker 간 통신: MessagePort와 Structured Clone

Worker 간에는 직접 메모리 공유가 안 되므로 **MessagePort**를 통해 통신합니다.

### MessagePort 연결 과정

#### 1단계: Worker 생성 시 양방향 채널 설정

Worker가 생성될 때 **이미 양쪽 포트가 연결**됨:

**`src/node_worker.cc:83-90`** - Worker 생성자:

```cpp
// Main Thread에서 실행
MessagePort* parent_port = MessagePort::New(env, env->context());  // Main용 포트 생성

child_port_data_ = std::make_unique<MessagePortData>(nullptr);     // Worker용 데이터 생성
MessagePort::Entangle(parent_port, child_port_data_.get());        // ⭐ 서로 연결!
```

`Entangle()`은 두 포트를 `SiblingGroup`으로 묶어서 서로를 알 수 있게 함:

```cpp
// src/node_messaging.cc:646-649
void MessagePortData::Entangle(MessagePortData* a, MessagePortData* b) {
  auto group = std::make_shared<SiblingGroup>();
  group->Entangle({a, b});  // 두 포트를 같은 그룹에 등록
}

// src/node_messaging.cc:1564-1570
void SiblingGroup::Entangle(std::initializer_list<MessagePortData*> ports) {
  for (MessagePortData* data : ports) {
    ports_.insert(data);        // Set에 포트 추가
    data->group_ = shared_from_this();  // 포트가 그룹 참조
  }
}
```

```
Worker 생성 직후 (Worker 스레드 시작 전):
┌─────────────────────────────────────────────────────────────┐
│                      SiblingGroup                            │
│   ports_ = { parent_port_data, child_port_data }            │
│             ↑                    ↑                          │
│         Main Thread           (아직 Worker                   │
│          MessagePort           Thread 없음)                  │
└─────────────────────────────────────────────────────────────┘
```

#### 2단계: Worker 스레드 시작 시 포트 연결

**`src/node_worker.cc:441-458`**:

```cpp
bool Worker::CreateEnvMessagePort(Environment* env) {
  HandleScope handle_scope(isolate_);
  std::unique_ptr<MessagePortData> data;
  {
    Mutex::ScopedLock lock(mutex_);
    data = std::move(child_port_data_);  // Main에서 생성한 데이터를 Worker로 이동
  }

  // Worker Thread에서 MessagePort 생성 (기존 child_port_data_ 사용)
  MessagePort* child_port = MessagePort::New(env,
                                             env->context(),
                                             std::move(data));
  if (child_port != nullptr)
    env->set_message_port(child_port->object(isolate_));

  return child_port;
}
```

```
Worker 스레드 시작 후:
┌─────────────────────────────────────────────────────────────┐
│                      SiblingGroup                            │
│   ports_ = { parent_port_data, child_port_data }            │
│             ↑                    ↑                          │
│         Main Thread           Worker Thread                  │
│          MessagePort           MessagePort                   │
│         (parent_port)          (child_port)                  │
└─────────────────────────────────────────────────────────────┘
```

### postMessage 내부 동작

#### 메시지 전송 흐름

```cpp
// src/node_messaging.cc:930-963
Maybe<bool> MessagePort::PostMessage(...) {
  // 1. 메시지 직렬화 (Structured Clone)
  msg->Serialize(env, context, message_v, transfer_v, obj);

  // 2. SiblingGroup을 통해 상대방에게 전달
  data_->Dispatch(msg, &error);
}
```

#### SiblingGroup이 상대 포트를 찾아서 전달

**`src/node_messaging.cc:1515-1557`**:

```cpp
Maybe<bool> SiblingGroup::Dispatch(MessagePortData* source, ...) {
  // SiblingGroup에 등록된 모든 포트 순회
  for (MessagePortData* port : ports_) {
    if (port == source)  // 자기 자신은 스킵
      continue;

    // ⭐ 상대방의 incoming queue에 직접 추가
    port->AddToIncomingQueue(message);
  }
}
```

#### 상대 스레드 깨우기

**`src/node_messaging.cc:635-643`**:

```cpp
void MessagePortData::AddToIncomingQueue(std::shared_ptr<Message> message) {
  // ⭐ 다른 스레드에서 호출될 수 있음!
  Mutex::ScopedLock lock(mutex_);  // 뮤텍스로 큐 보호
  incoming_messages_.emplace_back(std::move(message));  // 큐에 추가

  if (owner_ != nullptr) {
    owner_->TriggerAsync();  // 상대 스레드의 event loop 깨우기
  }
}

// src/node_messaging.cc:706-708
void MessagePort::TriggerAsync() {
  uv_async_send(&async_);  // ⭐ libuv async handle로 신호 전송
}
```

### 전체 메시지 전달 과정 시각화

```
Main Thread                              Worker Thread
    │                                        │
    │  worker.postMessage(data)              │
    ▼                                        │
┌──────────────────┐                         │
│ MessagePort      │                         │
│  ::PostMessage() │                         │
└────────┬─────────┘                         │
         │                                   │
         ▼                                   │
┌──────────────────┐                         │
│ Message::        │                         │
│  Serialize()     │  ← Structured Clone     │
└────────┬─────────┘                         │
         │                                   │
         ▼                                   │
┌──────────────────┐                         │
│ SiblingGroup     │                         │
│  ::Dispatch()    │                         │
│  ports_에서       │                         │
│  상대방 찾음      │                         │
└────────┬─────────┘                         │
         │                                   │
         ▼                                   │
┌──────────────────┐                         │
│ child_port_data  │                         │
│ ->AddToIncoming  │                         │
│    Queue()       │                         │
│                  │                         │
│ Mutex lock       │                         │
│ queue.push(msg)  │                         │
└────────┬─────────┘                         │
         │                                   │
         ▼                                   │
┌──────────────────┐    libuv async          │
│ uv_async_send()  │─────────────────────────▶ Event Loop 깨어남
└──────────────────┘                         │
                                             ▼
                                    ┌──────────────────┐
                                    │ MessagePort      │
                                    │  ::OnMessage()   │
                                    └────────┬─────────┘
                                             │
                                             ▼
                                    ┌──────────────────┐
                                    │ Message::        │
                                    │  Deserialize()   │
                                    └────────┬─────────┘
                                             │
                                             ▼
                                    parentPort.on('message')
                                         콜백 실행
```

### postMessage 핵심 정리

| 질문 | 답변 |
|------|------|
| Worker가 상대 정보를 가지고 생성? | **Yes**. 생성 시 `Entangle()`로 양방향 연결 |
| 연결 정보 저장 위치? | `SiblingGroup`의 `ports_` (Set 자료구조) |
| 메시지 전달 방식? | 상대방의 `incoming_messages_` 큐에 직접 push |
| 스레드 간 동기화? | `Mutex`로 큐 보호 |
| 상대 스레드 깨우기? | `uv_async_send()` (libuv async handle) |

### Structured Clone Algorithm

`postMessage()`로 데이터를 전송하면 **Structured Clone**이 수행됨. 이 알고리즘은 객체 타입에 따라 **세 가지 방식**으로 처리한다:

```
┌─────────────────────────────────────────────────────────────┐
│                 Structured Clone Algorithm                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. 일반 객체 (Object, Array, Map...)                        │
│     → Deep Copy (V8 Heap에 새로 할당)                        │
│                                                              │
│  2. Transferable (ArrayBuffer, MessagePort...)              │
│     → 소유권 이전 (원본 Detach, BackingStore 포인터 이동)      │
│                                                              │
│  3. SharedArrayBuffer                                        │
│     → 공유 (OS 공유 메모리 포인터만 전달, 양쪽 접근 가능)       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

#### 기본 동작: Deep Copy

```javascript
// main.js
const worker = new Worker('./worker.js');

const obj = { name: 'kim', data: [1, 2, 3] };
worker.postMessage(obj);  // 객체 복사됨 (원본과 별개)

obj.name = 'lee';  // Worker에 영향 없음
```

```
Main Thread (V8 Heap A)          Worker Thread (V8 Heap B)
┌─────────────┐                 ┌─────────────┐
│ { name:     │   Structured   │ { name:     │
│   'kim',    │ ───Clone───→   │   'kim',    │  ← 새로운 V8 Heap 메모리에 복사
│   data:     │                │   data:     │
│   [1,2,3] } │                │   [1,2,3] } │
└─────────────┘                └─────────────┘
    원본                           복사본
    ↓                              (완전히 독립)
 수정해도
 Worker에
 영향 없음
```

#### 세 가지 처리 방식 비교

```javascript
const data = { name: "test" };              // 일반 객체
const buffer = new ArrayBuffer(1024);       // Transferable
const shared = new SharedArrayBuffer(1024); // SharedArrayBuffer

worker.postMessage(
  { data, buffer, shared },  // 메시지
  [buffer]                    // transfer list (소유권 이전할 것들)
);

// 전송 후:
console.log(data);              // { name: "test" } - 그대로 (복사본 전송됨)
console.log(buffer.byteLength); // 0 - Detach됨 (소유권 이전)
console.log(shared.byteLength); // 1024 - 그대로! (공유)
```

| 타입 | transfer list 포함 | 동작 | 원본 상태 | 메모리 |
|------|-------------------|------|----------|--------|
| 일반 객체 | 상관없음 | **복사** | 유지 | V8 Heap에 새로 할당 |
| ArrayBuffer | X | **복사** | 유지 | V8 Heap에 새로 할당 |
| ArrayBuffer | O | **이전** | Detach | 동일 BackingStore (OS 메모리) |
| SharedArrayBuffer | 상관없음 | **공유** | 유지 | 동일 OS 공유 메모리 |

### Structured Clone이 지원하는 타입

| 지원 O | 지원 X |
|--------|--------|
| 원시값 (number, string, boolean) | Function |
| Array, Object | Symbol |
| Date, RegExp | WeakMap, WeakSet |
| Map, Set | DOM 노드 |
| ArrayBuffer, TypedArray | Error (일부만) |
| **SharedArrayBuffer** (공유!) | Proxy |

### Transferable Objects: 소유권 이전 메커니즘

복사 대신 **소유권 이전**:

```javascript
const buffer = new ArrayBuffer(1024);

// 복사가 아니라 소유권 이전
worker.postMessage(buffer, [buffer]);

console.log(buffer.byteLength);  // 0 (더 이상 사용 불가!)
```

#### 왜 가능한가? BackingStore의 비밀

ArrayBuffer는 **두 부분**으로 구성됨:

1. **V8 Heap 객체**: ArrayBuffer의 메타데이터 (byteLength, 포인터 등)
2. **BackingStore**: 실제 바이너리 데이터가 저장된 OS 프로세스 메모리 (malloc)

```
┌─────────────────────────────────────────────────────────────────┐
│                        V8 Isolate A                             │
│  ┌─────────────────────┐                                        │
│  │   ArrayBuffer       │                                        │
│  │   (V8 Heap 객체)    │─────┐                                  │
│  │   byteLength: 1024  │     │  포인터                          │
│  └─────────────────────┘     │                                  │
└──────────────────────────────│──────────────────────────────────┘
                               │
                               ▼
        ┌──────────────────────────────────────────┐
        │      BackingStore (OS 프로세스 메모리)     │
        │    0x7fff12340000 ~ 0x7fff12340400       │
        │     (malloc으로 할당된 raw 메모리)         │
        │     ← V8 Heap 바깥! Isolate 경계 넘을 수 있음
        └──────────────────────────────────────────┘
```

**핵심**: BackingStore는 **V8 Heap 바깥**에 있다. V8 Heap은 Isolate별로 격리되지만, BackingStore는 일반 OS 프로세스 메모리라서 Isolate 경계를 넘을 수 있다.

#### Node.js 내부 구현

**송신측 - BackingStore 추출 + Detach** (`src/node_messaging.cc:594-601`):

```cpp
for (Local<ArrayBuffer> ab : array_buffers) {
  // BackingStore 포인터 획득 (OS 메모리 주소)
  std::shared_ptr<BackingStore> backing_store = ab->GetBackingStore();

  // ArrayBuffer를 Detach (neutering) - V8 Heap의 객체를 사용 불가능하게 만듦
  if (ab->Detach(Local<Value>()).IsNothing()) {
    return Nothing<bool>();
  }

  // BackingStore를 Message 객체로 이동
  array_buffers_.emplace_back(std::move(backing_store));
}
```

Detach 후 원본 ArrayBuffer (V8 Heap 객체):
- `byteLength` → 0
- 내부 BackingStore 포인터 → null
- **더 이상 데이터에 접근 불가**

**수신측 - 새 ArrayBuffer 생성 (같은 BackingStore 사용)** (`src/node_messaging.cc:214-217`):

```cpp
// Worker의 V8 Heap에 새 ArrayBuffer 객체 생성, 기존 BackingStore 재사용
for (uint32_t i = 0; i < array_buffers_.size(); ++i) {
  Local<ArrayBuffer> ab =
      ArrayBuffer::New(env->isolate(), std::move(array_buffers_[i]));
  deserializer.TransferArrayBuffer(i, ab);
}
```

#### 전송 과정 시각화

```
[전송 전]
┌─────────────────────────────┐
│     V8 Isolate A (Main)     │
│     ┌── V8 Heap ──┐         │
│     │ ArrayBuffer │─────────┼───────┐
│     │ byteLength: │         │       │
│     │   1024      │         │       │
│     └─────────────┘         │       │
└─────────────────────────────┘       │
                                      ▼
                            ┌───────────────────┐
                            │   BackingStore    │
                            │   (OS 메모리)     │
                            │   0x7fff12340000  │
                            │   1024 bytes      │
                            └───────────────────┘

                    ─── postMessage(buffer, [buffer]) ───

[전송 후]
┌─────────────────────────────┐
│     V8 Isolate A (Main)     │
│     ┌── V8 Heap ──┐         │       ┌───────────────────┐
│     │ ArrayBuffer │         │       │   BackingStore    │
│     │ byteLength: │         │       │   (OS 메모리)     │
│     │   0         │ (Detach)│       │   0x7fff12340000  │
│     │ ptr: null   │         │       │   1024 bytes      │
│     └─────────────┘         │       └───────────────────┘
└─────────────────────────────┘                ▲
                                               │ 동일한 메모리!
┌─────────────────────────────┐                │
│     V8 Isolate B (Worker)   │                │
│     ┌── V8 Heap ──┐         │                │
│     │ ArrayBuffer │─────────┼────────────────┘
│     │ byteLength: │         │
│     │   1024      │         │
│     └─────────────┘         │
└─────────────────────────────┘
```

#### Transfer vs Share vs Copy 메모리 비교

| 구분 | 복사 (Copy) | 이전 (Transfer) | 공유 (Share) |
|------|-------------|-----------------|--------------|
| V8 Heap 객체 | 양쪽에 새로 생성 | 양쪽에 새로 생성 | 양쪽에 새로 생성 |
| 실제 데이터 위치 | **새 OS 메모리** | **동일 BackingStore** | **동일 OS 공유 메모리** |
| 원본 접근 | 가능 | **불가능 (Detach)** | 가능 |
| 동시 접근 | 불가능 (별개 메모리) | 불가능 (소유권 이전) | **가능 (Atomics 필요)** |
| 메모리 비용 | 2배 | 1배 | 1배 |

#### SharedArrayBuffer는 transfer 불가

```javascript
const sab = new SharedArrayBuffer(1024);

// SharedArrayBuffer는 transfer 불가 - 존재 목적이 공유이므로
worker.postMessage(sab, [sab]);
// TypeError: SharedArrayBuffer cannot be transferred

// 그냥 보내면 자동으로 공유됨
worker.postMessage({ sab });  // OK - 양쪽에서 접근 가능
```

---

## 6. SharedArrayBuffer 심화

### 메모리 구조

SharedArrayBuffer는 **V8 힙이 아니라 OS 레벨 공유 메모리**에 할당됨.

### Node.js 내부 코드

**`src/node_util.cc:443-465`** - SharedArrayBuffer 생성:

```cpp
void ConstructSharedArrayBuffer(const FunctionCallbackInfo<Value>& args) {
  Environment* env = Environment::GetCurrent(args);
  int64_t length;

  if (!args[0]->IntegerValue(env->context()).To(&length)) {
    return;
  }

  Local<SharedArrayBuffer> sab;
  // V8 엔진이 OS 레벨 공유 메모리 할당
  if (!SharedArrayBuffer::MaybeNew(env->isolate(), static_cast<size_t>(length))
           .ToLocal(&sab)) {
    env->ThrowRangeError("Array buffer allocation failed");
    return;
  }
  args.GetReturnValue().Set(sab);
}
```

**`lib/internal/worker.js:112-117`** - 실제 사용 예:

```javascript
// cwdCounter는 SharedArrayBuffer로 Worker와 Main Thread 간 공유
if (isMainThread) {
  cwdCounter = new Uint32Array(constructSharedArrayBuffer(4));

  const originalChdir = process.chdir;
  process.chdir = function(path) {
    AtomicsAdd(cwdCounter, 0, 1);  // 원자적 증가
    originalChdir(path);
  };
}
```

### 메모리 레이아웃

```
┌─────────────────── Node.js 프로세스 ───────────────────┐
│                                                        │
│  ┌──────────────┐  ┌──────────────┐                   │
│  │ V8 Instance  │  │ V8 Instance  │                   │
│  │ ┌──────────┐ │  │ ┌──────────┐ │                   │
│  │ │ V8 Heap  │ │  │ │ V8 Heap  │ │  ← 각자 격리      │
│  │ │          │ │  │ │          │ │                   │
│  │ │ ref:0xABC│─│──│─│ ref:0xABC│ │  ← 포인터만 저장  │
│  │ └──────────┘ │  │ └──────────┘ │                   │
│  └──────────────┘  └──────────────┘                   │
│                         │                              │
│                         ▼                              │
│  ┌─────────────────────────────────────────────────┐  │
│  │  OS 공유 메모리 영역 (mmap)            0xABC    │  │
│  │  ┌──┬──┬──┬──┬──┬──┬──┬──┐                     │  │
│  │  │00│00│00│2A│00│00│00│00│  ← 실제 바이트 데이터│  │
│  │  └──┴──┴──┴──┴──┴──┴──┴──┘                     │  │
│  └─────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
```

### Stack vs Heap 저장

```javascript
function example() {
  const buffer = new SharedArrayBuffer(8);
  const view = new Int32Array(buffer);
}
```

```
STACK
┌─────────────────┐
│ buffer: 0x001 ──────┐        V8 HEAP
│ view:   0x002 ──────│───┐    ┌─────────────────────┐
└─────────────────┘    │   │    │ SharedArrayBuffer   │
       ↑               │   │    │ 객체 (메타데이터)    │
   참조(포인터)만       │   │    │ └→ ptr: 0xABC      │ ← OS 공유 메모리 주소
                       │   │    │                     │
                       │   └───→│ Int32Array 객체     │
                       │        │ └→ buffer ref      │
                       │        └─────────────────────┘
                       │
                       ▼
              OS 공유 메모리 (0xABC)
              ┌──┬──┬──┬──┬──┬──┬──┬──┐
              │  실제 바이너리 데이터   │
              └──┴──┴──┴──┴──┴──┴──┴──┘
```

**정리:**

| 위치 | 저장되는 것 |
|------|-------------|
| Stack | 참조(포인터)만 |
| V8 Heap | SharedArrayBuffer 객체 메타데이터 |
| OS 공유 메모리 | 실제 바이너리 데이터 (이 부분만 공유됨) |

### 왜 원시 바이너리만 공유 가능?

| 구분 | JS 객체 | 원시 바이너리 |
|------|---------|---------------|
| 포인터 | 복잡하게 얽혀있음 | 없음 |
| 주소 체계 | V8마다 다름 | 상관없음 |
| `42`라는 값 | 객체에 따라 다르게 저장 | 어디서 봐도 42 |

### TypedArray로 해석하기

```javascript
const buffer = new SharedArrayBuffer(16); // 16바이트 메모리

// 같은 메모리를 다르게 해석
const int32View = new Int32Array(buffer);   // 4바이트씩 → 정수 4개
const int8View = new Int8Array(buffer);     // 1바이트씩 → 정수 16개
const float64View = new Float64Array(buffer); // 8바이트씩 → 실수 2개
```

```
SharedArrayBuffer (16바이트)
┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
│00│01│02│03│04│05│06│07│08│09│10│11│12│13│14│15│
└──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘

Int8Array:    [0][1][2][3][4][5][6][7][8][9][10][11][12][13][14][15]
Int32Array:   [    0    ][    1    ][    2    ][    3    ]
Float64Array: [        0        ][        1        ]
```

### Atomics: 안전한 공유 메모리 조작

```javascript
const buffer = new SharedArrayBuffer(4);
const view = new Int32Array(buffer);

// ❌ Race condition 가능
view[0] = view[0] + 1;

// ✅ 원자적 연산 (안전)
Atomics.add(view, 0, 1);                    // 원자적으로 1 더하기
Atomics.load(view, 0);                      // 원자적으로 읽기
Atomics.store(view, 0, 100);                // 원자적으로 쓰기
Atomics.compareExchange(view, 0, 100, 200); // CAS 연산

// 동기화 (락처럼 사용)
Atomics.wait(view, 0, 0);     // 값이 0이면 대기
Atomics.notify(view, 0, 1);   // 대기 중인 스레드 1개 깨우기
```

### 사용 예시

```javascript
// 메인 스레드
import { Worker } from 'worker_threads';

const sharedBuffer = new SharedArrayBuffer(4);
const sharedArray = new Int32Array(sharedBuffer);
sharedArray[0] = 0;

const worker = new Worker('./worker.js', {
  workerData: { sharedBuffer }
});

setInterval(() => {
  console.log('Main sees:', Atomics.load(sharedArray, 0));
}, 100);
```

```javascript
// worker.js
import { workerData } from 'worker_threads';

const sharedArray = new Int32Array(workerData.sharedBuffer);

setInterval(() => {
  Atomics.add(sharedArray, 0, 1); // 메인에서도 변경 보임!
}, 50);
```

---

## 7. V8 Locker: 스레드 안전성 보장

V8 Isolate는 기본적으로 스레드 안전하지 않음. Node.js는 **Locker**를 사용해 보호.

### 내부 코드

**`src/node_worker.cc:313-314`**:

```cpp
// Worker 실행 시 Isolate Locker 사용
Locker locker(isolate_);           // 이 스레드만 Isolate 접근 가능
Isolate::Scope isolate_scope(isolate_);

// 이 범위 안에서만 JS 코드 실행
// 다른 스레드가 이 Isolate에 접근하려 하면 블로킹됨
```

### Locker 동작 원리

```
Thread A                           Thread B
    │                                  │
    ▼                                  │
 Locker locker(isolate)               │
    │ (락 획득)                         │
    ▼                                  │
 JS 코드 실행 중...                    │
    │                                  ▼
    │                           Locker locker(isolate)
    │                                  │ (블로킹! A가 끝날 때까지 대기)
    ▼                                  │
 } // Locker 소멸자                    │
   (락 해제)                           │
                                       ▼
                                    (락 획득)
                                    JS 코드 실행
```

### 왜 필요한가?

- 하나의 Isolate를 여러 스레드가 동시에 접근하면 **V8이 크래시**
- Locker가 한 번에 하나의 스레드만 Isolate에 접근하게 보장
- 각 Worker가 독립된 Isolate를 가지므로 보통은 충돌 없음
- **하지만** MessagePort로 메시지 전달 시 잠깐 동안 접근할 수 있어서 필요

---

## 8. 면접 한 줄 정리

| 질문 | 답변 |
|------|------|
| Worker Thread가 OS 스레드? | 맞음. 단, V8 레벨에서 힙 격리 |
| 왜 힙 격리? | JS 객체에 락 없음 + 포인터 주소 체계 다름 + V8이 스레드 안전하지 않음 |
| Java와 차이? | Java는 힙 공유 + synchronized, Node는 힙 격리 + 메시지 패싱 |
| Worker 생성 비용 큰 이유? | V8 Isolate, IsolateData, Context, Environment 전체 초기화 |
| Worker 간 통신? | MessagePort + Structured Clone (객체 복사) |
| SharedArrayBuffer 저장 위치? | 실제 데이터는 OS 공유 메모리, V8 힙엔 메타데이터만 |
| 공유하려면? | SharedArrayBuffer + Atomics |
| V8 Locker? | Isolate 접근을 한 스레드로 제한하는 락 메커니즘 |

---

## 부록: Cluster vs Worker Threads

Node.js에서 병렬 처리하는 두 가지 방법:

### Cluster 모드

```javascript
import cluster from 'cluster';
import os from 'os';

if (cluster.isPrimary) {
  // 마스터 프로세스
  const numCPUs = os.cpus().length;
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork(); // 워커 프로세스 생성
  }
} else {
  // 워커 프로세스
  // 각자 HTTP 서버 실행
}
```

- **프로세스** 레벨 병렬화
- 완전히 독립된 메모리
- 주로 HTTP 서버 스케일링에 사용
- PM2가 내부적으로 이거 사용

### Worker Threads

```javascript
import { Worker } from 'worker_threads';

const worker = new Worker('./heavy-task.js');
```

- **스레드** 레벨 병렬화
- 같은 프로세스, V8 힙은 격리
- 주로 CPU 집약적 작업에 사용
- Cluster보다 가벼움 (그래도 무거움)

### 비교

| 구분 | Cluster | Worker Threads |
|------|---------|----------------|
| 단위 | 프로세스 | 스레드 |
| 메모리 | 완전 독립 | V8 힙 격리, SharedArrayBuffer로 공유 가능 |
| 생성 비용 | 매우 무거움 (프로세스 fork) | 무거움 (V8 Isolate 생성) |
| 용도 | HTTP 서버 스케일링 | CPU 집약적 작업 오프로드 |
| 통신 | IPC (프로세스 간) | postMessage, SharedArrayBuffer |

---

## 참고 자료

- [Node.js Worker Threads Documentation](https://nodejs.org/api/worker_threads.html)
- [V8 Isolates](https://v8.dev/docs/embed)
- [MDN SharedArrayBuffer](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer)
- [MDN Atomics](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics)
- Node.js Source Code: `src/node_worker.cc`, `src/api/environment.cc`
