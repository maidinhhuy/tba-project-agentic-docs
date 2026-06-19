# Story 2.2: Email Infrastructure — ResendEmailAdapter

**Status:** ready-for-dev
**Story ID:** 2.2
**Epic:** 2 — Authentication & Self-Registration

---

## Story

As a developer,
I want a `ResendEmailAdapter` implementing the `EmailClient` port with fire-and-forget semantics and a dev-mode fallback,
So that email notifications can be sent without blocking main flows, and local dev works without a real API key.

---

## Acceptance Criteria

**AC-1** — Prod mode (API key set):
**Given** `TBA_RESEND_API_KEY` được set trong environment,
**When** `EmailClient.send(EmailMessage)` được gọi,
**Then** `ResendEmailAdapter` gọi Resend HTTP API (`POST https://api.resend.com/emails`) với đúng `from`, `to`, `subject`, `html`.
**And** nếu Resend trả về lỗi hoặc timeout (>5s), exception bị catch và log ở `WARN` level — không propagate lên caller.

**AC-2** — Dev mode (không có API key):
**Given** `TBA_RESEND_API_KEY` không được set hoặc blank,
**When** `EmailClient.send(EmailMessage)` được gọi,
**Then** nội dung email được log ở `INFO` level với prefix `[EMAIL-DEV]` — không throw exception.

**AC-3** — Async / fire-and-forget:
**Given** email send được gọi từ `SubmitProjectService` hoặc `UpdateProjectStatusService`,
**When** main request flow hoàn thành,
**Then** email failure không rollback transaction, không trả lỗi cho client.
**And** method `send()` return ngay, không block request thread.

**AC-4** — Cấu trúc:
**And** `EmailClient` interface tại `modules/core/.../port/out/EmailClient.java` — KHÔNG sửa.
**And** `EmailMessage` record tại `modules/core/.../port/out/EmailMessage.java` — KHÔNG sửa (fields: `to`, `subject`, `htmlBody`).
**And** `ResendEmailAdapter` tại `modules/infrastructure/.../infrastructure/email/ResendEmailAdapter.java`.
**And** `DevEmailClient` được update: bỏ `@Primary`, thêm `@ConditionalOnMissingBean(EmailClient.class)`.

---

## Tasks / Subtasks

- [ ] Task 1: Update `DevEmailClient` — remove `@Primary`, add fallback condition
  - File: `modules/infrastructure/src/main/java/com/tba/agentic/infrastructure/email/DevEmailClient.java`
  - Bỏ annotation `@Primary`
  - Thêm annotation `@ConditionalOnMissingBean(EmailClient.class)`
  - Thêm import: `org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean`
  - Logic `send()` giữ nguyên — không thay đổi

- [ ] Task 2: Create `ResendEmailAdapter`
  - File: `modules/infrastructure/src/main/java/com/tba/agentic/infrastructure/email/ResendEmailAdapter.java`
  - Annotations: `@Service`, `@Primary`, `@Slf4j`
  - Inject `@Value("${resend.api-key:}")` → `String apiKey`
  - Inject `@Value("${tba.email.from:noreply@tba.dev}")` → `String fromAddress`
  - Field `private final HttpClient httpClient = HttpClient.newBuilder().connectTimeout(Duration.ofSeconds(5)).build()`
  - Field `private final ObjectMapper objectMapper = new ObjectMapper()`
  - Implement `send(EmailMessage message)` với `@Async`:
    - Nếu `apiKey` blank → log INFO `[EMAIL-DEV]`, return
    - Nếu có API key → build JSON, gọi Resend API, catch toàn bộ exception → log WARN

- [ ] Task 3: Update `application.yml` — thêm Resend config
  - File: `modules/infrastructure/src/main/resources/application.yml`
  - Thêm vào cuối file:
    ```yaml
    resend:
      api-key: ${TBA_RESEND_API_KEY:}

    tba:
      email:
        from: ${TBA_EMAIL_FROM:noreply@tba.dev}
    ```
  - **Lưu ý:** `tba:` block đã tồn tại, chỉ thêm `email.from` vào bên trong block đó (tránh duplicate key)

