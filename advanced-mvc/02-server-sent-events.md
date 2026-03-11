# Server-Sent Events (SSE) — SseEmitter와 장기 커넥션

---

## 🎯 핵심 질문

- `SseEmitter`가 Servlet 비동기 모드로 장기 커넥션을 유지하는 내부 메커니즘은?
- `text/event-stream` Content-Type의 이벤트 형식(data/id/event/retry 필드)은?
- `SseEmitter.send()`가 호출될 때 데이터가 전송되는 정확한 경로는?
- 클라이언트 연결 끊김을 서버에서 감지하는 방법은?
- `SseEmitter`를 Spring Bean에 저장해서 관리하는 올바른 패턴은?
- `ResponseBodyEmitter`와 `StreamingResponseBody`와의 차이는?

---

## 🔍 왜 이 메커니즘이 존재하는가

```
실시간 데이터 전달 방식 비교:
  폴링 (Polling):     클라이언트가 주기적으로 요청 → 불필요한 요청 증가
  Long Polling:       서버 이벤트까지 연결 유지 → 매번 재연결 오버헤드
  WebSocket:          양방향 통신 → 프로토콜 업그레이드 필요, 복잡
  SSE (Server-Sent Events):
    단방향 (서버 → 클라이언트) 실시간 스트림
    일반 HTTP 연결 위에서 동작 (프록시/방화벽 친화적)
    브라우저 자동 재연결 지원
    간단한 텍스트 프로토콜

적합한 사용 사례:
  실시간 알림, 뉴스 피드, 주식 가격, 진행 상황 보고
  → 서버가 데이터를 밀어주는 단방향 스트림
```

---

## 😱 흔한 오해 또는 실수

### Before: SseEmitter를 요청 스레드에서 send()하면 즉시 전송된다

```java
// ❌ 잘못된 패턴 — 요청 스레드에서 블로킹 전송
@GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter streamEvents() {
    SseEmitter emitter = new SseEmitter();
    // 요청 스레드에서 반복 전송 시도
    for (int i = 0; i < 10; i++) {
        emitter.send("event-" + i);  // 블로킹 I/O
        Thread.sleep(1000);
    }
    // → 요청 스레드를 1초씩 10번 점유
    // → 서블릿 컨테이너 스레드 고갈 위험
    return emitter;
}

// ✅ 올바른 패턴: 즉시 반환 + 별도 스레드에서 전송
@GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter streamEvents() {
    SseEmitter emitter = new SseEmitter(60_000L);  // 60초 타임아웃
    emitterRegistry.register(emitter);  // Bean에 저장하여 외부에서 관리

    // 요청 스레드는 즉시 반환
    return emitter;
    // 이후 별도 스레드 또는 이벤트 핸들러에서 emitter.send() 호출
}
```

### Before: 클라이언트가 연결을 끊으면 즉시 감지된다

```
❌ 잘못된 이해: "클라이언트 disconnect → 서버에서 즉시 알 수 있다"

✅ 실제:
  TCP 연결 끊김은 다음 send() 시도 전까지 서버가 모를 수 있음
  → 클라이언트가 정상 종료: FIN 패킷 → 서버 소켓 오류 감지
  → 클라이언트가 비정상 종료(프로세스 강제 종료 등):
     send() 시도 시 IOException 발생으로 감지
  → emitter.onCompletion(), emitter.onTimeout(), emitter.onError() 콜백 활용
```

---

## ✨ 올바른 이해와 사용

### After: SSE 이벤트 형식

```
text/event-stream 형식 (RFC 8895):

data: {"type":"message","content":"안녕하세요"}\n\n
id: 1\n\n
event: userJoined\n
data: {"userId":42,"name":"홍길동"}\n\n
retry: 3000\n\n
: 주석 (하트비트)\n\n

필드 설명:
  data:  이벤트 데이터 (여러 줄 = 여러 data: 필드)
  id:    이벤트 ID (클라이언트 Last-Event-ID 헤더로 재연결 시 전송)
  event: 이벤트 타입 (없으면 "message")
  retry: 재연결 대기 시간 (ms)
  : (콜론+공백): 주석 / 하트비트

이벤트 구분: 빈 줄 (\n\n)
```

