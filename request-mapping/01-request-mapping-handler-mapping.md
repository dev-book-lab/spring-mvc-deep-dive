# @RequestMapping 처리 과정 — RequestMappingHandlerMapping이 메서드를 등록하는 방법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `RequestMappingHandlerMapping`은 언제, 어떤 순서로 `@RequestMapping` 메서드를 스캔하는가?
- `@GetMapping`, `@PostMapping` 등 축약 어노테이션은 내부적으로 어떻게 처리되는가?
- `MappingRegistry`에 등록되는 정보의 구조는 무엇인가?
- CGLIB 프록시 클래스에서 `@RequestMapping`을 어떻게 찾는가?
- 클래스 레벨 `@RequestMapping`과 메서드 레벨 `@RequestMapping`은 어떻게 합쳐지는가?
- 같은 URL에 두 개의 핸들러를 등록하면 언제 예외가 발생하는가?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: URL과 핸들러 메서드 사이의 연결을 누가 언제 만드는가

```java
// 이 코드가 HTTP 요청과 연결되려면?
@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }
}
```

```
요청 처리 시마다 스캔한다면:
  GET /users/1 요청 → 모든 Bean 스캔 → @RequestMapping 탐색 → 매칭
  → 요청마다 수백 개 Bean, 수천 개 메서드 스캔
  → 응답 시간 폭증 (비현실적)

해결: 애플리케이션 시작 시 한 번만 스캔 + HashMap에 저장
  ApplicationContext refresh 완료
  → RequestMappingHandlerMapping.afterPropertiesSet()
  → 모든 @Controller Bean 스캔 + RequestMappingInfo 생성
  → MappingRegistry에 등록 (URL → HandlerMethod 매핑)
  → 요청 시: HashMap 조회 → O(1) 탐색
```

---

## 😱 흔한 오해 또는 실수

### Before: @GetMapping이 @RequestMapping과 다른 메커니즘으로 처리된다

```java
// ❌ 잘못된 이해
// "@GetMapping은 Spring MVC 전용 어노테이션이라 별도로 처리된다"

// ✅ 실제: @GetMapping은 @RequestMapping의 메타 어노테이션
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@RequestMapping(method = RequestMethod.GET)  // ← @RequestMapping 포함
public @interface GetMapping {
    @AliasFor(annotation = RequestMapping.class)
    String[] value() default {};
    // ...
}

// RequestMappingHandlerMapping은 getMappingForMethod() 시
// AnnotatedElementUtils.findMergedAnnotation(method, RequestMapping.class)로 탐색
// → @GetMapping, @PostMapping 등도 모두 @RequestMapping으로 수렴
// → 처리 로직 완전히 동일
```

### Before: 클래스 레벨과 메서드 레벨 @RequestMapping은 독립적이다

```java
// ❌ 잘못된 이해
// "클래스의 '/users'와 메서드의 '/{id}'는 별개로 등록된다"

@RestController
@RequestMapping("/users")   // 클래스 레벨
public class UserController {

    @GetMapping("/{id}")    // 메서드 레벨
    public User getUser(@PathVariable Long id) { ... }
}

// ✅ 실제:
// getMappingForMethod()에서 두 개를 combine()으로 합침
// 클래스 RequestMappingInfo("/users") + 메서드 RequestMappingInfo("/{id}")
// → 합쳐진 RequestMappingInfo("/users/{id}", GET)
// → MappingRegistry에 등록되는 것은 합쳐진 하나의 Info
```

---

## ✨ 올바른 이해와 사용

### After: 등록 과정 전체 흐름

```
ApplicationContext.refresh() 완료
  ↓
RequestMappingHandlerMapping.afterPropertiesSet()  [InitializingBean]
  ↓
AbstractHandlerMethodMapping.initHandlerMethods()
  ↓
  모든 Bean 이름 순회:
    getCandidateBeanNames() → ApplicationContext의 Bean 이름 목록
    ↓
    processCandidateBean(beanName)
      → getType(beanName) → Class<?> 획득
      → isHandler(beanType)? → @Controller 또는 @RequestMapping 있으면 true
      ↓
    detectHandlerMethods(beanName)
      → ClassUtils.getUserClass() → CGLIB 프록시면 원본 클래스 획득
      → MethodIntrospector.selectMethods()
          → 모든 메서드에 대해 getMappingForMethod() 호출
          → @RequestMapping이 있으면 RequestMappingInfo 생성, 없으면 null
      ↓
      각 메서드에 대해 registerHandlerMethod(beanName, method, mapping)
        → MappingRegistry.register(mapping, handler, method)
```

---

## 🔬 내부 동작 원리

