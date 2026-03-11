# preHandle / postHandle / afterCompletion 호출 시점

---

## 🎯 핵심 질문

- `preHandle` 반환값이 `false`일 때 이후 체인이 정확히 어떻게 끊기는가?
- 예외 발생 시 `postHandle`은 호출되지 않고 `afterCompletion`만 보장되는 이유는?
- `afterCompletion`의 `Exception ex` 파라미터에는 어떤 예외가 전달되는가?
- 리소스 정리(`ThreadLocal` 해제, 커넥션 반납 등)를 `afterCompletion`에서 해야 하는 이유는?
- `postHandle`에서 `ModelAndView`를 조작하는 것은 언제 유효하고 언제 무의미한가?

---

## 🔍 왜 이 메커니즘이 존재하는가

```
세 콜백이 각각 필요한 이유:

preHandle:
  → Controller 실행 전 → "실행해도 되는가?" 판단
  → false 반환으로 Controller 실행 자체를 막을 수 있음
  → 인증/인가, Rate Limiting, 요청 전처리

postHandle:
  → Controller 실행 후, View 렌더링 전 → 응답 데이터 조작 가능
  → Model에 공통 데이터 추가 (공통 헤더, 메뉴 목록 등)
  → @ResponseBody/@RestController에서는 이미 직렬화됐으므로 의미 없음

afterCompletion:
  → 렌더링까지 완전히 끝난 후 (또는 예외 발생 후)
  → 반드시 실행 보장 (try-finally처럼)
  → 리소스 정리 (ThreadLocal, MDC 해제, 성능 측정 완료)
```

---

## 😱 흔한 오해 또는 실수

### Before: @ResponseBody 컨트롤러에서 postHandle로 응답 본문을 수정할 수 있다

```java
// ❌ 잘못된 이해
@Override
public void postHandle(HttpServletRequest req, HttpServletResponse res,
                        Object handler, ModelAndView mv) throws Exception {
    // mv가 null이면 이미 응답 직렬화 완료 (setRequestHandled=true)
    // → 응답 본문 변경 불가
    if (mv != null) {
        mv.addObject("commonData", commonService.getData());  // 의미 있음
    }
    // @RestController 반환은 mv=null → 여기서 응답 수정 불가
}

// ✅ 올바른 이해:
// @ResponseBody / ResponseEntity → writeWithMessageConverters() 실행
//   → response 스트림에 직접 씀 → 완료
//   → postHandle 시점에는 이미 응답 본문이 스트림에 쓰임
//   → postHandle에서 mv는 null
//   → 응답 본문 추가 변경 불가 (이미 커밋 또는 버퍼에 씌어짐)

// MVC 뷰 기반 Controller → mv에 데이터 추가 가능 (렌더링 전)
```

### Before: afterCompletion의 ex 파라미터는 모든 예외를 포함한다

```
❌ 잘못된 이해: "afterCompletion(ex)의 ex = 모든 처리 중 발생한 예외"

✅ 실제:
  afterCompletion의 ex:
    Controller 또는 Interceptor(preHandle/postHandle)에서 발생한 예외 중
    HandlerExceptionResolver가 처리하지 못한 예외만 전달됨

  @ExceptionHandler로 처리된 예외 → ex = null (정상적으로 처리됨)
  404/405 등 Spring MVC 표준 예외 처리 → ex = null
  미처리 예외 → ex = 실제 예외 객체
```

---

## ✨ 올바른 이해와 사용

### After: 세 콜백의 정확한 호출 시점

```java
// DispatcherServlet.doDispatch() 축약 버전
private void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HandlerExecutionChain mappedHandler = null;
    Exception dispatchException = null;

    try {
        mappedHandler = getHandler(processedRequest);
        HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

        // ① preHandle (Controller 실행 전)
        if (!mappedHandler.applyPreHandle(processedRequest, response)) {
            return;  // false → 즉시 반환, Controller 실행 안 됨
        }

        // ② Controller 실행
        ModelAndView mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

        // ③ postHandle (Controller 실행 후, 렌더링 전)
        mappedHandler.applyPostHandle(processedRequest, response, mv);

    } catch (Exception ex) {
        dispatchException = ex;  // 예외 캡처
    }

    // ④ 렌더링 + afterCompletion (예외 여부 무관)
    processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
}

private void processDispatchResult(..., Exception exception) throws Exception {
    if (exception != null) {
        // 예외 처리 (HandlerExceptionResolver)
        mv = processHandlerException(request, response, handler, exception);
        errorView = (mv != null);
    }

    if (mv != null && !mv.wasCleared()) {
        render(mv, request, response);  // 뷰 렌더링
    }

    if (mappedHandler != null) {
        // ⑤ afterCompletion (렌더링 후 또는 예외 처리 후)
        // exception = 처리되지 못한 예외 (처리됐으면 null)
        mappedHandler.triggerAfterCompletion(request, response, mappedHandler, exception);
    }
}
```

