# Workflow — Quy trình làm việc & Bộ tài liệu dự án
## Phượng Hồng Memories — Kỷ yếu điện tử lớp 12A1

---

## 1. Vì sao cần tài liệu dù chỉ làm một mình

Ba lý do thực tế, không phải lý thuyết:

1. **Bạn của ba tháng sau là một người khác.** Đến lúc quay lại sửa lỗi, bạn sẽ không nhớ vì sao mình chọn cookie phiên thay vì JWT, hay vì sao ảnh phải phục vụ từ tên miền riêng. Tài liệu là ghi chú gửi cho chính mình.
2. **Đây là thứ mang đi phỏng vấn được.** Một dự án có BRD → PRD → SRS → HLD → LLD cho thấy tư duy hệ thống. Người phỏng vấn phân biệt rất nhanh giữa "làm theo hướng dẫn trên mạng" và "hiểu vì sao làm thế".
3. **Viết ra thì lộ ra chỗ nghĩ chưa tới.** Lúc viết luồng gỡ nội dung ở mục 4.5 của HLD mới nhận ra bước xoá đệm CDN — nếu bỏ qua, ảnh "đã gỡ" vẫn xem được. Đó là loại lỗi chỉ lộ ra khi viết, không lộ ra khi code.

**Nhưng đừng viết hết tài liệu rồi mới code.** Đó là cái bẫy lớn nhất. Xem thứ tự ưu tiên ở mục 2.

---

## 2. Bảy tài liệu: là gì và dùng khi nào

| Tài liệu | Trả lời câu hỏi | Viết khi nào | Mức ưu tiên |
|---|---|---|---|
| [`BRD`](MemoryBook-BRD.md) | Vì sao làm dự án này? | Trước tiên, viết một lần | ⭐⭐ Ngắn nhưng quan trọng — là nơi quay về khi phân vân |
| [`PRD`](MemoryBook-PRD.md) | Làm cái gì, cho ai? | Sau BRD | ⭐⭐⭐ Xác định phạm vi, chống làm lan man |
| [`SRS`](MemoryBook-SRS.md) | Hệ thống phải làm được gì, cụ thể? | Sau PRD | ⭐⭐ Là danh sách để đối chiếu khi nghiệm thu |
| [`HLD`](MemoryBook-HLD.md) | Kiến trúc trông như thế nào? | Trước khi code | ⭐⭐⭐ Sai kiến trúc thì sửa rất đắt |
| [`LLD`](MemoryBook-LLD.md) | Từng phần hoạt động ra sao? | **Song song với code** | ⭐⭐⭐ Giá trị nhất khi phỏng vấn |
| [`TechStack`](MemoryBook-TechStack.md) | Dùng gì và vì sao? | Trước khi code, cập nhật khi đổi | ⭐⭐ Mục "cố tình không dùng" hay được hỏi nhất |
| [`Security`](MemoryBook-Security.md) | Chống lại ai, bằng cách nào? | Trước khi code, dùng như checklist | ⭐⭐⭐ Không có thì không được công bố |

### Thứ tự ưu tiên khi thời gian eo hẹp

Nếu chỉ làm được ba tài liệu: **HLD, LLD, Security**. Ba tài liệu này ảnh hưởng trực tiếp tới việc code, và cũng là ba tài liệu người phỏng vấn quan tâm nhất.

BRD và PRD ngắn, viết một buổi là xong, và giúp chống lan man về sau — đáng bỏ ra vài giờ.

SRS là tài liệu dễ bị viết thành hình thức nhất. Nếu thấy đang viết cho có, hãy dừng lại và chỉ giữ phần yêu cầu phi chức năng (mục 4) — phần đó có ngưỡng đo được nên có giá trị thật.

### Quy tắc quan trọng nhất

**LLD viết song song với code, không viết trước.**

Chi tiết hiện thực thực tế luôn khác dự tính. Cách làm đúng: code xong một phần → viết lại phần LLD tương ứng cho khớp với thực tế → phần nào chưa làm thì đánh dấu `[CHƯA HIỆN THỰC]`. Một tài liệu nói dối còn tệ hơn không có tài liệu.

---

## 3. Lộ trình xây dựng theo giai đoạn

Nguyên tắc: **mỗi giai đoạn kết thúc bằng một thứ chạy được, không phải một thứ làm dở.**

### Giai đoạn 0 — Chuẩn bị (khoảng 1 tuần)

