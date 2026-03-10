# Custom Argument Resolver 작성 — 커스텀 어노테이션으로 인증 정보 주입하기

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `HandlerMethodArgumentResolver` 인터페이스의 두 메서드(`supportsParameter`, `resolveArgument`)는 각각 어떤 책임을 갖는가?
- `WebMvcConfigurer.addArgumentResolvers()`로 등록한 커스텀 Resolver는 기본 Resolver와 어떤 순서로 실행되는가?
- 기본 Resolver보다 먼저 실행되어야 하는 경우 어떻게 해야 하는가?
- 커스텀 Resolver에서 `WebDataBinderFactory`를 어떻게 활용하는가?
- `@CurrentUser` 같은 커스텀 어노테이션 기반 Resolver의 전체 구현 패턴은?
- 커스텀 Resolver를 테스트하는 방법은?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: 인증된 사용자 정보를 매 컨트롤러 메서드에서 반복 조회한다

```java
// Bad: 매 메서드마다 Security Context에서 유저 정보 추출
@GetMapping("/profile")
public User profile() {
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    String username = auth.getName();
    User user = userService.findByUsername(username);  // DB 조회
    return user;
}

@GetMapping("/orders")
public List<Order> orders() {
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    String username = auth.getName();
    User user = userService.findByUsername(username);  // 또 조회
    return orderService.findByUser(user);
}

// 문제:
// SecurityContextHolder 직접 접근 → 테스트 어려움
// userService.findByUsername() 반복 → 중복
// 인증 로직이 비즈니스 로직에 섞임

// Good: @CurrentUser 커스텀 Resolver
@GetMapping("/profile")
public User profile(@CurrentUser User user) {
    return user;  // Resolver가 User 주입
}

@GetMapping("/orders")
public List<Order> orders(@CurrentUser User user) {
    return orderService.findByUser(user);
}
```

---

## 😱 흔한 오해 또는 실수

### Before: addArgumentResolvers()로 추가하면 @RequestParam보다 먼저 실행된다

```java
// ❌ 잘못된 이해
@Override
public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
    resolvers.add(new MyResolver());
    // "MyResolver가 가장 먼저 실행된다"
}

// ✅ 실제:
// getDefaultArgumentResolvers() 의 끝에 추가됨
// 기본 30개+ Resolver 뒤에서 실행
// → 기본 Resolver가 처리 가능한 파라미터라면 MyResolver에 도달하지 못함
// → @RequestParam String과 같은 파라미터는 기본 Resolver가 먼저 잡음

// 해결: RequestMappingHandlerAdapter.setArgumentResolvers()로 전체 목록 직접 관리
// 또는 커스텀 어노테이션을 사용해 기본 Resolver와 충돌 방지
```

### Before: resolveArgument()는 항상 호출된다

```
❌ 잘못된 이해:
  "supportsParameter()와 resolveArgument() 모두 매 요청마다 호출된다"

✅ 실제:
  supportsParameter() → 첫 요청 시 한 번 탐색 → 캐시 저장
  이후 동일 MethodParameter → 캐시에서 Resolver 직접 반환 (supportsParameter 재호출 없음)

  resolveArgument() → 매 요청마다 호출 (캐시 없음)
  → 여기서 비용이 큰 DB 조회 주의
  → 요청 scope 캐시(request attribute) 활용 권장
```

---

## ✨ 올바른 이해와 사용

### After: 커스텀 Resolver 전체 구현 — @CurrentUser 예시

