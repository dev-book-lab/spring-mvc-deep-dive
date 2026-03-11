# ResponseEntity vs @ResponseStatus — HTTP 상태 코드 설정의 두 가지 방법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `HttpEntityMethodProcessor`가 `ResponseEntity`를 처리하는 단계별 과정은?
- `@ResponseStatus`는 언제 읽히며, `ResponseEntity`와 동시에 사용하면 어떻게 되는가?
- `ResponseEntity.status()` vs `ResponseEntity.ok()` 빌더 API는 내부적으로 어떻게 다른가?
- 동적으로 상태 코드를 결정해야 할 때 `@ResponseStatus`로는 불가능한 이유는?
- `ResponseEntity<Void>` vs `ResponseEntity.noContent().build()`의 차이는?
- `Location` 헤더 같은 커스텀 헤더를 응답에 추가하는 각 방법의 처리 경로는?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: 상황에 따라 다른 상태 코드와 헤더가 필요하다

```
REST API 설계 기준:
  POST /users         → 201 Created + Location: /users/42
  GET /users/42       → 200 OK 또는 404 Not Found (조건에 따라)
  DELETE /users/42    → 204 No Content
  PATCH /users/42     → 200 OK 또는 304 Not Modified

@ResponseStatus(HttpStatus.CREATED)  → 컴파일 타임 고정
  → 매번 201? 조건에 따라 200/201 선택 불가

ResponseEntity:
  → 런타임에 상태 코드·헤더·본문 완전 제어
  → 빌더 패턴으로 가독성 높은 응답 구성
```

---

## 😱 흔한 오해 또는 실수

### Before: @ResponseStatus와 ResponseEntity를 같이 쓰면 둘 다 적용된다

```java
// ❌ 잘못된 이해
@ResponseStatus(HttpStatus.CREATED)   // 201 설정
@PostMapping("/users")
public ResponseEntity<User> create(@RequestBody UserDto dto) {
    User user = userService.create(dto);
    return ResponseEntity.ok(user);   // 200 설정
    // "201과 200 중 어떤 게 적용될까?"
}

// ✅ 실제:
// ResponseEntity가 반환되면 ResponseEntity의 상태 코드가 우선
// → 200 OK 반환
// @ResponseStatus는 ResponseEntity가 없을 때(또는 null 반환 시)에만 적용

// 명확한 의도 표현:
// 항상 201이면 → @ResponseStatus(CREATED) + User 직접 반환
// 조건에 따라 달라지면 → ResponseEntity 사용, @ResponseStatus 제거
```

### Before: ResponseEntity.noContent()는 빈 JSON 객체 {}를 반환한다

```
❌ 잘못된 이해:
  "ResponseEntity.noContent().build() → {}"

✅ 실제:
  ResponseEntity.noContent().build() = ResponseEntity<Void>(204, no body)
  → HTTP/1.1 204 No Content
  → Content-Length: 0
  → 응답 본문 없음 (null body)
  → 204 상태 코드 자체가 "본문 없음"을 의미
```

---

## ✨ 올바른 이해와 사용

### After: ResponseEntity 처리 전체 흐름

```java
@PostMapping("/users")
public ResponseEntity<User> create(@RequestBody UserDto dto) {
    User user = userService.create(dto);
    URI location = URI.create("/users/" + user.getId());
    return ResponseEntity
        .status(HttpStatus.CREATED)         // 201
        .location(location)                  // Location 헤더
        .header("X-User-Id", user.getId().toString())
        .body(user);                         // 응답 본문
}

처리 흐름:
① ReturnValueHandlerComposite:
   HttpEntityMethodProcessor.supportsReturnType() → ResponseEntity → true

② HttpEntityMethodProcessor.handleReturnValue():
   outputMessage.setStatusCode(HttpStatus.CREATED)
   responseHeaders.set("Location", "/users/42")
   responseHeaders.set("X-User-Id", "42")
   writeWithMessageConverters(user, ...) → JSON 직렬화

③ mavContainer.setRequestHandled(true)

결과:
HTTP/1.1 201 Created
Location: /users/42
X-User-Id: 42
Content-Type: application/json
{"id":42,"name":"홍길동"}
```

