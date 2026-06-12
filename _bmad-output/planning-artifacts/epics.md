---
stepsCompleted: [1, 2, 3, 4]
inputDocuments:
  - _bmad-output/planning-artifacts/prds/prd-AI-driven-2026-06-11/prd.md
  - _bmad-output/planning-artifacts/architecture/architecture.md
  - _bmad-output/planning-artifacts/ux-designs/ux-AI-driven-2026-06-11/DESIGN.md
  - _bmad-output/planning-artifacts/ux-designs/ux-AI-driven-2026-06-11/EXPERIENCE.md
---

# TBA-agentic — Epic Breakdown

## Overview

Tài liệu này phân rã toàn bộ requirements của TBA-agentic thành các epics và stories có thể implement được, dựa trên PRD, Architecture, và UX Design specification.

## Requirements Inventory

### Functional Requirements

FR-1: Admin có thể tạo tài khoản Customer với email và mật khẩu tạm thời từ Admin Dashboard. Tài khoản tạo với role `customer`; không thể truy cập `/admin/*`.

FR-2: Customer có thể đăng nhập bằng email và mật khẩu. Thành công → redirect Customer Dashboard. Sai credentials → thông báo lỗi. 5 lần sai liên tiếp → khoá tạm thời 15 phút.

FR-3: Hệ thống ngăn Customer truy cập `/admin/*` và ngăn Admin truy cập Customer Dashboard. Mỗi Customer chỉ thấy Projects của chính mình trong mọi API response (tenant isolation).

FR-4: Customer có thể tạo Project mới bằng cách điền form và submit. Project được tạo với Status `SUBMITTED` và Milestones từ Milestone Template mặc định (3 milestones). Customer redirect về Dashboard; Project card xuất hiện ngay.

FR-5: Hệ thống validate dữ liệu trước khi tạo Project: tên (bắt buộc, max 100 ký tự), loại sản phẩm (bắt buộc, web/mobile), mô tả (bắt buộc, min 50, max 2000 ký tự), tham khảo (tuỳ chọn, max 500 ký tự). Submit button disabled cho đến khi hợp lệ. Lỗi hiển thị inline.

FR-6: Customer có thể xem tất cả Projects của mình dưới dạng card grid. Mỗi card: tên Project, Status badge, milestone đang active, timestamp cập nhật gần nhất. Sort mặc định: mới nhất trước. Empty state với CTA khi chưa có project. Skeleton loader khi fetch.

FR-7: Customer có thể xem chi tiết Project: Status badge ở header, Milestone list (done/active/pending với visual phân biệt), Revision Counter (hiển thị khi AWAITING_REVIEW hoặc IN_REVISION; ẩn ở các state khác; chuyển màu destructive khi ≤ 1).

FR-8: Customer có thể xem Activity Feed — danh sách Progress Notes theo thứ tự thời gian ngược. Mỗi entry: timestamp + nội dung. Ẩn section nếu chưa có note nào.

FR-9: Admin có thể xem tất cả Projects của tất cả Customers trong một table. Columns: customer name, project name, status badge, milestone active, timestamp, action link. Sortable theo Status và updated_at. Filter theo Status (multi-select). Skeleton loader khi fetch.

FR-10: Admin có thể thay đổi Status Project theo State Machine. Dropdown chỉ hiển thị transitions hợp lệ. Chuyển sang IN_REVISION → Revision Counter giảm 1. Khi Counter = 0 → cảnh báo Admin, cho phép override với xác nhận.

FR-11: Admin có thể thêm Progress Note khi cập nhật Status. Note là optional. Nội dung xuất hiện trong Activity Feed của Customer. Lưu kèm timestamp.

FR-12: Admin có thể xem và cập nhật Milestones của Project. Admin đánh dấu milestone done/active/pending. Admin có thể chỉnh sửa tên milestone. Milestone change không trigger email.

FR-13: Admin có thể tạo tài khoản Customer mới từ Admin Dashboard (form: email, tên hiển thị, mật khẩu tạm). (Xem FR-1 — đặt lại đây để rõ Admin workflow context.)

FR-14: Khi Customer submit Project thành công, hệ thống gửi email thông báo cho Admin. Nội dung: tên Project, tên Customer, loại sản phẩm, link Admin Project Management.

FR-15: Khi Admin lưu cập nhật Status, hệ thống gửi email thông báo cho Customer. Nội dung: tên Project, Status mới, Progress Note (nếu có), link Project Detail. Chỉ gửi nếu save thành công.

### NonFunctional Requirements

NFR-1: Security — RBAC nghiêm ngặt: Customer không thể truy cập `/admin/*`; Admin không truy cập Customer Dashboard. Mỗi Customer chỉ thấy data của mình (tenant isolation ở API layer).

NFR-2: Email reliability — sử dụng transactional email service (Resend). Timeout 5 giây; log lỗi và async retry nếu service không phản hồi. Email failure không block UI, không rollback state machine.

NFR-3: Performance — Standard web response times đủ. Không có real-time constraint ở v1.

NFR-4: Accessibility — WCAG 2.2 AA trên tất cả surfaces. Status badge luôn icon + màu + text label. Form errors dùng `aria-describedby`. Skeleton loaders dùng `aria-busy="true"`. Tab order match reading order.

NFR-5: Account security — Lockout 15 phút sau 5 failed login attempts (per email + IP).

NFR-6: Deployment — Vercel (Next.js) + Railway (Spring Boot + PostgreSQL). HTTPS bắt buộc từ ngày 1.

### Additional Requirements (từ Architecture)

- AR-1: Starter template `template/` (trong be repo) là reference cho auth stack (JwtTokenProvider, JwtAuthenticationFilter, LoginAttemptService, TokenManagementService, GlobalExceptionHandler) — Epic 1 Story Backend Init builds from this pattern.
- AR-2: Hexagonal Architecture (core/application layers) — jOOQ `DSLContext` và `Tables.*` chỉ tồn tại trong `application/infrastructure/repository`. Domain layer không biết jOOQ.
- AR-3: Flyway migrations (`V{n}__desc.sql`). jOOQ codegen chạy sau migrate. `make dev-setup` wrap toàn bộ: `docker compose up postgres` → `flywayMigrate` → `generateJooq` → `bootRun`.
- AR-4: Docker Compose — PostgreSQL 16-alpine local. `.env.local` gitignored (tại root của be repo).
- AR-5: JWT auth — access token 15 phút + optional refresh token 7 ngày (stored in `refresh_tokens` table). Next.js lưu JWT trong httpOnly cookie `tba_access_token`; Server Components forward `Authorization: Bearer` header đến Spring Boot.
- AR-6: API versioning — auth endpoints `/api/auth/*` không version; resource endpoints `/api/v1/customer/*` và `/api/v1/admin/*`.
- AR-7: Project Status State Machine enforcement — Domain layer owns logic; DB CHECK constraint làm guard thứ 2.
- AR-8: Audit trail — `project_status_history` table append-only. Doubles as source data cho Activity Feed (FR-8).
- AR-9: Optimistic locking — `version` column trên bảng `projects`.
- AR-10: Error response format — `{error, message, status, path, timestamp, trackingId, details}`. `GlobalExceptionHandler` từ template.
- AR-11: Next.js Server Components fetch server-to-server + Server Actions cho mutations. Browser không bao giờ gọi Spring Boot trực tiếp.
- AR-12: Email provider — Resend (fire-and-forget, log + async retry).

