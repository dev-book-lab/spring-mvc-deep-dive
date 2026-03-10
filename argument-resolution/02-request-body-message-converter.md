# @RequestBody와 HttpMessageConverter — JSON이 객체가 되는 과정

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `RequestResponseBodyMethodProcessor`가 `Content-Type`을 기반으로 `HttpMessageConverter`를 선택하는 과정은?
- `MappingJackson2HttpMessageConverter`는 JSON 바이트 스트림을 어떻게 Java 객체로 역직렬화하는가?
- `HttpMessageConverter.canRead(type, contentType)` 의 두 조건(타입 + 미디어 타입)은 각각 무엇을 검사하는가?
- `@RequestBody(required=false)` 일 때 본문이 비어있으면 어떻게 처리되는가?
- `RequestBodyAdvice`로 역직렬화 전/후에 개입하는 방법은?
- `@ResponseBody` 직렬화와 `@RequestBody` 역직렬화는 같은 Processor에서 처리되는가?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: 요청 본문의 형식이 JSON, XML, Form, Binary 등 다양하다

```
HTTP 요청 본문 → Java 객체 변환:
  Content-Type: application/json → JSON 파싱 → UserDto
  Content-Type: application/xml  → XML 파싱 → UserDto
  Content-Type: text/plain       → String 그대로
  Content-Type: application/octet-stream → byte[]

동일 @RequestBody 어노테이션으로 이 모든 형식을 처리해야 함
→ 형식별 변환 로직을 하나의 Processor가 직접 구현할 수 없음

해결: Strategy 패턴
  HttpMessageConverter 인터페이스 → 형식별 구현체
  RequestResponseBodyMethodProcessor:
    → Content-Type 헤더로 적합한 Converter 선택
    → Converter.read() 호출 → Java 객체 반환
    → 새 형식 지원 = 새 HttpMessageConverter 등록만 하면 됨
```

---

## 😱 흔한 오해 또는 실수

### Before: @RequestBody는 항상 JSON만 처리한다

```java
// ❌ 잘못된 이해
// "@RequestBody는 JSON 역직렬화 전용이다"

// ✅ 실제:
// @RequestBody는 HttpMessageConverter 체인에 위임
// 등록된 Converter와 Content-Type에 따라 다양한 형식 처리 가능

@PostMapping("/data")
public void receive(@RequestBody String raw) {
    // Content-Type: text/plain → StringHttpMessageConverter → String 그대로
}

@PostMapping("/bytes")
public void receiveBytes(@RequestBody byte[] data) {
    // Content-Type: application/octet-stream → ByteArrayHttpMessageConverter
}

@PostMapping("/xml")
public void receiveXml(@RequestBody UserDto dto) {
    // Content-Type: application/xml
    // → jackson-dataformat-xml 있으면 MappingJackson2XmlHttpMessageConverter
}
```

### Before: Content-Type이 없으면 @RequestBody가 동작하지 않는다

```
❌ 잘못된 이해:
  "Content-Type 헤더가 없으면 역직렬화 실패"

✅ 실제:
  Content-Type이 없으면 application/octet-stream 으로 취급
  → ByteArrayHttpMessageConverter 또는 StringHttpMessageConverter가 처리 가능
  application/json을 기대하는 경우라면:
  → canRead(UserDto.class, application/octet-stream) → false
  → 적합한 Converter 없음 → HttpMediaTypeNotSupportedException (415)

  일반적으로 JSON API라면 Content-Type: application/json 명시 필수
```

---

## ✨ 올바른 이해와 사용

### After: @RequestBody 역직렬화 전체 흐름

```
POST /users HTTP/1.1
Content-Type: application/json
{"name":"홍길동","email":"hong@example.com"}

① RequestResponseBodyMethodProcessor.resolveArgument()
   → EmptyBodyCheckingHttpInputMessage 생성 (본문 비어있는지 미리 확인)
   → RequestBodyAdvice.beforeBodyRead() 호출 (커스텀 개입 지점)
   → readWithMessageConverters() 호출

② AbstractMessageConverterMethodArgumentResolver.readWithMessageConverters()
   → Content-Type: application/json 파싱
   → 등록된 HttpMessageConverter 목록 순서대로:
      StringHttpMessageConverter.canRead(UserDto.class, application/json) → false
      ByteArrayHttpMessageConverter.canRead(UserDto.class, application/json) → false
      MappingJackson2HttpMessageConverter.canRead(UserDto.class, application/json) → true ✅
   → MappingJackson2HttpMessageConverter.read(UserDto.class, inputMessage)

③ MappingJackson2HttpMessageConverter.readJavaType()
   → objectMapper.readValue(inputStream, UserDto.class)
   → Jackson 역직렬화 → UserDto 인스턴스

④ RequestBodyAdvice.afterBodyRead() 호출
   → UserDto 반환 → Controller 메서드 파라미터에 주입
```

