# Node.js HTTP 요청 처리 내부 동작 원리

> 이 문서는 Node.js 소스 코드를 직접 분석하여 작성되었습니다.
> 공식 문서에서는 확인할 수 없는 내부 구현 상세를 다룹니다.

---

## 목차

1. [Request → Response 전체 경로](#1-request--response-전체-경로)
2. [데이터 변환 추적: Input → Output](#2-데이터-변환-추적-input--output)
3. [전체 아키텍처](#3-전체-아키텍처)
4. [HTTP 서버 생성](#4-http-서버-생성)
5. [연결 수립 및 처리](#5-연결-수립-및-처리)
6. [HTTP 파싱 (llhttp)](#6-http-파싱-llhttp)
7. [Request/Response 객체 생성](#7-requestresponse-객체-생성)
8. [이벤트 루프와 libuv](#8-이벤트-루프와-libuv)
9. [핵심 개념 설명](#9-핵심-개념-설명)
   - [socketOnTimeout의 this](#91-socketontimeout의-this는-무엇인가)
   - [parsers의 정체 (FreeList)](#92-parsers의-정체---freelist-객체-풀)
   - [Backpressure란?](#93-backpressure란)
10. [성능 최적화 기법](#10-성능-최적화-기법)
11. [핵심 파일 요약](#11-핵심-파일-요약)
12. [요약: 면접 답변용](#12-요약-면접-답변용)

---

## 1. Request → Response 전체 경로

HTTP 요청이 클라이언트에서 사용자 핸들러까지 도달하는 **전체 경로**를 단계별로 추적합니다.

```
[클라이언트] GET /users HTTP/1.1\r\nHost: localhost\r\n\r\n
      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 1. OS 커널 (TCP/IP 스택)                                            │
│    - TCP 3-way handshake 완료                                       │
│    - 소켓 파일 디스크립터 생성                                        │
└────────────────────────┬────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 2. libuv (C)                                                        │
│    - epoll_wait() / kqueue() / IOCP가 소켓 이벤트 감지              │
│    - uv__io_poll() → 소켓에 데이터 있음 확인                         │
│    - uv_read_start() 콜백 트리거                                     │
└────────────────────────┬────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 3. StreamBase (C++) - src/stream_base.cc                            │
│    - EmitAlloc() → Parser::OnStreamAlloc() 호출                     │
│      → 64KB 버퍼 반환                                                │
│    - read() 시스템콜로 데이터를 버퍼에 복사                           │
│    - EmitRead(nread, buf) → Parser::OnStreamRead() 호출            │
└────────────────────────┬────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 4. Parser::OnStreamRead() - src/node_http_parser.cc:768             │
│    - Execute(buf.base, nread) 호출                                  │
└────────────────────────┬────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 5. Parser::Execute() - src/node_http_parser.cc:812                  │
│    - llhttp_execute(&parser_, data, len) ← 실제 HTTP 파싱           │
│    - llhttp가 데이터를 파싱하면서 콜백 호출:                          │
│      ┌──────────────────────────────────────────────────────────┐   │
│      │ llhttp 콜백 순서:                                         │   │
│      │ ① on_message_begin()  → 새 메시지 시작                   │   │
│      │ ② on_url("GET")       → 메서드 파싱                      │   │
│      │ ③ on_url("/users")    → URL 파싱                         │   │
│      │ ④ on_header_field("Host") → 헤더 이름                    │   │
│      │ ⑤ on_header_value("localhost") → 헤더 값                 │   │
│      │ ⑥ on_headers_complete() → 헤더 파싱 완료 ★               │   │
│      │ ⑦ on_body(data)       → 본문 (있으면)                    │   │
│      │ ⑧ on_message_complete() → 메시지 완료                    │   │
│      └──────────────────────────────────────────────────────────┘   │
└────────────────────────┬────────────────────────────────────────────┘
                         │ ⑥ on_headers_complete() 시점
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 6. Parser::on_headers_complete() - src/node_http_parser.cc:375      │
│    - C++에서 JavaScript 콜백 호출                                    │
│    - MakeCallback(kOnHeadersComplete, argv)                         │
└────────────────────────┬────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 7. parserOnHeadersComplete() - lib/_http_common.js:75               │
│    ┌────────────────────────────────────────────────────────────┐   │
│    │ const incoming = parser.incoming = new IncomingMessage();  │   │
│    │ incoming.httpVersion = '1.1';                              │   │
│    │ incoming.url = '/users';                                   │   │
│    │ incoming.method = 'GET';                                   │   │
│    │ incoming._addHeaderLines(headers, n);  // 헤더 추가        │   │
│    │ return parser.onIncoming(incoming, shouldKeepAlive);       │   │
│    └────────────────────────────────────────────────────────────┘   │
└────────────────────────┬────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 8. parserOnIncoming() - lib/_http_server.js:1188                    │
│    ┌────────────────────────────────────────────────────────────┐   │
│    │ state.incoming.push(req);  // 요청 큐에 추가               │   │
│    │                                                            │   │
│    │ // ServerResponse 생성                                     │   │
│    │ const res = new ServerResponse(req);                       │   │
│    │ res.assignSocket(socket);                                  │   │
│    │                                                            │   │
│    │ // ★★★ 사용자 핸들러 호출 ★★★                             │   │
│    │ server.emit('request', req, res);                          │   │
│    └────────────────────────────────────────────────────────────┘   │
└────────────────────────┬────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 9. 사용자 코드 실행                                                  │
│    ┌────────────────────────────────────────────────────────────┐   │
│    │ http.createServer((req, res) => {                          │   │
│    │   // 여기가 실행됨!                                         │   │
│    │   res.writeHead(200);                                      │   │
│    │   res.end('Hello');                                        │   │
│    │ });                                                        │   │
│    └────────────────────────────────────────────────────────────┘   │
└────────────────────────┬────────────────────────────────────────────┘
                         │ res.end() 호출
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 10. 응답 전송                                                        │
│     - res.end() → OutgoingMessage._finish()                         │
│     - socket.write() → libuv uv_write()                             │
│     - TCP로 응답 전송                                                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. 데이터 변환 추적: Input → Output

HTTP 요청 데이터가 **어떤 타입/변수**로 들어와서 **어떻게 변환**되어 **어떤 형태로 나가는지** 추적합니다.

### 2.1 Request 데이터 변환 (Input → IncomingMessage)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 1: Raw TCP Data (바이트 스트림)                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│ Input:  uv_buf_t { base: char*, len: size_t }                               │
│ 값:     "GET /users HTTP/1.1\r\nHost: localhost\r\n\r\n" (바이트)           │
│ 위치:   src/stream_base.cc - EmitRead()                                     │
└────────────────────────┬────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 2: Parser::OnStreamRead()                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│ Input:  ssize_t nread, const uv_buf_t& buf                                  │
│ 호출:   Execute(buf.base, nread)                                            │
│ 위치:   src/node_http_parser.cc:768                                         │
└────────────────────────┬────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 3: llhttp_execute() - HTTP 파싱                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│ Input:  const char* data, size_t len                                        │
│ Output: 콜백으로 파싱된 값 전달                                              │
│         - on_url(at, len)         → "/users"                                │
│         - on_header_field(at,len) → "Host"                                  │
│         - on_header_value(at,len) → "localhost"                             │
│ 위치:   deps/llhttp/src/llhttp.c                                            │
└────────────────────────┬────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 4: parserOnHeadersComplete() - JavaScript 콜백                         │
├─────────────────────────────────────────────────────────────────────────────┤
│ Input 인자들:                                                                │
│   versionMajor: 1          (Number)                                         │
│   versionMinor: 1          (Number)                                         │
│   headers: ['Host', 'localhost']  (Array) - [field, value, field, value...] │
│   method: 1                (Number) - HTTP 메서드 인덱스 (GET=1)            │
│   url: '/users'            (String)                                         │
│   upgrade: false           (Boolean)                                        │
│   shouldKeepAlive: true    (Boolean)                                        │
│                                                                             │
│ 위치:   lib/_http_common.js:75-124                                          │
└────────────────────────┬────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 5: IncomingMessage 객체 생성                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│ Output 객체 (req):                                                          │
│   {                                                                         │
│     httpVersionMajor: 1,                                                    │
│     httpVersionMinor: 1,                                                    │
│     httpVersion: '1.1',                                                     │
│     method: 'GET',              // allMethods[1] → 'GET'                    │
│     url: '/users',                                                          │
│     headers: { host: 'localhost' },  // _addHeaderLines()로 변환            │
│     rawHeaders: ['Host', 'localhost'],                                      │
│     socket: Socket { ... },                                                 │
│     complete: false,                                                        │
│     // Readable 스트림 상속                                                  │
│   }                                                                         │
│                                                                             │
│ 위치:   lib/_http_incoming.js:50-93                                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

**실제 코드 - parserOnHeadersComplete:**

```javascript
// lib/_http_common.js:75-124
function parserOnHeadersComplete(versionMajor, versionMinor, headers, method,
                                 url, statusCode, statusMessage, upgrade,
                                 shouldKeepAlive) {
  const parser = this;
  const { socket } = parser;

  // IncomingMessage 생성
  const incoming = parser.incoming = new IncomingMessage(socket);

  // 파싱된 값들을 객체 속성으로 변환
  incoming.httpVersionMajor = versionMajor;  // 1
  incoming.httpVersionMinor = versionMinor;  // 1
  incoming.httpVersion = '1.1';
  incoming.url = url;                        // '/users'
  incoming.upgrade = upgrade;                // false

  // headers 배열을 객체로 변환: ['Host', 'localhost'] → { host: 'localhost' }
  incoming._addHeaderLines(headers, n);

  // method 숫자를 문자열로 변환: 1 → 'GET'
  if (typeof method === 'number') {
    incoming.method = allMethods[method];    // allMethods[1] = 'GET'
  }

  return parser.onIncoming(incoming, shouldKeepAlive);
}
```

---

### 2.2 Response 데이터 변환 (ServerResponse → Raw TCP Data)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 1: 사용자 코드에서 res 조작                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│ Input:                                                                      │
│   res.writeHead(200, { 'Content-Type': 'text/plain' });                     │
│   res.end('Hello World');                                                   │
│                                                                             │
│ res 객체 상태:                                                               │
│   {                                                                         │
│     statusCode: 200,                                                        │
│     [kOutHeaders]: { 'content-type': ['Content-Type', 'text/plain'] },     │
│     _header: null,        // 아직 헤더 문자열 미생성                         │
│     _headerSent: false,                                                     │
│   }                                                                         │
└────────────────────────┬────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 2: _implicitHeader() → _storeHeader() 호출                             │
├─────────────────────────────────────────────────────────────────────────────┤
│ Input:                                                                      │
│   statusCode: 200                                                           │
│   headers: { 'content-type': ['Content-Type', 'text/plain'] }              │
│                                                                             │
│ Output (_header 문자열 생성):                                                │
│   "HTTP/1.1 200 OK\r\n"                                                     │
│   "Content-Type: text/plain\r\n"                                            │
│   "Date: Mon, 02 Dec 2024 12:00:00 GMT\r\n"                                 │
│   "Connection: keep-alive\r\n"                                              │
│   "Transfer-Encoding: chunked\r\n"                                          │
│   "\r\n"                                                                    │
│                                                                             │
│ 위치:   lib/_http_outgoing.js:392-531                                       │
└────────────────────────┬────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 3: res.end('Hello World') → write_() 호출                              │
├─────────────────────────────────────────────────────────────────────────────┤
│ Input:                                                                      │
│   chunk: 'Hello World' (String)                                             │
│   encoding: 'utf8'                                                          │
│                                                                             │
│ Chunked Encoding 적용 시 변환:                                               │
│   len = 11 (바이트 길이)                                                     │
│   'b\r\n'              // 11을 16진수로                                      │
│   'Hello World'        // 실제 데이터                                        │
│   '\r\n'                                                                    │
│   '0\r\n\r\n'          // 종료 청크                                          │
│                                                                             │
│ 위치:   lib/_http_outgoing.js:887-972                                       │
└────────────────────────┬────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 4: _send() → socket.write() 호출                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│ Input:                                                                      │
│   data: String 또는 Buffer                                                  │
│   encoding: 'latin1' 또는 'utf8'                                            │
│                                                                             │
│ outputData 배열에 저장 (소켓에 바로 쓸 수 없을 때):                           │
│   this.outputData.push({ data, encoding, callback })                        │
│   this.outputSize += data.length                                            │
│                                                                             │
│ 위치:   lib/_http_outgoing.js:323-359                                       │
└────────────────────────┬────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 5: socket.write() → libuv uv_write()                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│ Input:                                                                      │
│   uv_buf_t bufs[] - 버퍼 배열                                                │
│   size_t nbufs    - 버퍼 개수                                                │
│                                                                             │
│ Output (TCP로 전송되는 최종 바이트):                                          │
│   "HTTP/1.1 200 OK\r\n"                                                     │
│   "Content-Type: text/plain\r\n"                                            │
│   "Date: Mon, 02 Dec 2024 12:00:00 GMT\r\n"                                 │
│   "Connection: keep-alive\r\n"                                              │
│   "Transfer-Encoding: chunked\r\n"                                          │
│   "\r\n"                                                                    │
│   "b\r\n"                                                                   │
│   "Hello World"                                                             │
│   "\r\n"                                                                    │
│   "0\r\n"                                                                   │
│   "\r\n"                                                                    │
│                                                                             │
│ 위치:   src/stream_base.cc - DoWrite()                                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

**실제 코드 - res.end():**

```javascript
// lib/_http_outgoing.js:1029-1107
OutgoingMessage.prototype.end = function end(chunk, encoding, callback) {
  // chunk가 있으면 write
  if (chunk) {
    write_(this, chunk, encoding, null, true);
  }

  // 헤더가 아직 없으면 생성
  if (!this._header) {
    this._contentLength = 0;
    this._implicitHeader();
  }

  // Chunked 인코딩이면 종료 청크 전송
  if (this._hasBody && this.chunkedEncoding) {
    this._send('0\r\n' + this._trailer + '\r\n', 'latin1', finish);
  }

  this.finished = true;
};
```

---

### 2.3 변수 타입 변환 요약표

| 단계 | 변수명 | 타입 | 예시 값 |
|------|--------|------|---------|
| **Request 입력** |
| TCP 수신 | `buf.base` | `char*` (C) | `"GET /users HTTP/1.1\r\n..."` |
| llhttp 파싱 후 | `method` | `Number` | `1` (GET의 인덱스) |
| llhttp 파싱 후 | `headers` | `Array` | `['Host', 'localhost', 'Accept', '*/*']` |
| llhttp 파싱 후 | `url` | `String` | `'/users'` |
| IncomingMessage | `req.method` | `String` | `'GET'` |
| IncomingMessage | `req.headers` | `Object` | `{ host: 'localhost', accept: '*/*' }` |
| IncomingMessage | `req.url` | `String` | `'/users'` |
| **Response 출력** |
| 사용자 코드 | `statusCode` | `Number` | `200` |
| 사용자 코드 | `headers` | `Object` | `{ 'Content-Type': 'text/plain' }` |
| _storeHeader 후 | `_header` | `String` | `'HTTP/1.1 200 OK\r\n...'` |
| write_ 후 | `chunk` | `String/Buffer` | `'Hello World'` |
| Chunked 변환 | `data` | `String` | `'b\r\nHello World\r\n'` |
| TCP 전송 | `uv_buf_t` | `struct` (C) | 바이트 스트림 |

---

### 2.4 핵심 변환 함수들

```javascript
// 1. HTTP 메서드 숫자 → 문자열
// lib/_http_common.js:31
const { methods, allMethods } = internalBinding('http_parser');
// allMethods = ['DELETE', 'GET', 'HEAD', 'POST', 'PUT', ...]
incoming.method = allMethods[method];  // 1 → 'GET'

// 2. headers 배열 → 객체
// lib/_http_incoming.js:244-263
function _addHeaderLines(headers, n) {
  // headers = ['Host', 'localhost', 'Accept', '*/*']
  // → { host: 'localhost', accept: '*/*' }
  for (let i = 0; i < n; i += 2) {
    this._addHeaderLine(headers[i], headers[i + 1], dest);
  }
}

// 3. statusCode + headers → HTTP 응답 문자열
// lib/_http_outgoing.js:392-531
function _storeHeader(firstLine, headers) {
  // statusCode: 200, headers: {...}
  // → "HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\n"
  let header = firstLine;
  for (const key in headers) {
    header += key + ': ' + headers[key] + '\r\n';
  }
  this._header = header + '\r\n';
}

// 4. body → chunked encoding
// lib/_http_outgoing.js:954-965
if (msg.chunkedEncoding && chunk.length !== 0) {
  // 'Hello World' (11 bytes)
  // → 'b\r\nHello World\r\n'
  msg._send(len.toString(16), 'latin1', null);  // 'b'
  msg._send(crlf_buf, null, null);               // '\r\n'
  msg._send(chunk, encoding, null, len);         // 'Hello World'
  msg._send(crlf_buf, null, callback);           // '\r\n'
}
```

---

## 3. 전체 아키텍처

```
┌─────────────────────────────────────────────────────────┐
│  Application Layer (JavaScript)                         │
│  http.createServer(), req, res                          │
│  파일: lib/http.js, lib/_http_server.js                 │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────┴────────────────────────────────┐
│  HTTP Parser Layer                                       │
│  llhttp (C) + JavaScript 콜백                            │
│  파일: lib/_http_common.js, src/node_http_parser.cc     │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────┴────────────────────────────────┐
│  Stream/TCP Layer                                        │
│  net.Server, Socket, StreamListener                      │
│  파일: lib/net.js, src/stream_base.h                    │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────┴────────────────────────────────┐
│  libuv (C)                                               │
│  이벤트 루프, 비동기 I/O (epoll/kqueue/IOCP)            │
└─────────────────────────────────────────────────────────┘
```

---

## 4. HTTP 서버 생성

### 4.1 `http.createServer()` 진입점

**파일: `lib/http.js:62-64`**

```javascript
function createServer(opts, requestListener) {
  return new Server(opts, requestListener);
}
```

### 4.2 Server 클래스 초기화

**파일: `lib/_http_server.js:568-612`**

```javascript
function Server(options, requestListener) {
  if (!(this instanceof Server)) return new Server(options, requestListener);

  // ... 옵션 검증 ...

  storeHTTPOptions.call(this, options);

  // 핵심: net.Server를 상속받아 TCP 서버 기능 활용
  net.Server.call(
    this,
    { allowHalfOpen: true, noDelay: options.noDelay ?? true,
      keepAlive: options.keepAlive,
      keepAliveInitialDelay: options.keepAliveInitialDelay,
      highWaterMark: options.highWaterMark });

  if (requestListener) {
    this.on('request', requestListener);  // 사용자 핸들러 등록
  }

  // 연결 이벤트 리스너 등록 - 핵심!
  this.on('connection', connectionListener);
  this.on('listening', setupConnectionsTracking);

  this.timeout = 0;
  this.maxHeadersCount = null;
  this.maxRequestsPerSocket = 0;
}
```

**핵심 포인트:**
- `net.Server`를 상속받아 TCP 기능 사용
- `'connection'` 이벤트에 `connectionListener` 등록 → 모든 HTTP 처리의 시작점

### 4.3 타임아웃 기본값

**파일: `lib/_http_server.js:473-514`**

```javascript
// storeHTTPOptions 함수 내부
const requestTimeout = options.requestTimeout;
if (requestTimeout !== undefined) {
  validateInteger(requestTimeout, 'requestTimeout', 0);
  this.requestTimeout = requestTimeout;
} else {
  this.requestTimeout = 300_000; // 5분 (기본값)
}

const headersTimeout = options.headersTimeout;
if (headersTimeout !== undefined) {
  // ...
} else {
  this.headersTimeout = MathMin(60_000, this.requestTimeout); // 60초 또는 requestTimeout 중 작은 값
}

const keepAliveTimeout = options.keepAliveTimeout;
if (keepAliveTimeout !== undefined) {
  // ...
} else {
  this.keepAliveTimeout = 5_000; // 5초 (기본값)
}
```

---

## 5. 연결 수립 및 처리

### 5.1 connectionListener - HTTP 처리의 시작

**파일: `lib/_http_server.js:701-801`**

```javascript
function connectionListener(socket) {
  defaultTriggerAsyncIdScope(
    getOrSetAsyncId(socket), connectionListenerInternal, this, socket,
  );
}

function connectionListenerInternal(server, socket) {
  debug('SERVER new http connection');

  socket.server = server;

  // 타임아웃 설정
  if (server.timeout && typeof socket.setTimeout === 'function')
    socket.setTimeout(server.timeout);
  socket.on('timeout', socketOnTimeout);

  // ★ 핵심: HTTP Parser 할당 (파서 풀에서 가져옴)
  const parser = parsers.alloc();

  const lenient = server.insecureHTTPParser === undefined ?
    isLenient() : server.insecureHTTPParser;

  // Parser 초기화 (C++ 바인딩)
  parser.initialize(
    HTTPParser.REQUEST,
    new HTTPServerAsyncResource('HTTPINCOMINGMESSAGE', socket),
    server.maxHeaderSize || 0,
    lenient ? kLenientAll : kLenientNone,
    server[kConnections],  // ConnectionsList (C++)
  );
  parser.socket = socket;
  socket.parser = parser;
```

### 5.2 상태 객체 생성 및 이벤트 바인딩

**파일: `lib/_http_server.js:744-801`**

```javascript
  // 상태 관리 객체
  const state = {
    onData: null,
    onEnd: null,
    onClose: null,
    onDrain: null,
    outgoing: [],           // 대기 중인 응답들
    incoming: [],           // 대기 중인 요청들
    outgoingData: 0,        // 출력 버퍼 크기 (backpressure용)
    requestsCount: 0,       // Keep-Alive 요청 카운트
    keepAliveTimeoutSet: false,
  };

  // 이벤트 핸들러 바인딩
  state.onData = socketOnData.bind(undefined, server, socket, parser, state);
  state.onEnd = socketOnEnd.bind(undefined, server, socket, parser, state);
  state.onClose = socketOnClose.bind(undefined, socket, state);
  state.onDrain = socketOnDrain.bind(undefined, socket, state);

  socket.on('data', state.onData);
  socket.on('error', socketOnError);
  socket.on('end', state.onEnd);
  socket.on('close', state.onClose);
  socket.on('drain', state.onDrain);

  // 헤더 파싱 완료 시 호출될 콜백
  parser.onIncoming = parserOnIncoming.bind(undefined, server, socket, state);

  // ★ 핵심: StreamListener로 Parser 등록 (제로카피에 가까운 데이터 전달)
  if (socket._handle?.isStreamBase && !socket._handle._consumed) {
    parser._consumed = true;
    socket._handle._consumed = true;
    parser.consume(socket._handle);  // C++ StreamListener 등록
  }

  // C++ 콜백 등록
  parser[kOnExecute] = onParserExecute.bind(undefined, server, socket, parser, state);
  parser[kOnTimeout] = onParserTimeout.bind(undefined, server, socket);
```

### 5.3 데이터 수신 처리

**파일: `lib/_http_server.js:873-879`**

```javascript
function socketOnData(server, socket, parser, state, d) {
  assert(!socket._paused);
  debug('SERVER socketOnData %d', d.length);

  // Parser.execute()가 llhttp_execute() 호출
  const ret = parser.execute(d);
  onParserExecuteCommon(server, socket, parser, state, ret, d);
}
```

---

## 6. HTTP 파싱 (llhttp)

### 6.1 Parser 풀 (FreeList)

**파일: `lib/_http_common.js:164-175`**

```javascript
// 메모리 재사용을 위한 Parser 풀
const parsers = new FreeList('parsers', 1000, function parsersCb() {
  const parser = new HTTPParser();

  cleanParser(parser);

  // 콜백 함수 등록
  parser[kOnHeaders] = parserOnHeaders;
  parser[kOnHeadersComplete] = parserOnHeadersComplete;
  parser[kOnBody] = parserOnBody;
  parser[kOnMessageComplete] = parserOnMessageComplete;

  return parser;
});
```

**최적화 포인트:**
- 최대 1000개의 Parser를 풀에 유지
- 새 연결마다 Parser를 생성하지 않고 재사용
- `parsers.alloc()`으로 가져오고, `parsers.free(parser)`로 반환

### 6.2 헤더 파싱 완료 콜백

**파일: `lib/_http_common.js:75-124`**

```javascript
// llhttp가 헤더 파싱을 완료하면 호출
function parserOnHeadersComplete(versionMajor, versionMinor, headers, method,
                                 url, statusCode, statusMessage, upgrade,
                                 shouldKeepAlive) {
  const parser = this;
  const { socket } = parser;

  // 조각난 헤더가 있으면 누적된 것 사용
  if (headers === undefined) {
    headers = parser._headers;
    parser._headers = [];
  }

  if (url === undefined) {
    url = parser._url;
    parser._url = '';
  }

  // ★ IncomingMessage 객체 생성
  const ParserIncomingMessage = (socket?.server?.[kIncomingMessage]) ||
                                 IncomingMessage;

  const incoming = parser.incoming = new ParserIncomingMessage(socket);
  incoming.httpVersionMajor = versionMajor;
  incoming.httpVersionMinor = versionMinor;
  incoming.httpVersion = versionMajor === 1 && versionMinor === 1 ?
    HTTP_VERSION_1_1 :
    `${versionMajor}.${versionMinor}`;
  incoming.url = url;
  incoming.upgrade = upgrade;

  // 헤더 추가
  incoming._addHeaderLines(headers, n);

  // 서버인 경우 method 설정
  if (typeof method === 'number') {
    incoming.method = allMethods[method];
  }

  // ★ parserOnIncoming 호출 → ServerResponse 생성 및 'request' 이벤트 발행
  return parser.onIncoming(incoming, shouldKeepAlive);
}
```

### 6.3 Body 파싱

**파일: `lib/_http_common.js:126-139`**

```javascript
function parserOnBody(b) {
  const stream = this.incoming;

  if (stream === null || stream[kSkipPendingData])
    return;

  // IncomingMessage 스트림에 데이터 push
  if (!stream._dumped) {
    const ret = stream.push(b);
    if (!ret)
      readStop(this.socket);  // backpressure: 소켓 읽기 중지
  }
}
```

### 6.4 C++ Parser 구현

**파일: `src/node_http_parser.cc:812-890`**

```cpp
Local<Value> Execute(const char* data, size_t len) {
  EscapableHandleScope scope(env()->isolate());

  current_buffer_len_ = len;
  current_buffer_data_ = data;
  got_exception_ = false;

  llhttp_errno_t err;

  if (data == nullptr) {
    err = llhttp_finish(&parser_);  // 스트림 종료
  } else {
    err = llhttp_execute(&parser_, data, len);  // ★ llhttp 파싱 실행
    Save();  // 조각난 헤더 저장
  }

  // 파싱된 바이트 수 계산
  size_t nread = len;
  if (err != HPE_OK) {
    nread = llhttp_get_error_pos(&parser_) - data;

    // Upgrade 요청 처리
    if (err == HPE_PAUSED_UPGRADE) {
      err = HPE_OK;
      llhttp_resume_after_upgrade(&parser_);
    }
  }

  // 에러 처리
  if (!parser_.upgrade && err != HPE_OK) {
    Local<Value> e = Exception::Error(env()->parse_error_string());
    // ... 에러 객체 생성 ...
    return scope.Escape(e);
  }

  return scope.Escape(nread_obj);
}
```

### 6.5 llhttp 콜백 설정

**파일: `src/node_http_parser.cc:1187-1236`**

```cpp
const llhttp_settings_t Parser::settings = {
    Proxy<Call, &Parser::on_message_begin>::Raw,
    nullptr,  // on_protocol
    Proxy<DataCall, &Parser::on_url>::Raw,
    Proxy<DataCall, &Parser::on_status>::Raw,
    nullptr,  // on_method
    nullptr,  // on_version
    Proxy<DataCall, &Parser::on_header_field>::Raw,
    Proxy<DataCall, &Parser::on_header_value>::Raw,
    Proxy<DataCall, &Parser::on_chunk_extension>::Raw,
    Proxy<DataCall, &Parser::on_chunk_extension>::Raw,
    Proxy<Call, &Parser::on_headers_complete>::Raw,
    Proxy<DataCall, &Parser::on_body>::Raw,
    Proxy<Call, &Parser::on_message_complete>::Raw,
    // ... 나머지 콜백들 ...
};
```

---

## 7. Request/Response 객체 생성

### 7.1 parserOnIncoming - 핵심 처리 함수

**파일: `lib/_http_server.js:1188-1319`**

```javascript
function parserOnIncoming(server, socket, state, req, keepAlive) {
  resetSocketTimeout(server, socket, state);

  // Upgrade 요청 체크 (WebSocket 등)
  if (req.upgrade) {
    req.upgrade = req.method === 'CONNECT' ||
                  !!server.shouldUpgradeCallback(req);
    if (req.upgrade) {
      return 0;
    }
  }

  // 요청을 상태 큐에 추가
  state.incoming.push(req);

  // ★ Backpressure 처리
  if (!socket._paused) {
    const ws = socket._writableState;
    if (ws.needDrain || state.outgoingData >= socket.writableHighWaterMark) {
      socket._paused = true;
      socket.pause();  // 소켓 읽기 일시 중지
    }
  }

  // ★ ServerResponse 객체 생성
  const res = new server[kServerResponse](req, {
    highWaterMark: socket.writableHighWaterMark,
    rejectNonStandardBodyWrites: server.rejectNonStandardBodyWrites,
  });
  res._keepAliveTimeout = server.keepAliveTimeout;
  res._maxRequestsPerSocket = server.maxRequestsPerSocket;
  res.shouldKeepAlive = keepAlive;

  // diagnostics_channel 이벤트 발행
  if (onRequestStartChannel.hasSubscribers) {
    onRequestStartChannel.publish({
      request: req,
      response: res,
      socket,
      server,
    });
  }

  // 소켓에 응답 연결
  if (socket._httpMessage) {
    state.outgoing.push(res);  // 이전 응답이 있으면 큐에 추가
  } else {
    res.assignSocket(socket);  // 소켓에 직접 연결
  }

  // 'finish' 이벤트 핸들러 등록
  res.on('finish', resOnFinish.bind(undefined, req, res, socket, state, server));

  // HTTP/1.1 Host 헤더 검증
  if (req.httpVersionMajor === 1 && req.httpVersionMinor === 1) {
    if (server.requireHostHeader && req.headers.host === undefined) {
      res.writeHead(400, ['Connection', 'close']);
      res.end();
      return 0;
    }
    // ... Expect 헤더 처리 등 ...
  }

  // ★ 'request' 이벤트 발행 → 사용자 핸들러 실행
  if (!handled) {
    server.emit('request', req, res);
  }

  return 0;
}
```

### 7.2 ServerResponse 클래스

**파일: `lib/_http_server.js:203-265`**

```javascript
function ServerResponse(req, options) {
  OutgoingMessage.call(this, options);

  if (req.method === 'HEAD') this._hasBody = false;

  this.req = req;
  this.sendDate = true;
  this._sent100 = false;
  this._expect_continue = false;

  // HTTP/1.0 호환성
  if (req.httpVersionMajor < 1 || req.httpVersionMinor < 1) {
    this.useChunkedEncodingByDefault = chunkExpression.test(req.headers.te);
    this.shouldKeepAlive = false;
  }

  // 성능 관찰자 등록
  if (hasObserver('http')) {
    startPerf(this, kServerResponseStatistics, {
      type: 'http',
      name: 'HttpRequest',
      detail: {
        req: {
          method: req.method,
          url: req.url,
          headers: req.headers,
        },
      },
    });
  }
}
```

### 7.3 IncomingMessage 클래스

**파일: `lib/_http_incoming.js:50-93`**

```javascript
function IncomingMessage(socket) {
  let streamOptions;

  if (socket) {
    streamOptions = {
      highWaterMark: socket.readableHighWaterMark,
    };
  }

  // Readable 스트림 상속
  Readable.call(this, streamOptions);

  this._readableState.readingMore = true;

  this.socket = socket;

  // HTTP 버전 정보
  this.httpVersionMajor = null;
  this.httpVersionMinor = null;
  this.httpVersion = null;
  this.complete = false;

  // 헤더 (lazy initialization)
  this[kHeaders] = null;
  this[kHeadersCount] = 0;
  this.rawHeaders = [];

  // 서버 요청용
  this.url = '';
  this.method = null;

  // 클라이언트 응답용
  this.statusCode = null;
  this.statusMessage = null;

  this._consuming = false;
  this._dumped = false;
}
```

### 7.4 응답 완료 처리

**파일: `lib/_http_server.js:1118-1171`**

```javascript
function resOnFinish(req, res, socket, state, server) {
  // diagnostics_channel 이벤트
  if (onResponseFinishChannel.hasSubscribers) {
    onResponseFinishChannel.publish({
      request: req,
      response: res,
      socket,
      server,
    });
  }

  // 요청 큐에서 제거
  state.incoming.shift();

  // 읽지 않은 요청 본문 버리기
  if (!req._consuming && !req._readableState.resumeScheduled)
    req._dump();

  // 소켓에서 응답 분리
  res.detachSocket(socket);
  clearIncoming(req);

  // Keep-Alive 처리
  if (res._last) {
    socket.destroySoon();  // 마지막 응답이면 소켓 종료
  } else if (state.outgoing.length === 0) {
    // Keep-Alive 타임아웃 설정
    if (keepAliveTimeout && typeof socket.setTimeout === 'function') {
      socket.setTimeout(keepAliveTimeout + keepAliveTimeoutBuffer);
      state.keepAliveTimeoutSet = true;
    }
  } else {
    // 다음 대기 중인 응답 처리
    const m = state.outgoing.shift();
    if (m) {
      m.assignSocket(socket);
    }
  }
}
```

---

## 8. 이벤트 루프와 libuv

### 8.1 StreamListener 인터페이스

**파일: `src/stream_base.h:116-180`**

```cpp
class StreamListener {
 public:
  virtual ~StreamListener();

  // 버퍼 할당 콜백 - 데이터 읽기 전 호출
  virtual uv_buf_t OnStreamAlloc(size_t suggested_size) = 0;

  // 데이터 읽기 완료 콜백
  virtual void OnStreamRead(ssize_t nread, const uv_buf_t& buf) = 0;

  // 쓰기 완료 콜백
  virtual void OnStreamAfterWrite(WriteWrap* w, int status);

 protected:
  // 이전 리스너에게 에러 전달
  inline void PassReadErrorToPreviousListener(ssize_t nread);

  StreamResource* stream_ = nullptr;
  StreamListener* previous_listener_ = nullptr;  // 리스너 체인
};
```

### 8.2 Parser의 StreamListener 구현

**파일: `src/node_http_parser.cc:753-809`**

```cpp
// Parser 클래스는 StreamListener를 상속
class Parser : public AsyncWrap, public StreamListener {

  // 버퍼 할당 - 64KB 정적 버퍼 재사용
  uv_buf_t OnStreamAlloc(size_t suggested_size) override {
    // 버퍼가 사용 중이면 동적 할당
    if (binding_data_->parser_buffer_in_use)
      return uv_buf_init(Malloc(suggested_size), suggested_size);

    binding_data_->parser_buffer_in_use = true;

    // 정적 버퍼 재사용 (성능 최적화)
    if (binding_data_->parser_buffer.empty())
      binding_data_->parser_buffer.resize(kAllocBufferSize);  // 64KB

    return uv_buf_init(binding_data_->parser_buffer.data(), kAllocBufferSize);
  }

  // ★ 데이터 수신 시 호출 - libuv → Parser
  void OnStreamRead(ssize_t nread, const uv_buf_t& buf) override {
    HandleScope scope(env()->isolate());

    // 버퍼 정리 (scope 종료 시)
    auto on_scope_leave = OnScopeLeave([&]() {
      if (buf.base == binding_data_->parser_buffer.data())
        binding_data_->parser_buffer_in_use = false;
      else
        free(buf.base);
    });

    if (nread < 0) {
      PassReadErrorToPreviousListener(nread);
      return;
    }

    if (nread == 0)
      return;

    // ★ HTTP 파싱 실행
    Local<Value> ret = Execute(buf.base, nread);

    if (ret.IsEmpty())
      return;

    // JavaScript 콜백 호출
    Local<Value> cb =
        object()->Get(env()->context(), kOnExecute).ToLocalChecked();

    if (!cb->IsFunction())
      return;

    current_buffer_len_ = nread;
    current_buffer_data_ = buf.base;

    MakeCallback(cb.As<Function>(), 1, &ret);

    current_buffer_len_ = 0;
    current_buffer_data_ = nullptr;
  }
};
```

### 8.3 Parser.consume() - StreamListener 등록

**파일: `src/node_http_parser.cc:715-722`**

```cpp
static void Consume(const FunctionCallbackInfo<Value>& args) {
  Parser* parser;
  ASSIGN_OR_RETURN_UNWRAP(&parser, args.This());
  CHECK(args[0]->IsObject());
  StreamBase* stream = StreamBase::FromObject(args[0].As<Object>());
  CHECK_NOT_NULL(stream);
  stream->PushStreamListener(parser);  // ★ Parser를 스트림 리스너로 등록
}
```

### 8.4 데이터 흐름

```
┌─────────────────────────────────────────────────────────────┐
│  libuv 이벤트 루프                                          │
│  uv_run() → uv_poll() → TCP 소켓 감지                       │
└────────────────────────┬────────────────────────────────────┘
                         │ 데이터 도착
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  uv_read_start() 콜백                                       │
│  → StreamResource::EmitAlloc() → Parser::OnStreamAlloc()   │
│  → 64KB 버퍼 반환                                           │
└────────────────────────┬────────────────────────────────────┘
                         │ 데이터 읽기
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  StreamResource::EmitRead()                                 │
│  → Parser::OnStreamRead(nread, buf)                        │
│  → Parser::Execute(data, len)                              │
│  → llhttp_execute(&parser_, data, len)                     │
└────────────────────────┬────────────────────────────────────┘
                         │ 파싱 콜백
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  on_headers_complete() → parserOnHeadersComplete()         │
│  → IncomingMessage 생성                                     │
│  → parserOnIncoming()                                       │
│  → ServerResponse 생성                                      │
│  → server.emit('request', req, res)                        │
└─────────────────────────────────────────────────────────────┘
```

---

## 9. 핵심 개념 설명

### 9.1 `socketOnTimeout`의 `this`는 무엇인가?

**파일: `lib/_http_server.js:830-839`**

```javascript
function socketOnTimeout() {
  const req = this.parser?.incoming;
  const reqTimeout = req && !req.complete && req.emit('timeout', this);
  const res = this._httpMessage;
  const resTimeout = res && res.emit('timeout', this);
  const serverTimeout = this.server.emit('timeout', this);

  if (!reqTimeout && !resTimeout && !serverTimeout)
    this.destroy();
}
```

**`this`는 `socket` 객체입니다.**

#### 왜 `this`가 `socket`인가?

```javascript
// lib/_http_server.js:719
socket.on('timeout', socketOnTimeout);
```

이것은 **EventEmitter의 동작 방식** 때문입니다:

```javascript
// EventEmitter 내부 구현 (간략화)
EventEmitter.prototype.emit = function(type, ...args) {
  const listeners = this._events[type];
  for (const listener of listeners) {
    listener.call(this, ...args);  // ← this를 바인딩!
  }
};
```

`socket.on('timeout', socketOnTimeout)`으로 등록하면:
1. `socket`에서 `'timeout'` 이벤트 발생
2. EventEmitter가 `socketOnTimeout.call(socket)` 호출
3. **`this`는 이벤트를 발생시킨 객체 = `socket`**

그래서 코드에서:
- `this.parser` → `socket.parser` (HTTP 파서)
- `this._httpMessage` → `socket._httpMessage` (현재 응답 객체)
- `this.server` → `socket.server` (HTTP 서버)
- `this.destroy()` → `socket.destroy()` (소켓 종료)

#### 반면 `bind()`를 사용한 경우

```javascript
// lib/_http_server.js:759-766
state.onData = socketOnData.bind(undefined, server, socket, parser, state);
socket.on('data', state.onData);
```

`bind(undefined, ...)`를 사용하면 `this`가 `undefined`가 되고, 필요한 값들은 인자로 전달합니다.

---

### 9.2 `parsers`의 정체 - FreeList (객체 풀)

**파일: `lib/_http_common.js:164-175`**

```javascript
const FreeList = require('internal/freelist');

const parsers = new FreeList('parsers', 1000, function parsersCb() {
  const parser = new HTTPParser();
  cleanParser(parser);
  parser[kOnHeaders] = parserOnHeaders;
  parser[kOnHeadersComplete] = parserOnHeadersComplete;
  parser[kOnBody] = parserOnBody;
  parser[kOnMessageComplete] = parserOnMessageComplete;
  return parser;
});
```

**`parsers`는 `FreeList` 인스턴스입니다.** 이것은 **객체 풀(Object Pool)** 패턴 구현입니다.

#### FreeList 구현

**파일: `lib/internal/freelist.js`**

```javascript
class FreeList {
  constructor(name, max, ctor) {
    this.name = name;
    this.ctor = ctor;   // 생성자 함수
    this.max = max;     // 최대 1000개
    this.list = [];     // 재사용 가능한 객체들
  }

  alloc() {
    // 풀에 객체가 있으면 꺼내서 반환
    // 없으면 새로 생성
    return this.list.length > 0 ?
      this.list.pop() :
      ReflectApply(this.ctor, this, arguments);
  }

  free(obj) {
    // 최대 개수 미만이면 풀에 넣고 재사용
    // 초과하면 버림 (GC가 처리)
    if (this.list.length < this.max) {
      this.list.push(obj);
      return true;
    }
    return false;
  }
}
```

#### 왜 필요한가?

```
요청 1: parsers.alloc() → 새 Parser 생성 → 사용 → parsers.free() → 풀에 반환
요청 2: parsers.alloc() → 풀에서 꺼냄   → 사용 → parsers.free() → 풀에 반환
요청 3: parsers.alloc() → 풀에서 꺼냄   → 사용 → parsers.free() → 풀에 반환
... (메모리 할당 없이 재사용)
```

- HTTP 요청마다 `new HTTPParser()`를 호출하면 **메모리 할당/해제 비용**이 큼
- Parser는 C++ 객체(`llhttp_t`)를 포함하므로 생성 비용이 높음
- 풀링으로 **최대 1000개 Parser를 재사용** → 성능 향상, GC 오버헤드 감소

---

### 9.3 Backpressure란?

**한 줄 정의:** 데이터 생산 속도가 소비 속도보다 빠를 때 **"잠깐 멈춰!"** 신호를 보내는 메커니즘

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│   Producer   │  ────▶  │    Buffer    │  ────▶  │   Consumer   │
│  (소켓 읽기) │  빠름    │  (메모리)    │  느림   │ (응답 처리)  │
└──────────────┘         └──────────────┘         └──────────────┘
                              │
                         버퍼 가득 참!
                              │
                              ▼
                    ┌─────────────────────┐
                    │ "잠깐! 읽기 중지해!" │  ← Backpressure
                    └─────────────────────┘
```

#### Node.js HTTP에서의 적용

**파일: `lib/_http_server.js:1204-1213`**

```javascript
if (!socket._paused) {
  const ws = socket._writableState;
  // 출력 버퍼가 highWaterMark(기본 16KB)를 초과하면
  if (ws.needDrain || state.outgoingData >= socket.writableHighWaterMark) {
    socket._paused = true;
    socket.pause();  // ★ 소켓 읽기 중지!
  }
}
```

#### 왜 필요한가?

**Backpressure가 없다면:**
1. 클라이언트가 1GB 파일을 POST
2. 서버가 계속 읽어서 메모리에 쌓음
3. 서버 메모리 부족 → 크래시 💥

**Backpressure가 있으면:**
1. 클라이언트가 1GB 파일을 POST
2. 버퍼가 16KB 차면 → `socket.pause()` (읽기 중지)
3. 처리가 끝나면 → `socket.resume()` (읽기 재개)
4. 메모리 안전하게 유지 ✅

#### 전체 흐름

```
[클라이언트] ──데이터──▶ [소켓 버퍼] ──읽기──▶ [Node.js 메모리]
                              │
                    버퍼 > highWaterMark?
                              │
                    ┌─────────┴─────────┐
                    │ Yes               │ No
                    ▼                   ▼
              socket.pause()      계속 읽기
              (TCP 윈도우 축소)
                    │
                    ▼
              처리 완료 후
              socket.resume()
```

---

## 10. 성능 최적화 기법

### 10.1 Parser 풀 (FreeList)

```javascript
// lib/_http_common.js:164
const parsers = new FreeList('parsers', 1000, function parsersCb() {
  const parser = new HTTPParser();
  // ...
  return parser;
});
```

- 최대 1000개 Parser 재사용
- GC 오버헤드 감소

### 10.2 정적 버퍼 재사용

```cpp
// src/node_http_parser.cc:751
static const size_t kAllocBufferSize = 64 * 1024;  // 64KB

uv_buf_t OnStreamAlloc(size_t suggested_size) override {
  if (binding_data_->parser_buffer_in_use)
    return uv_buf_init(Malloc(suggested_size), suggested_size);

  binding_data_->parser_buffer_in_use = true;
  // 64KB 정적 버퍼 반환
}
```

- 대부분의 요청에서 메모리 할당 없음
- 동시 요청 시에만 동적 할당

### 10.3 빠른 헤더 처리 경로

```cpp
// src/node_http_parser.cc:407-415
int on_headers_complete() {
  if (have_flushed_) {
    // 느린 경로: 헤더가 여러 패킷에 걸쳐 도착
    Flush();
  } else {
    // 빠른 경로: 헤더가 한 번에 도착 (대부분의 경우)
    argv[A_HEADERS] = CreateHeaders();
  }
}
```

### 10.4 ConnectionsList - 타임아웃 관리

```cpp
// src/node_http_parser.cc:211-251
class ConnectionsList : public BaseObject {
  std::set<Parser*, ParserComparator> all_connections_;
  std::set<Parser*, ParserComparator> active_connections_;

  // 만료된 연결 찾기
  static void Expired(const FunctionCallbackInfo<Value>& args);
};
```

```javascript
// lib/_http_server.js:550-560
function setupConnectionsTracking() {
  this[kConnections] ||= new ConnectionsList();

  // 주기적 타임아웃 체크 (기본 30초)
  this[kConnectionsCheckingInterval] =
    setInterval(checkConnections.bind(this), this.connectionsCheckingInterval).unref();
}
```

---

## 11. 핵심 파일 요약

| 파일 | 역할 | 핵심 함수/클래스 |
|------|------|------------------|
| `lib/http.js` | HTTP 모듈 진입점 | `createServer()`, `request()` |
| `lib/_http_server.js` | HTTP 서버 구현 | `Server`, `connectionListener()`, `parserOnIncoming()` |
| `lib/_http_common.js` | 파서 관리 | `parsers` (FreeList), `parserOnHeadersComplete()` |
| `lib/_http_incoming.js` | 요청 객체 | `IncomingMessage`, `_addHeaderLine()` |
| `lib/net.js` | TCP 소켓 | `Server`, `Socket` |
| `src/node_http_parser.cc` | C++ HTTP 파서 | `Parser`, `Execute()`, `OnStreamRead()` |
| `src/stream_base.h` | 스트림 인터페이스 | `StreamListener`, `StreamBase` |

---

## 12. 요약: 면접 답변용

1. **`http.createServer()` 호출**
   - `net.Server` 상속받아 TCP 서버 생성
   - `'connection'` 이벤트에 핸들러 등록

2. **클라이언트 연결 시**
   - `connectionListener()` 실행
   - **Parser 풀**에서 파서 할당 (메모리 재사용)
   - Parser를 **StreamListener**로 등록 → 소켓 데이터가 파서로 직접 전달

3. **데이터 수신 (libuv)**
   - `epoll/kqueue/IOCP`가 소켓 감지
   - `OnStreamRead()` 콜백 호출
   - **64KB 정적 버퍼** 재사용 (성능 최적화)

4. **HTTP 파싱 (llhttp)**
   - `llhttp_execute()` 실행 (C 라이브러리)
   - 상태 머신 기반 파싱
   - 콜백: `on_url` → `on_headers_complete` → `on_body` → `on_message_complete`

5. **JavaScript 객체 생성**
   - `parserOnHeadersComplete()`에서 **IncomingMessage** 생성
   - `parserOnIncoming()`에서 **ServerResponse** 생성
   - `server.emit('request', req, res)` → **사용자 핸들러 실행**

6. **응답 및 연결 관리**
   - Keep-Alive 지원 (기본 5초 타임아웃)
   - **Backpressure** 처리 (출력 버퍼 초과 시 읽기 중지)
   - **ConnectionsList**로 타임아웃 관리
