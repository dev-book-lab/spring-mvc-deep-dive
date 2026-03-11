# @ExceptionHandler 메서드 매칭 과정 — 가장 구체적인 핸들러를 찾는 알고리즘

---

## 🎯 핵심 질문

- `ExceptionHandlerExceptionResolver`가 발생 예외의 상속 계층을 탐색하는 알고리즘은?
- `@ExceptionHandler(value = {IllegalArgumentException.class, ...})`처럼 배열로 선언하면 어떻게 처리되는가?
- `@ExceptionHandler` 없이 파라미터 타입으로 예외를 선언하는 방식은 언제 동작하는가?
- `ExceptionHandlerMethodResolver`의 캐시 구조는?
- 같은 Controller에 `@ExceptionHandler(RuntimeException.class)`와 `@ExceptionHandler(IllegalArgumentException.class)` 둘 다 있을 때 어느 것이 선택되는가?
- `@ExceptionHandler`가 처리할 수 없는 예외가 파라미터로 넘겨지면 어떻게 되는가?

---

## 🔍 왜 이 메커니즘이 존재하는가

```
예외 계층이 있을 때:
  Exception
  └── RuntimeException
       ├── IllegalArgumentException
       │    └── InvalidUserIdException  ← 발생한 예외
       └── IllegalStateException

@ExceptionHandler 선언:
  @ExceptionHandler(RuntimeException.class) → 범용 처리
  @ExceptionHandler(IllegalArgumentException.class) → 더 구체적 처리

"발생한 예외에 가장 가까운(구체적인) 핸들러"를 선택해야 함
→ 세밀한 제어 가능 + 폴백 핸들러 동작
```

---

## 😱 흔한 오해 또는 실수

### Before: @ExceptionHandler 파라미터 타입과 value 속성이 달라도 된다

```java
// ❌ 잘못된 설정
@ExceptionHandler(UserNotFoundException.class)
public ResponseEntity<String> handle(IllegalArgumentException ex) {
    // value와 파라미터 타입이 다름
    // → 어떤 예외가 ex로 넘어오는가?
}

// ✅ 실제:
// value에 선언된 예외가 실제로 발생해야 이 핸들러가 선택됨
// 그런데 파라미터 타입이 UserNotFoundException을 받지 못하는 IllegalArgumentException
// → 타입 불일치 → IllegalStateException 발생 가능
//
// 올바른 패턴:
@ExceptionHandler(UserNotFoundException.class)
public ResponseEntity<String> handle(UserNotFoundException ex) { ... }
// 또는 파라미터 없이:
@ExceptionHandler(UserNotFoundException.class)
public ResponseEntity<String> handle() { ... }
// 또는 value 없이 파라미터 타입으로만:
@ExceptionHandler
public ResponseEntity<String> handle(UserNotFoundException ex) { ... }
```

### Before: @ExceptionHandler는 checked 예외도 처리한다

```
❌ 잘못된 이해:
  "@ExceptionHandler는 어떤 예외든 처리한다"

✅ 실제:
  @ExceptionHandler는 Controller 메서드 실행 중 발생한 예외를 처리
  → Controller 메서드에서 throw된 예외 (런타임, 체크드 모두 가능)
  → 단, InvocableHandlerMethod.invokeForRequest()가 모든 예외를 
    RuntimeException으로 래핑하지 않음 → 원래 예외 그대로 전달

  Spring 내부 컨버터/바인더 예외는 이미 처리됨:
    JSON 파싱 오류 → HttpMessageNotReadableException (RuntimeException 아님)
    → DefaultHandlerExceptionResolver 또는 @ExceptionHandler 처리 가능
```

---

## ✨ 올바른 이해와 사용

### After: 매칭 알고리즘 전체 흐름

