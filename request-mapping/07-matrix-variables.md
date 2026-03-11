# Matrix Variables와 사용 시점 — ;으로 전달하는 경로 파라미터

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Matrix Variable이란 무엇이고, 쿼리 파라미터와 무엇이 다른가?
- Spring MVC에서 `@MatrixVariable`을 사용하기 위해 `removeSemicolonContent=false`를 설정해야 하는 이유는?
- `PathPattern.matchAndExtract()`에서 Matrix Variable이 추출되는 내부 과정은?
- `@MatrixVariable`로 복수 값, 기본값, 특정 경로 세그먼트 지정은 어떻게 하는가?
- Matrix Variable의 실용적인 사용 사례와 현대적 대안은?
- Spring Boot에서 `removeSemicolonContent` 기본값은 무엇이고, 왜 그런가?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: 경로 세그먼트에 부가 속성을 전달하는 RFC 표준이 있다

```
RFC 3986 — URI Generic Syntax:
  path-segment = *pchar
  pchar        = unreserved / pct-encoded / sub-delims / ":" / "@"

W3C URI 권고 (Tim Berners-Lee):
  경로 파라미터를 ; 로 구분해 경로 세그먼트에 첨부 가능
  /path;param1=value1;param2=value2

실제 사용 예:
  /users/42;role=admin;lang=ko
  /color/blue;weight=80;type=dark
  /cars/BMW;color=red/seats/2

  이 형식의 장점:
    각 경로 세그먼트에 독립적인 속성 지정 가능
    /users/42;format=compact/orders;status=pending
    → users 세그먼트의 format과 orders 세그먼트의 status를 별도로 지정

  현실적 사용:
    REST API에서 콘텐츠 협상, 리소스 수식어
    레거시 프레임워크(Backbone.js 등)와의 호환성
    Java EE/Jakarta EE URL 관례 (;jsessionid=...)
```

---

## 😱 흔한 오해 또는 실수

### Before: Spring Boot에서 @MatrixVariable이 바로 동작한다

```java
// ❌ 잘못된 기대
// "@MatrixVariable 어노테이션만 붙이면 된다"

@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id,
                    @MatrixVariable String role) {  // 동작하지 않음!
    return userService.findById(id);
}
// GET /users/42;role=admin
// → role = null (값 없음)
// → Spring Boot 기본값: removeSemicolonContent=true
//   → ; 이후 내용을 URL에서 제거 (보안 목적 — jsessionid URL 노출 방지)
//   → /users/42;role=admin → /users/42 로 처리
//   → Matrix Variable 정보 소실

// ✅ 활성화 방법:
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void configurePathMatch(PathMatchConfigurer configurer) {
        configurer.setUseMatrixVariables(true);
        // 내부적으로 UrlPathHelper.removeSemicolonContent = false 설정
    }
}
```

### Before: Matrix Variable은 쿼리 파라미터와 동일하다

```
❌ 잘못된 이해:
  "/users/42;role=admin"의 ;role=admin은 ?role=admin과 같다

✅ 실제 차이:

  쿼리 파라미터 (?):
    URL 전체에 한 번 붙음
    /users/42?role=admin → 전체 요청의 파라미터
    @RequestParam String role

  Matrix Variable (;):
    각 경로 세그먼트에 개별로 붙음
    /users/42;role=admin/orders/7;status=open
    → "42" 세그먼트의 role = admin
    → "7" 세그먼트의 status = open
    → 세그먼트별로 독립적인 속성 표현 가능

  구조적 의미 차이:
    쿼리 파라미터: "이 요청의 부가 정보"
    Matrix Variable: "이 경로 세그먼트의 속성"
```

---

## ✨ 올바른 이해와 사용

### After: @MatrixVariable 사용 패턴 전체

