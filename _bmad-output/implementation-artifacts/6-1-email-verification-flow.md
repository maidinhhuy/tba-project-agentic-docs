# Story 6.1: Email Verification Flow (BE + FE)

**Status:** ready-for-dev
**Story ID:** 6.1
**Epic:** 6 — Authentication Enhancement

---

## Story

As a new Customer,
I want to verify my email address before my account is activated,
So that the system ensures only valid email owners can register and log in.

---

## Acceptance Criteria

### Backend

**AC-1** — Register creates unverified account + sends email:
**Given** `POST /api/v1/auth/register` với valid payload,
**When** email chưa tồn tại,
**Then** user được tạo với `is_active=false`, `is_email_verified=false`. Token UUID được lưu vào `email_verifications` (TTL 24h). Email verification được gửi async qua `EmailClient`. Trả về HTTP 201.

**AC-2** — Verify email with valid token:
**Given** `POST /api/v1/auth/verify-email?token={token}`,
**When** token tồn tại và chưa hết hạn,
**Then** user được update `is_email_verified=true`, `is_active=true`. Token record bị xóa khỏi DB. Trả về HTTP 200.

**AC-3** — Verify email with invalid/expired token:
**Given** `POST /api/v1/auth/verify-email?token={token}`,
**When** token không tồn tại hoặc đã hết hạn,
**Then** trả về HTTP 400 với error code `AUTHENTICATION_FAILED`.

**AC-4** — Login guard for unverified email:
**Given** `POST /api/v1/auth/login` với password đúng nhưng `is_email_verified=false`,
**Then** trả về HTTP 403 với error body `{ "error": "EMAIL_NOT_VERIFIED", ... }`.

**AC-5** — Resend verification:
**Given** `POST /api/v1/auth/resend-verification` với body `{ "email": "..." }`,
**When** email tồn tại và chưa verified,
**Then** token cũ bị xóa, token mới được lưu và email mới được gửi. Trả về HTTP 200.
**And** khi email đã verified hoặc không tồn tại → cũng trả về HTTP 200 (không leak info).

**AC-6** — Rate limit resend:
**Given** user đã request resend >= 3 lần trong 1 giờ,
**When** gọi resend lần thứ 4,
**Then** trả về HTTP 429 với error code `TOO_MANY_REQUESTS`.

**AC-7** — Security endpoints:
**And** `POST /api/v1/auth/verify-email` và `POST /api/v1/auth/resend-verification` đều là `permitAll()` — đã có sẵn trong `SecurityConfig`, KHÔNG cần thay đổi.

### Frontend

**AC-8** — Register redirect to verify-email page:
**Given** Customer submit form đăng ký thành công (HTTP 201),
**Then** redirect sang `/register/verify-email?email={encodedEmail}` — trang "Check your inbox".

**AC-9** — Email verification page:
**Given** Customer click link trong email (`/register/verify-email?token={token}&email={email}`),
**When** page load, VerifyEmailClient tự động gọi `verifyEmailAction(token)`,
**Then** Thành công → hiện "Email verified!" + nút "Go to Login". Thất bại → hiện lỗi + nút "Resend email".

**AC-10** — Login error for unverified email:
**Given** BE trả về `body.error === 'EMAIL_NOT_VERIFIED'`,
**Then** FE hiện "Please verify your email first." — **đã hoạt động**, KHÔNG cần thay đổi `login/actions.ts`.

---

## Tasks / Subtasks

### Backend Tasks

- [ ] **Task 1: Create `ResendVerificationUseCase` interface**
  - File: `modules/core/src/main/java/com/tba/agentic/port/in/ResendVerificationUseCase.java`
  - Interface content:
    ```java
    package com.tba.agentic.port.in;
    public interface ResendVerificationUseCase {
      void execute(String email);
    }
    ```
  - Không cần import thêm — String là Java built-in

