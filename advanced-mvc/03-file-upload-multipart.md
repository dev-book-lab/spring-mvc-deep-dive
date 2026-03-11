# File Upload — MultipartResolver와 MultipartFile 파싱 과정

---

## 🎯 핵심 질문

- `StandardServletMultipartResolver`가 `multipart/form-data`를 파싱해 `MultipartFile`을 생성하는 과정은?
- 파일 크기 제한(`maxFileSize`, `maxRequestSize`)이 적용되는 정확한 시점은?
- `MultipartFile`과 임시 파일의 관계, 그리고 업로드된 파일이 실제로 저장되기 전까지의 생명주기는?
- `@RequestParam MultipartFile` vs `@RequestPart MultipartFile`의 처리 경로 차이는?
- 파일 업로드 중 예외(`MaxUploadSizeExceededException`)가 발생하는 위치는?
- 대용량 파일 업로드 시 성능을 위한 설정 옵션은?

---

## 🔍 왜 이 메커니즘이 존재하는가

```
파일 업로드 = multipart/form-data 인코딩:
  Content-Type: multipart/form-data; boundary=----FormBoundary
  ----FormBoundary
  Content-Disposition: form-data; name="name"
  홍길동
  ----FormBoundary
  Content-Disposition: form-data; name="file"; filename="photo.jpg"
  Content-Type: image/jpeg
  [바이너리 데이터...]
  ----FormBoundary--

문제:
  기본 HttpServletRequest.getParameter()는 multipart 파싱 불가
  파일 크기 제한 없으면 OOM/디스크 고갈 위험
  임시 저장 위치 관리 필요

해결: MultipartResolver
  multipart 요청 감지 → 파싱 → MultipartHttpServletRequest 래핑
  각 파트(part)를 MultipartFile 객체로 추상화
```

---

## 😱 흔한 오해 또는 실수

### Before: maxFileSize를 설정하면 DispatcherServlet에서 크기 체크를 한다

```
❌ 잘못된 이해:
  "maxFileSize 설정 → DispatcherServlet에서 확인 후 거부"

✅ 실제:
  크기 제한은 MultipartResolver.resolveMultipart() 단계에서 적용
  → DispatcherServlet.checkMultipart() 호출 시
  → 즉, HandlerMapping, Interceptor 실행 전에 이미 체크됨
  → 초과 시 MaxUploadSizeExceededException 발생
  → @ExceptionHandler로 처리 가능

  Spring Boot의 경우:
  spring.servlet.multipart.max-file-size=10MB    # 파일 하나의 최대 크기
  spring.servlet.multipart.max-request-size=50MB # 요청 전체 최대 크기

  Tomcat 레벨에서도 제한:
  server.tomcat.max-http-post-size (기본 2MB)
  → 이 값이 더 낮으면 Tomcat이 먼저 거부 (Spring 이전)
```

### Before: MultipartFile.getBytes()로 파일을 메모리에 읽으면 항상 안전하다

```java
// ❌ 대용량 파일 시 OOM 위험
@PostMapping("/upload")
public String upload(@RequestParam MultipartFile file) {
    byte[] bytes = file.getBytes();  // 전체 파일을 메모리에 로드!
    fileService.save(bytes);  // 수백 MB 파일 → OOM
}

// ✅ 올바른 방법: 스트림으로 처리
@PostMapping("/upload")
public String upload(@RequestParam MultipartFile file) {
    try (InputStream inputStream = file.getInputStream()) {
        Files.copy(inputStream, targetPath, StandardCopyOption.REPLACE_EXISTING);
    }
    // 또는
    file.transferTo(targetPath.toFile());
    // → 내부적으로 스트림 복사, 메모리 효율적
}
```

---

## ✨ 올바른 이해와 사용

### After: 파일 업로드 전체 처리 흐름