```
발생 예외: InvalidUserIdException extends IllegalArgumentException extends RuntimeException

ExceptionHandlerMethodResolver 탐색:

① 로컬 Controller의 @ExceptionHandler 메서드 목록 수집
② 각 메서드의 처리 예외 타입 확인
③ 발생 예외와의 "거리(depth)" 계산:
   @ExceptionHandler(InvalidUserIdException.class) → depth=0 (완전 일치)
   @ExceptionHandler(IllegalArgumentException.class) → depth=1 (부모)
   @ExceptionHandler(RuntimeException.class) → depth=2 (조부모)
④ 가장 낮은 depth(가장 구체적) 핸들러 선택
```

---

## 🔬 내부 동작 원리

### 1. ExceptionHandlerMethodResolver — 초기화 시 메서드 등록

```java
// ExceptionHandlerMethodResolver.java
public class ExceptionHandlerMethodResolver {

    // 예외 타입 → @ExceptionHandler 메서드 캐시
    private final Map<Class<? extends Throwable>, Method> mappedMethods = new LinkedHashMap<>(16);

    // 예외 타입 → 메서드 런타임 캐시 (상속 계층 탐색 결과)
    private final Map<Class<? extends Throwable>, Method> exceptionLookupCache =
        new ConcurrentHashMap<>(16);

    // 생성자: Controller 클래스의 @ExceptionHandler 메서드 스캔
    public ExceptionHandlerMethodResolver(Class<?> handlerType) {
        for (Method method : MethodIntrospector.selectMethods(handlerType,
                (MethodIntrospector.MetadataLookup<Method>) m ->
                    AnnotatedElementUtils.hasAnnotation(m, ExceptionHandler.class) ? m : null)) {

            for (Class<? extends Throwable> exceptionType : detectExceptionMappings(method)) {
                addExceptionMapping(exceptionType, method);
                // mappedMethods: {UserNotFoundException → handleNotFound(), ...}
            }
        }
    }

    // @ExceptionHandler에서 처리 예외 타입 추출
    private List<Class<? extends Throwable>> detectExceptionMappings(Method method) {
        List<Class<? extends Throwable>> result = new ArrayList<>();
        // 1. @ExceptionHandler(value = {...}) 속성에서 추출
        ExceptionHandler ann = AnnotatedElementUtils.findMergedAnnotation(method, ExceptionHandler.class);
        if (ann != null) {
            result.addAll(Arrays.asList(ann.value()));
        }
        // 2. value가 비어있으면 파라미터 타입에서 추출
        if (result.isEmpty()) {
            for (Class<?> paramType : method.getParameterTypes()) {
                if (Throwable.class.isAssignableFrom(paramType)) {
                    result.add((Class<? extends Throwable>) paramType);
                }
            }
        }
        // 3. 둘 다 없으면 오류
        if (result.isEmpty()) {
            throw new IllegalStateException("No exception types mapped to " + method);
        }
        return result;
    }
}
```

### 2. resolveMethodByExceptionType() — 상속 계층 탐색

```java
// ExceptionHandlerMethodResolver.resolveMethodByExceptionType()
@Nullable
public Method resolveMethodByExceptionType(Class<? extends Throwable> exceptionType) {
    // 캐시 확인 (이미 탐색한 예외 타입)
    Method method = this.exceptionLookupCache.get(exceptionType);
    if (method == null) {
        // 캐시 미스 → 탐색
        method = resolveMethodByMatching(exceptionType);
        this.exceptionLookupCache.put(exceptionType, method != null ? method : NO_MATCHING_EXCEPTION_HANDLER_METHOD);
    }
    return (method != NO_MATCHING_EXCEPTION_HANDLER_METHOD ? method : null);
}

@Nullable
private Method resolveMethodByMatching(Class<? extends Throwable> exceptionType) {
    // 완전 일치 먼저
    Method method = this.mappedMethods.get(exceptionType);
    if (method != null) return method;

    // 없으면 상속 계층 탐색 (BFS 방식)
    // depth를 측정해 가장 가까운 부모 선택
    List<Class<? extends Throwable>> matches = new ArrayList<>();
    for (Class<? extends Throwable> mappedException : this.mappedMethods.keySet()) {
        if (mappedException.isAssignableFrom(exceptionType)) {
            matches.add(mappedException);
        }
    }
    if (!matches.isEmpty()) {
        if (matches.size() > 1) {
            // 여러 후보 → 상속 계층 깊이로 정렬 (구체적인 것 우선)
            matches.sort(new ExceptionDepthComparator(exceptionType));
        }
        return this.mappedMethods.get(matches.get(0));
    }
    return null;
}
```

