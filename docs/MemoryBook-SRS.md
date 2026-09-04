# SRS — Software Requirements Specification
## Phượng Hồng Memories — Kỷ yếu điện tử lớp 12A1

---

## 1. Giới thiệu

Tài liệu này chi tiết hoá các yêu cầu đã nêu ở [`MemoryBook-PRD.md`](MemoryBook-PRD.md) thành đặc tả mà phần mềm phải đáp ứng: từng chức năng cụ thể theo module, kèm các yêu cầu phi chức năng có ngưỡng đo được. Đây là tài liệu để đối chiếu khi nghiệm thu — mỗi dòng ở đây phải kiểm chứng được bằng một phép thử.

Quy ước mã: `FR-<MODULE>-<số>` cho yêu cầu chức năng, `NFR-<NHÓM>-<số>` cho yêu cầu phi chức năng.

## 2. Phạm vi hệ thống

Hệ thống gồm ba phần:

1. **Ứng dụng web công khai** — trang kỷ yếu cho khách xem và thành viên lớp; không đăng nhập.
2. **Khu quản trị** — giao diện kiểm duyệt cho ban tổ chức lớp; có đăng nhập và 2FA.
3. **Dịch vụ API** — Spring Boot, là nơi duy nhất giữ logic nghiệp vụ và là ranh giới tin cậy của hệ thống.

Mọi kiểm tra hợp lệ và mọi quyết định về quyền đều nằm ở phần 3. Front-end được coi là **môi trường không đáng tin**: nó chỉ làm cho trải nghiệm dễ chịu, không phải nơi thực thi quy tắc.

## 3. Yêu cầu chức năng (Functional Requirements)

### 3.1 Module Lời nhắn (MSG)

| Mã | Yêu cầu |
|---|---|
| FR-MSG-01 | Hệ thống cung cấp danh sách lời nhắn ở trạng thái `PUBLISHED`, sắp xếp theo thời gian tạo giảm dần, phân trang theo con trỏ (cursor), mặc định 20 mục, tối đa 50 mục mỗi lần |
| FR-MSG-02 | Khách gửi được lời nhắn gồm: `fullname` (tuỳ chọn, ≤ 60 ký tự), `nickname` (tuỳ chọn, ≤ 60 ký tự), `message` (bắt buộc, 1–500 ký tự sau khi cắt khoảng trắng thừa) |
| FR-MSG-03 | Tên hiển thị được suy ra theo thứ tự `nickname` → `fullname` → `"Ẩn danh"`; hệ thống không lưu tên hiển thị riêng mà tính lúc trả về |
| FR-MSG-04 | Lời nhắn mới luôn được tạo với trạng thái `PENDING`; API trả `202 Accepted` kèm định danh, **không** trả nội dung như thể đã công bố |
| FR-MSG-05 | Hệ thống chuẩn hoá Unicode về dạng NFC và loại bỏ ký tự điều khiển, ký tự vô hình (zero-width) trước khi lưu |
| FR-MSG-06 | Hệ thống từ chối lời nhắn chứa quá 3 URL, hoặc trùng khớp hoàn toàn với một lời nhắn đã gửi từ cùng IP trong 24 giờ qua |
| FR-MSG-07 | Quản trị viên chuyển được lời nhắn giữa các trạng thái `PENDING` → `PUBLISHED` / `REJECTED` / `HIDDEN`, và xoá mềm |
| FR-MSG-08 | Trạng thái hợp lệ của lời nhắn: `PENDING`, `PUBLISHED`, `REJECTED`, `HIDDEN`, `DELETED` (xoá mềm) |

### 3.2 Module Đóng góp ảnh (UPL)

