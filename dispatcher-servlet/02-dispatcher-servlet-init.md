# DispatcherServlet 초기화 과정 — onRefresh()가 Spring MVC를 살아있게 만드는 방법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `DispatcherServlet.onRefresh()`는 언제, 무엇에 의해 호출되는가?
- `initStrategies()`가 초기화하는 9가지 컴포넌트는 무엇이고 각각의 역할은?
- `detectAllHandlerMappings` 같은 플래그가 false일 때와 true일 때 초기화 방식이 어떻게 달라지는가?
- `DispatcherServlet.properties`에 정의된 default 전략은 어떻게 로딩되는가?
- Spring Boot의 `WebMvcAutoConfiguration`은 이 초기화 과정에 어떻게 개입하는가?
- 이미 등록된 컴포넌트 Bean이 없을 때 fallback이 동작하는 원리는?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: MVC 컴포넌트는 ApplicationContext에 Bean으로 존재하지만 DispatcherServlet이 모른다

```
DispatcherServlet 초기화 전:
  ApplicationContext: Bean이 등록되어 있음
    - RequestMappingHandlerMapping (Bean)
    - RequestMappingHandlerAdapter (Bean)
    - ContentNegotiatingViewResolver (Bean)
    - ...

  DispatcherServlet: 이 Bean들을 아직 모름
    - this.handlerMappings = null
    - this.handlerAdapters = null
    - this.viewResolvers = null
    → 요청이 들어와도 어디로 보내야 할지 모름

해결:
  ApplicationContext.refresh() 완료 후 onRefresh() 호출
  → initStrategies()에서 ApplicationContext에서 Bean을 조회해 필드에 세팅
  → 이후부터 doDispatch() 실행 가능
```

### 초기화가 ApplicationContext refresh 이후에 일어나야 하는 이유

```java
// FrameworkServlet.java
protected WebApplicationContext initWebApplicationContext() {
    // ...
    ConfigurableWebApplicationContext wac = createWebApplicationContext(rootContext);
    // ...
    if (!this.refreshEventReceived) {
        // refresh()가 이미 완료된 경우 직접 호출
        synchronized (this.onRefreshMonitor) {
            onRefresh(wac);  // ← Bean이 모두 등록된 후에야 호출
        }
    }
    return wac;
}

// 이벤트 기반 호출 경로:
// wac.refresh() → publishEvent(ContextRefreshedEvent)
//              → FrameworkServlet.onApplicationEvent() 수신
//              → onRefresh() 호출
```

---

## 😱 흔한 오해 또는 실수

### Before: DispatcherServlet이 시작되면 즉시 요청을 처리할 수 있다

```
❌ 잘못된 이해:
  "Tomcat이 DispatcherServlet을 생성하면 바로 요청을 받을 수 있다"

✅ 실제 초기화 순서:
  1. DispatcherServlet 인스턴스 생성 (new)
  2. HttpServletBean.init() → init-param 바인딩
  3. FrameworkServlet.initServletBean()
  4. WebApplicationContext 생성 및 refresh()  ← Bean 등록 완료
  5. ContextRefreshedEvent 발행
  6. DispatcherServlet.onRefresh()              ← 여기서야 MVC 컴포넌트 세팅
  7. initStrategies() — 9가지 컴포넌트 초기화
  → 7단계 완료 이후부터 요청 처리 가능

  spring.mvc.servlet.load-on-startup=1 설정:
  → Tomcat 시작 시 즉시 초기화 (첫 요청 대기 없이)
  → 기본값: 1 (Spring Boot)
  → 설정 안 하면: 첫 요청 시 초기화 (콜드 스타트 지연)
```

### Before: HandlerMapping은 무조건 하나만 존재한다

```java
// ❌ 잘못된 이해
// "DispatcherServlet은 HandlerMapping을 하나만 사용한다"

// ✅ 실제: detectAllHandlerMappings=true (기본값)이면
//    ApplicationContext에서 HandlerMapping 타입의 Bean을 모두 수집
//    Spring Boot 기본 설정 시 보통 3개 이상:
//    - RequestMappingHandlerMapping (order=0) ← @RequestMapping 처리
//    - BeanNameUrlHandlerMapping     (order=2)
//    - RouterFunctionMapping         (order=-1) ← @RouterFunction 처리
//    → getHandler()에서 순서대로 탐색, 첫 번째 match 반환
```

---

## ✨ 올바른 이해와 사용

### After: onRefresh → initStrategies → 9가지 컴포넌트 순서 초기화

