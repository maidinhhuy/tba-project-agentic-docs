# Story 8.6: BE — Presigned S3 Upload URL Endpoint

**Status:** ready-for-dev
**Story ID:** 8.6
**Epic:** 8 — Milestone Delivery & Customer Review

---

## Story

As a developer,
I want the Admin deliver endpoint to accept a pre-uploaded S3 key instead of a streamed file,
So that browser clients can upload files directly to S3 and bypass the multipart-forwarding Content-Length bug in Node.js fetch (undici).

---

## Root Cause (đọc kỹ trước khi code)

`S3FileStorageAdapter.upload()` gọi `RequestBody.fromInputStream(data, -1)` với content length = -1. Khi Next.js server action forward multipart FormData đến Spring Boot qua undici, undici không tính được `Content-Length` cho streamed file entries. Spring Boot multipart parser nhận negative Content-Length và trả về `400 BAD_REQUEST: "Content-length must not be negative"`.

**Fix**: tách upload khỏi deliver call. Browser PUT trực tiếp lên S3 qua presigned URL → deliver chỉ cần gửi `s3Key`.

---

## Acceptance Criteria

**AC-1** — Port method mới:
**Given** `FileStoragePort` (core/port/out),
**When** inspected sau story này,
**Then** có thêm method: `GenerateUploadUrlResult generatePresignedUploadUrl(String fileName, String contentType, String projectId, String milestoneId)`.
**And** `GenerateUploadUrlResult` là record `{ String uploadUrl, String s3Key }` trong `core/port/bound/`.

**AC-2** — Adapter implement presigned PUT:
**Given** `S3FileStorageAdapter` implements method mới,
**When** gọi với params hợp lệ,
**Then** generate presigned `PutObject` URL qua `S3Presigner.presignPutObject` với TTL = 15 phút.
**And** s3Key format: `deliverables/{projectId}/{milestoneId}/{uuid}-{fileName}`.
**And** presigned URL include Content-Type constraint matching `contentType` param.

**AC-3** — Endpoint mới request-upload-url:
**Given** `POST /api/admin/milestones/{milestoneId}/request-upload-url` với body `{ "fileName": "...", "contentType": "..." }`,
**When** Admin authenticated (`hasRole("ADMIN")`),
**Then** response 200 `{ "uploadUrl": "...", "s3Key": "..." }`.
**And** nếu `fileName` hoặc `contentType` blank → 400 `VALIDATION_FAILED`.
**And** nếu milestone không tồn tại → 404.

**AC-4** — Deliver endpoint chuyển sang JSON:
**Given** `POST /api/admin/milestones/{milestoneId}/deliver`,
**When** updated,
**Then** accept `Content-Type: application/json` body: `{ "deliverableUrl": "...", "deliverableS3Key": "...", "deliverableFileName": "...", "deliveryNote": "..." }`.
**And** tất cả fields đều optional riêng lẻ, nhưng ít nhất 1 trong `deliverableUrl` hoặc `deliverableS3Key` phải non-blank.
**And** không còn import `MultipartFile` hay `MultipartHttpServletRequest`.

**AC-5** — Command updated:
**Given** `DeliverMilestoneCommand` (core/port/bound),
**When** updated,
**Then** field `InputStream deliverableFile` bị **xóa**; field `String deliverableContentType` bị **xóa**; field `String deliverableS3Key` (nullable) được thêm vào.
**And** field `deliverableFileName` được giữ nguyên (vẫn cần lưu vào DB).

**AC-6** — Service bỏ upload step:
**Given** `DeliverMilestoneService.deliver()`,
**When** updated,
**Then** toàn bộ block `if (command.deliverableFile() != null) { fileStorage.upload(...) }` bị **xóa**.
**And** `milestoneRepository.saveDelivery(milestoneId, deliverableUrl, command.deliverableS3Key(), deliverableFileName, deliveryNote)` gọi trực tiếp với `deliverableS3Key` từ command.

---

## Tasks / Subtasks

### BE Core Layer

