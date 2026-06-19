# Story 2.3: Backend Self-Registration

Status: ready-for-dev

## Story

As a new customer,
I want to register for an account with my email, display name and password via `POST /api/v1/auth/register`,
so that I can login immediately without any email verification step.

## Acceptance Criteria

1. **Given** `POST /api/v1/auth/register` với body `{ "email": "user@example.com", "displayName": "Nguyen Van A", "password": "password123" }` hợp lệ,
   **When** email chưa tồn tại trong hệ thống,
   **Then** trả về HTTP 201 (no body). User được lưu với `role=CUSTOMER`, `is_active=true`, `is_email_verified=true`.

2. **Given** `POST /api/v1/auth/register` với email đã tồn tại trong DB,
   **When** gửi request,
   **Then** trả về HTTP 409 với error body `{ "error": "EMAIL_CONFLICT", "message": "...", "status": 409, ... }`.

3. **Given** `POST /api/v1/auth/register` với password < 8 ký tự, hoặc `email` null/blank, hoặc `displayName` null/blank,
   **When** gửi request,
   **Then** trả về HTTP 400 với error body `{ "error": "VALIDATION_FAILED", "message": "...", "status": 400, "details": { "fieldName": "error message" }, ... }`.

4. **Given** đăng ký thành công (HTTP 201),
   **When** customer gọi `POST /api/v1/auth/login` với đúng credentials (email + password),
   **Then** login thành công ngay — trả về HTTP 200 với access/refresh token cookies.

5. **And** endpoint `/api/v1/auth/register` được configure `permitAll()` trong `SecurityConfig` — KHÔNG cần thay đổi, đã có sẵn.

## Tasks / Subtasks

