# WebApplicationContext vs RootApplicationContext — 두 컨텍스트가 공존하는 이유

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `RootApplicationContext`와 `WebApplicationContext`는 각각 무엇을 담아야 하는가?
- 자식 컨텍스트가 부모 컨텍스트의 Bean을 찾을 수 있는 원리는?
- 반대로 부모 컨텍스트가 자식의 Bean을 조회하지 못하는 이유는 무엇이고, 이 설계가 왜 의도적인가?
- Spring Boot에서 두 컨텍스트 계층 구조가 어떻게 달라졌는가?
- `@Transactional`이 Service Bean에서 동작하려면 컨텍스트 계층이 어떻게 설정되어야 하는가?
- `@Controller`와 `@Service`를 같은 컨텍스트에 넣으면 안 되는 실제 이유는?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: 하나의 ApplicationContext에 모든 Bean을 넣으면 생기는 문제

```
전통적인 Spring MVC 애플리케이션 시나리오:
  하나의 웹 애플리케이션에서
  - DispatcherServlet-A: /api/* 처리 (REST API)
  - DispatcherServlet-B: /admin/* 처리 (관리자 화면)
  - 두 Servlet이 공통으로 사용하는 Service, Repository 계층

문제:
  단일 컨텍스트로 구성 시:
  → DispatcherServlet-A와 B가 같은 컨텍스트 공유
  → Controller Bean이 서로의 컨텍스트에서 보임
  → A용 @ExceptionHandler가 B 요청에도 적용될 수 있음

  완전 분리 구성 시:
  → 각 Servlet이 독립 컨텍스트 보유
  → UserService Bean이 두 컨텍스트에 각각 존재
  → 트랜잭션, 캐시 상태 공유 불가
  → Bean 중복으로 메모리 낭비

해결: 계층 구조
  공통 Bean (Service, Repository) → RootApplicationContext (부모)
  Servlet 전용 Bean (Controller) → WebApplicationContext (자식)
  → 자식은 부모 Bean 접근 가능
  → 부모는 자식 Bean 모름 (격리)
```

---

## 😱 흔한 오해 또는 실수

### Before: @Service Bean이 WebApplicationContext에 등록되어 AOP가 적용 안 됨

```java
// ❌ 잘못된 ComponentScan 설정 (전통적인 web.xml 방식)

// RootApplicationContext 설정 (applicationContext.xml)
// → 스캔 대상: com.example.service, com.example.repository
// → @Transactional, @Cacheable 등 AOP 프록시 생성

// WebApplicationContext 설정 (dispatcher-servlet.xml)
// → 스캔 대상: com.example  ← 너무 넓음!
// → com.example.service도 스캔 → WebApplicationContext에도 UserService 등록!

// 결과:
// RootApplicationContext: UserService (AOP 프록시 적용됨)
// WebApplicationContext: UserService (AOP 프록시 미적용!)
// DispatcherServlet은 자식(WebApplicationContext)에서 먼저 찾음
// → AOP 프록시 없는 UserService를 Controller에 주입
// → @Transactional 동작 안 함!
```

```java
// ✅ 올바른 설정 (ComponentScan 범위 분리)

// RootApplicationContext
@ComponentScan(basePackages = "com.example",
    excludeFilters = @Filter(type = FilterType.ANNOTATION,
        classes = {Controller.class, RestController.class}))
// → Service, Repository만 스캔

// WebApplicationContext
@ComponentScan(basePackages = "com.example",
    includeFilters = @Filter(type = FilterType.ANNOTATION,
        classes = {Controller.class, RestController.class}),
    useDefaultFilters = false)
// → Controller만 스캔
```

### Before: Spring Boot도 두 컨텍스트가 있을 것이라 생각한다

