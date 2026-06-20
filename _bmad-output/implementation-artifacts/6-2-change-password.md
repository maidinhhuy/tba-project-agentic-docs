# Story 6.2: Change Password (BE + FE)

**Status:** ready-for-dev
**Story ID:** 6.2
**Epic:** 6 — Authentication Enhancement

---

## Story

As a Customer,
I want to change my password by providing my current password for authentication,
So that I can update my credentials securely without requiring admin intervention.

---

## Acceptance Criteria

### Backend

**AC-1** — Change password success:
**Given** `POST /api/v1/auth/change-password` với body `{ oldPassword, newPassword }` và Bearer token hợp lệ (cookie `tba_access_token`),
**When** `oldPassword` match BCrypt hash hiện tại của user,
**Then** hash `newPassword` bằng BCrypt, update `password_hash` trong DB via `userRepository.save(user)`. Revoke tất cả refresh tokens của user bằng `tokenRepository.revokeAllForUser(userId)`. Trả về HTTP 200.

**AC-2** — Wrong old password:
**Given** `POST /api/v1/auth/change-password` với `oldPassword` sai,
**Then** trả về HTTP 400 với body `{ "error": "WRONG_PASSWORD", ... }`.

**AC-3** — Weak new password:
**Given** `POST /api/v1/auth/change-password` với `newPassword` không đủ mạnh (< 8 ký tự, thiếu uppercase, thiếu số),
**Then** trả về HTTP 400 với `{ "error": "VALIDATION_FAILED", "details": { "newPassword": "..." } }`.

**AC-4** — Unauthenticated:
**Given** `POST /api/v1/auth/change-password` không có Bearer token hoặc token hết hạn,
**Then** trả về HTTP 401 (handled tự động bởi Spring Security — `anyRequest().authenticated()`).

**AC-5** — SecurityConfig:
**Given** `SecurityConfig` hiện tại có `anyRequest().authenticated()`,
**Then** `POST /api/v1/auth/change-password` tự động yêu cầu authenticated — **KHÔNG cần thêm explicit rule vào SecurityConfig**.

### Frontend

**AC-6** — Profile page:
**Given** Customer navigate tới `/profile`,
**When** page load,
**Then** hiện form "Đổi mật khẩu" với 3 fields: Current Password, New Password, Confirm New Password.

**AC-7** — Client-side confirm validation:
**Given** Customer submit form với Confirm New Password không match New Password,
**Then** hiện lỗi "Mật khẩu xác nhận không khớp." Không gọi Server Action.

**AC-8** — Loading state:
**Given** form hợp lệ được submit,
**When** Server Action `changePasswordAction` đang gọi BE,
**Then** button đổi thành "Đang xử lý…" và disabled.

**AC-9** — Success flow:
**Given** BE trả về HTTP 200,
**When** Server Action nhận response,
**Then** toast success "Mật khẩu đã được thay đổi. Vui lòng đăng nhập lại." Sau đó gọi `logoutAction()` để clear cookies và redirect `/login`.

**AC-10** — Wrong password error:
**Given** BE trả về 400 với `error === 'WRONG_PASSWORD'`,
**Then** hiện lỗi "Mật khẩu hiện tại không đúng." trên field Current Password.

**AC-11** — Weak password error:
**Given** BE trả về 400 với `error === 'VALIDATION_FAILED'` và `details.newPassword`,
**Then** hiện lỗi validation từ `details.newPassword` trên field New Password.

---

## Tasks / Subtasks

### Backend Tasks

- [ ] **Task 1: Add `changePassword(PasswordHash newHash)` method to `User` entity**
  - File: `tba-project-agentic-be/core/src/main/java/com/tba/agentic/domain/entity/user/User.java`
  - Thêm method sau `activate()`:
    ```java
    public void changePassword(PasswordHash newHash) {
        this.account = new Account(account.email(), newHash, account.role(),
                account.isActive(), account.isEmailVerified());
    }
    ```
  - Import cần thêm: `com.tba.agentic.domain.value.user.PasswordHash` (đã có trong file)

