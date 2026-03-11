# Content Negotiation과 Accept 헤더 — 응답 미디어 타입 결정 알고리즘

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `ContentNegotiationManager`가 Accept 헤더·URL 파라미터를 종합해 미디어 타입을 결정하는 알고리즘은?
- `Accept: application/json;q=0.9, */*;q=0.8` 처럼 품질 계수(q)가 있으면 어떻게 정렬되는가?
- 브라우저가 보내는 `Accept: text/html,application/xhtml+xml,...` 에서 JSON API가 JSON을 반환하는 방법은?
- URL 확장자(`/users.json`) 기반 Content Negotiation은 왜 Spring Boot 기본에서 비활성화됐는가?
- `produces` 조건과 `ContentNegotiationManager`의 관계는?
- 커스텀 미디어 타입을 등록하고 지원하는 전체 과정은?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: 같은 리소스를 다양한 형식으로 제공해야 한다

```
GET /users/42

  모바일 앱     Accept: application/json → JSON
  레거시 시스템 Accept: application/xml  → XML
  브라우저      Accept: text/html        → HTML 뷰 렌더링
  기계 클라이언트 Accept: */*            → 서버 기본 형식

동일 컨트롤러, 동일 URL, 형식만 다름
→ Accept 헤더로 협상(negotiation)하여 가장 적합한 형식 결정
```

---

## 😱 흔한 오해 또는 실수

### Before: Accept: */* 이면 첫 번째 Converter가 선택된다

```
❌ 잘못된 이해:
  "Accept: */* → StringHttpMessageConverter가 먼저 등록됐으므로 항상 text/plain"

✅ 실제:
  Accept: */* 시 producible 타입 목록과 acceptable 타입 교집합 계산
  → producible 타입이 [application/json]이면 (Jackson Converter 지원)
     교집합 = [application/json]
  → StringHttpMessageConverter는 String 타입만 지원
     User 타입은 supports(User.class)=false → producible 목록에 없음
  → application/json이 선택됨 (순서 무관)

  */* 는 producible 타입 중 서버가 만들 수 있는 것을 선택
  = "서버야, 네가 줄 수 있는 거 아무거나"
```

### Before: URL 확장자 Content Negotiation은 안전하다

```
❌ 잘못된 이해:
  "/users/42.json 처럼 확장자로 형식을 지정할 수 있다. 편리하다."

✅ 실제 위험:
  Spring Framework 5.3, Spring Boot 2.6+ 에서 기본 비활성화 (보안 이유)
  → Path Traversal + Content Sniffing 공격 가능성
    예: /upload/malicious.exe → text/plain 반환되도록 조작
  → RFD (Reflected File Download) 취약점

  활성화 필요 시 명시적 설정 + 보안 검토 필요:
    configurer.favorPathExtension(true)
    → Spring Security와 함께 신중히 사용
```

---

## ✨ 올바른 이해와 사용

### After: ContentNegotiationManager 전체 결정 알고리즘

```
GET /users/42 HTTP/1.1
Accept: application/json;q=0.9, application/xml;q=0.7, */*;q=0.5

① ContentNegotiationManager.resolveMediaTypes(request)
   → 등록된 전략(Strategy) 목록 순서대로 실행:
     HeaderContentNegotiationStrategy (기본 활성):
       Accept 헤더 파싱 → [application/json(0.9), application/xml(0.7), *(0.5)]

② producibleMediaTypes 수집:
   등록된 Converter 목록 → canWrite() 가능한 타입들:
     MappingJackson2HttpMessageConverter → application/json ✅
     MappingJackson2XmlHttpMessageConverter → application/xml ✅
   → producible = [application/json, application/xml]

③ 교집합 계산:
   acceptable ∩ producible = [application/json, application/xml]

④ 품질 계수 정렬:
   application/json(q=0.9) > application/xml(q=0.7)
   → application/json 선택

⑤ 선택된 application/json으로 직렬화
```

---

## 🔬 내부 동작 원리

### 1. ContentNegotiationManager와 전략 목록