### UX Design Requirements

UX-DR1: Design token override — Teal Primary `#0D9488` thay thế shadcn default `primary`. Áp dụng cho: primary buttons, active nav items, IN_DEVELOPMENT status badge.

UX-DR2: Status semantic color system — 8 CSS token pairs (background + text), một cặp cho mỗi trạng thái Project. Tokens này chỉ dùng cho `status-badge`, không dùng decorative.

UX-DR3: Custom component `project-card` — layout: project name (semibold) + status badge cùng row, milestone subtitle bên dưới, timestamp + "View details" dưới cùng. Hover: `border-teal-300 shadow-sm`. Toàn bộ card surface là click target.

UX-DR4: Custom component `status-badge` — luôn render 3 elements: filled circle icon (4px, màu = badge text), state label, background pill. `rounded-full`. Không bao giờ render màu đơn thuần.

UX-DR5: Custom component `revision-counter` — inline chip, chỉ hiển thị trên Project Detail khi IN_REVISION hoặc AWAITING_REVIEW. Format: "Còn X lần chỉnh sửa". Chuyển `text-destructive` khi X ≤ 1; text cũng đổi thành "Còn 1 lần chỉnh sửa cuối" (không chỉ đổi màu).

UX-DR6: Loading states — shadcn `Skeleton` trên tất cả data surfaces với `aria-busy="true"`. Customer Dashboard: 3 card-shaped skeletons. Project Detail: skeleton rows. Admin Dashboard: 5-6 table row skeletons.

UX-DR7: Empty states — Customer Dashboard: heading "Bạn chưa có dự án nào" + subtext + CTA "Tạo dự án đầu tiên". Admin Dashboard: "Chưa có dự án nào được gửi." không có CTA.

UX-DR8: Error states — Form submit fail: Toast destructive giữ form data. Admin save fail: Toast destructive giữ form state. Unauthorized `/admin`: 403 page với link back. Project not found: 404 page với link back.

UX-DR9: Responsive layout — Customer Portal: 1-column `sm`, 2-column `md+` (`max-w-4xl`). Admin Dashboard: full table `lg+` (`max-w-6xl`), horizontal scroll `md`, card list `sm`. Padding: `px-4` sm / `px-6` md / `px-8` lg+.

UX-DR10: Accessibility — tất cả form inputs có visible labels (không placeholder-only). Error states dùng `aria-describedby`. Escape đóng tất cả dialogs/sheets. Tab order match reading order. Keyboard shortcuts chỉ theo shadcn/Radix defaults.

UX-DR11: Vietnamese microcopy — toàn bộ UI text tiếng Việt. Status labels có thể giữ tiếng Anh. Button loading states: "Đang gửi…" / "Đang lưu…". Toast success: "Dự án đã được gửi." / "Đã cập nhật và thông báo cho khách hàng."

UX-DR12: Admin Update Form — Status dropdown (chỉ hiển thị valid transitions từ current state) + Progress note textarea (optional). Save button → "Đang lưu…" → Toast success với auto-dismiss 4 giây.

UX-DR13: Submit form — single-page (không multi-step). Submit button disabled cho đến khi tất cả required fields hợp lệ. Submit on Enter khi focus ở last field.

### FR Coverage Map

| FR | Epic | Story | Mô tả |
|---|---|---|---|
| FR-1 (đã thay) | Epic 2 | 2.3 | Self-registration thay admin-tạo-account |
| FR-2 | Epic 2 | 2.1, 2.4 | Login + rate limiting + RBAC |
| FR-3 | Epic 2 | 2.5 | Route protection, tenant isolation |
| FR-4 | Epic 3 | 3.1 | Submit project |
| FR-5 | Epic 3 | 3.1 | Form validation |
| FR-6 | Epic 3 | 3.2 | Customer dashboard cards |
| FR-7 | Epic 3 | 3.3 | Project detail — status + milestones |
| FR-8 | Epic 3 | 3.3 | Activity feed |
| FR-9 | Epic 4 | 4.1 | Admin project table |
| FR-10 | Epic 4 | 4.2 | Status update + state machine |
| FR-11 | Epic 4 | 4.2 | Progress note |
| FR-12 | Epic 4 | 4.3 | Milestone management |
| FR-13 (đã thay) | Epic 2 | 2.3 | Self-registration (thay admin context) |
| FR-14 | Epic 3 | 3.1 | Admin notification — wired vào submit project |
| FR-15 | Epic 4 | 4.2 | Customer notification — wired vào status update |

**Quyết định đã chốt qua Party Mode:**
- Self-registration thay FR-1/FR-13 (admin tạo account) — AD-12 đảo ngược
- Milestone ↔ Project Status: **Option A** (độc lập) + constraint 1 milestone active/thời điểm
- ResendEmailAdapter tách thành Story 2.2 riêng
- `position INTEGER` column trong bảng `milestones`

## Epic List

### Epic 1: Foundation
Mọi dependency kỹ thuật sẵn sàng trước khi implement feature đầu tiên. Dependency graph: 1.1 → 1.2 → 1.3 (strict sequence); 1.4 và 1.5 chạy song song sau 1.1.
**FRs covered:** AR-1 đến AR-12 (technical prerequisites)

### Epic 2: Authentication & Self-Registration
Người dùng có thể tự đăng ký, xác thực email, và đăng nhập vào đúng portal của mình.
**FRs covered:** FR-2, FR-3, FR-1/FR-13 (self-reg)