- [ ] **Task 2: Add `WRONG_PASSWORD` type to `AuthenticationException`**
  - File: `tba-project-agentic-be/core/src/main/java/com/tba/agentic/domain/exception/AuthenticationException.java`
  - Thêm `WRONG_PASSWORD` vào enum `Type`:
    ```java
    public enum Type {
        ACCOUNT_NOT_FOUND,
        INVALID_CREDENTIALS,
        TOO_MANY_REQUESTS,
        EMAIL_NOT_VERIFIED,
        TOKEN_EXPIRED,
        INVALID_TOKEN,
        WRONG_PASSWORD  // NEW
    }
    ```

- [ ] **Task 3: Update `GlobalExceptionHandler` — handle WRONG_PASSWORD → 400**
  - File: `tba-project-agentic-be/application/src/main/java/com/tba/agentic/config/GlobalExceptionHandler.java`
  - Update method `handleDomainAuthentication` — thêm if-check WRONG_PASSWORD:
    ```java
    @ExceptionHandler(com.tba.agentic.domain.exception.AuthenticationException.class)
    public ResponseEntity<ObjectNode> handleDomainAuthentication(
        com.tba.agentic.domain.exception.AuthenticationException ex,
        HttpServletRequest request) {
      if (ex.getType() == com.tba.agentic.domain.exception.AuthenticationException.Type.WRONG_PASSWORD) {
        return buildErrorResponse("WRONG_PASSWORD", ex.getMessage(), HttpStatus.BAD_REQUEST, request);
      }
      return buildErrorResponse("AUTHENTICATION_FAILED", ex.getMessage(),
          HttpStatus.UNAUTHORIZED, request);
    }
    ```

- [ ] **Task 4: Create `ChangePasswordUseCase` interface**
  - File: `tba-project-agentic-be/core/src/main/java/com/tba/agentic/port/in/ChangePasswordUseCase.java`
  - Content:
    ```java
    package com.tba.agentic.port.in;

    public interface ChangePasswordUseCase {
        void execute(Long userId, String oldPassword, String newPassword);
    }
    ```

- [ ] **Task 5: Create `ChangePasswordService`**
  - File: `tba-project-agentic-be/application/src/main/java/com/tba/agentic/application/service/ChangePasswordService.java`
  - Content:
    ```java
    package com.tba.agentic.application.service;

    import com.tba.agentic.domain.entity.user.User;
    import com.tba.agentic.domain.exception.AuthenticationException;
    import com.tba.agentic.domain.value.user.PasswordHash;
    import com.tba.agentic.domain.value.user.UserId;
    import com.tba.agentic.port.in.ChangePasswordUseCase;
    import com.tba.agentic.port.out.PasswordHasher;
    import com.tba.agentic.port.out.TokenRepository;
    import com.tba.agentic.port.out.UserRepository;
    import lombok.RequiredArgsConstructor;
    import org.springframework.transaction.annotation.Transactional;

    @RequiredArgsConstructor
    @Transactional
    public class ChangePasswordService implements ChangePasswordUseCase {

        private final UserRepository userRepository;
        private final PasswordHasher passwordHasher;
        private final TokenRepository tokenRepository;

        @Override
        public void execute(Long userId, String oldPassword, String newPassword) {
            User user = userRepository.findById(new UserId(userId))
                    .orElseThrow(() -> new AuthenticationException(
                            AuthenticationException.Type.ACCOUNT_NOT_FOUND, "User not found"));

            boolean matches = passwordHasher.matches(
                    oldPassword, user.getAccount().passwordHash().value());
            if (!matches) {
                throw new AuthenticationException(
                        AuthenticationException.Type.WRONG_PASSWORD,
                        "Current password is incorrect");
            }

            String newHash = passwordHasher.hash(newPassword);
            user.changePassword(new PasswordHash(newHash));
            userRepository.save(user);

            tokenRepository.revokeAllForUser(userId);
        }
    }
    ```

