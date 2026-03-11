# ResponseEntityExceptionHandler 내부 구조 — Spring MVC 표준 예외 일괄 처리

---

## 🎯 핵심 질문

- `ResponseEntityExceptionHandler`가 처리하는 표준 예외 목록과 각각의 기본 응답 코드는?
- `handleExceptionInternal()`이 확장 포인트로 동작하는 방법은?
- `DefaultHandlerExceptionResolver`와 `ResponseEntityExceptionHandler`의 역할 분담은?
- 커스텀 오류 응답 형식을 전역으로 적용할 때 `handleExceptionInternal()`을 오버라이드하는 패턴은?
- Spring 6.x에서 `ErrorResponse` 인터페이스가 추가된 이유와 동작 방식은?

---

## 🔍 왜 이 메커니즘이 존재하는가

```
Spring MVC 표준 예외들:
  MethodArgumentNotValidException     → 400 (@Valid 실패)
  HttpMessageNotReadableException     → 400 (JSON 파싱 오류)
  HttpRequestMethodNotSupportedException → 405
  HttpMediaTypeNotSupportedException  → 415
  HttpMediaTypeNotAcceptableException → 406
  ... 15가지 이상

두 가지 처리 방식:
  DefaultHandlerExceptionResolver:
    response.sendError(status) 만 설정
    → 응답 본문 없음 (또는 서블릿 컨테이너 기본 오류 페이지)
    → REST API에서 상세한 오류 메시지 전달 불가

  ResponseEntityExceptionHandler:
    ResponseEntity로 상태 코드 + 커스텀 본문 반환
    → @ControllerAdvice와 함께 사용
    → 오류 본문 형식을 완전히 제어 가능
```

---

## 😱 흔한 오해 또는 실수

### Before: ResponseEntityExceptionHandler를 상속하면 자동으로 모든 표준 예외가 처리된다

```java
// ❌ 잘못된 이해
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {
    // 아무것도 오버라이드 안 해도 표준 예외 처리됨?
}

// ✅ 실제:
// 상속만 해도 표준 예외는 처리됨 (기본 구현이 있음)
// 단, 기본 구현의 body = null → 응답 본문 없음
// → REST API에서 유용하게 사용하려면 handleExceptionInternal() 오버라이드 필요

// 기본 동작:
// MethodArgumentNotValidException → 400, body=null
// 커스텀 동작 (오버라이드 후):
// MethodArgumentNotValidException → 400, body={"errors": [...]}
```

### Before: ResponseEntityExceptionHandler와 DefaultHandlerExceptionResolver가 같은 예외를 중복 처리한다

```
❌ 잘못된 이해:
  "두 개가 동시에 작동해서 응답이 두 번 쓰인다"

✅ 실제:
  ExceptionHandlerExceptionResolver (Order=0):
    → ResponseEntityExceptionHandler의 @ExceptionHandler 실행
    → ResponseEntity 반환 → 응답 완료 → mavContainer.setRequestHandled(true)
    → 이후 Resolver는 실행 안 됨

  DefaultHandlerExceptionResolver (Order=2):
    → 위에서 처리됐으므로 도달하지 않음

  따라서 ResponseEntityExceptionHandler가 있으면
  DefaultHandlerExceptionResolver의 동일 예외 처리는 사실상 불필요
```

---

## ✨ 올바른 이해와 사용

### After: ResponseEntityExceptionHandler가 처리하는 예외 전체 목록 (Spring 6.x 기준)

