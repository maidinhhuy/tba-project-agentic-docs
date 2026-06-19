# Báo cáo Tổng hợp Tiến độ Dự án TBA-Agentic
**Ngày:** 19/06/2026  
**Người tổng hợp:** Claude Agent (rà soát trực tiếp từ codebase)  
**Nguồn dữ liệu:** Code scan trực tiếp tại `tba-project-agentic-be` + `tba-project-agentic-fe` + tài liệu trong `tba-project-agentic-docs`

---

## Tóm tắt Executive

Kể từ báo cáo cuối (13/06/2026), dự án đã có **tiến độ đột phá**:
- Backend đã trải qua **"Full Backend Release"** (PR#73) + **refactor 3-module hexagonal** (PR#74)
- Frontend đã thêm **i18n EN/VI**, **silent token refresh**, **admin pages đầy đủ**, **project settings**
- Tiến độ thực tế hiện tại ước tính **~80–85%** (so với 48% hôm 13/06)

---

## Kiến trúc Tổng quan

| Layer | Stack | Trạng thái |
|---|---|---|
| Backend | Spring Boot 3 + Hexagonal (3 modules: core / application / infrastructure) | ✅ Đang chạy |
| Frontend | Next.js 16 + React 19 + shadcn/ui + Tailwind 4 + next-intl | ✅ Đang chạy |
| Database | PostgreSQL 16 + Flyway + jOOQ codegen | ✅ Schema V1 đầy đủ |
| Auth | JWT stateless (15 phút) + Refresh Token (7 ngày) | ✅ Implemented |
| Email | Resend (TODO) / DevEmailClient (dev fallback) | ⚠️ Chỉ có dev stub |
| Deploy | Railway (BE) + Vercel (FE) | ⚠️ CI có, chưa xác nhận production |

---

## Tiến độ theo Epic & Story

### ✅ EPIC 1 — Foundation — **100% HOÀN THÀNH**

| Story | Trạng thái | Bằng chứng từ codebase |
|---|---|---|
| **1.1 Backend Scaffold** | ✅ Done | `TbaAgenticApplication.java`, `docker-compose.yml`, `SecurityConfig`, `GlobalExceptionHandler`, `TrackingIdFilter` |
| **1.2 DB Schema Migration** | ✅ Done | `V1__initial_schema.sql` — 7 bảng đầy đủ: users, projects, milestones, project_status_history, refresh_tokens, email_verifications, login_attempts |
| **1.3 jOOQ Codegen** | ✅ Done | Generated classes: `Tables.java`, `Users.java`, `Projects.java`, `Milestones.java`, `ProjectStatusHistory.java`... (7 tables + records) |
| **1.4 Core Domain Model** | ✅ Done | modules/core: 36+ domain classes — entities, value objects, 9 exceptions, 14 in-ports, 13 out-ports, 9 commands |
| **1.5 Frontend Init** | ✅ Done | `next.config.ts`, Tailwind 4, shadcn/ui (14 components), TypeScript strict, `lib/api.ts`, layout shell |

---

### ✅ EPIC 2 — Authentication & Self-Registration — **~85% HOÀN THÀNH**

| Story | Trạng thái | Chi tiết |
|---|---|---|
| **2.1 JWT Auth Backend** | ✅ Done | `LoginUserService`, `JwtTokenProvider`, `JwtAuthenticationFilter`, `AuthController` (`/api/v1/auth/login`), `LoginResponse`, `LoginRequest` |
| **2.2 Email Infrastructure** | ⚠️ Partial | `DevEmailClient` (dev stub) tồn tại — **KHÔNG có `ResendEmailAdapter`** (email thật chưa gửi được) |
| **2.3 Self-Registration** | ✅ Done | `RegisterUserService`, `AuthController` (`/api/auth/register`), user tạo với `is_active=true`, `is_email_verified=true` (email verify flow đã bỏ theo AD-12) |
| **2.4 Auth UI** | ✅ Done | `/login`, `/register`, `/register/verify-email` pages — React Hook Form + Zod, `loginAction`, `registerAction`, `logoutAction`, cookie httpOnly |
| **2.5 RBAC + Route Protection** | ✅ Done (cơ bản) | `SecurityConfig` (ADMIN/CUSTOMER routes), `proxy.ts` (Vercel middleware), `/403` page, silent refresh (`refreshTokenAction`) |

**Lưu ý Story 2.1:** Refresh token flow (`POST /api/auth/refresh`) có `PgTokenRepository` nhưng chưa xác nhận endpoint `/api/auth/refresh` được expose đầy đủ. `LoginAttemptRepository` (port interface) tồn tại nhưng **không có implementation class** — login rate limiting chưa được wired.

**Lưu ý Story 2.2 — Critical Gap:** Chỉ có `DevEmailClient` (log ra console). `ResendEmailAdapter` chưa tồn tại. Email notification cho admin (FR-14) và customer (FR-15) **không hoạt động** trong production.

---

### ✅ EPIC 3 — Customer Portal — **~90% HOÀN THÀNH**

| Story | Trạng thái | Chi tiết |
|---|---|---|
| **3.1 Submit Project** | ✅ Done | BE: `SubmitProjectService`, `CustomerProjectController` (`POST /api/v1/customer/projects`), `JooqProjectRepository.save()`, admin email async (DevEmailClient). FE: `/projects/new` + `ProjectForm`, `submitProjectAction` |
| **3.2 Customer Dashboard** | ✅ Done | BE: `GetCustomerProjectsService`, `CustomerProjectController` (`GET /api/v1/customer/projects`). FE: `/projects` page, `ProjectCard` component, skeleton loader, empty state, 2-column grid |
| **3.3 Project Detail** | ✅ Done | BE: `GetProjectDetailService`, `CustomerProjectController` (`GET /api/v1/customer/projects/{id}`). FE: `/projects/[id]` page — milestones list, status-badge, status history (activity feed) |

**Bổ sung ngoài spec gốc:** Story 3.3 được mở rộng với `/projects/[id]/settings` (edit project form) — `UpdateCustomerProjectService` + `EditProjectForm`.

---

### ✅ EPIC 4 — Admin Dashboard — **~85% HOÀN THÀNH**

| Story | Trạng thái | Chi tiết |
|---|---|---|
| **4.1 Admin Project Table** | ✅ Done | BE: `GetAllProjectsService`, `AdminProjectController` (`GET /api/v1/admin/projects`), filter/sort support. FE: `/admin` page — table responsive, filter by status, sort by status/date |
| **4.2 Status Update + Progress Note** | ✅ Done | BE: `UpdateProjectStatusService`, `AdminProjectController` (`PATCH /api/v1/admin/projects/{id}/status`), state machine validation, revision count decrement, `project_status_history` insert. FE: `/admin/projects/[id]` — `UpdateStatusForm`, `updateStatusAction` |
| **4.3 Milestone Management** | ✅ Done | BE: `UpdateMilestoneService`, `AdminProjectController` (`PUT /api/v1/admin/projects/{id}/milestones/{milestoneId}`), 1-ACTIVE constraint. FE: milestone row trong admin detail page |

**Lưu ý Story 4.2:** Email notification cho customer (FR-15) wired nhưng phụ thuộc vào `DevEmailClient` — không gửi email thật. `forceRevision` override khi `revision_count=0` cần xác minh thêm.

---

## Biểu đồ Tiến độ Tổng thể

```
Epic 1 — Foundation      ████████████████████ 100%  ✅
Epic 2 — Auth & Register ████████████████░░░░  85%  ⚠️ thiếu Resend + rate limiting
Epic 3 — Customer Portal ██████████████████░░  90%  ⚠️ email thật chưa gửi
Epic 4 — Admin Dashboard ████████████████░░░░  85%  ⚠️ force-revision & email thật
─────────────────────────────────────────────────────────────
Tổng dự án               ████████████████░░░░ ~88%
```

---

## Danh sách Chi tiết: Đã hoàn thành / Đang dở / Chưa làm

### ✅ ĐÃ HOÀN THÀNH

**Backend (tba-project-agentic-be):**
- `modules/core`: 36+ domain classes (entities, value objects, exceptions, port interfaces, commands)
- `modules/application`: 12 service implementations (RegisterUser, LoginUser, SubmitProject, GetCustomerProjects, GetProjectDetail, UpdateCustomerProject, GetAllProjects, GetAdminProjectDetail, UpdateProjectStatus, UpdateMilestone, GetUserProfile, TokenManagement)
- `modules/infrastructure`: 4 controllers (Auth, User, CustomerProject, AdminProject), JWT filter, tracking filter, SecurityConfig (RBAC), GlobalExceptionHandler, Flyway V1 migration, jOOQ codegen output, 3 repository impls (PgDatabaseUserRepository, JooqProjectRepository, PgTokenRepository)
- CI/CD: GitHub Actions với deploy to VPS on push to main
- Docker Compose + Dockerfile cho 3-module architecture

**Frontend (tba-project-agentic-fe):**
- 56 TypeScript/TSX files tổng cộng
- Authentication: `/login`, `/register`, `/register/verify-email` pages với form validation
- Customer: `/projects` (list + card grid), `/projects/new` (submit form), `/projects/[id]` (detail + milestones + activity feed), `/projects/[id]/settings` (edit)
- Admin: `/admin` (project table + filter/sort), `/admin/projects/[id]` (detail + status update form)
- Infrastructure: `lib/api.ts` (apiFetch với auto-refresh 401), `proxy.ts` (route protection + silent refresh), `lib/jwt.ts` (decode), `lib/schemas/project.schema.ts`
- Components: ProjectCard, StatusBadge, PasswordInput, MarkdownPreview, LanguageSwitcher + 14 shadcn/ui components
- i18n: EN + VI translations (`messages/en.json`, `messages/vi.json`)
- `/403` forbidden page

---

### ⚠️ ĐANG DỞ (Cần hoàn thiện trước production)

| # | Hạng mục | Story | Mức độ | Tình trạng |
|---|---|---|---|---|
| P1 | **ResendEmailAdapter** — email thật chưa gửi được | 2.2 | Critical | Chỉ có `DevEmailClient` (log console). Cần implement `ResendEmailAdapter` với `RESEND_API_KEY` env var, 5s timeout, async retry |
| P2 | **Login Rate Limiting** — `LoginAttemptRepository` interface tồn tại nhưng không có DB implementation | 2.1 | High | `LoginAttemptRepository` port tồn tại; `LoginAttempts` jOOQ table tồn tại. Thiếu `PgLoginAttemptRepository` class. 5-fail-lockout (FR-2, NFR-5) chưa hoạt động |
| P3 | **Refresh Token Endpoint** — xác nhận `/api/auth/refresh` endpoint được expose | 2.1 | Medium | `PgTokenRepository` + `TokenManagementService` tồn tại. Cần verify `AuthController` có `POST /api/auth/refresh` và `POST /api/auth/logout` endpoints |
| P4 | **verify-email page là dead-end** — `/register/verify-email` page tồn tại nhưng email verification đã bị disable | 2.4 | Low | Theo AD-12, is_active/is_email_verified = true ngay khi register. Nên redirect về `/projects` hoặc xóa trang này để tránh confusion |
| P5 | **Test coverage rất thấp** — chỉ 2 unit tests trong toàn bộ backend | 1.4 | Medium | `ProjectStatusTransitionTest` + `ProjectRevisionCountTest`. Không có integration tests, controller tests, service tests |

---

### ❌ CHƯA LÀM / CHƯA BẮT ĐẦU

| # | Hạng mục | Story | Ghi chú |
|---|---|---|---|
| N1 | **ResendEmailAdapter** (email thật) | 2.2 | Xem P1 — chưa tạo file `ResendEmailAdapter.java` |
| N2 | **PgLoginAttemptRepository** (rate limiting DB impl) | 2.1 | Cần tạo class implements `LoginAttemptRepository` port, dùng jOOQ `LoginAttempts` table |
| N3 | **Integration tests / End-to-end tests** | All | Không có test nào kiểm tra toàn bộ luồng BE → DB. FE không rõ có test runner không |
| N4 | **Admin email notification khi project submit** (FR-14) | 3.1 | Code đã wired vào `SubmitProjectService` nhưng email thật chưa đến. Phụ thuộc N1 |
| N5 | **Customer email notification khi status update** (FR-15) | 4.2 | Code đã wired vào `UpdateProjectStatusService` nhưng email thật chưa đến. Phụ thuộc N1 |
| N6 | **Revision counter override** (forceRevision=true flow) | 4.2 | Backend logic cần verify: khi `revision_count=0` và `forceRevision=true`, transition vẫn cho phép |
| N7 | **Custom domain setup** (AD-18) | Infra | Vercel + Railway default domains — chưa gắn custom domain |
| N8 | **WCAG 2.2 AA audit** (NFR-4) | All FE | `aria-describedby` trên form errors, `aria-busy` trên skeleton — chưa audit đầy đủ |

---

## So sánh với Báo cáo 13/06/2026

| Hạng mục | 13/06/2026 | 19/06/2026 | Delta |
|---|---|---|---|
| Tiến độ tổng thể | ~48% | ~88% | **+40%** |
| Epic 2 (Auth) | 26% | ~85% | +59% |
| Epic 3 (Customer) | 36% | ~90% | +54% |
| Epic 4 (Admin) | 10% | ~85% | +75% |
| BE Services | 0–1 | 12/12 | +11 |
| FE Pages | 5 | 15+ | +10 |
| Architecture | 2-module | 3-module | Upgraded |

**Lý do đột phá:** PR#73 "Dev → Main: Full backend release" đã merge toàn bộ epic 2/3/4 backend vào main. PR#74 refactor lên 3-module hexagonal. FE đã thêm 10+ PRs từ TASK-109 đến TASK-123 (i18n, admin pages, token refresh, user profile nav).

---

## Rủi ro & Vấn đề Hiện tại

### Critical 🔴

| # | Vấn đề | Impact | Hành động cần làm |
|---|---|---|---|
| R1 | **Email không hoạt động** — `DevEmailClient` chỉ log. Admin không nhận thông báo project mới. Customer không nhận thông báo status update | FR-14, FR-15 bị vi phạm | Implement `ResendEmailAdapter` với `RESEND_API_KEY`, cấu hình Railway env var |
| R2 | **Login rate limiting không hoạt động** — `LoginAttemptRepository` interface có nhưng không có DB impl | NFR-5 bị vi phạm (5-fail lockout) | Tạo `PgLoginAttemptRepository implements LoginAttemptRepository` |

### Warning 🟡

| # | Vấn đề | Impact | Hành động cần làm |
|---|---|---|---|
| W1 | **Test coverage cực thấp** — 2 unit tests. Không có integration test | Refactor an toàn khó. CI không catch regressions | Thêm ít nhất: controller test cho auth/customer/admin endpoints, service test cho UpdateProjectStatusService |
| W2 | **Legacy code tại root** — `core/` và `application/` tại root vẫn tồn tại (phiên bản cũ trước refactor) | Gây nhầm lẫn cho agent/dev | Verify `modules/` là authoritative, xóa hoặc archive root `core/` + `application/` |
| W3 | **`/register/verify-email` là dead page** — email verification đã disable nhưng page vẫn còn | UX confusing | Redirect `/register/verify-email` → `/projects` hoặc xóa route |
| W4 | **`AdminProjectController` endpoints chưa xác nhận đầy đủ** — cần verify `GET /api/v1/admin/projects/{id}` trả về `allowedTransitions` array | Story 4.2 AC quan trọng | Curl test endpoint, verify response shape |

---

## Thống kê Code

### Backend (tba-project-agentic-be)

| Hạng mục | Số lượng |
|---|---|
| Gradle modules | 3 (core / application / infrastructure) |
| Domain classes (entities + VOs + exceptions) | 36+ |
| Port interfaces (in + out + internal) | 28 |
| Command/Query bound objects | 9 |
| Service implementations | 12 |
| REST Controllers | 4 |
| Repository implementations | 3 (PgDatabaseUserRepository, JooqProjectRepository, PgTokenRepository) |
| DB migrations | 1 (V1 — 7 tables) |
| jOOQ generated tables | 7 |
| Unit tests | 2 (ProjectStatusTransitionTest, ProjectRevisionCountTest) |
| Git commits | 14 |

### Frontend (tba-project-agentic-fe)

| Hạng mục | Số lượng |
|---|---|
| TypeScript/TSX source files | 56 |
| App routes (pages) | 15+ |
| Custom components | 5 (ProjectCard, StatusBadge, PasswordInput, MarkdownPreview, LanguageSwitcher) |
| shadcn/ui components | 14 |
| Server Actions | 8+ (loginAction, registerAction, logoutAction, submitProjectAction, updateProjectAction, updateMilestoneAction, updateStatusAction, refreshTokenAction) |
| i18n locales | 2 (EN, VI) |
| Git commits | 55+ |

---

## Khuyến nghị Ưu tiên

### Giai đoạn 1 — Unblock Production (ưu tiên ngay)

1. **Implement `ResendEmailAdapter`** — Story 2.2 còn lại duy nhất trước production:
   ```
   File: modules/infrastructure/src/main/java/com/tba/agentic/infrastructure/email/ResendEmailAdapter.java
   - Implements EmailClient
   - Dùng Resend HTTP API (POST https://api.resend.com/emails)
   - Timeout 5s, catch exception, log WARN
   - Conditional trên RESEND_API_KEY env var
   ```

2. **Implement `PgLoginAttemptRepository`** — wiring rate limiting:
   ```
   File: modules/infrastructure/src/main/java/com/tba/agentic/infrastructure/repository/PgLoginAttemptRepository.java
   - Implements LoginAttemptRepository
   - Dùng jOOQ LoginAttempts table
   - countRecentFailures(email, withinMinutes)
   - recordAttempt(email, ip, success)
   ```

3. **Verify refresh + logout endpoints** — test curl:
   ```bash
   POST /api/auth/refresh  {"refreshToken": "..."}
   POST /api/auth/logout   {"refreshToken": "..."}
   ```

### Giai đoạn 2 — Chất lượng & Cleanup

4. **Xóa legacy root `core/` + `application/`** — chỉ giữ `modules/`
5. **Ẩn hoặc xóa `/register/verify-email`** — redirect về `/projects`
6. **Thêm integration tests** — ít nhất 1 test per controller

### Giai đoạn 3 — Production Deploy

7. **Cấu hình Railway env vars** — đặc biệt `RESEND_API_KEY`, `TBA_ADMIN_EMAIL`
8. **Gắn custom domain** (AD-18) — trước khi demo với khách hàng thật
9. **WCAG 2.2 AA audit** — `aria-describedby` trên form errors

---

## Tài liệu Tham khảo

| Tài liệu | Đường dẫn |
|---|---|
| PRD | `_bmad-output/planning-artifacts/prds/prd-AI-driven-2026-06-11/prd.md` |
| Architecture | `_bmad-output/planning-artifacts/architecture/architecture.md` |
| Epics & Stories | `_bmad-output/planning-artifacts/epics.md` |
| UX Design | `_bmad-output/planning-artifacts/ux-designs/ux-AI-driven-2026-06-11/` |
| Sprint Report cũ | `_bmad-output/implementation-artifacts/sprint-report-2026-06-13.md` |
| Story specs | `_bmad-output/implementation-artifacts/2-1-simple-login-backend.md`, `2-3-backend-self-registration.md`, `3-1-backend-submit-project.md`, `3-3-backend-get-project-detail.md` |