---

## 🔬 내부 동작 원리

### 1. SseEmitter 내부 구조와 send() 경로

```java
// SseEmitter extends ResponseBodyEmitter

// 1. Controller가 SseEmitter 반환
// 2. ResponseBodyEmitterReturnValueHandler.handleReturnValue()
//    → mavContainer.setRequestHandled(true)
//    → emitter.initialize(outputMessage) (응답 스트림 연결)
//    → request.startAsync() (Servlet 비동기 모드)
//    → emitter를 비동기 요청에 연결

// 3. 클라이언트에 초기 응답 헤더 전송:
//    HTTP/1.1 200 OK
//    Content-Type: text/event-stream
//    Transfer-Encoding: chunked  (chunked 전송 → 스트리밍)
//    Cache-Control: no-cache

// 4. 이후 별도 스레드에서 send() 호출
public synchronized void send(SseEventBuilder builder) throws IOException {
    Set<DataWithMediaType> dataSet = builder.build();
    for (DataWithMediaType data : dataSet) {
        // "data: 내용\n\n" 형식으로 응답 스트림에 쓰기
        super.send(data.getData(), data.getMediaType());
    }
}

// 5. ResponseBodyEmitter.send() 내부:
protected void send(Object object, MediaType mediaType) throws IOException {
    synchronized (this) {
        // 아직 완료되지 않은 경우
        if (this.complete) throw new IllegalStateException("SseEmitter already completed");
        // 데이터 쓰기 핸들러 호출 (HttpMessageConverter 경유)
        this.handler.send(object, mediaType);
        // → response.getOutputStream().write(...)
        // → flush() → 클라이언트로 즉시 전송 (chunked)
    }
}
```

### 2. SseEmitter 레지스트리 패턴 — 실전 구현

```java
// SseEmitter 관리 레지스트리
@Component
public class SseEmitterRegistry {

    // userId → SseEmitter 목록 (하나의 사용자가 여러 탭 열 수 있음)
    private final Map<Long, List<SseEmitter>> emitters = new ConcurrentHashMap<>();

    public SseEmitter createEmitter(Long userId) {
        SseEmitter emitter = new SseEmitter(60_000L);  // 60초 타임아웃

        // 완료/타임아웃/오류 시 레지스트리에서 제거
        emitter.onCompletion(() -> remove(userId, emitter));
        emitter.onTimeout(() -> {
            remove(userId, emitter);
            emitter.complete();  // 타임아웃 시 명시적 완료
        });
        emitter.onError(e -> remove(userId, emitter));

        emitters.computeIfAbsent(userId, k -> new CopyOnWriteArrayList<>()).add(emitter);
        return emitter;
    }

    // 특정 사용자에게 이벤트 전송
    public void sendToUser(Long userId, Object data) {
        List<SseEmitter> userEmitters = emitters.getOrDefault(userId, List.of());
        List<SseEmitter> deadEmitters = new ArrayList<>();

        for (SseEmitter emitter : userEmitters) {
            try {
                emitter.send(SseEmitter.event()
                    .id(String.valueOf(System.currentTimeMillis()))
                    .event("notification")
                    .data(data, MediaType.APPLICATION_JSON)
                    .reconnectTime(3000L));
            } catch (IOException e) {
                // send() 실패 = 연결 끊어짐
                deadEmitters.add(emitter);
            }
        }
        userEmitters.removeAll(deadEmitters);
    }

    // 전체 브로드캐스트
    public void broadcast(Object data) {
        emitters.values().stream()
            .flatMap(List::stream)
            .forEach(emitter -> {
                try {
                    emitter.send(SseEmitter.event().data(data, MediaType.APPLICATION_JSON));
                } catch (IOException e) {
                    // 연결 끊어진 emitter는 자동 제거됨 (onError 콜백)
                }
            });
    }

    private void remove(Long userId, SseEmitter emitter) {
        List<SseEmitter> list = emitters.get(userId);
        if (list != null) {
            list.remove(emitter);
            if (list.isEmpty()) emitters.remove(userId);
        }
    }
}

// Controller
@RestController
@RequiredArgsConstructor
public class NotificationController {

    private final SseEmitterRegistry registry;

    @GetMapping(value = "/notifications/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter streamNotifications(@AuthenticationPrincipal Long userId) {
        SseEmitter emitter = registry.createEmitter(userId);

        // 초기 연결 확인 이벤트 (빈 data 전송으로 연결 확인)
        try {
            emitter.send(SseEmitter.event().comment("connected"));
        } catch (IOException e) {
            emitter.completeWithError(e);
        }

        return emitter;
    }
}

// 이벤트 발행 (서비스 레이어)
@Service
@RequiredArgsConstructor
public class NotificationService {

    private final SseEmitterRegistry registry;

    public void notifyUser(Long userId, NotificationDto notification) {
        userRepository.save(...);  // DB 저장
        registry.sendToUser(userId, notification);  // SSE 전송
    }
}
```

