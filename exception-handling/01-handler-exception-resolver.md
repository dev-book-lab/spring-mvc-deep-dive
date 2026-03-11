# HandlerExceptionResolver 체인 동작 — 예외가 응답으로 변환되는 경로

---

## 🎯 핵심 질문

- `processDispatchResult()`에서 예외가 전달되는 정확한 시점과 경로는?
- `ExceptionHandlerExceptionResolver` → `ResponseStatusExceptionResolver` → `DefaultHandlerExceptionResolver` 순서와 각 책임은?
- Resolver가 예외를 처리하지 못했을 때 최종적으로 어떻게 되는가?
- `HandlerExceptionResolver.resolveException()` 반환값 `ModelAndView`는 무엇을 의미하는가?
- 예외 처리 체인에서 중간에 예외가 다시 발생하면 어떻게 처리되는가?

---

## 🔍 왜 이 메커니즘이 존재하는가

```
Controller에서 발생 가능한 예외:
  RuntimeException, MethodArgumentNotValidException,
  HttpMessageNotReadableException, NoHandlerFoundException ...

예외마다 다른 응답 필요:
  비즈니스 예외 → 400/409 + 도메인 오류 메시지
  Spring MVC 표준 예외 → 표준화된 4xx 응답
  인증 예외 → 401/403
  미예상 예외 → 500

해결: Chain of Responsibility
  각 Resolver가 "내가 처리할 수 있는가?" 판단
  → 처리 가능한 첫 번째 Resolver가 응답 생성
  → 처리 못하면 다음 Resolver로 위임
```

---

## 😱 흔한 오해 또는 실수

### Before: @ExceptionHandler가 없으면 항상 500이 된다

```java
// ❌ 잘못된 이해
// "@ExceptionHandler 없으면 모든 예외 → 500"

// ✅ 실제:
// DefaultHandlerExceptionResolver가 Spring MVC 표준 예외들을 자동 처리
//   NoHandlerFoundException          → 404
//   HttpRequestMethodNotSupportedException → 405
//   HttpMediaTypeNotSupportedException → 415
//   MethodArgumentNotValidException   → 400
//   ...15가지 이상

// @ResponseStatus 어노테이션이 붙은 예외:
// ResponseStatusExceptionResolver가 처리
@ResponseStatus(HttpStatus.NOT_FOUND)
public class UserNotFoundException extends RuntimeException { }
// → 404 자동 반환

// @ExceptionHandler 없어도 표준 예외는 적절한 코드로 처리됨
```

### Before: HandlerExceptionResolver.resolveException()이 null을 반환하면 예외가 무시된다

```
❌ 잘못된 이해: "null 반환 → 예외 사라짐"

✅ 실제:
  resolveException() null 반환 = "나는 이 예외를 처리 못함"
  → 다음 Resolver로 위임
  → 모든 Resolver가 null 반환 → 예외가 서블릿 컨테이너로 전파
  → 서블릿 컨테이너(Tomcat) 기본 에러 페이지
  → Spring Boot의 BasicErrorController (/error 매핑) 처리
```

---

## ✨ 올바른 이해와 사용

### After: 예외 처리 전체 흐름

```
Controller 메서드 실행 중 예외 발생

① DispatcherServlet.doDispatch()
   try {
     mappedHandler = getHandler(request);
     ha = getHandlerAdapter(handler);
     mv = ha.handle(request, response, handler);  ← 여기서 예외
   } catch (Exception ex) {
     dispatchException = ex;
   }
   processDispatchResult(request, response, mappedHandler, mv, dispatchException);

② processDispatchResult()
   if (exception != null) {
     if (exception instanceof ModelAndViewDefiningException mavEx) {
       mv = mavEx.getModelAndView();  // 특수 케이스
     } else {
       mv = processHandlerException(request, response, handler, exception);
     }
   }

③ processHandlerException()
   for (HandlerExceptionResolver resolver : this.handlerExceptionResolvers) {
     ModelAndView exMv = resolver.resolveException(request, response, handler, ex);
     if (exMv != null) {
       if (exMv.isEmpty()) return null;  // 빈 ModelAndView = 이미 응답 완료
       return exMv;                      // 뷰 렌더링용 ModelAndView 반환
     }
   }
   throw ex;  // 모든 Resolver 실패 → 서블릿 컨테이너로 전파
```

