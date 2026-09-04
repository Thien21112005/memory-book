# LLD — Low-Level Design
## Phượng Hồng Memories — Kỷ yếu điện tử lớp 12A1

---

## 1. Mục đích tài liệu

Chi tiết hoá từng phần đã mô tả ở [`MemoryBook-HLD.md`](MemoryBook-HLD.md): lược đồ cơ sở dữ liệu, máy trạng thái, sơ đồ tuần tự, hợp đồng API và các đoạn mã then chốt. Đây là tài liệu **có giá trị nhất khi đi phỏng vấn**, vì nó cho thấy mức độ hiểu sâu chứ không dừng ở việc kể tên công nghệ.

> **Lưu ý cách dùng.** LLD nên được viết và cập nhật **song song với code**, không viết hết trước rồi mới code — chi tiết hiện thực thực tế luôn khác dự tính ban đầu. Phần nào chưa code thì đánh dấu `[CHƯA HIỆN THỰC]` thay vì để tài liệu nói dối.
>
> **Trạng thái hiện tại:** toàn bộ tài liệu này ở mức thiết kế. Thư mục `server/` còn trống, `client/` mới có bản dựng tạm bằng một tệp HTML.

---

## 2. Lược đồ cơ sở dữ liệu (PostgreSQL)

### 2.1 Quy ước chung

| Quy ước | Lý do |
|---|---|
| Khoá chính là `UUID` sinh phía cơ sở dữ liệu | Không lộ số lượng bản ghi qua định danh; ghép dữ liệu từ nhiều nguồn không đụng nhau |
| Mọi bảng có `created_at`, bảng thay đổi được có `updated_at` (`timestamptz`) | Luôn lưu kèm múi giờ; tránh cả lớp lỗi khi máy chủ đổi cấu hình |
| Xoá là **xoá mềm** bằng `deleted_at`, không `DELETE` | FR-ADM-09: khôi phục được trong 30 ngày |
| Trạng thái lưu dạng `VARCHAR` + ràng buộc `CHECK`, không dùng `ENUM` của PostgreSQL | Thêm giá trị mới không cần khoá bảng để `ALTER TYPE` |
| Tên bảng số nhiều, tên cột `snake_case` | Quy ước phổ biến, khớp mặc định của Hibernate |
| Mọi thay đổi lược đồ qua tệp di trú Flyway `V<n>__<mô_tả>.sql` | NFR-MAIN-02 — cấm sửa tay trên môi trường thật |

### 2.2 DDL

