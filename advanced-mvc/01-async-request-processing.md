# Async Request Processing — Callable / DeferredResult / CompletableFuture

---

## 🎯 핵심 질문

- `Callable`, `DeferredResult`, `CompletableFuture` 세 방식의 내부 처리 경로 차이는?
- `WebAsyncManager`와 Servlet `AsyncContext`(`startAsync`)의 관계는?
- 비동기 처리에 사용되는 `AsyncTaskExecutor`의 기본값과 커스터마이징 방법은?
- 요청 스레드가 반환된 후 비동기 결과가 완성되면 어떤 경로로 응답이 전송되는가?
- 타임아웃이 발생했을 때의 처리 경로는?
- `DeferredResult.setResult()` vs `DeferredResult.setErrorResult()`의 차이는?

---

## 🔍 왜 이 메커니즘이 존재하는가

```
동기 처리의 문제:
  스레드 1개 = 요청 1개 (블로킹)
  외부 API 호출(100ms) 동안 스레드가 묶임
  동시 요청 1000개 → 1000개 스레드 필요 → 메모리 고갈

비동기 처리 (Servlet 3.0+):
  request.startAsync() → 서블릿 컨테이너가 커넥션 유지
  요청 스레드는 즉시 반환 → 스레드 풀 재활용
  비동기 작업 완료 → asyncContext.dispatch() → 응답 전송

결과:
  적은 스레드로 많은 동시 커넥션 처리 가능
  I/O 바운드 작업에서 처리량 향상
```

---

## 😱 흔한 오해 또는 실수

### Before: Callable 반환 = 요청 스레드에서 병렬 실행이 아니다

```java
// ❌ 잘못된 이해
@GetMapping("/data")
public Callable<Data> getData() {
    return () -> heavyService.compute();
    // "이 Callable이 요청 스레드에서 바로 실행된다"
}

// ✅ 실제:
// 1. 요청 스레드 → Controller 실행 → Callable 반환
// 2. Spring MVC: WebAsyncManager.startCallableProcessing()
//    → request.startAsync()로 Servlet 비동기 모드 진입
//    → AsyncTaskExecutor에 Callable 제출 (별도 스레드)
//    → 요청 스레드 반환 (스레드 풀로)
// 3. 비동기 스레드: Callable.call() 실행
//    → 결과 → asyncContext.dispatch() → DispatcherServlet 재진입
// 4. 재진입 스레드: 결과 직렬화 → 응답 전송
```

### Before: DeferredResult와 Callable은 처리 방식이 같다

```
❌ 잘못된 이해: "둘 다 비동기 처리하므로 동일하다"

✅ 실제 차이:

Callable<T>:
  Spring이 제공한 AsyncTaskExecutor에서 실행
  → Spring이 스레드를 제어
  → 간단한 비동기 작업에 적합

DeferredResult<T>:
  결과 설정 주체가 외부 (메시지 큐, 이벤트 리스너, 웹소켓 콜백 등)
  → 특정 Executor와 무관
  → 다른 스레드에서 result.setResult() 호출 시 완료
  → Long Polling, 이벤트 기반 아키텍처에 적합

CompletableFuture<T>:
  Java 비동기 API 통합
  → thenApply/thenCompose 체이닝 가능
  → 자체 Executor 지정 가능
```

---

## ✨ 올바른 이해와 사용

### After: 세 방식 비교

```java
// 1. Callable<T> — Spring이 관리하는 스레드에서 실행
@GetMapping("/callable")
public Callable<List<User>> getUsers() {
    return () -> {
        // AsyncTaskExecutor 스레드에서 실행
        Thread.sleep(100);  // I/O 시뮬레이션
        return userService.findAll();
    };
}

// 2. DeferredResult<T> — 외부 트리거로 결과 설정
@GetMapping("/deferred")
public DeferredResult<List<User>> getUsersDeferred() {
    DeferredResult<List<User>> result = new DeferredResult<>(5000L);  // 5초 타임아웃

    // 다른 스레드 또는 이벤트 핸들러에서 결과 설정
    eventBus.subscribe("users.loaded", users -> result.setResult(users));
    result.onTimeout(() ->
        result.setErrorResult(ResponseEntity.status(503).body("타임아웃")));

    return result;
    // → 요청 스레드 즉시 반환, result.setResult() 호출 시 응답
}

// 3. CompletableFuture<T> — Java 비동기 API 활용
@GetMapping("/completable")
public CompletableFuture<List<User>> getUsersAsync() {
    return CompletableFuture
        .supplyAsync(() -> userService.findAll(), customExecutor)
        .thenApply(users -> users.stream()
            .filter(User::isActive)
            .toList());
}
```

---

## 🔬 내부 동작 원리

### 1. WebAsyncManager — 비동기 처리 핵심

