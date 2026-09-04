# HLD — High-Level Design
## Phượng Hồng Memories — Kỷ yếu điện tử lớp 12A1

---

## 1. Mục đích tài liệu

Mô tả kiến trúc tổng thể: các thành phần, cách chúng nói chuyện với nhau, ranh giới tin cậy, và **lý do** chọn như vậy — nhằm đáp ứng các yêu cầu đã đặt ở [`MemoryBook-SRS.md`](MemoryBook-SRS.md). Tài liệu này chưa đi vào lược đồ bảng hay chữ ký hàm; phần đó thuộc [`MemoryBook-LLD.md`](MemoryBook-LLD.md).

Danh sách công nghệ cụ thể kèm phương án thay thế nằm ở [`MemoryBook-TechStack.md`](MemoryBook-TechStack.md).

## 2. Sơ đồ kiến trúc tổng thể

```
                    ┌──────────────────────────────────────────────┐
                    │  Người dùng (trình duyệt, chủ yếu điện thoại) │
                    └───────────────────────┬──────────────────────┘
                                            │ HTTPS
                    ┌───────────────────────▼──────────────────────┐
                    │              Cloudflare (biên)                │
                    │  DNS · TLS · WAF · chống DDoS                 │
                    │  Cache ảnh · Turnstile · giới hạn tần suất thô │
                    └──────┬────────────────────────────┬──────────┘
                           │ app.<domain>               │ img.<domain>
                           │                            │  (tên miền tách riêng)
          ┌────────────────▼───────────────┐            │
          │   Caddy — reverse proxy (VPS)  │            │
          │   TLS gốc · header bảo mật     │            │
          │   phục vụ tệp tĩnh React        │            │
          └────┬──────────────────────┬────┘            │
               │ /  (tệp tĩnh)         │ /api/v1/*      │
     ┌─────────▼─────────┐   ┌─────────▼──────────────┐ │
     │  React SPA (build) │   │  Spring Boot API        │ │
     │  - Trang công khai │   │  ┌───────────────────┐  │ │
     │  - Khu quản trị    │──▶│  │ Spring Security    │  │ │
     └────────────────────┘   │  │ phiên · CSRF · 2FA │  │ │
                              │  ├───────────────────┤  │ │
                              │  │ Controller + DTO   │  │ │
                              │  │ + Bean Validation  │  │ │
                              │  ├───────────────────┤  │ │
                              │  │ Service (nghiệp vụ)│  │ │
                              │  ├───────────────────┤  │ │
                              │  │ Repository (JPA)   │  │ │
                              │  └───────────────────┘  │ │
                              │  Tác vụ nền (virtual    │ │
                              │  threads + outbox)      │ │
                              └──┬────┬────┬────┬───────┘ │
                                 │    │    │    │         │
        ┌────────────────────────┘    │    │    └──────────────────────┐
        │                             │    │                           │
┌───────▼────────┐   ┌────────────────▼┐  ┌▼─────────────────┐  ┌──────▼──────────┐
│  PostgreSQL     │   │     Redis       │  │  ClamAV (clamd)  │  │ Object Storage   │
│  Nguồn sự thật  │   │  Phiên đăng nhập│  │  Quét mã độc tệp │  │ R2 / MinIO       │
│  Flyway di trú  │   │  Bộ đếm tần suất│  │  tải lên          │  │ ảnh gốc + biến thể│
│  Xoá mềm 30 ngày│   │  Cache đọc      │  └──────────────────┘  │ (phục vụ qua CDN)│
└─────────────────┘   └─────────────────┘                        └──────────────────┘

┌─────────────────────────────────────────┐   ┌────────────────────────────────────┐
│  Giám sát: Actuator → Prometheus         │   │  Sao lưu (tác vụ định kỳ)           │
│  → Grafana · Loki (log có cấu trúc)      │   │  pg_dump hằng đêm + đồng bộ ảnh     │
│  Cảnh báo: 5xx, đĩa đầy, sao lưu lỗi     │   │  → nhà cung cấp khác, giữ 14 bản    │
└─────────────────────────────────────────┘   └────────────────────────────────────┘
```

### Ranh giới tin cậy (Trust Boundaries)

