# WebMvcConfigurer 커스터마이징 가이드 — 콜백 호출 시점과 AutoConfiguration 통합

---

## 🎯 핵심 질문

- `WebMvcConfigurer`의 각 콜백이 호출되는 정확한 시점과 해당 컴포넌트와의 관계는?
- `WebMvcAutoConfiguration`이 `WebMvcConfigurer` Bean들을 어떻게 통합하는가?
- `@EnableWebMvc`를 추가하면 자동 구성(AutoConfiguration)이 비활성화되는 이유는?
- `configureMessageConverters()` vs `extendMessageConverters()`의 차이는?
- `addArgumentResolvers()`로 등록된 Resolver가 기본 Resolver보다 항상 뒤에 오는 이유는?
- 여러 `WebMvcConfigurer` Bean이 있을 때 병합 방식은?

---

## 🔍 왜 이 메커니즘이 존재하는가

```
Spring MVC 기본 설정은 이미 방대하다:
  HandlerMapping 등록, HandlerAdapter 설정,
  MessageConverter 15종, ArgumentResolver 30종+,
  ExceptionResolver 3종, ViewResolver 체인...

하나의 XML/Java Config으로 전부 정의하면:
  → 작성량 방대, 실수 여지 多

WebMvcConfigurer 인터페이스:
  필요한 콜백만 오버라이드
  나머지는 Spring MVC 기본값 유지
  → 최소한의 코드로 필요한 부분만 커스터마이징
```

---

## 😱 흔한 오해 또는 실수

### Before: @EnableWebMvc를 붙이면 Spring MVC가 활성화된다

```java
// ❌ Spring Boot에서 @EnableWebMvc 추가
@SpringBootApplication
@EnableWebMvc  // ← 위험!
public class MyApp { ... }

// 실제로 발생하는 일:
// @EnableWebMvc → DelegatingWebMvcConfiguration import
//   → "WebMvc 설정을 내가 직접 한다" 선언
//   → WebMvcAutoConfiguration.@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
//      → DelegatingWebMvcConfiguration이 WebMvcConfigurationSupport이므로
//      → WebMvcAutoConfiguration 비활성화!
//
// 결과:
//   Jackson 자동 구성, 정적 리소스 핸들러, 기본 오류 처리 등 사라짐
//   → 직접 다 설정해야 함

// ✅ Spring Boot에서는 @EnableWebMvc 없이 WebMvcConfigurer만 사용
@Configuration
public class WebConfig implements WebMvcConfigurer {
    // 필요한 것만 오버라이드
}
```

### Before: configureMessageConverters()로 Jackson Converter를 추가할 수 있다

```java
// ❌ 의도와 다른 결과
@Override
public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
    converters.add(new MappingJackson2HttpMessageConverter());
    // "Jackson Converter가 기존 것에 추가된다"
}

// ✅ 실제:
// configureMessageConverters()에 아무것도 추가하지 않으면
//   → Spring이 기본 Converter 목록 생성 (Jackson, Gson 등 포함)
//   → 하나라도 추가하면 → 기본 목록 무시, 내가 추가한 것만 사용

// 기본 목록을 유지하면서 추가:
@Override
public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
    // 기본 Converter 목록을 받아서 수정/추가
    converters.add(0, new MyCustomConverter());  // 앞에 추가
}
```

---

## ✨ 올바른 이해와 사용

### After: WebMvcConfigurer 주요 콜백 전체 목록

