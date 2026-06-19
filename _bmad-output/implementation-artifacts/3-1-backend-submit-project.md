# Story 3.1: Backend — Submit Project

Status: ready-for-dev

## Story

As a customer,
I want to submit a new project via `POST /api/v1/customer/projects`,
so that the agency receives my project requirements and I receive the created project data with status `SUBMITTED`.

## Acceptance Criteria

1. **Given** `POST /api/v1/customer/projects` với payload hợp lệ: `name` (max 100 ký tự), `productType` (`WEB` hoặc `MOBILE`), `description` (min 50, max 2000 ký tự), `reference` (tuỳ chọn, max 500 ký tự),
   **When** Customer đã đăng nhập (JWT hợp lệ, role=CUSTOMER) gửi request,
   **Then** trả về HTTP 201 với body `SubmitProjectResponse`: `status=SUBMITTED`, `revisionCount=3`, và 3 milestones: `{ name: "UI/UX & Prototype", position: 1, status: ACTIVE }`, `{ name: "Backend Development", position: 2, status: PENDING }`, `{ name: "Frontend Integration & Deploy", position: 3, status: PENDING }` — mỗi milestone có `milestoneId` từ DB.
   **And** `project_status_history` có 1 record: `previous_status=null`, `new_status=SUBMITTED`, `changed_by=<customer_email>`.

2. **Given** payload thiếu `name`, hoặc `description` dưới 50 ký tự, hoặc `productType` không phải `WEB`/`MOBILE`, hoặc các field vi phạm max length,
   **When** gửi request,
   **Then** trả về HTTP 400 với error `"VALIDATION_FAILED"` và `details` map chỉ rõ field nào sai.

3. **Given** project vừa được tạo thành công (HTTP 201),
   **When** `POST /api/v1/customer/projects` returns 201,
   **Then** `EmailClient.send()` được gọi async với email tới `${TBA_ADMIN_EMAIL:admin@tba.com}`, subject `"Dự án mới cần xem xét"`, body gồm: tên Project, displayName Customer, loại sản phẩm, link `${TBA_APP_API_FRONTEND_BASE_URL}/admin/projects/{projectId}`.
   **And** email failure KHÔNG ảnh hưởng đến HTTP response — exception bị catch và log.

4. **Given** Customer A đã đăng nhập gửi request,
   **When** project được tạo thành công,
   **Then** `project.customer_id` = Customer A's `user_id` (tenant isolation đúng).

## Tasks / Subtasks

<!-- ════════════════════════════════════════════════════════
     TASK 1: DOMAIN DEFINITION — chỉ files trong core/
     ════════════════════════════════════════════════════════ -->

