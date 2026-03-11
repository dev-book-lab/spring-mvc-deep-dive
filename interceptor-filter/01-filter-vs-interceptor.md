# Filter vs HandlerInterceptor 차이점 — 실행 위치와 접근 범위

---

## 🎯 핵심 질문

- Filter와 HandlerInterceptor는 요청 처리 파이프라인의 어느 지점에서 실행되는가?
- Filter는 Spring Bean을 주입받을 수 있는가, 없는가? 그 이유는?
- `HttpServletRequest`를 래핑(`wrapping`)할 수 있는 시점은 각각 어디인가?
- Filter에서는 Spring Security가 작동하지만 Interceptor에서는 왜 Session 기반 인증이 더 어려운가?
- `DispatcherServlet` 예외를 Filter에서 잡을 수 없는 이유는?
- 같은 작업(예: 요청 로깅)을 Filter와 Interceptor 중 어디서 구현해야 하는가?

---

## 🔍 왜 이 메커니즘이 존재하는가

```
HTTP 요청 처리 파이프라인:

[클라이언트]
    ↓ HTTP 요청
[서블릿 컨테이너 (Tomcat)]
    ↓
[Filter Chain]        ← javax.servlet.Filter (Servlet 스펙)
    ↓
[DispatcherServlet]   ← Spring MVC 진입점
    ↓
[HandlerInterceptor]  ← Spring MVC 전용
    ↓
[Controller 메서드]

왜 두 가지?
  Filter:      Servlet 스펙 → Spring 이전에 실행
               → Spring을 쓰지 않아도 동작
               → 모든 요청 (정적 리소스 포함) 처리 가능
               → 요청/응답 객체 완전 교체 가능

  Interceptor: Spring MVC 전용 → Spring Bean 완전 활용
               → Handler(Controller) 정보 접근 가능
               → DispatcherServlet 내부에서만 실행
```

---

## 😱 흔한 오해 또는 실수

### Before: Filter는 Spring Bean을 주입받을 수 없다

```java
// ❌ 잘못된 이해 (과거에는 사실이었지만 현재는 아님)
public class MyFilter implements Filter {
    @Autowired
    private UserService userService;  // "주입 불가"
}

// ✅ 실제 (Spring Boot 환경):
// Spring Boot의 자동 구성이 Filter를 Spring Bean으로 관리
// @Component로 등록하거나 FilterRegistrationBean으로 등록하면 @Autowired 가능

@Component
public class MyFilter extends OncePerRequestFilter {
    @Autowired
    private UserService userService;  // ✅ 주입됨

    @Override
    protected void doFilterInternal(...) {
        userService.doSomething();  // Spring Bean 사용 가능
    }
}

// 단, web.xml로 등록한 전통적 Filter는 Spring 컨텍스트 밖에서 등록되므로 주입 불가
// → DelegatingFilterProxy 패턴으로 해결 (Spring Security가 사용하는 방식)
```

### Before: Interceptor는 모든 요청에 실행된다

```
❌ 잘못된 이해: "Interceptor가 등록되면 모든 HTTP 요청에 실행"

✅ 실제:
  Interceptor는 DispatcherServlet에 도달한 요청에만 실행
  → 정적 리소스 요청 (/static/**, /images/**)가
    DispatcherServlet으로 라우팅되면 실행됨
    (ResourceHttpRequestHandler를 통해)
  → 정적 리소스가 다른 서블릿(DefaultServlet 등)이 처리하면 미실행

  Filter는 서블릿 컨테이너 레벨 → 모든 요청에 실행
  (URL 패턴으로 범위 제한 가능)
```

---

## ✨ 올바른 이해와 사용

### After: 전체 파이프라인에서의 위치