### 예외 시나리오별 콜백 호출 표

```
시나리오                         | preHandle | postHandle | afterCompletion
---------------------------------|-----------|------------|----------------
정상 처리                        |     ✅    |     ✅     |      ✅
preHandle=false                  |     🔴    |     ❌     |   ✅(성공한 것만)
Controller 예외 + @ExceptionHandler|    ✅    |     ❌     |   ✅(ex=null)
Controller 예외 + 미처리          |     ✅    |     ❌     |   ✅(ex=예외)
postHandle 예외                  |     ✅    |     🔴     |   ✅(ex=예외)
afterCompletion 예외             |     ✅    |     ✅     |   🔴(로그+계속)
```

---

## 🔬 내부 동작 원리

### 1. postHandle — @ResponseBody에서 mv가 null인 이유

```java
// RequestResponseBodyMethodProcessor.handleReturnValue()
@Override
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
        ModelAndViewContainer mavContainer, ...) throws Exception {
    mavContainer.setRequestHandled(true);  // ← 핵심
    // ... JSON 직렬화 ...
}

// InvocableHandlerMethod.invokeAndHandle()
this.returnValueHandlers.handleReturnValue(returnValue, ...);

// DispatcherServlet에서:
ModelAndView mv = ha.handle(request, response, handler);
// @ResponseBody → mavContainer.isRequestHandled()=true → mv=null

// applyPostHandle(request, response, mv=null):
for (HandlerInterceptor interceptor : interceptors) {
    interceptor.postHandle(request, response, handler, null);
    // mv = null → postHandle에서 뷰 이름/모델 조작 불가
}
```

### 2. afterCompletion — 리소스 정리 패턴

```java
// 패턴 1: ThreadLocal 해제
@Component
public class MdcInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, ...) {
        String traceId = UUID.randomUUID().toString();
        MDC.put("traceId", traceId);          // ThreadLocal 설정
        MDC.put("requestId", request.getHeader("X-Request-Id"));
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                 Object handler, @Nullable Exception ex) {
        MDC.clear();  // ThreadLocal 반드시 해제
        // preHandle이 true였을 때만 호출되므로 MDC.put이 항상 실행됐음을 보장
    }
}

// 패턴 2: 성능 측정
@Component
public class MetricsInterceptor implements HandlerInterceptor {

    private static final String START_TIME = "startTime";

    @Override
    public boolean preHandle(HttpServletRequest request, ...) {
        request.setAttribute(START_TIME, System.currentTimeMillis());
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                 Object handler, @Nullable Exception ex) {
        Long startTime = (Long) request.getAttribute(START_TIME);
        if (startTime != null) {
            long duration = System.currentTimeMillis() - startTime;
            String uri = request.getRequestURI();
            int status = response.getStatus();
            metricsService.record(uri, status, duration, ex != null);
        }
    }
}
```

### 3. preHandle false 반환 시 응답 처리

```java
// preHandle이 false를 반환할 때 직접 응답을 써야 함
@Component
public class AuthInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                              Object handler) throws Exception {
        String token = request.getHeader("Authorization");
        if (token == null || !tokenService.isValid(token)) {
            // 응답을 직접 작성해야 함
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.setContentType("application/json");
            response.getWriter().write("{\"error\":\"Unauthorized\"}");
            return false;  // Controller 실행 안 됨
            // DispatcherServlet: applyPreHandle() = false → return (doDispatch 종료)
            // postHandle: 호출 안 됨
            // afterCompletion: 이 Interceptor 이전에 성공한 Interceptor들만 호출
        }
        return true;
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: @ResponseBody vs View 반환에서 postHandle mv 차이

```java
// @RestController
@GetMapping("/api/data")
public User apiData() { return new User(1L, "홍길동"); }

// @Controller
@GetMapping("/web/page")
public String webPage(Model model) {
    model.addAttribute("user", new User(1L, "홍길동"));
    return "userPage";
}

// Interceptor postHandle:
// /api/data → mv = null (setRequestHandled=true)
// /web/page → mv = ModelAndView{view="userPage", model={user=...}}
```

### 실험 2: 예외 발생 시 afterCompletion ex 파라미터

```java
@Override
public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                              Object handler, @Nullable Exception ex) {
    if (ex != null) {
        log.warn("요청 처리 중 미처리 예외: {}", ex.getMessage());
    } else {
        log.info("요청 정상 완료: {}", response.getStatus());
    }
    // @ExceptionHandler가 처리한 예외 → ex = null
    // 미처리 예외 → ex = 실제 예외
}
```

---

## 🌐 HTTP 레벨 분석

```
정상 요청 (@RestController):

