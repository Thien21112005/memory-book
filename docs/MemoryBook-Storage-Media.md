# Storage & Media — Lưu trữ và Xử lý ảnh ở quy mô 20 GB
## Phượng Hồng Memories — Kỷ yếu điện tử lớp 12A1

---

## 1. Vì sao có tài liệu riêng cho phần này

Quy mô thực tế của dự án: **ảnh gốc tới 20 MB mỗi tấm, tổng kho có thể lên 20 GB.**

Con số đó nghe không lớn với một hệ thống doanh nghiệp, nhưng nó vượt qua đúng ba ngưỡng khiến thiết kế "cho vui" gãy:

| Ngưỡng bị vượt | Hỏng ở đâu |
|---|---|
| **Ảnh > 10 MB** | Tải lên qua máy chủ ứng dụng bắt đầu chiếm dụng tài nguyên đáng kể; người dùng 4G chờ lâu và bấm gửi lại |
| **Ảnh > 20 megapixel** | Giải mã bằng thư viện Java thuần ngốn hàng trăm MB bộ nhớ **cho mỗi ảnh** — VPS 4 GB sẽ hết bộ nhớ |
| **Tổng kho > 10 GB** | Chi phí truyền dữ liệu ra (egress) trở thành khoản tốn nhất, vượt xa chi phí lưu trữ; và "tải cả album" thành một tệp nén 20 GB là bất khả thi |

Tài liệu này giải quyết cả ba, và là phần **cập nhật đè lên** các con số cũ ở [`MemoryBook-TechStack.md`](MemoryBook-TechStack.md) mục 5 và 8.4.

---

## 2. Tính toán quy mô thực tế

### 2.1 Giả định

| Thông số | Giá trị |
|---|---|
| Số ảnh | ~1.000 tấm |
| Kích thước trung bình ảnh gốc | 20 MB |
| Độ phân giải điển hình | 24–50 megapixel |
| **Tổng ảnh gốc** | **~20 GB** |
| Số người xem | ~40 thành viên lớp + vài trăm khách |
| Cao điểm | Vài đợt ngắn (ngày công bố, họp lớp) |

### 2.2 Bố cục lưu trữ đề xuất

Đây là quyết định quan trọng nhất của tài liệu này — **tách bản lưu trữ khỏi bản phục vụ**:

```
┌──────────────────────────────────────────────────────────────────┐
│  archive/          RIÊNG TƯ TUYỆT ĐỐI, không bao giờ phục vụ ra  │
│  {hash}/original   Bytes gốc y nguyên, không đụng vào            │
│                    → Đây LÀ cuốn kỷ yếu. Tài sản A1 ở BRD.       │
│                    ~20 GB                                        │
├──────────────────────────────────────────────────────────────────┤
│  public/           Phục vụ qua CDN trên tên miền riêng           │
│  {hash}/thumb.avif    400px   ~15 KB                             │
│  {hash}/thumb.jpg     400px   ~30 KB                             │
│  {hash}/medium.avif  1600px  ~150 KB                             │
│  {hash}/medium.jpg   1600px  ~350 KB                             │
│  {hash}/large.avif   4000px  ~900 KB                             │
│  {hash}/large.jpg    4000px  ~2,5 MB   ← đây là bản "Tải HD"     │
│                    Tất cả đã bỏ EXIF/GPS                         │
│                    ~4 GB cho 1.000 ảnh                           │
├──────────────────────────────────────────────────────────────────┤
│  quarantine/       Vùng cách ly, tự xoá sau 24 giờ               │
│                    Ảnh vừa tải lên, chưa quét, chưa xử lý        │
└──────────────────────────────────────────────────────────────────┘

Tổng: ~24 GB (làm tròn lên 30 GB để có chỗ dư)
```

**Vì sao tách `archive/` và `public/`:**

1. Bản gốc 20 MB **không bao giờ cần phục vụ trực tiếp cho ai.** Không ai xem ảnh 50 megapixel trên điện thoại.
2. Bản gốc còn nguyên EXIF — bao gồm **toạ độ GPS nơi chụp**. Phục vụ nó ra công khai là rò rỉ vị trí của học sinh. Bản `public/` đã bỏ sạch siêu dữ liệu.
3. Bản gốc giữ nguyên bytes nghĩa là **không mất mát chất lượng do nén lại** — đúng nguyên tắc "kỷ niệm không phục hồi được" ở BRD.
4. Bản `large` 4000px (~2,5 MB) đủ để in khổ 30×40 cm, đủ làm nút "Tải xuống (HD)", và **nhỏ hơn bản gốc 8 lần** — điều này giải quyết luôn bài toán tệp nén album ở mục 6.

Chi phí phải trả: lưu 24 GB thay vì 20 GB. Ở mức giá kho đối tượng hiện nay, phần chênh đó khoảng **1.000 đồng mỗi tháng**. Không có lý do gì để tiếc.

### 2.3 Vì sao 3 kích cỡ × 2 định dạng, không nhiều hơn

| Biến thể | Dùng ở đâu |
|---|---|
| `thumb` 400px | Lưới ảnh, tường polaroid, băng chuyền |
| `medium` 1600px | Khung xem toàn màn hình trên điện thoại và laptop |
| `large` 4000px | Nút "Tải xuống (HD)", màn hình 4K, in ấn |
| Định dạng `avif` | Trình duyệt hiện đại — nhỏ hơn JPEG khoảng 30–50% ở cùng chất lượng |
| Định dạng `jpg` | Dự phòng cho trình duyệt cũ; dùng `<picture>` để trình duyệt tự chọn |

