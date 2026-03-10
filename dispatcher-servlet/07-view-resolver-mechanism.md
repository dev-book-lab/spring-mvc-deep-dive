# ViewResolver 메커니즘 — View 이름이 실제 응답이 되는 과정

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `ViewResolver` 인터페이스의 유일한 메서드 `resolveViewName()`은 어떤 과정을 거치는가?
- `ContentNegotiatingViewResolver`가 다른 ViewResolver들과 어떻게 다른가?
- `@ResponseBody`가 있을 때 View 렌더링이 건너뛰어지는 정확한 메커니즘은?
- `InternalResourceViewResolver`와 `ThymeleafViewResolver`의 View 탐색 방식 차이는?
- ViewResolver 체인에서 우선순위는 어떻게 결정되는가?
- View 캐싱은 어떻게 동작하고, 개발 환경에서 왜 캐시를 꺼야 하는가?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: 다양한 View 기술을 DispatcherServlet이 모두 알 수 없다

```
View 기술의 다양성:
  JSP (InternalResourceView)
  Thymeleaf (ThymeleafView)
  FreeMarker (FreeMarkerView)
  Mustache (MustacheView)
  JSON 직렬화 (@ResponseBody, Jackson)
  XML 직렬화 (JAXB)
  PDF, Excel 생성 (Apache POI 기반 View)
  Redirect (RedirectView)
  Fragment (Spring 6.x partial rendering)

DispatcherServlet이 각 View 기술을 직접 알면:
  → 기술 추가/변경 시 DispatcherServlet 수정 필요
  → View 기술과 MVC 프레임워크의 강한 결합

해결: Strategy 패턴
  View 이름(String) → 실제 View 객체로 변환하는 책임을 ViewResolver에게 위임
  → DispatcherServlet은 View.render() 만 호출
  → 새 View 기술 = 새 ViewResolver 구현체만 등록
```

---

## 😱 흔한 오해 또는 실수

### Before: ViewResolver가 하나만 있다

```java
// ❌ 잘못된 이해
// "ViewResolver는 InternalResourceViewResolver 하나다"

// ✅ 실제 Spring Boot + Thymeleaf 설정 시 ViewResolver 목록:
// (order 순)
//  order=-2147483648  ContentNegotiatingViewResolver  ← 모든 ViewResolver 조율자
//  order=-1           BeanNameViewResolver             ← Bean 이름으로 View 탐색
//  order=1            ThymeleafViewResolver            ← .html 파일 처리
//  order=2147483647   InternalResourceViewResolver     ← JSP 폴백 (항상 match)
//
// InternalResourceViewResolver는 "항상 match"이므로 마지막에 두어야 함!
// → 순서를 잘못 설정하면 Thymeleaf 템플릿이 있어도 JSP를 찾으려 시도
```

### Before: @ResponseBody가 있으면 ViewResolver가 null을 반환한다

```
❌ 잘못된 이해:
  "@ResponseBody 반환 시 ViewResolver가 null을 반환해서 View 렌더링이 안 된다"

✅ 실제:
  @ResponseBody 처리는 ViewResolver와 무관
  → ReturnValueHandler(RequestResponseBodyMethodProcessor)가 응답 본문을 직접 write
  → mavContainer.setRequestHandled(true) 설정
  → invokeHandlerMethod()에서 getModelAndView() 호출 시 null 반환
  → doDispatch()에서 mv == null → processDispatchResult()에서 render() 자체를 호출 안 함
  → ViewResolver는 아예 호출되지 않음
```

---

## ✨ 올바른 이해와 사용

### After: View 렌더링 전체 흐름

