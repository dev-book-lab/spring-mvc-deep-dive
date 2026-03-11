# HandlerInterceptor 체인 실행 순서 — HandlerExecutionChain과 순서 제어

---

## 🎯 핵심 질문

- `HandlerExecutionChain`이 Interceptor 목록을 관리하는 구조는?
- `preHandle` → 핸들러 실행 → `postHandle` → `afterCompletion` 순서는 코드 어느 위치에서 보장되는가?
- `@Order`와 `Ordered` 인터페이스로 Interceptor 순서를 제어하는 방법은?
- `addInterceptors()` 등록 순서와 `@Order` 중 어느 것이 우선인가?
- `preHandle`이 중간에 `false`를 반환하면 `afterCompletion`은 어떻게 처리되는가?
- URL 패턴으로 특정 Interceptor를 특정 경로에만 적용하는 방법은?

---

## 🔍 왜 이 메커니즘이 존재하는가

```
여러 Interceptor가 등록될 때:
  AuthInterceptor     → 인증 확인
  RateLimitInterceptor → 요청 횟수 제한
  LoggingInterceptor  → 요청/응답 로깅
  MetricsInterceptor  → 성능 지표 수집

실행 순서가 중요:
  인증 먼저 → 실패 시 이후 작업 불필요
  로깅은 마지막 → 전체 처리 결과 기록
  preHandle: 등록 순서대로 (1, 2, 3...)
  postHandle: 역순 (...3, 2, 1)
  afterCompletion: 역순 + preHandle true인 것만
```

---

## 😱 흔한 오해 또는 실수

### Before: postHandle도 preHandle과 같은 순서(정방향)다

```java
// ❌ 잘못된 이해
// "preHandle: 1→2→3, postHandle: 1→2→3 (동일 순서)"

// ✅ 실제:
// preHandle:      1 → 2 → 3 (등록 순서)
// postHandle:     3 → 2 → 1 (역순)
// afterCompletion: 3 → 2 → 1 (역순, preHandle=true인 것만)

// 이유:
// Interceptor를 스택(Stack)처럼 처리
// preHandle: push 순서 (바깥 → 안쪽)
// postHandle/afterCompletion: pop 순서 (안쪽 → 바깥)
// → 마지막으로 시작한 것이 먼저 정리됨 (LIFO)
```

### Before: @Order로 Interceptor 실행 순서를 제어할 수 있다

```java
// ❌ 잘못된 이해
@Component
@Order(1)  // 이 어노테이션이 Interceptor 실행 순서를 결정?
public class AuthInterceptor implements HandlerInterceptor { ... }

// ✅ 실제:
// HandlerInterceptor의 실행 순서는 WebMvcConfigurer.addInterceptors()에서
// 등록된 순서로 결정됨
// @Order는 Spring Bean 주입 순서에는 영향을 주지만
// addInterceptors() 내에서의 add() 순서가 실행 순서를 결정
```

---

## ✨ 올바른 이해와 사용

### After: HandlerExecutionChain 구조

```java
// HandlerExecutionChain: Handler + Interceptor 목록을 함께 보관
public class HandlerExecutionChain {

    private final Object handler;  // Controller (HandlerMethod)
    private final List<HandlerInterceptor> interceptorList = new ArrayList<>();
    private int interceptorIndex = -1;  // preHandle 실행 인덱스 추적

    // DispatcherServlet.doDispatch()에서 사용:
    // mappedHandler.applyPreHandle(request, response)
    // ha.handle(request, response, mappedHandler.getHandler())
    // mappedHandler.applyPostHandle(request, response, mv)
    // mappedHandler.triggerAfterCompletion(request, response, ex)
}
```

---

## 🔬 내부 동작 원리

### 1. applyPreHandle() — 정방향 순서, 실패 시 afterCompletion 보장

```java
// HandlerExecutionChain.applyPreHandle()
boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
    for (int i = 0; i < this.interceptorList.size(); i++) {
        HandlerInterceptor interceptor = this.interceptorList.get(i);
        if (!interceptor.preHandle(request, response, this.handler)) {
            // false 반환 → 체인 중단
            // 지금까지 preHandle=true 반환한 Interceptor들의 afterCompletion 호출
            triggerAfterCompletion(request, response, null);
            return false;  // DispatcherServlet에 false 전달 → Controller 실행 안 함
        }
        this.interceptorIndex = i;  // 성공한 마지막 인덱스 기록
    }
    return true;
}
```