- [ ] Task 1: Tạo `RegisterRequest` DTO (AC: #1, #3)
  - [ ] File: `application/src/main/java/com/tba/agentic/adapter/transfer/request/RegisterRequest.java`
  - [ ] Java record với 3 fields: `email` (String), `password` (String), `displayName` (String)
  - [ ] Annotations: `@NotBlank` trên `email` và `displayName`; `@NotBlank @Size(min = 8)` trên `password`
  - [ ] Import `jakarta.validation.constraints.*`

- [ ] Task 2: Tạo `RegisterUserService` implementing `RegisterUserUseCase` (AC: #1, #2, #4)
  - [ ] File: `application/src/main/java/com/tba/agentic/application/service/RegisterUserService.java`
  - [ ] Annotate `@Service`; inject `UserRepository` và `PasswordEncoder` via constructor
  - [ ] Logic: (1) `userRepository.existsByEmail(email)` → throw `EmailConflictException` nếu true; (2) hash password bằng `passwordEncoder.encode()`; (3) tạo `User.create(email, passwordHash, Role.CUSTOMER, displayName)`; (4) gọi `user.activate()` để set `isActive=true, isEmailVerified=true`; (5) `userRepository.save(user)` rồi return

- [ ] Task 3: Thêm register endpoint vào `AuthController` (AC: #1, #2, #3)
  - [ ] File: `application/src/main/java/com/tba/agentic/adapter/controller/AuthController.java`
  - [ ] Thêm `@PostMapping("/register")` method
  - [ ] Inject `RegisterUserUseCase registerUserUseCase` vào constructor (thêm vào `@RequiredArgsConstructor` class)
  - [ ] Method signature: `public ResponseEntity<Void> register(@Valid @RequestBody RegisterRequest req)`
  - [ ] Gọi `registerUserUseCase.execute(new RegisterUserUseCase.Command(req.email(), req.password(), req.displayName()))`
  - [ ] Return `ResponseEntity.status(HttpStatus.CREATED).build()` khi thành công
  - [ ] KHÔNG tự handle exception — để `GlobalExceptionHandler` xử lý

- [ ] Task 4: Cập nhật `GlobalExceptionHandler` để handle các exception mới (AC: #2, #3)
  - [ ] File: `application/src/main/java/com/tba/agentic/config/GlobalExceptionHandler.java`
  - [ ] Thêm `@ExceptionHandler(EmailConflictException.class)` → 409 với error `"EMAIL_CONFLICT"`
  - [ ] Thêm `@ExceptionHandler(MethodArgumentNotValidException.class)` → 400 với error `"VALIDATION_FAILED"` và `details` map từ `ex.getBindingResult().getFieldErrors()`
  - [ ] Import: `com.tba.agentic.domain.exception.EmailConflictException` và `org.springframework.web.bind.MethodArgumentNotValidException`

- [ ] Task 5: Verify bằng `make compile` và manual curl test (AC: #1, #2, #3, #4)
  - [ ] `./gradlew :application:compileJava` — zero errors
  - [ ] `curl -X POST http://localhost:8080/api/v1/auth/register -H "Content-Type: application/json" -d '{"email":"test@example.com","displayName":"Test User","password":"password123"}' -v` → 201
  - [ ] Gọi lại lần 2 với cùng email → 409
  - [ ] Gọi với password < 8 ký tự → 400 với VALIDATION_FAILED và details
  - [ ] Login với credentials vừa đăng ký → 200 (thành công)

## Dev Notes

### Critical: Trạng thái hiện tại của codebase

**CÁC FILE ĐÃ TỒN TẠI — đừng tạo lại:**

| File | Trạng thái |
|------|-----------|
| `core/src/main/java/.../port/in/RegisterUserUseCase.java` | Tồn tại, interface có `Command` record và `execute(Command)` method |
| `infrastructure/repository/UserRepositoryImpl.java` | Tồn tại, `@Primary`, đã implement ĐẦYĐỦ `save()`, `existsByEmail()`, `findByEmail()` |
| `config/SecurityConfig.java` | Đã có `permitAll()` cho `/api/v1/auth/register` — KHÔNG THAY ĐỔI |
| `domain/exception/EmailConflictException.java` | Tồn tại |
| `domain/user/User.java` | Tồn tại với `User.create()` và `user.activate()` |

**CÓ HAI UserRepository implementations — dùng đúng cái:**
- `infrastructure/repository/UserRepositoryImpl.java` — `@Primary`, đã implement đầy đủ → **DÙNGcái này**
- `infrastructure/repository/account/DatabaseDslUserRepository.java` — KHÔNG có `@Primary`, tất cả methods throw `UnsupportedOperationException` → **KHÔNG dùng**

**AuthController hiện tại:**
- File: `application/src/main/java/com/tba/agentic/adapter/controller/AuthController.java`
- `@RequestMapping("/api/v1/auth")` → endpoint register sẽ là `/api/v1/auth/register`
- Đã có endpoints: `POST /login`, `POST /refresh`, `POST /logout`
- Class dùng `@RequiredArgsConstructor` — thêm field `RegisterUserUseCase registerUserUseCase` để inject
- Class đã có `private final` fields nhiều — inject theo constructor (Lombok sẽ tự tạo)

**Lưu ý URL:** Epic nói `/api/auth/register` nhưng thực tế là `/api/v1/auth/register` (vì `@RequestMapping("/api/v1/auth")`). SecurityConfig đã đúng với `/api/v1/auth/register`.

### Critical: `User.create()` tạo với isActive=false

```java
// User.create() source (User.java dòng 22-24):
public static User create(Email email, PasswordHash passwordHash, Role role, DisplayName displayName) {
    return new User(null, new Account(email, passwordHash, role, false, false), new Profile(displayName), 0);
    //                                                               ^^^^  ^^^^
    //                                                         isActive=false, isEmailVerified=false
}
```

**Phải gọi `user.activate()` trước khi save** để set cả hai thành `true`:
```java
User user = User.create(email, passwordHash, Role.CUSTOMER, displayName);
user.activate(); // ← bắt buộc, đặt isActive=true, isEmailVerified=true
userRepository.save(user);
```

### Cấu trúc `RegisterUserUseCase` (đã có, đừng thay đổi)

```java
public interface RegisterUserUseCase {
    record Command(String email, String password, String displayName) {}
    User execute(Command cmd);
}
```

### `GlobalExceptionHandler` — Thêm gì

File hiện tại handle: `NoHandlerFoundException`, `AuthenticationException`, `AccessDeniedException`, `BadCredentialsException`, `IllegalArgumentException`, `Exception` (generic).

**Thiếu — cần thêm:**

```java
// Handler cho EmailConflictException → 409
@ExceptionHandler(EmailConflictException.class)
public ResponseEntity<ObjectNode> handleEmailConflict(EmailConflictException ex, HttpServletRequest request) {
    return buildErrorResponse("EMAIL_CONFLICT", ex.getMessage(), HttpStatus.CONFLICT, request);
}

// Handler cho MethodArgumentNotValidException → 400 với field details
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ObjectNode> handleValidation(MethodArgumentNotValidException ex, HttpServletRequest request) {
    ObjectNode body = ...; // dùng pattern giống buildErrorResponse nhưng thêm details map
    ObjectNode details = objectMapper.createObjectNode();
    ex.getBindingResult().getFieldErrors()
        .forEach(fe -> details.put(fe.getField(), fe.getDefaultMessage()));
    // ... set error="VALIDATION_FAILED", status=400
}
```

### Pattern `RegisterRequest` DTO

```java
public record RegisterRequest(
    @NotBlank String email,
    @NotBlank @Size(min = 8) String password,
    @NotBlank String displayName
) {}
```

### Pattern `RegisterUserService`

```java
@Service
@RequiredArgsConstructor
public class RegisterUserService implements RegisterUserUseCase {
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    @Override
    public User execute(Command cmd) {
        Email email = new Email(cmd.email());
        if (userRepository.existsByEmail(email)) {
            throw new EmailConflictException(cmd.email());
        }
        PasswordHash hash = new PasswordHash(passwordEncoder.encode(cmd.password()));
        DisplayName displayName = new DisplayName(cmd.displayName());
        User user = User.create(email, hash, Role.CUSTOMER, displayName);
        user.activate(); // email verification skipped per Story 2.3 update
        return userRepository.save(user);
    }
}
```

### Dependencies đã có sẵn trong `build.gradle`

- Spring Validation: `spring-boot-starter-validation` — đã có (dùng cho `@Valid`, `@NotBlank`, `@Size`)
- BCryptPasswordEncoder: bean đã được define trong `SecurityConfig.passwordEncoder()`
- Lombok: `@RequiredArgsConstructor`, `@Service` — đã có

### Testing

Không bắt buộc viết unit test cho story này (không có test folder setup cho services). Verify bằng manual curl test như Task 5.

### Project Structure Notes

- Package root: `com.tba.agentic`
- New file locations:
  - `adapter/transfer/request/RegisterRequest.java` (cạnh `LoginRequest.java`)
  - `application/service/RegisterUserService.java` (cạnh `LoginAttemptService.java`)
  - Không tạo file mới trong `core/` — interface đã có sẵn

### References

- [Source: epics.md — Story 2.3 Self-Registration] — ACs gốc và quyết định skip email verification
- [Source: AuthController.java] — Pattern endpoint, URL prefix `/api/v1/auth`
- [Source: UserRepositoryImpl.java] — Implementation `@Primary` của UserRepository
- [Source: User.java] — `User.create()` factory + `activate()` method
- [Source: RegisterUserUseCase.java] — Interface cần implement
- [Source: GlobalExceptionHandler.java] — Pattern `buildErrorResponse`, cần extend

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

### File List