```
initStrategies() 호출 순서:

  1. initMultipartResolver()
     "multipartResolver" 이름의 Bean 조회
     → 없으면 null (파일 업로드 기능 비활성)

  2. initLocaleResolver()
     "localeResolver" 이름의 Bean 조회
     → 없으면 default: AcceptHeaderLocaleResolver

  3. initThemeResolver()
     "themeResolver" 이름의 Bean 조회
     → 없으면 default: FixedThemeResolver

  4. initHandlerMappings()      ← 핵심 1
     detectAllHandlerMappings=true(기본):
       HandlerMapping.class 타입 Bean 전체 수집 + 정렬
     false:
       "handlerMapping" 이름 단일 Bean만 조회
     → 없으면 default: RequestMappingHandlerMapping + BeanNameUrlHandlerMapping

  5. initHandlerAdapters()      ← 핵심 2
     detectAllHandlerAdapters=true(기본):
       HandlerAdapter.class 타입 Bean 전체 수집 + 정렬
     → 없으면 default: RequestMappingHandlerAdapter + HttpRequestHandlerAdapter + SimpleControllerHandlerAdapter

  6. initHandlerExceptionResolvers()  ← 핵심 3
     detectAllHandlerExceptionResolvers=true(기본):
       HandlerExceptionResolver 타입 Bean 전체 수집
     → 없으면 default: ExceptionHandlerExceptionResolver + ResponseStatusExceptionResolver + DefaultHandlerExceptionResolver

  7. initRequestToViewNameTranslator()
     "viewNameTranslator" 이름의 Bean 조회
     → 없으면 default: DefaultRequestToViewNameTranslator

  8. initViewResolvers()        ← 핵심 4
     detectAllViewResolvers=true(기본):
       ViewResolver 타입 Bean 전체 수집
     → 없으면 default: InternalResourceViewResolver

  9. initFlashMapManager()
     "flashMapManager" 이름의 Bean 조회
     → 없으면 default: SessionFlashMapManager
```

---

## 🔬 내부 동작 원리

### 1. onRefresh() → initStrategies() 진입

```java
// DispatcherServlet.java
@Override
protected void onRefresh(ApplicationContext context) {
    initStrategies(context);
}

protected void initStrategies(ApplicationContext context) {
    initMultipartResolver(context);
    initLocaleResolver(context);
    initThemeResolver(context);
    initHandlerMappings(context);
    initHandlerAdapters(context);
    initHandlerExceptionResolvers(context);
    initRequestToViewNameTranslator(context);
    initViewResolvers(context);
    initFlashMapManager(context);
}
```

### 2. initHandlerMappings() — 가장 중요한 초기화

```java
// DispatcherServlet.java
private void initHandlerMappings(ApplicationContext context) {
    this.handlerMappings = null;

    if (this.detectAllHandlerMappings) {
        // ApplicationContext 계층 전체(부모 포함)에서 HandlerMapping Bean 수집
        Map<String, HandlerMapping> matchingBeans =
            BeanFactoryUtils.beansOfTypeIncludingAncestors(
                context, HandlerMapping.class, true, false);

        if (!matchingBeans.isEmpty()) {
            this.handlerMappings = new ArrayList<>(matchingBeans.values());
            // Ordered 인터페이스 또는 @Order 기준으로 정렬
            AnnotationAwareOrderComparator.sort(this.handlerMappings);
        }
    } else {
        // "handlerMapping" 이름의 단일 Bean만 조회
        try {
            HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
            this.handlerMappings = Collections.singletonList(hm);
        } catch (NoSuchBeanDefinitionException ex) {
            // 이름으로 못 찾으면 fallback으로 이동
        }
    }

    // 하나도 없으면 DispatcherServlet.properties의 default 전략 로딩
    if (this.handlerMappings == null) {
        this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
        if (logger.isTraceEnabled()) {
            logger.trace("No HandlerMappings declared for servlet '"
                + getServletName() + "': using default strategies from DispatcherServlet.properties");
        }
    }
}
```

### 3. getDefaultStrategies() — DispatcherServlet.properties 로딩

