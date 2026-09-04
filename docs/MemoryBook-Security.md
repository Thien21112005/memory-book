# Security — Tài liệu Bảo mật
## Phượng Hồng Memories — Kỷ yếu điện tử lớp 12A1

---

## 1. Mục đích tài liệu

Đây là tài liệu bảo mật ở mức triển khai được ra internet công cộng. Nó trả lời ba câu: **bảo vệ cái gì, chống lại ai, và làm bằng cách nào** — kèm checklist kiểm chứng được trước khi công bố.

> ⚠️ **Miễn trừ.** Phần nói về pháp lý (Nghị định 13/2023/NĐ-CP) là tóm tắt để định hướng, **không phải tư vấn pháp lý**. Trước khi công bố một trang chứa ảnh của học sinh, nên hỏi thầy cô chủ nhiệm và người có chuyên môn.

Ba tài liệu liên quan: yêu cầu bảo mật cần nghiệm thu ở [`MemoryBook-SRS.md`](MemoryBook-SRS.md) mục 4.3; ranh giới tin cậy ở [`MemoryBook-HLD.md`](MemoryBook-HLD.md) mục 2; mã hiện thực ở [`MemoryBook-LLD.md`](MemoryBook-LLD.md) mục 6.

---

## 2. Mô hình đe doạ (Threat Model)

### 2.1 Tài sản cần bảo vệ, xếp theo mức thiệt hại nếu mất

| # | Tài sản | Mất thì sao | Mức |
|---|---|---|---|
| A1 | **Ảnh gốc của lớp** | Không phục hồi được. Đây là toàn bộ giá trị của dự án | Nghiêm trọng |
| A2 | **Quyền riêng tư của người trong ảnh** | Ảnh học sinh phát tán ra ngoài — không thu hồi được, ảnh hưởng thật đến người thật | Nghiêm trọng |
| A3 | Tài khoản quản trị | Kẻ tấn công đăng nội dung xấu dưới danh nghĩa lớp, hoặc xoá sạch dữ liệu | Cao |
| A4 | Lời nhắn đã duyệt | Mất kỷ niệm, nhưng ít hơn ảnh | Trung bình |
| A5 | Uy tín của trang | Trang mang tên thật của trường và học sinh; một lần bị bôi bẩn là ảnh hưởng đến người thật | Cao |
| A6 | Máy chủ (tài nguyên tính toán) | Bị dùng để đào tiền ảo hoặc làm bàn đạp tấn công | Trung bình |

Thứ tự này quyết định mọi đánh đổi sau đó. **Sao lưu (bảo vệ A1) được ưu tiên trước cả tính năng**, và **kiểm duyệt trước khi hiển thị (bảo vệ A2, A5) được ưu tiên trước sự tiện lợi của người đóng góp.**

### 2.2 Kẻ tấn công và động cơ

| Kẻ tấn công | Động cơ | Khả năng | Xác suất |
|---|---|---|---|
| **Bot quét tự động** | Tìm lỗ hổng đã biết, tìm máy chủ để chiếm | Thấp nhưng liên tục 24/7 | **Gần như chắc chắn** — sẽ xảy ra trong vài giờ đầu sau khi có tên miền |
| **Bot spam** | Chèn liên kết quảng cáo, SEO rác | Thấp | Rất cao |
| **Người quen nghịch phá** | Trêu chọc, đăng nội dung bậy | Trung bình — có thể biết đường dẫn và bối cảnh | Trung bình |
| **Người có mâu thuẫn cá nhân với ai đó trong lớp** | Bôi nhọ, lấy ảnh dùng sai mục đích | Trung bình — có động cơ nhắm đích | Thấp nhưng thiệt hại cao |
| **Kẻ tấn công có kỹ năng, nhắm đích** | Ít động cơ với một trang kỷ yếu lớp | Cao | Rất thấp |
| **Chính người phát triển (lỗi vận hành)** | Không có — xoá nhầm, cấu hình sai | — | **Cao** — đây là nguyên nhân mất dữ liệu phổ biến nhất |

Nhận định quan trọng: mối đe doạ thực tế nhất **không phải** kẻ tấn công tinh vi, mà là **bot tự động** và **lỗi vận hành của chính mình**. Thiết kế phải phản ánh điều đó: tự động hoá phần vá lỗi và sao lưu, và làm cho thao tác nguy hiểm khó thực hiện nhầm.

### 2.3 Các đường tấn công chính

```
[1] Biểu mẫu công khai (không đăng nhập)
    → spam, chèn mã, làm ngập dữ liệu
    ⇒ Chống: giới hạn tần suất · Turnstile · kiểm duyệt trước · kiểm tra dữ liệu

[2] Tải tệp lên  ← ĐƯỜNG NGUY HIỂM NHẤT
    → thực thi mã, XSS qua tệp, bom nén, phát tán mã độc
    ⇒ Chống: magic bytes · từ chối SVG · mã hoá lại · ClamAV · tên miền tách biệt

[3] Đăng nhập quản trị
    → dò mật khẩu, nhồi thông tin đăng nhập rò rỉ, chiếm phiên
    ⇒ Chống: Argon2id · 2FA · khoá tạm · cookie HttpOnly · CSRF

[4] API
    → truy cập trái phép, IDOR, tiêm SQL, lộ dữ liệu qua thông báo lỗi
    ⇒ Chống: mặc định từ chối · truy vấn tham số · DTO · Problem Details không lộ chi tiết

[5] Phụ thuộc & chuỗi cung ứng
    → thư viện có lỗ hổng, CDN bên thứ ba bị chiếm
    ⇒ Chống: quét tự động trong CI · ghim phiên bản · không nạp mã từ CDN ngoài

[6] Hạ tầng
    → SSH bị dò, cổng cơ sở dữ liệu phơi ra ngoài, container chạy quyền root
    ⇒ Chống: chỉ đăng nhập bằng khoá · không mở cổng thừa · người dùng thường trong container
```

---

## 3. Đối chiếu OWASP Top 10 (2021)

### A01 — Broken Access Control (Kiểm soát truy cập hỏng)

Đây là hạng mục đứng đầu bảng OWASP, và cũng là hạng mục dễ mắc nhất.