```java
// 설정: removeSemicolonContent = false 활성화
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void configurePathMatch(PathMatchConfigurer configurer) {
        configurer.setUseMatrixVariables(true);
    }
}

// 1. 기본 사용
// GET /users/42;role=admin
@GetMapping("/users/{id}")
public User getUser(
        @PathVariable Long id,
        @MatrixVariable String role) {   // role = "admin"
    return userService.findById(id, role);
}

// 2. 복수 값
// GET /users/42;role=admin,manager
@GetMapping("/users/{id}")
public User getUser(
        @PathVariable Long id,
        @MatrixVariable List<String> role) {  // role = ["admin", "manager"]
    return userService.findById(id, role);
}

// 3. 기본값 설정
// GET /users/42 (role 없음)
@GetMapping("/users/{id}")
public User getUser(
        @PathVariable Long id,
        @MatrixVariable(defaultValue = "user") String role) {  // role = "user"
    return userService.findById(id, role);
}

// 4. 특정 경로 세그먼트 지정
// GET /users/42;role=admin/orders/7;status=open
@GetMapping("/users/{userId}/orders/{orderId}")
public Order getOrder(
        @PathVariable Long userId,
        @PathVariable Long orderId,
        @MatrixVariable(name = "role", pathVar = "userId") String role,
        @MatrixVariable(name = "status", pathVar = "orderId") String status) {
    // role = "admin" (userId 세그먼트에서 추출)
    // status = "open" (orderId 세그먼트에서 추출)
    return orderService.findOrder(userId, orderId, role, status);
}

// 5. Map으로 전체 수신
// GET /users/42;role=admin;lang=ko
@GetMapping("/users/{id}")
public User getUser(
        @PathVariable Long id,
        @MatrixVariable MultiValueMap<String, String> matrixVars) {
    // matrixVars = {role=[admin], lang=[ko]}
    return userService.findById(id);
}
```

---

## 🔬 내부 동작 원리

### 1. removeSemicolonContent 설정 — 활성화/비활성화

```java
// UrlPathHelper.java
public String removeSemicolonContent(String requestUri) {
    if (!this.removeSemicolonContent) {
        // removeSemicolonContent = false → 그대로 통과
        return requestUri;
    }
    // removeSemicolonContent = true (기본값) → ; 이후 제거
    int semicolonIndex = requestUri.indexOf(';');
    if (semicolonIndex == -1) {
        return requestUri;
    }
    // "/users/42;role=admin/orders" → "/users/42/orders"
    StringBuilder sb = new StringBuilder(requestUri);
    while (semicolonIndex != -1) {
        int slashIndex = sb.indexOf("/", semicolonIndex + 1);
        String start = sb.substring(0, semicolonIndex);
        sb = (slashIndex != -1) ? new StringBuilder(start).append(sb.substring(slashIndex))
                                : new StringBuilder(start);
        semicolonIndex = sb.indexOf(";", semicolonIndex);
    }
    return sb.toString();
}

// Spring Boot 기본값:
// UrlPathHelper.removeSemicolonContent = true
// → Matrix Variable 기능 비활성 (;jsessionid 같은 세션 ID 노출 방지)

// configurer.setUseMatrixVariables(true) 호출 시:
// UrlPathHelper.setRemoveSemicolonContent(false) 설정
// → ; 이후 내용 보존 → PathContainer가 Matrix Variable 파싱
```

### 2. PathContainer — 세그먼트와 파라미터 파싱

```java
// PathContainer.parsePath("/users/42;role=admin;lang=ko/orders/7;status=open")
//
// PathContainer 구조:
//   SeparatorElement(/)
//   PathSegment("42", parameters={role=[admin], lang=[ko]})  ← 세그먼트 + 파라미터
//   SeparatorElement(/)
//   PathSegment("orders")
//   SeparatorElement(/)
//   PathSegment("7", parameters={status=[open]})

// PathSegment.parameters():
//   PathContainer.Element의 파라미터 맵
//   MultiValueMap<String, String>
//   → role=[admin], lang=[ko]

// CaptureVariablePathElement.matchAndExtract()에서:
matchingContext.set(
    this.variableName,          // "id"
    pathSegment.valueToMatch(), // "42" (;이후 제거된 순수 값)
    pathSegment.parameters()    // {role=[admin], lang=[ko]}  ← Matrix Variables
);
// → URI_TEMPLATE_VARIABLES_ATTRIBUTE: {id: "42"}
// → MATRIX_VARIABLES_ATTRIBUTE: {id: {role=[admin], lang=[ko]}}
```

### 3. handleMatch() — Matrix Variable request 속성 저장