```java
// WebAsyncManager.java
public class WebAsyncManager {

    private AsyncWebRequest asyncWebRequest;  // Servlet AsyncContext 래퍼
    private AsyncTaskExecutor taskExecutor;   // Callable 실행용 Executor
    private Object concurrentResult = RESULT_NONE;  // 비동기 결과 저장
    private volatile Object[] concurrentResultContext;

    // Callable 처리 시작
    public void startCallableProcessing(WebAsyncTask<?> webAsyncTask, ...) {
        AsyncWebRequest asyncRequest = this.asyncWebRequest;
        asyncRequest.startAsync();             // Servlet startAsync() 호출
        // ← 서블릿 컨테이너가 커넥션 유지, 요청 스레드 반환 허용

        List<CallableProcessingInterceptor> interceptors = ...;
        // afterConcurrentHandlingStarted 콜백

        this.taskExecutor.submit(() -> {
            // 별도 스레드에서 Callable 실행
            Object result = callable.call();
            // 결과를 concurrentResult에 저장
            this.concurrentResult = result;
            // DispatcherServlet 재진입 트리거
            asyncRequest.dispatch();
        });
    }

    // DeferredResult 처리 시작
    public void startDeferredResultProcessing(DeferredResult<?> deferredResult, ...) {
        asyncRequest.startAsync();

        deferredResult.setResultHandler(result -> {
            // result.setResult() 호출 시 여기 실행
            this.concurrentResult = result;
            asyncRequest.dispatch();  // DispatcherServlet 재진입
        });

        deferredResult.onTimeout(() -> {
            // 타임아웃 핸들러 실행
        });
    }
}
```

### 2. DispatcherServlet 재진입 (ASYNC dispatch)

```java
// DispatcherServlet.doDispatch() — ASYNC dispatch 시
// request.getDispatcherType() == DispatcherType.ASYNC

// ① WebAsyncManager.isConcurrentHandlingStarted() = true 확인
// ② concurrentResult를 꺼내 ReturnValueHandler에 전달
//    → Callable의 결과 List<User> → RequestResponseBodyMethodProcessor
//    → JSON 직렬화 → 응답 전송
// ③ Interceptor preHandle() 재호출
// ④ postHandle(), afterCompletion() 정상 호출

// ReturnValueHandler 관점:
// ASYNC dispatch = 이미 처리된 결과가 있는 상태
// → 원래 Callable/DeferredResult/CompletableFuture를 처리한 
//    AsyncHandlerMethodReturnValueHandler가 
//    concurrentResult를 꺼내 일반 반환값처럼 처리
```

### 3. AsyncTaskExecutor 설정

```java
// 기본값: SimpleAsyncTaskExecutor (스레드 재사용 없음, 실무 비권장)
// Spring Boot: TaskExecutionAutoConfiguration이 ThreadPoolTaskExecutor 등록

// 커스터마이징:
@Configuration
public class AsyncConfig implements WebMvcConfigurer {

    @Override
    public void configureAsyncSupport(AsyncSupportConfigurer configurer) {
        // 타임아웃 기본값 설정
        configurer.setDefaultTimeout(30_000);  // 30초

        // 커스텀 Executor 설정
        configurer.setTaskExecutor(asyncTaskExecutor());
    }

    @Bean
    public AsyncTaskExecutor asyncTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("mvc-async-");
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.initialize();
        return executor;
    }
}
```

### 4. WebAsyncTask — Callable + 타임아웃 + Executor 묶음

```java
// WebAsyncTask: 타임아웃/콜백/Executor를 하나로 묶는 래퍼
@GetMapping("/web-async-task")
public WebAsyncTask<List<User>> getUsersWithTimeout() {
    Callable<List<User>> callable = () -> userService.findAll();

    WebAsyncTask<List<User>> webAsyncTask =
        new WebAsyncTask<>(3000L, "customExecutor", callable);
    // 3초 타임아웃, 특정 Executor Bean 이름 지정

    webAsyncTask.onTimeout(() -> {
        return Collections.emptyList();  // 타임아웃 시 빈 목록
    });
    webAsyncTask.onError(() -> {
        throw new AsyncProcessingException("비동기 처리 오류");
    });

    return webAsyncTask;
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 비동기 처리 스레드 확인

```java
@GetMapping("/thread-check")
public DeferredResult<Map<String, String>> threadCheck() {
    String requestThread = Thread.currentThread().getName();
    DeferredResult<Map<String, String>> result = new DeferredResult<>();

    CompletableFuture.runAsync(() -> {
        String asyncThread = Thread.currentThread().getName();
        result.setResult(Map.of(
            "requestThread", requestThread,
            "asyncThread", asyncThread
        ));
    });

    return result;
}
// 응답:
// {"requestThread":"http-nio-8080-exec-1","asyncThread":"ForkJoinPool.commonPool-worker-1"}
```

### 실험 2: 타임아웃 동작 확인

```java
@GetMapping("/timeout-test")
public DeferredResult<String> timeoutTest() {
    DeferredResult<String> result = new DeferredResult<>(1000L);  // 1초

    result.onTimeout(() -> result.setErrorResult(
        ResponseEntity.status(503).body("처리 시간 초과")));

    // 2초 후 결과 설정 (타임아웃 후)
    CompletableFuture.delayedExecutor(2, TimeUnit.SECONDS)
        .execute(() -> result.setResult("늦은 응답"));

    return result;
}
// → 1초 후 503 Service Unavailable
```

---

## 🌐 HTTP 레벨 분석

```
GET /users HTTP/1.1  (DeferredResult 사용)