### 1. getMappingForMethod() — RequestMappingInfo 생성

```java
// RequestMappingHandlerMapping.java
@Override
@Nullable
protected RequestMappingInfo getMappingForMethod(Method method, Class<?> handlerType) {

    // ① 메서드 레벨 @RequestMapping 탐색
    RequestMappingInfo info = createRequestMappingInfo(method);
    if (info != null) {

        // ② 클래스 레벨 @RequestMapping 탐색
        RequestMappingInfo typeInfo = createRequestMappingInfo(handlerType);
        if (typeInfo != null) {
            // ③ 클래스 레벨 + 메서드 레벨 합치기
            info = typeInfo.combine(info);
        }

        // ④ prefix 설정 적용 (WebMvcConfigurer.configurePathMatch() 로 설정 가능)
        String prefix = getPathPrefix(handlerType);
        if (prefix != null) {
            info = RequestMappingInfo.paths(prefix)
                       .options(this.config).build().combine(info);
        }
    }
    return info;
}

private RequestMappingInfo createRequestMappingInfo(AnnotatedElement element) {
    // 메타 어노테이션 포함 탐색 (@GetMapping → @RequestMapping)
    RequestMapping requestMapping =
        AnnotatedElementUtils.findMergedAnnotation(element, RequestMapping.class);
    RequestCondition<?> condition = (element instanceof Class ?
        getCustomTypeCondition((Class<?>) element) :
        getCustomMethodCondition((Method) element));
    return (requestMapping != null
        ? createRequestMappingInfo(requestMapping, condition)
        : null);
}

protected RequestMappingInfo createRequestMappingInfo(
        RequestMapping requestMapping, @Nullable RequestCondition<?> customCondition) {
    // @RequestMapping의 모든 속성을 RequestMappingInfo로 변환
    return RequestMappingInfo
        .paths(resolveEmbeddedValuesInPatterns(requestMapping.value()))
        .methods(requestMapping.method())
        .params(requestMapping.params())
        .headers(requestMapping.headers())
        .consumes(requestMapping.consumes())
        .produces(requestMapping.produces())
        .mappingName(requestMapping.name())
        .customCondition(customCondition)
        .options(this.config)
        .build();
}
```

### 2. RequestMappingInfo.combine() — 클래스 + 메서드 합치기

```java
// RequestMappingInfo.java
@Override
public RequestMappingInfo combine(RequestMappingInfo other) {
    String name = combineNames(other);
    // 각 조건 결합:
    PathPatternsRequestCondition patterns =
        (this.pathPatternsCondition != null && other.pathPatternsCondition != null
            ? this.pathPatternsCondition.combine(other.pathPatternsCondition)
            : null);
    // "/users" + "/{id}" → "/users/{id}"
    // 패턴 조합 규칙:
    //   "/users" + "/{id}"  → "/users/{id}"
    //   "/users" + ""       → "/users"
    //   ""       + "/{id}"  → "/{id}"
    //   "/users/" + "/{id}" → "/users/{id}" (중복 / 제거)

    RequestMethodsRequestCondition methods =
        this.methodsCondition.combine(other.methodsCondition);
    // GET + (없음) → GET
    // (없음) + POST → POST
    // GET + POST → 두 메서드 모두 허용

    ParamsRequestCondition params =
        this.paramsCondition.combine(other.paramsCondition);
    HeadersRequestCondition headers =
        this.headersCondition.combine(other.headersCondition);
    ConsumesRequestCondition consumes =
        this.consumesCondition.combine(other.consumesCondition);
    ProducesRequestCondition produces =
        this.producesCondition.combine(other.producesCondition);

    return new RequestMappingInfo(name, patterns, patternCondition,
        methods, params, headers, consumes, produces, customCondition, options);
}
```

### 3. MappingRegistry.register() — 실제 등록

