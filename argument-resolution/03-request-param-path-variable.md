# @RequestParam vs @PathVariable 처리 — 두 Resolver의 소스 비교

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `RequestParamMethodArgumentResolver`와 `PathVariableMethodArgumentResolver`의 값 추출 위치가 다른 이유는?
- 타입 변환이 `ConversionService`를 거치는 구체적인 경로는?
- `required=true`일 때 값이 없으면 각각 어떤 예외가 발생하는가?
- `defaultValue`는 어떻게 처리되며, SpEL이나 `${property}` 참조도 가능한가?
- `@RequestParam Map<String, String>` 과 `@RequestParam MultiValueMap<String, String>`의 차이는?
- `@RequestParam` 없이 선언한 단순 타입 파라미터와 `@RequestParam`을 명시한 파라미터의 처리 차이는?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: 같은 값처럼 보이지만 출처가 다르다

```
GET /users/42?page=2&sort=name

  42   → URL 경로의 일부    → URI 변수 (HandlerMapping이 미리 추출)
  page → URL 쿼리 파라미터  → HttpServletRequest.getParameter()
  sort → URL 쿼리 파라미터  → HttpServletRequest.getParameter()

POST /users HTTP/1.1
Content-Type: application/x-www-form-urlencoded
name=홍길동&email=hong@test.com

  name  → 요청 본문 폼 데이터 → HttpServletRequest.getParameter()
  email → 요청 본문 폼 데이터 → HttpServletRequest.getParameter()

@RequestParam은 "쿼리 파라미터 + 폼 데이터" 통합 처리
  getParameter()는 두 곳 모두에서 찾아줌 (서블릿 스펙)
@PathVariable은 "URI 변수" 전용
  HandlerMapping이 미리 request 속성에 저장해 둔 Map에서 꺼냄
```

---

## 😱 흔한 오해 또는 실수

### Before: @RequestParam은 쿼리 파라미터만 읽는다

```java
// ❌ 잘못된 이해
// "@RequestParam은 ?key=value 쿼리 스트링만 읽는다"

// ✅ 실제:
// HttpServletRequest.getParameter()는 서블릿 스펙에 따라
// 쿼리 스트링 + application/x-www-form-urlencoded 본문 모두 반환

@PostMapping("/login")
public String login(
    @RequestParam String username,  // 폼 본문의 username도 읽음
    @RequestParam String password) { ... }
// curl -X POST -d "username=admin&password=1234" /login → 정상 동작

// 단, application/json 본문의 필드는 읽지 못함
// → JSON 본문은 @RequestBody로 읽어야 함
```

### Before: required=true + 값 없음 → 항상 400 응답

```
❌ 잘못된 이해:
  "@RequestParam(required=true) 에서 값 없으면 항상 400"

✅ 실제 예외 종류:
  @RequestParam(required=true) 값 없음
  → MissingServletRequestParameterException → 400 ✅

  @PathVariable(required=true) 값 없음
  → URL 패턴 자체가 매칭 안 됨 → 404 (핸들러 선택 전 실패)
  → 또는 변수가 Optional 없이 null → MissingPathVariableException → 500

  차이: @RequestParam은 URL 매칭 후 확인 → 400
        @PathVariable은 URL 매칭 자체가 조건 → 404 or 500
```

---

## ✨ 올바른 이해와 사용

### After: 두 Resolver의 값 추출 경로 비교

```
@RequestParam String page:
  RequestParamMethodArgumentResolver.resolveName()
  → request.getParameterValues("page")   ← HttpServletRequest API
  → String[] → 첫 번째 값 반환
  → ConversionService: String → 파라미터 타입

@PathVariable Long id:
  PathVariableMethodArgumentResolver.resolveName()
  → request.getAttribute(URI_TEMPLATE_VARIABLES_ATTRIBUTE)  ← HandlerMapping이 저장한 Map
  → Map.get("id") → String "42"
  → ConversionService: String → Long
```

---

## 🔬 내부 동작 원리

### 1. 공통 부모 — AbstractNamedValueMethodArgumentResolver