```java
@ExceptionHandler({
    // 바인딩/검증 관련
    MethodArgumentNotValidException.class,      // @Valid 실패 → 400
    BindException.class,                        // @ModelAttribute @Valid 실패 → 400
    HandlerMethodValidationException.class,     // 메서드 레벨 검증 실패 → 400/422

    // 요청 파싱
    HttpMessageNotReadableException.class,      // JSON 파싱 오류 → 400
    HttpMessageConversionException.class,       // 메시지 변환 오류 → 400

    // 요청 파라미터
    MissingServletRequestParameterException.class, // @RequestParam 누락 → 400
    MissingServletRequestPartException.class,      // @RequestPart 누락 → 400
    MissingPathVariableException.class,            // @PathVariable 누락 → 500
    UnsatisfiedServletRequestParameterException.class, // params 조건 미충족 → 400

    // HTTP 메서드/미디어 타입
    HttpRequestMethodNotSupportedException.class,  // 미지원 HTTP 메서드 → 405
    HttpMediaTypeNotSupportedException.class,       // 미지원 Content-Type → 415
    HttpMediaTypeNotAcceptableException.class,      // Accept 미지원 → 406

    // 타입 변환
    MethodArgumentTypeMismatchException.class,     // 파라미터 타입 불일치 → 400
    TypeMismatchException.class,                   // 타입 변환 실패 → 400

    // 기타
    NoHandlerFoundException.class,                 // 핸들러 없음 → 404
    NoResourceFoundException.class,                // 리소스 없음 → 404 (Spring 6.2+)
    AsyncRequestTimeoutException.class,            // 비동기 타임아웃 → 503
    ErrorResponseException.class,                  // ErrorResponse 구현 예외 → 가변
})
public final ResponseEntity<Object> handleException(Exception ex, WebRequest request) throws Exception {
    // 예외 타입별로 handleXxx() 메서드로 위임
}
```

### 핵심 구조

```java
// ResponseEntityExceptionHandler 핵심 구조
public abstract class ResponseEntityExceptionHandler implements MessageSourceAware {

    // ① 모든 표준 예외를 하나의 @ExceptionHandler에서 받음
    @ExceptionHandler({ ... 15가지 이상 ... })
    public final ResponseEntity<Object> handleException(Exception ex, WebRequest request) throws Exception {

        HttpHeaders headers = new HttpHeaders();
        if (ex instanceof MethodArgumentNotValidException manve) {
            return handleMethodArgumentNotValid(manve, headers, HttpStatus.BAD_REQUEST, request);
        } else if (ex instanceof HttpMessageNotReadableException hmnre) {
            return handleHttpMessageNotReadable(hmnre, headers, HttpStatus.BAD_REQUEST, request);
        } // ... else if 체인

        // 처리 대상 아닌 예외 → 그냥 throw
        throw ex;
    }

    // ② 각 예외별 처리 메서드 (오버라이드 가능)
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex, HttpHeaders headers,
            HttpStatusCode status, WebRequest request) {
        return handleExceptionInternal(ex, null, headers, status, request);
        // body = null (기본) → 오버라이드로 커스텀 본문 가능
    }

    // ③ 최종 공통 처리 — 핵심 확장 포인트
    protected ResponseEntity<Object> handleExceptionInternal(
            Exception ex, @Nullable Object body, HttpHeaders headers,
            HttpStatusCode statusCode, WebRequest request) {

        if (request instanceof ServletWebRequest servletWebRequest) {
            HttpServletResponse response = servletWebRequest.getResponse();
            if (response != null && response.isCommitted()) {
                logger.warn("Response already committed. Ignoring: " + ex);
                return null;
            }
        }

        if (body == null && ex instanceof ErrorResponse errorResponse) {
            body = errorResponse.getBody();
            // Spring 6.x: ProblemDetail 자동 사용
        }

        return ResponseEntity.status(statusCode).headers(headers).body(body);
    }
}
```

---

## 🔬 내부 동작 원리

### 1. handleExceptionInternal() 오버라이드 패턴

```java
// 커스텀 오류 응답 형식 전역 적용
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    // 커스텀 오류 응답 구조
    record ErrorResponse(String code, String message, List<FieldError> fieldErrors) {}
    record FieldError(String field, String rejectedValue, String reason) {}

    // ③ 핵심 확장 포인트 오버라이드
    @Override
    protected ResponseEntity<Object> handleExceptionInternal(
            Exception ex, @Nullable Object body, HttpHeaders headers,
            HttpStatusCode status, WebRequest request) {

        // 모든 표준 예외가 거치는 공통 처리
        ErrorResponse errorBody = buildErrorBody(ex, status);
        return super.handleExceptionInternal(ex, errorBody, headers, status, request);
    }

    private ErrorResponse buildErrorBody(Exception ex, HttpStatusCode status) {
        if (ex instanceof MethodArgumentNotValidException manve) {
            List<FieldError> fieldErrors = manve.getBindingResult().getFieldErrors().stream()
                .map(fe -> new FieldError(fe.getField(),
                    String.valueOf(fe.getRejectedValue()),
                    fe.getDefaultMessage()))
                .toList();
            return new ErrorResponse("VALIDATION_FAILED", "입력값 검증 실패", fieldErrors);
        }
        if (ex instanceof HttpMessageNotReadableException) {
            return new ErrorResponse("INVALID_JSON", "JSON 파싱 오류", null);
        }
        return new ErrorResponse("ERROR_" + status.value(), ex.getMessage(), null);
    }

    // 개별 예외 오버라이드 (특정 예외만 다른 처리 필요 시)
    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex, HttpHeaders headers,
            HttpStatusCode status, WebRequest request) {
        // 여기서 처리하면 handleExceptionInternal()보다 먼저 실행
        // → 특정 예외에만 다른 구조 적용 가능
        return super.handleMethodArgumentNotValid(ex, headers, status, request);
    }

    // 표준 예외 외 커스텀 예외 추가
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(UserNotFoundException ex) {
        return ResponseEntity.status(404)
            .body(new ErrorResponse("USER_NOT_FOUND", ex.getMessage(), null));
    }
}
```

