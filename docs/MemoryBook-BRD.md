# BRD — Business Requirements Document
## Phượng Hồng Memories — Kỷ yếu điện tử lớp 12A1

---

## 1. Bối cảnh (Background)

Lớp 12A1 trường THPT Mạc Đĩnh Chi sắp ra trường. Kỷ yếu giấy truyền thống có ba vấn đề: in một lần là chốt nội dung, ảnh và lời nhắn của người vắng mặt hôm chụp bị bỏ lại, và sau 5–10 năm thì cuốn sổ nằm trong tủ chứ không còn ai giở ra.

Hiện đã có một bản web chạy được: toàn bộ `client/index.html` — một trang cuộn liền mạch gồm thư viện ảnh, hành trình ba năm, hồ sơ học sinh/thầy cô, bảng lời nhắn, biểu mẫu đóng góp ảnh và một tranh động sân trường vẽ tay bằng SVG. Nhưng bản này **chạy hoàn toàn trên trình duyệt**: lời nhắn lưu trong `localStorage` của chính máy người xem, ảnh tải lên không rời khỏi thiết bị, 32 ảnh đang dùng là ảnh tạm do AI sinh. Nghĩa là mỗi người mở trang lên chỉ thấy đúng những gì chính mình đã gõ — chưa phải một cuốn kỷ yếu chung.

Người thực hiện là sinh viên IT đang tự học. Ngoài giá trị kỷ niệm cho lớp, đây còn là dự án để rèn nghề: dựng một hệ thống full-stack có back-end thật, bảo mật ở mức triển khai được ra internet công cộng, chứ không dừng ở bài tập CRUD.

## 2. Mục tiêu nghiệp vụ / Mục tiêu dự án (Business Goals)

- **G1 — Cuốn kỷ yếu chung, còn lại lâu dài**: mọi thành viên lớp và thầy cô mở cùng một đường dẫn là thấy cùng một nội dung; nội dung sống được ít nhất 10 năm mà không cần người thực hiện can thiệp hằng tuần.
- **G2 — Ai cũng góp được**: một bạn không biết code vẫn gửi được ảnh và lời nhắn từ điện thoại, không cần cài app, không cần tạo tài khoản.
- **G3 — Không thành nơi để người lạ phá**: trang công khai trên internet nhưng mọi nội dung do người ngoài gửi phải qua duyệt trước khi hiển thị.
- **G4 — An toàn cho người trong ảnh**: đây là ảnh của học sinh, phần lớn dưới 18 tuổi tại thời điểm chụp. Phải có cơ chế gỡ ảnh theo yêu cầu, không để ảnh bị máy tìm kiếm lập chỉ mục ngoài ý muốn, và không rò rỉ dữ liệu cá nhân.
- **G5 — Là dự án nghề nghiệp trình bày được**: có tài liệu đầy đủ (bộ này), kiến trúc rõ ràng, bảo mật theo chuẩn công nghiệp (OWASP), có kiểm thử và có triển khai thật — đủ để mang đi phỏng vấn thực tập.

## 3. Nguyên tắc định hướng (Guiding Principle)

**Kỷ niệm là thứ không phục hồi được — dữ liệu đúng và an toàn quan trọng hơn nhiều tính năng.**

Một tấm ảnh lớp mất là mất vĩnh viễn, không có bản gốc thứ hai. Một tấm ảnh riêng tư bị lộ ra ngoài thì không thu hồi được. Vì vậy thứ tự ưu tiên của dự án là: **giữ được dữ liệu → bảo vệ được dữ liệu → rồi mới đến trải nghiệm đẹp → cuối cùng mới đến tính năng mở rộng.** Sao lưu có kiểm chứng phục hồi được đặt trước mọi tính năng mới; kiểm duyệt đặt trước tự động hoá; quyền riêng tư đặt trước SEO.

## 4. Vấn đề nghiệp vụ cần giải quyết (Business Problem)