Thêm WebP vào giữa AVIF và JPEG là cám dỗ dễ mắc, nhưng nó tăng 50% số tệp và 50% thời gian xử lý để phục vụ một nhóm trình duyệt rất hẹp (những máy hỗ trợ WebP nhưng không hỗ trợ AVIF). Với dự án này, hai định dạng là đủ.

```html
<picture>
  <source srcset="…/thumb.avif 400w, …/medium.avif 1600w" type="image/avif" sizes="(max-width: 768px) 45vw, 300px">
  <img src="…/thumb.jpg" srcset="…/thumb.jpg 400w, …/medium.jpg 1600w"
       width="1600" height="1200" loading="lazy" decoding="async" alt="…">
</picture>
```

`width` và `height` đặt sẵn theo tỉ lệ thật là cách duy nhất giữ CLS < 0,1 (NFR-PERF-05) — đây là lý do API phải trả kích thước ảnh.

---

## 3. Chọn kho lưu trữ: R2, B2, S3 hay MinIO?

### 3.1 Điều quyết định không phải giá lưu trữ, mà là giá truyền ra

Đây là điểm nhiều người bỏ qua và trả giá sau. Với một trang toàn ảnh, **chi phí egress vượt xa chi phí lưu trữ.**

Ví dụ cụ thể: 40 bạn trong lớp mỗi người tải bộ ảnh HD 4 GB về máy → **160 GB truyền ra** trong một đợt.

| Nhà cung cấp | Lưu trữ 24 GB / tháng | 160 GB egress | Tổng đợt đó |
|---|---|---|---|
| **Cloudflare R2** | ~$0,21 | **$0** | **~$0,21** (~5.000đ) |
| **Backblaze B2** | ~$0,14 | Miễn phí tới 3× dung lượng lưu, sau đó ~$0,01/GB. Qua Cloudflare thì miễn phí | ~$0,14–1,5 |
| **AWS S3** | ~$0,55 | ~$14,40 | **~$15** (~380.000đ) |
| **MinIO trên VPS** | Chi phí đĩa VPS | Trong hạn mức băng thông VPS | Xem mục 3.3 |

Con số của S3 không phải lỗi đánh máy. Một đợt cả lớp tải ảnh về là gần 400.000 đồng. Lặp lại vài lần trong năm là hết ngân sách cả năm.

> ⚠️ Giá ở trên là mức tham khảo tại thời điểm viết. **Hãy kiểm tra lại bảng giá hiện hành trước khi quyết định** — nhưng *cấu trúc* của vấn đề thì không đổi: hãy chọn theo giá egress, không phải giá lưu trữ.

### 3.2 Khuyến nghị

```
┌─────────────────────────────────────────────────────────────┐
│  MÔI TRƯỜNG PHÁT TRIỂN (máy cá nhân)                        │
│  → MinIO trong docker-compose                               │
│    Cùng giao thức S3 ⇒ mã nguồn KHÔNG khác một dòng nào     │
│    Chỉ khác endpoint và bật path-style access               │
├─────────────────────────────────────────────────────────────┤
│  MÔI TRƯỜNG THẬT — kho chính                                │
│  → Cloudflare R2, gắn tên miền tuỳ chỉnh img.<domain>       │
│    Egress miễn phí · phục vụ và lưu đệm ngay trên mạng      │
│    Cloudflare · tương thích S3 hoàn toàn                    │
├─────────────────────────────────────────────────────────────┤
│  MÔI TRƯỜNG THẬT — kho sao lưu                              │
│  → Backblaze B2 (hoặc bất kỳ nhà cung cấp KHÁC R2)          │
│    Rẻ nhất cho dữ liệu để nguội · quan trọng là KHÁC nhà    │
│    cung cấp với kho chính                                   │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 Vì sao KHÔNG dùng MinIO cho môi trường thật

MinIO là phần mềm tốt và việc tự vận hành nó có giá trị học thuật thật. Nhưng với dự án này, có một lập luận quyết định:

**20 GB ảnh không phục hồi được, nằm trên đĩa của một VPS duy nhất.**

Đây chính xác là tài sản A1 trong mô hình đe doạ ([`Security`](MemoryBook-Security.md) mục 2.1) — thứ mà toàn bộ nguyên tắc định hướng của dự án được xây dựng để bảo vệ. Đĩa VPS hỏng thì mất sạch. RAID cũng không cứu được, vì RAID không phải sao lưu — nó chống hỏng đĩa, không chống xoá nhầm, không chống mã hoá tống tiền, không chống nhà cung cấp khoá tài khoản.

Kể cả khi tự dựng MinIO, bạn **vẫn phải** đẩy bản sao sang một kho đối tượng bên ngoài. Nghĩa là vẫn trả tiền cho R2 hoặc B2. Vậy tại sao không dùng thẳng nó làm kho chính, và bỏ luôn phần vận hành?

Thêm ba điểm nữa:

| Vấn đề | Chi tiết |
|---|---|
| Băng thông VPS có hạn mức | Gói VPS thường kèm 1–4 TB/tháng. Cả lớp tải album vài lần là chạm trần, vượt trần thì bị bóp băng thông hoặc tính phí |
| Không có CDN | Ảnh phục vụ từ một máy chủ duy nhất, người xem ở xa tải chậm |
| Đĩa phải mua trước | Cần VPS ≥ 100 GB đĩa ngay từ đầu, dù mới có 2 GB ảnh |

**Nhưng vẫn nên dùng MinIO ở môi trường phát triển**, vì nó cho đúng giao thức S3 mà không cần tài khoản đám mây, chạy ngoại tuyến, và xoá đi làm lại thoải mái. Chuyển từ dev sang thật chỉ là đổi ba biến môi trường:

```java
S3Client.builder()
    .endpointOverride(URI.create(endpoint))   // http://localhost:9000  →  https://….r2.cloudflarestorage.com
    .forcePathStyle(pathStyle)                // true cho MinIO         →  false cho R2
    .credentialsProvider(…)
    .region(Region.of(region))                // "us-east-1"            →  "auto"
    .build();
