# Path Pattern 매칭 전략 — AntPathMatcher에서 PathPatternParser로

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `AntPathMatcher`와 `PathPatternParser`는 내부 구조가 어떻게 다른가?
- Spring 5.3 이후 `PathPatternParser`가 기본값이 된 이유는 무엇인가?
- `/users/{id}`, `/users/*`, `/users/**`, `/users/{id:[0-9]+}` 각각의 매칭 규칙은?
- 두 패턴이 같은 요청을 매칭할 때 어느 것이 더 구체적으로 평가되는가?
- `PathPattern.matchAndExtract()`는 URI 템플릿 변수를 어떻게 추출하는가?
- Spring Boot에서 기본 매처를 바꾸는 방법과 주의사항은?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: 요청 URL과 등록된 패턴의 일치 여부를 어떻게 판단할 것인가

```
핸들러 매핑 시 "패턴 vs URL" 비교가 반드시 필요:
  등록된 패턴: "/users/{id}"
  요청 URL:    "/users/42"

  질문:
    이 URL이 패턴에 매칭되는가?
    매칭된다면 {id} = "42" 라는 정보를 어떻게 추출하는가?
    "/users/{id}"와 "/users/admin" 중 "/users/admin" 요청에 어느 것이 더 구체적인가?

AntPathMatcher (레거시):
  런타임에 문자열을 파싱해 매칭
  → 매 요청마다 패턴 문자열 파싱 반복
  → String 조작 기반 → 성능 한계

PathPatternParser (Spring 5.3+ 기본):
  시작 시 패턴을 파싱해 PathPattern 객체(트리 구조)로 컴파일
  → 요청 시 컴파일된 트리로 매칭 (문자열 파싱 없음)
  → 성능 대폭 향상, URL 디코딩 처리도 더 정확
```

---

## 😱 흔한 오해 또는 실수

### Before: `*`와 `**`의 동작이 같다

```java
// ❌ 잘못된 이해
// "* 와 ** 는 둘 다 모든 것을 매칭한다"

// ✅ 실제:
// *  → 단일 경로 세그먼트 내에서 임의의 문자열
//      "/users/*" → "/users/john" 매칭 ✅
//                → "/users/john/orders" 매칭 ❌ (세그먼트 경계 넘음)

// ** → 0개 이상의 경로 세그먼트 매칭
//      "/users/**" → "/users/john" 매칭 ✅
//                 → "/users/john/orders" 매칭 ✅
//                 → "/users" 매칭 ✅ (0개 세그먼트)

// PathPatternParser에서 ** 의 추가 제약:
// PathPatternParser는 ** 가 패턴의 끝에만 올 수 있음
// "/users/**/orders" → PathPatternParser에서 허용되지 않음
// → AntPathMatcher는 중간 ** 허용, PathPatternParser는 끝에만 허용
```

### Before: AntPathMatcher는 더 이상 사용되지 않는다

```
❌ 잘못된 이해:
  "Spring Boot 3.x에서는 AntPathMatcher를 쓰면 안 된다"

✅ 실제:
  RequestMappingHandlerMapping의 기본값은 PathPatternParser로 변경됨
  그러나 AntPathMatcher는 여전히 사용됨:
    → ResourceHandler (정적 리소스 경로 매칭)
    → Spring Security URL 패턴 매칭 (별도 설정)
    → @PathVariable 아닌 곳에서의 패턴 매칭
    → PathMatcher 직접 주입해서 사용하는 경우
  두 방식은 공존하며, 목적에 따라 선택
```

---

## ✨ 올바른 이해와 사용

### After: 패턴 종류별 매칭 규칙

