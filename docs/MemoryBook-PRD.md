# PRD — Product Requirements Document
## Phượng Hồng Memories — Kỷ yếu điện tử lớp 12A1

---

## 1. Tổng quan sản phẩm

Phượng Hồng Memories là một trang web kỷ yếu điện tử cho lớp 12A1 trường THPT Mạc Đĩnh Chi: một trang cuộn liền mạch gồm thư viện ảnh, hành trình ba năm, hồ sơ học sinh và thầy cô, bảng lời nhắn chung và biểu mẫu đóng góp ảnh, bọc trong phong cách "cuốn sổ lưu niệm số" đã mô tả ở [`../client/DESIGN.md`](../client/DESIGN.md).

Bản hiện tại (`client/index.html`) đã dựng xong phần nhìn nhưng **mọi dữ liệu đều là giả lập trong trình duyệt**. Sản phẩm mà tài liệu này đặc tả là bản có back-end thật: nội dung được lưu tập trung, có kiểm duyệt, có sao lưu và có thể chia sẻ cho cả lớp.

> **Lưu ý về phạm vi công nghệ.** Front-end hiện tại chỉ là bản dựng tạm để xem giao diện. Hướng đi là **React** cho front-end và **Spring Boot** cho back-end (chi tiết ở [`MemoryBook-TechStack.md`](MemoryBook-TechStack.md)), nhưng front-end có thể đổi công nghệ sau. Toàn bộ yêu cầu trong tài liệu này được viết ở mức **hành vi sản phẩm**, không ràng buộc vào framework.

## 2. Vấn đề cần giải quyết

| Góc nhìn người dùng | Vấn đề |
|---|---|
| Một bạn trong lớp | "Mình gõ lời nhắn xong, gửi cho bạn xem thì bạn không thấy gì cả." — lời nhắn chỉ nằm trên máy của chính người gõ |
| Một bạn có ảnh đẹp | "Mình chọn ảnh, bấm gửi, hiện thông báo thành công, nhưng ảnh không đi đâu cả." |
| Ban tổ chức lớp | "Muốn thêm ảnh vào thư viện thì phải nhờ người biết code sửa file HTML." |
| Người thực hiện | "Trang mở công khai thì ai cũng gửi được gì lên, không có cách nào duyệt." |
| Cả lớp, 5 năm sau | "Máy chủ hỏng thì mất hết, không có bản sao nào." |

## 3. Mục tiêu sản phẩm

| # | Mục tiêu | Đo bằng |
|---|---|---|
| PG1 | Nội dung chung cho mọi người xem | Hai thiết bị khác nhau thấy cùng dữ liệu |
| PG2 | Góp nội dung dễ như điền một biểu mẫu | Gửi được ảnh + lời nhắn từ điện thoại trong < 60 giây, không cần đăng nhập |
| PG3 | Không nội dung nào lên trang mà chưa qua người thật xem | Mặc định `PENDING`, chỉ Quản trị viên đổi sang `PUBLISHED` |
| PG4 | Tôn trọng người trong ảnh | Có kênh yêu cầu gỡ, có ghi nhận, có xử lý trong 48 giờ |
| PG5 | Trang vẫn đẹp và mượt như bản dựng tạm | Giữ nguyên hiệu ứng cuộn, tranh động SVG, và hỗ trợ `prefers-reduced-motion` |
| PG6 | Vận hành được bởi một người | Toàn bộ hệ thống khởi động bằng một lệnh `docker compose up`; sao lưu chạy tự động |

## 4. Đối tượng người dùng (User Personas)

### Persona 1: Khách xem (Visitor) — không đăng nhập

- **Là ai**: bất kỳ ai được chia sẻ đường dẫn — bạn lớp khác, thầy cô, phụ huynh, người quen.
- **Thiết bị**: đa số điện thoại Android/iOS, mạng 4G.
- **Muốn gì**: mở là xem được ngay, không phải đăng ký, xem ảnh phóng to, đọc lời nhắn.
- **Sợ gì**: trang tải lâu, chữ nhỏ, bấm nhầm vào quảng cáo.

### Persona 2: Thành viên lớp (Contributor) — không đăng nhập

- **Là ai**: 40 học sinh lớp 12A1 và thầy cô bộ môn.
- **Muốn gì**: gửi ảnh của mình vào thư viện chung, để lại lời nhắn ký tên mình, tải cả album về máy làm kỷ niệm.
- **Sợ gì**: gửi xong không biết có tới nơi không; ảnh mình gửi bị đăng mà mình chưa muốn.
- **Ghi chú**: cố tình **không** bắt đăng nhập — rào cản đăng ký sẽ giết luôn tỉ lệ đóng góp. Đổi lại, chi phí bảo mật chuyển sang phía máy chủ (kiểm duyệt, giới hạn tần suất, chống bot).

