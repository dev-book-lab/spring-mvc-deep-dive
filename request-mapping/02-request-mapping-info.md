# RequestMappingInfo 생성과 매칭 — 요청 조건을 캡슐화하는 방법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `RequestMappingInfo`는 어떤 조건들을 캡슐화하고, 각 조건의 역할은 무엇인가?
- `getMatchingCondition(request)`는 어떤 순서로 조건을 평가하고, 하나라도 실패하면 어떻게 되는가?
- URL이 같아도 `consumes`나 `produces` 조건이 다르면 서로 다른 핸들러로 매핑되는 원리는?
- 복수의 후보 `RequestMappingInfo` 중 "더 구체적인" 것을 선택하는 기준은?
- `params`와 `headers` 조건은 어떤 형식으로 작성하고, 어떻게 매칭되는가?
- `RequestMappingInfo`의 빌더 API를 활용한 프로그래밍적 매핑 등록은 어떻게 하는가?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: URL 하나만으로는 핸들러를 특정할 수 없는 경우가 있다

```java
// 같은 URL, 다른 Content-Type에 따라 다른 처리가 필요한 경우
@PostMapping(value = "/users", consumes = "application/json")
public User createFromJson(@RequestBody UserDto dto) { ... }

@PostMapping(value = "/users", consumes = "application/x-www-form-urlencoded")
public User createFromForm(@ModelAttribute UserDto dto) { ... }

// URL만으로는 구분 불가 → consumes 조건으로 구분
// Content-Type: application/json → 첫 번째 핸들러
// Content-Type: application/x-www-form-urlencoded → 두 번째 핸들러
```

```
URL 외에도 여러 기준으로 핸들러를 특정해야 하는 상황:
  같은 URL, 다른 HTTP 메서드     → @GetMapping vs @PostMapping
  같은 URL, 다른 Content-Type  → consumes 조건
  같은 URL, 다른 Accept 헤더   → produces 조건
  같은 URL, 특정 파라미터 존재  → params 조건
  같은 URL, 특정 헤더 존재     → headers 조건

해결: 모든 조건을 하나의 RequestMappingInfo에 캡슐화
  → 요청과 비교 시 모든 조건이 AND로 적용
  → 하나라도 불일치 → 이 핸들러는 선택 안 됨
```

---

## 😱 흔한 오해 또는 실수

### Before: params 조건은 요청 바디를 검사한다

```java
// ❌ 잘못된 이해
// "@RequestMapping(params = "action=create")가 POST body의 action 파라미터를 확인한다"

@GetMapping(value = "/users", params = "type=admin")
public List<User> getAdminUsers() { ... }

// ✅ 실제:
// params는 쿼리 파라미터(URL 쿼리 스트링) 또는 form 파라미터를 검사
// GET /users?type=admin    → 매칭 ✅
// GET /users?type=regular  → 불일치 ❌ (405 아닌 400/404 수준 동작)
// POST body의 파라미터는 검사하지 않음
// → @RequestBody JSON 바디와 관련 없음
```

### Before: produces 조건은 응답 Content-Type을 강제한다

```java
// ❌ 잘못된 이해
// "@GetMapping(produces = "application/json")이 항상 JSON을 반환한다"

@GetMapping(value = "/users/{id}", produces = "application/json")
public User getUser(@PathVariable Long id) { ... }

// ✅ 실제:
// produces 조건은 매핑 조건 (라우팅 기준)이지 응답 형식 지정이 아님
// "Accept: application/json"을 보낸 요청만 이 핸들러로 라우팅
// "Accept: text/html"을 보낸 요청 → 이 핸들러 매칭 안 됨 → 406 Not Acceptable
// 실제 응답 Content-Type은 HttpMessageConverter가 결정
```

---

## ✨ 올바른 이해와 사용

### After: RequestMappingInfo 7가지 조건 컴포넌트

