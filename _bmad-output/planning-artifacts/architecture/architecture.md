---
stepsCompleted: [1, 2, 3, 4]
inputDocuments:
  - _bmad-output/planning-artifacts/prds/prd-AI-driven-2026-06-11/prd.md
  - _bmad-output/planning-artifacts/ux-designs/ux-AI-driven-2026-06-11/DESIGN.md
  - _bmad-output/planning-artifacts/ux-designs/ux-AI-driven-2026-06-11/EXPERIENCE.md
  - backend-architecture/OVERVIEW.md
  - backend-architecture/DOMAIN_DEFINATION.md
  - backend-architecture/API_DEVELOPMENT.md
  - backend-architecture/NAMING_CONVENSION.md
  - template (starter template — auth pattern tham khảo, trong be repo)
workflowType: 'architecture'
project_name: 'AI-driven (TBA-agentic)'
user_name: 'Huy'
date: '2026-06-11'
status: 'complete — tất cả architectural decisions đã chốt (AD-01 đến AD-18). Sẵn sàng cho Epics & Stories.'
---

# Architecture Decision Document — TBA-agentic

_Tài liệu này được xây dựng qua collaborative discovery. Phiên 2026-06-11 đã hoàn thành Step 1 (Init) và Step 2 (Context Analysis + Party Mode). Step 3 (Starter Template) đã incorporate `template/` (trong be repo) với JWT auth pattern. Tiếp tục Step 4+ ở phiên mới._

---

## Project Context Analysis

### Requirements Overview

**Functional Requirements:**
15 FRs trong 5 nhóm: Account Management, Project Submission, Project Tracking (Customer Portal), Admin Project Management, và Email Notifications. Core flow: Customer tự đăng ký → xác thực email → submit project → Admin quản lý lifecycle qua 8-state machine → Email notification tự động ở key transitions. Không có file upload, không có real-time features ở v1.

**Non-Functional Requirements:**
- Security: RBAC nghiêm ngặt — customer không thể truy cập `/admin/*`; mỗi Customer chỉ thấy data của mình (tenant isolation ở API layer)
- Email reliability: Transactional email service (không SMTP trực tiếp); timeout 5s, async retry, không block UI
- Performance: Standard web response times đủ — không có real-time constraint
- Accessibility: WCAG 2.2 AA trên tất cả surfaces
- Account security: Lockout 15 phút sau 5 failed login attempts

**Scale & Complexity:**
- Primary domain: Full-stack web application
- Complexity level: Low-Medium (well-scoped MVP, single admin, controlled user base)
- 2 user roles (Customer, Admin), 1 core domain (Project)
- Estimated architectural components: ~6-8 backend use-cases, ~5 frontend routes

### Technical Constraints & Dependencies

- **Frontend stack (fixed):** Next.js + shadcn/ui + Tailwind CSS
- **Backend stack (fixed):** Java Spring Boot + Hexagonal Architecture (core/application) + jOOQ
- **Naming & code conventions (fixed):** NAMING_CONVENTION.md + DOMAIN_DEFINITION_GUIDE.md đã có
- **Database:** PostgreSQL — confirmed (jOOQ implies relational; không cần debate)
- **Email service (open → quyết định ở step tiếp):** Resend ưu tiên (developer experience tốt hơn SendGrid, free tier đủ dùng)
- **Auth:** JWT stateless — access token (15 min) + refresh token (7 days, stored in `refresh_tokens` table) — code pattern tham khảo từ `template/` trong be repo (xem AD-01)
- **Deployment:** Vercel + Railway — đã quyết định (xem §Decisions)

### Cross-Cutting Concerns Identified

1. **Authentication & Authorization** — JWT stateless (`Authorization: Bearer <access_token>`), `JwtAuthenticationFilter` ở Spring Boot, role routing ở cả Next.js middleware và backend filter. Token lưu qua Next.js-managed httpOnly cookie trên browser — không XSS-accessible, Next.js server forward thành `Authorization` header khi gọi Spring Boot.
2. **Email Notification Pipeline** — async, fire-and-forget tại v1; không block request chính
3. **Status State Machine Enforcement** — logic transition sống ở Domain layer; DB CHECK constraint làm guard thứ 2
4. **Tenant Isolation** — mọi Project query filter theo `customer_id` của user đang đăng nhập
5. **Error Handling** — consistent error response format từ Spring Boot (`error_code`, `message`, `details`) từ ngày đầu
6. **Accessibility** — WCAG 2.2 AA: semantic HTML, aria attributes, color contrast (shadcn defaults đảm bảo baseline)

