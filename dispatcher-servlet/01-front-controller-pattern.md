# Servlet과 Front Controller 패턴 — DispatcherServlet이 단일 진입점인 이유

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Java Servlet 스펙에서 HTTP 요청은 어떤 경로로 처리되는가?
- Servlet-per-URL 방식은 어떤 문제가 있어서 Front Controller 패턴이 등장했는가?
- `DispatcherServlet`은 Servlet 스펙 위에서 어떻게 단일 진입점을 구현하는가?
- `HttpServlet`, `HttpServletBean`, `FrameworkServlet`, `DispatcherServlet` 상속 계층 각 레이어의 책임은 무엇인가?
- `web.xml` 없이 `DispatcherServlet`이 자동 등록되는 메커니즘은?
- `DispatcherServlet`과 Spring의 `ApplicationContext`는 어떻게 연결되는가?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: Servlet-per-URL 방식의 폭발적 복잡도

Java 서블릿 스펙의 원래 모델은 URL마다 별도의 Servlet 클래스를 등록하는 방식이었습니다.

```xml
<!-- web.xml — Servlet-per-URL 방식 -->
<servlet>
    <servlet-name>UserServlet</servlet-name>
    <servlet-class>com.example.UserServlet</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>UserServlet</servlet-name>
    <url-pattern>/users</url-pattern>
</servlet-mapping>

<servlet>
    <servlet-name>OrderServlet</servlet-name>
    <servlet-class>com.example.OrderServlet</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>OrderServlet</servlet-name>
    <url-pattern>/orders</url-pattern>
</servlet-mapping>

<servlet>
    <servlet-name>ProductServlet</servlet-name>
    <servlet-class>com.example.ProductServlet</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>ProductServlet</servlet-name>
    <url-pattern>/products</url-pattern>
</servlet-mapping>

<!-- ... API 100개면 Servlet도 100개 -->
```

```
Servlet-per-URL 방식의 문제점:

  공통 처리 로직 중복
    → 인증 확인: 모든 Servlet에 동일 코드 반복
    → 로깅: 모든 Servlet에 동일 코드 반복
    → 예외 처리: 모든 Servlet에 동일 코드 반복
    → URL이 100개면 공통 로직도 100번 반복

  파라미터 파싱의 반복
    → request.getParameter("id") 매번 수동 파싱
    → JSON → Object 변환 매번 직접 구현
    → 타입 변환, 유효성 검사 매번 직접 처리

  URL 라우팅 정보가 설정 파일에 분산
    → web.xml이 수백 줄로 비대해짐
    → URL 변경 시 코드 + XML 동시 수정

  테스트 어려움
    → Servlet은 HttpServletRequest/Response 의존
    → 컨테이너 없이 단독 테스트 불가
```

### 해결: Front Controller 패턴

모든 HTTP 요청을 단 하나의 진입점(Front Controller)이 받아서, 내부적으로 올바른 핸들러에게 위임하는 패턴입니다.

```
Front Controller 패턴의 핵심:

  모든 요청          ┌──────────────────────────────┐
  → DispatcherServlet(Front Controller)           │
                   │                              │
                   ├─→ HandlerMapping (누가 처리?)  │
                   ├─→ HandlerAdapter (어떻게 실행?) │
                   ├─→ Controller Method (실제 로직) │
                   ├─→ ViewResolver (어떻게 응답?)   │
                   └──────────────────────────────┘

  공통 처리 (인증, 로깅, 예외처리) → 한 곳에서 일괄 처리
  URL 라우팅 정보 → 코드에 @RequestMapping으로 선언
  파라미터 파싱 → ArgumentResolver가 자동 처리
```

---

## 😱 흔한 오해 또는 실수

### Before: DispatcherServlet이 Servlet 스펙을 대체한다

