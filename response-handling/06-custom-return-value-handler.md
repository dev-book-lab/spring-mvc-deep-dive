# Custom Return Value Handler 작성 — 특수 반환 타입 처리와 암호화 응답 래핑

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `HandlerMethodReturnValueHandler`의 두 메서드 책임과 `ArgumentResolver`와의 차이는?
- `mavContainer.setRequestHandled(true)` 설정을 언제 해야 하는가?
- `addReturnValueHandlers()`로 등록한 핸들러의 위치와 한계는?
- 기본 핸들러보다 먼저 실행해야 하는 경우 `setReturnValueHandlers()`를 어떻게 사용하는가?
- 암호화 응답 래핑 — `@EncryptedResponse` 어노테이션 기반 핸들러 전체 구현 패턴은?
- `ResponseBodyAdvice` 대신 커스텀 ReturnValueHandler를 쓰는 것이 더 나은 시나리오는?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: 특수 반환 타입을 기본 핸들러가 처리하지 못한다

```java
// 시나리오 1: 페이징 결과를 항상 암호화해서 반환
@GetMapping("/secure/users")
@EncryptedResponse
public List<User> getUsers() {
    return userService.findAll();
    // 기본 핸들러: Jackson으로 JSON 직렬화
    // 목표: JSON → AES 암호화 → Base64 → 응답
}

// 시나리오 2: 커스텀 응답 타입
@GetMapping("/reports/{id}")
public CsvReport generateReport(@PathVariable Long id) {
    return reportService.generate(id);
    // CsvReport 타입 = 커스텀 타입
    // 기본 핸들러: 처리 못함 → 500 또는 뷰로 넘김
}

// 해결:
// HandlerMethodReturnValueHandler 구현
// → @EncryptedResponse / CsvReport 타입을 직접 처리
// → 응답 본문 완전 제어
```

---

## 😱 흔한 오해 또는 실수

### Before: addReturnValueHandlers()로 등록하면 @ResponseBody보다 먼저 실행된다

```java
// ❌ 잘못된 이해
@Override
public void addReturnValueHandlers(List<HandlerMethodReturnValueHandler> handlers) {
    handlers.add(new MyEncryptionHandler());
    // "이제 모든 @ResponseBody 반환에 암호화가 적용된다"
}

// ✅ 실제:
// addReturnValueHandlers()로 추가된 핸들러는
// 기본 핸들러 목록 뒤에 추가됨
// RequestResponseBodyMethodProcessor (@ResponseBody 처리) 가 먼저 실행
// → @EncryptedResponse 어노테이션 없이 전체 응답 암호화 불가

// 올바른 접근:
// Option 1: @EncryptedResponse 어노테이션 + 커스텀 핸들러 (addReturnValueHandlers 가능)
//            supportsReturnType()에서 어노테이션 체크
//            → 기본 핸들러가 @ResponseBody를 먼저 잡지만,
//              @EncryptedResponse만 true 반환하는 핸들러는 기본 핸들러가 처리 못함
//            ← 어노테이션 조합으로 충돌 방지

// Option 2: 전체 응답 암호화 → ResponseBodyAdvice 사용이 더 적합
```

### Before: handleReturnValue()에서 mavContainer.setRequestHandled(true)를 안 해도 된다

```java
// ❌ 빠뜨리면:
public void handleReturnValue(...) throws Exception {
    // 응답 직접 쓰기
    response.getWriter().write(encrypt(serialize(returnValue)));
    // setRequestHandled(true) 누락!
}
// → DispatcherServlet이 View 렌더링 시도
// → "홈" 같은 뷰 이름 추론 → ViewResolver 오류
// → 응답이 이미 커밋됐으면 IllegalStateException

// ✅ 반드시 포함:
public void handleReturnValue(...) throws Exception {
    mavContainer.setRequestHandled(true);  // View 렌더링 건너뜀
    // ... 응답 처리
}
```

---

## ✨ 올바른 이해와 사용

