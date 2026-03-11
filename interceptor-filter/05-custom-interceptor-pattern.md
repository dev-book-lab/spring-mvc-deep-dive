# Custom Interceptor 작성 패턴 — 인증 토큰 검증 실전 예시

---

## 🎯 핵심 질문

- `HandlerInterceptor` 구현 시 세 메서드 중 반드시 구현해야 하는 것은?
- `WebMvcConfigurer.addInterceptors()` 등록과 URL 패턴 제한의 올바른 방법은?
- JWT 토큰 검증 인터셉터에서 `HandlerMethod` 캐스팅이 필요한 이유는?
- Interceptor에서 Controller로 데이터를 전달하는 방법(`request.setAttribute` vs `ThreadLocal`)은?
- `@NoAuth` 같은 커스텀 어노테이션으로 특정 메서드를 Interceptor에서 제외하는 패턴은?
- Interceptor 내에서 Spring Bean을 안전하게 사용하는 방법은?

---

## 🔍 왜 이 메커니즘이 존재하는가

```
공통 관심사를 Interceptor로 분리:
  모든 API 요청에 JWT 토큰 검증 필요
    → 각 Controller에 검증 로직 중복 → 유지보수 어려움
    → Interceptor에 위임 → 단일 지점 관리

  Interceptor 장점:
    HandlerMethod(Controller 메서드 정보) 접근 → 어노테이션 기반 예외 처리
    Spring Bean(JwtTokenProvider 등) 주입 가능
    preHandle false → Controller 실행 차단 + 직접 응답 작성
```

---

## 😱 흔한 오해 또는 실수

### Before: handler를 HandlerMethod로 바로 캐스팅해도 된다

```java
// ❌ NullPointerException / ClassCastException 위험
@Override
public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                          Object handler) {
    HandlerMethod hm = (HandlerMethod) handler;  // 위험!
    // handler가 항상 HandlerMethod가 아님
    // ResourceHttpRequestHandler (정적 파일) → ClassCastException
    // HandlerFunction (WebFlux) → ClassCastException
}

// ✅ 올바른 방법:
@Override
public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                          Object handler) {
    // instanceof 체크 먼저
    if (!(handler instanceof HandlerMethod handlerMethod)) {
        return true;  // HandlerMethod 아닌 경우 (정적 리소스 등) → 통과
    }
    // 이후 handlerMethod 안전하게 사용
}
```

### Before: Interceptor에서 @Autowired로 Bean을 주입하면 null이 된다

```java
// ❌ 잘못된 등록 방식
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new JwtAuthInterceptor());
        // new로 직접 생성 → Spring이 관리하지 않는 객체
        // → @Autowired 필드가 null
    }
}

// ✅ 올바른 방법: @Bean 또는 @Component + @Autowired
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Autowired
    private JwtAuthInterceptor jwtAuthInterceptor;
    // Spring이 관리하는 Bean → @Autowired 필드 정상 주입됨

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(jwtAuthInterceptor)
            .addPathPatterns("/api/**");
    }
}
```

---

## ✨ 올바른 이해와 사용

### After: JWT 인증 Interceptor 전체 구현

```java
// ① 커스텀 어노테이션 — 인증 제외 대상 표시
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface NoAuth {
    // 이 어노테이션이 붙은 메서드/클래스는 인증 없이 접근 가능
}

// ② 사용할 Service/Component
@Component
@RequiredArgsConstructor
public class JwtTokenProvider {
    private final String secret = "...";

    public boolean isValid(String token) { ... }
    public Long getUserId(String token) { ... }
}

// ③ Interceptor 구현
@Component
@RequiredArgsConstructor
public class JwtAuthInterceptor implements HandlerInterceptor {

    private final JwtTokenProvider jwtTokenProvider;
    private final ObjectMapper objectMapper;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                              Object handler) throws Exception {

        // 정적 리소스 등 HandlerMethod가 아닌 경우 통과
        if (!(handler instanceof HandlerMethod handlerMethod)) {
            return true;
        }

        // @NoAuth 체크 — 클래스 또는 메서드 레벨
        if (isNoAuthRequired(handlerMethod)) {
            return true;
        }

        // Authorization 헤더에서 토큰 추출
        String token = resolveToken(request);
        if (token == null) {
            sendUnauthorized(response, "토큰이 없습니다.");
            return false;
        }

        // 토큰 유효성 검증
        if (!jwtTokenProvider.isValid(token)) {
            sendUnauthorized(response, "유효하지 않은 토큰입니다.");
            return false;
        }

        // 토큰에서 userId 추출 → request attribute로 전달 (Controller에서 사용)
        Long userId = jwtTokenProvider.getUserId(token);
        request.setAttribute("authenticatedUserId", userId);
        return true;
    }

    private boolean isNoAuthRequired(HandlerMethod handlerMethod) {
        // 메서드 레벨 먼저 확인
        if (handlerMethod.hasMethodAnnotation(NoAuth.class)) return true;
        // 클래스 레벨 확인
        return AnnotatedElementUtils.hasAnnotation(
            handlerMethod.getBeanType(), NoAuth.class);
    }

    private String resolveToken(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (StringUtils.hasText(bearerToken) && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }

    private void sendUnauthorized(HttpServletResponse response, String message) throws IOException {
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.setContentType("application/json;charset=UTF-8");
        Map<String, String> body = Map.of(
            "error", "UNAUTHORIZED",
            "message", message
        );
        response.getWriter().write(objectMapper.writeValueAsString(body));
    }
}

// ④ 등록
@Configuration
@RequiredArgsConstructor
public class WebConfig implements WebMvcConfigurer {

    private final JwtAuthInterceptor jwtAuthInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(jwtAuthInterceptor)
            .addPathPatterns("/api/**")
            .excludePathPatterns(
                "/api/auth/login",
                "/api/auth/refresh",
                "/api/public/**",
                "/api/health"
            );
    }
}

// ⑤ Controller에서 사용
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/me")
    public UserDto getMyProfile(HttpServletRequest request) {
        Long userId = (Long) request.getAttribute("authenticatedUserId");
        return userService.findById(userId);
    }

    @NoAuth
    @GetMapping("/public-info")
    public String publicInfo() {
        return "누구나 접근 가능";
    }
}
```