```

### 3.4 Dùng R2 ngay khi chưa có back-end

Back-end còn chưa viết, nhưng bộ ảnh thì cần chia sẻ cho lớp **bây giờ**. Không phải chờ: R2 dùng được độc lập, và làm vậy không phí công — chính cái bucket đó sau này thành kho chính khi API xong.

**Bốn bước:**

```bash
# 1. Tạo bucket trên Cloudflare R2 (bảng điều khiển), lấy Access Key + Secret

# 2. Cấu hình rclone — chọn loại "s3", nhà cung cấp "Cloudflare R2"
rclone config

# 3. Đẩy các phần lên. Nhớ dùng tiền tố khó đoán, xem cảnh báo bên dưới.
rclone copy ./KyYeu-12A1.part1.rar r2:memorybook-public/tai-ve/8f3a9c2e/ --progress

# 4. Gắn tên miền tuỳ chỉnh cho bucket trong phần Settings của R2
#    → https://anh.<tenmien>/tai-ve/8f3a9c2e/KyYeu-12A1.part1.rar
```

Dán đường dẫn thu được vào `KHO_ANH` trong `client/index.html` là xong. Front-end không cần biết ảnh đến từ đâu.

> ⚠️ **Bucket công khai nghĩa là ai có đường dẫn đều tải được.** Không có đăng nhập, không có kiểm soát. Với ảnh của học sinh, ba việc tối thiểu:
>
> 1. **Dùng tiền tố ngẫu nhiên khó đoán** cho đường dẫn (`/tai-ve/8f3a9c2e/…`), đừng để `/anh/lop-12a1/` — dễ đoán thì coi như công khai hoàn toàn. Đây cũng chính là lý do thiết kế chọn khoá theo mã băm nội dung ở mục 7.1.
> 2. **Chặn lập chỉ mục**: thêm `robots.txt` chặn toàn bộ trên tên miền phục vụ ảnh, và đặt header `X-Robots-Tag: noindex, noimageindex` bằng Cloudflare Transform Rules.
> 3. **Cần chặt hơn thì dùng Cloudflare Access** đặt trước tên miền ảnh, hoặc đường dẫn có chữ ký hạn ngắn — nhưng khi đó phải có back-end để ký, tức quay lại thiết kế ở mục 4.

**Hai điều cần biết trước khi bắt đầu:**

- R2 miễn phí 10 GB đầu, nhưng **Cloudflare vẫn yêu cầu thêm phương thức thanh toán** để bật R2. Với 24 GB thì phần vượt khoảng 5.000 đ/tháng — vẫn rẻ hơn Google One một bậc, nhưng cần thẻ.
- Giai đoạn này **chưa có kiểm duyệt, chưa quét mã độc, chưa bỏ EXIF**. Tệp bạn đẩy lên là tệp người ta tải về, nguyên vẹn. Nếu bộ ảnh có ảnh chụp bằng điện thoại thì **toạ độ GPS còn nguyên trong đó** — cân nhắc chạy `exiftool -gps:all= -overwrite_original *.jpg` trước khi nén và tải lên.

**Về sau đổi gì khi back-end xong:** bucket giữ nguyên, chỉ thêm phân vùng `archive/` và `public/`, và `KHO_ANH` trong front-end được thay bằng lời gọi `GET /api/v1/album`. Cấu trúc nút bấm trên giao diện không đổi.

---

## 4. Tải lên 20 MB — không đi qua máy chủ ứng dụng

### 4.1 Vì sao phải đổi thiết kế

Thiết kế ban đầu ở [`HLD`](MemoryBook-HLD.md) mục 4.3 cho ảnh đi qua Spring Boot. Ở mức 15 MB thì tạm được; ở mức 20 MB trở lên thì có bốn vấn đề thật:

1. **Băng thông đi hai lần**: 20 MB vào VPS, rồi 20 MB từ VPS ra kho lưu trữ. Vô ích.
2. **Chiếm dụng tài nguyên lâu**: người dùng 4G tải lên 20 MB mất 20–60 giây; suốt thời gian đó một luồng và một vùng đệm bị giữ.
3. **Rủi ro hết thời gian chờ**: proxy, Cloudflare và Tomcat đều có giới hạn thời gian; mạng di động chập chờn làm hỏng cả lượt tải.
4. **Không tận dụng được**: kho đối tượng vốn sinh ra để nhận tệp lớn trực tiếp, và **nhận dữ liệu vào (ingress) thì miễn phí**.

### 4.2 Luồng tải lên bằng đường dẫn ký sẵn (presigned upload)

```
Trình duyệt        Spring API       Kho đối tượng     Tác vụ nền
    │                   │                  │               │
    │─[1] POST /photos/upload-intent──────▶│               │
    │   { fileName, size, mimeHint }       │               │
    │                   │                  │               │
    │      · giới hạn tần suất + hạn ngạch dung lượng      │
    │      · Turnstile                                     │
    │      · từ chối ngay nếu size > 25 MB                 │
    │      · tạo bản ghi Photo (AWAITING_UPLOAD)           │
    │      · ký URL PUT trỏ vào quarantine/{uploadId}      │
    │        hạn dùng 10 phút                              │
    │                   │                  │               │
    │◀─[2] { photoId, uploadUrl, expiresAt }               │
    │                   │                  │               │
    │═[3] PUT 20 MB THẲNG VÀO KHO ════════▶│               │
    │   (KHÔNG đi qua VPS — thanh tiến trình chạy mượt,    │
    │    và đây là dữ liệu vào nên miễn phí)               │
    │                   │                  │               │
    │─[4] POST /photos/{id}/complete──────▶│               │
    │   { note, takenAt, category, consent }               │
    │                   │                  │               │
    │      · HEAD object → kiểm tra tệp CÓ THẬT và đúng kích thước
    │      · GET Range: bytes=0-31 → đọc magic bytes       │
    │        (chỉ tải 32 byte, không tải cả 20 MB)         │
    │      · từ chối SVG · kiểm tra số điểm ảnh            │
    │      · tính hash → kiểm tra trùng                    │
    │      · INSERT outbox (PROCESS_PHOTO)  ─ cùng giao dịch
    │                   │                  │               │
    │◀─[5] 202 { status: "PENDING" }                       │
    │                   │                  │               │
    │                   │      [6] tác vụ nền: quét, xử lý, sinh biến thể
