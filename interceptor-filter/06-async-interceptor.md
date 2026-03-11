# Async Request와 Interceptor — afterConcurrentHandlingStarted()

---

## 🎯 핵심 질문

- `DeferredResult`, `Callable` 반환 시 Interceptor 콜백 호출 순서가 어떻게 달라지는가?
- `AsyncHandlerInterceptor.afterConcurrentHandlingStarted()`가 필요한 이유는?
- 비동기 요청에서 `afterCompletion`은 어느 스레드에서, 언제 호출되는가?
- ThreadLocal 기반 Interceptor가 비동기 요청에서 위험한 이유는?
- `WebAsyncManager`가 비동기 처리에서 Interceptor를 어떻게 재호출하는가?
- 비동기 Interceptor 작성 시 `HandlerInterceptor`를 상속해야 하는가, `AsyncHandlerInterceptor`를 구현해야 하는가?

---

## 🔍 왜 이 메커니즘이 존재하는가

```
동기 요청 처리:
  요청 스레드 → preHandle → Controller → postHandle → afterCompletion
  단일 스레드, 순차 실행, ThreadLocal 안전

비동기 요청 처리 (DeferredResult / Callable):
  요청 스레드 → preHandle → Controller (DeferredResult 반환)
    → Controller가 즉시 반환 → postHandle/afterCompletion 안 됨?
    → 비동기 작업 완료 후 별도 스레드 → 실제 응답 작성

문제:
  요청 스레드: preHandle에서 ThreadLocal 설정
  비동기 스레드: ThreadLocal 없음 (다른 스레드)
  afterCompletion: 요청 스레드? 비동기 스레드?
  → 리소스 정리 타이밍을 어디서 해야 하는가?

해결: AsyncHandlerInterceptor
  afterConcurrentHandlingStarted():
    "비동기 처리 시작됨, 이 스레드(요청 스레드)의 리소스 정리해라"
  afterCompletion():
    "비동기 처리까지 완전히 끝남" (비동기 스레드 또는 타임아웃 시)
```

---

## 😱 흔한 오해 또는 실수

### Before: DeferredResult 반환 시 afterCompletion이 즉시 호출된다

```java
// ❌ 잘못된 이해
@GetMapping("/async")
public DeferredResult<String> asyncEndpoint() {
    DeferredResult<String> result = new DeferredResult<>();
    asyncService.processAsync(result);  // 별도 스레드에서 처리
    return result;
    // "Controller가 반환하면 afterCompletion이 바로 호출된다"
}

// ✅ 실제:
// Controller가 DeferredResult를 반환
//   → DispatcherServlet이 "비동기 처리 시작됨"을 감지
//   → afterConcurrentHandlingStarted() 호출 (요청 스레드에서)
//   → 요청 스레드 반환 (서블릿 컨테이너로)
//
// 비동기 스레드에서 result.setResult("done") 호출
//   → DispatcherServlet 재진입 (dispatch)
//   → preHandle 재호출 (새 dispatch이므로)
//   → postHandle, afterCompletion 호출
//   → 실제 응답 전송
```

### Before: HandlerInterceptor를 그냥 상속해도 비동기에서 문제없다

```java
// ❌ 위험한 코드
@Component
public class MdcInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(...) {
        MDC.put("requestId", UUID.randomUUID().toString());  // ThreadLocal 설정
        return true;
    }

    @Override
    public void afterCompletion(...) {
        MDC.clear();  // ThreadLocal 정리
    }
    // afterConcurrentHandlingStarted() 미구현
}

// 비동기 요청에서:
// 요청 스레드: preHandle → MDC 설정
// DeferredResult 반환 → afterConcurrentHandlingStarted() 기본 구현(아무것도 안 함)
// 요청 스레드 반환 → 스레드 풀로 돌아감 (MDC 남아있음!)
// 다른 요청에서 같은 스레드 재사용 → 오염된 MDC

// ✅ AsyncHandlerInterceptor 구현 필요
```

---

## ✨ 올바른 이해와 사용

### After: 비동기 요청의 전체 콜백 흐름

