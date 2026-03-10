# Content Negotiation — Accept 헤더로 응답 형식을 협상하는 방법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `ContentNegotiationManager`가 요청의 "원하는 미디어 타입"을 파악하는 세 가지 전략은 무엇인가?
- `ProducesRequestCondition`이 `Accept` 헤더를 검사하는 과정은?
- `406 Not Acceptable`이 발생하는 정확한 조건은?
- 브라우저가 보내는 `Accept: text/html,application/xhtml+xml,*/*;q=0.8` 같은 복잡한 헤더는 어떻게 처리되는가?
- `produces` 조건 없이 `@ResponseBody`만 있는 경우 Content Negotiation은 어떻게 동작하는가?
- `@RequestMapping(produces = "application/json")` 과 `HttpMessageConverter` 선택은 어떤 관계인가?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: 같은 리소스를 여러 형식으로 제공해야 하는 경우

```
하나의 /users/{id} 엔드포인트가:
  → 브라우저 요청     → HTML 페이지 반환
  → API 클라이언트   → JSON 반환
  → 레거시 시스템    → XML 반환

URL을 분리하는 방법 (/users/1.json, /users/1.xml) 대신:
  HTTP 표준 Content Negotiation 활용
  → 클라이언트: Accept 헤더로 원하는 형식 명시
  → 서버: 지원 가능한 형식 중 가장 적합한 것 선택
  → 같은 URL, 다른 Accept → 다른 응답 형식

Spring MVC에서 이 협상이 두 레벨에서 일어남:
  ① 핸들러 매핑 레벨: produces 조건 → 어떤 핸들러를 선택할 것인가?
  ② 응답 생성 레벨: HttpMessageConverter → 어떤 형식으로 직렬화할 것인가?
  (이 문서는 ① 핸들러 매핑 레벨 집중)
```

---

## 😱 흔한 오해 또는 실수

### Before: produces 조건이 없으면 Content Negotiation이 동작하지 않는다

```java
// ❌ 잘못된 이해
// "produces 없으면 항상 JSON을 반환한다"

@GetMapping("/users/{id}")  // produces 없음
@ResponseBody
public User getUser(@PathVariable Long id) {
    return userService.findById(id);
}

// ✅ 실제:
// produces 조건이 없어도 Content Negotiation은 응답 생성 레벨에서 동작
// → AcceptHeaderLocaleResolver가 Accept 헤더 읽음
// → HttpMessageConverter 목록에서 Accept와 호환되는 것 선택
// → JSON 컨버터: Accept: application/json, */* 에서 사용
// → XML 컨버터: Accept: application/xml 에서 사용 (Jackson-XML 있으면)

// produces 조건의 역할:
// "이 핸들러는 application/json만 생성 가능하다"고 매핑 단계에서 선언
// → Accept: text/html → 이 핸들러 매칭 안 됨 → 다른 핸들러 탐색
// → 없으면 406
```

### Before: 품질 계수(q)가 높을수록 낮은 우선순위다

```
❌ 잘못된 이해:
  "q=0.9는 q=1.0보다 덜 원하는 것이다"

✅ 실제:
  q 값은 0.0 ~ 1.0 범위, 기본값 1.0
  q 값이 높을수록 더 선호하는 형식

Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
  → text/html             q=1.0 (가장 원함)
  → application/xhtml+xml q=1.0
  → application/xml       q=0.9
  → */*                   q=0.8 (어떤 형식이든 받을 수 있지만 낮은 우선순위)
```

---

## ✨ 올바른 이해와 사용

### After: ContentNegotiationManager 세 가지 전략

```java
// ContentNegotiationManager는 여러 ContentNegotiationStrategy를 순서대로 적용
// 기본 전략 순서:

// 1. HeaderContentNegotiationStrategy (기본 활성화)
//    → Accept 헤더 파싱
//    → Accept: application/json, text/html;q=0.9 → [JSON(q=1.0), HTML(q=0.9)]

// 2. ParameterContentNegotiationStrategy (선택적 활성화)
//    → URL 파라미터로 형식 지정
//    → GET /users/1?format=json → application/json
//    → spring.mvc.contentnegotiation.parameter-name=format

// 3. PathExtensionContentNegotiationStrategy (Deprecated — Spring 5.3+)
//    → URL 확장자로 형식 지정
//    → GET /users/1.json → application/json
//    → PathExtensionContentNegotiationStrategy는 Spring Boot 3.x에서 기본 비활성

// WebMvcConfigurer로 커스터마이징:
@Override
public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
    configurer
        .favorParameter(true)                     // format=json 파라미터 활성화
        .parameterName("mediaType")               // 파라미터 이름 변경
        .ignoreAcceptHeader(false)                // Accept 헤더 무시 여부
        .defaultContentType(MediaType.APPLICATION_JSON)  // 기본 형식
        .mediaType("json", MediaType.APPLICATION_JSON)
        .mediaType("xml", MediaType.APPLICATION_XML);
}
```

