# RFC 7807 Problem Details 응답 설계 — ProblemDetail과 ErrorResponse

---

## 🎯 핵심 질문

- RFC 7807 Problem Details 스펙의 필수/선택 필드 구조는?
- Spring 6.x `ProblemDetail` 클래스는 어떻게 RFC 7807을 구현하는가?
- `ErrorResponse` 인터페이스와 `ProblemDetail`의 관계는?
- `application/problem+json` Content-Type은 어떻게 설정되는가?
- `spring.mvc.problemdetails.enabled=true` 설정이 활성화하는 것은?
- `ProblemDetail`에 커스텀 필드를 추가하는 방법은?

---

## 🔍 왜 이 메커니즘이 존재하는가

```
표준화 전 각 팀마다 다른 오류 응답 형식:
  팀 A: {"error": "NOT_FOUND", "msg": "..."}
  팀 B: {"status": 404, "message": "...", "path": "/users/42"}
  팀 C: {"code": "USER_404", "description": "..."}

RFC 7807 (Problem Details for HTTP APIs):
  오류 응답의 표준 형식 정의
  → 클라이언트가 형식을 예상할 수 있음
  → Content-Type: application/problem+json 으로 오류임을 명시
  → 머신 파싱 가능한 type URI로 오류 종류 식별

Spring 6.x: ProblemDetail 클래스로 RFC 7807 공식 지원
```

---

## 😱 흔한 오해 또는 실수

### Before: application/problem+json은 application/json과 같다

```
❌ 잘못된 이해:
  "두 Content-Type은 동일한 JSON이다"

✅ 실제 차이:
  application/json: 임의의 JSON 데이터
  application/problem+json: RFC 7807 Problem Details 형식의 JSON
    → 반드시 type, title, status 필드를 포함한 구조
    → 클라이언트가 "이것은 오류 응답임"을 명확히 인지 가능
    → 일부 HTTP 클라이언트 라이브러리가 자동으로 오류로 처리

  Spring 6.x: ProblemDetail 반환 시 자동으로 application/problem+json 설정
  → 단, Accept 헤더가 application/json이면 application/json으로 반환될 수 있음
     (Content Negotiation 적용)
```

### Before: ProblemDetail은 Spring 6.x에서만 사용 가능하다

```
✅ 사실이지만 맥락이 중요:
  ProblemDetail 클래스: Spring Framework 6.0 (Spring Boot 3.0+) 추가
  RFC 7807 준수 응답: 이전 버전에서도 구현 가능 (단, 직접 구현)
  
  Spring Boot 2.x 이하:
    ResponseEntityExceptionHandler 상속 + 커스텀 ErrorBody POJO로 유사 구현
    Content-Type: application/problem+json 수동 설정 필요
```

---

## ✨ 올바른 이해와 사용

### After: RFC 7807 스펙 구조

```json
// RFC 7807 Problem Details 표준 필드
{
  "type": "https://example.com/problems/user-not-found",
  // URI, 오류 종류 식별자 (머신이 프로그래밍적으로 처리 가능)
  // 없으면 "about:blank" 기본값

  "title": "User Not Found",
  // 사람이 읽을 수 있는 짧은 요약 (같은 type은 항상 같은 title)

  "status": 404,
  // HTTP 상태 코드 (숫자)

  "detail": "ID 42인 사용자를 찾을 수 없습니다.",
  // 이 특정 발생에 대한 상세 설명 (사람이 읽는 용도)

  "instance": "/api/users/42",
  // 이 특정 발생을 식별하는 URI (요청 경로 등)

  // 확장 필드 (RFC 허용):
  "errors": [
    {"field": "name", "reason": "이름은 필수입니다"}
  ],
  "timestamp": "2024-03-10T15:30:00Z"
}
```

---

## 🔬 내부 동작 원리

### 1. ProblemDetail 클래스 구조

