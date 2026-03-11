# HTTP 캐싱 — Cache-Control과 ETag

---

## 🎯 핵심 질문

- `ShallowEtagHeaderFilter`가 응답 본문 해시로 ETag를 생성하는 과정은?
- `Cache-Control` 헤더의 주요 지시자(`max-age`, `no-cache`, `no-store`, `must-revalidate`)의 차이는?
- `If-None-Match` / `If-Modified-Since` 조건부 요청이 Spring MVC에서 처리되는 경로는?
- `WebContentInterceptor`와 `CacheControl` 빌더 API로 헤더를 설정하는 방법은?
- `ShallowEtagHeaderFilter` vs 커스텀 ETag 전략의 트레이드오프는?
- 정적 리소스와 동적 API 응답의 캐싱 전략 차이는?

---

## 🔍 왜 이 메커니즘이 존재하는가

```
HTTP 캐싱의 목표:
  불필요한 네트워크 전송 제거
  서버 부하 감소
  클라이언트 응답 시간 단축

두 가지 캐싱 전략:
  1. 만료 기반 (Expiration):
     Cache-Control: max-age=3600 (1시간 동안 캐시 유효)
     → 캐시 유효 기간 내: 서버 요청 없이 로컬 캐시 사용
     → 유효 기간 후: 서버에 재검증 요청

  2. 검증 기반 (Validation):
     ETag / Last-Modified 헤더
     → 캐시 만료 후 서버에 "변경됐는가?" 확인
     → 변경 없으면: 304 Not Modified (본문 없음)
     → 변경됐으면: 200 OK + 새 본문

조합:
  Cache-Control: max-age=3600, must-revalidate
  + ETag: "abc123"
  → 1시간 내: 로컬 캐시 사용 (서버 요청 없음)
  → 1시간 후: 서버에 If-None-Match: "abc123" 전송
    → 304 Not Modified 또는 200 OK
```

---

## 😱 흔한 오해 또는 실수

### Before: no-cache는 캐시를 사용하지 않는다는 의미다

```
❌ 잘못된 이해:
  "Cache-Control: no-cache = 캐시하지 마라"

✅ 실제:
  no-cache = "캐시해도 되지만, 사용 전 항상 서버에서 재검증해라"
  → If-None-Match / If-Modified-Since 헤더와 함께 서버 요청
  → 304 Not Modified → 캐시된 응답 사용
  → 200 OK → 새 응답으로 캐시 갱신

  캐시 금지는 no-store:
  "저장하지 마라 (민감한 정보: 금융/의료 데이터)"
  → 클라이언트가 응답을 전혀 저장하지 않음
  → 매 요청마다 서버에서 새로 받아야 함

  정리:
  no-cache  → 저장 O, 사용 전 재검증 필수
  no-store  → 저장 X, 항상 서버에서 새로 받음
  max-age=0 → 즉시 만료, no-cache와 유사
```

### Before: ShallowEtagHeaderFilter가 서버 부하를 줄인다

```
❌ 잘못된 이해: "ETag로 304 응답 → Controller 실행 안 됨 → 서버 부하 감소"

✅ 실제:
  ShallowEtagHeaderFilter의 동작:
  1. 요청 → Controller 실행 → 응답 본문 생성 (항상 실행!)
  2. 생성된 본문 해시 계산 → ETag 생성
  3. If-None-Match와 비교 → 동일하면 304 (본문 전송 안 함)

  즉: Controller는 항상 실행됨, 네트워크 전송만 줄임
  서버 부하 감소: ❌ (Controller 실행 비용은 그대로)
  네트워크 비용 감소: ✅ (304 = 본문 없음)

  진정한 서버 부하 감소:
  → Cache-Control: max-age=N 으로 클라이언트가 아예 요청 안 보내게
  → 또는 커스텀 ETag (DB 버전/타임스탬프 기반) 으로
     Controller 실행 전에 304 반환
```

---

## ✨ 올바른 이해와 사용

### After: Cache-Control 지시자 비교

```
지시자          | 캐시 저장 | 재검증 필요 | 사용 사례
----------------|-----------|------------|------------------
max-age=N       | O         | 만료 후      | 정적 리소스 (이미지, JS)
no-cache        | O         | 항상         | HTML, API 응답
no-store        | X         | 항상 새로     | 민감한 데이터
must-revalidate | O         | 만료 후 필수   | 재검증 강제
public          | O (공유)   | -           | CDN 캐싱 허용
private         | O (개인)   | -           | 인증된 사용자 응답
immutable       | O         | 불필요        | 해시 포함 정적 리소스
```