```
doDispatch()
  → ha.handle() → ModelAndView("users/list", {users:[...]})
  → processDispatchResult()
    → render(mv, request, response)
      │
      ├─ mv.getView() != null → View 직접 사용 (ViewResolver 생략)
      │
      └─ mv.getViewName() = "users/list"
          → resolveViewName("users/list", locale, request)
            │
            ├─ ViewResolver 1: ContentNegotiatingViewResolver (order=-MAX)
            │   → 다른 ViewResolver들에게 위임 후 Accept 헤더로 최적 선택
            │
            ├─ ViewResolver 2: ThymeleafViewResolver (order=1)
            │   → templates/users/list.html 파일 확인
            │   → 있으면 ThymeleafView 반환
            │
            └─ ViewResolver N: InternalResourceViewResolver (order=MAX)
                → /WEB-INF/views/users/list.jsp 경로 구성
                → 항상 InternalResourceView 반환 (파일 없어도!)
            │
          → view.render(model, request, response)
            → 템플릿 엔진으로 HTML 생성 → response.getWriter()에 write
```

---

## 🔬 내부 동작 원리

### 1. DispatcherServlet.render() — ViewResolver 체인 호출

```java
// DispatcherServlet.java
protected void render(ModelAndView mv, HttpServletRequest request,
        HttpServletResponse response) throws Exception {

    // Locale 결정 (LocaleResolver 사용)
    Locale locale = (this.localeResolver != null
        ? this.localeResolver.resolveLocale(request)
        : request.getLocale());
    response.setLocale(locale);

    View view;
    String viewName = mv.getViewName();

    if (viewName != null) {
        // ViewResolver 체인으로 View 이름 → View 객체 변환
        view = resolveViewName(viewName, mv.getModelInternal(), locale, request);
        if (view == null) {
            throw new ServletException("Could not resolve view with name '" + mv.getViewName() + "'");
        }
    } else {
        // ModelAndView에 View 객체가 직접 있는 경우
        view = mv.getView();
    }

    // View.render() 실행
    try {
        if (mv.getStatus() != null) {
            request.setAttribute(View.RESPONSE_STATUS_ATTRIBUTE, mv.getStatus());
            response.setStatus(mv.getStatus().value());
        }
        view.render(mv.getModelInternal(), request, response);
    } catch (Exception ex) {
        throw ex;
    }
}

// ViewResolver 체인 탐색
@Nullable
protected View resolveViewName(String viewName, @Nullable Map<String, Object> model,
        Locale locale, HttpServletRequest request) throws Exception {

    if (this.viewResolvers != null) {
        for (ViewResolver viewResolver : this.viewResolvers) {
            // 각 ViewResolver에게 View 탐색 위임
            View view = viewResolver.resolveViewName(viewName, locale);
            if (view != null) {
                return view;  // 첫 번째 non-null View 반환
            }
        }
    }
    return null;
}
```

### 2. AbstractCachingViewResolver — View 캐싱

```java
// AbstractCachingViewResolver.java (InternalResourceViewResolver, ThymeleafViewResolver 등의 부모)
@Override
@Nullable
public View resolveViewName(String viewName, Locale locale) throws Exception {
    if (!isCache()) {
        // 캐시 비활성화 시 매번 새로 생성 (개발 환경)
        return createView(viewName, locale);
    }

    Object cacheKey = getCacheKey(viewName, locale);
    View view = this.viewAccessCache.get(cacheKey);  // 빠른 접근 캐시

    if (view == null) {
        synchronized (this.viewCreationCache) {
            view = this.viewCreationCache.get(cacheKey);  // 생성 락
            if (view == null) {
                view = createView(viewName, locale);  // 실제 View 생성
                // UNRESOLVED_VIEW = 탐색 실패 마커 (null 방지)
                if (view == null && this.cacheUnresolved) {
                    view = UNRESOLVED_VIEW;
                }
                if (view != null && this.cacheFilter.filter(view, viewName, locale)) {
                    this.viewCreationCache.put(cacheKey, view);
                    this.viewAccessCache.put(cacheKey, view);
                }
            }
        }
    }

    return (view != UNRESOLVED_VIEW ? view : null);
}
```

### 3. ContentNegotiatingViewResolver — Accept 헤더 기반 협상