```java
// AbstractHandlerMethodMapping.handleMatch()
protected void handleMatch(RequestMappingInfo info, String lookupPath,
        HttpServletRequest request) {
    // ...
    PathPattern.PathMatchInfo matchInfo = bestPattern.matchAndExtract(path);
    if (matchInfo != null) {
        // URI 변수 저장
        request.setAttribute(
            HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE,
            matchInfo.getUriVariables());          // {id: "42"}

        // Matrix Variable 저장
        Map<String, MultiValueMap<String, String>> matrixVariables =
            matchInfo.getMatrixVariables();
        request.setAttribute(
            HandlerMapping.MATRIX_VARIABLES_ATTRIBUTE,
            matrixVariables);                      // {id: {role=[admin], lang=[ko]}}
    }
}
```

### 4. MatrixVariableMethodArgumentResolver — Matrix Variable 추출

```java
// MatrixVariableMethodArgumentResolver.java
@Override
protected Object resolveName(String name, MethodParameter parameter,
        NativeWebRequest request) throws Exception {

    HttpServletRequest servletRequest = request.getNativeRequest(HttpServletRequest.class);
    @SuppressWarnings("unchecked")
    Map<String, MultiValueMap<String, String>> matrixVariables =
        (Map<String, MultiValueMap<String, String>>) servletRequest.getAttribute(
            HandlerMapping.MATRIX_VARIABLES_ATTRIBUTE);

    if (CollectionUtils.isEmpty(matrixVariables)) {
        return null;
    }

    MatrixVariable ann = parameter.getParameterAnnotation(MatrixVariable.class);
    String pathVariable = ann.pathVar();  // pathVar 지정 여부

    if (!pathVariable.equals(ValueConstants.DEFAULT_NONE)) {
        // pathVar="userId" → 특정 세그먼트의 Matrix Variable만 추출
        MultiValueMap<String, String> map = matrixVariables.get(pathVariable);
        return (map != null ? map.getOrDefault(name, null) : null);
    }

    // pathVar 미지정 → 전체 세그먼트에서 탐색
    List<String> allValues = new ArrayList<>();
    for (MultiValueMap<String, String> vars : matrixVariables.values()) {
        List<String> values = vars.get(name);
        if (values != null) {
            allValues.addAll(values);
        }
    }
    return (!allValues.isEmpty() ? allValues : null);
}
```

### 5. Spring Boot에서 활성화 확인 방법

```java
// 자동 설정 여부 확인
@Autowired
RequestMappingHandlerMapping handlerMapping;

@GetMapping("/check-matrix")
public Map<String, Object> checkMatrixVars() throws Exception {
    Field helperField = AbstractHandlerMapping.class
        .getDeclaredField("urlPathHelper");
    helperField.setAccessible(true);
    UrlPathHelper helper = (UrlPathHelper) helperField.get(handlerMapping);

    Map<String, Object> result = new LinkedHashMap<>();
    result.put("removeSemicolonContent",
        helper.shouldRemoveSemicolonContent());
    // true → Matrix Variable 비활성
    // false → Matrix Variable 활성
    return result;
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 활성화 전후 비교

```bash
# 활성화 전 (removeSemicolonContent=true, 기본값)
curl "http://localhost:8080/users/42;role=admin"
# DispatcherServlet이 받은 경로: /users/42 (;role=admin 제거됨)
# @MatrixVariable role = null → 기본값 또는 예외

# 활성화 후 (setUseMatrixVariables(true))
curl "http://localhost:8080/users/42;role=admin"
# DispatcherServlet이 받은 경로: /users/42;role=admin (보존됨)
# @MatrixVariable role = "admin"
```

### 실험 2: 다중 세그먼트 Matrix Variable

```java
@GetMapping("/pets/{petId}/owners/{ownerId}")
public Map<String, Object> find(
        @PathVariable String petId,
        @PathVariable String ownerId,
        @MatrixVariable(name = "q", pathVar = "petId") String petQ,
        @MatrixVariable(name = "q", pathVar = "ownerId") String ownerQ) {

    Map<String, Object> result = new LinkedHashMap<>();
    result.put("petId", petId);
    result.put("ownerId", ownerId);
    result.put("petQ", petQ);
    result.put("ownerQ", ownerQ);
    return result;
}
```

```bash
curl "http://localhost:8080/pets/rex;q=fluffy/owners/42;q=john"
```
```json
{
  "petId": "rex",
  "ownerId": "42",
  "petQ": "fluffy",
  "ownerQ": "john"
}
```

### 실험 3: MultiValueMap으로 전체 Matrix Variable 수신

```java
@GetMapping("/filters/{category}")
public Map<String, Object> filter(
        @PathVariable String category,
        @MatrixVariable MultiValueMap<String, String> matrixVars) {
    return Map.of("category", category, "filters", matrixVars);
}
```

```bash
curl "http://localhost:8080/filters/electronics;brand=samsung,apple;price=100-500"
```
```json
{
  "category": "electronics",
  "filters": {
    "brand": ["samsung", "apple"],
    "price": ["100-500"]
  }
}
```

---

## 🌐 HTTP 레벨 분석

```
Matrix Variable 요청 전체 흐름:

클라이언트 요청:
  GET /users/42;role=admin;lang=ko HTTP/1.1
  Host: localhost:8080

서버 처리 (removeSemicolonContent=false):
  1. UrlPathHelper: ;role=admin;lang=ko 제거 안 함 → 경로 그대로 유지
  2. PathContainer.parsePath():
       PathSegment("42", {role=[admin], lang=[ko]})
  3. PathPattern.matchAndExtract():
       URI_TEMPLATE_VARIABLES_ATTRIBUTE: {id: "42"}
       MATRIX_VARIABLES_ATTRIBUTE: {id: {role=[admin], lang=[ko]}}
  4. MatrixVariableMethodArgumentResolver:
       @PathVariable Long id → 42L
       @MatrixVariable String role → "admin"
       @MatrixVariable String lang → "ko"
  5. 핸들러 실행

서버 응답:
  HTTP/1.1 200 OK
  Content-Type: application/json
  {"id":42,"name":"홍길동","role":"admin"}

주의: 브라우저 URL 표시 바가 ; 를 자동 인코딩할 수 있음
  → curl 사용 시 "" 로 URL 감싸기 권장
  → JavaScript에서 fetch() 시 encodeURIComponent 사용 주의
     (세미콜론을 %3B로 인코딩하면 PathContainer가 다르게 처리)
```

---

## 🤔 트레이드오프

```
Matrix Variable vs Query Parameter:
  Matrix Variable:
    장점  각 경로 세그먼트에 독립적인 속성 표현 가능
          RESTful 리소스 계층 표현과 자연스럽게 결합
    단점  Spring Boot 기본값이 비활성 → 별도 설정 필요
          클라이언트 라이브러리 지원 미흡 (; 인코딩 이슈)
          캐시 처리 복잡 (Vary 헤더 필요, 캐시 키에 포함 여부 불분명)
          인지도 낮음 → 팀 전체의 이해 필요

  Query Parameter:
    장점  모든 HTTP 클라이언트가 완벽 지원
          캐싱 표준적 처리 (쿼리 스트링 포함 캐시 키)
          개발자 인지도 높음
    단점  세그먼트별 속성 구분 불가
          URL이 길어질 수 있음

현대적 대안:
  Matrix Variable 대신:
  → Request Header 사용 (X-Role: admin)
  → 경로에 직접 포함 (/users/42/role/admin)
  → Query Parameter (/users/42?role=admin)
  → Request Body (POST/PUT/PATCH 시)

실용적 결론:
  Matrix Variable은 레거시 시스템 호환이나 특수 요구사항이 없으면
  현대 REST API에서 거의 사용하지 않음
  → 주로 레거시 Java EE 앱 마이그레이션 시나리오
```

---

## 📌 핵심 정리

```
Matrix Variable 기본 개념
  ; 구분자로 경로 세그먼트에 부가 속성 전달
  /users/42;role=admin;lang=ko
  각 세그먼트에 독립적으로 적용 가능

활성화 방법 (Spring Boot 기본 비활성)
  WebMvcConfigurer.configurePathMatch()
    → configurer.setUseMatrixVariables(true)
    → 내부적으로 UrlPathHelper.removeSemicolonContent = false