---

## 🔬 내부 동작 원리

### 1. ProducesRequestCondition.getMatchingCondition()

```java
// ProducesRequestCondition.getMatchingCondition()
@Override
@Nullable
public ProducesRequestCondition getMatchingCondition(HttpServletRequest request) {
    if (CorsUtils.isPreFlightRequest(request)) {
        return PRE_FLIGHT_MATCH;  // Preflight는 검사 스킵
    }
    if (isEmpty()) {
        return this;  // produces 조건 없으면 항상 매칭
    }

    // 요청의 Accept 헤더에서 원하는 미디어 타입 목록 파악
    List<MediaType> acceptedMediaTypes = getAcceptedMediaTypes(request);

    // 조건에 있는 미디어 타입 중 클라이언트가 수용 가능한 것이 있는지 확인
    List<ProducesRequestCondition.MediaTypeExpression> result = new ArrayList<>();
    for (MediaTypeExpression expression : this.expressions) {
        if (expression.match(acceptedMediaTypes)) {
            result.add(expression);
        }
    }

    if (!result.isEmpty()) {
        return new ProducesRequestCondition(result, this.contentNegotiationManager);
    }
    return null;  // 매칭 미디어 타입 없음 → 406 후보
}

private List<MediaType> getAcceptedMediaTypes(HttpServletRequest request) {
    List<MediaType> result = this.mediaTypesCache.get(request);
    if (result == null) {
        // ContentNegotiationManager에게 위임
        try {
            result = this.contentNegotiationManager.resolveMediaTypes(
                new ServletWebRequest(request));
            this.mediaTypesCache.put(request, result);
        } catch (HttpMediaTypeNotAcceptableException ex) {
            return Collections.emptyList();
        }
    }
    return result;
}
```

### 2. 406 Not Acceptable 발생 경로

```java
// 1. 핸들러 매핑 레벨 — produces 조건 불일치
// lookupHandlerMethod() → handleNoMatch()
//   → hasProducesMismatch() → HttpMediaTypeNotAcceptableException
//   → DefaultHandlerExceptionResolver → 406

// 2. 응답 생성 레벨 — HttpMessageConverter 불일치
// AbstractMessageConverterMethodProcessor.writeWithMessageConverters()
//   → 지원 가능한 미디어 타입 목록 vs Accept 헤더
//   → 교집합 없음 → HttpMediaTypeNotAcceptableException → 406

// 두 레벨의 차이:
// 레벨 1: 핸들러 자체가 선택되지 않음
// 레벨 2: 핸들러는 실행됐으나 응답 형식 변환 실패
```

### 3. Accept 헤더 품질 계수 처리

```java
// MediaType.sortBySpecificityAndQuality() — 품질 계수 정렬
// Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8

List<MediaType> accepted = MediaType.parseMediaTypes(
    "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8");
MediaType.sortBySpecificityAndQuality(accepted);
// 정렬 결과:
// 1. text/html            q=1.0, specificity=highest (타입+서브타입 구체적)
// 2. application/xhtml+xml q=1.0
// 3. application/xml      q=0.9
// 4. */*                  q=0.8 (가장 낮음)

// 서버의 produces 조건이 "application/json"일 때:
// text/html과 application/json → 비호환
// application/xhtml+xml와 application/json → 비호환
// application/xml와 application/json → 비호환
// */* 와 application/json → 호환 ✅ (q=0.8이지만 허용)
// → 매칭 성공, 응답 Content-Type: application/json
```

### 4. produces 조건 없을 때의 응답 생성 레벨 Content Negotiation

```java
// AbstractMessageConverterMethodProcessor.writeWithMessageConverters()
protected <T> void writeWithMessageConverters(@Nullable T value,
        MethodParameter returnType,
        ServletServerHttpRequest inputMessage,
        ServletServerHttpResponse outputMessage) throws ... {

    // 1. 응답 가능한 미디어 타입 목록 수집 (등록된 컨버터 기반)
    List<MediaType> producibleMediaTypes = getProducibleMediaTypes(
        inputMessage.getServletRequest(), valueType, targetType);
    // → [application/json, application/xml, text/html, ...]

    // 2. 요청의 수용 가능한 미디어 타입 목록
    List<MediaType> acceptableMediaTypes = getAcceptableMediaTypes(
        inputMessage.getServletRequest());
    // → Accept 헤더 파싱 결과

    // 3. 교집합 계산 + 품질 계수 정렬
    List<MediaType> compatibleMediaTypes = new ArrayList<>();
    for (MediaType producible : producibleMediaTypes) {
        for (MediaType acceptable : acceptableMediaTypes) {
            if (acceptable.isCompatibleWith(producible)) {
                compatibleMediaTypes.add(getMostSpecificMediaType(acceptable, producible));
            }
        }
    }
    if (compatibleMediaTypes.isEmpty()) {
        throw new HttpMediaTypeNotAcceptableException(producibleMediaTypes);  // 406
    }

    MediaType.sortBySpecificityAndQuality(compatibleMediaTypes);
    MediaType selectedContentType = compatibleMediaTypes.get(0);

    // 4. 선택된 미디어 타입을 지원하는 HttpMessageConverter 선택 + write
    for (HttpMessageConverter<?> converter : this.messageConverters) {
        if (converter.canWrite(valueType, selectedContentType)) {
            // 직렬화 실행
            ((HttpMessageConverter) converter).write(body, selectedContentType, outputMessage);
            return;
        }
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: produces 조건 있는/없는 비교

```java
// produces 있음 — 매핑 레벨에서 Accept 검사
@GetMapping(value = "/typed", produces = MediaType.APPLICATION_JSON_VALUE)
public User typedResponse(@PathVariable Long id) {
    return userService.findById(id);
}

