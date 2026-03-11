# @ResponseBody — Object → JSON 변환 과정

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `RequestResponseBodyMethodProcessor.handleReturnValue()`가 직렬화 대상 미디어 타입을 결정하는 과정은?
- `canWrite(Class, MediaType)` 두 조건의 내부 구현은?
- `MappingJackson2HttpMessageConverter`가 Java 객체를 JSON 바이트로 변환하는 단계는?
- `@JsonView`를 활용한 직렬화 필드 선택은 어떻게 동작하는가?
- `ResponseBodyAdvice`로 직렬화 전후에 개입하는 방법은?
- `produces` 조건 없는 핸들러에서 Content-Type이 결정되는 과정은?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: 동일 객체를 여러 형식으로 직렬화해야 한다

```
같은 User 객체를:
  API 클라이언트  → JSON {"id":1,"name":"홍길동"}
  레거시 시스템   → XML <User><id>1</id><name>홍길동</name></User>
  특수 클라이언트 → CSV 1,홍길동

해결: HttpMessageConverter 체인
  Accept 헤더 → 클라이언트가 원하는 형식 선언
  canWrite(User.class, acceptedType) → 처리 가능 여부
  write() → 직렬화 실행
```

---

## 😱 흔한 오해 또는 실수

### Before: @ResponseBody가 항상 JSON을 반환한다

```java
// ❌ 잘못된 이해
// "@ResponseBody = JSON 응답"

// ✅ 실제: Accept 헤더와 등록된 Converter에 따라 다름
@GetMapping("/users/{id}")
@ResponseBody
public User getUser(@PathVariable Long id) {
    return userService.findById(id);
}

// Accept: application/json → JSON
// Accept: application/xml  → XML (Jackson-XML 있을 때)
// Accept: */*              → 등록된 첫 번째 호환 Converter
// Accept: text/html        → 406 (적합한 Converter 없음)
```

### Before: StringHttpMessageConverter는 String 반환 타입에만 동작한다

```java
// @ResponseBody String 반환 → StringHttpMessageConverter
// @ResponseBody Object 반환 → MappingJackson2HttpMessageConverter

// ✅ 실제 함정:
// StringHttpMessageConverter.canWrite(String.class, */*) → true
// MappingJackson2HttpMessageConverter.canWrite(String.class, application/json) → true (!)
// 등록 순서상 StringHttpMessageConverter가 먼저 → Accept: */* 시 String 타입이면 text/plain 반환
// → @ResponseBody String 반환 + Accept: */* → "Content-Type: text/plain"
//   (JSON 기대했다면 예상과 다른 결과)
```

---

## ✨ 올바른 이해와 사용

### After: 직렬화 전체 흐름

```
① Accept 헤더 파싱:
   ContentNegotiationManager → [application/json(q=1.0), */*( q=0.8)]

② produces 조건 확인 (핸들러에 produces="application/json" 있으면):
   매핑 단계에서 이미 필터됨 → 이 핸들러 = JSON만 지원

③ 응답 가능한 미디어 타입 목록 수집:
   등록된 HttpMessageConverter.canWrite(User.class, ?) 결과:
   StringHttpMessageConverter   → text/plain, text/html (User 타입은 false)
   ByteArrayHttpMessageConverter → application/octet-stream (User 타입은 false)
   MappingJackson2HttpMessageConverter → application/json (User 타입은 true) ✅

④ Accept ∩ producible = [application/json] → 선택

⑤ MappingJackson2HttpMessageConverter.write(user, application/json, response)
   → objectMapper.writeValue(outputStream, user)
   → {"id":1,"name":"홍길동","email":"hong@example.com"}
```

---

## 🔬 내부 동작 원리

### 1. writeWithMessageConverters() — 핵심 직렬화 로직