- [ ] **Task 6: Create `ChangePasswordRequest` DTO**
  - File: `tba-project-agentic-be/application/src/main/java/com/tba/agentic/adapter/transfer/request/ChangePasswordRequest.java`
  - Content:
    ```java
    package com.tba.agentic.adapter.transfer.request;

    import jakarta.validation.constraints.NotBlank;
    import jakarta.validation.constraints.Pattern;
    import jakarta.validation.constraints.Size;

    public record ChangePasswordRequest(
            @NotBlank String oldPassword,
            @NotBlank
            @Size(min = 8, message = "Password must be at least 8 characters")
            @Pattern(
                    regexp = "^(?=.*[A-Z])(?=.*[0-9]).+$",
                    message = "Password must contain at least one uppercase letter and one number")
            String newPassword
    ) {}
    ```

- [ ] **Task 7: Add `changePassword` endpoint to `AuthController`**
  - File: `tba-project-agentic-be/application/src/main/java/com/tba/agentic/adapter/controller/AuthController.java`
  - Thêm field `private final ChangePasswordUseCase changePasswordUseCase;` (Lombok `@RequiredArgsConstructor` inject)
  - Thêm import:
    ```java
    import com.tba.agentic.adapter.transfer.request.ChangePasswordRequest;
    import com.tba.agentic.port.in.ChangePasswordUseCase;
    import org.springframework.security.core.annotation.AuthenticationPrincipal;
    ```
  - Thêm endpoint method:
    ```java
    @PostMapping("/change-password")
    public ResponseEntity<Void> changePassword(
            @Valid @RequestBody ChangePasswordRequest req,
            @AuthenticationPrincipal Long userId) {
        changePasswordUseCase.execute(userId, req.oldPassword(), req.newPassword());
        return ResponseEntity.ok().build();
    }
    ```
  - **QUAN TRỌNG**: `@AuthenticationPrincipal Long userId` hoạt động vì `JwtAuthFilter` set principal là Long userId từ JWT subject — xem pattern đã dùng trong `CustomerProjectController`

- [ ] **Task 8: Register `changePasswordUseCase` bean trong `UseCaseConfig`**
  - File: `tba-project-agentic-be/application/src/main/java/com/tba/agentic/config/UseCaseConfig.java`
  - Thêm import:
    ```java
    import com.tba.agentic.application.service.ChangePasswordService;
    import com.tba.agentic.port.in.ChangePasswordUseCase;
    ```
  - Thêm bean method:
    ```java
    @Bean
    public ChangePasswordUseCase changePasswordUseCase(
            UserRepository users,
            PasswordHasher hasher,
            TokenRepository tokens) {
        return new ChangePasswordService(users, hasher, tokens);
    }
    ```

- [ ] **Task 9: Build verification**
  - `cd tba-project-agentic-be && make compile` → BUILD SUCCESSFUL

### Frontend Tasks

- [ ] **Task 10: Create `changePasswordAction` Server Action**
  - File: `tba-project-agentic-fe/src/app/[locale]/(dashboard)/profile/actions.ts`
  - Content:
    ```typescript
    'use server'
    import { cookies } from 'next/headers'

    export async function changePasswordAction(data: {
      oldPassword: string
      newPassword: string
    }) {
      try {
        const cookieStore = await cookies()
        const cookieHeader = cookieStore.toString()

        const res = await fetch(
          `${process.env.API_BASE_URL}/api/v1/auth/change-password`,
          {
            method: 'POST',
            headers: {
              'Content-Type': 'application/json',
              Cookie: cookieHeader,
            },
            body: JSON.stringify(data),
          }
        )

        if (!res.ok) {
          const body = await res.json().catch(() => ({}))
          return { error: body.error ?? 'UNKNOWN', details: body.details ?? {} }
        }

        return { success: true }
      } catch {
        return { error: 'CONNECTION_ERROR', details: {} }
      }
    }
    ```