```
패턴 문법 완전 정리:

  {변수명}
    단일 경로 세그먼트 전체 캡처
    "/users/{id}"  →  "/users/42"    → id=42 ✅
                   →  "/users/a/b"   ❌ (세그먼트 2개)

  {변수명:정규식}
    정규식 조건이 있는 세그먼트 캡처
    "/users/{id:[0-9]+}"  →  "/users/42"    → id=42 ✅
                          →  "/users/abc"   ❌ (숫자 아님)

  *
    단일 세그먼트 임의 문자열 (변수 캡처 없음)
    "/files/*.txt"  →  "/files/notes.txt"   ✅
                    →  "/files/a/b.txt"      ❌

  **
    0개 이상의 경로 세그먼트 (패턴 끝에만 사용 가능 — PathPatternParser)
    "/resources/**"  →  "/resources/"        ✅
                     →  "/resources/img/logo.png"  ✅

  {*변수명}
    0개 이상의 나머지 경로 전체 캡처 (PathPatternParser 전용)
    "/files/{*path}"  →  "/files/a/b/c.txt"  → path=/a/b/c.txt ✅
                      →  "/files/"             → path=/ ✅

구체성 순서 (높은 것이 우선):
  1. 정적 URL        "/users/admin"
  2. URI 변수        "/users/{id}"
  3. 단일 와일드카드  "/users/*"
  4. 이중 와일드카드  "/users/**"
```

---

## 🔬 내부 동작 원리

### 1. PathPatternParser — 컴파일 방식

```java
// PathPatternParser.java
public PathPattern parse(String pathPattern) {
    // 시작 시 한 번 파싱 → PathPattern 객체(내부적으로 트리 구조) 생성
    PathPatternParser pp = new PathPatternParser();
    return pp.parse(pathPattern);
    // "/users/{id}" →
    //   SeparatorPathElement("/")
    //   LiteralPathElement("users")
    //   SeparatorPathElement("/")
    //   CaptureVariablePathElement("id")  ← {id} 부분
}

// PathPattern.matches(PathContainer path)
// 컴파일된 트리를 순회하며 PathContainer(URL 구조체)와 비교
// 문자열 재파싱 없음 → 빠름
```

### 2. PathPattern.matchAndExtract() — 변수 추출

```java
// PathPatternRouteMatcher.java 내부 흐름
public boolean match(String pattern, String lookupPath) {
    PathPattern pp = this.pathPatternCache.computeIfAbsent(pattern,
        p -> this.pathPatternParser.parse(p));  // 캐시에서 가져옴

    return pp.matches(PathContainer.parsePath(lookupPath));
}

// 매핑 성공 후 변수 추출:
// AbstractHandlerMethodMapping.handleMatch()
@Override
protected void handleMatch(RequestMappingInfo info, String lookupPath,
        HttpServletRequest request) {
    super.handleMatch(info, lookupPath, request);

    PathPattern bestPattern = ...;
    PathContainer path = ServletRequestPathUtils.getParsedRequestPath(request).pathWithinApplication();

    // matchAndExtract(): 매칭 + 변수 값 추출을 동시에 수행
    PathPattern.PathMatchInfo matchInfo = bestPattern.matchAndExtract(path);

    if (matchInfo != null) {
        // URI 변수를 request 속성에 저장
        // → PathVariableMethodArgumentResolver가 나중에 꺼내 씀
        Map<String, String> uriVariables = matchInfo.getUriVariables();
        request.setAttribute(
            HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE,
            uriVariables);  // → {id: "42"}

        // 매트릭스 변수도 저장
        Map<String, MultiValueMap<String, String>> matrixVars =
            matchInfo.getMatrixVariables();
        request.setAttribute(
            HandlerMapping.MATRIX_VARIABLES_ATTRIBUTE,
            matrixVars);
    }
}
```

### 3. AntPathMatcher vs PathPatternParser 비교