```java
// DispatcherServlet.java
private static final String DEFAULT_STRATEGIES_PATH = "DispatcherServlet.properties";
private static final Properties defaultStrategies;

// 클래스 로딩 시 static 블록으로 프로퍼티 파일 로딩 (한 번만)
static {
    try {
        ClassPathResource resource =
            new ClassPathResource(DEFAULT_STRATEGIES_PATH, DispatcherServlet.class);
        defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
    } catch (IOException ex) {
        throw new IllegalStateException("Could not load '" + DEFAULT_STRATEGIES_PATH + "'", ex);
    }
}

protected <T> List<T> getDefaultStrategies(ApplicationContext context, Class<T> strategyInterface) {
    // 프로퍼티 파일에서 클래스명 읽기
    // "org.springframework.web.servlet.HandlerMapping" 키 조회
    String key = strategyInterface.getName();
    String value = defaultStrategies.getProperty(key);  // ← 쉼표로 구분된 클래스명 문자열

    if (value != null) {
        String[] classNames = StringUtils.commaDelimitedListToStringArray(value);
        List<T> strategies = new ArrayList<>(classNames.length);
        for (String className : classNames) {
            Class<?> clazz = ClassUtils.forName(className, DispatcherServlet.class.getClassLoader());
            // ApplicationContext로 인스턴스 생성 (BeanWrapper 이용)
            Object strategy = createDefaultStrategy(context, clazz);
            strategies.add((T) strategy);
        }
        return strategies;
    }
    return Collections.emptyList();
}
```

### 4. DispatcherServlet.properties 내용 (Spring MVC 소스)

```properties
# org/springframework/web/servlet/DispatcherServlet.properties

org.springframework.web.servlet.LocaleResolver=\
    org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver

org.springframework.web.servlet.ThemeResolver=\
    org.springframework.web.servlet.theme.FixedThemeResolver

org.springframework.web.servlet.HandlerMapping=\
    org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
    org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping,\
    org.springframework.web.servlet.function.support.RouterFunctionMapping

org.springframework.web.servlet.HandlerAdapter=\
    org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
    org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
    org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter,\
    org.springframework.web.servlet.function.support.HandlerFunctionAdapter

org.springframework.web.servlet.HandlerExceptionResolver=\
    org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver,\
    org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
    org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver

org.springframework.web.servlet.RequestToViewNameTranslator=\
    org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator

org.springframework.web.servlet.ViewResolver=\
    org.springframework.web.servlet.view.InternalResourceViewResolver

org.springframework.web.servlet.FlashMapManager=\
    org.springframework.web.servlet.support.SessionFlashMapManager
```

### 5. Spring Boot WebMvcAutoConfiguration이 Bean을 미리 등록하는 방식

```java
// WebMvcAutoConfiguration.java (Spring Boot)
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class })
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
public class WebMvcAutoConfiguration {

    // WebMvcAutoConfigurationAdapter가 실제 Bean 등록
    @Configuration(proxyBeanMethods = false)
    @Import(EnableWebMvcConfiguration.class)
    @EnableConfigurationProperties({ WebMvcProperties.class, WebProperties.class })
    public static class WebMvcAutoConfigurationAdapter implements WebMvcConfigurer {
        // ...
    }
}

// EnableWebMvcConfiguration → DelegatingWebMvcConfiguration → WebMvcConfigurationSupport
// WebMvcConfigurationSupport.java — 핵심 MVC Bean 등록
@Bean
@Override
public RequestMappingHandlerMapping requestMappingHandlerMapping(...) {
    RequestMappingHandlerMapping mapping = createRequestMappingHandlerMapping();
    mapping.setOrder(0);
    // ... 설정
    return mapping;
}

@Bean
@Override
public RequestMappingHandlerAdapter requestMappingHandlerAdapter(...) {
    RequestMappingHandlerAdapter adapter = createRequestMappingHandlerAdapter();
    // ArgumentResolver, ReturnValueHandler 등록
    // ...
    return adapter;
}
```

```
결과적인 초기화 흐름:

Spring Boot 기동
  → WebMvcAutoConfiguration 처리
  → ApplicationContext에 RequestMappingHandlerMapping, HandlerAdapter 등 Bean 등록
  → ApplicationContext.refresh() 완료
  → DispatcherServlet.onRefresh() 호출
  → initStrategies()
  → initHandlerMappings(): ApplicationContext에서 이미 등록된 Bean 수집 (default 전략 필요 없음)
  → this.handlerMappings 세팅 완료
```

---

## 💻 실험으로 확인하기

### 실험 1: initStrategies 후 등록된 컴포넌트 목록 출력

