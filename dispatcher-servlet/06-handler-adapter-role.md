# HandlerAdapter의 역할 — 다양한 핸들러 타입을 하나의 인터페이스로 실행하는 방법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `HandlerAdapter` 인터페이스가 존재하는 이유는 무엇인가?
- `DispatcherServlet`이 `HandlerAdapter.supports(handler)`를 호출하는 시점과 방식은?
- `RequestMappingHandlerAdapter`가 `HandlerMethod`를 실제로 실행하는 과정은?
- `InvocableHandlerMethod`와 `ServletInvocableHandlerMethod`의 역할 차이는?
- `RequestMappingHandlerAdapter`가 초기화 시 등록하는 기본 `ArgumentResolver`와 `ReturnValueHandler` 목록은?
- `ModelAndView`를 반환하는 Controller와 `@ResponseBody`를 반환하는 Controller가 어떻게 같은 `HandlerAdapter`로 처리되는가?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: 핸들러 타입이 제각각이면 DispatcherServlet이 각 타입을 다 알아야 한다

```java
// HandlerAdapter 없이 직접 처리한다면?
protected void doDispatch(...) {
    Object handler = getHandler(request);

    // 타입 확인을 DispatcherServlet이 직접 해야 함
    if (handler instanceof HandlerMethod) {
        HandlerMethod hm = (HandlerMethod) handler;
        // 리플렉션으로 직접 호출...
    } else if (handler instanceof HttpRequestHandler) {
        ((HttpRequestHandler) handler).handleRequest(request, response);
    } else if (handler instanceof Controller) {
        ((Controller) handler).handleRequest(request, response);
    } else if (handler instanceof RouterFunction) {
        // ...
    }
    // 핸들러 타입 추가될 때마다 DispatcherServlet 수정 필요!
}
```

```
해결: Adapter 패턴 적용
  DispatcherServlet은 HandlerAdapter 인터페이스만 알면 됨
  → ha.supports(handler)로 적합한 Adapter 선택
  → ha.handle(request, response, handler) 호출
  → 새 핸들러 타입 추가 시 HandlerAdapter 구현체만 추가
  → DispatcherServlet 코드 변경 없음
```

---

## 😱 흔한 오해 또는 실수

### Before: HandlerAdapter가 Controller를 직접 호출한다

```
❌ 잘못된 이해:
  "RequestMappingHandlerAdapter가 UserController.getUser()를 바로 호출한다"

✅ 실제 호출 체인:
  RequestMappingHandlerAdapter.handle()
    → invokeHandlerMethod()
      → ServletInvocableHandlerMethod.invokeAndHandle()
        → InvocableHandlerMethod.invokeForRequest()
          → getMethodArgumentValues()  ← ArgumentResolver 실행
          → doInvoke(args)             ← 리플렉션으로 메서드 호출
        → returnValueHandlers.handleReturnValue()  ← ReturnValueHandler 실행

  직접 호출이 아닌 리플렉션 기반 (Method.invoke())
  파라미터 준비(ArgumentResolver)와 반환값 처리(ReturnValueHandler)가 모두 여기서 일어남
```

### Before: RequestMappingHandlerAdapter와 RequestMappingHandlerMapping이 같은 것이다

```
❌ 잘못된 이해:
  "이름이 비슷하니까 같이 동작하는 단일 컴포넌트다"

✅ 실제:
  RequestMappingHandlerMapping:
    → URL + 조건으로 Handler(HandlerMethod) 찾는 역할
    → "어떤 메서드가 처리할 것인가"

  RequestMappingHandlerAdapter:
    → HandlerMethod를 실제로 실행하는 역할
    → "HandlerMethod를 어떻게 실행할 것인가"
    → ArgumentResolver, ReturnValueHandler 관리

  분리된 이유:
    각자 독립적으로 교체 가능
    매핑만 커스터마이징 or 실행만 커스터마이징 가능
```

---

## ✨ 올바른 이해와 사용

### After: HandlerAdapter 선택 과정

```java
// DispatcherServlet.getHandlerAdapter()
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
    if (this.handlerAdapters != null) {
        for (HandlerAdapter adapter : this.handlerAdapters) {
            // 각 Adapter가 이 핸들러를 처리할 수 있는지 확인
            if (adapter.supports(handler)) {
                return adapter;  // 첫 번째 지원 Adapter 반환
            }
        }
    }
    throw new ServletException("No adapter for handler [" + handler + "]...");
}
```

