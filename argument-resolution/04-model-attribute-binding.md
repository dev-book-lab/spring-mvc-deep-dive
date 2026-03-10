# @ModelAttribute 데이터 바인딩 — DataBinder가 폼 데이터를 객체 필드에 채우는 방법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `ServletModelAttributeMethodProcessor`가 객체를 생성하고 필드를 채우는 단계별 과정은?
- `WebDataBinder.bind()`는 폼 파라미터를 어떤 기준으로 객체 필드에 연결하는가?
- `@RequestBody`와 `@ModelAttribute`의 결정적 차이는 무엇인가?
- `@ModelAttribute` 없이 POJO를 파라미터로 선언하면 어떻게 동작하는가?
- `allowedFields`, `disallowedFields`로 바인딩 대상 필드를 제한하는 방법과 이유는?
- 중첩 객체(nested object)와 컬렉션 필드 바인딩은 어떻게 동작하는가?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: 폼 데이터를 필드 하나씩 수동으로 채우는 코드는 비효율적이다

```java
// Bad: 파라미터 하나씩 수동 처리
@PostMapping("/search")
public String search(
    @RequestParam String keyword,
    @RequestParam int page,
    @RequestParam int size,
    @RequestParam String sort,
    @RequestParam String direction) {
    SearchForm form = new SearchForm();
    form.setKeyword(keyword);
    form.setPage(page);
    // ... 10개 이상 필드면 더 복잡
}

// Good: @ModelAttribute로 자동 바인딩
@PostMapping("/search")
public String search(@ModelAttribute SearchForm form) {
    // keyword, page, size, sort, direction 자동으로 SearchForm에 바인딩
}

해결:
  DataBinder가 요청 파라미터 이름과 객체 필드명 매핑
  → 타입 변환 + 설정값 주입 + 검증까지 한 번에
  → 파라미터 추가/삭제 시 컨트롤러 코드 변경 없음
```

---

## 😱 흔한 오해 또는 실수

### Before: @ModelAttribute는 @RequestBody와 동일하게 JSON을 처리할 수 있다

```java
// ❌ 잘못된 이해
// "Content-Type: application/json 요청에서 @ModelAttribute를 쓸 수 있다"

@PostMapping("/users")
public User create(@ModelAttribute UserDto dto) { ... }

// curl -X POST -H "Content-Type: application/json" -d '{"name":"홍길동"}' /users
// → name = null  (JSON 본문이 바인딩되지 않음!)

// ✅ 실제:
// @ModelAttribute는 HttpServletRequest.getParameter() 기반
// → 쿼리 파라미터 + application/x-www-form-urlencoded 본문만 처리
// → application/json 본문은 getParameter()로 읽을 수 없음
// → JSON 본문에는 반드시 @RequestBody 사용
```

### Before: @ModelAttribute가 없으면 POJO 파라미터는 바인딩되지 않는다

```java
// ❌ 잘못된 이해
@GetMapping("/search")
public String search(SearchForm form) {  // @ModelAttribute 생략
    // "form의 필드들이 비어있다"
}

// ✅ 실제:
// 폴백 ServletModelAttributeMethodProcessor(useDefaultResolution=true)가
// 단순 타입이 아닌 POJO를 @ModelAttribute처럼 처리
// → SearchForm form = new SearchForm()
// → DataBinder로 쿼리 파라미터 바인딩
// @ModelAttribute 어노테이션 없어도 동작 (암묵적 @ModelAttribute)
```

---

## ✨ 올바른 이해와 사용

### After: @ModelAttribute 처리 전체 흐름

```
GET /search?keyword=spring&page=2&size=20

① ServletModelAttributeMethodProcessor.resolveArgument()
   → 객체 생성: createAttribute()
     → 기본 생성자로 SearchForm 인스턴스 생성 (또는 @ModelAttribute("name")으로 Model에서 조회)
   → WebDataBinder 생성: binderFactory.createBinder(request, attr, name)

② WebDataBinder.bind(PropertyValues pvs)
   → ServletRequestDataBinder.bind(request)
     → request.getParameterMap() → {keyword:["spring"], page:["2"], size:["20"]}
     → PropertyValues 생성
   → BeanWrapper로 각 필드에 값 설정:
     keyword → "spring" (String → String)
     page    → 2       (String → int, ConversionService)
     size    → 20      (String → int, ConversionService)

③ @Valid 있으면 검증
④ BindingResult에 바인딩/검증 결과 저장

⑤ SearchForm{keyword="spring", page=2, size=20} 반환
```

