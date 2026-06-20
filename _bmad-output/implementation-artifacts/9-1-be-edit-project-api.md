# Story 9.1: BE — Edit Project API (PATCH)

**Status:** ready-for-dev
**Story ID:** 9.1
**Epic:** 9 — Customer Portal UX Enhancement & Edit Project

---

## Story

As a customer,
I want to update my project's name, description, and reference link via API,
So that I can correct or improve project information after submission.

---

## Acceptance Criteria

**AC-1 — Partial update:**
**Given** Customer đã đăng nhập và gọi `PATCH /api/v1/customer/projects/{id}`,
**When** request body chứa một hoặc nhiều fields: `name`, `description`, `reference`,
**Then** chỉ các fields được gửi trong body mới được update; fields không có trong body giữ nguyên giá trị hiện tại.
**And** response trả về HTTP 200 với full project object đã được update.

**AC-2 — Tenant isolation (403):**
**Given** Customer A gọi `PATCH /api/v1/customer/projects/{id}` với `id` là project của Customer B,
**When** request được xử lý,
**Then** response trả về HTTP 403 Forbidden (tenant isolation — không tiết lộ project tồn tại hay không với customer A).

**AC-3 — Project not found (404):**
**Given** project không tồn tại (id không hợp lệ — không có trong DB),
**When** Customer gọi `PATCH /api/v1/customer/projects/{id}`,
**Then** response trả về HTTP 404.

**AC-4 — Name validation:**
**Given** `name` được gửi trong body,
**When** `name` rỗng (empty string) hoặc dài hơn 100 ký tự,
**Then** response trả về HTTP 400 với validation error message cho field `name`.

**AC-5 — Description validation:**
**Given** `description` được gửi trong body,
**When** `description` có độ dài < 50 ký tự hoặc > 2000 ký tự,
**Then** response trả về HTTP 400 với validation error message cho field `description`.

**AC-6 — Reference validation:**
**Given** `reference` được gửi trong body,
**When** `reference` dài hơn 500 ký tự,
**Then** response trả về HTTP 400 với validation error message cho field `reference`.

**AC-7 — Reference clear:**
**Given** `reference` được gửi với giá trị empty string `""`,
**When** request được xử lý,
**Then** field `reference` của project được set về `null` (customer xoá reference link).

**AC-8 — No status restriction:**
**Given** project ở bất kỳ status nào (SUBMITTED, IN_REVISION, ANALYZING, COMPLETED, v.v.),
**When** Customer gọi PATCH với valid body,
**Then** update thành công — không có restriction dựa trên project status.

---

## ⚠️ Dev Notes — Đọc kỹ trước khi code

### Endpoint đã tồn tại — đây là story sửa gaps

**Toàn bộ skeleton đã có sẵn. Đừng tạo lại từ đầu.** Story này chỉ fix 4 gaps cụ thể giữa implementation hiện tại và acceptance criteria.

### Cấu trúc hiện tại (ĐÃ có)

Tất cả files dưới đây **đã tồn tại** trong `modules/`:

| File | Path | Trạng thái |
|------|------|-----------|
| `UpdateCustomerProjectUseCase` | `modules/core/.../port/in/` | Tồn tại ✅ |
| `UpdateCustomerProjectCommand` | `modules/core/.../port/bound/project/` | Tồn tại ✅ |
| `UpdateCustomerProjectService` | `modules/application/.../service/` | Tồn tại ✅ |
| `UpdateCustomerProjectRequest` | `modules/infrastructure/.../request/` | Tồn tại ✅ |
| `@PatchMapping("/{projectId}")` | `CustomerProjectController` | Tồn tại ✅ |
| `Project.updateDetails()` | `modules/core/.../entity/project/Project.java` | Tồn tại nhưng CÓ BUG ❌ |
| `findByIdForCustomer` | `JooqProjectRepository` | Tồn tại ✅ |
| `updateProjectDetails` | `JooqProjectRepository` | Tồn tại ✅ |

### 4 Gaps cần fix

#### Gap 1 — Status restriction trong `Project.updateDetails()` (vi phạm AC-8)

**File:** `modules/core/src/main/java/com/tba/agentic/domain/entity/project/Project.java` line ~96

**Hiện tại:**
```java
public void updateDetails(String name, String description, String reference) {
  if (!Set.of(ProjectStatus.SUBMITTED, ProjectStatus.IN_REVISION).contains(this.status)) {
    throw new ProjectNotEditableException(this.projectId.value().toString(), this.status);
  }
  if (name != null) this.name = name;
  if (description != null) this.description = description;
  if (reference != null) this.reference = reference;
}
```