```java
public interface WebMvcConfigurer {

    // ① HandlerMapping 관련
    void configurePathMatch(PathMatchConfigurer configurer);
    // → PathPattern vs AntPathMatcher 설정
    // → suffix pattern matching 비활성화 (/users.json → /users)
    // → trailing slash 처리

    void configureContentNegotiation(ContentNegotiationConfigurer configurer);
    // → 미디어 타입 협상 전략 (헤더, 파라미터, 경로 확장자)

    // ② Async 처리
    void configureAsyncSupport(AsyncSupportConfigurer configurer);
    // → AsyncTaskExecutor 설정, 타임아웃 기본값

    // ③ 정적 리소스
    void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer);
    // → 정적 리소스를 DefaultServlet에 위임 (거의 사용 안 함)

    void addResourceHandlers(ResourceHandlerRegistry registry);
    // → 정적 리소스 경로 매핑 (/static/** → classpath:/static/)

    // ④ Interceptor / CORS
    void addInterceptors(InterceptorRegistry registry);
    void addCorsMappings(CorsRegistry registry);

    // ⑤ View 관련
    void configureViewResolvers(ViewResolverRegistry registry);
    void addViewControllers(ViewControllerRegistry registry);
    // → URL → View 이름 매핑 (Controller 없이)

    // ⑥ ArgumentResolver / ReturnValueHandler
    void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers);
    // → 기본 Resolver 뒤에 추가

    void addReturnValueHandlers(List<HandlerMethodReturnValueHandler> handlers);
    // → 기본 ReturnValueHandler 뒤에 추가

    // ⑦ MessageConverter
    void configureMessageConverters(List<HttpMessageConverter<?>> converters);
    // → 기본 Converter 목록 교체 (비어있으면 기본 유지)

    void extendMessageConverters(List<HttpMessageConverter<?>> converters);
    // → 기본 Converter 목록에 추가/수정

    // ⑧ ExceptionHandler
    void configureHandlerExceptionResolvers(List<HandlerExceptionResolver> resolvers);
    // → 기본 Resolver 목록 교체

    void extendHandlerExceptionResolvers(List<HandlerExceptionResolver> resolvers);
    // → 기본 Resolver 목록에 추가

    // ⑨ Formatter/Converter
    void addFormatters(FormatterRegistry registry);
    // → 타입 변환 (String → LocalDate 등)

    // ⑩ Validator
    default Validator getValidator() { return null; }
    // → 커스텀 Validator (null이면 기본 JSR-303)

    default MessageCodesResolver getMessageCodesResolver() { return null; }
}
```

---

## 🔬 내부 동작 원리

### 1. WebMvcAutoConfiguration과 DelegatingWebMvcConfiguration

```java
// WebMvcAutoConfiguration.java (Spring Boot)
@Configuration
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class })
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)  // ← 핵심 조건
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
public class WebMvcAutoConfiguration {

    // @EnableWebMvc가 없을 때만 활성화
    // DelegatingWebMvcConfiguration은 WebMvcConfigurationSupport 상속
    // → @EnableWebMvc 추가 → DelegatingWebMvcConfiguration 등록
    //   → @ConditionalOnMissingBean(WebMvcConfigurationSupport.class) 실패
    //   → WebMvcAutoConfiguration 비활성화

    @Configuration
    public static class EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration {
        // WebMvcConfigurer Bean들 수집 + 기본 Spring MVC 설정 병합
    }
}
```

### 2. DelegatingWebMvcConfiguration — WebMvcConfigurer Bean 수집

```java
// DelegatingWebMvcConfiguration.java
@Configuration
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {

    private final WebMvcConfigurerComposite configurers = new WebMvcConfigurerComposite();

    @Autowired(required = false)
    public void setConfigurers(List<WebMvcConfigurer> configurers) {
        if (!CollectionUtils.isEmpty(configurers)) {
            // ApplicationContext에서 WebMvcConfigurer Bean 전체 수집
            this.configurers.addWebMvcConfigurers(configurers);
        }
    }

    // 각 콜백을 수집된 모든 Configurer에 위임
    @Override
    protected void addInterceptors(InterceptorRegistry registry) {
        this.configurers.addInterceptors(registry);
        // → Configurer1.addInterceptors(registry)
        // → Configurer2.addInterceptors(registry)
        // → 모두 동일한 registry에 추가됨
    }

    @Override
    protected void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        this.configurers.configureMessageConverters(converters);
    }
}
```

