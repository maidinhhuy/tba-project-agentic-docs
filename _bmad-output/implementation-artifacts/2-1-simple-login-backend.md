# Story 2.1: Simple JWT Login Backend

Status: ready-for-dev

> **Scope note:** Simplified version — skip login attempt rate limiting, skip refresh token flow.
> Goal: validate credentials → generate access token → return to client.
> Full Story 2.1 AC (rate limiting, refresh, logout) deferred to a later story.

## Story

As a registered customer or admin,
I want to login with email and password via `POST /api/v1/auth/login`,
so that I receive a JWT access token to authenticate subsequent API requests.

## Acceptance Criteria

1. **Given** `POST /api/v1/auth/login` với body `{ "email": "user@example.com", "password": "password123" }` và user `is_active=true`,
   **When** email tồn tại và password đúng,
   **Then** trả về HTTP 200 với body `{ "token": "<jwt>", "expiresIn": 900 }`.

2. **Given** `POST /api/v1/auth/login` với password sai,
   **When** gửi request,
   **Then** trả về HTTP 401 với body `{ "error": "AUTHENTICATION_FAILED", "message": "...", "status": 401, ... }`.
   **And** response KHÔNG tiết lộ email có tồn tại hay không (cùng error code cho cả không tìm thấy user lẫn sai password).

3. **Given** `POST /api/v1/auth/login` với email không tồn tại trong DB,
   **When** gửi request,
   **Then** trả về HTTP 401 với body `{ "error": "AUTHENTICATION_FAILED", ... }` — cùng message như AC #2.

4. **Given** user có `is_active=false`,
   **When** login,
   **Then** trả về HTTP 401 với `{ "error": "AUTHENTICATION_FAILED", ... }`.

5. **And** endpoint `POST /api/v1/auth/login` đã được `permitAll()` trong `SecurityConfig` — KHÔNG cần thay đổi.

## Tasks / Subtasks

<!-- ════════════════════════════════════════════════════════
     TASK 1: DOMAIN DEFINITION — chỉ files trong core/
     ════════════════════════════════════════════════════════ -->