- [ ] **Task 11: Create `ChangePasswordForm` component**
  - File: `tba-project-agentic-fe/src/app/[locale]/(dashboard)/profile/_components/ChangePasswordForm.tsx`
  - Content:
    ```typescript
    'use client'
    import { useState } from 'react'
    import { Button } from '@/components/ui/button'
    import { Input } from '@/components/ui/input'
    import { Label } from '@/components/ui/label'
    import { changePasswordAction } from '../actions'
    import { logoutAction } from '@/actions/auth'
    import { toast } from 'sonner'

    export default function ChangePasswordForm() {
      const [oldPassword, setOldPassword] = useState('')
      const [newPassword, setNewPassword] = useState('')
      const [confirmPassword, setConfirmPassword] = useState('')
      const [loading, setLoading] = useState(false)
      const [errors, setErrors] = useState<{
        oldPassword?: string
        newPassword?: string
        confirm?: string
      }>({})

      async function handleSubmit(e: React.FormEvent) {
        e.preventDefault()
        setErrors({})

        if (newPassword !== confirmPassword) {
          setErrors({ confirm: 'Mật khẩu xác nhận không khớp.' })
          return
        }

        setLoading(true)
        try {
          const result = await changePasswordAction({ oldPassword, newPassword })

          if ('error' in result) {
            if (result.error === 'WRONG_PASSWORD') {
              setErrors({ oldPassword: 'Mật khẩu hiện tại không đúng.' })
            } else if (result.error === 'VALIDATION_FAILED' && result.details?.newPassword) {
              setErrors({ newPassword: result.details.newPassword })
            } else if (result.error === 'CONNECTION_ERROR') {
              toast.error('Lỗi kết nối. Vui lòng thử lại.')
            } else {
              toast.error('Đổi mật khẩu thất bại. Vui lòng thử lại.')
            }
            return
          }

          toast.success('Mật khẩu đã được thay đổi. Vui lòng đăng nhập lại.')
          await logoutAction()
        } finally {
          setLoading(false)
        }
      }

      return (
        <form onSubmit={handleSubmit} className="space-y-4 max-w-md">
          <div className="space-y-2">
            <Label htmlFor="oldPassword">Mật khẩu hiện tại</Label>
            <Input
              id="oldPassword"
              type="password"
              value={oldPassword}
              onChange={(e) => setOldPassword(e.target.value)}
              required
            />
            {errors.oldPassword && (
              <p className="text-sm text-red-600">{errors.oldPassword}</p>
            )}
          </div>

          <div className="space-y-2">
            <Label htmlFor="newPassword">Mật khẩu mới</Label>
            <Input
              id="newPassword"
              type="password"
              value={newPassword}
              onChange={(e) => setNewPassword(e.target.value)}
              required
            />
            {errors.newPassword && (
              <p className="text-sm text-red-600">{errors.newPassword}</p>
            )}
          </div>

          <div className="space-y-2">
            <Label htmlFor="confirmPassword">Xác nhận mật khẩu mới</Label>
            <Input
              id="confirmPassword"
              type="password"
              value={confirmPassword}
              onChange={(e) => setConfirmPassword(e.target.value)}
              required
            />
            {errors.confirm && (
              <p className="text-sm text-red-600">{errors.confirm}</p>
            )}
          </div>

          <Button type="submit" disabled={loading}>
            {loading ? 'Đang xử lý…' : 'Đổi mật khẩu'}
          </Button>
        </form>
      )
    }
    ```
  - **Lưu ý**: `toast` từ `sonner` — xem các file FE hiện tại để xác nhận package đã cài. Nếu chưa có `sonner`, dùng `alert()` hoặc state để hiện message.

