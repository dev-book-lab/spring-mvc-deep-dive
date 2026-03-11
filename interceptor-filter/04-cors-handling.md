# CORS 처리 — CorsFilter vs @CrossOrigin

---

## 🎯 핵심 질문

- Preflight `OPTIONS` 요청이 `CorsProcessor`를 통해 처리되는 정확한 경로는?
- `@CrossOrigin` 메타데이터가 `HandlerInterceptor`로 변환되는 시점과 방식은?
- `CorsFilter`와 Spring MVC 내 CORS 처리(Interceptor 방식)의 실행 위치 차이는?
- Spring Security와 함께 사용할 때 CORS 처리를 어디에 두어야 하는가?
- `corsConfiguration.setAllowedOriginPatterns()` vs `setAllowedOrigins()`의 차이는?
- `allowCredentials=true` + `allowedOrigins="*"` 조합이 왜 금지되는가?

---

## 🔍 왜 이 메커니즘이 존재하는가

```
브라우저의 Same-Origin Policy (SOP):
  https://frontend.example.com → https://api.example.com 요청
  → 다른 출처(origin) → 브라우저가 차단

CORS (Cross-Origin Resource Sharing):
  서버가 허용한 출처임을 브라우저에 알리는 HTTP 헤더 프로토콜
  Access-Control-Allow-Origin: https://frontend.example.com

Preflight 요청 (OPTIONS):
  브라우저가 실제 요청 전 서버에 "이 요청 허용하나요?" 사전 확인
  → 서버가 CORS 헤더로 허용 여부 응답
  → 허용이면 실제 요청 진행

Spring의 두 가지 CORS 처리 방식:
  1. CorsFilter         → 서블릿 컨테이너 레벨 (Filter)
  2. @CrossOrigin / CorsInterceptor → Spring MVC 레벨 (Interceptor)
```

---

## 😱 흔한 오해 또는 실수

### Before: @CrossOrigin만 있으면 Spring Security와 함께 동작한다

```java
// ❌ 잘못된 이해
@CrossOrigin(origins = "https://frontend.example.com")
@RestController
public class UserController { ... }

// Spring Security가 있을 때:
// Preflight OPTIONS 요청이 옴
//   → Spring Security Filter가 먼저 처리
//   → SecurityContext 없음 → 401 Unauthorized 반환
//   → @CrossOrigin Interceptor에 도달하지 못함
//   → CORS 헤더 없는 401 응답 → 브라우저가 CORS 오류로 해석

// ✅ 해결:
// 방법 1: Spring Security 설정에서 CORS 활성화
http.cors(cors -> cors.configurationSource(corsConfigurationSource()));
// → Spring Security가 CorsFilter를 자체적으로 앞에 배치

// 방법 2: CorsFilter를 Spring Security 앞에 명시적 등록
// → SecurityContextPersistenceFilter보다 앞선 순서로 CorsFilter 등록
```

### Before: allowedOrigins("*")와 allowCredentials(true)를 같이 쓸 수 있다

```
❌ 잘못된 이해: "모든 출처 허용 + 인증 정보 포함 = 편리하다"

✅ 실제:
  CORS 스펙상 allowCredentials=true이면 와일드카드(*) Origin 사용 불가
  → RFC 6454: 자격증명 있는 요청에 * 허용 시 보안 취약점
  → 브라우저가 Access-Control-Allow-Origin: * + 쿠키 요청을 거부
  → Spring도 DefaultCorsProcessor에서 이 조합을 에러로 처리

  올바른 방법:
  config.setAllowedOriginPatterns(List.of("https://*.example.com"))
  // → 패턴 매칭으로 유연성 유지, 자격증명과 함께 사용 가능
  config.setAllowCredentials(true);
```

---

## ✨ 올바른 이해와 사용

### After: Preflight OPTIONS 처리 전체 흐름

```
OPTIONS /api/users HTTP/1.1
Origin: https://frontend.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type, Authorization

[Spring MVC 내 CORS 처리 (@CrossOrigin)]

① HandlerMapping.getHandler(request)
   → HandlerExecutionChain + CorsInterceptor 추가
   (OPTIONS 요청이고 CORS 설정 있으면 CorsInterceptor 추가)

② CorsInterceptor.preHandle()
   → AbstractHandlerMapping.getCorsHandlerExecutionChain()
   → DefaultCorsProcessor.processRequest()
      corsConfig: allowedOrigins, allowedMethods 등 확인
      origin 허용 여부 검사
      → Access-Control-Allow-Origin 헤더 설정
      → Access-Control-Allow-Methods 헤더 설정
      → Access-Control-Max-Age 헤더 설정
      → 응답 완료 (preHandle에서 직접 응답 후 false 반환)

③ Controller 실행 안 됨 (Preflight이므로)

HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://frontend.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 3600
```