---

## 🔬 내부 동작 원리

### 1. RequestResponseBodyMethodProcessor — 역직렬화 + 직렬화 담당

```java
// RequestResponseBodyMethodProcessor.java
// HandlerMethodArgumentResolver (역직렬화) + HandlerMethodReturnValueHandler (직렬화) 동시 구현

@Override
public boolean supportsParameter(MethodParameter parameter) {
    return parameter.hasParameterAnnotation(RequestBody.class);
}

@Override
public Object resolveArgument(MethodParameter parameter,
        ModelAndViewContainer mavContainer,
        NativeWebRequest webRequest,
        WebDataBinderFactory binderFactory) throws Exception {

    parameter = parameter.nestedIfOptional();
    // EmptyBodyCheckingHttpInputMessage: 스트림을 미리 읽어 비어있는지 체크
    Object arg = readWithMessageConverters(webRequest, parameter,
        parameter.getNestedGenericParameterType());

    String name = Conventions.getVariableNameForParameter(parameter);
    if (binderFactory != null) {
        // @Valid 검증 처리
        WebDataBinder binder = binderFactory.createBinder(webRequest, arg, name);
        if (arg != null) {
            validateIfApplicable(binder, parameter);
            // BindingResult 파라미터가 없고 검증 오류 있으면 MethodArgumentNotValidException
            if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
                throw new MethodArgumentNotValidException(parameter, binder.getBindingResult());
            }
        }
        if (mavContainer != null) {
            mavContainer.addAttribute(BindingResult.MODEL_KEY_PREFIX + name, binder.getBindingResult());
        }
    }
    return adaptArgumentIfNecessary(arg, parameter);
}
```

### 2. readWithMessageConverters() — Converter 선택 로직

```java
// AbstractMessageConverterMethodArgumentResolver.readWithMessageConverters()
protected <T> Object readWithMessageConverters(HttpInputMessage inputMessage,
        MethodParameter parameter, Type targetType) throws ... {

    MediaType contentType;
    try {
        contentType = inputMessage.getHeaders().getContentType();
    } catch (InvalidMediaTypeException ex) {
        throw new HttpMediaTypeNotSupportedException(ex.getMessage());
    }
    if (contentType == null) {
        contentType = MediaType.APPLICATION_OCTET_STREAM;
    }

    Class<?> contextClass = parameter.getContainingClass();
    Class<T> targetClass = ... // 실제 타입 추출 (제네릭 포함)

    for (HttpMessageConverter<?> converter : this.messageConverters) {
        Class<HttpMessageConverter<?>> converterType = (Class<HttpMessageConverter<?>>) converter.getClass();

        if (converter instanceof GenericHttpMessageConverter<?> genericConverter) {
            // GenericHttpMessageConverter: 제네릭 타입 지원
            if (genericConverter.canRead(targetType, contextClass, contentType)) {
                // RequestBodyAdvice.beforeBodyRead() 호출
                inputMessage = getAdvice().beforeBodyRead(inputMessage, parameter, targetType, converterType);
                // 역직렬화
                body = genericConverter.read(targetType, contextClass, inputMessage);
                // RequestBodyAdvice.afterBodyRead() 호출
                body = getAdvice().afterBodyRead(body, inputMessage, parameter, targetType, converterType);
                break;
            }
        } else if (targetClass != null) {
            // 기본 HttpMessageConverter
            if (((HttpMessageConverter<T>) converter).canRead(targetClass, contentType)) {
                // 동일하게 Advice 호출 후 read()
                ...
            }
        }
    }
    // 적합한 Converter 없음
    if (body == NO_VALUE) {
        throw new HttpMediaTypeNotSupportedException(contentType, getSupportedMediaTypes(...));
    }
    return body;
}
```

### 3. MappingJackson2HttpMessageConverter — JSON 역직렬화