```
HTTP 요청:

┌─────────────────────────────────────────────────────────────┐
│ 서블릿 컨테이너 (Tomcat)                                        │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Filter Chain (javax.servlet.Filter)                  │   │
│  │   Filter1 (예: CharacterEncodingFilter)               │   │
│  │   Filter2 (예: SecurityContextPersistenceFilter)      │   │
│  │   Filter3 (예: CorsFilter)                            │   │
│  │       ↓                                              │   │
│  │  ┌─────────────────────────────────────────────┐     │   │
│  │  │ DispatcherServlet                           │     │   │
│  │  │                                             │     │   │
│  │  │  ┌────────────────────────────────────┐     │     │   │
│  │  │  │ HandlerInterceptor Chain           │     │     │   │
│  │  │  │   Interceptor1.preHandle()         │     │     │   │
│  │  │  │   Interceptor2.preHandle()         │     │     │   │
│  │  │  │       ↓                            │     │     │   │
│  │  │  │  Controller 메서드 실행               │     │     │   │
│  │  │  │       ↓                            │     │     │   │
│  │  │  │   Interceptor2.postHandle()        │     │     │   │
│  │  │  │   Interceptor1.postHandle()        │     │     │   │
│  │  │  │   Interceptor2.afterCompletion()   │     │     │   │
│  │  │  │   Interceptor1.afterCompletion()   │     │     │   │
│  │  │  └────────────────────────────────────┘     │     │   │
│  │  └─────────────────────────────────────────────┘     │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔬 내부 동작 원리

### 1. Filter 실행 — 서블릿 컨테이너 레벨

```java
// javax.servlet.Filter 인터페이스
public interface Filter {
    void init(FilterConfig config) throws ServletException;

    void doFilter(ServletRequest request, ServletResponse response,
                  FilterChain chain) throws IOException, ServletException;
    // chain.doFilter() → 다음 Filter 또는 최종 Servlet(DispatcherServlet) 호출
    // chain.doFilter() 전후로 요청/응답 처리

    void destroy();
}

// 실행 시점:
// DispatcherServlet.service() 호출 전 (Filter1 → Filter2 → ... → DispatcherServlet)
// DispatcherServlet.service() 종료 후 역순 (... → Filter2 → Filter1)
// → DispatcherServlet 내부의 모든 예외를 감쌀 수 있음

// 요청 객체 래핑 예시:
public class RequestCachingFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain chain) throws ... {
        // HttpServletRequest를 완전히 교체 가능 (ContentCachingRequestWrapper 등)
        ContentCachingRequestWrapper wrappedRequest = new ContentCachingRequestWrapper(request);
        chain.doFilter(wrappedRequest, response);
        // → 이후 모든 Filter, Interceptor, Controller가 wrappedRequest를 받음
    }
}
```

### 2. HandlerInterceptor 실행 — Spring MVC 레벨

```java
// HandlerInterceptor 인터페이스
public interface HandlerInterceptor {

    // Controller 실행 전
    // false 반환 → 체인 중단, Controller 실행 안 됨
    default boolean preHandle(HttpServletRequest request,
                               HttpServletResponse response,
                               Object handler) throws Exception {
        return true;
    }

    // Controller 실행 후, View 렌더링 전
    // 예외 발생 시 호출 안 됨
    default void postHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler,
                             @Nullable ModelAndView modelAndView) throws Exception {}

    // View 렌더링 후 (또는 예외 발생 후)
    // preHandle이 true를 반환한 경우만 보장
    default void afterCompletion(HttpServletRequest request,
                                  HttpServletResponse response,
                                  Object handler,
                                  @Nullable Exception ex) throws Exception {}
}

// 실행 시점 (DispatcherServlet.doDispatch() 내부):
// mappedHandler.applyPreHandle(request, response)
//   → Interceptor1.preHandle(), Interceptor2.preHandle()
// ha.handle() → Controller 실행
// mappedHandler.applyPostHandle(request, response, mv)
//   → Interceptor2.postHandle(), Interceptor1.postHandle() (역순)
// processDispatchResult() → View 렌더링 + afterCompletion
```

### 3. 접근 가능한 정보 비교

```java
// Filter에서 접근 가능:
doFilter(HttpServletRequest req, HttpServletResponse res, FilterChain chain) {
    req.getRequestURI();          // URL
    req.getHeader("Authorization"); // 헤더
    req.getInputStream();         // 요청 본문 (직접)
    res.setHeader("X-Custom", ""); // 응답 헤더 설정
    // handler(Controller) 정보: 접근 불가
    // Spring Bean: @Component로 등록 시 가능
}

