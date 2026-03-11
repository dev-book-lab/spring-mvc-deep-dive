# Static Resources Handling — ResourceHttpRequestHandler와 ResourceResolver 체인

---

## 🎯 핵심 질문

- `ResourceHttpRequestHandler`가 클래스패스·파일시스템에서 정적 리소스를 찾는 내부 경로는?
- `ResourceResolver` 체인의 구성과 각 Resolver의 책임은?
- WebJars 버전 중립 경로(`/webjars/jquery/jquery.min.js`)가 실제 버전 경로로 변환되는 방법은?
- `ResourceTransformer`가 CSS/JS의 URL 참조를 재작성하는 원리는?
- `Cache-Control` 헤더와 `Last-Modified` 기반 캐싱이 정적 리소스에 어떻게 적용되는가?
- Spring Boot의 기본 정적 리소스 경로와 우선순위는?

---

## 🔍 왜 이 메커니즘이 존재하는가

```
정적 리소스 제공의 요구사항:
  파일 존재 여부 확인 → 없으면 404
  Content-Type 추론 (image/jpeg, text/css 등)
  캐싱 헤더 (Cache-Control, ETag, Last-Modified)
  버전 관리 (파일 내용 변경 시 캐시 무효화)
  WebJars 지원 (Maven 의존성으로 프론트엔드 라이브러리 관리)

DispatcherServlet vs DefaultServlet:
  전통적으로 정적 파일은 DefaultServlet(Tomcat)이 처리
  → Spring MVC Controller와 URL 충돌 가능
  → ResourceHttpRequestHandler: Spring 내에서 정적 리소스 직접 처리
  → 일관된 캐싱/보안 정책 적용 가능
```

---

## 😱 흔한 오해 또는 실수

### Before: addResourceHandlers()에서 classpath를 지정할 때 경로 끝에 /가 필요없다

```java
// ❌ 잘못된 설정
registry.addResourceHandler("/static/**")
    .addResourceLocations("classpath:/static");  // 끝에 / 없음

// ✅ 올바른 설정
registry.addResourceHandler("/static/**")
    .addResourceLocations("classpath:/static/");  // 끝에 / 필수!

// 이유:
// ClassPathResource("static/") vs ClassPathResource("static")
// → "/" 없으면 "static" 디렉토리 자체를 리소스로 인식
// → "static/image.png" 조합 시 "staticimage.png"가 되어 파일을 찾을 수 없음
// Spring Boot 기본값도 모두 "/" 로 끝남:
//   "classpath:/META-INF/resources/"
//   "classpath:/resources/"
//   "classpath:/static/"
//   "classpath:/public/"
```

### Before: WebJars는 classpath 탐색으로 자동으로 버전이 해석된다

```
❌ 잘못된 이해: "WebJars 경로를 쓰면 자동으로 동작한다"

✅ 실제:
  WebJarsResourceResolver가 ResourceResolver 체인에 추가돼야 함
  Spring Boot: WebMvcAutoConfiguration이 자동 추가
  직접 설정 시:
    registry.addResourceHandler("/webjars/**")
        .addResourceLocations("classpath:/META-INF/resources/webjars/")
        .resourceChain(true)
        .addResolver(new WebJarsResourceResolver());
```

---

## ✨ 올바른 이해와 사용

### After: 정적 리소스 처리 전체 흐름

```
GET /static/css/app.css HTTP/1.1

① DispatcherServlet → HandlerMapping
   SimpleUrlHandlerMapping: "/static/**" → ResourceHttpRequestHandler

② ResourceHttpRequestHandler.handleRequest():
   ResourceResolver 체인으로 리소스 탐색:
     EncodedResourceResolver (gzip/brotli 압축 버전 탐색)
     → CachingResourceResolver (캐시 확인)
     → PathResourceResolver (실제 파일시스템/클래스패스 탐색)
   → Resource 찾으면 반환, 없으면 null

③ 리소스 존재 확인 및 응답:
   Last-Modified 헤더 확인 → 304 Not Modified 반환 가능
   Content-Type 추론 (MediaTypeFactory 사용)
   Cache-Control 헤더 설정
   응답 본문 스트리밍

HTTP/1.1 200 OK
Content-Type: text/css
Cache-Control: max-age=3600
Last-Modified: Mon, 01 Jan 2024 00:00:00 GMT
[CSS 내용...]
```