```java
// MappingJackson2HttpMessageConverter.java (AbstractJackson2HttpMessageConverter 상속)
@Override
public boolean canRead(Class<?> clazz, @Nullable MediaType mediaType) {
    return canRead(clazz, null, mediaType);
}

@Override
public boolean canRead(Type type, Class<?> contextClass, MediaType mediaType) {
    // ① 미디어 타입 확인: application/json, application/*+json 지원
    if (!canRead(mediaType)) return false;

    // ② Jackson이 이 타입을 역직렬화할 수 있는지 확인
    JavaType javaType = getJavaType(type, contextClass);
    AtomicReference<Throwable> causeRef = new AtomicReference<>();
    if (this.objectMapper.canDeserialize(javaType, causeRef)) return true;
    // canDeserialize: Jackson이 해당 타입의 deserializer를 찾을 수 있는지
    return false;
}

@Override
protected Object readInternal(Class<?> clazz, HttpInputMessage inputMessage) throws ... {
    JavaType javaType = getJavaType(clazz, null);
    return readJavaType(javaType, inputMessage);
}

private Object readJavaType(JavaType javaType, HttpInputMessage inputMessage) {
    try {
        InputStream inputStream = inputMessage.getBody();
        // 인코딩 확인
        if (inputMessage instanceof MappingJacksonInputMessage mappingInputMessage) {
            Class<?> deserializationView = mappingInputMessage.getDeserializationView();
            if (deserializationView != null) {
                ObjectReader objectReader = this.objectMapper.readerWithView(deserializationView)
                    .forType(javaType);
                return objectReader.readValue(inputStream);
            }
        }
        return this.objectMapper.readValue(inputStream, javaType);
        // → Jackson ObjectMapper가 JSON 파싱 + UserDto 인스턴스 생성
    } catch (InvalidDefinitionException ex) {
        throw new HttpMessageConversionException("...", ex);
    } catch (JsonProcessingException ex) {
        throw new HttpMessageNotReadableException("JSON parse error: " + ex.getMessage(), ex, inputMessage);
        // → 400 Bad Request
    }
}
```

### 4. EmptyBodyCheckingHttpInputMessage — 본문 비어있는지 확인

```java
// @RequestBody(required=true) + 본문 없는 경우 처리
public class EmptyBodyCheckingHttpInputMessage implements HttpInputMessage {
    private final HttpHeaders headers;
    @Nullable
    private final InputStream body;
    private final boolean hasBody;

    public EmptyBodyCheckingHttpInputMessage(HttpInputMessage inputMessage) throws IOException {
        this.headers = inputMessage.getHeaders();
        InputStream inputStream = inputMessage.getBody();
        // PushbackInputStream으로 1바이트 미리 읽어 비어있는지 확인
        if (inputStream.markSupported()) {
            inputStream.mark(1);
            this.hasBody = (inputStream.read() != -1);
            inputStream.reset();
        } else {
            PushbackInputStream pushbackInputStream = new PushbackInputStream(inputStream);
            int b = pushbackInputStream.read();
            if (b == -1) {
                this.hasBody = false;
                this.body = null;
            } else {
                this.hasBody = true;
                pushbackInputStream.unread(b);
                this.body = pushbackInputStream;
            }
        }
    }
}

// readWithMessageConverters()에서:
// EmptyBodyCheckingHttpInputMessage.hasBody() == false 이고
// required == true 이면:
//   HttpMessageNotReadableException("Required request body is missing") → 400
// required == false 이면:
//   null 반환 (파라미터에 null 주입)
```

### 5. RequestBodyAdvice — 개입 지점

```java
// RequestBodyAdvice 인터페이스
public interface RequestBodyAdvice {
    // 이 Advice가 이 파라미터에 적용될지 여부
    boolean supports(MethodParameter parameter, Type targetType,
                     Class<? extends HttpMessageConverter<?>> converterType);

    // 역직렬화 전 (inputMessage 수정 가능)
    HttpInputMessage beforeBodyRead(HttpInputMessage inputMessage,
                                    MethodParameter parameter, Type targetType,
                                    Class<? extends HttpMessageConverter<?>> converterType)
                                    throws IOException;

    // 역직렬화 후 (body 수정 가능)
    Object afterBodyRead(Object body, HttpInputMessage inputMessage,
                         MethodParameter parameter, Type targetType,
                         Class<? extends HttpMessageConverter<?>> converterType);

    // 본문이 비어있을 때 (null 대신 다른 값 반환 가능)
    @Nullable
    Object handleEmptyBody(@Nullable Object body, HttpInputMessage inputMessage,
                           MethodParameter parameter, Type targetType,
                           Class<? extends HttpMessageConverter<?>> converterType);
}

// 실전 사용 예시: 요청 본문 복호화
@ControllerAdvice
public class DecryptRequestBodyAdvice implements RequestBodyAdvice {

    @Override
    public boolean supports(MethodParameter parameter, Type targetType,
                            Class<? extends HttpMessageConverter<?>> converterType) {
        return parameter.hasParameterAnnotation(Encrypted.class);
    }

    @Override
    public HttpInputMessage beforeBodyRead(HttpInputMessage inputMessage, ...) throws IOException {
        // 암호화된 본문을 복호화해서 새 InputStream으로 래핑
        byte[] encrypted = inputMessage.getBody().readAllBytes();
        byte[] decrypted = decrypt(encrypted);
        return new HttpInputMessage() {
            public InputStream getBody() { return new ByteArrayInputStream(decrypted); }
            public HttpHeaders getHeaders() { return inputMessage.getHeaders(); }
        };
    }
    // ...
}
```