```java
// RequestMappingInfo 내부 구조
public final class RequestMappingInfo implements RequestCondition<RequestMappingInfo> {

    @Nullable
    private final PathPatternsRequestCondition pathPatternsCondition;
    // URL 패턴 조건: "/users/{id}", "/users/**"
    // PathPattern 기반 (Spring 5.3+ 기본)

    @Nullable
    private final PatternsRequestCondition patternsCondition;
    // URL 패턴 조건 (AntPathMatcher 기반, 레거시)

    private final RequestMethodsRequestCondition methodsCondition;
    // HTTP 메서드 조건: GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS, TRACE
    // 비어있으면 모든 메서드 허용

    private final ParamsRequestCondition paramsCondition;
    // 쿼리 파라미터 조건: "param1", "!param2", "param3=value"

    private final HeadersRequestCondition headersCondition;
    // 헤더 조건: "My-Header", "!My-Header", "My-Header=value"
    // Content-Type, Accept는 별도 consumes/produces로 처리

    private final ConsumesRequestCondition consumesCondition;
    // Content-Type 조건: "application/json", "!text/plain"
    // 요청 본문 미디어 타입 기준

    private final ProducesRequestCondition producesCondition;
    // Accept 헤더 조건: "application/json", "text/html"
    // 응답 미디어 타입 기준

    @Nullable
    private final RequestConditionHolder customConditionHolder;
    // 커스텀 조건 (RequestCondition 구현체)
}
```

### getMatchingCondition() — 조건 평가 순서

```java
// RequestMappingInfo.getMatchingCondition()
@Override
@Nullable
public RequestMappingInfo getMatchingCondition(HttpServletRequest request) {

    // ① HTTP 메서드 조건 평가 (가장 빠른 실패)
    RequestMethodsRequestCondition methods = this.methodsCondition.getMatchingCondition(request);
    if (methods == null) {
        return null;  // 즉시 탈락
    }

    // ② 파라미터 조건
    ParamsRequestCondition params = this.paramsCondition.getMatchingCondition(request);
    if (params == null) {
        return null;
    }

    // ③ 헤더 조건
    HeadersRequestCondition headers = this.headersCondition.getMatchingCondition(request);
    if (headers == null) {
        return null;
    }

    // ④ consumes 조건 (Content-Type 헤더)
    ConsumesRequestCondition consumes = this.consumesCondition.getMatchingCondition(request);
    if (consumes == null) {
        return null;  // → HttpMediaTypeNotSupportedException (415)
    }

    // ⑤ produces 조건 (Accept 헤더)
    ProducesRequestCondition produces = this.producesCondition.getMatchingCondition(request);
    if (produces == null) {
        return null;  // → HttpMediaTypeNotAcceptableException (406)
    }

    // ⑥ URL 패턴 조건 (PathPattern 매칭)
    PathPatternsRequestCondition patterns = (this.pathPatternsCondition != null
        ? this.pathPatternsCondition.getMatchingCondition(request) : null);
    if (patterns == null) {
        return null;
    }

    // ⑦ 커스텀 조건
    RequestConditionHolder custom = this.customConditionHolder.getMatchingCondition(request);
    if (custom == null) {
        return null;
    }

    // 모든 조건 통과 → 매칭된 새 RequestMappingInfo 반환
    return new RequestMappingInfo(this.name, patterns, patternCondition,
        methods, params, headers, consumes, produces, custom, this.options);
}
```

---

## 🔬 내부 동작 원리

### 1. ConsumesRequestCondition — Content-Type 매칭

```java
// ConsumesRequestCondition.getMatchingCondition()
@Override
@Nullable
public ConsumesRequestCondition getMatchingCondition(HttpServletRequest request) {
    // Preflight OPTIONS 요청은 consumes 검사 스킵
    if (CorsUtils.isPreFlightRequest(request)) {
        return PRE_FLIGHT_MATCH;
    }
    if (isEmpty()) {
        return this;  // consumes 조건 없으면 항상 매칭
    }

    Set<MediaType> contentTypes;
    try {
        // Content-Type 헤더 파싱
        contentTypes = new LinkedHashSet<>(MediaType.parseMediaTypes(
            request.getHeader(HttpHeaders.CONTENT_TYPE)));
    } catch (InvalidMediaTypeException ex) {
        return null;
    }
    if (contentTypes.isEmpty()) {
        contentTypes = Collections.singleton(MediaType.APPLICATION_OCTET_STREAM);
    }

    // 각 미디어 타입 표현식과 비교
    for (MediaTypeExpression expression : this.expressions) {
        // expression.match(): Content-Type이 조건과 호환되는지 확인
        // consumes = "application/json" → MediaType.APPLICATION_JSON.isCompatibleWith(contentType)
        if (!expression.match(contentTypes)) {
            return null;  // 불일치 → 이 핸들러 탈락
        }
    }
    return this;
}
```

### 2. 구체성 비교 — compareTo()

