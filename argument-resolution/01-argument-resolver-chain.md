# HandlerMethodArgumentResolver 체인 — 파라미터마다 적합한 Resolver를 찾는 과정

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `InvocableHandlerMethod.getMethodArgumentValues()`는 어떤 순서로 Resolver를 선택하는가?
- `HandlerMethodArgumentResolverComposite`의 캐시는 어떻게 동작하며 왜 필요한가?
- Resolver 등록 순서가 실제로 결과를 바꾸는 시나리오는 무엇인가?
- `supportsParameter()`가 true를 반환하는 기준은 Resolver마다 어떻게 다른가?
- 기본 Resolver 목록에서 폴백 Resolver가 맨 뒤에 있는 이유는?
- 커스텀 Resolver를 `addArgumentResolvers()`로 등록하면 기본 Resolver 앞에 오는가, 뒤에 오는가?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: 파라미터 타입과 어노테이션 조합이 수십 가지다

```
Controller 메서드 파라미터로 올 수 있는 것들:
  @RequestParam String name
  @PathVariable Long id
  @RequestBody UserDto dto
  @ModelAttribute SearchForm form
  @RequestHeader String authorization
  @CookieValue String token
  HttpServletRequest request
  HttpSession session
  Principal principal
  Model model
  BindingResult result
  Locale locale
  @Valid OrderDto order
  Optional<String> param
  ...30가지 이상

단일 if-else로 처리하면:
  if (param.hasAnnotation(RequestParam.class)) { ... }
  else if (param.hasAnnotation(PathVariable.class)) { ... }
  else if (param.hasAnnotation(RequestBody.class)) { ... }
  // ... 30개 이상의 분기
  // → InvocableHandlerMethod가 모든 Resolver 로직을 알아야 함
  // → 새 파라미터 타입 추가 시 핵심 클래스 수정 필요

해결: Chain of Responsibility 패턴
  각 Resolver가 "내가 처리할 수 있는가?"(supportsParameter)를 판단
  → 처리 가능한 첫 번째 Resolver가 값을 resolve
  → 새 파라미터 타입 = 새 Resolver 추가만 하면 됨
```

---

## 😱 흔한 오해 또는 실수

### Before: 커스텀 Resolver를 addArgumentResolvers()로 추가하면 기본 Resolver보다 먼저 실행된다

```java
// ❌ 잘못된 이해
@Override
public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
    resolvers.add(new MyCustomResolver());
    // "이제 MyCustomResolver가 기본 Resolver보다 먼저 선택된다"
}

// ✅ 실제:
// addArgumentResolvers()로 추가된 커스텀 Resolver는
// 기본 Resolver 목록 뒤에 추가됨
// (getDefaultArgumentResolvers() → customResolvers 순서)
//
// @RequestBody를 처리하는 커스텀 Resolver를
// 기본 RequestResponseBodyMethodProcessor보다 먼저 실행하려면:
// → RequestMappingHandlerAdapter.setArgumentResolvers()로 전체 목록 직접 지정
// → 또는 RequestBodyAdvice 활용 (더 안전한 방법)
```

### Before: supportsParameter()는 매 요청마다 30개 Resolver를 순회한다

```
❌ 잘못된 이해:
  "매 요청마다 모든 Resolver에 supportsParameter() 호출"

✅ 실제:
  HandlerMethodArgumentResolverComposite가 캐시 유지
  private final Map<MethodParameter, HandlerMethodArgumentResolver> argumentResolverCache

  첫 번째 호출 시에만 순회 탐색 → 캐시에 저장
  이후 동일 MethodParameter → 캐시에서 O(1) 반환

  캐시 키: MethodParameter (클래스 + 메서드 + 파라미터 인덱스)
  → 동일 Controller의 동일 메서드 파라미터는 항상 같은 Resolver
```

---

## ✨ 올바른 이해와 사용

### After: getMethodArgumentValues() 전체 흐름

