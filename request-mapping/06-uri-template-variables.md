# URI Template Variables 추출 — {id}가 42가 되기까지

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `PathPattern.matchAndExtract()`는 URI 변수를 어떻게 추출해 어디에 저장하는가?
- `@PathVariable`이 붙은 파라미터가 실제로 값을 받는 내부 처리 경로는?
- `{id:[0-9]+}` 정규식 제약이 있는 템플릿에서 변수 추출은 어떻게 다른가?
- `{*path}` 캡처 나머지 문법은 일반 `{id}`와 어떻게 다른가?
- 타입 변환 — `String "42"` 에서 `Long 42L` 로 변환되는 과정은?
- URL 인코딩된 값(`%20`, `%2F`)은 변수 추출 시 디코딩되는가?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: URL 경로의 일부가 데이터를 담고 있다

```
REST API에서 URL 경로 = 리소스 식별자:
  GET /users/42          → userId = 42
  GET /orders/2024/03    → year = 2024, month = 03
  GET /files/docs/readme.txt → path = docs/readme.txt

이 값들을 Controller 메서드에서 쓰려면:
  // Bad: 직접 파싱
  @GetMapping("/users/{id}")
  public User getUser(HttpServletRequest request) {
      String uri = request.getRequestURI();
      String id = uri.substring(uri.lastIndexOf('/') + 1);
      Long userId = Long.parseLong(id);
      return userService.findById(userId);
  }

  // Good: @PathVariable
  @GetMapping("/users/{id}")
  public User getUser(@PathVariable Long id) {  // 자동 추출 + 타입 변환
      return userService.findById(id);
  }

해결:
  HandlerMapping이 매칭 시 URI 변수를 request 속성에 저장
  ArgumentResolver가 @PathVariable 처리 시 꺼내서 타입 변환
```

---

## 😱 흔한 오해 또는 실수

### Before: @PathVariable 이름은 항상 변수명과 일치해야 한다

```java
// @GetMapping("/users/{userId}")
// public User getUser(@PathVariable Long id) — 이름 불일치!

// ✅ 실제:
// 컴파일 시 -parameters 옵션 있음 (Spring Boot 기본):
//   파라미터 이름 "id" 보존 → "userId"와 불일치 → MissingPathVariableException

// @PathVariable("userId")로 명시하면 항상 안전:
@GetMapping("/users/{userId}")
public User getUser(@PathVariable("userId") Long id) {
    return userService.findById(id);
}
```

### Before: 타입 변환 실패 시 500 에러가 난다

```
// curl http://localhost:8080/users/abc  (Long 변환 불가)

❌ 잘못된 이해: "500 Internal Server Error"

✅ 실제: MethodArgumentTypeMismatchException
  → DefaultHandlerExceptionResolver가 처리
  → 400 Bad Request (잘못된 형식의 요청)
  Spring Boot 기본: 400 + 에러 JSON
```

---

## ✨ 올바른 이해와 사용

### After: URI 변수 추출 전체 흐름

```
① HandlerMapping 매칭 단계
   PathPattern.matchAndExtract(pathContainer)
   → PathMatchInfo { uriVariables: {id:"42"}, matrixVariables: {} }
   → request.setAttribute(URI_TEMPLATE_VARIABLES_ATTRIBUTE, {"id":"42"})

② ArgumentResolver 단계
   PathVariableMethodArgumentResolver.resolveName()
   → request.getAttribute(URI_TEMPLATE_VARIABLES_ATTRIBUTE)
   → {"id": "42"} 에서 "id" → "42" (String)

③ 타입 변환 단계
   WebDataBinder.convertIfNecessary("42", Long.class)
   → ConversionService → StringToLongConverter → 42L
   → Controller 메서드 파라미터에 Long 42L 주입
```

---

## 🔬 내부 동작 원리

### 1. PathPattern.matchAndExtract() — 매칭과 동시에 변수 추출