---

## 🔬 내부 동작 원리

### 1. @CrossOrigin → CorsConfiguration 빌드

```java
// AbstractHandlerMapping.getHandlerInternal()
// → HandlerMethod 탐색 후 CORS 설정 확인
@Override
protected final Object getHandlerInternal(HttpServletRequest request) throws Exception {
    Object handler = getHandlerInternal(request);

    // CORS 요청인지 확인 (Origin 헤더 존재)
    if (hasCorsConfigurationSource(handler) || CorsUtils.isCorsRequest(request)) {
        // @CrossOrigin 어노테이션에서 CorsConfiguration 빌드
        CorsConfiguration config = getCorsConfiguration(handler, request);
        // 전역 설정과 병합
        if (getCorsConfigurationSource() != null) {
            CorsConfiguration globalConfig =
                getCorsConfigurationSource().getCorsConfiguration(request);
            config = (globalConfig != null ? globalConfig.combine(config) : config);
        }
        // → HandlerExecutionChain에 CorsInterceptor 추가
        return getCorsHandlerExecutionChain(request, chain, config);
    }
    return chain;
}

// @CrossOrigin 메타데이터 읽기:
// RequestMappingHandlerMapping.initCorsConfiguration()
@Override
protected CorsConfiguration initCorsConfiguration(Object handler, Method method,
        RequestMappingInfo mappingInfo) {
    // 클래스 레벨 @CrossOrigin
    CrossOrigin typeAnnotation = AnnotatedElementUtils.findMergedAnnotation(
        method.getDeclaringClass(), CrossOrigin.class);
    // 메서드 레벨 @CrossOrigin
    CrossOrigin methodAnnotation = AnnotatedElementUtils.findMergedAnnotation(
        method, CrossOrigin.class);

    if (typeAnnotation == null && methodAnnotation == null) return null;

    CorsConfiguration config = new CorsConfiguration();
    updateCorsConfig(config, typeAnnotation);
    updateCorsConfig(config, methodAnnotation);
    // 기본값 적용: methods = 해당 @RequestMapping의 HTTP 메서드
    config.applyPermitDefaultValues();
    return config;
}
```

### 2. DefaultCorsProcessor.processRequest() — 핵심 검증 로직

```java
// DefaultCorsProcessor.java
@Override
public boolean processRequest(@Nullable CorsConfiguration config,
        HttpServletRequest request, HttpServletResponse response) throws IOException {

    // CORS 요청 아니면 통과
    if (!CorsUtils.isCorsRequest(request)) return true;

    // 이미 CORS 헤더 있으면 통과 (다른 컴포넌트가 처리)
    if (response.getHeader(HttpHeaders.ACCESS_CONTROL_ALLOW_ORIGIN) != null) return true;

    boolean isPreFlight = CorsUtils.isPreFlightRequest(request);
    boolean rejectRequest = false;

    // ① Origin 검증
    String requestOrigin = request.getHeader(HttpHeaders.ORIGIN);
    String allowOrigin = checkOrigin(config, requestOrigin);
    if (allowOrigin == null) {
        rejectRequest = true;  // 허용되지 않은 Origin
    }

    // ② Preflight: 요청 메서드 검증
    HttpMethod requestMethod = getMethodToUse(request, isPreFlight);
    List<HttpMethod> allowMethods = checkMethods(config, requestMethod);
    if (allowMethods == null) rejectRequest = true;

    // ③ Preflight: 요청 헤더 검증
    List<String> requestHeaders = getHeadersToUse(request, isPreFlight);
    List<String> allowHeaders = checkHeaders(config, requestHeaders);
    if (isPreFlight && allowHeaders == null) rejectRequest = true;

    if (rejectRequest) {
        response.setStatus(HttpServletResponse.SC_FORBIDDEN);
        return false;  // 403 반환
    }

    // CORS 응답 헤더 설정
    response.addHeader(HttpHeaders.ACCESS_CONTROL_ALLOW_ORIGIN, allowOrigin);
    if (isPreFlight) {
        response.addHeader(HttpHeaders.ACCESS_CONTROL_ALLOW_METHODS,
            StringUtils.collectionToCommaDelimitedString(allowMethods));
        if (!allowHeaders.isEmpty()) {
            response.addHeader(HttpHeaders.ACCESS_CONTROL_ALLOW_HEADERS,
                StringUtils.collectionToCommaDelimitedString(allowHeaders));
        }
        if (config.getMaxAge() != null) {
            response.addHeader(HttpHeaders.ACCESS_CONTROL_MAX_AGE,
                config.getMaxAge().toString());
        }
        response.setStatus(HttpServletResponse.SC_NO_CONTENT);  // 204
        return false;  // Preflight 완료, Controller 실행 안 함
    }

    // 일반 CORS 요청 → 헤더만 추가하고 Controller 실행 허용
    if (Boolean.TRUE.equals(config.getAllowCredentials())) {
        response.addHeader(HttpHeaders.ACCESS_CONTROL_ALLOW_CREDENTIALS, "true");
    }
    return true;
}
```