- [ ] Task 1: Domain Definition — core module (AC: #1, #2, #3, #4)

  <!-- VOs: tất cả đã tồn tại, không cần tạo -->
  <!-- Email, Password, PasswordHash, Role, UserId — đã có trong domain/value/user/ -->

  <!-- Bound Command: cần tạo mới -->
  - [ ] Bound — Command: `LoginUserCommand`
    - File: `core/src/main/java/com/tba/agentic/port/bound/command/user/LoginUserCommand.java`
    - Java record với fields: `Email email`, `Password password`
    - Dùng VO domain — không dùng String thô

  <!-- Inbound Port: cần tạo mới -->
  - [ ] Inbound Port: `LoginUserUseCase`
    - File: `core/src/main/java/com/tba/agentic/port/in/LoginUserUseCase.java`
    - Method: `User execute(LoginUserCommand cmd)`
    - Throws: `AuthenticationException(AUTHENTICATION_FAILED)` nếu credentials sai hoặc user không active

  <!-- Không cần VOs mới, Exceptions mới, hay Outbound Ports mới -->
  <!-- AuthenticationException đã có trong domain/exception/ -->
  <!-- UserRepository.findByEmail(Email) đã có trong port/out/ -->

  - [ ] Verify: `./gradlew :core:compileJava` → BUILD SUCCESSFUL

<!-- ════════════════════════════════════════════════════════
     TASK 2: IMPLEMENTATION — chỉ files trong application/
     Depends On: Task 1 — Domain Definition
     ════════════════════════════════════════════════════════ -->

- [ ] Task 2: Implementation — application module (AC: #1, #2, #3, #4)
  **Depends On: Task 1 — Domain Definition**

  <!-- Không có DB Migration — không đổi schema -->

  <!-- Service: implements LoginUserUseCase từ Task 1 -->
  - [ ] Service: `LoginUserService implements LoginUserUseCase`
    - File: `application/src/main/java/com/tba/agentic/application/service/LoginUserService.java`
    - Logic: (1) `userRepository.findByEmail(cmd.email())` → nếu empty → throw `AuthenticationException(AUTHENTICATION_FAILED)`; (2) `passwordEncoder.matches(cmd.password().value(), user.getAccount().getPasswordHash().value())` → nếu false → throw `AuthenticationException(AUTHENTICATION_FAILED)`; (3) nếu `!user.getAccount().isActive()` → throw `AuthenticationException(AUTHENTICATION_FAILED)`; (4) return user
    - Inject: `UserRepository`, `PasswordEncoder` via constructor (`@RequiredArgsConstructor`)

  <!-- Response DTO — chỉ dùng primitive types: String, long -->
  - [ ] Response DTO: `LoginResponse`
    - File: `application/src/main/java/com/tba/agentic/adapter/transfer/response/LoginResponse.java`
    - Java record: `String token`, `long expiresIn`

  <!-- Controller: thêm endpoint /login vào AuthController -->
  - [ ] Controller: thêm `POST /login` vào `AuthController`
    - File: `application/src/main/java/com/tba/agentic/adapter/controller/AuthController.java`
    - Inject thêm: `LoginUserUseCase loginUserUseCase`, `JwtTokenProvider jwtTokenProvider`, `JwtProperties jwtProperties`
    - Method: `public ResponseEntity<LoginResponse> login(@Valid @RequestBody LoginRequest req)`
    - Logic: (1) call `loginUserUseCase.execute(new LoginUserCommand(req.getEmail(), req.getPassword()))`; (2) generate token: `jwtTokenProvider.generateAccessToken(user.getUserId().value(), user.getAccount().getEmail().value(), user.getAccount().getRole().name())`; (3) return `ResponseEntity.ok(new LoginResponse(token, jwtProperties.getAccessTokenExpiry()))`
    - KHÔNG tự catch exception — để `GlobalExceptionHandler` xử lý

  <!-- GlobalExceptionHandler: thêm handler cho domain AuthenticationException -->
  - [ ] GlobalExceptionHandler: thêm handler cho `com.tba.agentic.domain.exception.AuthenticationException`
    - File: `application/src/main/java/com/tba/agentic/config/GlobalExceptionHandler.java`
    - Thêm import: `com.tba.agentic.domain.exception.AuthenticationException` (alias với `DomainAuthException`)
    - Thêm method: `@ExceptionHandler(com.tba.agentic.domain.exception.AuthenticationException.class)` → 401 với error code `"AUTHENTICATION_FAILED"`
    - **Lưu ý**: handler này phải được đặt TRƯỚC handler cho `org.springframework.security.core.AuthenticationException` để không bị shadow. Dùng FQN để phân biệt 2 exception cùng tên.

  <!-- SecurityConfig: đã có permitAll cho /api/v1/auth/login, KHÔNG sửa -->

  - [ ] Verify: `./gradlew :application:compileJava` → BUILD SUCCESSFUL
  - [ ] API curl test (xem Dev Notes — phần API Test)

## Dev Notes

### Codebase State

**Files đã tồn tại — đừng tạo lại:**

| File | Trạng thái | Ghi chú |
|------|-----------|---------|
| `core/.../domain/value/user/Email.java` | Tồn tại | VO validate format email |
| `core/.../domain/value/user/Password.java` | Tồn tại | VO raw password (không validate min length) |
| `core/.../domain/value/user/PasswordHash.java` | Tồn tại | VO BCrypt hash |
| `core/.../domain/value/user/Role.java` | Tồn tại | Enum CUSTOMER/ADMIN |
| `core/.../domain/value/user/UserId.java` | Tồn tại | VO user ID |
| `core/.../domain/exception/AuthenticationException.java` | Tồn tại | Có Type enum: ACCOUNT_NOT_FOUND, INVALID_CREDENTIALS, TOO_MANY_REQUESTS, ... |
| `core/.../port/out/UserRepository.java` | Tồn tại | `findByEmail(Email)`, `findById(UserId)` |
| `application/.../adapter/transfer/request/LoginRequest.java` | Tồn tại | Có `getEmail(): Email`, `getPassword(): Password` |
| `application/.../security/JwtTokenProvider.java` | Tồn tại | `generateAccessToken(Long userId, String email, String role)` |
| `application/.../security/JwtProperties.java` | Tồn tại | `getAccessTokenExpiry()` = 900 |
| `application/.../config/SecurityConfig.java` | Tồn tại | Đã `permitAll()` cho POST `/api/v1/auth/login` — KHÔNG sửa |
| `application/.../config/GlobalExceptionHandler.java` | Tồn tại | Cần thêm handler cho domain AuthenticationException |
| `application/.../adapter/controller/AuthController.java` | Tồn tại | Đang có POST /register, cần thêm POST /login |

**Files cần tạo mới:**
- `core/.../port/bound/command/user/LoginUserCommand.java`
- `core/.../port/in/LoginUserUseCase.java`
- `application/.../application/service/LoginUserService.java`
- `application/.../adapter/transfer/response/LoginResponse.java`

### Critical: Naming Conflict — AuthenticationException

Codebase có **2 class cùng tên** `AuthenticationException`:

| FQN | Source | Mục đích |
|-----|--------|---------|
| `com.tba.agentic.domain.exception.AuthenticationException` | domain | Business auth failure |
| `org.springframework.security.core.AuthenticationException` | Spring Security | Framework auth filter |

`GlobalExceptionHandler` hiện tại đã import Spring's `AuthenticationException`. Khi thêm handler mới:
- Dùng **FQN** cho domain exception: `@ExceptionHandler(com.tba.agentic.domain.exception.AuthenticationException.class)`
- Đặt method này TRÊN method `handleAuthentication(AuthenticationException ex, ...)` để Spring ưu tiên đúng handler

### Domain Design: LoginUserService logic

```java
@Service
@RequiredArgsConstructor
public class LoginUserService implements LoginUserUseCase {
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    @Override
    public User execute(LoginUserCommand cmd) {
        User user = userRepository.findByEmail(cmd.email())
            .orElseThrow(() -> new com.tba.agentic.domain.exception.AuthenticationException(
                com.tba.agentic.domain.exception.AuthenticationException.Type.INVALID_CREDENTIALS,
                "Invalid credentials"));

        if (!passwordEncoder.matches(cmd.password().value(),
                user.getAccount().getPasswordHash().value())) {
            throw new com.tba.agentic.domain.exception.AuthenticationException(
                com.tba.agentic.domain.exception.AuthenticationException.Type.INVALID_CREDENTIALS,
                "Invalid credentials");
        }

        if (!user.getAccount().isActive()) {
            throw new com.tba.agentic.domain.exception.AuthenticationException(
                com.tba.agentic.domain.exception.AuthenticationException.Type.INVALID_CREDENTIALS,
                "Invalid credentials");
        }

        return user;
    }
}
```

> **Security note**: Cùng message "Invalid credentials" cho tất cả fail case — không tiết lộ email có tồn tại hay không (AC #2, #3).

### Pattern AuthController — thêm /login

```java
// Thêm vào constructor fields (Lombok @RequiredArgsConstructor tự generate):
private final LoginUserUseCase loginUserUseCase;
private final JwtTokenProvider jwtTokenProvider;
private final JwtProperties jwtProperties;

@PostMapping("/login")
public ResponseEntity<LoginResponse> login(@Valid @RequestBody LoginRequest req) {
    User user = loginUserUseCase.execute(
        new LoginUserCommand(req.getEmail(), req.getPassword()));
    String token = jwtTokenProvider.generateAccessToken(
        user.getUserId().value(),
        user.getAccount().getEmail().value(),
        user.getAccount().getRole().name());
    return ResponseEntity.ok(new LoginResponse(token, jwtProperties.getAccessTokenExpiry()));
}
```

### Pattern GlobalExceptionHandler — thêm handler

```java
// Thêm TRƯỚC handleAuthentication(AuthenticationException ...) hiện tại
@ExceptionHandler(com.tba.agentic.domain.exception.AuthenticationException.class)
public ResponseEntity<ObjectNode> handleDomainAuthentication(
    com.tba.agentic.domain.exception.AuthenticationException ex,
    HttpServletRequest request) {
    return buildErrorResponse("AUTHENTICATION_FAILED", ex.getMessage(),
        HttpStatus.UNAUTHORIZED, request);
}
```

### User aggregate — cách truy cập fields

Từ `User` entity, truy cập:
```java
user.getUserId()              // → UserId
user.getUserId().value()      // → Long (id)
user.getAccount().getEmail()             // → Email VO
user.getAccount().getEmail().value()     // → String
user.getAccount().getPasswordHash()      // → PasswordHash VO
user.getAccount().getPasswordHash().value() // → String (BCrypt hash)
user.getAccount().getRole()              // → Role enum
user.getAccount().getRole().name()       // → "CUSTOMER" / "ADMIN"
user.getAccount().isActive()             // → boolean
```

### API Test

```bash
# AC #1 — Happy path: login thành công
curl -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123"}' \
  -v
# Expected: 200 OK, body: {"token":"eyJ...","expiresIn":900}

# AC #2 — Sai password
curl -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"wrongpassword"}' \
  -v
# Expected: 401, body: {"error":"AUTHENTICATION_FAILED","status":401,...}

# AC #3 — Email không tồn tại
curl -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"notexist@example.com","password":"password123"}' \
  -v
# Expected: 401, body: {"error":"AUTHENTICATION_FAILED","status":401,...}
# Same error message như AC #2 — không tiết lộ email tồn tại

# Verify token hoạt động — dùng token từ AC #1
curl -X GET http://localhost:8080/api/v1/customer/projects \
  -H "Authorization: Bearer <token_from_login>" \
  -v
# Expected: 200 (nếu đã có project) hoặc 200 với [] (nếu chưa có)
```

### Package paths reference

```
# Core — Task 1 files mới:
core/src/main/java/com/tba/agentic/port/bound/command/user/LoginUserCommand.java
core/src/main/java/com/tba/agentic/port/in/LoginUserUseCase.java

# Application — Task 2 files mới/update:
application/src/main/java/com/tba/agentic/application/service/LoginUserService.java
application/src/main/java/com/tba/agentic/adapter/transfer/response/LoginResponse.java
application/src/main/java/com/tba/agentic/adapter/controller/AuthController.java     (UPDATE)
application/src/main/java/com/tba/agentic/config/GlobalExceptionHandler.java          (UPDATE)
```

### References

- [Source: docs/rule/02_DOMAIN_DEFINITION_GUIDE.md] — domain definition process
- [Source: docs/rule/03_API_DEVELOPMENT_GUIDE.md] — API development rules
- [Source: core/.../domain/exception/AuthenticationException.java] — exception type enum
- [Source: core/.../port/out/UserRepository.java] — findByEmail signature
- [Source: application/.../security/JwtTokenProvider.java] — generateAccessToken signature
- [Source: application/.../adapter/controller/AuthController.java] — pattern endpoint hiện tại
- [Source: application/.../config/GlobalExceptionHandler.java] — buildErrorResponse pattern
- [Source: _bmad-output/planning-artifacts/epics.md — Story 2.1] — original full AC

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

### File List