```java
// PathPattern.java
@Nullable
public PathMatchInfo matchAndExtract(PathContainer pathContainer) {
    MatchingContext matchingContext = new MatchingContext(pathContainer, true);

    if (this.head.matches(0, matchingContext)) {
        Map<String, String> uriVariables = matchingContext.getExtractedVariables();
        Map<String, MultiValueMap<String, String>> matrixVariables =
            matchingContext.getExtractedMatrixVariables();
        return new PathMatchInfo(uriVariables, matrixVariables);
    }
    return null;
}

// CaptureVariablePathElement — {id} 노드의 matches()
@Override
public boolean matches(int pathIndex, MatchingContext matchingContext) {
    if (pathIndex >= matchingContext.pathLength) {
        return false;
    }
    PathElement currentElement = matchingContext.pathElements.get(pathIndex);
    String value = currentElement.valueToMatch();

    // 정규식 조건 검사 ({id:[0-9]+} 인 경우)
    if (this.constraintPattern != null) {
        if (!this.constraintPattern.matcher(value).fullMatch()) {
            return false;  // 정규식 불일치 → 매칭 실패 → 다른 패턴으로 fallback
        }
    }

    // 변수값 저장: "id" → "42"
    matchingContext.set(this.variableName, value,
        currentElement instanceof PathSegment ps ? ps.parameters() : null);

    return this.next == null || this.next.matches(pathIndex + 1, matchingContext);
}
```

### 2. handleMatch() — request 속성에 저장

```java
// RequestMappingHandlerMapping.handleMatch()
private void extractMatchDetails(PathPatternsRequestCondition condition,
        String lookupPath, HttpServletRequest request) {

    PathPattern bestPattern = condition.getPatterns().iterator().next();
    request.setAttribute(BEST_MATCHING_PATTERN_ATTRIBUTE, bestPattern.getPatternString());

    PathPattern.PathMatchInfo result = bestPattern.matchAndExtract(
        ServletRequestPathUtils.getParsedRequestPath(request).pathWithinApplication());

    if (result != null) {
        // ① URI 변수 저장
        request.setAttribute(
            HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE,
            result.getUriVariables());
        // → {"id": "42"}

        // ② 매트릭스 변수 저장 (활성화된 경우)
        request.setAttribute(
            HandlerMapping.MATRIX_VARIABLES_ATTRIBUTE,
            result.getMatrixVariables());
    }
}
```

### 3. PathVariableMethodArgumentResolver.resolveArgument()

```java
// 이름으로 URI 변수 맵에서 값 꺼내기
@Override
@Nullable
protected Object resolveName(String name, MethodParameter parameter,
        NativeWebRequest request) throws Exception {

    HttpServletRequest servletRequest = request.getNativeRequest(HttpServletRequest.class);

    @SuppressWarnings("unchecked")
    Map<String, String> uriTemplateVars = (Map<String, String>) servletRequest
        .getAttribute(HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE);

    return (uriTemplateVars != null ? uriTemplateVars.get(name) : null);
    // "id" → "42" (String)
}

// 타입 변환은 부모 AbstractNamedValueMethodArgumentResolver.resolveArgument()에서:
// "42" (String) → binder.convertIfNecessary("42", Long.class)
//              → ConversionService → 42L
// 실패 시 MethodArgumentTypeMismatchException → 400 Bad Request
```

### 4. {*path} — 나머지 경로 전체 캡처

```java
// PathPatternParser 전용 — CaptureTheRestPathElement 노드
// 현재 위치에서 끝까지 모든 세그먼트를 단일 변수로 캡처

@GetMapping("/files/{*filePath}")
public String getFile(@PathVariable String filePath) {
    // GET /files/docs/2024/report.pdf
    // → filePath = "/docs/2024/report.pdf"  (앞의 / 포함!)
    return "파일 경로: " + filePath;
}

// 일반 {id}와의 차이:
//   {id}    → 단일 세그먼트, 앞 / 미포함  → "42"
//   {*path} → 다중 세그먼트, 앞 / 포함   → "/docs/report.pdf"

// AntPathMatcher의 "/**" 캡처와의 차이:
//   AntPathMatcher: extractPathWithinPattern() 별도 호출 필요
//   PathPatternParser {*path}: 자동으로 PathVariable에 담김
```

### 5. URL 인코딩 처리

```java
// URL: /users/hello%20world
// PathContainer가 세그먼트 단위로 디코딩 처리
// valueToMatch() → 디코딩된 "hello world"
// → @PathVariable String name → "hello world"

// %2F (인코딩된 슬래시) 처리:
// PathContainer는 %2F를 세그먼트 구분자로 처리하지 않음 (기본 설정)
// → {id} 변수가 "a%2Fb" 형태 문자열 캡처 가능
// (단, 서블릿 컨테이너 설정에 따라 다를 수 있음)
```