### After: @EncryptedResponse 커스텀 핸들러 전체 구현

```java
// ① 어노테이션 정의
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface EncryptedResponse {
    String algorithm() default "AES";
}

// ② HandlerMethodReturnValueHandler 구현
@Component
@RequiredArgsConstructor
public class EncryptedResponseReturnValueHandler
        implements HandlerMethodReturnValueHandler {

    private final ObjectMapper objectMapper;
    private final EncryptionService encryptionService;

    // ③ 처리 대상 판별: @EncryptedResponse 어노테이션 있는 메서드만
    @Override
    public boolean supportsReturnType(MethodParameter returnType) {
        return AnnotatedElementUtils.hasAnnotation(
            returnType.getContainingClass(), EncryptedResponse.class)
            || returnType.hasMethodAnnotation(EncryptedResponse.class);
    }

    // ④ 반환값 처리
    @Override
    public void handleReturnValue(@Nullable Object returnValue,
            MethodParameter returnType,
            ModelAndViewContainer mavContainer,
            NativeWebRequest webRequest) throws Exception {

        // 반드시 선언: 이 핸들러가 응답을 완전히 처리함
        mavContainer.setRequestHandled(true);

        HttpServletResponse response = webRequest.getNativeResponse(HttpServletResponse.class);

        if (returnValue == null) {
            response.setStatus(HttpServletResponse.SC_NO_CONTENT);
            return;
        }

        // JSON 직렬화
        String json = objectMapper.writeValueAsString(returnValue);

        // AES 암호화 + Base64 인코딩
        String encrypted = encryptionService.encrypt(json);

        // 응답 설정
        response.setContentType("application/octet-stream");
        response.setCharacterEncoding("UTF-8");
        response.setHeader("X-Encrypted", "true");
        response.setHeader("X-Algorithm", "AES");

        try (PrintWriter writer = response.getWriter()) {
            writer.write(encrypted);
        }
    }
}

// ⑤ 등록
@Configuration
@RequiredArgsConstructor
public class WebConfig implements WebMvcConfigurer {
    private final EncryptedResponseReturnValueHandler encryptedHandler;

    @Override
    public void addReturnValueHandlers(List<HandlerMethodReturnValueHandler> handlers) {
        handlers.add(encryptedHandler);
    }
}

// ⑥ 사용
@RestController
public class SecureController {

    @GetMapping("/secure/users")
    @EncryptedResponse
    public List<User> getUsers() {
        return userService.findAll();
        // → JSON → AES 암호화 → Base64 → 응답
    }
}
```

---

## 🔬 내부 동작 원리

### 1. HandlerMethodReturnValueHandler 인터페이스

```java
public interface HandlerMethodReturnValueHandler {

    // 이 핸들러가 이 반환 타입을 처리할 수 있는지
    // returnType: 메서드의 반환 타입 정보 (어노테이션, 제네릭 포함)
    boolean supportsReturnType(MethodParameter returnType);

    // 실제 반환값 처리
    // returnValue: 메서드가 실제로 반환한 값 (null 가능)
    // mavContainer: setRequestHandled(true) 호출로 View 렌더링 스킵 선언
    void handleReturnValue(@Nullable Object returnValue,
                           MethodParameter returnType,
                           ModelAndViewContainer mavContainer,
                           NativeWebRequest webRequest) throws Exception;
}
```

### 2. setReturnValueHandlers()로 우선순위 제어

```java
// @ResponseBody보다 먼저 실행되어야 하는 경우
@Configuration
public class WebConfig {

    @Autowired
    private RequestMappingHandlerAdapter adapter;

    @Autowired
    private MyHighPriorityHandler myHandler;

    @PostConstruct
    public void addHandlerFirst() {
        List<HandlerMethodReturnValueHandler> existing = adapter.getReturnValueHandlers();
        List<HandlerMethodReturnValueHandler> newList = new ArrayList<>();
        newList.add(myHandler);           // 커스텀을 맨 앞에
        if (existing != null) newList.addAll(existing);
        adapter.setReturnValueHandlers(newList);
    }
}
```