```java
// ProblemDetail.java (Spring 6.x)
public class ProblemDetail {

    private static final URI BLANK_TYPE = URI.create("about:blank");

    private URI type = BLANK_TYPE;      // RFC 7807 type
    private String title;               // RFC 7807 title (상태 코드에서 자동 유추)
    private int status;                 // RFC 7807 status
    @Nullable private String detail;    // RFC 7807 detail
    @Nullable private URI instance;     // RFC 7807 instance
    @Nullable private Map<String, Object> properties; // 확장 필드

    // 팩토리 메서드
    public static ProblemDetail forStatus(HttpStatus status) {
        return forStatus(status.value());
    }

    public static ProblemDetail forStatusAndDetail(HttpStatusCode status, String detail) {
        ProblemDetail pd = forStatus(status);
        pd.setDetail(detail);
        return pd;
    }

    // 확장 필드 추가
    public void setProperty(String name, @Nullable Object value) {
        if (this.properties == null) {
            this.properties = new LinkedHashMap<>();
        }
        this.properties.put(name, value);
    }

    // title 자동 유추 (status → HttpStatus.getReasonPhrase())
    @JsonInclude(JsonInclude.Include.NON_EMPTY)
    @Nullable
    public String getTitle() {
        if (this.title == null && this.status > 0) {
            HttpStatus httpStatus = HttpStatus.resolve(this.status);
            return (httpStatus != null ? httpStatus.getReasonPhrase() : null);
        }
        return this.title;
    }
}
```

### 2. ErrorResponse 인터페이스와 표준 예외 연결

```java
// ErrorResponse 인터페이스 (Spring 6.x)
public interface ErrorResponse {
    HttpStatusCode getStatusCode();
    default HttpHeaders getHeaders() { return HttpHeaders.EMPTY; }
    ProblemDetail getBody();

    // i18n 지원: MessageSource에서 메시지 코드로 title/detail 해석
    default String getTypeMessageCode() {
        return ErrorResponse.getTypeMessageCode(getClass());
    }
}

// 표준 예외들이 ErrorResponse 구현:
public class MethodArgumentNotValidException
        extends BindException implements ErrorResponse {

    private final ProblemDetail body;

    public MethodArgumentNotValidException(MethodParameter parameter, BindingResult result) {
        super(result);
        this.body = ProblemDetail.forStatusAndDetail(HttpStatus.BAD_REQUEST,
            "Invalid request content.");
        // getBody()로 ProblemDetail 접근 가능
    }

    @Override
    public ProblemDetail getBody() {
        return this.body;
    }

    @Override
    public HttpStatusCode getStatusCode() {
        return HttpStatus.BAD_REQUEST;
    }
}
```

### 3. spring.mvc.problemdetails.enabled — 자동 ProblemDetail 활성화

```java
// application.properties/yml:
// spring.mvc.problemdetails.enabled=true

// 이 설정이 활성화하는 것:
// ProblemDetailsExceptionHandler (Spring Boot 3.x)
// → ResponseEntityExceptionHandler를 상속하되
//   handleExceptionInternal()에서 자동으로 ProblemDetail 본문 사용

// Spring Boot의 자동 구성:
@Configuration
@ConditionalOnProperty(prefix = "spring.mvc.problemdetails", name = "enabled", havingValue = "true")
static class ProblemDetailsErrorHandlingConfiguration {
    @Bean
    @ConditionalOnMissingBean(ResponseEntityExceptionHandler.class)
    ProblemDetailsExceptionHandler problemDetailsExceptionHandler() {
        return new ProblemDetailsExceptionHandler();
    }
}

// ProblemDetailsExceptionHandler 내부:
@ControllerAdvice
class ProblemDetailsExceptionHandler extends ResponseEntityExceptionHandler {
    // handleExceptionInternal()을 오버라이드하지 않음
    // → 부모의 기본 구현이 ErrorResponse.getBody()를 자동 사용
    // → ProblemDetail이 응답 본문이 됨
    // → Content-Type: application/problem+json 자동 설정
}
```

### 4. application/problem+json Content-Type 설정 경로