```sql
-- =====================================================================
-- V1__init.sql
-- =====================================================================

CREATE EXTENSION IF NOT EXISTS "pgcrypto";   -- gen_random_uuid()

-- ---------------------------------------------------------------------
-- Lời nhắn
-- ---------------------------------------------------------------------
CREATE TABLE messages (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  fullname      VARCHAR(60),
  nickname      VARCHAR(60),
  body          VARCHAR(500)  NOT NULL,
  status        VARCHAR(16)   NOT NULL DEFAULT 'PENDING',
  ip_hash       CHAR(64),                    -- HMAC-SHA256(IP), KHÔNG lưu IP thô
  user_agent    VARCHAR(255),                -- cắt ngắn, chỉ để điều tra spam
  content_hash  CHAR(64)      NOT NULL,      -- SHA-256(body chuẩn hoá) — chống trùng
  created_at    TIMESTAMPTZ   NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ   NOT NULL DEFAULT now(),
  deleted_at    TIMESTAMPTZ,
  CONSTRAINT messages_status_chk
    CHECK (status IN ('PENDING','PUBLISHED','REJECTED','HIDDEN')),
  CONSTRAINT messages_body_not_blank
    CHECK (length(btrim(body)) > 0)
);

-- Truy vấn nóng nhất: danh sách công khai, mới nhất trước, phân trang con trỏ.
-- Chỉ mục một phần (partial index) — chỉ chứa hàng PUBLISHED, nên rất nhỏ.
CREATE INDEX messages_public_idx
  ON messages (created_at DESC, id DESC)
  WHERE status = 'PUBLISHED' AND deleted_at IS NULL;

-- Hàng chờ kiểm duyệt: cũ nhất trước (ai gửi trước được duyệt trước)
CREATE INDEX messages_queue_idx
  ON messages (created_at ASC)
  WHERE status = 'PENDING' AND deleted_at IS NULL;

-- Chống gửi trùng nội dung từ cùng một nguồn (FR-MSG-06)
CREATE INDEX messages_dedup_idx ON messages (ip_hash, content_hash, created_at DESC);

-- ---------------------------------------------------------------------
-- Ảnh
-- ---------------------------------------------------------------------
CREATE TABLE photos (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  original_name  VARCHAR(255),               -- chỉ để hiển thị, KHÔNG dùng làm tên tệp
  storage_key    VARCHAR(255)  NOT NULL,     -- khoá đối tượng do hệ thống sinh
  mime_type      VARCHAR(50)   NOT NULL,     -- xác định từ magic bytes, không tin client
  byte_size      BIGINT        NOT NULL,
  width          INTEGER,                    -- NULL cho tới khi xử lý xong
  height         INTEGER,
  content_hash   CHAR(64)      NOT NULL,     -- SHA-256 nội dung tệp (FR-UPL-12)
  note           VARCHAR(300),
  taken_at_text  VARCHAR(50),                -- văn bản tự do như người gửi nhập
  category       VARCHAR(16),
  status         VARCHAR(16)   NOT NULL DEFAULT 'PENDING',
  processing     VARCHAR(16)   NOT NULL DEFAULT 'QUEUED',
  reject_reason  VARCHAR(255),
  ip_hash        CHAR(64),
  consent_given  BOOLEAN       NOT NULL DEFAULT false,   -- FR-PRV-07
  created_at     TIMESTAMPTZ   NOT NULL DEFAULT now(),
  updated_at     TIMESTAMPTZ   NOT NULL DEFAULT now(),
  published_at   TIMESTAMPTZ,
  deleted_at     TIMESTAMPTZ,
  CONSTRAINT photos_status_chk
    CHECK (status IN ('AWAITING_UPLOAD','PENDING','PUBLISHED','REJECTED','HIDDEN','QUARANTINED')),
  CONSTRAINT photos_processing_chk
    CHECK (processing IN ('QUEUED','SCANNING','ARCHIVING','PROCESSING','READY','FAILED')),
  CONSTRAINT photos_category_chk
    CHECK (category IS NULL OR category IN ('GRADE_10','GRADE_11','GRADE_12','EVENT'))
);

-- Ảnh trùng nội dung bị từ chối — nhưng chỉ tính trong số ảnh chưa xoá
CREATE UNIQUE INDEX photos_content_hash_uk
  ON photos (content_hash) WHERE deleted_at IS NULL;

CREATE INDEX photos_public_idx
  ON photos (published_at DESC, id DESC)
  WHERE status = 'PUBLISHED' AND deleted_at IS NULL;

CREATE INDEX photos_public_category_idx
  ON photos (category, published_at DESC)
  WHERE status = 'PUBLISHED' AND deleted_at IS NULL;

CREATE INDEX photos_queue_idx
  ON photos (created_at ASC)
  WHERE status = 'PENDING' AND deleted_at IS NULL;

-- ---------------------------------------------------------------------
-- Biến thể ảnh (thumb/medium/original × webp/jpeg)
-- Tách bảng thay vì nhồi 6 cột URL vào photos: thêm định dạng mới (AVIF)
-- sau này chỉ là thêm hàng, không phải đổi lược đồ.
-- ---------------------------------------------------------------------
CREATE TABLE photo_variants (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  photo_id     UUID        NOT NULL REFERENCES photos(id) ON DELETE CASCADE,
  variant      VARCHAR(16) NOT NULL,       -- THUMB (400px) | MEDIUM (1600px) | LARGE (4000px)
  format       VARCHAR(8)  NOT NULL,       -- AVIF | JPEG
  storage_key  VARCHAR(255) NOT NULL,
  width        INTEGER     NOT NULL,
  height       INTEGER     NOT NULL,
  byte_size    BIGINT      NOT NULL,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  CONSTRAINT photo_variants_variant_chk CHECK (variant IN ('THUMB','MEDIUM','LARGE')),
  CONSTRAINT photo_variants_format_chk  CHECK (format  IN ('AVIF','JPEG')),
  CONSTRAINT photo_variants_uk UNIQUE (photo_id, variant, format)
);

CREATE INDEX photo_variants_photo_idx ON photo_variants (photo_id);

-- ---------------------------------------------------------------------
-- Tài khoản quản trị (rất ít hàng — 1 đến 3)
-- ---------------------------------------------------------------------
CREATE TABLE admin_users (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  username         VARCHAR(50)  NOT NULL UNIQUE,
  password_hash    VARCHAR(255) NOT NULL,      -- Argon2id
  display_name     VARCHAR(100),
  totp_secret_enc  VARCHAR(255) NOT NULL,      -- mã hoá bằng khoá ứng dụng, không lưu bản rõ
  totp_enabled     BOOLEAN      NOT NULL DEFAULT false,
  failed_attempts  SMALLINT     NOT NULL DEFAULT 0,
  locked_until     TIMESTAMPTZ,
  last_login_at    TIMESTAMPTZ,
  password_changed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  is_active        BOOLEAN      NOT NULL DEFAULT true,
  created_at       TIMESTAMPTZ  NOT NULL DEFAULT now(),
  updated_at       TIMESTAMPTZ  NOT NULL DEFAULT now()
);

-- Mã dự phòng 2FA — băm, dùng một lần
CREATE TABLE admin_backup_codes (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  admin_id   UUID         NOT NULL REFERENCES admin_users(id) ON DELETE CASCADE,
  code_hash  VARCHAR(255) NOT NULL,
  used_at    TIMESTAMPTZ,
  created_at TIMESTAMPTZ  NOT NULL DEFAULT now()
);

CREATE INDEX admin_backup_codes_admin_idx ON admin_backup_codes (admin_id) WHERE used_at IS NULL;

-- ---------------------------------------------------------------------
-- Nhật ký kiểm toán — CHỈ THÊM, không sửa, không xoá (FR-ADM-12)
-- ---------------------------------------------------------------------
CREATE TABLE audit_logs (
  id           BIGGENERATED_PLACEHOLDER,
  actor_id     UUID         REFERENCES admin_users(id),
  actor_name   VARCHAR(100) NOT NULL,       -- sao chép tại thời điểm ghi, để log còn đọc được
                                            -- kể cả khi tài khoản bị xoá sau này
  action       VARCHAR(50)  NOT NULL,       -- PHOTO_PUBLISH, MESSAGE_REJECT, LOGIN_FAILED...
  entity_type  VARCHAR(30),                 -- PHOTO | MESSAGE | ADMIN_USER | TAKEDOWN
  entity_id    UUID,
  before_state JSONB,
  after_state  JSONB,
  ip_hash      CHAR(64),
  created_at   TIMESTAMPTZ  NOT NULL DEFAULT now()
);

CREATE INDEX audit_logs_entity_idx  ON audit_logs (entity_type, entity_id, created_at DESC);
CREATE INDEX audit_logs_created_idx ON audit_logs (created_at DESC);
CREATE INDEX audit_logs_action_idx  ON audit_logs (action, created_at DESC);

-- Quyền ở tầng cơ sở dữ liệu: tài khoản ứng dụng chỉ được INSERT và SELECT.
-- Không phải "tin vào code không xoá" mà là "cơ sở dữ liệu không cho xoá".
REVOKE UPDATE, DELETE ON audit_logs FROM app_user;

-- ---------------------------------------------------------------------
-- Yêu cầu gỡ nội dung
-- ---------------------------------------------------------------------
CREATE TABLE takedown_requests (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  reference     VARCHAR(20)  NOT NULL UNIQUE,   -- mã tra cứu cho người gửi, vd "TD-7K2M9X"
  target_type   VARCHAR(16)  NOT NULL,          -- PHOTO | MESSAGE
  target_id     UUID         NOT NULL,
  reason        VARCHAR(500) NOT NULL,
  contact_email VARCHAR(255),                   -- tuỳ chọn
  status        VARCHAR(16)  NOT NULL DEFAULT 'OPEN',
  resolution    VARCHAR(500),
  handled_by    UUID REFERENCES admin_users(id),
  handled_at    TIMESTAMPTZ,
  ip_hash       CHAR(64),
  created_at    TIMESTAMPTZ  NOT NULL DEFAULT now(),
  CONSTRAINT takedown_status_chk CHECK (status IN ('OPEN','RESOLVED','REJECTED')),
  CONSTRAINT takedown_target_chk CHECK (target_type IN ('PHOTO','MESSAGE'))
);

CREATE INDEX takedown_open_idx ON takedown_requests (created_at ASC) WHERE status = 'OPEN';

-- ---------------------------------------------------------------------
-- Hàng đợi tác vụ nền (mẫu outbox)
-- ---------------------------------------------------------------------
CREATE TABLE outbox_jobs (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  job_type      VARCHAR(40)  NOT NULL,     -- PROCESS_PHOTO | BUILD_ALBUM | PURGE_DELETED
  payload       JSONB        NOT NULL,
  status        VARCHAR(16)  NOT NULL DEFAULT 'PENDING',
  attempts      SMALLINT     NOT NULL DEFAULT 0,
  max_attempts  SMALLINT     NOT NULL DEFAULT 3,
  last_error    TEXT,
  run_after     TIMESTAMPTZ  NOT NULL DEFAULT now(),   -- cho thử lại giãn cách tăng dần
  locked_at     TIMESTAMPTZ,
  locked_by     VARCHAR(64),                           -- định danh tiến trình xử lý
  created_at    TIMESTAMPTZ  NOT NULL DEFAULT now(),
  completed_at  TIMESTAMPTZ,
  CONSTRAINT outbox_status_chk CHECK (status IN ('PENDING','RUNNING','DONE','FAILED'))
);

CREATE INDEX outbox_pickup_idx
  ON outbox_jobs (run_after ASC)
  WHERE status = 'PENDING';

-- ---------------------------------------------------------------------
-- Dữ liệu cấu hình nội dung (FR-CFG-01, FR-CFG-02)
-- Thay cho các mảng DANH_SACH_LOP / DANH_SACH_THAY_CO đang viết cứng
-- trong <script> của client/index.html
-- ---------------------------------------------------------------------
CREATE TABLE class_members (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  kind        VARCHAR(16) NOT NULL,       -- STUDENT | TEACHER
  full_name   VARCHAR(100) NOT NULL,
  nickname    VARCHAR(60),
  subject     VARCHAR(100),               -- chỉ dùng cho TEACHER
  emoji       VARCHAR(16),
  quote       VARCHAR(300),
  sort_order  SMALLINT NOT NULL DEFAULT 0,
  is_visible  BOOLEAN  NOT NULL DEFAULT true,
  CONSTRAINT class_members_kind_chk CHECK (kind IN ('STUDENT','TEACHER'))
);

CREATE TABLE timeline_events (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  grade       SMALLINT     NOT NULL,      -- 10 | 11 | 12
  title       VARCHAR(150) NOT NULL,
  description VARCHAR(500),
  event_date  DATE,
  sort_order  SMALLINT     NOT NULL DEFAULT 0,
  CONSTRAINT timeline_grade_chk CHECK (grade BETWEEN 10 AND 12)
);
```

> **Ghi chú về `audit_logs.id`**: dùng `BIGINT GENERATED ALWAYS AS IDENTITY`. Ký hiệu `BIGGENERATED_PLACEHOLDER` ở trên là chỗ giữ chỗ để tránh nhầm khi copy — khi viết tệp di trú thật, thay bằng:
> ```sql
> id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
> ```
> Ở đây dùng số tăng dần thay vì UUID có chủ đích: nhật ký kiểm toán cần **thứ tự ghi** rõ ràng, và không có nhu cầu giấu số lượng bản ghi.

### 2.3 Vì sao dùng chỉ mục một phần (partial index)

Truy vấn nóng nhất của trang là:

```sql
SELECT * FROM photos
WHERE status = 'PUBLISHED' AND deleted_at IS NULL
ORDER BY published_at DESC, id DESC
LIMIT 20;
```