### 3. supportsReturnType() — 충돌 없는 조건 설계

```java
// 어노테이션 기반 (권장 — 기본 핸들러와 충돌 없음)
@Override
public boolean supportsReturnType(MethodParameter returnType) {
    // 클래스 레벨 또는 메서드 레벨 @EncryptedResponse 확인
    return AnnotatedElementUtils.hasAnnotation(
        returnType.getContainingClass(), EncryptedResponse.class)
        || returnType.hasMethodAnnotation(EncryptedResponse.class);
    // @ResponseBody와 @EncryptedResponse 모두 있는 경우:
    // 기본 핸들러 목록 탐색 순서상 RequestResponseBodyMethodProcessor가 먼저이지만
    // @EncryptedResponse 조건이 없으므로 이 핸들러는 선택 안 됨
    // addReturnValueHandlers()로 뒤에 추가된 경우:
    // @EncryptedResponse만 있고 @ResponseBody 없으면 기본 핸들러가 못 잡음
    // → 이 핸들러가 선택됨
}

// 타입 기반 (기본 핸들러와 충돌 가능성 있음)
@Override
public boolean supportsReturnType(MethodParameter returnType) {
    return CsvReport.class.isAssignableFrom(returnType.getParameterType());
    // CsvReport은 기본 핸들러가 처리 못함 → 폴백으로 이 핸들러 선택
    // (addReturnValueHandlers로 추가해도 됨)
}
```

### 4. ResponseBodyAdvice vs Custom ReturnValueHandler 선택 기준

```java
// ResponseBodyAdvice가 적합한 경우:
// - @ResponseBody / ResponseEntity 반환 타입의 body를 수정하고 싶을 때
// - 공통 래핑 구조 (ApiResponse), 로깅, 압축
// - 기존 MessageConverter 체인을 그대로 쓰고 싶을 때

@RestControllerAdvice
public class WrapperAdvice implements ResponseBodyAdvice<Object> {
    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        return true;  // 모든 @ResponseBody 적용
    }
    @Override
    public Object beforeBodyWrite(Object body, ...) {
        return ApiResponse.of(body);  // 래핑
    }
}

// Custom ReturnValueHandler가 적합한 경우:
// - 기본 MessageConverter와 완전히 다른 직렬화 (암호화, 바이너리 포맷)
// - 특수 반환 타입 처리 (CsvReport, BinaryProtocol 등)
// - mavContainer 직접 제어 필요 (View 결정, Model 조작 등)
// - Content-Type을 완전히 다른 것으로 설정해야 할 때

// 판단 기준:
// Jackson 체인 + body 수정만 필요 → ResponseBodyAdvice
// 완전히 다른 처리 경로 필요    → Custom ReturnValueHandler
```

---

## 💻 실험으로 확인하기

### 실험 1: @EncryptedResponse 동작 확인

```bash
# @EncryptedResponse 없는 일반 엔드포인트
curl http://localhost:8080/api/users
# Content-Type: application/json
# [{"id":1,"name":"홍길동"},...]

# @EncryptedResponse 있는 엔드포인트
curl http://localhost:8080/secure/users
# Content-Type: application/octet-stream
# X-Encrypted: true
# bWFkZSB5b3UgbG9vay4uLg==  ← Base64 암호화된 데이터
```

### 실험 2: mavContainer.setRequestHandled() 누락 실험

```java
// 의도적으로 setRequestHandled 누락
@Override
public void handleReturnValue(Object returnValue, ...) throws Exception {
    // setRequestHandled(true) 생략!
    response.getWriter().write("test");
}
```

```
# 결과: DispatcherServlet이 View 렌더링 시도
# → "No view found for name 'yourEndpointPath'"
# → 이미 응답이 쓰인 후 오류 발생
```

### 실험 3: 커스텀 핸들러 단위 테스트