```java
// AntPathMatcher (레거시) 방식
AntPathMatcher matcher = new AntPathMatcher();
// 매 호출마다 패턴 문자열 파싱
boolean match = matcher.match("/users/{id}", "/users/42");
Map<String, String> vars = matcher.extractUriTemplateVariables("/users/{id}", "/users/42");
// → {id: "42"}

// PathPatternParser (현대) 방식
PathPatternParser parser = new PathPatternParser();
PathPattern pattern = parser.parse("/users/{id}");  // 한 번만 파싱 (컴파일)

// 요청마다: 컴파일된 트리로 빠른 매칭
PathContainer path = PathContainer.parsePath("/users/42");
PathPattern.PathMatchInfo info = pattern.matchAndExtract(path);
// → uriVariables: {id: "42"}
```

### 4. PathPatternParser 설정 및 AntPathMatcher로 롤백

```java
// Spring Boot에서 PathPatternParser 설정 (기본값)
// application.yml:
// spring.mvc.pathmatch.use-suffix-pattern=false (기본)
// spring.mvc.pathmatch.use-registered-suffix-pattern=false (기본)

// AntPathMatcher로 롤백하는 경우:
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void configurePathMatch(PathMatchConfigurer configurer) {
        // PathPatternParser 비활성화, AntPathMatcher 사용
        configurer.setPatternParser(null);
        // 또는 suffix pattern 활성화가 필요한 레거시 코드 지원 시
    }
}

// Spring Boot 3.x에서 AntPathMatcher로 변경 시 주의:
// spring.mvc.pathmatch.matching-strategy=ant_path_matcher
// → 성능 저하, ** 중간 사용 가능해지지만 PathPatternParser 기능 미지원
```

### 5. 구체성 비교 — PathPattern.compareTo()

```java
// PathPattern.SPECIFICITY_COMPARATOR
// 두 PathPattern의 구체성 비교
Comparator<PathPattern> comparator = PathPattern.SPECIFICITY_COMPARATOR;

// 비교 기준 (우선순위 순):
// 1. 캡처 변수 수 적을수록 구체적
// 2. 와일드카드 수 적을수록 구체적
// 3. 캡처 변수 수
// 4. 길이 길수록 구체적

// 예시:
// "/users/admin"  (변수 0, 와일드카드 0, 길이 12)
// "/users/{id}"   (변수 1, 와일드카드 0, 길이 10)
// "/users/*"      (변수 0, 와일드카드 1, 길이 8)
// "/users/**"     (변수 0, 와일드카드 2, 길이 9)

// 순서: "/users/admin" > "/users/{id}" > "/users/*" > "/users/**"
```

---

## 💻 실험으로 확인하기

### 실험 1: 패턴별 매칭 동작 직접 확인

```java
@RestController
public class PatternTestController {

    @GetMapping("/test/admin")       // 정적
    public String s1() { return "정적: /test/admin"; }

    @GetMapping("/test/{id:[0-9]+}") // 정규식 변수
    public String s2(@PathVariable String id) { return "정규식 변수: " + id; }

    @GetMapping("/test/{name}")      // 일반 변수
    public String s3(@PathVariable String name) { return "변수: " + name; }

    @GetMapping("/test/*")           // 단일 와일드카드
    public String s4() { return "단일 와일드카드"; }

    @GetMapping("/test/**")          // 이중 와일드카드
    public String s5() { return "이중 와일드카드"; }
}
```

```bash
curl http://localhost:8080/test/admin    # → "정적: /test/admin"
curl http://localhost:8080/test/42       # → "정규식 변수: 42"
curl http://localhost:8080/test/john     # → "변수: john" (정규식 불일치)
curl http://localhost:8080/test/a/b/c   # → "이중 와일드카드"
```

### 실험 2: PathPatternParser 직접 사용

```java
@GetMapping("/pattern-test")
public Map<String, Object> patternTest() {
    PathPatternParser parser = new PathPatternParser();

    // 패턴 컴파일
    PathPattern p1 = parser.parse("/users/{id}");
    PathPattern p2 = parser.parse("/users/**");

    String testUrl = "/users/42/orders";
    PathContainer path = PathContainer.parsePath(testUrl);

    Map<String, Object> result = new LinkedHashMap<>();
    result.put("/users/{id} matches " + testUrl, p1.matches(path));
    result.put("/users/** matches " + testUrl, p2.matches(path));

    PathPattern.PathMatchInfo info = p1.matchAndExtract(PathContainer.parsePath("/users/42"));
    result.put("variables from /users/42", info != null ? info.getUriVariables() : null);

    return result;
}
```