```java
// ① 커스텀 어노테이션 정의
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface CurrentUser {
    boolean required() default true;
}

// ② HandlerMethodArgumentResolver 구현
@Component
@RequiredArgsConstructor
public class CurrentUserArgumentResolver implements HandlerMethodArgumentResolver {

    private final UserService userService;

    // supportsParameter: 이 Resolver가 처리할 파라미터 조건
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(CurrentUser.class)
            && User.class.isAssignableFrom(parameter.getParameterType());
        // @CurrentUser 어노테이션 + User 타입 파라미터만 처리
    }

    // resolveArgument: 실제 값 생성 로직
    @Override
    @Nullable
    public Object resolveArgument(MethodParameter parameter,
            ModelAndViewContainer mavContainer,
            NativeWebRequest webRequest,
            WebDataBinderFactory binderFactory) throws Exception {

        // 요청 속성 캐시 확인 (같은 요청에서 두 번 호출 시 DB 재조회 방지)
        HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
        User cached = (User) request.getAttribute("CURRENT_USER");
        if (cached != null) return cached;

        // Spring Security Authentication에서 사용자 이름 추출
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null || !auth.isAuthenticated()
                || "anonymousUser".equals(auth.getPrincipal())) {
            CurrentUser annotation = parameter.getParameterAnnotation(CurrentUser.class);
            if (annotation != null && annotation.required()) {
                throw new AuthenticationCredentialsNotFoundException("인증이 필요합니다");
            }
            return null;
        }

        // DB에서 User 조회
        String username = auth.getName();
        User user = userService.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException(username));

        // 요청 속성에 캐시 저장
        request.setAttribute("CURRENT_USER", user);

        return user;
    }
}

// ③ WebMvcConfigurer에 등록
@Configuration
@RequiredArgsConstructor
public class WebConfig implements WebMvcConfigurer {

    private final CurrentUserArgumentResolver currentUserArgumentResolver;

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(currentUserArgumentResolver);
    }
}

// ④ 컨트롤러에서 사용
@RestController
@RequiredArgsConstructor
public class UserController {

    @GetMapping("/profile")
    public UserResponse profile(@CurrentUser User user) {
        return UserResponse.from(user);
    }

    @GetMapping("/orders")
    public List<OrderResponse> orders(@CurrentUser User user) {
        return orderService.findByUser(user).stream()
            .map(OrderResponse::from)
            .toList();
    }

    // required=false: 비로그인도 허용
    @GetMapping("/public")
    public String publicPage(@CurrentUser(required = false) User user) {
        if (user != null) return "안녕하세요 " + user.getName();
        return "게스트님 환영합니다";
    }
}
```

---

## 🔬 내부 동작 원리

### 1. HandlerMethodArgumentResolver 인터페이스

```java
public interface HandlerMethodArgumentResolver {

    // 이 Resolver가 해당 파라미터를 처리할 수 있는지 여부
    // - 캐시 키로 사용되므로 빠르고 결정론적이어야 함
    // - MethodParameter 정보: 어노테이션, 타입, 메서드, 인덱스 포함
    boolean supportsParameter(MethodParameter parameter);

    // 실제 파라미터 값 생성
    // - 매 요청마다 호출됨 (캐시 없음)
    // - null 반환 가능 (파라미터에 null 주입)
    // - 예외 발생 시 Spring MVC가 적절한 응답 처리
    @Nullable
    Object resolveArgument(MethodParameter parameter,
                           @Nullable ModelAndViewContainer mavContainer,
                           NativeWebRequest webRequest,
                           @Nullable WebDataBinderFactory binderFactory)
                           throws Exception;
}
```

### 2. setArgumentResolvers()로 우선순위 제어

```java
// 기본 Resolver보다 먼저 실행해야 하는 경우
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Autowired
    private RequestMappingHandlerAdapter adapter;

    @PostConstruct
    public void addCustomResolversFirst() {
        List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>();
        // 커스텀 Resolver 먼저 추가
        resolvers.add(new HighPriorityCustomResolver());
        // 기존 기본 Resolver 목록 뒤에 추가
        List<HandlerMethodArgumentResolver> existing = adapter.getArgumentResolvers();
        if (existing != null) resolvers.addAll(existing);
        adapter.setArgumentResolvers(resolvers);
    }
}
```

### 3. WebDataBinderFactory 활용 — 타입 변환 포함

```java
// resolveArgument()에서 WebDataBinderFactory를 활용한 타입 변환
public class TenantIdArgumentResolver implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(TenantId.class);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter,
            ModelAndViewContainer mavContainer,
            NativeWebRequest webRequest,
            WebDataBinderFactory binderFactory) throws Exception {

        String raw = webRequest.getHeader("X-Tenant-Id");
        if (raw == null) return null;

        // WebDataBinderFactory로 타입 변환
        if (binderFactory != null) {
            WebDataBinder binder = binderFactory.createBinder(webRequest, null, "tenantId");
            return binder.convertIfNecessary(raw, parameter.getParameterType());
            // String → Long, UUID 등 자동 변환
        }
        return raw;
    }
}
```

### 4. MethodParameter 활용 — 파라미터 정보 추출

```java
// MethodParameter에서 꺼낼 수 있는 정보들
public boolean supportsParameter(MethodParameter parameter) {
    // 어노테이션 확인
    CurrentUser ann = parameter.getParameterAnnotation(CurrentUser.class);
    // 파라미터 타입
    Class<?> type = parameter.getParameterType();
    // 제네릭 타입 (Optional<User>의 경우)
    Type genericType = parameter.getGenericParameterType();
    // Optional 언래핑
    MethodParameter nested = parameter.nestedIfOptional();
    Class<?> actualType = nested.getNestedParameterType();
    // 메서드 정보
    Method method = parameter.getMethod();
    // 클래스 레벨 어노테이션
    RequiresAuth classAnn = parameter.getDeclaringClass().getAnnotation(RequiresAuth.class);

    return ann != null && User.class.isAssignableFrom(actualType);
}
```