### Epic 3: Customer Portal
Customer có thể submit dự án và theo dõi toàn bộ tiến độ — từ dashboard cards đến project detail với milestones, activity feed, revision counter.
**FRs covered:** FR-4, FR-5, FR-6, FR-7, FR-8, FR-14

### Epic 4: Admin Dashboard
Admin có thể quản lý toàn bộ vòng đời dự án — nhìn tổng quan, cập nhật status theo state machine, thêm progress notes, và quản lý milestones.
**FRs covered:** FR-9, FR-10, FR-11, FR-12, FR-15

---

**Tổng: 4 epics, 15 stories, 15 FRs covered**

```
Epic 1: Foundation        — 5 stories (1.1–1.5)
Epic 2: Auth & Register   — 5 stories (2.1–2.5)
Epic 3: Customer Portal   — 3 stories (3.1–3.3)
Epic 4: Admin Dashboard   — 3 stories (4.1–4.3)
```

---

## Epic 1: Foundation

Mọi dependency kỹ thuật sẵn sàng trước khi implement feature đầu tiên. Dependency order: 1.1 → 1.2 → 1.3 (strict); 1.4 và 1.5 chạy song song sau 1.1.

### Story 1.1: Backend Project Scaffold

As a developer,
I want the backend repository scaffolded with correct package structure, Docker setup, and base configuration,
So that all subsequent backend stories have a stable, convention-aligned foundation to build on.

**Acceptance Criteria:**

**Given** the repository is cloned and Docker is running,
**When** `docker compose up -d postgres` executes,
**Then** PostgreSQL 16-alpine starts on port 5432 with database `tba_agentic`, user `postgres`.

**Given** Docker PostgreSQL is running,
**When** `make dev-setup` executes,
**Then** it runs `flywayMigrate` → `generateJooq` → `bootRun` in sequence without error.

**Given** the app is running,
**When** `GET /actuator/health` is called,
**Then** returns `{"status":"UP"}` with HTTP 200.

**Given** a request with no Authorization header,
**When** any `/api/*` endpoint is hit,
**Then** returns HTTP 401 (SecurityConfig is active with STATELESS session).

**Given** any unhandled exception occurs,
**When** a request is processed,
**Then** `GlobalExceptionHandler` returns `{error, message, status, path, timestamp, trackingId}` — no raw stack traces exposed.

**And** `TrackingIdFilter` injects `X-Tracking-Id` header on every response.

**And** package structure follows `com.tba.agentic` with subdirectories: `adapter/controller/`, `adapter/filter/`, `adapter/transfer/request/`, `adapter/transfer/response/`, `application/service/`, `config/`, `infrastructure/repository/`, `security/`.

**And** `.env.local` (gitignored) exists with template vars: `TBA_DATASOURCE_URL`, `TBA_DATASOURCE_USERNAME`, `TBA_DATASOURCE_PASSWORD`, `TBA_APP_API_SECURITY_JWT_SECRET`, `TBA_APP_API_FRONTEND_BASE_URL`.

**And** `docker-compose.yml` at project root includes healthcheck on postgres and a named volume `postgres_data`.

---

### Story 1.2: Database Schema V1 Migration

As a developer,
I want the complete database schema created via a single Flyway migration,
So that jOOQ can generate type-safe classes and all backend stories share a consistent data model.

**Acceptance Criteria:**

**Given** Docker PostgreSQL is running,
**When** `./gradlew flywayMigrate` executes,
**Then** migration `V1__initial_schema.sql` completes with no errors and `flyway_schema_history` marks it SUCCESS.

**Given** the migration runs successfully,
**When** the schema is inspected,
**Then** the following tables exist with correct columns:

- `users`: `user_id` BIGINT PK, `email` VARCHAR(255) UNIQUE NOT NULL, `password_hash` VARCHAR(255), `role` ENUM(CUSTOMER/ADMIN) NOT NULL, `is_active` BOOLEAN DEFAULT false, `is_email_verified` BOOLEAN DEFAULT false, `display_name` VARCHAR(120), `version` INTEGER DEFAULT 0, `created_at` TIMESTAMPTZ, `updated_at` TIMESTAMPTZ.
- `projects`: `project_id` UUID PK DEFAULT gen_random_uuid(), `customer_id` BIGINT FK→users, `name` VARCHAR(100) NOT NULL, `product_type` ENUM(WEB/MOBILE), `description` TEXT, `reference` TEXT, `status` VARCHAR(30) NOT NULL, `revision_count` INTEGER DEFAULT 3, `version` INTEGER DEFAULT 0, `created_at` TIMESTAMPTZ, `updated_at` TIMESTAMPTZ.
- `milestones`: `milestone_id` BIGSERIAL PK, `project_id` UUID FK→projects, `name` VARCHAR(200) NOT NULL, `status` ENUM(PENDING/ACTIVE/DONE) DEFAULT PENDING, `position` INTEGER NOT NULL, `created_at` TIMESTAMPTZ, `updated_at` TIMESTAMPTZ.
- `project_status_history`: `id` UUID PK DEFAULT gen_random_uuid(), `project_id` UUID FK→projects ON DELETE CASCADE, `previous_status` VARCHAR(30), `new_status` VARCHAR(30) NOT NULL, `changed_by` VARCHAR(255) NOT NULL, `reason` TEXT, `changed_at` TIMESTAMPTZ DEFAULT now(), `ip_address` INET.
- `refresh_tokens`: `refresh_token_id` BIGINT PK, `user_id` BIGINT FK→users, `token` VARCHAR(500) UNIQUE NOT NULL, `expires_at` TIMESTAMPTZ, `revoked` BOOLEAN DEFAULT false, `created_at` TIMESTAMPTZ.
- `email_verifications`: `verification_id` BIGSERIAL PK, `user_id` BIGINT FK→users, `token` VARCHAR(255) NOT NULL, `expires_at` TIMESTAMPTZ, `created_at` TIMESTAMPTZ.
- `login_attempts`: `attempt_id` BIGSERIAL PK, `email` VARCHAR(255), `ip_address` INET, `user_agent` TEXT, `attempt_time` TIMESTAMPTZ DEFAULT now(), `success` BOOLEAN DEFAULT false, `failure_reason` VARCHAR(100), `user_id` BIGINT.

**And** `projects` has CHECK constraint: `status IN ('SUBMITTED','ANALYZING','IN_DEVELOPMENT','AWAITING_REVIEW','IN_REVISION','FINALIZING','DELIVERED','CANCELLED')`.

**And** `milestones` has UNIQUE constraint on `(project_id, position)`.