---

## 🔬 내부 동작 원리

### 1. Spring Boot 기본 정적 리소스 경로 우선순위

```java
// WebMvcAutoConfiguration.addResourceHandlers()
// 기본 경로 (우선순위 순):
// 1. classpath:/META-INF/resources/  (WebJars 등)
// 2. classpath:/resources/
// 3. classpath:/static/              (가장 일반적으로 사용)
// 4. classpath:/public/

// 커스터마이징 (application.yml):
// spring.web.resources.static-locations=classpath:/custom/,classpath:/static/
// spring.mvc.static-path-pattern=/static/**  (기본: /**)
```

### 2. ResourceResolver 체인 — 리소스 탐색 알고리즘

```java
// ResourceHttpRequestHandler.resolveResource()
@Nullable
private Resource resolveResource(@Nullable HttpServletRequest request,
        String requestPath) {
    for (ResourceResolver resolver : getResourceResolvers()) {
        Resource resolved = resolver.resolveResource(request, requestPath,
            getLocations(), this.resourceResolverChain);
        if (resolved != null) return resolved;
    }
    return null;
}

// ResourceResolverChain: 체인 패턴으로 다음 Resolver에 위임

// 1. CachingResourceResolver
//    → 이전에 탐색한 경로를 캐시 (ConcurrentHashMap)
//    → 캐시 히트 시 즉시 반환

// 2. EncodedResourceResolver (Spring 5.1+)
//    → Accept-Encoding: gzip 헤더 확인
//    → "app.css.gz" 파일이 있으면 압축 버전 반환
//    → Content-Encoding: gzip 헤더 추가

// 3. WebJarsResourceResolver
//    → "/webjars/jquery/jquery.min.js" 요청 시
//    → classpath에서 실제 버전 탐색:
//       "jquery/3.7.1/jquery.min.js" 발견
//    → 버전 포함 경로로 변환

// 4. PathResourceResolver (최종 탐색)
//    → 각 ResourceLocation에서 파일 탐색
//    → 보안: 상위 디렉토리 이동(..) 방지
//    → classpath:/static/ + "css/app.css" → 실제 파일
```

### 3. WebJarsResourceResolver 동작

```java
// WebJarsResourceResolver.java
@Override
@Nullable
public Resource resolveResource(@Nullable HttpServletRequest request,
        String requestPath, List<? extends Resource> locations,
        ResourceResolverChain chain) {

    Resource resolved = chain.resolveResource(request, requestPath, locations);
    if (resolved == null) {
        // "/webjars/jquery/jquery.min.js" 요청
        // → classpath에서 "META-INF/resources/webjars/jquery/" 탐색
        // → 버전 디렉토리 발견: "3.7.1"
        // → "META-INF/resources/webjars/jquery/3.7.1/jquery.min.js" 반환
        String webJarsResourcePath = findWebJarResourcePath(requestPath);
        if (webJarsResourcePath != null) {
            return chain.resolveResource(request, webJarsResourcePath, locations);
        }
    }
    return resolved;
}

// 사용 패턴:
// Maven 의존성:
// <dependency>
//   <groupId>org.webjars</groupId>
//   <artifactId>jquery</artifactId>
//   <version>3.7.1</version>
// </dependency>

// HTML에서 버전 중립 경로 사용:
// <script src="/webjars/jquery/jquery.min.js"></script>
// → 버전 업그레이드 시 pom.xml만 변경, HTML 수정 불필요
```