---

## 🔬 내부 동작 원리

### 1. HttpEntityMethodProcessor.handleReturnValue()

```java
// HttpEntityMethodProcessor.java
@Override
public void handleReturnValue(@Nullable Object returnValue,
        MethodParameter returnType,
        ModelAndViewContainer mavContainer,
        NativeWebRequest webRequest) throws Exception {

    mavContainer.setRequestHandled(true);  // View 렌더링 불필요 선언

    HttpEntity<?> httpEntity = (HttpEntity<?>) returnValue;
    if (httpEntity == null) {
        return; // null ResponseEntity → 응답 본문 없음
    }

    // ① 응답 헤더 설정
    HttpHeaders entityHeaders = httpEntity.getHeaders();
    HttpServletResponse servletResponse = webRequest.getNativeResponse(HttpServletResponse.class);
    ServerHttpResponse outputMessage = new ServletServerHttpResponse(servletResponse);

    if (!entityHeaders.isEmpty()) {
        entityHeaders.forEach((key, value) -> {
            if (HttpHeaders.VARY.equals(key) && outputMessage instanceof AbstractServerHttpResponse) {
                ((AbstractServerHttpResponse) outputMessage).getHeaders()
                    .addVary(StringUtils.toStringArray(value));
            } else {
                outputMessage.getHeaders().put(key, value);
            }
        });
    }

    // ② HTTP 상태 코드 설정
    if (httpEntity instanceof ResponseEntity<?> responseEntity) {
        int returnStatus = responseEntity.getStatusCode().value();
        outputMessage.getServletResponse().setStatus(returnStatus);

        // 304 Not Modified이면 본문 없이 반환
        if (returnStatus == HttpStatus.NOT_MODIFIED.value()) {
            outputMessage.flush();
            return;
        }
    }

    // ③ 본문 직렬화
    Type targetType = ...; // 제네릭 타입 추출
    writeWithMessageConverters(httpEntity.getBody(), returnType, inputMessage, outputMessage);
}
```

### 2. @ResponseStatus 처리 경로

```java
// ServletInvocableHandlerMethod.setResponseStatus()
// → invokeAndHandle() 에서 메서드 실행 후 호출
private void setResponseStatus(ServletWebRequest webRequest) throws IOException {
    HttpStatus status = getResponseStatus();  // @ResponseStatus에서 읽음
    if (status == null) return;

    HttpServletResponse response = webRequest.getResponse();
    String reason = getResponseStatusReason();
    if (StringUtils.hasLength(reason)) {
        response.sendError(status.value(), reason);
    } else {
        response.setStatus(status.value());
    }

    // 이후 ResponseEntity가 있으면 ResponseEntity의 setStatus()가 덮어씀
    // → ResponseEntity 우선
}

// @ResponseStatus가 유효한 경우:
// 1. 반환 타입이 ResponseEntity가 아닐 때
// 2. 예외 클래스에 @ResponseStatus 붙여서 예외 처리 시 상태 코드 지정할 때
@ResponseStatus(HttpStatus.NOT_FOUND)
public class UserNotFoundException extends RuntimeException { ... }
// → DefaultHandlerExceptionResolver → 404 자동 반환
```

### 3. ResponseEntity 빌더 API 내부