```
❌ 잘못된 이해:
  "Spring Boot도 전통 MVC처럼 Root + WebApplicationContext 두 개다"

✅ 실제:
  Spring Boot 단일 DispatcherServlet 시나리오:
  → 하나의 ApplicationContext만 생성
    (AnnotationConfigServletWebServerApplicationContext)
  → 이 컨텍스트가 Root 역할도 하고 WebApplicationContext 역할도 함
  → DispatcherServlet이 이 단일 컨텍스트를 사용

  SpringApplication.run() 결과로 반환되는 ctx:
    ctx instanceof WebApplicationContext → true
    ctx.getParent() → null (부모 없음)
    WebApplicationContextUtils.getWebApplicationContext(servletContext) → 동일 ctx
```

---

## ✨ 올바른 이해와 사용

### After: 계층 구조의 Bean 탐색 위임 방향

```java
// HierarchicalBeanFactory.java — 계층 구조의 핵심 인터페이스
public interface HierarchicalBeanFactory extends BeanFactory {
    // 부모 BeanFactory 참조 보유
    BeanFactory getParentBeanFactory();

    // 현재 컨텍스트에만 있는지 확인 (부모 제외)
    boolean containsLocalBean(String name);
}

// AbstractBeanFactory.getBean() — 탐색 방향
protected <T> T doGetBean(String name, ...) {
    // 1. 현재 컨텍스트에서 먼저 탐색
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null) {
        return (T) getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }

    // 2. 현재 컨텍스트에 없으면 부모에게 위임
    BeanFactory parentBeanFactory = getParentBeanFactory();
    if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
        // 부모에게 위임 → 부모도 없으면 NoSuchBeanDefinitionException
        return parentBeanFactory.getBean(nameToLookup, requiredType);
    }
    // ...
}
```

```
Bean 탐색 방향 요약:

  WebApplicationContext (자식)에서 UserService 요청:
    1. 자식 컨텍스트 탐색 → UserService 없음
    2. 부모(RootApplicationContext) 탐색 → UserService 있음 ✅

  RootApplicationContext (부모)에서 UserController 요청:
    1. 부모 컨텍스트 탐색 → UserController 없음
    2. 부모는 자식 참조 없음 → NoSuchBeanDefinitionException ❌

  → 단방향 의존: 자식 → 부모 가능 / 부모 → 자식 불가
  → Service → Controller 의존 방지 (레이어 위반 설계적 차단)
```

---

## 🔬 내부 동작 원리

### 1. ContextLoaderListener — RootApplicationContext 생성

```java
// ContextLoaderListener.java (전통적 web.xml 방식)
public class ContextLoaderListener extends ContextLoader
        implements ServletContextListener {

    @Override
    public void contextInitialized(ServletContextEvent event) {
        // RootApplicationContext 생성 및 초기화
        initWebApplicationContext(event.getServletContext());
    }
}

// ContextLoader.initWebApplicationContext()
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
    // 1. RootApplicationContext 생성
    this.context = createWebApplicationContext(servletContext);

    // 2. configureAndRefreshWebApplicationContext
    //    → web.xml의 contextConfigLocation 파라미터 읽기
    //    → Bean 등록 완료
    configureAndRefreshWebApplicationContext(cwac, servletContext);

    // 3. ServletContext에 속성으로 저장 (자식 컨텍스트가 나중에 참조할 수 있도록)
    servletContext.setAttribute(
        WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE,
        this.context);

    return this.context;
}
```

### 2. FrameworkServlet — WebApplicationContext 생성 및 부모 연결