```

### 4.3 Hệ quả về bảo mật và cách bù

Đổi sang tải trực tiếp có một cái giá: **tệp nằm trong kho trước khi máy chủ kịp kiểm tra nội dung.** Sáu biện pháp bù lại:

| # | Biện pháp | Chi tiết |
|---|---|---|
| 1 | URL ký sẵn hẹp và ngắn hạn | Chỉ cho `PUT`, chỉ đúng một khoá đã định, hạn 10 phút, dùng một lần |
| 2 | Ràng buộc kích thước ngay trong chữ ký | Nếu nhà cung cấp hỗ trợ presigned POST policy thì đặt `content-length-range`. Nếu chỉ có presigned PUT thì ký kèm `Content-Length` và **kiểm lại kích thước thật bằng `HEAD` ở bước [4]** |
| 3 | Vùng cách ly không bao giờ công khai | `quarantine/` không gắn tên miền, không có quyền đọc công khai. Tệp ở đó không ai truy cập được, kể cả người vừa tải lên |
| 4 | Đọc magic bytes bằng yêu cầu Range | `GET Range: bytes=0-31` — kiểm tra loại tệp thật mà chỉ tải về 32 byte |
| 5 | Quy tắc vòng đời tự dọn | Kho tự xoá mọi đối tượng trong `quarantine/` sau 24 giờ → tệp bỏ dở không tích tụ, và kẻ phá hoại không dùng được kho làm nơi chứa tệp |
| 6 | Hạn ngạch theo dung lượng, không chỉ theo số lượt | 10 lượt/giờ là chưa đủ khi mỗi lượt 20 MB. Thêm hạn ngạch **500 MB/IP/ngày** |

Điểm 5 quan trọng hơn vẻ ngoài của nó: không có nó, ai đó có thể xin 100 URL ký sẵn, tải lên 2 GB rác, không bao giờ gọi bước `complete`, và bạn trả tiền lưu trữ cho đống rác đó mãi mãi.

### 4.4 Tải lên nhiều phần (multipart) — khi nào cần

Với 20 MB, tải một lần là chấp nhận được (khoảng 20–60 giây trên 4G). Multipart chỉ đáng làm nếu thực tế cho thấy **tỉ lệ tải lên thất bại cao**, vì nó cho phép tiếp tục từ phần dở dang thay vì làm lại từ đầu.

Khuyến nghị: **làm bản đơn giản trước, đo, rồi mới quyết.** Thêm `retry` phía client với giãn cách tăng dần đã giải quyết phần lớn trường hợp. Nếu sau khi lớp dùng thật mà thấy nhiều lượt hỏng, lúc đó mới chia phần 5 MB bằng S3 multipart.

---

## 5. Xử lý ảnh 24–50 megapixel — vấn đề bộ nhớ

### 5.1 Vì sao thư viện Java thuần không dùng được ở quy mô này

Đây là **điểm sửa quan trọng nhất** so với [`TechStack`](MemoryBook-TechStack.md) mục 4.2, nơi đề xuất Thumbnailator / imgscalr.

Các thư viện đó dựa trên `ImageIO` của Java, và `ImageIO` giải mã ảnh thành một `BufferedImage` **nằm trọn trong bộ nhớ**:

```
Ảnh 24 megapixel  →  24.000.000 điểm ảnh × 4 byte  ≈  96 MB heap
Ảnh 50 megapixel  →  50.000.000 điểm ảnh × 4 byte  ≈  200 MB heap
                                        ── cho MỘT ảnh, chưa tính vùng đệm giải mã