### 3. addArgumentResolvers — 기본 Resolver 뒤에 추가되는 이유

```java
// RequestMappingHandlerAdapter.afterPropertiesSet()
public void afterPropertiesSet() {
    // 1. 기본 ArgumentResolver 목록 생성 (30종+)
    List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>();
    // @RequestParam, @PathVariable, @RequestBody ... 등 기본 Resolver 추가
    resolvers.addAll(getDefaultArgumentResolvers());

    // 2. WebMvcConfigurer.addArgumentResolvers()로 추가된 것을 뒤에 붙임
    if (this.customArgumentResolvers != null) {
        resolvers.addAll(this.customArgumentResolvers);
    }

    this.argumentResolvers = new HandlerMethodArgumentResolverComposite()
        .addResolvers(resolvers);
}

// 결과: 커스텀 Resolver가 항상 기본 Resolver 뒤에 위치
// → @RequestParam 등 기본 처리보다 우선할 수 없음
// 우선순위 높이려면:
@Override
public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
    // 불가능 — 이 방법으로는 기본 앞에 추가 불가
}

// 대신 RequestMappingHandlerAdapter Bean을 직접 수정:
@Bean
public RequestMappingHandlerAdapter requestMappingHandlerAdapter() {
    RequestMappingHandlerAdapter adapter = new RequestMappingHandlerAdapter();
    List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>();
    resolvers.add(new MyHighPriorityResolver());        // 앞에
    resolvers.addAll(adapter.getArgumentResolvers());   // 기본 뒤에
    adapter.setArgumentResolvers(resolvers);
    return adapter;
}
```

### 4. 콜백별 호출 시점 — 컴포넌트와의 연결

```java
// WebMvcConfigurationSupport가 @Bean 메서드로 컴포넌트를 생성할 때 콜백 호출

// requestMappingHandlerMapping() Bean 생성 시:
@Bean
public RequestMappingHandlerMapping requestMappingHandlerMapping() {
    RequestMappingHandlerMapping mapping = new RequestMappingHandlerMapping();
    // configurePathMatch() 결과 적용
    configurePathMatch(new PathMatchConfigurer());
    return mapping;
}

// requestMappingHandlerAdapter() Bean 생성 시:
@Bean
public RequestMappingHandlerAdapter requestMappingHandlerAdapter() {
    RequestMappingHandlerAdapter adapter = new RequestMappingHandlerAdapter();
    // configureMessageConverters() → getMessageConverters() 결과 적용
    // addArgumentResolvers() → customArgumentResolvers 설정
    // addReturnValueHandlers() → customReturnValueHandlers 설정
    return adapter;
}

// 정리: 콜백 → 컴포넌트 연결 테이블
// addInterceptors()         → RequestMappingHandlerMapping
// addArgumentResolvers()    → RequestMappingHandlerAdapter
// addReturnValueHandlers()  → RequestMappingHandlerAdapter
// configureMessageConverters() → RequestMappingHandlerAdapter
// addCorsMappings()         → AbstractHandlerMapping
// addResourceHandlers()     → ResourceHttpRequestHandler 등록
// configureViewResolvers()  → ViewResolverComposite
// configureAsyncSupport()   → RequestMappingHandlerAdapter + WebAsyncManager
```

### 5. 여러 WebMvcConfigurer 병합