```java
// AbstractHandlerMethodMapping.MappingRegistry
public void register(T mapping, Object handler, Method method) {
    this.readWriteLock.writeLock().lock();  // 쓰기 락
    try {
        // HandlerMethod 생성
        // = Bean 인스턴스 + Method 객체 + Bean 이름의 조합
        HandlerMethod handlerMethod = createHandlerMethod(handler, method);

        // 중복 매핑 검사
        validateMethodMapping(handlerMethod, mapping);

        // 직접 경로 (변수 없는 URL) 추출
        Set<String> directPaths = AbstractHandlerMethodMapping.this.getDirectPaths(mapping);
        for (String path : directPaths) {
            // directPathMappings: "/users" → [RequestMappingInfo]
            this.pathLookup.add(path, mapping);
        }

        // 이름 기반 매핑 (선택적)
        String name = null;
        if (getNamingStrategy() != null) {
            name = getNamingStrategy().getName(handlerMethod, mapping);
            addMappingName(name, handlerMethod);
        }

        // CORS 설정 수집
        CorsConfiguration corsConfig = initCorsConfiguration(handler, method, mapping);
        if (corsConfig != null) {
            corsConfig.validateAllowCredentials();
            this.corsLookup.put(handlerMethod, corsConfig);
        }

        // 핵심 등록: RequestMappingInfo → MappingRegistration
        this.registry.put(mapping,
            new MappingRegistration<>(mapping, handlerMethod, directPaths, name, corsConfig));

    } finally {
        this.readWriteLock.writeLock().unlock();
    }
}

// 중복 매핑 검사
private void validateMethodMapping(HandlerMethod handlerMethod, T mapping) {
    MappingRegistration<T> registration = this.registry.get(mapping);
    HandlerMethod existingHandlerMethod =
        (registration != null ? registration.getHandlerMethod() : null);

    if (existingHandlerMethod != null && !existingHandlerMethod.equals(handlerMethod)) {
        // 동일 RequestMappingInfo에 다른 HandlerMethod가 이미 있으면 예외!
        throw new IllegalStateException(
            "Ambiguous mapping. Cannot map '" + handlerMethod.getBean() + "' method \n" +
            handlerMethod + "\nto " + mapping + ": There is already '" +
            existingHandlerMethod.getBean() + "' bean method\n" +
            existingHandlerMethod + " mapped.");
    }
}
```

### 4. CGLIB 프록시 처리 — getUserClass()

```java
// ClassUtils.getUserClass() — CGLIB 프록시 원본 클래스 추출
public static Class<?> getUserClass(Class<?> clazz) {
    // CGLIB 프록시 클래스 이름에는 "$$" 포함
    // 예: UserController$$SpringCGLIB$$0
    if (clazz.getName().contains(CGLIB_CLASS_SEPARATOR)) {
        Class<?> superclass = clazz.getSuperclass();
        if (superclass != null && superclass != Object.class) {
            return superclass;  // 원본 클래스 반환
        }
    }
    return clazz;
}

// 왜 필요한가:
// @Configuration(proxyBeanMethods=true)인 경우 CGLIB 프록시 생성
// CGLIB 클래스에는 원본 메서드의 어노테이션이 없을 수 있음
// getUserClass()로 원본 클래스를 얻어야 @RequestMapping 어노테이션 탐색 가능
```

### 5. MethodIntrospector.selectMethods() — 메서드 필터링

```java
// MethodIntrospector.selectMethods()
public static <T> Map<Method, T> selectMethods(Class<?> targetType,
        MetadataLookup<T> metadataLookup) {

    Map<Method, T> methodMap = new LinkedHashMap<>();
    Set<Class<?>> handlerTypes = new LinkedHashSet<>();
    Class<?> specificHandlerType = null;

    // CGLIB 프록시가 아닌 실제 타입 우선
    if (!Proxy.isProxyClass(targetType)) {
        specificHandlerType = ClassUtils.getUserClass(targetType);
        handlerTypes.add(specificHandlerType);
    }
    // 인터페이스도 포함 (인터페이스에 @RequestMapping 선언 가능)
    handlerTypes.addAll(ClassUtils.getAllInterfacesForClassAsSet(targetType));

    for (Class<?> currentHandlerType : handlerTypes) {
        final Class<?> targetClass = specificHandlerType != null
            ? specificHandlerType : currentHandlerType;

        // 모든 메서드를 순회 (상속된 메서드 포함)
        ReflectionUtils.doWithMethods(currentHandlerType, method -> {
            Method specificMethod = ClassUtils.getMostSpecificMethod(method, targetClass);
            // metadataLookup = getMappingForMethod
            T result = metadataLookup.inspect(specificMethod);
            if (result != null) {
                // 브릿지 메서드 처리 (제네릭 타입 소거로 생성되는 synthetic 메서드)
                Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(specificMethod);
                if (bridgedMethod == specificMethod
                        || metadataLookup.inspect(bridgedMethod) == null) {
                    methodMap.put(specificMethod, result);
                }
            }
        }, ReflectionUtils.USER_DECLARED_METHODS);
    }
    return methodMap;
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 등록 시점 로그로 스캔 과정 확인

```yaml
logging:
  level:
    org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping: TRACE