```

Hai ảnh 50 MP xử lý song song là **hơn 400 MB heap**. Trên VPS 4 GB đang chạy PostgreSQL, Redis, ClamAV và JVM, đây không phải rủi ro lý thuyết — nó sẽ hết bộ nhớ.

### 5.2 Giải pháp: libvips

**libvips** xử lý ảnh theo luồng và theo ô, không nạp toàn bộ ảnh vào bộ nhớ. Bộ nhớ nó dùng gần như không phụ thuộc vào kích thước ảnh đầu vào. Với JPEG nó còn dùng được **shrink-on-load**: bộ giải mã JPEG xuất thẳng ra ảnh đã thu nhỏ 1/2, 1/4 hoặc 1/8, nên để tạo bản thumbnail 400px từ ảnh 24 MP, nó **không bao giờ phải giải mã đủ 24 triệu điểm ảnh.**

```bash
# Sinh một biến thể — libvips chọn sẵn đường tối ưu
vipsthumbnail input.jpg \
  --size 1600x1600 \
  --output "medium.avif[Q=55]" \
  --intent perceptual
```

Ba lệnh cần biết:

| Việc | Lệnh |
|---|---|
| Đọc kích thước mà không giải mã | `vipsheader -f width input.jpg` |
| Sinh biến thể (tự xoay theo EXIF) | `vipsthumbnail in.jpg --size 1600x1600 -o out.avif[Q=55]` |
| Bỏ toàn bộ siêu dữ liệu | thêm `--delete` hoặc dùng `vips copy in.jpg out.jpg[strip]` |

`vipsthumbnail` **tự động áp dụng phép xoay theo EXIF Orientation trước khi bỏ siêu dữ liệu** — giải quyết luôn cái bẫy "ảnh dọc chụp bằng điện thoại bị nằm ngang" đã cảnh báo ở LLD mục 9.

### 5.3 Gọi libvips từ Spring Boot cho an toàn

```java
@Component
public class VipsImageProcessor {

    private static final Path VIPS = Path.of("/usr/bin/vipsthumbnail");

    /**
     * QUAN TRỌNG: truyền tham số dạng DANH SÁCH, không bao giờ dựng chuỗi lệnh.
     * Ghép chuỗi có chứa dữ liệu người dùng là lỗ hổng tiêm lệnh hệ điều hành.
     * Ở đây tên tệp do hệ thống sinh, nhưng vẫn không được phép lười.
     */
    public void thumbnail(Path input, Path output, int maxEdge, int quality)
            throws IOException, InterruptedException {

        var cmd = List.of(
            VIPS.toString(),
            input.toString(),
            "--size", maxEdge + "x" + maxEdge,
            "--output", output + "[Q=" + quality + ",strip]",
            "--intent", "perceptual"
        );

        var pb = new ProcessBuilder(cmd);
        pb.redirectErrorStream(true);
        Process p = pb.start();

        // LUÔN đặt thời gian chờ. Một tiến trình con treo mà không có timeout
        // sẽ giữ luồng đó vĩnh viễn.
        if (!p.waitFor(60, TimeUnit.SECONDS)) {
            p.destroyForcibly();
            throw new ImageProcessingTimeoutException(input);
        }
        if (p.exitValue() != 0) {
            throw new ImageProcessingFailedException(readAll(p.getInputStream()));
        }
    }
}
```

Ba chi tiết bắt buộc:

1. **Tham số dạng danh sách**, không ghép chuỗi — chống tiêm lệnh hệ điều hành.
2. **Luôn có thời gian chờ** và `destroyForcibly()` — tiến trình con treo là lỗi rất khó chẩn đoán.
3. **Luôn đọc hết luồng đầu ra** của tiến trình con — nếu không, khi vùng đệm đầy, tiến trình con sẽ bị chặn và treo.

### 5.4 Giới hạn số lượng xử lý song song

```java
@Configuration
public class ImageWorkerConfig {