### 3. ExceptionDepthComparator — 상속 깊이 비교

```java
// ExceptionDepthComparator.java
public class ExceptionDepthComparator implements Comparator<Class<? extends Throwable>> {

    private final Class<? extends Throwable> targetException;

    @Override
    public int compare(Class<? extends Throwable> o1, Class<? extends Throwable> o2) {
        int depth1 = getDepth(o1, this.targetException, 0);
        int depth2 = getDepth(o2, this.targetException, 0);
        return depth1 - depth2;  // 낮은 depth = 더 구체적 = 우선
    }

    private int getDepth(Class<?> declaredException, Class<?> exceptionToMatch, int depth) {
        if (exceptionToMatch.equals(declaredException)) return depth;  // 일치 → depth
        if (exceptionToMatch == Throwable.class) return Integer.MAX_VALUE;  // 루트 도달 실패
        return getDepth(declaredException, exceptionToMatch.getSuperclass(), depth + 1);
    }
}

// 예시:
// 발생 예외: InvalidUserIdException
//   depth(InvalidUserIdException) = 0 (완전 일치)
//   depth(IllegalArgumentException) = 1 (부모)
//   depth(RuntimeException) = 2 (조부모)
// → InvalidUserIdException 처리기 선택
```

### 4. @ExceptionHandler 메서드 파라미터

```java
// @ExceptionHandler 메서드가 받을 수 있는 파라미터
@ExceptionHandler(UserNotFoundException.class)
public ResponseEntity<ErrorResponse> handle(
    UserNotFoundException ex,          // 발생한 예외 (필수 아님)
    HttpServletRequest request,        // 요청 정보
    HttpServletResponse response,      // 응답 객체
    WebRequest webRequest,             // Spring 추상화 요청
    Model model,                       // MVC 모델 (뷰 반환 시)
    Locale locale,                     // 로케일
    BindingResult bindingResult        // @Valid 실패 시 오류 정보
    // 단, BindingResult는 MethodArgumentNotValidException에서만 의미 있음
) {
    return ResponseEntity.status(404)
        .body(new ErrorResponse("USER_NOT_FOUND", ex.getMessage()));
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 상속 계층 매칭 확인

```java
@RestController
public class UserController {

    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        if (id < 0) throw new InvalidUserIdException("음수 ID: " + id);
        if (id == 0) throw new UserNotFoundException("ID 0 없음");
        throw new RuntimeException("알 수 없는 오류");
    }

    // depth=0 (완전 일치)
    @ExceptionHandler(InvalidUserIdException.class)
    public ResponseEntity<String> handleInvalid(InvalidUserIdException ex) {
        return ResponseEntity.badRequest().body("INVALID: " + ex.getMessage());
    }

    // depth=1 (IllegalArgumentException의 부모)
    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<String> handleIllegal(IllegalArgumentException ex) {
        return ResponseEntity.badRequest().body("ILLEGAL: " + ex.getMessage());
    }

    // depth=2
    @ExceptionHandler(RuntimeException.class)
    public ResponseEntity<String> handleRuntime(RuntimeException ex) {
        return ResponseEntity.internalServerError().body("RUNTIME: " + ex.getMessage());
    }
}
```

```bash
curl http://localhost:8080/users/-1
# → "INVALID: 음수 ID: -1"  (InvalidUserIdException, depth=0 선택)

curl http://localhost:8080/users/0
# → "RUNTIME: ID 0 없음"  (UserNotFoundException → RuntimeException 계층 매칭)
# UserNotFoundException이 RuntimeException 직계 자식이면 depth=1로 RuntimeException 핸들러

