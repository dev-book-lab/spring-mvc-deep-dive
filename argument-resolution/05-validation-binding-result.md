# Validation — @Valid와 BindingResult

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@Valid`가 `javax.validation` 체인을 트리거하는 정확한 시점과 내부 경로는?
- `BindingResult`를 파라미터에 선언했을 때와 안 했을 때 각각 어떤 일이 일어나는가?
- `MethodArgumentNotValidException`과 `BindException`의 차이는?
- `@Validated`와 `@Valid`는 어떤 차이가 있으며 Group Validation은 어떻게 동작하는가?
- `@Valid`가 적용되는 Resolver는 `@RequestBody` 전용인가, `@ModelAttribute`에도 동작하는가?
- `BindingResult`의 위치(파라미터 순서)가 중요한 이유는?

---

## 🔍 왜 이 메커니즘이 존재하는가

### 문제: 컨트롤러 코드에서 입력값 검증이 반복된다

```java
// Bad: 수동 검증
@PostMapping("/users")
public User create(@RequestBody UserDto dto) {
    if (dto.getName() == null || dto.getName().isEmpty()) {
        throw new BadRequestException("이름은 필수입니다");
    }
    if (dto.getAge() < 0 || dto.getAge() > 150) {
        throw new BadRequestException("나이가 유효하지 않습니다");
    }
    if (!isValidEmail(dto.getEmail())) {
        throw new BadRequestException("이메일 형식이 잘못됐습니다");
    }
    // 검증 코드가 비즈니스 로직을 가림
    return userService.create(dto);
}

// Good: 선언적 검증
public class UserDto {
    @NotBlank(message = "이름은 필수입니다")
    private String name;

    @Min(0) @Max(150)
    private int age;

    @Email
    private String email;
}

@PostMapping("/users")
public User create(@Valid @RequestBody UserDto dto) {
    return userService.create(dto);  // 검증 통과 후 실행 보장
}
```

---

## 😱 흔한 오해 또는 실수

### Before: BindingResult 위치가 어디든 상관없다

```java
// ❌ 잘못된 코드 — BindingResult가 검증 대상 파라미터 바로 뒤에 있지 않음
@PostMapping("/users")
public String create(
    @Valid @RequestBody UserDto dto,
    Model model,
    BindingResult result) {  // dto 바로 뒤가 아님!
    // → Spring이 BindingResult와 dto를 연결하지 못함
    // → 검증 실패 시 BindException 발생 (result가 채워지지 않음)
}

// ✅ 올바른 코드
@PostMapping("/users")
public String create(
    @Valid @RequestBody UserDto dto,
    BindingResult result,   // 반드시 검증 대상 파라미터 바로 다음!
    Model model) {
    if (result.hasErrors()) { ... }
}
```

### Before: @Valid는 @RequestBody에서만 동작한다

```
❌ 잘못된 이해:
  "@Valid는 JSON 역직렬화(RequestBody)에서만 검증한다"

✅ 실제:
  @Valid는 @RequestBody와 @ModelAttribute 모두에서 동작
  → 각 Processor의 validateIfApplicable() 메서드에서 공통 처리

  @RequestBody → MethodArgumentNotValidException (검증 실패 시)
  @ModelAttribute → BindException (검증 실패 시)
  → 예외 타입이 다르므로 @ExceptionHandler 작성 시 주의
```

---

## ✨ 올바른 이해와 사용

### After: @Valid 처리 전체 흐름

```
POST /users + @Valid @RequestBody UserDto dto

① RequestResponseBodyMethodProcessor.resolveArgument()
   → Jackson으로 역직렬화: UserDto 인스턴스 생성
   → validateIfApplicable(binder, parameter) 호출
     → parameter에 @Valid 또는 @Validated 어노테이션 확인
     → SmartValidator.validate(target, errors, validationHints)
       → javax.validation.Validator.validate(userDto)
         → @NotBlank(name) 검증 → 실패 → ConstraintViolation
         → @Email 검증 → 실패 → ConstraintViolation

② BindingResult에 오류 저장
   binder.getBindingResult().reject() / rejectValue()

③ isBindExceptionRequired(binder, parameter) 확인
   → 다음 파라미터가 BindingResult/Errors 타입인지 확인
   → 있으면: 예외 없이 컨트롤러 실행, BindingResult에 오류 저장
   → 없으면: MethodArgumentNotValidException throw → 400
```

---

## 🔬 내부 동작 원리

### 1. validateIfApplicable() — 검증 트리거 공통 로직

```java
// AbstractMessageConverterMethodArgumentResolver (RequestBody용)
// ModelAttributeMethodProcessor (ModelAttribute용)
// 둘 다 동일한 메서드 보유