```
DeferredResult<T> 반환 시:

[요청 스레드]
  1. preHandle()          → true
  2. Controller 실행      → DeferredResult 반환
  3. WebAsyncManager 감지 → "비동기 처리 시작"
  4. afterConcurrentHandlingStarted()  ← 요청 스레드 정리
  5. 요청 스레드 반환 (서블릿 컨테이너)

[비동기 스레드 or 타임아웃]
  6. result.setResult("done") 또는 타임아웃
  7. WebAsyncManager → DispatcherServlet.dispatch() 재호출
  8. preHandle()     ← 재호출 (새 dispatch로 인식)
  9. (비동기 결과 처리)
  10. postHandle()
  11. afterCompletion()  ← 최종 정리

Callable<T> 반환 시:
  동일하지만 비동기 스레드 = Spring의 AsyncTaskExecutor 스레드
```

---

## 🔬 내부 동작 원리

### 1. AsyncHandlerInterceptor 인터페이스

```java
// AsyncHandlerInterceptor.java
public interface AsyncHandlerInterceptor extends HandlerInterceptor {

    /**
     * 비동기 처리가 시작되면 호출됨 (요청 스레드에서)
     * postHandle, afterCompletion 대신 호출됨 (같은 dispatch에서)
     *
     * 사용 목적:
     * - 요청 스레드의 ThreadLocal 정리
     * - 요청 스레드에서 열린 리소스 해제
     */
    default void afterConcurrentHandlingStarted(HttpServletRequest request,
            HttpServletResponse response, Object handler) throws Exception {
        // 기본 구현: 아무것도 안 함
    }
}

// HandlerInterceptor vs AsyncHandlerInterceptor:
// HandlerInterceptor: afterConcurrentHandlingStarted 미구현 (아무것도 안 함)
// AsyncHandlerInterceptor: afterConcurrentHandlingStarted 오버라이드 필요
// → 비동기 요청이 있는 API에서는 AsyncHandlerInterceptor 구현 권장
```

### 2. WebAsyncManager가 Interceptor를 호출하는 방법

```java
// WebAsyncManager.java (핵심 로직 축약)
public void startCallableProcessing(Callable<?> callable, Object... processingContext) {
    // ... 비동기 처리 시작

    // afterConcurrentHandlingStarted 호출 (요청 스레드에서)
    for (AsyncHandlerInterceptor interceptor : this.asyncInterceptors) {
        interceptor.afterConcurrentHandlingStarted(request, response, handler);
    }

    // 서블릿 AsyncContext 열기
    AsyncContext asyncContext = request.startAsync();

    // 비동기 실행
    executor.submit(() -> {
        Object result = callable.call();
        // 비동기 결과 설정
        asyncContext.dispatch();  // → DispatcherServlet 재진입
    });
}

// DispatcherServlet 재진입 시 (dispatch):
// → 새로운 HandlerExecutionChain 생성
// → preHandle() 재호출 (새 dispatch이므로)
// → 비동기 결과를 ReturnValueHandler가 처리
// → postHandle(), afterCompletion() 정상 호출
```

### 3. 안전한 비동기 Interceptor 구현 — MDC 전파 패턴

```java
@Component
public class MdcAsyncInterceptor implements AsyncHandlerInterceptor {

    private static final String MDC_REQUEST_ID = "requestId";
    private static final String SAVED_MDC_ATTR = "savedMdcContext";

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                              Object handler) {
        // 최초 dispatch: MDC 설정
        String requestId = request.getHeader("X-Request-Id");
        if (requestId == null) requestId = UUID.randomUUID().toString();

        MDC.put(MDC_REQUEST_ID, requestId);

        // 비동기 dispatch에서 MDC 복원을 위해 request attribute에 저장
        Map<String, String> mdcContext = MDC.getCopyOfContextMap();
        if (mdcContext != null) {
            request.setAttribute(SAVED_MDC_ATTR, mdcContext);
        }
        return true;
    }

    @Override
    public void afterConcurrentHandlingStarted(HttpServletRequest request,
            HttpServletResponse response, Object handler) {
        // 요청 스레드 MDC 정리 (비동기 처리 시작됨)
        MDC.clear();
        // → 이 스레드가 스레드 풀로 반환될 때 MDC 오염 방지
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                 Object handler, @Nullable Exception ex) {
        // 비동기 처리 완료 후 최종 MDC 정리
        MDC.clear();
    }
}

// 비동기 작업에서 MDC 전파하기:
@Service
public class AsyncUserService {

    @Async
    public CompletableFuture<List<User>> findUsersAsync(HttpServletRequest request) {
        // request attribute에서 MDC 복원
        Map<String, String> savedMdc =
            (Map<String, String>) request.getAttribute("savedMdcContext");
        if (savedMdc != null) {
            MDC.setContextMap(savedMdc);
        }
        try {
            return CompletableFuture.completedFuture(userRepository.findAll());
        } finally {
            MDC.clear();  // 비동기 스레드 정리
        }
    }
}
```

