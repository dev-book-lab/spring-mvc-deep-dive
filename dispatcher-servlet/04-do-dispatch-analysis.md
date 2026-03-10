# doDispatch() 완전 분해 — HTTP 요청이 응답이 되기까지의 전체 여정

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `doDispatch()`의 각 단계는 정확히 어떤 순서로 실행되는가?
- `getHandler()`가 `null`을 반환하면 무슨 일이 일어나는가?
- `applyPreHandle()`이 `false`를 반환하면 왜 즉시 메서드가 종료되는가?
- `processDispatchResult()`에서 예외 처리와 View 렌더링은 어떻게 분기되는가?
- multipart 요청은 `doDispatch()` 어디서 처리되는가?
- 비동기 요청(`DeferredResult`, `Callable`)은 `doDispatch()` 흐름에서 어떻게 분기되는가?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: HTTP 요청 처리 과정의 각 단계가 독립적이어야 한다

```
요청 처리에 필요한 관심사:
  1. 누가 처리할 것인가?       → HandlerMapping
  2. 어떻게 실행할 것인가?     → HandlerAdapter
  3. 공통 전/후처리?           → Interceptor (preHandle, postHandle)
  4. 실제 비즈니스 로직?       → Controller Method
  5. 응답을 어떻게 렌더링?     → ViewResolver
  6. 예외가 나면?              → HandlerExceptionResolver
  7. 리소스 정리?              → afterCompletion (항상 보장)

이 관심사들을 하나의 메서드가 조율
  → 각 단계가 분리되어 있어 교체/확장 가능
  → 예외 발생 시에도 afterCompletion 실행 보장
  → 비동기 요청으로의 분기 지점 명확
```

---

## 😱 흔한 오해 또는 실수

### Before: postHandle은 항상 실행된다

```java
// ❌ 잘못된 이해
// "postHandle은 요청 처리 후 항상 실행된다"

// ✅ 실제:
// postHandle은 Handler(Controller) 실행이 정상 완료된 경우에만 실행
// → Controller에서 예외가 throw되면 postHandle 실행 안 됨!
// → afterCompletion만 보장됨

// 실무 함의:
// ❌ postHandle에 "반드시 실행되어야 하는 정리 로직" 두기
// ✅ 반드시 실행해야 하는 정리 로직은 afterCompletion에 두기
```

### Before: doDispatch()가 HttpMessageConverter를 직접 호출한다

```
❌ 잘못된 이해:
  "doDispatch()가 @RequestBody 역직렬화와 @ResponseBody 직렬화를 처리한다"

✅ 실제:
  doDispatch()는 조율자(Orchestrator) 역할만 함
  → ha.handle() 호출 시 HandlerAdapter 내부에서:
     - ArgumentResolver → HttpMessageConverter.read() (@RequestBody 역직렬화)
     - ReturnValueHandler → HttpMessageConverter.write() (@ResponseBody 직렬화)
  → doDispatch()는 이 과정을 직접 알지 못함
  → 관심사 분리 원칙 적용
```

---

## ✨ 올바른 이해와 사용

### After: doDispatch() 전체 흐름 조감도

```
doDispatch(request, response)
  │
  ├─① checkMultipart(request)
  │   → multipart/form-data? → MultipartResolver로 래핑
  │
  ├─② getHandler(processedRequest)
  │   → HandlerMapping 체인 탐색
  │   → 없으면 → noHandlerFound() → 404
  │
  ├─③ getHandlerAdapter(mappedHandler.getHandler())
  │   → supports() 체크 → 적합한 HandlerAdapter 선택
  │   → 없으면 → ServletException("No adapter for handler")
  │
  ├─④ (GET/HEAD) 304 Not Modified 조기 반환 체크
  │   → ha.getLastModified() vs If-Modified-Since 헤더 비교
  │
  ├─⑤ mappedHandler.applyPreHandle(request, response)
  │   → Interceptor preHandle 순서대로 실행
  │   → false 반환 시 → 즉시 return (이후 단계 모두 스킵)
  │
  ├─⑥ ha.handle(request, response, handler)
  │   → 실제 Controller 메서드 실행
  │   → ArgumentResolver → 메서드 호출 → ReturnValueHandler
  │   → ModelAndView 반환 (또는 null — @ResponseBody인 경우)
  │
  ├─⑦ applyDefaultViewName(request, mv)
  │   → view 이름 없으면 URL 기반으로 자동 생성
  │
  ├─⑧ mappedHandler.applyPostHandle(request, response, mv)
  │   → Interceptor postHandle 역순으로 실행
  │
  └─⑨ processDispatchResult(request, response, handler, mv, exception)
      ├─ 예외 있음 → HandlerExceptionResolver 체인으로 처리
      └─ 정상 → render(mv, request, response) → ViewResolver → View.render()
                → 완료 후 mappedHandler.triggerAfterCompletion() (afterCompletion 역순)
```