| Biện pháp | Hiện thực |
|---|---|
| **Mặc định từ chối** | `.anyRequest().denyAll()` trong `SecurityConfig`. Đường dẫn mới quên khai báo sẽ bị chặn, chứ không vô tình mở công khai |
| Kiểm quyền ở tầng service | `@PreAuthorize("hasRole('ADMIN')")` trên phương thức, không chỉ ở tầng controller |
| Không dựa vào giao diện | Giao diện quản trị ẩn nút "Duyệt" khi ảnh chưa xử lý xong; **tầng service vẫn kiểm lại và trả `409`** |
| Chống IDOR | Khoá chính là UUID, không đoán được. Mọi truy vấn quản trị vẫn kiểm quyền, không dựa vào việc "UUID khó đoán" |
| Tách rõ điểm cuối | Điểm cuối công khai và điểm cuối quản trị nằm ở hai nhánh đường dẫn khác nhau (`/api/v1/...` và `/api/v1/admin/...`), không trộn lẫn |
| CORS chặt | Danh sách nguồn cụ thể, không dùng `*`; `allowCredentials(true)` chỉ đi kèm danh sách nguồn tường minh |
| Không lộ dữ liệu thừa | API công khai trả `displayName` đã tính sẵn, **không trả `fullname`/`nickname` thô** |

**Kiểm chứng**: viết kiểm thử `@WebMvcTest` khẳng định mọi đường dẫn `/api/v1/admin/**` trả `401` khi chưa đăng nhập. Đây là kiểm thử phải có, không phải nên có.

### A02 — Cryptographic Failures (Lỗi mật mã)

| Biện pháp | Hiện thực |
|---|---|
| TLS mọi nơi | Caddy tự xin và gia hạn chứng chỉ Let's Encrypt; HTTP chuyển hướng sang HTTPS |
| HSTS | `max-age=31536000; includeSubDomains` — trình duyệt tự từ chối kết nối không mã hoá ở lần sau |
| Băm mật khẩu | **Argon2id** qua `Argon2PasswordEncoder`. Không dùng MD5, SHA-1, SHA-256 trần — chúng nhanh, mà nhanh chính là điều tồi tệ nhất cho việc băm mật khẩu |
| Bí mật TOTP | Mã hoá trước khi lưu vào cơ sở dữ liệu, không lưu bản rõ |
| Mã dự phòng 2FA | Băm, dùng một lần, đánh dấu `used_at` sau khi dùng |
| Băm IP | HMAC-SHA256 **có khoá bí mật**, không phải SHA-256 trần — không gian IPv4 chỉ 2³² giá trị, bảng tra cứu SHA-256 cho toàn bộ IPv4 dựng được trong vài giờ |
| Sao lưu mã hoá | Bản `pg_dump` được mã hoá trước khi đẩy sang nhà cung cấp bên ngoài |
| Không tự viết mật mã | Dùng thư viện chuẩn. Đây không phải chỗ để sáng tạo |

### A03 — Injection (Tiêm mã)

| Loại | Chống bằng |
|---|---|
| **SQL** | Spring Data JPA sinh truy vấn có tham số ràng buộc. Với native query, **luôn** dùng `:param`, tuyệt đối không ghép chuỗi. Ép quy tắc này bằng kiểm thử ArchUnit hoặc rà soát mã |
| **XSS** | React tự khử ký tự đặc biệt khi render. Quy tắc: **không bao giờ dùng `dangerouslySetInnerHTML`**. Cộng thêm CSP làm lớp phòng thủ thứ hai |
| **XSS qua tệp tải lên** | Từ chối SVG tuyệt đối; ảnh được mã hoá lại; phục vụ từ tên miền riêng — ba lớp độc lập |
| **Tiêm lệnh hệ điều hành** | Khi gọi `libvips` để xử lý HEIC: **luôn truyền tham số dạng mảng**, không bao giờ dựng chuỗi lệnh có chứa tên tệp do người dùng cung cấp. Ngoài ra tên tệp thật là do hệ thống sinh, không lấy từ đầu vào |
| **Tiêm header / phân tách phản hồi** | Không đưa dữ liệu người dùng vào header phản hồi |
| **Tiêm log** | Log ở dạng JSON có cấu trúc; dữ liệu người dùng là giá trị của trường, không phải một phần chuỗi log tự do |
| **Giải tuần tự hoá không an toàn** | Không dùng Java serialization cho dữ liệu từ bên ngoài. Chỉ JSON qua Jackson với DTO tường minh |

### A04 — Insecure Design (Thiết kế thiếu an toàn)

Đây là hạng mục không vá được bằng thư viện — nó nằm ở quyết định thiết kế. Các trường hợp lạm dụng đã được tính từ đầu:

| Kịch bản lạm dụng | Thiết kế đối phó |
|---|---|
| "Tôi gửi 10.000 lời nhắn quảng cáo" | Giới hạn tần suất + Turnstile + kiểm duyệt trước — nội dung không bao giờ tự lên trang |
| "Tôi tải lên ảnh khiêu dâm cho cả lớp thấy" | Kiểm duyệt trước (DD6). Đây là lý do tồn tại của quyết định đó |
| "Tôi tải lên 1.000 ảnh làm đầy đĩa" | Giới hạn tần suất theo IP + giới hạn dung lượng + cảnh báo khi đĩa còn dưới 15% |
| "Tôi gửi ảnh 2 KB giải nén thành 40 GB" | Chặn theo số điểm ảnh và tỉ lệ giải nén, đọc từ header trước khi giải mã |
| "Tôi lấy ảnh của bạn cùng lớp đăng lên để trêu" | Ô đồng thuận bắt buộc + kiểm duyệt + kênh yêu cầu gỡ |
| "Tôi dò mã tra cứu yêu cầu gỡ của người khác để đọc nội dung" | Điểm cuối tra cứu chỉ trả trạng thái, không trả lý do hay thông tin liên hệ |
| "Tôi dò tên tài khoản quản trị bằng cách đo thời gian phản hồi" | Luôn chạy phép băm Argon2id kể cả khi tài khoản không tồn tại |
| "Tôi xoá nhầm toàn bộ ảnh với tư cách quản trị viên" | Xoá mềm + hoàn tác 30 ngày + xác nhận hai bước + nhật ký kiểm toán |

### A05 — Security Misconfiguration (Cấu hình sai)