---

## Architecture Decisions Log

> Các quyết định dưới đây được thống nhất qua Party Mode discussion (2026-06-11).
> Phiên mới cần tiếp tục từ Step 3 — không cần re-open các quyết định này.

---

### AD-01: Authentication Mechanism

**Quyết định:** JWT stateless — access token (15 min) + optional refresh token (7 days).

**Không chọn:** ~~httpOnly cookie + Spring Session JDBC~~ (quyết định ban đầu — đã đổi sau khi xác nhận dùng `template/` làm auth pattern reference).

**Lý do đổi sang JWT:**
- `template/` đã có auth stack hoàn chỉnh (JwtTokenProvider, JwtAuthenticationFilter, LoginAttemptService, TokenManagementService) — tái dùng trực tiếp, không viết lại từ đầu
- JWT phù hợp hơn cho Next.js server-to-server pattern (AD-02): Next.js server lưu token trong httpOnly cookie riêng, forward `Authorization: Bearer` khi gọi Spring Boot — không phụ thuộc vào session store
- Refresh token có thể revoke ngay (stored + `revoked` flag trong DB) — giữ được khả năng revoke như Spring Session
- Loại bỏ dependency `spring-session-jdbc` + bảng `spring_session` — schema đơn giản hơn

**Token flow:**
```
Browser → (POST /login) → Next.js Server Action → Spring Boot /api/auth/login
  ↓
Spring Boot trả { token: "...", refreshToken: "..." }
  ↓
Next.js Server Action set httpOnly cookie: tba_access_token=<jwt>
  ↓
Subsequent Server Component requests: Next.js reads cookie → forward Authorization: Bearer <jwt> → Spring Boot
```

**Spring Boot config (`SecurityConfig.java` — từ template):**
```java
http.sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
    .addFilterAfter(jwtAuthenticationFilter, TrackingIdFilter.class)
```

**`JwtProperties` (`application.yml`):**
```yaml
tba-app-api:
  security:
    jwt:
      secret: ${TBA_APP_API_SECURITY_JWT_SECRET}
      access-token-minutes: 15
      refresh-token-days: 7
      authorization-header: Authorization
      token-prefix: "Bearer "
```

**Rate limiting (giữ nguyên từ template):**
- `LoginAttemptService` + `login_attempts` table — 5 failed attempts trong 15 phút → block
- Decorator pattern: `RateLimitingLoginUseCase` wraps `CoreLoginService`

**DB tables cần thiết:**
```sql
-- refresh_tokens: lưu refresh token (optional, chỉ khi rememberMe=true)
CREATE TABLE refresh_tokens (
    refresh_token_id BIGINT PRIMARY KEY,
    user_id          BIGINT NOT NULL REFERENCES users(user_id),
    token            VARCHAR(500) NOT NULL UNIQUE,
    expires_at       TIMESTAMPTZ NOT NULL,
    revoked          BOOLEAN NOT NULL DEFAULT FALSE,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- login_attempts: rate limiting
CREATE TABLE login_attempts (
    attempt_id    BIGSERIAL PRIMARY KEY,
    email         VARCHAR(255) NOT NULL,
    ip_address    INET NOT NULL,
    user_agent    TEXT,
    attempt_time  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    success       BOOLEAN NOT NULL DEFAULT FALSE,
    failure_reason VARCHAR(100),
    user_id       BIGINT
);
CREATE INDEX idx_login_attempts_email_time ON login_attempts(email, attempt_time DESC);
CREATE INDEX idx_login_attempts_ip_time    ON login_attempts(ip_address, attempt_time DESC);
```

---

### AD-02: API Call Pattern (Next.js → Spring Boot)

**Quyết định:** Next.js Server Components fetch server-to-server + Server Actions cho mutations. Browser không bao giờ gọi Spring Boot trực tiếp.

**Không chọn:** Client Component gọi thẳng Spring Boot; Next.js Route Handler làm full proxy.

**Lý do:**
- Loại bỏ CORS complexity giữa browser và Spring Boot
- Secrets (Spring Boot URL) ở server-side Next.js — không expose ra client
- Server Actions + `revalidateTag()` thay thế hoàn toàn TanStack Query cache invalidation