- [ ] **Task 2: Create `JooqEmailVerificationRepository`**
  - File: `modules/infrastructure/src/main/java/com/tba/agentic/infrastructure/repository/database/JooqEmailVerificationRepository.java`
  - Implement `EmailVerificationRepository` từ `core/port/out/`
  - Import `com.tba.agentic.domain.user.UserId` (khác với `domain.value.user.UserId` dùng trong UserRepository)
  - Xem full implementation pattern trong Dev Notes bên dưới

- [ ] **Task 3: Create `VerifyEmailService`**
  - File: `modules/application/src/main/java/com/tba/agentic/application/service/VerifyEmailService.java`
  - Implement `VerifyEmailUseCase`
  - Dependencies: `UserRepository`, `EmailVerificationRepository`
  - Xem full pattern trong Dev Notes bên dưới

- [ ] **Task 4: Create `ResendVerificationService`**
  - File: `modules/application/src/main/java/com/tba/agentic/application/service/ResendVerificationService.java`
  - Implement `ResendVerificationUseCase`
  - Dependencies: `UserRepository`, `EmailVerificationRepository`, `EmailClient`, `String frontendBaseUrl`
  - Xem full pattern trong Dev Notes bên dưới

- [ ] **Task 5: Update `RegisterUserService`**
  - File: `modules/application/src/main/java/com/tba/agentic/application/service/RegisterUserService.java`
  - **Thay đổi chính**: BỎ `user.activate()` — user phải tồn tại với `is_active=false`
  - Thêm dependencies: `EmailVerificationRepository`, `EmailClient`, `String frontendBaseUrl`
  - Sau khi save user: tạo token, lưu vào `email_verifications`, gửi email async
  - Xem full pattern trong Dev Notes bên dưới

- [ ] **Task 6: Update `LoginUserService` — add email verification check**
  - File: `modules/application/src/main/java/com/tba/agentic/application/service/LoginUserService.java`
  - Sau khi password check thành công, thêm:
    ```java
    if (!user.getAccount().isEmailVerified()) {
      throw new AuthenticationException(
          AuthenticationException.Type.EMAIL_NOT_VERIFIED,
          "Please verify your email before logging in");
    }
    ```
  - Đặt TRƯỚC check `isActive()` (hoặc thay thế nó — xem Dev Notes)

- [ ] **Task 7: Update `GlobalExceptionHandler` — handle EMAIL_NOT_VERIFIED → 403**
  - File: `modules/infrastructure/src/main/java/com/tba/agentic/config/GlobalExceptionHandler.java`
  - Update method `handleDomainAuthentication`:
    ```java
    @ExceptionHandler(com.tba.agentic.domain.exception.AuthenticationException.class)
    public ResponseEntity<ObjectNode> handleDomainAuthentication(
        com.tba.agentic.domain.exception.AuthenticationException ex, HttpServletRequest request) {
      if (ex.getType() == com.tba.agentic.domain.exception.AuthenticationException.Type.EMAIL_NOT_VERIFIED) {
        return buildErrorResponse("EMAIL_NOT_VERIFIED", ex.getMessage(), HttpStatus.FORBIDDEN, request);
      }
      return buildErrorResponse("AUTHENTICATION_FAILED", ex.getMessage(), HttpStatus.UNAUTHORIZED, request);
    }
    ```
  - Không thay đổi signature, chỉ thêm if-check bên trong method