protected void validateIfApplicable(WebDataBinder binder, MethodParameter parameter) {
    // 파라미터 어노테이션 전체 탐색
    for (Annotation ann : parameter.getParameterAnnotations()) {
        Object[] validationHints = determineValidationHints(ann);
        if (validationHints != null) {
            // validationHints: @Validated(groups)에서 추출한 Group 클래스 배열
            // @Valid이면 {} (기본 그룹)
            binder.validate(validationHints);
            break;
        }
    }
}

private Object[] determineValidationHints(Annotation ann) {
    // @Valid → javax.validation 트리거, hints = {}
    if (ann.annotationType() == Valid.class) {
        return EMPTY_HINTS;
    }
    // @Validated(SomeGroup.class) → Group 정보 추출
    Validated validatedAnn = AnnotationUtils.getAnnotation(ann, Validated.class);
    if (validatedAnn != null) {
        Object hints = validatedAnn.value();
        return (hints instanceof Object[] ? (Object[]) hints : new Object[]{hints});
    }
    // 커스텀 어노테이션에 @Validated가 메타 어노테이션으로 있을 때도 처리
    return null;
}
```

### 2. SmartValidator.validate() — javax.validation 실행

```java
// SpringValidatorAdapter.validate()
@Override
public void validate(Object target, Errors errors, Object... validationHints) {
    if (validationHints.length > 0) {
        // Group이 있으면 해당 Group으로만 검증
        Set<Class<?>> groups = new LinkedHashSet<>(validationHints.length);
        for (Object hint : validationHints) {
            if (hint instanceof Class<?> clazz) groups.add(clazz);
        }
        processConstraintViolations(
            this.targetValidator.validate(target, ClassUtils.toClassArray(groups)),
            errors);
    } else {
        // Group 없으면 Default 그룹 검증
        processConstraintViolations(this.targetValidator.validate(target), errors);
    }
}

// ConstraintViolation → BindingResult 오류로 변환
private void processConstraintViolations(
        Set<ConstraintViolation<Object>> violations, Errors errors) {
    for (ConstraintViolation<Object> violation : violations) {
        String field = determineField(violation);
        // → "name", "address.city" 등 중첩 필드 경로 포함
        FieldError fieldError = new FieldError(
            errors.getObjectName(),
            field,
            violation.getInvalidValue(),
            false,
            new String[]{violation.getConstraintDescriptor().getAnnotation().annotationType().getSimpleName()},
            null,
            violation.getMessage());
        errors.addError(fieldError);
    }
}
```

### 3. isBindExceptionRequired() — BindingResult 존재 여부 확인

```java
// AbstractMessageConverterMethodArgumentResolver
protected boolean isBindExceptionRequired(WebDataBinder binder, MethodParameter parameter) {
    // 현재 파라미터 인덱스의 다음 파라미터 확인
    int i = parameter.getParameterIndex();
    Class<?>[] paramTypes = parameter.getExecutable().getParameterTypes();
    boolean hasBindingResult = (paramTypes.length > (i + 1) &&
        Errors.class.isAssignableFrom(paramTypes[i + 1]));
    // 다음 파라미터가 Errors(BindingResult) 타입이면 false 반환 → 예외 던지지 않음
    return !hasBindingResult;
}

// 결과:
// @Valid UserDto dto, BindingResult result → hasBindingResult=true → isBindExceptionRequired=false
//   → 검증 실패해도 예외 없음, result에 오류 저장, 컨트롤러 실행
// @Valid UserDto dto                       → hasBindingResult=false → isBindExceptionRequired=true
//   → 검증 실패 시 예외 throw
//     @RequestBody → MethodArgumentNotValidException → 400
//     @ModelAttribute → BindException → 400
```

### 4. @Valid vs @Validated — 그룹 검증

```java
// 검증 그룹 정의
public interface CreateGroup {}
public interface UpdateGroup {}

public class UserDto {
    @NotBlank(groups = {CreateGroup.class, UpdateGroup.class})
    private String name;

    @Null(groups = CreateGroup.class)   // 생성 시 id는 null이어야 함
    @NotNull(groups = UpdateGroup.class) // 수정 시 id는 필수
    private Long id;
}

// @Valid → Default 그룹만 검증 (groups 없는 어노테이션)
@PostMapping("/users")
public User create(@Valid @RequestBody UserDto dto) { ... }
// → @Null(groups=CreateGroup) 검증 안 됨!

// @Validated → 특정 그룹 지정 가능 (Spring 전용)
@PostMapping("/users")
public User create(@Validated(CreateGroup.class) @RequestBody UserDto dto) { ... }
// → CreateGroup에 속한 어노테이션만 검증