```java
// AbstractMessageConverterMethodProcessor.writeWithMessageConverters()
protected <T> void writeWithMessageConverters(@Nullable T value,
        MethodParameter returnType,
        ServletServerHttpRequest inputMessage,
        ServletServerHttpResponse outputMessage) throws ... {

    Object body;
    Class<?> valueType;
    Type targetType;

    if (value instanceof CharSequence) {
        body = value.toString();
        valueType = String.class;
        targetType = String.class;
    } else {
        body = value;
        valueType = getReturnValueType(body, returnType);
        targetType = GenericTypeResolver.resolveType(
            getGenericType(returnType), returnType.getContainingClass());
    }

    // 응답 가능한 미디어 타입 수집 (등록된 Converter 기반)
    List<MediaType> producibleMediaTypes = getProducibleMediaTypes(
        inputMessage.getServletRequest(), valueType, targetType);

    // 클라이언트가 수용 가능한 미디어 타입 (Accept 헤더)
    List<MediaType> acceptableMediaTypes = getAcceptableMediaTypes(
        inputMessage.getServletRequest());

    // 교집합 계산 + 품질 계수 정렬
    List<MediaType> mediaTypesToUse = new ArrayList<>();
    for (MediaType requestedType : acceptableMediaTypes) {
        for (MediaType producibleType : producibleMediaTypes) {
            if (requestedType.isCompatibleWith(producibleType)) {
                mediaTypesToUse.add(getMostSpecificMediaType(requestedType, producibleType));
            }
        }
    }
    if (mediaTypesToUse.isEmpty()) {
        if (body != null) {
            throw new HttpMediaTypeNotAcceptableException(producibleMediaTypes); // 406
        }
        return;
    }
    MediaType.sortBySpecificityAndQuality(mediaTypesToUse);
    MediaType selectedContentType = mediaTypesToUse.get(0); // 가장 적합한 타입 선택

    // 선택된 타입을 지원하는 Converter로 직렬화
    for (HttpMessageConverter<?> converter : this.messageConverters) {
        GenericHttpMessageConverter genericConverter =
            (converter instanceof GenericHttpMessageConverter gc) ? gc : null;

        if (genericConverter != null ?
                genericConverter.canWrite(targetType, valueType, selectedContentType) :
                targetClass != null && ((HttpMessageConverter) converter).canWrite(valueType, selectedContentType)) {

            // ResponseBodyAdvice.beforeBodyWrite() 호출
            body = getAdvice().beforeBodyWrite(body, returnType, selectedContentType,
                (Class<? extends HttpMessageConverter<?>>) converter.getClass(),
                inputMessage, outputMessage);

            if (body != null) {
                // 헤더 설정
                outputMessage.getHeaders().setContentType(selectedContentType);
                // 직렬화 실행
                if (genericConverter != null) {
                    genericConverter.write(body, targetType, selectedContentType, outputMessage);
                } else {
                    ((HttpMessageConverter) converter).write(body, selectedContentType, outputMessage);
                }
            }
            return;
        }
    }
    // 적합한 Converter 없음 → 406
    if (body != null) {
        throw new HttpMediaTypeNotAcceptableException(this.allSupportedMediaTypes);
    }
}
```

### 2. MappingJackson2HttpMessageConverter.writeInternal()

```java
// AbstractJackson2HttpMessageConverter.writeInternal()
@Override
protected void writeInternal(Object object, Type type,
        HttpOutputMessage outputMessage) throws ... {

    MediaType contentType = outputMessage.getHeaders().getContentType();
    JsonEncoding encoding = getJsonEncoding(contentType);

    Class<?> serializationView = null;
    FilterProvider filters = null;
    Object value = object;
    JavaType javaType = null;

    // @JsonView 처리
    if (object instanceof MappingJacksonValue container) {
        value = container.getValue();
        serializationView = container.getSerializationView();
        filters = container.getFilters();
        javaType = container.getJavaType();
    }
    if (javaType == null) {
        javaType = (type != null && value != null && !value.getClass().equals(type) ?
            getJavaType(type, null) : null);
    }

    ObjectWriter objectWriter = (serializationView != null ?
        this.objectMapper.writerWithView(serializationView) :
        this.objectMapper.writer());
    if (filters != null) {
        objectWriter = objectWriter.with(filters);
    }
    if (javaType != null && (javaType.isContainerType() || !value.getClass().equals(javaType.getRawClass()))) {
        objectWriter = objectWriter.forType(javaType);
    }

    // 실제 직렬화
    JsonFactory jsonFactory = this.objectMapper.getFactory();
    JsonGenerator generator = jsonFactory.createGenerator(outputMessage.getBody(), encoding);
    objectWriter.writeValue(generator, value);
    // → OutputStream에 JSON 바이트 쓰기
}
```

### 3. @JsonView 활용