### 2. applyPostHandle() — 역순

```java
// HandlerExecutionChain.applyPostHandle()
void applyPostHandle(HttpServletRequest request, HttpServletResponse response,
                     @Nullable ModelAndView mv) throws Exception {
    for (int i = this.interceptorList.size() - 1; i >= 0; i--) {
        // 역순 실행
        HandlerInterceptor interceptor = this.interceptorList.get(i);
        interceptor.postHandle(request, response, this.handler, mv);
    }
}
```

### 3. triggerAfterCompletion() — 역순, preHandle=true인 것만

```java
// HandlerExecutionChain.triggerAfterCompletion()
void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response,
                              @Nullable Exception ex) {
    // interceptorIndex: preHandle이 성공(true)을 반환한 마지막 인덱스
    // → 그 이하 인덱스만 afterCompletion 호출 (역순)
    for (int i = this.interceptorIndex; i >= 0; i--) {
        HandlerInterceptor interceptor = this.interceptorList.get(i);
        try {
            interceptor.afterCompletion(request, response, this.handler, ex);
        } catch (Throwable ex2) {
            logger.error("HandlerInterceptor.afterCompletion threw exception", ex2);
            // afterCompletion 예외는 삼켜짐 (다른 Interceptor afterCompletion 계속 실행)
        }
    }
}
```

### 4. addInterceptors() 등록 순서 = 실행 순서

```java
// 실행 순서 제어: 등록 순서로 결정
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Autowired AuthInterceptor authInterceptor;
    @Autowired RateLimitInterceptor rateLimitInterceptor;
    @Autowired LoggingInterceptor loggingInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // 등록 순서 = preHandle 실행 순서

        registry.addInterceptor(authInterceptor)   // 인덱스 0 (가장 먼저)
            .addPathPatterns("/api/**");

        registry.addInterceptor(rateLimitInterceptor)  // 인덱스 1
            .addPathPatterns("/api/**");

        registry.addInterceptor(loggingInterceptor)    // 인덱스 2 (가장 나중)
            .addPathPatterns("/**");
    }
}

// preHandle 실행: auth(0) → rateLimit(1) → logging(2)
// postHandle 실행: logging(2) → rateLimit(1) → auth(0)
// afterCompletion: logging(2) → rateLimit(1) → auth(0)
```

### 5. URL 패턴 지정 — addPathPatterns / excludePathPatterns

```java
@Override
public void addInterceptors(InterceptorRegistry registry) {

    registry.addInterceptor(authInterceptor)
        .addPathPatterns("/api/**")         // /api/ 하위 모든 경로
        .excludePathPatterns(
            "/api/auth/login",              // 로그인은 제외
            "/api/auth/refresh",
            "/api/public/**"               // 공개 API 제외
        );

    registry.addInterceptor(adminInterceptor)
        .addPathPatterns("/admin/**");      // 관리자 경로만

    registry.addInterceptor(loggingInterceptor)
        .addPathPatterns("/**")             // 모든 경로
        .order(0);                          // 인터셉터 등록 내 순서 힌트 (실제 순서는 add 순서)
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 순서 확인

```java
// 세 Interceptor가 등록된 경우 실행 순서:
// auth(0), rateLimit(1), logging(2) 순서로 등록

// 정상 요청:
// preHandle:      auth → rateLimit → logging
// Controller 실행
// postHandle:     logging → rateLimit → auth
// afterCompletion: logging → rateLimit → auth

// auth.preHandle() = false인 경우:
// preHandle:      auth (false 반환 → 중단)
// interceptorIndex = -1 (auth도 미완료이므로)
// triggerAfterCompletion: 아무것도 호출 안 됨
// Controller 실행 안 됨

// rateLimit.preHandle() = false인 경우:
// preHandle:      auth (true), rateLimit (false 반환 → 중단)
// interceptorIndex = 0 (auth까지만 성공)
// triggerAfterCompletion: auth.afterCompletion()만 호출
// Controller 실행 안 됨
```

### 실험 2: afterCompletion 예외 무시 확인

```java
@Override
public void afterCompletion(...) {
    throw new RuntimeException("afterCompletion에서 예외!");
    // → 로그에 error로 남고 다음 afterCompletion 계속 실행
    // → 요청 처리에는 영향 없음 (응답은 이미 완료)
}
```

---

## 🌐 HTTP 레벨 분석

```
GET /api/users HTTP/1.1