```
Spring Boot 기본 HandlerAdapter 목록 (순서):

  RequestMappingHandlerAdapter    : HandlerMethod 지원 (@RequestMapping 계열)
    supports(): handler instanceof HandlerMethod

  HandlerFunctionAdapter          : HandlerFunction 지원 (RouterFunction)
    supports(): handler instanceof HandlerFunction

  HttpRequestHandlerAdapter       : HttpRequestHandler 지원 (정적 리소스 등)
    supports(): handler instanceof HttpRequestHandler

  SimpleControllerHandlerAdapter  : Controller 인터페이스 지원 (레거시)
    supports(): handler instanceof Controller
```

---

## 🔬 내부 동작 원리

### 1. RequestMappingHandlerAdapter — 초기화 시 Resolver/Handler 등록

```java
// RequestMappingHandlerAdapter.afterPropertiesSet()
@Override
public void afterPropertiesSet() {
    // 기본 ControllerAdvice(@ExceptionHandler 포함) Bean 초기화
    initControllerAdviceCache();

    // ArgumentResolver 초기화
    if (this.argumentResolvers == null) {
        List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
        this.argumentResolvers = new HandlerMethodArgumentResolverComposite()
            .addResolvers(resolvers);
    }

    // InitBinder용 ArgumentResolver (WebDataBinder 설정용)
    if (this.initBinderArgumentResolvers == null) {
        List<HandlerMethodArgumentResolver> resolvers = getDefaultInitBinderArgumentResolvers();
        this.initBinderArgumentResolvers = new HandlerMethodArgumentResolverComposite()
            .addResolvers(resolvers);
    }

    // ReturnValueHandler 초기화
    if (this.returnValueHandlers == null) {
        List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
        this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite()
            .addHandlers(handlers);
    }
}
```

### 2. 기본 ArgumentResolver 목록

```java
private List<HandlerMethodArgumentResolver> getDefaultArgumentResolvers() {
    List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>(30);

    // 어노테이션 기반
    resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));  // @RequestParam
    resolvers.add(new RequestParamMapMethodArgumentResolver());
    resolvers.add(new PathVariableMethodArgumentResolver());        // @PathVariable
    resolvers.add(new PathVariableMapMethodArgumentResolver());
    resolvers.add(new MatrixVariableMethodArgumentResolver());      // @MatrixVariable
    resolvers.add(new MatrixVariableMapMethodArgumentResolver());
    resolvers.add(new ServletModelAttributeMethodProcessor(false)); // @ModelAttribute
    resolvers.add(new RequestResponseBodyMethodProcessor(          // @RequestBody
        getMessageConverters(), this.requestResponseBodyAdvice));
    resolvers.add(new RequestPartMethodArgumentResolver(           // @RequestPart
        getMessageConverters(), this.requestResponseBodyAdvice));
    resolvers.add(new RequestHeaderMethodArgumentResolver(getBeanFactory()));  // @RequestHeader
    resolvers.add(new RequestHeaderMapMethodArgumentResolver());
    resolvers.add(new ServletCookieValueMethodArgumentResolver(getBeanFactory())); // @CookieValue
    resolvers.add(new ExpressionValueMethodArgumentResolver(getBeanFactory()));    // @Value (SpEL)
    resolvers.add(new SessionAttributeMethodArgumentResolver());   // @SessionAttribute
    resolvers.add(new RequestAttributeMethodArgumentResolver());   // @RequestAttribute

    // 타입 기반 (어노테이션 없이 타입으로 처리)
    resolvers.add(new ServletRequestMethodArgumentResolver());  // HttpServletRequest, HttpSession 등
    resolvers.add(new ServletResponseMethodArgumentResolver()); // HttpServletResponse
    resolvers.add(new HttpEntityMethodProcessor(               // HttpEntity<T>
        getMessageConverters(), this.requestResponseBodyAdvice));
    resolvers.add(new RedirectAttributesMethodArgumentResolver()); // RedirectAttributes
    resolvers.add(new ModelMethodProcessor());                  // Model
    resolvers.add(new MapMethodProcessor());                    // Map<String, Object>
    resolvers.add(new ErrorsMethodArgumentResolver());          // Errors, BindingResult
    resolvers.add(new SessionStatusMethodArgumentResolver());   // SessionStatus
    resolvers.add(new UriComponentsBuilderMethodArgumentResolver()); // UriComponentsBuilder

    // 커스텀 ArgumentResolver (WebMvcConfigurer.addArgumentResolvers()로 추가한 것)
    if (getCustomArgumentResolvers() != null) {
        resolvers.addAll(getCustomArgumentResolvers());
    }

    // 폴백 (어노테이션 없는 단순 타입 → @RequestParam으로 처리)
    resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));
    // 폴백 (어노테이션 없는 복합 객체 → @ModelAttribute로 처리)
    resolvers.add(new ServletModelAttributeMethodProcessor(true));

    return resolvers;
}
```