- [ ] **Task 1: Tạo `GenerateUploadUrlResult` record**
  - File: `modules/core/src/main/java/com/tba/agentic/port/bound/GenerateUploadUrlResult.java`
  ```java
  package com.tba.agentic.port.bound;

  public record GenerateUploadUrlResult(String uploadUrl, String s3Key) {}
  ```

- [ ] **Task 2: Thêm method vào `FileStoragePort`**
  - File: `modules/core/src/main/java/com/tba/agentic/port/out/FileStoragePort.java`
  - Thêm method signature (không xóa `upload()` hay `generatePresignedUrl()` — vẫn dùng bởi story 8.3):
  ```java
  import com.tba.agentic.port.bound.GenerateUploadUrlResult;

  GenerateUploadUrlResult generatePresignedUploadUrl(
      String fileName, String contentType, String projectId, String milestoneId);
  ```

- [ ] **Task 3: Cập nhật `DeliverMilestoneCommand` record**
  - File: `modules/core/src/main/java/com/tba/agentic/port/bound/DeliverMilestoneCommand.java`
  - **Xóa**: `String deliverableContentType`, `InputStream deliverableFile`
  - **Thêm**: `String deliverableS3Key` (nullable)
  - **Kết quả**:
  ```java
  package com.tba.agentic.port.bound;

  public record DeliverMilestoneCommand(
      Long milestoneId,
      String deliverableUrl,
      String deliverableFileName,
      String deliverableS3Key,
      String deliveryNote,
      Long adminUserId) {}
  ```

### BE Application Layer

- [ ] **Task 4: Cập nhật `DeliverMilestoneService`**
  - File: `modules/application/src/main/java/com/tba/agentic/application/service/DeliverMilestoneService.java`
  - **Xóa** toàn bộ block:
    ```java
    String fileKey = null;
    if (command.deliverableFile() != null) {
      fileKey = fileStorage.upload(...);
    }
    ```
  - **Đổi** `milestoneRepository.saveDelivery()` dùng `command.deliverableS3Key()` thay vì `fileKey`:
    ```java
    milestoneRepository.saveDelivery(
        command.milestoneId(),
        command.deliverableUrl(),
        command.deliverableS3Key(),
        command.deliverableFileName(),
        command.deliveryNote());
    ```
  - Field `fileStorage` vẫn được inject (dùng bởi `generatePresignedUrl` ở story 8.3) — **không xóa** khỏi constructor.
  - **Chú ý**: `FileStoragePort` import vẫn cần thiết. Chỉ xóa block upload, không xóa dependency.

### BE Infrastructure Layer

- [ ] **Task 5: Implement `generatePresignedUploadUrl` trong `S3FileStorageAdapter`**
  - File: `modules/infrastructure/src/main/java/com/tba/agentic/infrastructure/storage/S3FileStorageAdapter.java`
  - Thêm import:
    ```java
    import com.tba.agentic.port.bound.GenerateUploadUrlResult;
    import software.amazon.awssdk.services.s3.presigner.model.PutObjectPresignRequest;
    import software.amazon.awssdk.services.s3.model.PutObjectRequest;
    ```
  - Thêm method:
    ```java
    @Override
    public GenerateUploadUrlResult generatePresignedUploadUrl(
        String fileName, String contentType, String projectId, String milestoneId) {
      String fileKey = "deliverables/" + projectId + "/" + milestoneId
          + "/" + UUID.randomUUID() + "-" + fileName;
      String uploadUrl = s3Presigner.presignPutObject(PutObjectPresignRequest.builder()
              .signatureDuration(Duration.ofMinutes(15))
              .putObjectRequest(PutObjectRequest.builder()
                  .bucket(bucketName)
                  .key(fileKey)
                  .contentType(contentType)
                  .build())
              .build())
          .url()
          .toString();
      return new GenerateUploadUrlResult(uploadUrl, fileKey);
    }
    ```
  - **Chú ý**: `PutObjectPresignRequest` là class khác với `GetObjectPresignRequest` đang dùng ở method `generatePresignedUrl()`.