### 2. Spring 6.x ErrorResponse 인터페이스

```java
// ErrorResponse 인터페이스 (Spring 6.x 추가)
public interface ErrorResponse {
    HttpStatusCode getStatusCode();
    HttpHeaders getHeaders();
    ProblemDetail getBody();  // RFC 7807 형식

    default String getTypeMessageCode() { ... }
    default String getTitleMessageCode() { ... }
    default String getDetailMessageCode() { ... }
}

// Spring MVC 표준 예외들이 ErrorResponse 구현:
// MethodArgumentNotValidException implements ErrorResponse
// HttpRequestMethodNotSupportedException implements ErrorResponse
// → ex.getBody()로 ProblemDetail 직접 접근 가능

// handleExceptionInternal()에서의 처리:
if (body == null && ex instanceof ErrorResponse errorResponse) {
    body = errorResponse.getBody();
    // → ProblemDetail이 자동으로 응답 본문이 됨 (Spring 6.x 기본)
}
```

### 3. DefaultHandlerExceptionResolver와의 차이

```java
// DefaultHandlerExceptionResolver.handleMethodArgumentNotValidException():
protected ModelAndView handleMethodArgumentNotValidException(...) throws IOException {
    response.sendError(HttpServletResponse.SC_BAD_REQUEST);
    return new ModelAndView();
    // → 응답 본문 없음, 상태 코드만 설정
}

// ResponseEntityExceptionHandler.handleMethodArgumentNotValid():
protected ResponseEntity<Object> handleMethodArgumentNotValid(...) {
    return handleExceptionInternal(ex, null, headers, HttpStatus.BAD_REQUEST, request);
    // → ResponseEntity 반환 → 오버라이드로 본문 커스터마이징 가능
    // → JSON 오류 응답, 필드별 오류 메시지 등
}

// 두 방식 선택:
// sendError() → 서블릿 컨테이너 오류 처리 → BasicErrorController(/error)
// ResponseEntity → Spring MVC 내에서 응답 완성
// REST API: ResponseEntityExceptionHandler 방식 권장
```

---

## 💻 실험으로 확인하기

### 실험 1: 기본 동작 vs 오버라이드 후 비교

```bash
# ResponseEntityExceptionHandler 상속만 (오버라이드 없음)
# @Valid 실패 요청
curl -X POST http://localhost:8080/users \
     -H "Content-Type: application/json" \
     -d '{"name": ""}'
# HTTP/1.1 400 Bad Request
# (본문 없음 또는 null body)

# handleExceptionInternal() 오버라이드 후
# HTTP/1.1 400 Bad Request
# Content-Type: application/json
# {"code":"VALIDATION_FAILED","message":"입력값 검증 실패",
#  "fieldErrors":[{"field":"name","rejectedValue":"","reason":"이름은 필수입니다"}]}
```

### 실험 2: 처리 흐름 로그

```yaml
logging:
  level:
    org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver: DEBUG
```

```
DEBUG ExceptionHandlerExceptionResolver - Resolving exception from handler [...]:
  MethodArgumentNotValidException
DEBUG ExceptionHandlerExceptionResolver - Using @ExceptionHandler
  GlobalExceptionHandler#handleException(Exception, WebRequest)
```

---

## 🌐 HTTP 레벨 분석