### 3. 하트비트 — 프록시/방화벽 연결 유지

```java
// 장시간 이벤트가 없을 때 연결 끊김 방지
@Component
@RequiredArgsConstructor
public class SseHeartbeatScheduler {

    private final SseEmitterRegistry registry;

    @Scheduled(fixedDelay = 15_000)  // 15초마다
    public void sendHeartbeat() {
        registry.broadcast(
            SseEmitter.event().comment("heartbeat")
            // ":\n\n" 형식 — 데이터 없는 주석 (클라이언트 처리 불필요)
        );
    }
}
```

### 4. ResponseBodyEmitter vs StreamingResponseBody

```java
// ResponseBodyEmitter: 객체 단위 스트리밍 (MessageConverter 활용)
@GetMapping("/objects")
public ResponseBodyEmitter streamObjects() {
    ResponseBodyEmitter emitter = new ResponseBodyEmitter();
    executor.execute(() -> {
        emitter.send(user1, MediaType.APPLICATION_JSON);
        emitter.send(user2, MediaType.APPLICATION_JSON);
        emitter.complete();
    });
    return emitter;
}

// SseEmitter extends ResponseBodyEmitter (SSE 전용 빌더 추가)

// StreamingResponseBody: 바이트 단위 스트리밍 (직접 OutputStream 접근)
@GetMapping("/binary")
public StreamingResponseBody streamBinary() {
    return outputStream -> {
        // OutputStream에 직접 쓰기
        byte[] data = largeFileService.getBytes();
        outputStream.write(data);
        outputStream.flush();
    };
}
// → 대용량 파일 다운로드, 바이너리 스트리밍에 적합
```

---

## 💻 실험으로 확인하기

### 실험 1: 브라우저 SSE 연결

```javascript
// 클라이언트 코드
const eventSource = new EventSource('/notifications/stream');

eventSource.onopen = () => console.log('연결됨');

eventSource.addEventListener('notification', (e) => {
    const notification = JSON.parse(e.data);
    console.log('알림:', notification);
    console.log('ID:', e.lastEventId);
});

eventSource.onerror = (e) => {
    console.log('오류 또는 재연결:', e);
    // 연결 끊기면 브라우저가 자동으로 재연결 시도 (retry 필드 기반)
};
```

### 실험 2: curl로 SSE 스트림 확인

```bash
curl -N -H "Accept: text/event-stream" http://localhost:8080/notifications/stream
# :connected
#
# id:1710000000000
# event:notification
# data:{"type":"MESSAGE","content":"새 메시지","userId":42}
# retry:3000
#
# :heartbeat
#
```

---

## 🌐 HTTP 레벨 분석