| Mã | Yêu cầu |
|---|---|
| FR-UPL-01 | Khách tải lên được một ảnh mỗi lần gửi, kèm các trường tuỳ chọn `note` (≤ 300 ký tự), `takenAt` (văn bản tự do ≤ 50 ký tự), `category`. Tệp được tải **trực tiếp lên kho đối tượng bằng đường dẫn ký sẵn**, không đi qua máy chủ ứng dụng — xem [`MemoryBook-Storage-Media.md`](MemoryBook-Storage-Media.md) mục 4 |
| FR-UPL-02 | `category` chỉ nhận một trong bốn giá trị: `GRADE_10`, `GRADE_11`, `GRADE_12`, `EVENT`; giá trị khác bị từ chối với `400` |
| FR-UPL-03 | Kích thước tệp tối đa **25 MB**; vượt ngưỡng bị từ chối ngay ở bước xin đường dẫn ký sẵn, và kiểm lại bằng `HEAD` sau khi tải xong |
| FR-UPL-04 | Hệ thống xác định loại tệp bằng **magic bytes** đọc từ nội dung, chỉ chấp nhận JPEG, PNG, WebP, HEIC/HEIF; header `Content-Type` do client gửi bị bỏ qua hoàn toàn |
| FR-UPL-05 | Hệ thống **từ chối tệp SVG** kể cả khi hợp lệ về cú pháp — SVG là tài liệu thực thi được và là vector XSS |
| FR-UPL-06 | Hệ thống chặn ảnh có kích thước điểm ảnh bất thường: tổng số điểm ảnh > 80 megapixel, hoặc tỉ lệ nén giải ra > 100 lần (chống decompression bomb) |
| FR-UPL-07 | Ảnh gốc giữ nguyên bytes trong vùng `archive/` **riêng tư tuyệt đối, không bao giờ phục vụ ra ngoài**. Mọi bản công bố được **giải mã và mã hoá lại**, không giữ lại bất kỳ khối siêu dữ liệu nào (EXIF, XMP, GPS), và phải được xoay đúng chiều theo EXIF Orientation **trước khi** bỏ siêu dữ liệu |
| FR-UPL-08 | Hệ thống sinh ba kích cỡ công bố: `thumb` (400 px), `medium` (1600 px), `large` (4000 px), mỗi kích cỡ có cả **AVIF và JPEG** (JPEG là bản dự phòng cho trình duyệt cũ) |
| FR-UPL-09 | Tệp tải lên được quét mã độc trước khi ghi vào kho lưu trữ công bố; tệp nghi ngờ bị cách ly và không bao giờ hiển thị |
| FR-UPL-10 | Tên tệp gốc chỉ dùng để hiển thị; tên tệp thật khi lưu là định danh do hệ thống sinh, không lấy từ đầu vào của người dùng |
| FR-UPL-11 | Ảnh tải lên mặc định ở trạng thái `PENDING`; API trả `202 Accepted` |
| FR-UPL-12 | Hệ thống tính và lưu mã băm nội dung ảnh; ảnh trùng băm với ảnh đã có sẽ bị từ chối kèm thông báo "ảnh này đã có trong thư viện" |
| FR-UPL-13 | Trong lúc tải lên, front-end hiển thị tiến trình; nếu kết nối đứt, người dùng gửi lại được mà không mất dữ liệu đã nhập trong biểu mẫu |

### 3.3 Module Thư viện ảnh (GAL)

