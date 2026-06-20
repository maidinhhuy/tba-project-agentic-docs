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
Người dùng có thể tự đăng ký và đăng nhập trực tiếp vào đúng portal của mình (email verification tạm thời bỏ qua).
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

**And** port interfaces exist in `core/port/in/`: `LoginUseCase`, `RegisterUserUseCase`, `SubmitProjectUseCase`, `GetCustomerProjectsUseCase`, `GetProjectDetailUseCase`, `UpdateProjectStatusUseCase`, `UpdateMilestoneUseCase`, `GetAllProjectsUseCase`.

> ~~`VerifyEmailUseCase`~~ — removed (email verification tạm thời bỏ qua).

**And** port interfaces exist in `core/port/out/`: `UserRepository`, `ProjectRepository`, `MilestoneRepository`, `AuditRepository`, `TokenRepository`, `LoginAttemptRepository`, `EmailClient`.

> ~~`EmailVerificationRepository`~~ — removed (email verification tạm thời bỏ qua).

**And** value objects are immutable records/classes: `Email`, `Password`, `PasswordHash`, `Role`, `UserId`, `ProjectId`, `DisplayName`, `IpAddress`, `UserAgent`.

**And** exceptions defined: `InvalidStatusTransitionException`, `AuthenticationException` (with `Type` enum: ACCOUNT_NOT_FOUND, INVALID_CREDENTIALS, TOO_MANY_REQUESTS, TOKEN_EXPIRED), `ProjectNotFoundException`, `EmailConflictException`.

> ~~`EMAIL_NOT_VERIFIED`~~ — removed từ AuthenticationException (không còn verify email flow).

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

Người dùng có thể tự đăng ký và đăng nhập trực tiếp vào đúng portal của mình (email verification tạm thời bỏ qua).
**FRs covered:** FR-2, FR-3, FR-1/FR-13 (self-registration)

---

### Story 2.1: JWT Auth Backend

As a developer,
I want the JWT authentication stack wired from the `template/` starter pattern — login endpoint, token filter, refresh, logout, and rate limiting,
So that all API endpoints are protected and users can authenticate securely.

**Acceptance Criteria:**

**Given** a `POST /api/auth/login` request với email + password hợp lệ của user `is_active=true`,
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

### Story 2.3: Self-Registration

> **Thay đổi kế hoạch (2026-06-13):** Email verification tạm thời bỏ qua. Đăng ký thành công = đăng nhập được ngay. Các tasks liên quan đến email verification (TASK-52 VerifyEmailService, TASK-53 verify-email/resend endpoints) đã bị Cancel trên Notion.

As a new customer,
I want to register for an account with my email and be able to login immediately,
So that I can access the customer portal without waiting for email verification.

**Acceptance Criteria:**

**Given** `POST /api/auth/register` với `email`, `displayName`, `password` (min 8 ký tự) hợp lệ,
**When** email chưa tồn tại trong hệ thống,
**Then** trả về HTTP 201. User được tạo với `role=CUSTOMER`, `is_active=true`, `is_email_verified=true`.

**Given** `POST /api/auth/register` với email đã tồn tại,
**When** gửi request,
**Then** trả về HTTP 409 với error `"EMAIL_CONFLICT"`.

**Given** `POST /api/auth/register` với password < 8 ký tự hoặc thiếu field bắt buộc,
**When** gửi request,
**Then** trả về HTTP 400 với error `"VALIDATION_FAILED"`.

**Given** đăng ký thành công (HTTP 201),
**When** customer gọi `POST /api/auth/login` với đúng credentials,
**Then** login thành công ngay — không cần bước xác thực email.

**And** endpoint `/api/auth/register` được configure `permitAll()` trong `SecurityConfig`.

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
**Then** redirect sang `/login` với toast success "Đăng ký thành công. Vui lòng đăng nhập."

> ~~Redirect `/register/verify-email`~~ — removed (email verification tạm thời bỏ qua).

**Given** email đã tồn tại khi register,
**When** Server Action trả về 409,
**Then** hiển thị inline error tại email field: "Email này đã được sử dụng."

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