- [ ] Đọc lại toàn bộ bộ tài liệu này, sửa chỗ nào thấy sai
- [ ] Khởi tạo kho git, thiết lập `.gitignore` và `gitleaks`
- [ ] Dựng `docker-compose.yml` cho môi trường phát triển: PostgreSQL + Redis
- [ ] Khởi tạo dự án Spring Boot với các phụ thuộc ở TechStack mục 4.2
- [ ] Viết tệp di trú Flyway `V1__init.sql` theo LLD mục 2
- [ ] Dựng CI cơ bản: build + test

**Kết thúc giai đoạn**: `docker compose up` chạy được, ứng dụng khởi động, `/actuator/health` trả `UP`.

### Giai đoạn 1 — Lời nhắn đầu-cuối (1–2 tuần)

Chọn lời nhắn làm tính năng đầu tiên vì nó là tính năng **đơn giản nhất đi hết mọi tầng** — nếu làm xong tính năng này thì bộ khung đã đúng.

- [ ] Entity + Repository + Service + Controller cho lời nhắn
- [ ] Phân trang theo con trỏ (LLD mục 5.2)
- [ ] Kiểm tra dữ liệu vào, chuẩn hoá Unicode
- [ ] `GlobalExceptionHandler` với Problem Details
- [ ] Giới hạn tần suất với Bucket4j + Redis
- [ ] Kiểm thử tích hợp với Testcontainers
- [ ] Nối vào bản dựng tạm `client/index.html` — thay `loadSavedNotes()` và `saveNote()` bằng lời gọi API

**Kết thúc giai đoạn**: gửi lời nhắn từ điện thoại, mở trên máy khác thấy được (sau khi duyệt bằng tay trong cơ sở dữ liệu).

### Giai đoạn 2 — Quản trị & kiểm duyệt (1–2 tuần)

- [ ] Cấu hình Spring Security theo LLD mục 6.1
- [ ] Đăng nhập + TOTP + mã dự phòng
- [ ] Điểm cuối hàng chờ kiểm duyệt
- [ ] `AuditLogAspect`
- [ ] Giao diện quản trị tối giản — **ưu tiên dùng được trên điện thoại**, không cần đẹp

**Kết thúc giai đoạn**: 🎉 **Đây là mốc dùng thật được.** Lớp gửi lời nhắn, ban tổ chức duyệt trên điện thoại. Nếu hết thời gian ở đây thì dự án vẫn có giá trị.

### Giai đoạn 3 — Ảnh (2–3 tuần, phần khó nhất)

- [ ] `ImageValidator` đủ 5 bước (LLD mục 7.1)
- [ ] Kho đối tượng: vùng cách ly và vùng công bố
- [ ] Bảng outbox + poller
- [ ] Tích hợp ClamAV
- [ ] Mã hoá lại + xoay theo EXIF + sinh biến thể
- [ ] Điểm cuối danh sách ảnh với đầy đủ thông tin kích thước
- [ ] Nối vào biểu mẫu đóng góp; **thêm trường ẩn cho phân loại và ô đồng thuận** (hai thứ đang thiếu)

**Kết thúc giai đoạn**: gửi ảnh từ iPhone, duyệt, thấy ảnh hiện trên trang với ba kích cỡ.

### Giai đoạn 4 — Bảo mật & vận hành (1–2 tuần)

- [ ] Đi hết checklist ở [`Security`](MemoryBook-Security.md) mục 11
- [ ] Turnstile
- [ ] Trang "Quyền riêng tư" + `robots.txt` + kênh yêu cầu gỡ
- [ ] Sao lưu tự động + **một lần phục hồi thử có biên bản**
- [ ] Prometheus + Grafana + cảnh báo
- [ ] Caddy + tên miền + HTTPS
- [ ] Quét bảo mật trong CI

**Kết thúc giai đoạn**: trang chạy trên tên miền thật, đủ điều kiện công bố.

### Giai đoạn 5 — Viết lại front-end bằng React (2–4 tuần)

Đặt ở cuối có chủ đích. Bản dựng tạm đã dùng được, nên đây là việc **cải thiện**, không phải việc **chặn đường**. Nếu hết thời gian, bản HTML một tệp vẫn phục vụ được cả lớp.

- [ ] Khởi tạo Vite + React + TypeScript
- [ ] Sinh kiểu dữ liệu từ OpenAPI
- [ ] Chuyển từng phần theo bảng đối chiếu ở [`TechStack`](MemoryBook-TechStack.md) mục 3.4
- [ ] Chuyển tranh động SVG sân trường sang thành phần React
- [ ] Bỏ toàn bộ CDN bên thứ ba, siết CSP
- [ ] Kiểm thử Playwright cho ba luồng chính
- [ ] Kiểm tra Lighthouse đạt ngưỡng NFR-PERF-04 đến NFR-PERF-06