| Mã | Yêu cầu |
|---|---|
| FR-GAL-01 | Hệ thống cung cấp danh sách ảnh `PUBLISHED`, lọc được theo `category`, phân trang theo con trỏ |
| FR-GAL-02 | Mỗi ảnh trả về đủ đường dẫn các phiên bản, kích thước thật (rộng × cao) và tỉ lệ khung — để front-end đặt sẵn chỗ, tránh giật bố cục khi ảnh tải xong |
| FR-GAL-03 | Đường dẫn ảnh trỏ tới tên miền phục vụ tệp tĩnh riêng, tách khỏi tên miền ứng dụng |
| FR-GAL-04 | Hệ thống cung cấp điểm cuối tải toàn bộ ảnh đã duyệt dưới dạng tệp nén, truyền theo luồng (không dựng toàn bộ trong bộ nhớ) |
| FR-GAL-05 | Tệp nén được dựng sẵn theo lịch và lưu tạm; yêu cầu tải trong vòng 24 giờ dùng lại bản đã dựng thay vì dựng lại |
| FR-GAL-06 | Điểm cuối tải album có giới hạn tần suất riêng, chặt hơn các điểm cuối đọc thông thường |
| FR-GAL-07 | Khi một ảnh chuyển khỏi `PUBLISHED`, bộ nhớ đệm CDN cho ảnh đó phải bị vô hiệu trong vòng 60 giây |

### 3.4 Module Xác thực & Quản trị (ADM)

| Mã | Yêu cầu |
|---|---|
| FR-ADM-01 | Quản trị viên đăng nhập bằng tên đăng nhập + mật khẩu + mã TOTP 6 số; thiếu bất kỳ yếu tố nào đều bị từ chối |
| FR-ADM-02 | Mật khẩu được lưu dưới dạng băm bằng thuật toán hiện đại có kiểm soát chi phí (Argon2id hoặc bcrypt); không bao giờ lưu bản rõ, không bao giờ ghi vào log |
| FR-ADM-03 | Sai mật khẩu 5 lần liên tiếp trong 15 phút thì khoá tài khoản 15 phút; mọi lần sai đều được ghi nhật ký |
| FR-ADM-04 | Phiên đăng nhập hết hạn sau 8 giờ không hoạt động và tối đa 24 giờ kể cả khi vẫn hoạt động |
| FR-ADM-05 | Định danh phiên được cấp mới sau khi đăng nhập thành công (chống cố định phiên) |
| FR-ADM-06 | Đăng xuất huỷ phiên ở phía máy chủ, không chỉ xoá cookie ở trình duyệt |
| FR-ADM-07 | Quản trị viên xem được hàng chờ gộp cả ảnh và lời nhắn, sắp theo thời gian gửi tăng dần, lọc được theo loại và trạng thái |
| FR-ADM-08 | Quản trị viên thực hiện được: duyệt, từ chối, ẩn, xoá mềm, khôi phục — trên từng mục hoặc theo lô |
| FR-ADM-09 | Xoá mềm giữ dữ liệu 30 ngày rồi mới xoá vĩnh viễn bằng tác vụ nền; trong 30 ngày đó khôi phục được |
| FR-ADM-10 | Quản trị viên sửa được `note`, `takenAt`, `category` của ảnh và sửa lỗi chính tả trong nội dung lời nhắn trước khi duyệt |
| FR-ADM-11 | Mọi thao tác thay đổi dữ liệu của quản trị viên ghi vào nhật ký kiểm toán: ai, lúc nào, đối tượng nào, giá trị trước và sau |
| FR-ADM-12 | Nhật ký kiểm toán chỉ được thêm mới, không có API sửa hoặc xoá; giữ tối thiểu 24 tháng |
| FR-ADM-13 | Toàn bộ đường dẫn quản trị bị từ chối theo mặc định với mọi yêu cầu không có phiên hợp lệ và không có vai trò `ADMIN` |

### 3.5 Module Chống lạm dụng (ABU)