- [ ] **Task 8: Update `AuthController` — add verify-email and resend endpoints**
  - File: `modules/infrastructure/src/main/java/com/tba/agentic/adapter/controller/AuthController.java`
  - Thêm `VerifyEmailUseCase verifyEmailUseCase` và `ResendVerificationUseCase resendVerificationUseCase` vào các field `private final` (Lombok `@RequiredArgsConstructor` sẽ inject)
  - Thêm 2 endpoint methods:
    ```java
    @PostMapping("/verify-email")
    public ResponseEntity<Void> verifyEmail(@RequestParam("token") String token) {
      verifyEmailUseCase.execute(token);
      return ResponseEntity.ok().build();
    }

    @PostMapping("/resend-verification")
    public ResponseEntity<Void> resendVerification(@Valid @RequestBody ResendVerificationRequest req) {
      resendVerificationUseCase.execute(req.email());
      return ResponseEntity.ok().build();
    }
    ```
  - Thêm import: `org.springframework.web.bind.annotation.RequestParam`
  - Tạo DTO `ResendVerificationRequest` (xem Task 8b bên dưới)

- [ ] **Task 8b: Create `ResendVerificationRequest` DTO**
  - File: `modules/infrastructure/src/main/java/com/tba/agentic/adapter/transfer/request/ResendVerificationRequest.java`
  - Content:
    ```java
    package com.tba.agentic.adapter.transfer.request;
    import jakarta.validation.constraints.Email;
    import jakarta.validation.constraints.NotBlank;
    public record ResendVerificationRequest(@NotBlank @Email String email) {}
    ```

- [ ] **Task 9: Update `UseCaseConfig` — register new beans, update registerUserUseCase**
  - File: `modules/infrastructure/src/main/java/com/tba/agentic/config/UseCaseConfig.java`
  - Update `registerUserUseCase` bean — thêm `EmailVerificationRepository` và `EmailClient` dependencies
  - Thêm `verifyEmailUseCase` bean
  - Thêm `resendVerificationUseCase` bean
  - Xem full pattern trong Dev Notes bên dưới

### Frontend Tasks

- [ ] **Task 10: Update `register/actions.ts` — return email in success result**
  - File: `tba-project-agentic-fe/src/app/[locale]/(auth)/register/actions.ts`
  - Thay `return { success: true }` → `return { success: true, email: data.email }`
  - Update return type annotation

- [ ] **Task 11: Update `RegisterForm` — redirect to verify-email on success**
  - File: `tba-project-agentic-fe/src/app/[locale]/(auth)/register/_components/RegisterForm.tsx`
  - Thay:
    ```typescript
    } else {
      router.push('/login')
    }
    ```
    Thành:
    ```typescript
    } else if (result?.email) {
      router.push(`/register/verify-email?email=${encodeURIComponent(result.email)}`)
    }
    ```

- [ ] **Task 12: Verify existing FE pages work (no changes needed)**
  - `register/verify-email/actions.ts` — đã đúng: `POST /api/v1/auth/verify-email?token=` ✅
  - `register/verify-email/_components/VerifyEmailClient.tsx` — full UI đã implement ✅
  - `register/verify-email/page.tsx` — page wrapper đã có ✅
  - `login/actions.ts` — đã handle `EMAIL_NOT_VERIFIED` ✅

### Verification

- [ ] **Task 13: Build verification**
  - `cd tba-project-agentic-be && make compile` → BUILD SUCCESSFUL

- [ ] **Task 14: Manual test flow**
  - Register new user → nhận email verification (hoặc check log `[EMAIL-DEV]` nếu dev mode)
  - Click link → verify thành công → có thể đăng nhập
  - Login với tài khoản chưa verify → HTTP 403 `EMAIL_NOT_VERIFIED`
  - Resend verification → nhận email mới

---

## Dev Notes

### CRITICAL: Module Structure