```java
// FrameworkServlet.initWebApplicationContext()
protected WebApplicationContext initWebApplicationContext() {

    // ServletContext에서 RootApplicationContext 조회
    WebApplicationContext rootContext =
        WebApplicationContextUtils.getWebApplicationContext(getServletContext());
    // → servletContext.getAttribute(ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE)

    WebApplicationContext wac = null;

    if (this.webApplicationContext != null) {
        // 생성자에서 주입된 경우
        wac = this.webApplicationContext;
        if (wac instanceof ConfigurableWebApplicationContext cwac
                && !cwac.isActive()) {
            if (cwac.getParent() == null) {
                // ← 핵심: 부모를 RootApplicationContext로 설정
                cwac.setParent(rootContext);
            }
            configureAndRefreshWebApplicationContext(cwac);
        }
    }

    if (wac == null) {
        // ServletContext에서 기존 WebApplicationContext 탐색
        wac = findWebApplicationContext();
    }

    if (wac == null) {
        // 새로 생성
        wac = createWebApplicationContext(rootContext);  // rootContext를 부모로 전달
    }

    // ...
    return wac;
}

// createWebApplicationContext — 부모 설정
protected WebApplicationContext createWebApplicationContext(@Nullable ApplicationContext parent) {
    Class<?> contextClass = getContextClass();
    ConfigurableWebApplicationContext wac =
        (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);

    wac.setEnvironment(getEnvironment());
    wac.setParent(parent);  // ← RootApplicationContext를 부모로 설정
    // ...
    return wac;
}
```

### 3. Spring Boot에서의 단일 컨텍스트 구조

```java
// SpringApplication.java
protected ConfigurableApplicationContext createApplicationContext() {
    // 웹 환경이면 AnnotationConfigServletWebServerApplicationContext 생성
    return this.applicationContextFactory.create(this.webApplicationType);
}

// AnnotationConfigServletWebServerApplicationContext:
//   - ApplicationContext 기능: Bean 관리
//   - WebApplicationContext 기능: ServletContext 접근
//   - 내장 서버 시작: TomcatServletWebServerFactory 생성
//
// 부모 컨텍스트 없음 (단일 계층)
// DispatcherServlet이 이 컨텍스트 그대로 사용

// DispatcherServletAutoConfiguration이 이 컨텍스트를 DispatcherServlet에 주입
@Bean
public DispatcherServlet dispatcherServlet(WebMvcProperties webMvcProperties) {
    DispatcherServlet dispatcherServlet = new DispatcherServlet();
    // ApplicationContext는 FrameworkServlet.setApplicationContext()로 주입됨
    return dispatcherServlet;
}
```

### 4. 컨텍스트 계층에서 AOP 프록시가 교체되는 문제 재현

```java
// 문제 재현 코드
@Configuration
public class RootConfig {
    @ComponentScan(basePackages = "com.example.service") // UserService 스캔
}

@Configuration
@EnableTransactionManagement  // RootContext에서만 활성화
public class TransactionConfig { ... }

@Configuration
public class WebConfig {
    // ❌ 잘못된 설정: 너무 넓은 스캔 범위
    @ComponentScan(basePackages = "com.example")
    // → com.example.service.UserService도 WebContext에 등록됨!
    // → 이 UserService는 @EnableTransactionManagement 없는 컨텍스트 소속
    // → AOP 프록시 적용 안 됨
}

// 진단 코드:
@RestController
public class DiagController {
    @Autowired
    private UserService userService;

    @GetMapping("/diag")
    public String check() {
        // AOP 프록시면 CGLIB 프록시 클래스명 출력
        return userService.getClass().getName();
        // 정상: "com.example.service.UserService$$SpringCGLIB$$0"
        // 비정상: "com.example.service.UserService"
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: Spring Boot에서 컨텍스트 계층 확인

```java
@RestController
public class ContextController implements ApplicationContextAware {