- [ ] Task 1: Domain Definition — core module (AC: #1, #2, #3, #4)

  <!-- Project aggregate đã tồn tại, cần thêm reconstitute() factory -->
  - [ ] Modify `Project` aggregate — thêm `reconstitute()` static factory
    - File: `core/src/main/java/com/tba/agentic/domain/entity/project/Project.java`
    - Thêm static method:
      ```java
      public static Project reconstitute(
          ProjectId projectId, UserId customerId, String name,
          ProductType productType, String description, String reference,
          ProjectStatus status, int revisionCount, List<Milestone> milestones,
          int version, OffsetDateTime createdAt, OffsetDateTime updatedAt) {
        Project p = new Project();
        p.projectId = projectId; p.customerId = customerId; p.name = name;
        p.productType = productType; p.description = description;
        p.reference = reference; p.status = status;
        p.revisionCount = revisionCount; p.milestones = milestones;
        p.version = version; p.createdAt = createdAt; p.updatedAt = updatedAt;
        return p;
      }
      ```
    - Cần thiết vì repository phải trả về Project với milestone IDs từ DB (BIGSERIAL)

  <!-- Bound Command -->
  - [ ] Bound — Command: `SubmitProjectCommand`
    - File: `core/src/main/java/com/tba/agentic/port/bound/project/SubmitProjectCommand.java`
    - Java record với fields:
      ```
      UserId customerId       // VO — owner của project
      String name             // plain String — validate ở controller layer
      ProductType productType // domain enum
      String description      // plain String
      String reference        // plain String, nullable
      ```
    - Không bao gồm `changedBy` hay `displayName` — service tự load User từ `UserRepository`

  <!-- Inbound Port -->
  - [ ] Inbound Port: `SubmitProjectUseCase`
    - File: `core/src/main/java/com/tba/agentic/port/in/SubmitProjectUseCase.java`
    - Method: `Project execute(SubmitProjectCommand cmd)`

  <!-- Outbound Port — ProjectRepository -->
  - [ ] Outbound Port: `ProjectRepository`
    - File: `core/src/main/java/com/tba/agentic/port/out/ProjectRepository.java`
    - Methods:
      ```java
      Project save(Project project, String changedBy);
      // changedBy = customer email, dùng để ghi vào project_status_history
      ```

  <!-- Outbound Port — EmailClient -->
  - [ ] Outbound Port: `EmailClient`
    - File: `core/src/main/java/com/tba/agentic/port/out/EmailClient.java`
    - Method: `void send(EmailMessage message)`

  <!-- EmailMessage record -->
  - [ ] Value Object: `EmailMessage`
    - File: `core/src/main/java/com/tba/agentic/port/out/EmailMessage.java`
    - Java record:
      ```java
      public record EmailMessage(String to, String subject, String htmlBody) {}
      ```

  - [ ] Verify Task 1:
    - `./gradlew :core:compileJava` → BUILD SUCCESSFUL
    - `grep -r "import org.springframework" core/src` → zero results
    - `grep -r "import org.jooq" core/src` → zero results

<!-- ════════════════════════════════════════════════════════
     TASK 2: IMPLEMENTATION — chỉ files trong application/
     Depends On: Task 1 — Domain Definition
     ════════════════════════════════════════════════════════ -->

- [ ] Task 2: Implementation — application module (AC: #1, #2, #3, #4)
  **Depends On: Task 1 — Domain Definition**

  <!-- KHÔNG có DB Migration — schema đã đầy đủ từ Story 1.2 -->
  <!-- KHÔNG cần thay đổi SecurityConfig — rule hasRole("CUSTOMER") cho /api/v1/customer/** đã có -->
  <!-- KHÔNG cần thay đổi GlobalExceptionHandler — đã handle MethodArgumentNotValidException → 400 -->

  <!-- Request DTO -->
  - [ ] Request DTO: `SubmitProjectRequest`
    - File: `application/src/main/java/com/tba/agentic/adapter/transfer/request/SubmitProjectRequest.java`
    - Java record, dùng primitive String fields với Bean Validation:
      ```java
      public record SubmitProjectRequest(
          @NotBlank @Size(max = 100) String name,
          @NotNull @Pattern(regexp = "WEB|MOBILE", message = "productType must be WEB or MOBILE") String productType,
          @NotBlank @Size(min = 50, max = 2000) String description,
          @Size(max = 500) String reference    // optional, null allowed
      ) {}
      ```

  <!-- Repository implementation -->
  - [ ] Repository impl: `JooqProjectRepository implements ProjectRepository`
    - File: `application/src/main/java/com/tba/agentic/infrastructure/repository/database/JooqProjectRepository.java`
    - Annotate `@Repository @RequiredArgsConstructor`
    - Inject `DSLContext dsl`
    - Implement `save(Project project, String changedBy)` với `@Transactional`:
      1. INSERT INTO `PROJECTS` (project_id, customer_id, name, product_type, description, reference, status, revision_count, version) RETURNING * → lấy `createdAt`, `updatedAt`
      2. Batch INSERT INTO `MILESTONES` (project_id, name, status, position) RETURNING * → lấy `milestone_id` cho từng row (3 milestones)
      3. INSERT INTO `PROJECT_STATUS_HISTORY` (project_id, previous_status=null, new_status='SUBMITTED', changed_by) → không RETURNING
      4. Dùng `Project.reconstitute(...)` để trả về Project đầy đủ với milestone IDs từ DB
    - jOOQ batch milestone insert pattern:
      ```java
      var milestoneRecords = project.getMilestones().stream()
          .map(m -> dsl.insertInto(MILESTONES)
              .set(MILESTONES.PROJECT_ID, projectUuid)
              .set(MILESTONES.NAME, m.getName())
              .set(MILESTONES.STATUS, m.getStatus().name())
              .set(MILESTONES.POSITION, m.getPosition())
              .returning())
          .collect(Collectors.toList()); // execute individually to get RETURNING
      ```

  <!-- Email stub implementation (dev mode — Story 2.2 sẽ thay bằng ResendEmailClient) -->
  - [ ] Email stub: `DevEmailClient implements EmailClient`
    - File: `application/src/main/java/com/tba/agentic/infrastructure/email/DevEmailClient.java`
    - Annotate `@Service @Primary` (Story 2.2 sẽ tạo `ResendEmailClient` và replace `@Primary`)
    - Logic: log `[EMAIL-DEV] To: {to}, Subject: {subject}` ở INFO level, không throw exception
    - Sẽ bị replace bởi Story 2.2 (ResendEmailClient) — đây chỉ là dev-mode stub

  <!-- Service implementation -->
  - [ ] Service impl: `SubmitProjectService implements SubmitProjectUseCase`
    - File: `application/src/main/java/com/tba/agentic/application/service/SubmitProjectService.java`
    - Annotate `@Service @RequiredArgsConstructor`
    - Inject: `ProjectRepository projectRepository`, `UserRepository userRepository`, `EmailClient emailClient`
    - `@Value("${TBA_ADMIN_EMAIL:admin@tba.com}") String adminEmail`
    - `@Value("${TBA_APP_API_FRONTEND_BASE_URL:http://localhost:3000}") String frontendBaseUrl`
    - Logic `execute(SubmitProjectCommand cmd)`:
      1. `userRepository.findById(cmd.customerId())` → throw `ProjectNotFoundException` thay bằng `RuntimeException` nếu không tìm thấy (defensive — customer phải tồn tại vì đã authenticate)
      2. `Project project = Project.create(cmd.customerId(), cmd.name(), cmd.productType(), cmd.description(), cmd.reference())`
      3. `Project saved = projectRepository.save(project, user.getAccount().email().value())`
      4. Async email (fire-and-forget, catch mọi exception):
         ```java
         try {
           emailClient.send(new EmailMessage(
               adminEmail,
               "Dự án mới cần xem xét",
               buildAdminEmailHtml(saved, user, frontendBaseUrl)
           ));
         } catch (Exception e) {
           log.warn("Admin notification failed for project {}: {}", saved.getProjectId(), e.getMessage());
         }
         ```
      5. Return `saved`
    - `buildAdminEmailHtml()`: trả về HTML string gồm tên Project, displayName Customer, loại sản phẩm, link `{frontendBaseUrl}/admin/projects/{projectId}`

  <!-- REST Controller -->
  - [ ] Controller: `CustomerProjectController`
    - File: `application/src/main/java/com/tba/agentic/adapter/controller/CustomerProjectController.java`
    - Annotate `@RestController @RequestMapping("/api/v1/customer/projects") @RequiredArgsConstructor`
    - Inject `SubmitProjectUseCase submitProjectUseCase`
    - Endpoint `POST /`:
      ```java
      @PostMapping
      public ResponseEntity<SubmitProjectResponse> submit(
          @Valid @RequestBody SubmitProjectRequest req,
          @AuthenticationPrincipal Long userId) {
        SubmitProjectCommand cmd = new SubmitProjectCommand(
            new UserId(userId),
            req.name(),
            ProductType.valueOf(req.productType()),
            req.description(),
            req.reference()
        );
        Project saved = submitProjectUseCase.execute(cmd);
        return ResponseEntity.status(HttpStatus.CREATED).body(toResponse(saved));
      }
      ```
    - `toResponse(Project p)` helper: map `Project` → `SubmitProjectResponse` (milestones sorted by position)
    - `@AuthenticationPrincipal Long userId` works vì `JwtAuthenticationFilter` set principal = `Long userId`

  - [ ] Verify Task 2:
    - `./gradlew :application:compileJava` → BUILD SUCCESSFUL
    - Curl tests (xem Dev Notes bên dưới)

## Dev Notes

### Trạng thái hiện tại của codebase

| File | Trạng thái | Ghi chú |
|------|-----------|---------|
| `core/.../domain/entity/project/Project.java` | **ĐÃ TỒN TẠI** | Cần thêm `reconstitute()` |
| `core/.../domain/value/project/ProjectStatus.java` | **ĐÃ TỒN TẠI** | 8 states, `transitionTo()`, `allowedTransitions()` |
| `core/.../domain/value/project/ProductType.java` | **ĐÃ TỒN TẠI** | enum `WEB`, `MOBILE` |
| `core/.../domain/value/project/MilestoneStatus.java` | **ĐÃ TỒN TẠI** | enum `PENDING`, `ACTIVE`, `DONE` |
| `core/.../domain/value/project/Milestone.java` | **ĐÃ TỒN TẠI** | class với constructor(milestoneId, projectId, name, status, position) |
| `core/.../domain/value/user/UserId.java` | **ĐÃ TỒN TẠI** | record với `Long value()` |
| `core/.../domain/exception/ProjectNotFoundException.java` | **ĐÃ TỒN TẠI** | |
| `application/.../adapter/transfer/response/SubmitProjectResponse.java` | **ĐÃ TỒN TẠI** | record đúng format, không thay đổi |
| `application/.../config/SecurityConfig.java` | **ĐÃ TỒN TẠI** | `/api/v1/customer/**` → `hasRole("CUSTOMER")` — KHÔNG thay đổi |
| `application/.../config/GlobalExceptionHandler.java` | **ĐÃ TỒN TẠI** | Handle `MethodArgumentNotValidException` → 400 VALIDATION_FAILED — KHÔNG thay đổi |
| `application/.../infrastructure/repository/database/PgDatabaseUserRepository.java` | **ĐÃ TỒN TẠI** | `@Primary`, có `findById(UserId)` — inject trong service |

### `Project.create()` — Factory hiện tại

```java
// Project.java line 30-48
public static Project create(UserId customerId, String name, ProductType productType,
                              String description, String reference) {
  Project p = new Project();
  p.projectId = ProjectId.generate();   // UUID.randomUUID()
  p.status = ProjectStatus.SUBMITTED;
  p.revisionCount = 3;
  p.milestones = createDefaultMilestones(); // 3 milestones, milestoneId=null
  p.version = 0;
  ...
  return p;
}

// createDefaultMilestones() trả về:
// Milestone(null, null, "UI/UX & Prototype",            MilestoneStatus.ACTIVE, 1)
// Milestone(null, null, "Backend Development",           MilestoneStatus.PENDING, 2)
// Milestone(null, null, "Frontend Integration & Deploy", MilestoneStatus.PENDING, 3)
```

`milestoneId=null` khi tạo mới — repository phải INSERT RETURNING để lấy BIGSERIAL IDs từ DB.

### `Milestone` constructor

```java
// Milestone.java
public Milestone(Long milestoneId, ProjectId projectId, String name, MilestoneStatus status, int position) { ... }
```

Sau khi INSERT RETURNING, tạo Milestone với ID:
```java
new Milestone(rec.getMilestoneId(), new ProjectId(rec.getProjectId()), rec.getName(),
              MilestoneStatus.valueOf(rec.getStatus()), rec.getPosition())
```

### jOOQ Table References

```java
import static com.tba.agentic.infrastructure.jooq.Tables.PROJECTS;
import static com.tba.agentic.infrastructure.jooq.Tables.MILESTONES;
import static com.tba.agentic.infrastructure.jooq.Tables.PROJECT_STATUS_HISTORY;
```

| Table | Columns quan trọng |
|-------|-------------------|
| `PROJECTS` | `PROJECT_ID` (UUID), `CUSTOMER_ID` (Long), `NAME`, `PRODUCT_TYPE`, `DESCRIPTION`, `REFERENCE`, `STATUS`, `REVISION_COUNT`, `VERSION`, `CREATED_AT`, `UPDATED_AT` |
| `MILESTONES` | `MILESTONE_ID` (BIGSERIAL), `PROJECT_ID` (UUID), `NAME`, `STATUS`, `POSITION` |
| `PROJECT_STATUS_HISTORY` | `PROJECT_ID` (UUID), `PREVIOUS_STATUS` (nullable), `NEW_STATUS`, `CHANGED_BY`, `CHANGED_AT` (default now()) |

### Pattern INSERT projects + RETURNING

```java
ProjectsRecord rec = dsl.insertInto(PROJECTS)
    .set(PROJECTS.PROJECT_ID, UUID.fromString(project.getProjectId().value().toString()))
    .set(PROJECTS.CUSTOMER_ID, project.getCustomerId().value())
    .set(PROJECTS.NAME, project.getName())
    .set(PROJECTS.PRODUCT_TYPE, project.getProductType().name())
    .set(PROJECTS.DESCRIPTION, project.getDescription())
    .set(PROJECTS.REFERENCE, project.getReference())
    .set(PROJECTS.STATUS, project.getStatus().name())
    .set(PROJECTS.REVISION_COUNT, project.getRevisionCount())
    .set(PROJECTS.VERSION, project.getVersion())
    .returning()
    .fetchOne();
// rec.getCreatedAt(), rec.getUpdatedAt() dùng để reconstitute
```

### Pattern INSERT milestones

```java
List<Milestone> savedMilestones = new ArrayList<>();
for (Milestone m : project.getMilestones()) {
    MilestonesRecord mRec = dsl.insertInto(MILESTONES)
        .set(MILESTONES.PROJECT_ID, UUID.fromString(project.getProjectId().value().toString()))
        .set(MILESTONES.NAME, m.getName())
        .set(MILESTONES.STATUS, m.getStatus().name())
        .set(MILESTONES.POSITION, m.getPosition())
        .returning()
        .fetchOne();
    savedMilestones.add(new Milestone(
        mRec.getMilestoneId(),
        new ProjectId(mRec.getProjectId()),
        mRec.getName(),
        MilestoneStatus.valueOf(mRec.getStatus()),
        mRec.getPosition()
    ));
}
```

### Pattern INSERT project_status_history

```java
dsl.insertInto(PROJECT_STATUS_HISTORY)
    .set(PROJECT_STATUS_HISTORY.PROJECT_ID, UUID.fromString(project.getProjectId().value().toString()))
    .set(PROJECT_STATUS_HISTORY.PREVIOUS_STATUS, (String) null)
    .set(PROJECT_STATUS_HISTORY.NEW_STATUS, ProjectStatus.SUBMITTED.name())
    .set(PROJECT_STATUS_HISTORY.CHANGED_BY, changedBy)
    .execute();
```

### `@Transactional` cho repository save

```java
@Transactional
public Project save(Project project, String changedBy) {
  // 1. INSERT projects
  // 2. INSERT milestones (per-row, collect IDs)
  // 3. INSERT project_status_history
  // 4. return Project.reconstitute(...)
}
```

`@Transactional` đảm bảo atomic — nếu bất kỳ bước nào fail, rollback toàn bộ.

### Controller — lấy userId từ JWT

```java
// JwtAuthenticationFilter set: auth.setPrincipal(userId) — type Long
// @AuthenticationPrincipal Long userId work trực tiếp

@PostMapping
public ResponseEntity<SubmitProjectResponse> submit(
    @Valid @RequestBody SubmitProjectRequest req,
    @AuthenticationPrincipal Long userId) { ... }
```

### `toResponse()` mapping

```java
private SubmitProjectResponse toResponse(Project p) {
    List<SubmitProjectResponse.SubmitProjectMilestoneResponse> milestones = p.getMilestones().stream()
        .sorted(Comparator.comparingInt(Milestone::getPosition))
        .map(m -> new SubmitProjectResponse.SubmitProjectMilestoneResponse(
            m.getMilestoneId(), m.getName(), m.getStatus().name(), m.getPosition()))
        .toList();
    return new SubmitProjectResponse(
        p.getProjectId().value().toString(),
        p.getName(),
        p.getProductType().name(),
        p.getStatus().name(),
        p.getRevisionCount(),
        milestones,
        p.getVersion()
    );
}
```

### DevEmailClient — Sẽ bị replace bởi Story 2.2

```java
@Service
@Primary
public class DevEmailClient implements EmailClient {
    private static final Logger log = LoggerFactory.getLogger(DevEmailClient.class);

    @Override
    public void send(EmailMessage message) {
        log.info("[EMAIL-DEV] To: {}, Subject: {}, Body: {}",
            message.to(), message.subject(), message.htmlBody());
    }
}
```

Khi Story 2.2 (ResendEmailClient) được implement:
- `ResendEmailClient` thêm `@Primary` (hoặc `DevEmailClient` xóa `@Primary`)
- `DevEmailClient` có thể xóa hoặc đổi thành `@ConditionalOnMissingBean`

### Curl tests

```bash
# Setup — lấy token (sau khi register + login hoàn chỉnh)
TOKEN="<jwt_cookie_value>"

# Happy path — submit project hợp lệ
curl -X POST http://localhost:8080/api/v1/customer/projects \
  -H "Content-Type: application/json" \
  -H "Cookie: tba_access_token=$TOKEN" \
  -d '{
    "name": "Dự án thương mại điện tử",
    "productType": "WEB",
    "description": "Xây dựng platform thương mại điện tử cho khách hàng B2B với tích hợp thanh toán và quản lý kho.",
    "reference": "https://example.com/reference"
  }' -v
# Expected: 201 + SubmitProjectResponse body với 3 milestones có milestoneId

# Validation error — description quá ngắn
curl -X POST http://localhost:8080/api/v1/customer/projects \
  -H "Content-Type: application/json" \
  -H "Cookie: tba_access_token=$TOKEN" \
  -d '{"name": "Test", "productType": "WEB", "description": "Quá ngắn"}' -v
# Expected: 400 + {"error": "VALIDATION_FAILED", "details": {"description": "..."}}

# Validation error — productType sai
curl -X POST http://localhost:8080/api/v1/customer/projects \
  -H "Content-Type: application/json" \
  -H "Cookie: tba_access_token=$TOKEN" \
  -d '{"name": "Test", "productType": "INVALID", "description": "..."}' -v
# Expected: 400 + VALIDATION_FAILED

# Unauthorized — không có cookie
curl -X POST http://localhost:8080/api/v1/customer/projects \
  -H "Content-Type: application/json" \
  -d '{"name": "Test"}' -v
# Expected: 401
```

### Import statements cần thiết (Task 2)

**JooqProjectRepository:**
```java
import static com.tba.agentic.infrastructure.jooq.Tables.MILESTONES;
import static com.tba.agentic.infrastructure.jooq.Tables.PROJECT_STATUS_HISTORY;
import static com.tba.agentic.infrastructure.jooq.Tables.PROJECTS;
import com.tba.agentic.domain.entity.project.Project;
import com.tba.agentic.domain.value.project.*;
import com.tba.agentic.domain.value.user.UserId;
import com.tba.agentic.infrastructure.jooq.tables.records.MilestonesRecord;
import com.tba.agentic.infrastructure.jooq.tables.records.ProjectsRecord;
import com.tba.agentic.port.out.ProjectRepository;
import lombok.RequiredArgsConstructor;
import org.jooq.DSLContext;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;
```

**CustomerProjectController:**
```java
import com.tba.agentic.adapter.transfer.request.SubmitProjectRequest;
import com.tba.agentic.adapter.transfer.response.SubmitProjectResponse;
import com.tba.agentic.domain.entity.project.Project;
import com.tba.agentic.domain.value.project.Milestone;
import com.tba.agentic.domain.value.project.ProductType;
import com.tba.agentic.domain.value.user.UserId;
import com.tba.agentic.port.bound.project.SubmitProjectCommand;
import com.tba.agentic.port.in.SubmitProjectUseCase;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.web.bind.annotation.*;
import java.util.Comparator;
```

### References

- [Source: epics.md — Story 3.1] — ACs gốc, FR-4, FR-5, FR-14
- [Source: Project.java] — `Project.create()`, `createDefaultMilestones()`, private constructor
- [Source: Milestone.java] — constructor(milestoneId, projectId, name, status, position)
- [Source: JwtAuthenticationFilter.java] — principal = Long userId
- [Source: PgDatabaseUserRepository.java] — pattern jOOQ insert + toUser() mapping
- [Source: SubmitProjectResponse.java] — response DTO đã có, không thay đổi
- [Source: SecurityConfig.java] — `/api/v1/customer/**` → hasRole("CUSTOMER"), không cần thay đổi
- [Source: GlobalExceptionHandler.java] — đã handle MethodArgumentNotValidException → 400 VALIDATION_FAILED

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

### File List