HandlerExecutionChain:
  handler = UserController.getUsers (HandlerMethod)
  interceptors = [AuthInterceptor, RateLimitInterceptor, LoggingInterceptor]

applyPreHandle():
  [0] AuthInterceptor.preHandle():  토큰 검증 → true (interceptorIndex=0)
  [1] RateLimitInterceptor.preHandle(): 제한 확인 → true (interceptorIndex=1)
  [2] LoggingInterceptor.preHandle(): 시작 시간 기록 → true (interceptorIndex=2)
  → return true → Controller 실행

UserController.getUsers() 실행
  → List<User> 반환

applyPostHandle():
  [2] LoggingInterceptor.postHandle()
  [1] RateLimitInterceptor.postHandle()
  [0] AuthInterceptor.postHandle()

processDispatchResult() → View/JSON 렌더링

triggerAfterCompletion():
  [2] LoggingInterceptor.afterCompletion(): 응답 시간 기록
  [1] RateLimitInterceptor.afterCompletion(): 카운터 감소
  [0] AuthInterceptor.afterCompletion(): 리소스 정리

HTTP/1.1 200 OK
```

---

## 📌 핵심 정리

```
HandlerExecutionChain
  handler + List<HandlerInterceptor> + interceptorIndex

실행 순서
  preHandle:       정방향 (0 → 1 → 2)
  postHandle:      역순   (2 → 1 → 0)
  afterCompletion: 역순   (interceptorIndex → 0, preHandle=true인 것만)

preHandle false 반환 시
  triggerAfterCompletion(지금까지 true 반환한 것만)
  Controller 실행 안 됨

순서 제어
  addInterceptors()의 addInterceptor() 호출 순서 = 실행 순서
  @Order는 Interceptor 실행 순서와 직접 관련 없음

URL 패턴
  addPathPatterns(), excludePathPatterns()
  AntPathMatcher 패턴 사용 (/api/**, /admin/*)
```

---

## 🤔 생각해볼 문제

**Q1.** 3개의 Interceptor가 등록됐을 때 두 번째 Interceptor의 `postHandle()`에서 예외가 발생하면 첫 번째 Interceptor의 `afterCompletion()`은 실행되는가?

**Q2.** `addPathPatterns("/api/**")`와 `addPathPatterns("/api/{version}/**")`를 모두 적용한 Interceptor가 있을 때, `/api/v1/users` 요청에 Interceptor가 몇 번 실행되는가?

**Q3.** 동일한 Interceptor 인스턴스를 두 번 `addInterceptor()`로 등록하면 어떻게 되는가?

> 💡 **해설**
>
> **Q1.** 실행됩니다. `applyPostHandle()`은 `triggerAfterCompletion()`과 별개의 메서드입니다. `postHandle()`에서 예외가 발생하면 `DispatcherServlet.processDispatchResult()`에서 해당 예외를 잡아 `triggerAfterCompletion(request, response, exception)`을 호출합니다. `interceptorIndex`는 모든 `preHandle()`이 `true`였으므로 최대값이고, 역순으로 모든 `afterCompletion()`이 실행됩니다.
>
> **Q2.** 한 번입니다. `addPathPatterns()`는 OR 조건입니다. 하나의 `InterceptorRegistration`에 두 패턴을 추가하면 둘 중 하나라도 매칭되면 Interceptor가 실행됩니다. 같은 Interceptor를 두 번 `addInterceptor()`로 등록한 것이 아니므로 중복 실행은 없습니다.
>
> **Q3.** 같은 인스턴스가 두 번 실행됩니다. `InterceptorRegistry`는 등록 순서대로 목록에 추가하며 중복 체크를 하지 않습니다. 동일 인스턴스가 다른 URL 패턴으로 두 번 등록되면 매칭 시 두 번 실행됩니다. 의도하지 않은 이중 실행이 발생할 수 있으므로 주의가 필요합니다.

---

<div align="center">

**[⬅️ 이전: Filter vs Interceptor](./01-filter-vs-interceptor.md)** | **[홈으로 🏠](../README.md)** | **[다음: preHandle / postHandle / afterCompletion 호출 시점 ➡️](./03-interceptor-lifecycle.md)**

</div>