| Ranh giới | Bên ngoài (không tin) | Bên trong (tin có điều kiện) |
|---|---|---|
| **B1 — Internet → Cloudflare** | Mọi yêu cầu, kể cả yêu cầu trông hợp lệ | Cloudflare đã lọc bot và DDoS thô |
| **B2 — Cloudflare → VPS** | Header do client tự đặt (kể cả `X-Forwarded-For`) | Chỉ tin header chuyển tiếp khi kết nối đến từ dải IP của Cloudflare |
| **B3 — Trình duyệt → API** | **Toàn bộ front-end.** Mã JavaScript sửa được, biểu mẫu bỏ qua được, kiểm tra ở client vượt được | Không có gì được tin. Mọi quy tắc thực thi lại ở API |
| **B4 — API → Kho lưu trữ** | Nội dung tệp người dùng gửi lên | Chỉ tệp đã qua kiểm tra magic bytes, quét mã độc và mã hoá lại mới được ghi vào vùng công bố |
| **B5 — Tên miền ảnh** | Tệp phục vụ từ `img.<domain>` được coi như nội dung thù địch | Tách tên miền để dù có tệp độc lọt qua, nó cũng không chạm được cookie phiên của `app.<domain>` |

Nguyên tắc xuyên suốt: **ranh giới an ninh duy nhất của hệ thống là API.** Front-end chỉ là một client tiện lợi, không phải một lớp bảo vệ.

## 3. Các thành phần chính (Components)

### 3.1 Cloudflare (lớp biên)

- **Trách nhiệm**: DNS, chứng chỉ TLS, tường lửa ứng dụng, chống DDoS, bộ nhớ đệm ảnh toàn cầu, Turnstile chống bot, giới hạn tần suất mức thô.
- **Lý do chọn**: gói miễn phí đã đủ cho quy mô này và giải quyết trọn phần khó nhất mà một VPS đơn lẻ không tự làm được — hấp thụ lưu lượng tấn công trước khi nó chạm máy chủ.
- **Đánh đổi**: phụ thuộc một nhà cung cấp bên ngoài; đổi lại không phải tự vận hành WAF.

### 3.2 Caddy — reverse proxy

- **Trách nhiệm**: kết thúc TLS phía VPS, gắn header bảo mật, phục vụ tệp tĩnh của React, chuyển tiếp `/api/*` sang Spring Boot, nén phản hồi.
- **Lý do chọn**: tự động xin và gia hạn chứng chỉ Let's Encrypt, cấu hình ngắn gọn hơn Nginx đáng kể — hợp với người vận hành một mình.
- **Đánh đổi**: ít tài liệu và ví dụ hơn Nginx. Nginx là phương án thay thế hoàn toàn hợp lệ.

### 3.3 Front-end — React (tệp tĩnh)

- **Trách nhiệm**: hiển thị trang kỷ yếu và khu quản trị, gọi API, quản lý trạng thái giao diện, hoạt ảnh.
- **Đặc điểm quan trọng**: là **tệp tĩnh thuần** sau khi build — không có tiến trình Node chạy trên máy chủ. Điều này xoá bỏ nguyên một lớp bề mặt tấn công và làm việc vận hành đơn giản hẳn.
- **Trạng thái hiện tại**: bản dựng tạm `client/index.html` là một tệp HTML duy nhất dùng thư viện CDN. Bản chính thức sẽ được build, đóng gói thư viện cùng ứng dụng và không gọi CDN bên thứ ba (yêu cầu NFR-SEC-17, cần thiết để đặt được CSP nghiêm ngặt).
- **Lưu ý**: đây là phần **có thể thay công nghệ**. Kiến trúc không phụ thuộc vào React ở bất kỳ điểm nào ngoài chính thư mục `client/`.

### 3.4 Spring Boot API

- **Trách nhiệm**: toàn bộ logic nghiệp vụ, xác thực và phân quyền, kiểm tra dữ liệu vào, xử lý ảnh, kiểm duyệt, nhật ký kiểm toán.
- **Phân tầng**:

  | Tầng | Việc của tầng | Nguyên tắc |
  |---|---|---|
  | Controller | Nhận HTTP, ánh xạ DTO, kiểm tra cú pháp dữ liệu | Không chứa logic nghiệp vụ; không bao giờ nhận hay trả entity |
  | Service | Quy tắc nghiệp vụ, ranh giới giao dịch | Không biết gì về HTTP |
  | Repository | Truy cập dữ liệu qua Spring Data JPA | Truy vấn có tham số ràng buộc, không ghép chuỗi |
  | Cross-cutting | Bảo mật, giới hạn tần suất, nhật ký kiểm toán, xử lý lỗi | Cài bằng filter và aspect, không rải rác trong nghiệp vụ |