```
POST /api/upload HTTP/1.1
Content-Type: multipart/form-data; boundary=----Boundary123
[파일 데이터...]

① DispatcherServlet.checkMultipart():
   multipartResolver.isMultipart(request) → true
   multipartResolver.resolveMultipart(request)
   → StandardServletMultipartResolver:
      request.getParts() (Servlet 3.0 Part API 사용)
      → Tomcat이 parts 파싱 + 임시 파일 생성
      → StandardMultipartFile 객체 생성 (Part 래핑)
   → MultipartHttpServletRequest 반환

② HandlerMapping → Controller 매핑
③ Interceptor preHandle

④ ArgumentResolver:
   @RequestParam MultipartFile file
   → RequestParamMethodArgumentResolver
   → MultipartHttpServletRequest.getFile("file")
   → MultipartFile 반환

⑤ Controller 실행:
   file.getOriginalFilename()  // "photo.jpg"
   file.getContentType()       // "image/jpeg"
   file.getSize()              // 1048576 (bytes)
   file.transferTo(path)       // 임시 파일 → 목적지 복사

⑥ 임시 파일 정리:
   요청 완료 후 MultipartHttpServletRequest.cleanupMultipart()
   → 임시 파일 삭제
```

---

## 🔬 내부 동작 원리

### 1. StandardServletMultipartResolver — Servlet Part API 활용

```java
// StandardServletMultipartResolver.java
@Override
public MultipartHttpServletRequest resolveMultipart(HttpServletRequest request)
        throws MultipartException {
    return new StandardMultipartHttpServletRequest(request, this.resolveLazily);
    // resolveLazily=true: 실제 파라미터 접근 시 파싱 (기본 false)
}

// StandardMultipartHttpServletRequest 생성 시:
private void parseRequest(HttpServletRequest request) {
    try {
        Collection<Part> parts = request.getParts();
        // → Tomcat의 ApplicationPart (javax.servlet.http.Part)
        // → 크기 제한 체크 (maxFileSize, maxRequestSize)
        //    초과 시 → IllegalStateException → MaxUploadSizeExceededException

        MultiValueMap<String, MultipartFile> files = new LinkedMultiValueMap<>();
        Map<String, String[]> params = new LinkedHashMap<>();

        for (Part part : parts) {
            String headerValue = part.getHeader(HttpHeaders.CONTENT_DISPOSITION);
            ContentDisposition disposition = ContentDisposition.parse(headerValue);
            String filename = disposition.getFilename();

            if (filename != null) {
                // 파일 파트
                files.add(part.getName(), new StandardMultipartFile(part, filename));
            } else {
                // 텍스트 파라미터 파트
                String[] values = params.get(part.getName());
                params.put(part.getName(), addToArray(values, new String(part.getInputStream().readAllBytes())));
            }
        }
        setMultipartFiles(files);
        setMultipartParameters(params);
    } catch (IllegalStateException ex) {
        throw new MaxUploadSizeExceededException(-1, ex);
    }
}
```

### 2. StandardMultipartFile — Part 래퍼

```java
// StandardMultipartFile.java
public class StandardMultipartFile implements MultipartFile {

    private final Part part;  // Servlet Part (임시 파일 또는 메모리 보유)
    private final String filename;

    @Override
    public String getName() { return this.part.getName(); }

    @Override
    public String getOriginalFilename() {
        // Part의 Content-Disposition에서 filename 추출
        // ※ 브라우저가 전송한 파일 이름 (파일 시스템에 직접 사용 시 보안 주의)
        return this.filename;
    }

    @Override
    public long getSize() { return this.part.getSize(); }

    @Override
    public byte[] getBytes() throws IOException {
        return FileCopyUtils.copyToByteArray(this.part.getInputStream());
    }

    @Override
    public InputStream getInputStream() throws IOException {
        return this.part.getInputStream();
    }

    @Override
    public void transferTo(File dest) throws IOException, IllegalStateException {
        this.part.write(dest.getPath());
        // → Servlet Part.write() : 임시 파일을 목적지로 이동 (rename or copy)
    }
}
```

### 3. @RequestParam vs @RequestPart 차이

