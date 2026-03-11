# HandlerMethodReturnValueHandler 체인 — 반환 타입마다 적합한 핸들러를 찾는 과정

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `ServletInvocableHandlerMethod.invokeAndHandle()`은 반환값을 어떻게 처리하는가?
- `HandlerMethodReturnValueHandlerComposite`의 캐시 구조는 ArgumentResolver와 동일한가?
- `@ResponseBody`와 `ModelAndView` 반환이 서로 다른 핸들러로 처리되는 이유는?
- `mavContainer.setRequestHandled(true)`는 언제 설정되며 어떤 의미인가?
- 반환값이 `null`일 때 Spring MVC는 어떻게 동작하는가?
- 기본 ReturnValueHandler 등록 순서에서 우선순위가 중요한 시나리오는?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: 반환 타입이 수십 가지다

```
Controller 메서드 반환 타입으로 올 수 있는 것들:
  String             → View 이름으로 해석 (MVC 패턴)
  ModelAndView       → View + Model 직접 지정
  Object (+ @ResponseBody) → JSON/XML 직렬화
  ResponseEntity<T>  → 상태 코드 + 헤더 + 바디 완전 제어
  void               → 요청 처리 완료 (직접 응답 쓴 경우)
  HttpHeaders        → 헤더만 반환
  DeferredResult<T>  → 비동기 응답
  Callable<T>        → 비동기 응답
  StreamingResponseBody → 스트리밍
  Page<T>            → 커스텀 타입

각 타입마다 완전히 다른 처리 로직 필요
→ Chain of Responsibility 패턴으로 분리
```

---

## 😱 흔한 오해 또는 실수

### Before: @ResponseBody가 없으면 반환값이 무시된다

```java
// ❌ 잘못된 이해
@GetMapping("/data")
public UserDto getData() {  // @ResponseBody 없음
    return new UserDto("홍길동");
    // "반환값이 무시되고 404가 난다"
}

// ✅ 실제:
// @RestController = @Controller + @ResponseBody (클래스 레벨)
// → 모든 메서드에 @ResponseBody 효과 적용
// → 단순 @Controller에서 UserDto 반환 시:
//    ViewNameMethodReturnValueHandler가 처리를 시도
//    → UserDto는 String이 아니므로 supportsReturnType=false
//    → 폴백으로 ModelAttributeMethodProcessor가 처리
//    → "userDto" 이름으로 Model에 추가 + view 이름 결정 필요
//    → 뷰가 없으면 오류
```

### Before: null 반환은 항상 빈 응답 본문이 된다

```
❌ 잘못된 이해: "null 반환 → 빈 200 OK"

✅ 실제:
  @ResponseBody null 반환:
    → MessageConverter가 null을 직렬화 시도
    → 응답 본문 없음 (Content-Length: 0) → 200 OK

  String null 반환 (@RestController 제외):
    → ViewNameMethodReturnValueHandler: null이면 view 이름 결정 불가
    → RequestToViewNameTranslator로 URL에서 뷰 이름 추론

  ResponseEntity<Void> / ResponseEntity.noContent():
    → 명시적 204 No Content
```

---

## ✨ 올바른 이해와 사용

### After: invokeAndHandle() 전체 흐름

```java
// ServletInvocableHandlerMethod.invokeAndHandle()
public void invokeAndHandle(ServletWebRequest webRequest,
        ModelAndViewContainer mavContainer,
        Object... providedArgs) throws Exception {

    // ① 실제 Controller 메서드 실행 (ArgumentResolver로 파라미터 바인딩 후)
    Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);

    // ② 응답 상태 코드 설정 (@ResponseStatus 어노테이션 처리)
    setResponseStatus(webRequest);

    // ③ 반환값이 null이거나 요청이 이미 처리됐으면 완료
    if (returnValue == null) {
        if (isRequestNotModified(webRequest) || getResponseStatus() != null
                || mavContainer.isRequestHandled()) {
            disableContentCachingIfNecessary(webRequest);
            mavContainer.setRequestHandled(true);
            return;
        }
    }
    // ...
    // ④ ReturnValueHandler 체인에 위임
    try {
        this.returnValueHandlers.handleReturnValue(
            returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
    } catch (Exception ex) {
        throw ex;
    }
}
```

---

## 🔬 내부 동작 원리