---

## 🔬 내부 동작 원리

### 1. HandlerExceptionResolver 기본 등록 순서

```java
// DispatcherServlet.initHandlerExceptionResolvers()
// 기본 등록 (우선순위 순):

// 1. ExceptionHandlerExceptionResolver (order=0)
//    → @ExceptionHandler 메서드 처리
//    → @ControllerAdvice 전역 처리
//    → 가장 높은 우선순위 (커스텀 예외 처리 담당)

// 2. ResponseStatusExceptionResolver (order=1)
//    → @ResponseStatus 어노테이션 붙은 예외 처리
//    → ResponseStatusException 처리

// 3. DefaultHandlerExceptionResolver (order=2)
//    → Spring MVC 표준 예외 15가지 처리
//    → 가장 낮은 우선순위 (최후 방어선)

// 탐색 흐름:
// ExceptionHandlerExceptionResolver.resolveException()
//   → @ExceptionHandler 없으면 null 반환
// ResponseStatusExceptionResolver.resolveException()
//   → @ResponseStatus 없으면 null 반환
// DefaultHandlerExceptionResolver.resolveException()
//   → 표준 예외 아니면 null 반환
// → 모두 null → throw ex → 서블릿 컨테이너
```

### 2. ExceptionHandlerExceptionResolver

```java
// AbstractHandlerExceptionResolver.resolveException()
@Override
@Nullable
public ModelAndView resolveException(HttpServletRequest request,
        HttpServletResponse response, @Nullable Object handler, Exception ex) {

    if (shouldApplyTo(request, handler)) {
        // 이 Resolver가 이 handler에 적용되는지 확인
        // (mappedHandlers, mappedHandlerClasses 설정 기반)
        prepareResponse(ex, response);
        ModelAndView result = doResolveException(request, response, handler, ex);
        if (result != null) {
            logException(ex, request);
        }
        return result;
    }
    return null;
}

// ExceptionHandlerExceptionResolver.doResolveHandlerMethodException()
// 핵심: @ExceptionHandler 메서드를 찾아 실행
@Override
@Nullable
protected ModelAndView doResolveHandlerMethodException(HttpServletRequest request,
        HttpServletResponse response, @Nullable HandlerMethod handlerMethod, Exception exception) {

    // 1. 해당 Controller에서 @ExceptionHandler 찾기 (로컬 우선)
    ServletInvocableHandlerMethod exceptionHandlerMethod =
        getExceptionHandlerMethod(handlerMethod, exception);

    if (exceptionHandlerMethod == null) return null;  // 없으면 null → 다음 Resolver

    // 2. ArgumentResolver, ReturnValueHandler 설정
    exceptionHandlerMethod.invokeAndHandle(
        webRequest, mavContainer, exception, handlerMethod, cause);
    // → @ExceptionHandler 메서드 실행
    // → ResponseEntity, @ResponseBody 등 정상 처리와 동일한 ReturnValueHandler 사용

    if (mavContainer.isRequestHandled()) {
        return new ModelAndView();  // 빈 ModelAndView = "응답 처리 완료"
    }
    // ...
}
```

### 3. ResponseStatusExceptionResolver

```java
// ResponseStatusExceptionResolver.doResolveException()
@Override
@Nullable
protected ModelAndView doResolveException(HttpServletRequest request,
        HttpServletResponse response, @Nullable Object handler, Exception ex) {

    // 1. @ResponseStatus 어노테이션 확인
    ResponseStatus responseStatus = AnnotatedElementUtils.findMergedAnnotation(
        ex.getClass(), ResponseStatus.class);

    if (responseStatus != null) {
        return resolveResponseStatus(responseStatus, request, response, handler, ex);
    }

    // 2. ResponseStatusException 처리 (Spring 5.x)
    if (ex instanceof ResponseStatusException rse) {
        return resolveResponseStatusException(rse, request, response, handler);
    }

    // 3. 원인 예외(cause) 확인
    if (ex.getCause() instanceof Exception cause) {
        return doResolveException(request, response, handler, cause);
    }

    return null;
}

protected ModelAndView resolveResponseStatus(ResponseStatus responseStatus, ...) throws Exception {
    int statusCode = responseStatus.code().value();
    String reason = responseStatus.reason();
    if (!StringUtils.hasLength(reason)) {
        response.sendError(statusCode);
    } else {
        String resolvedReason = (this.messageSource != null ?
            this.messageSource.getMessage(reason, null, reason, LocaleContextHolder.getLocale()) : reason);
        response.sendError(statusCode, resolvedReason);
    }
    return new ModelAndView();  // 처리 완료
}
```

