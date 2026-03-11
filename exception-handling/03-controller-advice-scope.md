# @ControllerAdvice 적용 범위 — 전역 핸들러의 대상을 제한하는 방법

---

## 🎯 핵심 질문

- `@ControllerAdvice`의 네 가지 필터 속성은 각각 어떻게 동작하는가?
- `ExceptionHandlerExceptionResolver`는 언제 `@ControllerAdvice` Bean을 로드하고 어떻게 캐시하는가?
- 여러 `@ControllerAdvice`가 있을 때 적용 순서는 어떻게 결정되는가?
- 로컬 `@ExceptionHandler` vs `@ControllerAdvice` 우선순위는 코드 어느 지점에서 결정되는가?
- `@RestControllerAdvice`는 `@ControllerAdvice`와 정확히 무엇이 다른가?

---

## 🔍 왜 이 메커니즘이 존재하는가

```
대규모 애플리케이션에서:
  /api/**        → REST API 컨트롤러 → JSON 오류 응답 필요
  /admin/**      → 관리자 컨트롤러  → HTML 오류 페이지 필요
  /web/**        → MVC 컨트롤러    → 뷰 기반 오류 처리 필요

하나의 전역 @ControllerAdvice가 모두 처리하면:
  → 각 영역에 맞지 않는 응답 형식
  → API 요청에 HTML 오류 페이지 반환 등

해결: @ControllerAdvice 적용 범위 제한
  → 영역별로 다른 ExceptionHandler 적용 가능
```

---

## 😱 흔한 오해 또는 실수

### Before: @ControllerAdvice는 모든 Bean에 적용된다

```java
// ❌ 잘못된 이해
// "Spring의 모든 Bean 예외를 잡는다"

// ✅ 실제:
// @ControllerAdvice는 @RequestMapping이 있는 Controller에만 적용
// → Spring MVC 요청 처리 중 발생한 예외만 처리
// → Service, Repository 레이어 예외는 Controller까지 전파된 후에 처리
// → AOP @Around 예외는 처리 못 함 (Spring MVC 외부)
// → 필터(Filter)에서 발생한 예외는 처리 못 함 (DispatcherServlet 외부)
```

### Before: @ControllerAdvice 순서는 선언 순서다

```java
// ❌ 잘못된 이해
// "먼저 선언된 @ControllerAdvice가 먼저 실행"

// ✅ 실제:
// Ordered 인터페이스 또는 @Order 어노테이션으로 결정
@ControllerAdvice
@Order(1)  // 낮은 값 = 높은 우선순위
public class FirstAdvice { ... }

@ControllerAdvice
@Order(2)
public class SecondAdvice { ... }

// @Order 없으면: Ordered.LOWEST_PRECEDENCE (Integer.MAX_VALUE)
// → 여러 Advice가 동일 @Order면 등록 순서(빈 정의 순서)에 따름
```

---

## ✨ 올바른 이해와 사용

### After: 네 가지 필터 속성 비교

```java
// 1. basePackages — 패키지 경로 문자열
@ControllerAdvice(basePackages = "com.example.api")
public class ApiExceptionHandler { ... }
// → com.example.api 및 하위 패키지의 Controller에만 적용

// 2. basePackageClasses — 타입 안전한 패키지 지정
@ControllerAdvice(basePackageClasses = ApiController.class)
public class ApiExceptionHandler { ... }
// → ApiController.class가 속한 패키지 기준
// → 문자열 오타 방지 (리팩터링 안전)

// 3. annotations — 특정 어노테이션이 붙은 Controller만
@ControllerAdvice(annotations = RestController.class)
public class RestApiExceptionHandler { ... }
// → @RestController가 붙은 컨트롤러에만 적용
// → @Controller만 있는 MVC 컨트롤러는 제외

// 4. assignableTypes — 특정 타입(또는 하위 타입) Controller만
@ControllerAdvice(assignableTypes = {BaseApiController.class})
public class ApiExceptionHandler { ... }
// → BaseApiController를 상속한 모든 컨트롤러에 적용
```

---

## 🔬 내부 동작 원리

