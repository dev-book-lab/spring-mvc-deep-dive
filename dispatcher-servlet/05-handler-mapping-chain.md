# HandlerMapping 체인 동작 — 요청이 핸들러를 찾는 과정

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `getHandler(request)` 내부에서 여러 `HandlerMapping`이 탐색되는 순서는?
- `RequestMappingHandlerMapping`과 `BeanNameUrlHandlerMapping`의 차이와 우선순위는?
- `HandlerExecutionChain`이란 무엇이고 왜 Handler만 반환하지 않는가?
- `RequestMappingHandlerMapping`은 애플리케이션 시작 시 어떻게 `@RequestMapping` 정보를 수집하는가?
- `MappingRegistry`의 내부 구조는 어떻게 되어 있는가?
- 같은 URL 패턴을 가진 핸들러가 두 개 있으면 어떤 일이 발생하는가?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: 핸들러의 종류가 다양하고 매핑 전략도 다양하다

```
Spring MVC가 지원하는 핸들러 종류:
  ① @Controller + @RequestMapping 메서드  (가장 일반적)
  ② HttpRequestHandler 구현체             (정적 리소스 등)
  ③ Controller 인터페이스 구현체           (레거시)
  ④ RouterFunction                        (함수형 엔드포인트)
  ⑤ BeanName으로 URL 매핑된 Bean

매핑 전략도 다양:
  ① URL + HTTP 메서드 + Content-Type 조합
  ② Bean 이름이 곧 URL
  ③ 함수형 라우트 조건
  ④ 정적 리소스 경로 패턴

해결:
  HandlerMapping 인터페이스 → 구현체가 각자 전략 담당
  → DispatcherServlet은 순서대로 탐색, 첫 번째 match 사용
  → 새 매핑 전략 추가 시 DispatcherServlet 수정 불필요
```

---

## 😱 흔한 오해 또는 실수

### Before: HandlerMapping은 하나만 있다

```java
// ❌ 잘못된 이해
// "Spring MVC에는 하나의 HandlerMapping이 있고 그것이 @RequestMapping을 처리한다"

// ✅ 실제: Spring Boot 기본 설정 시 HandlerMapping 목록
// (initHandlerMappings에서 order 순으로 정렬됨)
//
//  order=-1    RouterFunctionMapping     → @RouterOperation, RouterFunction Bean
//  order=0     RequestMappingHandlerMapping → @RequestMapping, @GetMapping 등
//  order=1     ControllerEndpointHandlerMapping → Actuator @ControllerEndpoint (Actuator 있으면)
//  order=2     BeanNameUrlHandlerMapping → Bean 이름이 "/" 시작인 경우
//  order=2147483646  WelcomePageHandlerMapping → index.html 서빙
//  order=2147483647  WelcomePageNotFoundHandlerMapping → 404 fallback
//
// getHandler()는 이 순서대로 탐색하여 첫 번째 non-null 결과를 반환
```

### Before: @RequestMapping 매핑 탐색이 요청마다 전체 스캔된다

```
❌ 잘못된 이해:
  "매 요청마다 모든 @RequestMapping을 순차 탐색한다"

✅ 실제:
  애플리케이션 시작 시 MappingRegistry에 HashMap으로 등록
  → 요청이 오면 URL 기반 해시 탐색 (O(1) 수준)
  → URL + 조건(HTTP 메서드, Content-Type 등)으로 후보 좁히기
  → 여러 후보 중 가장 구체적인 것 선택 (정렬 + 비교)
```

---

## ✨ 올바른 이해와 사용

### After: getHandler()의 탐색 흐름

```java
// DispatcherServlet.java
@Nullable
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    if (this.handlerMappings != null) {
        for (HandlerMapping mapping : this.handlerMappings) {
            // 각 HandlerMapping에 탐색 위임
            HandlerExecutionChain handler = mapping.getHandler(request);
            if (handler != null) {
                // 첫 번째 non-null 반환 → 즉시 사용
                return handler;
            }
        }
    }
    return null;
}
```