### 3. CorsFilter vs @CrossOrigin — 실행 위치와 선택 기준

```java
// ━━━ CorsFilter (서블릿 컨테이너 레벨) ━━━
// Spring Security와 함께 사용 시 필수
@Bean
public CorsFilter corsFilter() {
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOriginPatterns(List.of("https://*.example.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(List.of("*"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L);
    source.registerCorsConfiguration("/api/**", config);
    return new CorsFilter(source);
}
// → Spring Security Filter보다 먼저 등록 → Preflight에 401 없음
// → 서블릿 레벨 → 모든 요청에 적용 (DispatcherServlet 전)

// ━━━ @CrossOrigin (Spring MVC 레벨) ━━━
@CrossOrigin(
    origins = {"https://frontend.example.com"},
    methods = {RequestMethod.GET, RequestMethod.POST},
    allowedHeaders = {"Authorization", "Content-Type"},
    allowCredentials = "true",
    maxAge = 3600
)
@RestController
@RequestMapping("/api")
public class UserController { ... }
// → Spring MVC Interceptor로 처리
// → Controller/메서드 단위 세밀한 제어 가능
// → Spring Security 없으면 충분

// ━━━ WebMvcConfigurer 전역 설정 (Spring MVC 레벨) ━━━
@Override
public void addCorsMappings(CorsRegistry registry) {
    registry.addMapping("/api/**")
        .allowedOriginPatterns("https://*.example.com")
        .allowedMethods("GET", "POST", "PUT", "DELETE")
        .allowedHeaders("*")
        .allowCredentials(true)
        .maxAge(3600);
}
// → CorsInterceptor로 등록 → Spring MVC 레벨
// → 전역 설정, @CrossOrigin과 병합 가능
```

### 4. Spring Security 통합 패턴

```java
// Spring Security 6.x 권장 방식
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            // → CorsFilter를 SecurityContextHolderFilter 앞에 자동 배치
            // → Preflight OPTIONS 요청은 인증 없이 통과
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()
                // 추가 안전장치: OPTIONS 전체 허용
                .anyRequest().authenticated()
            );
        return http.build();
    }

    @Bean
    CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOriginPatterns(List.of("https://*.example.com"));
        configuration.setAllowedMethods(List.of("GET","POST","PUT","DELETE","OPTIONS"));
        configuration.setAllowedHeaders(List.of("*"));
        configuration.setAllowCredentials(true);
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        return source;
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: Preflight OPTIONS 요청 흐름

```bash
# Preflight 요청
curl -i -X OPTIONS http://localhost:8080/api/users \
     -H "Origin: https://frontend.example.com" \
     -H "Access-Control-Request-Method: POST" \
     -H "Access-Control-Request-Headers: Content-Type"

# CORS 설정 있을 때:
# HTTP/1.1 204 No Content
# Access-Control-Allow-Origin: https://frontend.example.com
# Access-Control-Allow-Methods: GET,POST,PUT,DELETE
# Access-Control-Allow-Headers: Content-Type
# Access-Control-Max-Age: 3600

# CORS 설정 없거나 Origin 불허:
# HTTP/1.1 403 Forbidden
```

### 실험 2: allowedOriginPatterns vs allowedOrigins

```java
// allowedOrigins("*") + allowCredentials(true) → 시작 시 에러 또는 403
// allowedOriginPatterns("*") + allowCredentials(true) → 가능
//   (패턴 매칭 방식으로 실제 Origin을 검증 후 그 값을 헤더에 설정)
config.setAllowedOriginPatterns(List.of("*"));
config.setAllowCredentials(true);
// → 응답: Access-Control-Allow-Origin: https://frontend.example.com (실제 Origin)
//         (와일드카드 * 아닌 구체적 Origin)
```

---

## 🌐 HTTP 레벨 분석

```
일반 CORS 요청 (Preflight 없는 Simple Request):

