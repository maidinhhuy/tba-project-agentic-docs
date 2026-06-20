# Story 8.2: Admin Deliver Milestone API

**Status:** ready-for-dev
**Story ID:** 8.2
**Epic:** 8 — Milestone Delivery & Customer Review

---

## Story

As an admin,
I want to deliver a milestone by providing a deliverable URL and/or uploading a file with a delivery note,
So that the customer is notified and can review the work.

---

## Acceptance Criteria

**AC-1** — Deliver via URL only:
**Given** admin gọi `POST /api/admin/milestones/{milestoneId}/deliver` với `deliverableUrl` hợp lệ, `deliveryNote` tùy chọn,
**When** milestone đang ở trạng thái hợp lệ (chưa APPROVED),
**Then**:
- `milestones.deliverable_url` được lưu
- `milestones.review_status` → `PENDING_REVIEW`
- `milestones.delivered_at` = now()
- Project `status` → `AWAITING_REVIEW`
- Email gửi cho customer: "Milestone [name] đã được giao, hãy xem và phê duyệt"
- Response 200 với milestone đã cập nhật

**AC-2** — Deliver với file upload:
**Given** admin gọi `POST /api/admin/milestones/{milestoneId}/deliver` dưới dạng `multipart/form-data` với file,
**When** file hợp lệ (max 50MB),
**Then**:
- File được upload lên S3, `milestones.deliverable_file_key` và `deliverable_file_name` được lưu
- Các trường khác như AC-1

**AC-3** — Deliver với cả URL + file:
**Given** admin cung cấp cả `deliverableUrl` và file,
**Then** cả 2 đều được lưu — không loại trừ nhau.

**AC-4** — Không deliver milestone đã APPROVED:
**Given** milestone đã có `review_status = APPROVED`,
**When** admin thử deliver lại,
**Then** response 422 với message "Milestone đã được phê duyệt, không thể giao lại."

**AC-5** — Email fire-and-forget:
**Given** email service bị lỗi khi deliver,
**When** exception xảy ra,
**Then** transaction vẫn commit (delivery thành công), lỗi chỉ được log.

**AC-6** — Authentication & Authorization:
**Given** request không có JWT hoặc JWT của CUSTOMER role,
**Then** response 401/403.

**AC-7** — Project status transition:
**Given** project đang ở `IN_DEVELOPMENT` hoặc `IN_REVISION`,
**When** milestone được deliver,
**Then** project `status` → `AWAITING_REVIEW` (via existing state machine).

---

## Tasks / Subtasks

### BE Core Layer

- [ ] **Task 1: Tạo DeliverMilestoneCommand**
  - File: `modules/core/src/main/java/com/tba/agentic/port/bound/DeliverMilestoneCommand.java`
  ```java
  package com.tba.agentic.port.bound;

  import java.io.InputStream;

  public record DeliverMilestoneCommand(
      Long milestoneId,
      String deliverableUrl,
      String deliverableFileName,
      String deliverableContentType,
      InputStream deliverableFile,
      String deliveryNote,
      Long adminUserId
  ) {}
  ```

- [ ] **Task 2: Tạo DeliverMilestoneUseCase port**
  - File: `modules/core/src/main/java/com/tba/agentic/port/in/DeliverMilestoneUseCase.java`
  ```java
  package com.tba.agentic.port.in;

  import com.tba.agentic.port.bound.DeliverMilestoneCommand;

  public interface DeliverMilestoneUseCase {
      void deliver(DeliverMilestoneCommand command);
  }
  ```

- [ ] **Task 3: Cập nhật ProjectStatus — thêm IN_REVISION → AWAITING_REVIEW transition**
  - File: `modules/core/src/main/java/com/tba/agentic/domain/value/project/ProjectStatus.java`
  - Kiểm tra current transitions của `IN_REVISION`. Nếu chưa có `AWAITING_REVIEW`, thêm vào.
  - Milestone được deliver khi project `IN_REVISION` → project cần chuyển sang `AWAITING_REVIEW`

### BE Application Layer