```java
// WebMvcConfigurerComposite — 여러 Configurer를 하나로 묶음
class WebMvcConfigurerComposite implements WebMvcConfigurer {

    private final List<WebMvcConfigurer> delegates = new ArrayList<>();

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        for (WebMvcConfigurer delegate : this.delegates) {
            delegate.addInterceptors(registry);
        }
        // 모든 Configurer의 결과가 같은 registry에 누적
    }

    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        for (WebMvcConfigurer delegate : this.delegates) {
            delegate.configureMessageConverters(converters);
        }
        // 하나라도 추가하면 기본 목록 교체
    }
}

// 실전 패턴 — 목적별로 Configurer 분리
@Configuration
public class SecurityWebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(jwtInterceptor).addPathPatterns("/api/**");
    }
}

@Configuration
public class ApiWebConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**").allowedOriginPatterns("*");
    }
    @Override
    public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
        // 커스텀 Converter 추가
    }
}

@Configuration
public class AsyncWebConfig implements WebMvcConfigurer {
    @Override
    public void configureAsyncSupport(AsyncSupportConfigurer configurer) {
        configurer.setTaskExecutor(asyncTaskExecutor);
        configurer.setDefaultTimeout(30_000);
    }
}
// → 모두 자동으로 병합됨
```

---

## 💻 실험으로 확인하기

### 실험 1: 어떤 WebMvcConfigurer들이 등록됐는지 확인

```java
@Autowired
DelegatingWebMvcConfiguration webMvcConfig;

@GetMapping("/debug/configurers")
public List<String> listConfigurers() throws Exception {
    Field field = WebMvcConfigurerComposite.class.getDeclaredField("delegates");
    field.setAccessible(true);
    List<WebMvcConfigurer> delegates =
        (List<WebMvcConfigurer>) field.get(/* configurers field of DelegatingWebMvcConfiguration */);
    return delegates.stream()
        .map(c -> c.getClass().getSimpleName())
        .toList();
}
```

### 실험 2: @EnableWebMvc 추가 전/후 MessageConverter 목록 비교

```java
// @EnableWebMvc 없음 (Spring Boot 기본):
// [ByteArrayHttpMessageConverter, StringHttpMessageConverter,
//  ResourceHttpMessageConverter, MappingJackson2HttpMessageConverter, ...]
// → Jackson ObjectMapper: Spring Boot 자동 구성 적용 (날짜 형식 등)

// @EnableWebMvc 추가 후:
// [ByteArrayHttpMessageConverter, StringHttpMessageConverter, ...]
// → Jackson ObjectMapper: 기본값 (Spring Boot 자동 구성 미적용)
// → LocalDate 직렬화 오류 발생 가능
```

---

## 🌐 HTTP 레벨 분석

```
Spring Boot 초기화 시 WebMvcConfigurer 통합:

ApplicationContext 시작
  ↓
WebMvcAutoConfiguration 로드
  (@ConditionalOnMissingBean(WebMvcConfigurationSupport.class) = 통과)
  ↓
DelegatingWebMvcConfiguration 생성
  setConfigurers([SecurityWebConfig, ApiWebConfig, AsyncWebConfig, ...])
  ↓
각 @Bean 메서드 실행:
  requestMappingHandlerMapping() 생성 중:
    addInterceptors(registry)
    → SecurityWebConfig.addInterceptors() → JwtInterceptor 등록
    → ApiWebConfig.addInterceptors() (없음)
  
  requestMappingHandlerAdapter() 생성 중:
    extendMessageConverters(converters)
    → SecurityWebConfig.extendMessageConverters() (없음)
    → ApiWebConfig.extendMessageConverters() → CustomConverter 추가
    
    configureAsyncSupport(configurer)
    → AsyncWebConfig.configureAsyncSupport() → Executor, Timeout 설정

HTTP 요청 처리:
  JwtInterceptor가 /api/**에서 인증 검증
  CustomConverter가 커스텀 미디어 타입 처리
  AsyncTaskExecutor로 비동기 요청 처리
```

---

## 📌 핵심 정리

