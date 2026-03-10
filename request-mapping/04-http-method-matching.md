# HTTP Method 매칭과 OPTIONS 처리 — 405와 CORS Preflight의 내부

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `RequestMethodsRequestCondition`은 HTTP 메서드를 어떻게 비교하는가?
- `405 Method Not Allowed` 응답이 생성되는 정확한 시점과 `Allow` 헤더가 채워지는 방법은?
- `OPTIONS` 요청이 `doDispatch()`를 거치는가, 아니면 별도로 처리되는가?
- CORS Preflight `OPTIONS` 요청과 일반 `OPTIONS` 요청은 어떻게 구분되어 처리되는가?
- `HEAD` 요청은 `GET` 핸들러로 처리될 수 있는가?
- `@RequestMapping` 없이 모든 HTTP 메서드를 받으려면 어떻게 하는가?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: URL이 같아도 HTTP 메서드에 따라 다른 의미를 가진다

```
RESTful API 설계:
  GET    /users/{id} → 조회
  PUT    /users/{id} → 전체 수정
  PATCH  /users/{id} → 부분 수정
  DELETE /users/{id} → 삭제

요청 URL은 동일 → HTTP 메서드로 의미 구분

문제 상황:
  클라이언트가 DELETE /users/1 요청
  서버에는 GET /users/{id}만 등록되어 있음
  → URL은 매칭 → 메서드는 불일치
  → 단순히 404 반환하면 클라이언트는 URL 자체가 없다고 오해
  → 올바른 응답: 405 Method Not Allowed + Allow: GET, HEAD, OPTIONS
    → "이 URL은 존재하지만 이 메서드는 지원하지 않음"
```

---

## 😱 흔한 오해 또는 실수

### Before: HEAD는 별도로 구현해야 한다

```java
// ❌ 잘못된 이해
// "HEAD 요청을 처리하려면 별도 @RequestMapping(method=HEAD)가 필요하다"

// ✅ 실제:
// HEAD는 GET과 동일하게 처리되며, 응답 본문만 제거됨
// HEAD /users/1 → GET 핸들러인 UserController.getUser() 호출
//               → 응답 본문 생성 → 전송 전에 본문 제거
//               → 헤더만 클라이언트에 전달

// 내부 처리:
// FrameworkServlet.doHead() → processRequest() → doDispatch()
// → GET 핸들러 실행 → HttpServlet의 NoBodyResponse로 응답 본문 차단
```

### Before: OPTIONS 요청은 DispatcherServlet이 처리하지 않는다

```
❌ 잘못된 이해:
  "OPTIONS는 서버 레벨에서 자동으로 처리된다"

✅ 실제 (Spring Boot 기본값: dispatchOptionsRequest=true):
  OPTIONS → FrameworkServlet.doOptions()
    → dispatchOptionsRequest == true → processRequest() → doDispatch()
    → HandlerMapping에서 OPTIONS 전용 핸들러 탐색
    → 없으면 HttpOptionsHandler가 Allow 헤더 자동 생성

dispatchOptionsRequest=false (비기본값):
  OPTIONS → HttpServlet.doOptions()
    → 등록된 메서드 목록 기반 Allow 헤더만 반환
    → Spring MVC CORS 처리 미적용
```

---

## ✨ 올바른 이해와 사용

### After: HTTP 메서드 매칭 전체 흐름

```
요청: DELETE /users/1

RequestMappingHandlerMapping.getHandler()
  → lookupHandlerMethod("/users/1", DELETE 요청)
    → 후보 수집:
        {GET [/users/{id}]}    → DELETE 메서드 조건 평가 → 불일치
        {PUT [/users/{id}]}    → DELETE 메서드 조건 평가 → 불일치
        {DELETE [/users/{id}]} → DELETE 메서드 조건 평가 → 일치 ✅
    → 매칭 성공 → HandlerMethod 반환

요청: PATCH /users/1 (PATCH 핸들러 없을 때)
  → 모든 {URL 일치, 메서드 불일치} → null 반환
  → handleNoMatch() 호출
    → URL 일치하는 매핑의 지원 메서드 목록 수집
    → MethodNotAllowedException("PATCH", [GET, PUT, DELETE, OPTIONS]) throw
  → DefaultHandlerExceptionResolver.handleMethodNotAllowed()
    → response.addHeader("Allow", "GET, PUT, DELETE, HEAD, OPTIONS")
    → response.sendError(405)
```