### 1. HandlerMethodReturnValueHandlerComposite — 캐시와 위임

```java
// ArgumentResolverComposite와 구조 동일
public class HandlerMethodReturnValueHandlerComposite
        implements HandlerMethodReturnValueHandler {

    private final List<HandlerMethodReturnValueHandler> returnValueHandlers = new ArrayList<>();

    // 캐시: MethodParameter → ReturnValueHandler
    private final Map<MethodParameter, HandlerMethodReturnValueHandler>
        returnValueHandlerCache = new ConcurrentHashMap<>(64);

    @Override
    public void handleReturnValue(@Nullable Object returnValue,
            MethodParameter returnType,
            ModelAndViewContainer mavContainer,
            NativeWebRequest webRequest) throws Exception {

        HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
        if (handler == null) {
            throw new IllegalArgumentException("Unknown return value type: " +
                returnType.getParameterType().getName());
        }
        handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
    }

    @Nullable
    private HandlerMethodReturnValueHandler selectHandler(
            @Nullable Object value, MethodParameter returnType) {
        // 비동기 결과는 별도 처리
        boolean isAsyncValue = isAsyncReturnValue(value, returnType);
        for (HandlerMethodReturnValueHandler handler : this.returnValueHandlers) {
            if (isAsyncValue && !(handler instanceof AsyncHandlerMethodReturnValueHandler)) continue;
            if (handler.supportsReturnType(returnType)) {
                return handler;
            }
        }
        return null;
    }
}
```

### 2. 기본 ReturnValueHandler 등록 순서

```java
// RequestMappingHandlerAdapter.getDefaultReturnValueHandlers()
List<HandlerMethodReturnValueHandler> handlers = new ArrayList<>(20);

// --- 단일 목적 핸들러 (어노테이션/타입 기반) ---
handlers.add(new ModelAndViewMethodReturnValueHandler());     // ModelAndView 타입
handlers.add(new ModelMethodProcessor());                    // Model 타입
handlers.add(new ViewMethodReturnValueHandler());            // View 타입
handlers.add(new ResponseBodyEmitterReturnValueHandler(...));// ResponseBodyEmitter, SseEmitter
handlers.add(new StreamingResponseBodyReturnValueHandler()); // StreamingResponseBody
handlers.add(new HttpEntityMethodProcessor(...));            // HttpEntity, ResponseEntity
handlers.add(new HttpHeadersReturnValueHandler());           // HttpHeaders 타입
handlers.add(new CallableMethodReturnValueHandler());        // Callable<T>
handlers.add(new DeferredResultMethodReturnValueHandler());  // DeferredResult, ListenableFuture
handlers.add(new AsyncTaskMethodReturnValueHandler(...));    // WebAsyncTask

// --- 어노테이션 기반 ---
handlers.add(new ServletModelAttributeMethodProcessor(false)); // @ModelAttribute
handlers.add(new RequestResponseBodyMethodProcessor(...));    // @ResponseBody ← 중요

// --- 커스텀 (addReturnValueHandlers()로 추가된 것) ---
if (getCustomReturnValueHandlers() != null) {
    handlers.addAll(getCustomReturnValueHandlers());
}

// --- 폴백 (맨 마지막) ---
handlers.add(new ViewNameMethodReturnValueHandler());         // String → view 이름
handlers.add(new MapMethodProcessor());                       // Map → Model 속성
handlers.add(new ServletModelAttributeMethodProcessor(true)); // 나머지 타입 → Model 속성

// 비동기 결과: Reactive 타입 (Reactor Mono/Flux 등)
handlers.addAll(getDefaultReturnValueHandlers_ReactiveTypes());
```

### 3. supportsReturnType() — 핸들러별 처리 기준