```
❌ 잘못된 이해:
  "Spring MVC를 쓰면 Servlet 스펙은 관련 없다"
  "DispatcherServlet은 Spring이 만든 독자적인 요청 처리기다"

✅ 실제:
  DispatcherServlet은 javax.servlet.http.HttpServlet의 서브클래스
  → Servlet 컨테이너(Tomcat)가 DispatcherServlet을 등록하고 관리
  → 요청이 오면 Tomcat이 service() 메서드를 호출
  → DispatcherServlet은 그 위에서 Spring의 추가 기능을 제공
  → Servlet 스펙을 대체하는 게 아니라 확장하는 것
```

### Before: `@RestController` 메서드가 Servlet이 직접 호출된다

```java
// ❌ 잘못된 이해
@RestController
public class UserController {
    // "이 메서드가 HTTP 요청을 직접 받는다"
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }
}

// ✅ 실제 호출 경로:
// Tomcat → DispatcherServlet.service()
//        → DispatcherServlet.doDispatch()
//           → HandlerMapping → HandlerAdapter
//              → InvocableHandlerMethod.invokeForRequest()
//                 → UserController.getUser() ← 여기서야 비로소 호출됨
```

### Before: DispatcherServlet이 모든 URL을 처리해야 한다

```xml
<!-- ❌ 흔한 설정 실수: 정적 리소스도 DispatcherServlet이 처리하려 함 -->
<servlet-mapping>
    <servlet-name>dispatcher</servlet-name>
    <url-pattern>/*</url-pattern>  <!-- 모든 요청 — 정적 리소스도 404 발생! -->
</servlet-mapping>

<!-- ✅ 올바른 설정: REST API는 / 로, 정적 리소스는 default servlet에 위임 -->
<servlet-mapping>
    <servlet-name>dispatcher</servlet-name>
    <url-pattern>/</url-pattern>  <!-- / 는 default servlet을 오버라이드, /* 와 다름 -->
</servlet-mapping>
```

```
/  vs /*  차이:
  /  → default servlet을 오버라이드 (JSP 처리 제외)
       → DispatcherServlet이 처리 못 하면 default servlet으로 폴백 가능
  /* → 모든 요청 매핑 (JSP 포함)
       → JSP, 정적 리소스 모두 DispatcherServlet이 받아버림
       → 처리할 핸들러 없으면 그냥 404
```

---

## ✨ 올바른 이해와 사용

### After: DispatcherServlet은 Servlet 위에 올라탄 Spring의 Front Controller

```
Servlet 컨테이너(Tomcat) 레이어:
  소켓 수신 → HTTP 파싱 → HttpServletRequest/Response 생성
  → 등록된 Servlet 중 URL 패턴 일치하는 것 선택
  → Servlet.service() 호출

DispatcherServlet 레이어 (Servlet 스펙 구현):
  service() → processRequest() → doDispatch()
  → Spring MVC 컴포넌트 체인 실행

Spring MVC 컴포넌트 레이어:
  HandlerMapping → HandlerAdapter → ArgumentResolver
  → Controller Method → ReturnValueHandler → ViewResolver
```

```java
// DispatcherServlet 상속 계층 — 각 레이어의 책임
javax.servlet.http.HttpServlet
  ↑ service(req, res) → doGet() / doPost() ... 분기 처리
  
org.springframework.web.servlet.HttpServletBean
  ↑ init() 오버라이드 → ServletConfig의 init-param을 Bean 프로퍼티에 바인딩

org.springframework.web.servlet.FrameworkServlet
  ↑ initServletBean() → WebApplicationContext 초기화
  ↑ service() 오버라이드 → PATCH 지원 추가, processRequest() 위임
  ↑ processRequest() → LocaleContextHolder, RequestContextHolder 설정
                     → doService() 호출 (추상 메서드)

org.springframework.web.servlet.DispatcherServlet
  ↑ onRefresh() → Spring MVC 컴포넌트 초기화 (9가지)
  ↑ doService() → request에 컨텍스트 속성 세팅 후 doDispatch() 호출
  ↑ doDispatch() → 실제 요청 처리 핵심 로직
```

---

## 🔬 내부 동작 원리

### 1. Servlet 컨테이너가 DispatcherServlet을 초기화하는 과정