```json
// 출력:
{
  "/users/{id} matches /users/42/orders": false,
  "/users/** matches /users/42/orders": true,
  "variables from /users/42": {"id": "42"}
}
```

### 실험 3: ** 중간 사용 제약 확인

```java
// PathPatternParser에서 ** 중간 사용 시도
@GetMapping("/a/**/b")  // PathPatternParser에서 예외!
public String middle() { return "middle"; }

// org.springframework.web.util.pattern.PatternParseException:
//   '**' is only allowed at the end of a pattern,
//   e.g. '/foo/**'
//
// AntPathMatcher에서는 /a/**/b 허용됨
```

---

## 🌐 HTTP 레벨 분석

```bash
# 패턴과 URL 매칭 실패 케이스

# {id:[0-9]+} 정규식 불일치 → 다른 패턴으로 fallback
GET /users/abc
→ /users/{id:[0-9]+} : 정규식 불일치
→ /users/{name}      : 매칭 ✅ (정규식 없는 패턴)
→ 200 OK

# 모든 패턴 불일치
GET /users/a/b/c (** 없을 때)
→ /users/{id} : 세그먼트 3개, 불일치
→ /users/*    : 세그먼트 3개, 불일치
→ 핸들러 없음 → 404 Not Found

# 정적 리소스 경로
GET /static/js/app.js
→ DispatcherServlet의 ResourceHttpRequestHandler가 /static/** 패턴으로 처리
→ 200 OK (파일 있으면)
```

---

## 🤔 트레이드오프

```
PathPatternParser:
  장점  컴파일된 트리 구조 → 반복 파싱 없음 → 성능 향상
        URL 디코딩과 정규화가 PathContainer 레벨에서 통일 처리
        {*path} 같은 표현 지원
  단점  ** 중간 사용 불가 (레거시 패턴 마이그레이션 필요)
        suffix pattern (.json, .xml) 미지원

AntPathMatcher:
  장점  ** 중간 사용 가능 ("/**/" 스타일)
        suffix pattern 지원 ("/users.json" → "/users")
        오래된 레거시 코드와 호환
  단점  매 요청마다 문자열 파싱 반복 → 성능 저하
        URL 인코딩 처리 불일치 가능성

{변수명:정규식} 조건:
  장점  URL 레벨에서 타입 안전성 일부 확보
        잘못된 형식의 요청 조기 탈락 → 다른 패턴으로 fallback
  단점  정규식 복잡도 증가 시 가독성 저하
        Controller 내부에서 @Valid로 처리하는 것이 더 명확한 경우 많음
```

---

## 📌 핵심 정리

```
PathPatternParser (Spring 5.3+ 기본, Spring Boot 2.6+ 기본)
  시작 시 패턴을 PathPattern 객체로 컴파일
  요청 시 컴파일된 트리로 매칭 (재파싱 없음)
  ** 는 패턴의 끝에만 허용

패턴 문법
  {id}          → 단일 세그먼트 캡처
  {id:[0-9]+}   → 정규식 제약 캡처
  *             → 단일 세그먼트 와일드카드
  **            → 다중 세그먼트 와일드카드 (끝에만 — PathPatternParser)
  {*path}       → 나머지 경로 전체 캡처 (PathPatternParser 전용)

구체성 순서 (높은 것이 우선 선택)
  정적 URL > {정규식변수} > {변수} > * > **

matchAndExtract() 결과
  URI 변수 → request.setAttribute(URI_TEMPLATE_VARIABLES_ATTRIBUTE, map)
  → PathVariableMethodArgumentResolver가 @PathVariable 처리 시 꺼내 씀

AntPathMatcher로 롤백 시
  configurer.setPatternParser(null)
  또는 spring.mvc.pathmatch.matching-strategy=ant_path_matcher
  → ** 중간 사용 가능, suffix pattern 가능, 성능 저하
```