```java
// 1. ModelAndViewMethodReturnValueHandler
@Override
public boolean supportsReturnType(MethodParameter returnType) {
    return ModelAndView.class.isAssignableFrom(returnType.getParameterType());
}

// 2. HttpEntityMethodProcessor
@Override
public boolean supportsReturnType(MethodParameter returnType) {
    return (HttpEntity.class.isAssignableFrom(returnType.getParameterType())
            && !RequestEntity.class.isAssignableFrom(returnType.getParameterType()));
    // ResponseEntity 포함 (ResponseEntity extends HttpEntity)
}

// 3. RequestResponseBodyMethodProcessor
@Override
public boolean supportsReturnType(MethodParameter returnType) {
    return (AnnotatedElementUtils.hasAnnotation(returnType.getContainingClass(), ResponseBody.class)
            || returnType.hasMethodAnnotation(ResponseBody.class));
    // 클래스 레벨 @ResponseBody (@RestController 포함) 또는 메서드 레벨 @ResponseBody
}

// 4. ViewNameMethodReturnValueHandler (폴백)
@Override
public boolean supportsReturnType(MethodParameter returnType) {
    Class<?> paramType = returnType.getParameterType();
    return (void.class == paramType || CharSequence.class.isAssignableFrom(paramType));
    // void 또는 String 계열
}
```

### 4. mavContainer.setRequestHandled(true) — View 렌더링 스킵 신호

```java
// @ResponseBody / ResponseEntity 처리 후:
// RequestResponseBodyMethodProcessor.handleReturnValue()
@Override
public void handleReturnValue(@Nullable Object returnValue,
        MethodParameter returnType,
        ModelAndViewContainer mavContainer,
        NativeWebRequest webRequest) throws Exception {
    // 응답 본문 직렬화 완료 후:
    mavContainer.setRequestHandled(true);
    // ← 이 플래그가 true이면 DispatcherServlet.processDispatchResult()에서
    //   View 렌더링(render())을 건너뜀
    // ← @ResponseBody는 View가 필요 없으므로 렌더링 불필요
    // ...
    writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
}

// DispatcherServlet.processDispatchResult()에서:
if (mv != null && !mv.wasCleared()) {
    render(mv, request, response);
    // mavContainer.isRequestHandled() == true이면 mv == null → render 호출 안 됨
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 반환 타입별 처리 핸들러 확인

```java
@Autowired
RequestMappingHandlerAdapter adapter;

@GetMapping("/handler-for")
public String checkHandler() throws Exception {
    // 리플렉션으로 ReturnValueHandlerComposite 접근
    Field field = RequestMappingHandlerAdapter.class
        .getDeclaredField("returnValueHandlers");
    field.setAccessible(true);
    HandlerMethodReturnValueHandlerComposite composite =
        (HandlerMethodReturnValueHandlerComposite) field.get(adapter);

    // 등록된 핸들러 목록
    Field listField = HandlerMethodReturnValueHandlerComposite.class
        .getDeclaredField("returnValueHandlers");
    listField.setAccessible(true);
    List<?> handlers = (List<?>) listField.get(composite);

    return handlers.stream()
        .map(h -> h.getClass().getSimpleName())
        .collect(Collectors.joining("\n"));
}
```

### 실험 2: mavContainer.isRequestHandled() 흐름

```yaml
logging:
  level:
    org.springframework.web.servlet.DispatcherServlet: TRACE
```

```
로그 예시 (@ResponseBody 경우):
  TRACE DispatcherServlet - No view rendering, null ModelAndView
  → mavContainer.isRequestHandled()=true → ModelAndView=null → render 스킵

로그 예시 (String 반환 MVC 경우):
  TRACE DispatcherServlet - Rendering view [ThymeleafView: 'home'] ...
  → ViewNameMethodReturnValueHandler → "home" → ThymeleafViewResolver
```

---

## 🌐 HTTP 레벨 분석

```
반환 타입별 HTTP 응답 흐름:

① @ResponseBody Object 반환:
   handleReturnValue() → setRequestHandled(true)
   → writeWithMessageConverters() → Jackson → JSON 쓰기
   → DispatcherServlet: mv=null, render 스킵
   HTTP/1.1 200 OK, Content-Type: application/json

② ResponseEntity<T> 반환:
   HttpEntityMethodProcessor.handleReturnValue()
   → 상태 코드 설정, 헤더 복사
   → setRequestHandled(true)
   → writeWithMessageConverters()
   HTTP/1.1 201 Created, Location: /users/42

③ String 반환 (View 이름):
   ViewNameMethodReturnValueHandler
   → mavContainer에 viewName="home" 설정
   → setRequestHandled(false) (기본)
   → DispatcherServlet: render("home", ...)
   → ThymeleafViewResolver → HTML 렌더링
   HTTP/1.1 200 OK, Content-Type: text/html