---

## Epic 6: Authentication Enhancement — Email Verification & Password Management

**Goal:** Hoàn thiện authentication flow: bắt buộc xác nhận email khi đăng ký và cho phép Customer đổi mật khẩu.

**Scope:** Restore email verification (revert AD-12 temporary skip), add resend verification, add change password.

**Dependencies:** Epic 2 (auth infrastructure), Story 2.2 (email infrastructure/Resend), Story 5.1 (transactional email service).

---

### Story 6.1: Email Verification Flow (BE + FE)

As a new Customer,
I want to verify my email address before my account is activated,
So that the system ensures only valid email owners can register and log in.

**Acceptance Criteria:**

#### Backend

**Given** `POST /api/auth/register` với valid payload,
**When** request được xử lý,
**Then** user được tạo với `is_active=false`, `is_email_verified=false`. Một verification token được lưu vào `email_verifications` table (TTL 24h). Email verification được gửi async qua Resend. Response trả về HTTP 201.

**Given** `GET /api/auth/verify-email?token={token}`,
**When** token tồn tại và chưa hết hạn,
**Then** user được update `is_email_verified=true`, `is_active=true`. Token được đánh dấu used. Trả về HTTP 200.

**Given** `GET /api/auth/verify-email?token={token}`,
**When** token không tồn tại hoặc đã used,
**Then** trả về HTTP 400 với error code `INVALID_VERIFICATION_TOKEN`.

**Given** `GET /api/auth/verify-email?token={token}`,
**When** token đã hết hạn (>24h),
**Then** trả về HTTP 400 với error code `TOKEN_EXPIRED`.

**Given** `POST /api/auth/login` với email chưa verified,
**When** password đúng nhưng `is_email_verified=false`,
**Then** trả về HTTP 403 với error code `EMAIL_NOT_VERIFIED`.

**Given** `POST /api/auth/resend-verification` với body `{ email }`,
**When** email tồn tại và chưa verified,
**Then** tạo token mới (invalidate token cũ), gửi email mới. Trả về HTTP 200.

**Given** `POST /api/auth/resend-verification`,
**When** email đã verified hoặc không tồn tại,
**Then** trả về HTTP 200 (không leak thông tin — same response).

**Given** SecurityConfig,
**Then** các endpoint sau là `permitAll()`: `POST /api/auth/register`, `GET /api/auth/verify-email`, `POST /api/auth/resend-verification`.

#### Frontend

**Given** Customer submit form đăng ký thành công,
**When** Server Action nhận HTTP 201 từ backend,
**Then** redirect sang trang `/register/success` với message "Vui lòng kiểm tra email để xác nhận tài khoản."

**Given** Customer click link xác nhận trong email,
**When** `GET /verify-email?token={token}` được load,
**Then** Next.js page gọi Server Action verify token. Thành công → redirect `/login` với toast "Email đã được xác nhận. Bạn có thể đăng nhập." Thất bại (invalid/expired) → hiện trang lỗi với nút "Gửi lại email xác nhận".

**Given** Customer click "Gửi lại email xác nhận",
**When** Server Action `resendVerificationAction` gọi `POST /api/auth/resend-verification`,
**Then** hiện toast "Email xác nhận đã được gửi lại. Kiểm tra hộp thư của bạn."

**Given** Customer đăng nhập với tài khoản chưa xác nhận email,
**When** backend trả về 403 `EMAIL_NOT_VERIFIED`,
**Then** FE hiển thị message "Vui lòng xác nhận email trước khi đăng nhập." kèm link "Gửi lại email xác nhận".

**Technical Notes:**
- `email_verifications` table đã có trong V1 schema — không cần migration mới
- Token: UUID, lưu bcrypt hash (optional), TTL 24h
- Email template: subject "Xác nhận email TBA", link `{FRONTEND_BASE_URL}/verify-email?token={token}`
- `ResendEmailClient` đã live (Story 2.2) — chỉ cần implement `VerifyEmailUseCase`, `ResendVerificationUseCase`

---