| # | Vấn đề hiện tại | Hệ quả |
|---|---|---|
| P1 | Lời nhắn chỉ nằm trong `localStorage` của từng trình duyệt, giới hạn 40 mục | Không có bảng lời nhắn chung; xoá lịch sử trình duyệt là mất sạch |
| P2 | Biểu mẫu đóng góp ảnh chỉ xem trước tại máy, không gửi đi đâu | Ảnh của lớp không gom về được, coi như tính năng chưa tồn tại |
| P3 | 32 ảnh viết cứng trong HTML, lại là ảnh tạm do AI sinh và có thể tự đổi nội dung | Muốn thêm/bớt ảnh phải sửa mã và triển khai lại; ảnh hiện tại không dùng để công bố được |
| P4 | Nút "Tải xuống (HD)" chưa gắn xử lý | Không ai lấy được bộ ảnh gốc về máy để lưu riêng |
| P5 | Không có khái niệm người kiểm duyệt | Nếu mở công khai, bất kỳ ai cũng gửi được nội dung xấu lên trang mang tên trường và tên thật của học sinh |
| P6 | Không có bản sao lưu | Một lần hỏng đĩa hoặc xoá nhầm là mất toàn bộ kỷ yếu |

## 5. Phạm vi mục tiêu (Business Scope)

**Trong phạm vi:**

- Xây back-end thật (Spring Boot) thay thế toàn bộ phần mô phỏng trong trình duyệt: lời nhắn chung, đóng góp ảnh có duyệt, thư viện ảnh lấy từ dữ liệu, tải album.
- Viết lại front-end sang React để quản lý được phần giao diện đang phình to trong một file HTML 3.100 dòng.
- Một khu vực quản trị tối giản cho ban tổ chức lớp: duyệt / ẩn / xoá lời nhắn và ảnh.
- Bảo mật, sao lưu, giám sát và triển khai ở mức chạy được ngoài internet công cộng.
- Bộ tài liệu BRD → PRD → SRS → HLD → LLD + Tech Stack + Security + Workflow (chính là thư mục `docs/` này).

**Ngoài phạm vi (giai đoạn này):**

- Tài khoản cho từng học sinh, đăng nhập mạng xã hội, kết bạn, bình luận theo luồng.
- Ứng dụng di động native.
- Video, phát trực tiếp, nhạc do người dùng tải lên.
- Bán hàng, thanh toán, quảng cáo.
- Đa ngôn ngữ (giao diện tiếng Việt là đủ; README đã có sẵn song ngữ).

**Chưa chốt:**

- Công nghệ front-end. React là hướng đang chọn, nhưng bản `client/index.html` hiện tại **chỉ là bản dựng tạm để xem giao diện**, chưa phải bản cuối, và có thể đổi sang hướng khác (Next.js, Astro…) khi bắt đầu làm thật. Mọi tài liệu trong thư mục này vì vậy tách rõ phần **hợp đồng API** (ổn định) khỏi phần **công nghệ hiện thực front-end** (thay được).

## 6. Bên liên quan (Stakeholders)

| Vai trò | Ai | Quan tâm điều gì |
|---|---|---|
| Chủ dự án / Lập trình viên | Người thực hiện | Sản phẩm chạy được, học được nghề, có cái để đưa vào hồ sơ |
| Ban tổ chức lớp (Quản trị viên) | 1–3 bạn trong lớp | Duyệt nội dung nhanh trên điện thoại, gỡ được ngay khi có sự cố |
| Thành viên lớp 12A1 | ~40 học sinh | Gửi ảnh và lời nhắn dễ dàng, thấy tên mình đúng, ảnh mình được tôn trọng |
| Thầy cô | Giáo viên chủ nhiệm và bộ môn | Nội dung chỉn chu, đúng mực, không sai thông tin về trường |
| Phụ huynh / người trong ảnh | Gia đình học sinh | Ảnh con em không bị phát tán ngoài ý muốn, gỡ được khi có yêu cầu |
| Khách ghé thăm | Người được chia sẻ đường dẫn | Trang mở nhanh, xem tốt trên điện thoại |

## 7. Tiêu chí thành công ở tầm nghiệp vụ (Business Success Criteria)

