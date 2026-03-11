# 예외 처리 우선순위와 상속 — 무엇이 선택되는가

---

## 🎯 핵심 질문

- 로컬 `@ExceptionHandler` vs `@ControllerAdvice` 우선순위의 결정 코드 경로는?
- 예외 상속 계층에서 "가장 구체적인 핸들러"는 어떻게 결정되는가?
- `@ExceptionHandler(value = {ExA.class, ExB.class})` 배열 선언의 동작 방식과 주의점은?
- 여러 `@ControllerAdvice`가 동일 예외를 처리할 때의 우선순위 결정 방식은?
- `cause` 예외(`ex.getCause()`)는 어떻게 처리되는가?
- 예외 처리 전략을 설계할 때 상속 계층을 어떻게 활용해야 하는가?

---

## 🔍 왜 이 메커니즘이 존재하는가

```
복잡한 예외 계층과 여러 Advice가 공존할 때:

Exception
└── RuntimeException
     ├── BusinessException        → 도메인 오류
     │    ├── UserException
     │    │    ├── UserNotFoundException
     │    │    └── DuplicateUserException
     │    └── OrderException
     │         └── OrderNotFoundException
     └── SystemException          → 시스템 오류

@ExceptionHandler 선언 위치:
  UserController.handleUserException()  ← 로컬
  GlobalBusinessExceptionHandler        ← @ControllerAdvice (Order=1)
  GlobalExceptionHandler                ← @ControllerAdvice (Order=2)

"어느 핸들러가 선택되는가?" 규칙이 명확해야
→ 예측 가능한 예외 처리 전략 수립 가능
```

---

## 😱 흔한 오해 또는 실수

### Before: 더 구체적인 @ControllerAdvice가 더 범용적인 로컬 핸들러보다 우선이다

```java
@RestController
public class UserController {
    // 로컬, 범용
    @ExceptionHandler(Exception.class)
    public ResponseEntity<String> handleAll(Exception ex) { return ...; }
}

@RestControllerAdvice
public class SpecificAdvice {
    // 전역, 구체적
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<String> handleUserNotFound(...) { return ...; }
}

// ❌ 잘못된 예상: SpecificAdvice가 더 구체적이므로 선택
// ✅ 실제: 로컬 @ExceptionHandler가 항상 전역보다 우선
//    UserController에서 UserNotFoundException 발생
//    → 로컬 handleAll(Exception.class) 선택 (depth=N, 범용이지만 로컬)
//    → SpecificAdvice.handleUserNotFound() 무시
```

---

## ✨ 올바른 이해와 사용

### After: 우선순위 결정 전체 규칙

```
1순위: 로컬 @ExceptionHandler (발생한 Controller 내부)
  → 예외 타입 상속 계층에서 가장 낮은 depth 선택
  → 로컬에서 찾으면 전역 탐색 자체 안 함

2순위: @ControllerAdvice (전역, Order 낮은 값 우선)
  → isApplicableToBeanType() = true인 Advice만 대상
  → 각 Advice 내에서: 예외 타입 상속 계층 depth 비교
  → Order가 같은 여러 Advice에서 같은 depth면: Advice 등록 순서

3순위: ResponseStatusExceptionResolver
  → @ResponseStatus 어노테이션, ResponseStatusException

4순위: DefaultHandlerExceptionResolver
  → Spring MVC 표준 예외 15종

5순위: 서블릿 컨테이너 (BasicErrorController)
  → 위 모두 처리 못한 경우
```

---

## 🔬 내부 동작 원리

### 1. 전체 선택 흐름 소스 레벨 추적