---

## 🔬 내부 동작 원리

### 전체 doDispatch() 소스 주석 추적

```java
// DispatcherServlet.java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response)
        throws Exception {

    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;

    // 비동기 처리 매니저
    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

    try {
        ModelAndView mv = null;
        Exception dispatchException = null;

        try {
            // ─────────────────────────────────────────────────────
            // ① multipart 요청 처리
            // ─────────────────────────────────────────────────────
            // Content-Type: multipart/form-data 감지
            // MultipartResolver가 등록되어 있으면 MultipartHttpServletRequest로 래핑
            processedRequest = checkMultipart(request);
            multipartRequestParsed = (processedRequest != request);

            // ─────────────────────────────────────────────────────
            // ② 핸들러 탐색
            // ─────────────────────────────────────────────────────
            // HandlerMapping 체인 탐색 → HandlerExecutionChain 반환
            // HandlerExecutionChain = Handler + 적용될 Interceptor 목록
            mappedHandler = getHandler(processedRequest);

            if (mappedHandler == null) {
                // 처리할 핸들러가 없음
                noHandlerFound(processedRequest, response);
                // → response.sendError(404) 또는 NoHandlerFoundException throw
                return;
            }

            // ─────────────────────────────────────────────────────
            // ③ HandlerAdapter 탐색
            // ─────────────────────────────────────────────────────
            // 핸들러 타입에 맞는 Adapter 선택 (supports() 체크)
            HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

            // ─────────────────────────────────────────────────────
            // ④ Last-Modified 체크 (GET/HEAD 최적화)
            // ─────────────────────────────────────────────────────
            String method = request.getMethod();
            boolean isGet = HttpMethod.GET.matches(method);
            if (isGet || HttpMethod.HEAD.matches(method)) {
                long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                if (new ServletWebRequest(request, response)
                        .checkNotModified(lastModified) && isGet) {
                    // If-Modified-Since 조건 충족 → 304 Not Modified 즉시 반환
                    return;
                }
            }

            // ─────────────────────────────────────────────────────
            // ⑤ Interceptor preHandle
            // ─────────────────────────────────────────────────────
            // 등록된 인터셉터 순서대로 preHandle() 호출
            // 하나라도 false 반환하면 처리 중단
            if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                return;  // ← preHandle false → 응답은 인터셉터가 직접 처리한 것으로 간주
            }

            // ─────────────────────────────────────────────────────
            // ⑥ 실제 핸들러 실행 (핵심)
            // ─────────────────────────────────────────────────────
            // RequestMappingHandlerAdapter.handle()
            //   → InvocableHandlerMethod.invokeForRequest()
            //     → ArgumentResolver들로 파라미터 준비
            //     → 리플렉션으로 Controller 메서드 호출
            //     → ReturnValueHandler로 응답 처리
            mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

            // ─────────────────────────────────────────────────────
            // 비동기 처리 분기
            // ─────────────────────────────────────────────────────
            if (asyncManager.isConcurrentHandlingStarted()) {
                // Callable, DeferredResult 등 비동기 처리 시작됨
                // → 별도 스레드에서 결과가 올 때까지 현재 요청 처리 일시 중단
                return;  // 현재 스레드는 여기서 종료
            }

            // ─────────────────────────────────────────────────────
            // ⑦ 기본 View 이름 설정
            // ─────────────────────────────────────────────────────
            applyDefaultViewName(processedRequest, mv);

            // ─────────────────────────────────────────────────────
            // ⑧ Interceptor postHandle (역순)
            // ─────────────────────────────────────────────────────
            mappedHandler.applyPostHandle(processedRequest, response, mv);

        } catch (Exception ex) {
            // ⑥에서 예외 발생 시 dispatchException에 저장
            // (finally에서 처리하지 않고 ⑨에서 일괄 처리)
            dispatchException = ex;
        } catch (Throwable err) {
            dispatchException = new NestedServletException("Handler dispatch failed", err);
        }

        // ─────────────────────────────────────────────────────────
        // ⑨ 결과 처리 (View 렌더링 또는 예외 처리)
        // ─────────────────────────────────────────────────────────
        // dispatchException != null → HandlerExceptionResolver로 예외 처리
        // dispatchException == null → View 렌더링
        // 어느 경우든 afterCompletion은 이 메서드 내에서 보장
        processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);

    } catch (Exception ex) {
        // processDispatchResult에서도 예외 발생 시
        triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
        throw ex;
    } catch (Throwable err) {
        triggerAfterCompletion(processedRequest, response, mappedHandler,
            new NestedServletException("Handler processing failed", err));
        throw err;
    } finally {
        if (asyncManager.isConcurrentHandlingStarted()) {
            // 비동기 처리 중 → afterCompletion은 비동기 완료 시 호출
            if (mappedHandler != null) {
                mappedHandler.applyAfterConcurrentHandlingStarted(
                    processedRequest, response);
            }
        } else {
            // multipart 리소스 정리
            if (multipartRequestParsed) {
                cleanupMultipart(processedRequest);
            }
        }
    }
}
```