### 4. DefaultHandlerExceptionResolver — 15가지 표준 예외

```java
// DefaultHandlerExceptionResolver.doResolveException()
@Override
@Nullable
protected ModelAndView doResolveException(HttpServletRequest request,
        HttpServletResponse response, @Nullable Object handler, Exception ex) {
    try {
        if (ex instanceof HttpRequestMethodNotSupportedException subEx) {
            return handleHttpRequestMethodNotSupported(subEx, request, response, handler);
            // → 405 Method Not Allowed + Allow 헤더

        } else if (ex instanceof HttpMediaTypeNotSupportedException subEx) {
            return handleHttpMediaTypeNotSupported(subEx, request, response, handler);
            // → 415 Unsupported Media Type

        } else if (ex instanceof HttpMediaTypeNotAcceptableException subEx) {
            return handleHttpMediaTypeNotAcceptable(subEx, request, response, handler);
            // → 406 Not Acceptable

        } else if (ex instanceof MethodArgumentNotValidException subEx) {
            return handleMethodArgumentNotValidException(subEx, request, response, handler);
            // → 400 Bad Request

        } else if (ex instanceof NoHandlerFoundException subEx) {
            return handleNoHandlerFoundException(subEx, request, response, handler);
            // → 404 Not Found

        } else if (ex instanceof HttpMessageNotReadableException subEx) {
            return handleHttpMessageNotReadable(subEx, request, response, handler);
            // → 400 Bad Request

        } // ... 9가지 더
    } catch (Exception handlerEx) {
        // 예외 처리 중 예외 → 로그 + null 반환
    }
    return null;
}
```

### 5. ModelAndView 반환값의 의미

```java
// resolveException() 반환값 의미:
// null         → 이 Resolver는 처리 못함, 다음 Resolver로
// 빈 ModelAndView (new ModelAndView())
//              → 처리 완료 (응답이 직접 써졌음)
//              → DispatcherServlet: View 렌더링 스킵
// 내용 있는 ModelAndView
//              → 오류 뷰 렌더링
//              → View 이름 + Model 데이터 포함 가능

// 예시:
// @ExceptionHandler + @ResponseBody → 빈 ModelAndView (응답 직접 작성)
// @ExceptionHandler + View 이름 반환 → 내용 있는 ModelAndView
// DefaultHandlerExceptionResolver → response.sendError() 후 빈 ModelAndView
```

---

## 💻 실험으로 확인하기

### 실험 1: 각 Resolver 처리 경로 로그

```yaml
logging:
  level:
    org.springframework.web.servlet.DispatcherServlet: TRACE
    org.springframework.web.servlet.handler.AbstractHandlerExceptionResolver: DEBUG
```

```
# @ExceptionHandler 처리 시:
DEBUG ExceptionHandlerExceptionResolver - Resolving exception from handler [...]: UserNotFoundException
DEBUG ExceptionHandlerExceptionResolver - Using @ExceptionHandler UserController#handleNotFound

# @ResponseStatus 처리 시:
DEBUG ResponseStatusExceptionResolver - Resolving exception from handler [...]: UserNotFoundException
DEBUG ResponseStatusExceptionResolver - Sending error 404

# 미처리 예외:
TRACE DispatcherServlet - Publishing event: ServletRequestHandledEvent
# → 서블릿 컨테이너 → /error 엔드포인트
```

### 실험 2: Resolver 등록 순서 확인