```
POST /users {"name": ""} 요청 → @Valid 실패

① ExceptionHandlerExceptionResolver:
   GlobalExceptionHandler.handleException(MethodArgumentNotValidException, request)
   → handleMethodArgumentNotValid()
   → handleExceptionInternal()  ← 오버라이드된 구현
   → ErrorResponse{code:"VALIDATION_FAILED", ...}
   → ResponseEntity.status(400).body(errorResponse)

② ReturnValueHandler (ResponseEntity):
   HttpEntityMethodProcessor.handleReturnValue()
   → Jackson 직렬화
   → response.setStatus(400)
   → 응답 본문 쓰기

HTTP/1.1 400 Bad Request
Content-Type: application/json
{
  "code": "VALIDATION_FAILED",
  "message": "입력값 검증 실패",
  "fieldErrors": [
    {"field": "name", "rejectedValue": "", "reason": "이름은 필수입니다"}
  ]
}
```

---

## 📌 핵심 정리

```
ResponseEntityExceptionHandler 역할
  Spring MVC 표준 예외 15종+ 처리
  @ControllerAdvice와 함께 사용
  DefaultHandlerExceptionResolver 대신 ResponseEntity 기반 처리

확장 포인트 계층
  개별 handleXxx() 오버라이드 → 특정 예외 맞춤 처리
  handleExceptionInternal() 오버라이드 → 모든 표준 예외 공통 응답 형식

DefaultHandlerExceptionResolver 차이
  sendError() 방식 → 응답 본문 없음
  ResponseEntityExceptionHandler → ResponseEntity → 본문 커스터마이징

Spring 6.x 변화
  표준 예외들이 ErrorResponse 구현
  → getBody()로 ProblemDetail 접근
  → handleExceptionInternal()에서 자동으로 ProblemDetail을 body로 사용

사용 패턴
  상속 + handleExceptionInternal() 오버라이드 → 전역 오류 형식 통일
  + 개별 @ExceptionHandler → 커스텀 예외 추가 처리
```

---

## 🤔 생각해볼 문제

**Q1.** `ResponseEntityExceptionHandler`를 상속한 `@ControllerAdvice`와 별도의 `@ControllerAdvice`에 동일 표준 예외의 `@ExceptionHandler`가 있으면 어느 것이 선택되는가?

**Q2.** `handleExceptionInternal()`에서 반환한 `ResponseEntity`의 body가 `null`이면 실제 HTTP 응답은 어떻게 되는가?

**Q3.** `@ControllerAdvice`에 `@ExceptionHandler(Exception.class)`를 선언하면 `ResponseEntityExceptionHandler`의 `handleException()`보다 우선하는가?

> 💡 **해설**
>
> **Q1.** `@Order` 값이 낮은 쪽이 우선합니다. `ResponseEntityExceptionHandler`를 상속한 클래스에 별도 `@Order`가 없고 다른 `@ControllerAdvice`에도 없다면 Bean 정의 순서에 따릅니다. 명시적으로 분리하려면 `@Order`를 사용해야 합니다.
>
> **Q2.** `ResponseEntity.status(400).body(null)` 형태로 반환되면 `HttpEntityMethodProcessor`가 `body==null`임을 확인하고 `writeWithMessageConverters()`에서 본문을 쓰지 않습니다. `Content-Length: 0`으로 빈 응답이 됩니다. Spring 6.x에서는 `ErrorResponse` 구현체이면 `body==null`이어도 자동으로 `ProblemDetail`로 채워지므로 본문이 있을 수 있습니다.
>
> **Q3.** 같은 `@ControllerAdvice` 내에 있다면 `@ExceptionHandler(Exception.class)`는 `handleException()` 내부의 `else if` 체인보다 범위가 넓어 문제가 됩니다. 단 `handleException()`는 `final`이 아닌 `@ExceptionHandler`로 선언되어 있고, 그 안에 특정 타입 체크가 있습니다. 별도 `@ControllerAdvice`라면 `@Order`로 우선순위가 결정됩니다. 동일 클래스에서 `handleException()`과 `@ExceptionHandler(Exception.class)` 충돌은 `ambiguous @ExceptionHandler` 오류를 유발합니다.

---

<div align="center">

**[⬅️ 이전: @ControllerAdvice 범위](./03-controller-advice-scope.md)** | **[홈으로 🏠](../README.md)** | **[다음: RFC 7807 Problem Details ➡️](./05-problem-details-response.md)**

</div>