| Hạng mục | Cấu hình đúng |
|---|---|
| Dấu vết ngăn xếp | `server.error.include-stacktrace: never` và `include-message: never` |
| Điểm cuối Actuator | Chỉ mở `health` và `prometheus`, chạy trên cổng riêng 9090, **không phơi ra internet** |
| Swagger UI | `springdoc.swagger-ui.enabled=false` ở môi trường thật |
| Cổng cơ sở dữ liệu | PostgreSQL và Redis chỉ nghe trên mạng nội bộ Docker. **Không bao giờ mở 5432 ra internet** |
| Tài khoản mặc định | Không có tài khoản seed nào trong tệp di trú. Tài khoản quản trị đầu tiên tạo bằng lệnh thủ công lúc cài đặt |
| Header bảo mật | Bộ đầy đủ ở mục 6 dưới đây |
| Danh sách thư mục | Caddy không liệt kê thư mục |
| Header phiên bản | Ẩn `Server` và mọi header lộ phiên bản phần mềm |
| Quyền trong container | Chạy bằng người dùng thường (UID 1001), không phải `root` |
| Biến môi trường | Không có bí mật nào trong image Docker hay trong kho mã |

### A06 — Vulnerable and Outdated Components (Thành phần có lỗ hổng)

| Biện pháp | Công cụ |
|---|---|
| Quét phụ thuộc tự động | Trivy hoặc OWASP Dependency-Check trong CI; **lỗ hổng nghiêm trọng làm dừng bản dựng** |
| Quét image Docker | Trivy quét cả tầng hệ điều hành cơ sở |
| Cập nhật tự động | Dependabot hoặc Renovate mở pull request hằng tuần |
| Ghim phiên bản | Ghim phiên bản chính xác trong `pom.xml` và `package-lock.json`; ghim thẻ image cơ sở, không dùng `latest` |
| **Không nạp mã từ CDN bên thứ ba** | Bản dựng tạm đang nạp Tailwind, GSAP, Swiper từ CDN. Mỗi CDN là một bên có thể chèn mã tuỳ ý vào trang. Bản chạy thật đóng gói tất cả cùng ứng dụng (NFR-SEC-17) |
| SBOM | Sinh danh mục thành phần (CycloneDX) khi build, để tra ngược khi có lỗ hổng mới công bố |

Quy tắc vận hành: **chặn bản dựng khi có lỗ hổng nghiêm trọng, chứ không chỉ cảnh báo.** Cảnh báo mà không chặn thì sẽ bị bỏ qua sau đúng hai tuần.

### A07 — Identification and Authentication Failures (Lỗi xác thực)

| Biện pháp | Hiện thực |
|---|---|
| Đa yếu tố | TOTP bắt buộc cho tài khoản quản trị, có mã dự phòng dùng một lần |
| Chống dò mật khẩu | Khoá tài khoản 15 phút sau 5 lần sai; giới hạn tần suất theo IP 10 lần/15 phút |
| Chống dò tên tài khoản | Thông báo lỗi chung "Thông tin đăng nhập không đúng"; luôn chạy phép băm để thời gian phản hồi không đổi |
| Chính sách mật khẩu | Tối thiểu 12 ký tự; đối chiếu với danh sách mật khẩu đã rò rỉ; **không** ép đổi định kỳ vô nghĩa (khiến người dùng đặt mật khẩu yếu dần) |
| Chống cố định phiên | Cấp định danh phiên mới sau khi đăng nhập thành công |
| Thời hạn phiên | 8 giờ không hoạt động, tối đa 24 giờ |
| Đăng xuất thật | Huỷ phiên ở phía máy chủ, không chỉ xoá cookie |
| Cookie an toàn | `HttpOnly` (JavaScript không đọc được) · `Secure` (chỉ qua HTTPS) · `SameSite=Lax` (chống CSRF cơ bản) |
| Ghi nhận | Mọi lần đăng nhập thành công và thất bại đều vào nhật ký kiểm toán |

**Vì sao cookie phiên chứ không phải JWT trong `localStorage`:** token nằm ở `localStorage` thì bất kỳ đoạn JavaScript nào chạy trên trang cũng đọc được — một lỗ hổng XSS duy nhất là mất tài khoản. Cookie `HttpOnly` thì JavaScript không chạm tới được. Thêm nữa, phiên thu hồi được ngay lập tức, còn JWT thì phải chờ hết hạn hoặc phải dựng thêm danh sách đen — tức là lại quay về cần một kho trạng thái tập trung, đúng thứ mà JWT được quảng cáo là giúp tránh.

### A08 — Software and Data Integrity Failures (Lỗi toàn vẹn)

| Biện pháp | Hiện thực |
|---|---|
| Toàn vẹn phụ thuộc | `package-lock.json` và `pom.xml` ghim phiên bản, commit vào kho mã |
| CI đáng tin | Chỉ triển khai từ pipeline, không bao giờ build trên máy cá nhân rồi đẩy lên |
| Gắn thẻ image theo commit | Mỗi image mang mã commit → truy ngược được chính xác mã nguồn đang chạy |
| Quay lui được | Giữ ba thẻ image gần nhất để quay lui nhanh khi triển khai lỗi |
| CSRF | Bật cho mọi yêu cầu ghi dùng cookie phiên; token qua cookie `XSRF-TOKEN` + header `X-XSRF-TOKEN` |
| Không giải tuần tự hoá dữ liệu không tin cậy | Chỉ JSON qua DTO tường minh, không dùng Java serialization |

### A09 — Security Logging and Monitoring Failures (Thiếu ghi nhận và giám sát)

| Ghi nhận cái gì | Chi tiết |
|---|---|
| Xác thực | Đăng nhập thành công, thất bại, khoá tài khoản, xác thực 2FA thất bại |
| Kiểm duyệt | Mọi thao tác duyệt/từ chối/ẩn/xoá, kèm giá trị trước và sau |
| Vượt giới hạn | Mỗi lần trả `429`, để phát hiện đợt tấn công đang diễn ra |
| Bảo mật tệp | Mỗi lần ClamAV phát hiện mã độc, mỗi lần từ chối tệp không hợp lệ |
| Lỗi hệ thống | Mọi phản hồi `5xx`, kèm `traceId` |

| **Không** được ghi | Vì sao |
|---|---|
| Mật khẩu, kể cả khi sai | Nhật ký bị lộ là mất luôn mật khẩu |
| Cookie phiên, token CSRF | Ai đọc được log là chiếm được phiên |
| Địa chỉ IP thô | Dữ liệu cá nhân — chỉ lưu HMAC (NFR-PRV-01) |
| Nội dung ảnh hoặc lời nhắn đầy đủ | Không cần thiết cho việc gỡ lỗi |