### Giai đoạn 6 — Hoàn thiện (nếu còn thời gian)

- [ ] Tải album (`GET /api/v1/album`)
- [ ] Dữ liệu cấu hình: danh sách lớp, mốc hành trình, thống kê
- [ ] Kiểm thử tải bằng k6
- [ ] Thay toàn bộ 32 ảnh tạm bằng ảnh thật của lớp

---

## 4. Áp dụng AI vào quy trình — và cách không bị hổng kiến thức

AI giúp đi nhanh hơn nhiều ở dự án này. Nhưng có một rủi ro rất cụ thể: **code chạy được mà không giải thích được, và khi bị hỏi trong phỏng vấn thì đuối.**

| Giai đoạn | Dùng AI thế nào | Ranh giới |
|---|---|---|
| Viết tài liệu | Dựng khung, gợi ý mục còn thiếu, viết lại cho gọn | Nội dung quyết định phải là của mình — AI không biết lớp bạn cần gì |
| Thiết kế | Hỏi về đánh đổi giữa các phương án, tìm trường hợp biên chưa nghĩ ra | Tự chọn phương án cuối và **tự viết lý do bằng lời của mình** |
| Code | Sinh mã lặp lại (entity, DTO, mapper, cấu hình), giải thích lỗi | Phần lõi (xử lý ảnh, bảo mật, outbox) tự viết trước, dùng AI để rà lại |
| Kiểm thử | Sinh trường hợp kiểm thử, đặc biệt là trường hợp biên | Tự kiểm tra kiểm thử có thật sự kiểm đúng thứ mình nghĩ không |
| Rà soát | Nhờ tìm lỗ hổng bảo mật, lỗi N+1, lỗi tương tranh | Không tin ngay — tự xác minh trước khi sửa |

**Quy tắc một câu**: sau mỗi lần dùng AI, **tự giải thích lại đoạn mã đó bằng lời của mình, không nhìn màn hình.** Nếu không giải thích được thì chưa hiểu — quay lại đọc cho tới khi hiểu. Đây là ranh giới giữa dùng AI để đi nhanh hơn và dùng AI để tự lừa mình.

Ba câu hỏi tự kiểm tra trước mỗi lần commit:

1. Đoạn này giải quyết vấn đề gì? (nếu không trả lời được → có thể là mã thừa)
2. Nếu bỏ nó đi thì hỏng ở đâu?
3. Có cách nào đơn giản hơn không?

---

## 5. Chiến lược kiểm thử

Hình tháp: nhiều kiểm thử đơn vị ở đáy, ít kiểm thử đầu-cuối ở đỉnh.

### 5.1 Kiểm thử đơn vị (nhiều nhất, chạy nhanh)

Kiểm quy tắc nghiệp vụ thuần, không đụng cơ sở dữ liệu:

- Chuyển trạng thái: `PENDING` → `PUBLISHED` hợp lệ; `REJECTED` → `PUBLISHED` thì sao?
- Suy ra tên hiển thị: `nickname` → `fullname` → `"Ẩn danh"`
- Chuẩn hoá Unicode và đếm URL trong lời nhắn
- Mã hoá / giải mã con trỏ phân trang, bao gồm cả con trỏ hỏng

### 5.2 Kiểm thử tích hợp (Testcontainers — quan trọng nhất ở dự án này)

Chạy PostgreSQL, Redis và ClamAV thật trong Docker:

- Phân trang theo con trỏ cho kết quả đúng khi có bản ghi mới chèn vào giữa chừng
- Chỉ mục một phần thật sự được dùng (kiểm bằng `EXPLAIN`)
- Giới hạn tần suất đúng khi có nhiều yêu cầu đồng thời
- Bảng outbox: `FOR UPDATE SKIP LOCKED` không cấp cùng một việc cho hai tiến trình
- Tác vụ xử lý ảnh chạy hai lần cho cùng kết quả
- `UPDATE`/`DELETE` trên `audit_logs` bị cơ sở dữ liệu từ chối

**Không dùng H2 thay PostgreSQL.** H2 không có chỉ mục một phần, không có `JSONB`, không có `FOR UPDATE SKIP LOCKED` — tức là không kiểm thử được đúng những phần khó nhất.

### 5.3 Kiểm thử bảo mật (phần đặc thù của dự án này)