```

```
출력 예시:
  TRACE o.s.w.s.m.m.a.RequestMappingHandlerMapping -
    "Mapped to UserController#getUser(Long)"  ← detectHandlerMethods 완료

  TRACE o.s.w.s.m.m.a.RequestMappingHandlerMapping -
    Mapped 5 handler methods for UserController

  INFO  o.s.w.s.m.m.a.RequestMappingHandlerMapping -
    Mapped "{GET [/users/{id}]}" onto UserController#getUser(Long)
    Mapped "{POST [/users], consumes [application/json]}" onto UserController#createUser(UserDto)
    Mapped "{GET [/users]}" onto UserController#listUsers(int, int)
```

### 실험 2: 런타임에 등록된 매핑 목록 전체 확인

```java
@Autowired
RequestMappingHandlerMapping mapping;

@GetMapping("/admin/mappings")
public Map<String, String> allMappings() {
    return mapping.getHandlerMethods().entrySet().stream()
        .collect(Collectors.toMap(
            e -> e.getKey().toString(),
            e -> e.getValue().toString(),
            (a, b) -> a,
            LinkedHashMap::new
        ));
}
```

```json
// 출력:
{
  "{GET [/users/{id}]}": "UserController#getUser(Long)",
  "{POST [/users], consumes [application/json]}": "UserController#createUser(UserDto)",
  "{DELETE [/users/{id}]}": "UserController#deleteUser(Long)"
}
```

### 실험 3: 중복 매핑 의도적으로 만들어 예외 확인

```java
@RestController
public class DuplicateController {

    @GetMapping("/dup")
    public String method1() { return "1"; }

    @GetMapping("/dup")  // 동일 URL + 동일 메서드 → 예외!
    public String method2() { return "2"; }
}

// 애플리케이션 시작 시:
// java.lang.IllegalStateException: Ambiguous mapping.
//   Cannot map 'duplicateController' method
//   public String DuplicateController.method2()
//   to {GET [/dup]}: There is already 'duplicateController' bean method
//   public String DuplicateController.method1() mapped.
```

### 실험 4: 인터페이스에 @RequestMapping 선언

```java
// 인터페이스에 선언도 가능
public interface UserApi {
    @GetMapping("/users/{id}")
    User getUser(@PathVariable Long id);
}

@RestController
public class UserController implements UserApi {
    @Override
    public User getUser(@PathVariable Long id) {  // 어노테이션 없어도 OK
        return userService.findById(id);
    }
}
// selectMethods()가 인터페이스 포함 탐색하므로
// UserApi의 @GetMapping("/users/{id}")가 UserController.getUser()에 적용됨
```

---

## 🌐 HTTP 레벨 분석

```
매핑 등록 후 요청 처리 흐름:

GET /users/42 HTTP/1.1
Host: localhost:8080

→ DispatcherServlet.getHandler()
→ RequestMappingHandlerMapping.getHandler()
  → lookupHandlerMethod("/users/42", request)
    → directPathMappings 탐색: "/users/42" → null (변수 있어 직접 경로 아님)
    → 전체 registry 탐색:
        "{GET [/users/{id}]}" → PathPattern.matches("/users/42") → true ✅
        "{POST [/users]}"    → PathPattern.matches("/users/42") → false
        "{GET [/users]}"     → PathPattern.matches("/users/42") → false
    → 매칭된 후보: [{GET [/users/{id}]}] → HandlerMethod: UserController#getUser
    → PathPattern.matchAndExtract() → {id: "42"} → request 속성에 저장
  → HandlerExecutionChain(HandlerMethod, [인터셉터들]) 반환

→ RequestMappingHandlerAdapter.handle() → getUser(42L) 실행
→ HTTP/1.1 200 OK
   Content-Type: application/json
   {"id":42,"name":"홍길동"}
```

---

## 🤔 트레이드오프

```
시작 시 일괄 스캔 방식:
  장점  요청 처리 시 탐색 비용 없음 (HashMap O(1))
        중복 매핑을 시작 시점에 즉시 발견 → 빠른 피드백
  단점  시작 시간 증가 (Bean 수에 비례)
        런타임 매핑 변경 어려움 (읽기/쓰기 락으로 가능하지만 비권장)

어노테이션 기반 선언 방식:
  장점  코드와 URL이 함께 있어 가독성 높음
        IDE 지원 (navigation, rename refactoring)
        조건 조합 유연 (URL + 메서드 + Content-Type + 헤더 + 파라미터)
  단점  URL 목록을 한 곳에서 볼 수 없음
        → actuator /mappings 엔드포인트나 직접 조회 API로 보완

RouterFunction 방식 (대안):
  장점  URL이 코드에 집중되어 한 눈에 파악
        람다로 동적 조건 설정 가능
  단점  어노테이션 기반보다 생태계 도구 지원 적음
