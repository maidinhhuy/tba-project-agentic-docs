# Story 8.3: Customer Review API + State Machine Update

**Status:** ready-for-dev
**Story ID:** 8.3
**Epic:** 8 — Milestone Delivery & Customer Review

---

## Story

As a customer,
I want to review a delivered milestone by approving it or requesting revision with a comment,
So that I can control quality before the project moves forward.

---

## Acceptance Criteria

**AC-1** — Customer approve milestone:
**Given** customer gọi `POST /api/customer/milestones/{milestoneId}/review` với `action=APPROVED`,
**When** milestone có `review_status = PENDING_REVIEW`,
**Then**:
- `milestone_reviews` nhận 1 record mới: action=APPROVED, comment=null (optional), reviewed_by=customerId, reviewed_at=now()
- `milestones.review_status` → `APPROVED`
- Nếu đây là milestone CUỐI cùng (tất cả milestones đều APPROVED): project `status` → `DELIVERED`
- Nếu KHÔNG phải milestone cuối: project `status` → `IN_DEVELOPMENT` (tiếp tục làm milestone tiếp theo)
- Email gửi cho admin: "Customer đã phê duyệt milestone [name]"
- Response 200

**AC-2** — Customer request revision:
**Given** customer gọi `POST /api/customer/milestones/{milestoneId}/review` với `action=REVISION_REQUESTED` và `comment` bắt buộc,
**When** milestone có `review_status = PENDING_REVIEW`,
**Then**:
- `milestone_reviews` nhận 1 record: action=REVISION_REQUESTED, comment=..., reviewed_by=customerId
- `milestones.review_status` → `REVISION_REQUESTED`
- `projects.revision_count` giảm 1 (min 0, không âm)
- Project `status` → `IN_REVISION`
- Email gửi admin: "Customer yêu cầu chỉnh sửa: [comment]"
- Response 200

**AC-3** — Revision khi revision_count = 0:
**Given** customer request revision khi `projects.revision_count = 0`,
**When** submit review,
**Then** vẫn cho phép (revision_count giữ nguyên 0, không âm), nhưng response body có thêm flag `warningRevisionExhausted: true`. Email admin bao gồm cảnh báo "Đây là lần vượt quá revision."

**AC-4** — Comment bắt buộc khi REVISION_REQUESTED:
**Given** customer gọi với `action=REVISION_REQUESTED` và `comment` null/empty,
**Then** response 400 với message "Comment là bắt buộc khi yêu cầu chỉnh sửa."

**AC-5** — Không review milestone không phải của mình:
**Given** customer gọi với milestoneId thuộc project của customer khác,
**Then** response 403.

**AC-6** — Không review milestone đã APPROVED:
**Given** milestone đã `review_status = APPROVED`,
**When** customer thử review lại,
**Then** response 422 "Milestone đã được phê duyệt."

**AC-7** — Không review milestone ở `PENDING_DELIVERY`:
**Given** milestone chưa được deliver (`review_status = PENDING_DELIVERY`),
**Then** response 422 "Milestone chưa được giao để review."

**AC-8** — Presigned URL cho file download:
**Given** customer gọi `GET /api/customer/milestones/{milestoneId}/download-url`,
**When** milestone có `deliverable_file_key` != null,
**Then** response 200 với `{ "url": "https://s3.amazonaws.com/..." }` hợp lệ trong 30 phút.

**AC-9** — State machine transitions đúng:
**Given** `ProjectStatus.java` được cập nhật,
**Then**:
- `AWAITING_REVIEW` → `IN_DEVELOPMENT` (approve non-last milestone)
- `AWAITING_REVIEW` → `DELIVERED` (approve last milestone)
- `AWAITING_REVIEW` → `IN_REVISION` (revision requested)

---

## Tasks / Subtasks

> ⚠️ **AGENT FILE BOUNDARY RULE (BẮT BUỘC — ĐỌC TRƯỚC KHI CODE):**
> - **1 Java file = đúng 1 top-level class/record/interface.** KHÔNG ĐƯỢC append thêm class, record, hay interface vào cuối bất kỳ file nào.
> - Mỗi task chỉ động đến đúng 1 file. Sau khi xong 1 task → lưu file → rồi mới mở file tiếp theo.
> - `core/` = declarations only. `application/` = service logic. `infrastructure/` = adapters, repos, controllers, config.