Nếu tạo chỉ mục thường trên `(published_at DESC)`, chỉ mục chứa **mọi** hàng, kể cả hàng `PENDING`, `REJECTED`, đã xoá. Với chỉ mục một phần kèm `WHERE`, chỉ mục chỉ chứa các hàng thực sự được truy vấn — nhỏ hơn, nằm gọn trong bộ nhớ, và PostgreSQL không cần lọc lại sau khi quét chỉ mục. Ở quy mô 2.000 ảnh thì khác biệt chưa lớn, nhưng đây là thói quen đúng và là điểm giải thích được trong phỏng vấn.

---

## 3. Máy trạng thái nội dung (State Machine)

### 3.1 Trạng thái kiểm duyệt (dùng chung cho ảnh và lời nhắn)

```
                     ┌──────────────────────────────────────────┐
                     │                                          │
   [gửi lên]         ▼                                          │
      │        ┌───────────┐   duyệt      ┌───────────┐         │
      └───────▶│  PENDING  │─────────────▶│ PUBLISHED │         │
               └─────┬─────┘              └─────┬─────┘         │
                     │                          │               │
                     │ từ chối                  │ gỡ / khiếu nại│
                     ▼                          ▼               │
               ┌───────────┐              ┌───────────┐         │
               │ REJECTED  │              │  HIDDEN   │─────────┘
               └─────┬─────┘              └─────┬─────┘  duyệt lại
                     │                          │
                     │  xoá mềm                 │  xoá mềm
                     ▼                          ▼
               ┌──────────────────────────────────────┐
               │  deleted_at IS NOT NULL              │
               │  (giữ 30 ngày, khôi phục được)       │
               └──────────────┬───────────────────────┘
                              │ tác vụ PURGE_DELETED sau 30 ngày
                              ▼
                    [xoá vĩnh viễn — cả bản ghi lẫn tệp]
```

Riêng ảnh có thêm một trạng thái ngoài luồng trên:

```
   PENDING ──[ClamAV phát hiện mã độc]──▶ QUARANTINED  (ngõ cụt)
                                          │
                                          └─▶ cảnh báo quản trị, tệp không bao giờ
                                              rời khỏi vùng quarantine/
```

### 3.2 Trạng thái xử lý ảnh (`photos.processing`) — độc lập với trạng thái kiểm duyệt

```
QUEUED ──▶ SCANNING ──▶ PROCESSING ──▶ READY
   │           │             │
   │           │             └──[lỗi, còn lượt thử]──▶ QUEUED (giãn cách tăng dần)
   │           │
   │           └──[phát hiện mã độc]──▶ (photos.status = QUARANTINED)
   │
   └──[hết lượt thử]──▶ FAILED  → quản trị viên thấy trong hàng chờ, xử lý tay
```

Tách hai trục trạng thái là quyết định có chủ đích: một ảnh có thể **đã xử lý xong nhưng chưa được duyệt** (`processing = READY`, `status = PENDING`), hoặc **đã được duyệt nhưng xử lý lỗi** (không thể xảy ra, vì giao diện quản trị chỉ cho duyệt ảnh `READY`). Nhồi cả hai vào một cột sẽ sinh ra những trạng thái lai vô nghĩa.

**Quy tắc bất biến:** giao diện quản trị **chỉ hiển thị nút Duyệt khi `processing = READY`.** Đây là ràng buộc cũng phải kiểm ở tầng service, không chỉ ẩn nút.

---

## 4. Sơ đồ tuần tự

### 4.1 Tải ảnh lên — từ lúc bấm gửi tới lúc sẵn sàng duyệt

```
Người dùng   React      Spring API    PostgreSQL   Kho lưu trữ   ClamAV   Tác vụ nền
    │          │             │             │            │           │          │
    │─chọn ảnh▶│             │             │            │           │          │
    │          │─kiểm tra sơ bộ (kích thước, đuôi) — chỉ để báo lỗi sớm       │
    │          │             │             │            │           │          │
    │          │─POST /api/v1/photos (multipart) ──────▶│           │          │
    │          │             │                          │           │          │
    │          │      [1] giới hạn tần suất (Redis)     │           │          │
    │          │      [2] xác minh Turnstile            │           │          │
    │          │      [3] chặn dung lượng > 15MB        │           │          │
    │          │      [4] đọc magic bytes → loại thật   │           │          │
    │          │      [5] đọc kích thước ảnh, chặn bom nén                     │
    │          │      [6] SHA-256 → kiểm tra trùng      │           │          │
    │          │             │                          │           │          │
    │          │             │──ghi tệp thô─────────────▶ quarantine/          │
    │          │             │             │            │           │          │
    │          │             │─BEGIN───────▶            │           │          │
    │          │             │  INSERT photos (PENDING, QUEUED)     │          │
    │          │             │  INSERT outbox_jobs (PROCESS_PHOTO)  │          │
    │          │             │─COMMIT──────▶            │           │          │
    │          │             │                          │           │          │
    │◀─202 Accepted { id, status: PENDING } ────────────│           │          │
    │  "Ảnh đã gửi, đang chờ duyệt"                     │           │          │
    │          │             │             │            │           │          │
    │          │             │             │      [tác vụ nền lấy việc]────────▶│
    │          │             │             │◀─SELECT ... FOR UPDATE SKIP LOCKED─│
    │          │             │             │            │◀─đọc tệp─────────────│
    │          │             │             │            │           │◀─quét────│
    │          │             │             │            │           │──sạch───▶│
    │          │             │             │            │                      │
    │          │             │             │      [giải mã → mã hoá lại]        │
    │          │             │             │      → EXIF/GPS biến mất ở đây     │
    │          │             │             │      [sinh 3 kích cỡ × 2 định dạng]│
    │          │             │             │            │◀─ghi biến thể────────│
    │          │             │             │◀─UPDATE photos SET processing=READY│
    │          │             │             │  INSERT photo_variants ×6          │
    │          │             │             │  UPDATE outbox_jobs SET status=DONE│
    │          │             │             │            │           │          │
    │          │             │        [ảnh sẵn sàng, hiện trong hàng chờ duyệt] │
```

**Vì sao `INSERT photos` và `INSERT outbox_jobs` phải nằm trong cùng một giao dịch:** nếu tách ra, có thể xảy ra trường hợp bản ghi ảnh đã ghi nhưng tác vụ xử lý chưa kịp tạo (tiến trình chết ở giữa) — ảnh sẽ nằm im mãi ở `QUEUED` mà không ai xử lý. Đặt chung một giao dịch thì hoặc cả hai cùng có, hoặc không có gì (NFR-REL-05).

**Vì sao dùng `FOR UPDATE SKIP LOCKED`:** khi có nhiều tiến trình xử lý chạy song song, mỗi tiến trình lấy một việc khác nhau mà không chờ nhau, cũng không có việc nào bị xử lý hai lần.

### 4.2 Đăng nhập quản trị — chống dò mật khẩu và dò tên tài khoản

```
Quản trị   React      Spring Security      PostgreSQL     Redis
   │         │               │                  │           │
   │─user+pass▶              │                  │           │
   │         │─POST /admin/auth/login──────────▶│           │
   │         │               │                  │           │
   │         │        [1] giới hạn tần suất theo IP ────────▶│
   │         │            vượt → 429            │           │
   │         │               │                  │           │
   │         │        [2] SELECT admin_users WHERE username=?│
   │         │               │◀─────────────────│           │
   │         │               │                  │           │
   │         │        [3] nếu locked_until > now() → 423    │
   │         │               │                  │           │
   │         │        [4] so khớp Argon2id                  │
   │         │            ── LUÔN chạy phép băm, kể cả khi  │
   │         │               tài khoản không tồn tại, để    │
   │         │               thời gian phản hồi không tiết  │
   │         │               lộ tên tài khoản có thật hay không
   │         │               │                  │           │
   │         │        sai → failed_attempts++, 401 "Thông tin đăng nhập không đúng"
   │         │               │                  │           │
   │         │        đúng → tạo phiên tạm "chờ 2FA" (TTL 5 phút)──▶│
   │◀─200 { mfaRequired: true } ─────────────────│           │
   │         │               │                  │           │
   │─mã TOTP─▶│─POST /admin/auth/mfa { code }──▶│           │
   │         │        [5] xác thực TOTP (±1 khoảng 30 giây) │
   │         │            hoặc mã dự phòng (dùng một lần)   │
   │         │               │                  │           │
   │         │        [6] HUỶ phiên tạm, cấp phiên MỚI ─────▶│
   │         │            (chống cố định phiên — FR-ADM-05)  │
   │         │        [7] reset failed_attempts, ghi last_login_at
   │         │        [8] INSERT audit_logs (LOGIN_SUCCESS)  │
   │         │               │                  │           │
   │◀─200 + Set-Cookie: SESSION=... HttpOnly; Secure; SameSite=Lax
   │◀─       + XSRF-TOKEN cookie (đọc được bởi JS, gửi lại qua header)
```