```java
// ContentNegotiatingViewResolver.java
@Override
@Nullable
public View resolveViewName(String viewName, Locale locale) throws Exception {
    RequestAttributes attrs = RequestContextHolder.getRequestAttributes();
    HttpServletRequest request = ((ServletRequestAttributes) attrs).getRequest();

    // 1. 요청에서 원하는 미디어 타입 목록 파악
    //    Accept 헤더, URL 확장자, 파라미터 등을 종합
    List<MediaType> requestedMediaTypes = getMediaTypes(request);

    if (requestedMediaTypes != null) {
        // 2. 모든 ViewResolver에게 후보 View 수집
        List<View> candidateViews = getCandidateViews(viewName, locale, requestedMediaTypes);

        // 3. 요청 미디어 타입과 View가 생성할 수 있는 미디어 타입 비교
        View bestView = getBestView(candidateViews, requestedMediaTypes, attrs);
        if (bestView != null) {
            return bestView;
        }
    }

    // ...
    return null;
}

private List<View> getCandidateViews(String viewName, Locale locale,
        List<MediaType> requestedMediaTypes) throws Exception {
    List<View> candidateViews = new ArrayList<>();

    if (this.viewResolvers != null) {
        // 등록된 모든 ViewResolver에게 View 요청
        for (ViewResolver viewResolver : this.viewResolvers) {
            View view = viewResolver.resolveViewName(viewName, locale);
            if (view != null) {
                candidateViews.add(view);
            }
        }
    }

    // 미디어 타입별 기본 View도 후보에 추가
    for (MediaType requestedMediaType : requestedMediaTypes) {
        for (View defaultView : this.defaultViews) {
            if (defaultView.getContentType() != null
                    && requestedMediaType.isCompatibleWith(
                        MediaType.parseMediaType(defaultView.getContentType()))) {
                candidateViews.add(defaultView);
            }
        }
    }
    return candidateViews;
}
```

### 4. InternalResourceViewResolver — "항상 match"의 의미

```java
// InternalResourceViewResolver.java
@Override
protected View loadView(String viewName, Locale locale) throws Exception {
    // AbstractUrlBasedView 생성
    AbstractUrlBasedView view = buildView(viewName);
    // → prefix + viewName + suffix 로 URL 구성
    // → 예: /WEB-INF/views/users/list.jsp

    View result = applyLifecycleMethods(viewName, view);
    // 파일 존재 여부 확인하지 않음!
    // → InternalResourceView는 항상 반환됨
    return (view.checkResource(locale) ? result : null);
    // checkResource()도 기본적으로 true 반환
    // → "파일이 없어도 View 반환 → 실제 렌더링 시 FileNotFoundException"
}
```

```
InternalResourceViewResolver를 마지막에 두어야 하는 이유:
  → resolveViewName()이 항상 non-null 반환
  → ViewResolver 체인에서 InternalResourceViewResolver 이후 ViewResolver는 절대 실행 안 됨
  → Thymeleaf, FreeMarker 등이 InternalResourceViewResolver 뒤에 있으면 무시됨

Spring Boot에서 Thymeleaf 사용 시:
  → ThymeleafViewResolver (order=1)
  → InternalResourceViewResolver (order=MAX)
  → Thymeleaf가 먼저 탐색, 템플릿 없으면 InternalResourceViewResolver가 처리
```

---

## 💻 실험으로 확인하기

### 실험 1: ViewResolver 목록과 순서 확인

```java
@Autowired
DispatcherServlet dispatcherServlet;

@GetMapping("/view-resolvers")
public List<String> viewResolvers() throws Exception {
    Field field = DispatcherServlet.class.getDeclaredField("viewResolvers");
    field.setAccessible(true);
    List<ViewResolver> resolvers = (List<ViewResolver>) field.get(dispatcherServlet);
    return resolvers.stream()
        .map(r -> r.getClass().getSimpleName()
            + (r instanceof Ordered o ? " (order=" + o.getOrder() + ")" : ""))
        .collect(Collectors.toList());
}
```

### 실험 2: Content Negotiation — Accept 헤더로 응답 형식 전환

```java
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return userService.findById(id);
}
// 이 메서드 하나로 Accept 헤더에 따라 다른 응답 가능:
```