    private ApplicationContext context;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.context = ctx;
    }

    @GetMapping("/context-info")
    public Map<String, Object> contextInfo() {
        Map<String, Object> info = new LinkedHashMap<>();
        info.put("contextClass", context.getClass().getSimpleName());
        info.put("parent", context.getParent() != null
            ? context.getParent().getClass().getSimpleName()
            : "없음 (단일 컨텍스트)");
        info.put("isWebApplicationContext", context instanceof WebApplicationContext);

        // 서블릿 컨텍스트에 등록된 Root 컨텍스트 확인
        if (context instanceof WebApplicationContext wac) {
            WebApplicationContext root = WebApplicationContextUtils
                .getWebApplicationContext(wac.getServletContext());
            info.put("rootContext", root == context ? "동일한 컨텍스트" : "별도 Root 존재");
        }
        return info;
    }
}
```

```json
// Spring Boot 응답:
{
  "contextClass": "AnnotationConfigServletWebServerApplicationContext",
  "parent": "없음 (단일 컨텍스트)",
  "isWebApplicationContext": true,
  "rootContext": "동일한 컨텍스트"
}
```

### 실험 2: 자식에서 부모 Bean 접근, 부모에서 자식 Bean 접근 비교

```java
// 두 개의 컨텍스트를 직접 만들어 계층 확인
AnnotationConfigApplicationContext parent = new AnnotationConfigApplicationContext();
parent.register(ParentConfig.class);  // ServiceBean 등록
parent.refresh();

AnnotationConfigApplicationContext child = new AnnotationConfigApplicationContext();
child.setParent(parent);  // ← 부모 설정
child.register(ChildConfig.class);  // ControllerBean 등록
child.refresh();

// 자식 → 부모: 가능
ServiceBean fromChild = child.getBean(ServiceBean.class);  // ✅ 부모에서 찾음

// 부모 → 자식: 불가
try {
    ControllerBean fromParent = parent.getBean(ControllerBean.class);  // ❌
} catch (NoSuchBeanDefinitionException e) {
    System.out.println("부모에서 자식 Bean 접근 불가: " + e.getMessage());
}
```

---

## 🌐 HTTP 레벨 분석

```
요청 처리 시 컨텍스트 접근 흐름:

GET /users/1 → DispatcherServlet

DispatcherServlet이 보유한 WebApplicationContext:
  → getHandler() 호출 시 HandlerMapping Bean 조회
  → HandlerMapping은 WebApplicationContext에서 UserController Bean 탐색
  → UserController.getUser() 실행 중 UserService 필요
  → UserService는 이미 @Autowired로 주입된 상태
    (ApplicationContext 초기화 시점에 자식이 부모에서 찾아 주입)
  → 런타임 요청 처리 중에는 컨텍스트 탐색 없음

컨텍스트 접근이 런타임에 일어나는 예외 상황:
  → ApplicationContext.getBean() 직접 호출 (안티패턴)
  → @Scope("request") Bean 조회 → 요청마다 새 인스턴스 생성
  → @Lazy Bean 최초 접근
```

---

## 🤔 트레이드오프

```
계층 구조 (Root + Web) 방식:
  장점  관심사 분리: Service/Repository vs Controller 격리
        여러 DispatcherServlet이 공통 Bean 공유 가능
        부모 컨텍스트에서 트랜잭션, 보안 AOP 적용 집중
  단점  ComponentScan 설정 실수로 AOP 프록시 교체 문제 발생
        설정이 두 군데 나뉘어 관리 복잡
        Spring Boot에서는 실질적으로 필요 없음

단일 컨텍스트 (Spring Boot 방식):
  장점  설정 단순, 실수 가능성 낮음
        단일 DispatcherServlet 애플리케이션에 최적
  단점  여러 DispatcherServlet이 동일 Bean 공유 시
        Controller Bean이 서로에게 노출됨
        → 실제로 Spring Boot에서 복수 DispatcherServlet 사용 사례 드묾
```

---

## 📌 핵심 정리

```
컨텍스트 계층 구조 (전통 방식)

  RootApplicationContext (ContextLoaderListener 생성)
    ├── Service, Repository, DataSource Bean
    ├── @Transactional, @Cacheable AOP 프록시
    └── 여러 DispatcherServlet이 공유

  WebApplicationContext (DispatcherServlet 자식 컨텍스트)
    ├── Controller, HandlerMapping, HandlerAdapter Bean
    ├── 부모 Bean에 접근 가능 (단방향)
    └── 각 DispatcherServlet이 독립적으로 보유