preHandle: AuthInterceptor → LoggingInterceptor (true)
Controller: UserController.getUser() → ResponseEntity 반환
  → writeWithMessageConverters() → JSON 직렬화
  → mavContainer.setRequestHandled(true) → mv = null
postHandle: LoggingInterceptor(mv=null) → AuthInterceptor(mv=null)
  → mv가 null이므로 뷰 조작 불가
afterCompletion: LoggingInterceptor(ex=null) → AuthInterceptor(ex=null)
  → 리소스 정리

HTTP/1.1 200 OK
Content-Type: application/json
{"id":1,"name":"홍길동"}

예외 발생 (@ExceptionHandler로 처리됨):

preHandle: AuthInterceptor → LoggingInterceptor (true)
Controller: UserController.getUser() → UserNotFoundException
  → ExceptionHandlerExceptionResolver → @ExceptionHandler 실행
  → 404 응답 완료 (예외 처리됨)
postHandle: 호출 안 됨 (예외 발생)
afterCompletion: LoggingInterceptor(ex=null) → AuthInterceptor(ex=null)
  → ex=null (예외가 처리됨)
```

---

## 📌 핵심 정리

```
콜백 호출 보장 조건
  preHandle:       항상 (등록된 순서대로)
  postHandle:      Controller 정상 완료 시만 (예외 발생 시 미호출)
  afterCompletion: preHandle=true 반환한 것들에 대해 항상 보장

afterCompletion의 ex 파라미터
  처리된 예외 (@ExceptionHandler) → null
  미처리 예외 → 예외 객체

postHandle의 mv
  @ResponseBody/@RestController → null (이미 응답 완료)
  View 기반 Controller → ModelAndView 객체

리소스 정리는 afterCompletion에서
  ThreadLocal (MDC, SecurityContext)
  성능 측정 종료
  커스텀 요청 속성 해제

preHandle false 응답 처리
  response.setStatus() + response.getWriter().write()
  직접 응답 작성 필요 (DispatcherServlet이 응답 안 씀)
```

---

## 🤔 생각해볼 문제

**Q1.** `afterCompletion`에서 `response.getWriter().write("추가 데이터")`로 응답에 내용을 추가하면 어떻게 되는가?

**Q2.** `preHandle`에서 `ThreadLocal`에 데이터를 설정하고, `afterCompletion`에서 해제하는 패턴을 사용합니다. `preHandle`이 `false`를 반환하면 `ThreadLocal` 해제가 보장되는가?

**Q3.** `postHandle`에서 `response.setHeader("X-Processed-By", "MyInterceptor")`를 호출하면 실제 응답 헤더에 포함되는가?

> 💡 **해설**
>
> **Q1.** 응답이 이미 커밋(commit)됐거나 본문이 쓰여진 후라면 `IllegalStateException`이 발생할 수 있습니다. 응답이 아직 커밋되지 않았다면 추가 내용이 쓰이지만, 이미 Content-Length 헤더가 설정됐으면 클라이언트에서 파싱 오류가 발생합니다. `afterCompletion`은 렌더링이 완료된 후 호출되므로 응답 본문 수정은 권장하지 않습니다.
>
> **Q2.** 보장되지 않습니다. `preHandle`이 `false`를 반환하면 `interceptorIndex`가 갱신되지 않으므로(이 Interceptor의 index 미기록), `triggerAfterCompletion()`에서 이 Interceptor의 `afterCompletion()`이 호출되지 않습니다. 따라서 `preHandle`에서 `false`를 반환하기 전에 직접 `ThreadLocal`을 해제해야 합니다. 또는 `try-finally` 패턴으로 `preHandle` 내에서 정리하거나, `preHandle`에서 ThreadLocal 설정 전에 false 반환 조건을 먼저 체크해야 합니다.
>
> **Q3.** 포함됩니다. `postHandle`은 `applyPostHandle()` 내에서 호출되고, 이 시점에는 아직 `processDispatchResult()` → `render()` 단계 전입니다. `response.setHeader()`는 헤더를 설정하며, 응답이 아직 커밋되지 않았다면 실제 HTTP 응답 헤더에 포함됩니다. 단, `@ResponseBody` 처리 후 이미 응답 스트림이 flush됐다면 헤더 설정이 무시될 수 있습니다.

---

<div align="center">

**[⬅️ 이전: Interceptor 체인 순서](./02-interceptor-chain-order.md)** | **[홈으로 🏠](../README.md)** | **[다음: CORS 처리 ➡️](./04-cors-handling.md)**

</div>