```java
@ExtendWith(MockitoExtension.class)
class EncryptedResponseReturnValueHandlerTest {

    @Mock ObjectMapper objectMapper;
    @Mock EncryptionService encryptionService;
    @InjectMocks EncryptedResponseReturnValueHandler handler;

    @Test
    void supportsReturnType_true_when_EncryptedResponse_present() throws Exception {
        Method method = TestController.class.getMethod("secureEndpoint");
        MethodParameter returnType = new MethodParameter(method, -1);
        assertThat(handler.supportsReturnType(returnType)).isTrue();
    }

    @Test
    void handleReturnValue_writes_encrypted_json() throws Exception {
        // given
        List<User> users = List.of(new User(1L, "홍길동"));
        when(objectMapper.writeValueAsString(users)).thenReturn("[{\"id\":1}]");
        when(encryptionService.encrypt("[{\"id\":1}]")).thenReturn("ENCRYPTED");

        MockHttpServletResponse response = new MockHttpServletResponse();
        NativeWebRequest webRequest = new ServletWebRequest(new MockHttpServletRequest(), response);
        ModelAndViewContainer mavContainer = new ModelAndViewContainer();

        // when
        handler.handleReturnValue(users, null, mavContainer, webRequest);

        // then
        assertThat(mavContainer.isRequestHandled()).isTrue();
        assertThat(response.getContentAsString()).isEqualTo("ENCRYPTED");
        assertThat(response.getHeader("X-Encrypted")).isEqualTo("true");
    }
}
```

---

## 🌐 HTTP 레벨 분석

```
@EncryptedResponse 처리 흐름:

GET /secure/users HTTP/1.1
Accept: */*

① ReturnValueHandlerComposite 탐색:
   ModelAndViewMethodReturnValueHandler:  supportsReturnType=false
   ...
   RequestResponseBodyMethodProcessor:   @ResponseBody 없음 → false
   EncryptedResponseReturnValueHandler:  @EncryptedResponse 있음 → true ✅

② EncryptedResponseReturnValueHandler.handleReturnValue():
   mavContainer.setRequestHandled(true)
   objectMapper.writeValueAsString(users) → JSON
   encryptionService.encrypt(json) → Base64(AES(json))
   response.setContentType("application/octet-stream")
   response.getWriter().write(encrypted)

HTTP/1.1 200 OK
Content-Type: application/octet-stream
X-Encrypted: true
X-Algorithm: AES
bWFkZSB5b3UgbG9vay4uLg==

DispatcherServlet:
  mavContainer.isRequestHandled()=true → ModelAndView=null → render 스킵
```

---

## 🤔 트레이드오프

```
Custom ReturnValueHandler vs ResponseBodyAdvice:
  ReturnValueHandler:
    + 완전한 응답 제어 (Content-Type, 헤더, 본문 형식)
    + MessageConverter 체인 완전 우회 가능
    + 특수 반환 타입 지원
    - 구현 복잡도 높음
    - 기존 기능(Accept 헤더 협상 등) 직접 구현 필요

  ResponseBodyAdvice:
    + 기존 MessageConverter 체인 재사용
    + Content Negotiation 자동 처리
    + 구현 간단
    - body 변환만 가능 (Content-Type 등 완전 변경 어려움)

어노테이션 기반 조건 vs 타입 기반:
  어노테이션 기반: 명시적 표시, 기본 핸들러와 충돌 없음
  타입 기반:       새 반환 타입에 자동 적용, 어노테이션 불필요
  → 공존 필요 시 어노테이션 기반 권장
```

---

## 📌 핵심 정리