### Layer 1 — Domain & Core Ports

- [ ] **Task 1 — File: `ProjectStatus.java`**
  - `modules/core/src/main/java/com/tba/agentic/domain/value/project/ProjectStatus.java`
  - Tìm dòng `AWAITING_REVIEW, Set.of(IN_REVISION, FINALIZING, CANCELLED)` sửa thành:
  ```java
  AWAITING_REVIEW, Set.of(IN_DEVELOPMENT, IN_REVISION, DELIVERED, FINALIZING, CANCELLED),
  ```

- [ ] **Task 2 — File mới: `ReviewMilestoneCommand.java`**
  - `modules/core/src/main/java/com/tba/agentic/port/bound/ReviewMilestoneCommand.java`
  ```java
  package com.tba.agentic.port.bound;

  public record ReviewMilestoneCommand(
      Long milestoneId,
      Long customerUserId,
      String action,
      String comment
  ) {}
  ```

- [ ] **Task 3 — File mới: `ReviewMilestoneResult.java`**
  - `modules/core/src/main/java/com/tba/agentic/port/bound/ReviewMilestoneResult.java`
  ```java
  package com.tba.agentic.port.bound;

  public record ReviewMilestoneResult(boolean warningRevisionExhausted) {}
  ```

- [ ] **Task 4 — File mới: `ReviewMilestoneUseCase.java`**
  - `modules/core/src/main/java/com/tba/agentic/port/in/ReviewMilestoneUseCase.java`
  ```java
  package com.tba.agentic.port.in;

  import com.tba.agentic.port.bound.ReviewMilestoneCommand;
  import com.tba.agentic.port.bound.ReviewMilestoneResult;

  public interface ReviewMilestoneUseCase {
      ReviewMilestoneResult review(ReviewMilestoneCommand command);
  }
  ```

- [ ] **Task 5 — File: `MilestoneRepository.java`** (chỉ thêm method signatures, KHÔNG thêm class/record nào)
  - `modules/core/src/main/java/com/tba/agentic/port/out/MilestoneRepository.java`
  ```java
  void saveReviewStatus(Long milestoneId, String reviewStatus);
  void insertReview(Long milestoneId, String action, String comment, Long reviewedBy);
  boolean allMilestonesApproved(Long projectId);
  ```

- [ ] **Task 6 — File: `ProjectRepository.java`** (chỉ thêm 1 method signature)
  - `modules/core/src/main/java/com/tba/agentic/port/out/ProjectRepository.java`
  ```java
  void decrementRevisionCount(Long projectId);
  ```

- [ ] **Task 7 — File: `EmailClient.java`** (chỉ thêm 2 abstract method signatures, KHÔNG có body, KHÔNG có `default`)
  - `modules/core/src/main/java/com/tba/agentic/port/out/EmailClient.java`
  ```java
  void sendMilestoneApprovedEmail(String adminEmail, String projectName, String milestoneName);
  void sendRevisionRequestedEmail(String adminEmail, String projectName, String milestoneName, String comment, boolean revisionExhausted);
  ```

### Layer 2 — Email Infrastructure

- [ ] **Task 8 — File: `DevEmailClient.java`** (chỉ thêm 2 @Override methods vào class hiện có — KHÔNG thêm class nào sau closing `}`)
  - `modules/infrastructure/src/main/java/com/tba/agentic/infrastructure/email/DevEmailClient.java`
  ```java
  @Override
  public void sendMilestoneApprovedEmail(String adminEmail, String projectName, String milestoneName) {
      log.info("[EMAIL-DEV] sendMilestoneApprovedEmail to={} project={} milestone={}", adminEmail, projectName, milestoneName);
  }

  @Override
  public void sendRevisionRequestedEmail(String adminEmail, String projectName, String milestoneName, String comment, boolean revisionExhausted) {
      log.info("[EMAIL-DEV] sendRevisionRequestedEmail to={} project={} milestone={} comment={} exhausted={}", adminEmail, projectName, milestoneName, comment, revisionExhausted);
  }
  ```