- **Mô hình luồng**: Spring Web MVC chạy trên **virtual threads** (Java 21+). Tải công việc ở đây thuần I/O (cơ sở dữ liệu, kho lưu trữ, ClamAV) — virtual threads cho gần hết lợi ích của mô hình bất đồng bộ mà vẫn giữ được mã kiểu tuần tự, dễ đọc và dễ gỡ lỗi.
- **Lý do chọn Spring Boot**: hệ sinh thái bảo mật (Spring Security) trưởng thành nhất trong các framework phổ biến, đúng trọng tâm của dự án này; ngoài ra là thứ có giá trị nhất khi đi phỏng vấn ở thị trường Việt Nam.

### 3.5 PostgreSQL — nguồn sự thật

- **Trách nhiệm**: lưu lời nhắn, siêu dữ liệu ảnh, tài khoản quản trị, nhật ký kiểm toán, yêu cầu gỡ nội dung, dữ liệu cấu hình.
- **Lý do chọn**: cần giao dịch ACID (một lượt tải ảnh phải hoặc thành công trọn vẹn, hoặc không để lại gì — NFR-REL-05), cần ràng buộc toàn vẹn ở tầng dữ liệu, và cần công cụ sao lưu/phục hồi đáng tin.
- **Ghi chú**: **không** lưu tệp ảnh trong cơ sở dữ liệu. Cơ sở dữ liệu giữ siêu dữ liệu và khoá đối tượng; tệp nằm ở kho lưu trữ đối tượng.

### 3.6 Redis — trạng thái phù du

- **Trách nhiệm**: kho phiên đăng nhập (Spring Session), bộ đếm giới hạn tần suất, bộ nhớ đệm cho các truy vấn đọc nặng (thống kê trang chủ, trang đầu của thư viện).
- **Lý do chọn**: giới hạn tần suất phải đúng khi có nhiều tiến trình ứng dụng (FR-ABU-06), và phiên phải sống sót qua lần khởi động lại ứng dụng (NFR-REL-07). Hai yêu cầu này đều loại bỏ phương án lưu trong bộ nhớ tiến trình.
- **Đánh đổi**: thêm một thành phần phải vận hành. Chấp nhận, vì nó là thành phần rẻ nhất và ổn định nhất trong nhóm.
- **Xử lý khi Redis chết**: hệ thống **fail-closed** cho ghi (từ chối nhận nội dung mới thay vì bỏ qua giới hạn tần suất) và **fail-open** cho đọc (bỏ qua cache, đọc thẳng cơ sở dữ liệu).

### 3.7 Kho lưu trữ đối tượng + CDN ảnh

- **Trách nhiệm**: giữ ảnh gốc và các biến thể; nhận trực tiếp tệp tải lên; phục vụ ảnh qua CDN.
- **Quy mô thực tế**: ảnh gốc tới 25 MB mỗi tấm, tổng kho khoảng 24 GB. Chi tiết tính toán, so sánh nhà cung cấp và các điểm tối ưu nằm ở [`MemoryBook-Storage-Media.md`](MemoryBook-Storage-Media.md).
- **Bố cục ba vùng**:

  | Vùng | Nội dung | Ai đọc được |
  |---|---|---|
  | `quarantine/` | Tệp vừa nhận, chưa quét xong. Tự xoá sau 24 giờ | Chỉ ứng dụng, không phơi ra ngoài |
  | `archive/` | **Bytes gốc y nguyên**, còn EXIF, ~20 GB | **Không ai** — kể cả người vừa tải lên. Đây là bản lưu trữ kỷ niệm |
  | `public/` | Biến thể đã bỏ siêu dữ liệu: 3 kích cỡ × AVIF/JPEG, ~4 GB | Công khai qua `img.<domain>` |

- **Vì sao tách `archive/` khỏi `public/`**: bản gốc còn nguyên toạ độ GPS nơi chụp — phục vụ ra công khai là rò rỉ vị trí của học sinh. Đồng thời giữ nguyên bytes gốc nghĩa là không mất mát chất lượng do nén lại, đúng nguyên tắc "kỷ niệm không phục hồi được" ở BRD.
- **Nhà cung cấp**: Cloudflare R2 cho môi trường thật (**egress miễn phí** — yếu tố quyết định với một trang toàn ảnh), MinIO trong Docker cho môi trường phát triển. Cùng giao thức S3 nên mã nguồn không khác một dòng.

- **Lý do tách tên miền**: nếu một tệp độc lọt qua mọi lớp kiểm tra, nó chạy trên `img.<domain>` — một nguồn (origin) khác — nên không đọc được cookie phiên của `app.<domain>`. Đây là biện pháp phòng thủ theo chiều sâu, xem chi tiết ở [`MemoryBook-Security.md`](MemoryBook-Security.md).

### 3.8 ClamAV — quét mã độc