---

## 🔬 내부 동작 원리

### 1. ServletModelAttributeMethodProcessor.resolveArgument()

```java
// ModelAttributeMethodProcessor.resolveArgument() (부모 클래스)
@Override
@Nullable
public final Object resolveArgument(MethodParameter parameter, ...) throws Exception {

    String name = ModelFactory.getNameForParameter(parameter);
    // → @ModelAttribute("form") 있으면 "form"
    // → 없으면 타입명의 camelCase: "SearchForm" → "searchForm"

    // ① Model에 이미 있으면 재사용 (다른 메서드에서 미리 @ModelAttribute로 추가된 경우)
    Object attribute = (mavContainer.containsAttribute(name) ?
        mavContainer.getModel().get(name) : createAttribute(name, parameter, binderFactory, webRequest));

    // ② WebDataBinder 생성
    WebDataBinder binder = binderFactory.createBinder(webRequest, attribute, name);

    // ③ 실제 바인딩 (파라미터 → 필드)
    if (!mavContainer.isBindingDisabled(name)) {
        bindRequestParameters(binder, webRequest);
    }

    // ④ @Valid / @Validated 검증
    validateIfApplicable(binder, parameter);
    if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
        throw new BindException(binder.getBindingResult());
        // BindingResult 파라미터가 없으면 400 Bad Request
    }

    // ⑤ Model에 attribute와 BindingResult 저장
    Map<String, Object> bindingResultModel = binder.getBindingResult().getModel();
    mavContainer.removeAttributes(bindingResultModel);
    mavContainer.addAllAttributes(bindingResultModel);

    return binder.convertIfNecessary(binder.getTarget(), parameter.getParameterType(), parameter);
}
```

### 2. createAttribute() — 객체 생성 방법

```java
// ModelAttributeMethodProcessor.createAttribute()
protected Object createAttribute(String attributeName, MethodParameter parameter,
        WebDataBinderFactory binderFactory, NativeWebRequest request) throws Exception {

    // ① 생성자 파라미터에서 @ModelAttribute 값으로 직접 주입 가능 (Spring 5.1+)
    MethodParameter nestedParameter = parameter.nestedIfOptional();
    Class<?> clazz = nestedParameter.getNestedParameterType();

    Constructor<?> ctor = BeanUtils.getResolvableConstructor(clazz);
    // → 기본 생성자가 있으면 그것 사용
    // → 기본 생성자 없고 생성자가 하나뿐이면 그 생성자 사용 (파라미터는 요청값에서 채움)

    Object attribute = constructAttribute(ctor, attributeName, parameter, binderFactory, request);
    // 기본 생성자: new SearchForm() → 빈 객체 생성 후 DataBinder로 채움
    // 유일한 생성자: new SearchForm(keyword, page) → 생성자 파라미터도 요청값으로 채움
    return attribute;
}
```

### 3. DataBinder.bind() — 필드에 값 설정

```java
// ServletRequestDataBinder.bind()
@Override
public void bind(ServletRequest request) {
    // 요청 파라미터 전체를 PropertyValues로 변환
    MutablePropertyValues mpvs = new ServletRequestParameterPropertyValues(request);
    // → request.getParameterMap() → {keyword:["spring"], page:["2"]}

    // multipart 처리 (파일 업로드)
    MultipartRequest multipartRequest = WebUtils.getNativeRequest(request, MultipartRequest.class);
    if (multipartRequest != null) {
        bindMultipart(multipartRequest.getMultiFileMap(), mpvs);
    }

    // allowedFields/disallowedFields 필터링
    addBindValues(mpvs, request);

    // 실제 바인딩
    doBind(mpvs);
}

// BeanWrapper를 통해 각 필드에 값 설정
protected void doBind(MutablePropertyValues mpvs) {
    // allowedFields: 허용된 필드만 바인딩
    // disallowedFields: 금지된 필드 제외 후 바인딩
    checkAllowedFields(mpvs);
    checkRequiredFields(mpvs);
    applyPropertyValues(mpvs);
    // → BeanWrapper.setPropertyValue("keyword", "spring")
    // → ConversionService 통해 타입 변환 포함
}
```

### 4. 중첩 객체와 컬렉션 바인딩