// Interceptor에서 접근 가능:
preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
    req.getRequestURI();          // URL
    req.getHeader("Authorization"); // 헤더
    // handler = HandlerMethod (Controller 정보 포함!)
    HandlerMethod hm = (HandlerMethod) handler;
    hm.getMethod();              // 실행될 Controller 메서드
    hm.getBeanType();            // Controller 클래스
    hm.getMethodAnnotation(RequiresRole.class); // 메서드 어노테이션
    // Spring Bean: DispatcherServlet 컨텍스트에서 완전히 접근 가능
}
```

### 4. DelegatingFilterProxy — Spring Security의 Filter 통합 방법

```java
// Spring Security는 Filter로 동작하지만 Spring Bean을 사용해야 함
// → DelegatingFilterProxy 패턴

// 서블릿 컨테이너에 등록: DelegatingFilterProxy("springSecurityFilterChain")
// → 요청 시 Spring ApplicationContext에서 "springSecurityFilterChain" Bean 조회
// → 실제 Filter 로직은 Spring Bean인 FilterChainProxy에 위임

// 핵심:
// DelegatingFilterProxy = 서블릿 컨테이너 레벨 등록
// FilterChainProxy = Spring Bean, 실제 로직 처리
// → Filter가 Spring Bean의 장점(DI, 생명주기 관리) 모두 활용 가능
```

---

## 💻 실험으로 확인하기

### 실험 1: 실행 순서 로그로 확인

```java
@Component
public class LoggingFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(...) throws ... {
        System.out.println("1. Filter 진입");
        chain.doFilter(request, response);
        System.out.println("6. Filter 종료");
    }
}

@Component
public class LoggingInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(...) {
        System.out.println("2. Interceptor preHandle");
        return true;
    }
    @Override
    public void postHandle(...) {
        System.out.println("4. Interceptor postHandle");
    }
    @Override
    public void afterCompletion(...) {
        System.out.println("5. Interceptor afterCompletion");
    }
}

@GetMapping("/test")
public String test() {
    System.out.println("3. Controller 실행");
    return "ok";
}
```

```
출력:
1. Filter 진입
2. Interceptor preHandle
3. Controller 실행
4. Interceptor postHandle
5. Interceptor afterCompletion
6. Filter 종료
```

### 실험 2: DispatcherServlet 예외를 Filter에서 잡기

```java
@Component
public class ExceptionCapturingFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(...) throws ... {
        try {
            chain.doFilter(request, response);
            // DispatcherServlet 내부 예외는 HandlerExceptionResolver가 이미 처리
            // → Filter까지 예외가 전파되지 않음 (응답 코드만 설정됨)
        } catch (Exception e) {
            // 여기서는 잡히지 않음 (이미 처리된 예외)
            // 단, HandlerExceptionResolver가 처리 못한 예외는 전파됨
        }
    }
}
```

---

## 🌐 HTTP 레벨 분석

```
GET /api/users HTTP/1.1
Authorization: Bearer token123

서블릿 컨테이너 레벨 처리:
  CharacterEncodingFilter:     인코딩 설정
  SecurityContextFilter:       JWT 파싱 → SecurityContext 설정
  ← Spring Security는 여기서 인증 처리 (DispatcherServlet 전)

DispatcherServlet 진입
  HandlerInterceptor 체인:
    AuthCheckInterceptor.preHandle():
      handler = HandlerMethod(UserController.getUsers)
      @RequiresRole 어노테이션 확인 → SecurityContext 활용 → 통과
    → UserController.getUsers() 실행

Filter가 적합한 경우:
  CORS 처리 (모든 요청 포함 Preflight)
  요청 본문 캐싱 (InputStream은 한 번만 읽힘)
  인코딩 설정
  Spring Security 인증/인가

Interceptor가 적합한 경우:
  로그인 세션 확인 (Spring Bean 활용)
  Controller 어노테이션 기반 권한 체크
  요청/응답 로깅 (Controller 정보 포함)
  ModelAndView 조작
```

---

## 🤔 트레이드오프

```
Filter 선택:
  + Servlet 표준 → Spring 없이도 동작
  + 모든 요청 처리 (DispatcherServlet 이전 포함)
  + 요청/응답 객체 완전 래핑 가능
  + Spring Security 등 인프라 레벨 처리에 적합
  - Controller/Handler 정보 접근 불가
  - ModelAndView 조작 불가

