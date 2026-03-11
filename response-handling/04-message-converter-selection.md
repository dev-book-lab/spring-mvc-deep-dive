# HttpMessageConverter 선택 과정 — canWrite() 체크 순서와 우선순위 함정

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `canWrite(Class<?>, MediaType)` 두 조건은 내부적으로 어떻게 평가되는가?
- Spring Boot에 기본 등록되는 Converter 목록과 순서는?
- `StringHttpMessageConverter`가 `MappingJackson2HttpMessageConverter`보다 먼저 선택되는 시나리오와 해결책은?
- `ByteArrayHttpMessageConverter`, `ResourceHttpMessageConverter`의 역할은?
- 커스텀 Converter를 등록하는 올바른 방법(`configureMessageConverters` vs `extendMessageConverters`)은?
- `AllEncompassingFormHttpMessageConverter`는 무엇을 처리하는가?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: 요청/응답 본문의 형식이 다양하다

```
요청 Content-Type: application/json   → Jackson으로 역직렬화
응답 Accept:       application/json   → Jackson으로 직렬화
요청 Content-Type: multipart/form-data → 폼 + 파일 업로드
응답 Accept:       text/plain          → 문자열 그대로
응답 Accept:       application/octet-stream → 바이트 배열

"어떤 Converter가 이 타입과 미디어 타입 조합을 처리할 수 있는가?"
→ HttpMessageConverter.canRead() / canWrite() 체인
```

---

## 😱 흔한 오해 또는 실수

### Before: @ResponseBody String 반환 시 항상 application/json이 설정된다

```java
// ❌ 잘못된 이해
@GetMapping("/message")
@ResponseBody
public String message() {
    return "hello";
    // "Content-Type: application/json으로 반환된다"
}

// ✅ 실제 문제 시나리오:
// Accept: */*  또는 Accept: text/plain 요청 시:
//   StringHttpMessageConverter (먼저 등록됨)
//   → canWrite(String.class, */*) = true
//   → Content-Type: text/plain;charset=UTF-8 선택 ← JSON이 아님!
//
// Accept: application/json 요청 시:
//   StringHttpMessageConverter.canWrite(String.class, application/json)
//   → getSupportedMediaTypes()에 */* 포함 → true
//   → Content-Type: application/json 설정, "hello" 텍스트 그대로 응답
//      (JSON 직렬화 아님! 따옴표 없는 문자열)
//
// 해결: 문자열을 JSON으로 응답하려면 Map이나 DTO 반환
```

### Before: configureMessageConverters()와 extendMessageConverters()는 같다

```
❌ 잘못된 이해:
  "두 메서드 모두 커스텀 Converter를 추가하는 방법이다"

✅ 실제 차이:
  configureMessageConverters(List<> converters):
    → 이 메서드에서 목록을 채우면 기본 Converter를 모두 교체
    → 기본 Converter 중 필요한 것을 직접 추가해야 함
    → 실수로 Jackson Converter 누락 → JSON 처리 불가

  extendMessageConverters(List<> converters):
    → 기본 Converter가 이미 채워진 목록에 추가/수정
    → 기존 목록 유지 + 커스텀 추가
    → 커스텀 Converter를 앞에 삽입해 우선순위 제어 가능

  실무 권장: extendMessageConverters() 사용
```

---

## ✨ 올바른 이해와 사용

### After: Spring Boot 기본 Converter 등록 순서

```
Spring Boot (WebMvcAutoConfiguration) 기본 등록 순서:

1. ByteArrayHttpMessageConverter
   - 타입: byte[]
   - 미디어: application/octet-stream, */*

2. StringHttpMessageConverter
   - 타입: String (CharSequence)
   - 미디어: text/plain, */*
   ⚠️ */* 지원 → 모든 Accept에 응답 가능

3. ResourceHttpMessageConverter
   - 타입: Resource (파일, 클래스패스 리소스)
   - 미디어: */*

4. ResourceRegionHttpMessageConverter
   - 타입: ResourceRegion (범위 요청, Range 헤더)
   - 미디어: */*

5. AllEncompassingFormHttpMessageConverter
   - 타입: MultiValueMap (폼 데이터 + 멀티파트)
   - 미디어: application/x-www-form-urlencoded, multipart/form-data

6. MappingJackson2HttpMessageConverter (jackson-databind 있을 때)
   - 타입: Object (canDeserialize/canSerialize 체크)
   - 미디어: application/json, application/*+json

7. Jaxb2RootElementHttpMessageConverter (JAXB 있을 때)
   - 타입: @XmlRootElement 어노테이션 클래스
   - 미디어: application/xml, text/xml

8. MappingJackson2XmlHttpMessageConverter (jackson-dataformat-xml 있을 때)
   - 미디어: application/xml, text/xml
```

