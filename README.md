<div align="center">

# 🌐 Spring MVC Deep Dive

**"HTTP 요청이 Controller 메서드에 도달하는 전체 여정"**

<br/>

> *"@RestController를 쓰는 것과, DispatcherServlet이 어떻게 요청을 처리하는지 아는 것은 다르다"*

DispatcherServlet.doDispatch() 한 줄씩 분석부터 HandlerMapping → HandlerAdapter → ViewResolver 전체 체인,  
@RequestBody가 객체가 되는 과정, Exception이 JSON으로 변환되는 메커니즘까지  
**왜 이렇게 설계됐는가** 라는 질문으로 Spring MVC 내부를 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![Java](https://img.shields.io/badge/Java-17%2B-orange?style=flat-square&logo=openjdk)](https://www.java.com)
[![Spring](https://img.shields.io/badge/Spring_MVC-6.x-6DB33F?style=flat-square&logo=spring&logoColor=white)](https://docs.spring.io/spring-framework/reference/web/webmvc.html)
[![Docs](https://img.shields.io/badge/Docs-45개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

Spring MVC에 관한 자료는 넘쳐납니다. 하지만 대부분은 **"어떻게 쓰나"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "`@RestController`를 붙이면 JSON으로 응답됩니다" | `DispatcherServlet` → `HandlerAdapter` → `HttpMessageConverter` 전 과정, Jackson이 객체를 직렬화하는 정확한 시점 |
| "`@RequestBody`로 요청 본문을 받습니다" | `RequestResponseBodyMethodProcessor`가 `MappingJackson2HttpMessageConverter`를 선택해 역직렬화하는 내부 코드 |
| "`@ExceptionHandler`로 예외를 처리합니다" | `ExceptionHandlerExceptionResolver`가 `@ControllerAdvice` Bean에서 핸들러 메서드를 탐색하는 매칭 알고리즘 |
| "`@PathVariable`로 URL 변수를 받습니다" | `UriTemplateVariablesHandlerInterceptor`가 템플릿을 추출하는 시점과 `HandlerMethodArgumentResolver` 체인에서 바인딩되는 과정 |
| "인터셉터로 공통 로직을 처리합니다" | `preHandle → postHandle → afterCompletion` 각 호출 시점과 예외 발생 시 실행 보장 여부, Filter와의 정확한 차이 |
| 이론 나열 | 실행 가능한 코드 + Spring MVC 소스코드 직접 추적 + curl 명령어 실험 + MockMvc 검증 |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Dispatcher](https://img.shields.io/badge/🔹_DispatcherServlet-Servlet과_Front_Controller_패턴-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./dispatcher-servlet/01-front-controller-pattern.md)
[![Mapping](https://img.shields.io/badge/🔹_Request_Mapping-@RequestMapping_처리_과정-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./request-mapping/01-request-mapping-handler-mapping.md)
[![Argument](https://img.shields.io/badge/🔹_Argument_Resolution-HandlerMethodArgumentResolver_체인-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./argument-resolution/01-argument-resolver-chain.md)
[![Response](https://img.shields.io/badge/🔹_Response_Handling-@ResponseBody_처리_과정-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./response-handling/01-return-value-handler-chain.md)
[![Exception](https://img.shields.io/badge/🔹_Exception_Handling-HandlerExceptionResolver_체인-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./exception-handling/01-handler-exception-resolver.md)
[![Interceptor](https://img.shields.io/badge/🔹_Interceptor_&_Filter-Filter_vs_HandlerInterceptor-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./interceptor-filter/01-filter-vs-interceptor.md)
[![Advanced](https://img.shields.io/badge/🔹_Advanced_MVC-Async_Request_Processing-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./advanced-mvc/01-async-request-processing.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: DispatcherServlet Architecture

> **핵심 질문:** HTTP 요청이 들어왔을 때 `DispatcherServlet.doDispatch()`는 정확히 어떤 순서로 무슨 일을 하는가?

<details>
<summary><b>Servlet 초기화부터 View 렌더링까지 완전 분해 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Servlet과 Front Controller 패턴](./dispatcher-servlet/01-front-controller-pattern.md) | Servlet 스펙과 Front Controller 패턴의 관계, `DispatcherServlet`이 왜 단일 진입점인가, 기존 Servlet-per-URL 방식과의 차이 |
| [02. DispatcherServlet 초기화 과정 (onRefresh)](./dispatcher-servlet/02-dispatcher-servlet-init.md) | `onRefresh()` → `initStrategies()` 호출 흐름, `HandlerMapping`, `HandlerAdapter`, `ViewResolver` 등 9가지 컴포넌트 초기화 순서와 default 전략 로딩 |
| [03. WebApplicationContext vs RootApplicationContext](./dispatcher-servlet/03-web-app-context-hierarchy.md) | 두 컨텍스트의 계층 구조, `DispatcherServlet`이 자식 컨텍스트를 갖는 이유, Bean 탐색 위임 방향과 스코프 분리 전략 |
| [04. doDispatch() 완전 분해](./dispatcher-servlet/04-do-dispatch-analysis.md) | `getHandler()` → `getHandlerAdapter()` → `applyPreHandle()` → `ha.handle()` → `applyPostHandle()` → `processDispatchResult()` 각 단계 소스 추적 |
| [05. HandlerMapping 체인 동작](./dispatcher-servlet/05-handler-mapping-chain.md) | 여러 `HandlerMapping`이 순서대로 탐색되는 과정, `RequestMappingHandlerMapping` vs `BeanNameUrlHandlerMapping` 우선순위와 선택 로직 |
| [06. HandlerAdapter의 역할](./dispatcher-servlet/06-handler-adapter-role.md) | `HandlerAdapter` 인터페이스가 존재하는 이유(다양한 핸들러 타입 지원), `RequestMappingHandlerAdapter`가 메서드를 리플렉션으로 호출하는 과정 |
| [07. ViewResolver 메커니즘](./dispatcher-servlet/07-view-resolver-mechanism.md) | `ViewResolver` 체인에서 View를 탐색하는 과정, `ContentNegotiatingViewResolver`의 우선순위 전략, `@ResponseBody`가 있을 때 View 렌더링을 건너뛰는 이유 |

</details>

<br/>

### 🔹 Chapter 2: Request Mapping

> **핵심 질문:** `@RequestMapping`이 붙은 메서드는 어떻게 등록되고, 들어온 HTTP 요청은 어떤 기준으로 매핑되는가?

<details>
<summary><b>@RequestMapping 등록부터 URI 템플릿 추출까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. @RequestMapping 처리 과정](./request-mapping/01-request-mapping-handler-mapping.md) | `RequestMappingHandlerMapping`이 컨텍스트 초기화 시 `@RequestMapping` 메서드를 스캔해 `MappingRegistry`에 등록하는 전 과정 |
| [02. RequestMappingInfo 생성과 매칭](./request-mapping/02-request-mapping-info.md) | `RequestMappingInfo`가 URL·메서드·헤더·파라미터 조건을 어떻게 캡슐화하는가, `getMatchingCondition()`이 요청과 비교하는 알고리즘 |
| [03. Path Pattern 매칭 전략](./request-mapping/03-path-pattern-matching.md) | 레거시 `AntPathMatcher`와 Spring 5.3 이후 기본값인 `PathPatternParser` 비교, `/users/{id}` vs `/users/*` vs `/users/**` 매칭 규칙 |
| [04. HTTP Method 매칭과 OPTIONS 처리](./request-mapping/04-http-method-matching.md) | `GET`/`POST`/`PUT`/`DELETE` 조건 매칭 내부 구조, `OPTIONS` 요청에 대해 `Allow` 헤더를 자동 생성하는 메커니즘, `405 Method Not Allowed` 발생 시점 |
| [05. Content Negotiation — Accept 헤더 처리](./request-mapping/05-content-negotiation.md) | `ContentNegotiationStrategy`가 클라이언트의 `Accept` 헤더를 해석하는 과정, `produces` 조건과 `406 Not Acceptable` 발생 조건 |
| [06. URI Template Variables 추출](./request-mapping/06-uri-template-variables.md) | `PathPattern.matchAndExtract()`가 `{id}` 템플릿에서 값을 추출해 요청 속성에 저장하는 과정, 정규식 제약 `{id:[0-9]+}` 처리 |
| [07. Matrix Variables와 사용 시점](./request-mapping/07-matrix-variables.md) | `;` 구분자로 전달되는 Matrix Variable 스펙, `@MatrixVariable` 처리를 위해 `removeSemicolonContent=false` 설정이 필요한 이유 |

</details>

<br/>

### 🔹 Chapter 3: Argument Resolution

> **핵심 질문:** Controller 메서드 파라미터(`@RequestBody`, `@PathVariable`, `HttpServletRequest` 등)는 어떻게 채워지는가?

<details>
<summary><b>ArgumentResolver 체인부터 Custom Resolver 작성까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. HandlerMethodArgumentResolver 체인](./argument-resolution/01-argument-resolver-chain.md) | `InvocableHandlerMethod`가 파라미터마다 `supportsParameter()`를 순서대로 호출해 적합한 Resolver를 찾는 과정, Resolver 등록 순서가 중요한 이유 |
| [02. @RequestBody와 HttpMessageConverter](./argument-resolution/02-request-body-message-converter.md) | `RequestResponseBodyMethodProcessor`가 `Content-Type` 헤더를 기반으로 `HttpMessageConverter`를 선택해 JSON → Object 역직렬화하는 전 과정 |
| [03. @RequestParam vs @PathVariable 처리](./argument-resolution/03-request-param-path-variable.md) | 두 Resolver의 소스 비교, 타입 변환이 `ConversionService`를 거치는 방식, 기본값·필수 여부 처리 로직 |
| [04. @ModelAttribute 데이터 바인딩 과정](./argument-resolution/04-model-attribute-binding.md) | `ServletModelAttributeMethodProcessor`가 쿼리 파라미터·폼 데이터를 객체 필드에 바인딩하는 `DataBinder` 동작, `@RequestBody`와의 결정적 차이 |
| [05. Validation — @Valid와 BindingResult](./argument-resolution/05-validation-binding-result.md) | `@Valid`가 `javax.validation` 체인을 트리거하는 시점, `BindingResult`를 파라미터에 선언했을 때와 안 했을 때 예외 발생 차이 |
| [06. Servlet API 주입](./argument-resolution/06-servlet-api-injection.md) | `HttpServletRequest`, `HttpServletResponse`, `HttpSession`, `Principal` 등을 파라미터로 주입해주는 `ServletRequestMethodArgumentResolver` 내부 |
| [07. Custom Argument Resolver 작성](./argument-resolution/07-custom-argument-resolver.md) | `HandlerMethodArgumentResolver` 구현 + `WebMvcConfigurer.addArgumentResolvers()` 등록 전 과정, 커스텀 어노테이션 기반 인증 정보 주입 실전 예시 |

</details>

<br/>

### 🔹 Chapter 4: Response Handling

> **핵심 질문:** Controller 메서드의 반환값(`Object`, `ResponseEntity`, `String`)은 어떻게 HTTP 응답으로 변환되는가?

<details>
<summary><b>ReturnValueHandler부터 Content Negotiation까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. HandlerMethodReturnValueHandler 체인](./response-handling/01-return-value-handler-chain.md) | 반환 타입마다 `supportsReturnType()`을 순서대로 확인해 핸들러를 선택하는 과정, `@ResponseBody`와 `ModelAndView` 반환이 다른 핸들러로 처리되는 이유 |
| [02. @ResponseBody — Object → JSON 변환](./response-handling/02-response-body-conversion.md) | `RequestResponseBodyMethodProcessor`가 `Accept` 헤더와 `canWrite()` 체크를 거쳐 `MappingJackson2HttpMessageConverter`를 선택하는 과정 |
| [03. ResponseEntity vs @ResponseStatus](./response-handling/03-response-entity-vs-status.md) | 두 방식의 HTTP 상태 코드·헤더 설정 시점 차이, `ResponseEntity`가 `HttpEntityMethodProcessor`로 처리되는 과정, 선택 기준 |
| [04. HttpMessageConverter 선택 과정](./response-handling/04-message-converter-selection.md) | `canWrite(Class, MediaType)` 체크 순서, 등록된 컨버터 목록과 우선순위, `StringHttpMessageConverter`가 `MappingJackson2HttpMessageConverter`보다 먼저 선택되는 함정 |
| [05. Content Negotiation과 Accept 헤더](./response-handling/05-content-negotiation-accept.md) | `ContentNegotiationManager`가 `Accept` 헤더·URL 확장자·파라미터를 종합해 응답 미디어 타입을 결정하는 알고리즘, 커스텀 미디어 타입 등록 |
| [06. Custom Return Value Handler 작성](./response-handling/06-custom-return-value-handler.md) | `HandlerMethodReturnValueHandler` 구현으로 특수 반환 타입 처리, `WebMvcConfigurer.addReturnValueHandlers()` 등록, 암호화 응답 래핑 실전 예시 |

</details>

<br/>

### 🔹 Chapter 5: Exception Handling

> **핵심 질문:** Controller에서 예외가 발생했을 때 어떤 경로로 처리되고, `@ExceptionHandler`는 어떻게 매핑되는가?

<details>
<summary><b>ExceptionResolver 체인부터 커스텀 에러 응답 설계까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. HandlerExceptionResolver 체인 동작](./exception-handling/01-handler-exception-resolver.md) | `processDispatchResult()`에서 예외가 전달되는 시점, `ExceptionHandlerExceptionResolver` → `ResponseStatusExceptionResolver` → `DefaultHandlerExceptionResolver` 순서와 각 책임 |
| [02. @ExceptionHandler 메서드 매칭 과정](./exception-handling/02-exception-handler-matching.md) | `ExceptionHandlerExceptionResolver`가 발생 예외의 상속 계층을 탐색해 가장 구체적인 `@ExceptionHandler` 메서드를 선택하는 알고리즘 |
| [03. @ControllerAdvice 적용 범위](./exception-handling/03-controller-advice-scope.md) | `basePackages`, `basePackageClasses`, `annotations`, `assignableTypes` 속성으로 적용 대상을 제한하는 방식, 전역 vs 로컬 `@ExceptionHandler` 우선순위 |
| [04. ResponseEntityExceptionHandler 내부 구조](./exception-handling/04-response-entity-exception-handler.md) | Spring MVC 표준 예외(`MethodArgumentNotValidException`, `HttpMessageNotReadableException` 등)를 일괄 처리하는 `ResponseEntityExceptionHandler`, `handleExceptionInternal()` 확장 포인트 |
| [05. RFC 7807 Problem Details 응답 설계](./exception-handling/05-problem-details-response.md) | Spring 6.x `ProblemDetail`과 `ErrorResponse`가 RFC 7807 형식의 JSON 오류 응답을 생성하는 과정, `application/problem+json` Content-Type 처리 |
| [06. Exception 처리 우선순위와 상속](./exception-handling/06-exception-priority-inheritance.md) | 동일 예외에 대해 로컬 `@ExceptionHandler` vs `@ControllerAdvice` 중 무엇이 선택되는가, 예외 상속 계층과 `@ExceptionHandler` 타입 배열 선언 전략 |

</details>

<br/>

### 🔹 Chapter 6: Interceptor & Filter

> **핵심 질문:** Filter와 HandlerInterceptor는 어디서 실행되는가? 그 차이는 무엇이고 언제 어떤 것을 써야 하는가?

<details>
<summary><b>Filter vs Interceptor 차이부터 Async 요청 처리까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Filter vs HandlerInterceptor 차이점](./interceptor-filter/01-filter-vs-interceptor.md) | Servlet 컨테이너 레벨 Filter와 Spring MVC 레벨 Interceptor의 실행 위치 차이, Spring Bean 주입 가능 여부, 요청 객체 래핑 가능 시점 |
| [02. HandlerInterceptor 체인 실행 순서](./interceptor-filter/02-interceptor-chain-order.md) | `HandlerExecutionChain`이 `preHandle` → 핸들러 실행 → `postHandle` → `afterCompletion` 순서를 보장하는 방식, `@Order`와 `Ordered` 인터페이스로 순서 제어 |
| [03. preHandle / postHandle / afterCompletion 호출 시점](./interceptor-filter/03-interceptor-lifecycle.md) | `preHandle` 반환값이 false일 때 체인이 끊기는 메커니즘, 예외 발생 시 `afterCompletion`만 보장되는 이유, 리소스 정리 패턴 |
| [04. CORS 처리 — CorsFilter vs @CrossOrigin](./interceptor-filter/04-cors-handling.md) | Preflight `OPTIONS` 요청이 `CorsProcessor`를 통해 처리되는 과정, `@CrossOrigin` 메타데이터가 `CorsInterceptor`로 변환되는 방식, `CorsFilter` vs `CorsInterceptor` 선택 기준 |
| [05. Custom Interceptor 작성 패턴](./interceptor-filter/05-custom-interceptor-pattern.md) | `HandlerInterceptor` 구현 + `WebMvcConfigurer.addInterceptors()` 등록, URL 패턴 제한, 인증 토큰 검증 인터셉터 실전 예시 |
| [06. Async Request와 Interceptor](./interceptor-filter/06-async-interceptor.md) | `AsyncHandlerInterceptor`의 `afterConcurrentHandlingStarted()` 콜백이 필요한 이유, 비동기 요청에서 `afterCompletion`이 다른 스레드에서 호출되는 타이밍 |

</details>

<br/>

### 🔹 Chapter 7: Advanced MVC Topics

> **핵심 질문:** 파일 업로드, 비동기 처리, SSE, HTTP 캐싱은 Spring MVC 내부에서 어떻게 동작하는가?

<details>
<summary><b>Async 처리부터 WebMvcConfigurer 커스터마이징까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Async Request Processing](./advanced-mvc/01-async-request-processing.md) | `Callable` vs `DeferredResult` vs `CompletableFuture` 반환 시 `AsyncTaskExecutor`가 별도 스레드에서 처리하는 과정, Servlet 비동기 스펙(`startAsync`)과의 연동 |
| [02. Server-Sent Events (SSE)](./advanced-mvc/02-server-sent-events.md) | `SseEmitter`가 Servlet 비동기 모드로 장기 커넥션을 유지하는 방식, `text/event-stream` Content-Type 처리, 연결 끊김 감지와 재연결 전략 |
| [03. File Upload — MultipartResolver](./advanced-mvc/03-file-upload-multipart.md) | `StandardServletMultipartResolver`가 `multipart/form-data`를 파싱해 `MultipartFile` 객체를 생성하는 과정, 파일 크기 제한 설정이 적용되는 시점 |
| [04. Static Resources Handling](./advanced-mvc/04-static-resources-handling.md) | `ResourceHttpRequestHandler`가 클래스패스·파일시스템에서 정적 리소스를 찾는 과정, `ResourceResolver` 체인, WebJars 버전 중립 경로 처리 |
| [05. HTTP Caching — Cache-Control과 ETag](./advanced-mvc/05-http-caching.md) | `ShallowEtagHeaderFilter`가 응답 본문 해시로 ETag를 생성하는 방식, `Cache-Control` 헤더 설정 전략, `If-None-Match` 검증 요청 처리 흐름 |
| [06. WebMvcConfigurer 커스터마이징 가이드](./advanced-mvc/06-web-mvc-configurer.md) | `WebMvcConfigurer`의 각 콜백(`addInterceptors`, `addArgumentResolvers`, `configureMessageConverters` 등) 호출 시점과 `WebMvcAutoConfiguration`이 이를 어떻게 통합하는가 |

</details>

---

## 🗺️ 목적별 학습 경로

<details>
<summary><b>🟢 "DispatcherServlet 면접 질문에 막힌다" — 핵심 흐름 파악 (1주)</b></summary>

<br/>

```
Day 1  Ch1-01  Servlet과 Front Controller 패턴
Day 2  Ch1-02  DispatcherServlet 초기화 과정 (onRefresh)
Day 3  Ch1-04  doDispatch() 완전 분해 ← 핵심
Day 4  Ch2-01  @RequestMapping 처리 과정
Day 5  Ch3-02  @RequestBody와 HttpMessageConverter
Day 6  Ch4-02  @ResponseBody — Object → JSON 변환
Day 7  Ch5-01  HandlerExceptionResolver 체인 동작
```

</details>

<details>
<summary><b>🔵 "ArgumentResolver, ExceptionHandler를 커스터마이징해야 한다" — 확장 포인트 집중 (1주)</b></summary>

<br/>

```
Day 1  Ch3-01  HandlerMethodArgumentResolver 체인
Day 2  Ch3-07  Custom Argument Resolver 작성
Day 3  Ch4-01  HandlerMethodReturnValueHandler 체인
Day 4  Ch4-06  Custom Return Value Handler 작성
Day 5  Ch5-02  @ExceptionHandler 메서드 매칭 과정
Day 6  Ch5-03  @ControllerAdvice 적용 범위
Day 7  Ch7-06  WebMvcConfigurer 커스터마이징 가이드
```

</details>

<details>
<summary><b>🔴 "Spring MVC 소스코드를 직접 읽고 내부를 완전히 이해하고 싶다" — 전체 정복 (7주)</b></summary>

<br/>

```
1주차  Chapter 1 전체 — DispatcherServlet 완전 분해
        → doDispatch() 에 브레이크포인트를 걸고 직접 스택 트레이스 확인

2주차  Chapter 2 전체 — Request Mapping 내부
        → RequestMappingHandlerMapping 소스에서 detectHandlerMethods() 직접 읽기

3주차  Chapter 3 전체 — Argument Resolution
        → @RequestBody 역직렬화 중 HttpMessageConverter.read() 에 브레이크포인트

4주차  Chapter 4 전체 — Response Handling
        → Content Negotiation 흐름을 curl -H "Accept: application/xml" 로 실험

5주차  Chapter 5 전체 — Exception Handling
        → @ControllerAdvice 계층별 우선순위 MockMvc 테스트로 검증

6주차  Chapter 6 전체 — Interceptor & Filter
        → preHandle false 반환 시 체인 동작을 직접 구현해서 확인

7주차  Chapter 7 전체 — Advanced Topics
        → DeferredResult로 실시간 알림 엔드포인트 직접 구현
```

</details>

---

## 📖 각 문서 구성 방식

모든 문서는 동일한 구조로 작성됩니다.

| 섹션 | 설명 |
|------|------|
| 🎯 **핵심 질문** | 이 문서를 읽고 나면 답할 수 있는 질문 |
| 🔍 **왜 이 메커니즘이 존재하는가** | 문제 상황과 설계 배경 |
| 😱 **흔한 오해 또는 실수** | Before — 많은 개발자가 틀리는 방식 |
| ✨ **올바른 이해와 사용** | After — 원리를 알고 난 후의 올바른 접근 |
| 🔬 **내부 동작 원리** | Spring MVC 소스코드 직접 추적 + 다이어그램 |
| 💻 **실험으로 확인하기** | curl 명령어 + MockMvc + 디버거 브레이크포인트 |
| 🌐 **HTTP 레벨 분석** | Request/Response 헤더 직접 분석 |
| 🤔 **트레이드오프** | 이 설계의 장단점, 언제 다른 방법을 택할 것인가 |
| 📌 **핵심 정리** | 한 화면 요약 |
| 🤔 **생각해볼 문제** | 개념을 더 깊이 이해하기 위한 질문 + 해설 |

---

## 🔬 핵심 분석 대상 — doDispatch()

이 레포의 모든 챕터는 아래 한 메서드를 완전히 이해하는 것을 목표로 합니다.

```java
// DispatcherServlet.java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {

    // ① Ch1-05: HandlerMapping 체인에서 Handler + Interceptor 묶음 탐색
    HandlerExecutionChain mappedHandler = getHandler(request);

    // ② Ch1-06: 핸들러 타입에 맞는 HandlerAdapter 선택
    HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

    // ③ Ch6-03: Interceptor preHandle 순서대로 실행, false 반환 시 즉시 종료
    if (!mappedHandler.applyPreHandle(processedRequest, response)) {
        return;
    }

    // ④ Ch3, Ch4: ArgumentResolver → 메서드 호출 → ReturnValueHandler
    ModelAndView mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

    // ⑤ Ch6-03: Interceptor postHandle 역순으로 실행
    mappedHandler.applyPostHandle(processedRequest, response, mv);

    // ⑥ Ch1-07, Ch5: View 렌더링 또는 예외 처리
    processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
}
```

---

## 🔗 선행 학습 레포지토리

| 레포 | 주요 내용 | 연관 챕터 |
|------|----------|-----------|
| [spring-core-deep-dive](https://github.com/dev-book-lab/spring-core-deep-dive) | IoC 컨테이너, DI, AOP, Bean 생명주기, Proxy | Ch1(DispatcherServlet Bean 초기화), Ch6(Interceptor AOP 비교) |
| [spring-boot-internals](https://github.com/dev-book-lab/spring-boot-internals) | Auto-configuration, `WebMvcAutoConfiguration` | Ch1(DispatcherServlet 자동 등록 과정) |
| [spring-data-transaction](https://github.com/dev-book-lab/spring-data-transaction) | JPA, 트랜잭션 | Ch5(예외가 트랜잭션 롤백과 만나는 지점) |

> 💡 MVC는 독립적인 영역이므로 선행 레포 없이도 학습 가능합니다. 단, `AOP / Proxy` 개념이 Ch6(Interceptor vs AOP 비교)에서 도움이 됩니다.

---

## 🙏 Reference

- [Spring Framework Reference — Web MVC](https://docs.spring.io/spring-framework/reference/web/webmvc.html)
- [Spring Framework Source Code (GitHub)](https://github.com/spring-projects/spring-framework)
- [DispatcherServlet Javadoc](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/DispatcherServlet.html)
- [Spring MVC Test (MockMvc)](https://docs.spring.io/spring-framework/reference/testing/spring-mvc-test-framework.html)
- [RFC 7807 — Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc7807)
- [Baeldung Spring MVC Guides](https://www.baeldung.com/spring-mvc)

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by [Dev Book Lab](https://github.com/dev-book-lab)

<br/>

*"@RestController를 쓰는 것과, DispatcherServlet이 어떻게 요청을 처리하는지 아는 것은 다르다"*

</div>