GET /api/users HTTP/1.1
Origin: https://frontend.example.com

CorsInterceptor.preHandle():
  DefaultCorsProcessor.processRequest()
  → Origin 검증: ✅
  → 일반 요청 → Access-Control-Allow-Origin 헤더만 추가
  → return true (Controller 실행 허용)

Controller: UserController.getUsers() 실행

HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://frontend.example.com
Access-Control-Allow-Credentials: true
Vary: Origin
Content-Type: application/json
[{"id":1,"name":"홍길동"}]

Vary: Origin:
  응답이 Origin 헤더에 따라 달라질 수 있음 → 캐시 분리 지시
```

---

## 📌 핵심 정리

```
CORS 처리 방식 선택
  Spring Security 없음:
    @CrossOrigin (컨트롤러/메서드 단위)
    WebMvcConfigurer.addCorsMappings() (전역)

  Spring Security 있음:
    HttpSecurity.cors() + CorsConfigurationSource Bean
    → CorsFilter가 Security Filter 앞에 자동 배치
    → Preflight에 401 없음

Preflight OPTIONS 처리 흐름
  Origin + Access-Control-Request-Method 헤더 확인
  DefaultCorsProcessor: Origin/메서드/헤더 검증
  → 허용: 204 + CORS 헤더, Controller 미실행
  → 거부: 403

allowedOrigins vs allowedOriginPatterns
  setAllowedOrigins("*") + allowCredentials=true → 금지 (보안)
  setAllowedOriginPatterns("*") + allowCredentials=true → 허용
  → 패턴 매칭 후 실제 Origin 값을 헤더에 반환

Vary: Origin 자동 추가
  Origin별로 응답이 다를 수 있으므로 캐시 분리 필요
```

---

## 🤔 생각해볼 문제

**Q1.** `@CrossOrigin`과 `WebMvcConfigurer.addCorsMappings()` 설정이 동시에 있을 때, 같은 경로에서 두 설정이 충돌하면 어떻게 처리되는가?

**Q2.** Preflight 요청은 `Access-Control-Max-Age` 동안 브라우저에서 캐시됩니다. 서버에서 CORS 설정을 변경했을 때 클라이언트가 즉시 반영받으려면 어떻게 해야 하는가?

**Q3.** `CorsUtils.isPreFlightRequest(request)`는 어떤 조건을 체크하는가?

> 💡 **해설**
>
> **Q1.** 두 설정은 `CorsConfiguration.combine()`으로 병합됩니다. `addCorsMappings()` 전역 설정이 먼저 적용되고, `@CrossOrigin` 설정이 그 위에 병합됩니다. 병합 시 `null`이 아닌 값이 있으면 덮어씁니다. 예를 들어 전역에서 `allowedOrigins=["*"]`이고 `@CrossOrigin`에서 `origins=["https://a.com"]`이면 최종은 `["https://a.com"]`이 됩니다.
>
> **Q2.** `Access-Control-Max-Age`를 0 또는 매우 낮은 값(예: `-1`)으로 설정하면 브라우저 캐시를 비활성화할 수 있습니다. 이미 캐시된 설정을 즉시 무효화하는 서버 측 방법은 없으므로, 설정 변경 시 캐시 시간만큼 기다리거나, 브라우저 개발자 도구에서 캐시를 수동으로 비워야 합니다. 운영 배포 시 `Max-Age`를 짧게 유지하거나 배포 후 캐시 만료를 기다리는 것이 일반적인 접근입니다.
>
> **Q3.** `CorsUtils.isPreFlightRequest()`는 세 조건을 모두 체크합니다: ① HTTP 메서드가 `OPTIONS`이고, ② `Origin` 헤더가 존재하고, ③ `Access-Control-Request-Method` 헤더가 존재하면 Preflight로 판단합니다. 세 조건 중 하나라도 없으면 일반 CORS 요청 또는 비 CORS 요청으로 처리합니다.

---

<div align="center">

**[⬅️ 이전: Interceptor 호출 시점](./03-interceptor-lifecycle.md)** | **[홈으로 🏠](../README.md)** | **[다음: Custom Interceptor 작성 패턴 ➡️](./05-custom-interceptor-pattern.md)**

</div>