---

## 💻 실험으로 확인하기

### 실험 1: Content-Type별 Converter 선택 확인

```bash
# JSON
curl -X POST http://localhost:8080/users \
     -H "Content-Type: application/json" \
     -d '{"name":"홍길동"}'
# → MappingJackson2HttpMessageConverter 선택 → UserDto

# XML (jackson-dataformat-xml 있을 때)
curl -X POST http://localhost:8080/users \
     -H "Content-Type: application/xml" \
     -d '<UserDto><name>홍길동</name></UserDto>'
# → MappingJackson2XmlHttpMessageConverter 선택 → UserDto

# 지원하지 않는 Content-Type
curl -X POST http://localhost:8080/users \
     -H "Content-Type: application/pdf" \
     -d 'data'
# HTTP/1.1 415 Unsupported Media Type
```

### 실험 2: 비어있는 본문 처리

```bash
# required=true (기본) + 빈 본문
curl -X POST http://localhost:8080/users \
     -H "Content-Type: application/json"
# HTTP/1.1 400 Bad Request
# "Required request body is missing"

# required=false + 빈 본문
# @RequestBody(required=false) UserDto dto
curl -X POST http://localhost:8080/users \
     -H "Content-Type: application/json"
# → dto = null (정상 처리)
```

### 실험 3: 등록된 HttpMessageConverter 목록 확인

```java
@Autowired
RequestMappingHandlerAdapter adapter;

@GetMapping("/converters")
public List<String> listConverters() {
    return adapter.getMessageConverters().stream()
        .map(c -> c.getClass().getSimpleName() + " → " +
            c.getSupportedMediaTypes().toString())
        .collect(Collectors.toList());
}
```

---

## 🌐 HTTP 레벨 분석

```
POST /users HTTP/1.1
Content-Type: application/json;charset=UTF-8
Content-Length: 42
{"name":"홍길동","email":"hong@example.com"}

처리 흐름:
  ① ContentType 파싱: "application/json" (charset 무시)
  ② Converter 탐색:
     ByteArrayHttpMessageConverter  canRead(UserDto, application/json) → false
     StringHttpMessageConverter     canRead(UserDto, application/json) → false
     ResourceHttpMessageConverter   canRead(UserDto, application/json) → false
     MappingJackson2HttpMessageConverter canRead(UserDto, application/json) → true ✅
  ③ objectMapper.readValue(body, UserDto.class)
  ④ UserDto{name="홍길동", email="hong@example.com"} 반환

JSON 파싱 오류 시:
  POST body: {"name": 잘못된 JSON}
  → JsonProcessingException
  → HttpMessageNotReadableException
  HTTP/1.1 400 Bad Request
  {"status":400,"error":"Bad Request","message":"JSON parse error: ..."}
```

---

## 🤔 트레이드오프

```
HttpMessageConverter 체인:
  장점  Content-Type 기반 자동 형식 선택 → 하나의 엔드포인트가 다중 형식 지원
        새 형식 = 새 Converter 등록만으로 확장
        @RequestBody, @ResponseBody 동일 Converter 재사용
  단점  Converter 등록 순서가 중요 (첫 번째 canRead()==true 선택)
        커스텀 ObjectMapper 설정 시 Bean 등록 순서 주의 필요

RequestBodyAdvice:
  장점  역직렬화 전/후 공통 로직 적용 (로깅, 복호화, 변환)
        @ControllerAdvice로 전역 적용 또는 supports()로 선택적 적용
  단점  Advice 체인이 많으면 디버깅 복잡
        beforeBodyRead()에서 InputStream 소비 시 실제 read() 실패 가능
        → 반드시 새 InputStream으로 래핑해서 반환해야 함
```