**Cảnh báo tự động** (NFR-OBS-03): tỉ lệ `5xx` > 1% trong 5 phút · đĩa còn dưới 15% · sao lưu đêm thất bại · chứng chỉ TLS còn dưới 14 ngày · ClamAV phát hiện mã độc · hơn 20 lần đăng nhập thất bại trong một giờ.

Nhật ký kiểm toán được bảo vệ ở tầng cơ sở dữ liệu, không chỉ ở tầng mã:

```sql
REVOKE UPDATE, DELETE ON audit_logs FROM app_user;
```

Đây là khác biệt giữa "code không xoá nhật ký" và "cơ sở dữ liệu không cho xoá nhật ký". Chỉ cái thứ hai mới đứng vững khi chính ứng dụng bị chiếm.

### A10 — Server-Side Request Forgery (SSRF)

Hiện tại **hệ thống không có tính năng nào tải nội dung từ URL do người dùng cung cấp** — đây là cách phòng chống tốt nhất: không mở đường tấn công.

Nếu sau này thêm tính năng "nhập ảnh từ đường dẫn" (một yêu cầu rất dễ phát sinh), bắt buộc phải có: danh sách giao thức cho phép (chỉ `https`), **chặn dải IP nội bộ** (`127.0.0.0/8`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `169.254.0.0/16` — đặc biệt dải cuối là điểm truy cập siêu dữ liệu của nhà cung cấp đám mây), kiểm tra lại IP **sau khi** phân giải DNS để chống tấn công đổi bản ghi DNS giữa chừng, không đi theo chuyển hướng, và đặt thời gian chờ ngắn.

---

## 4. Bảo mật tải tệp lên — phân tích sâu

Đây là bề mặt tấn công lớn nhất của dự án. Một trang cho người lạ tải tệp lên mà không có lớp phòng thủ đúng là một trong những cách nhanh nhất để mất máy chủ.

### 4.1 Bảy lớp phòng thủ

```
Tệp người dùng gửi lên
   │
   ├─[1] Giới hạn dung lượng ────── chặn trước khi đọc nội dung (rẻ nhất)
   │
   ├─[2] Magic bytes ────────────── loại thật đọc từ nội dung tệp,
   │                                KHÔNG tin Content-Type do client khai
   │
   ├─[3] Từ chối SVG ────────────── tuyệt đối, kể cả khi hợp lệ về cú pháp
   │
   ├─[4] Chặn bom nén ───────────── giới hạn số điểm ảnh và tỉ lệ giải nén,
   │                                đọc từ header trước khi giải mã
   │
   ├─[5] Cách ly ────────────────── ghi vào quarantine/, chưa công bố
   │
   ├─[6] Quét mã độc ────────────── ClamAV; nhiễm thì không bao giờ ra khỏi đây
   │
   ├─[7] Mã hoá lại ─────────────── giải mã thành điểm ảnh rồi ghi ra tệp mới.
   │                                Mọi thứ không phải điểm ảnh biến mất:
   │                                EXIF, GPS, XMP, và cả mã nhúng nếu có
   │
   └─▶ public/ trên tên miền riêng ─ lớp cuối: dù có gì lọt qua, nó cũng không
                                     chạm được cookie phiên của tên miền ứng dụng
```

### 4.2 Vì sao từ chối SVG tuyệt đối

SVG **không phải ảnh** theo nghĩa an toàn — nó là tài liệu XML có thể chứa `<script>`, `<foreignObject>` với HTML nhúng, và tham chiếu tài nguyên bên ngoài. Một tệp `.svg` được phục vụ với `Content-Type: image/svg+xml` và mở trực tiếp trong trình duyệt sẽ **thực thi JavaScript trong ngữ cảnh của tên miền đó**.

Có thể lọc SVG bằng thư viện khử độc, nhưng đó là cuộc chạy đua không đáng tham gia cho một trang kỷ yếu — không ai cần gửi ảnh vector vào album lớp. Từ chối thẳng là lựa chọn đúng.

### 4.3 Vì sao mã hoá lại thay vì chỉ loại bỏ EXIF

Có thư viện chỉ xoá khối EXIF khỏi tệp gốc. Cách đó nhanh hơn nhưng yếu hơn: nó giả định mình biết hết các chỗ có thể giấu dữ liệu. Mã hoá lại thì khác về bản chất — ảnh đầu ra được **dựng lại từ mảng điểm ảnh**, nên bất cứ thứ gì không phải điểm ảnh đều không có đường đi vào tệp mới. Dữ liệu GPS trong ảnh chụp bằng điện thoại biến mất; mã nhúng trong khối siêu dữ liệu cũng vậy.

Một chi tiết bắt buộc đi kèm: **phải áp dụng phép xoay theo EXIF Orientation trước khi mã hoá lại.** Nếu không, ảnh chụp dọc bằng điện thoại sẽ nằm ngang sau khi xử lý — lỗi rất hay gặp và rất khó hiểu nếu không biết trước.

### 4.4 Vì sao tách tên miền phục vụ ảnh

Ảnh phục vụ từ `img.<domain>`, không phải `app.<domain>`. Đây là lớp phòng thủ theo chiều sâu: giả sử mọi lớp trên đều thất bại và một tệp thực thi được lọt qua, nó sẽ chạy trong ngữ cảnh của `img.<domain>` — một nguồn (origin) khác — nên **không đọc được cookie phiên** của ứng dụng. Chi phí là một bản ghi DNS; lợi ích là biến một sự cố "mất tài khoản quản trị" thành "một tệp rác trong kho".

Kèm theo:

```
X-Content-Type-Options: nosniff
Content-Disposition: inline; filename="..."
Content-Security-Policy: sandbox
```

### 4.5 Đánh đổi nếu bỏ ClamAV

ClamAV ngốn khoảng 1 GB RAM cho cơ sở dữ liệu mẫu virus — đáng kể trên một VPS nhỏ. Nếu buộc phải bỏ:

- Vẫn còn sáu lớp phòng thủ khác, và lớp mã hoá lại (bước 7) đã loại bỏ phần lớn nguy cơ với ảnh.
- Rủi ro còn lại chủ yếu là **phát tán**: ảnh nhiễm mã độc nằm trong album được người khác tải về.
- Với quy mô 40 người quen biết nhau, rủi ro này thấp — nhưng phải là một quyết định có ý thức, ghi lại, chứ không phải bỏ qua vì quên.

**Phương án trung gian đáng cân nhắc**: giữ ClamAV nhưng chạy **theo lô** (`clamscan` mỗi 10 phút cho nhóm ảnh đang chờ) thay vì chạy thường trực (`clamd`). RAM chỉ bị chiếm trong lúc quét. Vì ảnh vốn đã phải nằm chờ người duyệt, việc chờ thêm 10 phút không mất mát gì về trải nghiệm. Chi tiết ở [`MemoryBook-Storage-Media.md`](MemoryBook-Storage-Media.md) mục 8.

### 4.6 Bảo mật khi tải lên trực tiếp bằng đường dẫn ký sẵn

Ở quy mô ảnh 25 MB, tệp được tải **thẳng từ trình duyệt vào kho đối tượng**, không qua máy chủ ứng dụng (quyết định DD12). Điều này đổi mô hình bảo mật: **tệp nằm trong kho trước khi máy chủ kịp kiểm tra nội dung.**

Sáu biện pháp bù lại:

| # | Biện pháp | Chống được gì |
|---|---|---|
| 1 | **Đường dẫn ký sẵn hẹp và ngắn hạn** — chỉ `PUT`, chỉ đúng một khoá đã định, hạn 10 phút | Không dùng lại được, không ghi đè được tệp khác, không liệt kê được bucket |
| 2 | **Ràng buộc kích thước trong chữ ký** — `content-length-range` nếu nhà cung cấp hỗ trợ presigned POST; nếu chỉ có presigned PUT thì ký kèm `Content-Length` và **kiểm lại bằng `HEAD` ở bước xác nhận** | Khai 1 MB rồi tải lên 5 GB |
| 3 | **`quarantine/` không bao giờ công khai** — không gắn tên miền, không có quyền đọc công khai | Dùng kho làm nơi chứa và phát tán tệp; và tệp độc chưa quét không ai chạm được, kể cả người vừa tải lên |
| 4 | **Đọc magic bytes bằng yêu cầu Range** — `GET Range: bytes=0-31` | Kiểm tra loại tệp thật mà chỉ tải về 32 byte thay vì 25 MB |
| 5 | **Quy tắc vòng đời tự xoá sau 24 giờ** | Xin 100 đường dẫn ký sẵn, tải lên 2 GB rác, không bao giờ gọi `complete` — bạn sẽ trả tiền lưu trữ cho đống rác đó mãi mãi nếu thiếu biện pháp này |
| 6 | **Hạn ngạch theo dung lượng, không chỉ theo số lượt** — 500 MB/IP/ngày | "10 lượt/giờ" là vô nghĩa khi mỗi lượt có thể là 25 MB |

Điểm quan trọng nhất về mặt nguyên tắc: **đường dẫn ký sẵn là một khoản uỷ quyền có giới hạn, không phải một cánh cửa mở.** Nó chỉ cho phép ghi đúng một tệp, vào đúng một chỗ, trong đúng một khoảng thời gian ngắn, và chỗ đó là vùng cách ly mà không ai đọc được.

---

## 5. Chống lạm dụng

| Điểm cuối | Giới hạn | Vì sao |
|---|---|---|
| `POST /messages` | 5 / IP / giờ, 30 / IP / ngày | Một người bình thường viết vài lời nhắn, không phải vài trăm |
| `POST /photos/upload-intent` | 10 / IP / giờ, 50 / IP / ngày | Đủ để tải một album nhỏ trong một buổi |
| **Dung lượng ảnh** | **500 MB / IP / ngày** | Đếm theo số lượt là chưa đủ khi mỗi lượt tới 25 MB — đây mới là hạn ngạch có tác dụng thật |
| `POST /takedown-requests` | 3 / IP / giờ | Đủ cho nhu cầu thật, chặn được việc làm ngập hàng chờ |
| `POST /admin/auth/login` | 10 / IP / 15 phút | Chống dò mật khẩu |
| `GET /album` | 3 / IP / giờ | Tệp nén nặng, chống làm cạn băng thông |
| Các điểm cuối đọc | 120 / IP / phút | Rộng rãi cho người dùng thật, vẫn chặn được bot quét |

Ba chi tiết dễ làm sai:

1. **Bộ đếm phải nằm ở kho dùng chung (Redis)**, không phải bộ nhớ tiến trình — nếu không, chạy hai bản ứng dụng là giới hạn tăng gấp đôi.
2. **Chỉ tin header `X-Forwarded-For` / `CF-Connecting-IP` khi kết nối đến từ dải IP của proxy đã cấu hình.** Tin vô điều kiện thì ai cũng tự đặt header để vượt giới hạn — biến toàn bộ cơ chế thành trang trí.
3. **Redis chết thì từ chối ghi (`503`), không bỏ qua kiểm tra.** Fail-closed cho ghi, fail-open cho đọc.

**Cloudflare Turnstile** đứng trước tất cả, xác minh phía máy chủ. Không dùng CAPTCHA hình ảnh: nó cản người dùng thật (đặc biệt trên điện thoại) nhiều hơn cản bot.

---

## 6. Header bảo mật & CSP

```
Content-Security-Policy: default-src 'self';
    script-src 'self';
    style-src 'self';
    img-src 'self' https://img.example data:;
    font-src 'self';
    connect-src 'self' https://challenges.cloudflare.com;
    frame-src https://challenges.cloudflare.com;
    object-src 'none';
    base-uri 'self';
    form-action 'self';
    frame-ancestors 'none';
    upgrade-insecure-requests

Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=(), interest-cohort=()
X-Robots-Tag: noindex, noimageindex, nofollow
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Resource-Policy: same-site
```

Điểm quan trọng nhất: **`script-src 'self'` — không có `'unsafe-inline'`, không có tên miền CDN nào.** Đây là ràng buộc khiến bản chạy thật không thể giữ cách làm của bản dựng tạm (nạp Tailwind/GSAP/Swiper từ CDN, đặt `tailwind.config` trong thẻ `<script>` nội tuyến). Một CSP có `'unsafe-inline'` gần như vô dụng trước XSS — đó chính là kiểu tấn công mà nó sinh ra để chặn.