Điểm [4] là chi tiết hay bị bỏ qua: nếu chỉ chạy phép băm khi tài khoản tồn tại, thì phản hồi cho tên tài khoản có thật sẽ chậm hơn rõ rệt (Argon2id cố tình tốn thời gian). Kẻ tấn công đo thời gian là biết tên tài khoản nào có thật.

---

## 5. Hợp đồng API (API Contract v1)

### 5.1 Quy ước chung

| Hạng mục | Quy ước |
|---|---|
| Đường dẫn gốc | `/api/v1` — có phiên bản ngay từ đầu (NFR-MAIN-06) |
| Định dạng | JSON, `UTF-8`; riêng tải ảnh dùng `multipart/form-data` |
| Thời gian | ISO 8601 kèm múi giờ, ví dụ `2026-05-18T09:12:00Z` |
| Phân trang | Theo con trỏ (cursor), không dùng `offset` |
| Lỗi | Theo RFC 9457 Problem Details (`application/problem+json`) |
| Xác thực | Cookie phiên cho khu quản trị; điểm cuối công khai không cần xác thực |
| Ghi dữ liệu | Yêu cầu ghi có phiên phải kèm header `X-XSRF-TOKEN` |

### 5.2 Vì sao phân trang theo con trỏ chứ không phải `offset`

Với `LIMIT 20 OFFSET 200`, PostgreSQL vẫn phải đọc và bỏ đi 200 hàng đầu — càng lật sâu càng chậm. Nghiêm trọng hơn: nếu có nội dung mới được duyệt trong lúc người dùng đang cuộn, mọi hàng bị đẩy xuống một bậc, và người dùng sẽ **thấy lặp lại một mục đã xem** hoặc **bị nhảy cóc mất một mục**. Con trỏ dựa trên `(created_at, id)` không có vấn đề đó:

```sql
SELECT * FROM messages
WHERE status = 'PUBLISHED' AND deleted_at IS NULL
  AND (created_at, id) < (:cursorCreatedAt, :cursorId)   -- so sánh bộ đôi
ORDER BY created_at DESC, id DESC
LIMIT :limit;
```

Ghép thêm `id` vào khoá sắp xếp là bắt buộc: hai lời nhắn có thể trùng `created_at` tới từng micro giây, và khi đó thứ tự không ổn định sẽ làm hỏng phân trang.

### 5.3 Điểm cuối công khai

#### Lời nhắn

```http
GET /api/v1/messages?limit=20&cursor=<opaque>
```

```json
{
  "items": [
    {
      "id": "0193f2a1-...",
      "displayName": "Thanh Thủy",
      "body": "Dù sau này có đi đâu…",
      "createdAt": "2026-05-18T09:12:00Z"
    }
  ],
  "nextCursor": "eyJjIjoiMjAyNi0wNS0xOFQwOToxMjowMFoiLCJpIjoiMDE5M2YyYTEifQ"
}
```

Lưu ý: API trả `displayName` đã tính sẵn (`nickname` → `fullname` → `"Ẩn danh"`), **không** trả `fullname` và `nickname` thô. Front-end không cần biết tên thật của người gửi, nên không gửi cho nó — nguyên tắc tối thiểu hoá dữ liệu (NFR-PRV-01).

```http
POST /api/v1/messages
Content-Type: application/json

{
  "fullname": "Nguyễn Bảo An",
  "nickname": "An Mèo",
  "message": "Nhớ cả lớp nhiều!",
  "turnstileToken": "0.abc..."
}
```

| Mã | Khi nào |
|---|---|
| `202 Accepted` | Nhận thành công, đang chờ duyệt — trả `{ "id": "...", "status": "PENDING" }` |
| `400 Bad Request` | Dữ liệu không hợp lệ (rỗng, quá dài, quá nhiều URL, Turnstile sai) |
| `409 Conflict` | Trùng nội dung đã gửi trong 24 giờ |
| `429 Too Many Requests` | Vượt giới hạn tần suất, kèm `Retry-After` |

> **Tên trường giữ nguyên `fullname` / `nickname` / `message`** để khớp với các ô nhập đang có trong `client/index.html`, giảm công sửa khi nối API vào bản dựng tạm.

#### Ảnh

```http
GET /api/v1/photos?category=GRADE_12&limit=24&cursor=<opaque>
```

```json
{
  "items": [
    {
      "id": "0193f2b7-...",
      "note": "Buổi liên hoan cuối năm",
      "takenAt": "Tháng 5, 2026",
      "category": "GRADE_12",
      "width": 3024,
      "height": 4032,
      "aspectRatio": 0.75,
      "variants": {
        "thumb":    { "webp": "https://img.example/…/t.webp", "jpeg": "https://img.example/…/t.jpg", "width": 360, "height": 480 },
        "medium":   { "webp": "https://img.example/…/m.webp", "jpeg": "https://img.example/…/m.jpg", "width": 960, "height": 1280 },
        "original": { "webp": "https://img.example/…/o.webp", "jpeg": "https://img.example/…/o.jpg", "width": 3024, "height": 4032 }
      },
      "publishedAt": "2026-05-19T02:00:00Z"
    }
  ],
  "nextCursor": null
}
```

`width`, `height` và `aspectRatio` trả về là để front-end đặt sẵn khung đúng tỉ lệ trước khi ảnh tải xong — đây chính là cách đạt CLS < 0,1 (NFR-PERF-05). Bỏ ba trường này đi là bố cục sẽ nhảy mỗi lần một ảnh tải xong.

Tải ảnh đi qua **hai bước**, vì ở mức 25 MB thì tệp không nên đi qua máy chủ ứng dụng (xem [`MemoryBook-Storage-Media.md`](MemoryBook-Storage-Media.md) mục 4).

**Bước 1 — xin đường dẫn ký sẵn:**

```http
POST /api/v1/photos/upload-intent
Content-Type: application/json

{
  "fileName": "IMG_4821.HEIC",
  "size": 21504000,
  "mimeHint": "image/heic",
  "turnstileToken": "0.abc..."
}
```

```json
{
  "photoId": "0193f2b7-...",
  "uploadUrl": "https://….r2.cloudflarestorage.com/quarantine/0193f2b7…?X-Amz-Signature=…",
  "expiresAt": "2026-09-03T10:15:00Z"
}
```

| Mã | Khi nào |
|---|---|
| `200 OK` | Cấp đường dẫn thành công |
| `400` | Turnstile sai, `size` thiếu hoặc không hợp lệ |
| `413` | `size` > 25 MB — từ chối ngay, chưa tốn byte nào |
| `429` | Vượt giới hạn tần suất **hoặc hạn ngạch dung lượng 500 MB/IP/ngày** |

**Bước 2 — trình duyệt `PUT` bytes thẳng vào `uploadUrl`.** Không đi qua máy chủ ứng dụng.

**Bước 3 — xác nhận và gửi kèm thông tin:**

```http
POST /api/v1/photos/{photoId}/complete
Content-Type: application/json

{
  "note": "Buổi liên hoan cuối năm",
  "takenAt": "Tháng 5, 2026",
  "category": "GRADE_12",
  "consent": true
}
```

Máy chủ lúc này mới kiểm tra nội dung: `HEAD` để xác nhận tệp có thật và đúng kích thước đã khai, `GET Range: bytes=0-31` để đọc magic bytes (chỉ tải 32 byte, không tải 25 MB), kiểm tra số điểm ảnh, tính mã băm và chống trùng.

| Mã | Khi nào |
|---|---|
| `202 Accepted` | Hợp lệ → `{ "id": "...", "status": "PENDING" }` |
| `400` | Thiếu `consent`, `category` sai giá trị |
| `404` | Chưa có tệp ở đường dẫn đã cấp (chưa `PUT`, hoặc đã quá hạn 10 phút) |
| `409` | Ảnh đã có trong thư viện (trùng mã băm nội dung) |
| `413` | Kích thước thật khác kích thước đã khai, hoặc ảnh vượt 80 megapixel |
| `415` | Không phải ảnh, hoặc là SVG |