### 3. invokeHandlerMethod() — 실제 실행 준비

```java
// RequestMappingHandlerAdapter.invokeHandlerMethod()
@Nullable
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
        HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

    ServletWebRequest webRequest = new ServletWebRequest(request, response);
    try {
        // WebDataBinder 팩토리 (타입 변환, 유효성 검사용)
        WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
        // Model 팩토리 (@ModelAttribute 메서드 처리용)
        ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

        // ServletInvocableHandlerMethod 생성
        ServletInvocableHandlerMethod invocableMethod =
            createInvocableHandlerMethod(handlerMethod);
        // ArgumentResolver, ReturnValueHandler, DataBinder 주입
        if (this.argumentResolvers != null) {
            invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
        }
        if (this.returnValueHandlers != null) {
            invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
        }
        invocableMethod.setDataBinderFactory(binderFactory);
        invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

        // Model + View 컨테이너 생성
        ModelAndViewContainer mavContainer = new ModelAndViewContainer();
        mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
        modelFactory.initModel(webRequest, mavContainer, invocableMethod);
        mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

        // 비동기 요청 처리 설정 (Callable, DeferredResult 등)
        AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
        asyncWebRequest.setTimeout(this.asyncRequestTimeout);
        WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
        asyncManager.setTaskExecutor(this.taskExecutor);
        asyncManager.setAsyncWebRequest(asyncWebRequest);
        asyncManager.registerCallableInterceptors(this.callableInterceptors);
        asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

        if (asyncManager.hasConcurrentResult()) {
            // 비동기 처리 결과가 이미 있는 경우 (재진입)
            Object result = asyncManager.getConcurrentResult();
            mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
            asyncManager.clearConcurrentResult();
            invocableMethod = invocableMethod.wrapConcurrentResult(result);
        }

        // ← 핵심: 실제 메서드 실행
        invocableMethod.invokeAndHandle(webRequest, mavContainer);

        if (asyncManager.isConcurrentHandlingStarted()) {
            return null;  // 비동기 처리 중 → ModelAndView 없음
        }

        // ModelAndViewContainer → ModelAndView 변환
        return getModelAndView(mavContainer, modelFactory, webRequest);
    }
    finally {
        webRequest.requestCompleted();
    }
}
```

### 4. InvocableHandlerMethod.invokeForRequest() — 파라미터 준비 + 리플렉션 호출

```java
// InvocableHandlerMethod.java
@Nullable
public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
        Object... providedArgs) throws Exception {

    // ArgumentResolver 체인으로 메서드 파라미터 준비
    Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);

    if (logger.isTraceEnabled()) {
        logger.trace("Arguments: " + Arrays.toString(args));
    }

    // 리플렉션으로 Controller 메서드 호출
    return doInvoke(args);
}

protected Object[] getMethodArgumentValues(NativeWebRequest request,
        @Nullable ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {

    MethodParameter[] parameters = getMethodParameters();
    if (ObjectUtils.isEmpty(parameters)) {
        return EMPTY_ARGS;
    }

    Object[] args = new Object[parameters.length];
    for (int i = 0; i < parameters.length; i++) {
        MethodParameter parameter = parameters[i];
        parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);

        // 미리 제공된 args 중 타입이 맞는 것 사용
        args[i] = findProvidedArgument(parameter, providedArgs);
        if (args[i] != null) continue;

        // ArgumentResolver 체인에서 처리 가능한 것 찾기
        if (!this.resolvers.supportsParameter(parameter)) {
            throw new IllegalStateException("No suitable resolver for argument " + i + " ...");
        }
        try {
            args[i] = this.resolvers.resolveArgument(parameter, mavContainer,
                request, this.dataBinderFactory);
        } catch (Exception ex) {
            // ...
        }
    }
    return args;
}

protected Object doInvoke(Object... args) throws Exception {
    Method method = getMethod();
    // 접근 제어 우회 (private 메서드도 호출 가능)
    ReflectionUtils.makeAccessible(method);
    try {
        // 리플렉션으로 Controller 메서드 실행
        return method.invoke(getBean(), args);
    } catch (InvocationTargetException ex) {
        // Controller 메서드에서 throw된 예외를 언래핑
        Throwable targetException = ex.getTargetException();
        // ...
        throw new IllegalStateException(...);
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 등록된 ArgumentResolver 목록 출력

```java
@Autowired
RequestMappingHandlerAdapter handlerAdapter;

