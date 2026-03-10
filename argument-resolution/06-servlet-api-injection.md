# Servlet API 주입 — HttpServletRequest부터 Principal까지

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `ServletRequestMethodArgumentResolver`가 지원하는 타입 목록과 각 타입의 주입 방법은?
- `HttpSession`을 주입받으면 세션이 없을 때 자동으로 생성되는가?
- `Principal`을 파라미터로 받으면 Spring Security의 `Authentication`과 어떻게 연결되는가?
- `InputStream` / `Reader`를 주입받으면 `@RequestBody`와 어떻게 다른가?
- `Locale`, `TimeZone`은 어디서 결정되어 주입되는가?
- `WebRequest`와 `NativeWebRequest`의 차이는 무엇인가?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: Servlet API 객체에 직접 접근하고 싶지만 테스트하기 어렵다

```java
// Bad: HttpServletRequest 직접 의존
@GetMapping("/info")
public String info(HttpServletRequest request) {
    String ip = request.getRemoteAddr();
    String agent = request.getHeader("User-Agent");
    HttpSession session = request.getSession();
    // → 테스트 시 HttpServletRequest Mock 필요
    // → 서블릿 API에 강하게 결합
}

// Good: 필요한 것만 추출
@GetMapping("/info")
public String info(
    HttpServletRequest request,   // 전체 접근이 필요할 때
    HttpSession session,          // 세션만 필요할 때
    Principal principal,          // 인증 정보만 필요할 때
    Locale locale,                // 로케일만 필요할 때
    WebRequest webRequest         // Spring 추상화된 접근이 필요할 때
) { ... }

해결:
  ServletRequestMethodArgumentResolver가 타입을 보고 적합한 객체를 꺼내줌
  → 필요한 것만 선언 → 관심사 명확
  → WebRequest 사용 시 서블릿 API 직접 의존 없애기 가능
```

---

## 😱 흔한 오해 또는 실수

### Before: HttpSession을 주입받으면 항상 세션이 있다

```java
// ❌ 잘못된 이해
// "HttpSession session 파라미터가 있으면 항상 세션 객체가 주입된다"

@GetMapping("/info")
public String info(HttpSession session) {
    // session이 null인 경우는 없는가?
}

// ✅ 실제:
// HttpSession 파라미터는 request.getSession(true) 호출
// → 세션 없으면 새로 생성해서 반환
// → 항상 non-null (단, 서버 설정에 따라 다를 수 있음)

// 세션을 생성하고 싶지 않으면:
@GetMapping("/info")
public String info(HttpServletRequest request) {
    HttpSession session = request.getSession(false);
    // → 세션 없으면 null 반환 (새로 생성 안 함)
}
```

### Before: Principal은 Spring Security와 무관하다

```
❌ 잘못된 이해:
  "Principal 파라미터는 서블릿 컨테이너 레벨의 보안 정보다"

✅ 실제:
  Spring Security가 있을 때:
    SecurityContextHolder의 Authentication이 Principal을 구현
    → principal.getName() = Authentication.getName() = 사용자 이름
    → (Authentication) principal 캐스팅 가능

  Spring Security 없을 때:
    서블릿 컨테이너(Tomcat) 레벨 보안 또는 null
    → request.getUserPrincipal() 반환

  Principal 주입 처리:
    ServletRequestMethodArgumentResolver
    → request.getUserPrincipal()
    → Spring Security가 SecurityContextHolderAwareRequestWrapper로 request를 래핑
       → getUserPrincipal() → SecurityContextHolder에서 Authentication 반환
```

---

## ✨ 올바른 이해와 사용

### After: 지원 타입 전체 목록과 주입 방법