> Tệp tải lên dở dang (xin đường dẫn nhưng không bao giờ gọi `complete`) được kho đối tượng tự xoá sau 24 giờ bằng quy tắc vòng đời — FR-ABU-02b.

> **Thay đổi bắt buộc ở front-end**: biểu mẫu trong `client/index.html` hiện **chưa gửi** phân loại đang chọn (bốn nút trong `#contribute-categories` chỉ đổi giao diện). Cần thêm một trường ẩn được cập nhật khi bấm nút, và thêm ô đánh dấu đồng thuận.

#### Album

```http
GET /api/v1/album
```

| Mã | Khi nào |
|---|---|
| `302 Found` | Chuyển hướng tới đường dẫn có chữ ký ở kho lưu trữ, hạn dùng 15 phút |
| `202 Accepted` | Album đang được dựng lại, thử lại sau |
| `429` | Vượt giới hạn tần suất riêng của điểm cuối này |

#### Cấu hình nội dung

```http
GET /api/v1/config/members     → danh sách học sinh và thầy cô
GET /api/v1/config/timeline    → các mốc "Hành trình"
GET /api/v1/config/stats       → { "photoCount": 128, "messageCount": 57, "memberCount": 40 }
```

Ba điểm cuối này có bộ nhớ đệm 5 phút ở Redis và header `Cache-Control: public, max-age=300`.

#### Yêu cầu gỡ nội dung

```http
POST /api/v1/takedown-requests

{
  "targetType": "PHOTO",
  "targetId": "0193f2b7-...",
  "reason": "Ảnh có mặt tôi, tôi không muốn đăng công khai",
  "contactEmail": "…@…",
  "turnstileToken": "0.abc..."
}
```

→ `202 Accepted` với `{ "reference": "TD-7K2M9X" }`

```http
GET /api/v1/takedown-requests/{reference}
```

→ `{ "reference": "TD-7K2M9X", "status": "OPEN", "createdAt": "..." }`

Điểm cuối tra cứu **chỉ trả trạng thái**, không trả lý do hay thông tin liên hệ — nếu không, ai đoán được mã tra cứu sẽ đọc được nội dung khiếu nại của người khác.

### 5.4 Điểm cuối quản trị (yêu cầu phiên + vai trò `ADMIN`)

```http
POST   /api/v1/admin/auth/login        { username, password }
POST   /api/v1/admin/auth/mfa          { code }
POST   /api/v1/admin/auth/logout
GET    /api/v1/admin/me

GET    /api/v1/admin/queue?type=all|photo|message&status=pending&limit=20&cursor=…
PATCH  /api/v1/admin/photos/{id}       { status?, note?, takenAt?, category?, rejectReason? }
PATCH  /api/v1/admin/messages/{id}     { status?, body?, rejectReason? }
DELETE /api/v1/admin/photos/{id}       → xoá mềm
DELETE /api/v1/admin/messages/{id}     → xoá mềm
POST   /api/v1/admin/photos/{id}/restore
POST   /api/v1/admin/messages/{id}/restore

POST   /api/v1/admin/photos/bulk       { ids: [...], action: "PUBLISH"|"REJECT"|"HIDE" }

GET    /api/v1/admin/takedown-requests?status=open
PATCH  /api/v1/admin/takedown-requests/{id}  { status, resolution }

GET    /api/v1/admin/audit-logs?entityType=&entityId=&action=&limit=&cursor=
POST   /api/v1/admin/album/rebuild     → xếp tác vụ dựng lại album
```

Ràng buộc quan trọng: `PATCH /admin/photos/{id}` với `status: "PUBLISHED"` **phải trả `409 Conflict`** nếu ảnh đó chưa có `processing = READY`. Kiểm tra ở tầng service, không phụ thuộc vào việc giao diện có ẩn nút hay không (NFR-SEC-05).

### 5.5 Định dạng lỗi (RFC 9457)

```json
{
  "type": "https://memorybook.example/errors/file-too-large",
  "title": "Ảnh vượt quá dung lượng cho phép",
  "status": 413,
  "detail": "Ảnh bạn gửi nặng 32,4 MB. Giới hạn hiện tại là 25 MB.",
  "instance": "/api/v1/photos",
  "traceId": "b3f1c2d4e5a6"
}
```

Quy tắc: `detail` viết bằng tiếng Việt, hướng tới người dùng cuối, và **không bao giờ chứa dấu vết ngăn xếp, tên lớp Java, phiên bản thư viện hay câu truy vấn SQL** (NFR-SEC-13). `traceId` là cầu nối duy nhất giữa thông báo người dùng nhìn thấy và log chi tiết ở phía máy chủ.

```java
@RestControllerAdvice
class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(FileTooLargeException.class)
    ProblemDetail handleTooLarge(FileTooLargeException ex) {
        var pd = ProblemDetail.forStatus(HttpStatus.PAYLOAD_TOO_LARGE);
        pd.setType(URI.create("https://memorybook.example/errors/file-too-large"));
        pd.setTitle("Ảnh vượt quá dung lượng cho phép");
        pd.setDetail("Ảnh bạn gửi nặng %s. Giới hạn hiện tại là %s."
                .formatted(humanSize(ex.actualBytes()), humanSize(ex.limitBytes())));
        pd.setProperty("traceId", MDC.get("traceId"));
        return pd;
    }

    // Lưới an toàn cuối cùng: mọi lỗi không lường trước
    @ExceptionHandler(Exception.class)
    ProblemDetail handleUnexpected(Exception ex) {
        String traceId = MDC.get("traceId");
        log.error("Lỗi không lường trước [traceId={}]", traceId, ex);   // chi tiết chỉ nằm ở log

        var pd = ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
        pd.setTitle("Có lỗi xảy ra");
        pd.setDetail("Hệ thống gặp sự cố. Vui lòng thử lại sau ít phút.");
        pd.setProperty("traceId", traceId);      // người dùng đọc mã này cho quản trị viên
        return pd;
    }
}
```

---

## 6. Module Bảo mật — chi tiết hiện thực

> Phần này chỉ nêu mã then chốt. Phân tích đầy đủ theo OWASP nằm ở [`MemoryBook-Security.md`](MemoryBook-Security.md).