- [ ] **Task 6: Tạo `DeliverMilestoneRequest` DTO**
  - File: `modules/infrastructure/src/main/java/com/tba/agentic/adapter/controller/request/DeliverMilestoneRequest.java`
  - Hoặc đặt inline trong controller nếu prefer:
    ```java
    package com.tba.agentic.adapter.controller.request;

    public record DeliverMilestoneRequest(
        String deliverableUrl,
        String deliverableS3Key,
        String deliverableFileName,
        String deliveryNote) {}
    ```

- [ ] **Task 7: Tạo `RequestUploadUrlRequest` DTO**
  - File: `modules/infrastructure/src/main/java/com/tba/agentic/adapter/controller/request/RequestUploadUrlRequest.java`
  ```java
  package com.tba.agentic.adapter.controller.request;

  public record RequestUploadUrlRequest(String fileName, String contentType) {}
  ```

- [ ] **Task 8: Cập nhật `AdminMilestoneController`**
  - File: `modules/infrastructure/src/main/java/com/tba/agentic/adapter/controller/AdminMilestoneController.java`
  - **Xóa** imports: `MultipartFile`, `IOException`, `jakarta.servlet.http.HttpServletRequest` (giữ lại HttpServletRequest nếu cần cho ExceptionHandler)
  - **Thêm** inject: `FileStoragePort fileStorage`, `MilestoneRepository milestoneRepository`
  - **Thêm** imports cần thiết:
    ```java
    import com.tba.agentic.port.out.FileStoragePort;
    import com.tba.agentic.port.out.MilestoneRepository;
    import com.tba.agentic.adapter.controller.request.DeliverMilestoneRequest;
    import com.tba.agentic.adapter.controller.request.RequestUploadUrlRequest;
    import org.springframework.web.bind.annotation.RequestBody;
    import java.util.Map;
    ```
  - **Thay thế** endpoint `/deliver`:
    ```java
    @PostMapping("/{milestoneId}/deliver")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Void> deliver(
        @PathVariable Long milestoneId,
        @RequestBody DeliverMilestoneRequest request,
        @AuthenticationPrincipal Long adminId) {
      String url = request.deliverableUrl();
      String s3Key = request.deliverableS3Key();
      if ((url == null || url.isBlank()) && (s3Key == null || s3Key.isBlank())) {
        return ResponseEntity.badRequest().build();
      }
      deliverMilestoneUseCase.deliver(new DeliverMilestoneCommand(
          milestoneId,
          url,
          request.deliverableFileName(),
          s3Key,
          request.deliveryNote(),
          adminId));
      return ResponseEntity.ok().build();
    }
    ```
  - **Thêm** endpoint `/request-upload-url`:
    ```java
    @PostMapping("/{milestoneId}/request-upload-url")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<GenerateUploadUrlResult> requestUploadUrl(
        @PathVariable Long milestoneId,
        @RequestBody RequestUploadUrlRequest request) {
      if (request.fileName() == null || request.fileName().isBlank()
          || request.contentType() == null || request.contentType().isBlank()) {
        return ResponseEntity.badRequest().build();
      }
      var milestone = milestoneRepository.findById(milestoneId)
          .orElseThrow(() -> new com.tba.agentic.domain.exception.MilestoneNotFoundException(
              milestoneId.toString()));
      String projectId = milestone.getProjectId().value().toString();
      GenerateUploadUrlResult result = fileStorage.generatePresignedUploadUrl(
          request.fileName(), request.contentType(), projectId, milestoneId.toString());
      return ResponseEntity.ok(result);
    }
    ```
  - **Thêm** import:
    ```java
    import com.tba.agentic.port.bound.GenerateUploadUrlResult;
    ```

- [ ] **Task 9: `make compile` để kiểm tra không có lỗi**

- [ ] **Task 10: `make test` để đảm bảo không có regression**

---

## Dev Notes

### Tại Sao Inject `MilestoneRepository` Vào Controller?