```java
public class OrderForm {
    private String title;
    private AddressForm address;        // 중첩 객체
    private List<ItemForm> items;       // 컬렉션
    private Map<String, String> extras; // Map
}

// 중첩 객체 바인딩:
// GET /order?title=주문&address.city=서울&address.street=강남대로
// → address.city=서울 → OrderForm.address.city = "서울"
// BeanWrapper의 점(.) 구문 지원

// 컬렉션 바인딩 (HTML form 기준):
// GET /order?items[0].name=A&items[0].qty=2&items[1].name=B&items[1].qty=1
// → items[0].name=A → OrderForm.items.get(0).name = "A"
// BeanWrapper의 인덱스([N]) 구문 지원

// 단, JSON 방식의 중첩 구조는 @RequestBody로 처리해야 함
```

### 5. allowedFields / disallowedFields — 바인딩 보안

```java
// Mass Assignment 공격 방지:
// 클라이언트가 숨겨진 필드(role, admin 등)를 폼에 추가해 바인딩 시도
// POST /register?username=alice&password=1234&role=ADMIN  ← 위험!

// ① InitBinder로 컨트롤러별 설정
@InitBinder
public void initBinder(WebDataBinder binder) {
    binder.setAllowedFields("username", "password", "email");
    // 허용된 필드만 바인딩 → role은 무시됨
    // 또는
    binder.setDisallowedFields("role", "admin", "createdAt");
    // 금지된 필드 제외
}

// ② 전역 설정 (ControllerAdvice)
@ControllerAdvice
public class GlobalBindingInitializer {
    @InitBinder
    public void globalInitBinder(WebDataBinder binder) {
        binder.setDisallowedFields("*.class", "class.*", "Class");
        // class 필드 접근 차단 (Java 직렬화 공격 방지)
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: @RequestBody vs @ModelAttribute 비교

```bash
# @ModelAttribute — 쿼리 파라미터 바인딩 ✅
curl "http://localhost:8080/search?keyword=spring&page=2"
# → SearchForm{keyword="spring", page=2}

# @ModelAttribute — JSON 본문 바인딩 ❌
curl -X POST http://localhost:8080/search \
     -H "Content-Type: application/json" \
     -d '{"keyword":"spring","page":2}'
# → SearchForm{keyword=null, page=0}  (JSON이 바인딩 안 됨)

# @RequestBody — JSON 본문 바인딩 ✅
curl -X POST http://localhost:8080/users \
     -H "Content-Type: application/json" \
     -d '{"name":"홍길동"}'
# → UserDto{name="홍길동"}
```

### 실험 2: 중첩 객체 바인딩

```bash
curl "http://localhost:8080/order?title=주문&address.city=서울&address.street=강남대로"
# → OrderForm{title="주문", address={city="서울", street="강남대로"}}
```

### 실험 3: allowedFields 보안 설정 확인

```bash
# disallowedFields에 "role" 설정 후
curl "http://localhost:8080/register?username=alice&password=1234&role=ADMIN"
# → UserForm{username="alice", password="1234", role=null}  (role 무시)
```

---

## 🌐 HTTP 레벨 분석

```
HTML 폼 제출:

POST /register HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 45
username=alice&password=1234&email=alice@test.com

처리:
  request.getParameterMap() → {username:["alice"], password:["1234"], email:["alice@test.com"]}
  PropertyValues 생성
  BeanWrapper.setPropertyValue("username", "alice")
  BeanWrapper.setPropertyValue("password", "1234")
  BeanWrapper.setPropertyValue("email", "alice@test.com")
  → UserForm{username="alice", password="1234", email="alice@test.com"}

바인딩 오류 (타입 불일치):
  POST /register?age=not-a-number  (age: int 필드)
  → TypeMismatchException → BindingResult에 오류 저장
  → BindingResult 파라미터 있으면: 컨트롤러 계속 실행 (오류 직접 처리)
  → BindingResult 파라미터 없으면: BindException → 400 Bad Request
```

---

## 🤔 트레이드오프

```
@ModelAttribute vs @RequestBody 선택:
  @ModelAttribute:
    + HTML 폼, 쿼리 파라미터 처리에 적합
    + Spring MVC View 기반 앱에서 Model에 자동 추가
    + 파일 업로드(MultipartFile)와 함께 사용 가능
    - JSON 본문 처리 불가
    - 중첩 JSON 구조 표현 불편

  @RequestBody:
    + JSON/XML 등 다양한 본문 형식 처리
    + REST API에 자연스러운 설계
    + 중첩 객체, 제네릭 타입 지원
    - 파일 업로드 시 @RequestPart 필요
    - HTML 폼 데이터 처리 불편