```java
// InvocableHandlerMethod.getMethodArgumentValues()
protected Object[] getMethodArgumentValues(NativeWebRequest request,
        ModelAndViewContainer mavContainer,
        Object... providedArgs) throws Exception {

    MethodParameter[] parameters = getMethodParameters();
    if (ObjectUtils.isEmpty(parameters)) return EMPTY_ARGS;

    Object[] args = new Object[parameters.length];
    for (int i = 0; i < parameters.length; i++) {
        MethodParameter parameter = parameters[i];
        parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);

        // ① 미리 제공된 args에서 타입 매칭 시도 (비동기 처리 재진입 등)
        args[i] = findProvidedArgument(parameter, providedArgs);
        if (args[i] != null) continue;

        // ② Resolver 체인에서 처리 가능한 Resolver 탐색
        if (!this.resolvers.supportsParameter(parameter)) {
            throw new IllegalStateException(
                "No suitable resolver for argument " + i + " [" + parameter.getGenericParameterType() + "]");
        }

        // ③ 선택된 Resolver로 실제 값 resolve
        args[i] = this.resolvers.resolveArgument(
            parameter, mavContainer, request, this.dataBinderFactory);
    }
    return args;
}
```

---

## 🔬 내부 동작 원리

### 1. HandlerMethodArgumentResolverComposite — 캐시와 위임

```java
// HandlerMethodArgumentResolverComposite.java
public class HandlerMethodArgumentResolverComposite
        implements HandlerMethodArgumentResolver {

    private final List<HandlerMethodArgumentResolver> argumentResolvers = new ArrayList<>();

    // MethodParameter → Resolver 캐시 (ConcurrentHashMap)
    private final Map<MethodParameter, HandlerMethodArgumentResolver>
        argumentResolverCache = new ConcurrentHashMap<>(256);

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return getArgumentResolver(parameter) != null;
    }

    @Override
    public Object resolveArgument(MethodParameter parameter,
            ModelAndViewContainer mavContainer,
            NativeWebRequest webRequest,
            WebDataBinderFactory binderFactory) throws Exception {

        HandlerMethodArgumentResolver resolver = getArgumentResolver(parameter);
        if (resolver == null) {
            throw new IllegalArgumentException("Unsupported parameter type [" +
                parameter.getParameterType().getName() + "]...");
        }
        return resolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory);
    }

    @Nullable
    private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
        // 캐시 조회 (O(1))
        HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
        if (result == null) {
            // 캐시 미스 → 순회 탐색
            for (HandlerMethodArgumentResolver resolver : this.argumentResolvers) {
                if (resolver.supportsParameter(parameter)) {
                    result = resolver;
                    // 캐시에 저장 (이후 같은 파라미터는 O(1))
                    this.argumentResolverCache.put(parameter, result);
                    break;
                }
            }
        }
        return result;
    }
}
```

### 2. 기본 Resolver 등록 순서 (우선순위 순)

```java
// RequestMappingHandlerAdapter.getDefaultArgumentResolvers()
// 등록 순서 = 탐색 순서 = 우선순위

List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>(30);

// --- 어노테이션 기반 (구체적) ---
resolvers.add(new RequestParamMethodArgumentResolver(factory, false));  // @RequestParam (required=true)
resolvers.add(new RequestParamMapMethodArgumentResolver());             // @RequestParam Map
resolvers.add(new PathVariableMethodArgumentResolver());                // @PathVariable
resolvers.add(new PathVariableMapMethodArgumentResolver());             // @PathVariable Map
resolvers.add(new MatrixVariableMethodArgumentResolver());              // @MatrixVariable
resolvers.add(new MatrixVariableMapMethodArgumentResolver());           // @MatrixVariable Map
resolvers.add(new ServletModelAttributeMethodProcessor(false));        // @ModelAttribute (required=true)
resolvers.add(new RequestResponseBodyMethodProcessor(converters));     // @RequestBody
resolvers.add(new RequestPartMethodArgumentResolver(converters));      // @RequestPart
resolvers.add(new RequestHeaderMethodArgumentResolver(factory));       // @RequestHeader
resolvers.add(new RequestHeaderMapMethodArgumentResolver());           // @RequestHeader Map
resolvers.add(new ServletCookieValueMethodArgumentResolver(factory));  // @CookieValue
resolvers.add(new ExpressionValueMethodArgumentResolver(factory));     // @Value (SpEL)
resolvers.add(new SessionAttributeMethodArgumentResolver());           // @SessionAttribute
resolvers.add(new RequestAttributeMethodArgumentResolver());           // @RequestAttribute

// --- 타입 기반 (어노테이션 없이 타입으로 처리) ---
resolvers.add(new ServletRequestMethodArgumentResolver());   // HttpServletRequest, WebRequest 등
resolvers.add(new ServletResponseMethodArgumentResolver()); // HttpServletResponse
resolvers.add(new HttpEntityMethodProcessor(converters));   // HttpEntity<T>
resolvers.add(new RedirectAttributesMethodArgumentResolver());
resolvers.add(new ModelMethodProcessor());                  // Model
resolvers.add(new MapMethodProcessor());                    // Map
resolvers.add(new ErrorsMethodArgumentResolver());          // Errors, BindingResult
resolvers.add(new SessionStatusMethodArgumentResolver());   // SessionStatus
resolvers.add(new UriComponentsBuilderMethodArgumentResolver());

// --- 커스텀 Resolver (addArgumentResolvers()로 추가된 것) ---
if (getCustomArgumentResolvers() != null) {
    resolvers.addAll(getCustomArgumentResolvers());
}

// --- 폴백 (어노테이션 없는 파라미터 — 맨 마지막) ---
resolvers.add(new RequestParamMethodArgumentResolver(factory, true)); // 단순 타입 → @RequestParam 처리
resolvers.add(new ServletModelAttributeMethodProcessor(true));        // 복합 타입 → @ModelAttribute 처리
```

