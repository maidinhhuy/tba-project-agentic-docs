# Story 3.3: Backend — Get Project Detail

Status: ready-for-dev

## Story

As a customer,
I want to retrieve the full detail of one of my projects via `GET /api/v1/customer/projects/{id}`,
so that I can see its status, milestones, and activity feed (status history).

## Acceptance Criteria

1. **Given** `GET /api/v1/customer/projects/{id}` với JWT hợp lệ (role=CUSTOMER) và project thuộc về customer đang đăng nhập,
   **When** request được xử lý,
   **Then** trả về HTTP 200 với: tất cả project fields, `milestones` array sorted by `position ASC`, `statusHistory` array sorted by `changed_at DESC`.

2. **Given** `GET /api/v1/customer/projects/{id}` với JWT hợp lệ (role=CUSTOMER) nhưng project KHÔNG thuộc về customer đang đăng nhập (hoặc không tồn tại),
   **When** request được xử lý,
   **Then** trả về HTTP 404 — tenant isolation, không tiết lộ project có tồn tại hay không.

## Tasks / Subtasks

<!-- ════════════════════════════════════════════════════════
     TASK 1: DOMAIN DEFINITION — chỉ files trong core/
     ════════════════════════════════════════════════════════ -->

- [ ] Task 1: Domain Definition — core module (AC: #1, #2)

  <!-- Value Object: StatusHistoryEntry -->
  - [ ] Value Object: `StatusHistoryEntry`
    - File: `core/src/main/java/com/tba/agentic/domain/value/project/StatusHistoryEntry.java`
    - Java record:
      ```java
      public record StatusHistoryEntry(
          String previousStatus,   // nullable — null cho entry đầu tiên (SUBMITTED)
          String newStatus,
          String changedBy,
          String reason,           // nullable — chỉ có khi admin thêm progress note
          java.time.OffsetDateTime changedAt
      ) {}
      ```
    - Không expose `ipAddress` — customer-facing endpoint

  <!-- Value Object: ProjectDetail — wrapper cho Project + statusHistory -->
  - [ ] Value Object: `ProjectDetail`
    - File: `core/src/main/java/com/tba/agentic/domain/value/project/ProjectDetail.java`
    - Java record:
      ```java
      public record ProjectDetail(
          Project project,
          java.util.List<StatusHistoryEntry> statusHistory
      ) {}
      ```
    - Use case trả về type này — tách biệt query model khỏi Project aggregate

  <!-- Command: GetProjectDetailCommand -->
  - [ ] Command: `GetProjectDetailCommand`
    - File: `core/src/main/java/com/tba/agentic/port/bound/project/GetProjectDetailCommand.java`
    - Java record:
      ```java
      public record GetProjectDetailCommand(UserId customerId, ProjectId projectId) {}
      ```

  <!-- Inbound Port: GetProjectDetailUseCase -->
  - [ ] Inbound Port: `GetProjectDetailUseCase`
    - File: `core/src/main/java/com/tba/agentic/port/in/GetProjectDetailUseCase.java`
    - Interface:
      ```java
      public interface GetProjectDetailUseCase {
          ProjectDetail execute(GetProjectDetailCommand cmd);
          // throws ProjectNotFoundException nếu không tìm thấy hoặc tenant isolation fail
      }
      ```

  <!-- Update Outbound Port: ProjectRepository — thêm 2 methods -->
  - [ ] Update Outbound Port: `ProjectRepository`
    - File: `core/src/main/java/com/tba/agentic/port/out/ProjectRepository.java`
    - Thêm 2 methods vào interface (giữ `save` không thay đổi):
      ```java
      java.util.Optional<Project> findByIdForCustomer(ProjectId projectId, UserId customerId);
      java.util.List<StatusHistoryEntry> findStatusHistory(ProjectId projectId);
      ```
    - `findByIdForCustomer` trả về `Optional.empty()` khi project không tồn tại HOẶC customer_id không khớp (tenant isolation — không phân biệt 2 case)
    - `findStatusHistory` trả về list empty nếu không có history (không throw)

  - [ ] Verify Task 1:
    - `./gradlew :core:compileJava` → BUILD SUCCESSFUL
    - `grep -r "import org.springframework" core/src` → zero results
    - `grep -r "import org.jooq" core/src` → zero results

<!-- ════════════════════════════════════════════════════════
     TASK 2: IMPLEMENTATION — chỉ files trong application/
     Depends On: Task 1 — Domain Definition
     ════════════════════════════════════════════════════════ -->

- [ ] Task 2: Implementation — application module (AC: #1, #2)
  **Depends On: Task 1 — Domain Definition**

  <!-- KHÔNG có DB Migration — schema đã đầy đủ (project_status_history đã tồn tại) -->
  <!-- KHÔNG cần thay đổi SecurityConfig — /api/v1/customer/** → hasRole("CUSTOMER") đã có -->
  <!-- KHÔNG cần thay đổi GlobalExceptionHandler — ProjectNotFoundException → 404 đã được handle -->

  <!-- Update Response DTO: ProjectDetailResponse — thêm statusHistory -->
  - [ ] Update Response DTO: `ProjectDetailResponse`
    - File: `application/src/main/java/com/tba/agentic/adapter/transfer/response/ProjectDetailResponse.java`
    - Rewrite toàn bộ record — thêm `statusHistory` field và static factory `from()`:
      ```java
      public record ProjectDetailResponse(
          String projectId,
          String name,
          String productType,
          String description,
          String reference,
          String status,
          int revisionCount,
          List<MilestoneDto> milestones,
          List<StatusHistoryDto> statusHistory,
          String createdAt,
          String updatedAt,
          int version) {

        public record MilestoneDto(Long milestoneId, String name, String status, int position) {}

        public record StatusHistoryDto(
            String previousStatus,
            String newStatus,
            String changedBy,
            String reason,
            String changedAt) {}

        public static ProjectDetailResponse from(ProjectDetail detail) {
          Project p = detail.project();
          List<MilestoneDto> milestones = p.getMilestones().stream()
              .sorted(Comparator.comparingInt(Milestone::getPosition))
              .map(m -> new MilestoneDto(m.getMilestoneId(), m.getName(), m.getStatus().name(), m.getPosition()))
              .toList();
          List<StatusHistoryDto> history = detail.statusHistory().stream()
              .map(h -> new StatusHistoryDto(
                  h.previousStatus(), h.newStatus(), h.changedBy(), h.reason(),
                  h.changedAt() != null ? h.changedAt().toString() : null))
              .toList();
          return new ProjectDetailResponse(
              p.getProjectId().value().toString(),
              p.getName(),
              p.getProductType().name(),
              p.getDescription(),
              p.getReference(),
              p.getStatus().name(),
              p.getRevisionCount(),
              milestones,
              history,
              p.getCreatedAt() != null ? p.getCreatedAt().toString() : null,
              p.getUpdatedAt() != null ? p.getUpdatedAt().toString() : null,
              p.getVersion());
        }
      }
      ```

  <!-- Service implementation -->
  - [ ] Service impl: `GetProjectDetailService implements GetProjectDetailUseCase`
    - File: `application/src/main/java/com/tba/agentic/application/service/GetProjectDetailService.java`
    - Annotate `@Service @RequiredArgsConstructor`
    - Inject `ProjectRepository projectRepository`
    - Logic `execute(GetProjectDetailCommand cmd)`:
      ```java
      Project project = projectRepository
          .findByIdForCustomer(cmd.projectId(), cmd.customerId())
          .orElseThrow(() -> new ProjectNotFoundException(cmd.projectId().value().toString()));
      List<StatusHistoryEntry> history = projectRepository.findStatusHistory(cmd.projectId());
      return new ProjectDetail(project, history);
      ```

  <!-- JooqProjectRepository — thêm 2 methods mới -->
  - [ ] Update Repository impl: `JooqProjectRepository`
    - File: `application/src/main/java/com/tba/agentic/infrastructure/repository/database/JooqProjectRepository.java`
    - Thêm import: `import com.tba.agentic.domain.value.project.StatusHistoryEntry;`
    - Thêm import: `import org.jooq.SortField;` (đã có DSLContext)
    - Implement `findByIdForCustomer(ProjectId projectId, UserId customerId)`:
      ```java
      @Override
      public Optional<Project> findByIdForCustomer(ProjectId projectId, UserId customerId) {
          ProjectsRecord pr = dsl.selectFrom(PROJECTS)
              .where(PROJECTS.PROJECT_ID.eq(projectId.value()))
              .and(PROJECTS.CUSTOMER_ID.eq(customerId.value()))
              .fetchOne();
          if (pr == null) return Optional.empty();

          UUID projectUuid = pr.getProjectId();
          List<Milestone> milestones = dsl.selectFrom(MILESTONES)
              .where(MILESTONES.PROJECT_ID.eq(projectUuid))
              .orderBy(MILESTONES.POSITION.asc())
              .fetch()
              .map(m -> Milestone.reconstitute(
                  m.getMilestoneId(), m.getName(),
                  MilestoneStatus.valueOf(m.getStatus()), m.getPosition()));

          return Optional.of(Project.reconstitute(
              new ProjectId(projectUuid),
              new UserId(pr.getCustomerId()),
              pr.getName(),
              ProductType.valueOf(pr.getProductType()),
              pr.getDescription(),
              pr.getReference(),
              ProjectStatus.valueOf(pr.getStatus()),
              pr.getRevisionCount(),
              milestones,
              pr.getVersion(),
              pr.getCreatedAt(),
              pr.getUpdatedAt()));
      }
      ```
    - Implement `findStatusHistory(ProjectId projectId)`:
      ```java
      @Override
      public List<StatusHistoryEntry> findStatusHistory(ProjectId projectId) {
          return dsl.selectFrom(PROJECT_STATUS_HISTORY)
              .where(PROJECT_STATUS_HISTORY.PROJECT_ID.eq(projectId.value()))
              .orderBy(PROJECT_STATUS_HISTORY.CHANGED_AT.desc())
              .fetch()
              .map(h -> new StatusHistoryEntry(
                  h.getPreviousStatus(),
                  h.getNewStatus(),
                  h.getChangedBy(),
                  h.getReason(),
                  h.getChangedAt()));
          // jOOQ Result.map() đã trả về java.util.List — không cần stream().toList()
      }
      ```

  <!-- Controller — thêm GET /{projectId} endpoint -->
  - [ ] Update Controller: `CustomerProjectController`
    - File: `application/src/main/java/com/tba/agentic/adapter/controller/CustomerProjectController.java`
    - Thêm field injection: `private final GetProjectDetailUseCase getProjectDetailUseCase;`
    - Thêm imports: `GetProjectDetailCommand`, `GetProjectDetailUseCase`, `ProjectDetail`, `ProjectDetailResponse`, `GetMapping`, `PathVariable`, `UUID`
    - Thêm endpoint:
      ```java
      @GetMapping("/{projectId}")
      public ResponseEntity<ProjectDetailResponse> getProjectDetail(
          @PathVariable String projectId,
          @AuthenticationPrincipal Long userId) {
        GetProjectDetailCommand cmd = new GetProjectDetailCommand(
            new UserId(userId),
            new ProjectId(UUID.fromString(projectId)));
        ProjectDetail detail = getProjectDetailUseCase.execute(cmd);
        return ResponseEntity.ok(ProjectDetailResponse.from(detail));
      }
      ```
    - `UUID.fromString(projectId)` sẽ throw `IllegalArgumentException` nếu format sai → GlobalExceptionHandler xử lý → 400 BAD_REQUEST (acceptable)

  - [ ] Verify Task 2:
    - `./gradlew :application:compileJava` → BUILD SUCCESSFUL
    - Curl tests (xem Dev Notes bên dưới)

## Dev Notes

### Trạng thái hiện tại của codebase

| File | Trạng thái | Ghi chú |
|------|-----------|---------|
| `core/.../domain/value/project/StatusHistoryEntry.java` | **CHƯA TỒN TẠI** | Tạo mới |
| `core/.../domain/value/project/ProjectDetail.java` | **CHƯA TỒN TẠI** | Tạo mới |
| `core/.../port/bound/project/GetProjectDetailCommand.java` | **CHƯA TỒN TẠI** | Tạo mới |
| `core/.../port/in/GetProjectDetailUseCase.java` | **CHƯA TỒN TẠI** | Tạo mới |
| `core/.../port/out/ProjectRepository.java` | **ĐÃ TỒN TẠI** | Chỉ có `save()` — cần thêm 2 methods |
| `application/.../adapter/transfer/response/ProjectDetailResponse.java` | **ĐÃ TỒN TẠI** | Thiếu `statusHistory` — cần update |
| `application/.../application/service/GetProjectDetailService.java` | **CHƯA TỒN TẠI** | Tạo mới |
| `application/.../infrastructure/repository/database/JooqProjectRepository.java` | **ĐÃ TỒN TẠI** | Chỉ có `save()` — cần thêm 2 methods |
| `application/.../adapter/controller/CustomerProjectController.java` | **ĐÃ TỒN TẠI** | Chỉ có POST — cần thêm GET /{projectId} |
| `application/.../config/SecurityConfig.java` | **ĐÃ TỒN TẠI** | `/api/v1/customer/**` → CUSTOMER — KHÔNG thay đổi |
| `application/.../config/GlobalExceptionHandler.java` | **ĐÃ TỒN TẠI** | `ProjectNotFoundException` → 404 — KHÔNG thay đổi |

### jOOQ Table References

```java
import static com.tba.agentic.infrastructure.jooq.Tables.PROJECTS;
import static com.tba.agentic.infrastructure.jooq.Tables.MILESTONES;
import static com.tba.agentic.infrastructure.jooq.Tables.PROJECT_STATUS_HISTORY;
```

| Table | Columns quan trọng |
|-------|-------------------|
| `PROJECTS` | `PROJECT_ID` (UUID), `CUSTOMER_ID` (Long), `NAME`, `PRODUCT_TYPE`, `DESCRIPTION`, `REFERENCE`, `STATUS`, `REVISION_COUNT`, `VERSION`, `CREATED_AT`, `UPDATED_AT` |
| `MILESTONES` | `MILESTONE_ID` (BIGSERIAL/Long), `PROJECT_ID` (UUID), `NAME`, `STATUS`, `POSITION` |
| `PROJECT_STATUS_HISTORY` | `PROJECT_ID` (UUID), `PREVIOUS_STATUS` (nullable String), `NEW_STATUS`, `CHANGED_BY`, `REASON` (nullable), `CHANGED_AT` (OffsetDateTime), `IP_ADDRESS` (deprecated/unknown type — skip) |

### Tenant Isolation Pattern

Tất cả project queries cho customer PHẢI filter theo cả `project_id` VÀ `customer_id` trong cùng 1 WHERE clause. Không bao giờ query by `project_id` alone rồi check ownership trong memory — DB phải enforce isolation:

```java
// ĐÚNG — tenant isolation tại DB level
dsl.selectFrom(PROJECTS)
    .where(PROJECTS.PROJECT_ID.eq(projectId.value()))
    .and(PROJECTS.CUSTOMER_ID.eq(customerId.value()))   // <-- MANDATORY
    .fetchOne();
```

### Controller — UUID parsing

```java
// UUID.fromString sẽ throw IllegalArgumentException nếu format không hợp lệ
// GlobalExceptionHandler xử lý IllegalArgumentException → 400 BAD_REQUEST
// Không cần try/catch trong controller
new ProjectId(UUID.fromString(projectId))
```

### `ProjectDetailResponse.from()` — import cần thiết

```java
import com.tba.agentic.domain.entity.project.Project;
import com.tba.agentic.domain.value.project.Milestone;
import com.tba.agentic.domain.value.project.ProjectDetail;
import com.tba.agentic.domain.value.project.StatusHistoryEntry;
import java.util.Comparator;
import java.util.List;
```

### Import statements cho các file mới (Task 2)

**GetProjectDetailService:**
```java
import com.tba.agentic.domain.entity.project.Project;
import com.tba.agentic.domain.exception.ProjectNotFoundException;
import com.tba.agentic.domain.value.project.ProjectDetail;
import com.tba.agentic.domain.value.project.StatusHistoryEntry;
import com.tba.agentic.port.bound.project.GetProjectDetailCommand;
import com.tba.agentic.port.in.GetProjectDetailUseCase;
import com.tba.agentic.port.out.ProjectRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import java.util.List;
```

**JooqProjectRepository (thêm vào file hiện có):**
```java
import com.tba.agentic.domain.value.project.StatusHistoryEntry;
import java.util.Optional;
// (các import khác đã có)
```

**CustomerProjectController (thêm vào file hiện có):**
```java
import com.tba.agentic.adapter.transfer.response.ProjectDetailResponse;
import com.tba.agentic.domain.value.project.ProjectDetail;
import com.tba.agentic.domain.value.project.ProjectId;
import com.tba.agentic.port.bound.project.GetProjectDetailCommand;
import com.tba.agentic.port.in.GetProjectDetailUseCase;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import java.util.UUID;
```

### Curl tests

```bash
# Setup — lấy token sau khi login
TOKEN="<jwt_cookie_value>"
PROJECT_ID="<uuid-của-project>"

# Happy path — lấy detail project của chính mình
curl -X GET http://localhost:8080/api/v1/customer/projects/$PROJECT_ID \
  -H "Cookie: tba_access_token=$TOKEN" -v
# Expected: 200 + ProjectDetailResponse body với milestones sorted by position ASC
#           và statusHistory sorted by changed_at DESC

# Tenant isolation — project của customer khác
curl -X GET http://localhost:8080/api/v1/customer/projects/<uuid-khác> \
  -H "Cookie: tba_access_token=$TOKEN" -v
# Expected: 404 + {"error": "PROJECT_NOT_FOUND", ...}

# Unauthorized — không có cookie
curl -X GET http://localhost:8080/api/v1/customer/projects/$PROJECT_ID -v
# Expected: 401

# Invalid UUID format
curl -X GET http://localhost:8080/api/v1/customer/projects/not-a-uuid \
  -H "Cookie: tba_access_token=$TOKEN" -v
# Expected: 400 BAD_REQUEST
```

### References

- [Source: epics.md — Story 3.3] — ACs gốc, tenant isolation requirement (HTTP 404)
- [Source: Project.java] — `reconstitute()` signature (11 params), private constructor
- [Source: Milestone.java] — `reconstitute(milestoneId, name, status, position)` 4-param static factory
- [Source: ProjectRepository.java] — interface hiện tại (chỉ có `save`)
- [Source: JooqProjectRepository.java] — pattern jOOQ SELECT + mapping đã dùng trong `save()`
- [Source: ProjectStatusHistoryRecord.java] — columns: id, project_id, previous_status, new_status, changed_by, reason, changed_at, ip_address
- [Source: CustomerProjectController.java] — pattern `@AuthenticationPrincipal Long userId`
- [Source: GlobalExceptionHandler.java] — `ProjectNotFoundException` → 404 đã handle, `IllegalArgumentException` → 400 đã handle
- [Source: SecurityConfig.java] — `/api/v1/customer/**` → hasRole("CUSTOMER"), không cần thay đổi
- [Source: 3-1-backend-submit-project.md] — pattern story trước, jOOQ insert patterns

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

### File List