allowedFields vs disallowedFields:
  allowedFields:  허용 목록 방식 → 보안 강도 높음, 필드 추가 시 명시 필요
  disallowedFields: 금지 목록 방식 → 편의성 높음, 새 위험 필드 추가 시 누락 가능
  → 보안이 중요한 필드(권한, 관리 정보)는 allowedFields 방식 권장
```

---

## 📌 핵심 정리

```
@ModelAttribute 처리 주체
  ServletModelAttributeMethodProcessor

바인딩 소스
  request.getParameterMap() → 쿼리 파라미터 + 폼 본문
  (JSON 본문 아님)

처리 순서
  ① 객체 생성 (기본 생성자 또는 유일 생성자)
  ② WebDataBinder.bind() → PropertyValues → BeanWrapper.setPropertyValue()
  ③ 타입 변환 (ConversionService)
  ④ @Valid 검증 (있으면)
  ⑤ BindingResult에 결과 저장

@ModelAttribute vs @RequestBody 결정적 차이
  @ModelAttribute: getParameter() 기반 → 폼/쿼리
  @RequestBody:    HttpMessageConverter 기반 → JSON/XML 본문

보안: Mass Assignment 방지
  @InitBinder + setAllowedFields() / setDisallowedFields()
  허용 목록 방식(allowedFields)이 더 안전

암묵적 @ModelAttribute
  어노테이션 없는 POJO 파라미터 → 폴백 Processor가 동일하게 처리
```

---

## 🤔 생각해볼 문제

**Q1.** `@ModelAttribute` 처리 중 `DataBinder`가 객체의 `getClass()` 메서드를 통해 `class` 필드에 접근하면 어떤 보안 문제가 생기는가? Spring은 이 문제를 어떻게 방어하는가?

**Q2.** `@ModelAttribute User user`에서 `User` 클래스에 기본 생성자가 없고 `User(String name, int age)` 생성자만 있는 경우 어떻게 동작하는가?

**Q3.** 같은 이름의 파라미터가 쿼리 스트링과 폼 본문에 모두 있을 때 `@ModelAttribute`는 어느 값을 사용하는가?

> 💡 **해설**
>
> **Q1.** `DataBinder`가 `class.classLoader.URLs[0]=malicious` 같은 파라미터를 처리하면 `getClass().getClassLoader().setURLs()` 같은 메서드가 호출될 수 있어 원격 코드 실행(RCE)이 가능한 취약점(CVE-2010-1622, Spring MVC 취약점 계열)이 발생합니다. Spring은 `DataBinder`에 기본적으로 `"class.*"`, `"Class.*"`, `"*.class.*"` 패턴을 `disallowedFields`에 추가해 방어합니다. Spring Boot 2.x 이상은 자동으로 `class` 관련 필드 접근을 차단합니다.
>
> **Q2.** Spring 5.1+ 에서는 단일 생성자가 있으면 그 생성자를 사용합니다. `constructAttribute()` 내부에서 `BeanUtils.getResolvableConstructor(clazz)`가 `User(String name, int age)` 생성자를 선택하고, 생성자 파라미터 `name`과 `age`를 요청 파라미터에서 찾아 주입합니다. 즉 `?name=홍길동&age=30`이면 `new User("홍길동", 30)`으로 객체가 생성됩니다. 이후 DataBinder는 남은 파라미터를 setter로 바인딩합니다.
>
> **Q3.** 서블릿 스펙에 따르면 `request.getParameterValues(name)`은 쿼리 스트링의 값을 폼 본문의 값보다 먼저 반환합니다. 즉, `GET /search?keyword=query` 에 `POST body: keyword=form` 이라면 `getParameterValues("keyword")`는 `["query", "form"]` 순서로 반환합니다. `DataBinder`는 첫 번째 값(`"query"`)을 사용합니다. 따라서 쿼리 파라미터가 폼 본문보다 우선합니다.

---

<div align="center">

**[⬅️ 이전: @RequestParam vs @PathVariable](./03-request-param-path-variable.md)** | **[홈으로 🏠](../README.md)** | **[다음: Validation — @Valid와 BindingResult ➡️](./05-validation-binding-result.md)**

</div>