---

## 🔬 내부 동작 원리

### 1. ShallowEtagHeaderFilter — 응답 본문 해시 기반 ETag

```java
// ShallowEtagHeaderFilter.java
public class ShallowEtagHeaderFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                     FilterChain chain) throws ... {

        // 응답 본문을 캡처하는 래퍼
        ShallowEtagResponseWrapper responseWrapper =
            new ShallowEtagResponseWrapper(response);

        // Controller 실행 (응답 본문이 responseWrapper에 캡처됨)
        chain.doFilter(request, responseWrapper);

        // ETag 생성 (MD5 해시)
        byte[] body = responseWrapper.toByteArray();
        String eTag = generateETagHeaderValue(body, false);
        // → '"' + MD5(body) + '"' 형식 (약한 ETag: W/"...")

        // If-None-Match 조건부 요청 처리
        String ifNoneMatch = request.getHeader(HttpHeaders.IF_NONE_MATCH);
        if (ifNoneMatch != null && eTagMatch(ifNoneMatch, eTag)) {
            // 매칭 → 304 Not Modified
            response.setStatus(HttpServletResponse.SC_NOT_MODIFIED);
            response.setHeader(HttpHeaders.ETAG, eTag);
            // 본문 전송 안 함
        } else {
            // 미매칭 → 200 + 본문 + ETag 헤더
            response.setHeader(HttpHeaders.ETAG, eTag);
            response.setContentLength(body.length);
            FileCopyUtils.copy(body, response.getOutputStream());
        }
    }

    protected String generateETagHeaderValue(byte[] bytes, boolean weak) {
        StringBuilder builder = new StringBuilder();
        if (weak) builder.append("W/");
        builder.append('"');
        DigestUtils.appendMd5DigestAsHex(bytes, builder);  // MD5
        builder.append('"');
        return builder.toString();
    }
}
```

### 2. WebRequest.checkNotModified() — Controller 수준 조건부 응답

```java
// Controller에서 직접 조건부 응답 처리
// → Controller 실행 전에 304 반환 가능 (ShallowEtag와 달리)

@GetMapping("/users/{id}")
public ResponseEntity<User> getUser(@PathVariable Long id, WebRequest request) {
    User user = userService.findById(id);

    // 방법 1: ETag 기반
    String eTag = '"' + user.getVersion().toString() + '"';  // 엔티티 버전으로 ETag
    if (request.checkNotModified(eTag)) {
        return null;  // 304 Not Modified (Spring이 알아서 처리)
    }

    return ResponseEntity.ok()
        .eTag(eTag)
        .cacheControl(CacheControl.noCache())
        .body(user);
}

// 방법 2: LastModified 기반
@GetMapping("/articles/{id}")
public ResponseEntity<Article> getArticle(@PathVariable Long id, WebRequest request) {
    Article article = articleService.findById(id);
    long lastModified = article.getUpdatedAt().toEpochMilli();

    if (request.checkNotModified(lastModified)) {
        return null;  // 304
    }

    return ResponseEntity.ok()
        .lastModified(lastModified)
        .cacheControl(CacheControl.maxAge(1, TimeUnit.HOURS))
        .body(article);
}

// checkNotModified() 내부:
// ① If-None-Match 헤더와 eTag 비교 (ETag 방식)
// ② If-Modified-Since 헤더와 lastModified 비교 (시간 방식)
// → 매칭 시 response.setStatus(304) + return true
// → Controller에서 null 반환 → DispatcherServlet이 304 처리
```

### 3. CacheControl 빌더 API

