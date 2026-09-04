# Build Guide — Hướng dẫn dựng dự án
## Phượng Hồng Memories — Kỷ yếu điện tử lớp 12A1

---

## 1. Tài liệu này dùng thế nào

Các tài liệu khác trả lời **làm cái gì** và **vì sao**. Tài liệu này trả lời **gõ gì, theo thứ tự nào**.

Nguyên tắc xuyên suốt: **lát cắt dọc, không phải tầng ngang.**

```
❌ SAI: viết hết entity → viết hết repository → viết hết service → viết hết controller
   Ba tuần không có gì chạy được. Đến lúc ghép lại thì sai ở đâu không biết.

✅ ĐÚNG: làm XONG một tính năng đi hết mọi tầng, chạy được, có test.
   Rồi mới sang tính năng thứ hai.
```

Tính năng đầu tiên nên là **Lời nhắn** — nó đơn giản nhất nhưng vẫn đi qua đủ mọi tầng. Làm xong nó nghĩa là bộ khung đã đúng, và mọi tính năng sau chỉ là lặp lại khuôn đó.

---

## 2. Bước 0 — Chuẩn bị máy

| Phần mềm | Phiên bản | Kiểm tra bằng |
|---|---|---|
| JDK | Bản LTS hiện hành (21 hoặc 25) | `java -version` |
| Docker Desktop | Mới nhất | `docker --version` |
| Node.js | LTS (20 hoặc mới hơn) | `node -v` |
| Git | Bất kỳ | `git --version` |
| IDE | IntelliJ IDEA Community (đủ dùng) hoặc VS Code | |

Cài thêm cho tiện:

```powershell
# Windows — dùng winget
winget install EclipseAdoptium.Temurin.21.JDK
winget install Docker.DockerDesktop
winget install OpenJS.NodeJS.LTS
winget install Git.Git
```

Công cụ dòng lệnh nên có: `rclone` (cho sao lưu), `httpie` hoặc `curl` (thử API).

---

## 3. Bước 1 — Môi trường phát triển bằng Docker

Tạo `docker-compose.dev.yml` ở thư mục gốc dự án. Đây là toàn bộ hạ tầng phát triển — chạy một lệnh là có đủ.

```yaml
services:
  postgres:
    image: postgres:17-alpine
    environment:
      POSTGRES_DB: memorybook
      POSTGRES_USER: memorybook
      POSTGRES_PASSWORD: devonly_khong_dung_that
    ports:
      - "127.0.0.1:5432:5432"     # chỉ nghe trên localhost, không ra mạng
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U memorybook"]
      interval: 5s
      retries: 10

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass devonly_khong_dung_that
    ports:
      - "127.0.0.1:6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "devonly_khong_dung_that", "ping"]
      interval: 5s
      retries: 10

  # MinIO — bản S3 chạy tại máy. Cùng giao thức với Cloudflare R2,
  # nên code không khác một dòng nào khi chuyển sang môi trường thật.
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: devonly_khong_dung_that
    ports:
      - "127.0.0.1:9000:9000"     # API S3
      - "127.0.0.1:9001:9001"     # giao diện web, mở bằng trình duyệt để xem tệp
    volumes:
      - miniodata:/data
    healthcheck:
      test: ["CMD", "mc", "ready", "local"]
      interval: 5s
      retries: 10

  # Tạo sẵn ba bucket rồi tự thoát
  minio-init:
    image: minio/mc:latest
    depends_on:
      minio:
        condition: service_healthy
    entrypoint: >
      /bin/sh -c "
      mc alias set local http://minio:9000 minioadmin devonly_khong_dung_that;
      mc mb -p local/memorybook-archive;
      mc mb -p local/memorybook-public;
      mc mb -p local/memorybook-quarantine;
      mc anonymous set download local/memorybook-public;
      echo 'Đã tạo xong bucket';
      "

  clamav:
    image: clamav/clamav:latest
    ports:
      - "127.0.0.1:3310:3310"
    healthcheck:
      test: ["CMD", "clamdscan", "--ping", "1"]
      interval: 30s
      start_period: 120s          # lần đầu tải cơ sở dữ liệu mẫu rất lâu
      retries: 5

volumes:
  pgdata:
  miniodata:
```

```bash
docker compose -f docker-compose.dev.yml up -d
docker compose -f docker-compose.dev.yml ps     # đợi tất cả "healthy"
```

Mở `http://localhost:9001` để xem giao diện MinIO — rất hữu ích khi gỡ lỗi phần lưu ảnh, vì bạn nhìn thấy tệp thật nằm ở đâu.