```java
@Component
public class MvcComponentPrinter implements ApplicationContextAware {

    @Autowired
    private DispatcherServlet dispatcherServlet;

    @EventListener(ContextRefreshedEvent.class)
    public void printAfterRefresh() {
        // DispatcherServlet의 private 필드를 리플렉션으로 접근
        try {
            Field handlerMappingsField =
                DispatcherServlet.class.getDeclaredField("handlerMappings");
            handlerMappingsField.setAccessible(true);
            List<?> handlerMappings = (List<?>) handlerMappingsField.get(dispatcherServlet);

            System.out.println("=== 등록된 HandlerMapping ===");
            if (handlerMappings != null) {
                handlerMappings.forEach(hm ->
                    System.out.println("  " + hm.getClass().getSimpleName()
                        + " (order=" + ((Ordered) hm).getOrder() + ")"));
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

```
출력 예시 (Spring Boot 기본):
  === 등록된 HandlerMapping ===
    RouterFunctionMapping          (order=-1)
    RequestMappingHandlerMapping   (order=0)
    BeanNameUrlHandlerMapping      (order=2)
    WelcomePageHandlerMapping      (order=2147483646)
    WelcomePageNotFoundHandlerMapping (order=2147483647)
```

### 실험 2: 초기화 로그 활성화

```yaml
logging:
  level:
    org.springframework.web.servlet.DispatcherServlet: TRACE
```

```
[DispatcherServlet] - Detected AcceptHeaderLocaleResolver
[DispatcherServlet] - Detected FixedThemeResolver
[DispatcherServlet] - Detected [RouterFunctionMapping@6a, order=-1]
[DispatcherServlet] - Detected [RequestMappingHandlerMapping@7b, order=0]
[DispatcherServlet] - Detected [BeanNameUrlHandlerMapping@8c, order=2]
[DispatcherServlet] - Detected [HttpRequestHandlerAdapter@9d]
[DispatcherServlet] - Detected [RequestMappingHandlerAdapter@ae]
[DispatcherServlet] - Detected [ExceptionHandlerExceptionResolver@bf, order=0]
[DispatcherServlet] - Detected [ResponseStatusExceptionResolver@c0, order=1]
[DispatcherServlet] - Detected [DefaultHandlerExceptionResolver@d1, order=2]
[DispatcherServlet] - Completed initialization in 312 ms
```

### 실험 3: detectAllHandlerMappings=false 효과 확인

```java
// DispatcherServlet에 플래그 설정
@Bean
public DispatcherServletRegistrationBean dispatcherServletRegistration(
        DispatcherServlet dispatcherServlet) {
    dispatcherServlet.setDetectAllHandlerMappings(false); // 단일 Bean만 조회
    DispatcherServletRegistrationBean registration =
        new DispatcherServletRegistrationBean(dispatcherServlet, "/");
    return registration;
}

// 이 경우 "handlerMapping" 이름의 Bean 하나만 사용
// → RouterFunctionMapping, BeanNameUrlHandlerMapping 무시됨
// → @RequestMapping 만 동작하는 최소 구성
```

### 실험 4: DispatcherServlet.properties 직접 확인

```bash
# spring-webmvc JAR에서 파일 추출
find ~/.gradle/caches -name "spring-webmvc-*.jar" | head -1 | \
  xargs -I{} unzip -p {} org/springframework/web/servlet/DispatcherServlet.properties
```

---

## 🌐 HTTP 레벨 분석

```
초기화 완료 전 요청 도착 시:
  Tomcat: load-on-startup 값에 따라 처리가 다름

  load-on-startup < 0 (lazy):
    첫 요청이 도착할 때 init() 호출 시작
    → 초기화 완료까지 해당 요청 스레드 블로킹
    → 첫 요청 응답 시간 = 초기화 시간 + 처리 시간

  load-on-startup >= 0 (eager):
    Tomcat 시작 시 init() 완료
    → 첫 요청부터 즉시 처리
    → Spring Boot 기본값: 1 (항상 eager 초기화)

Spring Boot application.yml:
  spring:
    mvc:
      servlet:
        load-on-startup: 1  # 기본값
```

---

## 🤔 트레이드오프

```
detectAllHandlerMappings=true (기본):
  장점  ApplicationContext의 모든 HandlerMapping 자동 수집
        → 라이브러리가 자체 HandlerMapping 추가 가능 (Router Function 등)
  단점  Bean이 많으면 탐색 비용 증가 (실제로는 미미)
        → 순서 충돌 가능성 (Order 값 관리 필요)

false:
  장점  명시적이고 단순한 설정
  단점  하나의 HandlerMapping만 동작
        → 라이브러리 추가 HandlerMapping 무시됨

default 전략 (DispatcherServlet.properties) 의존:
  장점  Bean 없어도 기본 동작 보장
  단점  커스터마이징 어려움
        → 그러나 Spring Boot 환경에서는 항상 Bean으로 등록되므로 default 전략 사용되지 않음