### Persona 3: Quản trị viên / Người kiểm duyệt (Admin) — có đăng nhập

- **Là ai**: 1–3 bạn trong ban tổ chức lớp; không phải dân kỹ thuật.
- **Thiết bị**: chủ yếu điện thoại.
- **Muốn gì**: mở khu quản trị, thấy ngay hàng chờ, vuốt duyệt hoặc từ chối; gỡ nhanh khi có sự cố.
- **Sợ gì**: bấm nhầm nút xoá; quên mật khẩu; tài khoản bị chiếm.
- **Ghi chú**: đây là tài khoản duy nhất trong hệ thống, và là mục tiêu tấn công giá trị nhất → bắt buộc 2FA, xem [`MemoryBook-Security.md`](MemoryBook-Security.md).

### Persona 4: Người yêu cầu gỡ nội dung (Data Subject)

- **Là ai**: người xuất hiện trong ảnh (hoặc phụ huynh của họ) muốn gỡ ảnh xuống.
- **Muốn gì**: một cách liên hệ rõ ràng, không phải đăng nhập, và được phản hồi.
- **Ghi chú**: đây không phải tính năng "cho vui" — với ảnh của người vị thành niên, đây là nghĩa vụ, xem mục Quyền riêng tư ở [`MemoryBook-Security.md`](MemoryBook-Security.md).

## 5. Phạm vi (Scope) — 7 module

### Có trong bản này (v1)

#### Module 1 — Trang kỷ yếu công khai (đã có phần nhìn, cần nối dữ liệu)

| Mục | Neo | Trạng thái hiện tại | v1 cần làm gì |
|---|---|---|---|
| Trang chủ | `#home` | Xong (hero, thống kê đếm số, lưới bento, tranh động SVG sân trường) | Số liệu thống kê lấy từ API thay vì viết cứng |
| Hành trình | `#journey` | Xong (dòng thời gian 3 năm, đường kẻ vẽ theo cuộn) | Nội dung mốc thời gian đưa vào dữ liệu cấu hình |
| Thư viện | `#gallery` | Xong phần nhìn, ảnh viết cứng | Lấy ảnh từ `GET /api/v1/photos` |
| Lớp mình | `#lop-minh` | Xong (12 thẻ lật + khối tri ân thầy cô) | Lấy danh sách từ dữ liệu cấu hình, không viết cứng trong `<script>` |
| Lời nhắn | `#messages` | Xong phần nhìn, lưu `localStorage` | Đọc/ghi qua API |
| Đóng góp | `#contribute` | Xong phần nhìn, không gửi đi đâu | Tải ảnh thật lên máy chủ |

#### Module 2 — Lời nhắn chung

- Xem danh sách lời nhắn đã duyệt, mới nhất trước, cuộn tới đâu tải tới đó.
- Gửi lời nhắn (họ tên, biệt danh tuỳ chọn, nội dung).
- Sau khi gửi: báo rõ "đang chờ duyệt", không giả vờ là đã đăng.
- Chống spam: giới hạn tần suất theo IP + Turnstile.

#### Module 3 — Đóng góp ảnh

- Kéo thả hoặc chọn ảnh, xem trước tại máy trước khi gửi.
- Kèm ghi chú, ngày chụp (nhập tự do), phân loại (`Lớp 10` / `Lớp 11` / `Lớp 12` / `Sự kiện`).
- Ảnh gửi lên mặc định ở trạng thái chờ duyệt.
- Có thanh tiến trình khi tải; báo lỗi rõ ràng khi file quá lớn hoặc sai định dạng.
- **Thay đổi bắt buộc ở front-end**: biểu mẫu hiện tại chưa gửi kèm phân loại đang chọn — cần thêm trường ẩn.

#### Module 4 — Thư viện ảnh & tải album

- Băng chuyền 3D và tường ảnh polaroid dựng từ dữ liệu máy chủ.
- Xem toàn màn hình, điều khiển được bằng bàn phím.
- Lọc theo phân loại.
- Ảnh phục vụ theo nhiều kích cỡ (thumbnail / vừa / gốc) và định dạng hiện đại để tiết kiệm băng thông.
- Nút "Tải xuống (HD)" trả về tệp nén chứa các ảnh đã duyệt.

#### Module 5 — Khu quản trị & kiểm duyệt