---

## 🔬 내부 동작 원리

### 1. canWrite() 두 조건 평가

```java
// AbstractHttpMessageConverter.canWrite()
@Override
public final boolean canWrite(Class<?> clazz, @Nullable MediaType mediaType) {
    // 조건 1: 이 Converter가 이 타입을 처리할 수 있는가?
    if (!supports(clazz)) return false;

    // 조건 2: 미디어 타입이 지원 목록과 호환되는가?
    if (mediaType == null || MediaType.ALL.equals(mediaType)) {
        return true;  // 미디어 타입 무관 → 타입만 맞으면 OK
    }
    for (MediaType supportedMediaType : getSupportedMediaTypes(clazz)) {
        if (supportedMediaType.isCompatibleWith(mediaType)) {
            return true;
        }
    }
    return false;
}

// GenericHttpMessageConverter (제네릭 타입 지원 버전)
// canWrite(Type type, Class<?> contextClass, MediaType mediaType)
// → 제네릭 정보(List<User> 등) 포함해서 처리 가능 여부 판단

// MappingJackson2HttpMessageConverter.canWrite() 내부:
// ① canWrite(mediaType): application/json 또는 */*+json 체크
// ② objectMapper.canSerialize(javaType): Jackson이 직렬화 가능한지 체크
//    → @JsonIgnoreProperties 같은 설정 반영
//    → 직렬화 불가 타입이면 false
```

### 2. StringHttpMessageConverter — */* 지원의 함정

```java
// StringHttpMessageConverter.java
@Override
public List<MediaType> getSupportedMediaTypes() {
    // 지원 미디어 타입 목록:
    // text/plain, */*
    // → */* 때문에 모든 Accept 헤더에 호환
}

@Override
protected boolean supports(Class<?> clazz) {
    return String.class == clazz;
    // String 타입만 처리
}

// 시나리오 1: String 반환 + Accept: */*
// 탐색 순서:
//   ByteArrayHttpMessageConverter: String 타입 → supports()=false
//   StringHttpMessageConverter:    String 타입 + */* 호환 → true ✅ 선택!
//   → Content-Type: text/plain

// 시나리오 2: String 반환 + Accept: application/json
// 탐색 순서:
//   StringHttpMessageConverter:
//     supports(String.class) = true
//     canWrite(application/json) = */* 포함 → isCompatibleWith(application/json) = true ✅
//   → text/plain 아닌 application/json Content-Type 설정
//   → "hello" 텍스트 (따옴표 없는 JSON이 아닌 plain text)

// 해결책: String을 JSON으로 응답하려면:
// Option 1: return Map.of("message", "hello")
// Option 2: return new StringWrapper("hello")
// Option 3: produces = MediaType.APPLICATION_JSON_VALUE 명시
```

### 3. MappingJackson2HttpMessageConverter.canWrite() 상세

```java
// AbstractJackson2HttpMessageConverter
@Override
protected boolean canWrite(MediaType mediaType) {
    if (mediaType == null || MediaType.ALL.isCompatibleWith(mediaType)) {
        return true;
    }
    for (MediaType supportedMediaType : getSupportedMediaTypes()) {
        // application/json, application/*+json
        if (supportedMediaType.isCompatibleWith(mediaType)) {
            return true;
        }
    }
    return false;
}

@Override
public boolean canWrite(Type type, @Nullable Class<?> contextClass, @Nullable MediaType mediaType) {
    if (!canWrite(mediaType)) return false;  // 미디어 타입 먼저 체크

    if (type == null) return true;
    if (contextClass == null) return true;

    JavaType javaType = getJavaType(type, contextClass);
    AtomicReference<Throwable> causeRef = new AtomicReference<>();
    // Jackson ObjectMapper.canSerialize() 체크
    return this.objectMapper.canSerialize(javaType.getRawClass(), causeRef);
}
```

### 4. configureMessageConverters vs extendMessageConverters