### processDispatchResult() — View 렌더링과 예외 처리의 분기점

```java
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
        @Nullable HandlerExecutionChain mappedHandler,
        @Nullable ModelAndView mv,
        @Nullable Exception exception) throws Exception {

    boolean errorView = false;

    if (exception != null) {
        if (exception instanceof ModelAndViewDefiningException mavDefiningException) {
            // ModelAndView를 직접 정의하는 예외
            mv = mavDefiningException.getModelAndView();
        } else {
            // ← HandlerExceptionResolver 체인 실행
            Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
            mv = processHandlerException(request, response, handler, exception);
            // → ExceptionHandlerExceptionResolver(@ExceptionHandler)
            // → ResponseStatusExceptionResolver(@ResponseStatus)
            // → DefaultHandlerExceptionResolver(스프링 표준 예외)
            // → null 반환 시 예외 다시 throw
            errorView = (mv != null);
        }
    }

    // View 렌더링
    if (mv != null && !mv.wasCleared()) {
        render(mv, request, response);  // ViewResolver → View.render()
        if (errorView) {
            WebUtils.clearErrorRequestAttributes(request);
        }
    }

    // afterCompletion 실행 보장
    if (mappedHandler != null) {
        // 예외 여부와 관계없이 항상 실행
        mappedHandler.triggerAfterCompletion(request, response, null);
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 각 단계별 로그로 흐름 추적

```java
// 커스텀 인터셉터로 doDispatch() 흐름 관찰
@Component
public class FlowTracingInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        System.out.println("[②→⑤] preHandle: " + req.getRequestURI());
        System.out.println("       handler: " + handler.getClass().getSimpleName());
        return true; // true → 계속 / false → doDispatch 즉시 return
    }

    @Override
    public void postHandle(HttpServletRequest req, HttpServletResponse res,
                           Object handler, ModelAndView mv) {
        System.out.println("[⑥→⑧] postHandle 실행됨 (Controller 정상 완료)");
        System.out.println("       ModelAndView: " + mv);
    }

    @Override
    public void afterCompletion(HttpServletRequest req, HttpServletResponse res,
                                Object handler, Exception ex) {
        System.out.println("[⑨→] afterCompletion: 항상 실행됨");
        if (ex != null) System.out.println("       예외: " + ex.getMessage());
    }
}
```

```
정상 요청 로그:
  [②→⑤] preHandle: /users/1
  [⑥→⑧] postHandle 실행됨 (Controller 정상 완료)
  [⑨→]  afterCompletion: 항상 실행됨