---

## 🔬 내부 동작 원리

### 1. RequestMethodsRequestCondition — 메서드 비교

```java
// RequestMethodsRequestCondition.getMatchingCondition()
@Override
@Nullable
public RequestMethodsRequestCondition getMatchingCondition(HttpServletRequest request) {
    // Preflight OPTIONS 요청 → 조건 없이 통과 (CORS 처리를 위해)
    if (CorsUtils.isPreFlightRequest(request)) {
        return PRE_FLIGHT_MATCH;
    }

    if (getMethods().isEmpty()) {
        // HTTP 메서드 조건이 없으면 모든 메서드 허용
        if (RequestMethod.OPTIONS.name().equals(request.getMethod())
                && !DispatcherType.ERROR.equals(request.getDispatcherType())) {
            return null;  // OPTIONS는 별도 처리
        }
        return this;
    }

    return matchRequestMethod(request.getMethod());
}

private RequestMethodsRequestCondition matchRequestMethod(String httpMethodValue) {
    RequestMethod requestMethod;
    try {
        requestMethod = RequestMethod.valueOf(httpMethodValue);

        // HEAD 요청 → GET 핸들러로 처리
        if (requestMethod == RequestMethod.HEAD && getMethods().contains(RequestMethod.GET)) {
            return GET_CONDITION;  // GET 조건을 반환 (HEAD를 GET으로 처리)
        }

        // OPTIONS 요청 → 암묵적으로 지원
        if (requestMethod == RequestMethod.OPTIONS) {
            if (!getMethods().isEmpty()
                    && !getMethods().contains(RequestMethod.OPTIONS)) {
                return null;  // OPTIONS 핸들러 없음 → HttpOptionsHandler로 폴백
            }
        }

    } catch (IllegalArgumentException ex) {
        return null;  // 알 수 없는 HTTP 메서드
    }

    if (getMethods().contains(requestMethod)) {
        return new RequestMethodsRequestCondition(requestMethod);
    }
    return null;  // 메서드 불일치
}
```

### 2. handleNoMatch() — 405 응답 생성

```java
// RequestMappingHandlerMapping.handleNoMatch()
@Override
@Nullable
protected HandlerMethod handleNoMatch(
        Set<RequestMappingInfo> infos,
        String lookupPath,
        HttpServletRequest request) throws ServletException {

    PartialMatchHelper helper = new PartialMatchHelper(infos, request);

    if (helper.isEmpty()) {
        return null;  // URL 자체가 등록되지 않음 → 404
    }

    // URL은 일치하는 매핑이 있음 → 어떤 조건이 불일치인지 분석

    if (helper.hasMethodsMismatch()) {
        // HTTP 메서드 불일치 → 405
        Set<String> methods = helper.getAllowedMethods();
        // HEAD, OPTIONS는 자동 추가
        if (HttpMethod.OPTIONS.matches(request.getMethod())) {
            Set<MediaType> mediaTypes = helper.getConsumableMediatypes();
            HttpOptionsHandler handler = new HttpOptionsHandler(methods, mediaTypes);
            return new HandlerMethod(handler,
                HttpOptionsHandler.class.getMethod("handle"));
        }
        throw new MethodNotAllowedException(request.getMethod(), methods);
    }

    if (helper.hasConsumesMismatch()) {
        // Content-Type 불일치 → 415
        Set<MediaType> mediaTypes = helper.getConsumableMediatypes();
        throw new HttpMediaTypeNotSupportedException(
            request.getContentType() != null
                ? MediaType.parseMediaType(request.getContentType()) : null,
            new ArrayList<>(mediaTypes));
    }

    if (helper.hasProducesMismatch()) {
        // Accept 불일치 → 406
        Set<MediaType> mediaTypes = helper.getProducibleMediatypes();
        throw new HttpMediaTypeNotAcceptableException(new ArrayList<>(mediaTypes));
    }

    if (helper.hasParamsMismatch()) {
        throw new UnsatisfiedServletRequestParameterException(
            helper.getParamConditions(), request.getParameterMap());
    }

    return null;
}
```