    /**
     * libvips dùng bộ nhớ NGOÀI heap của JVM. Đặt -Xmx không kiểm soát được nó.
     * Cách duy nhất giữ được mức tiêu thụ trong tầm là giới hạn số ảnh
     * xử lý cùng lúc.
     *
     * VPS 4 GB → 2 luồng.  VPS 8 GB → 4 luồng.
     */
    @Bean("imageWorkerExecutor")
    public Executor imageWorkerExecutor(
            @Value("${app.image.worker-threads:2}") int threads) {
        var ex = new ThreadPoolTaskExecutor();
        ex.setCorePoolSize(threads);
        ex.setMaxPoolSize(threads);
        ex.setQueueCapacity(100);
        ex.setThreadNamePrefix("img-worker-");
        ex.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        return ex;
    }
}
```

Kèm biến môi trường cho libvips trong Dockerfile:

```dockerfile
ENV VIPS_CONCURRENCY=1
# Mỗi tiến trình libvips chỉ dùng 1 luồng. Song song ở mức "nhiều ảnh cùng lúc"
# do Java kiểm soát, không để libvips tự sinh luồng ngoài tầm kiểm soát.
```

### 5.5 Ngân sách bộ nhớ trên VPS

| Thành phần | RAM 4 GB | RAM 8 GB (khuyến nghị) |
|---|---|---|
| PostgreSQL | 512 MB | 1 GB |
| Redis | 256 MB | 512 MB |
| JVM (Spring Boot) | 1 GB heap | 2 GB heap |
| libvips (ngoài heap, 2–4 luồng) | ~400 MB | ~800 MB |
| ClamAV | ~1,2 GB | ~1,2 GB |
| Hệ điều hành + Caddy + giám sát | ~500 MB | ~700 MB |
| **Tổng** | **~3,9 GB — quá sát** | **~6,2 GB — thoải mái** |

Ở mức 20 GB ảnh và ảnh 50 megapixel, **VPS 8 GB là lựa chọn đúng.** Nếu ngân sách chỉ cho 4 GB, xem cách xử lý ClamAV ở mục 8.

---

## 6. Tải album — vấn đề tệp nén 20 GB

### 6.1 Vì sao thiết kế cũ không dùng được

[`LLD`](MemoryBook-LLD.md) mục 4.7 nói "dựng sẵn album.zip hằng đêm". Ở quy mô 20 GB thì:

- Cần 20 GB dung lượng tạm để dựng.
- Trình duyệt trên điện thoại gần như chắc chắn thất bại khi tải một tệp 20 GB.
- Không tiếp tục được khi đứt giữa chừng — phải làm lại từ đầu.
- Nhiều hệ tệp và công cụ giải nén cũ gặp giới hạn ở mức 4 GB.

### 6.2 Thiết kế thay thế

**Bước một — nén bản `large` chứ không phải bản gốc.** Bản `large` 4000px đã bỏ EXIF, khoảng 2,5 MB mỗi ảnh. 1.000 ảnh → **~2,5 GB thay vì 20 GB.** Vừa nhỏ hơn 8 lần, vừa không rò rỉ toạ độ GPS.

**Bước hai — chia theo phân loại:**

```
album-lop-10.zip     ~600 MB
album-lop-11.zip     ~700 MB
album-lop-12.zip     ~800 MB
album-su-kien.zip    ~400 MB
```

Mỗi tệp dưới 1 GB — tải được trên điện thoại, và hỏng một phần thì chỉ tải lại phần đó.

**Bước ba — dùng phương thức `STORED`, không nén:**

```java
var zos = new ZipOutputStream(out);
zos.setMethod(ZipOutputStream.STORED);   // ← KHÔNG nén
```

JPEG và AVIF **đã được nén rồi**. Nén lại bằng Deflate cho kết quả giảm gần như bằng không nhưng tốn đúng lượng CPU đó. Với 1.000 ảnh, chuyển sang `STORED` tiết kiệm hàng phút CPU mỗi lần dựng. (Lưu ý: `STORED` yêu cầu tự đặt `size` và `crc` cho mỗi mục trước khi ghi.)

**Bước bốn — dựng nền, phục vụ bằng đường dẫn ký sẵn:**

```
Tác vụ định kỳ hằng đêm (chỉ chạy khi có ảnh mới được duyệt):
   dựng 4 tệp nén → ghi vào kho đối tượng

GET /api/v1/album?category=GRADE_12
   → 302 chuyển hướng tới URL ký sẵn, hạn 1 giờ
   → người dùng tải THẲNG từ kho: không tốn băng thông VPS,
     hỗ trợ tiếp tục tải dở, và trên R2 thì egress miễn phí
