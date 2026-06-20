# Story 8.1: DB Migration V2 + S3 File Storage Infrastructure

**Status:** ready-for-dev
**Story ID:** 8.1
**Epic:** 8 — Milestone Delivery & Customer Review

---

## Story

As a developer,
I want the database extended with milestone delivery/review columns and a milestone_reviews history table, and AWS S3 file storage wired up as a port/adapter,
So that all subsequent Epic 8 stories have a stable data and storage foundation to build on.

---

## Acceptance Criteria

**AC-1** — Flyway V2 migration runs clean:
**Given** Docker PostgreSQL đang chạy,
**When** `./gradlew flywayMigrate`,
**Then** `V2__milestone_delivery.sql` hoàn thành không lỗi, `flyway_schema_history` mark SUCCESS.

**AC-2** — Milestone columns added:
**Given** migration chạy xong,
**When** schema được inspect,
**Then** bảng `milestones` có thêm:
- `deliverable_url TEXT`
- `deliverable_file_key TEXT`
- `deliverable_file_name TEXT`
- `delivery_note TEXT`
- `delivered_at TIMESTAMPTZ`
- `review_status VARCHAR(30) NOT NULL DEFAULT 'PENDING_DELIVERY'` với CHECK IN ('PENDING_DELIVERY','PENDING_REVIEW','APPROVED','REVISION_REQUESTED')

**AC-3** — milestone_reviews table exists:
**Given** migration chạy xong,
**When** schema được inspect,
**Then** bảng `milestone_reviews` tồn tại với:
- `id UUID PK DEFAULT gen_random_uuid()`
- `milestone_id BIGINT NOT NULL REFERENCES milestones(milestone_id) ON DELETE CASCADE`
- `action VARCHAR(30) NOT NULL` CHECK IN ('APPROVED','REVISION_REQUESTED')
- `comment TEXT`
- `reviewed_by BIGINT REFERENCES users(user_id)`
- `reviewed_at TIMESTAMPTZ NOT NULL DEFAULT now()`
- Index: `idx_milestone_reviews_milestone_id ON milestone_reviews(milestone_id)`

**AC-4** — jOOQ codegen sau migration:
**Given** migration V2 đã chạy,
**When** `./gradlew generateJooq`,
**Then** generated classes `MilestoneReviews` và updated `Milestones` tồn tại trong `infrastructure/jooq/`. Build pass không lỗi.