@GetMapping("/adapter-info")
public List<String> adapterInfo() throws Exception {
    Field field = RequestMappingHandlerAdapter.class
        .getDeclaredField("argumentResolvers");
    field.setAccessible(true);
    HandlerMethodArgumentResolverComposite composite =
        (HandlerMethodArgumentResolverComposite) field.get(handlerAdapter);

    Field resolversField = HandlerMethodArgumentResolverComposite.class
        .getDeclaredField("argumentResolvers");
    resolversField.setAccessible(true);
    List<HandlerMethodArgumentResolver> resolvers =
        (List<HandlerMethodArgumentResolver>) resolversField.get(composite);

    return resolvers.stream()
        .map(r -> r.getClass().getSimpleName())
        .collect(Collectors.toList());
}
```

```json
// 출력 예시:
[
  "RequestParamMethodArgumentResolver",
  "RequestParamMapMethodArgumentResolver",
  "PathVariableMethodArgumentResolver",
  "PathVariableMapMethodArgumentResolver",
  "RequestResponseBodyMethodProcessor",
  "ServletRequestMethodArgumentResolver",
  "ServletResponseMethodArgumentResolver",
  "HttpEntityMethodProcessor",
  ...
]
```

### 실험 2: 브레이크포인트로 ArgumentResolver 선택 과정 추적

```
브레이크포인트 설정:
  InvocableHandlerMethod.getMethodArgumentValues()
  HandlerMethodArgumentResolverComposite.resolveArgument()

@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id, @RequestParam String format)
                                                  ↑                    ↑
                              PathVariableMethodArgumentResolver  RequestParamMethodArgumentResolver
가 각각 선택되는 것을 확인할 수 있음
```

### 실험 3: ModelAndView vs null 반환 비교

```java
// Case 1: View 이름 반환 → ModelAndView 생성됨
@GetMapping("/hello")
public String hello(Model model) {
    model.addAttribute("message", "안녕");
    return "hello";  // ViewResolver가 처리할 View 이름
    // → ModelAndView("hello", {...})
}

// Case 2: @ResponseBody → ModelAndView null
@GetMapping("/api/hello")
@ResponseBody
public Map<String, String> apiHello() {
    return Map.of("message", "안녕");
    // → RequestResponseBodyMethodProcessor가 응답 본문에 직접 씀
    // → mavContainer.setRequestHandled(true)
    // → invokeHandlerMethod() 에서 null 반환
}
```

---

## 🌐 HTTP 레벨 분석

```
ha.handle() 결과에 따른 HTTP 응답 처리 분기:

Case 1: ModelAndView 반환 (View 렌더링)
  ha.handle() → ModelAndView("users/list", {users: [...]})
  → doDispatch: applyPostHandle() → processDispatchResult()
  → render() → InternalResourceViewResolver → JSP 렌더링
  HTTP/1.1 200 OK
  Content-Type: text/html;charset=UTF-8

Case 2: @ResponseBody (응답 본문 직접 작성)
  ha.handle() → null (이미 response에 씀)
  → mavContainer.isRequestHandled() == true
  → doDispatch: applyPostHandle() → processDispatchResult()
  → mv == null → View 렌더링 건너뜀
  HTTP/1.1 200 OK
  Content-Type: application/json

Case 3: ResponseEntity<T>
  ha.handle() → null (HttpEntityMethodProcessor가 응답 완성)
  → doDispatch: 동일하게 mv == null
  HTTP/1.1 201 Created
  Location: /users/1
  Content-Type: application/json
```

---

## 🤔 트레이드오프

```
Adapter 패턴 적용:
  장점  DispatcherServlet ↔ 핸들러 사이의 강한 결합 제거
        새 핸들러 타입 추가 시 DispatcherServlet 수정 불필요
        각 Adapter 독립 테스트 가능
  단점  간단한 기능도 Adapter 계층을 반드시 거쳐야 함
        디버깅 시 호출 스택이 깊어짐 (InvocableHandlerMethod 등 여러 래퍼)

RequestMappingHandlerAdapter의 ArgumentResolver 선택 방식:
  장점  순서 기반의 단순하고 예측 가능한 선택 알고리즘
        커스텀 Resolver 추가가 쉬움 (WebMvcConfigurer.addArgumentResolvers)
  단점  Resolver 수가 많으면 supportsParameter() 순차 호출 오버헤드
        → 실제로는 캐시(HandlerMethodArgumentResolverComposite)로 최적화됨
        → 파라미터 타입별 Resolver를 Map으로 캐시하여 재탐색 없음
```

---

## 📌 핵심 정리

```
HandlerAdapter의 존재 이유
  다양한 핸들러 타입을 DispatcherServlet이 알 필요 없도록 Adapter 패턴 적용
  supports(handler): 처리 가능 여부 확인
  handle(request, response, handler): 실제 실행