Controller 예외 발생 시:
  [②→⑤] preHandle: /users/999
  (postHandle 없음 — Controller 예외로 스킵)
  [⑨→]  afterCompletion: 항상 실행됨
           예외: User not found: 999
```

### 실험 2: 브레이크포인트 설정 위치

```
IntelliJ 브레이크포인트 권장 위치:

1. DispatcherServlet.doDispatch()         ← 요청 처리 시작 전체 확인
2. DispatcherServlet.getHandler()         ← 어떤 HandlerMapping이 선택됐는지
3. HandlerExecutionChain.applyPreHandle() ← Interceptor 실행 확인
4. RequestMappingHandlerAdapter.handle()  ← Controller 진입 직전
5. DispatcherServlet.processDispatchResult() ← 예외/View 분기 확인

브레이크포인트 조건 설정 예시:
  doDispatch() 에서 request.getRequestURI().contains("/users")
  → 특정 URL만 멈추도록 필터링
```

### 실험 3: preHandle false 동작 확인

```java
@Component
public class AuthInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler)
            throws Exception {
        String auth = req.getHeader("Authorization");
        if (auth == null) {
            // false 반환 → doDispatch() 즉시 return
            // Controller 실행 없음, postHandle 없음
            // 단, 이미 실행된 preHandle들의 afterCompletion은 실행됨
            res.sendError(HttpServletResponse.SC_UNAUTHORIZED);
            return false;
        }
        return true;
    }
}
```

```bash
# 인증 헤더 없이 요청
curl -v http://localhost:8080/users/1
# HTTP/1.1 401 Unauthorized
# (Controller 코드 한 줄도 실행 안 됨)

# 인증 헤더 포함
curl -v -H "Authorization: Bearer token" http://localhost:8080/users/1
# HTTP/1.1 200 OK
```

---

## 🌐 HTTP 레벨 분석

### 각 단계별 HTTP 관찰 포인트

```
요청: GET /users/999 HTTP/1.1

② getHandler() 실패 (핸들러 없음):
  → response.sendError(404)
  HTTP/1.1 404 Not Found

⑤ preHandle false (인증 실패):
  → 인터셉터가 직접 response 작성
  HTTP/1.1 401 Unauthorized

⑥ ha.handle() 예외 → ⑨ ExceptionResolver 처리:
  → @ExceptionHandler가 JSON 응답 작성
  HTTP/1.1 404 Not Found
  Content-Type: application/json
  {"error":"User not found","id":999}

⑥ @ResponseBody 반환값 → ReturnValueHandler:
  → HttpMessageConverter.write()가 응답 본문 작성
  HTTP/1.1 200 OK
  Content-Type: application/json
  {"id":1,"name":"홍길동"}
```

---

## 🤔 트레이드오프

```
doDispatch()의 조율자 패턴:
  장점  각 관심사 완전 분리 → 단위 테스트 가능
        새 HandlerMapping, Adapter, Interceptor 추가가 쉬움
        afterCompletion 실행 보장 (finally 블록 구조)
  단점  간단한 요청도 여러 컴포넌트를 거쳐야 함
        → 성능보다 확장성 우선 설계
        (실제 오버헤드는 수십 마이크로초 수준)

예외 처리 구조:
  dispatchException을 catch해서 ⑨에서 처리하는 이유:
  → afterCompletion이 반드시 실행되어야 함
  → ⑥에서 예외를 즉시 throw하면 ⑨(processDispatchResult)를 거치지 못함
  → catch → dispatchException 저장 → ⑨에서 ExceptionResolver 실행 → afterCompletion
```

---

## 📌 핵심 정리

```
doDispatch() 9단계 요약

  ① checkMultipart      파일 업로드 요청 래핑
  ② getHandler          HandlerMapping 체인 탐색 (없으면 404)
  ③ getHandlerAdapter   Handler 타입에 맞는 Adapter 선택
  ④ Last-Modified 체크  304 조기 반환 최적화
  ⑤ applyPreHandle      Interceptor 순서대로, false면 즉시 return
  ⑥ ha.handle()         Controller 메서드 실행 (ArgumentResolver + ReturnValueHandler)
  ⑦ 기본 View 이름      view 이름 없으면 URL 기반 자동 생성
  ⑧ applyPostHandle     Interceptor 역순 (Controller 예외 시 스킵)
  ⑨ processDispatch     예외→ExceptionResolver / 정상→View렌더링 / afterCompletion 보장