- Đăng nhập bằng mật khẩu + mã 2FA (TOTP).
- Hàng chờ duyệt gộp cả ảnh lẫn lời nhắn, sắp theo thời gian gửi.
- Thao tác: **Duyệt** / **Từ chối** / **Ẩn** / **Xoá vĩnh viễn**; xoá vĩnh viễn phải xác nhận hai bước.
- Chỉnh sửa nhẹ trước khi duyệt (sửa lỗi chính tả trong ghi chú ảnh, đổi phân loại).
- Nhật ký thao tác (ai làm gì, lúc nào, trước/sau ra sao).

#### Module 6 — Quyền riêng tư & yêu cầu gỡ nội dung

- Trang "Quyền riêng tư" nêu rõ: dữ liệu nào được lưu, lưu bao lâu, liên hệ ai để gỡ.
- Biểu mẫu yêu cầu gỡ, gắn được vào từng ảnh cụ thể (nút "Yêu cầu gỡ ảnh này" trong khung xem ảnh).
- Yêu cầu gỡ vào thẳng hàng chờ ưu tiên cao của Quản trị viên.
- `robots.txt` và thẻ `noindex` chặn máy tìm kiếm lập chỉ mục ảnh.

#### Module 7 — Vận hành

- Sao lưu tự động cơ sở dữ liệu + ảnh sang nơi lưu trữ khác máy chủ, hằng đêm.
- Trang trạng thái nội bộ (health check) và bảng giám sát.
- Trang lỗi thân thiện khi máy chủ bận hoặc bảo trì; front-end vẫn hiển thị được phần tĩnh nếu API chết.

### Không có trong v1 (để ở Roadmap)

| Tính năng | Vì sao hoãn |
|---|---|
| Tài khoản riêng cho từng học sinh | Rào cản đăng ký làm giảm mạnh tỉ lệ đóng góp; v1 chọn "không đăng nhập + kiểm duyệt" |
| Bình luận / thả tim theo từng ảnh | Mở thêm bề mặt spam trong khi chưa có lực kiểm duyệt |
| Video, nhạc do người dùng tải lên | Chi phí lưu trữ và băng thông tăng đột biến; quét mã độc phức tạp hơn nhiều |
| Nhận diện khuôn mặt để tự gắn thẻ tên | Vấn đề pháp lý và đạo đức với dữ liệu sinh trắc của người vị thành niên — cân nhắc rất kỹ trước khi làm |
| Ứng dụng di động native | Web đáp ứng đủ; không đáng công sức |
| Đa ngôn ngữ giao diện | Người dùng đều là người Việt |
| Thông báo đẩy khi có lời nhắn mới | Không giải quyết vấn đề cốt lõi nào |

## 6. User Stories theo vai trò

### Khách xem

| ID | Câu chuyện | Tiêu chí chấp nhận |
|---|---|---|
| US-V-01 | Là khách, tôi muốn mở trang trên điện thoại và xem được ngay, để không phải đăng ký gì | Trang hiển thị nội dung chính trong < 2,5 giây trên 4G; không có bước đăng nhập |
| US-V-02 | Là khách, tôi muốn phóng to từng ảnh, để nhìn rõ mặt mọi người | Bấm ảnh mở khung toàn màn hình; chuyển ảnh bằng vuốt hoặc phím mũi tên; `Esc` để đóng |
| US-V-03 | Là khách, tôi muốn đọc lời nhắn của cả lớp | Danh sách mới nhất trước, tự tải thêm khi cuộn |
| US-V-04 | Là khách bị say chuyển động, tôi muốn tắt hiệu ứng | Bật `prefers-reduced-motion` ở hệ điều hành là mọi hoạt ảnh dừng |
| US-V-05 | Là khách, tôi muốn xem được cả khi máy chủ đang lỗi | API chết thì phần tĩnh vẫn hiển thị, kèm thông báo nhẹ nhàng thay vì trang trắng |

### Thành viên lớp