```java
// ResponseEntity 주요 팩토리 메서드들
ResponseEntity.ok(body)
// = new ResponseEntity<>(body, null, HttpStatus.OK)
// = ResponseEntity.status(200).body(body)

ResponseEntity.created(location)
// = ResponseEntity.status(201).location(location).build()
// 주의: body 없음 → build()로 마무리 (body 추가하려면 .body() 사용)

ResponseEntity.noContent()
// = DefaultBuilder(204) → .build() → ResponseEntity<Void>(204, null)
// Content-Length: 0, 본문 없음

ResponseEntity.notFound()
// = DefaultBuilder(404) → .build()

ResponseEntity.status(HttpStatus.ACCEPTED)
// = new DefaultBuilder(202) → 체이닝으로 헤더/본문 추가

// 헤더 설정 체이닝:
ResponseEntity.status(201)
    .header("X-Custom", "value")       // 커스텀 헤더
    .contentType(MediaType.APPLICATION_JSON)  // Content-Type
    .eTag("\"abc123\"")               // ETag 헤더
    .lastModified(Instant.now())       // Last-Modified 헤더
    .cacheControl(CacheControl.maxAge(1, TimeUnit.HOURS))  // Cache-Control
    .body(user);
```

### 4. @ResponseStatus vs ResponseEntity 처리 시점 비교

```
@ResponseStatus:
  읽히는 시점: invokeAndHandle() → setResponseStatus()
               → 메서드 실행 후, ReturnValueHandler 처리 전
  설정 방법:  response.setStatus(code)
  덮어쓰기:   이후 ResponseEntity가 있으면 ResponseEntity가 덮어씀

ResponseEntity:
  읽히는 시점: HttpEntityMethodProcessor.handleReturnValue()
               → ReturnValueHandler 처리 중
  설정 방법:  servletResponse.setStatus(code)
  우선순위:   @ResponseStatus 이후에 실행되므로 항상 최종 값

결론:
  ResponseEntity와 @ResponseStatus 동시 사용 → ResponseEntity 상태 코드 최종 적용
  @ResponseStatus만 사용 → 설정된 상태 코드 적용
  예외 클래스에 @ResponseStatus → 예외 처리 시 자동 상태 코드
```

---

## 💻 실험으로 확인하기

### 실험 1: ResponseEntity 우선순위 확인

```java
@ResponseStatus(HttpStatus.CREATED)  // 201 설정
@PostMapping("/test")
public ResponseEntity<String> test() {
    return ResponseEntity.ok("hello");  // 200 설정
}
```

```bash
curl -i -X POST http://localhost:8080/test
# HTTP/1.1 200 OK  ← ResponseEntity의 200이 우선
# Content-Type: text/plain
# hello
```

### 실험 2: 조건부 상태 코드

```java
@GetMapping("/users/{id}")
public ResponseEntity<User> getUser(@PathVariable Long id) {
    return userService.findById(id)
        .map(user -> ResponseEntity.ok(user))
        .orElse(ResponseEntity.notFound().build());
    // 있으면 200, 없으면 404
}
```

```bash
curl -i http://localhost:8080/users/1    # → 200 OK + body
curl -i http://localhost:8080/users/999  # → 404 Not Found (body 없음)
```

### 실험 3: 201 Created + Location 헤더

```bash
curl -i -X POST http://localhost:8080/users \
     -H "Content-Type: application/json" \
     -d '{"name":"홍길동"}'
# HTTP/1.1 201 Created
# Location: http://localhost:8080/users/42
# Content-Type: application/json
# {"id":42,"name":"홍길동"}
```

---

## 🌐 HTTP 레벨 분석

```
ResponseEntity 처리:
  반환값: ResponseEntity.status(201).location(URI("/users/42")).body(user)

  HttpEntityMethodProcessor:
    response.setStatus(201)
    response.setHeader("Location", "http://localhost:8080/users/42")
    Jackson → {"id":42,"name":"홍길동"}

HTTP/1.1 201 Created
Location: http://localhost:8080/users/42
Content-Type: application/json;charset=UTF-8
Content-Length: 32
{"id":42,"name":"홍길동"}

@ResponseStatus(NOT_FOUND) 예외:
  throw new UserNotFoundException("42")
  → HandlerExceptionResolver
  → response.sendError(404)
HTTP/1.1 404 Not Found
```

---

## 🤔 트레이드오프