```

**Bước năm — thêm nút tải từng ảnh.** Trong khung xem ảnh, nút "Tải ảnh này (HD)" trỏ thẳng tới `public/{hash}/large.jpg`. Đây là thứ đa số người dùng thực sự cần, và nó không tốn gì để làm.

### 6.3 Nếu ai đó thật sự cần bản gốc 20 MB

Không đưa vào tệp nén. Thay vào đó cấp đường dẫn ký sẵn cho từng ảnh khi quản trị viên duyệt yêu cầu — hiếm khi cần, và giữ được nguyên tắc: **bản gốc trong `archive/` không bao giờ phơi ra công khai.**

---

## 7. Bộ nhớ đệm và phân phối

### 7.1 Khoá đối tượng theo nội dung — một quyết định, ba lợi ích

```
public/{sha256-của-ảnh-gốc}/large.avif
```

| Lợi ích | Vì sao |
|---|---|
| **Chống trùng miễn phí** | Hai người gửi cùng một tấm ảnh → cùng mã băm → cùng khoá → chỉ lưu một lần |
| **Lưu đệm vĩnh viễn an toàn** | Nội dung tại một khoá **không bao giờ đổi**, nên đặt được `Cache-Control: max-age=31536000, immutable` mà không sợ người dùng thấy ảnh cũ |
| **Chạy lại tác vụ vô hại** | Tác vụ xử lý chạy lại ghi đè đúng tệp đó bằng đúng nội dung đó — đáp ứng NFR-REL-06 mà không cần thêm gì |

### 7.2 Cấu hình đệm

| Loại nội dung | Header |
|---|---|
| Biến thể ảnh (khoá theo nội dung) | `Cache-Control: public, max-age=31536000, immutable` |
| Danh sách ảnh (API) | `Cache-Control: public, max-age=60` |
| Thống kê trang chủ | `Cache-Control: public, max-age=300` |
| Trang HTML | `Cache-Control: no-cache` (luôn hỏi lại, nhưng dùng ETag) |

Gắn tên miền tuỳ chỉnh `img.<domain>` cho bucket R2 → ảnh được phục vụ và lưu đệm ngay trên mạng Cloudflare. Sau lần tải đầu tiên, các lượt sau lấy từ máy chủ biên gần người xem nhất, và **không phát sinh thêm lượt đọc nào vào kho.**

### 7.3 Khi gỡ ảnh thì phải xoá đệm

Vì đặt `immutable` với thời hạn một năm, việc gỡ ảnh **bắt buộc** phải gọi API xoá đệm của Cloudflare. Đây là bước đã nhấn mạnh ở [`HLD`](MemoryBook-HLD.md) mục 4.5 — ở quy mô này nó càng quan trọng, vì không xoá đệm thì ảnh "đã gỡ" vẫn phục vụ được suốt một năm.

---

## 8. Quét mã độc ở quy mô này

ClamAV chiếm khoảng 1,2 GB RAM khi chạy ở chế độ nền (`clamd`) — chiếm gần một phần ba VPS 4 GB.

**Ba phương án, xếp theo mức khuyến nghị:**

| Phương án | Cách làm | Đánh đổi |
|---|---|---|
| **A — VPS 8 GB, chạy `clamd` thường trực** | Như thiết kế ban đầu | Tốn thêm ~150.000đ/tháng. Đơn giản nhất, quét ngay lập tức |
| **B — Quét theo lô bằng `clamscan`** | Tác vụ định kỳ mỗi 10 phút, gom các ảnh chờ, chạy `clamscan` một lần rồi thoát | RAM chỉ dùng trong lúc quét. Mỗi lần chạy tốn 15–30 giây nạp cơ sở dữ liệu mẫu. Ảnh chờ thêm tối đa 10 phút — **hoàn toàn chấp nhận được vì dù sao cũng phải chờ người duyệt** |
| **C — Bỏ ClamAV** | Dựa vào 6 lớp phòng thủ còn lại | Rủi ro còn lại chủ yếu là phát tán tệp nhiễm qua album. Phải là quyết định có ý thức, ghi lại |

**Khuyến nghị: phương án B** nếu dùng VPS 4 GB. Nó gần như không mất mát gì về mặt an toàn — ảnh vốn đã phải nằm chờ người duyệt, nên chờ thêm 10 phút để được quét là miễn phí về mặt trải nghiệm.

---

## 9. Sao lưu 20 GB

### 9.1 Vì sao không sao lưu kiểu ảnh chụp toàn bộ

20 GB × 14 bản hằng ngày = **280 GB**. Vừa tốn tiền, vừa vô nghĩa — vì ảnh gốc là **bất biến**: một khi đã tải lên, nó không bao giờ thay đổi.

### 9.2 Chiến lược đúng: tách theo tính chất dữ liệu

| Loại dữ liệu | Tính chất | Cách sao lưu |
|---|---|---|
| Cơ sở dữ liệu | Thay đổi liên tục, dung lượng nhỏ (vài chục MB) | `pg_dump` **hằng đêm**, giữ 14 bản. Rẻ và nhanh |
| Ảnh gốc trong `archive/` | Bất biến, dung lượng lớn | `rclone copy` (**không phải `sync`**) sang nhà cung cấp khác, hằng tuần |
| Biến thể trong `public/` | **Tái tạo được từ ảnh gốc** | Không cần sao lưu. Mất thì chạy lại tác vụ xử lý |

Nhận ra rằng biến thể không cần sao lưu là một tối ưu đáng kể: chỉ phải bảo vệ 20 GB thay vì 24 GB, và quan trọng hơn là bớt được một thứ phải nghĩ.

### 9.3 `copy` chứ không phải `sync` — chi tiết cứu dự án

```bash
# ĐÚNG: chỉ thêm tệp mới, không bao giờ xoá ở đích
rclone copy r2:memorybook-archive b2:memorybook-backup \
  --transfers 4 --checksum