- [ ] **Task 4: Tạo DeliverMilestoneService**
  - File: `modules/application/src/main/java/com/tba/agentic/application/service/DeliverMilestoneService.java`
  ```java
  package com.tba.agentic.application.service;

  import com.tba.agentic.port.bound.DeliverMilestoneCommand;
  import com.tba.agentic.port.in.DeliverMilestoneUseCase;
  import com.tba.agentic.port.out.EmailClient;
  import com.tba.agentic.port.out.FileStoragePort;
  import com.tba.agentic.port.out.MilestoneRepository;
  import com.tba.agentic.port.out.ProjectRepository;
  import com.tba.agentic.port.out.UserRepository;
  import lombok.RequiredArgsConstructor;
  import lombok.extern.slf4j.Slf4j;
  import org.springframework.transaction.annotation.Transactional;

  @Slf4j
  @RequiredArgsConstructor
  public class DeliverMilestoneService implements DeliverMilestoneUseCase {

      private final MilestoneRepository milestoneRepository;
      private final ProjectRepository projectRepository;
      private final UserRepository userRepository;
      private final FileStoragePort fileStorage;
      private final EmailClient emailClient;

      @Override
      @Transactional
      public void deliver(DeliverMilestoneCommand command) {
          var milestone = milestoneRepository.findByIdOrThrow(command.milestoneId());

          if ("APPROVED".equals(milestone.reviewStatus())) {
              throw new IllegalStateException("Milestone đã được phê duyệt, không thể giao lại.");
          }

          String fileKey = null;
          if (command.deliverableFile() != null) {
              fileKey = fileStorage.upload(
                  command.deliverableFileName(),
                  command.deliverableContentType(),
                  command.deliverableFile(),
                  String.valueOf(milestone.projectId()),
                  String.valueOf(command.milestoneId())
              );
          }

          milestoneRepository.saveDelivery(
              command.milestoneId(),
              command.deliverableUrl(),
              fileKey,
              command.deliverableFileName(),
              command.deliveryNote()
          );

          var project = projectRepository.findByIdOrThrow(milestone.projectId());
          project.transitionTo(com.tba.agentic.domain.value.project.ProjectStatus.AWAITING_REVIEW);
          projectRepository.save(project);

          var customer = userRepository.findByIdOrThrow(project.customerId());
          try {
              emailClient.sendMilestoneDeliveredEmail(
                  customer.email(),
                  project.name(),
                  milestone.name()
              );
          } catch (Exception e) {
              log.warn("Failed to send milestone delivery email to {}", customer.email(), e);
          }
      }
  }
  ```

### BE Infrastructure Layer

- [ ] **Task 5: Tạo MilestoneRepository port (nếu chưa có)**
  - File: `modules/core/src/main/java/com/tba/agentic/port/out/MilestoneRepository.java`
  - Kiểm tra xem port này đã tồn tại chưa (có thể đã có từ epic trước)
  - Nếu chưa có, tạo với methods:
  ```java
  package com.tba.agentic.port.out;

  import com.tba.agentic.domain.entity.Milestone;

  public interface MilestoneRepository {
      Milestone findByIdOrThrow(Long milestoneId);
      void saveDelivery(Long milestoneId, String url, String fileKey, String fileName, String note);
  }
  ```
  - Nếu đã có, chỉ thêm method `saveDelivery` vào interface

- [ ] **Task 6: Tạo/cập nhật JooqMilestoneRepository**
  - Xác định file repo tương ứng trong infrastructure (tìm file implement MilestoneRepository)
  - Implement `saveDelivery`:
  ```java
  @Override
  public void saveDelivery(Long milestoneId, String url, String fileKey, String fileName, String note) {
      dsl.update(MILESTONES)
          .set(MILESTONES.DELIVERABLE_URL, url)
          .set(MILESTONES.DELIVERABLE_FILE_KEY, fileKey)
          .set(MILESTONES.DELIVERABLE_FILE_NAME, fileName)
          .set(MILESTONES.DELIVERY_NOTE, note)
          .set(MILESTONES.REVIEW_STATUS, "PENDING_REVIEW")
          .set(MILESTONES.DELIVERED_AT, OffsetDateTime.now())
          .where(MILESTONES.MILESTONE_ID.eq(milestoneId))
          .execute();
  }
  ```

- [ ] **Task 7: Thêm email method vào EmailClient port**
  - File: `modules/core/src/main/java/com/tba/agentic/port/out/EmailClient.java`
  - Thêm: `void sendMilestoneDeliveredEmail(String customerEmail, String projectName, String milestoneName);`
  - Implement trong `ResendEmailClient` (infrastructure) và `DevEmailClient` (dev/test)