```java
// ExceptionHandlerExceptionResolver.doResolveHandlerMethodException()
//   → getExceptionHandlerMethod()

@Nullable
private ServletInvocableHandlerMethod getExceptionHandlerMethod(
        @Nullable HandlerMethod handlerMethod, Exception exception) {

    Class<?> handlerType = null;

    // ━━━ 1순위: 로컬 탐색 ━━━
    if (handlerMethod != null) {
        handlerType = handlerMethod.getBeanType();
        ExceptionHandlerMethodResolver resolver =
            this.exceptionHandlerCache.computeIfAbsent(
                handlerType, ExceptionHandlerMethodResolver::new);

        Method method = resolver.resolveMethodByThrowable(exception);
        if (method != null) {
            // 로컬에서 찾음 → 즉시 반환, 전역 탐색 없음
            return new ServletInvocableHandlerMethod(
                handlerMethod.getBean(), method, this.applicationContext);
        }
        // 여기 도달 = 로컬에 적합한 @ExceptionHandler 없음
    }

    // ━━━ 2순위: 전역 탐색 ━━━
    for (Map.Entry<ControllerAdviceBean, ExceptionHandlerMethodResolver> entry :
            this.exceptionHandlerAdviceCache.entrySet()) {
        ControllerAdviceBean advice = entry.getKey();

        // 이 Advice가 현재 Controller에 적용되는지 필터 체크
        if (advice.isApplicableToBeanType(handlerType)) {
            ExceptionHandlerMethodResolver resolver = entry.getValue();
            Method method = resolver.resolveMethodByThrowable(exception);
            if (method != null) {
                // 전역에서 찾음 → 반환 (더 이상 탐색 안 함)
                return new ServletInvocableHandlerMethod(
                    advice.resolveBean(), method, this.applicationContext);
            }
            // 이 Advice에 없으면 다음 Advice로
        }
    }
    return null;
}
```

### 2. resolveMethodByThrowable() — cause 체인 탐색

```java
// ExceptionHandlerMethodResolver.resolveMethodByThrowable()
@Nullable
public Method resolveMethodByThrowable(Throwable exception) {
    // 1. 발생 예외 직접 탐색
    Method method = resolveMethodByExceptionType(exception.getClass());
    if (method != null) return method;

    // 2. cause 예외 탐색 (래핑된 예외 처리)
    Throwable cause = exception.getCause();
    if (cause != null) {
        method = resolveMethodByExceptionType(cause.getClass());
        if (method != null) return method;
    }

    return null;
}

// 예시: InvocationTargetException이 래핑한 경우
// 실제 예외: InvocationTargetException(cause=UserNotFoundException)
// → resolveMethodByExceptionType(InvocationTargetException) = null
// → resolveMethodByExceptionType(UserNotFoundException) = handleNotFound() ✅
```

### 3. @ExceptionHandler 배열 선언 전략

```java
// 패턴 1: 배열로 여러 예외 묶기
@ExceptionHandler({UserNotFoundException.class, OrderNotFoundException.class})
public ResponseEntity<ErrorResponse> handleNotFound(RuntimeException ex) {
    // 두 예외 모두 이 핸들러로 처리
    // 파라미터 타입은 공통 부모(RuntimeException) 또는 예외 없이
    String code = ex instanceof UserNotFoundException ? "USER_NOT_FOUND" : "ORDER_NOT_FOUND";
    return ResponseEntity.status(404).body(new ErrorResponse(code, ex.getMessage()));
}

// 패턴 2: 개별 선언 (각기 다른 처리 필요 시)
@ExceptionHandler(UserNotFoundException.class)
public ResponseEntity<ErrorResponse> handleUserNotFound(UserNotFoundException ex) { ... }

@ExceptionHandler(OrderNotFoundException.class)
public ResponseEntity<ErrorResponse> handleOrderNotFound(OrderNotFoundException ex) { ... }

// 패턴 3: 상위 예외 하나로 처리
@ExceptionHandler(NotFoundException.class)  // UserNFE, OrderNFE의 공통 부모
public ResponseEntity<ErrorResponse> handleNotFound(NotFoundException ex) {
    return ResponseEntity.status(404).body(new ErrorResponse("NOT_FOUND", ex.getMessage()));
}
// UserNotFoundException → depth=1 (NotFoundException 자식)
// OrderNotFoundException → depth=1 (같은 depth)
// → 같은 핸들러 선택
```

### 4. 전략 설계 — 계층 기반 예외 처리 아키텍처