```java
// 여러 후보 RequestMappingInfo 중 가장 구체적인 것을 선택
// AbstractHandlerMethodMapping에서 bestMatch 선택 시 사용

// RequestMappingInfo.compareTo()
@Override
public int compareTo(RequestMappingInfo other, HttpServletRequest request) {
    int result;

    // 패턴 구체성 비교 우선
    if (this.pathPatternsCondition != null && other.pathPatternsCondition != null) {
        result = this.pathPatternsCondition.compareTo(other.pathPatternsCondition, request);
        if (result != 0) return result;
    }

    // 파라미터 조건 비교 (조건 많을수록 구체적)
    result = this.paramsCondition.compareTo(other.paramsCondition, request);
    if (result != 0) return result;

    // 헤더 조건 비교
    result = this.headersCondition.compareTo(other.headersCondition, request);
    if (result != 0) return result;

    // consumes 조건 비교
    result = this.consumesCondition.compareTo(other.consumesCondition, request);
    if (result != 0) return result;

    // produces 조건 비교
    result = this.producesCondition.compareTo(other.producesCondition, request);
    if (result != 0) return result;

    // 메서드 조건 비교 (구체적 메서드 > 모든 메서드 허용)
    result = this.methodsCondition.compareTo(other.methodsCondition, request);
    return result;
}

// 패턴 구체성 예시:
//   "/users/admin"  vs "/users/{id}" → "/users/admin" 더 구체적 (변수 없음)
//   "/users/{id}"   vs "/users/**"   → "/users/{id}" 더 구체적 (이중 와일드카드보다)
//   "/users/{id}"   vs "/users/*"    → "/users/{id}" 더 구체적 (단일 세그먼트 변수)
```

### 3. params 조건 표현식

```java
// ParamsRequestCondition 표현식 형식:
// "param"        → 파라미터 존재 여부
// "!param"       → 파라미터 없어야 함
// "param=value"  → 파라미터가 특정 값이어야 함
// "param!=value" → 파라미터가 특정 값이 아니어야 함

@GetMapping(value = "/users", params = "type=admin")
public List<User> adminOnly() { ... }
// GET /users?type=admin → 매칭 ✅
// GET /users?type=user  → 불일치 ❌

@GetMapping(value = "/users", params = "!deleted")
public List<User> activeUsers() { ... }
// GET /users            → 매칭 ✅ (deleted 파라미터 없음)
// GET /users?deleted=true → 불일치 ❌

// 내부 구현:
// ParamExpression.match(request):
//   "type=admin" → request.getParameter("type").equals("admin")
//   "!deleted"   → request.getParameter("deleted") == null
```

### 4. headers 조건 표현식

```java
// headers 조건 표현식 형식 (params와 동일 패턴):
@GetMapping(value = "/users", headers = "X-API-Version=2")
public List<User> v2Users() { ... }
// X-API-Version: 2 헤더가 있는 요청만 매칭

@PostMapping(value = "/data", headers = "!X-Deprecated")
public void postData() { ... }
// X-Deprecated 헤더가 없는 요청만 매칭

// 주의: Content-Type, Accept 헤더는 headers가 아닌 consumes/produces로 처리
// headers = "Content-Type=application/json" → 동작하지만 consumes 사용 권장
// headers = "Accept=application/json"       → 동작하지만 produces 사용 권장
```

### 5. 프로그래밍적 매핑 등록 (RequestMappingInfo 빌더)

```java
// 런타임에 또는 코드로 직접 매핑 등록
@Autowired
RequestMappingHandlerMapping requestMappingHandlerMapping;

@PostConstruct
public void registerProgrammaticMapping() throws NoSuchMethodException {
    // RequestMappingInfo 직접 생성
    RequestMappingInfo info = RequestMappingInfo
        .paths("/api/dynamic")
        .methods(RequestMethod.GET)
        .produces(MediaType.APPLICATION_JSON_VALUE)
        .params("version=2")
        .options(requestMappingHandlerMapping.getBuilderConfiguration())
        .build();

    // 핸들러 메서드와 연결
    Method method = DynamicController.class.getDeclaredMethod("handle");
    requestMappingHandlerMapping.registerMapping(info,
        applicationContext.getBean(DynamicController.class), method);
}
```

---

## 💻 실험으로 확인하기

### 실험 1: consumes 조건 차이 확인

```java
@RestController
@RequestMapping("/data")
public class ContentTypeController {

    @PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE)
    public String fromJson(@RequestBody String body) {
        return "JSON: " + body;
    }

    @PostMapping(consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE)
    public String fromForm(@RequestParam String body) {
        return "Form: " + body;
    }
}
```