```java
// ProblemDetail 응답 시 Content-Type 설정:
// 1. MappingJackson2HttpMessageConverter가 ProblemDetail 직렬화
// 2. Content-Type 결정: Accept 헤더 협상 + 반환 타입

// Spring 6.x AbstractErrorController에서:
// ProblemDetail 반환 시 MediaType.APPLICATION_PROBLEM_JSON 설정:
public static final MediaType APPLICATION_PROBLEM_JSON =
    MediaType.valueOf("application/problem+json");

// ResponseEntityExceptionHandler에서 직접 헤더 설정:
protected ResponseEntity<Object> handleExceptionInternal(...) {
    if (body instanceof ProblemDetail pd) {
        if (pd.getInstance() == null) {
            URI path = URI.create(((ServletWebRequest) request).getRequest().getRequestURI());
            pd.setInstance(path);
            // instance 자동 설정
        }
    }
    // Content-Type은 Content Negotiation에 의해 결정
    // Accept: application/problem+json → 그대로
    // Accept: application/json → application/json
    // Accept: */* → application/problem+json (ProblemDetail의 기본)
}
```

### 5. 커스텀 ProblemDetail 확장 패턴

```java
// 커스텀 필드 추가
@RestControllerAdvice
public class CustomProblemDetailHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public ProblemDetail handleUserNotFound(UserNotFoundException ex,
                                             HttpServletRequest request) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, ex.getMessage());

        // 표준 필드
        pd.setType(URI.create("https://api.example.com/errors/user-not-found"));
        pd.setTitle("사용자를 찾을 수 없음");
        pd.setInstance(URI.create(request.getRequestURI()));

        // 커스텀 확장 필드
        pd.setProperty("userId", ex.getUserId());
        pd.setProperty("timestamp", Instant.now());
        pd.setProperty("traceId", MDC.get("traceId"));

        return pd;
        // @ResponseBody + ProblemDetail → application/problem+json
    }

    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex, HttpHeaders headers,
            HttpStatusCode status, WebRequest request) {

        ProblemDetail pd = ex.getBody();  // 기본 ProblemDetail 가져오기
        pd.setDetail("입력값 검증에 실패했습니다.");

        // 검증 오류 목록 커스텀 필드로 추가
        List<Map<String, String>> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(fe -> Map.of(
                "field", fe.getField(),
                "reason", String.valueOf(fe.getDefaultMessage())))
            .toList();
        pd.setProperty("errors", errors);

        return ResponseEntity.status(status).headers(headers).body(pd);
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: spring.mvc.problemdetails.enabled=true 효과

```bash
# 활성화 전
curl -X POST http://localhost:8080/users -d '{"name": ""}'
# HTTP/1.1 400
# Content-Type: application/json
# (본문 없거나 기존 오류 형식)

# 활성화 후 (spring.mvc.problemdetails.enabled=true)
curl -X POST http://localhost:8080/users -d '{"name": ""}'
# HTTP/1.1 400
# Content-Type: application/problem+json
# {
#   "type": "about:blank",
#   "title": "Bad Request",
#   "status": 400,
#   "detail": "Invalid request content.",
#   "instance": "/users"
# }
```

### 실험 2: 커스텀 ProblemDetail

```bash
curl http://localhost:8080/users/999
# HTTP/1.1 404
# Content-Type: application/problem+json
# {
#   "type": "https://api.example.com/errors/user-not-found",
#   "title": "사용자를 찾을 수 없음",
#   "status": 404,
#   "detail": "ID 999인 사용자가 없습니다.",
#   "instance": "/users/999",
#   "userId": 999,
#   "timestamp": "2024-03-10T15:30:00Z",
#   "traceId": "abc123"
# }
```

---

## 🌐 HTTP 레벨 분석

```
@Valid 실패 + spring.mvc.problemdetails.enabled=true:

MethodArgumentNotValidException 발생
  → ex.getBody() = ProblemDetail{status=400, detail="Invalid request content."}

ResponseEntityExceptionHandler.handleExceptionInternal():
  body == null check → body instanceof ErrorResponse
  → body = ex.getBody() (ProblemDetail 자동 사용)
  instance 자동 설정: pd.setInstance("/api/users")
  → ResponseEntity.status(400).body(problemDetail)