```
ResponseEntity 사용:
  장점  런타임 상태 코드 / 헤더 완전 제어
        빌더 패턴 → 가독성
        Content Negotiation, ETag, Cache-Control 등 고급 기능 표현 가능
  단점  반환 타입이 ResponseEntity<T>로 고정 → 테스트 시 unwrap 필요
        간단한 경우에도 장황한 코드

@ResponseStatus 사용:
  장점  단순 고정 상태 코드 → 선언적이고 간결
        예외 클래스에 붙여 전역 처리에 유용
  단점  컴파일 타임 고정 → 동적 상태 코드 불가
        헤더 추가 불가

가이드:
  항상 동일 상태:  @ResponseStatus
  조건부 상태:    ResponseEntity
  헤더 필요:      ResponseEntity
  예외 상태 코드: @ResponseStatus on Exception class
```

---

## 📌 핵심 정리

```
처리 주체
  ResponseEntity → HttpEntityMethodProcessor
  @ResponseStatus → setResponseStatus() (invokeAndHandle 내)

처리 순서
  setResponseStatus() (메서드 실행 후)
  → HttpEntityMethodProcessor.handleReturnValue() (ResponseEntity 있으면 덮어씀)
  → 따라서 ResponseEntity 상태 코드가 최종

ResponseEntity 빌더
  ok() → 200, created(uri) → 201, noContent() → 204
  notFound() → 404, status(code) → 임의 코드
  체이닝: .header(), .contentType(), .eTag(), .cacheControl(), .body()

@ResponseStatus 유효 사용처
  단순 고정 상태 + 본문 직접 반환
  예외 클래스에 붙여 자동 상태 코드 매핑

noContent() 주의
  204 No Content = 본문 없음
  Content-Length: 0, body = null
```

---

## 🤔 생각해볼 문제

**Q1.** `ResponseEntity.created(uri)`를 사용하면 `Location` 헤더가 설정됩니다. `uri`로 상대 경로 `/users/42`를 넣으면 클라이언트는 절대 URL을 받는가, 상대 경로를 받는가?

**Q2.** `ResponseEntity<Void>`를 반환하면서 `Jackson`이 `null` 본문을 직렬화하려 하면 어떻게 되는가?

**Q3.** `@ExceptionHandler` 메서드에서 `ResponseEntity`를 반환하면 `@ResponseStatus`가 붙은 예외가 throw됐을 때 상태 코드는 어느 것이 적용되는가?

> 💡 **해설**
>
> **Q1.** `Location` 헤더에는 `URI.toString()` 값이 그대로 설정됩니다. `/users/42` 같은 상대 경로를 넣으면 그대로 `Location: /users/42`로 응답됩니다. RFC에 따르면 `Location`은 절대 URI를 권장하므로 실무에서는 `UriComponentsBuilder.fromCurrentRequest().path("/{id}").build(user.getId())`처럼 현재 요청 기반 절대 URL을 생성하는 것이 안전합니다.
>
> **Q2.** `ResponseEntity<Void>` 또는 `ResponseEntity.noContent().build()`의 경우 `httpEntity.getBody()`가 `null`을 반환합니다. `writeWithMessageConverters()`는 `body == null`이면 실제 쓰기를 수행하지 않습니다. `204 No Content` 상태 코드이면 서블릿 컨테이너 자체도 본문 출력을 억제합니다. 따라서 Jackson이 호출되지 않고 Content-Length: 0인 빈 응답이 정상적으로 반환됩니다.
>
> **Q3.** `@ExceptionHandler` 메서드가 `ResponseEntity`를 반환하면 `ResponseEntity`의 상태 코드가 최종 적용됩니다. 예외 클래스에 붙은 `@ResponseStatus`는 `DefaultHandlerExceptionResolver`나 `ResponseStatusExceptionResolver`가 직접 처리할 때만 사용됩니다. `@ExceptionHandler`가 먼저 매칭되면 해당 메서드의 `ResponseEntity`가 응답을 완전히 결정합니다.

---

<div align="center">

**[⬅️ 이전: @ResponseBody 변환](./02-response-body-conversion.md)** | **[홈으로 🏠](../README.md)** | **[다음: HttpMessageConverter 선택 과정 ➡️](./04-message-converter-selection.md)**

</div>