```bash
# JSON으로 요청
curl -X POST http://localhost:8080/data \
     -H "Content-Type: application/json" \
     -d '{"key":"value"}'
# → "JSON: {\"key\":\"value\"}"

# Form으로 요청
curl -X POST http://localhost:8080/data \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -d "body=hello"
# → "Form: hello"

# 지원하지 않는 Content-Type
curl -X POST http://localhost:8080/data \
     -H "Content-Type: text/plain" \
     -d "plain"
# HTTP/1.1 415 Unsupported Media Type
```

### 실험 2: produces 조건 + Accept 헤더 매칭

```java
@GetMapping(value = "/format", produces = MediaType.APPLICATION_JSON_VALUE)
public Map<String, String> asJson() {
    return Map.of("format", "json");
}

@GetMapping(value = "/format", produces = MediaType.APPLICATION_XML_VALUE)
public String asXml() {
    return "<format>xml</format>";
}
```

```bash
curl -H "Accept: application/json" http://localhost:8080/format
# → {"format":"json"}

curl -H "Accept: application/xml" http://localhost:8080/format
# → <format>xml</format>

curl -H "Accept: text/html" http://localhost:8080/format
# HTTP/1.1 406 Not Acceptable
```

### 실험 3: 구체성 비교 직접 확인

```java
@GetMapping("/items/special")   // 구체적 URL
public String special() { return "special"; }

@GetMapping("/items/{id}")       // 패턴 URL
public String byId(@PathVariable String id) { return "id=" + id; }

@GetMapping("/items/**")         // 이중 와일드카드
public String all() { return "all"; }
```

```bash
curl http://localhost:8080/items/special  # → "special"  (가장 구체적)
curl http://localhost:8080/items/123      # → "id=123"   (패턴 매칭)
curl http://localhost:8080/items/a/b/c   # → "all"       (이중 와일드카드)
```

---

## 🌐 HTTP 레벨 분석

```
조건별 실패 시 HTTP 상태 코드:

HTTP 메서드 불일치:
  GET /users 요청 → DELETE /users만 있음
  HTTP/1.1 405 Method Not Allowed
  Allow: DELETE, HEAD, OPTIONS

consumes 불일치 (Content-Type):
  POST /users -H "Content-Type: text/plain"
  → consumes = "application/json" 조건 불일치
  HTTP/1.1 415 Unsupported Media Type

produces 불일치 (Accept):
  GET /users -H "Accept: application/pdf"
  → produces = "application/json" 조건 불일치
  HTTP/1.1 406 Not Acceptable

params 불일치:
  GET /users (params="type=admin" 조건)
  → URL은 같지만 다른 핸들러 탐색 → 없으면 404
  HTTP/1.1 404 Not Found (다른 핸들러 없을 때)

headers 불일치:
  GET /users (headers="X-API-Version=2" 조건)
  → 헤더 없으면 다른 핸들러 탐색 → 없으면 404
  HTTP/1.1 404 Not Found
```

---

## 🤔 트레이드오프

```
consumes/produces 조건 활용:
  장점  같은 URL에서 Content-Type/Accept 기반 분기 → RESTful 설계
        URL 구조 단순하게 유지 가능
        Spring의 Content Negotiation과 자연스럽게 통합
  단점  설정을 모르는 클라이언트는 415/406 오류로 혼란
        → 문서화 필수 (OpenAPI/Swagger)

params/headers 조건 활용:
  장점  버전 관리에 유용 (headers = "API-Version=2")
        A/B 테스트, 기능 플래그 구현에 활용 가능
  단점  URL만 봐서는 어떤 핸들러가 선택되는지 알기 어려움
        캐시 처리 복잡 (동일 URL, 다른 헤더 → 다른 응답)
        RESTful 순수주의 관점에서 URL 설계 원칙에 어긋남

커스텀 RequestCondition:
  장점  임의의 조건으로 핸들러 선택 가능 (요청 IP, 세션 속성 등)
  단점  복잡도 증가, 예측하기 어려운 라우팅
```

---

## 📌 핵심 정리