```java
@Autowired
DispatcherServlet dispatcherServlet;

@GetMapping("/resolvers")
public List<String> listResolvers() throws Exception {
    Field field = DispatcherServlet.class.getDeclaredField("handlerExceptionResolvers");
    field.setAccessible(true);
    List<?> resolvers = (List<?>) field.get(dispatcherServlet);
    return resolvers.stream()
        .map(r -> r.getClass().getSimpleName())
        .toList();
}
// → ["ExceptionHandlerExceptionResolver",
//    "ResponseStatusExceptionResolver",
//    "DefaultHandlerExceptionResolver"]
```

---

## 🌐 HTTP 레벨 분석

```
UserNotFoundException 발생 시:

ExceptionHandlerExceptionResolver:
  로컬 @ExceptionHandler(UserNotFoundException.class) 찾음
  → handleNotFound() 실행
  → ResponseEntity.notFound().build()
HTTP/1.1 404 Not Found

@ResponseStatus(NOT_FOUND) UserNotFoundException (핸들러 없을 때):
  ResponseStatusExceptionResolver:
  → response.sendError(404)
HTTP/1.1 404 Not Found

IllegalStateException (아무 핸들러도 없을 때):
  세 Resolver 모두 null → 예외 전파
  → Tomcat 기본 에러 처리
  → Spring Boot BasicErrorController(/error)
HTTP/1.1 500 Internal Server Error
{"timestamp":"...","status":500,"error":"Internal Server Error"}
```

---

## 📌 핵심 정리

```
예외 처리 흐름
  doDispatch() 예외 캐치
  → processHandlerException()
  → Resolver 체인 순서대로 resolveException()
  → null이면 다음, null 아니면 종료
  → 모두 null → 서블릿 컨테이너 전파

기본 Resolver 순서와 책임
  ExceptionHandlerExceptionResolver (1순위):
    @ExceptionHandler 메서드 탐색 및 실행
  ResponseStatusExceptionResolver (2순위):
    @ResponseStatus 어노테이션 / ResponseStatusException 처리
  DefaultHandlerExceptionResolver (3순위):
    Spring MVC 표준 예외 15종 처리

ModelAndView 반환값 의미
  null       → 처리 불가, 다음 Resolver
  빈 MV      → 처리 완료 (응답 직접 작성)
  내용 있는 MV → 오류 뷰 렌더링
```

---

## 🤔 생각해볼 문제

**Q1.** `@ExceptionHandler`가 정의된 `@ControllerAdvice`가 있어도, 동일 예외에 대해 로컬 `@ExceptionHandler`가 있으면 로컬이 우선합니다. 이 우선순위는 어느 시점에 결정되는가?

**Q2.** `DefaultHandlerExceptionResolver`가 `MethodArgumentNotValidException`을 400으로 처리합니다. 그런데 `@ControllerAdvice`에 같은 예외의 `@ExceptionHandler`가 있다면 어느 쪽이 먼저 실행되는가?

**Q3.** `HandlerExceptionResolver.resolveException()`에서 예외 처리 도중 또 다른 예외가 발생하면 어떻게 되는가?

> 💡 **해설**
>
> **Q1.** `ExceptionHandlerExceptionResolver.getExceptionHandlerMethod()`에서 결정됩니다. 이 메서드는 ① 현재 Controller(handlerMethod)의 `@ExceptionHandler` 탐색 → ② `@ControllerAdvice` 전역 탐색 순서로 실행합니다. 로컬에서 먼저 찾으면 전역 탐색 자체를 수행하지 않으므로 로컬이 항상 우선합니다.
>
> **Q2.** `ExceptionHandlerExceptionResolver`(order=0)가 `DefaultHandlerExceptionResolver`(order=2)보다 먼저 실행되므로, `@ControllerAdvice`의 `@ExceptionHandler`가 먼저 선택됩니다. `DefaultHandlerExceptionResolver`에 도달하지 않습니다.
>
> **Q3.** `AbstractHandlerExceptionResolver`의 `resolveException()` 내부에서 `doResolveException()`을 try-catch로 감쌉니다. 처리 중 예외가 발생하면 catch 블록에서 `WARN` 레벨로 로그를 남기고 `null`을 반환합니다. 이후 다음 Resolver로 넘어가거나, 모두 실패하면 원래 예외가 서블릿 컨테이너로 전파됩니다.

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: @ExceptionHandler 메서드 매칭 ➡️](./02-exception-handler-matching.md)**

</div>