```
Tomcat 시작
  → web.xml 또는 ServletContainerInitializer 읽기
  → DispatcherServlet 인스턴스 생성
  → Servlet.init(ServletConfig) 호출
     ↓
HttpServletBean.init()
  → PropertyValues로 init-param을 Bean 프로퍼티에 바인딩
  → initServletBean() 호출 (추상 메서드 위임)
     ↓
FrameworkServlet.initServletBean()
  → initWebApplicationContext() 호출
     ↓
FrameworkServlet.initWebApplicationContext()
  → 부모 WebApplicationContext 탐색 (RootApplicationContext)
  → 자신의 WebApplicationContext 생성 또는 조회
  → wac.refresh() 호출 → Bean 등록 완료
  → onRefresh(wac) 호출
     ↓
DispatcherServlet.onRefresh()
  → initStrategies(context) 호출
     ↓
DispatcherServlet.initStrategies()
  → initMultipartResolver()        // 파일 업로드
  → initLocaleResolver()           // 국제화
  → initThemeResolver()            // 테마
  → initHandlerMappings()          // ← URL → 핸들러 매핑
  → initHandlerAdapters()          // ← 핸들러 실행 방법
  → initHandlerExceptionResolvers() // ← 예외 처리
  → initRequestToViewNameTranslator()
  → initViewResolvers()            // ← View 선택
  → initFlashMapManager()
```

### 2. HTTP 요청이 들어왔을 때 Servlet 레이어 처리

```java
// FrameworkServlet.java — service() 오버라이드
@Override
protected void service(HttpServletRequest request, HttpServletResponse response)
        throws ServletException, IOException {

    // PATCH 메서드는 HttpServlet이 처리 못 함 → 직접 처리
    HttpMethod httpMethod = HttpMethod.resolve(request.getMethod());
    if (httpMethod == HttpMethod.PATCH || httpMethod == null) {
        processRequest(request, response);
    } else {
        // GET, POST, PUT, DELETE 등 → 부모 HttpServlet의 service() 위임
        // → doGet(), doPost() 등으로 분기
        super.service(request, response);
    }
}

// doGet() / doPost() 등은 모두 processRequest()로 수렴
@Override
protected final void doGet(HttpServletRequest request, HttpServletResponse response)
        throws ServletException, IOException {
    processRequest(request, response);
}

// processRequest() — 공통 처리 전/후 작업
protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
        throws ServletException, IOException {

    long startTime = System.currentTimeMillis();
    Throwable failureCause = null;

    // 이전 LocaleContext 백업 (중첩 요청 처리를 위해)
    LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
    LocaleContext localeContext = buildLocaleContext(request);

    // 이전 RequestAttributes 백업
    RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
    ServletRequestAttributes requestAttributes = buildRequestAttributes(request, response, previousAttributes);

    // ThreadLocal에 현재 요청 컨텍스트 등록
    // → 어느 레이어에서도 RequestContextHolder.getRequestAttributes()로 접근 가능
    initContextHolders(request, localeContext, requestAttributes);

    try {
        // 실제 처리 위임 — DispatcherServlet.doService()
        doService(request, response);
    } catch (ServletException | IOException ex) {
        failureCause = ex;
        throw ex;
    } finally {
        // ThreadLocal 정리 (메모리 누수 방지)
        resetContextHolders(request, previousLocaleContext, previousAttributes);
        // 요청 처리 완료 이벤트 발행
        publishRequestHandledEvent(request, response, startTime, failureCause);
    }
}
```

### 3. DispatcherServlet.doService() — doDispatch() 진입 직전

```java
// DispatcherServlet.java
@Override
protected void doService(HttpServletRequest request, HttpServletResponse response)
        throws Exception {

    logRequest(request);  // DEBUG 레벨 요청 로깅

    // request에 Spring MVC 컴포넌트 참조를 속성으로 저장
    // → 핸들러, 인터셉터 등이 request를 통해 컨텍스트에 접근 가능
    request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
    request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
    request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
    request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());

    // Flash attributes 처리 (리다이렉트 후 데이터 전달용)
    FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
    if (inputFlashMap != null) {
        request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
    }
    request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
    request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);

    try {
        // ← 핵심: 실제 요청 처리
        doDispatch(request, response);
    } finally {
        // ...
    }
}
```