```
modules/
  core/src/main/java/com/tba/agentic/
    domain/entity/user/User.java           ← entity (dùng trong application layer)
    domain/value/user/UserId.java          ← value object, dùng trong UserRepository
    domain/user/UserId.java                ← KHÁC! dùng trong EmailVerificationRepository
    port/in/VerifyEmailUseCase.java        ← ĐÃ TỒN TẠI (chỉ có interface)
    port/in/ResendVerificationUseCase.java ← CẦN TẠO MỚI
    port/out/EmailVerificationRepository.java ← ĐÃ TỒN TẠI
  application/src/main/java/com/tba/agentic/
    application/service/RegisterUserService.java ← UPDATE (bỏ activate())
    application/service/LoginUserService.java    ← UPDATE (thêm email check)
    application/service/VerifyEmailService.java  ← TẠO MỚI
    application/service/ResendVerificationService.java ← TẠO MỚI
  infrastructure/src/main/java/com/tba/agentic/
    adapter/controller/AuthController.java           ← UPDATE
    adapter/transfer/request/ResendVerificationRequest.java ← TẠO MỚI
    config/GlobalExceptionHandler.java               ← UPDATE
    config/UseCaseConfig.java                        ← UPDATE
    infrastructure/repository/database/JooqEmailVerificationRepository.java ← TẠO MỚI
```

### CRITICAL: Hai loại UserId — KHÔNG nhầm lẫn

Có 2 `UserId` khác nhau trong codebase:
- `com.tba.agentic.domain.value.user.UserId` — dùng trong `UserRepository` và `User` entity
- `com.tba.agentic.domain.user.UserId` — dùng trong `EmailVerificationRepository` (legacy domain)

Khi cần convert giữa hai loại:
```java
// domain.value.user.UserId → domain.user.UserId (cho EmailVerificationRepository)
com.tba.agentic.domain.user.UserId emailVerifUserId =
    new com.tba.agentic.domain.user.UserId(user.getUserId().value());

// domain.user.UserId → domain.value.user.UserId (cho UserRepository.findById)
com.tba.agentic.domain.value.user.UserId userRepoId =
    new com.tba.agentic.domain.value.user.UserId(emailVerifUserId.value());
```

### JooqEmailVerificationRepository — Full Implementation

```java
package com.tba.agentic.infrastructure.repository.database;

import static com.tba.agentic.infrastructure.jooq.Tables.EMAIL_VERIFICATIONS;

import com.tba.agentic.domain.user.UserId;
import com.tba.agentic.port.out.EmailVerificationRepository;
import java.time.Instant;
import java.time.OffsetDateTime;
import java.time.ZoneOffset;
import java.util.Optional;
import lombok.RequiredArgsConstructor;
import org.jooq.DSLContext;
import org.springframework.stereotype.Repository;

@Repository
@RequiredArgsConstructor
public class JooqEmailVerificationRepository implements EmailVerificationRepository {

  private final DSLContext dsl;

  @Override
  public void save(UserId userId, String token, Instant expiresAt) {
    dsl.insertInto(EMAIL_VERIFICATIONS)
        .set(EMAIL_VERIFICATIONS.USER_ID, userId.value())
        .set(EMAIL_VERIFICATIONS.TOKEN, token)
        .set(EMAIL_VERIFICATIONS.EXPIRES_AT, OffsetDateTime.ofInstant(expiresAt, ZoneOffset.UTC))
        .execute();
  }

  @Override
  public Optional<UserId> findUserIdByToken(String token) {
    return dsl.select(EMAIL_VERIFICATIONS.USER_ID)
        .from(EMAIL_VERIFICATIONS)
        .where(EMAIL_VERIFICATIONS.TOKEN.eq(token))
        .and(EMAIL_VERIFICATIONS.EXPIRES_AT.gt(OffsetDateTime.now(ZoneOffset.UTC)))
        .fetchOptional()
        .map(r -> new UserId(r.value1()));
  }

  @Override
  public void deleteByUserId(UserId userId) {
    dsl.deleteFrom(EMAIL_VERIFICATIONS)
        .where(EMAIL_VERIFICATIONS.USER_ID.eq(userId.value()))
        .execute();
  }

  @Override
  public int countRecentByUserId(UserId userId, Instant since) {
    return dsl.fetchCount(
        EMAIL_VERIFICATIONS,
        EMAIL_VERIFICATIONS.USER_ID.eq(userId.value())
            .and(EMAIL_VERIFICATIONS.CREATED_AT.gt(OffsetDateTime.ofInstant(since, ZoneOffset.UTC))));
  }
}
```