**Pattern:**
```typescript
// lib/api.ts — "server only"
import "server-only";
import { cookies } from "next/headers";

export async function getProjects() {
  const token = (await cookies()).get("tba_access_token")?.value;
  const res = await fetch(`${process.env.SPRING_BOOT_URL}/api/projects`, {
    headers: { Authorization: `Bearer ${token}` },
    next: { tags: ["projects"] },
  });
  if (!res.ok) throw new Error(`API error: ${res.status}`);
  return res.json();
}
```

**Login Server Action pattern:**
```typescript
// lib/actions/auth.ts
"use server";
import { cookies } from "next/headers";

export async function loginAction(formData: FormData) {
  const res = await fetch(`${process.env.SPRING_BOOT_URL}/api/auth/login`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ email: formData.get("email"), password: formData.get("password") }),
  });
  if (!res.ok) throw new Error("Login failed");
  const { token, refreshToken } = await res.json();
  const cookieStore = await cookies();
  cookieStore.set("tba_access_token", token, { httpOnly: true, secure: true, sameSite: "lax", path: "/" });
  if (refreshToken) {
    cookieStore.set("tba_refresh_token", refreshToken, { httpOnly: true, secure: true, sameSite: "lax", path: "/" });
  }
}
```

---

### AD-03: CORS Configuration

**Quyết định:** CORS cho phép Next.js server origin. `allowCredentials: true` vẫn cần thiết vì Next.js server forward JWT qua header (không phải cookie) — nhưng đơn giản hơn AD ban đầu vì không cần xử lý `SameSite=None` cho session cookie.

**Spring Boot CORS config (`SecurityConfig.java`):**
```java
config.setAllowedOrigins(List.of(frontendBaseUrl));   // từ env TBA_APP_API_FRONTEND_BASE_URL
config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
config.setAllowedHeaders(List.of("*"));
config.setAllowCredentials(true);
```

**Ghi chú:** Browser không bao giờ gọi Spring Boot trực tiếp (AD-02) — CORS config chủ yếu phục vụ server-to-server và phòng ngừa misconfiguration.

---

### AD-04: Project Structure — 2 Separate Repos

**Quyết định:** 2 Git repos riêng biệt.

| Repo | URL | Stack |
|---|---|---|
| `tba-project-agentic-be` | https://github.com/maidinhhuy/tba-project-agentic-be | Spring Boot |
| `tba-project-agentic-fe` | https://github.com/maidinhhuy/tba-project-agentic-fe | Next.js |

**Không chọn:** ~~Monorepo (`/frontend`, `/backend` trong 1 repo)~~

**Lý do:** Repos đã được khởi tạo sẵn theo 2 repo riêng. Mỗi repo có CI pipeline độc lập, deploy độc lập lên Railway (be) và Vercel (fe).

---

### AD-05: Deployment

**Quyết định:** Vercel (Next.js) + Railway (Spring Boot + PostgreSQL).

**Không chọn:** Single VPS.

**Lý do:** Solo developer — không nên tốn thời gian ops (nginx, SSL renewal, security patches). Vercel preview deployments có giá trị khi iterate nhanh.

**Lưu ý quan trọng:** Upgrade Railway lên Hobby plan ($5/month) ngay từ đầu — free tier có sleep, Spring Boot cold start 30-60s không chấp nhận được.

**Dockerfile (`Dockerfile` tại root của be repo):**
```dockerfile
FROM eclipse-temurin:21-jre-alpine
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

**Railway env vars (Spring Boot):**
```bash
TBA_DATASOURCE_URL=jdbc:postgresql://...railway.internal:5432/railway
TBA_DATASOURCE_USERNAME=postgres
TBA_DATASOURCE_PASSWORD=xxx
TBA_APP_API_SECURITY_JWT_SECRET=<min-32-chars-random-string>
TBA_APP_API_SECURITY_JWT_ACCESS_TOKEN_MINUTES=15
TBA_APP_API_SECURITY_JWT_REFRESH_TOKEN_DAYS=7
TBA_APP_API_FRONTEND_BASE_URL=https://your-app.vercel.app
EMAIL_API_KEY=xxx
SPRING_PROFILES_ACTIVE=prod
```

**Vercel env vars (Next.js):**
```bash
SPRING_BOOT_URL=https://your-backend.up.railway.app   # server-only, no NEXT_PUBLIC_
```

---

### AD-06: Status State Machine Enforcement

**Quyết định:** Domain layer owns transition logic. Database làm guard thứ 2 (CHECK constraint only, không dùng trigger).

**Implementation:**
```java
// core/domain/model/ProjectStatus.java
public enum ProjectStatus {
    SUBMITTED, ANALYZING, IN_DEVELOPMENT,
    AWAITING_REVIEW, IN_REVISION, FINALIZING, DELIVERED, CANCELLED;