### 3. supportsParameter() — Resolver별 처리 기준

```java
// 1. RequestParamMethodArgumentResolver (useDefaultResolution=false)
//    → @RequestParam 어노테이션이 있거나
//    → Map 아닌 단순 타입 + @RequestParam 어노테이션 (required 버전)
@Override
public boolean supportsParameter(MethodParameter parameter) {
    if (parameter.hasParameterAnnotation(RequestParam.class)) {
        if (Map.class.isAssignableFrom(parameter.nestedIfOptional().getNestedParameterType())) {
            RequestParam requestParam = parameter.getParameterAnnotation(RequestParam.class);
            return (requestParam != null && StringUtils.hasText(requestParam.name()));
        }
        return true;
    }
    // ...
}

// 2. RequestResponseBodyMethodProcessor
//    → @RequestBody 어노테이션만 있으면 무조건 true
@Override
public boolean supportsParameter(MethodParameter parameter) {
    return parameter.hasParameterAnnotation(RequestBody.class);
}

// 3. ServletRequestMethodArgumentResolver
//    → 특정 타입 목록과 일치하면 true (어노테이션 없이 타입으로만 처리)
@Override
public boolean supportsParameter(MethodParameter parameter) {
    Class<?> paramType = parameter.getParameterType();
    return (WebRequest.class.isAssignableFrom(paramType) ||
            ServletRequest.class.isAssignableFrom(paramType) ||
            MultipartRequest.class.isAssignableFrom(paramType) ||
            HttpSession.class.isAssignableFrom(paramType) ||
            PushBuilder.class.isAssignableFrom(paramType) ||
            Principal.class.isAssignableFrom(paramType) ||
            InputStream.class.isAssignableFrom(paramType) ||
            Reader.class.isAssignableFrom(paramType) ||
            HttpMethod.class == paramType ||
            Locale.class == paramType ||
            TimeZone.class == paramType ||
            ZoneId.class == paramType);
}

// 4. 폴백 RequestParamMethodArgumentResolver (useDefaultResolution=true)
//    → @RequestParam 없어도 단순 타입(String, Long, int 등)이면 true
//    → BeanUtils.isSimpleProperty() 체크
@Override
public boolean supportsParameter(MethodParameter parameter) {
    if (parameter.hasParameterAnnotation(RequestParam.class)) { ... }
    else if (parameter.hasParameterAnnotation(RequestPart.class)) return false;
    else if (this.useDefaultResolution) {
        return BeanUtils.isSimpleProperty(parameter.nestedIfOptional().getNestedParameterType());
    }
    return false;
}
```

### 4. 등록 순서가 결과를 바꾸는 시나리오