### VerifyEmailService — Full Implementation

```java
package com.tba.agentic.application.service;

import com.tba.agentic.domain.entity.user.User;
import com.tba.agentic.domain.exception.AuthenticationException;
import com.tba.agentic.domain.user.UserId;
import com.tba.agentic.domain.value.user.UserId as UserRepoId;
import com.tba.agentic.port.in.VerifyEmailUseCase;
import com.tba.agentic.port.out.EmailVerificationRepository;
import com.tba.agentic.port.out.UserRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.transaction.annotation.Transactional;
```

> ⚠️ Không dùng alias `as` trong Java! Import cả 2 rồi dùng fully-qualified:

```java
package com.tba.agentic.application.service;

import com.tba.agentic.domain.entity.user.User;
import com.tba.agentic.domain.exception.AuthenticationException;
import com.tba.agentic.port.in.VerifyEmailUseCase;
import com.tba.agentic.port.out.EmailVerificationRepository;
import com.tba.agentic.port.out.UserRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.transaction.annotation.Transactional;

@RequiredArgsConstructor
@Transactional
public class VerifyEmailService implements VerifyEmailUseCase {

  private final UserRepository userRepository;
  private final EmailVerificationRepository emailVerificationRepository;

  @Override
  public void execute(String token) {
    // domain.user.UserId (EmailVerificationRepository contract)
    com.tba.agentic.domain.user.UserId emailVerifUserId =
        emailVerificationRepository
            .findUserIdByToken(token)
            .orElseThrow(
                () ->
                    new AuthenticationException(
                        AuthenticationException.Type.INVALID_TOKEN,
                        "Invalid or expired verification token"));

    // Convert to domain.value.user.UserId for UserRepository
    com.tba.agentic.domain.value.user.UserId userRepoId =
        new com.tba.agentic.domain.value.user.UserId(emailVerifUserId.value());

    User user =
        userRepository
            .findById(userRepoId)
            .orElseThrow(
                () ->
                    new AuthenticationException(
                        AuthenticationException.Type.ACCOUNT_NOT_FOUND,
                        "User not found for verification token"));

    user.activate();
    userRepository.save(user);
    emailVerificationRepository.deleteByUserId(emailVerifUserId);
  }
}
```

### ResendVerificationService — Full Implementation

```java
package com.tba.agentic.application.service;

import com.tba.agentic.domain.entity.user.User;
import com.tba.agentic.domain.exception.AuthenticationException;
import com.tba.agentic.domain.value.user.Email;
import com.tba.agentic.port.in.ResendVerificationUseCase;
import com.tba.agentic.port.out.EmailClient;
import com.tba.agentic.port.out.EmailMessage;
import com.tba.agentic.port.out.EmailVerificationRepository;
import com.tba.agentic.port.out.UserRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.transaction.annotation.Transactional;
import java.time.Instant;
import java.util.Optional;
import java.util.UUID;

@RequiredArgsConstructor
@Transactional
public class ResendVerificationService implements ResendVerificationUseCase {

  private final UserRepository userRepository;
  private final EmailVerificationRepository emailVerificationRepository;
  private final EmailClient emailClient;
  private final String frontendBaseUrl;

  @Override
  public void execute(String email) {
    Optional<User> userOpt = userRepository.findByEmail(new Email(email));
    if (userOpt.isEmpty() || userOpt.get().getAccount().isEmailVerified()) {
      return; // Don't reveal if email exists or already verified
    }
    User user = userOpt.get();
    com.tba.agentic.domain.user.UserId userId =
        new com.tba.agentic.domain.user.UserId(user.getUserId().value());

    // Rate limit: max 3 resends per hour
    int recentCount = emailVerificationRepository.countRecentByUserId(
        userId, Instant.now().minusSeconds(3600));
    if (recentCount >= 3) {
      throw new AuthenticationException(
          AuthenticationException.Type.TOO_MANY_REQUESTS,
          "Too many verification emails. Please wait before requesting again.");
    }

    // Invalidate old tokens, create new one
    emailVerificationRepository.deleteByUserId(userId);
    String token = UUID.randomUUID().toString();
    Instant expiresAt = Instant.now().plusSeconds(86400); // 24h
    emailVerificationRepository.save(userId, token, expiresAt);

    String link = frontendBaseUrl + "/register/verify-email?token=" + token
        + "&email=" + java.net.URLEncoder.encode(email, java.nio.charset.StandardCharsets.UTF_8);
    emailClient.send(new EmailMessage(
        email,
        "Verify your TBA email",
        "<p>Click the link below to verify your email address:</p>"
            + "<p><a href=\"" + link + "\">Verify Email</a></p>"
            + "<p>This link expires in 24 hours.</p>"));
  }
}
```