```bash
# JSON 요청
curl -H "Accept: application/json" http://localhost:8080/users/1
# HTTP/1.1 200 OK
# Content-Type: application/json
# {"id":1,"name":"홍길동"}

# XML 요청 (jackson-dataformat-xml 의존성 필요)
curl -H "Accept: application/xml" http://localhost:8080/users/1
# HTTP/1.1 200 OK
# Content-Type: application/xml
# <User><id>1</id><name>홍길동</name></User>

# HTML 요청 (Thymeleaf + View 있을 경우)
curl -H "Accept: text/html" http://localhost:8080/users/1
# → ThymeleafViewResolver가 users/1 View 탐색
```

### 실험 3: View 캐시 비활성화 (개발 환경)

```yaml
# Thymeleaf 캐시 비활성화
spring:
  thymeleaf:
    cache: false  # AbstractCachingViewResolver.setCache(false)

# → 매 요청마다 templates/*.html 파일을 다시 읽음
# → 서버 재시작 없이 템플릿 변경 확인 가능
```

### 실험 4: RedirectView 동작 확인

```java
@GetMapping("/old-path")
public RedirectView redirect() {
    return new RedirectView("/new-path");
    // → ModelAndView에 RedirectView가 직접 설정됨
    // → render() → RedirectView.renderMergedOutputModel()
    // → response.sendRedirect("/new-path")
}
```

```bash
curl -v http://localhost:8080/old-path
# HTTP/1.1 302 Found
# Location: http://localhost:8080/new-path
```

---

## 🌐 HTTP 레벨 분석

```
View 타입별 HTTP 응답 헤더:

ThymeleafView (HTML 렌더링):
  HTTP/1.1 200 OK
  Content-Type: text/html;charset=UTF-8
  Content-Length: 1234
  (응답 본문: HTML)

@ResponseBody + JSON (ViewResolver 미사용):
  HTTP/1.1 200 OK
  Content-Type: application/json
  (응답 본문: JSON)

RedirectView:
  HTTP/1.1 302 Found
  Location: http://host/new-path
  (응답 본문 없음)

InternalResourceView (JSP Forward):
  HTTP/1.1 200 OK
  Content-Type: text/html;charset=UTF-8
  (서버 내부 forward → 클라이언트는 URL 변화 모름)

View 이름에 "redirect:" 접두사:
  return "redirect:/new-path"
  → UrlBasedViewResolver가 RedirectView 생성
  → 302 응답

View 이름에 "forward:" 접두사:
  return "forward:/internal-path"
  → InternalResourceView로 서버 내부 포워드
  → 200 응답 (클라이언트 투명)
```

---

## 🤔 트레이드오프

```
ViewResolver 체인 방식:
  장점  View 기술을 플러그인처럼 교체 가능
        여러 View 기술 공존 가능 (HTML + JSON + PDF)
        ContentNegotiating으로 하나의 핸들러가 다양한 형식 지원
  단점  ViewResolver 순서 설정 실수 시 예기치 않은 동작
        InternalResourceViewResolver를 잘못 배치하면 다른 모든 Resolver 무시

View 캐싱:
  장점  운영 환경에서 파일 시스템 접근 최소화 → 성능 향상
        Thymeleaf 기본: cache=true
  단점  개발 환경에서 템플릿 수정 후 서버 재시작 필요
        → Spring Boot DevTools가 자동으로 cache=false로 설정해줌

@ResponseBody vs View 기반 응답:
  @ResponseBody: HttpMessageConverter 직접 사용
    → ViewResolver, View.render() 완전 건너뜀
    → REST API에 적합, 성능 오버헤드 없음
  View 기반: ViewResolver → View.render()
    → 서버 사이드 렌더링에 적합
    → Model 데이터를 템플릿에 바인딩
```

---

## 📌 핵심 정리