    private static final Map<ProjectStatus, Set<ProjectStatus>> ALLOWED_TRANSITIONS = Map.of(
        SUBMITTED,       Set.of(ANALYZING, CANCELLED),
        ANALYZING,       Set.of(IN_DEVELOPMENT, CANCELLED),
        IN_DEVELOPMENT,  Set.of(AWAITING_REVIEW, CANCELLED),
        AWAITING_REVIEW, Set.of(IN_REVISION, FINALIZING, CANCELLED),
        IN_REVISION,     Set.of(AWAITING_REVIEW, CANCELLED),
        FINALIZING,      Set.of(DELIVERED, CANCELLED),
        DELIVERED,       Set.of(),
        CANCELLED,       Set.of()
    );

    public ProjectStatus transitionTo(ProjectStatus next) {
        if (!ALLOWED_TRANSITIONS.get(this).contains(next))
            throw new InvalidStatusTransitionException(this, next);
        return next;
    }
}
```

Aggregate Root `Project.transitionStatus()` gọi `transitionTo()` và raise `ProjectStatusChangedEvent`.

**DB constraint:**
```sql
ALTER TABLE projects ADD CONSTRAINT valid_status
CHECK (status IN ('SUBMITTED','ANALYZING','IN_DEVELOPMENT',
                  'AWAITING_REVIEW','IN_REVISION','FINALIZING',
                  'DELIVERED','CANCELLED'));