- **Trách nhiệm**: quét mọi tệp tải lên trước khi nó rời khỏi vùng cách ly.
- **Lý do chọn**: mã nguồn mở, chạy được trong một container nhỏ, tích hợp qua giao thức TCP đơn giản.
- **Đánh đổi**: tốn khoảng 1 GB RAM cho cơ sở dữ liệu mẫu virus — đáng kể trên VPS nhỏ. Nếu bộ nhớ quá chật, phương án thay thế là gọi dịch vụ quét bên ngoài, nhưng khi đó tệp của người dùng rời khỏi hệ thống của mình, cần cân nhắc về quyền riêng tư.

### 3.9 Xử lý ảnh (tác vụ nền trong tiến trình)

- **Trách nhiệm**: quét mã độc, chép bytes gốc sang `archive/`, sinh ba kích cỡ × hai định dạng (qua đó loại bỏ siêu dữ liệu), ghi vào vùng công bố, cập nhật bản ghi.
- **Công cụ**: **libvips** gọi qua tiến trình con, không dùng thư viện xử lý ảnh thuần Java — xem DD13 và [`MemoryBook-Storage-Media.md`](MemoryBook-Storage-Media.md) mục 5.
- **Giới hạn song song**: số ảnh xử lý cùng lúc bị chặn cứng (2 luồng trên VPS 4 GB, 4 luồng trên 8 GB). libvips dùng bộ nhớ **ngoài** heap JVM nên `-Xmx` không kiểm soát được nó — giới hạn số luồng là cách duy nhất chặn được trần bộ nhớ.
- **Cách chạy**: hàng đợi ghi trong cơ sở dữ liệu theo mẫu **outbox**, một nhóm luồng đọc và xử lý. Không dùng message broker riêng.
- **Lý do**: khối lượng là vài chục ảnh mỗi ngày trong đợt cao điểm. Thêm RabbitMQ hay Kafka vào đây là thêm một thứ phải vận hành mà không giải quyết vấn đề nào có thật. Mẫu outbox trong cơ sở dữ liệu cho đúng những gì cần: bền vững qua khởi động lại, thử lại được, và chạy lại không sinh kết quả sai (NFR-REL-06).
- **Đường nâng cấp**: nếu sau này khối lượng tăng, đổi phần đọc hàng đợi sang một broker mà không phải sửa logic nghiệp vụ.

### 3.10 Giám sát & Nhật ký

- **Trách nhiệm**: phát số liệu, gom log, cảnh báo.
- **Thành phần**: Spring Boot Actuator + Micrometer → Prometheus → Grafana; log JSON có cấu trúc, gắn định danh yêu cầu, gom bằng Loki.
- **Quan trọng**: điểm cuối số liệu **không** phơi ra internet (NFR-SEC-16, FR-OPS-02) — chỉ mở trong mạng nội bộ của Docker.

### 3.11 Sao lưu

- **Trách nhiệm**: giữ cho dự án không mất dữ liệu — theo nguyên tắc định hướng ở BRD, đây là thành phần quan trọng nhất chứ không phải thứ làm sau cùng.
- **Cách làm**: `pg_dump` hằng đêm, nén và mã hoá, đẩy sang nhà cung cấp **khác** với nơi chạy ứng dụng; ảnh đồng bộ sang kho thứ hai hằng tuần; giữ 14 bản.
- **Bắt buộc**: mỗi quý phải phục hồi thử vào môi trường tạm và ghi lại kết quả. Một bản sao lưu chưa từng phục hồi thử thì chưa được tính là bản sao lưu.

---

## 4. Luồng xử lý chính (Key Flows — mức tổng quan)

### 4.1 Xem trang công khai

```
Trình duyệt → Cloudflare (cache HTML/JS/CSS) → Caddy → tệp tĩnh React
React khởi động → GET /api/v1/photos?status=published
                → GET /api/v1/messages
                → GET /api/v1/config/stats   (có cache 5 phút ở Redis)
Ảnh tải trực tiếp từ img.<domain> (Cloudflare cache), không đi qua API
```

Điểm đáng chú ý: **ảnh không đi qua Spring Boot.** API chỉ trả đường dẫn. Đây là lý do một VPS nhỏ vẫn phục vụ được đợt cao điểm — phần tốn băng thông nhất do CDN gánh.

### 4.2 Gửi lời nhắn