### Updated RegisterUserService — Full Implementation

```java
package com.tba.agentic.application.service;

import com.tba.agentic.domain.entity.user.User;
import com.tba.agentic.domain.exception.EmailConflictException;
import com.tba.agentic.domain.value.user.PasswordHash;
import com.tba.agentic.domain.value.user.Role;
import com.tba.agentic.port.bound.user.RegisterUserCommand;
import com.tba.agentic.port.in.RegisterUserUseCase;
import com.tba.agentic.port.out.EmailClient;
import com.tba.agentic.port.out.EmailMessage;
import com.tba.agentic.port.out.EmailVerificationRepository;
import com.tba.agentic.port.out.PasswordHasher;
import com.tba.agentic.port.out.UserRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.transaction.annotation.Transactional;
import java.time.Instant;
import java.util.UUID;

@RequiredArgsConstructor
@Transactional
public class RegisterUserService implements RegisterUserUseCase {

  private final UserRepository userRepository;
  private final PasswordHasher passwordHasher;
  private final EmailVerificationRepository emailVerificationRepository;
  private final EmailClient emailClient;
  private final String frontendBaseUrl;

  @Override
  public User execute(RegisterUserCommand cmd) {
    if (userRepository.existsByEmail(cmd.email())) {
      throw new EmailConflictException(cmd.email().toString());
    }

    PasswordHash hash = new PasswordHash(passwordHasher.hash(cmd.password().value()));
    User user = User.create(cmd.email(), hash, Role.CUSTOMER, cmd.displayName());
    // ⚠️ KHÔNG gọi user.activate() — user phải chờ email verification
    User saved = userRepository.save(user);

    // Save verification token
    com.tba.agentic.domain.user.UserId userId =
        new com.tba.agentic.domain.user.UserId(saved.getUserId().value());
    String token = UUID.randomUUID().toString();
    Instant expiresAt = Instant.now().plusSeconds(86400); // 24h
    emailVerificationRepository.save(userId, token, expiresAt);

    // Send verification email (async, fire-and-forget via @Async in ResendEmailClient)
    String email = cmd.email().value();
    String link = frontendBaseUrl + "/register/verify-email?token=" + token
        + "&email=" + java.net.URLEncoder.encode(email, java.nio.charset.StandardCharsets.UTF_8);
    emailClient.send(new EmailMessage(
        email,
        "Verify your TBA email",
        "<p>Welcome to TBA! Click the link below to verify your email address:</p>"
            + "<p><a href=\"" + link + "\">Verify Email</a></p>"
            + "<p>This link expires in 24 hours.</p>"));

    return saved;
  }
}
```

### Updated UseCaseConfig — Relevant Changes