```java
// @RequestParam MultipartFile:
//   RequestParamMethodArgumentResolver가 처리
//   MultipartHttpServletRequest.getFile(name)으로 조회
//   → 단순 파일 업로드에 적합

// @RequestPart:
//   RequestPartMethodArgumentResolver가 처리
//   request.getPart(name) → Part 객체
//   → Content-Type에 따라 HttpMessageConverter로 역직렬화 가능!
//   → 파일 + JSON 본문을 같은 요청에 받을 때 유용

@PostMapping("/upload-with-metadata")
public ResponseEntity<String> uploadWithMetadata(
    @RequestPart("file") MultipartFile file,        // 파일
    @RequestPart("metadata") UserMetadata metadata  // JSON으로 역직렬화
) {
    // metadata는 Content-Type: application/json 파트
    // → MappingJackson2HttpMessageConverter로 UserMetadata 역직렬화
    fileService.save(file, metadata);
    return ResponseEntity.ok("업로드 완료");
}
```

### 4. 보안 고려사항 — 파일명 처리

```java
// 원본 파일명을 직접 사용하면 보안 취약점!
// path traversal: "../../etc/passwd"
// null byte injection: "file.jpg\0.exe"

@PostMapping("/upload")
public ResponseEntity<String> upload(@RequestParam MultipartFile file) {
    String originalFilename = file.getOriginalFilename();

    // ① 파일명 정제
    String cleanFilename = StringUtils.cleanPath(originalFilename);
    if (cleanFilename.contains("..")) {
        throw new IllegalArgumentException("잘못된 파일명: " + cleanFilename);
    }

    // ② 새 파일명 생성 (UUID 권장)
    String extension = FilenameUtils.getExtension(cleanFilename);
    String savedFilename = UUID.randomUUID() + "." + extension;

    // ③ 허용 확장자 화이트리스트
    Set<String> allowedExtensions = Set.of("jpg", "jpeg", "png", "pdf");
    if (!allowedExtensions.contains(extension.toLowerCase())) {
        throw new IllegalArgumentException("허용되지 않는 파일 형식: " + extension);
    }

    // ④ Content-Type 검증 (MIME sniffing 방어)
    String contentType = file.getContentType();
    // file magic bytes 검사 (Tika 라이브러리 등)

    Path targetPath = uploadDir.resolve(savedFilename);
    file.transferTo(targetPath);
    return ResponseEntity.ok(savedFilename);
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 파일 업로드 크기 제한 확인

```yaml
# application.yml
spring:
  servlet:
    multipart:
      max-file-size: 5MB       # 단일 파일 최대 크기
      max-request-size: 20MB   # 요청 전체 최대 크기
      file-size-threshold: 2MB # 이 크기 이하: 메모리 보관, 이상: 임시 파일
      location: /tmp/uploads   # 임시 파일 저장 위치
```

```bash
# 5MB 초과 파일 업로드 시도
curl -F "file=@large.jpg" http://localhost:8080/api/upload
# HTTP/1.1 413 Payload Too Large (또는 400)
# → MaxUploadSizeExceededException → @ExceptionHandler 처리
```

### 실험 2: 임시 파일 생성 확인

```java
@PostMapping("/upload-debug")
public String uploadDebug(@RequestParam MultipartFile file) throws IOException {
    // 임시 파일 경로 확인
    if (file instanceof StandardMultipartFile smf) {
        Part part = (Part) /* 리플렉션으로 part 필드 접근 */ ...;
        log.info("Part 클래스: {}", part.getClass().getName());
        // Tomcat: ApplicationPart → DiskFileItem (commons-fileupload) 또는 java.io.tmpdir
    }
    return "size=" + file.getSize();
}
```

---

## 🌐 HTTP 레벨 분석

```
POST /api/upload HTTP/1.1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary
Content-Length: 1048576

------WebKitFormBoundary
Content-Disposition: form-data; name="name"
홍길동
------WebKitFormBoundary
Content-Disposition: form-data; name="file"; filename="photo.jpg"
Content-Type: image/jpeg
[이진 데이터 1MB]
------WebKitFormBoundary--

① Tomcat: multipart 파싱
   → /tmp/tomcat-upload-xxx.tmp 임시 파일 생성
   → ApplicationPart 객체 생성