### 1. ControllerAdviceBean — Advice 등록과 캐시

```java
// ExceptionHandlerExceptionResolver.afterPropertiesSet()
// → Spring 컨텍스트 초기화 시 @ControllerAdvice Bean 스캔

@Override
public void afterPropertiesSet() {
    // ApplicationContext에서 @ControllerAdvice Bean 수집
    initExceptionHandlerAdviceCache();
    // ...
}

private void initExceptionHandlerAdviceCache() {
    List<ControllerAdviceBean> adviceBeans =
        ControllerAdviceBean.findAnnotatedBeans(getApplicationContext());
    // @ControllerAdvice가 붙은 모든 Bean 탐색

    for (ControllerAdviceBean adviceBean : adviceBeans) {
        Class<?> beanType = adviceBean.getBeanType();
        ExceptionHandlerMethodResolver resolver =
            new ExceptionHandlerMethodResolver(beanType);
        if (resolver.hasExceptionMappings()) {
            // @ExceptionHandler 메서드가 있는 것만 캐시
            this.exceptionHandlerAdviceCache.put(adviceBean, resolver);
        }
    }
}
// exceptionHandlerAdviceCache: LinkedHashMap<ControllerAdviceBean, ExceptionHandlerMethodResolver>
// LinkedHashMap → 삽입 순서 보존 → @Order 정렬 후 삽입
```

### 2. ControllerAdviceBean.isApplicableToBeanType() — 필터 적용

```java
// ControllerAdviceBean.java
public boolean isApplicableToBeanType(@Nullable Class<?> beanType) {
    return this.beanTypePredicate.test(beanType);
}

// HandlerTypePredicate (beanTypePredicate 구현체)
public boolean test(Class<?> controllerType) {
    if (!hasSelectors()) {
        return true;  // 필터 없음 → 모든 컨트롤러에 적용
    }
    // basePackages 체크
    for (String basePackage : this.basePackages) {
        if (controllerType.getName().startsWith(basePackage)) return true;
    }
    // assignableTypes 체크
    for (Class<?> clazz : this.assignableTypes) {
        if (ClassUtils.isAssignable(clazz, controllerType)) return true;
    }
    // annotations 체크
    for (Class<? extends Annotation> annotation : this.annotations) {
        if (AnnotationUtils.findAnnotation(controllerType, annotation) != null) return true;
    }
    return false;
}
```

### 3. getExceptionHandlerMethod() — 로컬 vs 전역 탐색 순서

```java
// ExceptionHandlerExceptionResolver.getExceptionHandlerMethod()
@Nullable
private ServletInvocableHandlerMethod getExceptionHandlerMethod(
        @Nullable HandlerMethod handlerMethod, Exception exception) {

    Class<?> handlerType = null;

    // ① 로컬 탐색 (Controller 자체의 @ExceptionHandler)
    if (handlerMethod != null) {
        handlerType = handlerMethod.getBeanType();
        ExceptionHandlerMethodResolver resolver =
            this.exceptionHandlerCache.get(handlerType);
        if (resolver == null) {
            resolver = new ExceptionHandlerMethodResolver(handlerType);
            this.exceptionHandlerCache.put(handlerType, resolver);
        }
        Method method = resolver.resolveMethodByThrowable(exception);
        if (method != null) {
            // 로컬에서 찾음 → 전역 탐색 안 함, 바로 반환
            return new ServletInvocableHandlerMethod(handlerMethod.getBean(), method, this.applicationContext);
        }
    }

    // ② 전역 탐색 (@ControllerAdvice) — 로컬에서 못 찾은 경우만
    for (Map.Entry<ControllerAdviceBean, ExceptionHandlerMethodResolver> entry :
            this.exceptionHandlerAdviceCache.entrySet()) {
        ControllerAdviceBean advice = entry.getKey();
        // 이 Advice가 현재 컨트롤러에 적용 대상인지 확인
        if (advice.isApplicableToBeanType(handlerType)) {
            ExceptionHandlerMethodResolver resolver = entry.getValue();
            Method method = resolver.resolveMethodByThrowable(exception);
            if (method != null) {
                return new ServletInvocableHandlerMethod(advice.resolveBean(), method, this.applicationContext);
            }
        }
    }
    return null;
}
```