---

## 💻 실험으로 확인하기

### 실험 1: @CurrentUser Resolver MockMvc 테스트

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired MockMvc mockMvc;

    @MockBean UserService userService;

    @Test
    void profile_authenticated() throws Exception {
        // 커스텀 Resolver를 MockMvc에 등록
        User mockUser = new User(1L, "홍길동");
        when(userService.findByUsername("hong")).thenReturn(Optional.of(mockUser));

        mockMvc.perform(get("/profile")
            .with(user("hong").roles("USER")))  // Spring Security Test
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("홍길동"));
    }

    @Test
    void profile_unauthenticated() throws Exception {
        mockMvc.perform(get("/profile"))
            .andExpect(status().isUnauthorized());
    }
}
```

### 실험 2: Resolver 직접 단위 테스트

```java
class CurrentUserArgumentResolverTest {

    CurrentUserArgumentResolver resolver;
    UserService userService = mock(UserService.class);

    @BeforeEach
    void setUp() {
        resolver = new CurrentUserArgumentResolver(userService);
    }

    @Test
    void supportsParameter_true_when_currentUser_annotation_and_user_type() throws Exception {
        Method method = UserController.class.getMethod("profile", User.class);
        MethodParameter param = new MethodParameter(method, 0);
        assertThat(resolver.supportsParameter(param)).isTrue();
    }

    @Test
    void resolveArgument_returns_user_when_authenticated() throws Exception {
        User expected = new User(1L, "홍길동");
        when(userService.findByUsername("hong")).thenReturn(Optional.of(expected));

        SecurityContextHolder.getContext().setAuthentication(
            new UsernamePasswordAuthenticationToken("hong", null, List.of()));

        MockHttpServletRequest request = new MockHttpServletRequest();
        NativeWebRequest webRequest = new ServletWebRequest(request);
        Method method = UserController.class.getMethod("profile", User.class);
        MethodParameter param = new MethodParameter(method, 0);

        User result = (User) resolver.resolveArgument(param, null, webRequest, null);
        assertThat(result.getName()).isEqualTo("홍길동");
    }
}
```

### 실험 3: 요청 속성 캐시 효과 측정

```java
// resolveArgument() 안에서 DB 호출 횟수 측정
@GetMapping("/multi-user")
public Map<String, Object> multiUser(
    @CurrentUser User user1,   // 첫 번째 @CurrentUser
    @CurrentUser User user2) { // 두 번째 @CurrentUser

    // request attribute 캐시가 있으면 userService.findByUsername() 1번만 호출
    // 캐시 없으면 2번 호출
    return Map.of("user1", user1.getName(), "user2", user2.getName());
}
```

---

## 🌐 HTTP 레벨 분석

```
GET /profile HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...

처리 흐름:
  ① Spring Security Filter:
     JWT 토큰 파싱 → "hong" → UsernamePasswordAuthenticationToken
     SecurityContextHolder.getContext().setAuthentication(auth)

  ② DispatcherServlet → Controller 메서드 파라미터 resolve:
     @CurrentUser User user 파라미터
     → HandlerMethodArgumentResolverComposite:
        기본 Resolver 30개 → 모두 supportsParameter=false
        CurrentUserArgumentResolver → supportsParameter=true ✅
     → resolveArgument():
        SecurityContextHolder → auth.getName() = "hong"
        userService.findByUsername("hong") → User{id=1, name="홍길동"}
        request.setAttribute("CURRENT_USER", user)  캐시 저장

  ③ UserController.profile(User{id=1, name="홍길동"}) 실행

HTTP/1.1 200 OK
{"id":1,"name":"홍길동","email":"hong@example.com"}
```

---

## 🤔 트레이드오프

```
커스텀 Resolver 도입:
  장점  인증 로직과 비즈니스 로직 완전 분리
        컨트롤러 코드 간결 → 가독성 향상
        테스트 시 Resolver를 별도로 단위 테스트 가능
        SecurityContextHolder 직접 의존 제거
  단점  추가 추상화 레이어 → 코드 흐름 파악 어려움
        @CurrentUser 의미를 팀 전체가 알아야 함
        잘못 구현 시 보안 취약점 (인증 체크 누락 등)

addArgumentResolvers vs setArgumentResolvers:
  addArgumentResolvers: 간단, 기본 Resolver 유지, 뒤에 추가
  setArgumentResolvers: 전체 제어, 순서 자유, 기본 설정 날아갈 위험

요청 속성 캐시 (request.setAttribute):
  장점  같은 요청 내 동일 정보 재조회 방지
  단점  캐시 무효화 불필요 (요청 종료 시 자연 소멸)
        스레드 안전: HttpServletRequest는 요청당 한 스레드 → 문제 없음