**And** 1 admin user seeded: `email=admin@tba.com` (or configurable), `role=ADMIN`, `is_active=true`, `is_email_verified=true`, password BCrypt-hashed from env var `TBA_ADMIN_SEED_PASSWORD`.

**And** indexes created: `idx_projects_customer_id`, `idx_projects_status`, `idx_milestones_project_id`, `idx_psh_project_id`, `idx_psh_changed_at DESC`, `idx_login_attempts_email_time`, `idx_refresh_tokens_user_id`.

---

### Story 1.3: jOOQ Codegen Pipeline

As a developer,
I want jOOQ to generate type-safe Java classes from the V1 schema automatically,
So that the infrastructure repository layer has compile-time-safe database access with no string-based column references.

**Acceptance Criteria:**

**Given** Flyway V1 migration has run and PostgreSQL is running,
**When** `./gradlew generateJooq` executes,
**Then** it completes without errors.

**Given** codegen completes,
**When** the generated sources directory is inspected,
**Then** Java classes exist for all 7 tables: `Users`, `Projects`, `Milestones`, `ProjectStatusHistory`, `RefreshTokens`, `EmailVerifications`, `LoginAttempts` — each with typed column accessors (e.g., `PROJECTS.STATUS`, `USERS.EMAIL`).

**Given** generated classes exist,
**When** `./gradlew :application:build` runs,
**Then** compilation passes with zero errors referencing generated classes.

**And** `Makefile` has a `generate` target that runs `flywayMigrate` then `generateJooq` in sequence.

**And** generated sources directory is gitignored (in `.gitignore`).

**And** jOOQ codegen uses `PostgresDatabase` mode (live DB) — Docker must be running for codegen to work.

---

### Story 1.4: Core Domain Model *(parallel — start after 1.1)*

As a developer,
I want all domain entities, value objects, and port interfaces defined in the `core` module with zero infrastructure dependencies,
So that application services can implement use cases independently of database or framework specifics.

**Acceptance Criteria:**

**Given** the `core` module,
**When** `./gradlew :core:build` runs,
**Then** it compiles with zero dependencies on Spring, jOOQ, Resend, or any infrastructure framework.

**Given** the domain model,
**When** inspected,
**Then** the following aggregates exist:
- `User` aggregate root: `userId` (UserId), `account` (Account), `profile` (Profile). `Account` contains: email (Email), passwordHash (PasswordHash), role (Role), isActive, isEmailVerified. `Profile` contains: displayName (DisplayName).
- `Project` aggregate root: `projectId` (ProjectId), `customerId` (UserId), `name`, `productType` (WEB/MOBILE), `description`, `reference`, `status` (ProjectStatus), `revisionCount` (int), `milestones` (List<Milestone>), `version` (int).
- `Milestone` entity: `milestoneId`, `projectId`, `name`, `status` (MilestoneStatus: PENDING/ACTIVE/DONE), `position` (int).

**And** `ProjectStatus` enum has 8 values with `transitionTo(ProjectStatus next)` method — throws `InvalidStatusTransitionException` for invalid transitions. Valid transitions match AD-06 state machine.

**And** port interfaces exist in `core/port/in/`: `LoginUseCase`, `RegisterUserUseCase`, `VerifyEmailUseCase`, `SubmitProjectUseCase`, `GetCustomerProjectsUseCase`, `GetProjectDetailUseCase`, `UpdateProjectStatusUseCase`, `UpdateMilestoneUseCase`, `GetAllProjectsUseCase`.

**And** port interfaces exist in `core/port/out/`: `UserRepository`, `ProjectRepository`, `MilestoneRepository`, `AuditRepository`, `TokenRepository`, `LoginAttemptRepository`, `EmailVerificationRepository`, `EmailClient`.

**And** value objects are immutable records/classes: `Email`, `Password`, `PasswordHash`, `Role`, `UserId`, `ProjectId`, `DisplayName`, `IpAddress`, `UserAgent`.

**And** exceptions defined: `InvalidStatusTransitionException`, `AuthenticationException` (with `Type` enum: ACCOUNT_NOT_FOUND, INVALID_CREDENTIALS, TOO_MANY_REQUESTS, EMAIL_NOT_VERIFIED, TOKEN_EXPIRED), `ProjectNotFoundException`, `EmailConflictException`.

**And** `ProjectStatus.transitionTo()` unit test covers all valid transitions and all invalid transitions (throws exception).

---

### Story 1.5: Frontend Project Init *(parallel — independent)*

As a developer,
I want the Next.js frontend initialized with the design system, app shell, and API helper,
So that all frontend stories have consistent styling, layout, and server-side fetch patterns to build on.

**Acceptance Criteria:**

**Given** the repository root,
**When** `npm run dev` starts,
**Then** `http://localhost:3000` returns HTTP 200.

**Given** the design system setup,
**When** the app loads,
**Then** Tailwind CSS `globals.css` overrides shadcn `--primary` with `#0D9488` (teal) and defines 8 status semantic CSS custom properties: `--status-submitted-bg`, `--status-submitted-text`, ... (one pair per state per DESIGN.md).

**And** shadcn/ui is installed; `components/ui/` contains at minimum: `Button`, `Card`, `Input`, `Textarea`, `Select`, `Badge`, `Skeleton`, `Toast`.

**And** TypeScript configured: `strict: true`, `noUncheckedIndexedAccess: true`, `exactOptionalPropertyTypes: true`.

**And** `lib/api.ts` (marked `"use server"` + `import "server-only"`) exports `apiFetch(path, options)` that reads `tba_access_token` cookie from `next/headers` and forwards `Authorization: Bearer <token>` to `process.env.SPRING_BOOT_URL`.

**And** `app/layout.tsx` renders a top navigation shell with logo placeholder, nav link placeholders, and user avatar placeholder.

**And** `app/page.tsx` renders Customer Dashboard placeholder (authenticated route).

**And** `app/admin/page.tsx` renders Admin Dashboard placeholder (authenticated admin route).

**And** `app/login/page.tsx` renders Login page placeholder (public route).

**And** `middleware.ts` checks `tba_access_token` cookie: missing cookie on protected route (`/`, `/projects/**`, `/admin/**`) → redirect `/login`.

**And** `.env.local` (gitignored) template: `SPRING_BOOT_URL=http://localhost:8080`.

**And** `next.config.ts` sets `output: 'standalone'` for Railway deployment.

