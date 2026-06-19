# Story 5.1: Transactional Service Layer

**Status:** ready-for-dev
**Story ID:** 5.1
**Epic:** 5 — Infrastructure Hardening

---

## Story

As a developer,
I want all service methods annotated with `@Transactional` (write) or `@Transactional(readOnly = true)` (read),
So that database operations are atomic, consistent, and read queries benefit from connection optimization.

---

## Acceptance Criteria

**AC-1** — Write operations:
**Given** bất kỳ service method nào thực hiện INSERT/UPDATE/DELETE,
**When** method throw exception (runtime hoặc checked),
**Then** toàn bộ DB changes trong method đó bị rollback tự động.

**AC-2** — Read operations:
**Given** service method chỉ thực hiện SELECT,
**When** được gọi,
**Then** method chạy trong transaction readOnly = true (Hibernate flush mode NEVER, connection optimization).

**AC-3** — Email không trigger rollback:
**Given** `emailClient.send()` được gọi bên trong transaction (SubmitProjectService, UpdateProjectStatusService),
**When** email send fail,
**Then** transaction KHÔNG bị rollback — email call được bọc trong try-catch riêng, tách biệt với transaction.

**AC-4** — Build:
**And** `./gradlew :modules:application:compileJava` → BUILD SUCCESSFUL.

---

## Tasks / Subtasks

- [ ] Task 1: Thêm `@Transactional` vào write services
  - `RegisterUserService.execute()` — tạo user
  - `SubmitProjectService.execute()` — tạo project + milestones
  - `UpdateProjectStatusService.execute()` — update status + insert history
  - `UpdateCustomerProjectService.execute()` — update project
  - `UpdateMilestoneService.execute()` — update milestone
  - `TokenManagementService` — các method write: `issueTokenPair()`, `refresh()`, `revoke()`

- [ ] Task 2: Thêm `@Transactional(readOnly = true)` vào read services
  - `LoginUserService.execute()` — findByEmail
  - `GetCustomerProjectsService.execute()` — findByCustomer
  - `GetProjectDetailService.execute()` — findByIdForCustomer
  - `GetAllProjectsService.execute()` — findAll
  - `GetAdminProjectDetailService.execute()` — findByIdForAdmin
  - `GetUserProfileService.execute()` — findById

- [ ] Task 3: Verify build + smoke test
  - `./gradlew :modules:application:compileJava` → BUILD SUCCESSFUL
  - Trigger 1 write flow (ví dụ: submit project) → verify không lỗi

---

## Dev Notes

### Module Structure

```
modules/application/src/main/java/com/tba/agentic/application/service/
  RegisterUserService.java          ← @Transactional (write)
  SubmitProjectService.java         ← @Transactional (write)
  UpdateProjectStatusService.java   ← @Transactional (write)
  UpdateCustomerProjectService.java ← @Transactional (write)
  UpdateMilestoneService.java       ← @Transactional (write)
  TokenManagementService.java       ← @Transactional (write)
  LoginUserService.java             ← @Transactional(readOnly = true)
  GetCustomerProjectsService.java   ← @Transactional(readOnly = true)
  GetProjectDetailService.java      ← @Transactional(readOnly = true)
  GetAllProjectsService.java        ← @Transactional(readOnly = true)
  GetAdminProjectDetailService.java ← @Transactional(readOnly = true)
  GetUserProfileService.java        ← @Transactional(readOnly = true)
```

> ⚠️ Root-level `application/` là legacy (pre-PR#74) — KHÔNG đụng vào.

### Spring @Transactional với plain Java classes

Services là **plain Java classes** (không có `@Service`), được wire thủ công qua `UseCaseConfig.java`:

```java
// UseCaseConfig.java
@Bean
public SubmitProjectUseCase submitProjectUseCase(...) {
    return new SubmitProjectService(...);
}
```

`@Transactional` vẫn hoạt động vì:
- Bean được Spring container quản lý qua `@Bean`
- Spring tạo CGLIB proxy wrapping `SubmitProjectService`
- Proxy intercept method call → open transaction → call real method → commit/rollback

**Đặt `@Transactional` trên class level** (annotate trên class declaration, không phải interface):

```java
import org.springframework.transaction.annotation.Transactional;

@Transactional          // write service
@RequiredArgsConstructor
public class SubmitProjectService implements SubmitProjectUseCase {
    @Override
    public Project execute(SubmitProjectCommand cmd) { ... }
}

@Transactional(readOnly = true)   // read service
@RequiredArgsConstructor
public class GetProjectDetailService implements GetProjectDetailUseCase {
    @Override
    public Project execute(...) { ... }
}
```

### Email không rollback transaction (AC-3)

`SubmitProjectService` và `UpdateProjectStatusService` đã có try-catch bao quanh `emailClient.send()`:

```java
try {
    emailClient.send(new EmailMessage(...));
} catch (Exception e) {
    log.error("...", e);  // swallow — không re-throw
}
```

Exception bị swallow → không propagate ra transaction → transaction commit bình thường. Giữ nguyên pattern này, **không sửa**.

### Import đúng

```java
import org.springframework.transaction.annotation.Transactional;
// KHÔNG dùng: javax.transaction.Transactional hoặc jakarta.transaction.Transactional
```

### TokenManagementService — method phân loại

Xem lại TokenManagementService để phân loại đúng:
- `issueTokenPair()` / `refresh()` / `revoke()` → write → `@Transactional` ở class level đủ
- Nếu có method read-only riêng → override với `@Transactional(readOnly = true)` ở method level

---

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Completion Notes List

- Services là plain Java class wire qua UseCaseConfig.java — Spring proxy @Transactional vẫn hoạt động
- Email failure không trigger rollback nhờ try-catch đã có sẵn trong callers
- Import: org.springframework.transaction.annotation.Transactional

### File List

**UPDATE (12 files):**
- `modules/application/src/main/java/com/tba/agentic/application/service/RegisterUserService.java`
- `modules/application/src/main/java/com/tba/agentic/application/service/SubmitProjectService.java`
- `modules/application/src/main/java/com/tba/agentic/application/service/UpdateProjectStatusService.java`
- `modules/application/src/main/java/com/tba/agentic/application/service/UpdateCustomerProjectService.java`
- `modules/application/src/main/java/com/tba/agentic/application/service/UpdateMilestoneService.java`
- `modules/application/src/main/java/com/tba/agentic/application/service/TokenManagementService.java`
- `modules/application/src/main/java/com/tba/agentic/application/service/LoginUserService.java`
- `modules/application/src/main/java/com/tba/agentic/application/service/GetCustomerProjectsService.java`
- `modules/application/src/main/java/com/tba/agentic/application/service/GetProjectDetailService.java`
- `modules/application/src/main/java/com/tba/agentic/application/service/GetAllProjectsService.java`
- `modules/application/src/main/java/com/tba/agentic/application/service/GetAdminProjectDetailService.java`
- `modules/application/src/main/java/com/tba/agentic/application/service/GetUserProfileService.java`