### 4. Spring Boot에서 DispatcherServlet이 자동 등록되는 경로

```
@SpringBootApplication
  → @EnableAutoConfiguration
     → DispatcherServletAutoConfiguration (spring.factories)
        ↓
DispatcherServletAutoConfiguration
  → DispatcherServletConfiguration
     → @Bean DispatcherServlet 등록
  → DispatcherServletRegistrationConfiguration
     → @Bean DispatcherServletRegistrationBean 등록
        → urlMappings: "/"  (기본값)
        → ServletRegistrationBean이 Tomcat 시작 시 Servlet 등록

TomcatServletWebServerFactory
  → Tomcat 인스턴스 생성
  → ServletRegistrationBean.onStartup() 호출
  → DispatcherServlet을 Tomcat에 등록 완료
```

```java
// DispatcherServletAutoConfiguration.java (Spring Boot 소스)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@AutoConfiguration(after = ServletWebServerFactoryAutoConfiguration.class)
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass(DispatcherServlet.class)
public class DispatcherServletAutoConfiguration {

    @Configuration(proxyBeanMethods = false)
    @Conditional(DefaultDispatcherServletCondition.class)
    @ConditionalOnClass(ServletRegistration.class)
    @EnableConfigurationProperties(WebMvcProperties.class)
    protected static class DispatcherServletConfiguration {

        @Bean(name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
        public DispatcherServlet dispatcherServlet(WebMvcProperties webMvcProperties) {
            DispatcherServlet dispatcherServlet = new DispatcherServlet();
            // spring.mvc.dispatch-options-request 등 설정 적용
            dispatcherServlet.setDispatchOptionsRequest(
                webMvcProperties.isDispatchOptionsRequest());
            dispatcherServlet.setDispatchTraceRequest(
                webMvcProperties.isDispatchTraceRequest());
            dispatcherServlet.setThrowExceptionIfNoHandlerFound(
                webMvcProperties.isThrowExceptionIfNoHandlerFound());
            dispatcherServlet.setPublishEvents(
                webMvcProperties.isPublishRequestHandledEvents());
            dispatcherServlet.setEnableLoggingRequestDetails(
                webMvcProperties.isLogRequestDetails());
            return dispatcherServlet;
        }
    }

    @Configuration(proxyBeanMethods = false)
    @Conditional(DispatcherServletRegistrationCondition.class)
    @ConditionalOnClass(ServletRegistration.class)
    @EnableConfigurationProperties(WebMvcProperties.class)
    @Import(DispatcherServletConfiguration.class)
    protected static class DispatcherServletRegistrationConfiguration {

        @Bean(name = DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME)
        @ConditionalOnBean(value = DispatcherServlet.class,
                           name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
        public DispatcherServletRegistrationBean dispatcherServletRegistration(
                DispatcherServlet dispatcherServlet,
                WebMvcProperties webMvcProperties,
                ObjectProvider<MultipartConfigElement> multipartConfig) {
            DispatcherServletRegistrationBean registration =
                new DispatcherServletRegistrationBean(dispatcherServlet,
                    webMvcProperties.getServlet().getPath());  // 기본값: "/"
            registration.setName(DEFAULT_DISPATCHER_SERVLET_BEAN_NAME);
            registration.setLoadOnStartup(
                webMvcProperties.getServlet().getLoadOnStartup());
            multipartConfig.ifAvailable(registration::setMultipartConfig);
            return registration;
        }
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: DispatcherServlet 초기화 로그 확인

```yaml
# application.yml
logging:
  level:
    org.springframework.web.servlet.DispatcherServlet: TRACE
    org.springframework.web.servlet.FrameworkServlet: DEBUG