`frame-ancestors 'none'` chống nhúng trang vào iframe để lừa người dùng bấm nhầm. `X-Robots-Tag` là một phần của yêu cầu quyền riêng tư (FR-PRV-06), không phải tối ưu tìm kiếm.

---

## 7. Bảo mật hạ tầng

### 7.1 Máy chủ

| Hạng mục | Cấu hình |
|---|---|
| SSH | **Chỉ đăng nhập bằng khoá**, tắt hoàn toàn đăng nhập bằng mật khẩu, tắt đăng nhập `root`, đổi cổng mặc định |
| Tường lửa | `ufw` mặc định từ chối vào; chỉ mở 22 (SSH, giới hạn theo IP nếu được), 80, 443 |
| Chống dò | `fail2ban` cho SSH |
| Vá lỗi | `unattended-upgrades` cho bản vá bảo mật của hệ điều hành |
| Người dùng | Ứng dụng chạy bằng người dùng thường, không phải `root` |
| Giám sát đĩa | Cảnh báo khi còn dưới 15% — đĩa đầy làm PostgreSQL dừng ghi |

### 7.2 Docker

| Hạng mục | Cấu hình |
|---|---|
| Người dùng trong container | UID 1001, không phải `root` |
| Hệ tệp | `read_only: true` cho container ứng dụng, cấp `tmpfs` cho thư mục tạm |
| Nâng quyền | `security_opt: ["no-new-privileges:true"]` |
| Quyền hệ thống | `cap_drop: [ALL]`, chỉ thêm lại nếu thật sự cần |
| Cổng | Chỉ `caddy` mở cổng ra ngoài. Mọi container khác chỉ nghe trên mạng nội bộ |
| Image cơ sở | Ghim thẻ cụ thể, không dùng `latest`; quét bằng Trivy |
| Giới hạn tài nguyên | Đặt `mem_limit` cho từng dịch vụ để một container không kéo sập cả máy |

### 7.3 Bí mật & cấu hình

| Nguyên tắc | Cách làm |
|---|---|
| Không bao giờ vào kho mã | `.env` nằm trong `.gitignore`; cung cấp `.env.example` chỉ có tên biến, không có giá trị |
| Phát hiện lỡ commit | `gitleaks` chạy như git hook và trong CI |
| Nếu lỡ commit | **Coi bí mật đó là đã lộ vĩnh viễn.** Xoay vòng khoá ngay. Xoá khỏi lịch sử git là không đủ — nó đã nằm trên máy người khác |
| Danh sách bí mật | Mật khẩu PostgreSQL · mật khẩu Redis · khoá kho đối tượng · khoá HMAC băm IP · khoá mã hoá bí mật TOTP · khoá bí mật Turnstile · token API Cloudflare · khoá SSH triển khai |
| Xoay vòng | Ít nhất mỗi năm một lần, và ngay lập tức khi nghi ngờ lộ |

---

## 8. Quyền riêng tư & Tuân thủ

### 8.1 Vì sao mục này quan trọng hơn vẻ ngoài của nó

Đây không phải một trang thương mại điện tử. Đây là **ảnh của người thật, phần lớn còn ở tuổi vị thành niên tại thời điểm chụp, kèm tên thật và tên trường thật.** Một sự cố ở đây không phải là "mất dữ liệu" — nó ảnh hưởng trực tiếp đến cuộc sống của những người trong ảnh.

### 8.2 Nguyên tắc

| Nguyên tắc | Áp dụng |
|---|---|
| **Thu thập tối thiểu** | Chỉ tên hiển thị, nội dung, ảnh và bản băm IP. Không thu thập email trừ khi người dùng tự nguyện để lại cho việc gỡ nội dung |
| **Mục đích rõ ràng** | Trang "Quyền riêng tư" nêu rõ dữ liệu dùng làm gì |
| **Thời hạn lưu** | Bản băm IP xoá sau 90 ngày; nội dung bị từ chối xoá sau 30 ngày |
| **Đồng thuận** | Ô đánh dấu bắt buộc khi gửi ảnh: người gửi xác nhận có quyền chia sẻ và những người trong ảnh đồng ý |
| **Quyền được gỡ** | Kênh yêu cầu gỡ không cần tài khoản, xử lý trong 48 giờ |
| **Không theo dõi** | Không công cụ phân tích đặt cookie theo dõi; nếu cần đo lượt xem, dùng công cụ không dùng cookie |
| **Không lập chỉ mục** | `robots.txt` chặn toàn bộ + header `X-Robots-Tag: noindex, noimageindex` |

### 8.3 Về Nghị định 13/2023/NĐ-CP

Nghị định về bảo vệ dữ liệu cá nhân đặt ra một số nghĩa vụ liên quan trực tiếp tới dự án này: hình ảnh cá nhân là dữ liệu cá nhân; việc xử lý cần có sự đồng ý của chủ thể dữ liệu; chủ thể có quyền yêu cầu xoá dữ liệu; và **việc xử lý dữ liệu cá nhân của trẻ em có yêu cầu chặt chẽ hơn, cần cả sự đồng ý của cha mẹ hoặc người giám hộ.**

Áp dụng thực tế:

- Nếu bất kỳ ai trong ảnh còn dưới 16 tuổi tại thời điểm đăng, cần sự đồng ý của người giám hộ — không chỉ của người gửi ảnh.
- Nên có một buổi thông báo với cả lớp trước khi công bố, thay vì đăng rồi chờ ai phản đối thì gỡ.
- Cân nhắc nghiêm túc phương án **chỉ ai có đường dẫn mới xem được**, hoặc thêm mật khẩu chung cho cả lớp. Với một cuốn kỷ yếu, việc công khai cho toàn internet gần như không mang lại lợi ích gì, trong khi rủi ro thì có thật.
- **Không làm tính năng nhận diện khuôn mặt để tự gắn thẻ tên.** Dữ liệu sinh trắc thuộc nhóm dữ liệu nhạy cảm với yêu cầu chặt chẽ hơn nhiều, và lợi ích không xứng với rủi ro.

### 8.4 Quy trình xử lý yêu cầu gỡ nội dung