| ID | Câu chuyện | Tiêu chí chấp nhận |
|---|---|---|
| US-C-01 | Là thành viên lớp, tôi muốn gửi lời nhắn ký tên mình | Gửi thành công nhận `202`; giao diện nói rõ "chờ duyệt", không hiển thị giả là đã đăng |
| US-C-02 | Là thành viên lớp, tôi muốn gửi ảnh chất lượng cao từ điện thoại | Chọn được ảnh từ thư viện máy; **ảnh ≤ 25 MB được chấp nhận, giữ nguyên bản gốc không nén lại**; thanh tiến trình chạy đúng suốt 30–60 giây tải lên trên 4G |
| US-C-03 | Là thành viên lớp, tôi muốn biết ảnh mình gửi thuộc mốc thời gian nào | Chọn được một trong bốn phân loại và phân loại đó **thực sự được gửi lên** |
| US-C-04 | Là thành viên lớp, tôi muốn tải cả album về máy | Bấm "Tải xuống (HD)" tải về tệp nén ảnh đã duyệt |
| US-C-05 | Là thành viên lớp, tôi muốn hệ thống không cho gửi nhầm file không phải ảnh | Báo lỗi rõ ràng ngay tại biểu mẫu trước khi tải lên |

### Quản trị viên

| ID | Câu chuyện | Tiêu chí chấp nhận |
|---|---|---|
| US-A-01 | Là quản trị viên, tôi muốn đăng nhập an toàn | Mật khẩu + mã TOTP; sai 5 lần bị khoá tạm; phiên hết hạn sau 8 giờ không hoạt động |
| US-A-02 | Là quản trị viên, tôi muốn thấy hàng chờ duyệt trên điện thoại | Danh sách gộp ảnh + lời nhắn, có ảnh xem trước, thao tác một chạm |
| US-A-03 | Là quản trị viên, tôi muốn từ chối nội dung xấu | Từ chối là chuyển sang `REJECTED`, không hiển thị công khai, vẫn lưu để đối chiếu |
| US-A-04 | Là quản trị viên, tôi muốn gỡ ngay một ảnh khi có người yêu cầu | Một thao tác chuyển ảnh sang `HIDDEN`; ảnh biến mất khỏi trang công khai trong < 60 giây kể cả khi đã lên CDN |
| US-A-05 | Là quản trị viên, tôi muốn biết ai đã làm gì | Nhật ký thao tác xem được, không sửa được, giữ tối thiểu 24 tháng |
| US-A-06 | Là quản trị viên, tôi muốn tránh bấm nhầm nút xoá | Xoá vĩnh viễn cần xác nhận hai bước và có thời gian hoàn tác 30 ngày (xoá mềm trước) |

### Người yêu cầu gỡ nội dung

| ID | Câu chuyện | Tiêu chí chấp nhận |
|---|---|---|
| US-P-01 | Là người trong ảnh, tôi muốn yêu cầu gỡ mà không cần tài khoản | Nút "Yêu cầu gỡ ảnh này" ngay trong khung xem ảnh, mở biểu mẫu ngắn |
| US-P-02 | Là người yêu cầu, tôi muốn biết yêu cầu đã được nhận | Nhận mã tra cứu; nếu để lại email thì nhận phản hồi khi xử lý xong |
| US-P-03 | Là người yêu cầu, tôi muốn biết trang lưu gì về tôi | Trang "Quyền riêng tư" liệt kê rõ từng loại dữ liệu và thời hạn lưu |

## 7. Yêu cầu chức năng chính (Functional Requirements)

| ID | Yêu cầu | Ưu tiên |
|---|---|---|
| FR-01 | Hiển thị danh sách lời nhắn đã duyệt, phân trang theo con trỏ, mới nhất trước | Bắt buộc |
| FR-02 | Nhận lời nhắn mới từ khách, mặc định trạng thái `PENDING` | Bắt buộc |
| FR-03 | Nhận ảnh tải lên từ khách kèm ghi chú / ngày chụp / phân loại, mặc định `PENDING` | Bắt buộc |
| FR-04 | Kiểm tra ảnh tải lên là ảnh thật (đọc nội dung tệp, không tin phần mở rộng), chặn tệp quá lớn và ảnh có kích thước bất thường | Bắt buộc |
| FR-05 | Sinh các phiên bản ảnh (thumbnail / vừa / gốc) và loại bỏ siêu dữ liệu EXIF trước khi công bố | Bắt buộc |
| FR-06 | Cung cấp danh sách ảnh đã duyệt, lọc theo phân loại | Bắt buộc |
| FR-07 | Đóng gói ảnh đã duyệt thành tệp nén cho nút "Tải xuống (HD)" | Nên có |
| FR-08 | Đăng nhập quản trị bằng mật khẩu + TOTP, có khoá tạm khi sai nhiều lần | Bắt buộc |
| FR-09 | Hàng chờ kiểm duyệt: xem, duyệt, từ chối, ẩn, xoá mềm, khôi phục | Bắt buộc |
| FR-10 | Ghi nhật ký mọi thao tác thay đổi dữ liệu của quản trị viên, không sửa/xoá được | Bắt buộc |
| FR-11 | Giới hạn tần suất gửi lời nhắn và ảnh theo IP đã băm | Bắt buộc |
| FR-12 | Xác minh người thật bằng Cloudflare Turnstile trước khi nhận nội dung | Nên có |
| FR-13 | Nhận yêu cầu gỡ nội dung và đưa vào hàng chờ ưu tiên cao | Bắt buộc |
| FR-14 | Trang "Quyền riêng tư" và `robots.txt` chặn lập chỉ mục | Bắt buộc |
| FR-15 | Sao lưu tự động cơ sở dữ liệu và ảnh hằng đêm sang nơi lưu trữ khác máy chủ | Bắt buộc |
| FR-16 | Điểm cuối kiểm tra sức khoẻ và số liệu giám sát cho hệ thống theo dõi | Nên có |
| FR-17 | Danh sách học sinh / thầy cô và các mốc "Hành trình" lấy từ dữ liệu cấu hình, không viết cứng trong mã | Nên có |
| FR-18 | Front-end vẫn hiển thị được phần tĩnh khi API không phản hồi | Nên có |