**And** `Makefile` tại root của FE repo định nghĩa: `test` target (`npm test`), `dev` target (`npm run dev`), `build` target (`npm run build`), `compile` target (`npx tsc --noEmit`). Agent chạy test đồng nhất qua `make test` (AD-19).

---

## Epic 2: Authentication & Self-Registration

Người dùng có thể tự đăng ký, xác thực email, và đăng nhập vào đúng portal của mình.
**FRs covered:** FR-2, FR-3, FR-1/FR-13 (self-registration)

---

### Story 2.1: JWT Auth Backend

As a developer,
I want the JWT authentication stack wired from the `template/` starter pattern — login endpoint, token filter, refresh, logout, and rate limiting,
So that all API endpoints are protected and users can authenticate securely.

**Acceptance Criteria:**

**Given** a `POST /api/auth/login` request với email + password hợp lệ của user `is_active=true`, `is_email_verified=true`,
**When** credentials match,
**Then** trả về HTTP 200 với body `{ token: "<jwt>", expiresIn: 900, refreshToken: "<token>" }` (refreshToken chỉ có khi `rememberMe: true`).

**Given** `POST /api/auth/login` với credentials sai,
**When** gửi request,
**Then** trả về HTTP 401 với error body `{ error: "AUTHENTICATION_FAILED", message: "...", status: 401, ... }` — không tiết lộ email có tồn tại hay không.

**Given** cùng một email gửi 5 lần sai liên tiếp trong 15 phút,
**When** lần thứ 5 fail,
**Then** lần thứ 6 trả về HTTP 429 với error `"TOO_MANY_REQUESTS"`.
**And** bảng `login_attempts` có đủ records ghi lại mỗi attempt (email, ip_address, attempt_time, success=false).
**And** sau 15 phút tính từ attempt đầu tiên, account unlock tự động.

**Given** request đến bất kỳ `/api/v1/**` endpoint với header `Authorization: Bearer <valid_jwt>`,
**When** token còn hạn và chưa bị revoke,
**Then** `JwtAuthenticationFilter` xác thực thành công, `SecurityContextHolder` có authentication với đúng role.

**Given** request với token hết hạn hoặc sai format,
**When** gửi đến protected endpoint,
**Then** trả về HTTP 401 với error `"TOKEN_EXPIRED"` hoặc `"INVALID_TOKEN"`.

**Given** `POST /api/auth/refresh` với refreshToken hợp lệ chưa hết hạn và chưa bị revoke,
**When** gửi request,
**Then** trả về HTTP 200 với access token mới. Old refreshToken bị revoke, refreshToken mới được cấp.

**Given** `POST /api/auth/logout` với refreshToken hợp lệ,
**When** gửi request,
**Then** trả về HTTP 204, bảng `refresh_tokens` mark `revoked=true` cho token đó.

**And** `SecurityConfig` dùng `SessionCreationPolicy.STATELESS`, `JwtAuthenticationFilter` được add vào filterChain.
**And** `JwtProperties` đọc từ env: `TBA_APP_API_SECURITY_JWT_SECRET`, access token 15 phút, refresh token 7 ngày.

---

### Story 2.2: Email Infrastructure

As a developer,
I want a `ResendEmailAdapter` implementing the `EmailClient` port with fire-and-forget semantics and a dev-mode fallback,
So that email notifications can be sent without blocking main flows, and local dev works without a real API key.

**Acceptance Criteria:**

**Given** `RESEND_API_KEY` được set trong environment,
**When** `EmailClient.send(EmailMessage)` được gọi,
**Then** `ResendEmailAdapter` gọi Resend HTTP API với đúng `from`, `to`, `subject`, `html`.
**And** nếu Resend trả về lỗi hoặc timeout (>5s), exception bị catch và log ở `WARN` level — không propagate lên caller.

**Given** `RESEND_API_KEY` không được set (dev mode),
**When** `EmailClient.send(EmailMessage)` được gọi,
**Then** nội dung email được log ở `INFO` level với prefix `[EMAIL-DEV]` — không throw exception.

**Given** email send bất đồng bộ (async),
**When** main request flow hoàn thành,
**Then** email failure không rollback transaction, không trả lỗi cho client.

**And** `EmailClient` interface nằm ở `core/port/out/`, `ResendEmailAdapter` nằm ở `infrastructure/email/`.
**And** `EmailMessage` record: `to`, `subject`, `htmlBody`, `textBody`.

---

### Story 2.3: Self-Registration + Email Verification

As a new customer,
I want to register for an account with my email and verify it before logging in,
So that I can access the customer portal with a confirmed identity.

**Acceptance Criteria:**

**Given** `POST /api/auth/register` với `email`, `displayName`, `password` (min 8 ký tự) hợp lệ,
**When** email chưa tồn tại trong hệ thống,
**Then** trả về HTTP 201. User được tạo với `role=CUSTOMER`, `is_active=false`, `is_email_verified=false`.
**And** record `email_verifications` được tạo với `token` UUID và `expires_at = now() + 24 giờ`.
**And** Email verification được gửi tới địa chỉ đã đăng ký (qua `EmailClient`).

**Given** `POST /api/auth/register` với email đã tồn tại,
**When** gửi request,
**Then** trả về HTTP 409 với error `"EMAIL_CONFLICT"`.

**Given** `GET /api/auth/verify-email?token=<valid_token>` (chưa expired, chưa dùng),
**When** gọi endpoint,
**Then** trả về HTTP 200. User được update: `is_email_verified=true`, `is_active=true`. Token record bị xóa.

**Given** `GET /api/auth/verify-email?token=<expired_token>` (quá 24 giờ),
**When** gọi endpoint,
**Then** trả về HTTP 400 với error `"VERIFICATION_EXPIRED"`.

**Given** `POST /api/auth/resend-verification` với email đã register nhưng chưa verify,
**When** đã gửi < 3 lần trong 24 giờ gần nhất,
**Then** trả về HTTP 200, gửi lại email verification mới (token cũ invalid).

**Given** `POST /api/auth/resend-verification` khi đã resend ≥ 3 lần trong 24 giờ,
**When** gửi request,
**Then** trả về HTTP 429 với error `"RESEND_LIMIT_EXCEEDED"`.

**Given** `POST /api/auth/login` với user `is_email_verified=false`,
**When** gửi credentials đúng,
**Then** trả về HTTP 401 với error `"EMAIL_NOT_VERIFIED"`.

**And** endpoint `/api/auth/register` và `/api/auth/verify-email` được configure `permitAll()` trong `SecurityConfig`.

---

### Story 2.4: Auth UI