---

## 💻 실험으로 확인하기

### 실험 1: URI 변수 request 속성 직접 확인

```java
@GetMapping("/users/{id}/orders/{orderId}")
public Map<String, Object> debug(@PathVariable Long id,
                                  @PathVariable Long orderId,
                                  HttpServletRequest request) {
    @SuppressWarnings("unchecked")
    Map<String, String> uriVars = (Map<String, String>)
        request.getAttribute(HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE);

    return Map.of(
        "id", id,
        "orderId", orderId,
        "rawUriVars", uriVars   // {"id":"42", "orderId":"7"} — 모두 String
    );
}
```

```bash
curl http://localhost:8080/users/42/orders/7
# {
#   "id": 42,
#   "orderId": 7,
#   "rawUriVars": {"id": "42", "orderId": "7"}
# }
```

### 실험 2: 타입 변환 실패 + 정규식 fallback

```java
@GetMapping("/items/{id:[0-9]+}")  // 숫자만
public String byNumber(@PathVariable Long id) {
    return "number: " + id;
}

@GetMapping("/items/{name}")       // 모든 문자열
public String byName(@PathVariable String name) {
    return "name: " + name;
}
```

```bash
curl http://localhost:8080/items/42   # → "number: 42"  (정규식 일치)
curl http://localhost:8080/items/abc  # → "name: abc"   (정규식 불일치 → fallback)

# Long 파라미터에 문자열 — 정규식 없는 경우
# @GetMapping("/users/{id}")  → Long id
curl http://localhost:8080/users/abc
# HTTP/1.1 400 Bad Request
```

### 실험 3: {*path} 나머지 캡처 확인

```java
@GetMapping("/resources/{*filePath}")
public String resource(@PathVariable String filePath) {
    return "filePath = '" + filePath + "'";
}
```

```bash
curl http://localhost:8080/resources/css/style.css
# → "filePath = '/css/style.css'"  (앞 / 포함)

curl http://localhost:8080/resources/
# → "filePath = '/'"

curl http://localhost:8080/resources/a/b/c/d.txt
# → "filePath = '/a/b/c/d.txt'"
```

### 실험 4: Map으로 모든 URI 변수 수집

```java
@GetMapping("/orders/{year}/{month}/{id}")
public Map<String, String> allVars(
        @PathVariable Map<String, String> pathVars) {
    // PathVariableMapMethodArgumentResolver가 처리
    return pathVars;  // {"year":"2024", "month":"03", "id":"99"}
}
```

---

## 🌐 HTTP 레벨 분석

```
URI 변수 추출 전체 흐름:

요청: GET /users/42 HTTP/1.1

① Tomcat: getRequestURI() = "/users/42"
② DispatcherServlet → getHandler()
③ PathPattern("/users/{id}").matchAndExtract("/users/42")
   → {id: "42"} 추출
   → request.setAttribute(URI_TEMPLATE_VARIABLES_ATTRIBUTE, {"id":"42"})
④ RequestMappingHandlerAdapter:
   → PathVariableMethodArgumentResolver:
      → getAttribute → {"id":"42"} → get("id") → "42"
      → convertIfNecessary("42", Long.class) → 42L
   → UserController.getUser(42L) 실행
⑤ HTTP/1.1 200 OK, {"id":42,"name":"홍길동"}

타입 변환 실패:
   → MethodArgumentTypeMismatchException
   → DefaultHandlerExceptionResolver → 400 Bad Request
   HTTP/1.1 400 Bad Request
   {"status":400,"error":"Bad Request","message":"..."}
```

---

## 🤔 트레이드오프

```
@PathVariable 자동 이름 매핑 (-parameters 옵션 기반):
  장점  어노테이션 값 생략 가능 → 코드 간결
  단점  컴파일 옵션 의존 → IDE/빌드 환경 차이로 오동작 가능
        → @PathVariable("userId") 명시가 명확하고 안전

{변수명:정규식} 조건:
  장점  형식 오류 조기 감지, 자동 다른 패턴으로 fallback
  단점  정규식 복잡도 증가, 패턴 가독성 저하
        Controller 내부 @Valid 검증이 더 명확한 경우 많음

{*path} 나머지 캡처:
  장점  파일 경로, 동적 리소스 경로 처리에 강력
        AntPathMatcher/** 보다 명시적으로 변수에 담김
  단점  항상 "/" 로 시작 → 후처리에서 strip 필요
        패턴 끝에만 위치 가능 (PathPatternParser 제약)
```