Bean 탐색 방향
  자식 → 자식 → 부모 → 부모의 부모 ... (한 방향)
  부모 → 자식 불가 (설계적 레이어 위반 차단)

Spring Boot
  단일 AnnotationConfigServletWebServerApplicationContext
  부모 컨텍스트 없음 (getParent() == null)
  Root + Web 역할을 하나의 컨텍스트가 담당

ComponentScan 범위 실수 → AOP 프록시 교체 문제
  Service가 WebContext에도 등록 → @Transactional 미적용
  진단: proxy class 이름에 $$SpringCGLIB$$ 포함 여부 확인
```

---

## 🤔 생각해볼 문제

**Q1.** 전통적인 Spring MVC 설정에서 `@Transactional`이 붙은 `UserService`가 `RootApplicationContext`와 `WebApplicationContext`에 모두 등록됐을 때, `@Controller`에 주입되는 것은 어느 컨텍스트의 Bean인가? 트랜잭션은 정상 동작하는가?

**Q2.** Spring Boot에서 단일 컨텍스트를 사용하면 `@Controller`와 `@Service`가 같은 컨텍스트에 있습니다. `@Service`가 `@Controller` Bean을 `@Autowired`로 주입받을 수 있는가? 할 수 있다면 해야 하는가?

**Q3.** `WebApplicationContextUtils.getRequiredWebApplicationContext(servletContext)`는 어떤 컨텍스트를 반환하는가? `DispatcherServlet`이 여러 개일 때 각 Servlet의 `WebApplicationContext`를 가져오려면 어떻게 해야 하는가?

> 💡 **해설**
>
> **Q1.** `@Controller`에 주입되는 것은 `WebApplicationContext`의 Bean입니다. Bean 탐색 시 자신의 컨텍스트를 먼저 확인하고, 거기서 찾으면 부모로 올라가지 않기 때문입니다. `WebApplicationContext`의 `UserService`는 `@EnableTransactionManagement`가 `RootApplicationContext`에만 설정되어 있다면 AOP 프록시가 적용되지 않아 `@Transactional`이 동작하지 않습니다. 해결책은 `WebApplicationContext`의 ComponentScan에서 Service 계층을 제외하거나, `@EnableTransactionManagement`를 `WebApplicationContext` 설정에도 추가하는 것입니다.
>
> **Q2.** 기술적으로는 가능합니다. 같은 컨텍스트에 있으므로 `@Autowired`로 `@Controller` Bean을 `@Service`에 주입할 수 있습니다. 그러나 해서는 안 됩니다. MVC 레이어 아키텍처에서 Service는 Controller를 알아선 안 됩니다(단방향 의존). 이를 허용하면 순환 의존 가능성, 테스트 어려움, 레이어 결합도 증가가 발생합니다. 전통 방식에서 계층 분리가 이 실수를 구조적으로 막아주는 효과가 있었습니다.
>
> **Q3.** `WebApplicationContextUtils.getRequiredWebApplicationContext(servletContext)`는 `servletContext.getAttribute(ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE)`를 반환합니다. 즉 Root 컨텍스트를 반환합니다. 각 `DispatcherServlet`의 자식 컨텍스트는 `servletContext.getAttribute(FrameworkServlet.SERVLET_CONTEXT_PREFIX + servletName)`에 저장됩니다. 따라서 특정 Servlet의 컨텍스트를 가져오려면 `servletContext.getAttribute("org.springframework.web.servlet.FrameworkServlet.CONTEXT." + servletName)`으로 조회해야 합니다.

---

<div align="center">

**[⬅️ 이전: DispatcherServlet 초기화 과정](./02-dispatcher-servlet-init.md)** | **[홈으로 🏠](../README.md)** | **[다음: doDispatch() 완전 분해 ➡️](./04-do-dispatch-analysis.md)**

</div>