As a user,
I want to register, log in, and log out via dedicated pages with Vietnamese microcopy,
So that I can access the correct portal (customer or admin) after authentication.

**Acceptance Criteria:**

**Given** `/login` page,
**When** load trang,
**Then** render form với: email input (label "Email"), password input (label "Mật khẩu"), button "Đăng nhập". Trang dùng React Hook Form + Zod validation.

**Given** submit login với credentials đúng của Customer,
**When** Server Action `loginAction` thành công,
**Then** `tba_access_token` cookie được set (httpOnly, secure, sameSite=lax). Redirect về `/` (Customer Dashboard).
**And** nếu role là ADMIN, redirect về `/admin` thay thế.

**Given** submit login với credentials sai,
**When** Server Action trả về lỗi,
**Then** toast destructive hiện "Email hoặc mật khẩu không đúng." Form data giữ nguyên (không clear).

**Given** submit login với account bị locked (429),
**When** Server Action trả về lỗi,
**Then** toast destructive hiện "Tài khoản bị khoá tạm thời. Vui lòng thử lại sau 15 phút."

**Given** `/register` page,
**When** load trang,
**Then** render form: email, tên hiển thị, mật khẩu (min 8 ký tự), xác nhận mật khẩu. Submit button "Đăng ký" disabled cho đến khi form hợp lệ.

**Given** submit register thành công,
**When** Server Action `registerAction` trả về 201,
**Then** redirect sang `/register/verify-email` với thông báo "Vui lòng kiểm tra email để xác thực tài khoản." và nút "Gửi lại email".

**Given** email đã tồn tại khi register,
**When** Server Action trả về 409,
**Then** hiển thị inline error tại email field: "Email này đã được sử dụng."

**Given** user click "Gửi lại email" trên `/register/verify-email`,
**When** gọi Server Action `resendVerificationAction`,
**Then** button chuyển sang "Đang gửi…" trong khi processing. Thành công → toast "Đã gửi lại email xác thực."

**And** không có route `/admin/login` riêng — admin dùng chung `/login`.
**And** Logout: Server Action `logoutAction` clear cookie `tba_access_token`, redirect về `/login`.

---

### Story 2.5: RBAC + Route Protection

As the system,
I want strict role enforcement on all protected routes and tenant isolation on all customer data queries,
So that customers and admins can only access their respective areas and data.

**Acceptance Criteria:**

**Given** Customer đã đăng nhập (role=CUSTOMER) gọi bất kỳ `GET/POST /api/v1/admin/**`,
**When** request đến Spring Boot,
**Then** trả về HTTP 403 với error `"ACCESS_DENIED"`.

**Given** Admin đã đăng nhập (role=ADMIN) gọi bất kỳ `GET/POST /api/v1/customer/**`,
**When** request đến Spring Boot,
**Then** trả về HTTP 403 với error `"ACCESS_DENIED"`.

**Given** Customer A đã đăng nhập,
**When** gọi `GET /api/v1/customer/projects` hoặc `GET /api/v1/customer/projects/{id}`,
**Then** response chỉ chứa projects của Customer A (filter `customer_id = authenticated user id` ở repository layer).
**And** project thuộc Customer B không bao giờ xuất hiện trong response, kể cả khi Customer A đoán đúng `projectId`.

**Given** request tới `/admin/**` trong Next.js (browser) với cookie chứa JWT role=CUSTOMER,
**When** Next.js `middleware.ts` kiểm tra,
**Then** redirect sang `/403` page với message "Bạn không có quyền truy cập trang này." và link quay về `/`.

**Given** request tới bất kỳ protected route (`/`, `/projects/**`, `/admin/**`) trong Next.js mà không có cookie `tba_access_token`,
**When** `middleware.ts` kiểm tra,
**Then** redirect sang `/login`.

**Given** Server Action hoặc Server Component nhận HTTP 401 từ Spring Boot,
**When** xử lý response,
**Then** cookie `tba_access_token` bị clear, user bị redirect về `/login` (auto-logout).

**And** `SecurityConfig` rules (ưu tiên từ trên xuống):
- `/api/auth/**` → `permitAll()`
- `/api/v1/admin/**` → `hasRole("ADMIN")`
- `/api/v1/customer/**` → `hasRole("CUSTOMER")`
- Mọi request khác → `authenticated()`

---

## Epic 3: Customer Portal

Customer có thể submit dự án và theo dõi toàn bộ tiến độ — từ dashboard cards đến project detail với milestones, activity feed, revision counter.
**FRs covered:** FR-4, FR-5, FR-6, FR-7, FR-8, FR-14

---

### Story 3.1: Submit Project

As a customer,
I want to submit a new project by filling out a validated form,
So that the agency receives my project requirements and I am redirected to my dashboard immediately after.

**Acceptance Criteria:**

**Given** `POST /api/v1/customer/projects` với payload hợp lệ: `name` (max 100 ký tự), `productType` (WEB/MOBILE), `description` (min 50, max 2000 ký tự), `reference` (tuỳ chọn, max 500 ký tự),
**When** Customer đã đăng nhập gửi request,
**Then** trả về HTTP 201 với project vừa tạo: `status=SUBMITTED`, 3 milestones được tạo tự động: `{ name: "UI/UX & Prototype", position: 1, status: ACTIVE }`, `{ name: "Backend Development", position: 2, status: PENDING }`, `{ name: "Frontend Integration & Deploy", position: 3, status: PENDING }`.
**And** `project_status_history` có 1 record: `previous_status=null`, `new_status=SUBMITTED`, `changed_by=<customer_email>`.

**Given** payload thiếu `name` hoặc `description` dưới 50 ký tự hoặc `productType` không hợp lệ,
**When** gửi request,
**Then** trả về HTTP 400 với error `"VALIDATION_FAILED"` và `details` map chỉ rõ field nào sai.

**Given** Admin notification (FR-14): project vừa được tạo thành công,
**When** `POST /api/v1/customer/projects` returns 201,
**Then** `EmailClient.send()` được gọi async với email tới admin (`admin@tba.com`), subject "Dự án mới cần xem xét", body gồm: tên Project, tên Customer (`displayName`), loại sản phẩm, link `/admin/projects/{id}`.
**And** email failure không ảnh hưởng đến HTTP response.

**Given** Frontend form tại `/projects/new`,
**When** load trang,
**Then** render single-page form: Project name, Product type (select: Web/Mobile), Description (textarea), Reference (textarea, optional). Submit button "Gửi dự án" disabled cho đến khi tất cả required fields hợp lệ.