```java
// 잘못된 방법: configureMessageConverters 사용
@Override
public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
    // 이 목록에 추가하면 Spring의 기본 Converter 자동 등록 취소!
    converters.add(new MyCsvConverter());
    // → ByteArrayHttpMessageConverter, StringHttpMessageConverter 등 모두 사라짐
    // → Jackson Converter도 없음 → JSON 처리 불가
}

// 올바른 방법 1: extendMessageConverters 사용 (기본 유지)
@Override
public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
    converters.add(0, new HighPriorityConverter());  // 앞에 삽입 (우선순위 높음)
    converters.add(new MyCsvConverter());             // 뒤에 추가 (낮은 우선순위)
}

// 올바른 방법 2: configureMessageConverters + 기본 Converter 명시
@Override
public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
    converters.add(new ByteArrayHttpMessageConverter());
    converters.add(new StringHttpMessageConverter());
    // ... 필요한 기본 Converter 모두 명시
    converters.add(new MappingJackson2HttpMessageConverter(customObjectMapper));
    converters.add(new MyCsvConverter());
}

// 커스텀 ObjectMapper로 Jackson Converter 교체 (Spring Boot 권장):
@Bean
public Jackson2ObjectMapperBuilderCustomizer jacksonCustomizer() {
    return builder -> builder
        .dateFormat(new SimpleDateFormat("yyyy-MM-dd"))
        .featuresToEnable(SerializationFeature.INDENT_OUTPUT);
}
```

### 5. 커스텀 Converter 구현 예시 — CSV

```java
public class CsvHttpMessageConverter extends AbstractGenericHttpMessageConverter<Object> {

    private static final MediaType CSV_TYPE = new MediaType("text", "csv");

    public CsvHttpMessageConverter() {
        super(CSV_TYPE);
    }

    @Override
    protected boolean supports(Class<?> clazz) {
        return List.class.isAssignableFrom(clazz);
    }

    @Override
    protected Object readInternal(Class<?> clazz, HttpInputMessage inputMessage) throws ... {
        throw new UnsupportedOperationException("CSV 읽기 미지원");
    }

    @Override
    protected void writeInternal(Object object, Type type,
                                  HttpOutputMessage outputMessage) throws ... {
        List<?> list = (List<?>) object;
        Writer writer = new OutputStreamWriter(outputMessage.getBody(), StandardCharsets.UTF_8);
        // CSV 헤더 + 데이터 쓰기
        for (Object item : list) {
            // reflection 또는 직접 처리
            writer.write(toCsvLine(item));
            writer.write("\n");
        }
        writer.flush();
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: Converter 탐색 순서 로그

```yaml
logging:
  level:
    org.springframework.web.servlet.mvc.method.annotation.AbstractMessageConverterMethodProcessor: TRACE
```

### 실험 2: StringHttpMessageConverter 함정 재현

```java
@GetMapping("/string-trap")
@ResponseBody
public String stringTrap() {
    return "hello world";
}
```

```bash
# Accept 헤더 없음 (브라우저 기본)
curl http://localhost:8080/string-trap
# Content-Type: text/plain;charset=UTF-8
# hello world

# Accept: application/json 명시
curl -H "Accept: application/json" http://localhost:8080/string-trap
# Content-Type: application/json  ← JSON으로 Content-Type 설정되지만
# hello world                      ← JSON 형식 아님 (따옴표 없는 문자열)
```

### 실험 3: 등록된 Converter 전체 목록 확인

```java
@Autowired
RequestMappingHandlerAdapter adapter;

@GetMapping("/converters")
public List<String> listConverters() {
    return adapter.getMessageConverters().stream()
        .map(c -> c.getClass().getSimpleName()
            + " → " + c.getSupportedMediaTypes())
        .collect(Collectors.toList());
}
```

---

## 🌐 HTTP 레벨 분석

```
@ResponseBody User 반환 + Accept: application/json

Converter 탐색:
  ByteArrayHttpMessageConverter:  supports(User.class)=false → skip
  StringHttpMessageConverter:     supports(User.class)=false → skip
  ResourceHttpMessageConverter:   supports(User.class)=false → skip
  MappingJackson2HttpMessageConverter:
    canWrite(application/json)=true ✅
    objectMapper.canSerialize(User.class)=true ✅
    → 선택!

직렬화:
  objectMapper.writeValue(response.getOutputStream(), user)
  → {"id":1,"name":"홍길동"}

HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 28
{"id":1,"name":"홍길동"}