```

```
실행 시 출력되는 로그:
  DEBUG o.s.web.servlet.DispatcherServlet - Initializing servlet 'dispatcherServlet'
  TRACE o.s.web.servlet.DispatcherServlet - Detected AcceptHeaderLocaleResolver
  TRACE o.s.web.servlet.DispatcherServlet - Detected FixedThemeResolver
  TRACE o.s.web.servlet.DispatcherServlet - Detected [org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping@...]
  TRACE o.s.web.servlet.DispatcherServlet - Detected [org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping@...]
  TRACE o.s.web.servlet.DispatcherServlet - Detected [org.springframework.web.servlet.function.support.RouterFunctionMapping@...]
  TRACE o.s.web.servlet.DispatcherServlet - Detected [org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter@...]
  DEBUG o.s.web.servlet.DispatcherServlet - Completed initialization in 856 ms
```

### 실험 2: 브레이크포인트로 호출 스택 확인

```
IntelliJ에서 DispatcherServlet.doDispatch()에 브레이크포인트 설정 후
GET /users/1 요청 시 확인할 수 있는 스택 트레이스:

doDispatch(HttpServletRequest, HttpServletResponse)  ← 브레이크포인트
  doService(HttpServletRequest, HttpServletResponse)
    processRequest(HttpServletRequest, HttpServletResponse)
      service(HttpServletRequest, HttpServletResponse) [FrameworkServlet]
        service(ServletRequest, ServletResponse) [HttpServlet]
          internalDoFilter(ServletRequest, ServletResponse) [ApplicationFilterChain]
            doFilter(ServletRequest, ServletResponse) [ApplicationFilterChain]
              ...Filter 체인...
                connector.CoyoteAdapter.service(Request, Response) [Tomcat]