추출 과정
  PathContainer가 세그먼트 파싱 시 ; 이후를 parameters로 저장
  PathPattern.matchAndExtract() → PathMatchInfo.matrixVariables
  → request.setAttribute(MATRIX_VARIABLES_ATTRIBUTE, map)
  → MatrixVariableMethodArgumentResolver가 해당 속성에서 값 추출

@MatrixVariable 주요 속성
  name         : Matrix Variable 이름 (기본: 파라미터 이름)
  pathVar      : 특정 경로 변수 세그먼트 지정 (다중 세그먼트 구분)
  required     : 필수 여부 (기본: true)
  defaultValue : 기본값

removeSemicolonContent=true (기본) 목적
  보안: ;jsessionid=xxx URL 노출 방지
  Spring Security의 세션 고정 방지 토큰 처리와 연계
```

---

## 🤔 생각해볼 문제

**Q1.** `removeSemicolonContent=true`가 Spring Boot 기본값인 주요 보안 이유는 무엇인가? OWASP 관점에서 URL의 `;jsessionid=...` 노출이 왜 위험한가?

**Q2.** Matrix Variable을 사용하는 URL `/users/42;role=admin`을 `UriComponentsBuilder`로 프로그래밍적으로 생성하는 방법은?

**Q3.** `@MatrixVariable(pathVar = "userId")`에서 `pathVar`에 지정한 경로 변수가 존재하지 않는 URL이 요청됐을 때(`/orders/7`처럼 userId 세그먼트 없음), Spring MVC는 어떻게 동작하는가?

> 💡 **해설**
>
> **Q1.** `JSESSIONID`가 URL에 노출되면 세 가지 보안 위협이 발생합니다. (1) **세션 하이재킹**: URL이 로그, 브라우저 히스토리, Referer 헤더에 노출되면 공격자가 세션 ID를 탈취할 수 있습니다. (2) **세션 고정 공격(Session Fixation)**: 공격자가 미리 만든 세션 ID를 URL에 포함해 피해자에게 전달하면, 피해자가 로그인할 때 공격자의 세션이 인증됩니다. (3) **Cross-Site Request Forgery 보호 우회**: URL 기반 세션 추적은 CSRF 토큰과 결합이 복잡합니다. `removeSemicolonContent=true`는 이러한 위험을 방지하기 위해 `;jsessionid=...`를 요청 처리 전에 제거합니다. Spring Security는 이 설정과 협력해 URL에 세션 ID가 포함된 요청을 거부하도록 추가 설정할 수 있습니다.
>
> **Q2.** `UriComponentsBuilder`는 Matrix Variable을 직접 지원합니다: `UriComponentsBuilder.fromPath("/users/{userId}").matrixVars(matrixVars).buildAndExpand(Map.of("userId", "42"))`. 더 직접적으로는 `UriComponentsBuilder.newInstance().path("/users").pathSegment("{id}").buildAndExpand("42").toUriString()` 후 수동으로 `;role=admin` 추가. 또는 `UriComponentsBuilder.fromUriString("/users/{id}").buildAndExpand("42;role=admin").encode().toUriString()` — 단, `encode()`는 `;`를 `%3B`로 인코딩하므로 주의. `MvcUriComponentsBuilder`를 사용하면 Controller 메서드 참조로 더 타입 안전하게 생성할 수 있습니다.
>
> **Q3.** `pathVar = "userId"`로 지정했지만 매핑된 URL 패턴에 `{userId}` 경로 변수가 없으면, `MATRIX_VARIABLES_ATTRIBUTE` 맵에 `"userId"` 키가 존재하지 않습니다. `MatrixVariableMethodArgumentResolver.resolveName()`에서 해당 키의 `MultiValueMap`이 `null`이므로 `null`을 반환합니다. `required = true`(기본값)이면 `MissingMatrixVariableException`이 발생해 400 Bad Request가 반환됩니다. `required = false` 또는 `defaultValue`를 설정하면 기본값이 사용됩니다. 실제로는 `pathVar`에 지정한 경로 변수명이 `@GetMapping` 패턴의 `{변수명}`과 일치하는지 개발자가 주의해서 관리해야 합니다.

---

<div align="center">

**[⬅️ 이전: URI Template Variables 추출](./06-uri-template-variables.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 3 — Argument Resolution ➡️](../argument-resolution/01-argument-resolver-chain.md)**

</div>