```
HandlerExecutionChain = Handler + Interceptor 목록

  Handler: 실제 처리를 담당할 객체
    → @RequestMapping의 경우: HandlerMethod 인스턴스
       (Controller 인스턴스 + Method 객체 + Bean 이름)
    → HttpRequestHandler의 경우: Handler Bean 자체

  Interceptor 목록: 이 핸들러에 적용될 인터셉터
    → URL 패턴 기반으로 필터링된 인터셉터만 포함
    → CORS 처리용 CorsInterceptor가 있으면 추가됨
```

---

## 🔬 내부 동작 원리

### 1. RequestMappingHandlerMapping — 시작 시 매핑 등록

```java
// AbstractHandlerMethodMapping.java (RequestMappingHandlerMapping의 부모)
@Override
public void afterPropertiesSet() {
    initHandlerMethods();  // ApplicationContext 초기화 완료 후 호출
}

protected void initHandlerMethods() {
    // ApplicationContext의 모든 Bean 이름 조회
    for (String beanName : getCandidateBeanNames()) {
        if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
            // Bean 타입 확인 후 핸들러 메서드 탐지
            processCandidateBean(beanName);
        }
    }
}

protected void processCandidateBean(String beanName) {
    Class<?> beanType = obtainApplicationContext().getType(beanName);
    // @Controller 또는 @RequestMapping이 붙은 클래스인지 확인
    if (beanType != null && isHandler(beanType)) {
        detectHandlerMethods(beanName);
    }
}

// RequestMappingHandlerMapping.isHandler() 구현
@Override
protected boolean isHandler(Class<?> beanType) {
    return (AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) ||
            AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class));
}
```

### 2. detectHandlerMethods() — 메서드 스캔 및 MappingRegistry 등록

```java
// AbstractHandlerMethodMapping.detectHandlerMethods()
protected void detectHandlerMethods(Object handler) {
    Class<?> handlerType = (handler instanceof String beanName ?
        obtainApplicationContext().getType(beanName) : handler.getClass());

    if (handlerType != null) {
        Class<?> userType = ClassUtils.getUserClass(handlerType); // CGLIB 프록시 언래핑

        // 메서드마다 @RequestMapping 정보 수집
        Map<Method, T> methods = MethodIntrospector.selectMethods(userType,
            (MethodIntrospector.MetadataLookup<T>) method -> {
                try {
                    return getMappingForMethod(method, userType);
                    // → RequestMappingInfo 생성
                    //   (URL, HTTP 메서드, produces, consumes, params, headers 조건 캡슐화)
                } catch (Throwable ex) {
                    throw new IllegalStateException("...", ex);
                }
            });

        // MappingRegistry에 등록
        methods.forEach((method, mapping) -> {
            Method invocableMethod = AopUtils.selectInvocableMethod(method, userType);
            registerHandlerMethod(handler, invocableMethod, mapping);
        });
    }
}
```

### 3. MappingRegistry — 내부 자료구조

```java
// AbstractHandlerMethodMapping.MappingRegistry
class MappingRegistry {

    // 모든 매핑 정보 저장 (RequestMappingInfo → MappingRegistration)
    private final Map<T, MappingRegistration<T>> registry = new LinkedHashMap<>();

    // URL → 매핑 목록 (빠른 탐색용)
    // PathPatternParser 사용 시: MultiValueMap<String, T>
    private final Map<String, List<T>> pathLookup = new LinkedHashMap<>();

    // 직접 URL (변수 없는 경로) → 매핑 목록 (더 빠른 탐색용)
    // "/users" 같은 정적 경로
    private final MultiValueMap<String, T> directPathMappings = new LinkedMultiValueMap<>();

    // CORS 설정 매핑
    private final Map<HandlerMethod, CorsConfiguration> corsLookup = new ConcurrentHashMap<>();

    // 읽기/쓰기 락 (동시성 보호 — 런타임에 새 매핑 추가 가능)
    private final ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();
}
```