### 4. 타임아웃 처리

```java
// DeferredResult 타임아웃 시:
DeferredResult<String> result = new DeferredResult<>(5000L);  // 5초 타임아웃
result.onTimeout(() -> {
    result.setErrorResult(
        ResponseEntity.status(503).body("처리 시간 초과")
    );
});

// 타임아웃 발생 시 Interceptor 호출 흐름:
// afterConcurrentHandlingStarted() → (비동기 대기) → 타임아웃
// → WebAsyncManager.handleTimeoutCallback()
// → 타임아웃 핸들러(result.setErrorResult()) 실행
// → DispatcherServlet 재dispatch
// → preHandle() 재호출 → afterCompletion(ex=null) 호출
```

---

## 💻 실험으로 확인하기

### 실험 1: 비동기 Interceptor 콜백 순서 로그

```java
@Component
public class AsyncLoggingInterceptor implements AsyncHandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        log.info("[{}] preHandle (스레드: {})", getDispatchType(req), Thread.currentThread().getName());
        return true;
    }

    @Override
    public void postHandle(...) {
        log.info("[{}] postHandle (스레드: {})", getDispatchType(req), Thread.currentThread().getName());
    }

    @Override
    public void afterCompletion(...) {
        log.info("[{}] afterCompletion (스레드: {})", getDispatchType(req), Thread.currentThread().getName());
    }

    @Override
    public void afterConcurrentHandlingStarted(...) {
        log.info("afterConcurrentHandlingStarted (스레드: {})", Thread.currentThread().getName());
    }

    private String getDispatchType(HttpServletRequest req) {
        return req.getDispatcherType().name();  // REQUEST 또는 ASYNC
    }
}
```

```
[REQUEST] preHandle            (스레드: http-nio-8080-exec-1)
[REQUEST] afterConcurrentHandlingStarted (스레드: http-nio-8080-exec-1)
... 비동기 처리 ...
[ASYNC]   preHandle            (스레드: task-1)  ← 비동기 스레드
[ASYNC]   postHandle           (스레드: task-1)
[ASYNC]   afterCompletion      (스레드: task-1)
```

### 실험 2: ThreadLocal 오염 확인

```java
// afterConcurrentHandlingStarted() 없이 MDC 사용 시:
// 요청 1: 스레드 exec-1, MDC["requestId"] = "req-1"
// DeferredResult 반환 → afterConcurrentHandlingStarted 미호출
// 스레드 exec-1 반환 → 스레드 풀
// 요청 2: 스레드 exec-1 재사용 → MDC["requestId"] 여전히 "req-1" (오염!)
```

---

## 🌐 HTTP 레벨 분석

```
GET /api/async-users HTTP/1.1

[요청 스레드: exec-1]
preHandle():                  MDC 설정, return true
Controller:                   DeferredResult 반환
WebAsyncManager:              "비동기 시작" 감지
afterConcurrentHandlingStarted(): MDC.clear() ← exec-1 정리
AsyncContext.startAsync():    서블릿 비동기 모드 진입
exec-1 반환:                  스레드 풀로 돌아감 (MDC 깨끗)

[비동기 스레드: task-1]
userRepository.findAllAsync() 실행
result.setResult(userList)
asyncContext.dispatch()       → DispatcherServlet 재진입

[비동기 스레드: task-1 - ASYNC dispatch]
preHandle():                  MDC 복원 (request attribute에서)
ReturnValueHandler:           비동기 결과 처리 → JSON 직렬화
postHandle()
afterCompletion():            MDC.clear() 최종 정리

HTTP/1.1 200 OK
Content-Type: application/json
[{"id":1,"name":"홍길동"},...]
```