④ void 반환:
   ViewNameMethodReturnValueHandler: void → mavContainer 변경 없음
   → setRequestHandled(false)
   → 직접 응답을 썼으면: 이미 커밋된 응답
   → 안 썼으면: RequestToViewNameTranslator로 URL에서 뷰 이름 추론
```

---

## 🤔 트레이드오프

```
ReturnValueHandler 체인:
  장점  반환 타입 종류 확장 = 새 핸들러 추가만으로 가능
        각 핸들러 독립 테스트
  단점  supportsReturnType() 순서 관리 필요
        커스텀 핸들러가 기본보다 나중에 실행됨 (addReturnValueHandlers)

mavContainer.setRequestHandled(true):
  @ResponseBody / ResponseEntity → true → View 렌더링 스킵
  → REST API에서 불필요한 ViewResolver 호출 없음
  → 명확한 책임 분리: "이 핸들러가 응답을 완전히 처리했다"는 선언
```

---

## 📌 핵심 정리

```
invokeAndHandle() 흐름
  ① invokeForRequest() → 메서드 실행 → returnValue
  ② @ResponseStatus 설정
  ③ null/이미 처리됨 → 조기 종료
  ④ ReturnValueHandlerComposite.handleReturnValue()

기본 핸들러 우선순위 요약
  ModelAndView > HttpEntity/ResponseEntity > @ResponseBody
  > (커스텀) > String/void 폴백

mavContainer.setRequestHandled(true)
  @ResponseBody, ResponseEntity 처리 후 설정
  → DispatcherServlet의 View 렌더링 건너뜀

캐시 구조
  ConcurrentHashMap<MethodParameter, ReturnValueHandler>
  → ArgumentResolverComposite와 동일한 패턴
```

---

## 🤔 생각해볼 문제

**Q1.** `@RestController` 클래스의 메서드가 `String`을 반환하면 어느 핸들러가 처리하는가? `ViewNameMethodReturnValueHandler`인가, `RequestResponseBodyMethodProcessor`인가?

**Q2.** `ModelAndView`를 반환하면서 동시에 `@ResponseBody`가 붙어 있으면 어떤 핸들러가 선택되는가?

**Q3.** 커스텀 ReturnValueHandler를 `addReturnValueHandlers()`로 등록했는데 `ResponseEntity`를 처리하는 핸들러로 만들었습니다. 기본 `HttpEntityMethodProcessor`보다 먼저 실행되게 하려면 어떻게 해야 하는가?

> 💡 **해설**
>
> **Q1.** `RequestResponseBodyMethodProcessor`가 처리합니다. `supportsReturnType()` 체크 시, 이 핸들러는 클래스 레벨 `@ResponseBody`(`@RestController`에 포함)를 먼저 확인하므로 반환 타입이 `String`이어도 `true`를 반환합니다. 등록 순서에서도 `RequestResponseBodyMethodProcessor`(인덱스 ~11)가 `ViewNameMethodReturnValueHandler`(맨 마지막 폴백)보다 앞에 있습니다. 따라서 `@RestController`에서 `String` 반환은 View 이름이 아닌 JSON/텍스트로 직렬화되어 응답 본문에 담깁니다.
>
> **Q2.** 등록 순서상 `ModelAndViewMethodReturnValueHandler`(인덱스 0)가 `RequestResponseBodyMethodProcessor`(인덱스 ~11)보다 앞에 있습니다. `ModelAndViewMethodReturnValueHandler.supportsReturnType()`은 반환 타입이 `ModelAndView`이면 `true`를 반환하므로 이 핸들러가 먼저 선택됩니다. `@ResponseBody`는 무시됩니다. 실제로 `ModelAndView`와 `@ResponseBody`를 같이 쓰는 것은 설계상 모순이므로 권장하지 않습니다.
>
> **Q3.** `addReturnValueHandlers()`로 추가하면 기본 핸들러보다 나중에 실행됩니다. 기본 핸들러보다 먼저 실행하려면 `RequestMappingHandlerAdapter`를 `@Bean`으로 직접 생성하거나, `@PostConstruct` 내에서 `adapter.getReturnValueHandlers()`를 가져온 뒤 리스트 앞에 커스텀 핸들러를 삽입하고 `adapter.setReturnValueHandlers(newList)`로 교체해야 합니다.

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: @ResponseBody — Object → JSON 변환 ➡️](./02-response-body-conversion.md)**

</div>