### Story 6.2: Change Password (BE + FE)

As a Customer,
I want to change my password by providing my current password for authentication,
So that I can update my credentials securely without requiring admin intervention.

**Acceptance Criteria:**

#### Backend

**Given** `POST /api/auth/change-password` với body `{ oldPassword, newPassword }` và Bearer token hợp lệ,
**When** `oldPassword` match BCrypt hash hiện tại,
**Then** hash password mới, update `password_hash` trong DB. Revoke tất cả refresh tokens của user (force re-login trên tất cả devices). Trả về HTTP 200.

**Given** `POST /api/auth/change-password`,
**When** `oldPassword` không match,
**Then** trả về HTTP 400 với error code `WRONG_PASSWORD`.

**Given** `POST /api/auth/change-password`,
**When** `newPassword` không đủ mạnh (< 8 ký tự, thiếu uppercase/number),
**Then** trả về HTTP 400 với error code `WEAK_PASSWORD` và validation message.

**Given** `POST /api/auth/change-password`,
**When** không có Bearer token hoặc token hết hạn,
**Then** trả về HTTP 401.

**Given** SecurityConfig,
**Then** `POST /api/auth/change-password` yêu cầu `authenticated()`.

#### Frontend

**Given** Customer navigate tới trang `/profile` (hoặc `/settings`),
**When** page load,
**Then** hiện form "Đổi mật khẩu" với 3 fields: Current Password, New Password, Confirm New Password.

**Given** Customer submit form đổi mật khẩu,
**When** Confirm New Password không match New Password,
**Then** client-side validation hiện lỗi "Mật khẩu xác nhận không khớp." Không gọi Server Action.

**Given** form hợp lệ được submit,
**When** Server Action `changePasswordAction` gọi `POST /api/auth/change-password`,
**Then** button đổi thành "Đang xử lý…" và disabled.

**Given** backend trả về HTTP 200,
**When** Server Action nhận response,
**Then** toast success "Mật khẩu đã được thay đổi. Vui lòng đăng nhập lại." Sau 2 giây, logout và redirect `/login`.

**Given** backend trả về 400 `WRONG_PASSWORD`,
**When** Server Action nhận response,
**Then** hiện lỗi "Mật khẩu hiện tại không đúng." trên field Current Password.

**Given** backend trả về 400 `WEAK_PASSWORD`,
**When** Server Action nhận response,
**Then** hiện lỗi validation trên field New Password.

**Technical Notes:**
- Route: `/[locale]/(dashboard)/profile/page.tsx` (authenticated route, reuse existing layout)
- Server Action: `changePasswordAction` trong `src/app/[locale]/(dashboard)/profile/actions.ts`
- Sau khi đổi mật khẩu thành công → gọi `signOut()` (NextAuth) và redirect `/login`
- Password strength validation: đồng bộ rule FE và BE (≥8 ký tự, ≥1 uppercase, ≥1 số)

---

## Epic 7: UI Bug Fixes

Khắc phục các bug UI nhỏ phát sinh sau khi hoàn thành các epic chính. Các fix này không thay đổi business logic hay BE, chỉ cải thiện tính đúng đắn và đầy đủ của giao diện.

**FRs covered:** Không có FR mới — đây là bug fix cho FR-9 (Admin project table/detail)

---

### Story 7.1: Admin Project Detail — Hiển thị Description và Reference URL

As an admin,
I want to see the project's description and reference URL on the Admin Project Detail page,
So that I have complete project context without having to ask the customer.

**Root Cause (đã xác nhận qua code review):**
- BE: `AdminProjectDetailResponse` đã trả về `description` và `reference` — API đúng
- FE: TypeScript interface `AdminProjectDetail` trong `page.tsx` thiếu 2 field này; page không render chúng

**Acceptance Criteria:**

**Given** `GET /api/v1/admin/projects/{id}` trả về response có `description` và `reference`,
**When** Admin load `/admin/projects/{id}`,
**Then** TypeScript interface `AdminProjectDetail` trong `page.tsx` phải có `description: string | null` và `reference: string | null`.