### 4. 요청 시 매핑 탐색 과정

```java
// AbstractHandlerMethodMapping.lookupHandlerMethod()
@Nullable
protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request)
        throws Exception {

    List<Match> matches = new ArrayList<>();

    // ① directPathMappings에서 정확히 일치하는 URL 먼저 탐색 (빠름)
    List<T> directPathMatches = this.mappingRegistry.getMappingsByDirectPath(lookupPath);
    if (directPathMatches != null) {
        addMatchingMappings(directPathMatches, matches, request);
    }

    // ② 정확히 일치하는 것이 없으면 전체 매핑 탐색 (패턴 매칭)
    if (matches.isEmpty()) {
        addMatchingMappings(this.mappingRegistry.getRegistrations().keySet(), matches, request);
    }

    if (!matches.isEmpty()) {
        // ③ 여러 후보 중 가장 구체적인 것 선택
        Match bestMatch = matches.get(0);
        if (matches.size() > 1) {
            // compareTo()로 정렬 — 더 구체적인 패턴이 우선
            // "/users/1" vs "/users/{id}" → "/users/1"이 더 구체적
            Comparator<Match> comparator = new MatchComparator(getMappingComparator(request));
            matches.sort(comparator);
            bestMatch = matches.get(0);

            // 상위 두 개가 동등하게 구체적이면 예외 (모호한 매핑)
            Match secondBestMatch = matches.get(1);
            if (comparator.compare(bestMatch, secondBestMatch) == 0) {
                // → IllegalStateException: Ambiguous handler methods mapped
            }
        }
        request.setAttribute(BEST_MATCHING_HANDLER_ATTRIBUTE, bestMatch.getHandlerMethod());
        handleMatch(bestMatch.mapping, lookupPath, request);
        return bestMatch.getHandlerMethod();
    }

    // 매칭 실패 → handleNoMatch()
    return handleNoMatch(this.mappingRegistry.getRegistrations().keySet(), lookupPath, request);
}
```

### 5. BeanNameUrlHandlerMapping — 레거시 방식

```java
// BeanNameUrlHandlerMapping.java
// Bean 이름이 "/" 로 시작하면 URL로 처리
@Override
protected void detectHandlers() throws BeansException {
    ApplicationContext applicationContext = obtainApplicationContext();
    String[] beanNames = applicationContext.getBeanNamesForType(Object.class);

    for (String beanName : beanNames) {
        // "/" 시작 Bean 이름 = URL 패턴
        String[] urls = determineUrlsForHandler(beanName);
        if (!ObjectUtils.isEmpty(urls)) {
            registerHandler(urls, beanName);
        }

        // "/" 시작 alias도 URL로 처리
        String[] aliases = applicationContext.getAliases(beanName);
        for (String alias : aliases) {
            if (alias.startsWith("/")) {
                registerHandler(alias, beanName);
            }
        }
    }
}

// 예: @Component("/users") UserController
// → "/users" URL에 UserController Bean 매핑
// → 현대 Spring MVC에서는 거의 사용되지 않음
```

---

## 💻 실험으로 확인하기

### 실험 1: 등록된 매핑 전체 목록 출력

```java
@Autowired
RequestMappingHandlerMapping requestMappingHandlerMapping;

@GetMapping("/mappings")
public Map<String, Object> listMappings() {
    Map<RequestMappingInfo, HandlerMethod> handlerMethods =
        requestMappingHandlerMapping.getHandlerMethods();

    Map<String, Object> result = new LinkedHashMap<>();
    handlerMethods.forEach((info, method) ->
        result.put(info.toString(),
            method.getBeanType().getSimpleName() + "#" + method.getMethod().getName()));
    return result;
}
```

```json
// 출력 예시:
{
  "{GET [/users/{id}]}": "UserController#getUser",
  "{POST [/users], consumes [application/json]}": "UserController#createUser",
  "{GET [/users], params [page, size]}": "UserController#listUsers",
  "{PUT [/users/{id}]}": "UserController#updateUser",
  "{DELETE [/users/{id}]}": "UserController#deleteUser"
}
```