```java
// ContentNegotiationManager.java
public class ContentNegotiationManager implements ContentNegotiationStrategy {

    private final List<ContentNegotiationStrategy> strategies = new ArrayList<>();

    @Override
    public List<MediaType> resolveMediaTypes(NativeWebRequest request)
            throws HttpMediaTypeNotAcceptableException {
        for (ContentNegotiationStrategy strategy : this.strategies) {
            List<MediaType> mediaTypes = strategy.resolveMediaTypes(request);
            if (mediaTypes.equals(MEDIA_TYPE_ALL_LIST)) continue;
            return mediaTypes;
        }
        return MEDIA_TYPE_ALL_LIST;  // 결정 못 하면 */*
    }
}

// Spring Boot 기본 활성 전략 (순서대로):
// 1. HeaderContentNegotiationStrategy — Accept 헤더 파싱
// 2. (비활성) ParameterContentNegotiationStrategy — ?format=json 파라미터
// 3. (비활성) PathExtensionContentNegotiationStrategy — .json 확장자
```

### 2. HeaderContentNegotiationStrategy — Accept 파싱

```java
// HeaderContentNegotiationStrategy.java
@Override
public List<MediaType> resolveMediaTypes(NativeWebRequest request)
        throws HttpMediaTypeNotAcceptableException {
    String[] headerValueArray = request.getHeaderValues(HttpHeaders.ACCEPT);
    if (headerValueArray == null) {
        return MEDIA_TYPE_ALL_LIST;  // Accept 없으면 */*
    }
    List<String> headerValues = Arrays.asList(headerValueArray);
    try {
        List<MediaType> mediaTypes = MediaType.parseMediaTypes(headerValues);
        MediaType.sortBySpecificityAndQuality(mediaTypes);
        // → 품질 계수(q) 기준 정렬
        // application/json;q=0.9 > */*;q=0.8
        return !CollectionUtils.isEmpty(mediaTypes) ? mediaTypes : MEDIA_TYPE_ALL_LIST;
    } catch (InvalidMediaTypeException ex) {
        throw new HttpMediaTypeNotAcceptableException(...);
    }
}
```

### 3. 품질 계수(q) 정렬 알고리즘

```java
// MediaType.sortBySpecificityAndQuality()
// 정렬 기준 (내림차순, 값이 클수록 우선):
// 1. q 값 (기본 1.0, 없으면 1.0으로 처리)
// 2. 구체성 (application/json > application/* > */*)
// 3. 파라미터 수 (더 많은 파라미터 = 더 구체적)

// 예시:
// Accept: text/html, application/json;q=0.9, */*;q=0.5
// 정렬 후:
//   text/html                    (q=1.0, 구체적)
//   application/json             (q=0.9, 구체적)
//   */*                          (q=0.5, 와일드카드)

// 브라우저 일반 Accept 헤더:
// Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
// 정렬 후 (품질 계수 기준):
//   text/html           (q=1.0)
//   application/xhtml+xml (q=1.0)
//   image/avif          (q=1.0)
//   image/webp          (q=1.0)
//   application/xml     (q=0.9)
//   */*                 (q=0.8)
// → Jackson이 application/xml도 생산 가능하면 application/xml 먼저 선택
// → JSON API에서 @RestController + produces="application/json" 명시 필요
```

### 4. produces 조건과 Content Negotiation 관계

```java
// produces 조건이 있는 핸들러:
@GetMapping(value = "/users/{id}", produces = MediaType.APPLICATION_JSON_VALUE)
public User getUser(@PathVariable Long id) { ... }

// 요청 처리 흐름:
// ① 핸들러 매핑 단계: ProducesRequestCondition.getMatchingCondition()
//    Accept 헤더와 produces 교집합 확인
//    Accept: text/html → 교집합 없음 → 이 핸들러 매핑 실패 (406 후보)
//    Accept: application/json → 교집합 있음 → 매핑 성공

// ② 직렬화 단계: producibleMediaTypes = [application/json] (produces 조건)
//    Content Negotiation = produces와 Accept 교집합 → application/json

// produces 없는 경우:
// ② 직렬화 단계: producibleMediaTypes = Converter.canWrite()로 결정
//    브라우저 Accept (text/html 포함) → 교집합 = */* (와일드카드 매칭)
//    → 서버 기본 형식(Jackson → JSON) 반환

// 결론: REST API에서 produces = "application/json" 명시 권장
//       브라우저 요청에도 JSON 반환 보장
```

### 5. ParameterContentNegotiationStrategy — 명시적 활성화