### 3. OPTIONS 처리 — 두 가지 경로

```java
// 경로 1: 명시적 @RequestMapping(method=OPTIONS) 핸들러 있을 때
@RequestMapping(value = "/users", method = RequestMethod.OPTIONS)
public ResponseEntity<Void> options() {
    return ResponseEntity.ok()
        .allow(HttpMethod.GET, HttpMethod.POST, HttpMethod.OPTIONS)
        .build();
}
// → 일반 핸들러처럼 처리

// 경로 2: OPTIONS 핸들러 없을 때 (가장 일반적)
// FrameworkServlet.doOptions() → dispatchOptionsRequest=true → doDispatch()
// → handleNoMatch() → HttpOptionsHandler 자동 생성
public class HttpOptionsHandler {
    private final Set<HttpMethod> supportedMethods;
    private final Set<MediaType> acceptPatch;

    @RequestMapping
    public HttpHeaders handle() {
        HttpHeaders headers = new HttpHeaders();
        headers.setAllow(this.supportedMethods);
        if (!this.acceptPatch.isEmpty()) {
            headers.setAcceptPatch(new ArrayList<>(this.acceptPatch));
        }
        return headers;
    }
}
```

### 4. CORS Preflight 구분 처리

```java
// CorsUtils.isPreFlightRequest()
public static boolean isPreFlightRequest(HttpServletRequest request) {
    return (HttpMethod.OPTIONS.matches(request.getMethod()) &&
            request.getHeader(HttpHeaders.ORIGIN) != null &&
            request.getHeader(HttpHeaders.ACCESS_CONTROL_REQUEST_METHOD) != null);
}
// Preflight 조건:
//   HTTP 메서드 = OPTIONS
//   + Origin 헤더 존재
//   + Access-Control-Request-Method 헤더 존재

// Preflight 처리 흐름:
// doDispatch() → getHandler()
// → AbstractHandlerMapping.getHandler()
//   → getCorsHandlerExecutionChain()
//     → isPreFlightRequest() == true
//     → PreFlightHandler 반환 (실제 핸들러 대신)
//     → PreFlightHandler: CORS 설정 검증 + CORS 응답 헤더 작성
//     → 실제 핸들러 메서드 실행 없음
```

---

## 💻 실험으로 확인하기

### 실험 1: 각 HTTP 메서드 응답 확인

```bash
# GET — 정상
curl -v http://localhost:8080/users/1
# HTTP/1.1 200 OK

# HEAD — 응답 본문 없이 헤더만
curl -v -X HEAD http://localhost:8080/users/1
# HTTP/1.1 200 OK
# Content-Type: application/json  ← 헤더는 있음
# Content-Length: 42              ← 길이는 있음
# (응답 본문 없음)

# OPTIONS — 지원 메서드 목록 확인
curl -v -X OPTIONS http://localhost:8080/users/1
# HTTP/1.1 200 OK
# Allow: GET, HEAD, OPTIONS

# DELETE — 미등록 시 405
curl -v -X DELETE http://localhost:8080/users/1
# HTTP/1.1 405 Method Not Allowed
# Allow: GET, HEAD, OPTIONS
```

### 실험 2: CORS Preflight vs 일반 OPTIONS 구분