- [ ] **Task 12: Create `/profile` page**
  - File: `tba-project-agentic-fe/src/app/[locale]/(dashboard)/profile/page.tsx`
  - Content:
    ```typescript
    import ChangePasswordForm from './_components/ChangePasswordForm'

    export default function ProfilePage() {
      return (
        <div className="max-w-2xl">
          <h1 className="text-2xl font-semibold mb-6">Hồ sơ</h1>
          <div className="bg-white rounded-lg border p-6">
            <h2 className="text-lg font-medium mb-4">Đổi mật khẩu</h2>
            <ChangePasswordForm />
          </div>
        </div>
      )
    }
    ```

- [ ] **Task 13: Verify `sonner` toast library**
  - Check `tba-project-agentic-fe/package.json` xem `sonner` đã có chưa
  - Nếu chưa: thay thế `toast.success/error` bằng `alert()` hoặc tạo simple state message
  - Nếu có: import `import { toast } from 'sonner'` và đảm bảo `<Toaster />` có trong root layout

- [ ] **Task 14: Add avatar link to `/profile` in dashboard layout**
  - File: `tba-project-agentic-fe/src/app/[locale]/(dashboard)/layout.tsx`
  - Bọc component `<UserAvatar />` trong `<Link href="/profile">` để user click vào avatar navigate tới trang đổi mật khẩu:
    ```tsx
    import Link from 'next/link'

    // Trong phần render của DashboardLayout, thay:
    {profile && (
      <UserAvatar displayName={profile.displayName} email={profile.email} />
    )}
    // Thành:
    {profile && (
      <Link href="/profile" title="Hồ sơ / Đổi mật khẩu">
        <UserAvatar displayName={profile.displayName} email={profile.email} />
      </Link>
    )}
    ```
  - `Link` đã được import sẵn trong file — không cần thêm import mới
  - Avatar sẽ có `cursor-pointer` behavior tự nhiên từ `<a>` tag của Next.js Link

### Verification

- [ ] **Task 15: Manual test flow BE**
  - Login → lấy access token cookie
  - `POST /api/v1/auth/change-password` với đúng `oldPassword` và `newPassword` mạnh → HTTP 200
  - Kiểm tra refresh tokens bị revoke: thử dùng refresh token cũ → thất bại
  - `POST /api/v1/auth/change-password` với sai `oldPassword` → HTTP 400 `WRONG_PASSWORD`
  - `POST /api/v1/auth/change-password` với `newPassword` yếu (< 8 ký tự) → HTTP 400 `VALIDATION_FAILED`
  - `POST /api/v1/auth/change-password` không có token → HTTP 401

- [ ] **Task 16: Manual test flow FE**
  - Click vào avatar ở góc phải header → navigate tới `/profile`
  - Navigate tới `/profile` khi đã login
  - Submit với confirm không match → lỗi client-side, không gọi API
  - Submit với sai current password → lỗi "Mật khẩu hiện tại không đúng."
  - Submit thành công → toast + redirect `/login`

---

## Dev Notes

### CRITICAL: Module Structure

Project BE có **2 layers khác nhau** trong `tba-project-agentic-be/`:

```
tba-project-agentic-be/
  core/src/main/java/com/tba/agentic/
    domain/entity/user/User.java           ← EDIT: add changePassword()
    domain/entity/user/Account.java        ← Record, immutable — tạo instance mới trong changePassword()
    domain/exception/AuthenticationException.java ← EDIT: add WRONG_PASSWORD type
    domain/value/user/PasswordHash.java    ← record PasswordHash(String value) {}
    domain/value/user/UserId.java          ← record UserId(Long value) {}
    port/in/ChangePasswordUseCase.java     ← CREATE NEW
    port/out/UserRepository.java           ← HAS findById(UserId), save(User)
    port/out/PasswordHasher.java           ← HAS matches(raw, hash), hash(raw)
    port/out/TokenRepository.java          ← HAS revokeAllForUser(Long userId) — đây là TokenRepository, KHÔNG phải RefreshTokenRepository!

  application/src/main/java/com/tba/agentic/
    application/service/ChangePasswordService.java  ← CREATE NEW
    adapter/controller/AuthController.java          ← EDIT: add endpoint + field
    adapter/transfer/request/ChangePasswordRequest.java ← CREATE NEW
    config/UseCaseConfig.java               ← EDIT: add changePasswordUseCase bean
    config/GlobalExceptionHandler.java      ← EDIT: add WRONG_PASSWORD → 400 case
```