Đây là nhóm kiểm thử đáng đầu tư nhất, vì tải tệp lên là bề mặt tấn công lớn nhất:

- [ ] Tệp SVG đổi đuôi thành `.jpg` → phải bị từ chối `415`
- [ ] Tệp PNG bom nén → phải bị từ chối `413`
- [ ] Tệp thử EICAR → phải bị cách ly, không bao giờ lên `public/`
- [ ] Ảnh đầu ra không còn EXIF/GPS → dùng công cụ xem siêu dữ liệu để xác nhận
- [ ] Ảnh dọc chụp bằng điện thoại hiển thị đúng chiều
- [ ] Mọi đường dẫn `/api/v1/admin/**` trả `401` khi chưa đăng nhập
- [ ] Không phản hồi lỗi nào chứa tên lớp Java hoặc dấu vết ngăn xếp
- [ ] Yêu cầu ghi không kèm token CSRF bị từ chối
- [ ] Gửi vượt giới hạn tần suất → nhận `429` kèm `Retry-After`
- [ ] Đặt header `X-Forwarded-For` giả không vượt được giới hạn tần suất

### 5.4 Kiểm thử đầu-cuối (ít nhất, chỉ ba luồng)

Playwright: xem trang · gửi lời nhắn · tải ảnh lên.

### 5.5 Chạy cái gì khi nào

| Thời điểm | Chạy gì | Thời gian mục tiêu |
|---|---|---|
| Khi lưu tệp (máy cá nhân) | Kiểm thử đơn vị của phần đang sửa | < 5 giây |
| Trước khi commit | Toàn bộ kiểm thử đơn vị + lint | < 30 giây |
| Trong CI (mỗi lần đẩy mã) | Đơn vị + tích hợp + kiểm thử bảo mật + quét lỗ hổng | < 10 phút |
| Trước khi triển khai | Thêm kiểm thử đầu-cuối | < 15 phút |
| Hằng tuần | Kiểm thử tải k6 | — |

---

## 6. Quy trình Git

Làm một mình nên không cần quy trình nhánh phức tạp:

```
main            ← luôn ở trạng thái triển khai được
 └── feat/…     ← một nhánh cho một tính năng, gộp vào main khi CI xanh
 └── fix/…
```

| Quy tắc | Vì sao |
|---|---|
| Không commit thẳng vào `main` khi CI đang đỏ | `main` phải luôn triển khai được |
| Một commit làm một việc | Quay lui được từng phần khi cần |
| Thông điệp commit viết rõ **vì sao**, không chỉ **cái gì** | "sửa lỗi ảnh xoay ngang do bỏ EXIF" hữu ích hơn "sửa xử lý ảnh" |
| Không bao giờ commit `.env` | Có `gitleaks` chặn, nhưng đừng dựa vào nó |
| Gắn thẻ phiên bản khi triển khai | Truy ngược được chính xác mã đang chạy |

---

## 7. Checklist tổng hợp

### Trước khi bắt đầu code
- [ ] Đã đọc hết bộ tài liệu và sửa chỗ thấy sai
- [ ] Đã chốt: công khai hoàn toàn hay chỉ ai có đường dẫn mới xem được
- [ ] Đã hỏi ý kiến lớp về việc đăng ảnh
- [ ] Đã dựng môi trường phát triển bằng Docker Compose

### Trong lúc code
- [ ] Cập nhật LLD song song, đánh dấu `[CHƯA HIỆN THỰC]` phần chưa làm
- [ ] Mỗi tính năng có kiểm thử trước khi coi là xong
- [ ] Tự giải thích lại được mọi đoạn mã do AI sinh
- [ ] Không bao giờ commit bí mật

### Trước khi công bố
- [ ] Đi hết checklist ở [`Security`](MemoryBook-Security.md) mục 11 — **không bỏ mục nào**
- [ ] Đã phục hồi thử từ sao lưu thành công ít nhất một lần
- [ ] Đã thay hết 32 ảnh tạm bằng ảnh thật của lớp
- [ ] Đã có ít nhất một bạn khác trong lớp biết cách khởi động lại hệ thống

### Sau khi công bố
- [ ] Duyệt hàng chờ hằng ngày trong tuần đầu
- [ ] Theo dõi cảnh báo
- [ ] Ghi lại sự cố và cách xử lý
- [ ] Cập nhật tài liệu khi hệ thống thay đổi

---

*Chỉ mục toàn bộ bộ tài liệu: [`README.md`](README.md).*