```
1. Người dùng điền biểu mẫu, Turnstile sinh token
2. POST /api/v1/messages  { fullname, nickname, message, turnstileToken }
3. Bộ lọc giới hạn tần suất: kiểm tra bộ đếm theo HMAC(IP) ở Redis
   └─ vượt ngưỡng → 429 + Retry-After
4. Xác minh token Turnstile với Cloudflare (phía máy chủ)
   └─ không hợp lệ → 400
5. Kiểm tra dữ liệu: độ dài, chuẩn hoá Unicode, đếm URL, chống trùng lặp
6. Lưu bản ghi với trạng thái PENDING
7. Trả 202 Accepted + { id, status: "PENDING" }
8. Giao diện hiển thị: "Lời nhắn của bạn đã được gửi, đang chờ duyệt"
```

Bước 8 là một quyết định sản phẩm có chủ đích: **không giả vờ lời nhắn đã đăng.** Người gửi cần biết sự thật, nếu không họ sẽ gửi lại nhiều lần vì tưởng lỗi.

### 4.3 Tải ảnh lên (luồng phức tạp nhất)

Ở quy mô 25 MB mỗi ảnh, tệp **không đi qua Spring Boot**. Máy chủ chỉ cấp phép và kiểm tra; bytes đi thẳng từ trình duyệt vào kho đối tượng.

```
── Giai đoạn 1: Xin phép (nhanh, không có bytes ảnh) ─────────────
1. POST /api/v1/photos/upload-intent  { fileName, size, mimeHint, turnstileToken }
2. Giới hạn tần suất + hạn ngạch dung lượng (500 MB/IP/ngày) + Turnstile
3. size > 25 MB → 413 ngay, chưa tốn byte nào
4. Tạo bản ghi Photo (AWAITING_UPLOAD) + ký URL PUT vào quarantine/, hạn 10 phút
5. Trả { photoId, uploadUrl, expiresAt }

── Giai đoạn 2: Tải lên (bytes KHÔNG qua VPS) ────────────────────
6. Trình duyệt PUT 25 MB thẳng vào kho đối tượng
   → thanh tiến trình chạy đúng, băng thông không tính hai lần,
     và dữ liệu vào kho là miễn phí

── Giai đoạn 3: Xác nhận (nhanh) ─────────────────────────────────
7.  POST /api/v1/photos/{id}/complete  { note, takenAt, category, consent }
8.  HEAD object → tệp có thật? đúng kích thước đã khai?
9.  GET Range: bytes=0-31 → đọc magic bytes  (chỉ tải 32 byte, không tải 25 MB)
    └─ không phải ảnh → 415 · là SVG → 415 (từ chối tuyệt đối)
10. Đọc kích thước ảnh mà chưa giải mã
    └─ > 80 megapixel hoặc tỉ lệ giải nén bất thường → 413
11. Tính mã băm nội dung → trùng ảnh đã có → 409
12. Cập nhật Photo (PENDING) + tạo bản ghi outbox   ← cùng một giao dịch
13. Trả 202 Accepted + { id, status: "PENDING" }

── Giai đoạn 4: Xử lý nền ────────────────────────────────────────
14. Quét mã độc bằng ClamAV
    └─ nhiễm → QUARANTINED, cảnh báo quản trị, tệp không bao giờ rời quarantine/
15. Chép bytes gốc y nguyên sang archive/   ← bản lưu trữ, không đụng vào
16. libvips sinh 6 biến thể: thumb/medium/large × AVIF/JPEG
    → tự xoay theo EXIF rồi bỏ sạch siêu dữ liệu ở bước này
17. Ghi biến thể vào vùng chờ công bố, cập nhật bản ghi
18. Đánh dấu outbox đã xử lý
    └─ thất bại → thử lại tối đa 3 lần với khoảng cách tăng dần, rồi báo lỗi

── Giai đoạn 5: Kiểm duyệt ───────────────────────────────────────
19. Quản trị viên duyệt → chuyển biến thể sang public/, trạng thái PUBLISHED
20. Ảnh xuất hiện trên trang; Cloudflare bắt đầu lưu đệm
```

Bốn lý do cho thiết kế này, chi tiết ở [`MemoryBook-Storage-Media.md`](MemoryBook-Storage-Media.md) mục 4: băng thông không đi hai lần; không chiếm dụng tài nguyên máy chủ suốt 30–60 giây; không vướng giới hạn thời gian chờ của proxy; và tận dụng đúng thứ kho đối tượng sinh ra để làm.

Cái giá phải trả là tệp nằm trong kho trước khi máy chủ kịp kiểm tra nội dung — được bù bằng sáu biện pháp ở mục 4.3 của tài liệu Storage, trong đó quan trọng nhất là `quarantine/` không bao giờ công khai và có quy tắc vòng đời tự xoá sau 24 giờ.

### 4.4 Kiểm duyệt & công bố