| Mã | Yêu cầu |
|---|---|
| FR-ABU-01 | Giới hạn tần suất gửi lời nhắn: 5 lượt / IP / giờ và 30 lượt / IP / ngày |
| FR-ABU-02 | Giới hạn tần suất tải ảnh: 10 lượt / IP / giờ và 50 lượt / IP / ngày, **cộng thêm hạn ngạch dung lượng 500 MB / IP / ngày** — đếm theo số lượt là chưa đủ khi mỗi lượt có thể tới 25 MB |
| FR-ABU-02b | Kho đối tượng phải có quy tắc vòng đời tự xoá mọi đối tượng trong `quarantine/` sau 24 giờ, để tệp tải lên dở dang không tích tụ và kho không bị dùng làm nơi chứa tệp |
| FR-ABU-03 | Giới hạn tần suất đăng nhập quản trị: 10 lượt / IP / 15 phút |
| FR-ABU-04 | Giới hạn tần suất các điểm cuối đọc: 120 lượt / IP / phút |
| FR-ABU-05 | Vượt giới hạn trả `429 Too Many Requests` kèm header `Retry-After` |
| FR-ABU-06 | Bộ đếm giới hạn tần suất lưu ở kho dùng chung để vẫn đúng khi có nhiều tiến trình ứng dụng |
| FR-ABU-07 | Địa chỉ IP không được lưu ở dạng thô: chỉ lưu HMAC-SHA256 của IP với khoá bí mật của máy chủ |
| FR-ABU-08 | Điểm cuối ghi dữ liệu công khai yêu cầu token Cloudflare Turnstile hợp lệ, xác minh phía máy chủ |
| FR-ABU-09 | Hệ thống nhận diện IP thật qua header do proxy tin cậy đặt, và **chỉ tin header đó khi yêu cầu đến từ dải IP của proxy đã cấu hình** |

### 3.6 Module Quyền riêng tư & Gỡ nội dung (PRV)

| Mã | Yêu cầu |
|---|---|
| FR-PRV-01 | Khách gửi được yêu cầu gỡ nội dung gồm: định danh nội dung, lý do, thông tin liên hệ tuỳ chọn |
| FR-PRV-02 | Yêu cầu gỡ vào hàng chờ ưu tiên cao, hiển thị tách biệt trong khu quản trị |
| FR-PRV-03 | Hệ thống trả mã tra cứu để người gửi kiểm tra tình trạng mà không cần tài khoản |
| FR-PRV-04 | Khi nội dung bị gỡ, hệ thống xoá cả bản gốc lẫn mọi phiên bản dẫn xuất khỏi kho lưu trữ và vô hiệu bộ nhớ đệm CDN |
| FR-PRV-05 | Trang "Quyền riêng tư" nêu rõ: loại dữ liệu thu thập, mục đích, thời hạn lưu, cách yêu cầu gỡ, thông tin liên hệ |
| FR-PRV-06 | `robots.txt` chặn toàn bộ trình thu thập; các trang trả header `X-Robots-Tag: noindex, noimageindex` |
| FR-PRV-07 | Biểu mẫu đóng góp có ô xác nhận đồng thuận: người gửi khẳng định có quyền chia sẻ ảnh và những người trong ảnh đồng ý |

### 3.7 Module Nội dung tĩnh & Cấu hình (CFG)

| Mã | Yêu cầu |
|---|---|
| FR-CFG-01 | Danh sách học sinh và thầy cô đọc từ dữ liệu cấu hình, không viết cứng trong mã front-end |
| FR-CFG-02 | Các mốc trong mục "Hành trình" đọc từ dữ liệu cấu hình |
| FR-CFG-03 | Số liệu thống kê ở trang chủ (số ảnh, số lời nhắn, số thành viên) tính từ dữ liệu thật, có bộ nhớ đệm 5 phút |
| FR-CFG-04 | Sửa dữ liệu cấu hình không cần triển khai lại front-end |

### 3.8 Module Vận hành (OPS)