---

## 🔬 내부 동작 원리

### 1. request.setAttribute vs ThreadLocal — 데이터 전달 방법 비교

```java
// 방법 1: request.setAttribute (권장)
// preHandle에서 설정
request.setAttribute("authenticatedUserId", userId);
// Controller에서 읽기
Long userId = (Long) request.getAttribute("authenticatedUserId");
// 비동기 요청에도 안전 (request 객체는 비동기에도 유지)
// afterCompletion에서 수동 정리 불필요 (request 소멸 시 자동 정리)

// 방법 2: ThreadLocal (주의 필요)
// preHandle에서 설정
UserContext.set(new UserContext(userId));  // ThreadLocal
// Controller에서 읽기
UserContext.get();
// afterCompletion에서 반드시 정리!
@Override
public void afterCompletion(...) {
    UserContext.clear();  // ← 필수! 스레드 재사용 시 데이터 누수 방지
}
// 비동기(@Async, DeferredResult) 환경에서는 다른 스레드로 전파 안 됨

// ✅ 권장: request.setAttribute
// ThreadLocal은 비동기나 스레드 풀 재사용 환경에서 위험
```

### 2. @CurrentUser 커스텀 ArgumentResolver 연동

```java
// Interceptor에서 userId만 전달 → ArgumentResolver에서 User 객체로 변환
// → Controller 파라미터: @CurrentUser User user

@Component
@RequiredArgsConstructor
public class CurrentUserArgumentResolver implements HandlerMethodArgumentResolver {

    private final UserRepository userRepository;

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(CurrentUser.class)
            && parameter.getParameterType().equals(User.class);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, ..., NativeWebRequest webRequest) {
        HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
        Long userId = (Long) request.getAttribute("authenticatedUserId");
        if (userId == null) throw new IllegalStateException("인증 정보 없음");
        return userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
    }
}

// Controller:
@GetMapping("/me")
public UserDto getMyProfile(@CurrentUser User user) {
    return UserDto.from(user);
}
```

### 3. URL 패턴 전략

```java
@Override
public void addInterceptors(InterceptorRegistry registry) {
    // 패턴 1: API 전체 + 특정 경로 제외
    registry.addInterceptor(jwtAuthInterceptor)
        .addPathPatterns("/api/**")
        .excludePathPatterns("/api/auth/**", "/api/public/**");

    // 패턴 2: 복수 경로 적용
    registry.addInterceptor(rateLimitInterceptor)
        .addPathPatterns("/api/**", "/admin/**")
        .excludePathPatterns("/api/health", "/admin/status");

    // 패턴 3: 전체 적용 (로깅, 메트릭)
    registry.addInterceptor(loggingInterceptor)
        .addPathPatterns("/**")
        .excludePathPatterns(
            "/actuator/**",    // Spring Actuator 제외
            "/error",          // 에러 페이지 제외
            "/**/*.html",      // 정적 HTML 제외
            "/**/*.js",
            "/**/*.css"
        );
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 인증 헤더 없는 요청

```bash
curl -i http://localhost:8080/api/users/me
# HTTP/1.1 401 Unauthorized
# Content-Type: application/json;charset=UTF-8
# {"error":"UNAUTHORIZED","message":"토큰이 없습니다."}
```

### 실험 2: @NoAuth 적용 확인

```bash
# 토큰 없이도 접근 가능
curl http://localhost:8080/api/users/public-info
# HTTP/1.1 200 OK
# "누구나 접근 가능"
```

### 실험 3: MockMvc로 Interceptor 단위 테스트

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    MockMvc mockMvc;

    @MockBean
    JwtTokenProvider jwtTokenProvider;

    @Test
    void 인증_토큰_없으면_401() throws Exception {
        mockMvc.perform(get("/api/users/me"))
            .andExpect(status().isUnauthorized())
            .andExpect(jsonPath("$.error").value("UNAUTHORIZED"));
    }

    @Test
    void 유효한_토큰이면_200() throws Exception {
        when(jwtTokenProvider.isValid("valid-token")).thenReturn(true);
        when(jwtTokenProvider.getUserId("valid-token")).thenReturn(1L);

        mockMvc.perform(get("/api/users/me")
                .header("Authorization", "Bearer valid-token"))
            .andExpect(status().isOk());
    }
}
```