```java
// Spring의 CacheControl 빌더
@GetMapping("/images/{name}")
public ResponseEntity<byte[]> getImage(@PathVariable String name) {
    byte[] imageData = imageService.load(name);

    return ResponseEntity.ok()
        .cacheControl(CacheControl
            .maxAge(30, TimeUnit.DAYS)    // 30일 캐시
            .cachePublic()                // CDN 캐싱 허용
        )
        .header("Content-Type", "image/jpeg")
        .body(imageData);
}

// API 응답 (자주 변경)
@GetMapping("/api/products")
public ResponseEntity<List<Product>> getProducts() {
    return ResponseEntity.ok()
        .cacheControl(CacheControl
            .noCache()                    // 항상 재검증
            .cachePrivate()               // 개인 캐시만
        )
        .body(productService.findAll());
}

// 민감한 데이터
@GetMapping("/api/account")
public ResponseEntity<Account> getAccount() {
    return ResponseEntity.ok()
        .cacheControl(CacheControl.noStore())  // 저장 금지
        .body(accountService.getCurrent());
}

// 해시 버전 정적 리소스 (/static/app.a1b2c3.js)
@GetMapping("/versioned/{filename}")
public ResponseEntity<byte[]> getVersionedResource(@PathVariable String filename) {
    return ResponseEntity.ok()
        .cacheControl(CacheControl
            .maxAge(365, TimeUnit.DAYS)
            .immutable()  // 변경 없음, 재검증 불필요
        )
        .body(resourceService.load(filename));
}
```

### 4. WebContentInterceptor — 경로별 Cache-Control 일괄 설정

```java
// Interceptor로 경로별 캐시 정책 일괄 적용
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        WebContentInterceptor interceptor = new WebContentInterceptor();

        // 경로별 Cache-Control 설정
        interceptor.addCacheMapping(
            CacheControl.maxAge(1, TimeUnit.DAYS).cachePublic(),
            "/api/categories/**",   // 변경 빈도 낮은 카테고리
            "/api/config/**"
        );
        interceptor.addCacheMapping(
            CacheControl.noStore(),
            "/api/account/**",      // 민감한 계정 정보
            "/api/payment/**"
        );
        interceptor.addCacheMapping(
            CacheControl.noCache().cachePrivate(),
            "/api/**"               // 나머지 API
        );

        registry.addInterceptor(interceptor).addPathPatterns("/**");
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: ShallowEtagHeaderFilter 등록과 동작

```java
// Spring Boot: 빈 등록으로 자동 적용
@Bean
public ShallowEtagHeaderFilter shallowEtagHeaderFilter() {
    return new ShallowEtagHeaderFilter();
}
```

```bash
# 첫 번째 요청
curl -i http://localhost:8080/api/users/1
# HTTP/1.1 200 OK
# ETag: "a3f4b2c1d5e6..."
# {"id":1,"name":"홍길동"}

# ETag로 조건부 요청
curl -i -H 'If-None-Match: "a3f4b2c1d5e6..."' http://localhost:8080/api/users/1
# HTTP/1.1 304 Not Modified
# ETag: "a3f4b2c1d5e6..."
# (본문 없음)

# 데이터 변경 후 같은 요청
# HTTP/1.1 200 OK
# ETag: "f1g2h3i4j5k6..."  ← 해시 변경
# {"id":1,"name":"홍길동(수정)"}
```

### 실험 2: 커스텀 ETag로 Controller 실행 절약

```java
// ShallowEtag: Controller 항상 실행
// 커스텀 ETag: Controller 실행 전 304 반환 가능

@GetMapping("/expensive/{id}")
public ResponseEntity<ExpensiveData> getExpensiveData(
        @PathVariable Long id, WebRequest webRequest) {

    // DB에서 버전만 조회 (저렴한 쿼리)
    String version = repository.findVersionById(id);
    String eTag = '"' + version + '"';

    // ETag 매칭 → Controller 본체 실행 없이 304
    if (webRequest.checkNotModified(eTag)) {
        return null;
    }

    // 미매칭 → 비싼 쿼리 실행
    ExpensiveData data = repository.findFullDataById(id);
    return ResponseEntity.ok().eTag(eTag).body(data);
}
```

---

## 🌐 HTTP 레벨 분석

```
ETag 검증 흐름:

① 최초 요청:
GET /api/users/1 HTTP/1.1

HTTP/1.1 200 OK
ETag: "v42"
Cache-Control: no-cache, private
{"id":1,"name":"홍길동","version":42}

② 브라우저 캐시 만료 후 재검증:
GET /api/users/1 HTTP/1.1
If-None-Match: "v42"

  서버: checkNotModified("v42") → user.version=42 → 매칭!
  → response.setStatus(304)
  → return null (Controller 본문 실행 안 함)

HTTP/1.1 304 Not Modified
ETag: "v42"
(본문 없음 → 브라우저가 캐시된 응답 사용)