Controller cần `milestoneId → projectId` để build S3 key. Không có domain logic ở đây (chỉ đọc dữ liệu). Hexagonal cho phép infrastructure adapter gọi `out` ports cho utility operations — chỉ không được gọi ngược lại (từ domain vào infrastructure). Controller (`AdminMilestoneController`) là adapter infrastructure, inject `MilestoneRepository` (port out) là hợp lệ.

### Method `findById` Trên `MilestoneRepository`

Xem file `modules/core/src/main/java/com/tba/agentic/port/out/MilestoneRepository.java` để confirm signature. Method có thể là `findById(Long milestoneId)` trả về `Optional<Milestone>`.

Confirm `milestone.getProjectId().value()` trả về `UUID` hay `Long` trước khi dùng `.toString()`.

### S3 CORS Configuration (Yêu Cầu Một Lần)

Browser PUT trực tiếp lên S3 yêu cầu S3 bucket CORS cho phép `PUT` method từ frontend origin.

Cấu hình 1 lần trên AWS Console (S3 bucket → Permissions → CORS):
```json
[
  {
    "AllowedHeaders": ["Content-Type"],
    "AllowedMethods": ["PUT"],
    "AllowedOrigins": ["http://localhost:3001", "https://your-prod-domain.com"],
    "MaxAgeSeconds": 3600
  }
]
```

Không cần code trong `S3Config.java` — cấu hình AWS Console là persistent.

### `DeliverMilestoneService` — `fileStorage` Vẫn Được Giữ

Field `private final FileStoragePort fileStorage;` trong `DeliverMilestoneService` vẫn tồn tại vì:
- `DeliverMilestoneService` được wire trong `UseCaseConfig.java` với `fileStorage` là param
- Dù service không dùng `upload()` nữa, vẫn giữ để tránh breaking `UseCaseConfig` wiring

Nếu muốn clean hơn: xóa `fileStorage` khỏi `DeliverMilestoneService` và cập nhật `UseCaseConfig` tương ứng. Tùy quyết định dev agent.

### AWS SDK — `PutObjectPresignRequest`

Import từ `software.amazon.awssdk.services.s3.presigner.model.PutObjectPresignRequest`. Không nhầm với `GetObjectPresignRequest`.

Method build:
```java
s3Presigner.presignPutObject(
    PutObjectPresignRequest.builder()
        .signatureDuration(Duration.ofMinutes(15))
        .putObjectRequest(r -> r.bucket(bucketName).key(fileKey).contentType(contentType))
        .build()
).url().toString()
```

### Files Cần Sửa

| File | Loại | Thay đổi |
|------|------|----------|
| `core/port/bound/GenerateUploadUrlResult.java` | NEW | Record mới |
| `core/port/bound/DeliverMilestoneCommand.java` | UPDATE | Xóa InputStream + contentType, thêm deliverableS3Key |
| `core/port/out/FileStoragePort.java` | UPDATE | Thêm `generatePresignedUploadUrl()` |
| `application/service/DeliverMilestoneService.java` | UPDATE | Xóa upload block, dùng deliverableS3Key |
| `infrastructure/storage/S3FileStorageAdapter.java` | UPDATE | Implement presignPutObject |
| `infrastructure/controller/AdminMilestoneController.java` | UPDATE | Đổi /deliver sang JSON, thêm /request-upload-url |
| `infrastructure/controller/request/DeliverMilestoneRequest.java` | NEW | HTTP JSON DTO |
| `infrastructure/controller/request/RequestUploadUrlRequest.java` | NEW | HTTP JSON DTO |

---

## References

- `S3FileStorageAdapter.java` — line 37: `RequestBody.fromInputStream(data, -1)` root cause
- `S3Config.java` — `S3Presigner` bean đã tồn tại
- `DeliverMilestoneService.java` — block `fileStorage.upload()` cần xóa (lines 40-49)
- `AdminMilestoneController.java` — endpoint cần đổi từ multipart sang JSON
- Epic 8.6 spec: `epics.md` lines 1260-1312