```java
// UPDATE: registerUserUseCase — thêm emailVerifications và emailClient
@Bean
public RegisterUserUseCase registerUserUseCase(
    UserRepository users,
    PasswordHasher hasher,
    EmailVerificationRepository emailVerifications,
    EmailClient emailClient,
    @Value("${tba.frontend.base-url}") String frontendBaseUrl) {
  return new RegisterUserService(users, hasher, emailVerifications, emailClient, frontendBaseUrl);
}

// ADD: verifyEmailUseCase
@Bean
public VerifyEmailUseCase verifyEmailUseCase(
    UserRepository users,
    EmailVerificationRepository emailVerifications) {
  return new VerifyEmailService(users, emailVerifications);
}

// ADD: resendVerificationUseCase
@Bean
public ResendVerificationUseCase resendVerificationUseCase(
    UserRepository users,
    EmailVerificationRepository emailVerifications,
    EmailClient emailClient,
    @Value("${tba.frontend.base-url}") String frontendBaseUrl) {
  return new ResendVerificationService(users, emailVerifications, emailClient, frontendBaseUrl);
}
```

Thêm imports vào đầu file `UseCaseConfig.java`:
```java
import com.tba.agentic.application.service.VerifyEmailService;
import com.tba.agentic.application.service.ResendVerificationService;
import com.tba.agentic.port.in.VerifyEmailUseCase;
import com.tba.agentic.port.in.ResendVerificationUseCase;
import com.tba.agentic.port.out.EmailVerificationRepository;
```

### Updated LoginUserService — New Email Check

```java
// Sau dòng password check, TRƯỚC dòng isActive check:
if (!user.getAccount().isEmailVerified()) {
  throw new AuthenticationException(
      AuthenticationException.Type.EMAIL_NOT_VERIFIED,
      "Please verify your email before logging in");
}

if (!user.getAccount().isActive()) {
  throw new AuthenticationException(
      AuthenticationException.Type.INVALID_CREDENTIALS, "Invalid credentials");
}
```

### FE: register/actions.ts Update

```typescript
// Trước: return { success: true }
// Sau:
return { success: true, email: data.email }
```

Return type suy ra tự động từ TypeScript — không cần khai báo explicit type.

### FE: RegisterForm.tsx Update

```typescript
// Tìm dòng: router.push('/login')
// Thay bằng:
if (result?.email) {
  router.push(`/register/verify-email?email=${encodeURIComponent(result.email)}`)
}
```

### AuthController — Thêm imports cần thiết

```java
import com.tba.agentic.port.in.VerifyEmailUseCase;
import com.tba.agentic.port.in.ResendVerificationUseCase;
import com.tba.agentic.adapter.transfer.request.ResendVerificationRequest;
import org.springframework.web.bind.annotation.RequestParam;
```

### email_verifications Table Schema (V1, không cần migration)

```sql
CREATE TABLE email_verifications (
    verification_id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    token VARCHAR(255) NOT NULL,
    expires_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
);
```

**Không có `used` column** — token "consumed" bằng cách DELETE bởi `deleteByUserId()`.

jOOQ generated tables:
- `com.tba.agentic.infrastructure.jooq.tables.EmailVerifications` — access via `Tables.EMAIL_VERIFICATIONS`
- Fields: `EMAIL_VERIFICATIONS.USER_ID`, `EMAIL_VERIFICATIONS.TOKEN`, `EMAIL_VERIFICATIONS.EXPIRES_AT`, `EMAIL_VERIFICATIONS.CREATED_AT`

### ResendEmailClient — @Async đã hoạt động

`emailClient.send()` là `@Async` trong `ResendEmailClient` — return ngay, không block thread. Transaction trong `RegisterUserService` sẽ commit trước khi email gửi. Nếu email fail → log WARN, không rollback transaction. Behavior này là đúng (email failure không cancel registration).

### AuthenticationException type → HTTP mapping (sau khi sửa GlobalExceptionHandler)