```
1. Quản trị viên đăng nhập (xem 4.6)
2. GET /api/v1/admin/queue?type=all&status=pending
3. PATCH /api/v1/admin/photos/{id}  { "status": "PUBLISHED" }
4. Ghi nhật ký kiểm toán: ai, lúc nào, giá trị trước → sau
5. Xoá cache Redis liên quan
6. Ảnh/lời nhắn xuất hiện ở lần gọi API tiếp theo của trang công khai
```

### 4.5 Gỡ nội dung khẩn

```
1. Người trong ảnh bấm "Yêu cầu gỡ ảnh này" → POST /api/v1/takedown-requests
2. Yêu cầu vào hàng chờ ưu tiên cao, quản trị viên nhận cảnh báo
3. Quản trị viên bấm gỡ:
   a. Đổi trạng thái sang HIDDEN  → biến mất khỏi mọi API công khai ngay
   b. Xoá mọi biến thể khỏi public/
   c. Gọi API Cloudflare xoá đệm theo đường dẫn   ← bước hay bị quên
   d. Ghi nhật ký kiểm toán
4. Trong vòng 60 giây, ảnh không còn truy cập được từ bất kỳ đâu (FR-GAL-07)
```

Bước 3c là bước dễ bỏ sót nhất và cũng là bước quan trọng nhất: một ảnh đã bị "gỡ" nhưng vẫn nằm trong đệm CDN thì với người yêu cầu, nó chưa hề bị gỡ.

### 4.6 Đăng nhập quản trị (2FA)

```
1. POST /api/v1/admin/auth/login  { username, password }
2. Giới hạn tần suất theo IP + đếm số lần sai của tài khoản
   └─ sai ≥ 5 lần / 15 phút → 423 Locked
3. So khớp mật khẩu (Argon2id). Sai → 401 với thông điệp chung
   ── luôn tốn thời gian như nhau dù tài khoản có tồn tại hay không
4. Đúng → tạo phiên tạm "chờ 2FA" (thời hạn 5 phút), trả 200 + { mfaRequired: true }
5. POST /api/v1/admin/auth/mfa  { code }
6. Xác thực TOTP (cho phép lệch ±1 khoảng 30 giây)
7. Đúng → cấp định danh phiên MỚI (chống cố định phiên), đặt cookie
          HttpOnly · Secure · SameSite=Lax
8. Cấp token CSRF cho các yêu cầu ghi tiếp theo
9. Ghi nhật ký đăng nhập thành công
```

### 4.7 Tải album

Một tệp nén chứa toàn bộ ảnh gốc sẽ nặng 20 GB — không tải nổi trên điện thoại. Thiết kế thực tế:

```
1. Tác vụ hằng đêm (chỉ chạy khi có ảnh mới được duyệt) dựng BỐN tệp nén,
   chia theo phân loại, và nén từ bản `large` 4000px chứ không phải bản gốc:
       album-lop-10.zip   ~600 MB
       album-lop-11.zip   ~700 MB
       album-lop-12.zip   ~800 MB
       album-su-kien.zip  ~400 MB
   → mỗi tệp dưới 1 GB, tải được trên điện thoại, hỏng thì chỉ tải lại phần đó
   → bản `large` đã bỏ EXIF nên tệp nén không rò rỉ toạ độ GPS
   → dùng phương thức STORED (không nén): JPEG/AVIF đã nén rồi, nén lại
     tốn CPU mà giảm gần bằng không

2. GET /api/v1/album?category=GRADE_12
   → 302 chuyển hướng tới đường dẫn ký sẵn, hạn 1 giờ
3. Người dùng tải thẳng từ kho đối tượng: không tốn băng thông VPS,
   tiếp tục được khi đứt giữa chừng, và trên R2 thì egress miễn phí
4. Nếu tệp nén đang được dựng lại → 202 kèm thông báo
5. Ngoài ra, khung xem ảnh có nút "Tải ảnh này (HD)" trỏ thẳng tới bản `large`
   — đây mới là thứ đa số người dùng thực sự cần
```

### 4.8 Sao lưu & phục hồi

```
Hằng đêm 02:00:
  pg_dump → nén → mã hoá → đẩy sang nhà cung cấp lưu trữ khác
  ghi lại kết quả; thất bại → cảnh báo ngay
Hằng tuần:
  đồng bộ toàn bộ vùng public/ sang kho sao lưu thứ hai
Hằng quý:
  phục hồi thử vào môi trường tạm, đối chiếu số bản ghi, ghi biên bản
```

---

## 5. Các quyết định thiết kế quan trọng & Đánh đổi (Design Decisions)