```java
// ServletRequestMethodArgumentResolver.supportsParameter() — 지원 타입들:
WebRequest                  // Spring의 웹 요청 추상화 (서블릿 API 비의존)
NativeWebRequest            // WebRequest + getNativeRequest() 접근
ServletRequest              // HttpServletRequest의 상위 인터페이스
MultipartRequest            // 파일 업로드 요청
HttpSession                 // request.getSession(true)
PushBuilder                 // HTTP/2 Push 빌더
Principal                   // request.getUserPrincipal()
InputStream                 // request.getInputStream()
Reader                      // request.getReader()
HttpMethod                  // HttpMethod.resolve(request.getMethod())
Locale                      // LocaleContextHolder 또는 LocaleResolver에서
TimeZone                    // LocaleContextHolder에서
ZoneId                      // LocaleContextHolder에서

// 각각 어떻게 꺼내는지:
@Override
@Nullable
public Object resolveArgument(MethodParameter parameter, ...) throws Exception {
    Class<?> paramType = parameter.getParameterType();
    HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);

    if (WebRequest.class.isAssignableFrom(paramType)) {
        return webRequest;
    } else if (ServletRequest.class.isAssignableFrom(paramType)
               || MultipartRequest.class.isAssignableFrom(paramType)) {
        return request;
    } else if (HttpSession.class.isAssignableFrom(paramType)) {
        return request.getSession();  // getSession(true) → 없으면 생성
    } else if (PushBuilder.class == paramType) {
        return resolvePushBuilder(request, paramType);
    } else if (Principal.class.isAssignableFrom(paramType)) {
        Principal userPrincipal = request.getUserPrincipal();
        if (userPrincipal != null && !paramType.isInstance(userPrincipal)) {
            throw new IllegalStateException("...");
        }
        return userPrincipal;
    } else if (InputStream.class.isAssignableFrom(paramType)) {
        return request.getInputStream();
    } else if (Reader.class.isAssignableFrom(paramType)) {
        return request.getReader();
    } else if (HttpMethod.class == paramType) {
        return HttpMethod.resolve(request.getMethod());
    } else if (Locale.class == paramType) {
        return RequestContextUtils.getLocale(request);
        // → LocaleResolver (AcceptHeaderLocaleResolver 등) 사용
    } else if (TimeZone.class == paramType) {
        TimeZone timeZone = RequestContextUtils.getTimeZone(request);
        return (timeZone != null ? timeZone : TimeZone.getDefault());
    } else if (ZoneId.class == paramType) {
        TimeZone timeZone = RequestContextUtils.getTimeZone(request);
        return (timeZone != null ? timeZone.toZoneId() : ZoneId.systemDefault());
    }
    // ...
}
```

### 2. WebRequest vs NativeWebRequest vs HttpServletRequest

```java
// WebRequest — 서블릿 API 추상화 (테스트 용이)
@GetMapping("/info")
public String info(WebRequest webRequest) {
    String name = webRequest.getParameter("name");      // getParameter()
    String header = webRequest.getHeader("X-Custom");   // getHeader()
    // 서블릿 API 미의존 → MockWebRequest로 테스트 가능
}

// NativeWebRequest — WebRequest + 네이티브 객체 접근
@GetMapping("/info")
public String info(NativeWebRequest webRequest) {
    HttpServletRequest req = webRequest.getNativeRequest(HttpServletRequest.class);
    // 필요할 때만 네이티브 객체 접근
}

// HttpServletRequest — 서블릿 API 직접 접근 (가장 강력하지만 의존성 높음)
@GetMapping("/info")
public String info(HttpServletRequest request) {
    String ip = request.getRemoteAddr();    // 클라이언트 IP
    String contextPath = request.getContextPath();
    Enumeration<String> headers = request.getHeaderNames();
    // ... 모든 서블릿 API 직접 사용
}
```

### 3. InputStream 주입 — @RequestBody와의 차이

```java
// InputStream 주입 — 원시 바이트 스트림 직접 접근
@PostMapping("/raw")
public String raw(InputStream body) throws IOException {
    // request.getInputStream() 직접
    byte[] bytes = body.readAllBytes();
    return "받은 바이트: " + bytes.length;
    // 타입 변환 없음, HttpMessageConverter 없음
    // 본문을 직접 파싱해야 함
}

// @RequestBody — HttpMessageConverter 체인 거침
@PostMapping("/parsed")
public String parsed(@RequestBody UserDto dto) {
    // Jackson이 JSON → UserDto 자동 변환
    return dto.getName();
}

// 주의: InputStream은 한 번만 읽을 수 있음
// → @RequestBody와 InputStream을 같은 핸들러에서 함께 쓰면
//    둘 중 나중에 읽는 것은 빈 스트림을 받음
```

### 4. Locale 주입 — LocaleResolver와 연결