| Type | HTTP | Error Code |
|------|------|------------|
| `EMAIL_NOT_VERIFIED` | 403 | `EMAIL_NOT_VERIFIED` |
| Tất cả types khác | 401 | `AUTHENTICATION_FAILED` |

### TOO_MANY_REQUESTS Exception Handling

`AuthenticationException(TOO_MANY_REQUESTS, ...)` sẽ bị bắt bởi `handleDomainAuthentication` hiện tại → 401. Để trả về 429 đúng chuẩn, thêm case trong GlobalExceptionHandler:

```java
if (ex.getType() == AuthenticationException.Type.TOO_MANY_REQUESTS) {
  return buildErrorResponse("TOO_MANY_REQUESTS", ex.getMessage(), HttpStatus.TOO_MANY_REQUESTS, request);
}
if (ex.getType() == AuthenticationException.Type.EMAIL_NOT_VERIFIED) {
  return buildErrorResponse("EMAIL_NOT_VERIFIED", ex.getMessage(), HttpStatus.FORBIDDEN, request);
}
return buildErrorResponse("AUTHENTICATION_FAILED", ex.getMessage(), HttpStatus.UNAUTHORIZED, request);
```

### Existing Infrastructure — KHÔNG cần thay đổi

- `SecurityConfig` — đã có `POST /api/v1/auth/verify-email` và `POST /api/v1/auth/resend-verification` như `permitAll()` ✅
- `DevEmailClient` và `ResendEmailClient` — không cần thay đổi ✅
- `login/actions.ts` — đã handle `EMAIL_NOT_VERIFIED` ✅
- `register/verify-email/actions.ts` — đã có `verifyEmailAction` và `resendVerificationAction` ✅
- `register/verify-email/_components/VerifyEmailClient.tsx` — full UI đã implement ✅
- `register/verify-email/page.tsx` — wrapper đã có ✅

---

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Completion Notes List

- `VerifyEmailUseCase`, `EmailVerificationRepository`, và `AuthenticationException.Type.EMAIL_NOT_VERIFIED` đều đã tồn tại trong codebase
- `SecurityConfig` đã allow `POST /api/v1/auth/verify-email` và `POST /api/v1/auth/resend-verification` — không cần thay đổi
- FE verify-email pages/actions đã implement hoàn chỉnh — chỉ cần update RegisterForm redirect
- Hai loại `UserId` trong codebase: `domain.value.user.UserId` (UserRepository) vs `domain.user.UserId` (EmailVerificationRepository) — phải convert carefully
- `email_verifications` table không có `used` column — consume token bằng DELETE

### File List

**CREATE (BE):**
- `modules/core/src/main/java/com/tba/agentic/port/in/ResendVerificationUseCase.java`
- `modules/application/src/main/java/com/tba/agentic/application/service/VerifyEmailService.java`
- `modules/application/src/main/java/com/tba/agentic/application/service/ResendVerificationService.java`
- `modules/infrastructure/src/main/java/com/tba/agentic/infrastructure/repository/database/JooqEmailVerificationRepository.java`
- `modules/infrastructure/src/main/java/com/tba/agentic/adapter/transfer/request/ResendVerificationRequest.java`

**UPDATE (BE):**
- `modules/application/src/main/java/com/tba/agentic/application/service/RegisterUserService.java`
- `modules/application/src/main/java/com/tba/agentic/application/service/LoginUserService.java`
- `modules/infrastructure/src/main/java/com/tba/agentic/adapter/controller/AuthController.java`
- `modules/infrastructure/src/main/java/com/tba/agentic/config/GlobalExceptionHandler.java`
- `modules/infrastructure/src/main/java/com/tba/agentic/config/UseCaseConfig.java`

**UPDATE (FE):**
- `tba-project-agentic-fe/src/app/[locale]/(auth)/register/actions.ts`
- `tba-project-agentic-fe/src/app/[locale]/(auth)/register/_components/RegisterForm.tsx`