---

## 📌 핵심 정리

```
비동기 요청 Interceptor 콜백 순서
  최초 dispatch (REQUEST):
    preHandle → (Controller: DeferredResult반환)
    → afterConcurrentHandlingStarted (요청 스레드)
    → 요청 스레드 반환

  비동기 완료 후 재dispatch (ASYNC):
    preHandle → postHandle → afterCompletion (비동기 스레드)

afterConcurrentHandlingStarted 용도
  요청 스레드의 ThreadLocal 정리 (MDC, SecurityContext 등)
  → 스레드 풀 반환 전 오염 방지

AsyncHandlerInterceptor vs HandlerInterceptor
  비동기 API 있으면 AsyncHandlerInterceptor 구현 권장
  afterConcurrentHandlingStarted() 오버라이드로 스레드 정리

MDC 비동기 전파 패턴
  preHandle: MDC 설정 + request.setAttribute(mdcSnapshot)
  afterConcurrentHandlingStarted: MDC.clear() (요청 스레드)
  비동기 작업: request.getAttribute(mdcSnapshot)으로 복원
  afterCompletion: MDC.clear() (비동기 스레드)

preHandle 재호출 (ASYNC dispatch 시)
  두 번째 preHandle은 ASYNC dispatch → DispatcherType으로 구분 가능
  request.getDispatcherType() == DispatcherType.ASYNC
```

---

## 🤔 생각해볼 문제

**Q1.** `AsyncHandlerInterceptor`를 구현한 Interceptor에서 `afterConcurrentHandlingStarted()` 메서드를 오버라이드하지 않으면 어떤 일이 발생하는가?

**Q2.** `Callable<T>`를 반환하는 컨트롤러에서 `SecurityContextHolder`의 인증 정보가 비동기 스레드로 전파되지 않는 문제를 어떻게 해결하는가?

**Q3.** 비동기 dispatch 시 `preHandle()`이 재호출될 때, 첫 번째 dispatch(REQUEST)와 두 번째 dispatch(ASYNC)를 구분해 다른 처리를 하려면 어떻게 해야 하는가?

> 💡 **해설**
>
> **Q1.** 기본 구현(`default void`)이 "아무것도 안 함"이므로 컴파일·런타임 오류는 없습니다. 단, 요청 스레드의 ThreadLocal(MDC, SecurityContext 등)이 정리되지 않은 채 스레드 풀로 반환됩니다. 다음 요청에서 같은 스레드를 재사용하면 이전 요청의 ThreadLocal 데이터가 남아 있어 데이터 오염이 발생합니다. `afterConcurrentHandlingStarted()`를 오버라이드해 `MDC.clear()` 등을 반드시 구현해야 합니다.
>
> **Q2.** Spring Security의 `SecurityContextHolder`는 기본적으로 `ThreadLocal` 전략을 사용해 스레드 간 전파가 안 됩니다. 해결 방법은 `SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_INHERITABLETHREADLOCAL)`로 전략을 변경하거나, Spring Security의 `DelegatingSecurityContextCallable`로 Callable을 래핑해 SecurityContext를 명시적으로 전파하는 것입니다. 또는 `@Async` + `SecurityContextAsyncTaskDecorator`를 설정해 자동 전파하는 방법도 있습니다.
>
> **Q3.** `request.getDispatcherType()`으로 구분합니다. `DispatcherType.REQUEST`이면 최초 요청 dispatch, `DispatcherType.ASYNC`이면 비동기 완료 후 재dispatch입니다. `preHandle()` 내에서 `if (request.getDispatcherType() == DispatcherType.ASYNC) { return true; }` 처럼 분기하면 비동기 dispatch 시 인증 재검사 등 불필요한 처리를 건너뛸 수 있습니다.

---

<div align="center">

**[⬅️ 이전: Custom Interceptor 패턴](./05-custom-interceptor-pattern.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 7 — Advanced MVC Topics ➡️](../advanced-mvc/01-async-request-processing.md)**

</div>