RequestMappingHandlerAdapter 실행 체인
  handle()
    → invokeHandlerMethod()
      → ServletInvocableHandlerMethod.invokeAndHandle()
        → InvocableHandlerMethod.invokeForRequest()
          → getMethodArgumentValues() ← ArgumentResolver 선택 + 실행
          → doInvoke() ← Method.invoke() 리플렉션 호출
        → returnValueHandlers.handleReturnValue() ← ReturnValueHandler

기본 ArgumentResolver (30개 이상)
  어노테이션 기반: @RequestParam, @PathVariable, @RequestBody, @ModelAttribute 등
  타입 기반: HttpServletRequest, HttpSession, Model, Map 등
  폴백: 어노테이션 없는 단순 타입 → @RequestParam, 복합 객체 → @ModelAttribute

ModelAndView null 반환 조건
  @ResponseBody, ResponseEntity → ReturnValueHandler가 응답 직접 작성
  → mavContainer.setRequestHandled(true)
  → invokeHandlerMethod()가 null 반환
  → doDispatch에서 View 렌더링 건너뜀
```

---

## 🤔 생각해볼 문제

**Q1.** `HandlerMethodArgumentResolverComposite`는 내부적으로 `argumentResolverCache`라는 `Map<MethodParameter, HandlerMethodArgumentResolver>`를 유지합니다. 이 캐시가 없다면 어떤 성능 문제가 발생하고, 이 캐시의 키가 `MethodParameter`인 이유는 무엇인가?

**Q2.** `WebMvcConfigurer.addArgumentResolvers()`로 커스텀 `ArgumentResolver`를 추가하면 기본 Resolver들보다 앞에 추가되는가, 뒤에 추가되는가? 만약 `@RequestBody`를 처리하는 커스텀 Resolver를 기본 `RequestResponseBodyMethodProcessor`보다 먼저 동작하게 하려면 어떻게 해야 하는가?

**Q3.** Controller 메서드의 파라미터가 `@RequestBody UserDto user`이고, 요청 본문이 비어있는 경우(`Content-Length: 0`) 어떤 일이 일어나는가? `required = false`는 `@RequestBody`에서 어떻게 처리되는가?

> 💡 **해설**
>
> **Q1.** 캐시 없이 매 요청마다 30개 이상의 Resolver에 대해 `supportsParameter()`를 순차 호출하면, 많은 파라미터가 있는 Controller는 수십 번의 Resolver 체인 순회가 발생합니다. `MethodParameter`가 캐시 키인 이유는 파라미터의 동일성을 "어떤 클래스의 어떤 메서드의 몇 번째 파라미터인가"로 식별해야 하기 때문입니다. `MethodParameter`는 `Method` + 인덱스 + 타입을 조합한 동등성을 가지므로 동일한 Controller 메서드의 동일 파라미터는 항상 같은 Resolver를 재사용합니다.
>
> **Q2.** `addArgumentResolvers()`로 추가된 커스텀 Resolver는 기본 Resolver들 **뒤에** 추가됩니다(`getDefaultArgumentResolvers()` 목록 다음). 따라서 기본 Resolver가 먼저 처리됩니다. `@RequestBody`를 가로채려면 `RequestMappingHandlerAdapter.setArgumentResolvers()`로 전체 Resolver 목록을 직접 지정하거나, `RequestMappingHandlerAdapter`를 커스텀 설정으로 빈 등록해 커스텀 Resolver를 맨 앞에 두어야 합니다. 혹은 `@ControllerAdvice` + `RequestBodyAdvice`를 구현하는 방법이 더 안전합니다.
>
> **Q3.** `required = true`(기본값)이면 `HttpMessageNotReadableException`이 발생합니다. 이는 `RequestResponseBodyMethodProcessor`가 `HttpMessageConverter.read()`를 호출할 때 빈 본문을 파싱할 수 없어 예외를 던지기 때문입니다. `@RequestBody(required = false)`로 설정하면 본문이 비어있을 때 `null`이 파라미터에 바인딩됩니다. 내부적으로는 `EmptyBodyCheckingHttpInputMessage`가 스트림의 비어있음을 확인하고, `required=false`이면 `null`을 반환하고, `required=true`이면 예외를 throw합니다.

---

<div align="center">

**[⬅️ 이전: HandlerMapping 체인 동작](./05-handler-mapping-chain.md)** | **[홈으로 🏠](../README.md)** | **[다음: ViewResolver 메커니즘 ➡️](./07-view-resolver-mechanism.md)**

</div>