```
RequestMappingInfo 7가지 조건 컴포넌트
  PathPatternsRequestCondition   : URL 패턴 (/users/{id})
  RequestMethodsRequestCondition : HTTP 메서드 (GET, POST...)
  ParamsRequestCondition         : 쿼리 파라미터 (param=value, !param)
  HeadersRequestCondition        : 요청 헤더 (Header=value, !Header)
  ConsumesRequestCondition       : Content-Type (application/json)
  ProducesRequestCondition       : Accept 헤더 (application/json)
  RequestConditionHolder         : 커스텀 조건

getMatchingCondition() 평가 순서
  메서드 → 파라미터 → 헤더 → consumes → produces → URL 패턴 → 커스텀
  하나라도 null 반환 → 즉시 탈락 (단락 평가)

조건 불일치 시 HTTP 상태 코드
  HTTP 메서드 → 405 Method Not Allowed + Allow 헤더
  consumes    → 415 Unsupported Media Type
  produces    → 406 Not Acceptable
  params/headers/URL → 404 Not Found (다른 핸들러 없을 때)

구체성 비교 우선순위
  정적 URL > URI 변수 > 단일 와일드카드 > 이중 와일드카드
  조건 많을수록 구체적 (params 2개 > params 1개)
```

---

## 🤔 생각해볼 문제

**Q1.** `@GetMapping(value = "/users", params = "page")` 와 `@GetMapping(value = "/users")` 두 핸들러가 등록되어 있을 때 `GET /users?page=2` 요청이 오면 어느 핸들러가 선택되는가? `compareTo()`의 params 비교 기준으로 설명하라.

**Q2.** `consumes` 조건과 `headers = "Content-Type=application/json"` 조건은 둘 다 Content-Type 헤더를 검사하는 것처럼 보입니다. 기술적으로 어떤 차이가 있는가?

**Q3.** Spring의 `RequestMappingInfo`는 `RequestCondition<RequestMappingInfo>` 인터페이스를 구현합니다. `combine()` 메서드는 두 Info를 합치고 `compareTo()`는 구체성을 비교합니다. 이 설계로 가능한 것과 불가능한 것은 무엇인가? 예를 들어 `@RequestMapping` 없이 완전히 커스텀한 라우팅 조건을 만들 수 있는가?

> 💡 **해설**
>
> **Q1.** `params = "page"` 조건이 있는 핸들러가 선택됩니다. `compareTo()`에서 params 비교 시 조건을 많이 가진 쪽이 더 구체적으로 평가됩니다. `params = "page"` 조건 1개 vs 빈 params 조건은 전자가 구체적입니다. `GET /users?page=2` 는 두 핸들러 모두 매칭되지만 구체성 비교에서 `params = "page"` 조건이 있는 핸들러가 이깁니다. 이 동작은 특정 파라미터 존재 시 전용 핸들러를, 그 외에는 기본 핸들러를 제공하는 패턴으로 활용됩니다.
>
> **Q2.** `consumes` 조건은 `ConsumesRequestCondition`이 처리하며, Content-Type을 `MediaType` 파싱 + `isCompatibleWith()` 비교를 수행합니다. 예: `consumes = "application/*"` → `application/json`과 호환됩니다. 반면 `headers = "Content-Type=application/json"` 는 `HeadersRequestCondition`이 단순 문자열 동등 비교(`request.getHeader("Content-Type").equals("application/json")`)를 수행합니다. 따라서 와일드카드(`application/*`)나 품질 계수(`q=0.9`)가 있는 미디어 타입 협상이 `headers` 조건에서는 동작하지 않습니다. 또한 `consumes`는 CORS Preflight 요청에서 자동으로 skip되지만 `headers`는 그렇지 않습니다.
>
> **Q3.** `RequestCondition` 인터페이스(`getMatchingCondition`, `compareTo`, `combine`)를 구현해 커스텀 조건을 만들 수 있습니다. `RequestMappingHandlerMapping.getCustomMethodCondition()`을 오버라이드하면 각 메서드에 커스텀 조건을 추가할 수 있습니다. 예: 특정 HTTP 헤더 값이나 요청 IP 기반 조건. 단, `@RequestMapping` 자체를 완전히 대체하는 것은 `HandlerMapping` 인터페이스를 새로 구현해야 합니다 — `RequestMappingHandlerMapping`은 `@RequestMapping` 어노테이션 스캔을 기반으로 동작하므로, 완전히 다른 라우팅 규칙(예: gRPC 메서드명 기반)은 별도 `HandlerMapping` 구현이 필요합니다.

---

<div align="center">

**[⬅️ 이전: @RequestMapping 처리 과정](./01-request-mapping-handler-mapping.md)** | **[홈으로 🏠](../README.md)** | **[다음: Path Pattern 매칭 전략 ➡️](./03-path-pattern-matching.md)**

</div>