**Given** project có `description` (không null),
**When** Admin xem Admin Project Detail page,
**Then** render một card "Thông tin dự án" hiển thị `description` dưới dạng text thuần, đặt sau Project Header Card và trước UpdateStatusForm.

**Given** project có `reference` (không null, không rỗng),
**When** Admin xem Admin Project Detail page,
**Then** card "Thông tin dự án" hiển thị `reference` dưới dạng hyperlink có `target="_blank" rel="noopener noreferrer"`, label "Tài liệu tham khảo".

**Given** `description` là null hoặc rỗng,
**When** Admin xem Admin Project Detail page,
**Then** không render description section (không hiện label rỗng).

**Given** `reference` là null hoặc rỗng,
**When** Admin xem Admin Project Detail page,
**Then** không render reference section (không hiện label rỗng).

**Given** cả `description` và `reference` đều null,
**When** Admin xem Admin Project Detail page,
**Then** không render card "Thông tin dự án" (toàn bộ card ẩn).

**Technical Notes:**
- File cần sửa: `src/app/[locale]/(admin)/admin/projects/[id]/page.tsx`
- Chỉ sửa FE — KHÔNG sửa BE
- Style: cùng pattern với các card hiện có (`.bg-white border border-gray-200/80 rounded-xl p-5 shadow-sm`)
- `productType` đã có trong response — có thể hiển thị cùng card nếu phù hợp UX

---

## Epic 8: Milestone Delivery & Customer Review

**Goal:** Sau mỗi milestone hoàn thành, Admin giao kết quả (link + file) cho Customer xem. Customer có thể chấp nhận hoặc yêu cầu chỉnh sửa có kèm feedback. Milestone cuối được chấp nhận → project tự chuyển DELIVERED.

**FRs covered:** FR-7 (project detail — revision counter có ý nghĩa thực), FR-8 (activity feed — thêm review events), FR-10 (state machine — transitions mới từ AWAITING_REVIEW), FR-15 (email notification — extend cho review events)

**New state machine transitions:**
- `IN_DEVELOPMENT → AWAITING_REVIEW` — triggered khi Admin giao milestone (đã có, nay tự động)
- `AWAITING_REVIEW → IN_DEVELOPMENT` — triggered khi Customer approve milestone giữa (mới)
- `AWAITING_REVIEW → DELIVERED` — triggered khi Customer approve milestone cuối (mới)

**Dependencies:** Epic 2 (auth), Epic 3 (customer portal), Epic 4 (admin dashboard), Story 2.2 (email), Story 5.1 (transactional)

---

### Story 8.1: DB Migration V2 + S3 File Storage Infrastructure

As a developer,
I want the database extended with milestone delivery/review columns and a milestone_reviews history table, and S3 file storage wired up as a port/adapter,
So that all subsequent Epic 8 stories have a stable data and storage foundation to build on.

**Acceptance Criteria:**

**Given** Flyway V2 migration chạy,
**When** `./gradlew flywayMigrate` executes,
**Then** migration `V2__milestone_delivery.sql` hoàn thành không lỗi.

**Given** migration chạy xong,
**When** schema được inspect,
**Then** bảng `milestones` có thêm các columns:
- `deliverable_url TEXT` — link demo/figma/staging (nullable)
- `deliverable_file_key TEXT` — S3 object key (nullable)
- `deliverable_file_name TEXT` — tên file gốc để hiển thị (nullable)
- `delivery_note TEXT` — ghi chú Admin khi giao (nullable)
- `delivered_at TIMESTAMPTZ` — thời điểm giao (nullable)
- `review_status VARCHAR(30) NOT NULL DEFAULT 'PENDING_DELIVERY'` — CHECK IN ('PENDING_DELIVERY','PENDING_REVIEW','APPROVED','REVISION_REQUESTED')