**Given** user submit form,
**When** Server Action `submitProjectAction` đang xử lý,
**Then** button đổi thành "Đang gửi…" và disabled.

**Given** submit thành công,
**When** Server Action trả về 201,
**Then** toast success "Dự án đã được gửi." và redirect về `/` (Customer Dashboard). Project card mới xuất hiện ngay (revalidateTag).

**Given** submit thất bại (lỗi server),
**When** Server Action trả về lỗi,
**Then** toast destructive hiện thông báo lỗi. Form data giữ nguyên, không clear.

**And** submit on Enter khi focus ở last field (textarea reference).
**And** inline validation errors dùng `aria-describedby` (NFR-4 / UX-DR10).

---

### Story 3.2: Customer Dashboard

As a customer,
I want to see all my projects displayed as a card grid with their current status and active milestone,
So that I can get a quick overview of all my projects and navigate to any of them.

**Acceptance Criteria:**

**Given** `GET /api/v1/customer/projects`,
**When** Customer đã đăng nhập gửi request,
**Then** trả về HTTP 200 với array projects của Customer (tenant isolation: chỉ projects có `customer_id = authenticated user id`). Sort mặc định: `updated_at DESC`.

**Given** Customer có ≥ 1 project,
**When** load `/` (Customer Dashboard),
**Then** render card grid: 1 column trên `sm`, 2 columns trên `md+` (`max-w-4xl`). Mỗi card:
- Dòng 1: tên project (semibold) + `status-badge` (component UX-DR4) cùng hàng
- Dòng 2: tên milestone đang `ACTIVE` (subtitle text)
- Dòng 3: timestamp `updated_at` (format relative) + link "Xem chi tiết"
- Hover: `border-teal-300 shadow-sm`, toàn bộ card surface là click target

**Given** loading data,
**When** fetch chưa hoàn thành,
**Then** hiện 3 card-shaped skeletons với `aria-busy="true"` (UX-DR6).

**Given** Customer chưa có project nào,
**When** load Dashboard,
**Then** hiện empty state: heading "Bạn chưa có dự án nào", subtext mô tả, CTA button "Tạo dự án đầu tiên" → navigate `/projects/new` (UX-DR7).

**And** `status-badge` component (UX-DR4) luôn render 3 elements: filled circle icon (4px, màu = badge text color), state label, background pill. `rounded-full`. Không bao giờ chỉ hiện màu đơn thuần.
**And** 8 trạng thái CSS token pairs đã được define ở Story 1.5 được dùng cho `status-badge`.

---

### Story 3.3: Project Detail

As a customer,
I want to view a project's full detail page with status, milestone list, revision counter, and activity feed,
So that I can track the progress of my project at any point in time.

**Acceptance Criteria:**

**Given** `GET /api/v1/customer/projects/{id}`,
**When** Customer đã đăng nhập và project thuộc về họ,
**Then** trả về HTTP 200 với: project fields, `milestones` array (sorted by `position ASC`), `statusHistory` array (sorted by `changed_at DESC`) — doubles as Activity Feed.

**Given** project không thuộc Customer đang đăng nhập,
**When** gọi `GET /api/v1/customer/projects/{id}`,
**Then** trả về HTTP 404 (không tiết lộ project tồn tại hay không — tenant isolation).

**Given** load `/projects/{id}` (Project Detail page),
**When** fetch hoàn thành,
**Then** header hiện: tên project + `status-badge` component với đúng trạng thái.
**And** Milestone list hiện 3 milestones sorted by `position`, với visual phân biệt rõ ràng: DONE (checked icon, muted), ACTIVE (highlighted, teal), PENDING (muted, no icon).

**Given** project có `status = AWAITING_REVIEW` hoặc `status = IN_REVISION`,
**When** render Project Detail,
**Then** hiện `revision-counter` chip (UX-DR5) với text "Còn {n} lần chỉnh sửa".
**And** khi `revisionCount ≤ 1`: chip đổi sang `text-destructive`, text đổi thành "Còn 1 lần chỉnh sửa cuối".

**Given** project có `status` khác AWAITING_REVIEW và IN_REVISION,
**When** render Project Detail,
**Then** `revision-counter` chip không hiển thị.

**Given** `statusHistory` có ≥ 1 entry có `reason` không null,
**When** render Activity Feed section,
**Then** hiện danh sách các entries theo thứ tự thời gian ngược: mỗi entry gồm timestamp + nội dung `reason`.

**Given** chưa có entry nào trong `statusHistory` có `reason`,
**When** render Activity Feed section,
**Then** section Activity Feed bị ẩn hoàn toàn (FR-8).

**Given** loading Project Detail,
**When** fetch chưa hoàn thành,
**Then** hiện skeleton rows với `aria-busy="true"` (UX-DR6).

**Given** project không tồn tại hoặc URL sai,
**When** load `/projects/{id}`,
**Then** hiện 404 page với message và link "Quay lại Dashboard" (UX-DR8).

---

## Epic 4: Admin Dashboard

Admin có thể quản lý toàn bộ vòng đời dự án — nhìn tổng quan, cập nhật status theo state machine, thêm progress notes, và quản lý milestones.
**FRs covered:** FR-9, FR-10, FR-11, FR-12, FR-15

---

### Story 4.1: Admin Project Table

As an admin,
I want to see all projects from all customers in a sortable, filterable table,
So that I can quickly find and prioritize projects that need attention.

**Acceptance Criteria:**

**Given** `GET /api/v1/admin/projects`,
**When** Admin đã đăng nhập gửi request,
**Then** trả về HTTP 200 với array tất cả projects (không giới hạn theo customer). Response mỗi item gồm: `projectId`, `customerName` (displayName của owner), `projectName`, `status`, `activeMilestoneName`, `updatedAt`.

**Given** query param `?status=SUBMITTED,ANALYZING` (multi-value),
**When** gửi request,
**Then** chỉ trả về projects có status nằm trong danh sách filter.

**Given** query param `?sortBy=status&sortDir=asc` hoặc `?sortBy=updatedAt&sortDir=desc`,
**When** gửi request,
**Then** kết quả được sort đúng theo field và hướng yêu cầu.

**Given** load `/admin` (Admin Dashboard),
**When** fetch hoàn thành,
**Then** render table với columns: Customer, Tên dự án, Trạng thái (status-badge), Milestone đang active, Cập nhật lúc, Action ("Quản lý").
**And** Table responsive: full table trên `lg+` (`max-w-6xl`), horizontal scroll trên `md`, card list trên `sm` (UX-DR9).
**And** Header columns "Trạng thái" và "Cập nhật lúc" có thể click để sort (toggle asc/desc).
**And** Multi-select filter dropdown "Lọc theo trạng thái" với 8 status options — áp dụng filter real-time (revalidate on change).

