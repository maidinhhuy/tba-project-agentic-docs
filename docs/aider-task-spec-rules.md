# Aider Task Ticket Rules — Notion Agent Tasks Database

Tài liệu này định nghĩa các rule bắt buộc khi tạo ticket trong Notion database **🤖 Agent Tasks**.
Được rút ra từ phân tích thực tế các lỗi trong aider-logs (S-1.1, S-1.4) ngày 2026-06-12.

---

## Rule 1 — "Files to Edit": mỗi file PHẢI có full directory path

**Đây là lỗi phổ biến nhất và gây ra ~80% compile failures.**

Field "Files to Edit" trong Notion dùng format rich text: `<path>/[filename.ext](http://filename.ext)`.
Script pull xuống giữ lại phần text thuần trước link và text của link — nếu thiếu path prefix, aider nhận được bare filename và tạo file ở workspace root thay vì đúng package.

```
# SAI — chỉ file đầu có path, các file còn lại là bare link
core/src/main/java/com/tba/agentic/domain/user/[User.java](http://User.java), [UserId.java](http://UserId.java), [Email.java](http://Email.java)

# ĐÚNG — mỗi entry đều có full directory path trước link
core/src/main/java/com/tba/agentic/domain/user/[User.java](http://User.java), core/src/main/java/com/tba/agentic/domain/user/[UserId.java](http://UserId.java), core/src/main/java/com/tba/agentic/domain/user/[Email.java](http://Email.java)
```

**Pattern bắt buộc cho mỗi entry:** `<module>/<src_path>/<package_path>/[FileName.java](http://FileName.java)`

Ví dụ full format đúng:
```
core/src/main/java/com/tba/agentic/domain/user/[User.java](http://User.java), core/src/main/java/com/tba/agentic/domain/user/[UserId.java](http://UserId.java), core/src/main/java/com/tba/agentic/domain/user/[Email.java](http://Email.java), core/src/main/java/com/tba/agentic/domain/user/[Account.java](http://Account.java)
```

---

## Rule 2 — Non-Java files không được để trong "Files to Edit"

**Vấn đề:** Script xử lý tất cả entries trong "Files to Edit" như Java files, tự động tạo empty `.java` file. File `application.yml` bị tạo thành `application.yml.java`.

```
# SAI — .yml và .env nằm trong Files to Edit
application/src/main/java/com/tba/agentic/[TbaAgenticApplication.java](http://TbaAgenticApplication.java), application/src/main/resources/application.yml, application/src/main/resources/.env.example

# ĐÚNG — chỉ để Java files trong "Files to Edit"
# Mô tả resource files (.yml, .xml, .env, .properties) trong page content
application/src/main/java/com/tba/agentic/[TbaAgenticApplication.java](http://TbaAgenticApplication.java)
```

Resource files nên được mô tả trong **page content** của ticket (phần "Files cần tạo") với nội dung đầy đủ để aider tự tạo.

---

## Rule 3 — Mỗi task chỉ thuộc một Gradle module

**Vấn đề:** Task mix files của `core/` và `application/` trong cùng "Files to Edit". Khi compile fail ở module A, aider không biết có nên sửa module B không.

```
# SAI — mix 2 modules
core/src/main/java/com/tba/agentic/domain/exception/[AuthenticationException.java](http://AuthenticationException.java), application/src/main/java/com/tba/agentic/config/[GlobalExceptionHandler.java](http://GlobalExceptionHandler.java)

# ĐÚNG — tách thành 2 tasks, dùng "Depends On" relation
# TASK-3a: core module
core/src/main/java/com/tba/agentic/domain/exception/[AuthenticationException.java](http://AuthenticationException.java), core/src/main/java/com/tba/agentic/domain/exception/[EmailConflictException.java](http://EmailConflictException.java)

# TASK-3b: application module — Depends On: TASK-3a
application/src/main/java/com/tba/agentic/config/[GlobalExceptionHandler.java](http://GlobalExceptionHandler.java)
```

Dùng field **"Depends On"** (relation) để express ordering — đây là lý do field đó tồn tại trong schema.

---

## Rule 4 — Page content: code snippet không được forward-reference class chưa tồn tại

**Vấn đề:** S-1.1/TASK-3 tạo `GlobalExceptionHandler` với handler cho `AuthenticationException` — nhưng class đó chỉ tồn tại sau S-1.4/TASK-3. Aider compile fail ngay từ round 1.

```java
// SAI — viết trong task tạo GlobalExceptionHandler ban đầu
@ExceptionHandler(AuthenticationException.class) // class chưa tồn tại!
public ResponseEntity<?> handleAuth(AuthenticationException ex) { ... }

// ĐÚNG — dùng TODO comment
// TODO(S-1.4/TASK-3b): Thêm handler cho AuthenticationException sau khi domain exceptions được tạo
// TODO(S-1.4/TASK-3b): Thêm handler cho InvalidStatusTransitionException
```