| # | Quyết định | Vì sao | Đánh đổi chấp nhận |
|---|---|---|---|
| DD1 | **Front-end là tệp tĩnh (SPA), không dựng phía máy chủ** | Yêu cầu FR-PRV-06 đã chặn máy tìm kiếm lập chỉ mục — nghĩa là **SEO không phải yêu cầu của dự án này**, mà SEO chính là lý do lớn nhất người ta chọn SSR. Bỏ SSR đi thì không còn tiến trình Node nào phải vận hành và bảo mật | Ảnh xem trước khi chia sẻ link (thẻ Open Graph) là tĩnh, không đổi theo từng ảnh. Chấp nhận được |
| DD2 | **Xác thực bằng cookie phiên, không phải JWT lưu ở `localStorage`** | Token trong `localStorage` là món quà cho bất kỳ lỗ hổng XSS nào. Cookie `HttpOnly` thì JavaScript không đọc được. Ngoài ra phiên thu hồi được ngay lập tức, còn JWT thì phải chờ hết hạn | Phải xử lý CSRF (Spring Security có sẵn) và cần kho phiên dùng chung (đã có Redis) |
| DD3 | **Hàng đợi outbox trong cơ sở dữ liệu, không dùng message broker** | Vài chục ảnh mỗi ngày. Thêm RabbitMQ là thêm một thứ phải vận hành, giám sát, sao lưu — để giải quyết một vấn đề chưa tồn tại | Không mở rộng được lên hàng nghìn tác vụ/giây. Không cần, và đổi được sau mà không đụng nghiệp vụ |
| DD4 | **Một PostgreSQL cho mọi dữ liệu quan hệ** | Toàn vẹn dữ liệu quan trọng hơn tối ưu hoá sớm. Một cơ sở dữ liệu là một thứ phải sao lưu, một thứ phải phục hồi | Không tối ưu riêng cho từng loại truy vấn. Ở quy mô này không đo được khác biệt |
| DD5 | **Ảnh nằm ở kho đối tượng, phục vụ từ tên miền riêng** | Băng thông do CDN gánh; và tách nguồn (origin) khiến tệp độc — nếu lọt — không chạm được cookie phiên | Thêm một dịch vụ và một tên miền phụ phải cấu hình |
| DD6 | **Kiểm duyệt trước, không phải hậu kiểm** | Trang mang tên thật của trường và của học sinh. Một nội dung xấu hiển thị 10 phút cũng đủ gây hại thật | Người đóng góp phải chờ. Đổi lại: tiêu chí S4 (0 sự cố nội dung) là thứ duy nhất đảm bảo được bằng thiết kế |
| DD7 | **Người đóng góp không cần đăng nhập** | Bắt 40 học sinh tạo tài khoản là cách chắc chắn để không ai gửi gì cả. Mục tiêu G2 quan trọng hơn sự tiện lợi khi kiểm soát | Chi phí chống lạm dụng dồn hết sang máy chủ: giới hạn tần suất, Turnstile, kiểm duyệt. Đã tính trong module ABU |
| DD8 | **Một VPS + Docker Compose, không Kubernetes** | Một người vận hành. Kubernetes cho quy mô này chỉ thêm chỗ để hỏng | Không tự mở rộng, có thời điểm ngừng khi triển khai. Chấp nhận với uptime mục tiêu 99% |
| DD9 | **Spring Web MVC + virtual threads, không WebFlux** | Tải công việc thuần I/O; virtual threads cho phần lớn lợi ích mà giữ được mã tuần tự. WebFlux khiến việc gỡ lỗi và bắt lỗi khó hơn hẳn — chi phí thật với một người mới | Thông lượng cực đại thấp hơn WebFlux. Cách xa mọi ngưỡng của dự án này |
| DD10 | **Không nạp thư viện từ CDN bên thứ ba trong bản chạy thật** | Bản dựng tạm hiện đang nạp Tailwind/GSAP/Swiper từ CDN. Như vậy thì không đặt được CSP nghiêm ngặt, và mỗi CDN là một bên có thể chèn mã vào trang | Gói JavaScript to hơn, phải có bước build. Đây là cái giá bắt buộc để đạt NFR-SEC-11 và NFR-SEC-17 |
| DD11 | **Hợp đồng API có phiên bản và được chốt trước** | Front-end được tuyên bố là "có thể đổi công nghệ sau". Chỉ đúng được nếu ranh giới giữa hai bên rõ ràng và ổn định | Đổi hợp đồng API tốn công hơn; đúng như mong muốn |
| DD12 | **Tải ảnh trực tiếp lên kho bằng URL ký sẵn, không qua máy chủ ứng dụng** | Ở mức 25 MB, cho tệp đi qua VPS là tính băng thông hai lần và chiếm dụng tài nguyên 30–60 giây mỗi lượt. Kho đối tượng sinh ra để làm đúng việc này, và dữ liệu vào thì miễn phí | Tệp nằm trong kho trước khi kiểm tra được nội dung. Bù bằng vùng cách ly không công khai + quy tắc vòng đời tự xoá + kiểm tra magic bytes bằng yêu cầu Range |
| DD13 | **libvips thay cho thư viện xử lý ảnh thuần Java** | `ImageIO` nạp toàn bộ ảnh vào bộ nhớ: ảnh 50 megapixel tốn ~200 MB heap **cho một ảnh**. Trên VPS 4–8 GB đang chạy PostgreSQL, Redis và ClamAV, đây là lỗi hết bộ nhớ chắc chắn xảy ra, không phải rủi ro lý thuyết | Phải cài binary trong image Docker và gọi qua tiến trình con — kèm theo là phải cẩn thận tuyệt đối khi dựng tham số lệnh |
| DD14 | **Tách `archive/` (bytes gốc, riêng tư) khỏi `public/` (biến thể, công khai)** | Bản gốc chứa toạ độ GPS; và giữ nguyên bytes nghĩa là không mất chất lượng do nén lại. Đồng thời làm tệp nén album nhỏ đi 8 lần | Lưu 24 GB thay vì 20 GB — chênh khoảng 1.000 đồng mỗi tháng |
| DD15 | **Cloudflare R2 làm kho chính, không phải S3 hay MinIO tự dựng** | Egress miễn phí. Một đợt cả lớp tải album (160 GB) tốn 0 đồng trên R2 và khoảng 380.000 đồng trên S3. Còn MinIO tự dựng nghĩa là đặt 20 GB ảnh không phục hồi được lên một đĩa VPS duy nhất | Phụ thuộc một nhà cung cấp. Giảm nhẹ bằng việc dùng giao thức S3 chuẩn — đổi nhà cung cấp chỉ là đổi cấu hình |