```

### 실험 3: RequestContextHolder로 어느 레이어에서나 요청 정보 접근

```java
// FrameworkServlet.processRequest()가 ThreadLocal에 등록해두기 때문에
// Service 레이어에서도 현재 요청 정보에 접근 가능
@Service
public class UserService {
    public void someMethod() {
        // Controller → Service 호출 중에도 접근 가능
        ServletRequestAttributes attrs =
            (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();

        if (attrs != null) {
            HttpServletRequest request = attrs.getRequest();
            System.out.println("현재 요청 URI: " + request.getRequestURI());
            System.out.println("클라이언트 IP: " + request.getRemoteAddr());
        }
    }
}
```

### 실험 4: 핸들러 없는 URL 요청 시 동작 차이 확인

```bash
# DispatcherServlet이 처리하는 URL 패턴: /

# Case 1: 핸들러 없는 API 경로 → 404
curl -v http://localhost:8080/api/nonexistent
# HTTP/1.1 404
# {"timestamp":...,"status":404,"error":"Not Found","path":"/api/nonexistent"}

# Case 2: 정적 리소스 경로 → ResourceHttpRequestHandler가 처리
curl -v http://localhost:8080/static/image.png
# (src/main/resources/static/image.png 파일이 있으면) HTTP/1.1 200

# Case 3: spring.mvc.throw-exception-if-no-handler-found=true 설정 시
# → NoHandlerFoundException 발생 → @ExceptionHandler로 처리 가능
```

---

## 🌐 HTTP 레벨 분석

### DispatcherServlet이 처리하는 요청의 헤더 흐름

```bash
# curl -v 로 전체 HTTP 대화 확인
curl -v -X GET http://localhost:8080/users/1 \
     -H "Accept: application/json" \
     -H "Authorization: Bearer token123"
```

```http
# 요청 (클라이언트 → 서버)
GET /users/1 HTTP/1.1
Host: localhost:8080
Accept: application/json
Authorization: Bearer token123

# 이 요청이 도달하는 경로:
# Tomcat NIO Connector (포트 8080 수신)
#   → HTTP/1.1 파싱
#   → HttpServletRequest 객체 생성
#     request.getMethod()      = "GET"
#     request.getRequestURI()  = "/users/1"
#     request.getHeader("Accept") = "application/json"
#   → URL 패턴 "/" 에 매핑된 DispatcherServlet 선택
#   → DispatcherServlet.service() 호출
```

```http
# 응답 (서버 → 클라이언트)
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 42
Date: Tue, 10 Mar 2026 00:00:00 GMT

{"id":1,"name":"홍길동","email":"hong@example.com"}

# Content-Type: application/json
# → HttpMessageConverter가 Accept 헤더를 보고 선택한 결과
# → DispatcherServlet → ReturnValueHandler → MessageConverter 체인
```

### Servlet 컨테이너와 DispatcherServlet의 책임 경계

```
Servlet 컨테이너(Tomcat)가 하는 것:
  ✅ TCP 소켓 관리 (연결 수락, 유지, 종료)
  ✅ HTTP 프로토콜 파싱 (요청 라인, 헤더, 바디)
  ✅ HttpServletRequest / HttpServletResponse 객체 생성
  ✅ 스레드 풀 관리 (요청당 스레드 할당)
  ✅ SSL/TLS 처리
  ✅ Servlet 생명주기 관리 (init, service, destroy)

DispatcherServlet(Spring MVC)이 하는 것:
  ✅ URL → 핸들러 메서드 라우팅
  ✅ 메서드 파라미터 자동 바인딩 (@RequestBody, @PathVariable 등)
  ✅ 응답 직렬화 (Object → JSON)
  ✅ 예외를 HTTP 응답으로 변환
  ✅ 인터셉터 체인 실행
  ✅ View 렌더링
```

---

## 🤔 트레이드오프

```
Front Controller 패턴의 장점:
  공통 처리 로직 중앙 집중
    → 인증, 로깅, 예외 처리를 한 곳에서 관리
  URL 라우팅 유연성
    → @RequestMapping으로 코드에 직접 선언
    → 정규식, 와일드카드, 조건부 매핑 가능
  테스트 용이성
    → MockMvc로 Servlet 컨테이너 없이 MVC 레이어 테스트
    → @WebMvcTest로 슬라이스 테스트

Front Controller 패턴의 단점:
  단일 진입점 = 단일 장애점
    → DispatcherServlet 초기화 실패 시 전체 웹 요청 불가
    → 복잡한 초기화 로직이 한 클래스에 집중됨
  HandlerMapping 체인 탐색 비용
    → 요청마다 등록된 HandlerMapping을 순서대로 탐색
    → 수천 개의 @RequestMapping이 있으면 탐색 시간 증가
    → 실제로는 해시맵 기반으로 최적화되어 있어 무시할 수준

Servlet-per-URL이 여전히 유리한 상황:
  특정 엔드포인트에 극한의 성능이 필요한 경우
    → 별도 Servlet으로 등록 시 Spring MVC 처리 체인 오버헤드 없음
    → Actuator의 일부 엔드포인트가 이 방식 활용
  DispatcherServlet과 다른 경로 처리가 필요한 경우
    → 헬스체크 전용 Servlet을 /health 에 직접 매핑
    → Spring Security 등 Filter 체인 우회 가능
```

---

## 📌 핵심 정리

```
DispatcherServlet 포지셔닝