### 실험 2: 모호한 매핑 의도적으로 만들기

```java
@GetMapping("/test")
public String test1() { return "test1"; }

// 동일 URL, 동일 HTTP 메서드 → 시작 시 예외
@GetMapping("/test")
public String test2() { return "test2"; }

// Caused by: java.lang.IllegalStateException:
//   Ambiguous mapping. Cannot map 'userController' method
//   public String UserController.test2()
//   to {GET [/test]}: There is already 'userController' bean method
//   public String UserController.test1() mapped.
```

### 실험 3: 구체적인 패턴 vs 와일드카드 우선순위 확인

```java
@GetMapping("/users/admin")  // 정확한 URL
public String adminUser() { return "admin"; }

@GetMapping("/users/{id}")   // 패턴 URL
public String getUser(@PathVariable String id) { return "user: " + id; }

// GET /users/admin → "admin" 반환 (더 구체적인 패턴 선택)
// GET /users/123   → "user: 123" 반환 (패턴 매칭)
```

```bash
curl http://localhost:8080/users/admin  # → "admin"
curl http://localhost:8080/users/123   # → "user: 123"
```

---

## 🌐 HTTP 레벨 분석

```bash
# 매핑 없는 URL 요청 시
curl -v http://localhost:8080/nonexistent
# HTTP/1.1 404 Not Found
# → getHandler()가 null 반환
# → noHandlerFound() 호출
# → spring.mvc.throw-exception-if-no-handler-found=false (기본):
#     response.sendError(404)
# → spring.mvc.throw-exception-if-no-handler-found=true:
#     throw NoHandlerFoundException → @ExceptionHandler로 처리 가능

# 핸들러는 있지만 HTTP 메서드 불일치
curl -v -X DELETE http://localhost:8080/users  # GET만 있는 경우
# HTTP/1.1 405 Method Not Allowed
# Allow: GET, HEAD  ← 지원하는 메서드 목록 자동 추가
# → handleNoMatch()에서 MethodNotAllowedException throw
# → DefaultHandlerExceptionResolver가 405 응답 생성

# Content-Type 불일치 (consumes 조건)
curl -v -X POST http://localhost:8080/users \
     -H "Content-Type: text/plain" -d "data"
# HTTP/1.1 415 Unsupported Media Type
# → consumes 조건 불일치 → HttpMediaTypeNotSupportedException
```

---

## 🤔 트레이드오프

```
RequestMappingHandlerMapping (어노테이션 기반):
  장점  @GetMapping, @PostMapping 등으로 직관적 선언
        조건 조합이 풍부 (URL + 메서드 + Content-Type + 헤더 + 파라미터)
        URL 변수 추출 (@PathVariable) 지원
  단점  컴파일 타임에 URL 정보가 분산
        리플렉션 기반 초기화 (시작 시간에 영향)

RouterFunctionMapping (함수형):
  장점  코드에서 라우트 조합 가능 (동적 조건 설정 쉬움)
        자바 코드로 라우트를 조합·테스트하기 편함
  단점  선언적이지 않아 한 눈에 파악 어려움
        @RequestMapping보다 생태계 도구 지원 적음

BeanNameUrlHandlerMapping:
  장점  설정이 단순 (Bean 이름 = URL)
  단점  현대 Spring에서 거의 쓰이지 않음
        URL이 Bean 이름에 종속
```

---

## 📌 핵심 정리