@PutMapping("/users/{id}")
public User update(@Validated(UpdateGroup.class) @RequestBody UserDto dto) { ... }
// → UpdateGroup에 속한 어노테이션만 검증
```

### 5. @ExceptionHandler로 검증 오류 처리

```java
// 전역 처리
@RestControllerAdvice
public class ValidationExceptionHandler {

    // @RequestBody + @Valid 실패
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, Object>> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex) {
        Map<String, String> errors = new LinkedHashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(err ->
            errors.put(err.getField(), err.getDefaultMessage()));
        return ResponseEntity.badRequest().body(Map.of(
            "status", 400,
            "errors", errors
        ));
    }

    // @ModelAttribute + @Valid 실패
    @ExceptionHandler(BindException.class)
    public ResponseEntity<Map<String, Object>> handleBindException(BindException ex) {
        // MethodArgumentNotValidException extends BindException이므로
        // BindException 하나로 두 케이스 통합 처리 가능
        Map<String, String> errors = new LinkedHashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(err ->
            errors.put(err.getField(), err.getDefaultMessage()));
        return ResponseEntity.badRequest().body(Map.of("errors", errors));
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: BindingResult 유무에 따른 동작 차이

```java
// Case 1: BindingResult 없음 → 검증 실패 시 예외
@PostMapping("/v1/users")
public User createV1(@Valid @RequestBody UserDto dto) {
    return userService.create(dto);
}

// Case 2: BindingResult 있음 → 컨트롤러에서 직접 처리
@PostMapping("/v2/users")
public ResponseEntity<?> createV2(
    @Valid @RequestBody UserDto dto,
    BindingResult result) {
    if (result.hasErrors()) {
        return ResponseEntity.badRequest().body(result.getAllErrors());
    }
    return ResponseEntity.ok(userService.create(dto));
}
```

```bash
# 검증 실패 요청
curl -X POST http://localhost:8080/v1/users \
     -H "Content-Type: application/json" \
     -d '{"name":"","age":-1}'
# v1: HTTP 400, MethodArgumentNotValidException 응답
# v2: HTTP 400, BindingResult 오류 목록 응답 (컨트롤러 실행됨)
```

### 실험 2: 중첩 객체 검증 (@Valid 중첩)

```java
public class OrderDto {
    @NotBlank
    private String title;

    @Valid               // 중첩 객체도 검증하려면 @Valid 필요
    @NotNull
    private AddressDto address;
}

public class AddressDto {
    @NotBlank
    private String city;
    @NotBlank
    private String street;
}

// @Valid OrderDto → address의 @NotBlank도 검증됨
// @Valid 없으면 address 내부 필드는 검증 안 됨
```

### 실험 3: Group Validation 실험

```bash
# CreateGroup: id=null 필수, name 필수
curl -X POST http://localhost:8080/users \
     -H "Content-Type: application/json" \
     -d '{"id":1,"name":"홍길동"}'
# → id가 있으면 @Null(groups=CreateGroup) 위반 → 400

curl -X POST http://localhost:8080/users \
     -H "Content-Type: application/json" \
     -d '{"name":"홍길동"}'
# → id=null, name 있음 → 검증 통과 → 201
```

---

## 🌐 HTTP 레벨 분석

```
검증 실패 시 응답:

POST /users HTTP/1.1
Content-Type: application/json
{"name": "", "age": -5, "email": "not-an-email"}

처리:
  Jackson 역직렬화 성공 → UserDto{name="", age=-5, email="not-an-email"}
  @Valid 검증:
    @NotBlank(name) → 실패: "이름은 필수입니다"
    @Min(0)(age) → 실패: "0 이상이어야 합니다"
    @Email(email) → 실패: "올바른 이메일 형식이 아닙니다"
  BindingResult에 3개 오류 저장
  BindingResult 파라미터 없음 → MethodArgumentNotValidException

HTTP/1.1 400 Bad Request
Content-Type: application/json
{
  "status": 400,
  "errors": {
    "name": "이름은 필수입니다",
    "age": "0 이상이어야 합니다",
    "email": "올바른 이메일 형식이 아닙니다"
  }
}
```

---

## 🤔 트레이드오프

```
BindingResult 사용 여부:
  사용:   컨트롤러에서 오류를 직접 처리 → 세밀한 분기 가능
           View에 오류 정보 전달 (Spring MVC 폼 재표시)
  미사용: @ExceptionHandler로 전역 처리 → 코드 간결
           REST API에서 대부분 전역 처리가 더 적합

@Valid vs @Validated:
  @Valid:      표준 (javax.validation), 기본 그룹만 검증
  @Validated:  Spring 전용, 그룹 지정 가능
  → 그룹 검증이 필요하면 @Validated, 간단한 경우 @Valid

검증 레이어 위치:
  Controller(@Valid): 빠른 실패, 클라이언트 입력 형식 검증
  Service(@Validated): 비즈니스 규칙 검증
  DB(Constraint): 데이터 무결성 최종 보장
  → 세 레이어 모두 필요, 중복처럼 보여도 각 레이어의 책임이 다름
```

---

## 📌 핵심 정리

```
@Valid 트리거 조건
  파라미터에 @Valid 또는 @Validated 어노테이션 존재
  → validateIfApplicable() → SmartValidator.validate()
  → javax.validation.Validator.validate()

BindingResult 위치 규칙
  반드시 검증 대상 파라미터 바로 다음 위치
  → isBindExceptionRequired()가 다음 파라미터 타입 확인

예외 종류
  @RequestBody + @Valid 실패 → MethodArgumentNotValidException → 400
  @ModelAttribute + @Valid 실패 → BindException → 400
  MethodArgumentNotValidException extends BindException
  → BindException 하나로 통합 처리 가능

@Valid vs @Validated
  @Valid:     Default 그룹, javax.validation 표준
  @Validated: 그룹 지정 가능, Spring 전용
  → 그룹 검증 필요 시 @Validated(Group.class) 사용

중첩 객체 검증
  중첩 필드도 검증하려면 해당 필드에 @Valid 추가 필요
```

---

## 🤔 생각해볼 문제

**Q1.** `@Valid @RequestBody UserDto dto, BindingResult result` 순서로 선언했을 때, `UserDto`의 역직렬화(JSON 파싱) 자체가 실패하면 `result`에 오류가 담기는가, 아니면 별도 예외가 발생하는가?

**Q2.** `@Validated(CreateGroup.class)` 와 `@Validated({CreateGroup.class, Default.class})` 의 차이는? Default 그룹을 명시하지 않으면 어떤 어노테이션은 검증되지 않는가?

**Q3.** 서비스 레이어에서 `@Validated`를 사용하는 AOP 기반 검증과 컨트롤러 레이어 `@Valid` 검증의 차이는? AOP 기반 검증 실패 시 발생하는 예외 타입은?

> 💡 **해설**
>
> **Q1.** JSON 파싱 실패(`JsonProcessingException`)는 `HttpMessageNotReadableException`으로 변환되어 `result`와 관계없이 즉시 throw됩니다. `BindingResult`는 역직렬화 성공 후 Bean Validation(@Valid) 단계의 오류만 담습니다. 즉, JSON 구문 오류(잘못된 JSON)나 타입 불일치(`"age": "not-a-number"`) 같은 역직렬화 오류는 `BindingResult`로 처리되지 않고 400 예외로 바로 응답됩니다.
>
> **Q2.** `@Validated(CreateGroup.class)` 만 지정하면 `Default` 그룹의 어노테이션(`groups` 속성 없는 `@NotBlank`, `@NotNull` 등)은 검증되지 않습니다. 예를 들어 `@NotBlank String name` (groups 없음)은 Default 그룹에 속하므로 `CreateGroup`만 지정하면 name 검증이 스킵됩니다. 두 그룹 모두 검증하려면 `@Validated({CreateGroup.class, Default.class})`로 명시해야 합니다. 이 차이를 모르면 필수 검증이 의도치 않게 건너뛰어지는 버그가 발생합니다.
>
> **Q3.** AOP 기반 검증은 `MethodValidationPostProcessor` + `@Validated`를 서비스 Bean에 붙여 사용합니다. 메서드 파라미터나 반환값에 `@Valid`를 붙이면 AOP 프록시가 메서드 호출 전/후에 검증을 수행합니다. 검증 실패 시 `javax.validation.ConstraintViolationException`이 발생합니다 — 컨트롤러 레이어의 `MethodArgumentNotValidException`과 다른 타입입니다. 따라서 `@ExceptionHandler`에서 별도로 처리해야 합니다. 또한 같은 클래스 내 메서드 간 호출은 AOP 프록시를 거치지 않으므로 검증이 동작하지 않는다는 Spring AOP self-invocation 제약도 있습니다.

---

<div align="center">

**[⬅️ 이전: @ModelAttribute 데이터 바인딩](./04-model-attribute-binding.md)** | **[홈으로 🏠](../README.md)** | **[다음: Servlet API 주입 ➡️](./06-servlet-api-injection.md)**

</div>