③ 다른 클라이언트가 user 수정 → version=43:
GET /api/users/1 HTTP/1.1
If-None-Match: "v42"

  서버: checkNotModified("v42") → user.version=43 → 미매칭!
  → 200 OK + 새 본문

HTTP/1.1 200 OK
ETag: "v43"
{"id":1,"name":"홍길동(수정)","version":43}
```

---

## 📌 핵심 정리

```
Cache-Control 주요 지시자
  max-age=N    → N초 동안 캐시 유효
  no-cache     → 저장 O, 사용 전 재검증 필수
  no-store     → 저장 X (민감한 데이터)
  public       → CDN/공유 캐시 허용
  private      → 브라우저 전용 캐시
  immutable    → 변경 없음, 재검증 불필요 (버전 정적 리소스)

ShallowEtagHeaderFilter
  응답 본문 MD5 해시 → ETag 자동 생성
  Controller는 항상 실행 (네트워크 절약만)
  간단하게 ETag 지원 시 유용

WebRequest.checkNotModified()
  Controller 실행 전 304 반환 가능
  DB 버전 필드 기반 ETag → Controller 실행 비용 절약
  커스텀 캐싱 전략 구현에 사용

CacheControl 빌더
  ResponseEntity.cacheControl(...) → Cache-Control 헤더 설정
  정적 리소스: maxAge + public + immutable
  API: noCache + private 또는 noStore

전략 선택
  정적 리소스 (이미지/JS/CSS): max-age=long + immutable
  동적 API (변경 가능): no-cache + ETag 검증
  민감한 데이터: no-store
```

---

## 🤔 생각해볼 문제

**Q1.** `Cache-Control: max-age=3600, must-revalidate`와 `Cache-Control: max-age=3600, no-cache`의 차이는 무엇인가?

**Q2.** `ShallowEtagHeaderFilter`를 사용할 때 `gzip` 압축 필터와 함께 사용하면 어떤 문제가 발생할 수 있는가?

**Q3.** 클라이언트 A가 ETag `"v42"`를 가지고 있고, 클라이언트 B가 데이터를 수정해 `v43`이 됐다가 다시 원래 값으로 되돌려 `v44`가 됐을 때(값은 동일, 버전만 증가), 클라이언트 A는 `If-None-Match: "v42"` 요청 시 어떤 응답을 받는가?

> 💡 **해설**
>
> **Q1.** `max-age=3600, must-revalidate`는 3600초 동안은 서버 확인 없이 캐시 사용, 만료 후에는 반드시 재검증(중간 프록시도 포함)하라는 의미입니다. `must-revalidate`는 캐시가 만료됐는데 서버가 응답하지 않으면 504를 반환해야 합니다. `max-age=3600, no-cache`는 캐시는 저장하되 매 요청마다 재검증이 필요합니다. 실질적으로 max-age가 무의미해지는 차이가 있습니다. 현실에서는 `max-age + must-revalidate`가 "N초 후 반드시 확인"의 의미로 더 많이 사용됩니다.
>
> **Q2.** gzip 압축 필터가 `ShallowEtagHeaderFilter` 이후에 실행되면 문제가 없습니다. 하지만 gzip 필터가 먼저 실행되면 `ShallowEtagHeaderFilter`는 압축된 본문의 해시를 계산합니다. 클라이언트가 `Accept-Encoding: gzip` 없이 요청하면 비압축 응답을 받고, 다음 요청에서 `gzip`을 지원하면 다른 ETag를 받아 항상 200이 됩니다. 해결: `ShallowEtagHeaderFilter`를 Filter 체인에서 gzip 필터보다 앞에 등록합니다.
>
> **Q3.** `checkNotModified("v42")`는 현재 버전 `"v44"`와 비교하므로 미매칭 → 200 OK가 반환됩니다. ETag는 내용(content)이 아닌 버전 식별자로 동작하기 때문입니다. 실제 내용이 같아도 버전이 다르면 새 응답을 받습니다. 이 문제를 해결하려면 버전 대신 실제 내용의 해시를 ETag로 사용하는 방법(`ShallowEtagHeaderFilter` 방식)이 있지만, 항상 트레이드오프가 존재합니다.

---

<div align="center">

**[⬅️ 이전: Static Resources](./04-static-resources-handling.md)** | **[홈으로 🏠](../README.md)** | **[다음: WebMvcConfigurer 커스터마이징 ➡️](./06-web-mvc-configurer.md)**

</div>