항상 실행 보장 계층
  afterCompletion: mappedHandler.triggerAfterCompletion()
  → processDispatchResult의 마지막에 실행
  → 예외 발생 시에도 try-catch로 잡아서 triggerAfterCompletion 재호출

비동기 분기
  asyncManager.isConcurrentHandlingStarted() == true
  → ⑥ 이후 현재 스레드 즉시 종료
  → 비동기 처리 완료 시 별도 스레드로 ⑦~⑨ 이후 처리 재개
```

---

## 🤔 생각해볼 문제

**Q1.** 두 개의 인터셉터 A, B가 순서대로 등록되어 있고 A의 `preHandle()`은 true, B의 `preHandle()`은 false를 반환했습니다. 이때 `afterCompletion()`은 A와 B 중 어느 것이 실행되는가?

**Q2.** `ha.handle()` 호출 중 `RuntimeException`이 발생했습니다. `postHandle()`과 `afterCompletion()` 중 어느 것이 실행되는가? `processDispatchResult()`에서 이 예외를 처리하지 못하면(모든 ExceptionResolver가 null 반환) 어떻게 되는가?

**Q3.** `@ResponseBody`가 붙은 Controller 메서드는 `ha.handle()` 내부에서 이미 응답 본문을 `response.getWriter()`에 씁니다. 이 경우 `mv`는 null입니다. `processDispatchResult()`에서 `mv == null`이면 View 렌더링은 어떻게 되는가?

> 💡 **해설**
>
> **Q1.** A의 `afterCompletion()`만 실행됩니다. `applyPreHandle()`은 각 인터셉터의 `preHandle()`을 순서대로 호출하면서, 성공한 인터셉터 인덱스를 `interceptorIndex` 필드에 저장합니다. B가 false를 반환하면 이후 인터셉터를 건너뛰고 `triggerAfterCompletion()`을 호출하는데, 이때 `interceptorIndex`까지만 역순으로 `afterCompletion()`을 실행합니다. B의 `preHandle()`이 false를 반환한 시점의 인덱스는 A가 성공한 인덱스이므로, A의 `afterCompletion()`만 실행됩니다.
>
> **Q2.** `postHandle()`은 실행되지 않습니다. `ha.handle()` 예외는 catch 블록에서 `dispatchException`에 저장되고, `postHandle()`(`applyPostHandle()`)은 try 블록 내에서 호출되므로 예외가 발생하면 도달하지 않습니다. `afterCompletion()`은 `processDispatchResult()` 내에서 `triggerAfterCompletion()`을 호출하므로 항상 실행됩니다. 모든 ExceptionResolver가 null을 반환하면 `processHandlerException()`이 원래 예외를 다시 throw하고, 이 예외는 `doDispatch()` 바깥 try-catch에서 잡혀 `triggerAfterCompletion()`을 한 번 더 호출한 뒤 Servlet 컨테이너로 전파됩니다.
>
> **Q3.** `processDispatchResult()`에서 `mv != null && !mv.wasCleared()` 조건을 확인합니다. `@ResponseBody`의 경우 `ha.handle()` 내에서 반환값이 `null`이므로 `mv == null` 조건으로 View 렌더링 블록을 완전히 건너뜁니다. 응답 본문은 이미 `ReturnValueHandler → HttpMessageConverter.write()`에 의해 작성된 상태입니다. `afterCompletion()`은 `mv` 값과 관계없이 항상 실행됩니다.

---

<div align="center">

**[⬅️ 이전: WebApplicationContext vs RootApplicationContext](./03-web-app-context-hierarchy.md)** | **[홈으로 🏠](../README.md)** | **[다음: HandlerMapping 체인 동작 ➡️](./05-handler-mapping-chain.md)**

</div>