```java
// 두 Resolver 모두 AbstractNamedValueMethodArgumentResolver 상속
// 공통 흐름:
@Override
public final Object resolveArgument(MethodParameter parameter, ...) throws Exception {
    NamedValueInfo namedValueInfo = getNamedValueInfo(parameter);
    // → @RequestParam("name") 또는 파라미터명에서 이름 결정

    // 각 서브클래스가 구현: 실제 값 조회 위치가 다름
    Object arg = resolveName(resolvedName.toString(), nestedParameter, webRequest);

    // required 처리
    if (arg == null) {
        if (namedValueInfo.defaultValue != null) {
            arg = resolveStringValue(namedValueInfo.defaultValue); // SpEL/프로퍼티 처리
        } else if (namedValueInfo.required && !nestedParameter.isOptional()) {
            handleMissingValue(namedValueInfo.name, nestedParameter, webRequest);
            // @RequestParam → MissingServletRequestParameterException (400)
            // @PathVariable  → MissingPathVariableException (500)
        }
        arg = handleNullValue(namedValueInfo.name, arg, nestedParameter.getNestedParameterType());
    }

    // 타입 변환
    if (binderFactory != null) {
        WebDataBinder binder = binderFactory.createBinder(webRequest, null, namedValueInfo.name);
        arg = binder.convertIfNecessary(arg, parameter.getParameterType(), parameter);
        // → ConversionService → StringToLongConverter 등
    }
    return arg;
}
```

### 2. RequestParamMethodArgumentResolver.resolveName()

```java
@Override
@Nullable
protected Object resolveName(String name, MethodParameter parameter,
        NativeWebRequest request) throws Exception {

    HttpServletRequest servletRequest = request.getNativeRequest(HttpServletRequest.class);

    // Multipart 요청 처리
    if (servletRequest != null) {
        if (MultipartResolutionDelegate.isMultipartArgument(parameter)) {
            // MultipartFile 타입이면 multipart 파트에서 처리
            if (!MultipartResolutionDelegate.isMultipartRequest(servletRequest)) {
                return UNRESOLVABLE;
            }
            return MultipartResolutionDelegate.resolveMultipartArgument(name, parameter, servletRequest);
        }
    }

    Object mpArg = MultipartResolutionDelegate.resolveMultipartArgument(name, parameter, servletRequest);
    if (mpArg != MultipartResolutionDelegate.UNRESOLVABLE) {
        return mpArg;
    }

    Object arg = null;
    MultipartRequest multipartRequest = request.getNativeRequest(MultipartRequest.class);
    if (multipartRequest != null) {
        List<MultipartFile> files = multipartRequest.getFiles(name);
        if (!files.isEmpty()) return (files.size() == 1 ? files.get(0) : files);
    }

    // 일반 파라미터: 쿼리 스트링 + 폼 데이터 통합
    String[] paramValues = request.getParameterValues(name);
    //  request.getParameterValues():
    //    → 쿼리 스트링 ?name=value
    //    → application/x-www-form-urlencoded 본문의 name=value
    //    → 둘 다 합쳐서 반환

    if (paramValues != null) {
        arg = (paramValues.length == 1 ? paramValues[0] : paramValues);
        // List<String> 파라미터라면 배열 그대로, String이면 첫 번째 값
    }
    return arg;
}
```

### 3. PathVariableMethodArgumentResolver.resolveName()

```java
@Override
@Nullable
protected Object resolveName(String name, MethodParameter parameter,
        NativeWebRequest request) throws Exception {

    HttpServletRequest servletRequest = request.getNativeRequest(HttpServletRequest.class);

    @SuppressWarnings("unchecked")
    Map<String, String> uriTemplateVars = (Map<String, String>)
        servletRequest.getAttribute(HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE);
    // ← HandlerMapping.handleMatch()에서 미리 저장해 둔 Map
    // ← {id: "42", userId: "7", ...}

    return (uriTemplateVars != null ? uriTemplateVars.get(name) : null);
    // name = "id" → "42"
}
```

### 4. ConversionService를 통한 타입 변환

```java
// binder.convertIfNecessary("42", Long.class)
// → WebDataBinder → ConversionService 위임

// DefaultFormattingConversionService (Spring Boot 기본):
//   StringToLongConverter         "42" → 42L
//   StringToIntegerConverter      "2"  → 2
//   StringToEnumConverter         "ACTIVE" → Status.ACTIVE
//   StringToLocalDateConverter    "2024-03-10" → LocalDate
//   StringToUUIDConverter         "uuid-str" → UUID
//   ... 수십 가지

// 변환 실패 시:
// TypeMismatchException
// → AbstractNamedValueMethodArgumentResolver 에서 캐치
// → MethodArgumentTypeMismatchException 로 래핑
// → DefaultHandlerExceptionResolver → 400 Bad Request

// 커스텀 Converter 등록:
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addConverter(new StringToMoneyConverter());
    }
}
```

### 5. defaultValue 처리 — SpEL과 프로퍼티 참조