```
ViewResolver 체인 동작
  DispatcherServlet.render()
    → resolveViewName() → ViewResolver 순서대로 탐색
    → 첫 번째 non-null View 반환
    → view.render(model, request, response)

View 렌더링이 건너뛰어지는 조건
  mv == null: @ResponseBody, ResponseEntity
  mv.wasCleared(): 응답이 완료된 것으로 표시
  → ViewResolver 자체가 호출되지 않음

ContentNegotiatingViewResolver (order=최우선)
  → 모든 ViewResolver에서 후보 수집
  → Accept 헤더와 View가 지원하는 ContentType 비교
  → 가장 적합한 View 선택

InternalResourceViewResolver 주의사항
  → checkResource() 기본 true → 항상 non-null 반환
  → 마지막 순서(order=MAX)에 배치 필수
  → 앞에 오면 이후 ViewResolver 모두 무시됨

View 캐시
  AbstractCachingViewResolver.viewAccessCache / viewCreationCache
  → 운영: cache=true (성능)
  → 개발: cache=false (실시간 반영)
  → Spring Boot DevTools: 자동 cache=false
```

---

## 🤔 생각해볼 문제

**Q1.** `return "redirect:/users"` 와 `return new RedirectView("/users")` 는 동일하게 동작하는가? 내부 처리 경로가 같은가?

**Q2.** `ContentNegotiatingViewResolver`가 등록되어 있을 때, 브라우저가 `Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8`을 보내고 서버에 ThymeleafViewResolver와 JSON 응답 기본 View가 있다면 어떤 View가 선택되는가?

**Q3.** Thymeleaf를 사용하는 환경에서 Controller가 `"error"` 라는 View 이름을 반환했는데, `templates/error.html`이 없다면 어떻게 되는가? Spring Boot의 에러 처리 메커니즘과 어떤 관계가 있는가?

> 💡 **해설**
>
> **Q1.** 동일하지 않습니다. `return "redirect:/users"` 는 `AbstractView` 기반 ViewResolver(예: `UrlBasedViewResolver` 또는 `InternalResourceViewResolver`)가 View 이름에서 `redirect:` 접두사를 감지해 `RedirectView`를 생성합니다. 반면 `return new RedirectView("/users")`는 `HandlerMethodReturnValueHandler` 중 `ViewMethodReturnValueHandler`가 이미 생성된 `RedirectView`를 `ModelAndView`에 직접 넣습니다. 결과적으로 `302` 응답을 생성한다는 면에서 동일하지만, View 이름 처리 경로가 다릅니다. `"redirect:"` 방식은 문자열 접두사를 파싱하는 과정이 추가되고, `RedirectView` 방식은 ViewResolver 탐색 없이 직접 사용됩니다.
>
> **Q2.** 브라우저의 `Accept` 헤더에서 `text/html`의 품질 계수(q)가 가장 높으므로(기본 1.0), `ContentNegotiatingViewResolver`는 `text/html`과 호환되는 View를 우선 선택합니다. ThymeleafView는 `text/html`을 생성할 수 있으므로 ThymeleafViewResolver의 View가 선택됩니다. `*/*;q=0.8`은 JSON 기본 View와도 호환되지만 품질 계수가 낮아 우선순위가 낮습니다. 따라서 Thymeleaf HTML 응답이 반환됩니다.
>
> **Q3.** `ThymeleafViewResolver.resolveViewName("error", locale)`를 호출하면 `templates/error.html`이 없으므로 `null`을 반환합니다. 그러면 다음 ViewResolver로 넘어가고, 최종적으로 모든 ViewResolver가 `null`을 반환하면 `DispatcherServlet.render()`에서 `ServletException("Could not resolve view with name 'error'")`이 발생합니다. 그러나 Spring Boot는 `BasicErrorController`를 통해 `/error` 경로를 자체 처리하고, 기본 WhitelabelErrorPage(`/error.html` 또는 Whitelabel 페이지)를 제공합니다. Spring Boot의 `ErrorMvcAutoConfiguration`이 `/error`를 처리하는 별도의 컨트롤러와 View를 등록해두기 때문에, 직접 `"error"` View 이름을 반환하는 것과 Spring Boot의 에러 처리는 별개의 경로입니다.

---

<div align="center">

**[⬅️ 이전: HandlerAdapter의 역할](./06-handler-adapter-role.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 2 — @RequestMapping 처리 과정 ➡️](../request-mapping/01-request-mapping-handler-mapping.md)**

</div>
