# V8 엔진의 Async/Await 처리 분석

`async/await`는 V8 엔진에서 내부적으로 Promise 기반으로 처리된다. 이 문서는 V8 소스 코드를 분석하여 `await`가 어떻게 `Promise.then()`으로 변환되는지 설명한다.

## 목차

1. [사전 지식](#사전-지식) - Generator, Suspend/Resume, 콜스택에서 빠져나감
2. [핵심 요약](#핵심-요약)
3. [V8 내부 구현 분석](#v8-내부-구현-분석) - 콜스택/마이크로태스크 큐 시각화 포함
4. [실행 흐름 요약](#실행-흐름-요약)
5. [주요 파일 경로](#주요-파일-경로)

---

## 사전 지식

V8 코드를 이해하기 전에 알아야 할 개념들을 정리한다.

### JavaScript Generator란?

Generator는 **실행을 중간에 멈췄다가 나중에 다시 시작할 수 있는 함수**다.

```javascript
function* myGenerator() {
  console.log('1단계');
  yield 'first';           // 여기서 멈춤
  console.log('2단계');
  yield 'second';          // 여기서 멈춤
  console.log('3단계');
  return 'done';
}

const gen = myGenerator();
gen.next();  // "1단계" 출력, { value: 'first', done: false }
gen.next();  // "2단계" 출력, { value: 'second', done: false }
gen.next();  // "3단계" 출력, { value: 'done', done: true }
```

- `yield`: 함수 실행을 일시 정지(suspend)
- `next()`: 함수 실행을 재개(resume)

### Suspend/Resume 패턴

"Suspend/Resume"은 **함수 실행을 중단했다가 나중에 이어서 실행하는 패턴**이다.

```
[실행 중] → [suspend: 멈춤] → ... 시간 경과 ... → [resume: 재개] → [실행 중]
```

Generator에서:
- `yield` = suspend (멈춤)
- `next()` = resume (재개)

Async/Await에서:
- `await` = suspend (Promise 기다리며 멈춤)
- Promise 완료 = resume (재개)

#### "멈춤"의 실체: 콜스택에서 빠져나감

**중요한 오해**: `await`에서 함수가 "멈춰서 기다리는" 게 아니다.

실제로는:
1. `await`를 만나면 async 함수가 **콜스택에서 빠져나감** (return)
2. 상태(변수, 실행 위치)를 힙 메모리에 저장
3. Promise 완료 시 **마이크로태스크 큐**를 통해 다시 콜스택에 올라옴
4. 저장된 상태를 복원해서 이어서 실행

```
┌─────────────────────────────────────────────────────────────────┐
│  "중단" = 콜스택에서 빠져나감 + 상태 저장                          │
│  "재개" = 마이크로태스크 큐 → 콜스택 진입 + 상태 복원               │
└─────────────────────────────────────────────────────────────────┘
```

### 왜 async/await 얘기에 Generator가 나오는가?

**V8은 async/await를 Generator와 동일한 메커니즘으로 구현했기 때문이다.**

```javascript
// 이 코드는...
async function foo() {
  const a = await promise1;
  const b = await promise2;
  return a + b;
}

// 내부적으로 이런 식으로 동작한다 (개념적 설명)
function* foo() {
  const a = yield promise1;  // await → yield
  const b = yield promise2;  // await → yield
  return a + b;
}
```

둘 다 "함수 실행을 중간에 멈췄다가 나중에 재개"해야 하므로, V8은 같은 내부 구조(`Suspend` 클래스)를 공유한다.

### 단항 연산자 (Unary Operator)

**피연산자가 하나만 필요한 연산자**다.

```javascript
// 단항 연산자 (피연산자 1개)
-x        // 음수
!x        // 논리 NOT
typeof x  // 타입 확인
await x   // await도 단항 연산자!

// 이항 연산자 (피연산자 2개)
a + b
a * b
a && b
```

`await`는 `await somePromise`처럼 뒤에 값 하나만 받으므로 단항 연산자다.

### ** 연산자 (거듭제곱)

```javascript
2 ** 3    // 8 (2의 3승)
10 ** 2   // 100 (10의 2승)

// await와 ** 함께 사용 불가
await 2 ** 3  // SyntaxError!
(await 2) ** 3  // 이렇게 괄호로 감싸야 함
```

우선순위 문제로 `await x ** y`가 `await (x ** y)`인지 `(await x) ** y`인지 모호하기 때문에 금지된다.

### async* (Async Generator)

`async`와 `generator`를 합친 것이다.

```javascript
// 일반 async function
async function foo() {
  return await promise;
}

// async generator (async + generator)
async function* bar() {
  yield await promise1;
  yield await promise2;
}

// 사용법
const gen = bar();
await gen.next();  // { value: ..., done: false }
await gen.next();  // { value: ..., done: false }
```

async generator는 `await`와 `yield`를 둘 다 사용할 수 있다.

---

## 핵심 요약

### 한 줄 요약

> **`await value`는 V8 내부에서 `value.then(onResolve, onReject)` 호출로 변환된다.**

### 처리 과정 (단순화)

```javascript
// 1. 원본 코드
async function foo() {
  const result = await somePromise;
  return result;
}

// 2. V8이 내부적으로 하는 일 (개념적 설명)
function foo() {
  // async 함수 시작 시 Promise 생성
  const returnPromise = new Promise();

  // await를 만나면...
  somePromise.then(
    (result) => {
      // Promise가 성공하면 여기서 함수 재개
      // return result; 실행
      returnPromise.resolve(result);
    },
    (error) => {
      // Promise가 실패하면 여기서 에러 발생
      returnPromise.reject(error);
    }
  );

  return returnPromise;
}
```

### 핵심 코드 위치

V8 소스에서 `await`가 `Promise.then()`으로 변환되는 핵심 코드:

```
파일: src/builtins/builtins-async-gen.cc
라인: 159
```

```cpp
// await value를 Promise.then(onResolve, onReject)로 변환
CallBuiltin(Builtin::kPerformPromiseThen, native_context, value,
            on_resolve, on_reject, ...);
```

`kPerformPromiseThen`은 JavaScript의 `Promise.prototype.then()`을 호출하는 V8 내장 함수다.

---

## V8 내부 구현 분석

V8이 JavaScript 코드를 실행하는 과정:

```
JavaScript 코드
     ↓
[1. 파싱] 코드를 AST(트리 구조)로 변환
     ↓
[2. 바이트코드 생성] AST를 바이트코드로 컴파일
     ↓
[3. 실행] 바이트코드 실행 (필요시 머신코드로 최적화)
```

### 1. 파싱 단계: 코드를 트리로 변환

**파일**: `src/parsing/parser-base.h`

V8의 파서가 `await somePromise`를 만나면:

```cpp
// 줄 3793-3824
ExpressionT ParseAwaitExpression() {
  Consume(Token::kAwait);                    // 'await' 키워드 소비
  ExpressionT value = ParseUnaryExpression(); // 뒤의 표현식 파싱

  // Await 노드 생성
  ExpressionT expr = factory()->NewAwait(value, await_pos);
  function_state_->AddSuspend();  // "이 함수는 중단점이 있다" 표시
  return expr;
}
```

**쉽게 설명하면:**
1. `await` 키워드를 인식
2. `await` 뒤에 오는 표현식(`somePromise`)을 파싱
3. "Await 노드"라는 트리 구조로 저장
4. 이 함수에 "중단점(suspend point)이 있다"고 표시

#### AST란?

AST(Abstract Syntax Tree)는 코드를 트리 구조로 표현한 것이다.

```javascript
const x = await promise;

// AST로 표현하면:
VariableDeclaration
├── name: "x"
└── init: Await
         └── expression: Identifier("promise")
```

V8의 `Await` 클래스가 이 "Await 노드"를 표현한다:

```cpp
// src/ast/ast.h
class Await : public Suspend {
  // Await는 Suspend를 상속
  // expression_ = await 뒤의 표현식 (예: promise)
};

class Suspend : public Expression {
  Expression* expression_;  // yield/await 뒤의 값
};
```

**Suspend 클래스**: `yield`(Generator)와 `await`(Async)가 공유하는 부모 클래스. 둘 다 "함수 실행을 중단"하는 역할이므로 같은 구조를 사용한다.

**Expression 클래스**: V8에서 "값을 반환하는 모든 것"의 부모 클래스. `1 + 2`, `foo()`, `await x` 등이 모두 Expression이다.

### 2. 바이트코드 생성: 실행 가능한 명령어로 변환

**파일**: `src/interpreter/bytecode-generator.cc`

AST의 Await 노드를 만나면 바이트코드로 변환한다:

```cpp
// 줄 6337-6387
void BuildAwait(int position) {
  // 1. AsyncFunctionAwait 런타임 함수 호출 준비
  //    - generator: 현재 async 함수의 상태 객체
  //    - value: await 뒤의 값
  builder()
      ->MoveRegister(generator_object(), args[0])  // generator
      .StoreAccumulatorInRegister(args[1])         // await value
      .CallRuntime(kInlineAsyncFunctionAwait, args);

  // 2. 여기서 함수 실행 중단 (suspend)
  BuildSuspendPoint(position);

  // 3. 나중에 재개(resume)되면 여기부터 실행
  //    resume_mode를 확인해서 분기:
  //    - kNext: Promise 성공 → 계속 진행
  //    - kThrow: Promise 실패 → 에러 던지기
  if (resume_mode == kNext) {
    // 정상 진행
  } else {
    // 에러 재발생
    ReThrow();
  }
}
```

**쉽게 설명하면:**
1. `await value`를 실행하기 위한 런타임 함수 호출
2. 함수 실행을 중단 (suspend point)
3. Promise가 완료되면 재개 (resume)
4. 성공이면 계속 진행, 실패면 에러 발생

#### 바이트코드란?

JavaScript 코드를 컴퓨터가 실행할 수 있는 중간 형태의 명령어로 변환한 것이다.

```javascript
await promise;

// 대략 이런 바이트코드로 변환됨 (개념적 설명):
LOAD promise
CALL AsyncFunctionAwait
SUSPEND          ← 여기서 멈춤
// ... Promise 완료 후 ...
RESUME           ← 여기서 재개
CHECK_RESUME_MODE
JUMP_IF_THROW error_handler
CONTINUE
```

### 3. 런타임 실행: 실제 Promise.then() 연결

**파일**: `src/builtins/builtins-async-function-gen.cc`, `src/builtins/builtins-async-gen.cc`

바이트코드가 실행되면 V8의 내장 함수(Builtin)가 호출된다.

#### 3.1 Async 함수 시작: Promise 생성

async 함수가 호출되면 가장 먼저 실행되는 부분:

```cpp
// builtins-async-function-gen.cc 줄 66-131
TF_BUILTIN(AsyncFunctionEnter, ...) {
  // 1. 이 async 함수가 반환할 Promise 생성
  TNode<JSPromise> promise = NewJSPromise(context);

  // 2. Async Function Object 생성 (함수 상태 저장용)
  //    - 현재 실행 위치
  //    - 로컬 변수들
  //    - 반환할 Promise
  TNode<JSAsyncFunctionObject> async_function_object = ...;

  // Promise를 async function object에 저장
  StoreObjectField(async_function_object, kPromiseOffset, promise);

  return async_function_object;
}
```

**쉽게 설명하면:**
```javascript
async function foo() { ... }
foo();  // 호출 시:

// 1. 반환할 Promise 생성
const returnPromise = new Promise();

// 2. 함수 상태 객체 생성 (V8 내부)
const asyncFunctionObject = {
  promise: returnPromise,
  context: 현재_실행_상태,
  resumePoint: 0,  // 어디까지 실행했는지
};

return returnPromise;  // 즉시 Promise 반환
```

#### 3.2 Await 실행: Promise.then() 연결 (핵심!)

`await value`를 만나면 실행되는 부분:

```cpp
// builtins-async-gen.cc 줄 27-159
TNode<Object> Await(..., TNode<JSAny> value, ...) {

  // 1. value가 Promise인지 확인
  if (!IsPromise(value)) {
    // Promise가 아니면 Promise로 감싸기
    // await 42 → await Promise.resolve(42)
    value = NewJSPromise();
    ResolvePromise(value, original_value);
  }

  // 2. resolve/reject 핸들러(콜백 함수) 생성
  //    - on_resolve: Promise 성공 시 호출될 함수
  //    - on_reject: Promise 실패 시 호출될 함수
  auto [on_resolve, on_reject] = CreateClosures(...);

  // 3. ★★★ 핵심: Promise.then() 호출 ★★★
  //    value.then(on_resolve, on_reject)
  return CallBuiltin(Builtin::kPerformPromiseThen,
                     value, on_resolve, on_reject);
}
```

**쉽게 설명하면:**
```javascript
// await somePromise 를 만나면...

// 1. Promise인지 확인
let value = somePromise;
if (!(value instanceof Promise)) {
  value = Promise.resolve(value);
}

// 2. 콜백 함수 생성
function on_resolve(result) {
  // Promise 성공 시: 함수 재개
  generator.resume(result, 'next');
}

function on_reject(error) {
  // Promise 실패 시: 에러와 함께 함수 재개
  generator.resume(error, 'throw');
}

// 3. ★ 핵심: then 연결 ★
value.then(on_resolve, on_reject);

// 4. 여기서 함수가 콜스택에서 빠져나감 (return)
//    on_resolve/on_reject는 Promise 완료 시 마이크로태스크 큐에 추가됨
```

**콜스택 관점에서 보면:**
```
await 실행 직전:                    await 실행 직후:
┌─────────────┐                    ┌─────────────┐
│ foo()       │                    │ (비어있음)   │  ← foo()가 빠져나감!
│  ↳ await    │        →          └─────────────┘
└─────────────┘
                                   Heap에 저장됨:
                                   ┌─────────────────────┐
                                   │ AsyncFunctionObject │
                                   │ ├─ 실행위치: 3번줄  │
                                   │ └─ 변수들: {...}    │
                                   └─────────────────────┘
```

#### 3.3 Promise 완료 후: 함수 재개

Promise가 resolve/reject되면 **마이크로태스크 큐**에 콜백이 추가된다.
이벤트 루프가 콜스택이 비었을 때 마이크로태스크 큐에서 콜백을 꺼내서 실행한다:

```
Promise 완료 시:                     이벤트 루프가 실행:

Microtask Queue:                    Call Stack:
┌─────────────────┐                 ┌─────────────────┐
│ on_resolve(42)  │      →         │ on_resolve(42)  │
└─────────────────┘                 │  ↳ foo() 재개   │
                                    └─────────────────┘
```

```cpp
// builtins-async-function-gen.cc 줄 171-188

// Promise 성공 시 호출됨
TF_BUILTIN(AsyncFunctionAwaitResolveClosure, ...) {
  const auto sentValue = Parameter(kSentValue);  // resolve된 값

  // 함수 재개: resume_mode = kNext (정상 진행)
  AsyncFunctionAwaitResumeClosure(context, sentValue, kNext);
}

// Promise 실패 시 호출됨
TF_BUILTIN(AsyncFunctionAwaitRejectClosure, ...) {
  const auto sentError = Parameter(kSentError);  // reject된 에러

  // 함수 재개: resume_mode = kThrow (에러 던지기)
  AsyncFunctionAwaitResumeClosure(context, sentError, kThrow);
}
```

```cpp
// 줄 32-64: 실제 재개 로직
void AsyncFunctionAwaitResumeClosure(..., resume_mode) {
  // 저장해둔 async function object 가져오기
  TNode<JSAsyncFunctionObject> async_function_object =
      LoadFromContext(EXTENSION_INDEX);

  // resume mode 저장 (kNext 또는 kThrow)
  StoreObjectField(async_function_object, kResumeModeOffset, resume_mode);

  // 함수 재개! (중단했던 지점부터 다시 실행)
  CallBuiltin(kResumeGeneratorTrampoline, sent_value, async_function_object);
}
```

**쉽게 설명하면:**
```javascript
// Promise가 resolve(42)로 완료되면...
function on_resolve(result) {  // result = 42
  asyncFunctionObject.resumeMode = 'next';
  asyncFunctionObject.inputValue = result;

  // 중단했던 지점으로 돌아가서 실행 재개
  resumeGenerator(asyncFunctionObject);

  // const x = await promise;
  //           ↑ 여기서 멈췄다가
  // x에 42가 할당되고 다음 줄 실행
}

// Promise가 reject(new Error())로 완료되면...
function on_reject(error) {
  asyncFunctionObject.resumeMode = 'throw';
  asyncFunctionObject.inputValue = error;

  resumeGenerator(asyncFunctionObject);

  // const x = await promise;
  //           ↑ 여기서 에러가 throw됨
  // try-catch가 있으면 catch로, 없으면 함수 전체가 reject
}
```

#### 3.4 Async 함수 종료: 최종 Promise 완료

```cpp
// 정상 return 시
TF_BUILTIN(AsyncFunctionResolve, ...) {
  TNode<JSPromise> promise = LoadObjectField(async_function_object, kPromiseOffset);

  // async 함수가 반환할 Promise를 resolve
  CallBuiltin(kResolvePromise, promise, return_value);
}

// throw 발생 시
TF_BUILTIN(AsyncFunctionReject, ...) {
  TNode<JSPromise> promise = LoadObjectField(async_function_object, kPromiseOffset);

  // async 함수가 반환할 Promise를 reject
  CallBuiltin(kRejectPromise, promise, error);
}
```

---

## 실행 흐름 요약

### 전체 흐름 다이어그램

```
┌────────────────────────────────────────────────────────────────┐
│  async function foo() {                                        │
│    const result = await somePromise;                           │
│    return result;                                              │
│  }                                                             │
│  foo();                                                        │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│  1. foo() 호출                                                  │
│     - returnPromise = new Promise() 생성                        │
│     - asyncFunctionObject 생성 (함수 상태 저장)                   │
│     - returnPromise 즉시 반환                                    │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│  2. await somePromise 도달                                      │
│     - somePromise가 Promise인지 확인                             │
│     - on_resolve, on_reject 콜백 생성                            │
│     - somePromise.then(on_resolve, on_reject) 호출  ◀━━ 핵심!   │
│     - 함수 실행 중단 (suspend)                                   │
└────────────────────────────────────────────────────────────────┘
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
┌──────────────────────────┐ ┌──────────────────────────┐
│  3a. somePromise 성공     │ │  3b. somePromise 실패     │
│      (resolve)           │ │      (reject)            │
│                          │ │                          │
│  on_resolve(value) 호출   │ │  on_reject(error) 호출    │
│  resumeMode = 'next'     │ │  resumeMode = 'throw'    │
│  함수 재개                │ │  함수 재개                │
│                          │ │                          │
│  const result = value;   │ │  에러 throw              │
│  return result; 실행      │ │  → catch로 이동 또는      │
│                          │ │    함수 전체 reject       │
│  returnPromise.resolve() │ │  returnPromise.reject()  │
└──────────────────────────┘ └──────────────────────────┘
```

### 콜스택과 마이크로태스크 큐로 보는 실행 과정

```javascript
async function foo() {
  console.log('1');
  const result = await somePromise;
  console.log('2');
  return result;
}

console.log('A');
foo();
console.log('B');
```

**Step 1**: 동기 코드 실행
```
Call Stack: [log('A')]  →  출력: A
Call Stack: [foo(), log('1')]  →  출력: A, 1
```

**Step 2**: await를 만남 - foo()가 콜스택에서 빠짐
```
Call Stack: [foo() await 실행]
  ↓
somePromise.then(on_resolve) 등록
foo() 상태 저장 후 return
  ↓
Call Stack: [(비어있음)]     Heap: [AsyncFunctionObject 저장됨]
```

**Step 3**: 동기 코드 계속 실행
```
Call Stack: [log('B')]  →  출력: A, 1, B
Call Stack: [(비어있음)]
```

**Step 4**: somePromise 완료 → 마이크로태스크 큐에 추가
```
Call Stack: [(비어있음)]
Microtask Queue: [on_resolve(result)]  ← Promise 완료로 추가됨
```

**Step 5**: 이벤트 루프가 마이크로태스크 실행 → foo() 재개
```
Call Stack: [on_resolve → foo() 재개 → log('2')]  →  출력: A, 1, B, 2
```

**타임라인 요약:**
```
시간 →
Call Stack:    │log(A)│foo()│log(1)│await│    │log(B)│    │on_resolve│log(2)│
               │      │     │      │(pop)│    │      │    │(foo재개) │      │
               └──────┴─────┴──────┴─────┴────┴──────┴────┴──────────┴──────┘
                                     ↓                      ↑
                              foo() 빠짐              마이크로태스크 실행
                              상태 저장               상태 복원

출력:              A           1                  B                    2
```

### JavaScript로 표현한 내부 동작

```javascript
// 원본 코드
async function foo() {
  console.log('start');
  const result = await somePromise;
  console.log('end');
  return result;
}

// V8 내부 동작 (의사 코드)
function foo() {
  console.log('start');

  // 1. 반환할 Promise 생성
  const returnPromise = new Promise((resolve, reject) => {

    // 2. await를 then으로 변환
    Promise.resolve(somePromise).then(
      // 3a. 성공 시 - 마이크로태스크로 실행됨
      (result) => {
        console.log('end');
        resolve(result);  // returnPromise를 resolve
      },
      // 3b. 실패 시 - 마이크로태스크로 실행됨
      (error) => {
        reject(error);    // returnPromise를 reject
      }
    );
  });

  return returnPromise;  // 즉시 반환! (await 이후 코드는 나중에 실행)
}
```

### 핵심 포인트 정리

| 개념 | 설명 |
|------|------|
| **async function** | 항상 Promise를 반환하는 함수. 내부적으로 generator처럼 동작 |
| **await** | 함수 실행을 중단하고, Promise가 완료되면 재개 |
| **내부 변환** | `await value` → `value.then(onResolve, onReject)` |
| **suspend** | 콜스택에서 빠져나감 + 상태를 힙에 저장 |
| **resume** | 마이크로태스크 큐 → 콜스택 진입 + 상태 복원 |
| **마이크로태스크** | Promise.then 콜백이 들어가는 큐. setTimeout보다 먼저 실행 |
| **resume mode** | `kNext`(성공, 값 반환) 또는 `kThrow`(실패, 에러 발생) |

---

## 주요 파일 경로

| 역할 | 파일 경로 | 핵심 내용 |
|------|----------|----------|
| AST 정의 | `src/ast/ast.h` | Await, Suspend 클래스 정의 |
| 파싱 | `src/parsing/parser-base.h:3793` | await 표현식 파싱 |
| 바이트코드 생성 | `src/interpreter/bytecode-generator.cc:6337` | await → suspend/resume 변환 |
| **Promise.then 연결** | `src/builtins/builtins-async-gen.cc:159` | **핵심! await → then 변환** |
| 함수 시작 | `src/builtins/builtins-async-function-gen.cc:66` | Promise 생성 |
| 함수 재개 | `src/builtins/builtins-async-function-gen.cc:32` | Promise 완료 후 resume |

---

## 부록

### 마이크로태스크 vs 태스크(매크로태스크) 큐

Promise.then() 콜백은 **마이크로태스크 큐**에 들어간다:

```javascript
console.log('1');

setTimeout(() => console.log('timeout'), 0);  // 태스크 큐

Promise.resolve().then(() => console.log('promise'));  // 마이크로태스크 큐

console.log('2');

// 출력: 1, 2, promise, timeout
```

**이벤트 루프 우선순위**: 콜스택 → 마이크로태스크 큐 (전부) → 태스크 큐 (하나씩)

```
┌───────────────────────────────────────────────────────────┐
│ 이벤트 루프 한 사이클:                                      │
│                                                           │
│ 1. 콜스택 비어있나?                                        │
│    ↓                                                      │
│ 2. 마이크로태스크 큐 전부 실행                               │
│    (Promise.then, await 재개, queueMicrotask)             │
│    ↓                                                      │
│ 3. 태스크 큐에서 하나 실행                                  │
│    (setTimeout, setInterval, I/O)                        │
│    ↓                                                      │
│ 4. 1번으로 돌아감                                          │
└───────────────────────────────────────────────────────────┘
```

async/await 재개는 마이크로태스크이므로 setTimeout보다 먼저 실행된다:

```javascript
async function foo() {
  console.log('async 1');
  await Promise.resolve();
  console.log('async 2');  // 마이크로태스크
}

console.log('sync 1');
setTimeout(() => console.log('timeout'), 0);  // 태스크
foo();
console.log('sync 2');

// 출력: sync 1, async 1, sync 2, async 2, timeout
```

### C++ 코드 읽는 법 (간단 가이드)

V8 코드에서 자주 보이는 패턴:

```cpp
// 1. TNode<타입> = V8에서 사용하는 타입이 지정된 노드
TNode<JSPromise> promise = NewJSPromise();
// → JavaScript: const promise = new Promise();

// 2. LoadObjectField = 객체의 필드 읽기
TNode<JSPromise> promise = LoadObjectField(obj, kPromiseOffset);
// → JavaScript: const promise = obj.promise;

// 3. StoreObjectField = 객체의 필드 쓰기
StoreObjectField(obj, kValueOffset, value);
// → JavaScript: obj.value = value;

// 4. CallBuiltin = V8 내장 함수 호출
CallBuiltin(Builtin::kResolvePromise, promise, value);
// → JavaScript: Promise.resolve(promise, value); (개념적으로)

// 5. TF_BUILTIN = V8 내장 함수 정의
TF_BUILTIN(AsyncFunctionAwait, ...) { ... }
// → JavaScript: function AsyncFunctionAwait() { ... }
```