> **Lưu ý**: `clamav` lần đầu khởi động mất 2–5 phút để tải cơ sở dữ liệu mẫu virus. Trong lúc chờ, cứ làm tiếp bước 2 — chưa cần tới nó cho tới Giai đoạn 3.

---

## 4. Bước 2 — Khởi tạo dự án Spring Boot

Vào [start.spring.io](https://start.spring.io) và chọn:

| Mục | Giá trị |
|---|---|
| Project | Maven |
| Language | Java |
| Spring Boot | Bản ổn định mới nhất |
| Group | `vn.memorybook` |
| Artifact | `server` |
| Packaging | Jar |
| Java | Bản LTS đã cài |

**Dependencies cần chọn:**

```
Spring Web                  ← REST API
Spring Data JPA             ← truy cập cơ sở dữ liệu
Spring Security             ← xác thực, phân quyền
Validation                  ← @NotBlank, @Size
Spring Data Redis           ← cache, phiên, giới hạn tần suất
Spring Session              ← lưu phiên vào Redis
Flyway Migration            ← di trú lược đồ
PostgreSQL Driver
Spring Boot Actuator        ← health check, số liệu giám sát
Lombok                      ← bớt mã lặp
Testcontainers              ← kiểm thử với cơ sở dữ liệu thật
```

Giải nén vào thư mục `server/`. Thêm vào `pom.xml` các phụ thuộc không có trên start.spring.io:

```xml
<dependencies>
    <!-- Kho đối tượng: dùng chung cho MinIO (dev) và Cloudflare R2 (thật) -->
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>s3</artifactId>
        <version>${awssdk.version}</version>
    </dependency>

    <!-- Giới hạn tần suất phân tán -->
    <dependency>
        <groupId>com.bucket4j</groupId>
        <artifactId>bucket4j-redis</artifactId>
        <version>${bucket4j.version}</version>
    </dependency>

    <!-- Ánh xạ DTO ↔ Entity, sinh mã lúc biên dịch -->
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
        <version>${mapstruct.version}</version>
    </dependency>

    <!-- Tài liệu OpenAPI + Swagger UI -->
    <dependency>
        <groupId>org.springdoc</groupId>
        <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
        <version>${springdoc.version}</version>
    </dependency>

    <!-- Số liệu theo định dạng Prometheus -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
        <scope>runtime</scope>
    </dependency>

    <!-- Log JSON có cấu trúc -->
    <dependency>
        <groupId>net.logstash.logback</groupId>
        <artifactId>logstash-logback-encoder</artifactId>
        <version>${logstash.version}</version>
    </dependency>
</dependencies>
```

> Tra phiên bản mới nhất của từng thư viện trên Maven Central trước khi ghim. Đừng chép số phiên bản từ tài liệu này — nó sẽ cũ.

Tạo `.env` ở thư mục gốc (và **thêm ngay vào `.gitignore`**):

```bash
DATABASE_URL=jdbc:postgresql://localhost:5432/memorybook
DATABASE_USER=memorybook
DATABASE_PASSWORD=devonly_khong_dung_that

REDIS_HOST=localhost
REDIS_PASSWORD=devonly_khong_dung_that

S3_ENDPOINT=http://localhost:9000
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=devonly_khong_dung_that
S3_PATH_STYLE=true
S3_REGION=us-east-1
S3_BUCKET_ARCHIVE=memorybook-archive
S3_BUCKET_PUBLIC=memorybook-public
S3_BUCKET_QUARANTINE=memorybook-quarantine

CLAMAV_HOST=localhost
CLAMAV_PORT=3310

IP_HASH_SECRET=doi_thanh_chuoi_ngau_nhien_64_ky_tu
```

Và `.env.example` — bản sao **chỉ có tên biến, không có giá trị** — thì commit vào git.

---

## 5. Bước 3 — Cấu trúc package

Tạo cây thư mục này ngay từ đầu, kể cả khi nhiều thư mục còn trống:

```
server/src/main/java/vn/memorybook/
├── MemoryBookApplication.java
├── common/
│   ├── ApiError.java              GlobalExceptionHandler + Problem Details
│   ├── CursorCodec.java           mã hoá/giải mã con trỏ phân trang
│   ├── IpHasher.java              HMAC-SHA256 địa chỉ IP
│   └── Audited.java               annotation cho nhật ký kiểm toán
├── config/
│   ├── SecurityConfig.java
│   ├── S3Config.java
│   ├── RedisConfig.java
│   └── OpenApiConfig.java
├── message/                       ← TÍNH NĂNG ĐẦU TIÊN
│   ├── Message.java               entity
│   ├── MessageStatus.java         enum
│   ├── MessageRepository.java
│   ├── MessageService.java
│   ├── MessageController.java
│   ├── MessageMapper.java
│   └── dto/
│       ├── CreateMessageRequest.java
│       ├── MessageResponse.java
│       └── MessagePageResponse.java
├── photo/
├── admin/
├── takedown/
├── storage/
├── outbox/
└── security/
```

Chia **theo tính năng, không theo tầng**. Sửa gì liên quan tới lời nhắn thì mở đúng một thư mục.

Tạo tệp di trú đầu tiên: `server/src/main/resources/db/migration/V1__init.sql` — chép lược đồ từ [`LLD`](MemoryBook-LLD.md) mục 2.2, nhớ thay chỗ giữ chỗ `BIGGENERATED_PLACEHOLDER` bằng `BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY`.

Đặt `application.yml` theo [`TechStack`](MemoryBook-TechStack.md) mục 4.3. **Bốn dòng không được sai**: `ddl-auto: validate`, `open-in-view: false`, `include-stacktrace: never`, `management.server.port: 9090`.

Chạy thử:

```bash
cd server
./mvnw spring-boot:run
```

Flyway sẽ chạy `V1__init.sql`. Vào cơ sở dữ liệu kiểm tra bảng đã có:

```bash
docker compose -f ../docker-compose.dev.yml exec postgres \
  psql -U memorybook -d memorybook -c "\dt"
```

**Mốc hoàn thành bước 3**: ứng dụng khởi động, `http://localhost:9090/actuator/health` trả `{"status":"UP"}`, bảng đã có trong cơ sở dữ liệu.

---

## 6. Bước 4 — Lát cắt dọc đầu tiên: Lời nhắn

Viết đúng theo thứ tự này. Mỗi tệp xong thì biên dịch lại trước khi sang tệp tiếp theo.

### 6.1 Entity

```java
package vn.memorybook.message;

@Entity
@Table(name = "messages")
@Getter @Setter
@NoArgsConstructor
public class Message {

    @Id
    @GeneratedValue
    private UUID id;

    @Column(length = 60)
    private String fullname;

    @Column(length = 60)
    private String nickname;

    @Column(nullable = false, length = 500)
    private String body;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 16)
    private MessageStatus status = MessageStatus.PENDING;

    @Column(name = "ip_hash", length = 64)
    private String ipHash;

    @Column(name = "content_hash", nullable = false, length = 64)
    private String contentHash;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt = Instant.now();

    @Column(name = "updated_at", nullable = false)
    private Instant updatedAt = Instant.now();

    @Column(name = "deleted_at")
    private Instant deletedAt;

    @Version                       // khoá lạc quan: hai quản trị viên duyệt
    private Long version;          // cùng lúc thì người thứ hai nhận 409

    /**
     * Tên hiển thị tính lúc trả về, KHÔNG lưu vào cơ sở dữ liệu.
     * API công khai chỉ trả trường này, không trả fullname/nickname thô
     * — nguyên tắc tối thiểu hoá dữ liệu (NFR-PRV-01).
     */
    public String displayName() {
        if (nickname != null && !nickname.isBlank()) return nickname;
        if (fullname != null && !fullname.isBlank()) return fullname;
        return "Ẩn danh";
    }
}
```

### 6.2 DTO — không bao giờ để entity ra tới controller

```java
public record CreateMessageRequest(
    @Size(max = 60, message = "Họ tên không quá 60 ký tự")
    String fullname,

    @Size(max = 60, message = "Biệt danh không quá 60 ký tự")
    String nickname,

    @NotBlank(message = "Vui lòng nhập lời nhắn")
    @Size(max = 500, message = "Lời nhắn không quá 500 ký tự")
    String message,

    @NotBlank
    String turnstileToken
) {}

public record MessageResponse(
    UUID id,
    String displayName,
    String body,
    Instant createdAt
) {}
```

### 6.3 Repository — phân trang theo con trỏ

```java
public interface MessageRepository extends JpaRepository<Message, UUID> {

    /**
     * So sánh bộ đôi (created_at, id) — KHÔNG dùng OFFSET.
     * Ghép thêm id vào khoá sắp xếp là bắt buộc: hai lời nhắn có thể
     * trùng created_at tới micro giây, thứ tự không ổn định sẽ làm hỏng
     * phân trang (LLD mục 5.2).
     */
    @Query("""
        SELECT m FROM Message m
        WHERE m.status = 'PUBLISHED' AND m.deletedAt IS NULL
          AND (:cursorAt IS NULL
               OR m.createdAt < :cursorAt
               OR (m.createdAt = :cursorAt AND m.id < :cursorId))
        ORDER BY m.createdAt DESC, m.id DESC
        """)
    List<Message> findPublishedBefore(@Param("cursorAt") Instant cursorAt,
                                      @Param("cursorId") UUID cursorId,
                                      Pageable pageable);

    boolean existsByIpHashAndContentHashAndCreatedAtAfter(
            String ipHash, String contentHash, Instant since);
}
```

### 6.4 Service — nơi duy nhất có quy tắc nghiệp vụ

```java
@Service
@RequiredArgsConstructor
public class MessageService {

    private final MessageRepository repository;
    private final IpHasher ipHasher;
    private final TurnstileVerifier turnstile;
    private final Clock clock;

    private static final Pattern URL_PATTERN = Pattern.compile("https?://", CASE_INSENSITIVE);

    @Transactional
    public UUID submit(CreateMessageRequest req, String clientIp) {

        if (!turnstile.verify(req.turnstileToken(), clientIp)) {
            throw new InvalidCaptchaException();
        }

        // Chuẩn hoá Unicode về NFC và bỏ ký tự điều khiển / ký tự vô hình
        String body = normalize(req.message());

        if (countUrls(body) > 3) {
            throw new TooManyLinksException();
        }

        String ipHash = ipHasher.hash(clientIp);
        String contentHash = sha256(body);

        // Chống gửi trùng trong 24 giờ (FR-MSG-06)
        if (repository.existsByIpHashAndContentHashAndCreatedAtAfter(
                ipHash, contentHash, clock.instant().minus(Duration.ofDays(1)))) {
            throw new DuplicateContentException();
        }

        var message = new Message();
        message.setFullname(trimToNull(req.fullname()));
        message.setNickname(trimToNull(req.nickname()));
        message.setBody(body);
        message.setStatus(MessageStatus.PENDING);   // ← LUÔN chờ duyệt (FR-MSG-04)
        message.setIpHash(ipHash);
        message.setContentHash(contentHash);

        return repository.save(message).getId();
    }

    private static String normalize(String s) {
        return Normalizer.normalize(s.trim(), Normalizer.Form.NFC)
                         .replaceAll("[\\p{Cc}\\p{Cf}]", "");
    }
}
```

### 6.5 Controller — mỏng, chỉ ánh xạ HTTP

```java
@RestController
@RequestMapping("/api/v1/messages")
@RequiredArgsConstructor
public class MessageController {

    private final MessageService service;
    private final MessageQueryService queryService;

    @GetMapping
    public MessagePageResponse list(
            @RequestParam(required = false) String cursor,
            @RequestParam(defaultValue = "20") @Min(1) @Max(50) int limit) {
        return queryService.listPublished(cursor, limit);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.ACCEPTED)       // ← 202, KHÔNG phải 201.
    public Map<String, Object> submit(         //   Nội dung chưa được công bố.
            @Valid @RequestBody CreateMessageRequest req,
            HttpServletRequest http) {
        UUID id = service.submit(req, ClientIp.of(http));
        return Map.of("id", id, "status", "PENDING");
    }
}
```

### 6.6 Kiểm thử tích hợp — chạy trên PostgreSQL thật

```java
@SpringBootTest
@AutoConfigureMockMvc
@Testcontainers
class MessageIntegrationTest {

    @Container
    @ServiceConnection                          // Spring tự nối, không cần cấu hình tay
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:17-alpine");

    @Container
    @ServiceConnection
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
            .withExposedPorts(6379);

    @Autowired MockMvc mvc;

    @Test
    void loi_nhan_moi_gui_len_phai_o_trang_thai_cho_duyet() throws Exception {
        mvc.perform(post("/api/v1/messages")
                .contentType(APPLICATION_JSON)
                .content("""
                    {"fullname":"Nguyễn Bảo An","nickname":"An Mèo",
                     "message":"Nhớ cả lớp nhiều!","turnstileToken":"test"}
                    """))
           .andExpect(status().isAccepted())
           .andExpect(jsonPath("$.status").value("PENDING"));

        // Điều quan trọng nhất: nó KHÔNG được xuất hiện ở API công khai
        mvc.perform(get("/api/v1/messages"))
           .andExpect(jsonPath("$.items").isEmpty());
    }

    @Test
    void phan_trang_con_tro_khong_lap_khi_co_ban_ghi_moi_chen_vao() {
        // Đây là kiểm thử H2 không làm được — cần PostgreSQL thật
    }
}
```

**Mốc hoàn thành bước 4**: gửi được lời nhắn bằng `curl`, nó nằm trong cơ sở dữ liệu ở trạng thái `PENDING`, không hiện ở API công khai; đổi tay trong cơ sở dữ liệu sang `PUBLISHED` thì nó hiện ra.

---

## 7. Bước 5 — Thêm lớp bảo mật

Thứ tự này quan trọng: **bảo mật thêm ngay sau tính năng đầu tiên**, không để tới cuối. Thêm sau thì phải sửa lại mọi thứ đã viết.

| # | Việc | Tham chiếu |
|---|---|---|
| 1 | `SecurityConfig` với `.anyRequest().denyAll()` | [LLD](MemoryBook-LLD.md) 6.1 |
| 2 | `GlobalExceptionHandler` với Problem Details | [LLD](MemoryBook-LLD.md) 5.5 |
| 3 | `IpHasher` — HMAC-SHA256, không lưu IP thô | [LLD](MemoryBook-LLD.md) 6.2 |
| 4 | `RateLimitFilter` với Bucket4j + Redis | [LLD](MemoryBook-LLD.md) 6.2 |
| 5 | `ClientIp` — chỉ tin header khi đến từ proxy tin cậy | [LLD](MemoryBook-LLD.md) 6.2 |
| 6 | Kiểm thử: `/api/v1/admin/**` trả `401` | [Security](MemoryBook-Security.md) A01 |

Viết kiểm thử này ngay, và giữ nó mãi mãi:

```java
@Test
void moi_duong_dan_quan_tri_deu_bi_chan_khi_chua_dang_nhap() throws Exception {
    mvc.perform(get("/api/v1/admin/queue")).andExpect(status().isUnauthorized());
    mvc.perform(get("/api/v1/admin/audit-logs")).andExpect(status().isUnauthorized());
    mvc.perform(patch("/api/v1/admin/photos/" + UUID.randomUUID()))
       .andExpect(status().isUnauthorized());
}
```

---

## 8. Bước 6 — Quản trị và kiểm duyệt

Đây là mốc **dùng thật được**. Sau bước này, lớp có thể gửi lời nhắn và ban tổ chức duyệt trên điện thoại.

| # | Việc |
|---|---|
| 1 | Entity `AdminUser` + `AdminBackupCode` |
| 2 | Lệnh dòng lệnh tạo tài khoản quản trị đầu tiên (`CommandLineRunner` chỉ chạy khi có cờ) — **không** seed tài khoản trong tệp di trú |
| 3 | Đăng nhập: mật khẩu Argon2id → phiên tạm chờ 2FA → xác thực TOTP → cấp phiên mới |
| 4 | Sinh mã QR để quét bằng ứng dụng xác thực trên điện thoại |
| 5 | `GET /admin/queue` — hàng chờ gộp ảnh và lời nhắn |
| 6 | `PATCH /admin/messages/{id}` — duyệt / từ chối / ẩn |
| 7 | `AuditLogAspect` |
| 8 | Giao diện quản trị tối giản — **ưu tiên dùng được trên điện thoại**, không cần đẹp |

Chi tiết luồng đăng nhập ở [`LLD`](MemoryBook-LLD.md) mục 4.2. Nhớ chi tiết chống dò tên tài khoản: **luôn chạy phép băm kể cả khi tài khoản không tồn tại.**

---

## 9. Bước 7 — Ảnh (phần khó nhất)

Đọc kỹ [`MemoryBook-Storage-Media.md`](MemoryBook-Storage-Media.md) trước khi bắt đầu bước này — nó chứa các quyết định về quy mô 20 MB / 20 GB.

### 9.1 Thứ tự làm

```
[1] StorageService — giao diện trừu tượng cho kho đối tượng
    · put, get, head, delete, presignPut, presignGet, copy
    · Hiện thực bằng AWS SDK v2 → chạy được cả MinIO lẫn R2

[2] Điểm cuối upload-intent: tạo bản ghi + ký URL PUT
    · Kiểm tra hạn ngạch dung lượng TRƯỚC khi ký

[3] Client PUT thẳng lên MinIO (thử bằng curl trước, chưa cần giao diện)

[4] Điểm cuối complete: HEAD kiểm tra kích thước
    · GET Range: bytes=0-31 → đọc magic bytes
    · từ chối SVG, kiểm tra số điểm ảnh, tính hash, chống trùng
    · INSERT photos + INSERT outbox — CÙNG MỘT GIAO DỊCH

[5] Bảng outbox + poller với FOR UPDATE SKIP LOCKED

[6] Tích hợp ClamAV

[7] VipsImageProcessor — gọi vipsthumbnail qua tiến trình con
    · CÓ thời gian chờ, CÓ đọc hết luồng đầu ra

[8] Sinh 3 kích cỡ × 2 định dạng, ghi vào public/

[9] GET /photos — trả kèm width, height, aspectRatio
```

### 9.2 Cài libvips cho môi trường phát triển

```powershell
# Windows: tải bản dựng sẵn từ trang phát hành chính thức của libvips,
# giải nén rồi thêm thư mục bin vào PATH
# Kiểm tra:
vips --version
vipsthumbnail --help
```

Trong Docker thì thêm vào Dockerfile:

```dockerfile
RUN apt-get update \
 && apt-get install -y --no-install-recommends libvips-tools \
 && rm -rf /var/lib/apt/lists/*
ENV VIPS_CONCURRENCY=1
```

### 9.3 Thử bằng tay trước khi viết mã

Luôn chạy thử lệnh bằng tay trước, rồi mới bọc vào Java. Nếu lệnh sai, gỡ lỗi ở dòng lệnh nhanh hơn nhiều so với gỡ lỗi qua `ProcessBuilder`:

```bash
# Đọc kích thước mà không giải mã
vipsheader -f width  anh-that-20mb.jpg
vipsheader -f height anh-that-20mb.jpg

# Sinh biến thể, tự xoay theo EXIF, bỏ sạch siêu dữ liệu
vipsthumbnail anh-that-20mb.jpg --size 1600x1600 -o "medium.avif[Q=55,strip]"

# Kiểm chứng siêu dữ liệu ĐÃ biến mất — bước này BẮT BUỘC làm
exiftool medium.avif | grep -i -E "gps|make|model|serial"
# Không ra dòng nào = đúng
```

**Dùng ảnh thật 20 MB từ điện thoại của bạn để thử, không dùng ảnh mẫu nhỏ.** Toàn bộ vấn đề về bộ nhớ và HEIC chỉ lộ ra với ảnh thật.

---

## 10. Bước 8 — Front-end React

Chỉ bắt đầu sau khi API đã chạy được. Front-end nối vào API có sẵn thì dễ; làm ngược lại thì cả hai đều dở dang.

```bash
cd client
npm create vite@latest app -- --template react-ts
cd app
npm install

npm install @tanstack/react-query react-router-dom zustand \
            react-hook-form zod @hookform/resolvers
npm install -D tailwindcss @tailwindcss/vite openapi-typescript \
               vitest @testing-library/react @playwright/test
```

Sinh kiểu dữ liệu từ tài liệu OpenAPI của Spring Boot — làm việc này **trước khi** viết lời gọi API:

```bash
npx openapi-typescript http://localhost:8080/v3/api-docs -o src/api/schema.d.ts
```

Từ đó trở đi, đổi API mà quên sửa front-end sẽ là lỗi biên dịch, không phải lỗi lúc chạy.

Thứ tự chuyển từ bản dựng tạm: theo bảng đối chiếu ở [`TechStack`](MemoryBook-TechStack.md) mục 3.4. Làm theo thứ tự **Lời nhắn → Thư viện ảnh → Đóng góp → Tranh động SVG → Khu quản trị**, chuyển xong phần nào thì xoá phần tương ứng khỏi `index.html` cũ.

Cấu hình proxy để phát triển không vướng CORS:

```ts
// vite.config.ts
export default defineConfig({
  server: {
    proxy: { '/api': { target: 'http://localhost:8080', changeOrigin: true } }
  }
});
```

---

## 11. Bước 9 — Triển khai lên VPS

### 11.1 Chuẩn bị máy chủ

```bash
# Trên VPS mới, làm ngay và làm đủ:
adduser deploy && usermod -aG sudo,docker deploy
# chép khoá công khai SSH vào /home/deploy/.ssh/authorized_keys

# Tắt đăng nhập bằng mật khẩu — đây là việc quan trọng nhất trong cả mục này
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl restart ssh

sudo ufw default deny incoming
sudo ufw allow 22/tcp && sudo ufw allow 80/tcp && sudo ufw allow 443/tcp
sudo ufw enable

sudo apt install -y fail2ban unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

> **Trước khi đóng cửa sổ SSH hiện tại, hãy mở một cửa sổ SSH thứ hai và xác nhận đăng nhập được.** Cấu hình sai SSH rồi đóng phiên duy nhất là cách nhanh nhất để tự khoá mình khỏi máy chủ.

### 11.2 Caddyfile

```
app.memorybook.example {
    encode zstd gzip

    handle /api/* {
        reverse_proxy app:8080
    }

    handle {
        root * /srv/www
        try_files {path} /index.html      # điều hướng phía client
        file_server
    }

    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains"
        X-Content-Type-Options    "nosniff"
        Referrer-Policy           "strict-origin-when-cross-origin"
        Permissions-Policy        "camera=(), microphone=(), geolocation=()"
        X-Robots-Tag              "noindex, noimageindex, nofollow"
        Content-Security-Policy   "default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' https://img.memorybook.example data:; font-src 'self'; connect-src 'self' https://challenges.cloudflare.com; frame-src https://challenges.cloudflare.com; object-src 'none'; base-uri 'self'; form-action 'self'; frame-ancestors 'none'"
        -Server
    }
}
```

Caddy tự xin và gia hạn chứng chỉ Let's Encrypt — không phải làm gì thêm.

### 11.3 Khác biệt của `docker-compose.prod.yml`

| So với bản dev | Thay đổi |
|---|---|
| MinIO | **Bỏ hẳn** — dùng Cloudflare R2 |
| Cổng | Chỉ `caddy` mở 80/443. PostgreSQL, Redis **không có mục `ports`** |
| Bí mật | Đọc từ `.env` trên máy chủ, quyền tệp `600` |
| Tài nguyên | Đặt `mem_limit` cho từng dịch vụ |
| Bảo mật container | `read_only: true`, `no-new-privileges`, `cap_drop: [ALL]` |
| Khởi động lại | `restart: unless-stopped` |

```yaml
  app:
    image: ghcr.io/<user>/memorybook-server:${TAG}
    env_file: .env
    mem_limit: 2g
    read_only: true
    tmpfs:
      - /tmp:size=2g          # libvips cần chỗ ghi tạm
    security_opt:
      - no-new-privileges:true
    cap_drop: [ALL]
    restart: unless-stopped
    # KHÔNG có mục ports — chỉ Caddy trong mạng nội bộ gọi tới
```

### 11.4 Sao lưu

```bash
#!/usr/bin/env bash
# /home/deploy/backup.sh — chạy bằng cron lúc 02:00 hằng ngày
set -euo pipefail

STAMP=$(date +%Y%m%d-%H%M)

# Cơ sở dữ liệu: nhỏ, sao lưu hằng ngày
docker compose exec -T postgres pg_dump -U memorybook memorybook \
  | gzip \
  | gpg --encrypt --recipient backup@memorybook.example \
  > "/tmp/db-${STAMP}.sql.gz.gpg"

rclone copyto "/tmp/db-${STAMP}.sql.gz.gpg" \
  "b2:memorybook-backup/db/db-${STAMP}.sql.gz.gpg"
rm -f "/tmp/db-${STAMP}.sql.gz.gpg"

# Giữ 14 bản gần nhất
rclone delete b2:memorybook-backup/db --min-age 14d

# Ảnh gốc: bất biến — dùng copy, TUYỆT ĐỐI không dùng sync
rclone copy r2:memorybook-archive b2:memorybook-backup/archive \
  --transfers 4 --checksum

# Báo cho hệ thống giám sát biết sao lưu đã chạy xong.
# Không có dòng này thì sao lưu hỏng âm thầm — tình huống tệ nhất.
curl -fsS "https://<dịch-vụ-giám-sát>/ping/<uuid>" > /dev/null
```

---

## 12. Gặp lỗi này thì làm gì

| Triệu chứng | Nguyên nhân thường gặp |
|---|---|
| `Flyway: Validate failed` | Đã sửa tệp di trú cũ. **Không bao giờ sửa tệp di trú đã chạy** — tạo tệp `V2__` mới |
| `LazyInitializationException` | Đã tắt `open-in-view` (đúng), nhưng service trả entity ra ngoài giao dịch. Sửa: ánh xạ sang DTO **bên trong** service |
| Truy vấn nhiều bất thường trong log | Vấn đề N+1. Dùng `JOIN FETCH` hoặc `@EntityGraph` |
| `OutOfMemoryError` khi xử lý ảnh | Còn dùng `ImageIO`. Chuyển sang libvips ([Storage](MemoryBook-Storage-Media.md) mục 5) |
| Tiến trình xử lý ảnh treo | Thiếu thời gian chờ, hoặc không đọc hết luồng đầu ra của tiến trình con |
| Ảnh dọc bị nằm ngang | Bỏ EXIF trước khi xoay. `vipsthumbnail` xử lý đúng thứ tự — hãy dùng nó |
| Ảnh iPhone không mở được | HEIC. libvips cần được dựng kèm libheif |
| `403` khi PUT vào MinIO | Thiếu `forcePathStyle(true)` cho MinIO |
| `403` khi PUT vào R2 | Còn để `forcePathStyle(true)`. R2 dùng `false` và `region = "auto"` |
| Giới hạn tần suất không có tác dụng | Bộ đếm đang ở bộ nhớ tiến trình, hoặc đang tin `X-Forwarded-For` vô điều kiện |
| Bấm gửi thì lỗi `403` | Thiếu token CSRF. Đọc cookie `XSRF-TOKEN`, gửi lại qua header `X-XSRF-TOKEN` |
| CSP chặn mất giao diện | Còn thư viện nạp từ CDN, hoặc còn `<script>` nội tuyến. Đây là điều CSP **phải** làm |
| Ảnh "đã gỡ" vẫn xem được | Chưa xoá đệm Cloudflare. Bước bắt buộc trong luồng gỡ |

---

## 13. Bảng theo dõi tiến độ

```
GIAI ĐOẠN 0 — Chuẩn bị
  [ ] Máy đã cài đủ công cụ
  [ ] docker-compose.dev.yml chạy được, mọi dịch vụ healthy
  [ ] Dự án Spring Boot khởi động được
  [ ] V1__init.sql chạy xong, bảng đã có
  [ ] .env trong .gitignore, .env.example đã commit

GIAI ĐOẠN 1 — Lời nhắn                    ← bộ khung đúng hay sai lộ ra ở đây
  [ ] Entity + Repository + Service + Controller
  [ ] Phân trang theo con trỏ
  [ ] Kiểm thử tích hợp với Testcontainers
  [ ] Nối vào client/index.html

GIAI ĐOẠN 2 — Bảo mật + Quản trị          ← 🎉 MỐC DÙNG THẬT ĐƯỢC
  [ ] SecurityConfig với denyAll
  [ ] Đăng nhập + TOTP
  [ ] Hàng chờ kiểm duyệt
  [ ] Nhật ký kiểm toán
  [ ] Giao diện quản trị dùng được trên điện thoại

GIAI ĐOẠN 3 — Ảnh                         ← phần khó nhất
  [ ] StorageService chạy được với MinIO
  [ ] Tải lên bằng URL ký sẵn
  [ ] Kiểm tra magic bytes qua yêu cầu Range
  [ ] Outbox + poller
  [ ] libvips sinh biến thể
  [ ] ĐÃ KIỂM CHỨNG bằng exiftool: siêu dữ liệu biến mất
  [ ] ĐÃ THỬ với ảnh thật 20 MB từ điện thoại

GIAI ĐOẠN 4 — Vận hành
  [ ] Đã đi hết checklist Security mục 11
  [ ] Sao lưu chạy + ĐÃ PHỤC HỒI THỬ THÀNH CÔNG
  [ ] Giám sát + cảnh báo
  [ ] Tên miền + HTTPS

GIAI ĐOẠN 5 — React
  [ ] Vite + React + TypeScript
  [ ] Kiểu dữ liệu sinh từ OpenAPI
  [ ] Chuyển hết từng phần
  [ ] Bỏ sạch CDN bên thứ ba, CSP nghiêm ngặt
  [ ] Lighthouse đạt ngưỡng

GIAI ĐOẠN 6 — Hoàn thiện
  [ ] Tải album theo phân loại
  [ ] Dữ liệu cấu hình (danh sách lớp, hành trình)
  [ ] Thay hết ảnh tạm bằng ảnh thật
```

---

*Tài liệu này hiện thực hoá lộ trình ở [`MemoryBook-Workflow-Docs.md`](MemoryBook-Workflow-Docs.md) mục 3. Quyết định về lưu trữ và xử lý ảnh: [`MemoryBook-Storage-Media.md`](MemoryBook-Storage-Media.md).*