```bash
# 일반 OPTIONS (Preflight 아님)
curl -v -X OPTIONS http://localhost:8080/users/1
# HTTP/1.1 200 OK
# Allow: GET, HEAD, OPTIONS

# CORS Preflight (Origin + Access-Control-Request-Method 포함)
curl -v -X OPTIONS http://localhost:8080/users/1 \
     -H "Origin: http://frontend.example.com" \
     -H "Access-Control-Request-Method: GET" \
     -H "Access-Control-Request-Headers: Content-Type"
# HTTP/1.1 200 OK
# Access-Control-Allow-Origin: http://frontend.example.com
# Access-Control-Allow-Methods: GET
# Access-Control-Allow-Headers: Content-Type
# Access-Control-Max-Age: 1800
# (CORS 응답 헤더 포함)
```

### 실험 3: 메서드 없는 @RequestMapping

```java
// HTTP 메서드 조건 없음 → 모든 메서드 허용
@RequestMapping("/all-methods")
public String anyMethod(HttpServletRequest request) {
    return "메서드: " + request.getMethod();
}
```

```bash
curl -X GET    http://localhost:8080/all-methods  # → "메서드: GET"
curl -X POST   http://localhost:8080/all-methods  # → "메서드: POST"
curl -X DELETE http://localhost:8080/all-methods  # → "메서드: DELETE"
```

---

## 🌐 HTTP 레벨 분석

```
HTTP 메서드 매칭 결과별 응답:

정상 매칭:
  GET /users/1 → UserController#getUser
  HTTP/1.1 200 OK
  Content-Type: application/json
  {"id":1,"name":"홍길동"}

405 Method Not Allowed:
  DELETE /users/1 (DELETE 핸들러 없음)
  HTTP/1.1 405 Method Not Allowed
  Allow: GET, HEAD, OPTIONS     ← 지원하는 메서드 목록
  Content-Type: application/json (Spring Boot 기본 에러 응답)
  {"timestamp":"...","status":405,"error":"Method Not Allowed",...}

HEAD 응답:
  HEAD /users/1
  HTTP/1.1 200 OK
  Content-Type: application/json
  Content-Length: 42
  Date: ...
  (빈 응답 본문 — NoBodyResponseWrapper로 본문 차단)

OPTIONS 응답 (CORS 설정 없을 때):
  OPTIONS /users/1
  HTTP/1.1 200 OK
  Allow: GET, HEAD, OPTIONS
  Content-Length: 0

OPTIONS 응답 (CORS 설정 있을 때):
  OPTIONS /users/1 + Origin 헤더
  HTTP/1.1 200 OK
  Access-Control-Allow-Origin: *
  Access-Control-Allow-Methods: GET, POST
  Access-Control-Max-Age: 3600
  Allow: GET, HEAD, OPTIONS
```

---

## 🤔 트레이드오프

```
메서드 레벨 라우팅 (@GetMapping, @PostMapping 분리):
  장점  URL당 명확한 HTTP 의미 분리 (RESTful)
        405 응답으로 클라이언트에게 명확한 오류 피드백
        OPTIONS 자동 처리로 CORS Preflight 간소화
  단점  같은 URL에 여러 메서드를 별개 핸들러로 분산 → 응집성 저하
        → @RequestMapping 하나에 method={GET,POST}로 묶어 응집할 수도 있음

dispatchOptionsRequest=true (기본):
  장점  OPTIONS도 Spring MVC 파이프라인(인터셉터 등) 거침
        CORS 처리, 커스텀 OPTIONS 핸들러 등록 가능
  단점  추가 처리 오버헤드
        OPTIONS 요청도 인터셉터를 거쳐야 할 수도 있음 (주의)

HEAD 자동 처리 (GET으로 폴백):
  장점  HEAD 핸들러를 별도 구현하지 않아도 됨
  단점  GET 핸들러가 실제로 실행됨 (데이터 조회 등 부작용)
        → 비용이 큰 GET 핸들러의 경우 HEAD 전용 구현 고려
```

---

## 📌 핵심 정리