| Mã | Yêu cầu |
|---|---|
| FR-OPS-01 | Hệ thống có điểm cuối kiểm tra sức khoẻ phân biệt được "sống" (tiến trình còn chạy) và "sẵn sàng" (đã kết nối được cơ sở dữ liệu và kho lưu trữ) |
| FR-OPS-02 | Điểm cuối số liệu giám sát chỉ truy cập được từ mạng nội bộ, không phơi ra internet |
| FR-OPS-03 | Sao lưu cơ sở dữ liệu tự động hằng đêm sang nơi lưu trữ khác máy chủ, giữ 14 bản gần nhất |
| FR-OPS-04 | Ảnh được đồng bộ sang kho sao lưu thứ hai ở nhà cung cấp khác, ít nhất mỗi tuần |
| FR-OPS-05 | Log ghi ở dạng có cấu trúc, mỗi dòng gắn định danh yêu cầu để tra ngược toàn bộ một lượt gọi |
| FR-OPS-06 | Log không được chứa mật khẩu, token, cookie phiên, hay địa chỉ IP thô |
| FR-OPS-07 | Di trú cơ sở dữ liệu chạy tự động khi khởi động ứng dụng và có phiên bản rõ ràng |

---

## 4. Yêu cầu phi chức năng (Non-functional Requirements)

### 4.1 Hiệu năng (Performance)

| Mã | Yêu cầu | Chỉ số mục tiêu | Cách đo |
|---|---|---|---|
| NFR-PERF-01 | Thời gian phản hồi API danh sách lời nhắn | < 300 ms (p95) | k6 / Gatling trên môi trường staging |
| NFR-PERF-02 | Thời gian phản hồi API danh sách ảnh | < 300 ms (p95) | k6 / Gatling |
| NFR-PERF-03 | Thời gian xử lý một ảnh tải lên (quét + sinh 6 biến thể) | < 15 giây cho ảnh 25 MB / 50 megapixel, dùng libvips | Đo thời gian tác vụ nền |
| NFR-PERF-03b | Bộ nhớ tiêu thụ khi xử lý một ảnh | < 200 MB bất kể kích thước ảnh đầu vào — đạt được nhờ libvips xử lý theo luồng | Đo bằng `docker stats` khi xử lý ảnh 50 MP |
| NFR-PERF-04 | Largest Contentful Paint của trang công khai trên 4G, thiết bị tầm trung | < 2,5 giây | Lighthouse, WebPageTest |
| NFR-PERF-05 | Cumulative Layout Shift | < 0,1 | Lighthouse — đạt được nhờ FR-GAL-02 (biết trước tỉ lệ khung ảnh) |
| NFR-PERF-06 | Kích thước gói JavaScript ban đầu (đã nén) | < 200 KB | Báo cáo phân tích gói khi build |
| NFR-PERF-07 | Thời gian dựng tệp nén album cho 500 ảnh | < 60 giây, chạy nền, không chặn tiến trình phục vụ | Đo tác vụ nền |
| NFR-PERF-08 | Ứng dụng chịu được 100 người xem đồng thời | Tỉ lệ lỗi < 0,5%, p95 < 500 ms | Kịch bản k6 mô phỏng lượt xem cao điểm |

### 4.2 Độ tin cậy & Khả dụng (Reliability & Availability)

| Mã | Yêu cầu |
|---|---|
| NFR-REL-01 | Uptime mục tiêu ≥ 99% tính theo tháng |
| NFR-REL-02 | API không phản hồi thì front-end vẫn hiển thị phần tĩnh kèm thông báo, không để trang trắng |
| NFR-REL-03 | Không được mất dữ liệu đã ghi nhận thành công: mục tiêu RPO ≤ 24 giờ, RTO ≤ 4 giờ |
| NFR-REL-04 | Phải diễn tập phục hồi từ bản sao lưu ít nhất mỗi quý và ghi lại kết quả |
| NFR-REL-05 | Tải ảnh lên không được để lại trạng thái dở dang: hoặc bản ghi và tệp cùng tồn tại, hoặc cả hai đều không |
| NFR-REL-06 | Tác vụ nền phải thử lại được và **bất biến với việc chạy lại** (chạy hai lần cho cùng kết quả) |
| NFR-REL-07 | Ứng dụng khởi động lại được mà không mất phiên đăng nhập của quản trị viên (phiên lưu ngoài tiến trình) |