---

## 6. Ánh xạ với yêu cầu SRS (Requirement Mapping)

| Thành phần | Đáp ứng yêu cầu |
|---|---|
| Cloudflare | NFR-SEC-01, FR-ABU-08, NFR-PERF-04, FR-GAL-07 |
| Caddy | NFR-SEC-01, NFR-SEC-11, NFR-PERF-04 |
| React SPA | FR-MSG-01, FR-UPL-13, FR-GAL-01, NFR-REL-02, NFR-A11Y-01 → NFR-A11Y-06 |
| Spring Security | FR-ADM-01 → FR-ADM-06, FR-ADM-13, NFR-SEC-02, NFR-SEC-05 → NFR-SEC-09 |
| Tầng Controller + Validation | FR-MSG-02, FR-UPL-01 → FR-UPL-03, NFR-SEC-03, NFR-SEC-13 |
| Tầng Service | FR-MSG-04 → FR-MSG-08, FR-ADM-07 → FR-ADM-10 |
| Tầng Repository + JPA | NFR-SEC-04, NFR-MAIN-02 |
| PostgreSQL + Flyway | FR-OPS-07, NFR-REL-03, NFR-REL-05, NFR-MAIN-02 |
| Redis | FR-ABU-01 → FR-ABU-06, FR-CFG-03, NFR-REL-07 |
| Kho đối tượng + CDN | FR-GAL-03, FR-UPL-08, NFR-SEC-10, NFR-PERF-04 |
| ClamAV | FR-UPL-09, NFR-SEC-10 |
| Xử lý ảnh nền | FR-UPL-07, FR-UPL-08, NFR-PERF-03, NFR-REL-06 |
| Aspect nhật ký kiểm toán | FR-ADM-11, FR-ADM-12 |
| Actuator + Prometheus | FR-OPS-01, FR-OPS-02, NFR-OBS-01 → NFR-OBS-04 |
| Tác vụ sao lưu | FR-OPS-03, FR-OPS-04, NFR-REL-03, NFR-REL-04 |

---

*Tài liệu này dựa trên [`MemoryBook-SRS.md`](MemoryBook-SRS.md). Chi tiết lược đồ dữ liệu, hợp đồng API và máy trạng thái nằm ở [`MemoryBook-LLD.md`](MemoryBook-LLD.md); danh sách công nghệ cụ thể ở [`MemoryBook-TechStack.md`](MemoryBook-TechStack.md).*