```java
// 뷰 정의
public class Views {
    public interface Public {}
    public interface Admin extends Public {}
}

public class User {
    @JsonView(Views.Public.class)
    public Long id;

    @JsonView(Views.Public.class)
    public String name;

    @JsonView(Views.Admin.class)
    public String email;    // Admin에만 포함

    public String password; // 어느 뷰에도 없음 → 항상 제외
}

@GetMapping("/users/{id}")
@JsonView(Views.Public.class)  // Public 뷰만 직렬화
public User getUser(@PathVariable Long id) {
    return userService.findById(id);
    // → {"id":1,"name":"홍길동"}  (email, password 제외)
}

@GetMapping("/admin/users/{id}")
@JsonView(Views.Admin.class)   // Admin 뷰 직렬화 (Public + Admin 필드)
public User getUserAdmin(@PathVariable Long id) {
    return userService.findById(id);
    // → {"id":1,"name":"홍길동","email":"hong@example.com"}
}
```

### 4. ResponseBodyAdvice — 직렬화 전후 개입

```java
// ResponseBodyAdvice 인터페이스
public interface ResponseBodyAdvice<T> {
    boolean supports(MethodParameter returnType,
                     Class<? extends HttpMessageConverter<?>> converterType);

    @Nullable
    T beforeBodyWrite(@Nullable T body,
                      MethodParameter returnType,
                      MediaType selectedContentType,
                      Class<? extends HttpMessageConverter<?>> selectedConverterType,
                      ServerHttpRequest request,
                      ServerHttpResponse response);
}

// 실전 예시: 모든 응답을 공통 래퍼로 감싸기
@RestControllerAdvice
public class ApiResponseWrapper implements ResponseBodyAdvice<Object> {

    @Override
    public boolean supports(MethodParameter returnType,
                            Class<? extends HttpMessageConverter<?>> converterType) {
        // String 반환 타입은 제외 (이미 String → text/plain)
        // ApiResponse 타입도 제외 (이중 래핑 방지)
        return !returnType.getParameterType().equals(String.class)
            && !ApiResponse.class.isAssignableFrom(returnType.getParameterType());
    }

    @Override
    public Object beforeBodyWrite(Object body, ...) {
        return ApiResponse.success(body);
        // {"status":"success","data":{"id":1,"name":"홍길동"}}
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: Accept 헤더별 Content-Type 확인

```bash
curl -H "Accept: application/json" http://localhost:8080/users/1
# Content-Type: application/json
# {"id":1,"name":"홍길동"}

curl -H "Accept: application/xml" http://localhost:8080/users/1
# Content-Type: application/xml (jackson-dataformat-xml 있을 때)
# <User><id>1</id><name>홍길동</name></User>

curl -H "Accept: text/csv" http://localhost:8080/users/1
# HTTP/1.1 406 Not Acceptable (CSV Converter 없으면)
```

### 실험 2: @JsonView 동작 확인

```bash
curl http://localhost:8080/users/1      # Public view
# {"id":1,"name":"홍길동"}