```java
// 시나리오: 커스텀 Resolver가 @RequestBody를 처리하고 싶을 때

// 문제 상황:
// 기본 RequestResponseBodyMethodProcessor (index=7)
// 커스텀 MyRequestBodyResolver (index=24 — addArgumentResolvers 위치)
//
// @RequestBody 파라미터 탐색:
//   index=7: RequestResponseBodyMethodProcessor.supportsParameter() → true ← 여기서 선택됨!
//   index=24: 도달하지 않음
//
// 커스텀 Resolver가 선택되려면:
// RequestMappingHandlerAdapter.setArgumentResolvers()로 전체 목록에서
// 커스텀 Resolver를 index=7 앞에 배치해야 함

@Bean
public RequestMappingHandlerAdapter requestMappingHandlerAdapter(
        ApplicationContext ctx) {
    RequestMappingHandlerAdapter adapter = new RequestMappingHandlerAdapter();

    List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>();
    resolvers.add(new MyRequestBodyResolver());  // 먼저 추가
    resolvers.addAll(adapter.getArgumentResolvers()); // 기본 뒤에 추가
    adapter.setArgumentResolvers(resolvers);

    return adapter;
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 캐시 동작 확인 — 첫 요청 vs 두 번째 요청

```java
@Autowired
RequestMappingHandlerAdapter adapter;

@GetMapping("/resolver-cache")
public String checkCache() throws Exception {
    Field compositeField = RequestMappingHandlerAdapter.class
        .getDeclaredField("argumentResolvers");
    compositeField.setAccessible(true);
    HandlerMethodArgumentResolverComposite composite =
        (HandlerMethodArgumentResolverComposite) compositeField.get(adapter);

    Field cacheField = HandlerMethodArgumentResolverComposite.class
        .getDeclaredField("argumentResolverCache");
    cacheField.setAccessible(true);
    Map<?, ?> cache = (Map<?, ?>) cacheField.get(composite);

    return "캐시 크기: " + cache.size();
    // 첫 요청 후: 이 메서드의 파라미터 수만큼 증가
    // 반복 요청: 캐시 크기 고정 (재탐색 없음)
}
```

### 실험 2: Resolver 탐색 순서 TRACE 로그

```yaml
logging:
  level:
    org.springframework.web.method.support.InvocableHandlerMethod: TRACE
    org.springframework.web.method.support.HandlerMethodArgumentResolverComposite: TRACE
```

### 실험 3: 폴백 Resolver 동작 확인

```java
// 어노테이션 없는 단순 타입 파라미터
@GetMapping("/fallback")
public String fallback(String name, int page) {
    // name: 폴백 RequestParamMethodArgumentResolver (useDefaultResolution=true)
    //       → request.getParameter("name")
    // page: 동일 폴백 → request.getParameter("page") → Integer 변환
    return name + " page=" + page;
}
```

```bash
curl "http://localhost:8080/fallback?name=홍길동&page=2"
# → "홍길동 page=2"
# @RequestParam 없어도 쿼리 파라미터로 바인딩됨 (폴백 동작)
```

---

## 🌐 HTTP 레벨 분석

```
파라미터별 Resolver 선택 예시:

@PostMapping("/orders")
public Order createOrder(
    @RequestBody OrderDto dto,      // ① RequestResponseBodyMethodProcessor
    @RequestParam String source,    // ② RequestParamMethodArgumentResolver
    @PathVariable Long userId,      // ③ PathVariableMethodArgumentResolver
    HttpServletRequest request,     // ④ ServletRequestMethodArgumentResolver
    BindingResult result            // ⑤ ErrorsMethodArgumentResolver
)

POST /orders?source=web HTTP/1.1
Content-Type: application/json
{"productId":1,"quantity":2}

처리 순서:
  ① Content-Type: application/json → Jackson 역직렬화 → OrderDto
  ② request.getParameter("source") → "web"
  ③ URI_TEMPLATE_VARIABLES_ATTRIBUTE → userId
  ④ NativeWebRequest.getNativeRequest(HttpServletRequest.class)
  ⑤ DataBinder의 BindingResult 반환
```

---

## 🤔 트레이드오프

```
Chain of Responsibility 패턴:
  장점  새 파라미터 타입 = 새 Resolver 추가만으로 확장
        각 Resolver 독립 테스트 가능
        기본 동작 변경 없이 커스텀 Resolver 삽입 가능
  단점  디버깅 시 어느 Resolver가 처리했는지 파악 어려움
        Resolver 순서 관리 필요 (잘못된 순서 → 의도치 않은 Resolver 선택)