## 8. Yêu cầu phi chức năng (tóm tắt — chi tiết ở SRS)

| Nhóm | Yêu cầu tóm tắt |
|---|---|
| Hiệu năng | Nội dung chính hiển thị < 2,5 giây trên 4G; API danh sách < 300 ms (p95) |
| Khả dụng | Uptime ≥ 99%; API chết không làm trắng trang |
| Bảo mật | Toàn bộ qua HTTPS; kiểm duyệt trước khi hiển thị; 2FA cho quản trị; theo OWASP Top 10 |
| Quyền riêng tư | Không lưu IP thô (chỉ lưu bản băm); chặn lập chỉ mục; có quy trình gỡ nội dung |
| Khả năng tiếp cận | Tương phản đạt WCAG 2.1 AA; điều khiển được bằng bàn phím; tôn trọng `prefers-reduced-motion` |
| Bảo trì | Hợp đồng API có tài liệu OpenAPI; di trú cơ sở dữ liệu bằng Flyway; kiểm thử tự động cho luồng lõi |
| Chi phí | Tổng chi phí vận hành mục tiêu < 200.000 đ/tháng |

## 9. Ràng buộc & Giả định (Constraints & Assumptions)

**Ràng buộc:**

- Một người phát triển, làm song song với việc học → ưu tiên công nghệ dễ vận hành, ít thành phần.
- Ngân sách hạ tầng ở mức sinh viên → một VPS + object storage tính theo dung lượng, không dùng dịch vụ quản lý đắt tiền.
- Back-end đã chốt là **Spring Boot**; front-end định hướng **React** nhưng chưa chốt cuối.
- Nội dung là dữ liệu cá nhân của người vị thành niên → chịu ràng buộc của Nghị định 13/2023/NĐ-CP.

**Giả định:**

- Quy mô rất nhỏ: khoảng 40 người đóng góp, vài trăm đến vài nghìn lượt xem, tổng ảnh dưới 2.000 tấm.
- Lưu lượng bùng nổ chỉ xảy ra vài đợt ngắn (hôm công bố, ngày ra trường, họp lớp).
- Không có yêu cầu thời gian thực; chậm vài giây là chấp nhận được.
- Ban tổ chức lớp sẵn sàng duyệt nội dung hằng ngày trong giai đoạn đầu.

## 10. Tiêu chí thành công của sản phẩm (Success Metrics)

| # | Chỉ số | Mục tiêu |
|---|---|---|
| M1 | Số ảnh đã duyệt và hiển thị | ≥ 100 trong tháng đầu |
| M2 | Số lời nhắn đã duyệt | ≥ 40 (trung bình mỗi bạn một lời) |
| M3 | Tỉ lệ thành viên lớp có đóng góp ít nhất một nội dung | ≥ 70% |
| M4 | Thời gian trung bình từ lúc gửi tới lúc duyệt | < 24 giờ |
| M5 | Tỉ lệ lỗi khi tải ảnh lên | < 2% số lượt gửi |
| M6 | Sự cố nội dung xấu lọt ra công khai | 0 |
| M7 | Diễn tập phục hồi từ bản sao lưu | Thành công ít nhất 1 lần trước ngày công bố |

---

*Tài liệu này dựa trên [`MemoryBook-BRD.md`](MemoryBook-BRD.md) và là cơ sở để viết yêu cầu chi tiết trong [`MemoryBook-SRS.md`](MemoryBook-SRS.md).*