### 6.1 Cấu hình Spring Security

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity                       // bật @PreAuthorize ở tầng service
public class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http, RateLimitFilter rateLimitFilter)
            throws Exception {

        http
            // ---- CSRF: bắt buộc vì xác thực bằng cookie ----
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                .csrfTokenRequestHandler(new SpaCsrfTokenRequestHandler())
                // Điểm cuối công khai không dùng cookie phiên → không cần CSRF,
                // nhưng vẫn được bảo vệ bằng Turnstile + giới hạn tần suất.
                .ignoringRequestMatchers("/api/v1/messages", "/api/v1/photos",
                                         "/api/v1/takedown-requests"))

            // ---- Phân quyền: MẶC ĐỊNH TỪ CHỐI ----
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(HttpMethod.GET,
                        "/api/v1/messages", "/api/v1/photos",
                        "/api/v1/album", "/api/v1/config/**",
                        "/api/v1/takedown-requests/*").permitAll()
                .requestMatchers(HttpMethod.POST,
                        "/api/v1/messages", "/api/v1/photos",
                        "/api/v1/takedown-requests",
                        "/api/v1/admin/auth/login", "/api/v1/admin/auth/mfa").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().denyAll())        // ← đường dẫn mới quên khai báo sẽ BỊ CHẶN,
                                                //   thay vì vô tình mở công khai

            // ---- Phiên ----
            .sessionManagement(s -> s
                .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
                .sessionFixation().newSession()          // FR-ADM-05
                .maximumSessions(3))

            // ---- Header bảo mật ----
            .headers(h -> h
                .contentSecurityPolicy(csp -> csp.policyDirectives(CSP))
                .frameOptions(HeadersConfigurer.FrameOptionsConfig::deny)
                .httpStrictTransportSecurity(hsts -> hsts
                    .includeSubDomains(true).maxAgeInSeconds(31_536_000))
                .referrerPolicy(r -> r.policy(
                    ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN))
                .permissionsPolicyHeader(p -> p.policy(
                    "camera=(), microphone=(), geolocation=(), interest-cohort=()")))

            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .addFilterBefore(rateLimitFilter, UsernamePasswordAuthenticationFilter.class)

            // ---- Không lộ chi tiết khi chưa đăng nhập ----
            .exceptionHandling(e -> e
                .authenticationEntryPoint((req, res, ex) ->
                    res.sendError(HttpStatus.UNAUTHORIZED.value())));

        return http.build();
    }

    private static final String CSP = String.join("; ",
        "default-src 'self'",
        "script-src 'self'",                       // KHÔNG 'unsafe-inline', KHÔNG CDN ngoài
        "style-src 'self'",
        "img-src 'self' https://img.example data:",
        "font-src 'self'",
        "connect-src 'self' https://challenges.cloudflare.com",
        "frame-src https://challenges.cloudflare.com",   // Turnstile
        "object-src 'none'",
        "base-uri 'self'",
        "form-action 'self'",
        "frame-ancestors 'none'",
        "upgrade-insecure-requests");

    @Bean
    PasswordEncoder passwordEncoder() {
        // Argon2id với tham số mặc định của Spring Security 5.8+
        return Argon2PasswordEncoder.defaultsForSpringSecurity_v5_8();
    }

    @Bean
    CorsConfigurationSource corsConfigurationSource() {
        var config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://app.example"));  // KHÔNG dùng "*"
        config.setAllowedMethods(List.of("GET", "POST", "PATCH", "DELETE"));
        config.setAllowedHeaders(List.of("Content-Type", "X-XSRF-TOKEN"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        var source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return source;
    }
}
```

Hai dòng đáng chú ý nhất:

- `.anyRequest().denyAll()` — mọi đường dẫn chưa khai báo đều bị chặn. Nếu sau này thêm một controller mới mà quên khai báo, hậu quả là "không dùng được" (phát hiện ngay khi thử) chứ không phải "vô tình mở công khai cho cả internet" (phát hiện khi đã muộn).
- `script-src 'self'` — chính là ràng buộc khiến bản chạy thật **không thể** nạp thư viện từ CDN như bản dựng tạm đang làm (quyết định DD10 ở HLD).

### 6.2 Bộ lọc giới hạn tần suất

```java
@Component
public class RateLimitFilter extends OncePerRequestFilter {

    private final ProxyManager<String> buckets;   // Bucket4j trên Redis
    private final IpHasher ipHasher;

    private static final Map<String, RateLimitRule> RULES = Map.of(
        "POST:/api/v1/messages",           new RateLimitRule(5,  Duration.ofHours(1)),
        "POST:/api/v1/photos",             new RateLimitRule(10, Duration.ofHours(1)),
        "POST:/api/v1/admin/auth/login",   new RateLimitRule(10, Duration.ofMinutes(15)),
        "POST:/api/v1/takedown-requests",  new RateLimitRule(3,  Duration.ofHours(1)),
        "GET:/api/v1/album",               new RateLimitRule(3,  Duration.ofHours(1))
    );

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {

        RateLimitRule rule = RULES.getOrDefault(key(req), DEFAULT_READ_RULE);
        String bucketKey = rule.name() + ":" + ipHasher.hash(clientIp(req));

        var probe = buckets.builder().build(bucketKey, rule.config())
                           .tryConsumeAndReturnRemaining(1);

        if (!probe.isConsumed()) {
            res.setStatus(429);
            res.setHeader("Retry-After",
                String.valueOf(probe.getNanosToWaitForRefill() / 1_000_000_000L));
            res.setContentType("application/problem+json");
            res.getWriter().write(TOO_MANY_REQUESTS_BODY);
            return;
        }
        chain.doFilter(req, res);
    }
}
```

```java
@Component
public class IpHasher {
    private final Mac mac;   // HMAC-SHA256 với khoá bí mật đọc từ biến môi trường

    /**
     * KHÔNG BAO GIỜ lưu IP thô (FR-ABU-07, NFR-PRV-01).
     * Dùng HMAC chứ không phải SHA-256 trần: không gian IPv4 chỉ có 2^32 giá trị,
     * một bảng tra cứu SHA-256 cho toàn bộ IPv4 dựng được trong vài giờ.
     * Có khoá bí mật thì bảng tra cứu đó vô dụng.
     */
    public String hash(String ip) {
        return HexFormat.of().formatHex(mac.doFinal(ip.getBytes(UTF_8)));
    }
}
```

```java
/**
 * Lấy IP thật của client.
 * Chỉ tin X-Forwarded-For khi kết nối đến TỪ proxy đã cấu hình (FR-ABU-09).
 * Nếu tin vô điều kiện, bất kỳ ai cũng tự đặt header này để vượt giới hạn tần suất.
 */
private String clientIp(HttpServletRequest req) {
    String remote = req.getRemoteAddr();
    if (!trustedProxies.contains(remote)) {
        return remote;                       // không phải proxy tin cậy → dùng IP kết nối
    }
    String forwarded = req.getHeader("CF-Connecting-IP");
    return forwarded != null ? forwarded : remote;
}
```

### 6.3 Nhật ký kiểm toán bằng aspect

```java
@Aspect
@Component
public class AuditLogAspect {

    @AfterReturning(pointcut = "@annotation(audited)", returning = "result")
    public void record(JoinPoint jp, Audited audited, Object result) {
        auditLogRepository.save(AuditLog.builder()
            .actorId(currentAdmin().id())
            .actorName(currentAdmin().displayName())   // sao chép, không chỉ tham chiếu
            .action(audited.action())
            .entityType(audited.entityType())
            .entityId(extractId(jp))
            .beforeState(snapshotBefore(jp))
            .afterState(toJson(result))
            .ipHash(ipHasher.hash(currentRequestIp()))
            .build());
    }
}
```

```java
@Audited(action = "PHOTO_PUBLISH", entityType = "PHOTO")
@PreAuthorize("hasRole('ADMIN')")
@Transactional
public PhotoDto publish(UUID photoId) {
    Photo photo = photoRepository.findById(photoId).orElseThrow(PhotoNotFound::new);

    // Ràng buộc bất biến — kiểm ở service, không tin vào việc giao diện ẩn nút
    if (photo.getProcessing() != Processing.READY) {
        throw new PhotoNotReadyException(photoId, photo.getProcessing());
    }

    storageService.promoteToPublic(photo);      // quarantine/ → public/
    photo.publish(clock.instant());
    cacheService.evictPhotoLists();
    return photoMapper.toDto(photo);
}
```

Ghi `actorName` như một bản sao chứ không chỉ giữ khoá ngoại là có chủ đích: nhật ký kiểm toán phải đọc được cả khi tài khoản gây ra hành động đã bị xoá. Một nhật ký chỉ còn `actor_id` trỏ vào hư không thì không dùng được để điều tra.

---

## 7. Module Xử lý ảnh

### 7.1 Kiểm tra tệp — theo thứ tự rẻ trước, đắt sau

> **Lưu ý về vị trí chạy.** Từ khi chuyển sang tải lên trực tiếp bằng đường dẫn ký sẵn (DD12), việc kiểm tra không còn chạy trên một `MultipartFile` mà chạy trên **đối tượng đã nằm trong vùng cách ly** của kho lưu trữ, ở bước `complete`. Bước [1] trở thành `HEAD` đối chiếu kích thước, bước [2] dùng `GET Range: bytes=0-31` để chỉ tải về 32 byte đầu. **Thứ tự và logic kiểm tra thì không đổi** — đó mới là phần quan trọng.

```java
public ValidatedImage validate(MultipartFile file) {

    // [1] Rẻ nhất: dung lượng — chặn trước khi đọc nội dung
    if (file.getSize() > MAX_BYTES) {
        throw new FileTooLargeException(file.getSize(), MAX_BYTES);
    }

    // [2] Magic bytes — KHÔNG tin file.getContentType(), đó là do client tự khai
    byte[] header = readFirstBytes(file, 32);
    ImageFormat format = MagicBytes.detect(header)
        .orElseThrow(() -> new UnsupportedMediaTypeException("Tệp không phải là ảnh"));

    // [3] SVG bị từ chối tuyệt đối: đó là tài liệu XML thực thi được,
    //     có thể chứa <script> và là vector XSS kinh điển (FR-UPL-05)
    if (format == ImageFormat.SVG) {
        throw new UnsupportedMediaTypeException("Không hỗ trợ định dạng SVG");
    }

    // [4] Đọc kích thước từ phần header, CHƯA giải nén toàn bộ ảnh.
    //     Chống "decompression bomb": một tệp PNG 2 KB có thể giải ra
    //     thành ảnh 50.000 × 50.000 điểm ảnh và ngốn hàng chục GB RAM.
    Dimension dim = ImageHeaderReader.readDimensions(file);
    long pixels = (long) dim.width * dim.height;
    if (pixels > MAX_PIXELS) {                       // 80 megapixel
        throw new ImageTooLargeException(dim, MAX_PIXELS);
    }
    if ((double) pixels / file.getSize() > MAX_PIXELS_PER_BYTE) {
        throw new SuspiciousCompressionException();
    }

    // [5] Đắt nhất: băm toàn bộ nội dung để chống trùng
    String hash = sha256(file.getInputStream());

    return new ValidatedImage(format, dim, hash);
}
```

Thứ tự này quan trọng: một kẻ tấn công gửi liên tục tệp 25 MB sẽ bị chặn ở bước [1] với chi phí gần bằng không, thay vì khiến máy chủ băm SHA-256 hàng chục lần mỗi giây. Với luồng tải lên trực tiếp, bước [1] còn rẻ hơn nữa — kích thước bị từ chối ngay khi xin đường dẫn ký sẵn, trước khi có một byte nào được truyền.

### 7.2 Mã hoá lại và sinh biến thể

```java
public void process(UUID photoId) {
    Photo photo = photoRepository.findById(photoId).orElseThrow();

    photo.setProcessing(Processing.SCANNING);
    try (InputStream in = storage.readQuarantine(photo.getStorageKey())) {

        // [1] Quét mã độc TRƯỚC khi giải mã (FR-UPL-09)
        ScanResult scan = clamAvClient.scan(in);
        if (scan.infected()) {
            photo.quarantine(scan.signature());
            alertService.notifyAdmins("Phát hiện mã độc trong tệp tải lên", photo.getId());
            return;                                    // tệp KHÔNG BAO GIỜ rời quarantine/
        }
    }

    // [2] Chép bytes GỐC y nguyên sang archive/ — không đụng vào, không nén lại.
    //     Đây là bản lưu trữ kỷ niệm, riêng tư tuyệt đối (DD14).
    photo.setProcessing(Processing.ARCHIVING);
    storage.copyQuarantineToArchive(photo.getStorageKey(), photo.getContentHash());

    // [3] Sinh biến thể bằng libvips.
    //     KHÔNG dùng ImageIO/Thumbnailator: ảnh 50 megapixel tốn ~200 MB heap
    //     cho MỘT ảnh và sẽ làm hết bộ nhớ VPS (DD13).
    photo.setProcessing(Processing.PROCESSING);
    Path local = storage.downloadToTemp(photo.getStorageKey());
    try {
        // vipsheader đọc kích thước mà không giải mã ảnh
        Dimension dim = vips.readDimensions(local);

        for (Variant v : List.of(Variant.THUMB, Variant.MEDIUM, Variant.LARGE)) {
            for (Format f : List.of(Format.AVIF, Format.JPEG)) {
                Path out = tempFor(v, f);

                // vipsthumbnail tự XOAY THEO EXIF trước, RỒI mới bỏ siêu dữ liệu.
                // Đúng thứ tự này là điều kiện để ảnh dọc chụp bằng điện thoại
                // không bị nằm ngang — lỗi rất hay gặp nếu tự làm hai bước.
                vips.thumbnail(local, out, v.maxEdge(), f.quality());

                // Khoá theo mã băm nội dung: chống trùng + cho phép đặt
                // Cache-Control immutable + chạy lại tác vụ vô hại (mục 7.3)
                String key = "public/%s/%s.%s".formatted(
                        photo.getContentHash(), v.slug(), f.ext());
                storage.writePending(key, out);

                photo.addVariant(new PhotoVariant(v, f, key, …));
                Files.deleteIfExists(out);
            }
        }
        photo.setWidth(dim.width());
        photo.setHeight(dim.height());
        photo.setProcessing(Processing.READY);
    } finally {
        Files.deleteIfExists(local);      // luôn dọn tệp tạm, kể cả khi lỗi
    }
}
```

Ba điều bắt buộc khi gọi tiến trình con, chi tiết ở [`MemoryBook-Storage-Media.md`](MemoryBook-Storage-Media.md) mục 5.3:

1. **Truyền tham số dạng danh sách**, không ghép chuỗi — chống tiêm lệnh hệ điều hành.
2. **Luôn đặt thời gian chờ** và `destroyForcibly()` — tiến trình con treo là lỗi rất khó chẩn đoán.
3. **Luôn đọc hết luồng đầu ra** — vùng đệm đầy sẽ làm tiến trình con bị chặn và treo.

> **Về ảnh HEIC từ iPhone**: libvips xử lý được nếu bản dựng có kèm libheif — kiểm tra bằng `vips --version` và thử một ảnh HEIC thật ngay từ đầu. Đây là chi tiết dễ bỏ sót cho tới lúc bạn đầu tiên trong lớp gửi ảnh từ iPhone và nhận lỗi.

### 7.3 Tác vụ nền và tính bất biến khi chạy lại

```java
@Scheduled(fixedDelay = 5_000)
public void pollOutbox() {
    List<OutboxJob> jobs = outboxRepository.claimBatch(WORKER_ID, 5);
    for (OutboxJob job : jobs) {
        try {
            handlers.get(job.getJobType()).handle(job.getPayload());
            job.markDone();
        } catch (Exception ex) {
            job.recordFailure(ex.getMessage());
            if (job.getAttempts() >= job.getMaxAttempts()) {
                job.markFailed();
                alertService.notifyAdmins("Tác vụ nền thất bại", job.getId());
            } else {
                // Giãn cách tăng dần: 1 phút → 5 phút → 25 phút
                job.rescheduleAfter(Duration.ofMinutes((long) Math.pow(5, job.getAttempts())));
            }
        }
    }
}
```

```sql
-- claimBatch: lấy việc mà không tranh chấp giữa các tiến trình
UPDATE outbox_jobs
SET status = 'RUNNING', locked_at = now(), locked_by = :workerId, attempts = attempts + 1
WHERE id IN (
  SELECT id FROM outbox_jobs
  WHERE status = 'PENDING' AND run_after <= now()
  ORDER BY run_after ASC
  LIMIT :batchSize
  FOR UPDATE SKIP LOCKED          -- ← mấu chốt: không tiến trình nào chờ tiến trình nào
)
RETURNING *;
```

**Tính bất biến khi chạy lại (NFR-REL-06)**: tác vụ xử lý ảnh có thể bị chạy lại nếu tiến trình chết giữa chừng. Nó phải cho ra cùng kết quả:

- Khoá đối tượng của biến thể được tính **tất định** từ `(photoId, variant, format)`, không dùng số ngẫu nhiên hay dấu thời gian → ghi lại lần hai chỉ đè lên chính tệp đó.
- `photo_variants` có ràng buộc `UNIQUE (photo_id, variant, format)` → chèn lại dùng `ON CONFLICT DO UPDATE`.
- Kết quả cuối cùng giống hệt nhau dù tác vụ chạy một lần hay năm lần.

Cần thêm một tác vụ dọn dẹp cho các việc bị kẹt ở `RUNNING` quá lâu (tiến trình chết trước khi kịp cập nhật trạng thái):

```sql
-- Chạy mỗi 10 phút: trả các việc "mồ côi" về hàng đợi
UPDATE outbox_jobs
SET status = 'PENDING', locked_at = NULL, locked_by = NULL
WHERE status = 'RUNNING' AND locked_at < now() - INTERVAL '15 minutes';
```

---

## 8. Cấu trúc thư mục đề xuất

### 8.1 `server/` — Spring Boot

```
server/
├── pom.xml
├── Dockerfile
├── src/main/java/vn/memorybook/
│   ├── MemoryBookApplication.java
│   ├── config/            SecurityConfig, CorsConfig, RedisConfig, StorageConfig,
│   │                      OpenApiConfig, AsyncConfig
│   ├── common/            ProblemDetail handler, IpHasher, CursorCodec, @Audited
│   ├── message/           Controller · Service · Repository · Entity · Dto · Mapper
│   ├── photo/             Controller · Service · Repository · Entity · Dto · Mapper
│   │   ├── upload/        ImageValidator, MagicBytes, ImageHeaderReader
│   │   └── processing/    ImageProcessingService, ClamAvClient, VariantGenerator
│   ├── admin/             AuthController, ModerationController, AuditLogService
│   ├── takedown/          Controller · Service · Repository · Entity
│   ├── config_content/    MembersController, TimelineController, StatsService
│   ├── storage/           ObjectStorageService (giao diện) + S3StorageService
│   ├── outbox/            OutboxJob, OutboxRepository, OutboxPoller, JobHandler
│   └── security/          RateLimitFilter, TurnstileVerifier, TotpService
├── src/main/resources/
│   ├── application.yml            cấu hình chung, KHÔNG chứa bí mật
│   ├── application-prod.yml       ghi đè cho môi trường thật
│   └── db/migration/              V1__init.sql, V2__…  (Flyway)
└── src/test/java/vn/memorybook/
    ├── integration/       Testcontainers: PostgreSQL + Redis thật
    └── unit/
```

Tổ chức **theo tính năng, không theo tầng**: mọi thứ liên quan tới ảnh nằm trong `photo/`. Cách chia `controllers/`, `services/`, `repositories/` khiến mỗi lần sửa một tính năng phải nhảy qua ba thư mục.

### 8.2 `client/` — React (khi bắt đầu làm thật)

```
client/
├── index.html                 ← bản dựng tạm hiện tại, giữ lại làm tham chiếu thiết kế
├── DESIGN.md
├── backup/
└── app/                       ← bản React
    ├── package.json
    ├── vite.config.ts
    ├── src/
    │   ├── main.tsx
    │   ├── api/               client HTTP, kiểu dữ liệu sinh từ OpenAPI
    │   ├── features/
    │   │   ├── gallery/       PhotoGrid, Lightbox, Coverflow
    │   │   ├── messages/      MessageBoard, MessageForm
    │   │   ├── contribute/    UploadForm, DropZone, ProgressBar
    │   │   ├── journey/       Timeline
    │   │   ├── members/       StudentCard, TeacherTribute
    │   │   └── admin/         LoginForm, MfaForm, ModerationQueue, AuditLog
    │   ├── scene/             tranh động SVG sân trường (chuyển từ index.html)
    │   ├── components/        thành phần dùng chung
    │   ├── hooks/
    │   └── styles/
    └── public/
        └── robots.txt         chặn toàn bộ trình thu thập (FR-PRV-06)
```

---

## 9. Xử lý lỗi & trường hợp biên cần cân nhắc

| Tình huống | Xử lý |
|---|---|
| Người dùng bấm Gửi hai lần liên tiếp | Kiểm tra trùng mã băm nội dung → lần hai nhận `409`, giao diện coi như thành công thay vì báo lỗi đỏ |
| Mạng đứt giữa lúc tải ảnh | Không có bản ghi nào được tạo (giao dịch chưa commit). Người dùng gửi lại, biểu mẫu giữ nguyên dữ liệu đã nhập |
| Tiến trình chết sau khi ghi tệp nhưng trước khi commit | Tệp thừa nằm lại trong `quarantine/`. Tác vụ dọn dẹp hằng đêm xoá tệp trong `quarantine/` không có bản ghi tương ứng |
| Tiến trình chết giữa lúc xử lý ảnh | Việc kẹt ở `RUNNING` được trả về hàng đợi sau 15 phút, xử lý lại — an toàn vì tác vụ bất biến với việc chạy lại |
| ClamAV không phản hồi | Tác vụ thất bại và thử lại. **Không** bỏ qua bước quét — thà ảnh chờ lâu còn hơn công bố tệp chưa quét |
| Redis chết | Ghi dữ liệu: từ chối (`503`) vì không kiểm tra được giới hạn tần suất. Đọc dữ liệu: bỏ qua cache, đọc thẳng cơ sở dữ liệu. Phiên quản trị mất → phải đăng nhập lại |
| Kho lưu trữ đối tượng chết | Tải ảnh lên trả `503`. Ảnh đã công bố vẫn xem được vì đang nằm trong đệm CDN |
| Ảnh HEIC từ iPhone | Xem cảnh báo ở mục 7.2 — cần libvips trong image Docker |
| Ảnh xoay ngang do EXIF Orientation | Áp dụng phép xoay theo EXIF **trước** khi mã hoá lại; nếu không, ảnh sẽ nằm ngang sau khi siêu dữ liệu bị loại bỏ. Đây là lỗi rất hay gặp và rất khó hiểu nếu không biết trước |
| Quản trị viên duyệt ảnh chưa xử lý xong | Service trả `409 Conflict`, không phụ thuộc vào việc giao diện có ẩn nút |
| Hai quản trị viên duyệt cùng một ảnh | Khoá lạc quan (`@Version`) → người thứ hai nhận `409`, giao diện tải lại và cho thấy trạng thái mới |
| Người dùng gửi ảnh đã có trong thư viện | `409` với thông điệp "ảnh này đã có trong thư viện" — không phải lỗi, chỉ là thông tin |
| Người xem mở trang khi API chết | Phần tĩnh vẫn hiển thị; các mục lấy dữ liệu hiện khung chờ rồi chuyển sang thông báo nhẹ nhàng (NFR-REL-02) |
| Album đang được dựng lại lúc có người bấm tải | Trả `202` kèm thông báo, không bắt người dùng chờ trên kết nối mở |
| Nội dung bị gỡ nhưng vẫn nằm trong đệm CDN | Bước xoá đệm Cloudflare là **bắt buộc** trong luồng gỡ; thiếu bước này thì việc gỡ chưa hoàn tất (mục 4.5 ở HLD) |

---

## 10. Checklist LLD cần hoàn thiện dần khi code

Đánh dấu khi phần tương ứng đã có mã chạy được và có kiểm thử:

- [ ] Tệp di trú Flyway `V1__init.sql` khớp đúng lược đồ ở mục 2
- [ ] Entity JPA + Repository cho `messages`, `photos`, `photo_variants`
- [ ] `CursorCodec` — mã hoá / giải mã con trỏ phân trang, có kiểm thử với dữ liệu hỏng
- [ ] `ImageValidator` đầy đủ 5 bước ở mục 7.1, kèm kiểm thử với tệp độc hại thật (SVG đội lốt `.jpg`, PNG bom nén)
- [ ] `ClamAvClient` + kiểm thử tích hợp bằng tệp thử EICAR
- [ ] Sinh biến thể + kiểm chứng **siêu dữ liệu EXIF thật sự đã biến mất** khỏi ảnh đầu ra
- [ ] Xoay ảnh theo EXIF Orientation trước khi mã hoá lại
- [ ] Poller outbox + kiểm thử chạy lại hai lần cho cùng kết quả
- [ ] `SecurityConfig` + kiểm thử `@WebMvcTest` xác nhận đường dẫn quản trị trả `401` khi chưa đăng nhập
- [ ] Đăng nhập TOTP + mã dự phòng dùng một lần
- [ ] `RateLimitFilter` + kiểm thử tích hợp với Redis trong Testcontainers
- [ ] `AuditLogAspect` + kiểm thử xác nhận `UPDATE`/`DELETE` trên `audit_logs` bị cơ sở dữ liệu từ chối
- [ ] `GlobalExceptionHandler` + kiểm thử xác nhận không có phản hồi lỗi nào chứa tên lớp Java
- [ ] Dựng album theo luồng, kiểm thử với 500 ảnh giả
- [ ] Kịch bản sao lưu + **một lần phục hồi thật đã được ghi biên bản**
- [ ] Tài liệu OpenAPI sinh tự động, đối chiếu với mục 5 của tài liệu này

---

*Tài liệu này chi tiết hoá [`MemoryBook-HLD.md`](MemoryBook-HLD.md). Danh sách công nghệ và lý do chọn: [`MemoryBook-TechStack.md`](MemoryBook-TechStack.md). Phân tích bảo mật đầy đủ: [`MemoryBook-Security.md`](MemoryBook-Security.md).*