Interceptor 선택:
  + HandlerMethod 정보 (메서드, 어노테이션) 접근
  + postHandle에서 ModelAndView 조작
  + Spring Bean 완전 활용 (Spring 컨텍스트 내부)
  - DispatcherServlet에 도달한 요청만 처리
  - 요청 본문 래핑 불가 (InputStream 이미 읽혔을 수 있음)

결정 기준:
  Spring 컨텍스트 외부 작업     → Filter
  Controller 정보 필요           → Interceptor
  인증/인가 (Spring Security)    → Filter (DelegatingFilterProxy)
  비즈니스 로직 기반 권한 검사   → Interceptor
  모든 요청 처리                 → Filter
```

---

## 📌 핵심 정리

```
실행 위치
  Filter:      서블릿 컨테이너 레벨, DispatcherServlet 감싸기
  Interceptor: DispatcherServlet 내부, Controller 감싸기

Handler 정보 접근
  Filter:      불가 (Controller 어노테이션, 메서드 정보)
  Interceptor: preHandle의 Object handler → HandlerMethod 캐스팅

Spring Bean 주입
  Filter:      @Component 또는 FilterRegistrationBean으로 등록 시 가능
               (전통 web.xml 방식은 DelegatingFilterProxy 필요)
  Interceptor: 항상 가능 (Spring 컨텍스트 내부)

ModelAndView 조작
  postHandle(ModelAndView mv) → 뷰 이름/모델 변경 가능
  Filter: 불가

DispatcherServlet 예외 포착
  Filter: HandlerExceptionResolver가 처리한 예외는 전파되지 않음
  Interceptor: afterCompletion(Exception ex)로 예외 정보 수신
```

---

## 🤔 생각해볼 문제

**Q1.** Spring Security의 `@PreAuthorize`는 Filter에서 실행되는가, Interceptor에서 실행되는가?

**Q2.** `OncePerRequestFilter`를 사용하는 이유는? 일반 Filter와 어떤 차이가 있는가?

**Q3.** `ContentCachingRequestWrapper`로 요청 본문을 캐싱하는 Filter가 있을 때, 이후 Controller에서 `@RequestBody`로 동일 본문을 읽을 수 있는가?

> 💡 **해설**
>
> **Q1.** `@PreAuthorize`는 AOP 기반으로 동작합니다. Spring Security의 `MethodSecurityInterceptor`가 AOP 어드바이스로 Controller 메서드 호출 전에 실행됩니다. Filter도 Interceptor도 아닌 AOP 레이어입니다. 실행 순서는 Filter → HandlerInterceptor → AOP(`@PreAuthorize`) → Controller 메서드입니다.
>
> **Q2.** `OncePerRequestFilter`는 하나의 요청에서 딱 한 번만 `doFilterInternal()`이 실행됨을 보장합니다. 서블릿 컨테이너에서 포워드(forward)나 인클루드(include) 처리 시 Filter가 여러 번 호출될 수 있는데, `OncePerRequestFilter`는 `REQUEST_ALREADY_FILTERED_ATTRIBUTE`를 요청 속성으로 체크해 중복 실행을 방지합니다. Spring의 `CharacterEncodingFilter`, `SecurityContextFilter` 등이 모두 `OncePerRequestFilter`를 상속합니다.
>
> **Q3.** 가능합니다. `ContentCachingRequestWrapper`는 `InputStream`을 읽을 때 내용을 내부 버퍼에 복사합니다. `HttpServletRequest.getInputStream()`은 표준으로 한 번만 읽을 수 있지만, `ContentCachingRequestWrapper`는 이미 읽은 내용을 `getContentAsByteArray()`로 다시 반환합니다. 단, `@RequestBody` 처리 시 `HttpMessageConverter`가 `request.getInputStream()`을 사용하므로, `ContentCachingRequestWrapper`를 사용해야 `Filter`와 `Controller` 모두 본문을 읽을 수 있습니다.

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: HandlerInterceptor 체인 실행 순서 ➡️](./02-interceptor-chain-order.md)**

</div>