---

## 📌 핵심 정리

```
@RequestBody 처리 주체
  RequestResponseBodyMethodProcessor
  (HandlerMethodArgumentResolver + HandlerMethodReturnValueHandler 겸용)

Converter 선택 알고리즘
  Content-Type 헤더 파싱 → MediaType 객체
  등록된 HttpMessageConverter 순서대로:
    canRead(타입, contentType) → true인 첫 번째 선택
  없으면 415 Unsupported Media Type

MappingJackson2HttpMessageConverter
  지원 미디어: application/json, application/*+json
  역직렬화: objectMapper.readValue(inputStream, targetType)
  파싱 오류: HttpMessageNotReadableException → 400

빈 본문 처리
  EmptyBodyCheckingHttpInputMessage로 본문 비어있는지 미리 확인
  required=true + 빈 본문 → 400
  required=false + 빈 본문 → null 주입

RequestBodyAdvice 개입 지점
  beforeBodyRead(): InputStream 변환 가능 (복호화, 압축 해제)
  afterBodyRead():  역직렬화된 객체 변환 가능
  @ControllerAdvice + supports()로 적용 범위 제어
```

---

## 🤔 생각해볼 문제

**Q1.** `List<UserDto>` 타입의 `@RequestBody`로 `[{"name":"홍"},{"name":"길"}]` JSON 배열을 받을 수 있는가? `MappingJackson2HttpMessageConverter.canRead()`에서 `List<UserDto>`의 제네릭 타입 정보는 어떻게 처리되는가?

**Q2.** 같은 `ObjectMapper` Bean을 공유하는 두 개의 `MappingJackson2HttpMessageConverter`가 등록되어 있을 때, `@RequestBody` 처리 시 어느 것이 선택되는가?

**Q3.** `RequestBodyAdvice.beforeBodyRead()`에서 `inputMessage.getBody()`를 전부 읽어버리면 어떤 문제가 생기는가? 어떻게 해결해야 하는가?

> 💡 **해설**
>
> **Q1.** 가능합니다. `readWithMessageConverters()`에서 `targetType`은 `MethodParameter.getGenericParameterType()`으로 얻으며, `List<UserDto>`의 제네릭 정보가 그대로 전달됩니다. `MappingJackson2HttpMessageConverter`는 `GenericHttpMessageConverter`를 구현하므로 `canRead(Type, Class<?>, MediaType)` 메서드로 `List<UserDto>` 타입을 받습니다. `ObjectMapper.constructType(type)`으로 `JavaType`을 생성하고 `canDeserialize(javaType)`으로 Jackson이 이 타입을 처리할 수 있는지 확인합니다. Jackson은 `List<UserDto>`의 제네릭 정보를 유지하므로 각 요소를 `UserDto`로 역직렬화합니다.
>
> **Q2.** 등록 순서가 앞선 것이 선택됩니다. 두 Converter 모두 `canRead(UserDto.class, application/json) == true`이므로 첫 번째 것이 선택됩니다. Spring Boot의 자동 구성에서는 `JacksonHttpMessageConvertersConfiguration`이 하나의 `MappingJackson2HttpMessageConverter`만 등록하므로 이런 상황은 드뭅니다. 커스텀 Converter를 추가할 때는 `WebMvcConfigurer.configureMessageConverters()`(전체 교체) vs `extendMessageConverters()`(기존 목록에 추가)의 차이를 주의해야 합니다.
>
> **Q3.** `InputStream`은 한 번 읽으면 다시 읽을 수 없습니다. `beforeBodyRead()`에서 스트림을 모두 소비하면 이후 `converter.read()`에서 빈 스트림을 읽어 빈 객체 또는 예외가 발생합니다. 해결책: 읽은 바이트를 `ByteArrayInputStream`에 복사해 새 `HttpInputMessage`로 래핑해서 반환해야 합니다. `ContentCachingRequestWrapper`를 서블릿 필터 레벨에서 사용하면 스트림을 여러 번 읽을 수 있도록 캐싱할 수 있습니다.

---

<div align="center">

**[⬅️ 이전: ArgumentResolver 체인](./01-argument-resolver-chain.md)** | **[홈으로 🏠](../README.md)** | **[다음: @RequestParam vs @PathVariable 처리 ➡️](./03-request-param-path-variable.md)**

</div>