**And** bảng mới `milestone_reviews` tồn tại với columns:
- `id UUID PK DEFAULT gen_random_uuid()`
- `milestone_id BIGINT NOT NULL FK→milestones(milestone_id) ON DELETE CASCADE`
- `action VARCHAR(30) NOT NULL` — CHECK IN ('APPROVED','REVISION_REQUESTED')
- `comment TEXT` — bắt buộc khi action=REVISION_REQUESTED
- `reviewed_by BIGINT REFERENCES users(user_id)`
- `reviewed_at TIMESTAMPTZ DEFAULT now()`

**And** indexes: `idx_milestone_reviews_milestone_id`.

**Given** S3 configuration,
**When** app khởi động với env vars `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, `AWS_S3_BUCKET_NAME`,
**Then** `S3FileStorageAdapter` implements `FileStoragePort` (core/port/out) khởi động không lỗi.

**Given** `FileStoragePort.upload(fileName, contentType, inputStream)` được gọi,
**When** upload thành công,
**Then** trả về `fileKey` (String) — dạng `deliverables/{projectId}/{milestoneId}/{uuid}-{fileName}`.

**Given** `FileStoragePort.generatePresignedUrl(fileKey, ttlMinutes)` được gọi,
**When** fileKey tồn tại,
**Then** trả về URL presigned hợp lệ với TTL = `ttlMinutes`.

**And** `FileStoragePort` interface nằm ở `core/port/out/`, `S3FileStorageAdapter` nằm ở `infrastructure/storage/`.
**And** `.env.local` template thêm: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, `AWS_S3_BUCKET_NAME`.
**And** jOOQ codegen chạy lại sau migration — generated classes cho `milestone_reviews` tồn tại.

---

### Story 8.2: BE Admin — Deliver Milestone API

As an admin,
I want to deliver a milestone by attaching a link and/or file upload along with a delivery note,
So that the customer is notified and can review the result immediately.

**Acceptance Criteria:**

**Given** `POST /api/v1/admin/projects/{id}/milestones/{milestoneId}/deliver` với `multipart/form-data`: `deliverableUrl` (optional String), `deliveryNote` (optional String), `file` (optional binary),
**When** milestone có `status=ACTIVE` và project thuộc về `{id}`,
**Then** trả về HTTP 200. Milestone được update: `deliverable_url`, `deliverable_file_key` (nếu có file → upload S3 → lưu key), `deliverable_file_name`, `delivery_note`, `delivered_at=now()`, `review_status=PENDING_REVIEW`.

**Given** cả `deliverableUrl` và `file` đều null/empty,
**When** gửi request,
**Then** trả về HTTP 400 với error `"VALIDATION_FAILED"` — phải có ít nhất 1 trong 2.

**Given** file được upload,
**When** file > 50MB,
**Then** trả về HTTP 400 với error `"FILE_TOO_LARGE"`.

**Given** deliver thành công,
**When** `review_status` chuyển sang `PENDING_REVIEW`,
**Then** project status tự động chuyển sang `AWAITING_REVIEW` (trong cùng transaction).
**And** `project_status_history` có record: `previous_status=IN_DEVELOPMENT`, `new_status=AWAITING_REVIEW`, `changed_by=<admin_email>`, `reason="Milestone {name} đã hoàn thành và sẵn sàng để review."`.

**Given** email notification,
**When** deliver thành công,
**Then** `EmailClient.send()` được gọi async đến customer email, subject "Kết quả milestone đã sẵn sàng: {milestoneName}", body gồm: tên Project, tên Milestone, delivery note (nếu có), link `/projects/{id}`.
**And** email failure không block response.

**And** endpoint yêu cầu `hasRole("ADMIN")`.
**And** nếu milestone không thuộc project `{id}` → HTTP 404.

---

### Story 8.3: BE Customer — Review API + State Machine

As a customer,
I want to approve or request revision for a delivered milestone with a comment,
So that the agency knows exactly what to fix and the project lifecycle progresses correctly.

**Acceptance Criteria:**

**Given** `POST /api/v1/customer/projects/{id}/milestones/{milestoneId}/review` với body `{ action: "APPROVE" }`,
**When** milestone có `review_status=PENDING_REVIEW` và project thuộc Customer đang đăng nhập,
**Then** trả về HTTP 200. Record mới trong `milestone_reviews`: `action=APPROVED`, `reviewed_by=customer_id`, `reviewed_at=now()`. Milestone `review_status=APPROVED`.

**Given** Customer approve và còn milestone tiếp theo (position + 1),
**When** request được xử lý,
**Then** milestone tiếp theo chuyển `status=ACTIVE`, `review_status=PENDING_DELIVERY`. Project status chuyển `IN_DEVELOPMENT`. `project_status_history` có record tương ứng.

**Given** Customer approve và đây là milestone cuối (không còn milestone nào có position lớn hơn),
**When** request được xử lý,
**Then** project status chuyển `DELIVERED`. `project_status_history` có record: `new_status=DELIVERED`, `changed_by=<customer_email>`, `reason="Customer đã chấp nhận milestone cuối."`.
**And** Admin nhận email async: subject "Dự án {projectName} đã hoàn thành", body gồm tên Customer, tên Project, link admin detail.

**Given** `POST /api/v1/customer/projects/{id}/milestones/{milestoneId}/review` với body `{ action: "REQUEST_REVISION", comment: "..." }`,
**When** milestone có `review_status=PENDING_REVIEW`,
**Then** trả về HTTP 200. Record mới trong `milestone_reviews`: `action=REVISION_REQUESTED`, `comment=comment`, `reviewed_by`, `reviewed_at`. Milestone `review_status=REVISION_REQUESTED`.

**Given** `action=REQUEST_REVISION` mà `comment` null hoặc blank,
**When** gửi request,
**Then** trả về HTTP 400 với error `"VALIDATION_FAILED"` — comment bắt buộc.

**Given** `action=REQUEST_REVISION` và `project.revision_count > 0`,
**When** xử lý,
**Then** `revision_count - 1`. Project status chuyển `IN_REVISION`. `project_status_history` có record tương ứng.

**Given** `action=REQUEST_REVISION` và `project.revision_count = 0`,
**When** xử lý,
**Then** `revision_count` giữ nguyên 0. Project status chuyển `IN_REVISION` (override — không block). `project_status_history` có record với `reason` ghi rõ "Extra revision requested".

**Given** revision request thành công,
**When** xử lý xong,
**Then** Admin nhận email async: subject "Yêu cầu chỉnh sửa: {projectName}", body gồm tên Milestone, tên Customer, comment đầy đủ, link `/admin/projects/{id}`.
**And** email failure không block response.

**Given** `GET /api/v1/customer/projects/{id}/milestones/{milestoneId}/deliverable`,
**When** milestone có `deliverable_file_key` (có file upload),
**Then** trả về HTTP 302 redirect đến presigned S3 URL với TTL 30 phút.

**Given** `GET /api/v1/admin/projects/{id}` (extend response hiện có),
**When** Admin gọi endpoint,
**Then** mỗi milestone trong response có thêm field `reviews: List<MilestoneReviewResponse>` — sorted `reviewed_at DESC`. Mỗi item: `id`, `action`, `comment`, `reviewedAt`.

**And** tất cả customer endpoints enforce tenant isolation — Customer chỉ review project của chính mình.

---

### Story 8.4: FE Admin — Form Giao Milestone + Review History

As an admin,
I want a delivery form on each active milestone and a review history timeline,
So that I can share results with customers and track all their feedback in one place.

**Acceptance Criteria:**

**Given** Admin load `/admin/projects/{id}`,
**When** milestone có `status=ACTIVE` và `review_status=PENDING_DELIVERY`,
**Then** hiện button "Giao milestone" trên milestone row đó.

**Given** Admin click "Giao milestone",
**When** modal/sheet mở,
**Then** render form gồm:
- Input "Link kết quả" (URL, optional, placeholder "https://...")
- File upload dropzone (optional, max 50MB, hiện tên file sau khi chọn)
- Textarea "Ghi chú cho khách hàng" (optional)
- Submit button "Giao ngay"
- Validation: phải có ít nhất 1 trong 2 (link hoặc file) trước khi enable Submit.

**Given** Admin submit form hợp lệ,
**When** Server Action `deliverMilestoneAction` đang xử lý,
**Then** button đổi thành "Đang giao…" và disabled.

**Given** deliver thành công,
**When** Server Action trả về 200,
**Then** modal đóng, toast success "Đã giao milestone và thông báo cho khách hàng." Page revalidate — milestone badge đổi sang "Đang chờ review".

**Given** deliver thất bại,
**When** Server Action trả về lỗi,
**Then** toast destructive. Form giữ nguyên dữ liệu.

**Given** milestone có `review_status` không phải `PENDING_DELIVERY`,
**When** Admin xem milestone row,
**Then** hiện badge trạng thái: "Đang chờ review" (PENDING_REVIEW), "Đã chấp nhận" (APPROVED), "Yêu cầu chỉnh sửa" (REVISION_REQUESTED).

**Given** Admin xem Admin Project Detail page,
**When** milestone có `reviews` array không rỗng,
**Then** mỗi milestone row có expandable "Lịch sử review" — timeline gồm các entries:
- APPROVED: icon check xanh + "Khách hàng đã chấp nhận" + `reviewedAt`
- REVISION_REQUESTED: icon warning đỏ + comment đầy đủ + `reviewedAt`

**And** file upload dùng `<input type="file">` với `encType="multipart/form-data"` — Server Action forward multipart đến Spring Boot.

---

### Story 8.5: FE Customer — Xem Deliverable + Form Review

As a customer,
I want to see the delivered result for each milestone and submit my approval or revision request with feedback,
So that I can clearly communicate what I want changed and track the project towards completion.

**Acceptance Criteria:**

**Given** Customer load `/projects/{id}`,
**When** milestone có `review_status=PENDING_REVIEW`,
**Then** milestone row hiện section "Kết quả milestone" gồm:
- Nếu có `deliverable_url`: button "Xem kết quả" → `target="_blank"` mở link
- Nếu có file: button "Tải xuống" → gọi Server Action `downloadDeliverableAction` → redirect presigned URL
- Nếu có `delivery_note`: text note của Admin
- Form review: button "Chấp nhận" + button "Yêu cầu chỉnh sửa"

**Given** Customer click "Yêu cầu chỉnh sửa",
**When** button được click,
**Then** expand textarea "Mô tả yêu cầu chỉnh sửa *" (bắt buộc) + nút "Gửi yêu cầu". Button "Gửi yêu cầu" disabled cho đến khi textarea có nội dung.

**Given** `project.revision_count = 1`,
**When** Customer xem form review,
**Then** hiện cảnh báo vàng: "Bạn còn 1 lần chỉnh sửa. Hãy mô tả đầy đủ yêu cầu."

**Given** `project.revision_count = 0`,
**When** Customer xem form review,
**Then** hiện cảnh báo đỏ: "Bạn đã dùng hết số lần chỉnh sửa. Lần này sẽ được xem xét đặc biệt — hãy mô tả thật chi tiết."

**Given** Customer submit "Chấp nhận",
**When** Server Action `reviewMilestoneAction` thành công,
**Then** toast success "Đã chấp nhận milestone." Page revalidate. Nếu milestone cuối: toast thêm "Dự án đã hoàn thành!". Project status badge đổi tương ứng.

**Given** Customer submit "Gửi yêu cầu" (revision),
**When** Server Action thành công,
**Then** toast success "Đã gửi yêu cầu chỉnh sửa." Page revalidate. Project status badge đổi sang IN_REVISION.

**Given** submit thất bại,
**When** Server Action trả về lỗi,
**Then** toast destructive. Form giữ nguyên dữ liệu (comment không bị clear).

**Given** milestone có `review_status=APPROVED`,
**When** Customer xem milestone row,
**Then** milestone hiện badge "Đã chấp nhận" (green). Không hiện form review.

**Given** milestone có `review_status=REVISION_REQUESTED`,
**When** Customer xem milestone row,
**Then** milestone hiện badge "Đang chỉnh sửa". Không hiện form review (đang chờ Admin giao lại).