```java
// ?format=json 파라미터 방식 (기본 비활성)
// 활성화 설정:
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        configurer
            .favorParameter(true)           // ?format= 파라미터 활성화
            .parameterName("format")        // 기본값 "format"
            .mediaType("json", MediaType.APPLICATION_JSON)  // format=json → application/json
            .mediaType("xml", MediaType.APPLICATION_XML)    // format=xml → application/xml
            .mediaType("csv", new MediaType("text", "csv")) // 커스텀 미디어 타입
            .defaultContentType(MediaType.APPLICATION_JSON);// 기본값
    }
}

// 사용:
// GET /users/42?format=json → application/json
// GET /users/42?format=xml  → application/xml
// GET /users/42?format=csv  → text/csv
```

### 6. 커스텀 미디어 타입 전체 등록 과정

```java
// 1. 커스텀 미디어 타입 정의
public static final MediaType APPLICATION_CSV = new MediaType("text", "csv");

// 2. 커스텀 Converter 구현
public class CsvHttpMessageConverter extends AbstractGenericHttpMessageConverter<List<?>> {
    public CsvHttpMessageConverter() { super(APPLICATION_CSV); }
    @Override protected boolean supports(Class<?> clazz) { return List.class.isAssignableFrom(clazz); }
    @Override protected void writeInternal(List<?> list, Type type, HttpOutputMessage out) throws ... {
        Writer w = new OutputStreamWriter(out.getBody(), StandardCharsets.UTF_8);
        // CSV 직렬화
    }
    @Override protected List<?> readInternal(Class<? extends List<?>> clazz, HttpInputMessage in) { return null; }
}

// 3. Converter + Content Negotiation 등록
@Configuration
public class CsvConfig implements WebMvcConfigurer {
    @Override
    public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
        converters.add(new CsvHttpMessageConverter());
    }

    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        configurer
            .favorParameter(true)
            .mediaType("csv", APPLICATION_CSV);
    }
}

// 4. 사용:
// GET /users?format=csv
// Accept: text/csv
// → CSV 직렬화
```

---

## 💻 실험으로 확인하기

### 실험 1: Accept 헤더별 응답 형식

```bash
# JSON 요청
curl -H "Accept: application/json" http://localhost:8080/users/1
# Content-Type: application/json, {"id":1,...}

# XML 요청 (jackson-dataformat-xml 있을 때)
curl -H "Accept: application/xml" http://localhost:8080/users/1
# Content-Type: application/xml, <User>...</User>

# 브라우저 Accept (text/html 포함)
curl -H "Accept: text/html,*/*;q=0.8" http://localhost:8080/users/1
# produces 없으면: Content-Type: application/json (Jackson 지원 타입)
# produces="application/json" 있으면: Content-Type: application/json
```

### 실험 2: 품질 계수 우선순위

```bash
# q=0.5인 application/json보다 q=0.9인 */*가 더 높지는 않음
curl -H "Accept: */*;q=0.9, application/json;q=0.5" http://localhost:8080/users/1
# producible=[application/json], acceptable 정렬=[*(0.9), json(0.5)]
# 교집합: application/json (유일)
# → application/json 선택 (q값보다 producible 목록이 제한)
```

### 실험 3: 406 발생 조건

```bash
curl -H "Accept: text/csv" http://localhost:8080/users/1
# CSV Converter 없으면:
# HTTP/1.1 406 Not Acceptable
```

---

## 🌐 HTTP 레벨 분석

```
Content Negotiation 전체 흐름:

GET /api/users/1 HTTP/1.1
Accept: application/json;q=1.0, application/xml;q=0.8, */*;q=0.5

① HeaderContentNegotiationStrategy:
   acceptable = [json(1.0), xml(0.8), *(0.5)]

② producibleMediaTypes 수집:
   MappingJackson2:    application/json ✅
   MappingJackson2Xml: application/xml ✅
   producible = [application/json, application/xml]

③ 교집합:
   json(1.0), xml(0.8) 모두 포함

④ 최고 품질 선택:
   application/json (q=1.0)

⑤ 응답:
HTTP/1.1 200 OK
Content-Type: application/json
Vary: Accept
{"id":1,"name":"홍길동"}

Vary: Accept 헤더:
  응답이 Accept 헤더에 따라 달라질 수 있음을 캐시/프록시에 알림
  → 동일 URL이라도 Accept가 다르면 다른 캐시 항목으로 처리
```

---

## 🤔 트레이드오프