```

---

### AD-07: Audit Trail

**Quyết định:** Append-only `project_status_history` table. Domain raises event, Application layer persists trong cùng `@Transactional` với state change.

**Schema:**
```sql
CREATE TABLE project_status_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    previous_status VARCHAR(30) NOT NULL,
    new_status      VARCHAR(30) NOT NULL,
    changed_by      VARCHAR(255) NOT NULL,   -- admin email hoặc 'system'
    reason          TEXT,                     -- optional progress note từ admin
    changed_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    ip_address      INET
);
CREATE INDEX idx_psh_project_id ON project_status_history(project_id);
CREATE INDEX idx_psh_changed_at ON project_status_history(changed_at DESC);
```

**Layer ownership:**
- `core/domain`: `ProjectStatusChangedEvent`
- `core/port/out`: `AuditRepository` interface
- `application/infrastructure/repository`: `JooqAuditRepository` implementation

**Ghi chú:** Bảng này doubles như nguồn data cho Activity Feed (FR-8) của Customer — không cần bảng riêng cho activity feed.

---

### AD-08: Email Failure Policy

**Quyết định:** Fire-and-forget tại v1. Email failure là acceptable loss — không rollback state machine transition.

**Lý do:** Độ phức tạp của compensating transaction không xứng với risk ở scale v1. Log lỗi + async retry là đủ.

**Email provider:** Resend (developer experience tốt hơn SendGrid, API clean, free tier đủ dùng).

---

### AD-09: Frontend Stack

**Quyết định:**

| Concern | Decision | Packages |
|---|---|---|
| Framework | Next.js 15 + App Router | next |
| UI | shadcn/ui + Tailwind CSS | (đã quyết định) |
| State management | useState/useContext only — không TanStack Query, không Zustand | — |
| Forms | React Hook Form + Zod | react-hook-form, zod, @hookform/resolvers |
| Data fetching | Server Components + Server Actions + revalidateTag | next/cache |
| TypeScript | strict + noUncheckedIndexedAccess | tsc |
| Toast | sonner | sonner |

**Folder structure:**
```
src/
├── app/
│   ├── layout.tsx
│   ├── page.tsx                     # Customer Dashboard
│   ├── login/page.tsx
│   ├── projects/new/page.tsx
│   ├── projects/[id]/page.tsx
│   └── admin/
│       ├── layout.tsx               # admin auth guard
│       ├── page.tsx
│       └── projects/[id]/page.tsx
├── components/
│   ├── ui/                          # shadcn generated
│   ├── customer/
│   └── admin/
├── lib/
│   ├── api.ts                       # server-only fetch wrapper
│   ├── actions/                     # Server Actions
│   └── validations/                 # Zod schemas
└── types/
```

**tsconfig.json (key options):**
```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  }
}
```

---

### AD-10: jOOQ Codegen Pipeline

**Quyết định:** Docker Compose với PostgreSQL bắt buộc trong local dev setup từ sprint 1. CI (GitHub Actions) dùng service containers để chạy `jooq-codegen` trước khi compile.

**Lý do:** jOOQ generated classes phải được tạo từ Flyway migration schema. Nếu không setup từ đầu, CI sẽ break ở sprint 1.

**Boundary rule:** jOOQ `DSLContext` và generated `Tables`/`Records` chỉ tồn tại trong `application/infrastructure/repository`. Không bao giờ leak vào `core` layer.

---

### AD-11: Optimistic Locking

**Quyết định:** Thêm `version` column (optimistic locking) vào bảng `projects` từ đầu.

```sql
ALTER TABLE projects ADD COLUMN version INTEGER NOT NULL DEFAULT 0;
```

jOOQ hỗ trợ optimistic locking built-in. Tránh lost updates khi Admin và Customer đồng thời read/write một project record.

---

### AD-12: Account Creation Flow — Self-Registration

**Quyết định:** Public `/register` endpoint (`permitAll()`). Customer tự đăng ký, xác thực email trước khi đăng nhập.

**Không chọn:** ~~Admin-only account creation~~ (quyết định ban đầu — đã đổi sau Party Mode review).

**Lý do:** Self-registration giảm tải vận hành cho admin, cải thiện UX. Email verification đảm bảo chỉ user thật được active.

**Flow:**
1. Customer `POST /api/auth/register` → tạo account với `is_active=false`, `is_email_verified=false`
2. Email verification gửi tới customer email
3. Customer click link → `GET /api/verification/verify-email?token=...` → `is_email_verified=true`, `is_active=true`
4. Customer có thể đăng nhập

**SecurityConfig:**
```java
.requestMatchers(HttpMethod.POST, "/api/auth/register").permitAll()
.requestMatchers(HttpMethod.GET, "/api/verification/verify-email").permitAll()
.requestMatchers(HttpMethod.POST, "/api/verification/resend-verification").permitAll()
```

**Adaptation từ template:** `RegisterUserService` và `EmailVerificationService` dùng nguyên từ `template/` — không cần thay đổi.

---

### AD-13: Database Migration — Flyway

**Quyết định:** Flyway với SQL migrations (`V{n}__{description}.sql`).

**Không chọn:** Liquibase.

**Lý do:** Template đã dùng Flyway (V1/V2/V3). SQL-first — dễ đọc, dễ review, không cần học XML/YAML. Convention `V{n}__description.sql` đủ cho solo dev.

```
src/main/resources/db/migration/
├── V1__initial_schema.sql      # users, refresh_tokens, login_attempts
├── V2__projects_schema.sql     # projects, project_status_history
└── ...
```

---

### AD-14: Spring Boot Package Structure

**Quyết định:** Follow template structure, đổi tên package theo domain của project.

**Base package:** `com.tba.agentic`

```
application/src/main/java/com/tba/agentic/
├── adapter/
│   ├── controller/
│   │   ├── auth/           # AuthController
│   │   ├── customer/       # CustomerProjectController
│   │   └── admin/          # AdminProjectController, AdminAccountController
│   ├── filter/             # JwtAuthenticationFilter, TrackingIdFilter
│   └── transfer/
│       ├── request/auth/   customer/   admin/
│       └── response/auth/  customer/   admin/
├── application/
│   └── service/
│       ├── auth/           # CoreLoginService, RateLimitingLoginUseCase, RefreshTokenService
│       ├── token/          # TokenManagementService
│       ├── account/        # CreateAccountService (admin-only)
│       └── project/        # SubmitProjectService, UpdateProjectStatusService,
│                           # GetProjectService, ListProjectsService
├── config/                 # JwtProperties, JooqConfiguration, GlobalExceptionHandler
├── infrastructure/
│   ├── email/              # ResendEmailAdapter
│   └── repository/
│       ├── account/        # DatabaseDslAccountRepository
│       ├── project/        # DatabaseDslProjectRepository
│       ├── audit/          # DatabaseDslAuditRepository
│       ├── token/          # DatabaseDslTokenRepository
│       └── security/       # DatabaseDslLoginAttemptRepository
└── security/               # JwtTokenProvider