### 4. ResourceTransformer — CSS/JS URL 재작성

```java
// ContentVersionStrategy: 파일 내용 기반 해시 버전
// CssLinkResourceTransformer: CSS 내 url() 참조 재작성

@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    VersionResourceResolver versionResolver = new VersionResourceResolver()
        .addContentVersionStrategy("/**");  // 파일 내용 해시를 URL에 포함

    registry.addResourceHandler("/static/**")
        .addResourceLocations("classpath:/static/")
        .setCacheControl(CacheControl.maxAge(365, TimeUnit.DAYS).cachePublic())
        .resourceChain(true)
        .addResolver(versionResolver)
        .addTransformer(new CssLinkResourceTransformer());
        // CssLinkResourceTransformer: CSS 내 url("image.png") →
        //   url("image-abc123.png") 자동 재작성
}

// 결과:
// app.css → /static/css/app-d41d8cd9.css (해시 포함)
// → Cache-Control: max-age=365d → 긴 캐싱
// → 내용 변경 시 해시 변경 → 자동 캐시 무효화 (Cache Busting)
```

### 5. Last-Modified 기반 조건부 요청

```java
// ResourceHttpRequestHandler.handleRequest()
@Override
public void handleRequest(HttpServletRequest request, HttpServletResponse response)
        throws IOException {
    Resource resource = getResource(request);

    // 리소스 없음 → 404
    if (resource == null) {
        response.sendError(HttpServletResponse.SC_NOT_FOUND);
        return;
    }

    // HEAD 요청 처리
    if (HttpMethod.HEAD.name().equals(request.getMethod())) {
        setHeaders(response, resource, mediaType);
        return;
    }

    // 조건부 요청 처리 (If-None-Match, If-Modified-Since)
    if (new ServletWebRequest(request, response).checkNotModified(resource.lastModified())) {
        return;  // 304 Not Modified 자동 반환
    }

    // 응답 헤더 설정
    setHeaders(response, resource, mediaType);
    // 응답 본문 스트리밍
    resource.getInputStream().transferTo(response.getOutputStream());
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 캐싱 헤더 확인

```yaml
# application.yml
spring:
  web:
    resources:
      cache:
        period: 3600        # Cache-Control: max-age=3600
        cachecontrol:
          max-age: 3600
          cache-public: true
```

```bash
# 첫 번째 요청
curl -i http://localhost:8080/static/app.css
# HTTP/1.1 200 OK
# Cache-Control: max-age=3600, public
# Last-Modified: Mon, 01 Jan 2024 00:00:00 GMT
# ETag: "abc123"

# 두 번째 요청 (캐시 검증)
curl -i -H "If-None-Match: \"abc123\"" http://localhost:8080/static/app.css
# HTTP/1.1 304 Not Modified
# → 응답 본문 없음 (캐시 사용)
```

### 실험 2: ResourceResolver 체인 로그

```yaml
logging:
  level:
    org.springframework.web.servlet.resource: TRACE
```

```
TRACE PathResourceResolver - Resolved resource from location 'class path resource [static/]'
TRACE CachingResourceResolver - Resolved to cached resource: classpath:/static/css/app.css
```

---

## 🌐 HTTP 레벨 분석

```
GET /static/css/app.css HTTP/1.1
If-None-Match: "d41d8cd9"
If-Modified-Since: Mon, 01 Jan 2024 00:00:00 GMT

ResourceHttpRequestHandler:
  CachingResourceResolver → 캐시 미스
  PathResourceResolver → classpath:/static/css/app.css 발견

  checkNotModified(lastModified):
    resource.lastModified() = 2024-01-01 00:00:00
    If-Modified-Since와 비교 → 동일
    → 304 Not Modified 자동 반환

HTTP/1.1 304 Not Modified
ETag: "d41d8cd9"
Cache-Control: max-age=3600, public
[응답 본문 없음 → 클라이언트 캐시 사용]

--- 파일 변경 후 요청 ---