| # | Tiêu chí | Ngưỡng đạt |
|---|---|---|
| S1 | Lời nhắn là chung, không còn cục bộ theo máy | Hai thiết bị khác nhau mở cùng đường dẫn thấy cùng danh sách lời nhắn |
| S2 | Lớp gom được ảnh thật | ≥ 100 ảnh do thành viên lớp gửi, đã duyệt và hiển thị; 0 ảnh tạm do AI sinh còn sót |
| S3 | Kiểm duyệt kịp thời | 100% nội dung người ngoài gửi ở trạng thái chờ duyệt; thời gian duyệt trung bình < 24 giờ |
| S4 | Không có sự cố nội dung | 0 lần nội dung xấu hiển thị công khai trong suốt thời gian vận hành |
| S5 | Dữ liệu không mất | Sao lưu hằng đêm; **đã diễn tập phục hồi thành công ít nhất 1 lần** và ghi lại kết quả |
| S6 | Gỡ theo yêu cầu | Yêu cầu gỡ ảnh được xử lý trong vòng 48 giờ, có nhật ký ghi nhận |
| S7 | Chạy thật, ổn định | Trang có tên miền + HTTPS, uptime ≥ 99% trong 3 tháng đầu sau khi công bố |
| S8 | Giá trị nghề nghiệp | Trình bày được kiến trúc và các quyết định bảo mật trong 20–30 phút phỏng vấn mà không phải mở lại tài liệu |

## 8. Rủi ro ở tầm nghiệp vụ (Business Risks)

| Rủi ro | Ảnh hưởng | Cách giảm thiểu |
|---|---|---|
| Trang công khai bị spam / nội dung xấu | Bôi nhọ tên lớp và tên trường thật ghi trên trang | Duyệt trước khi hiển thị (mặc định `PENDING`), giới hạn tần suất theo IP, Cloudflare Turnstile chống bot |
| Ảnh học sinh (đa số vị thành niên) bị phát tán hoặc dùng sai mục đích | Ảnh hưởng thật đến người trong ảnh; có thể vi phạm Nghị định 13/2023/NĐ-CP về bảo vệ dữ liệu cá nhân | Xin đồng thuận trước khi đăng, có quy trình gỡ, `robots.txt` chặn lập chỉ mục, cân nhắc chế độ chỉ ai có đường dẫn mới xem được |
| Dữ liệu mất do hỏng máy chủ hoặc xoá nhầm | Mất kỷ niệm không phục hồi được — thiệt hại nặng nhất của dự án | Sao lưu cả cơ sở dữ liệu lẫn ảnh sang nơi khác máy chủ, hằng đêm, giữ 14 bản; diễn tập phục hồi định kỳ |
| Chi phí vận hành vượt khả năng sinh viên | Hết tiền → tắt máy chủ → mất trang | Chọn hạ tầng chi phí thấp (một VPS + object storage tính theo dung lượng), ước tính chi phí trước ở tài liệu Tech Stack, đặt cảnh báo ngân sách |
| Làm quá nhiều, không phần nào xong | Đến ngày ra trường vẫn chưa có gì để chia sẻ cho lớp | Bám mốc: hết Giai đoạn 2 là đã dùng thật được; các tính năng còn lại đẩy sang Roadmap |
| Đổi công nghệ front-end giữa chừng | Phải viết lại phần đã làm | Chốt **hợp đồng API** trước và giữ ổn định; front-end coi như thay được, không để logic nghiệp vụ nằm ở front-end |
| Người thực hiện bận thi cử, dự án đứng | Trang hỏng mà không ai sửa | Ưu tiên vận hành đơn giản (một `docker compose`), tài liệu vận hành rõ, để ít nhất một bạn khác trong lớp biết cách khởi động lại |

---

*Tài liệu này là điểm khởi đầu của chuỗi tài liệu dự án: **BRD (tài liệu này) → PRD → SRS → HLD → LLD**. Khi cần nhắc lại "vì sao làm dự án này", quay lại đọc file này trước tiên. Chỉ mục toàn bộ bộ tài liệu: [`README.md`](README.md).*