- [ ] **Task 9 — File: `ResendEmailClient.java`** (chỉ thêm 2 @Override methods vào class hiện có)
  - `modules/infrastructure/src/main/java/com/tba/agentic/infrastructure/email/ResendEmailClient.java`
  ```java
  @Override
  public void sendMilestoneApprovedEmail(String adminEmail, String projectName, String milestoneName) {
      send(new EmailMessage(
          adminEmail,
          "Milestone Approved: " + milestoneName,
          "<p>Milestone <b>" + milestoneName + "</b> of project <b>" + projectName + "</b> has been approved.</p>"
      ));
  }

  @Override
  public void sendRevisionRequestedEmail(String adminEmail, String projectName, String milestoneName, String comment, boolean revisionExhausted) {
      String warning = revisionExhausted ? "<p><b>Warning:</b> Revision quota exhausted.</p>" : "";
      send(new EmailMessage(
          adminEmail,
          "Revision Requested: " + milestoneName,
          "<p>Revision requested for milestone <b>" + milestoneName + "</b>. Comment: " + comment + "</p>" + warning
      ));
  }
  ```

### Layer 3 — Application Service

- [ ] **Task 10 — File mới: `ReviewMilestoneService.java`** (file riêng trong module `application`, KHÔNG đặt trong `infrastructure`)
  - `modules/application/src/main/java/com/tba/agentic/application/service/ReviewMilestoneService.java`
  ```java
  package com.tba.agentic.application.service;

  import com.tba.agentic.domain.value.project.ProjectStatus;
  import com.tba.agentic.port.bound.ReviewMilestoneCommand;
  import com.tba.agentic.port.bound.ReviewMilestoneResult;
  import com.tba.agentic.port.in.ReviewMilestoneUseCase;
  import com.tba.agentic.port.out.EmailClient;
  import com.tba.agentic.port.out.MilestoneRepository;
  import com.tba.agentic.port.out.ProjectRepository;
  import com.tba.agentic.port.out.UserRepository;
  import lombok.RequiredArgsConstructor;
  import lombok.extern.slf4j.Slf4j;
  import org.springframework.transaction.annotation.Transactional;

  @Slf4j
  @RequiredArgsConstructor
  public class ReviewMilestoneService implements ReviewMilestoneUseCase {

      private final MilestoneRepository milestoneRepository;
      private final ProjectRepository projectRepository;
      private final UserRepository userRepository;
      private final EmailClient emailClient;

      @Override
      @Transactional
      public ReviewMilestoneResult review(ReviewMilestoneCommand command) {
          if ("REVISION_REQUESTED".equals(command.action())
                  && (command.comment() == null || command.comment().isBlank())) {
              throw new IllegalArgumentException("Comment la bat buoc khi yeu cau chinh sua.");
          }

          var milestone = milestoneRepository.findByIdOrThrow(command.milestoneId());

          if ("PENDING_DELIVERY".equals(milestone.getReviewStatus())) {
              throw new IllegalStateException("Milestone chua duoc giao de review.");
          }
          if ("APPROVED".equals(milestone.getReviewStatus())) {
              throw new IllegalStateException("Milestone da duoc phe duyet.");
          }

          var project = projectRepository.findByIdOrThrow(milestone.getProjectId().value());
          if (!project.customerId().equals(command.customerUserId())) {
              throw new SecurityException("Khong co quyen review milestone nay.");
          }

          milestoneRepository.insertReview(
              command.milestoneId(), command.action(), command.comment(), command.customerUserId());

          boolean warningRevisionExhausted = false;

          if ("APPROVED".equals(command.action())) {
              milestoneRepository.saveReviewStatus(command.milestoneId(), "APPROVED");
              boolean allApproved = milestoneRepository.allMilestonesApproved(milestone.getProjectId().value());
              project.transitionTo(allApproved ? ProjectStatus.DELIVERED : ProjectStatus.IN_DEVELOPMENT);
              projectRepository.save(project);

              var admin = userRepository.findAdminUser();
              try {
                  emailClient.sendMilestoneApprovedEmail(admin.email(), project.name(), milestone.getName());
              } catch (Exception e) {
                  log.warn("Failed to send approval email", e);
              }
          } else {
              milestoneRepository.saveReviewStatus(command.milestoneId(), "REVISION_REQUESTED");
              warningRevisionExhausted = project.revisionCount() <= 0;
              if (!warningRevisionExhausted) {
                  projectRepository.decrementRevisionCount(milestone.getProjectId().value());
              }
              project.transitionTo(ProjectStatus.IN_REVISION);
              projectRepository.save(project);

              var admin = userRepository.findAdminUser();
              try {
                  emailClient.sendRevisionRequestedEmail(
                      admin.email(), project.name(), milestone.getName(), command.comment(), warningRevisionExhausted);
              } catch (Exception e) {
                  log.warn("Failed to send revision email", e);
              }
          }

          return new ReviewMilestoneResult(warningRevisionExhausted);
      }
  }
  ```