MappingJackson2HttpMessageConverter:
  ProblemDetail 직렬화
  Content-Type 협상: application/problem+json 선택

HTTP/1.1 400 Bad Request
Content-Type: application/problem+json
{
  "type": "about:blank",
  "title": "Bad Request",
  "status": 400,
  "detail": "Invalid request content.",
  "instance": "/api/users"
}
```

---

## 📌 핵심 정리

```
RFC 7807 필수/선택 필드
  type (URI, 기본 "about:blank"), title, status
  detail (선택), instance (선택), 커스텀 확장 필드

ProblemDetail 클래스 (Spring 6.x)
  setProperty(name, value)로 확장 필드 추가
  forStatus(), forStatusAndDetail() 팩토리 메서드
  title: status 코드에서 자동 유추 가능

ErrorResponse 인터페이스
  Spring MVC 표준 예외들이 구현
  getBody()로 ProblemDetail 접근
  → handleExceptionInternal()에서 자동 사용

spring.mvc.problemdetails.enabled=true
  ProblemDetailsExceptionHandler 자동 등록
  표준 예외 → ProblemDetail 자동 응답
  Content-Type: application/problem+json 자동 설정

커스텀 확장
  pd.setProperty()로 필드 추가
  handleMethodArgumentNotValid() 오버라이드로 errors 필드 추가
```

---

## 🤔 생각해볼 문제

**Q1.** `ProblemDetail`의 `type` 필드에 `about:blank`를 사용하면 클라이언트에서 어떤 의미로 해석되어야 하는가? 실무에서 `type`을 설정해야 하는 이유는?

**Q2.** `Accept: application/json`을 보낸 클라이언트에게 `ProblemDetail`을 반환하면 Content-Type은 `application/json`인가 `application/problem+json`인가?

**Q3.** `ProblemDetail`을 `@ResponseBody`로 반환하는 `@ExceptionHandler`에서 `@ResponseStatus(HttpStatus.NOT_FOUND)`를 붙이면 `ProblemDetail.status`와 HTTP 응답 상태 코드 중 어느 것이 실제 응답 코드가 되는가?

> 💡 **해설**
>
> **Q1.** `about:blank`는 "추가 정보 없음, HTTP 상태 코드 자체가 설명"을 의미합니다. RFC 7807에서 허용하지만, 클라이언트가 오류 종류를 프로그래밍적으로 구분할 수 없게 됩니다. 실무에서는 `https://api.example.com/errors/user-not-found` 같이 팀 내 오류 카탈로그 URL을 사용하면, 클라이언트가 type URI를 키로 사용해 오류별 다른 처리 로직을 구현할 수 있고, 개발 문서와도 연결됩니다.
>
> **Q2.** Content Negotiation에 따라 `application/json`이 됩니다. `ProblemDetail`은 Jackson으로 직렬화되며, Content-Type은 Accept 헤더와 Converter의 지원 타입 교집합으로 결정됩니다. `Accept: application/json`이면 `application/json`이 선택됩니다. `application/problem+json`은 `application/json`의 하위 타입(subtype)이므로 `application/json`을 받는 클라이언트도 파싱할 수 있습니다.
>
> **Q3.** `@ResponseBody`로 `ProblemDetail`을 반환할 때 `@ResponseStatus`가 붙어 있으면 `@ResponseStatus`의 HTTP 상태 코드가 실제 응답 코드가 됩니다. `ProblemDetail.status`는 단순히 JSON 본문의 필드일 뿐, HTTP 응답 코드와 자동으로 동기화되지 않습니다. 일관성을 위해 `ProblemDetail.status`와 HTTP 응답 코드를 일치시키거나, `ResponseEntity`로 반환해 명시적으로 코드를 설정하는 것이 좋습니다.

---

<div align="center">

**[⬅️ 이전: ResponseEntityExceptionHandler](./04-response-entity-exception-handler.md)** | **[홈으로 🏠](../README.md)** | **[다음: 예외 처리 우선순위와 상속 ➡️](./06-exception-priority-inheritance.md)**

</div>