```
HTTP 메서드 매칭
  RequestMethodsRequestCondition.getMatchingCondition()
  → 비어있으면 모든 메서드 허용
  → HEAD → GET 핸들러로 폴백 (본문은 NoBodyResponseWrapper로 차단)
  → Preflight OPTIONS → PRE_FLIGHT_MATCH (항상 통과)

405 Method Not Allowed 생성 경로
  lookupHandlerMethod() → 후보 없음 → handleNoMatch()
  → URL 일치 + 메서드 불일치 감지
  → 지원 메서드 목록 수집 + Allow 헤더 작성
  → MethodNotAllowedException → DefaultHandlerExceptionResolver → 405

OPTIONS 처리 두 가지 경로
  ① 명시적 @RequestMapping(method=OPTIONS) → 일반 핸들러처럼 처리
  ② 없을 때 → HttpOptionsHandler 자동 생성 → Allow 헤더 반환

CORS Preflight 구분
  CorsUtils.isPreFlightRequest():
    OPTIONS + Origin 헤더 + Access-Control-Request-Method 헤더
  → PreFlightHandler로 분기 → CORS 헤더 응답 → 실제 핸들러 실행 없음
```

---

## 🤔 생각해볼 문제

**Q1.** `GET /users/{id}` 핸들러만 등록된 상태에서 `HEAD /users/1` 요청을 보내면 `Content-Length` 헤더가 응답에 포함되는가? 포함된다면 그 값은 실제 응답 본문의 길이인가?

**Q2.** CORS 설정(`@CrossOrigin`)이 특정 Controller에만 있을 때, 다른 Controller의 URL에 대해 Preflight 요청이 오면 어떻게 처리되는가? 403이 반환되는가, 200이 반환되는가?

**Q3.** `@RequestMapping(method = {GET, POST})` 와 각각 `@GetMapping`과 `@PostMapping`을 선언한 두 가지 방식이 있습니다. OPTIONS 요청 시 Allow 헤더의 내용이 다를 수 있는가?

> 💡 **해설**
>
> **Q1.** `Content-Length` 헤더가 포함됩니다. `HEAD` 요청 처리 시 Spring MVC는 GET 핸들러를 실제로 실행하고 `NoBodyResponseWrapper`(또는 `HttpServlet`의 내장 `NoBodyResponse`)로 응답 본문을 차단합니다. 이 래퍼는 본문에 쓰이는 바이트 수를 카운트해 `Content-Length`에 설정합니다. 따라서 `Content-Length`는 실제 GET 응답 본문의 크기와 동일합니다. 이것이 HEAD의 목적 — 본문을 전송하지 않고 리소스 크기 확인 — 을 실현하는 방식입니다.
>
> **Q2.** CORS 설정이 없는 Controller의 URL에 Preflight가 오면, Spring MVC는 해당 URL에 CORS 설정이 없다고 판단합니다. 이 경우 `CorsProcessor`(기본: `DefaultCorsProcessor`)가 `Origin` 헤더를 허용하지 않아 `403 Forbidden`이 반환됩니다. Preflight 요청은 "이 URL에 Cross-Origin 요청을 보내도 되는가"를 미리 묻는 것이므로, 허용 설정이 없으면 거부합니다. 전역 CORS 설정은 `WebMvcConfigurer.addCorsMappings()`로 추가할 수 있습니다.
>
> **Q3.** 기술적으로는 Allow 헤더 내용이 동일합니다. 두 방식 모두 `RequestMappingInfo`에 `{GET, POST}` 메서드 조건을 가진 매핑이 등록됩니다. `handleNoMatch()` 에서 `helper.getAllowedMethods()`는 등록된 모든 매핑의 메서드 조건을 합산해 `{GET, POST, HEAD, OPTIONS}`를 반환합니다. 차이가 생길 수 있는 경우는 각 메서드를 별도 `@RequestMapping`으로 나눌 때 한쪽에만 `produces` 조건이 추가된다면 두 Info가 분리되고 OPTIONS 핸들러 탐색 시 조건 평가가 다를 수 있습니다.

---

<div align="center">

**[⬅️ 이전: Path Pattern 매칭 전략](./03-path-pattern-matching.md)** | **[홈으로 🏠](../README.md)** | **[다음: Content Negotiation — Accept 헤더 처리 ➡️](./05-content-negotiation.md)**

</div>