### Layer 4 — Repository Infrastructure

- [ ] **Task 11 — File: `JooqMilestoneRepository.java`** (chỉ thêm 3 @Override methods vào class hiện có)
  - Tìm file implement MilestoneRepository trong `modules/infrastructure/`
  - Thêm imports nếu chưa có: `import static com.tba.agentic.infrastructure.jooq.Tables.MILESTONE_REVIEWS;` và `import java.time.OffsetDateTime;`
  ```java
  @Override
  public void saveReviewStatus(Long milestoneId, String reviewStatus) {
      dsl.update(MILESTONES)
          .set(MILESTONES.REVIEW_STATUS, reviewStatus)
          .where(MILESTONES.MILESTONE_ID.eq(milestoneId))
          .execute();
  }

  @Override
  public void insertReview(Long milestoneId, String action, String comment, Long reviewedBy) {
      dsl.insertInto(MILESTONE_REVIEWS)
          .set(MILESTONE_REVIEWS.MILESTONE_ID, milestoneId)
          .set(MILESTONE_REVIEWS.ACTION, action)
          .set(MILESTONE_REVIEWS.COMMENT, comment)
          .set(MILESTONE_REVIEWS.REVIEWED_BY, reviewedBy)
          .set(MILESTONE_REVIEWS.REVIEWED_AT, OffsetDateTime.now())
          .execute();
  }

  @Override
  public boolean allMilestonesApproved(Long projectId) {
      Integer count = dsl.selectCount()
          .from(MILESTONES)
          .where(MILESTONES.PROJECT_ID.eq(projectId)
              .and(MILESTONES.REVIEW_STATUS.ne("APPROVED")))
          .fetchOneInto(Integer.class);
      return count != null && count == 0;
  }
  ```

- [ ] **Task 12 — File: `JooqProjectRepository.java`** (chỉ thêm 1 @Override method)
  - Tìm file implement ProjectRepository trong `modules/infrastructure/`
  ```java
  @Override
  public void decrementRevisionCount(Long projectId) {
      dsl.update(PROJECTS)
          .set(PROJECTS.REVISION_COUNT, DSL.greatest(PROJECTS.REVISION_COUNT.minus(1), DSL.val(0)))
          .where(PROJECTS.PROJECT_ID.eq(projectId))
          .execute();
  }
  ```

### Layer 5 — HTTP Controller & Config