- [ ] Task 4: Verify build và test
  - `./gradlew :modules:infrastructure:compileJava` → BUILD SUCCESSFUL
  - Smoke test dev mode: start app không set `TBA_RESEND_API_KEY`, trigger submit project, xem log `[EMAIL-DEV]`
  - Verify không có `NoUniqueBeanDefinitionException` khi start

---

## Dev Notes

### Module Structure — Authoritative

Sau PR#74 refactor, codebase dùng **3-module hexagonal** dưới `modules/`:

```
modules/
  core/src/main/java/com/tba/agentic/
    port/out/EmailClient.java        ← interface (KHÔNG SỬA)
    port/out/EmailMessage.java       ← record (KHÔNG SỬA)
  application/                       ← use cases (không cần sửa cho story này)
  infrastructure/src/main/java/com/tba/agentic/
    infrastructure/email/
      DevEmailClient.java            ← UPDATE (bỏ @Primary)
      ResendEmailAdapter.java        ← CREATE NEW
    resources/application.yml        ← UPDATE (thêm resend config)
```

> ⚠️ Root-level `core/` và `application/` là legacy (pre-PR#74) — KHÔNG đụng vào.

### EmailMessage Record — Không cần sửa

```java
// modules/core/.../port/out/EmailMessage.java
public record EmailMessage(String to, String subject, String htmlBody) {}
```

Chỉ có 3 fields. Epics.md đề cập `textBody` nhưng record hiện tại không có — **không thêm** để tránh breaking change vào 2 service đang dùng.

### @EnableAsync — Đã có sẵn

```java
// TbaAgenticApplication.java — KHÔNG cần sửa
@SpringBootApplication
@EnableAsync
public class TbaAgenticApplication { ... }
```

→ Chỉ cần thêm `@Async` vào method `send()` trong `ResendEmailAdapter`. Không cần tạo config class mới.

### DevEmailClient — Sau khi update

```java
@Service
@ConditionalOnMissingBean(EmailClient.class)  // ← thêm
@Slf4j
public class DevEmailClient implements EmailClient {
  // bỏ @Primary
  @Override
  public void send(EmailMessage message) {
    log.info("[EMAIL-DEV] to={} subject={}", message.to(), message.subject());
  }
}
```

`@ConditionalOnMissingBean` đảm bảo `DevEmailClient` chỉ được tạo khi không có bean `EmailClient` nào khác — tức là khi `ResendEmailAdapter` active, `DevEmailClient` bị bỏ qua hoàn toàn.

### ResendEmailAdapter — Full Implementation

```java
package com.tba.agentic.infrastructure.email;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.tba.agentic.port.out.EmailClient;
import com.tba.agentic.port.out.EmailMessage;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Primary;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;
import java.util.List;
import java.util.Map;

@Service
@Primary
@Slf4j
public class ResendEmailAdapter implements EmailClient {

    @Value("${resend.api-key:}")
    private String apiKey;

    @Value("${tba.email.from:noreply@tba.dev}")
    private String fromAddress;

    private final HttpClient httpClient = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(5))
            .build();

    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    @Async
    public void send(EmailMessage message) {
        if (apiKey == null || apiKey.isBlank()) {
            log.info("[EMAIL-DEV] to={} subject={}", message.to(), message.subject());
            return;
        }
        try {
            Map<String, Object> payload = Map.of(
                    "from", fromAddress,
                    "to", List.of(message.to()),
                    "subject", message.subject(),
                    "html", message.htmlBody()
            );
            String body = objectMapper.writeValueAsString(payload);

            HttpRequest request = HttpRequest.newBuilder()
                    .uri(URI.create("https://api.resend.com/emails"))
                    .header("Authorization", "Bearer " + apiKey)
                    .header("Content-Type", "application/json")
                    .timeout(Duration.ofSeconds(5))
                    .POST(HttpRequest.BodyPublishers.ofString(body))
                    .build();

            HttpResponse<String> response = httpClient.send(request, HttpResponse.BodyHandlers.ofString());

            if (response.statusCode() < 200 || response.statusCode() >= 300) {
                log.warn("Resend API error: status={} to={} body={}", response.statusCode(), message.to(), response.body());
            } else {
                log.debug("Email sent via Resend: to={} subject={}", message.to(), message.subject());
            }
        } catch (Exception e) {
            log.warn("Email send failed for to={}: {}", message.to(), e.getMessage());
        }
    }
}
```

### application.yml — Cách update đúng

File hiện tại có block `tba:` đã tồn tại. Thêm `email.from` vào **bên trong** block `tba:` đó, không tạo block `tba:` mới:

```yaml
tba:
  security:
    jwt: ...
  frontend:
    base-url: ...
  admin:
    email: ...
    seed-password: ...
  email:                                    # ← thêm vào đây
    from: ${TBA_EMAIL_FROM:noreply@tba.dev}

resend:                                     # ← thêm block mới
  api-key: ${TBA_RESEND_API_KEY:}
```

### Callers hiện tại — Không cần sửa

Cả 2 services đã có try-catch bao quanh `emailClient.send()`:

```java
// SubmitProjectService.java — giữ nguyên
try {
    emailClient.send(new EmailMessage(adminEmail, "...", "..."));
} catch (Exception e) {
    log.error("Failed to send email notification for project submission", e);
}

// UpdateProjectStatusService.java — giữ nguyên
try {
    emailClient.send(new EmailMessage(customerEmail, "...", html));
} catch (Exception e) {
    log.warn("Failed to send status update email for project {}: ...", ...);
}
```

Với `@Async`, method `send()` return `void` ngay lập tức — exception trong async thread sẽ được log bởi Spring's `AsyncUncaughtExceptionHandler` (default behavior). Try-catch trong callers trở thành dead code nhưng không gây hại.

### Resend API Reference

- Endpoint: `POST https://api.resend.com/emails`
- Auth: `Authorization: Bearer <RESEND_API_KEY>`
- Request body:
  ```json
  {
    "from": "noreply@tba.dev",
    "to": ["customer@example.com"],
    "subject": "...",
    "html": "<p>...</p>"
  }
  ```
- Success: HTTP 200, body `{"id": "..."}`
- Error codes: 422 (validation), 429 (rate limit), 500 (Resend server error)
- `from` address phải là domain đã verify trên Resend dashboard. Trong dev không cần verify vì dev mode (no API key) không gọi API thật.

### Railway Env Vars cần thêm (production)

```
TBA_RESEND_API_KEY=re_xxxxxxxxxxxx   # từ Resend dashboard → API Keys
TBA_EMAIL_FROM=noreply@yourdomain.com  # phải là domain đã verify trên Resend
```

### No New Gradle Dependencies

`java.net.http.HttpClient` là built-in Java 17+ — không cần thêm dependency.
`com.fasterxml.jackson.databind.ObjectMapper` đã có qua `spring-boot-starter-web`.

---

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Completion Notes List

- Phân tích codebase trực tiếp tại `modules/` (post-PR#74 3-module hexagonal)
- `EmailMessage` không có `textBody` field — không thay đổi record để tránh breaking change
- `@EnableAsync` đã có trong `TbaAgenticApplication.java` — không cần config thêm
- Jackson `ObjectMapper` dùng trực tiếp vì đã có qua `spring-boot-starter-web`

### File List

**UPDATE:**
- `modules/infrastructure/src/main/java/com/tba/agentic/infrastructure/email/DevEmailClient.java`
- `modules/infrastructure/src/main/resources/application.yml`

**CREATE:**
- `modules/infrastructure/src/main/java/com/tba/agentic/infrastructure/email/ResendEmailAdapter.java`