```

---

## 📌 핵심 정리

```
onRefresh() 호출 시점
  ApplicationContext.refresh() 완료 → ContextRefreshedEvent
  → FrameworkServlet이 이벤트 수신 → onRefresh() 호출
  → Bean 등록 완료 후 호출 보장

initStrategies() 9가지 컴포넌트
  MultipartResolver, LocaleResolver, ThemeResolver
  HandlerMappings (복수), HandlerAdapters (복수)
  HandlerExceptionResolvers (복수)
  ViewNameTranslator, ViewResolvers (복수), FlashMapManager

컴포넌트 수집 방식 (HandlerMapping 기준)
  detectAllHandlerMappings=true: 타입으로 전체 Bean 수집 → 정렬
  detectAllHandlerMappings=false: "handlerMapping" 이름 단일 Bean
  Bean 없음: DispatcherServlet.properties default 전략 (static 블록에서 로딩)

Spring Boot에서 default 전략이 사용 안 되는 이유
  WebMvcAutoConfiguration → WebMvcConfigurationSupport
  → RequestMappingHandlerMapping, RequestMappingHandlerAdapter 등 Bean 등록
  → initStrategies()에서 이미 Bean을 찾으므로 properties fallback 불필요
```

---

## 🤔 생각해볼 문제

**Q1.** `DispatcherServlet.properties`의 default 전략 인스턴스는 `createDefaultStrategy(context, clazz)`를 통해 생성됩니다. 이렇게 생성된 인스턴스는 일반 Spring Bean과 어떤 차이가 있는가? `@Autowired`나 `@Value` 주입이 동작하는가?

**Q2.** 하나의 애플리케이션에서 두 개의 `DispatcherServlet`이 동일한 `WebApplicationContext`를 공유할 경우, `initStrategies()`가 두 번 호출됩니다. 이것이 문제가 되는 경우와 안 되는 경우는?

**Q3.** `detectAllHandlerMappings=true`(기본값)일 때, 커스텀 라이브러리가 `HandlerMapping` Bean을 `order=Integer.MIN_VALUE`로 등록했다면 어떤 일이 발생하는가?

> 💡 **해설**
>
> **Q1.** `createDefaultStrategy()`는 내부적으로 `AutowireCapableBeanFactory.createBean(clazz)`을 호출합니다. 이는 `@Autowired`, `@Value` 등 의존성 주입과 BeanPostProcessor(예: `@PostConstruct`) 처리가 모두 수행됩니다. 따라서 일반 Bean처럼 DI가 동작합니다. 단, `BeanDefinition`이 등록되지 않으므로 `@Qualifier`, Bean 이름 기반 조회, `ApplicationContext.getBean()`으로 가져올 수는 없습니다.
>
> **Q2.** 문제가 되는 경우: `initStrategies()`는 `this.handlerMappings`와 같은 DispatcherServlet 인스턴스 필드를 업데이트하므로 두 번 호출돼도 DispatcherServlet 인스턴스가 다르면 서로 독립적입니다. 문제가 생기는 것은 `HandlerMapping` 같은 Bean 내부에 상태가 있을 때입니다. 예를 들어 `RequestMappingHandlerMapping`은 처음 초기화 시 컨트롤러를 스캔하는데, 공유 컨텍스트를 쓰면 이미 초기화된 Bean을 재사용하므로 실제로는 문제없습니다. 단, 동일 Bean의 `afterPropertiesSet()`이 두 번 호출되는 것을 Bean이 지원하지 않으면 문제가 될 수 있습니다.
>
> **Q3.** `RequestMappingHandlerMapping`(order=0)보다 먼저 처리됩니다. 모든 요청이 커스텀 HandlerMapping을 먼저 거치게 됩니다. 커스텀 HandlerMapping이 `null`을 반환하면 다음 HandlerMapping으로 넘어가므로 정상 동작합니다. 하지만 커스텀 HandlerMapping이 모든 요청에 대해 항상 핸들러를 반환하면 `@RequestMapping` 매핑이 전혀 동작하지 않는 상황이 발생합니다. 따라서 라이브러리 제공자는 `order` 값을 신중하게 선택해야 하며, `Ordered.LOWEST_PRECEDENCE - 1` 같은 높은 값(낮은 우선순위)을 쓰는 것이 일반적입니다.

---

<div align="center">

**[⬅️ 이전: Servlet과 Front Controller 패턴](./01-front-controller-pattern.md)** | **[홈으로 🏠](../README.md)** | **[다음: WebApplicationContext vs RootApplicationContext ➡️](./03-web-app-context-hierarchy.md)**

</div>