curl http://localhost:8080/admin/users/1  # Admin view
# {"id":1,"name":"홍길동","email":"hong@example.com"}
```

### 실험 3: ResponseBodyAdvice 등록 후 응답 래핑

```bash
curl http://localhost:8080/users/1
# 래핑 전: {"id":1,"name":"홍길동"}
# 래핑 후: {"status":"success","data":{"id":1,"name":"홍길동"},"timestamp":"..."}
```

---

## 🌐 HTTP 레벨 분석

```
GET /users/1 HTTP/1.1
Accept: application/json, */*;q=0.8

처리:
  producibleTypes: [application/json] (Jackson Converter가 User 지원)
  acceptableTypes: [application/json(1.0), *(0.8)]
  교집합: [application/json]
  선택: application/json

  ResponseBodyAdvice.beforeBodyWrite() 호출
  MappingJackson2HttpMessageConverter.write()
  → objectMapper.writeValue(outputStream, user)

HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 42
Vary: Accept
{"id":1,"name":"홍길동","email":"hong@example.com"}

Vary: Accept 헤더:
  Accept에 따라 응답이 달라질 수 있음을 프록시/CDN에 알림
  → Spring이 produces 조건 있거나 Content Negotiation 시 자동 추가
```

---

## 🤔 트레이드오프

```
ResponseBodyAdvice 래퍼 패턴:
  장점  모든 API 응답에 일관된 구조 적용
        로깅, 암호화 등 횡단 관심사 처리
  단점  클라이언트가 항상 래퍼 구조를 알아야 함
        오류 응답과 정상 응답 구조 통일 필요
        서드파티 도구(Swagger 등) 스펙 연동 복잡

@JsonView vs DTO 분리:
  @JsonView: 같은 도메인 객체, 뷰에 따라 필드 선택 → 코드 중복 없음
  별도 DTO:  명확한 관심사 분리, 컴파일 타임 안전 → 클래스 수 증가
  → 필드 수가 많고 뷰가 여러 개면 @JsonView
  → 응답 구조가 도메인과 많이 다르면 별도 DTO
```

---

## 📌 핵심 정리

```
직렬화 핵심 흐름
  writeWithMessageConverters():
    ① producibleTypes 수집 (canWrite 가능한 Converter + 타입)
    ② acceptableTypes 파싱 (Accept 헤더)
    ③ 교집합 → 품질 계수 정렬 → selectedContentType
    ④ ResponseBodyAdvice.beforeBodyWrite()
    ⑤ Converter.write() → OutputStream

406 발생 조건
  acceptableTypes ∩ producibleTypes = ∅
  → HttpMediaTypeNotAcceptableException → 406

@JsonView 처리
  @JsonView 어노테이션 → MappingJacksonValue 래퍼
  → writeInternal()에서 objectMapper.writerWithView()

ResponseBodyAdvice
  beforeBodyWrite(): 직렬화 직전 body 변환 가능
  supports(): 적용 범위 제어
  @RestControllerAdvice로 전역 등록
```

---

## 🤔 생각해볼 문제

**Q1.** `@ResponseBody String` 반환 시 `StringHttpMessageConverter`가 선택됩니다. 이 경우 Content-Type은 무엇인가? `Accept: application/json`을 보내면 JSON으로 직렬화되는가?

**Q2.** `ResponseBodyAdvice.beforeBodyWrite()`에서 반환값을 `null`로 반환하면 어떻게 되는가?

**Q3.** `MappingJackson2HttpMessageConverter`에 커스텀 `ObjectMapper`를 주입해서 날짜 형식을 변경하고 싶습니다. Spring Boot에서 이미 자동 구성된 Converter의 ObjectMapper 설정을 바꾸는 올바른 방법은?

> 💡 **해설**
>
> **Q1.** `StringHttpMessageConverter`는 `text/plain;charset=UTF-8`이나 `*/*`를 기본 미디어 타입으로 지원합니다. `Accept: application/json`과 `String` 타입 반환을 함께 사용하면, `StringHttpMessageConverter.canWrite(String.class, application/json)`은 `*/*`를 지원하므로 `application/json`도 호환됩니다. 따라서 Content-Type이 `application/json`으로 설정되고 `"홍길동"` 같은 문자열 그대로 응답됩니다 — Jackson이 JSON으로 직렬화하는 것이 아닙니다. JSON 문자열을 원한다면 반환 타입을 `Map` 또는 DTO로 바꿔야 합니다.
>
> **Q2.** `writeWithMessageConverters()`에서 `body != null` 조건으로 실제 쓰기를 수행하므로, `null` 반환 시 응답 본문에 아무것도 쓰지 않습니다. `Content-Length: 0`으로 빈 응답이 반환됩니다. `Content-Type` 헤더는 이미 설정된 상태일 수 있습니다.
>
> **Q3.** `Jackson2ObjectMapperBuilderCustomizer` Bean을 등록하는 것이 가장 안전합니다. 이 Bean은 Spring Boot 자동 구성의 `JacksonAutoConfiguration`이 `ObjectMapper`를 생성할 때 커스터마이징 훅으로 호출됩니다. 또는 `Jackson2ObjectMapperBuilder` Bean을 등록하거나, `application.yml`에서 `spring.jackson.*` 프로퍼티를 설정하는 방법도 있습니다. `MappingJackson2HttpMessageConverter` Bean을 직접 재정의하면 자동 구성과 충돌할 수 있어 주의가 필요합니다.

---

<div align="center">

**[⬅️ 이전: ReturnValueHandler 체인](./01-return-value-handler-chain.md)** | **[홈으로 🏠](../README.md)** | **[다음: ResponseEntity vs @ResponseStatus ➡️](./03-response-entity-vs-status.md)**

</div>