```java
// Locale 주입 처리:
// RequestContextUtils.getLocale(request)
//   → DispatcherServlet이 요청 시 설정한 LocaleResolver 사용

// Spring Boot 기본: AcceptHeaderLocaleResolver
// → Accept-Language: ko-KR,ko;q=0.9,en;q=0.8
// → Locale: ko_KR

// 커스텀 LocaleResolver 설정:
@Bean
public LocaleResolver localeResolver() {
    SessionLocaleResolver resolver = new SessionLocaleResolver();
    resolver.setDefaultLocale(Locale.KOREAN);
    return resolver;
    // → 세션에 저장된 Locale 사용
    // → LocaleChangeInterceptor와 함께 언어 변경 기능 구현 가능
}

@GetMapping("/greet")
public String greet(Locale locale, MessageSource messageSource) {
    return messageSource.getMessage("greeting", null, locale);
    // locale = ko_KR → "안녕하세요"
    // locale = en_US → "Hello"
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 각 타입별 주입값 확인

```java
@GetMapping("/all-types")
public Map<String, Object> allTypes(
    HttpServletRequest request,
    HttpSession session,
    Principal principal,
    Locale locale,
    HttpMethod method,
    WebRequest webRequest) {

    return Map.of(
        "remoteAddr",   request.getRemoteAddr(),
        "sessionId",    session.getId(),
        "principal",    principal != null ? principal.getName() : "anonymous",
        "locale",       locale.toString(),
        "method",       method.name(),
        "webRequest",   webRequest.getClass().getSimpleName()
    );
}
```

```bash
curl -H "Accept-Language: ko-KR" http://localhost:8080/all-types
# {
#   "remoteAddr": "127.0.0.1",
#   "sessionId": "ABC123...",
#   "principal": "anonymous",   (Spring Security 없을 때)
#   "locale": "ko_KR",
#   "method": "GET",
#   "webRequest": "ServletWebRequest"
# }
```

### 실험 2: Spring Security + Principal

```java
// Spring Security 설정 후
@GetMapping("/me")
public Map<String, Object> me(Principal principal) {
    Authentication auth = (Authentication) principal;  // 캐스팅 가능
    return Map.of(
        "name",        auth.getName(),
        "roles",       auth.getAuthorities().toString(),
        "class",       principal.getClass().getSimpleName()
        // "UsernamePasswordAuthenticationToken" 또는 커스텀 타입
    );
}
```

### 실험 3: HttpSession 생성 여부 제어

```java
@GetMapping("/session-test")
public Map<String, Object> sessionTest(
    HttpServletRequest request,
    HttpSession session) {  // ← 세션 생성됨

    HttpSession optionalSession = request.getSession(false);  // ← 이미 생성돼 있으면 반환, 없으면 null

    return Map.of(
        "sessionId",             session.getId(),       // 항상 있음
        "optionalSessionExists", optionalSession != null
    );
}
```

---

## 🌐 HTTP 레벨 분석

```
GET /info HTTP/1.1
Host: localhost:8080
Accept-Language: ko-KR,ko;q=0.9
Cookie: JSESSIONID=ABC123
Authorization: Bearer eyJ...

파라미터 주입 매핑:
  HttpServletRequest → Tomcat이 생성한 request 객체 그대로
  HttpSession       → JSESSIONID=ABC123 쿠키로 세션 조회 → 없으면 새 세션
  Principal         → Authorization 헤더 → Spring Security가 Authentication 설정
                      → request.getUserPrincipal() → Authentication 객체
  Locale            → Accept-Language: ko-KR → LocaleResolver → Locale.KOREAN
  HttpMethod        → "GET" → HttpMethod.GET 열거형

Spring Security 처리 흐름:
  FilterChainProxy (Security Filter Chain)
  → AuthenticationFilter: Bearer 토큰 검증
  → SecurityContextHolder.getContext().setAuthentication(auth)
  → SecurityContextHolderAwareRequestWrapper로 request 래핑
  → getUserPrincipal() → SecurityContextHolder에서 Authentication 반환

  Controller에서 Principal 파라미터
  → ServletRequestMethodArgumentResolver.resolveArgument()
  → request.getUserPrincipal() → 래핑된 request에서 Authentication 반환
```

---

## 🤔 트레이드오프

```
HttpServletRequest 직접 주입 vs 추상화 타입:
  HttpServletRequest:
    + 모든 서블릿 API 접근 (커스텀 헤더, 원시 데이터 등)
    - 서블릿 의존 → 단위 테스트 시 Mock 필요
    - 컨테이너 전환(서블릿 → 반응형) 시 코드 변경 필요

  WebRequest:
    + 서블릿 API 비의존 → MockWebRequest로 테스트 가능
    + WebFlux 등 다른 런타임에서도 동작
    - 서블릿 전용 기능(멀티파트, 세션 세부 제어) 접근 어려움

Principal vs Authentication 직접 주입:
  Principal:   표준 자바 타입, 서블릿 API 레벨
  Authentication: Spring Security 전용, 더 풍부한 정보
  → 테스트 편의성 + 추상화 → Principal
  → Role/Authority 확인 등 Security 기능 필요 → Authentication