② DispatcherServlet.checkMultipart():
   maxFileSize=5MB 검사 → 1MB < 5MB → 통과
   StandardMultipartHttpServletRequest 생성

③ ArgumentResolver:
   getFile("file") → StandardMultipartFile{part=ApplicationPart, filename="photo.jpg"}

④ Controller:
   file.transferTo(Path.of("/uploads/abc123.jpg"))
   → Part.write("/uploads/abc123.jpg")
   → 임시 파일 이동 (rename) 또는 복사

⑤ 요청 완료:
   cleanupMultipart() → /tmp/tomcat-upload-xxx.tmp 삭제

HTTP/1.1 200 OK
{"filename":"abc123.jpg","size":1048576}
```

---

## 📌 핵심 정리

```
파싱 흐름
  DispatcherServlet.checkMultipart()
  → StandardServletMultipartResolver.resolveMultipart()
  → request.getParts() (Servlet Part API, Tomcat 처리)
  → StandardMultipartFile 생성 (Part 래핑)
  → MultipartHttpServletRequest 반환

크기 제한 적용 시점
  request.getParts() 호출 시 Tomcat이 파싱 중 체크
  → HandlerMapping 전에 이미 예외 가능
  → MaxUploadSizeExceededException → @ExceptionHandler

임시 파일 생명주기
  파싱 시 생성 → transferTo() 또는 직접 읽기 → 요청 완료 후 삭제
  file-size-threshold: 메모리/디스크 기준 크기

보안
  originalFilename 직접 사용 금지 (Path Traversal)
  UUID 파일명 생성 권장
  확장자/Content-Type 화이트리스트 검증

@RequestPart vs @RequestParam
  @RequestParam: 단순 파일, MultipartFile 직접
  @RequestPart: 파일 + JSON 본문 혼합, MessageConverter 역직렬화
```

---

## 🤔 생각해볼 문제

**Q1.** `file.transferTo(File dest)`와 `Files.copy(file.getInputStream(), path)` 두 방법의 성능 차이는?

**Q2.** `multipart.file-size-threshold`를 0으로 설정하면 어떤 일이 발생하는가?

**Q3.** 파일 업로드 중 클라이언트가 연결을 끊으면 서버의 임시 파일은 어떻게 되는가?

> 💡 **해설**
>
> **Q1.** `transferTo(File)`가 일반적으로 더 효율적입니다. Servlet Part의 `write()` 메서드는 내부적으로 임시 파일이 이미 존재하면 OS 레벨의 파일 이동(`rename` 시스템 콜)을 수행합니다. `rename`은 같은 파일 시스템 내에서는 O(1) 메타데이터 수정만으로 완료됩니다. 반면 `Files.copy(getInputStream(), path)`는 항상 데이터를 바이트 단위로 복사하므로 I/O 비용이 더 높습니다. 단, 목적지가 다른 파일 시스템이면 `transferTo`도 복사를 수행합니다.
>
> **Q2.** 모든 파일이 크기와 무관하게 임시 파일에 저장됩니다. 메모리 버퍼를 사용하지 않으므로 메모리 사용량은 줄지만, 작은 파일도 디스크 I/O가 발생합니다. 반대로 `threshold`를 높게 설정하면 작은 파일은 메모리에만 유지되어 디스크 쓰기가 없습니다. 실무에서는 파일 크기와 메모리 여유에 따라 적절한 값을 설정합니다.
>
> **Q3.** 연결 끊김 시 Tomcat은 현재 파싱 중이던 파트 처리를 중단하고 이미 생성된 임시 파일은 즉시 삭제되지 않을 수 있습니다. 이후 요청 처리가 완료되거나 실패하면 `cleanupMultipart()`가 호출되어 임시 파일이 삭제됩니다. 비정상적인 경우(JVM 크래시 등) 임시 디렉토리에 남아 있을 수 있어 주기적인 임시 파일 정리가 필요합니다.

---

<div align="center">

**[⬅️ 이전: Server-Sent Events](./02-server-sent-events.md)** | **[홈으로 🏠](../README.md)** | **[다음: Static Resources Handling ➡️](./04-static-resources-handling.md)**

</div>