[요청 스레드: exec-1]
  preHandle() → Controller.getUsersDeferred()
  → DeferredResult 반환
  → WebAsyncManager.startDeferredResultProcessing()
     → request.startAsync() (커넥션 유지)
     → afterConcurrentHandlingStarted()
  exec-1 반환 (스레드 풀)

커넥션: 유지됨 (서블릿 컨테이너가 소유)

[이벤트 스레드 또는 비동기 스레드]
  result.setResult(userList)
  → asyncRequest.dispatch()
  → DispatcherServlet 재진입 (ASYNC dispatch)

[재진입 스레드: exec-2]
  preHandle() 재호출
  → concurrentResult = userList
  → RequestResponseBodyMethodProcessor → JSON 직렬화
  → postHandle(), afterCompletion()

HTTP/1.1 200 OK
Content-Type: application/json
[{"id":1,"name":"홍길동"},...]
```

---

## 📌 핵심 정리

```
세 방식 요약
  Callable     → Spring Executor 스레드에서 실행, 단순 비동기
  DeferredResult → 외부 트리거로 결과 설정, Long Polling/이벤트 기반
  CompletableFuture → Java 비동기 체이닝, 자체 Executor 지정

핵심 컴포넌트
  WebAsyncManager: startAsync() 호출, Executor 관리, dispatch() 트리거
  AsyncContext:    Servlet 3.0 비동기 스펙, 커넥션 유지 및 재dispatch

ASYNC dispatch
  DispatcherType.ASYNC → preHandle 재호출
  concurrentResult → 일반 반환값처럼 처리
  postHandle, afterCompletion 정상 호출

AsyncTaskExecutor
  기본: SimpleAsyncTaskExecutor (실무 비권장)
  권장: ThreadPoolTaskExecutor (configureAsyncSupport)

타임아웃 처리
  DeferredResult(timeout): onTimeout 핸들러
  WebAsyncTask: onTimeout, onError 콜백
  기본값: configurer.setDefaultTimeout()
```

---

## 🤔 생각해볼 문제

**Q1.** `Callable<T>` 반환 방식과 `CompletableFuture<T>` 반환 방식은 모두 비동기 처리를 하지만 스레드 풀 관리 측면에서 어떻게 다른가?

**Q2.** `DeferredResult.setResult()`를 요청 스레드와 동일 스레드에서 즉시 호출하면 비동기 처리가 발생하는가?

**Q3.** `request.startAsync()` 후 `asyncContext.setTimeout(0)`을 설정하면 어떻게 되는가?

> 💡 **해설**
>
> **Q1.** `Callable<T>`는 Spring의 `AsyncTaskExecutor`(보통 `ThreadPoolTaskExecutor`)에 의해 실행됩니다. `WebMvcConfigurer.configureAsyncSupport()`에서 설정한 Executor를 사용하므로 Spring이 스레드 풀을 중앙에서 관리합니다. `CompletableFuture<T>`는 자체 Executor(`CompletableFuture.supplyAsync(() -> ..., myExecutor)`)를 지정할 수 있어 더 세밀한 제어가 가능합니다. 단, Executor를 지정하지 않으면 `ForkJoinPool.commonPool()`을 사용하는데, 이는 Spring의 스레드 풀과 별개이므로 스레드 풀 모니터링과 제어가 어렵습니다.
>
> **Q2.** `setResult()`를 `startAsync()` 이전에 또는 요청 처리 중에 즉시 호출해도, Spring은 내부적으로 결과를 `concurrentResult`에 저장하고 `asyncRequest.dispatch()`를 트리거합니다. 만약 `startAsync()`가 아직 호출되지 않은 상태라면, Spring은 동기적으로 결과를 처리하거나 즉시 dispatch를 실행합니다. 결과적으로 비동기 오버헤드 없이 동기처럼 동작하는 경우도 있습니다.
>
> **Q3.** `setTimeout(0)`은 타임아웃을 무한으로 설정하는 의미입니다 (서블릿 스펙). 비동기 작업이 영원히 완료되지 않아도 타임아웃이 발생하지 않습니다. 실무에서는 명시적인 타임아웃 값을 설정해 리소스 누수를 방지해야 합니다.

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Server-Sent Events ➡️](./02-server-sent-events.md)**

</div>