// produces 없음 — 응답 생성 레벨에서 Accept 검사
@GetMapping("/untyped/{id}")
public User untypedResponse(@PathVariable Long id) {
    return userService.findById(id);
}
```

```bash
# produces 있음, Accept 호환 → 정상
curl -H "Accept: application/json" http://localhost:8080/typed
# HTTP/1.1 200 OK, Content-Type: application/json

# produces 있음, Accept 불호환 → 매핑 레벨 406
curl -H "Accept: text/html" http://localhost:8080/typed
# HTTP/1.1 406 Not Acceptable (핸들러 실행 안 됨)

# produces 없음, Accept 불호환 → 컨버터 레벨 406
curl -H "Accept: application/pdf" http://localhost:8080/untyped/1
# HTTP/1.1 406 Not Acceptable (핸들러는 실행됐다가 직렬화 실패)
```

### 실험 2: Accept 헤더 품질 계수 우선순위 실험

```bash
# 브라우저 스타일 Accept 헤더
curl -H "Accept: text/html,application/json;q=0.9,*/*;q=0.8" \
     http://localhost:8080/users/1
# produces = "application/json" 인 핸들러와 produces = "text/html" 인 핸들러가 있을 때:
# → text/html 핸들러 선택 (q=1.0 > q=0.9)

# JSON만 원하는 경우
curl -H "Accept: application/json" http://localhost:8080/users/1
# → application/json 핸들러 선택
```

### 실험 3: ContentNegotiationManager 커스터마이징

```java
// format 파라미터로 미디어 타입 지정
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        configurer
            .favorParameter(true)
            .parameterName("format")
            .mediaType("json", MediaType.APPLICATION_JSON)
            .mediaType("xml",  MediaType.APPLICATION_XML);
    }
}
```

```bash
curl http://localhost:8080/users/1?format=json
# → Content-Type: application/json

curl http://localhost:8080/users/1?format=xml
# → Content-Type: application/xml
```

---

## 🌐 HTTP 레벨 분석

```
Content Negotiation 실제 HTTP 대화:

클라이언트 요청:
  GET /users/1 HTTP/1.1
  Accept: application/json, text/html;q=0.9, */*;q=0.8

서버 처리:
  ContentNegotiationManager: [application/json(1.0), text/html(0.9), *(0.8)]
  produces = "application/json" 핸들러: application/json → 호환 ✅
  → UserController#getUser 선택
  → Jackson이 User 객체 → JSON 직렬화

서버 응답:
  HTTP/1.1 200 OK
  Content-Type: application/json
  Content-Length: 42
  Vary: Accept           ← 프록시 캐시에 Accept 헤더에 따라 다른 응답임을 알림
  {"id":1,"name":"홍길동"}

Vary 헤더의 중요성:
  Content Negotiation 사용 시 같은 URL도 Accept에 따라 다른 응답
  → 프록시/CDN 캐시가 Accept 헤더를 캐시 키에 포함하도록 지시
  → Spring MVC가 produces 조건 있으면 자동으로 Vary: Accept 추가
```

---

## 🤔 트레이드오프

```
produces 조건 사용:
  장점  핸들러 매핑 레벨에서 조기 분기 → 불필요한 핸들러 실행 없음
        같은 URL에 미디어 타입별 다른 핸들러 → 명확한 책임 분리
  단점  같은 리소스에 여러 핸들러 → 코드 중복 가능
        produces 조건 변경 시 매핑 재등록 필요

ContentNegotiationManager 전략:
  Accept 헤더 방식:
    + HTTP 표준 준수, URL 구조 깔끔
    - 일부 클라이언트가 Accept 헤더를 올바르게 설정하지 않음
    - 캐시 처리 복잡 (Vary 헤더 필요)

  format 파라미터 방식:
    + 명시적, 디버깅 쉬움 (?format=json 로 브라우저에서 테스트)
    - URL에 응답 형식 정보 포함 → RESTful 원칙 위반
    - 캐시 키에 파라미터 포함 필요

  URL 확장자 방식 (Deprecated):
    + 매우 명시적 (/users/1.json)
    - Spring 5.3+ 에서 비권장 (보안 취약점 원인)
    - ReflectedFileDownload attack 가능성
```

---

## 📌 핵심 정리

```
Content Negotiation 두 레벨
  ① 핸들러 매핑 레벨 (ProducesRequestCondition)
     Accept 헤더 vs produces 조건 비교
     불일치 → 핸들러 탈락, 다른 핸들러 탐색
     모두 탈락 → 406 Not Acceptable

  ② 응답 생성 레벨 (AbstractMessageConverterMethodProcessor)
     produces 없을 때 적용
     Accept 헤더 vs 등록된 HttpMessageConverter 지원 타입 교집합 계산
     교집합 없음 → 406 Not Acceptable

ContentNegotiationManager 전략 (순서대로 적용)
  HeaderContentNegotiationStrategy  : Accept 헤더 (기본 활성)
  ParameterContentNegotiationStrategy: URL 파라미터 (선택적)
  PathExtensionContentNegotiationStrategy: URL 확장자 (Deprecated)

Accept 헤더 q 값
  0.0~1.0 범위, 기본 1.0
  클수록 더 선호 → 정렬 시 우선순위 결정

Vary 헤더
  produces 조건 있을 때 Spring MVC가 Vary: Accept 자동 추가
  → 프록시 캐시에게 Accept에 따라 응답이 달라짐을 알림
```

---

## 🤔 생각해볼 문제

**Q1.** `produces = {"application/json", "application/xml"}` 조건이 있는 핸들러와 `produces = "application/json"` 조건이 있는 핸들러가 같은 URL에 등록되어 있을 때, `Accept: application/json` 요청이 오면 어느 핸들러가 선택되는가? `compareTo()`의 produces 비교 기준으로 설명하라.

**Q2.** `produces` 조건이 없는 핸들러에서 `Accept: */*` 를 보내면 어떤 `Content-Type`으로 응답이 오는가? Spring Boot 기본 설정에서 JSON이 선택되는 이유는 무엇인가?

**Q3.** REST API에서 Content Negotiation을 활용해 같은 `/users/{id}` URL이 Accept 헤더에 따라 `application/json` 또는 `text/csv` 를 반환하도록 구현하려면 어떤 방법이 있는가? 각 방법의 장단점은?

> 💡 **해설**
>
> **Q1.** `produces = "application/json"` 조건이 있는 핸들러가 선택됩니다. `ProducesRequestCondition.compareTo()`는 표현식 수가 적을수록 더 구체적으로 평가합니다. `produces = "application/json"` (1개) vs `produces = {"application/json", "application/xml"}` (2개)에서 1개가 더 구체적입니다. 더 적은 미디어 타입을 선언한 핸들러가 해당 요청에 더 특화되어 있다고 판단하는 것입니다.
>
> **Q2.** Spring Boot 기본 설정에서는 `*/*` 에 대해 등록된 첫 번째 `HttpMessageConverter`가 선택됩니다. Spring Boot의 기본 컨버터 등록 순서에서 `MappingJackson2HttpMessageConverter`(JSON)는 상위에 위치하므로 JSON이 선택됩니다. 단, `StringHttpMessageConverter`가 `String` 반환 타입에 대해 더 먼저 동작합니다. 반환 타입이 POJO이면 Jackson JSON 컨버터가 먼저 `canWrite()`에서 true를 반환하므로 `application/json`으로 응답됩니다.
>
> **Q3.** 세 가지 방법이 있습니다. (1) **두 개의 produces 조건 핸들러**: `@GetMapping(produces="application/json")`과 `@GetMapping(produces="text/csv")`를 각각 선언해 JSON/CSV 변환 로직을 분리합니다. 명확하지만 코드 중복 발생. (2) **produces 없이 ReturnValueHandler 커스텀**: `HttpMessageConverter` 를 직접 구현해 `User` 클래스를 CSV로 변환하는 컨버터를 등록합니다. 하나의 핸들러로 두 형식 지원 가능. (3) **ResponseEntity + ContentNegotiationManager**: 핸들러 내에서 `ContentNegotiationManager`로 원하는 타입을 확인 후 직접 분기합니다. 가장 유연하지만 가장 복잡합니다.

---

<div align="center">

**[⬅️ 이전: HTTP Method 매칭과 OPTIONS 처리](./04-http-method-matching.md)** | **[홈으로 🏠](../README.md)** | **[다음: URI Template Variables 추출 ➡️](./06-uri-template-variables.md)**

</div>