```
WebMvcConfigurer 핵심 원칙
  @EnableWebMvc 없이 사용 → Spring Boot 자동 구성 유지
  필요한 콜백만 오버라이드 → 나머지는 기본값
  여러 Configurer → 자동 병합 (같은 registry/list에 누적)

configure* vs extend* vs add*
  configureMessageConverters(): 기본 목록 교체 (값 추가하면)
  extendMessageConverters():    기본 목록 유지 + 추가/수정
  addArgumentResolvers():       기본 Resolver 뒤에 추가
  → 기본 유지 원칙: extend* / add* 권장

addArgumentResolvers() 한계
  기본 Resolver 앞에 추가 불가
  → RequestMappingHandlerAdapter Bean 직접 조작 필요

병합 방식
  WebMvcConfigurerComposite: 모든 Configurer를 순서대로 순회
  addInterceptors: 누적 (InterceptorRegistry에 모두 추가)
  configureMessageConverters: 하나라도 추가하면 기본 교체

콜백과 컴포넌트 연결
  addInterceptors     → RequestMappingHandlerMapping
  configureMessage*   → RequestMappingHandlerAdapter
  addCorsMappings     → AbstractHandlerMapping
  configureAsync*     → RequestMappingHandlerAdapter + WebAsyncManager
```

---

## 🤔 생각해볼 문제

**Q1.** Spring Boot 환경에서 `WebMvcConfigurer.configureMessageConverters()`를 구현해 `MappingJackson2HttpMessageConverter`를 추가하면 Spring Boot의 Jackson 자동 구성(날짜 형식, null 처리 등)이 적용된 ObjectMapper가 사용되는가?

**Q2.** 두 개의 `WebMvcConfigurer`에서 각각 `addResourceHandlers()`를 구현해 같은 경로 패턴(`/static/**`)을 서로 다른 실제 경로에 매핑하면 어떻게 되는가?

**Q3.** `WebMvcConfigurer.getValidator()`를 오버라이드해 커스텀 `Validator`를 반환하면 기존 JSR-303 Bean Validation과 함께 동작하는가, 대체되는가?

> 💡 **해설**
>
> **Q1.** 그렇지 않습니다. `configureMessageConverters()`에서 `new MappingJackson2HttpMessageConverter()`를 직접 생성하면 Spring Boot의 자동 구성 ObjectMapper(`JacksonAutoConfiguration`이 등록한 Bean)가 아닌 기본 ObjectMapper가 사용됩니다. Spring Boot의 Jackson 설정을 유지하면서 Converter를 추가하려면 `extendMessageConverters()`를 사용하거나, `@Autowired Jackson2ObjectMapperBuilder`를 주입받아 Converter를 생성해야 합니다.
>
> **Q2.** 두 Configurer가 모두 적용됩니다. `WebMvcConfigurerComposite`는 모든 Configurer를 순서대로 순회하며 같은 `ResourceHandlerRegistry`에 추가합니다. 같은 경로 패턴이 두 번 등록되면 `ResourceHttpRequestHandler` 두 개가 생성됩니다. 요청 처리 시 첫 번째로 매칭된 핸들러가 리소스를 반환하며, 없으면 두 번째 핸들러에서 찾습니다. 즉 두 경로를 순서대로 탐색하는 효과가 됩니다.
>
> **Q3.** 대체됩니다. `WebMvcConfigurationSupport.mvcValidator()`는 `getValidator()`를 먼저 호출하고, null이 아니면 해당 Validator를 사용합니다. null이면 클래스패스에서 JSR-303 구현체(Hibernate Validator)를 찾아 `LocalValidatorFactoryBean`을 생성합니다. 따라서 커스텀 Validator를 반환하면 JSR-303이 비활성화됩니다. 두 가지를 함께 사용하려면 커스텀 Validator 내에서 JSR-303 Validator를 직접 호출하는 위임(delegation) 패턴을 구현해야 합니다.

---

<div align="center">

**[⬅️ 이전: HTTP 캐싱](./05-http-caching.md)** | **[홈으로 🏠](../README.md)**

*🎉 Spring MVC Deep Dive 시리즈 완결*

</div>