---

## 📌 핵심 정리

```
URI 변수 추출 두 단계
  ① HandlerMapping 단계:
     PathPattern.matchAndExtract() → {id:"42"}
     request.setAttribute(URI_TEMPLATE_VARIABLES_ATTRIBUTE, map)

  ② ArgumentResolver 단계:
     PathVariableMethodArgumentResolver.resolveName()
     → getAttribute → "42" → ConversionService → Long 42L

패턴 문법
  {id}           : 단일 세그먼트 캡처
  {id:[0-9]+}    : 정규식 제약 캡처, 불일치 시 다음 패턴 fallback
  {*path}        : 나머지 경로 전체 캡처, 앞 "/" 포함 (PathPatternParser 전용)

타입 변환 실패
  MethodArgumentTypeMismatchException → 400 Bad Request

URL 인코딩
  PathContainer가 세그먼트 디코딩 → @PathVariable에 디코딩된 값
  %2F는 경로 구분자로 처리 안 됨 (기본 설정)

@PathVariable Map<String, String>
  PathVariableMapMethodArgumentResolver 처리
  → 현재 요청의 모든 URI 변수를 Map으로 반환
```

---

## 🤔 생각해볼 문제

**Q1.** `@PathVariable Map<String, String> allVars` 처럼 Map 타입으로 선언하면 어떤 동작을 하는가? 어떤 상황에서 유용한가?

**Q2.** `@GetMapping("/users/{id}")` 핸들러에서 `@PathVariable` 없이 `HttpServletRequest`로 URI 변수를 꺼낼 수 있는가? 가능하다면 어떤 속성 키를 사용하는가?

**Q3.** `Optional<Long>` 타입으로 `@PathVariable Optional<Long> id`를 선언하면 어떻게 동작하는가? URI 변수가 없는 상황이 발생할 수 있는가?

> 💡 **해설**
>
> **Q1.** `@PathVariable Map<String, String> allVars`는 해당 요청의 모든 URI 변수를 Map 형태로 받습니다. `/users/{userId}/orders/{orderId}` 패턴에서 `{userId: "42", orderId: "7"}` 전체가 Map으로 주입됩니다. `PathVariableMapMethodArgumentResolver`가 `Map` 타입을 처리하며, `URI_TEMPLATE_VARIABLES_ATTRIBUTE`에서 전체 맵을 반환합니다. URL 구조가 동적으로 변할 수 있는 프록시/게이트웨이 구현이나 URI 변수 전체를 다른 서비스에 그대로 전달해야 하는 상황에서 유용합니다.
>
> **Q2.** 가능합니다. `request.getAttribute(HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE)`로 꺼낼 수 있습니다. 상수 값은 `"org.springframework.web.servlet.HandlerMapping.uriTemplateVariables"`이며 반환 타입은 `Map<String, String>`입니다. 단, 이는 Spring MVC 내부 구현에 의존하는 방식이므로 `@PathVariable` 사용이 더 권장됩니다.
>
> **Q3.** `@PathVariable Optional<Long> id`는 URI 변수가 존재하면 `Optional.of(42L)`, 없으면 `Optional.empty()`를 주입합니다. 단, `@GetMapping("/users/{id}")` 패턴이라면 `{id}`가 반드시 있어야 라우팅되므로 실제로 변수가 없는 상황은 이 핸들러로 라우팅 자체가 되지 않습니다. `Optional`이 실용적인 경우는 `@RequestMapping({"/users", "/users/{id}"})` 처럼 두 패턴을 하나의 핸들러에 등록한 경우입니다. `/users` 요청 시 `id` 변수가 없어 `Optional.empty()`가 바인딩됩니다.

---

<div align="center">

**[⬅️ 이전: Content Negotiation](./05-content-negotiation.md)** | **[홈으로 🏠](../README.md)** | **[다음: Matrix Variables와 사용 시점 ➡️](./07-matrix-variables.md)**

</div>