  Servlet 스펙 (javax.servlet.http.HttpServlet)
    ↑ 확장
  HttpServletBean        : init-param → Bean 프로퍼티 바인딩
    ↑ 확장
  FrameworkServlet       : WebApplicationContext 초기화
                           processRequest() → ThreadLocal 관리
    ↑ 확장
  DispatcherServlet      : Spring MVC 9가지 컴포넌트 초기화 (onRefresh)
                           doDispatch() → 요청 처리 핵심 로직

Front Controller 패턴의 핵심 가치
  모든 HTTP 요청 → 단일 진입점 → 핸들러에게 위임
  공통 처리 중앙 집중 (인증·로깅·예외·인터셉터)
  @RequestMapping 기반 선언적 라우팅

Spring Boot 자동 등록 경로
  DispatcherServletAutoConfiguration
    → DispatcherServlet Bean 생성
    → DispatcherServletRegistrationBean으로 Tomcat에 등록
    → 기본 URL 패턴: "/"

processRequest()의 책임
  LocaleContextHolder ThreadLocal 설정/해제
  RequestContextHolder ThreadLocal 설정/해제
  → Service 레이어에서도 현재 요청 정보 접근 가능
  → 요청 처리 후 반드시 ThreadLocal 정리 (finally)
```

---

## 🤔 생각해볼 문제

**Q1.** `FrameworkServlet.processRequest()`는 finally 블록에서 반드시 `resetContextHolders()`를 호출합니다. 만약 이를 호출하지 않으면 어떤 문제가 발생하는가? 특히 스레드 풀 기반 서버 환경에서 구체적으로 어떤 증상이 나타날 수 있는가?

**Q2.** `DispatcherServlet`의 URL 패턴을 `/*` 대신 `/`로 설정하는 이유를 Tomcat의 `DefaultServlet`과의 관계로 설명하라. `/`로 설정했을 때 `.jsp` 요청은 어떻게 처리되는가?

**Q3.** 하나의 Spring Boot 애플리케이션에 두 개의 `DispatcherServlet`을 등록하는 것이 가능한가? 가능하다면 어떤 시나리오에서 활용할 수 있는가?

```java
// 힌트: 아래 설정이 의도대로 동작할까?
@Bean
public DispatcherServletRegistrationBean adminDispatcher(
        WebApplicationContext wac) {
    DispatcherServlet servlet = new DispatcherServlet(wac);
    DispatcherServletRegistrationBean bean =
        new DispatcherServletRegistrationBean(servlet, "/admin/*");
    bean.setName("adminDispatcher");
    return bean;
}
```

> 💡 **해설**
>
> **Q1.** Tomcat 같은 서블릿 컨테이너는 스레드 풀에서 스레드를 재사용합니다. `ThreadLocal`은 스레드에 종속되므로, `resetContextHolders()`를 호출하지 않으면 이전 요청의 `RequestAttributes`가 다음 요청에 그대로 남아있게 됩니다. 구체적인 증상으로는 요청 A의 사용자 인증 정보가 요청 B에서 잘못 노출되거나, 요청 A의 로케일 정보가 요청 B에도 적용되는 등 "요청 간 데이터 오염(thread-local leakage)"이 발생합니다. 또한 `WebRequest`를 완료하지 않으면 `HttpSession` 참조가 GC되지 않아 메모리 누수로 이어질 수 있습니다. `finally` 블록 사용은 예외 발생 시에도 ThreadLocal이 반드시 정리됨을 보장합니다.
>
> **Q2.** Tomcat에는 정적 리소스를 처리하는 `DefaultServlet`이 `url-pattern="/"` 로 기본 등록되어 있습니다. Spring MVC도 `"/"` 로 등록하면 더 구체적인 패턴이 없을 때 Spring의 `DispatcherServlet`이 DefaultServlet을 오버라이드합니다. 단, JSP는 Tomcat 내부의 `JspServlet`이 `*.jsp` 패턴으로 등록되어 있어 더 구체적인 패턴 규칙에 의해 JSP 요청은 여전히 `JspServlet`이 처리합니다. 반면 `/*` 로 등록하면 모든 경로(JSP 포함)가 DispatcherServlet으로 오고, JSP도 처리하지 못해 404가 됩니다.
>
> **Q3.** 가능합니다. `DispatcherServletRegistrationBean`에 서로 다른 이름과 URL 패턴을 지정하면 됩니다. 활용 시나리오: (1) `/api/*` 에는 REST API용 DispatcherServlet을, `/admin/*` 에는 관리자 인터페이스용 DispatcherServlet을 분리 등록해 각각 다른 `WebApplicationContext`와 Security 설정 적용. (2) 서로 다른 HandlerMapping 전략이 필요한 경우. 단, 두 DispatcherServlet이 같은 `WebApplicationContext`를 공유하면 컴포넌트 초기화(`initStrategies`)를 각자 수행하므로 Bean 중복 탐색이 발생할 수 있어, 별도의 자식 컨텍스트를 만들어 주는 것이 권장됩니다.

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: DispatcherServlet 초기화 과정 ➡️](./02-dispatcher-servlet-init.md)**

</div>