### CRITICAL: TokenRepository vs RefreshTokenRepository

Dù có cả `TokenRepository` và `RefreshTokenRepository` trong `port/out/`, AuthController và UseCaseConfig đều dùng **`TokenRepository`**. Đây là interface đúng:

```java
public interface TokenRepository {
  void save(Long userId, String token, Instant expiresAt);
  Optional<RefreshTokenData> findByToken(String token);
  void revoke(String token);
  void revokeAllForUser(Long userId);  // ← dùng method này để revoke tất cả refresh tokens
}
```

**Dùng `revokeAllForUser(Long userId)`** — nhận `Long` (không phải domain object).

### CRITICAL: @AuthenticationPrincipal Long userId

Pattern đã established trong `CustomerProjectController`:
```java
@AuthenticationPrincipal Long userId
```

JWT filter set principal là `Long userId` (từ JWT subject claim). AuthController endpoint cần inject nó cùng cách:
```java
@PostMapping("/change-password")
public ResponseEntity<Void> changePassword(
        @Valid @RequestBody ChangePasswordRequest req,
        @AuthenticationPrincipal Long userId) {
    changePasswordUseCase.execute(userId, req.oldPassword(), req.newPassword());
    return ResponseEntity.ok().build();
}
```

Thêm import: `import org.springframework.security.core.annotation.AuthenticationPrincipal;`

### Account record — how to change password

`Account` là immutable record. Method `changePassword()` trong `User` tạo `Account` mới:

```java
// Account record definition:
public record Account(
    Email email, PasswordHash passwordHash, Role role, boolean isActive, boolean isEmailVerified) {}

// User.changePassword() - same pattern as activate():
public void changePassword(PasswordHash newHash) {
    this.account = new Account(account.email(), newHash, account.role(),
            account.isActive(), account.isEmailVerified());
}
```

### PasswordHasher interface

```java
public interface PasswordHasher {
  String hash(String rawPassword);          // trả về String (BCrypt hash)
  boolean matches(String rawPassword, String hashedPassword);
}
```

Trong `ChangePasswordService`:
```java
boolean matches = passwordHasher.matches(oldPassword, user.getAccount().passwordHash().value());
// passwordHash() là PasswordHash record — lấy String value bằng .value()
```

### SecurityConfig — KHÔNG cần thay đổi

`SecurityConfig` hiện tại:
```java
.requestMatchers(HttpMethod.POST, "/api/v1/auth/login").permitAll()
.requestMatchers(HttpMethod.POST, "/api/v1/auth/register").permitAll()
// ... other permitAll endpoints ...
.anyRequest().authenticated()  // ← covers /api/v1/auth/change-password tự động
```

`POST /api/v1/auth/change-password` **không** cần explicit rule — `anyRequest().authenticated()` đã bao phủ.

### FE: logoutAction sau khi đổi mật khẩu

Sau khi đổi mật khẩu thành công, cần logout user. Import từ `@/actions/auth`:

```typescript
import { logoutAction } from '@/actions/auth'
// Sau toast.success():
await logoutAction()  // clears cookies, redirects to /login
```

`logoutAction` gọi BE logout endpoint, xóa cookies, và redirect `/login` via next/navigation.

### FE: Sonner toast

Kiểm tra `package.json` trước:
```bash
grep sonner tba-project-agentic-fe/package.json
```
- **Nếu có**: dùng `import { toast } from 'sonner'`
- **Nếu không có**: thay bằng state variable `const [successMsg, setSuccessMsg] = useState('')` và render `<p className="text-green-600">{successMsg}</p>`