**AC-5** — S3 adapter khởi động:
**Given** env vars `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, `AWS_S3_BUCKET_NAME` được set,
**When** app khởi động,
**Then** `S3FileStorageAdapter` bean được khởi tạo không lỗi.

**AC-6** — Upload file trả về fileKey:
**Given** `FileStoragePort.upload(fileName, contentType, inputStream, projectId, milestoneId)` được gọi,
**When** upload thành công lên S3,
**Then** trả về `fileKey` dạng `deliverables/{projectId}/{milestoneId}/{uuid}-{fileName}`.

**AC-7** — Presigned URL generation:
**Given** `FileStoragePort.generatePresignedUrl(fileKey, ttlMinutes)` được gọi với fileKey hợp lệ,
**When** call thành công,
**Then** trả về URL presigned dạng String, hợp lệ trong `ttlMinutes` phút.

**AC-8** — Env template updated:
**And** `.env.local` template thêm 4 biến: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, `AWS_S3_BUCKET_NAME`.

---

## Tasks / Subtasks

### Migration Tasks

- [ ] **Task 1: Tạo V2__milestone_delivery.sql**
  - File: `modules/infrastructure/src/main/resources/db/migration/V2__milestone_delivery.sql`
  - Nội dung đầy đủ:
    ```sql
    -- Add delivery/review columns to milestones
    ALTER TABLE milestones
      ADD COLUMN deliverable_url       TEXT,
      ADD COLUMN deliverable_file_key  TEXT,
      ADD COLUMN deliverable_file_name TEXT,
      ADD COLUMN delivery_note         TEXT,
      ADD COLUMN delivered_at          TIMESTAMPTZ,
      ADD COLUMN review_status         VARCHAR(30) NOT NULL DEFAULT 'PENDING_DELIVERY';

    ALTER TABLE milestones
      ADD CONSTRAINT milestones_review_status_check
      CHECK (review_status IN ('PENDING_DELIVERY','PENDING_REVIEW','APPROVED','REVISION_REQUESTED'));

    -- Milestone review history
    CREATE TABLE milestone_reviews (
      id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      milestone_id BIGINT NOT NULL REFERENCES milestones(milestone_id) ON DELETE CASCADE,
      action       VARCHAR(30) NOT NULL,
      comment      TEXT,
      reviewed_by  BIGINT REFERENCES users(user_id),
      reviewed_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
      CONSTRAINT milestone_reviews_action_check CHECK (action IN ('APPROVED','REVISION_REQUESTED'))
    );

    CREATE INDEX idx_milestone_reviews_milestone_id ON milestone_reviews(milestone_id);
    ```

- [ ] **Task 2: Chạy generate để verify**
  - `make generate` (flyway migrate + jooq codegen)
  - Confirm `MilestoneReviews.java` tồn tại trong generated folder
  - Confirm `Milestones.java` có thêm các field mới
  - `make compile` phải pass

### S3 Infrastructure Tasks

- [ ] **Task 3: Thêm AWS SDK dependency**
  - File: `build.gradle` của module `infrastructure`
  - Thêm:
    ```groovy
    implementation 'software.amazon.awssdk:s3:2.25.0'
    implementation 'software.amazon.awssdk:sts:2.25.0'
    ```
  - Kiểm tra version mới nhất tại https://central.sonatype.com/artifact/software.amazon.awssdk/s3

- [ ] **Task 4: Tạo FileStoragePort interface**
  - File: `modules/core/src/main/java/com/tba/agentic/port/out/FileStoragePort.java`
  ```java
  package com.tba.agentic.port.out;

  import java.io.InputStream;

  public interface FileStoragePort {
    String upload(String fileName, String contentType, InputStream data, String projectId, String milestoneId);
    String generatePresignedUrl(String fileKey, int ttlMinutes);
  }
  ```

- [ ] **Task 5: Tạo S3FileStorageAdapter**
  - File: `modules/infrastructure/src/main/java/com/tba/agentic/infrastructure/storage/S3FileStorageAdapter.java`
  ```java
  package com.tba.agentic.infrastructure.storage;

  import com.tba.agentic.port.out.FileStoragePort;
  import java.io.InputStream;
  import java.time.Duration;
  import java.util.UUID;
  import lombok.RequiredArgsConstructor;
  import lombok.extern.slf4j.Slf4j;
  import org.springframework.beans.factory.annotation.Value;
  import org.springframework.stereotype.Component;
  import software.amazon.awssdk.core.sync.RequestBody;
  import software.amazon.awssdk.services.s3.S3Client;
  import software.amazon.awssdk.services.s3.model.PutObjectRequest;
  import software.amazon.awssdk.services.s3.presigner.S3Presigner;
  import software.amazon.awssdk.services.s3.presigner.model.GetObjectPresignRequest;

  @Slf4j
  @Component
  @RequiredArgsConstructor
  public class S3FileStorageAdapter implements FileStoragePort {

    private final S3Client s3Client;
    private final S3Presigner s3Presigner;

    @Value("${aws.s3.bucket-name}")
    private String bucketName;

    @Override
    public String upload(String fileName, String contentType, InputStream data, String projectId, String milestoneId) {
      String fileKey = "deliverables/" + projectId + "/" + milestoneId + "/" + UUID.randomUUID() + "-" + fileName;
      s3Client.putObject(
          PutObjectRequest.builder()
              .bucket(bucketName)
              .key(fileKey)
              .contentType(contentType)
              .build(),
          RequestBody.fromInputStream(data, -1));
      return fileKey;
    }

    @Override
    public String generatePresignedUrl(String fileKey, int ttlMinutes) {
      return s3Presigner.presignGetObject(GetObjectPresignRequest.builder()
          .signatureDuration(Duration.ofMinutes(ttlMinutes))
          .getObjectRequest(r -> r.bucket(bucketName).key(fileKey))
          .build())
          .url()
          .toString();
    }
  }
  ```

- [ ] **Task 6: Tạo S3Config bean**
  - File: `modules/infrastructure/src/main/java/com/tba/agentic/config/S3Config.java`
  ```java
  package com.tba.agentic.config;

  import org.springframework.beans.factory.annotation.Value;
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.Configuration;
  import software.amazon.awssdk.auth.credentials.AwsBasicCredentials;
  import software.amazon.awssdk.auth.credentials.StaticCredentialsProvider;
  import software.amazon.awssdk.regions.Region;
  import software.amazon.awssdk.services.s3.S3Client;
  import software.amazon.awssdk.services.s3.presigner.S3Presigner;

  @Configuration
  public class S3Config {

    @Value("${aws.access-key-id}")
    private String accessKeyId;

    @Value("${aws.secret-access-key}")
    private String secretAccessKey;

    @Value("${aws.region}")
    private String region;

    @Bean
    public S3Client s3Client() {
      return S3Client.builder()
          .region(Region.of(region))
          .credentialsProvider(StaticCredentialsProvider.create(
              AwsBasicCredentials.create(accessKeyId, secretAccessKey)))
          .build();
    }

    @Bean
    public S3Presigner s3Presigner() {
      return S3Presigner.builder()
          .region(Region.of(region))
          .credentialsProvider(StaticCredentialsProvider.create(
              AwsBasicCredentials.create(accessKeyId, secretAccessKey)))
          .build();
    }
  }
  ```

- [ ] **Task 7: Thêm env config vào application.yml**
  - File: `modules/infrastructure/src/main/resources/application.yml`
  - Thêm section:
    ```yaml
    aws:
      access-key-id: ${AWS_ACCESS_KEY_ID}
      secret-access-key: ${AWS_SECRET_ACCESS_KEY}
      region: ${AWS_REGION}
      s3:
        bucket-name: ${AWS_S3_BUCKET_NAME}
    ```

- [ ] **Task 8: Cập nhật .env.local template**
  - Thêm vào README hoặc docs:
    ```bash
    AWS_ACCESS_KEY_ID=your-access-key
    AWS_SECRET_ACCESS_KEY=your-secret-key
    AWS_REGION=ap-southeast-1
    AWS_S3_BUCKET_NAME=tba-agentic-deliverables
    ```

- [ ] **Task 9: make compile + make test**
  - `make compile` phải pass
  - `make test` phải pass (không có test mới cần viết ở story này — infra tests ở story sau)

---

## Dev Notes

### Module Structure Thực Tế

```
modules/
├── core/        → domain, port/in, port/out (pure Java, zero framework)
├── application/ → use case services (Lombok, @Transactional OK, no Spring beans)
└── infrastructure/ → Spring Boot, jOOQ, controllers, config, storage
```

Module `infrastructure` → `application` → `core` (dependency direction).

### jOOQ Generated Code Location

```
modules/infrastructure/src/main/java/com/tba/agentic/infrastructure/jooq/
├── Tables.java
├── tables/
│   ├── Milestones.java    ← sẽ có thêm columns sau migration
│   └── MilestoneReviews.java  ← mới
```

Import pattern trong repository:
```java
import static com.tba.agentic.infrastructure.jooq.Tables.MILESTONE_REVIEWS;
import static com.tba.agentic.infrastructure.jooq.Tables.MILESTONES;
```

### Flyway Migration Numbering

File migration hiện có: `V1__initial_schema.sql`
→ Story 8.1 tạo: `V2__milestone_delivery.sql`
→ KHÔNG được skip số — Flyway sẽ fail nếu có gap.

### AWS SDK v2 Pattern

Dùng **AWS SDK v2** (software.amazon.awssdk), KHÔNG dùng v1 (com.amazonaws). SDK v2 là async-first, có builder pattern rõ ràng hơn.

`RequestBody.fromInputStream(data, -1)` — contentLength = -1 khi không biết trước size (file upload từ multipart). S3 SDK sẽ dùng chunked transfer encoding.

### File Key Convention

```
deliverables/{projectId}/{milestoneId}/{uuid}-{originalFileName}
```

Ví dụ: `deliverables/abc123/42/550e8400-e29b-41d4-a716-446655440000-wireframe.pdf`

UUID prefix đảm bảo uniqueness khi cùng tên file được upload nhiều lần. ProjectId/milestoneId làm prefix cho dễ quản lý và set S3 bucket policy nếu cần.

### Không Cần Test Riêng

Story 8.1 là pure infrastructure plumbing. Integration test thực sự cần real S3 — defer sang story 8.2/8.3 khi có flow end-to-end. Hiện tại chỉ cần `make compile` pass và `make test` không break test hiện có.