HttpSession 직접 주입의 위험:
  세션 자동 생성 → 예상치 못한 메모리 사용
  → getSession(false) + null 체크가 더 안전
  → 또는 @SessionAttribute 사용
```

---

## 📌 핵심 정리

```
ServletRequestMethodArgumentResolver 지원 타입
  WebRequest, NativeWebRequest, ServletRequest, MultipartRequest
  HttpSession       → request.getSession() (없으면 생성)
  Principal         → request.getUserPrincipal() (Spring Security와 연동)
  InputStream/Reader → request.getInputStream()/getReader()
  HttpMethod        → HttpMethod.resolve(request.getMethod())
  Locale            → LocaleContextHolder / LocaleResolver
  TimeZone, ZoneId  → LocaleContextHolder

Principal ← Spring Security 연결
  Spring Security가 SecurityContextHolderAwareRequestWrapper로 래핑
  → getUserPrincipal() → SecurityContextHolder → Authentication

HttpSession 생성 제어
  HttpSession 파라미터 → getSession(true) → 없으면 생성
  getSession(false) → 없으면 null (생성 안 함)

InputStream 주의
  한 번 읽으면 다시 읽기 불가
  @RequestBody와 동시에 사용 불가
  → 원시 바이트 처리 필요 시에만 사용

Locale 결정
  RequestContextUtils.getLocale(request)
  → DispatcherServlet이 요청 시 LocaleResolver로 결정해 설정
```

---

## 🤔 생각해볼 문제

**Q1.** `HttpServletResponse`를 파라미터로 주입받아 응답에 직접 쓰면 (예: `response.getWriter().write("hello")`), 이후 컨트롤러가 반환값을 갖고 있으면 어떻게 되는가?

**Q2.** `Principal`을 파라미터로 선언한 컨트롤러를 단위 테스트할 때 `MockMvc`를 어떻게 구성해야 하는가?

**Q3.** `HttpSession session` 파라미터가 있는 컨트롤러는 로드밸런서 환경에서 어떤 문제를 일으킬 수 있는가? 어떻게 해결하는가?

> 💡 **해설**
>
> **Q1.** `response.getWriter().write("hello")`로 직접 응답을 쓰면, 이후 컨트롤러 반환값이 있어도 Spring MVC의 `ViewResolver`나 `MessageConverter`가 응답을 다시 쓰려고 할 때 이미 커밋된 응답이므로 `IllegalStateException: Cannot call sendError() after the response has been committed`가 발생합니다. 반환값 없이 `void`를 반환하면 Spring MVC는 응답이 이미 처리됐다고 판단(`mavContainer.setRequestHandled(true)`)하고 View 렌더링을 생략합니다. 직접 응답을 쓰려면 반환 타입을 `void`로 선언하거나 `ResponseEntity`로 처리하는 것이 안전합니다.
>
> **Q2.** `MockMvc`에서 `Principal`을 주입하는 방법: `MockMvcRequestBuilders.get("/me").principal(new UsernamePasswordAuthenticationToken("user", null, authorities))`으로 Principal을 명시적으로 지정할 수 있습니다. Spring Security Test를 사용한다면 `@WithMockUser(username = "user", roles = "USER")` 어노테이션으로 더 간단하게 설정 가능합니다. Spring Security가 없다면 `MockHttpServletRequest`에 직접 `Principal` 구현체를 설정합니다.
>
> **Q3.** `HttpSession`은 기본적으로 WAS 메모리에 저장됩니다. 로드밸런서가 요청을 다른 서버로 분배하면 해당 서버에 세션이 없어 `null`이 반환되거나 새 세션이 생성됩니다 — 이를 세션 불일치(Session Affinity 문제)라고 합니다. 해결 방법: (1) **Sticky Session**: 로드밸런서가 같은 사용자는 항상 같은 서버로 보내도록 설정 — 단순하지만 특정 서버 과부하 가능. (2) **Session Clustering**: Hazelcast, Redis 등을 사용해 세션을 외부 저장소에 공유 (`spring-session-data-redis`). (3) **JWT 등 Stateless 인증**: 세션 자체를 없애고 토큰 기반 인증으로 전환 — 가장 현대적인 방법.

---

<div align="center">

**[⬅️ 이전: Validation — @Valid와 BindingResult](./05-validation-binding-result.md)** | **[홈으로 🏠](../README.md)** | **[다음: Custom Argument Resolver 작성 ➡️](./07-custom-argument-resolver.md)**

</div>