**Quy tắc:** Mọi class được reference trong code snippet phải đã tồn tại (từ task trước, committed vào branch) hoặc được tạo trong cùng task này.

---

## Rule 5 — Page content: không để section bị truncate

**Vấn đề:** Description có dòng `- Handlers tối thiểu:` nhưng không có nội dung tiếp theo. Aider đọc story-spec.md làm context và hallucinate handlers, dẫn đến forward-reference lỗi.

```
# SAI — heading không có content
- Handlers tối thiểu:

# ĐÚNG — liệt kê đầy đủ
- Handlers tối thiểu:
  - MethodArgumentNotValidException → HTTP 400, error code: VALIDATION_FAILED
  - HttpMessageNotReadableException → HTTP 400, error code: MALFORMED_REQUEST  
  - ResponseStatusException → extract status từ exception
  - Exception (catch-all) → HTTP 500, error code: INTERNAL_ERROR
```

---

## Rule 6 — Giới hạn "Files to Edit": tối đa 6 entries

**Vấn đề:** Task tạo 8 files cùng lúc. Khi 1 file bị sai path, lỗi cascade sang toàn bộ các files còn lại.

```
# SAI — 8 files trong 1 task
core/.../[User.java], core/.../[UserId.java], core/.../[Email.java],
core/.../[PasswordHash.java], core/.../[Role.java], core/.../[Account.java],
core/.../[Profile.java], core/.../[DisplayName.java]   ← 8 files, quá nhiều

# ĐÚNG — tách thành 2 tasks
# TASK-1a (6 files): Value objects — UserId, Email, PasswordHash, Role, DisplayName, Account
# TASK-1b (2 files): Aggregate — Profile, User — Depends On: TASK-1a
```

---

## Rule 7 — Story Acceptance Criteria không được paste nguyên vào task

**Vấn đề:** Tất cả tasks trong S-1.4 đều chứa toàn bộ Story AC (gồm cả port interfaces, test requirements v.v.). Aider cố implement tất cả thay vì chỉ task hiện tại.

Story AC là field trong **Story page** — không copy sang Task page.

Trong Task page, phần **"Verify"** (trong page content) chỉ mô tả câu lệnh verify cụ thể cho task đó:
```
## Verify
- `./gradlew :core:compileJava` → BUILD SUCCESSFUL
- `grep -r "import org.springframework" core/src` → zero results
```

---

## Checklist tạo ticket (copy để dùng)

Trước khi set **Agent Status = Ready**, kiểm tra:

```
□ Mỗi entry trong "Files to Edit" có format: path/to/dir/[FileName.java](http://FileName.java)
□ KHÔNG có bare filename không có path prefix
□ KHÔNG có file .yml, .xml, .env, .properties trong "Files to Edit"
□ Tất cả files thuộc cùng một Gradle module (core/ hoặc application/)
□ Nếu task cần update file từ task trước → đã khai báo "Depends On"
□ Code snippet KHÔNG reference class từ task sau
□ Không có heading/list item nào bị truncate trong page content
□ "Files to Edit" có ≤ 6 entries
□ Phần "Verify" chỉ chứa lệnh verify cho task này, không copy Story AC
```

---

## Tham chiếu: Format "Files to Edit" theo module

### Core module (domain)
```
core/src/main/java/com/tba/agentic/domain/<domain>/[ClassName.java](http://ClassName.java)
core/src/main/java/com/tba/agentic/domain/exception/[ExceptionName.java](http://ExceptionName.java)
core/src/main/java/com/tba/agentic/port/in/[UseCaseName.java](http://UseCaseName.java)
core/src/main/java/com/tba/agentic/port/out/[RepositoryName.java](http://RepositoryName.java)
```

### Application module (adapters, services, infrastructure)
```
application/src/main/java/com/tba/agentic/adapter/controller/[ControllerName.java](http://ControllerName.java)
application/src/main/java/com/tba/agentic/adapter/filter/[FilterName.java](http://FilterName.java)
application/src/main/java/com/tba/agentic/adapter/transfer/request/[RequestDto.java](http://RequestDto.java)
application/src/main/java/com/tba/agentic/adapter/transfer/response/[ResponseDto.java](http://ResponseDto.java)
application/src/main/java/com/tba/agentic/application/service/[ServiceName.java](http://ServiceName.java)
application/src/main/java/com/tba/agentic/config/[ConfigName.java](http://ConfigName.java)
application/src/main/java/com/tba/agentic/infrastructure/repository/[RepositoryImpl.java](http://RepositoryImpl.java)
application/src/main/java/com/tba/agentic/security/[SecurityClass.java](http://SecurityClass.java)
```

### Test files
```
core/src/test/java/com/tba/agentic/domain/<domain>/[ClassNameTest.java](http://ClassNameTest.java)
application/src/test/java/com/tba/agentic/..../[ClassNameTest.java](http://ClassNameTest.java)
```