```

---

## 📌 핵심 정리

```
등록 시점: afterPropertiesSet() → initHandlerMethods()
  ApplicationContext refresh 완료 이후 한 번만 실행
  모든 Bean 중 isHandler() (= @Controller/@RequestMapping) 인 것만 스캔

getMappingForMethod() 처리 순서
  ① 메서드 레벨 @RequestMapping → RequestMappingInfo
  ② 클래스 레벨 @RequestMapping → RequestMappingInfo
  ③ combine(): 클래스 Info + 메서드 Info → 최종 합친 Info
  ④ 경로 prefix 적용 (설정된 경우)

@GetMapping = @RequestMapping(method=GET)
  AnnotatedElementUtils.findMergedAnnotation()으로 메타 어노테이션 포함 탐색
  → 모든 축약 어노테이션이 @RequestMapping으로 수렴

MappingRegistry 구조
  registry: Map<RequestMappingInfo, MappingRegistration>
  pathLookup: Map<String, List<RequestMappingInfo>> (직접 경로 빠른 탐색)
  corsLookup: Map<HandlerMethod, CorsConfiguration>
  ReadWriteLock으로 동시성 보호

중복 매핑: 시작 시 IllegalStateException
  동일한 RequestMappingInfo(URL+조건)에 다른 HandlerMethod 등록 시 즉시 예외
```

---

## 🤔 생각해볼 문제

**Q1.** `@RequestMapping` 어노테이션이 클래스 레벨에만 있고 메서드 레벨에는 없는 경우, `getMappingForMethod()`는 어떻게 동작하는가? 해당 클래스의 모든 메서드가 매핑되는가?

**Q2.** `RequestMappingHandlerMapping`을 상속해서 커스텀 `HandlerMapping`을 만들 때, `isHandler()` 메서드를 오버라이드하면 어떤 효과가 있는가? 어떤 실용적인 사용 시나리오가 있는가?

**Q3.** `@RequestMapping(value = "/users", method = {GET, POST})`처럼 여러 HTTP 메서드를 한 메서드에 등록하면 `MappingRegistry`에 몇 개의 항목이 생성되는가? `RequestMappingInfo`의 동등성(equality)은 어떻게 정의되는가?

> 💡 **해설**
>
> **Q1.** 메서드 레벨에 `@RequestMapping`이 없으면 `getMappingForMethod()`는 `null`을 반환합니다. 따라서 해당 메서드는 등록되지 않습니다. 클래스 레벨 `@RequestMapping`만으로는 어떤 메서드도 핸들러로 등록되지 않습니다. 클래스 레벨은 메서드 레벨 `@RequestMapping`에 대한 공통 prefix/조건을 제공하는 역할입니다. 모든 메서드를 매핑하려면 각 메서드에 `@GetMapping`, `@PostMapping` 등 메서드 레벨 어노테이션이 필요합니다.
>
> **Q2.** `isHandler()`를 오버라이드하면 `@Controller`/`@RequestMapping` 외의 어노테이션으로 핸들러를 지정할 수 있습니다. 실용적 시나리오: (1) 사내 커스텀 어노테이션 `@ApiEndpoint`를 만들어 `@Controller` 대신 사용, (2) `@Service` Bean 중 특정 인터페이스 구현체만 핸들러로 등록 — gRPC 게이트웨이나 WebSocket 핸들러 통합 시 유용합니다. 단, `getMappingForMethod()`도 함께 오버라이드해 해당 어노테이션에서 `RequestMappingInfo`를 생성하도록 해야 합니다.
>
> **Q3.** `MappingRegistry`에는 **하나의 항목**만 생성됩니다. `RequestMappingInfo`는 `RequestMethodsRequestCondition`에 `{GET, POST}` 두 메서드를 하나의 조건으로 캡슐화합니다. 이 하나의 Info가 하나의 `HandlerMethod`에 매핑됩니다. `RequestMappingInfo`의 동등성은 모든 조건(패턴, 메서드, 파라미터, 헤더, consumes, produces)의 동등성 조합으로 결정됩니다. `value={"/users"}, method={GET}` 와 `value={"/users"}, method={POST}`는 서로 다른 `RequestMappingInfo`이므로 같은 URL이지만 서로 다른 메서드 매핑은 중복이 아닙니다.

---

<div align="center">

**[⬅️ 이전: Chapter 1 — ViewResolver 메커니즘](../dispatcher-servlet/07-view-resolver-mechanism.md)** | **[홈으로 🏠](../README.md)** | **[다음: RequestMappingInfo 생성과 매칭 ➡️](./02-request-mapping-info.md)**

</div>