- [ ] **Task 8: Tạo DeliverMilestoneController**
  - File: `modules/infrastructure/src/main/java/com/tba/agentic/adapter/controller/AdminMilestoneController.java`
  ```java
  package com.tba.agentic.adapter.controller;

  import com.tba.agentic.port.bound.DeliverMilestoneCommand;
  import com.tba.agentic.port.in.DeliverMilestoneUseCase;
  import lombok.RequiredArgsConstructor;
  import org.springframework.http.ResponseEntity;
  import org.springframework.security.access.prepost.PreAuthorize;
  import org.springframework.web.bind.annotation.PathVariable;
  import org.springframework.web.bind.annotation.PostMapping;
  import org.springframework.web.bind.annotation.RequestMapping;
  import org.springframework.web.bind.annotation.RequestParam;
  import org.springframework.web.bind.annotation.RestController;
  import org.springframework.web.multipart.MultipartFile;

  import java.io.IOException;

  @RestController
  @RequiredArgsConstructor
  @RequestMapping("/api/admin/milestones")
  public class AdminMilestoneController {

      private final DeliverMilestoneUseCase deliverMilestoneUseCase;

      @PostMapping("/{milestoneId}/deliver")
      @PreAuthorize("hasRole('ADMIN')")
      public ResponseEntity<Void> deliver(
          @PathVariable Long milestoneId,
          @RequestParam(required = false) String deliverableUrl,
          @RequestParam(required = false) String deliveryNote,
          @RequestParam(required = false) MultipartFile file,
          @AuthenticationPrincipal JwtUserDetails userDetails
      ) throws IOException {
          deliverMilestoneUseCase.deliver(new DeliverMilestoneCommand(
              milestoneId,
              deliverableUrl,
              file != null ? file.getOriginalFilename() : null,
              file != null ? file.getContentType() : null,
              file != null ? file.getInputStream() : null,
              deliveryNote,
              userDetails.userId()
          ));
          return ResponseEntity.ok().build();
      }
  }
  ```
  - Thêm import `@AuthenticationPrincipal` và `JwtUserDetails` — kiểm tra cách pattern auth hiện tại đang dùng (xem các controller khác trong cùng package)

- [ ] **Task 9: Wire trong UseCaseConfig**
  - File: `modules/infrastructure/src/main/java/com/tba/agentic/config/UseCaseConfig.java`
  - Thêm bean:
  ```java
  @Bean
  public DeliverMilestoneUseCase deliverMilestoneUseCase(
      MilestoneRepository milestoneRepository,
      ProjectRepository projectRepository,
      UserRepository userRepository,
      FileStoragePort fileStorage,
      EmailClient emailClient
  ) {
      return new DeliverMilestoneService(milestoneRepository, projectRepository, userRepository, fileStorage, emailClient);
  }
  ```

- [ ] **Task 10: make compile + make test**

---

## Dev Notes

### Pattern Tham Khảo: `UpdateProjectStatusService`

File: `modules/application/src/main/java/com/tba/agentic/application/service/UpdateProjectStatusService.java`

- `@RequiredArgsConstructor` + `@Transactional` (annotation của Spring OK trong application layer — chỉ không dùng Spring `@Component`)
- Fire-and-forget email: `try { emailClient.send... } catch (Exception e) { log.warn(..., e) }`
- Method trả về entity hoặc void

### Multipart File Upload Pattern

Controller nhận `MultipartFile file` từ Spring MVC. `file.getInputStream()` trả về `InputStream` để pass vào `FileStoragePort.upload()`.

File size limit cần config trong `application.yml`:
```yaml
spring:
  servlet:
    multipart:
      max-file-size: 50MB
      max-request-size: 52MB
```

### Auth Pattern

Xem controller hiện có (VD: `AdminProjectController`) để biết cách extract `userId` từ JWT principal. Pattern thường là `@AuthenticationPrincipal JwtUserDetails userDetails` hoặc `SecurityContextHolder`.

### Email Template

Email content cho `sendMilestoneDeliveredEmail`:
```
Subject: [TBA] Milestone "[milestoneName]" của dự án "[projectName]" đã được giao
Body: Milestone ... đã sẵn sàng để bạn xem xét và phê duyệt. Vui lòng đăng nhập vào hệ thống để xem chi tiết.
```

### Không Có Milestone Entity Trong Core?

Nếu không có `Milestone` entity trong `core/domain/entity/`, cần tạo một simple POJO:
```java
public record Milestone(Long milestoneId, Long projectId, String name, String reviewStatus) {}
```
Hoặc dùng data class phù hợp với convention của project.

### MilestoneRepository — Tồn Tại Chưa?

Cần kiểm tra xem `MilestoneRepository` port đã có chưa (có thể được tạo trong Epic 4 khi làm milestone management). Nếu đã tồn tại, chỉ thêm method `saveDelivery` vào interface và implementation.