```
Accept 헤더 기반 Content Negotiation:
  장점  표준 HTTP 메커니즘, REST 원칙에 충실
        하나의 URL로 다양한 형식 제공
  단점  브라우저 Accept 헤더가 예상치 못한 형식 선택 유발
        → REST API에서는 produces 명시로 방어

URL 파라미터 방식 (?format=json):
  장점  URL만으로 형식 명시, 브라우저에서 테스트 쉬움
  단점  URL에 표현 계층 정보 혼입, REST 설계 원칙 위반 논란

URL 확장자 방식 (/users.json):
  장점  직관적
  단점  보안 취약점 (RFD), Spring Boot 기본 비활성화
  → 사용하지 말 것

produces 조건 명시:
  장점  명확한 API 계약, 406 조기 감지
  단점  유연성 감소 (다중 형식 지원 시 별도 엔드포인트 필요)
  → REST API에서 JSON 단일 형식이면 produces="application/json" 권장
```

---

## 📌 핵심 정리

```
ContentNegotiationManager 전략
  기본 활성: HeaderContentNegotiationStrategy (Accept 헤더)
  명시 활성: ParameterContentNegotiationStrategy (?format=)
  보안 비활성: PathExtensionContentNegotiationStrategy (.json 확장자)

미디어 타입 결정 알고리즘
  ① Accept 파싱 → acceptable 목록 (q값 기준 정렬)
  ② Converter.canWrite() → producible 목록
  ③ acceptable ∩ producible → 후보
  ④ 가장 높은 q값 + 구체성 → 최종 선택
  ⑤ 교집합 없음 → 406 Not Acceptable

브라우저 Accept 헤더 방어
  produces = MediaType.APPLICATION_JSON_VALUE 명시
  → 브라우저 text/html 요청에도 JSON 보장

Vary: Accept 헤더
  Content Negotiation 시 자동 추가
  → 캐시 서버에 "Accept 헤더별로 캐시 분리" 지시

커스텀 미디어 타입
  HttpMessageConverter 구현 + extendMessageConverters() 등록
  ContentNegotiationConfigurer.mediaType() 으로 파라미터/확장자 매핑
```

---

## 🤔 생각해볼 문제

**Q1.** `Accept: */*` 요청과 `Accept` 헤더가 아예 없는 요청은 Content Negotiation에서 동일하게 처리되는가?

**Q2.** `@GetMapping(produces = {"application/json", "application/xml"})` 으로 두 형식을 모두 지원하는 핸들러에서, `Accept: application/xml;q=1.0, application/json;q=0.9` 요청이 오면 어느 형식이 선택되는가?

**Q3.** `ContentNegotiationConfigurer.defaultContentType(MediaType.APPLICATION_JSON)` 설정은 어떤 경우에 적용되는가? Accept 헤더가 있어도 적용되는가?

> 💡 **해설**
>
> **Q1.** 거의 동일하게 처리되지만 정확히는 다릅니다. `Accept: */*`는 `HeaderContentNegotiationStrategy`가 `[*/*]` 목록을 반환합니다. Accept 헤더가 없으면 `headerValueArray == null` → `MEDIA_TYPE_ALL_LIST`(`[*/*]`)를 반환합니다. 실질적으로 둘 다 `*/*`로 처리되어 producible 타입 중 첫 번째로 호환되는 것이 선택됩니다. 다만 `ParameterContentNegotiationStrategy`가 활성화된 경우, Accept 헤더 없는 요청은 파라미터 전략으로 넘어갈 수 있습니다.
>
> **Q2.** `application/xml`이 선택됩니다. produces 조건이 있으므로 `producibleMediaTypes = [application/json, application/xml]`입니다. `acceptableMediaTypes` 정렬 결과: `application/xml(q=1.0) > application/json(q=0.9)`. 교집합 후 첫 번째 후보인 `application/xml`이 선택됩니다.
>
> **Q3.** `defaultContentType`은 ContentNegotiationManager가 `*/*`(즉, 결정을 못 한 경우)를 반환했을 때 사용하는 마지막 폴백입니다. Accept 헤더가 있고 그 값이 `*/*`가 아닌 구체적인 타입이면 `defaultContentType`은 적용되지 않습니다. Accept 헤더가 없거나 `*/*`인 경우, 그리고 다른 전략도 결정을 못 한 경우에만 적용됩니다. `FixedContentNegotiationStrategy`로 구현되어 항상 지정된 타입을 반환합니다.

---

<div align="center">

**[⬅️ 이전: HttpMessageConverter 선택 과정](./04-message-converter-selection.md)** | **[홈으로 🏠](../README.md)** | **[다음: Custom Return Value Handler 작성 ➡️](./06-custom-return-value-handler.md)**

</div>