```java
// 권장 설계 패턴

// 1. 도메인 예외 계층 정의
public abstract class BusinessException extends RuntimeException {
    private final String errorCode;
    public BusinessException(String errorCode, String message) { ... }
}
public class UserNotFoundException extends BusinessException {
    public UserNotFoundException(Long id) {
        super("USER_NOT_FOUND", "ID " + id + "인 사용자 없음");
    }
}

// 2. @ResponseStatus 대신 ErrorCode 열거형 활용
public enum ErrorCode {
    USER_NOT_FOUND(404), DUPLICATE_USER(409), ORDER_NOT_FOUND(404);
    final int httpStatus;
}

// 3. @ControllerAdvice 계층 분리

// 가장 구체적 (Order=1): 도메인 예외
@RestControllerAdvice @Order(1)
public class BusinessExceptionHandler {
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ProblemDetail> handleUserNotFound(UserNotFoundException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        pd.setType(URI.create("https://api.example.com/errors/user-not-found"));
        return ResponseEntity.status(404).body(pd);
    }

    // 나머지 BusinessException 폴백
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ProblemDetail> handleBusiness(BusinessException ex) {
        HttpStatus status = HttpStatus.resolve(ex.getErrorCode().getHttpStatus());
        return ResponseEntity.status(status)
            .body(ProblemDetail.forStatusAndDetail(status, ex.getMessage()));
    }
}

// 중간 (Order=2): Spring MVC 표준 예외
@RestControllerAdvice @Order(2)
public class SpringMvcExceptionHandler extends ResponseEntityExceptionHandler {
    @Override
    protected ResponseEntity<Object> handleExceptionInternal(...) { ... }
}

// 최후 폴백 (Order=3): 미처리 예외
@RestControllerAdvice @Order(3)
public class FallbackExceptionHandler {
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ProblemDetail> handleAll(Exception ex) {
        // 로그 + 500
        log.error("Unhandled exception", ex);
        return ResponseEntity.internalServerError()
            .body(ProblemDetail.forStatus(500));
    }
}
```

---

## 💻 실험으로 확인하기

### 실험: 우선순위 매트릭스

```java
// 시나리오별 선택 핸들러 확인:

// Case 1: 로컬 범용 vs 전역 구체적
// 발생: UserNotFoundException in UserController
// 로컬: @ExceptionHandler(Exception.class)
// 전역: @ExceptionHandler(UserNotFoundException.class)
// → 로컬 선택 (로컬 우선)

// Case 2: 로컬 없음, 전역 여러 개
// 발생: UserNotFoundException
// 전역 A (Order=1): @ExceptionHandler(RuntimeException.class) depth=2
// 전역 B (Order=2): @ExceptionHandler(UserNotFoundException.class) depth=0
// → 전역 A 선택 (Order=1이 먼저 탐색, 매칭됨)
// ← 주의! 더 구체적인 B보다 Order가 낮은 A가 우선

// Case 3: 같은 Order, 다른 depth
// 전역 A (Order=1): @ExceptionHandler(UserNotFoundException.class) depth=0
// 전역 B (Order=1): @ExceptionHandler(RuntimeException.class) depth=2
// → 전역 A 선택 (같은 Order → 등록 순서 → A가 먼저면 A 선택)
// ← A에서 UserNotFoundException depth=0 매칭, B는 탐색 안 함
```

---

## 🌐 HTTP 레벨 분석

```
우선순위 흐름 요약:

UserController → UserNotFoundException 발생

① 로컬 탐색 (UserController):
   @ExceptionHandler 없음 → null

② 전역 탐색 (exceptionHandlerAdviceCache, Order 순):
   BusinessExceptionHandler (Order=1):
     isApplicableToBeanType(UserController) = true
     resolveMethodByThrowable(UserNotFoundException):
       handleUserNotFound(UserNFE) → depth=0 ✅
   → 선택

③ SpringMvcExceptionHandler, FallbackExceptionHandler는 탐색 안 함

HTTP/1.1 404 Not Found
Content-Type: application/problem+json
{
  "type": "https://api.example.com/errors/user-not-found",
  "title": "Not Found",
  "status": 404,
  "detail": "ID 42인 사용자 없음",
  "instance": "/users/42"
}
```

---

## 📌 핵심 정리