---

## 🌐 HTTP 레벨 분석

```
GET /api/users/me HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...

JwtAuthInterceptor.preHandle():
  handler = HandlerMethod(UserController.getMyProfile)
  isNoAuthRequired() → @NoAuth 없음 → false
  resolveToken() → "eyJhbGciOiJIUzI1NiJ9..."
  jwtTokenProvider.isValid() → true
  jwtTokenProvider.getUserId() → 42L
  request.setAttribute("authenticatedUserId", 42L)
  return true → Controller 실행

UserController.getMyProfile():
  userId = (Long) request.getAttribute("authenticatedUserId")  // 42
  userService.findById(42) → User{id=42, name="홍길동"}
  → ResponseEntity.ok(UserDto.from(user))

HTTP/1.1 200 OK
Content-Type: application/json
{"id":42,"name":"홍길동","email":"hong@example.com"}
```

---

## 📌 핵심 정리

```
handler instanceof HandlerMethod 체크 필수
  정적 리소스, ResourceHttpRequestHandler 등도 handler로 전달됨
  → 바로 캐스팅하면 ClassCastException

Bean 주입
  @Component + @Autowired (또는 @RequiredArgsConstructor)
  addInterceptors()에서 new로 생성하면 주입 불가

@NoAuth 어노테이션 패턴
  handlerMethod.hasMethodAnnotation(NoAuth.class)
  AnnotatedElementUtils.hasAnnotation(BeanType, NoAuth.class)
  → 클래스/메서드 레벨 모두 체크

데이터 전달
  request.setAttribute() → 비동기 안전, 자동 정리 ✅
  ThreadLocal → afterCompletion에서 수동 정리 필요 ⚠️

preHandle false 시
  response.setStatus() + response.getWriter().write() 직접 응답
  objectMapper.writeValueAsString() → JSON 응답 작성
```

---

## 🤔 생각해볼 문제

**Q1.** `preHandle`에서 `request.setAttribute("userId", 42L)` 설정 후 `@Async` 메서드를 호출하면 다른 스레드에서 `request.getAttribute("userId")`로 값을 읽을 수 있는가?

**Q2.** `JwtAuthInterceptor`를 `@Component`로 등록했는데 순환 참조(circular dependency)가 발생하는 경우 어떻게 해결하는가?

**Q3.** 여러 Interceptor가 각각 `request.setAttribute()`로 데이터를 저장할 때 키 충돌을 방지하는 좋은 방법은?

> 💡 **해설**
>
> **Q1.** 읽을 수 없습니다. `HttpServletRequest`는 단일 HTTP 요청-응답 사이클에 묶여 있으며, 스레드 간 공유되지 않습니다. `@Async` 메서드는 별도 스레드에서 실행되므로 원본 request 객체에 접근할 수 없습니다. 비동기 환경에서 인증 정보를 전달하려면 `SecurityContextHolder`의 전파 설정(`InheritableThreadLocal` 사용) 또는 파라미터로 직접 전달해야 합니다.
>
> **Q2.** `@Lazy` 어노테이션을 순환 참조 관계의 의존성 주입에 사용하거나, `ApplicationContext`에서 직접 Bean을 가져오는 방식(`ApplicationContextAware`)으로 해결할 수 있습니다. 더 나은 방법은 설계를 재검토해 Interceptor가 너무 많은 의존성을 가지고 있는 것이 아닌지 확인하고, 공통 기능을 별도 서비스로 추출하는 것입니다.
>
> **Q3.** 각 Interceptor 클래스명을 접두사로 사용하는 상수를 정의하는 것이 안전합니다. 예: `public static final String USER_ID_ATTR = JwtAuthInterceptor.class.getName() + ".userId"`. 또는 전용 요청 컨텍스트 객체를 하나의 키로 저장하고 Interceptor마다 해당 객체에 필드를 추가하는 패턴도 사용됩니다. 이렇게 하면 키 충돌 없이 여러 Interceptor의 데이터를 관리할 수 있습니다.

---

<div align="center">

**[⬅️ 이전: CORS 처리](./04-cors-handling.md)** | **[홈으로 🏠](../README.md)** | **[다음: Async Request와 Interceptor ➡️](./06-async-interceptor.md)**

</div>