### Password validation rules (đồng bộ FE và BE)

| Rule | BE (Jakarta Validation) | FE (client-side) |
|------|------------------------|------------------|
| ≥ 8 ký tự | `@Size(min = 8)` | `newPassword.length >= 8` |
| ≥ 1 uppercase | `@Pattern(regexp = "^(?=.*[A-Z]).*$")` | `/[A-Z]/.test(newPassword)` |
| ≥ 1 số | `@Pattern(regexp = "^(?=.*[0-9]).*$")` | `/[0-9]/.test(newPassword)` |

FE validation trong `ChangePasswordForm` có thể thêm client-side check trước khi gọi Server Action để improve UX.

### GlobalExceptionHandler — WRONG_PASSWORD mapping

```
AuthenticationException.Type.WRONG_PASSWORD → HTTP 400, error: "WRONG_PASSWORD"
AuthenticationException.Type.* (others)     → HTTP 401, error: "AUTHENTICATION_FAILED"
```

Pattern chỉnh sửa `handleDomainAuthentication`:
```java
@ExceptionHandler(com.tba.agentic.domain.exception.AuthenticationException.class)
public ResponseEntity<ObjectNode> handleDomainAuthentication(
    com.tba.agentic.domain.exception.AuthenticationException ex,
    HttpServletRequest request) {
  if (ex.getType() == com.tba.agentic.domain.exception.AuthenticationException.Type.WRONG_PASSWORD) {
    return buildErrorResponse("WRONG_PASSWORD", ex.getMessage(), HttpStatus.BAD_REQUEST, request);
  }
  return buildErrorResponse("AUTHENTICATION_FAILED", ex.getMessage(),
      HttpStatus.UNAUTHORIZED, request);
}
```

---

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Completion Notes List

- `TokenRepository.revokeAllForUser(Long userId)` đã tồn tại — không cần thêm method
- `SecurityConfig.anyRequest().authenticated()` tự động bảo vệ `/api/v1/auth/change-password` — không cần explicit rule
- `@AuthenticationPrincipal Long userId` là pattern established trong CustomerProjectController — reuse nguyên vẹn
- `Account` là immutable record — changePassword() tạo Account mới (same pattern as activate())
- Không có `profile` page trong FE hiện tại — tạo mới tại `(dashboard)/profile/`
- `logoutAction` từ `@/actions/auth` là đúng cách logout (không có NextAuth trong project)

### File List

**CREATE (BE):**
- `tba-project-agentic-be/core/src/main/java/com/tba/agentic/port/in/ChangePasswordUseCase.java`
- `tba-project-agentic-be/application/src/main/java/com/tba/agentic/application/service/ChangePasswordService.java`
- `tba-project-agentic-be/application/src/main/java/com/tba/agentic/adapter/transfer/request/ChangePasswordRequest.java`

**UPDATE (BE):**
- `tba-project-agentic-be/core/src/main/java/com/tba/agentic/domain/entity/user/User.java` (add changePassword method)
- `tba-project-agentic-be/core/src/main/java/com/tba/agentic/domain/exception/AuthenticationException.java` (add WRONG_PASSWORD type)
- `tba-project-agentic-be/application/src/main/java/com/tba/agentic/adapter/controller/AuthController.java` (add field + endpoint)
- `tba-project-agentic-be/application/src/main/java/com/tba/agentic/config/UseCaseConfig.java` (add changePasswordUseCase bean)
- `tba-project-agentic-be/application/src/main/java/com/tba/agentic/config/GlobalExceptionHandler.java` (add WRONG_PASSWORD → 400)

**CREATE (FE):**
- `tba-project-agentic-fe/src/app/[locale]/(dashboard)/profile/page.tsx`
- `tba-project-agentic-fe/src/app/[locale]/(dashboard)/profile/actions.ts`
- `tba-project-agentic-fe/src/app/[locale]/(dashboard)/profile/_components/ChangePasswordForm.tsx`