```java
// defaultValue는 문자열이지만 표현식 처리 가능
@GetMapping("/users")
public List<User> list(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") int size,
    @RequestParam(defaultValue = "${app.default-sort:name}") String sort
    //  → ${app.default-sort:name}: 프로퍼티 없으면 "name" 사용
) { ... }

// 내부 처리: AbstractNamedValueMethodArgumentResolver.resolveStringValue()
@Nullable
private Object resolveStringValue(String value) {
    if (this.configurableBeanFactory == null) return value;
    String placeholdersResolved = this.configurableBeanFactory.resolveEmbeddedValue(value);
    // ${property.key} 처리
    BeanExpressionResolver exprResolver = this.configurableBeanFactory.getBeanExpressionResolver();
    if (exprResolver == null || this.expressionContext == null) return placeholdersResolved;
    return exprResolver.evaluate(placeholdersResolved, this.expressionContext);
    // #{SpEL expression} 처리
}
```

### 6. @RequestParam Map 변형들

```java
// 모든 파라미터를 Map으로 받기
@GetMapping("/search")
public String search(@RequestParam Map<String, String> params) {
    // GET /search?q=spring&type=doc&page=1
    // → {q: "spring", type: "doc", page: "1"}
    // 같은 이름 파라미터가 여러 개면 첫 번째 값만 담김
}

// MultiValueMap: 같은 이름 파라미터 여러 개 지원
@GetMapping("/filter")
public String filter(@RequestParam MultiValueMap<String, String> params) {
    // GET /filter?tag=java&tag=spring&tag=mvc
    // → {tag: ["java", "spring", "mvc"]}
}

// 처리 Resolver:
// Map<String,String>       → RequestParamMapMethodArgumentResolver
// MultiValueMap<String,String> → RequestParamMapMethodArgumentResolver (multiValueMap=true)
```

---

## 💻 실험으로 확인하기

### 실험 1: @RequestParam vs @PathVariable 값 출처 확인

```java
@GetMapping("/test/{id}")
public Map<String, Object> test(
    @PathVariable String id,
    @RequestParam(required = false) String id2,
    HttpServletRequest req) {

    return Map.of(
        "pathVariable(id)",      id,
        "requestParam(id2)",     String.valueOf(id2),
        "requestURI",            req.getRequestURI(),
        "queryString",           String.valueOf(req.getQueryString())
    );
}
```

```bash
curl "http://localhost:8080/test/42?id2=99"
# {
#   "pathVariable(id)": "42",       ← URI 변수 맵에서
#   "requestParam(id2)": "99",      ← getParameter()에서
#   "requestURI": "/test/42",
#   "queryString": "id2=99"
# }
```

### 실험 2: 타입 변환 체인 확인

```java
@GetMapping("/convert")
public Map<String, Object> convert(
    @RequestParam LocalDate from,       // "2024-03-10" → LocalDate
    @RequestParam Status status,        // "ACTIVE" → enum Status
    @RequestParam Optional<Long> limit  // 없으면 Optional.empty()
) { ... }
```

```bash
curl "http://localhost:8080/convert?from=2024-03-10&status=ACTIVE"
# → from=2024-03-10(LocalDate), status=ACTIVE(enum), limit=Optional.empty
```

### 실험 3: required 동작 비교

```bash
# @RequestParam(required=true) 값 없음
curl "http://localhost:8080/users"   # name 파라미터 없이
# HTTP/1.1 400 Bad Request
# "Required request parameter 'name' is not present"

# @RequestParam(required=false, defaultValue="guest") 값 없음
curl "http://localhost:8080/users"
# → name="guest" (defaultValue 사용)
```

---

## 🌐 HTTP 레벨 분석

```
GET /search?q=spring&tag=java&tag=mvc&page=2 HTTP/1.1

파라미터 추출:
  @RequestParam String q          → "spring"
  @RequestParam List<String> tag  → ["java", "mvc"]  (같은 이름 복수)
  @RequestParam int page          → 2 (String→int 변환)

  request.getParameterValues("q")   → ["spring"]
  request.getParameterValues("tag") → ["java", "mvc"]
  request.getParameterValues("page")→ ["2"]

POST /login HTTP/1.1
Content-Type: application/x-www-form-urlencoded
username=admin&password=secret

  @RequestParam String username → "admin"
  @RequestParam String password → "secret"
  request.getParameter("username") → "admin" (본문에서 읽음)

타입 변환 실패:
  GET /items?count=abc  (@RequestParam int count)
  → ConversionFailedException
  → MethodArgumentTypeMismatchException
  HTTP/1.1 400 Bad Request
```

---

## 🤔 트레이드오프