```

---

## 📌 핵심 정리

```
HandlerMethodArgumentResolver 인터페이스
  supportsParameter(): Resolver 선택 여부 → 결과 캐시됨, 빠르게 구현
  resolveArgument():   매 요청마다 호출 → 실제 값 생성 로직

등록 방법별 차이
  addArgumentResolvers(): 기본 Resolver 뒤에 추가 (간단, 기본 보존)
  setArgumentResolvers(): 전체 목록 직접 지정 (순서 제어 가능, 주의 필요)
  @PostConstruct + adapter.setArgumentResolvers(): 기존 목록에 앞에 삽입

@CurrentUser 구현 패턴
  ① 커스텀 어노테이션 정의 (@Target, @Retention 필수)
  ② supportsParameter: 어노테이션 + 타입 조건
  ③ resolveArgument: SecurityContext → username → DB 조회 → 캐시
  ④ WebMvcConfigurer.addArgumentResolvers() 등록

테스트 전략
  단위 테스트: MethodParameter 직접 생성 + SecurityContextHolder 설정
  통합 테스트: @WebMvcTest + Spring Security Test (.with(user("name")))

MethodParameter 주요 메서드
  hasParameterAnnotation(Ann.class)  : 어노테이션 확인
  getParameterType()                 : 파라미터 타입
  nestedIfOptional()                 : Optional<T> 언래핑
  getDeclaringClass()                : 선언 클래스
```

---

## 🤔 생각해볼 문제

**Q1.** `CurrentUserArgumentResolver`가 `@Component`로 등록되어 있고, `WebMvcConfigurer.addArgumentResolvers()`에서 `@Autowired`로 주입받아 추가합니다. 이때 `CurrentUserArgumentResolver`에 `UserService`가 `@Autowired`로 주입되어 있다면 순환 참조 문제가 발생할 가능성이 있는가?

**Q2.** `supportsParameter()`가 `parameter.hasParameterAnnotation(CurrentUser.class)`만 체크하고 타입 체크를 하지 않으면 어떤 문제가 생기는가? 예를 들어 `@CurrentUser String name` 파라미터가 있다면?

**Q3.** 비동기 처리(`@Async`, `CompletableFuture`) 환경에서 `resolveArgument()` 내부에서 `SecurityContextHolder.getContext()`를 호출하면 어떤 문제가 생길 수 있는가? 어떻게 해결하는가?

> 💡 **해설**
>
> **Q1.** 가능성은 낮지만 발생할 수 있습니다. `WebMvcConfigurer` 구현체가 `@Configuration` 클래스이고 `CurrentUserArgumentResolver`를 `@Autowired`로 주입받을 때, `CurrentUserArgumentResolver`가 `UserService`를 주입받고 `UserService`가 다시 어떤 웹 레이어 Bean에 의존한다면 순환이 생깁니다. 일반적으로는 `UserService`가 순수한 서비스 레이어에만 의존하므로 문제가 없습니다. 순환 참조가 걱정된다면 `ObjectProvider<UserService>`나 `@Lazy`를 사용하거나, Resolver를 `@Bean`으로 직접 생성하는 방식으로 우회합니다.
>
> **Q2.** `@CurrentUser String name` 파라미터에서 `resolveArgument()`가 `User` 타입 객체를 반환하면 타입 불일치로 `ClassCastException`이 발생합니다. 이는 스택 트레이스가 복잡해 디버깅이 어렵습니다. `supportsParameter()`에서 타입 체크(`User.class.isAssignableFrom(parameter.getParameterType())`)를 함께 해야 잘못된 파라미터 선언을 조기에 차단할 수 있습니다. 좋은 Resolver는 `supportsParameter()`를 최대한 엄격하게 구현합니다.
>
> **Q3.** Spring Security의 `SecurityContextHolder`는 기본적으로 `MODE_THREADLOCAL` 전략을 사용합니다. `@Async`로 비동기 메서드를 호출하면 새 스레드에서 실행되며, 이 스레드에는 원래 요청 스레드의 `SecurityContext`가 전파되지 않아 `SecurityContextHolder.getContext().getAuthentication()`이 `null`을 반환합니다. 해결책: (1) `SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_INHERITABLETHREADLOCAL)` 설정으로 자식 스레드에 컨텍스트 전파. (2) `DelegatingSecurityContextExecutor`로 비동기 실행기를 래핑. (3) `resolveArgument()` 시점(동기)에 이미 User를 조회했으므로 비동기 코드에는 User 객체를 직접 전달하는 방법을 권장합니다.

---

<div align="center">

**[⬅️ 이전: Servlet API 주입](./06-servlet-api-injection.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 4 — Response Handling ➡️](../response-handling/01-return-value-handler.md)**

</div>