### 4. @RestControllerAdvice vs @ControllerAdvice

```java
// @RestControllerAdvice 정의:
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@ControllerAdvice
@ResponseBody  // ← 유일한 차이
public @interface RestControllerAdvice {
    // @ControllerAdvice와 동일한 속성들
}

// 실질적 차이:
// @ControllerAdvice:
//   @ExceptionHandler 메서드가 String 반환 → View 이름으로 처리 가능
//   ModelAndView 반환 → 뷰 렌더링
//   ResponseEntity 반환 → JSON/XML 등

// @RestControllerAdvice:
//   모든 @ExceptionHandler 메서드에 @ResponseBody 효과 자동 적용
//   → String, Object 반환 → 직렬화되어 응답 본문에 쓰임
//   REST API 전용 예외 처리에 사용

@RestControllerAdvice  // = @ControllerAdvice + @ResponseBody
public class GlobalRestExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(UserNotFoundException ex) {
        return new ErrorResponse("USER_NOT_FOUND", ex.getMessage());
        // @ResponseBody 덕분에 자동으로 JSON 직렬화
    }
}
```

### 5. 여러 @ControllerAdvice 우선순위 시나리오

```java
// 시나리오: API와 Web 영역 분리

@ControllerAdvice(annotations = RestController.class)
@Order(1)  // REST API 예외 처리 우선
public class ApiExceptionHandler {
    @ExceptionHandler(RuntimeException.class)
    public ResponseEntity<ErrorResponse> handleRuntime(RuntimeException ex) {
        return ResponseEntity.internalServerError()
            .body(new ErrorResponse("INTERNAL_ERROR", ex.getMessage()));
    }
}

@ControllerAdvice  // 나머지 Controller (MVC 뷰 기반)
@Order(2)
public class WebExceptionHandler {
    @ExceptionHandler(RuntimeException.class)
    public ModelAndView handleRuntime(RuntimeException ex) {
        ModelAndView mav = new ModelAndView("error/500");
        mav.addObject("message", ex.getMessage());
        return mav;
    }
}

// @RestController 에서 RuntimeException 발생:
//   ApiExceptionHandler (Order=1, annotations=RestController 매칭) → JSON 응답
// @Controller 에서 RuntimeException 발생:
//   ApiExceptionHandler (Order=1, annotations=RestController) → 미매칭
//   WebExceptionHandler (Order=2, 전체 적용) → HTML 뷰
```

---

## 💻 실험으로 확인하기

### 실험 1: basePackages 필터 동작 확인

```java
// com.example.api 패키지의 Controller → ApiExceptionHandler 적용
// com.example.web 패키지의 Controller → WebExceptionHandler 적용

@ControllerAdvice(basePackages = "com.example.api")
public class ApiExceptionHandler {
    @ExceptionHandler(Exception.class)
    public ResponseEntity<String> handle(Exception ex) {
        return ResponseEntity.internalServerError().body("API ERROR: " + ex.getMessage());
    }
}
```

```bash
# com.example.api.UserController에서 예외
curl http://localhost:8080/api/users
# → "API ERROR: ..."

# com.example.web.HomeController에서 예외
curl http://localhost:8080/web/home
# → ApiExceptionHandler 미적용 → DefaultHandlerExceptionResolver 또는 500
```

### 실험 2: @Order 우선순위 확인

```java
// 같은 예외를 처리하는 두 Advice
@ControllerAdvice @Order(1) class First { @ExceptionHandler(Exception.class) ... }
@ControllerAdvice @Order(2) class Second { @ExceptionHandler(Exception.class) ... }
// → First가 항상 선택 (Order=1이 우선)
```

---

## 🌐 HTTP 레벨 분석