```
GET /notifications/stream HTTP/1.1
Accept: text/event-stream

[요청 스레드]
  Controller → SseEmitter 반환
  ResponseBodyEmitterReturnValueHandler:
    → response.setContentType("text/event-stream")
    → response.setHeader("Cache-Control", "no-cache")
    → response.flushBuffer() (헤더 즉시 전송)
    → request.startAsync()
  요청 스레드 반환

HTTP/1.1 200 OK
Content-Type: text/event-stream;charset=UTF-8
Cache-Control: no-cache
Transfer-Encoding: chunked
[연결 유지]

[이벤트 발생 시: 다른 스레드]
  registry.sendToUser(userId, notification)
  → emitter.send(event)
  → response.getOutputStream().write("id:...\nevent:notification\ndata:{...}\nretry:3000\n\n")
  → flush() → 클라이언트로 즉시 전송 (chunked)

id:1710000000000
event:notification
data:{"type":"MESSAGE","content":"새 메시지"}
retry:3000

[타임아웃 또는 complete() 호출]
  → asyncContext.complete() 또는 dispatch()
  → 연결 종료
```

---

## 📌 핵심 정리

```
SseEmitter 처리 흐름
  Controller → SseEmitter 반환
  ResponseBodyEmitterReturnValueHandler → startAsync()
  초기 응답 헤더 전송 (text/event-stream, chunked)
  요청 스레드 반환

이벤트 전송
  별도 스레드에서 emitter.send() → flush() → 클라이언트

연결 수명 관리
  onCompletion/onTimeout/onError 콜백
  레지스트리에서 제거
  타임아웃: 명시적 complete() 필요

연결 끊김 감지
  다음 send() 시 IOException 발생
  → 죽은 emitter 레지스트리에서 제거

하트비트
  SseEmitter.event().comment("heartbeat")
  → ": heartbeat\n\n" — 클라이언트 처리 불필요
  → 프록시/방화벽 연결 유지용

SseEmitter vs StreamingResponseBody
  SseEmitter: text/event-stream, SSE 프로토콜
  StreamingResponseBody: 바이너리, OutputStream 직접 제어
```

---

## 🤔 생각해볼 문제

**Q1.** SSE 연결에서 `Last-Event-ID` 헤더는 무엇을 의미하며, 서버는 이를 어떻게 활용해야 하는가?

**Q2.** `SseEmitter` 생성 시 타임아웃을 `-1`로 설정하면 어떻게 되는가?

**Q3.** 여러 서버 인스턴스(스케일 아웃)가 있을 때 SSE 레지스트리 패턴의 문제점과 해결 방법은?

> 💡 **해설**
>
> **Q1.** `id:` 필드로 각 이벤트에 ID를 부여하면, 클라이언트가 연결 끊김 후 재연결 시 `Last-Event-ID: {마지막 받은 id}` 헤더를 함께 전송합니다. 서버는 이 헤더를 확인해 클라이언트가 마지막으로 받은 이벤트 이후의 누락된 이벤트를 재전송할 수 있습니다. `@RequestHeader("Last-Event-ID")`로 값을 받아 DB나 큐에서 해당 ID 이후의 이벤트를 조회해 재전송하는 패턴을 구현할 수 있습니다.
>
> **Q2.** 타임아웃 `-1`은 타임아웃 없음(무한 대기)을 의미합니다. 서블릿 컨테이너의 기본 비동기 타임아웃도 비활성화됩니다. 클라이언트가 연결을 끊어도 다음 `send()` 시도 전까지 서버가 알 수 없으므로, 메모리에 무한히 `SseEmitter`가 쌓일 수 있습니다. 반드시 하트비트 + `onError` 콜백으로 죽은 연결을 정리하는 로직을 구현해야 합니다.
>
> **Q3.** 각 서버 인스턴스가 자체 메모리에 `SseEmitter` 맵을 가지므로, 서버 A에 연결한 클라이언트는 서버 B에서 발생한 이벤트를 받을 수 없습니다. 해결 방법: Redis Pub/Sub를 브로커로 사용해 모든 서버 인스턴스가 메시지를 구독하고, 자신에게 연결된 클라이언트에게만 전달합니다. 또는 Spring의 `@EventListener` + Redis 메시지 리스너를 조합해 분산 이벤트 브로드캐스트를 구현합니다.

---

<div align="center">

**[⬅️ 이전: Async Request Processing](./01-async-request-processing.md)** | **[홈으로 🏠](../README.md)** | **[다음: File Upload ➡️](./03-file-upload-multipart.md)**

</div>