```
1. Tiếp nhận qua biểu mẫu → cấp mã tra cứu → cảnh báo quản trị viên
2. Xác minh sơ bộ (không cần chứng minh nhân thân — với một trang kỷ yếu lớp,
   rào cản xác minh gây hại nhiều hơn lợi; nguyên tắc là gỡ trước, hỏi sau)
3. Gỡ nội dung:
   a. Đổi trạng thái sang HIDDEN
   b. Xoá mọi biến thể khỏi kho công bố
   c. XOÁ ĐỆM CLOUDFLARE  ← bước hay bị quên, và thiếu nó thì việc gỡ chưa hoàn tất
   d. Ghi nhật ký kiểm toán
4. Phản hồi người yêu cầu nếu họ để lại thông tin liên hệ
5. Toàn bộ hoàn tất trong 48 giờ
```

---

## 9. Sao lưu & Phục hồi

Theo nguyên tắc định hướng ở BRD, đây là hạng mục bảo mật quan trọng nhất — mối đe doạ có xác suất cao nhất là mất dữ liệu do lỗi vận hành, không phải bị tấn công.

| Hạng mục | Yêu cầu |
|---|---|
| Cơ sở dữ liệu | `pg_dump` hằng đêm, nén, **mã hoá**, đẩy sang nhà cung cấp khác |
| Ảnh | Đồng bộ sang kho thứ hai ở nhà cung cấp khác, hằng tuần |
| Lưu giữ | 14 bản gần nhất |
| Tách biệt | **Sao lưu phải nằm ở nhà cung cấp khác với nơi chạy ứng dụng.** Sao lưu cùng chỗ với bản gốc không phải sao lưu |
| Chống mã hoá tống tiền | Bật khoá ghi một lần / phiên bản đối tượng ở kho sao lưu, để kẻ chiếm được máy chủ không xoá được bản sao lưu |
| Giám sát | Sao lưu thất bại phải **cảnh báo ngay**. Sao lưu hỏng âm thầm là tình huống tệ nhất |
| **Diễn tập phục hồi** | Mỗi quý một lần: phục hồi vào môi trường tạm, đối chiếu số bản ghi, ghi biên bản |
| RPO / RTO | Mất tối đa 24 giờ dữ liệu; phục hồi trong tối đa 4 giờ |

**Một bản sao lưu chưa từng được phục hồi thử thì chưa phải là một bản sao lưu** — nó chỉ là một tệp mà bạn hy vọng là dùng được.

---

## 10. Ứng phó sự cố

### 10.1 Nội dung xấu lọt lên trang

```
1. Ẩn ngay (chưa cần điều tra) → HIDDEN + xoá đệm CDN
2. Xem nhật ký kiểm toán và log: vào lúc nào, từ bản băm IP nào
3. Chặn bản băm IP đó; siết giới hạn tần suất nếu cần
4. Kiểm tra còn nội dung nào cùng nguồn đang chờ duyệt không
5. Ghi lại: vì sao lọt qua được, và sửa gì để lần sau không lặp lại
```

### 10.2 Nghi ngờ tài khoản quản trị bị chiếm

```
1. Vô hiệu hoá TẤT CẢ phiên (xoá toàn bộ khoá phiên trong Redis)
2. Đổi mật khẩu, tạo lại bí mật TOTP, tạo lại mã dự phòng
3. Đọc nhật ký kiểm toán tìm thao tác bất thường
4. Khôi phục nội dung bị đổi hoặc bị xoá (xoá mềm 30 ngày là cứu cánh ở đây)
5. Xoay vòng toàn bộ bí mật hệ thống
6. Rà lại đường vào: mật khẩu bị đoán? máy cá nhân nhiễm mã độc? bí mật lỡ commit?
```

### 10.3 Nghi ngờ máy chủ bị chiếm

```
1. Cô lập: chặn lưu lượng ngoài qua Cloudflare, giữ máy chủ chạy để còn dấu vết
2. Chụp ảnh trạng thái (log, danh sách tiến trình, kết nối mạng) TRƯỚC khi sửa gì
3. Xoay vòng mọi bí mật — giả định tất cả đã lộ
4. Dựng lại máy chủ từ đầu, khôi phục dữ liệu từ bản sao lưu sạch gần nhất
   (KHÔNG "dọn dẹp" máy đã bị chiếm — không bao giờ chắc chắn đã sạch)
5. Thông báo cho lớp nếu có khả năng dữ liệu cá nhân bị lộ
```

### 10.4 Mất dữ liệu

```
1. DỪNG mọi thao tác ghi ngay lập tức — thao tác tiếp theo có thể ghi đè thứ còn cứu được
2. Xác định phạm vi: mất từ mốc thời gian nào
3. Phục hồi vào môi trường TẠM trước, đối chiếu, rồi mới chuyển qua môi trường thật
4. Ghi biên bản, và bổ sung biện pháp để lần sau không tái diễn
```

---

## 11. Checklist trước khi công bố

Không công bố cho tới khi mọi mục dưới đây đã đánh dấu.

### Xác thực & Phân quyền
- [ ] `.anyRequest().denyAll()` đang có trong cấu hình bảo mật
- [ ] Có kiểm thử tự động khẳng định `/api/v1/admin/**` trả `401` khi chưa đăng nhập
- [ ] 2FA đã bật cho mọi tài khoản quản trị; mã dự phòng đã sinh và cất ở nơi an toàn
- [ ] Không còn tài khoản mặc định hoặc tài khoản thử nghiệm nào
- [ ] Mật khẩu quản trị dài ≥ 12 ký tự, không trùng với mật khẩu dùng ở nơi khác

### Tải tệp lên
- [ ] Xác định loại tệp bằng magic bytes, đã kiểm thử với tệp SVG đội lốt `.jpg`
- [ ] SVG bị từ chối
- [ ] Đã kiểm thử với tệp bom nén
- [ ] Đã kiểm chứng ảnh đầu ra **không còn EXIF/GPS** (mở bằng công cụ xem siêu dữ liệu để xác nhận)
- [ ] Ảnh dọc chụp bằng điện thoại hiển thị đúng chiều sau khi xử lý
- [ ] ClamAV hoạt động, đã kiểm thử bằng tệp thử EICAR
- [ ] Ảnh phục vụ từ tên miền riêng
- [ ] Đường dẫn ký sẵn hạn 10 phút, chỉ cho `PUT`, chỉ đúng một khoá
- [ ] Bucket `quarantine/` **không** có quyền đọc công khai (đã thử mở bằng trình duyệt và nhận `403`)
- [ ] Bucket `archive/` **không** có quyền đọc công khai — đây là nơi bytes gốc còn nguyên GPS
- [ ] Quy tắc vòng đời tự xoá `quarantine/` sau 24 giờ đã bật và đã kiểm chứng
- [ ] Hạn ngạch dung lượng theo IP hoạt động (thử tải vượt 500 MB trong ngày)
- [ ] Đã thử với **ảnh thật 25 MB / 50 megapixel** — không hết bộ nhớ, không quá thời gian chờ
- [ ] Đã thử với ảnh HEIC thật từ iPhone