HTTP/1.1 200 OK
Content-Type: text/css
ETag: "abc987ef"  ← 새 해시
Cache-Control: max-age=3600, public
Last-Modified: Tue, 02 Jan 2024 12:00:00 GMT
[새 CSS 내용...]
```

---

## 📌 핵심 정리

```
Spring Boot 기본 경로 우선순위
  classpath:/META-INF/resources/ > /resources/ > /static/ > /public/
  spring.web.resources.static-locations로 커스터마이징

ResourceResolver 체인
  CachingResourceResolver → 이전 탐색 결과 캐시
  EncodedResourceResolver → gzip/brotli 압축 버전
  WebJarsResourceResolver → 버전 중립 WebJars 경로 해석
  PathResourceResolver    → 실제 파일 탐색 (최종)

addResourceLocations() 경로 끝 / 필수
  "classpath:/static/" ← 맞음
  "classpath:/static"  ← 틀림 (파일 못 찾음)

Cache Busting
  VersionResourceResolver + ContentVersionStrategy
  → 파일 내용 해시를 URL에 포함
  → 장기 캐싱 + 자동 무효화

조건부 요청
  Last-Modified / If-Modified-Since → 304
  ETag / If-None-Match → 304
  → 응답 본문 없이 캐시 검증
```

---

## 🤔 생각해볼 문제

**Q1.** `PathResourceResolver`의 보안 처리는 어떻게 동작하는가? `/static/../WEB-INF/web.xml` 같은 path traversal 시도를 어떻게 방어하는가?

**Q2.** `CachingResourceResolver`의 캐시는 언제 초기화(eviction)되는가? 운영 중 정적 파일을 교체했을 때 캐시를 갱신하는 방법은?

**Q3.** `ResourceHttpRequestHandler`가 처리하는 경로에 `HandlerInterceptor`가 적용되는가?

> 💡 **해설**
>
> **Q1.** `PathResourceResolver.isResourceUnderLocation()`에서 요청된 Resource의 실제 절대 경로가 허용된 Location의 절대 경로 아래에 있는지 확인합니다. `resource.getURL().toExternalForm()`으로 정규화된 실제 경로를 얻고, `location.getURL().toExternalForm()`으로 시작하는지(`startsWith`) 검사합니다. `../` 같은 상위 디렉토리 이동은 정규화 후 Location 경로를 벗어나므로 거부됩니다. 또한 Spring은 `StringUtils.cleanPath()`로 경로를 먼저 정규화합니다.
>
> **Q2.** `CachingResourceResolver`의 캐시는 기본적으로 만료 시간이 없습니다(`ConcurrentHashMap` 기반). 애플리케이션 재시작 시 초기화됩니다. 운영 중 캐시를 갱신하려면: `ResourceUrlProvider.clearCache()`를 호출하거나, `CaffeineCacheManager`나 `ConcurrentMapCacheManager`를 사용하는 경우 해당 캐시를 직접 evict합니다. 실무에서는 `VersionResourceResolver`와 배포 시 파일 해시 변경으로 자동 무효화하는 것이 더 안정적입니다.
>
> **Q3.** 적용됩니다. `ResourceHttpRequestHandler`도 `HandlerMapping`에서 찾은 Handler이므로 `HandlerExecutionChain`에 Interceptor가 추가됩니다. 단, `addPathPatterns`로 특정 경로만 지정했다면 정적 리소스 경로가 제외될 수 있습니다. `/**`로 등록된 Interceptor는 정적 리소스 요청에도 실행됩니다. 정적 리소스에는 Interceptor를 적용하지 않으려면 `excludePathPatterns("/static/**")` 등으로 명시 제외해야 합니다.

---

<div align="center">

**[⬅️ 이전: File Upload](./03-file-upload-multipart.md)** | **[홈으로 🏠](../README.md)** | **[다음: HTTP Caching ➡️](./05-http-caching.md)**

</div>