curl http://localhost:8080/users/99
# → "RUNTIME: 알 수 없는 오류"  (RuntimeException 직접 일치)
```

### 실험 2: value 배열로 여러 예외 처리

```java
@ExceptionHandler({UserNotFoundException.class, OrderNotFoundException.class})
public ResponseEntity<String> handleNotFound(RuntimeException ex) {
    return ResponseEntity.notFound().build();
    // 두 예외 중 어느 것이 발생해도 이 핸들러 선택
}
```

---

## 🌐 HTTP 레벨 분석

```
발생 예외 → 매칭 → 응답:

InvalidUserIdException (extends IllegalArgumentException):
  ExceptionHandlerMethodResolver.resolveMethodByExceptionType(InvalidUserIdException):
    mappedMethods 탐색:
      InvalidUserIdException → handleInvalid (depth=0) ✅ 선택
  handleInvalid() 실행 → ResponseEntity.badRequest()

HTTP/1.1 400 Bad Request
Content-Type: application/json
{"error":"INVALID_ID","message":"음수 ID: -1"}
```

---

## 📌 핵심 정리

```
매칭 알고리즘
  ① 완전 일치 먼저 확인
  ② 없으면 isAssignableFrom()으로 부모 클래스 탐색
  ③ 여러 후보 → ExceptionDepthComparator로 정렬
  ④ 가장 낮은 depth(구체적) 선택

캐시
  exceptionLookupCache: ConcurrentHashMap
  → 동일 예외 타입은 두 번째 발생부터 O(1) 매칭

처리 예외 타입 선언 우선순위
  @ExceptionHandler(value = {ExA.class, ExB.class}) → value에서 추출
  @ExceptionHandler 없이 파라미터 타입만 → 파라미터에서 추출
  둘 다 있으면 value 우선

구체적 vs 범용 패턴
  구체적 예외 처리기: 세밀한 오류 메시지
  RuntimeException 처리기: 최후 폴백
  두 가지를 조합해 계층적 예외 처리 구성 가능
```

---

## 🤔 생각해볼 문제

**Q1.** `@ExceptionHandler(Exception.class)`를 로컬에 선언하면 `@ControllerAdvice`의 더 구체적인 핸들러보다 우선하는가?

**Q2.** `@ExceptionHandler` 메서드 자체에서 예외가 발생하면 어떻게 처리되는가?

**Q3.** 같은 예외를 처리하는 두 개의 `@ExceptionHandler`를 같은 Controller에 선언하면 어떻게 되는가?

> 💡 **해설**
>
> **Q1.** 네, 로컬 `@ExceptionHandler(Exception.class)`가 `@ControllerAdvice`의 더 구체적인 핸들러보다 우선합니다. `getExceptionHandlerMethod()`는 로컬 탐색을 먼저 수행하고, 로컬에서 찾으면 전역 탐색을 건너뜁니다. 즉 로컬 우선 원칙은 구체성보다 강합니다.
>
> **Q2.** `@ExceptionHandler` 메서드 실행 중 예외가 발생하면 해당 예외는 다시 `HandlerExceptionResolver` 체인으로 가지 않습니다. Spring은 `@ExceptionHandler` 실행 중 예외는 처리하지 않고 그냥 전파합니다. 결국 서블릿 컨테이너로 전파되어 500 오류가 됩니다. 따라서 `@ExceptionHandler` 내부는 반드시 예외-안전하게 작성해야 합니다.
>
> **Q3.** `ExceptionHandlerMethodResolver` 초기화 시 `addExceptionMapping()`에서 중복 예외 타입이 감지되면 `IllegalStateException: Ambiguous @ExceptionHandler methods mapped...`가 발생합니다. 애플리케이션 시작 시 오류가 발생하므로 같은 예외를 같은 범위(로컬/전역)에서 두 핸들러로 처리하는 것은 불가능합니다.

---

<div align="center">

**[⬅️ 이전: ExceptionResolver 체인](./01-handler-exception-resolver.md)** | **[홈으로 🏠](../README.md)** | **[다음: @ControllerAdvice 적용 범위 ➡️](./03-controller-advice-scope.md)**

</div>