### Cấu hình
- [ ] `ddl-auto: validate`, không phải `update`
- [ ] `open-in-view: false`
- [ ] `include-stacktrace: never` — đã thử gây lỗi và xác nhận phản hồi không lộ chi tiết
- [ ] Swagger UI đã tắt ở môi trường thật
- [ ] Điểm cuối Actuator không truy cập được từ internet
- [ ] Cổng PostgreSQL và Redis không phơi ra internet (kiểm bằng cách quét cổng từ máy ngoài)
- [ ] CORS chỉ cho phép tên miền cụ thể

### Header & TLS
- [ ] Kiểm tra bằng công cụ đánh giá TLS, đạt hạng A
- [ ] CSP đã đặt và **không có `'unsafe-inline'` trong `script-src`**
- [ ] Đủ bộ header ở mục 6
- [ ] HTTP chuyển hướng sang HTTPS

### Quyền riêng tư
- [ ] `robots.txt` chặn toàn bộ trình thu thập
- [ ] Header `X-Robots-Tag: noindex, noimageindex` có ở mọi phản hồi
- [ ] Trang "Quyền riêng tư" đã viết và có liên kết ở chân trang
- [ ] Ô đồng thuận có trong biểu mẫu đóng góp và là bắt buộc
- [ ] Kênh yêu cầu gỡ hoạt động, đã thử gửi thật một lần
- [ ] **Đã thông báo và xin ý kiến cả lớp trước khi công bố**
- [ ] Đã cân nhắc và quyết định rõ: công khai hoàn toàn hay chỉ ai có đường dẫn

### Chống lạm dụng
- [ ] Giới hạn tần suất hoạt động, đã kiểm thử bằng cách gửi vượt ngưỡng
- [ ] Turnstile hoạt động và được xác minh ở phía máy chủ
- [ ] IP chỉ lưu ở dạng băm — đã kiểm tra trong cơ sở dữ liệu để xác nhận
- [ ] Header IP chuyển tiếp chỉ được tin khi đến từ Cloudflare

### Hạ tầng
- [ ] SSH chỉ đăng nhập bằng khoá, đã tắt đăng nhập mật khẩu
- [ ] Tường lửa chỉ mở 22, 80, 443
- [ ] Container chạy bằng người dùng thường
- [ ] `unattended-upgrades` đã bật
- [ ] Không có bí mật nào trong kho mã (đã chạy `gitleaks` trên toàn bộ lịch sử)

### Sao lưu
- [ ] Sao lưu tự động chạy hằng đêm và đã chạy thành công ít nhất 7 đêm liên tiếp
- [ ] Sao lưu nằm ở nhà cung cấp khác với nơi chạy ứng dụng
- [ ] Sao lưu được mã hoá
- [ ] **Đã phục hồi thử thành công ít nhất một lần và ghi biên bản**
- [ ] Sao lưu thất bại thì có cảnh báo

### Giám sát
- [ ] Cảnh báo hoạt động, đã thử gây lỗi để xác nhận cảnh báo có tới
- [ ] Log không chứa mật khẩu, token hay IP thô — đã tự đọc kiểm tra
- [ ] Nhật ký kiểm toán ghi được và **không sửa/xoá được** (đã thử `DELETE` và xác nhận bị từ chối)

---

## 12. Bảo trì định kỳ

| Tần suất | Việc |
|---|---|
| **Hằng ngày** (giai đoạn đầu) | Duyệt hàng chờ; liếc qua cảnh báo |
| **Hằng tuần** | Xem lại các pull request cập nhật phụ thuộc; kiểm tra sao lưu đã chạy |
| **Hằng tháng** | Đọc nhật ký kiểm toán tìm bất thường; xem báo cáo quét lỗ hổng; kiểm tra dung lượng đĩa |
| **Hằng quý** | **Diễn tập phục hồi từ sao lưu**; rà lại checklist mục 11; kiểm tra hạn chứng chỉ |
| **Hằng năm** | Xoay vòng toàn bộ bí mật; nâng cấp phiên bản chính của Java/Spring Boot; rà lại mô hình đe doạ |

---

## 13. Nếu chỉ làm được một phần — thứ tự ưu tiên

Trong trường hợp thời gian không cho phép làm hết, đây là thứ tự bỏ dần từ dưới lên. **Sáu mục đầu là mức tối thiểu tuyệt đối — dưới mức này thì không nên công bố trang.**

1. **HTTPS + sao lưu tự động có kiểm chứng phục hồi** — không có hai thứ này thì mọi thứ khác vô nghĩa
2. **Kiểm duyệt trước khi hiển thị** — lớp bảo vệ chính cho uy tín và cho người trong ảnh
3. **Kiểm tra tệp tải lên: magic bytes + từ chối SVG + mã hoá lại** — bề mặt tấn công lớn nhất
4. **Mặc định từ chối + kiểm quyền ở tầng service** — hạng mục số một của OWASP
5. **Băm mật khẩu đúng cách + 2FA cho quản trị** — tài khoản duy nhất, giá trị cao nhất
6. **Giới hạn tần suất** — không có thì bot sẽ làm ngập trang trong tuần đầu
7. Bộ header bảo mật đầy đủ + CSP nghiêm ngặt
8. Nhật ký kiểm toán
9. Quét lỗ hổng tự động trong CI
10. ClamAV
11. Bảng giám sát và cảnh báo đầy đủ

---

*Tài liệu này chi tiết hoá các yêu cầu bảo mật ở [`MemoryBook-SRS.md`](MemoryBook-SRS.md) mục 4.3 và 4.4. Mã hiện thực ở [`MemoryBook-LLD.md`](MemoryBook-LLD.md) mục 6; công cụ ở [`MemoryBook-TechStack.md`](MemoryBook-TechStack.md) mục 6.*