- [ ] **Task 13 — File mới: `CustomerMilestoneController.java`** (chỉ 1 class, DTOs là inner records bên trong)
  - `modules/infrastructure/src/main/java/com/tba/agentic/adapter/controller/CustomerMilestoneController.java`
  - Xem AdminProjectController để lấy đúng import JwtUserDetails
  ```java
  package com.tba.agentic.adapter.controller;

  import com.tba.agentic.port.bound.ReviewMilestoneCommand;
  import com.tba.agentic.port.in.ReviewMilestoneUseCase;
  import com.tba.agentic.port.out.FileStoragePort;
  import com.tba.agentic.port.out.MilestoneRepository;
  import lombok.RequiredArgsConstructor;
  import org.springframework.http.ResponseEntity;
  import org.springframework.security.access.prepost.PreAuthorize;
  import org.springframework.security.core.annotation.AuthenticationPrincipal;
  import org.springframework.web.bind.annotation.*;

  @RestController
  @RequiredArgsConstructor
  @RequestMapping("/api/customer/milestones")
  public class CustomerMilestoneController {

      private final ReviewMilestoneUseCase reviewMilestoneUseCase;
      private final FileStoragePort fileStorage;
      private final MilestoneRepository milestoneRepository;

      public record ReviewRequestDto(String action, String comment) {}
      public record ReviewResponseDto(boolean warningRevisionExhausted) {}
      public record DownloadUrlResponseDto(String url) {}

      @PostMapping("/{milestoneId}/review")
      @PreAuthorize("hasRole('CUSTOMER')")
      public ResponseEntity<ReviewResponseDto> review(
          @PathVariable Long milestoneId,
          @RequestBody ReviewRequestDto request,
          @AuthenticationPrincipal JwtUserDetails userDetails
      ) {
          var result = reviewMilestoneUseCase.review(new ReviewMilestoneCommand(
              milestoneId, userDetails.userId(), request.action(), request.comment()));
          return ResponseEntity.ok(new ReviewResponseDto(result.warningRevisionExhausted()));
      }

      @GetMapping("/{milestoneId}/download-url")
      @PreAuthorize("hasRole('CUSTOMER')")
      public ResponseEntity<DownloadUrlResponseDto> downloadUrl(@PathVariable Long milestoneId) {
          var milestone = milestoneRepository.findByIdOrThrow(milestoneId);
          if (milestone.getDeliverableFileKey() == null) {
              return ResponseEntity.notFound().build();
          }
          String url = fileStorage.generatePresignedUrl(milestone.getDeliverableFileKey(), 30);
          return ResponseEntity.ok(new DownloadUrlResponseDto(url));
      }
  }
  ```

- [ ] **Task 14 — File: `UseCaseConfig.java`** (chỉ thêm 1 @Bean method, không xóa beans hiện có)
  - `modules/infrastructure/src/main/java/com/tba/agentic/config/UseCaseConfig.java`
  ```java
  @Bean
  public ReviewMilestoneUseCase reviewMilestoneUseCase(
      MilestoneRepository milestoneRepository,
      ProjectRepository projectRepository,
      UserRepository userRepository,
      EmailClient emailClient
  ) {
      return new ReviewMilestoneService(milestoneRepository, projectRepository, userRepository, emailClient);
  }
  ```

- [ ] **Task 15 — Verify:** `make compile` + `make test` pass


## Dev Notes

### State Machine Hiện Tại

File: `modules/core/src/main/java/com/tba/agentic/domain/value/project/ProjectStatus.java`

Current `AWAITING_REVIEW` allowed transitions: `Set.of(IN_REVISION, FINALIZING, CANCELLED)`

Cần thêm:
- `IN_DEVELOPMENT` — khi approve milestone KHÔNG phải cuối
- `DELIVERED` — khi approve milestone CUỐI (allMilestonesApproved = true)

`DELIVERED` là terminal state (không có allowed transitions về phía forward). Kiểm tra xem `DELIVERED` đã có dòng trong enum chưa — có thể đã có từ initial schema.

### Logic "Milestone Cuối"

`allMilestonesApproved(projectId)` — query đếm số milestones thuộc project mà `review_status != 'APPROVED'`. Nếu count = 0 → tất cả đã approved → đây là milestone cuối.

**Lưu ý**: chỉ sau khi `saveReviewStatus(milestoneId, 'APPROVED')` mới gọi `allMilestonesApproved` — vì vậy query sẽ tính milestone vừa approve.

### revision_count Không Âm

```sql
-- PostgreSQL: GREATEST(revision_count - 1, 0)
UPDATE projects SET revision_count = GREATEST(revision_count - 1, 0) WHERE project_id = ?
```

Trong jOOQ: `DSL.greatest(PROJECTS.REVISION_COUNT.minus(1), DSL.val(0))`

### Security: Customer Chỉ Review Milestone Của Mình

Check: `project.customerId().equals(command.customerUserId())`

Không cần query thêm — `findByIdOrThrow(milestoneId)` trả về milestoneId → projectId, rồi `findByIdOrThrow(projectId)` trả về project với customerId.

### `findAdminUser()`

Nếu UserRepository chưa có method này, cần thêm. Admin user là user đầu tiên có role ADMIN trong hệ thống. Hoặc dùng email admin hardcoded từ config — tùy cách project đang làm.

Kiểm tra `UserRepository` port hiện tại để biết method nào đang có.

### presignedUrl TTL

30 phút là đủ để customer download. URL được generate on-demand khi customer click download — không cần cache.