core/src/main/java/com/tba/agentic/core/
├── domain/
│   ├── aggregation/
│   │   ├── user/           # User aggregate (Account + Profile)
│   │   └── project/        # Project aggregate
│   ├── entity/
│   │   └── account/        # JwtToken, RefreshToken entities
│   └── value/
│       ├── account/        # Email, Password, Role, ...
│       └── project/        # ProjectStatus, RevisionCount, ...
├── exception/
│   ├── auth/
│   └── project/            # InvalidStatusTransitionException
└── port/
    ├── in/                 # Use case interfaces (auth/, account/, project/)
    ├── out/                # Repository interfaces (account/, project/, audit/, token/, email/)
    └── bound/              # Commands & Queries (command/, query/)
```

---

### AD-15: API Versioning

**Quyết định:** Version resource endpoints, không version auth endpoints — follow template pattern.

```
/api/auth/login          # không version — auth contracts ổn định
/api/auth/refresh
/api/v1/customer/projects        # có version — business resource
/api/v1/customer/projects/{id}
/api/v1/admin/projects
/api/v1/admin/projects/{id}/status
/api/v1/admin/accounts
```

**Lý do:** Auth endpoints hiếm khi thay đổi breaking. Resource endpoints có thể cần v2 khi business logic evolve. Phân tách rõ ràng.

---

### AD-16: Error Response Format

**Quyết định:** Dùng `ErrorResponse` từ template — giữ nguyên, không thay đổi.

```json
{
  "error": "AUTHENTICATION_FAILED",
  "message": "Email hoặc mật khẩu không đúng",
  "status": 401,
  "path": "/api/auth/login",
  "timestamp": "2026-06-11T10:30:00.000Z",
  "trackingId": "abc-123",
  "details": null
}
```

`details` chỉ populated khi `VALIDATION_FAILED` — map field → error message:
```json
{
  "error": "VALIDATION_FAILED",
  "details": { "email": "must not be blank", "password": "size must be between 8 and 100" }
}
```

`GlobalExceptionHandler` từ template xử lý toàn bộ: `AuthenticationException`, `DomainException`, `MethodArgumentNotValidException`, `DataAccessException`, generic `Exception`. Project này chỉ cần thêm handlers cho `InvalidStatusTransitionException` và `ProjectNotFoundException`.

---

### AD-17: Local Dev Environment

**Quyết định:** Docker Compose cho PostgreSQL local. Không containerize Spring Boot trong local dev.

**`docker-compose.yml` (project root):**
```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: tba_agentic
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

**`.env.local` (gitignored, tại root của be repo):**
```bash
TBA_DATASOURCE_URL=jdbc:postgresql://localhost:5432/tba_agentic
TBA_DATASOURCE_USERNAME=postgres
TBA_DATASOURCE_PASSWORD=postgres
TBA_APP_API_SECURITY_JWT_SECRET=local-dev-secret-min-32-chars-xxxx
TBA_APP_API_FRONTEND_BASE_URL=http://localhost:3000
```

**jOOQ codegen workflow:**
```bash
docker compose up -d postgres   # start DB
./gradlew flywayMigrate          # run migrations
./gradlew generateJooq           # generate classes từ schema
./gradlew bootRun                # start app
```

`Makefile` target `dev-setup` wrap 3 lệnh trên — developer chỉ cần `make dev-setup`.

---

### AD-18: Custom Domain Strategy

**Quyết định:** Dùng Railway/Vercel default domains cho đến khi onboard customer thật đầu tiên.

**Lý do:** Domain gắn vào là cam kết với URL — đổi sau khi đã share link với customer sẽ phức tạp. Giữ default domains trong development và internal testing, chỉ gắn custom domain ngay trước buổi demo/onboarding thật.

**Timeline:**
- Giai đoạn build: `*.railway.app` + `*.vercel.app`
- Trước customer onboarding đầu tiên: gắn custom domain vào Vercel, update `TBA_APP_API_FRONTEND_BASE_URL` trên Railway

---

## Open Decisions

_Tất cả open decisions đã được chốt (AD-13 đến AD-18). Không còn OD nào._

---

## Trạng thái Workflow

- ✅ Step 1: Init — document khởi tạo, input documents loaded
- ✅ Step 2: Context Analysis — phân tích requirements + Party Mode (11 architectural decisions logged)
- ✅ Step 3: Starter Template — `template/` (be repo) incorporated. AD-01→JWT, AD-12→self-registration.
- ✅ Step 4: Detailed Decisions — tất cả Open Decisions đã chốt (AD-13 Flyway, AD-14 Package Structure, AD-15 API Versioning, AD-16 Error Format, AD-17 Local Dev, AD-18 Custom Domain).
- ⏳ Step 5: Epics & Stories — **sẵn sàng chạy `bmad-create-epics-and-stories`**