```
우선순위 요약 (높은 순)
  1. 로컬 @ExceptionHandler (Controller 내부)
     → 내부에서 depth 비교 (구체적인 것 우선)
  2. @ControllerAdvice (Order 낮은 값 우선)
     → 각 Advice 내에서 depth 비교
     → Order가 같으면 Advice 등록 순서
  3. ResponseStatusExceptionResolver (@ResponseStatus)
  4. DefaultHandlerExceptionResolver (MVC 표준 예외)
  5. 서블릿 컨테이너

중요: Order vs 구체성
  Order가 낮은 Advice의 범용 핸들러 > Order가 높은 Advice의 구체적 핸들러
  → Advice 설계 시 Order와 처리 범위를 일관되게 관리

cause 탐색
  resolveMethodByThrowable()은 cause까지 탐색
  → 래핑된 예외도 원인 예외로 처리 가능

배열 선언
  @ExceptionHandler({ExA.class, ExB.class}) → 같은 핸들러로 처리
  → 파라미터 타입은 공통 부모 또는 Exception 사용

권장 전략
  도메인 예외 계층 + @Order로 처리 우선순위 명시
  BusinessException → Spring MVC 표준 → 폴백 순서로 Advice 분리
```

---

## 🤔 생각해볼 문제

**Q1.** `@ControllerAdvice(Order=1)`에서 `@ExceptionHandler(RuntimeException.class)`를 선언하고, `@ControllerAdvice(Order=2)`에서 `@ExceptionHandler(UserNotFoundException.class)`를 선언했을 때, `UserNotFoundException`이 발생하면 어느 핸들러가 선택되는가? 의도적으로 Order=2 쪽을 선택되게 하려면 어떻게 해야 하는가?

**Q2.** `resolveMethodByThrowable()`이 `cause`까지 탐색하는 것은 어떤 실제 시나리오에서 유용한가?

**Q3.** 로컬 `@ExceptionHandler`와 `@ControllerAdvice`의 우선순위 관계가 "구체성"이 아닌 "위치(로컬/전역)"로 결정되는 설계 이유는 무엇인가?

> 💡 **해설**
>
> **Q1.** Order=1의 `RuntimeException` 핸들러가 선택됩니다. `exceptionHandlerAdviceCache`는 Order 순으로 정렬된 LinkedHashMap이므로 Order=1 Advice가 먼저 탐색되고, `UserNotFoundException`이 `RuntimeException`을 상속하므로 `isAssignableFrom()=true` → 매칭됩니다. Order=2 Advice는 탐색 자체를 안 합니다. 의도적으로 Order=2를 선택되게 하려면: Order=1 Advice에서 `UserNotFoundException`을 명시적으로 제외하거나(`@ExceptionHandler({BusinessException.class})`로 범위 축소), 또는 `UserNotFoundException`에 대한 처리를 Order=1 Advice로 이동시켜야 합니다.
>
> **Q2.** JPA나 트랜잭션 처리에서 예외가 래핑되는 경우입니다. 예를 들어 `TransactionSystemException(cause=ConstraintViolationException)`처럼 Spring이 예외를 래핑할 때, `@ExceptionHandler(ConstraintViolationException.class)`를 선언해도 직접 매칭이 안 됩니다. `cause` 탐색 덕분에 `TransactionSystemException`이 발생해도 내부 `ConstraintViolationException`을 처리하는 핸들러가 선택됩니다.
>
> **Q3.** "위치 우선" 설계는 Controller 작성자가 자신의 Controller에서 발생하는 예외에 대한 완전한 제어권을 갖도록 보장합니다. 만약 구체성이 우선이라면, 전역 Advice 작성자가 특정 예외를 더 구체적으로 선언함으로써 의도치 않게 Controller의 로컬 처리를 override할 수 있습니다. 로컬 우선 원칙은 "내 Controller 안의 예외는 내가 처리한다"는 캡슐화를 보장하며, 전역 Advice는 로컬에서 처리 못한 경우의 폴백 역할을 합니다.

---

<div align="center">

**[⬅️ 이전: RFC 7807 Problem Details](./05-problem-details-response.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 6 — Interceptor & Filter ➡️](../interceptor-filter/01-handler-interceptor-lifecycle.md)**

</div>