@ResponseBody String 반환 + Accept: */*

Converter 탐색:
  ByteArrayHttpMessageConverter:  supports(String.class)=false → skip
  StringHttpMessageConverter:
    supports(String.class)=true ✅
    canWrite(*/*) → */* 지원 → true ✅
    → 선택! (MappingJackson2보다 먼저 등록됨)

HTTP/1.1 200 OK
Content-Type: text/plain;charset=UTF-8  ← JSON 아님
hello
```

---

## 🤔 트레이드오프

```
configureMessageConverters vs extendMessageConverters:
  configureMessageConverters:
    + 완전한 제어 (정확히 원하는 것만)
    - 기본 Converter 누락 위험, 유지보수 부담
  extendMessageConverters:
    + 기본 유지 + 커스텀 추가
    + 앞 삽입으로 우선순위 조정
    - 기본 Converter와 충돌 가능

StringHttpMessageConverter */* 지원:
  설계 의도: 모든 요청에 기본 텍스트 응답 가능
  부작용:    String 타입 반환 시 예상치 못한 text/plain 응답
  해결책:    String 반환 지양, DTO/Map 사용
             또는 produces = "application/json" 명시
```

---

## 📌 핵심 정리

```
canWrite() 두 조건
  ① supports(Class<?>): 이 타입을 처리할 수 있는가
  ② 미디어 타입 호환: getSupportedMediaTypes()와 isCompatibleWith()

Spring Boot 기본 등록 순서 (핵심)
  ByteArray → String → Resource → Form → Jackson → XML

StringHttpMessageConverter */* 함정
  String 타입 + */* → text/plain으로 응답
  → JSON API에서 String 반환 지양
  → produces = APPLICATION_JSON_VALUE 명시로 해결

커스텀 Converter 등록
  extendMessageConverters(): 기본 유지 + 추가 (권장)
  configureMessageConverters(): 전체 교체 (신중히)
  앞에 삽입: converters.add(0, new MyConverter()) → 우선순위 최상

Jackson ObjectMapper 커스터마이징
  Jackson2ObjectMapperBuilderCustomizer Bean 등록 (Spring Boot 권장)
  직접 MappingJackson2HttpMessageConverter Bean 재정의 (충돌 주의)
```

---

## 🤔 생각해볼 문제

**Q1.** `List<User>`를 `@ResponseBody`로 반환할 때 `MappingJackson2HttpMessageConverter`의 `canWrite()`는 `List.class`를 평가하는가, `List<User>.class`를 평가하는가? 제네릭 타입 정보는 어떻게 유지되는가?

**Q2.** `configureMessageConverters()`를 구현해서 `MappingJackson2HttpMessageConverter`만 등록하면 `String` 반환 시 어떻게 되는가?

**Q3.** 같은 미디어 타입(`application/json`)을 지원하는 두 개의 Converter가 등록되어 있을 때 (예: 커스텀 Converter + Jackson), 어느 것이 선택되는가? 우선순위를 커스텀 쪽으로 바꾸려면 어떻게 하는가?

> 💡 **해설**
>
> **Q1.** `MappingJackson2HttpMessageConverter`는 `GenericHttpMessageConverter`를 구현하므로 `canWrite(Type type, Class<?> contextClass, MediaType mediaType)` 메서드를 사용합니다. `type`에 `List<User>`의 제네릭 타입 정보(`ParameterizedType`)가 그대로 전달됩니다. `getJavaType(type, contextClass)`를 통해 Jackson의 `CollectionType`으로 변환되어 `List<User>`의 제네릭 정보가 보존됩니다. Jackson은 이 타입 정보를 활용해 배열의 각 요소를 `User`로 직렬화합니다.
>
> **Q2.** `StringHttpMessageConverter`가 등록되지 않으므로 `String` 타입에 대해 `supports(String.class)==true`인 Converter가 없습니다. `MappingJackson2HttpMessageConverter.canWrite(String.class, */*)`는 `objectMapper.canSerialize(String.class)=true`이므로 true가 될 수 있습니다. 결과적으로 Jackson이 `String`을 처리하게 되어 `"hello"` (따옴표 포함 JSON 문자열)로 직렬화됩니다. 의도치 않은 동작이므로 `extendMessageConverters`를 사용해 기본 Converter를 유지하는 것이 안전합니다.
>
> **Q3.** 등록 순서가 앞선 것이 선택됩니다. `writeWithMessageConverters()`는 Converter 목록을 순서대로 탐색하여 첫 번째로 `canWrite()==true`인 것을 선택합니다. 커스텀 Converter를 Jackson보다 먼저 선택되게 하려면 `extendMessageConverters()`에서 `converters.add(0, new MyConverter())`로 리스트 맨 앞에 삽입하면 됩니다.

---

<div align="center">

**[⬅️ 이전: ResponseEntity vs @ResponseStatus](./03-response-entity-vs-status.md)** | **[홈으로 🏠](../README.md)** | **[다음: Content Negotiation과 Accept 헤더 ➡️](./05-content-negotiation-accept.md)**

</div>
