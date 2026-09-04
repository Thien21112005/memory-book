# Phượng Hồng Memories — Back-end

The API that replaces the front-end's browser-only storage. Designed, not yet implemented.

API thay thế cơ chế lưu tạm trên trình duyệt của phần front-end. Đã thiết kế xong, chưa hiện thực.

**[English](#english) · [Tiếng Việt](#tiếng-việt)** · [Project index](../README.md) · [Front-end](../client/README.md) · [Engineering docs](../docs/README.md)

---

> ### ⚠️ Status / Trạng thái
>
> **EN** — This directory is **empty**. No code has been written. The design is complete and lives in [`../docs/`](../docs/README.md); this file is the summary you need when you open this folder.
>
> **VI** — Thư mục này **đang trống**. Chưa có dòng mã nào. Thiết kế đã hoàn chỉnh và nằm ở [`../docs/`](../docs/README.md); file này là bản tóm tắt cần đọc khi mở thư mục này ra.

---

# English

## What goes here

A **Spring Boot** application that owns every business rule in the system. The front-end is treated as an untrusted environment: it makes the experience pleasant, it does not enforce anything.

Responsibilities:

| Area | What the server does |
| --- | --- |
| Messages | Store, moderate and serve the shared message board |
| Photos | Issue presigned upload URLs, validate, scan, re-encode, generate variants, moderate |
| Gallery | Serve the published archive as data so the page updates without a code change |
| Album | Package approved photographs into downloadable archives |
| Admin | Password + TOTP authentication, moderation queue, audit log |
| Privacy | Takedown requests, IP hashing, retention limits |

## Why a back-end is needed

The front-end works as a static page, but four features are currently simulated in the browser:

| Feature | Current behaviour | What the back-end adds |
| --- | --- | --- |
| Message board | Saved to `localStorage` under `pkm-notes`, capped at 40 | Shared, permanent storage visible to every visitor |
| Photo contribution | The form previews the file locally and never sends it | Real upload with moderation before publication |
| Gallery and carousel | Photo URLs hard-coded in `index.html` | Photos served from the archive |
| Album download | The "Tải xuống (HD)" button is presentational | Archives of the approved photographs |

Visitor display names (`pkm-name`) stay in `localStorage`. The site has no accounts for contributors and deliberately does not need any — see [design decision DD7](../docs/MemoryBook-HLD.md).

## Planned stack

| Layer | Choice |
| --- | --- |
| Runtime | Java LTS (21 or 25), Spring Boot, Web MVC on virtual threads |
| Security | Spring Security 6 — session cookies in Redis, CSRF, TOTP two-factor |
| Persistence | PostgreSQL + Spring Data JPA, Flyway migrations |
| Cache / sessions / rate limits | Redis, Bucket4j |
| Object storage | Cloudflare R2 in production, MinIO in development (identical S3 protocol) |
| Image processing | **libvips** via subprocess — not a pure-Java library |
| Malware scanning | ClamAV |
| Observability | Actuator + Micrometer → Prometheus, structured JSON logs |
| Build | Maven; multi-stage Dockerfile, non-root user |

Rationale for each choice, and the list of technologies deliberately **not** used, is in [`../docs/MemoryBook-TechStack.md`](../docs/MemoryBook-TechStack.md).

> **Why libvips and not Thumbnailator/imgscalr**: photographs in this project reach 25 MB and 24–50 megapixels. `ImageIO` decodes an image fully into memory — roughly 200 MB of heap for a single 50 MP photograph — which will exhaust a small VPS. libvips streams, and its memory use is nearly independent of input size. See [`../docs/MemoryBook-Storage-Media.md`](../docs/MemoryBook-Storage-Media.md) §5.

## API contract

Base path `/api/v1`. JSON unless stated otherwise, timestamps ISO 8601 UTC, cursor-based pagination, errors as RFC 9457 Problem Details.

The full contract with request and response bodies is in [`../docs/MemoryBook-LLD.md`](../docs/MemoryBook-LLD.md) §5. Summary:

### Public

```http
GET    /api/v1/messages?limit=20&cursor=…
POST   /api/v1/messages                      → 202, always PENDING

POST   /api/v1/photos/upload-intent          → presigned PUT URL, 10 min
PUT    <presigned url>                       → browser uploads DIRECTLY to storage
POST   /api/v1/photos/{id}/complete          → 202, always PENDING

GET    /api/v1/photos?category=…&limit=…&cursor=…
GET    /api/v1/album?category=…              → 302 to a signed URL

GET    /api/v1/config/members
GET    /api/v1/config/timeline
GET    /api/v1/config/stats

POST   /api/v1/takedown-requests             → 202 with a lookup reference
GET    /api/v1/takedown-requests/{reference} → status only
```

### Admin — session required, role `ADMIN`

```http
POST   /api/v1/admin/auth/login              → { mfaRequired: true }
POST   /api/v1/admin/auth/mfa                → sets session cookie
POST   /api/v1/admin/auth/logout

GET    /api/v1/admin/queue?type=…&status=…
PATCH  /api/v1/admin/photos/{id}
PATCH  /api/v1/admin/messages/{id}
DELETE /api/v1/admin/{photos|messages}/{id}  → soft delete, 30-day recovery
POST   /api/v1/admin/{photos|messages}/{id}/restore
POST   /api/v1/admin/photos/bulk

GET    /api/v1/admin/takedown-requests?status=open
PATCH  /api/v1/admin/takedown-requests/{id}
GET    /api/v1/admin/audit-logs
POST   /api/v1/admin/album/rebuild
```

### Two contract details that are easy to get wrong

**Uploads are two-step, and bytes never pass through this application.** At 25 MB per photograph, routing files through the server pays for the bandwidth twice and occupies resources for 30–60 seconds per upload. The client asks for a presigned URL, uploads directly to object storage, then confirms. Validation happens on the stored object — magic bytes are read with a `GET Range: bytes=0-31` request, so 32 bytes are fetched rather than 25 MB. The security model for this is in [`../docs/MemoryBook-Security.md`](../docs/MemoryBook-Security.md) §4.6.

**Submissions return `202`, never `201`.** Nothing a visitor submits is published without a human approving it. The response must not pretend otherwise; the UI says "waiting for approval", because a contributor who believes their message failed will submit it three more times.

## Data model

Seven tables. Full DDL with indexes and constraints in [`../docs/MemoryBook-LLD.md`](../docs/MemoryBook-LLD.md) §2.

```
messages           id · fullname · nickname · body · status · ip_hash
                   content_hash · created_at · deleted_at

photos             id · storage_key · mime_type · byte_size · width · height
                   content_hash · note · taken_at_text · category
                   status · processing · consent_given · published_at · deleted_at

photo_variants     photo_id · variant (THUMB|MEDIUM|LARGE)
                   format (AVIF|JPEG) · storage_key · width · height

admin_users        username · password_hash (Argon2id) · totp_secret_enc
                   failed_attempts · locked_until

admin_backup_codes admin_id · code_hash · used_at

audit_logs         actor_id · actor_name · action · entity_type · entity_id
                   before_state · after_state        ← append-only, enforced in SQL

takedown_requests  reference · target_type · target_id · reason · status

outbox_jobs        job_type · payload · status · attempts · run_after · locked_by
```

Conventions: UUID primary keys, soft delete via `deleted_at` with 30-day recovery, status as `VARCHAR` + `CHECK` rather than a PostgreSQL `ENUM`, and partial indexes on the hot public queries.

Two constraints worth stating here because they are easy to skip:

- **IP addresses are never stored raw.** Only `HMAC-SHA256(ip, secret)`. A plain SHA-256 would be useless — the entire IPv4 space is 2³² values and a lookup table is buildable in hours.
- **`audit_logs` is append-only at the database level**, not merely by convention: `REVOKE UPDATE, DELETE ON audit_logs FROM app_user`. The difference matters when the application itself is compromised.

## Object storage layout

Three zones. Details in [`../docs/MemoryBook-Storage-Media.md`](../docs/MemoryBook-Storage-Media.md) §2.

| Zone | Contents | Readable by |
| --- | --- | --- |
| `quarantine/` | Freshly uploaded, unscanned. Lifecycle-deleted after 24 h | The application only |
| `archive/` | **Original bytes, untouched**, EXIF intact, ~20 GB | **Nobody** — not even the uploader |
| `public/` | Variants, metadata stripped, ~4 GB | Public, via `img.<domain>` |

The `archive/` and `public/` split exists because originals carry GPS coordinates. Serving them publicly would leak where students were photographed.

## Local development

```bash
# 1. Infrastructure — PostgreSQL, Redis, MinIO, ClamAV
docker compose -f ../docker-compose.dev.yml up -d

# 2. The API
cd server
./mvnw spring-boot:run          # http://localhost:8080

# 3. The front-end preview
cd ../client
python -m http.server 5173      # http://localhost:5173
```

Configuration is read from the environment; a `.env.example` lists every variable with no values. **Never commit `.env`.**

Step-by-step build instructions, including the `docker-compose.dev.yml` file itself, are in [`../docs/MemoryBook-Build-Guide.md`](../docs/MemoryBook-Build-Guide.md).

## Front-end integration points

The exact places in `client/index.html` that change:

| Behaviour | Current code | Replace with |
| --- | --- | --- |
| Load messages | `loadSavedNotes()` reads `localStorage` | `GET /api/v1/messages` |
| Submit a message | `saveNote()` writes `localStorage` | `POST /api/v1/messages`, show "awaiting approval" |
| Submit a photo | `contribute-form` handler calls `openSuccess()` only | `upload-intent` → direct `PUT` → `complete` |
| Gallery contents | Hard-coded `<img>` and `<figure>` markup | Render from `GET /api/v1/photos` |
| Album download | Button has no handler | Link to `GET /api/v1/album?category=…` |
| Class roster | `DANH_SACH_LOP` array in `<script>` | `GET /api/v1/config/members` |

Two front-end changes are **required** and currently missing:

1. The four category chips in `#contribute-categories` change appearance but are never submitted. A hidden input must carry the selection.
2. There is no consent checkbox. One is mandatory — the contributor confirms they may share the photograph and that the people in it agree.

`createNote(dateStr, text, name, styleIndex)` already builds elements with `textContent`, so server-provided strings stay safe to render without escaping work.

## Non-negotiables

These are requirements, not suggestions. Each is verifiable, and the checklist form is in [`../docs/MemoryBook-Security.md`](../docs/MemoryBook-Security.md) §11.

- Everything a visitor submits starts as `PENDING`. No exceptions.
- `.anyRequest().denyAll()` — a route nobody declared must be blocked, not accidentally public.
- Uploaded file types are determined from **magic bytes**, never from the `Content-Type` header. SVG is rejected outright: it is an executable XML document and a classic XSS vector.
- Images are re-encoded, and rotated per EXIF Orientation **before** metadata is stripped. Reverse that order and every portrait photograph comes out sideways.
- Rate limiting is by hashed IP in Redis, and `CF-Connecting-IP` is trusted **only** when the connection comes from a configured proxy range.
- Error responses never contain stack traces, Java class names, library versions or SQL.
- Backups run nightly to a **different provider**, and a restore has actually been rehearsed. A backup never restored is not a backup.

---

# Tiếng Việt

## Thư mục này chứa gì

Một ứng dụng **Spring Boot** giữ toàn bộ logic nghiệp vụ của hệ thống. Front-end được coi là môi trường không đáng tin: nó làm cho trải nghiệm dễ chịu, nó không thực thi quy tắc nào.

Trách nhiệm:

| Mảng | Máy chủ làm gì |
| --- | --- |
| Lời nhắn | Lưu, kiểm duyệt và cung cấp bảng lời nhắn chung |
| Ảnh | Cấp đường dẫn tải lên ký sẵn, kiểm tra, quét, mã hoá lại, sinh biến thể, kiểm duyệt |
| Thư viện | Cung cấp kho ảnh dưới dạng dữ liệu để trang cập nhật mà không phải sửa mã |
| Album | Đóng gói ảnh đã duyệt thành tệp nén tải về |
| Quản trị | Xác thực mật khẩu + TOTP, hàng chờ kiểm duyệt, nhật ký kiểm toán |
| Quyền riêng tư | Yêu cầu gỡ nội dung, băm IP, thời hạn lưu trữ |

## Vì sao cần back-end

Front-end chạy trọn vẹn dưới dạng trang tĩnh, nhưng bốn tính năng hiện chỉ được mô phỏng trên trình duyệt:

| Tính năng | Hiện tại | Back-end bổ sung |
| --- | --- | --- |
| Bảng lời nhắn | Lưu vào `localStorage` khoá `pkm-notes`, giới hạn 40 mục | Kho lưu trữ chung, lâu dài, mọi người đều thấy |
| Đóng góp ảnh | Biểu mẫu chỉ xem trước tại máy, không gửi đi đâu | Tải ảnh lên thật, có duyệt trước khi công bố |
| Thư viện và băng chuyền | Đường dẫn ảnh viết cứng trong `index.html` | Lấy ảnh từ kho |
| Tải album | Nút "Tải xuống (HD)" chỉ mang tính minh hoạ | Tệp nén chứa ảnh đã duyệt |

Tên hiển thị của người xem (`pkm-name`) tiếp tục để ở `localStorage`. Trang không có tài khoản cho người đóng góp và **cố tình** không cần — xem [quyết định thiết kế DD7](../docs/MemoryBook-HLD.md).

## Công nghệ dự kiến

| Tầng | Lựa chọn |
| --- | --- |
| Nền tảng | Java LTS (21 hoặc 25), Spring Boot, Web MVC trên virtual threads |
| Bảo mật | Spring Security 6 — cookie phiên lưu ở Redis, CSRF, xác thực hai yếu tố TOTP |
| Lưu trữ dữ liệu | PostgreSQL + Spring Data JPA, di trú bằng Flyway |
| Cache / phiên / giới hạn tần suất | Redis, Bucket4j |
| Kho đối tượng | Cloudflare R2 cho môi trường thật, MinIO cho phát triển (cùng giao thức S3) |
| Xử lý ảnh | **libvips** gọi qua tiến trình con — không dùng thư viện thuần Java |
| Quét mã độc | ClamAV |
| Giám sát | Actuator + Micrometer → Prometheus, log JSON có cấu trúc |
| Build | Maven; Dockerfile nhiều tầng, chạy bằng người dùng thường |

Lý do chọn từng thứ, và danh sách những công nghệ **cố tình không dùng**, nằm ở [`../docs/MemoryBook-TechStack.md`](../docs/MemoryBook-TechStack.md).

> **Vì sao libvips chứ không phải Thumbnailator/imgscalr**: ảnh trong dự án này tới 25 MB và 24–50 megapixel. `ImageIO` giải mã toàn bộ ảnh vào bộ nhớ — khoảng 200 MB heap cho **một** ảnh 50 MP — và sẽ làm hết bộ nhớ VPS nhỏ. libvips xử lý theo luồng, bộ nhớ gần như không phụ thuộc kích thước đầu vào. Xem [`../docs/MemoryBook-Storage-Media.md`](../docs/MemoryBook-Storage-Media.md) mục 5.

## Hợp đồng API

Đường dẫn gốc `/api/v1`. Mặc định JSON, thời gian theo ISO 8601 UTC, phân trang theo con trỏ, lỗi theo RFC 9457 Problem Details.

Hợp đồng đầy đủ kèm thân yêu cầu và phản hồi nằm ở [`../docs/MemoryBook-LLD.md`](../docs/MemoryBook-LLD.md) mục 5. Tóm tắt:

### Công khai

```http
GET    /api/v1/messages?limit=20&cursor=…
POST   /api/v1/messages                      → 202, luôn ở trạng thái PENDING

POST   /api/v1/photos/upload-intent          → trả URL PUT ký sẵn, hạn 10 phút
PUT    <url ký sẵn>                          → trình duyệt tải THẲNG vào kho
POST   /api/v1/photos/{id}/complete          → 202, luôn ở trạng thái PENDING

GET    /api/v1/photos?category=…&limit=…&cursor=…
GET    /api/v1/album?category=…              → 302 tới đường dẫn ký sẵn

GET    /api/v1/config/members
GET    /api/v1/config/timeline
GET    /api/v1/config/stats

POST   /api/v1/takedown-requests             → 202 kèm mã tra cứu
GET    /api/v1/takedown-requests/{reference} → chỉ trả trạng thái
```

### Quản trị — cần phiên đăng nhập, vai trò `ADMIN`

```http
POST   /api/v1/admin/auth/login              → { mfaRequired: true }
POST   /api/v1/admin/auth/mfa                → đặt cookie phiên
POST   /api/v1/admin/auth/logout

GET    /api/v1/admin/queue?type=…&status=…
PATCH  /api/v1/admin/photos/{id}
PATCH  /api/v1/admin/messages/{id}
DELETE /api/v1/admin/{photos|messages}/{id}  → xoá mềm, khôi phục trong 30 ngày
POST   /api/v1/admin/{photos|messages}/{id}/restore
POST   /api/v1/admin/photos/bulk

GET    /api/v1/admin/takedown-requests?status=open
PATCH  /api/v1/admin/takedown-requests/{id}
GET    /api/v1/admin/audit-logs
POST   /api/v1/admin/album/rebuild
```

### Hai chi tiết trong hợp đồng dễ làm sai

**Tải ảnh đi qua hai bước, và bytes không bao giờ đi qua ứng dụng này.** Ở mức 25 MB mỗi ảnh, cho tệp đi qua máy chủ là trả tiền băng thông hai lần và chiếm dụng tài nguyên 30–60 giây mỗi lượt. Client xin đường dẫn ký sẵn, tải thẳng lên kho đối tượng, rồi mới xác nhận. Việc kiểm tra chạy trên đối tượng đã lưu — magic bytes đọc bằng yêu cầu `GET Range: bytes=0-31`, tức chỉ tải về 32 byte thay vì 25 MB. Mô hình bảo mật cho luồng này ở [`../docs/MemoryBook-Security.md`](../docs/MemoryBook-Security.md) mục 4.6.

**Nội dung gửi lên trả `202`, không bao giờ trả `201`.** Không có thứ gì khách gửi được công bố mà chưa qua người thật duyệt. Phản hồi không được giả vờ ngược lại; giao diện phải nói "đang chờ duyệt" — vì người gửi mà tưởng là lỗi thì sẽ bấm gửi thêm ba lần nữa.

## Mô hình dữ liệu

Bảy bảng. DDL đầy đủ kèm chỉ mục và ràng buộc ở [`../docs/MemoryBook-LLD.md`](../docs/MemoryBook-LLD.md) mục 2.

```
messages           id · fullname · nickname · body · status · ip_hash
                   content_hash · created_at · deleted_at

photos             id · storage_key · mime_type · byte_size · width · height
                   content_hash · note · taken_at_text · category
                   status · processing · consent_given · published_at · deleted_at

photo_variants     photo_id · variant (THUMB|MEDIUM|LARGE)
                   format (AVIF|JPEG) · storage_key · width · height

admin_users        username · password_hash (Argon2id) · totp_secret_enc
                   failed_attempts · locked_until

admin_backup_codes admin_id · code_hash · used_at

audit_logs         actor_id · actor_name · action · entity_type · entity_id
                   before_state · after_state        ← chỉ thêm, ép ở tầng SQL

takedown_requests  reference · target_type · target_id · reason · status

outbox_jobs        job_type · payload · status · attempts · run_after · locked_by
```

Quy ước: khoá chính UUID, xoá mềm bằng `deleted_at` với 30 ngày khôi phục, trạng thái dùng `VARCHAR` + `CHECK` thay vì `ENUM` của PostgreSQL, và chỉ mục một phần cho các truy vấn công khai nóng.

Hai ràng buộc cần nói rõ ở đây vì rất dễ bỏ qua:

- **Không bao giờ lưu địa chỉ IP thô.** Chỉ lưu `HMAC-SHA256(ip, khoá_bí_mật)`. Dùng SHA-256 trần là vô dụng — toàn bộ không gian IPv4 chỉ có 2³² giá trị, bảng tra cứu dựng được trong vài giờ.
- **`audit_logs` chỉ được thêm mới, ép ở tầng cơ sở dữ liệu** chứ không phải bằng quy ước: `REVOKE UPDATE, DELETE ON audit_logs FROM app_user`. Khác biệt này quan trọng đúng vào lúc chính ứng dụng bị chiếm.

## Bố cục kho đối tượng

Ba vùng. Chi tiết ở [`../docs/MemoryBook-Storage-Media.md`](../docs/MemoryBook-Storage-Media.md) mục 2.

| Vùng | Nội dung | Ai đọc được |
| --- | --- | --- |
| `quarantine/` | Vừa tải lên, chưa quét. Tự xoá sau 24 giờ | Chỉ ứng dụng |
| `archive/` | **Bytes gốc y nguyên**, còn EXIF, ~20 GB | **Không ai** — kể cả người vừa tải lên |
| `public/` | Biến thể đã bỏ siêu dữ liệu, ~4 GB | Công khai qua `img.<domain>` |

Việc tách `archive/` và `public/` tồn tại vì ảnh gốc mang theo toạ độ GPS. Phục vụ chúng ra công khai là làm lộ nơi học sinh được chụp ảnh.

## Chạy khi phát triển

```bash
# 1. Hạ tầng — PostgreSQL, Redis, MinIO, ClamAV
docker compose -f ../docker-compose.dev.yml up -d

# 2. API
cd server
./mvnw spring-boot:run          # http://localhost:8080

# 3. Bản dựng tạm của front-end
cd ../client
python -m http.server 5173      # http://localhost:5173
```

Cấu hình đọc từ biến môi trường; tệp `.env.example` liệt kê đủ tên biến mà không có giá trị. **Tuyệt đối không commit `.env`.**

Hướng dẫn dựng từng bước, kèm cả nội dung `docker-compose.dev.yml`, nằm ở [`../docs/MemoryBook-Build-Guide.md`](../docs/MemoryBook-Build-Guide.md).

## Điểm tích hợp phía front-end

Những chỗ chính xác trong `client/index.html` sẽ phải sửa:

| Hành vi | Mã hiện tại | Thay bằng |
| --- | --- | --- |
| Nạp lời nhắn | `loadSavedNotes()` đọc `localStorage` | `GET /api/v1/messages` |
| Gửi lời nhắn | `saveNote()` ghi `localStorage` | `POST /api/v1/messages`, hiện "đang chờ duyệt" |
| Gửi ảnh | Trình xử lý `contribute-form` chỉ gọi `openSuccess()` | `upload-intent` → `PUT` trực tiếp → `complete` |
| Nội dung thư viện | Thẻ `<img>` và `<figure>` viết cứng | Dựng từ `GET /api/v1/photos` |
| Tải album | Nút chưa gắn xử lý | Trỏ tới `GET /api/v1/album?category=…` |
| Danh sách lớp | Mảng `DANH_SACH_LOP` trong `<script>` | `GET /api/v1/config/members` |

Hai thay đổi phía front-end là **bắt buộc** và hiện đang thiếu:

1. Bốn nút phân loại trong `#contribute-categories` chỉ đổi giao diện chứ không bao giờ được gửi đi. Cần một trường ẩn mang giá trị đang chọn.
2. Chưa có ô đánh dấu đồng thuận. Ô này là bắt buộc — người gửi xác nhận có quyền chia sẻ ảnh và những người trong ảnh đồng ý.

Hàm `createNote(dateStr, text, name, styleIndex)` vốn đã dựng phần tử bằng `textContent`, nên chuỗi do máy chủ trả về vẫn hiển thị an toàn mà không cần thêm bước khử ký tự đặc biệt.

## Những điều không thương lượng

Đây là yêu cầu, không phải gợi ý. Mỗi mục đều kiểm chứng được, và dạng checklist nằm ở [`../docs/MemoryBook-Security.md`](../docs/MemoryBook-Security.md) mục 11.

- Mọi thứ khách gửi lên đều bắt đầu ở trạng thái `PENDING`. Không có ngoại lệ.
- `.anyRequest().denyAll()` — đường dẫn nào chưa ai khai báo thì phải bị chặn, chứ không phải vô tình mở công khai.
- Loại tệp tải lên xác định từ **magic bytes**, không bao giờ từ header `Content-Type`. SVG bị từ chối thẳng: nó là tài liệu XML thực thi được và là vector XSS kinh điển.
- Ảnh được mã hoá lại, và xoay theo EXIF Orientation **trước khi** bỏ siêu dữ liệu. Làm ngược thứ tự là mọi ảnh chụp dọc sẽ nằm ngang.
- Giới hạn tần suất theo IP đã băm, lưu ở Redis; và `CF-Connecting-IP` **chỉ** được tin khi kết nối đến từ dải IP của proxy đã cấu hình.
- Phản hồi lỗi không bao giờ chứa dấu vết ngăn xếp, tên lớp Java, phiên bản thư viện hay câu SQL.
- Sao lưu chạy hằng đêm sang **nhà cung cấp khác**, và việc phục hồi đã thực sự được diễn tập. Một bản sao lưu chưa từng phục hồi thử thì không phải là bản sao lưu.