```
HandlerMapping 탐색 순서 (Spring Boot 기본)
  order=-1    RouterFunctionMapping
  order=0     RequestMappingHandlerMapping  ← @RequestMapping 계열
  order=2     BeanNameUrlHandlerMapping
  order=MAX   WelcomePageHandlerMapping, WelcomePageNotFoundHandlerMapping

HandlerExecutionChain 구성
  Handler: HandlerMethod (Controller + Method) 또는 HttpRequestHandler
  Interceptors: URL 패턴 매칭된 HandlerInterceptor 목록

MappingRegistry 등록 시점
  RequestMappingHandlerMapping.afterPropertiesSet() → initHandlerMethods()
  → 모든 @Controller Bean 스캔 → detectHandlerMethods()
  → RequestMappingInfo(URL+조건) → HandlerMethod 매핑 → registry에 등록

요청 시 탐색 최적화
  ① 정확한 URL: directPathMappings에서 O(1) 조회
  ② 패턴 URL: 전체 registry 순회 후 가장 구체적인 것 선택
  ③ 모호한 경우 (동등 구체도): IllegalStateException

HandlerMapping 반환 null 시
  → DispatcherServlet.noHandlerFound()
  → 기본: response.sendError(404)
  → throw-exception-if-no-handler-found=true: NoHandlerFoundException
```

---

## 🤔 생각해볼 문제

**Q1.** 두 개의 URL 패턴 `/users/{userId}` 와 `/users/{accountId}` 가 같은 Controller에 등록되어 있다면 어떻게 되는가? 이 경우 `@PathVariable` 이름이 다른데 충돌이 발생하는가?

**Q2.** `RequestMappingHandlerMapping`은 `@Controller` Bean의 `@RequestMapping` 정보를 시작 시 수집합니다. 런타임에 새로운 `@Controller` Bean을 동적으로 등록하고 즉시 HTTP 요청을 처리하게 할 수 있는가? 어떻게 해야 하는가?

**Q3.** `GET /users`에는 핸들러가 있고, `GET /users/1`에도 핸들러가 있습니다. `OPTIONS /users`로 요청하면 어떤 응답이 오는가? `DispatcherServlet`이 `OPTIONS` 메서드를 어떻게 처리하는가?

> 💡 **해설**
>
> **Q1.** 이것은 Spring MVC 관점에서 **모호한 매핑(Ambiguous mapping)** 입니다. URL 패턴 `/users/{userId}`와 `/users/{accountId}`는 구조적으로 동일한 패턴(`/users/{변수명}`)이므로 동일한 `RequestMappingInfo` 구체도를 가집니다. 시작 시점에 `MappingRegistry.register()`가 기존 매핑을 발견하면 `IllegalStateException: Ambiguous handler methods mapped to '/users/{userId}'`를 throw합니다. 경로 변수 이름은 매핑의 구체도 비교에 영향을 주지 않습니다.
>
> **Q2.** 가능합니다. `RequestMappingHandlerMapping.detectHandlerMethods(beanName)`을 직접 호출하면 됩니다. 단, `MappingRegistry`는 `ReentrantReadWriteLock`으로 보호되므로 thread-safe합니다. 단, `ApplicationContext.registerBeanDefinition()` + `getBean()`으로 Bean을 먼저 등록한 뒤 `detectHandlerMethods()`를 호출해야 합니다. Spring Actuator의 `@Endpoint` 동적 노출이나 플러그인 아키텍처에서 이 패턴이 사용됩니다.
>
> **Q3.** Spring MVC는 `OPTIONS` 요청에 대해 자동으로 `Allow` 헤더를 생성합니다. `FrameworkServlet.doOptions()` 내부에서 `dispatchOptionsRequest=true`이면 `doDispatch()`를 통해 `handleNoMatch()` → `HttpOptions` 처리로 가서 해당 URL에 등록된 모든 HTTP 메서드를 수집해 `Allow: GET, HEAD, OPTIONS` 형태로 응답합니다. `dispatchOptionsRequest=false`이면 `HttpServlet.doOptions()`가 처리합니다. Spring Boot는 기본값으로 `dispatchOptionsRequest=true`를 설정합니다.

---

<div align="center">

**[⬅️ 이전: doDispatch() 완전 분해](./04-do-dispatch-analysis.md)** | **[홈으로 🏠](../README.md)** | **[다음: HandlerAdapter의 역할 ➡️](./06-handler-adapter-role.md)**

</div>