```
HandlerMethodReturnValueHandler 인터페이스
  supportsReturnType(): 반환 타입 조건 → 캐시됨 (빠르게 구현)
  handleReturnValue():  매 요청마다 호출, null 처리 포함

mavContainer.setRequestHandled(true) 필수
  응답 직접 쓰는 핸들러라면 반드시 설정
  → DispatcherServlet의 View 렌더링 스킵

등록 위치와 우선순위
  addReturnValueHandlers(): 기본 뒤에 추가
  setReturnValueHandlers(): 전체 목록 직접 제어 (우선순위 자유)
  어노테이션 조건이면 addReturnValueHandlers()로도 충분

ResponseBodyAdvice vs ReturnValueHandler
  body 변환만  → ResponseBodyAdvice
  완전히 다른 처리 경로 → ReturnValueHandler

테스트 전략
  단위: MethodParameter 생성 + MockHttpServletResponse
  통합: @WebMvcTest + MockMvc
```

---

## 🤔 생각해볼 문제

**Q1.** `EncryptedResponseReturnValueHandler`에서 `mavContainer.setRequestHandled(true)` 후 `response.getWriter()`로 응답을 쓰면, 이후 Spring MVC의 `HttpMessageConverter`가 추가로 응답을 덧쓸 수 있는가?

**Q2.** `@EncryptedResponse`와 `@ResponseBody`를 모두 붙인 메서드가 있을 때, `addReturnValueHandlers()`로 등록한 `EncryptedResponseReturnValueHandler`가 선택되는가, `RequestResponseBodyMethodProcessor`가 선택되는가?

**Q3.** `handleReturnValue()` 내에서 비동기 처리를 하고 싶다면 어떻게 해야 하는가? 단순히 `CompletableFuture.runAsync()`로 응답을 쓰면 안 되는 이유는?

> 💡 **해설**
>
> **Q1.** `mavContainer.setRequestHandled(true)` 설정 후에는 `DispatcherServlet`이 `ModelAndView`를 `null`로 취급하여 `render()` 호출을 건너뜁니다. `RequestResponseBodyMethodProcessor` 같은 다른 ReturnValueHandler도 이미 `selectHandler()`에서 하나의 핸들러만 선택했으므로 추가로 실행되지 않습니다. 즉, 커스텀 핸들러가 응답을 완전히 소유하고 Spring MVC가 추가로 덧쓸 수 없습니다. 단, `response.flushBuffer()`나 `getWriter().flush()`를 호출해 응답을 커밋하면 이후 어떤 코드도 헤더를 변경할 수 없습니다.
>
> **Q2.** `addReturnValueHandlers()`로 추가된 핸들러는 기본 핸들러 목록 뒤에 위치합니다. `RequestResponseBodyMethodProcessor`(기본, 앞쪽)는 `supportsReturnType()`에서 `@ResponseBody` 어노테이션 존재 여부만 확인합니다 — `@EncryptedResponse`는 무관합니다. 따라서 `@ResponseBody`가 있으면 `RequestResponseBodyMethodProcessor`가 먼저 선택됩니다. `@EncryptedResponse`만 있고 `@ResponseBody`가 없으면 `EncryptedResponseReturnValueHandler`가 선택됩니다. 두 어노테이션이 동시에 있으면 `RequestResponseBodyMethodProcessor`가 우선입니다.
>
> **Q3.** `handleReturnValue()` 내에서 `CompletableFuture.runAsync()`로 별도 스레드에 응답을 쓰면 메인 스레드가 이미 반환된 후 비동기 스레드에서 응답 쓰기가 일어납니다. 이때 서블릿 컨테이너가 요청을 이미 완료 처리했거나, `HttpServletResponse` 객체가 재활용(풀링)됐을 수 있어 `IllegalStateException`이 발생합니다. 비동기 응답이 필요하면 `DeferredResult<T>`, `Callable<T>`, `CompletableFuture<T>` 같은 Spring MVC 비동기 반환 타입을 사용해야 합니다. 이 경우 `AsyncHandlerMethodReturnValueHandler`가 서블릿 비동기 컨텍스트(`AsyncContext`)를 열고 안전하게 비동기 응답을 처리합니다.

---

<div align="center">

**[⬅️ 이전: Content Negotiation과 Accept 헤더](./05-content-negotiation-accept.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 5 — Exception Handling ➡️](../exception-handling/01-handler-exception-resolver.md)**

</div>