```
@RequestParam vs @PathVariable 설계 선택:
  @PathVariable:
    + URL 구조에 자원 식별자 포함 → RESTful (GET /users/42)
    + 필수값 → URL 매칭 자체가 보장
    - URL 패턴 변경 시 클라이언트 URL 변경 필요
  @RequestParam:
    + 선택적 파라미터, 기본값 설정 용이
    + 필터/정렬/페이징 등 부가 정보 전달에 적합
    - URL이 길어질 수 있음

required=true (기본) vs required=false:
  required=true:  누락 시 400 → 클라이언트 오류 즉시 피드백
  required=false: 누락 허용 → defaultValue 또는 null 처리 필요
  → 필수 식별자는 required=true, 선택 필터는 required=false

defaultValue 활용:
  장점  파라미터 없어도 합리적 기본 동작 제공
        API 하위 호환성 유지 (새 파라미터 추가 시)
  단점  기본값이 문서화되지 않으면 클라이언트 혼란
        → @Parameter(description="기본값: 20") 등 API 문서 병행 권장
```

---

## 📌 핵심 정리

```
값 추출 위치 차이
  @RequestParam → request.getParameterValues(name)
                  (쿼리 스트링 + form 본문 통합)
  @PathVariable → request.getAttribute(URI_TEMPLATE_VARIABLES_ATTRIBUTE)
                  (HandlerMapping이 미리 저장한 Map)

공통 처리 흐름 (AbstractNamedValueMethodArgumentResolver)
  ① resolveName() → 각 서브클래스가 위치 결정
  ② null이면 defaultValue 적용 (${property}, #{SpEL} 지원)
  ③ required 체크 → 없으면 예외
  ④ ConversionService로 타입 변환

타입 변환 실패 → 400 Bad Request
  MethodArgumentTypeMismatchException
  → DefaultHandlerExceptionResolver

required 누락 예외 종류
  @RequestParam(required=true) 누락 → MissingServletRequestParameterException → 400
  @PathVariable 누락 → URL 매칭 실패(404) 또는 MissingPathVariableException(500)

@RequestParam Map 변형
  Map<String,String>         → 동일 이름 복수 시 첫 번째 값만
  MultiValueMap<String,String> → 동일 이름 복수 지원
```

---

## 🤔 생각해볼 문제

**Q1.** `@RequestParam String[] tags` 와 `@RequestParam List<String> tags` 는 동작이 다른가? `GET /items?tags=a&tags=b&tags=c` 요청에서 각각 어떤 값이 주입되는가?

**Q2.** `@RequestParam(defaultValue = "#{T(java.time.LocalDate).now()}")` 처럼 SpEL로 현재 날짜를 기본값으로 설정하면 매 요청마다 새로운 날짜가 반환되는가?

**Q3.** `@PathVariable(required = false) Long id` 를 선언했을 때 두 가지 URL 패턴 `@GetMapping({"/users", "/users/{id}"})` 로 매핑하면 어떻게 동작하는가? `/users` 요청 시 `id`는 `null`인가 예외인가?

> 💡 **해설**
>
> **Q1.** 동작은 동일합니다. `RequestParamMethodArgumentResolver.resolveName()`에서 `request.getParameterValues("tags")` 로 `["a","b","c"]` String 배열을 얻고, 파라미터 타입이 `String[]`이면 그대로 반환, `List<String>`이면 `Arrays.asList()`로 변환합니다. 두 경우 모두 `["a","b","c"]`가 주입됩니다. `ConversionService`의 `ArrayToCollectionConverter`가 배열→List 변환을 처리합니다.
>
> **Q2.** 네, 매 요청마다 새로운 날짜가 반환됩니다. `resolveStringValue()`는 `BeanExpressionResolver.evaluate()`를 매 호출마다 실행합니다. `T(java.time.LocalDate).now()`는 호출 시점의 현재 날짜를 반환하는 SpEL 표현식이므로 매 요청 시 평가되어 그 날의 날짜가 기본값으로 사용됩니다.
>
> **Q3.** `/users` 요청 시 `id`는 `null`이 됩니다. `@PathVariable(required = false)`는 해당 URI 변수가 없을 때 `null` 또는 `Optional.empty()`를 허용합니다. `/users` 패턴으로 매핑된 경우 `URI_TEMPLATE_VARIABLES_ATTRIBUTE` 맵에 `id` 키가 없으므로 `resolveName()`이 `null`을 반환합니다. `required=false`이므로 `handleMissingValue()`가 호출되지 않고 `null`이 그대로 주입됩니다. 단, `Long` 타입이면 null이 가능하지만 `long` primitive 타입이면 NullPointerException이 발생합니다.

---

<div align="center">

**[⬅️ 이전: @RequestBody와 HttpMessageConverter](./02-request-body-message-converter.md)** | **[홈으로 🏠](../README.md)** | **[다음: @ModelAttribute 데이터 바인딩 ➡️](./04-model-attribute-binding.md)**

</div>