캐시 전략:
  장점  첫 요청 후 동일 파라미터에 대한 탐색 비용 제거
        트래픽 증가 시 성능 선형 유지
  단점  런타임에 Resolver 목록 변경 시 캐시 무효화 필요
        → 실제로는 Resolver 목록이 시작 시 고정되므로 문제 없음
```

---

## 📌 핵심 정리

```
Resolver 선택 흐름
  InvocableHandlerMethod.getMethodArgumentValues()
  → 파라미터마다 HandlerMethodArgumentResolverComposite.getArgumentResolver()
  → 캐시 히트: O(1) 반환
  → 캐시 미스: 등록 순서대로 supportsParameter() 탐색 → 캐시 저장

기본 Resolver 등록 순서 요약
  어노테이션 기반 (구체적): @RequestParam, @PathVariable, @RequestBody 등
  타입 기반: HttpServletRequest, HttpSession, Model 등
  커스텀: addArgumentResolvers()로 추가된 것 (기본보다 뒤)
  폴백: 어노테이션 없는 단순 타입 → @RequestParam 처리
         어노테이션 없는 복합 타입 → @ModelAttribute 처리

커스텀 Resolver를 기본보다 먼저 실행하려면
  setArgumentResolvers()로 전체 목록 직접 지정
  또는 RequestBodyAdvice / ResponseBodyAdvice 활용

캐시 키: MethodParameter (클래스 + 메서드 + 파라미터 인덱스)
```

---

## 🤔 생각해볼 문제

**Q1.** `HandlerMethodArgumentResolverComposite`의 캐시는 `ConcurrentHashMap`을 사용합니다. 왜 `HashMap`이 아닌가? 그리고 캐시 크기를 256으로 초기화한 이유는?

**Q2.** `@RequestParam`이 없는 `String name` 파라미터는 폴백 Resolver(`useDefaultResolution=true`)가 처리합니다. 만약 쿼리 파라미터에 `name`이 없으면 어떻게 되는가? 기본값은 `null`인가 예외인가?

**Q3.** 동일한 Controller 메서드가 두 개의 서로 다른 `DispatcherServlet`(각각 별도의 `RequestMappingHandlerAdapter`)에 등록되어 있다면, Resolver 캐시는 공유되는가 독립적인가?

> 💡 **해설**
>
> **Q1.** `ConcurrentHashMap`을 사용하는 이유는 `DispatcherServlet`이 멀티스레드 환경에서 동작하기 때문입니다. 여러 요청 스레드가 동시에 캐시에 읽고 쓸 수 있으므로 thread-safe한 자료구조가 필요합니다. 초기 크기 256은 일반적인 Spring MVC 애플리케이션의 Controller 메서드 파라미터 수를 고려한 휴리스틱입니다. `HashMap` 기본 크기(16)로 시작하면 파라미터가 많을 때 잦은 rehashing이 발생하므로 미리 큰 초기 크기를 설정합니다.
>
> **Q2.** 폴백 `RequestParamMethodArgumentResolver(useDefaultResolution=true)`의 경우, `@RequestParam`이 명시되지 않았으므로 `required` 속성도 없습니다. 이 경우 내부적으로 `required=false`로 처리되어 파라미터가 없으면 `null`이 반환됩니다. 단, 파라미터 타입이 `int`나 `long` 같은 primitive type이면 `null`을 주입할 수 없어 `NullPointerException` 또는 `TypeMismatchException`이 발생합니다. 이 경우 `Integer`나 `Long` 같은 wrapper 타입을 사용하거나 명시적 `@RequestParam(required=false)`를 선언해야 합니다.
>
> **Q3.** 독립적입니다. `argumentResolverCache`는 `HandlerMethodArgumentResolverComposite` 인스턴스 필드이고, `HandlerMethodArgumentResolverComposite`는 `RequestMappingHandlerAdapter` 인스턴스가 생성할 때 내부적으로 생성됩니다. 두 개의 `DispatcherServlet`이 각자의 `RequestMappingHandlerAdapter`를 가진다면 캐시도 각각 독립적으로 유지됩니다. 같은 Controller 메서드가 두 Adapter 모두에서 처음 호출될 때 각자 캐시를 채우게 됩니다.

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: @RequestBody와 HttpMessageConverter ➡️](./02-request-body-message-converter.md)**

</div>