---

## 🤔 생각해볼 문제

**Q1.** `/api/{version}/users`와 `/api/v1/users` 두 패턴이 등록되어 있을 때 `GET /api/v1/users` 요청이 오면 어느 패턴이 선택되는가? `{version:[a-z][0-9]+}`처럼 정규식을 추가하면 선택이 달라지는가?

**Q2.** `PathPatternParser`에서 URL 경로에 `%2F`(인코딩된 `/`)가 포함된 경우 어떻게 처리되는가? `{id}`가 `a%2Fb`를 캡처할 수 있는가? `AntPathMatcher`와 비교하면 어떻게 다른가?

**Q3.** Spring Security와 Spring MVC가 모두 URL 패턴 매칭을 사용합니다. Spring Security가 `AntPathMatcher`를 쓰고 Spring MVC가 `PathPatternParser`를 쓸 때, 같은 URL 패턴이 서로 다르게 매칭되어 보안 취약점이 생길 수 있는가? 어떤 설정으로 일치시킬 수 있는가?

> 💡 **해설**
>
> **Q1.** 정적 URL `/api/v1/users`가 더 구체적이므로 `/api/v1/users` 패턴이 선택됩니다. `{version:[a-z][0-9]+}` 정규식 패턴을 추가하면 `v1`은 정규식과 매칭되므로 정규식 변수 패턴도 후보가 됩니다. 그러나 정적 URL이 여전히 가장 구체적이므로 선택은 변하지 않습니다. 만약 `/api/v2/users` 요청이 온다면, 정적 `/api/v1/users`는 불일치, `{version:[a-z][0-9]+}` 패턴이 매칭됩니다.
>
> **Q2.** `PathPatternParser`는 `PathContainer`가 URL을 파싱할 때 `/`로 구분된 세그먼트로 분리합니다. `%2F`는 인코딩된 `/`이므로 두 개의 세그먼트로 분리될 수 있습니다. 기본적으로 `PathPatternParser`는 `%2F`를 세그먼트 구분자로 처리하지 않고 단일 세그먼트 내의 문자로 처리하므로 `{id}`가 `a%2Fb`를 캡처합니다. `AntPathMatcher`는 인코딩된 URL을 디코딩하지 않고 처리하므로 동작이 다를 수 있습니다. 이 차이는 보안 취약점(Path Traversal)의 원인이 될 수 있으므로, Spring MVC는 서블릿 컨테이너 레벨에서 `%2F`를 거부하는 설정을 권장합니다.
>
> **Q3.** 네, 실제로 이 불일치로 인한 보안 취약점이 CVE로 보고된 바 있습니다. 예: Spring MVC가 `/admin/**`를 PathPatternParser로 처리하고 Spring Security가 `/admin/**`를 AntPathMatcher로 처리할 때, 특수 인코딩이나 경로 정규화 차이로 MVC는 매칭되지만 Security는 매칭되지 않는 URL이 존재할 수 있습니다. 해결책: Spring Security 5.8+ 에서는 `PathPatternRequestMatcher`를 사용해 Spring MVC와 동일한 PathPatternParser 기반으로 일치시키거나, Spring Boot의 보안 설정에서 `spring.mvc.pathmatch.use-path-pattern-parser=true`와 Security를 함께 구성합니다.

---

<div align="center">

**[⬅️ 이전: RequestMappingInfo 생성과 매칭](./02-request-mapping-info.md)** | **[홈으로 🏠](../README.md)** | **[다음: HTTP Method 매칭과 OPTIONS 처리 ➡️](./04-http-method-matching.md)**

</div>