# SAI — TUYỆT ĐỐI KHÔNG DÙNG:
# rclone sync ...
# Xoá nhầm ở kho chính sẽ được "đồng bộ" sang kho sao lưu.
# Bạn vừa nhân đôi một sai lầm thay vì phòng ngừa nó.
```

Kèm hai lớp nữa ở kho sao lưu:

- **Bật lưu phiên bản đối tượng (versioning)** — ghi đè không mất bản cũ.
- **Bật khoá đối tượng (Object Lock) nếu nhà cung cấp hỗ trợ** — kẻ chiếm được máy chủ cũng không xoá được bản sao lưu. Đây là biện pháp chống mã hoá tống tiền quan trọng nhất.

### 9.4 Diễn tập phục hồi ở quy mô này

Phục hồi 20 GB không phải chuyện tức thì. Diễn tập hằng quý nên gồm:

1. Phục hồi cơ sở dữ liệu vào môi trường tạm — đo thời gian thật.
2. Phục hồi **một mẫu 20 ảnh** từ kho sao lưu, mở ra xem, đối chiếu mã băm với bản ghi trong cơ sở dữ liệu.
3. Chạy lại tác vụ sinh biến thể cho 20 ảnh đó — xác nhận đường phục hồi hoàn chỉnh có thật sự chạy được.
4. Ghi biên bản: mất bao lâu, vướng gì.

Không cần phục hồi đủ 20 GB mỗi quý. Nhưng **phải phục hồi thật ít nhất một lần đủ 20 GB** trước ngày công bố, để biết con số RTO thật là bao nhiêu.

---

## 10. Chi phí cập nhật

Thay thế cho [`TechStack`](MemoryBook-TechStack.md) mục 8.4.

| Khoản | Lựa chọn | Chi phí/tháng |
|---|---|---|
| VPS 4 vCPU / 8 GB / 80 GB SSD | Cần cho libvips + ClamAV thường trực | 350.000 – 450.000đ |
| *(hoặc)* VPS 2 vCPU / 4 GB | Kèm phương án B ở mục 8 | 180.000 – 250.000đ |
| Cloudflare R2 — 24 GB | 10 GB đầu miễn phí | ~5.000đ |
| Cloudflare R2 — thao tác | Xa dưới hạn mức miễn phí | 0đ |
| Cloudflare R2 — egress | **Miễn phí** | **0đ** |
| Backblaze B2 sao lưu — 20 GB | | ~4.000đ |
| Cloudflare (CDN, WAF, Turnstile) | Gói miễn phí | 0đ |
| Tên miền | Quy đổi theo tháng | ~25.000 – 60.000đ |
| **Tổng (VPS 8 GB)** | | **~390.000 – 520.000đ** |
| **Tổng (VPS 4 GB, quét theo lô)** | | **~215.000 – 320.000đ** |

Đối chiếu nếu chọn sai kho lưu trữ: cũng khối lượng đó trên **AWS S3**, chỉ riêng một đợt cả lớp tải album (160 GB egress) đã là **~380.000đ** — bằng cả tháng vận hành. Đây là lý do mục 3.1 đặt egress lên đầu.

---

## 11. Danh sách các điểm tối ưu — tóm tắt để đối chiếu khi build

Xếp theo mức ảnh hưởng, cao nhất trước.

| # | Tối ưu | Được gì |
|---|---|---|
| 1 | **Chọn kho có egress miễn phí (R2)** | Chênh vài trăm nghìn đồng mỗi đợt cả lớp tải ảnh |
| 2 | **Tách `archive/` riêng tư và `public/` phục vụ** | Không rò rỉ GPS; tệp nén album nhỏ đi 8 lần; giữ nguyên bản gốc không mất chất lượng |
| 3 | **Tải lên trực tiếp bằng URL ký sẵn** | 20 MB không đi qua VPS; thanh tiến trình chạy đúng; băng thông không tính hai lần |
| 4 | **libvips thay cho thư viện Java thuần** | Không hết bộ nhớ với ảnh 50 MP; nhanh hơn nhiều lần; tự xoay theo EXIF |
| 5 | **Khoá đối tượng theo mã băm nội dung** | Chống trùng + lưu đệm vĩnh viễn + chạy lại tác vụ vô hại, cả ba chỉ từ một quyết định |
| 6 | **Chỉ sao lưu ảnh gốc, không sao lưu biến thể** | Giảm 20% dung lượng và bớt một thứ phải nghĩ |
| 7 | **`rclone copy`, không phải `sync`** | Xoá nhầm không lan sang bản sao lưu |
| 8 | **Tệp nén dùng `STORED`, chia theo phân loại** | Tiết kiệm hàng phút CPU; tải được trên điện thoại |
| 9 | **AVIF + JPEG dự phòng, bỏ WebP ở giữa** | Nhỏ hơn 30–50% so với chỉ dùng JPEG, mà không nhân đôi số tệp |
| 10 | **Giới hạn số ảnh xử lý song song** | Chặn trần bộ nhớ mà `-Xmx` không kiểm soát được |
| 11 | **Quy tắc vòng đời tự xoá `quarantine/` sau 24 giờ** | Không tích tụ rác; kho không bị dùng làm nơi chứa tệp lậu |
| 12 | **Hạn ngạch theo dung lượng, không chỉ theo số lượt** | 10 lượt/giờ vô nghĩa khi mỗi lượt 20 MB |
| 13 | **`vipsheader` đọc kích thước trước khi xử lý** | Chặn bom nén mà không phải giải mã |
| 14 | **Quét mã độc theo lô nếu VPS nhỏ** | Tiết kiệm 1,2 GB RAM gần như không mất gì |
| 15 | **`width`/`height` trong phản hồi API** | CLS < 0,1 — bố cục không nhảy khi ảnh tải xong |

---

## 12. Những gì thay đổi so với tài liệu đã viết

| Tài liệu | Mục | Thay đổi |
|---|---|---|
| [`SRS`](MemoryBook-SRS.md) | FR-UPL-03 | Giới hạn 15 MB → **25 MB** |
| | FR-UPL-08 | 3 kích cỡ × WebP/JPEG → **3 kích cỡ (400/1600/4000) × AVIF/JPEG**, và tách `archive/` |
| | NFR-PERF-03 | Thời gian xử lý tính lại theo libvips |
| [`HLD`](MemoryBook-HLD.md) | 4.3 | Ảnh đi qua API → **tải trực tiếp bằng URL ký sẵn** |
| | 4.7 | Một tệp nén → **bốn tệp nén theo phân loại, từ bản `large`** |
| [`LLD`](MemoryBook-LLD.md) | 5.3 | Thêm điểm cuối `upload-intent` và `complete` |
| | 7.2 | Thumbnailator → **libvips gọi qua tiến trình con** |
| [`TechStack`](MemoryBook-TechStack.md) | 4.2 | Xử lý ảnh chuyển sang libvips là chính |
| | 5 | Thêm bảng so sánh R2 / B2 / S3 / MinIO |
| | 8.4 | Chi phí tính lại cho 24 GB và VPS 8 GB |
| [`Security`](MemoryBook-Security.md) | 4 | Thêm mô hình bảo mật cho tải lên trực tiếp |

Các sửa đổi này **đã được áp dụng vào chính các tài liệu đó**, không chỉ ghi ở đây.

---

*Tài liệu này bổ sung cho [`MemoryBook-TechStack.md`](MemoryBook-TechStack.md) và cập nhật các quyết định liên quan tới lưu trữ ở [`MemoryBook-HLD.md`](MemoryBook-HLD.md) và [`MemoryBook-LLD.md`](MemoryBook-LLD.md). Hướng dẫn dựng thực tế: [`MemoryBook-Build-Guide.md`](MemoryBook-Build-Guide.md).*