**Vấn đề:** `ProjectNotEditableException` được throw nếu status không phải SUBMITTED/IN_REVISION. AC-8 yêu cầu không có restriction này.

**Fix:** Xóa status check. `updateDetails()` chỉ được gọi từ `UpdateCustomerProjectService` (verified bằng grep) — an toàn để sửa.

#### Gap 2 — Tenant isolation trả 404 thay vì 403 (vi phạm AC-2 vs AC-3)

**File:** `modules/application/src/main/java/com/tba/agentic/application/service/UpdateCustomerProjectService.java`

**Hiện tại:** `findByIdForCustomer` trả `Optional.empty()` cho CẢ HAI case: project không tồn tại AND project thuộc customer khác → throws `ProjectNotFoundException` → 404.

**Vấn đề:** AC yêu cầu phân biệt: wrong-customer → 403, không tồn tại → 404.

**Fix cần làm:**
1. Thêm method `existsById(ProjectId projectId)` vào `ProjectRepository` port
2. Implement trong `JooqProjectRepository`
3. Tạo `ProjectAccessDeniedException` trong `core/domain/exception/`
4. Thêm handler vào `GlobalExceptionHandler` (→ 403)
5. Sửa `UpdateCustomerProjectService`: nếu `findByIdForCustomer` trả empty → check `existsById` → if exists: throw `ProjectAccessDeniedException` (403); if not: throw `ProjectNotFoundException` (404)

#### Gap 3 — `name` validation cho phép empty string (vi phạm AC-4)

**File:** `modules/infrastructure/src/main/java/com/tba/agentic/adapter/transfer/request/UpdateCustomerProjectRequest.java`

**Hiện tại:** `@Size(max = 100) String name` — allows `""` (size 0 ≤ 100).

**Fix:** Đổi thành `@Size(min = 1, max = 100) String name`. `@Size` với null String → valid (null = field vắng mặt, PATCH semantics OK).

#### Gap 4 — Reference empty string không set về null (vi phạm AC-7)

**File:** `modules/core/src/main/java/com/tba/agentic/domain/entity/project/Project.java`

**Hiện tại trong `updateDetails()`:**
```java
if (reference != null) this.reference = reference;
```
`""` passes null check → `this.reference = ""` (không set về null).

**Fix:** Thêm empty-string → null conversion:
```java
if (reference != null) {
    this.reference = reference.isEmpty() ? null : reference;
}
```

**Note về PATCH null semantics:** `null` trong JSON body = field vắng mặt = không update. `""` = explicitly clear. Client muốn xóa reference phải gửi `{"reference": ""}`, không phải `{"reference": null}`.

---

## Tasks / Subtasks

### Task 1: Sửa `Project.updateDetails()` — xóa status restriction + fix reference clear

**File:** `modules/core/src/main/java/com/tba/agentic/domain/entity/project/Project.java`

Sửa method `updateDetails` (line ~96):

```java
public void updateDetails(String name, String description, String reference) {
  if (name != null) this.name = name;
  if (description != null) this.description = description;
  if (reference != null) {
    this.reference = reference.isEmpty() ? null : reference;
  }
}
```

Xóa toàn bộ status check và `import` của `ProjectNotEditableException` nếu không còn dùng ở đây (class vẫn tồn tại, chỉ không dùng trong method này).

- [ ] Xóa status restriction block
- [ ] Thêm empty-string → null conversion cho reference

---

### Task 2: Tạo `ProjectAccessDeniedException`

**File mới:** `modules/core/src/main/java/com/tba/agentic/domain/exception/ProjectAccessDeniedException.java`

```java
package com.tba.agentic.domain.exception;

public class ProjectAccessDeniedException extends RuntimeException {
  public ProjectAccessDeniedException(String projectId) {
    super("Access denied to project: " + projectId);
  }
}
```

- [ ] Tạo file

---

### Task 3: Thêm `existsById` vào `ProjectRepository` port

**File:** `modules/core/src/main/java/com/tba/agentic/port/out/ProjectRepository.java`

Thêm method sau `findByIdForCustomer`:

```java
boolean existsById(ProjectId projectId);
```

- [ ] Thêm method vào interface (với import `ProjectId` — đã có trong file)

---

### Task 4: Implement `existsById` trong `JooqProjectRepository`

**File:** `modules/infrastructure/src/main/java/com/tba/agentic/infrastructure/repository/database/JooqProjectRepository.java`

Thêm method implementation:

```java
@Override
public boolean existsById(ProjectId projectId) {
  return dsl.selectCount()
      .from(PROJECTS)
      .where(PROJECTS.PROJECT_ID.eq(projectId.value()))
      .fetchOne(0, Integer.class) > 0;
}
```

- [ ] Thêm method vào class

---

### Task 5: Sửa `UpdateCustomerProjectService` — phân biệt 403 vs 404

**File:** `modules/application/src/main/java/com/tba/agentic/application/service/UpdateCustomerProjectService.java`

**Hiện tại:**
```java
Project project = projectRepository
    .findByIdForCustomer(cmd.projectId(), cmd.customerId())
    .orElseThrow(() -> new ProjectNotFoundException(cmd.projectId().value().toString()));
```

**Sửa thành:**
```java
Project project = projectRepository
    .findByIdForCustomer(cmd.projectId(), cmd.customerId())
    .orElseThrow(() -> {
      if (projectRepository.existsById(cmd.projectId())) {
        return new ProjectAccessDeniedException(cmd.projectId().value().toString());
      }
      return new ProjectNotFoundException(cmd.projectId().value().toString());
    });
```

Thêm import cho `ProjectAccessDeniedException`.

- [ ] Sửa orElseThrow logic
- [ ] Thêm import `com.tba.agentic.domain.exception.ProjectAccessDeniedException`

---

### Task 6: Thêm handler vào `GlobalExceptionHandler`

**File:** `modules/infrastructure/src/main/java/com/tba/agentic/config/GlobalExceptionHandler.java`

Thêm sau handler của `ProjectNotFoundException` (line ~97):

```java
@ExceptionHandler(com.tba.agentic.domain.exception.ProjectAccessDeniedException.class)
public ResponseEntity<ObjectNode> handleProjectAccessDenied(
    com.tba.agentic.domain.exception.ProjectAccessDeniedException ex, HttpServletRequest request) {
  return buildErrorResponse("ACCESS_DENIED", ex.getMessage(), HttpStatus.FORBIDDEN, request);
}
```

Thêm import nếu cần (có thể dùng fully-qualified name như trên để tránh conflict).

- [ ] Thêm @ExceptionHandler method

---

### Task 7: Fix `name` validation trong `UpdateCustomerProjectRequest`

**File:** `modules/infrastructure/src/main/java/com/tba/agentic/adapter/transfer/request/UpdateCustomerProjectRequest.java`

**Hiện tại:**
```java
public record UpdateCustomerProjectRequest(
    @Size(max = 100) String name,
    @Size(min = 50, max = 2000) String description,
    @Size(max = 500) String reference) {}
```

**Sửa thành:**
```java
public record UpdateCustomerProjectRequest(
    @Size(min = 1, max = 100) String name,
    @Size(min = 50, max = 2000) String description,
    @Size(max = 500) String reference) {}
```

- [ ] Đổi `@Size(max = 100)` → `@Size(min = 1, max = 100)` cho `name`

---

## Verification Checklist

- [ ] `make compile` không có lỗi
- [ ] Test AC-1: PATCH với `{"name": "New Name"}` → chỉ name update, description/reference giữ nguyên
- [ ] Test AC-2: PATCH với projectId của customer khác → 403 (không phải 404)
- [ ] Test AC-3: PATCH với UUID không tồn tại trong DB → 404
- [ ] Test AC-4: PATCH với `{"name": ""}` → 400 validation error
- [ ] Test AC-4: PATCH với `{"name": "a".repeat(101)}` → 400 validation error
- [ ] Test AC-5: PATCH với `{"description": "short"}` → 400 validation error
- [ ] Test AC-6: PATCH với `{"reference": "a".repeat(501)}` → 400 validation error
- [ ] Test AC-7: PATCH với `{"reference": ""}` → reference field set về null trong DB
- [ ] Test AC-8: PATCH trên project ở status ANALYZING (hay bất kỳ status nào) → 200 OK (không bị 422)
- [ ] Test: PATCH không có auth header → 401
- [ ] `make test` pass

---

## Files Touched Summary

| Action | File |
|--------|------|
| MODIFY | `modules/core/.../entity/project/Project.java` |
| NEW | `modules/core/.../exception/ProjectAccessDeniedException.java` |
| MODIFY | `modules/core/.../port/out/ProjectRepository.java` |
| MODIFY | `modules/application/.../service/UpdateCustomerProjectService.java` |
| MODIFY | `modules/infrastructure/.../repository/database/JooqProjectRepository.java` |
| MODIFY | `modules/infrastructure/.../config/GlobalExceptionHandler.java` |
| MODIFY | `modules/infrastructure/.../request/UpdateCustomerProjectRequest.java` |