### 4.3 Bảo mật (Security)

> Chi tiết cách hiện thực nằm ở [`MemoryBook-Security.md`](MemoryBook-Security.md). Mục này chỉ nêu yêu cầu cần nghiệm thu.

| Mã | Yêu cầu |
|---|---|
| NFR-SEC-01 | Toàn bộ giao tiếp qua HTTPS, TLS 1.2 trở lên; HTTP chuyển hướng sang HTTPS; bật HSTS |
| NFR-SEC-02 | Mật khẩu quản trị băm bằng Argon2id (hoặc bcrypt) với tham số chi phí phù hợp; không lưu bản rõ |
| NFR-SEC-03 | Mọi dữ liệu vào đều được kiểm tra ở tầng máy chủ; kiểm tra ở front-end chỉ để hỗ trợ trải nghiệm |
| NFR-SEC-04 | Truy vấn cơ sở dữ liệu luôn dùng tham số ràng buộc; cấm ghép chuỗi truy vấn từ dữ liệu người dùng |
| NFR-SEC-05 | Quyền truy cập kiểm tra ở tầng máy chủ cho mọi yêu cầu; ẩn nút ở giao diện không được tính là biện pháp bảo vệ |
| NFR-SEC-06 | Mặc định từ chối: đường dẫn nào chưa được khai báo công khai thì phải yêu cầu xác thực |
| NFR-SEC-07 | Xác thực dùng cookie phiên với `HttpOnly`, `Secure`, `SameSite=Lax`; nếu dùng token thì tuyệt đối không lưu ở `localStorage` |
| NFR-SEC-08 | Mọi yêu cầu ghi dữ liệu dùng cookie đều có bảo vệ CSRF |
| NFR-SEC-09 | Tài khoản quản trị bắt buộc 2FA (TOTP), có mã dự phòng dùng một lần |
| NFR-SEC-10 | Tệp người dùng tải lên phải: giới hạn dung lượng, xác định loại bằng magic bytes, mã hoá lại, quét mã độc, và phục vụ từ tên miền tách biệt |
| NFR-SEC-11 | Trang gửi đủ bộ header bảo mật: `Content-Security-Policy`, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy`, `frame-ancestors` |
| NFR-SEC-12 | CORS chỉ cho phép danh sách nguồn đã khai báo; cấm dùng `*` khi có kèm chứng danh |
| NFR-SEC-13 | Thông điệp lỗi trả cho client không được lộ dấu vết ngăn xếp, tên lớp, phiên bản thư viện hay câu truy vấn |
| NFR-SEC-14 | Bí mật (mật khẩu cơ sở dữ liệu, khoá kho lưu trữ, khoá HMAC) đọc từ biến môi trường hoặc kho bí mật; tuyệt đối không nằm trong kho mã |
| NFR-SEC-15 | Phụ thuộc được quét lỗ hổng tự động trong CI; lỗ hổng mức nghiêm trọng làm dừng bản dựng |
| NFR-SEC-16 | Ảnh bìa giao diện quản trị và điểm cuối số liệu giám sát không được lộ ra internet công khai |
| NFR-SEC-17 | Không tải mã từ nguồn ngoài trong bản triển khai thật; thư viện đóng gói cùng ứng dụng thay vì gọi CDN của bên thứ ba |

### 4.4 Quyền riêng tư & Tuân thủ (Privacy & Compliance)

| Mã | Yêu cầu |
|---|---|
| NFR-PRV-01 | Chỉ thu thập dữ liệu thật sự cần: tên hiển thị, nội dung, ảnh, và bản băm IP để chống lạm dụng |
| NFR-PRV-02 | Không dùng công cụ phân tích của bên thứ ba đặt cookie theo dõi; nếu cần đo lượt xem, dùng công cụ không dùng cookie |
| NFR-PRV-03 | Bản băm IP tự động xoá sau 90 ngày |
| NFR-PRV-04 | Có quy trình tiếp nhận và xử lý yêu cầu gỡ nội dung trong 48 giờ |
| NFR-PRV-05 | Toàn bộ trang chặn lập chỉ mục bởi máy tìm kiếm |
| NFR-PRV-06 | Có văn bản đồng thuận cho việc đăng ảnh; với người trong ảnh còn ở tuổi vị thành niên tại thời điểm đăng, cần đồng thuận của người giám hộ |

### 4.5 Khả năng tiếp cận & Trải nghiệm (Accessibility & UX)

| Mã | Yêu cầu |
|---|---|
| NFR-A11Y-01 | Tương phản màu đạt WCAG 2.1 mức AA cho toàn bộ chữ nội dung |
| NFR-A11Y-02 | Khung xem ảnh, thẻ lật và biểu mẫu điều khiển được hoàn toàn bằng bàn phím, có vòng lặp tiêu điểm đúng |
| NFR-A11Y-03 | Mọi hoạt ảnh tắt khi hệ điều hành bật `prefers-reduced-motion: reduce` |
| NFR-A11Y-04 | Ảnh nội dung có mô tả thay thế; ảnh và SVG trang trí bị ẩn khỏi trình đọc màn hình |
| NFR-A11Y-05 | Biểu mẫu có nhãn gắn đúng ô nhập; lỗi được thông báo cho trình đọc màn hình chứ không chỉ đổi màu viền |
| NFR-A11Y-06 | Giao diện dùng được ở bề rộng 320 px trở lên, không cuộn ngang |

### 4.6 Khả năng bảo trì & Mở rộng (Maintainability & Scalability)

| Mã | Yêu cầu |
|---|---|
| NFR-MAIN-01 | API có tài liệu OpenAPI sinh tự động từ mã, luôn khớp với hiện thực |
| NFR-MAIN-02 | Thay đổi lược đồ cơ sở dữ liệu chỉ qua tệp di trú có phiên bản; cấm sửa tay trên môi trường thật |
| NFR-MAIN-03 | Độ phủ kiểm thử ≥ 70% cho tầng nghiệp vụ; các luồng lõi (tải ảnh, kiểm duyệt, xác thực) phải có kiểm thử tích hợp chạy trên cơ sở dữ liệu thật trong container |
| NFR-MAIN-04 | Tầng ứng dụng không giữ trạng thái, cho phép chạy nhiều bản song song khi cần |
| NFR-MAIN-05 | Front-end chỉ phụ thuộc vào hợp đồng API; đổi framework front-end không kéo theo sửa back-end |
| NFR-MAIN-06 | API có phiên bản trong đường dẫn (`/api/v1/...`) để thay đổi phá vỡ tương thích không làm hỏng client cũ |

### 4.7 Khả năng quan sát (Observability)

| Mã | Yêu cầu |
|---|---|
| NFR-OBS-01 | Ứng dụng phát số liệu về lưu lượng, độ trễ, tỉ lệ lỗi, và trạng thái bể kết nối cơ sở dữ liệu |
| NFR-OBS-02 | Log tập trung, tra cứu được theo định danh yêu cầu |
| NFR-OBS-03 | Có cảnh báo khi: tỉ lệ lỗi 5xx > 1% trong 5 phút, đĩa còn dưới 15%, sao lưu đêm thất bại, chứng chỉ TLS còn dưới 14 ngày |
| NFR-OBS-04 | Có bảng theo dõi hàng chờ kiểm duyệt để quản trị viên biết còn tồn bao nhiêu mục |

---

## 5. Ràng buộc kỹ thuật (Technical Constraints)

- Back-end chốt là **Spring Boot trên Java LTS**; không dùng ngôn ngữ khác cho dịch vụ chính.
- Front-end định hướng **React**; đây là ràng buộc mềm — hợp đồng API phải giữ nguyên nếu đổi công nghệ.
- Triển khai trên **một máy chủ ảo với Docker Compose**, không dùng Kubernetes ở giai đoạn này.
- Ngân sách hạ tầng mục tiêu dưới 200.000 đ/tháng.
- Một người phát triển và vận hành → mọi giải pháp phải trả lời được câu hỏi "3 giờ sáng hệ thống hỏng thì một người có sửa nổi không".
- Dữ liệu là dữ liệu cá nhân của người vị thành niên → chịu ràng buộc của Nghị định 13/2023/NĐ-CP về bảo vệ dữ liệu cá nhân.

## 6. Ma trận truy vết yêu cầu (Traceability)

| PRD Ref | SRS Ref liên quan | Nơi hiện thực |
|---|---|---|
| FR-01 (danh sách lời nhắn) | FR-MSG-01, NFR-PERF-01 | Server — `MessageController`, `MessageQueryService` |
| FR-02 (gửi lời nhắn) | FR-MSG-02 → FR-MSG-06 | Server — `MessageCommandService` |
| FR-03 (tải ảnh) | FR-UPL-01, FR-UPL-02, FR-UPL-11 | Server — `PhotoUploadController` |
| FR-04 (xác thực tệp ảnh) | FR-UPL-04, FR-UPL-05, FR-UPL-06, NFR-SEC-10 | Server — `ImageValidator` |
| FR-05 (mã hoá lại + sinh phiên bản) | FR-UPL-07, FR-UPL-08, NFR-PERF-03 | Server — `ImageProcessingService` (tác vụ nền) |
| FR-06 (danh sách ảnh) | FR-GAL-01, FR-GAL-02, FR-GAL-03 | Server — `PhotoQueryService` |
| FR-07 (tải album) | FR-GAL-04, FR-GAL-05, FR-GAL-06, NFR-PERF-07 | Server — `AlbumArchiveService` |
| FR-08 (đăng nhập quản trị) | FR-ADM-01 → FR-ADM-06, NFR-SEC-02, NFR-SEC-07, NFR-SEC-09 | Server — cấu hình Spring Security |
| FR-09 (hàng chờ kiểm duyệt) | FR-ADM-07 → FR-ADM-10 | Server + Client — khu quản trị |
| FR-10 (nhật ký kiểm toán) | FR-ADM-11, FR-ADM-12 | Server — `AuditLogAspect` |
| FR-11 (giới hạn tần suất) | FR-ABU-01 → FR-ABU-07, FR-ABU-09 | Server — bộ lọc giới hạn tần suất |
| FR-12 (chống bot) | FR-ABU-08 | Server + Client |
| FR-13 (yêu cầu gỡ nội dung) | FR-PRV-01 → FR-PRV-04, NFR-PRV-04 | Server + Client |
| FR-14 (quyền riêng tư, chặn lập chỉ mục) | FR-PRV-05, FR-PRV-06, FR-PRV-07, NFR-PRV-05 | Client + cấu hình máy chủ web |
| FR-15 (sao lưu) | FR-OPS-03, FR-OPS-04, NFR-REL-03, NFR-REL-04 | Hạ tầng — tác vụ định kỳ |
| FR-16 (kiểm tra sức khoẻ, giám sát) | FR-OPS-01, FR-OPS-02, NFR-OBS-01 → NFR-OBS-04 | Server — Actuator + Prometheus |
| FR-17 (dữ liệu cấu hình) | FR-CFG-01 → FR-CFG-04 | Server — nhóm điểm cuối cấu hình |
| FR-18 (chịu lỗi khi API chết) | NFR-REL-02 | Client |

---

*Tài liệu này dựa trên [`MemoryBook-PRD.md`](MemoryBook-PRD.md) và là cơ sở để thiết kế kiến trúc trong [`MemoryBook-HLD.md`](MemoryBook-HLD.md) và chi tiết hiện thực trong [`MemoryBook-LLD.md`](MemoryBook-LLD.md).*