```
@RestController UserController에서 RuntimeException 발생

ExceptionHandlerExceptionResolver.getExceptionHandlerMethod():

  ① 로컬 탐색: UserController에 @ExceptionHandler 없음

  ② 전역 탐색 (exceptionHandlerAdviceCache, Order 순):
     ApiExceptionHandler (Order=1, annotations=RestController):
       isApplicableToBeanType(UserController.class):
         UserController에 @RestController 있음 → true ✅
       resolveMethodByThrowable(RuntimeException):
         @ExceptionHandler(RuntimeException.class) 매칭 ✅
       → 선택!

  WebExceptionHandler는 탐색하지 않음

HTTP/1.1 500 Internal Server Error
Content-Type: application/json
{"error":"INTERNAL_ERROR","message":"..."}
```

---

## 📌 핵심 정리

```
@ControllerAdvice 필터 속성
  basePackages       → 패키지 경로 (문자열)
  basePackageClasses → 패키지 기준 타입 (타입 안전)
  annotations        → 특정 어노테이션 붙은 Controller
  assignableTypes    → 특정 클래스 상속/구현 Controller
  필터 없음          → 모든 Controller에 적용

초기화 시점
  afterPropertiesSet() → @ControllerAdvice Bean 스캔 → exceptionHandlerAdviceCache 구성
  LinkedHashMap (Order 기준 정렬 후 삽입)

로컬 vs 전역 탐색 순서
  ① 발생 Controller의 @ExceptionHandler 먼저 탐색
  ② 없으면 @ControllerAdvice 목록을 Order 순으로 탐색
  ③ isApplicableToBeanType() = true인 Advice만 적용

@RestControllerAdvice
  = @ControllerAdvice + @ResponseBody
  모든 핸들러 메서드에 @ResponseBody 자동 적용

다중 @ControllerAdvice 우선순위
  @Order(낮은 값) > @Order(높은 값) > @Order 없음
```

---

## 🤔 생각해볼 문제

**Q1.** `@ControllerAdvice(basePackages = "com.example")` 와 `@ControllerAdvice(annotations = RestController.class)`가 둘 다 있을 때, `com.example` 패키지의 `@RestController`에서 예외가 발생하면 어느 Advice가 우선인가?

**Q2.** `@ControllerAdvice`에 선언된 `@ExceptionHandler`가 현재 요청의 Controller에 적용 대상이 아님에도 실행될 수 있는 시나리오가 있는가?

**Q3.** `@ControllerAdvice` Bean이 `@Lazy`로 선언되면 어떤 문제가 생기는가?

> 💡 **해설**
>
> **Q1.** `@Order` 값이 우선입니다. 두 Advice 모두 해당 Controller에 `isApplicableToBeanType()=true`이면, `exceptionHandlerAdviceCache`의 탐색 순서(즉 `@Order`)가 낮은 것이 먼저 선택됩니다. `@Order`가 같거나 없으면 Bean 정의 순서에 따릅니다.
>
> **Q2.** 없습니다. `isApplicableToBeanType()` 체크를 반드시 통과해야 `@ExceptionHandler` 탐색이 수행됩니다. 단, `handlerMethod`가 `null`인 경우(핸들러를 못 찾은 경우, 예: 404 상황에서 `NoHandlerFoundException`)에는 `handlerType`도 `null`이 되어 필터 없는 `@ControllerAdvice`(전체 적용)가 실행됩니다.
>
> **Q3.** `afterPropertiesSet()`에서 `@ControllerAdvice` Bean을 찾아 캐시를 구성하는데, `@Lazy` Bean은 이 시점에 실제 인스턴스화되지 않아 `ControllerAdviceBean.findAnnotatedBeans()`가 찾을 수 없거나 프록시 타입을 잘못 인식할 수 있습니다. 결과적으로 해당 Advice의 `@ExceptionHandler`가 동작하지 않을 수 있습니다. `@ControllerAdvice`는 `@Lazy` 없이 즉시 초기화로 등록해야 합니다.

---

<div align="center">

**[⬅️ 이전: @ExceptionHandler 매칭](./02-exception-handler-matching.md)** | **[홈으로 🏠](../README.md)** | **[다음: ResponseEntityExceptionHandler ➡️](./04-response-entity-exception-handler.md)**

</div>