**Given** loading data,
**When** fetch chưa hoàn thành,
**Then** hiện 5–6 table row skeletons với `aria-busy="true"` (UX-DR6).

**Given** chưa có project nào trong hệ thống,
**When** load Admin Dashboard,
**Then** hiện empty state: "Chưa có dự án nào được gửi." — không có CTA (UX-DR7).

---

### Story 4.2: Status Update + Progress Note

As an admin,
I want to update a project's status via a dropdown that only shows valid transitions, add an optional progress note, and have the customer notified automatically,
So that the project lifecycle is enforced correctly and customers are kept informed.

**Acceptance Criteria:**

**Given** `GET /api/v1/admin/projects/{id}`,
**When** Admin đã đăng nhập gửi request,
**Then** trả về HTTP 200 với đầy đủ project data, milestones, statusHistory, và `allowedTransitions` array (danh sách status hợp lệ từ state hiện tại theo AD-06).

**Given** `PATCH /api/v1/admin/projects/{id}/status` với body `{ newStatus: "ANALYZING", note: "Đang phân tích yêu cầu", forceRevision: false }`,
**When** `newStatus` là transition hợp lệ từ current status,
**Then** trả về HTTP 200. Project status được update. `project_status_history` có record mới: `previous_status`, `new_status`, `changed_by=<admin_email>`, `reason=note`, `changed_at=now()`, `ip_address`.

**Given** `newStatus = IN_REVISION` và transition hợp lệ,
**When** request được xử lý,
**Then** `projects.revision_count` giảm 1 (trong cùng transaction với status update).

**Given** `newStatus = IN_REVISION` nhưng `revision_count = 0` và `forceRevision = false`,
**When** request được xử lý,
**Then** trả về HTTP 422 với error `"REVISION_LIMIT_REACHED"` — không thực hiện transition.

**Given** `newStatus = IN_REVISION` nhưng `revision_count = 0` và `forceRevision = true`,
**When** request được xử lý,
**Then** transition được thực hiện. `revision_count` không giảm thêm (giữ nguyên 0).

**Given** `newStatus` không phải transition hợp lệ từ current status,
**When** request được xử lý,
**Then** trả về HTTP 422 với error `"INVALID_STATUS_TRANSITION"`.

**Given** `PATCH /api/v1/admin/projects/{id}/status` bị race condition (version mismatch),
**When** optimistic locking phát hiện conflict,
**Then** trả về HTTP 409 với error `"OPTIMISTIC_LOCK_CONFLICT"`.

**Given** Customer notification (FR-15): status update thành công,
**When** `PATCH` returns 200,
**Then** `EmailClient.send()` được gọi async với email tới customer owner, subject "Cập nhật dự án: {projectName}", body gồm: tên Project, Status mới (label tiếng Việt), Progress Note nếu có, link `/projects/{id}`.
**And** email failure không rollback state change, không trả lỗi cho admin.

**Given** load `/admin/projects/{id}` (Admin Project Detail page),
**When** fetch hoàn thành,
**Then** render Admin Update Form (UX-DR12): Status dropdown chỉ hiện `allowedTransitions` từ current status + Progress Note textarea (optional, placeholder "Ghi chú cho khách hàng…") + Save button "Lưu thay đổi".

**Given** `revision_count = 0` và admin chọn `IN_REVISION` trong dropdown,
**When** render form,
**Then** hiện warning dialog "Khách hàng đã dùng hết số lần chỉnh sửa. Bạn có chắc muốn tiếp tục không?" với nút xác nhận. Confirm → set `forceRevision: true` trong request.

**Given** Save button được click,
**When** Server Action `updateProjectStatusAction` đang xử lý,
**Then** button đổi thành "Đang lưu…" và disabled.

**Given** save thành công,
**When** Server Action trả về 200,
**Then** toast success "Đã cập nhật và thông báo cho khách hàng." auto-dismiss sau 4 giây (UX-DR12). Page data revalidate.

**Given** save thất bại (lỗi server hoặc conflict),
**When** Server Action trả về lỗi,
**Then** toast destructive hiện thông báo lỗi. Form state giữ nguyên (UX-DR8).

---

### Story 4.3: Milestone Management

As an admin,
I want to view and update milestones for a project — marking them done/active/pending and editing names — independently of the project status,
So that I can reflect actual delivery progress without being constrained by the status state machine.

**Acceptance Criteria:**

**Given** `PUT /api/v1/admin/projects/{id}/milestones/{milestoneId}` với body `{ name: "...", status: "DONE" }`,
**When** Admin đã đăng nhập gửi request,
**Then** trả về HTTP 200 với milestone đã được update.

**Given** request update milestone status sang `ACTIVE` trong khi đã có milestone khác có `status=ACTIVE` trên cùng project,
**When** request được xử lý,
**Then** milestone cũ tự động chuyển sang `PENDING` trước khi milestone mới được set `ACTIVE` (constraint: 1 ACTIVE per project tại một thời điểm).

**Given** update milestone name (không đổi status),
**When** request được xử lý,
**Then** chỉ `name` được update, `status` và `position` giữ nguyên.

**Given** milestone không thuộc project `{id}`,
**When** gửi request,
**Then** trả về HTTP 404.

**Given** load `/admin/projects/{id}` (Admin Project Detail),
**When** fetch hoàn thành,
**Then** Milestone section hiện 3 milestones (sorted by `position`). Mỗi milestone row: tên milestone (editable inline) + status selector (PENDING/ACTIVE/DONE) + auto-save khi blur/change.

**Given** admin chỉnh sửa tên milestone inline,
**When** blur khỏi input,
**Then** Server Action `updateMilestoneAction` được gọi. Thành công → toast subtle "Đã lưu." Thất bại → toast destructive, value revert về giá trị cũ.

**Given** admin thay đổi milestone status,
**When** select value mới,
**Then** Server Action được gọi ngay. Nếu set ACTIVE và có milestone ACTIVE khác → server tự xử lý (constraint đã mô tả). UI revalidate và hiển thị đúng trạng thái mới.

**And** Milestone change không trigger email notification (FR-12).
**And** Milestone update không phụ thuộc vào project status — có thể thực hiện ở bất kỳ project status nào.
