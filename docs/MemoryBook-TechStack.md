# Tech Stack — Công nghệ sử dụng
## Phượng Hồng Memories — Kỷ yếu điện tử lớp 12A1

---

## 1. Nguyên tắc chọn công nghệ

Dự án này có một ràng buộc chi phối mọi lựa chọn: **một người vừa phát triển vừa vận hành, làm song song với việc học.** Vì vậy mỗi công nghệ đưa vào phải trả lời được ba câu:

1. **Nó giải quyết vấn đề nào đang có thật?** Không thêm công nghệ vì "sau này có thể cần".
2. **Ba giờ sáng hệ thống hỏng, một người có sửa nổi không?** Thứ nào cần cả một đội để vận hành thì loại.
3. **Học nó có giá trị lâu dài không?** Ưu tiên thứ phổ biến trong thị trường tuyển dụng Việt Nam và có tài liệu tốt.

Hệ quả của ba nguyên tắc này: stack dưới đây **cố tình nhỏ**. Phần lớn công sức nằm ở việc dùng đúng vài công nghệ, không phải dùng qua loa nhiều công nghệ.

> **Về số phiên bản.** Các con số dưới đây là mốc tham khảo tại thời điểm viết tài liệu. Trước khi khởi tạo dự án, hãy kiểm tra bản LTS hiện hành và bản ổn định mới nhất — đặc biệt với Java và Spring Boot, vì chọn sai dòng LTS là thứ phải trả giá suốt vòng đời dự án.

---

## 2. Bảng tóm tắt

| Tầng | Công nghệ | Trạng thái quyết định |
|---|---|---|
| Front-end | React + TypeScript + Vite | **Định hướng, chưa chốt** |
| Back-end | Java LTS + Spring Boot | **Đã chốt** |
| Cơ sở dữ liệu | PostgreSQL | Đã chốt |
| Bộ nhớ đệm / phiên / giới hạn tần suất | Redis | Đã chốt |
| Lưu trữ ảnh | Cloudflare R2 (hoặc MinIO tự dựng) | Đã chốt hướng, chọn nhà cung cấp sau |
| Biên / CDN / WAF | Cloudflare | Đã chốt |
| Reverse proxy | Caddy | Đã chốt, Nginx là phương án thay |
| Đóng gói & triển khai | Docker + Docker Compose | Đã chốt |
| CI/CD | GitHub Actions | Đã chốt |
| Giám sát | Actuator + Prometheus + Grafana + Loki | Đã chốt |

---

## 3. Front-end — React

> ⚠️ **Đây là phần chưa chốt.** Bản `client/index.html` hiện tại chỉ là bản dựng tạm để xem giao diện: một tệp HTML 3.100 dòng, nạp thư viện từ CDN, không có bước build. Nó **không phải** bản sẽ chạy thật. Toàn bộ mục này là đề xuất cho bản chính thức; nếu sau này đổi sang Next.js, Astro hay gì khác, phần back-end và hợp đồng API **không phải sửa gì** — đó chính là lý do kiến trúc được tách như ở HLD.

### 3.1 Nền tảng

| Công nghệ | Phiên bản tham khảo | Vai trò | Vì sao chọn |
|---|---|---|---|
| **React** | 19.x | Thư viện giao diện | Yêu cầu của chủ dự án; hệ sinh thái lớn nhất, dễ tìm tài liệu và người hỗ trợ |
| **TypeScript** | 5.x, chế độ `strict` | Kiểu tĩnh | Bắt lỗi lúc viết thay vì lúc chạy. Với dự án nối API, kiểu dữ liệu sinh từ OpenAPI là thứ tiết kiệm thời gian nhiều nhất |
| **Vite** | 6.x/7.x | Công cụ build & máy chủ phát triển | Khởi động gần như tức thì, cấu hình tối thiểu. Không cần SSR (xem DD1 ở HLD) nên không cần framework nặng hơn |

**Vì sao Vite mà không phải Next.js?** Lý do lớn nhất người ta chọn Next.js là dựng trang phía máy chủ để tối ưu SEO. Nhưng dự án này **cố tình chặn máy tìm kiếm lập chỉ mục** (FR-PRV-06) vì đây là ảnh riêng tư của học sinh vị thành niên. SEO không phải yêu cầu — nó là thứ bị cấm. Bỏ SSR đi thì:

- Không có tiến trình Node nào chạy trên máy chủ → bớt một bề mặt tấn công và một thứ phải vá lỗi.
- Bản build là tệp tĩnh thuần → Caddy phục vụ trực tiếp, Cloudflare lưu đệm toàn bộ.
- Triển khai chỉ là chép thư mục.

Nếu sau này muốn ảnh xem trước khi chia sẻ link (thẻ Open Graph) khác nhau theo từng ảnh, đó là lúc cân nhắc lại Next.js.

### 3.2 Thư viện

| Nhu cầu | Chọn | Vì sao / Đánh đổi |
|---|---|---|
| Điều hướng | **React Router** 7.x | Chỉ cần hai vùng: trang công khai và `/admin`. Khu quản trị nạp theo yêu cầu (lazy) nên không làm nặng gói của khách xem |
| Trạng thái dữ liệu máy chủ | **TanStack Query** 5.x | Lo hộ cache, tải lại, thử lại, phân trang vô hạn cho thư viện ảnh. Đây là thư viện đáng giá nhất trong danh sách này |
| Trạng thái giao diện | **Zustand** | Rất ít trạng thái toàn cục (khung xem ảnh đang mở, ảnh đang chọn). Redux là thừa cho quy mô này |
| Biểu mẫu + kiểm tra | **React Hook Form** + **Zod** | Ít lần render lại; lược đồ Zod dùng chung cho kiểu TypeScript và kiểm tra dữ liệu. **Lưu ý: kiểm tra ở client chỉ để trải nghiệm dễ chịu, không phải bảo mật** — máy chủ luôn kiểm lại (NFR-SEC-03) |
| Giao diện | **Tailwind CSS** 4.x, **cài qua bước build** | Bản dựng tạm đang dùng Play CDN. Play CDN chỉ dành cho thử nghiệm: nó biên dịch CSS ngay trên trình duyệt (chậm) và cần `script-src` cho phép mã ngoài, phá vỡ CSP nghiêm ngặt ở NFR-SEC-11 |
| Thành phần giao diện | **Radix UI** hoặc **shadcn/ui** | Hộp thoại, khung xem ảnh, menu — được xử lý sẵn phần bẫy tiêu điểm và ARIA, đáp ứng NFR-A11Y-02. Tự viết những thứ này gần như chắc chắn sẽ thiếu khả năng tiếp cận |
| Hoạt ảnh | **GSAP** + ScrollTrigger | Giữ nguyên vì tranh động SVG sân trường và các hiệu ứng cuộn đã được viết bằng GSAP; viết lại là lãng phí. **Kiểm tra lại điều khoản giấy phép trước khi công bố** |
| Băng chuyền ảnh | **Swiper** 11.x | Đã dùng ở bản dựng tạm, chạy tốt. **Embla Carousel** nhẹ hơn nếu muốn giảm dung lượng gói |
| Hiệu ứng gõ chữ | Tự viết bằng `useEffect` | Typed.js là ~10 KB cho một hiệu ứng khoảng 30 dòng mã. Bỏ được một phụ thuộc |
| Pháo giấy | **canvas-confetti** | Nhẹ, không phụ thuộc gì. Giữ lại |
| Phông chữ | **Fontsource** (tự phục vụ) | Bản dựng tạm gọi Google Fonts. Tự phục vụ thì nhanh hơn (bớt một lần bắt tay DNS + TLS), không rò rỉ IP người xem sang máy chủ Google, và giữ được `font-src 'self'` trong CSP |
| Kiểu dữ liệu API | **openapi-typescript** | Sinh kiểu TypeScript thẳng từ tài liệu OpenAPI của Spring Boot. Đổi API mà quên sửa front-end là lỗi biên dịch, không phải lỗi lúc chạy |

### 3.3 Chất lượng mã

| Công cụ | Vai trò |
|---|---|
| **ESLint** 9 (flat config) + `eslint-plugin-jsx-a11y` | Bắt lỗi khả năng tiếp cận ngay khi viết |
| **Prettier** | Định dạng thống nhất, hết tranh cãi về dấu cách |
| **Vitest** + **React Testing Library** | Kiểm thử đơn vị và thành phần |
| **Playwright** | Kiểm thử đầu-cuối cho 3 luồng chính: xem trang, gửi lời nhắn, tải ảnh |
| **rollup-plugin-visualizer** | Xem gói JavaScript to ở đâu — cần để giữ NFR-PERF-06 (< 200 KB) |

### 3.4 Chuyển từ bản dựng tạm sang bản chính thức

| Bản dựng tạm hiện tại | Bản chính thức | Lý do phải đổi |
|---|---|---|
| Tailwind Play CDN | Tailwind qua bước build | Play CDN phá CSP và chậm |
| GSAP / Swiper / Typed.js / confetti từ cdnjs, jsDelivr | Cài qua npm, đóng gói cùng ứng dụng | NFR-SEC-17 — mỗi CDN bên thứ ba là một bên có thể chèn mã vào trang |
| Google Fonts | Fontsource tự phục vụ | Quyền riêng tư + CSP + tốc độ |
| 32 URL ảnh viết cứng | `GET /api/v1/photos` | FR-GAL-01 |
| Mảng `DANH_SACH_LOP`, `DANH_SACH_THAY_CO` trong `<script>` | `GET /api/v1/config/members` | FR-CFG-01 |
| `loadSavedNotes()` đọc `localStorage` | `GET /api/v1/messages` | FR-MSG-01 |
| `saveNote()` ghi `localStorage` | `POST /api/v1/messages` | FR-MSG-02 |
| Biểu mẫu đóng góp chỉ gọi `openSuccess()` | `POST /api/v1/photos` dạng `FormData` | FR-UPL-01 |
| Nút "Tải xuống (HD)" chưa gắn xử lý | `GET /api/v1/album` | FR-GAL-04 |
| Bốn nút phân loại chỉ đổi giao diện | Thêm trường ẩn, gửi kèm `category` | FR-UPL-02 — **hiện đang thiếu** |
| Chưa có ô đồng thuận | Thêm ô bắt buộc đánh dấu | FR-PRV-07 |
| `pkm-name` trong `localStorage` | Giữ nguyên | Tên hiển thị của người xem, không cần lên máy chủ |

Hai điều đáng giữ lại nguyên vẹn từ bản dựng tạm, vì đã làm đúng:

- Lời nhắn dựng bằng `textContent` chứ không phải `innerHTML` → không chèn được HTML. React mặc định làm đúng điều này; chỉ cần **không bao giờ dùng `dangerouslySetInnerHTML`**.
- Hỗ trợ đầy đủ `prefers-reduced-motion` và mọi truy cập `localStorage` đều bọc trong `try/catch`.

---

## 4. Back-end — Spring Boot

### 4.1 Nền tảng

| Công nghệ | Phiên bản tham khảo | Vì sao chọn |
|---|---|---|
| **Java** | Bản LTS hiện hành (21 hoặc 25) | Bắt buộc dùng LTS: dự án này sống nhiều năm với một người bảo trì, không có chỗ cho việc nâng cấp mỗi 6 tháng |
| **Spring Boot** | 3.5.x (hoặc dòng ổn định mới hơn) | Yêu cầu của chủ dự án. Quan trọng hơn: **Spring Security là hệ sinh thái bảo mật trưởng thành nhất trong các framework phổ biến** — đúng trọng tâm của dự án này |
| **Spring Web MVC** | đi kèm | Không dùng WebFlux — xem DD9 ở HLD |
| **Virtual threads** | `spring.threads.virtual.enabled=true` | Tải công việc thuần I/O. Cho gần hết lợi ích của lập trình bất đồng bộ mà mã vẫn tuần tự, dễ đọc, dễ gỡ lỗi |

### 4.2 Thư viện chính

| Nhu cầu | Chọn | Ghi chú |
|---|---|---|
| Truy cập dữ liệu | **Spring Data JPA** + Hibernate | Đủ cho mọi truy vấn của dự án. Truy vấn phức tạp hiếm gặp thì dùng native query **có tham số ràng buộc** |
| Di trú lược đồ | **Flyway** | SQL thuần, đọc hiểu ngay. Liquibase mạnh hơn nhưng XML/YAML thêm một lớp trừu tượng không cần thiết ở đây |
| Bể kết nối | **HikariCP** | Mặc định của Spring Boot. Đặt `maximum-pool-size` khoảng 10 — VPS nhỏ, đặt cao chỉ làm PostgreSQL chậm đi |
| Kiểm tra dữ liệu | **Jakarta Bean Validation** (Hibernate Validator) | `@NotBlank`, `@Size`, `@Pattern` ngay trên DTO |
| Ánh xạ DTO ↔ Entity | **MapStruct** | Sinh mã lúc biên dịch, không phản chiếu lúc chạy. **Nguyên tắc bất di bất dịch: controller không bao giờ nhận hay trả entity** — làm vậy là lộ cột nội bộ và mở đường cho lỗi gán tràn thuộc tính |
| Tài liệu API | **springdoc-openapi** | Sinh Swagger UI và tệp OpenAPI từ mã. **Tắt Swagger UI ở môi trường thật** hoặc đặt sau xác thực |
| Giới hạn tần suất | **Bucket4j** + Redis | Thuật toán token bucket, hỗ trợ sẵn kho phân tán |
| Kho phiên | **Spring Session Data Redis** | Phiên sống sót qua khởi động lại ứng dụng (NFR-REL-07) |
| Bộ nhớ đệm | **Spring Cache** + Redis | `@Cacheable` cho thống kê trang chủ và trang đầu thư viện |
| Kho đối tượng | **AWS SDK for Java v2** | Cloudflare R2 và MinIO đều tương thích giao thức S3 → đổi nhà cung cấp chỉ là đổi cấu hình |
| Xử lý ảnh | **libvips** (`vipsthumbnail`) gọi qua tiến trình con | ⚠️ **Đây là bản sửa so với đề xuất ban đầu.** Ở quy mô ảnh 25 MB / 24–50 megapixel của dự án, `ImageIO` (nền tảng của Thumbnailator và imgscalr) nạp toàn bộ ảnh vào bộ nhớ — khoảng 200 MB heap cho **một** ảnh 50 MP, và sẽ gây hết bộ nhớ trên VPS. libvips xử lý theo luồng, bộ nhớ gần như không phụ thuộc kích thước ảnh, và tự xoay theo EXIF trước khi bỏ siêu dữ liệu. Chi tiết ở [`MemoryBook-Storage-Media.md`](MemoryBook-Storage-Media.md) mục 5 |
| Quét mã độc | **ClamAV** qua giao thức clamd | Chạy trong container riêng |
| TOTP (2FA) | **dev.samstevens.totp** hoặc thư viện TOTP tương đương | Sinh và xác thực mã 6 số, sinh mã QR để quét bằng ứng dụng xác thực |
| Log có cấu trúc | **Logback** + `logstash-logback-encoder` | Log JSON, gắn `traceId` qua MDC |
| Số liệu giám sát | **Spring Boot Actuator** + **Micrometer** | Xuất theo định dạng Prometheus |

### 4.3 Cấu hình tiêu biểu

```yaml
# application.yml — KHÔNG chứa bí mật, mọi bí mật đọc từ biến môi trường
spring:
  threads:
    virtual:
      enabled: true                      # bật virtual threads

  datasource:
    url: ${DATABASE_URL}
    username: ${DATABASE_USER}
    password: ${DATABASE_PASSWORD}
    hikari:
      maximum-pool-size: 10
      leak-detection-threshold: 30000    # cảnh báo khi kết nối bị giữ quá lâu

  jpa:
    hibernate:
      ddl-auto: validate                 # ← BẮT BUỘC. Không bao giờ dùng update/create
                                         #   ở môi trường thật: Flyway là nguồn sự thật
    open-in-view: false                  # ← tắt. Mặc định bật là một cái bẫy:
                                         #   nó giữ kết nối cơ sở dữ liệu suốt vòng đời
                                         #   request và sinh truy vấn ngoài ý muốn
    properties:
      hibernate.jdbc.batch_size: 25

  servlet:
    multipart:
      max-file-size: 15MB
      max-request-size: 16MB

  data:
    redis:
      host: ${REDIS_HOST}
      port: 6379
      password: ${REDIS_PASSWORD}

  session:
    store-type: redis
    timeout: 8h                          # FR-ADM-04

server:
  forward-headers-strategy: native       # tin header chuyển tiếp từ proxy đã cấu hình
  error:
    include-stacktrace: never            # ← NFR-SEC-13
    include-message: never
  tomcat:
    max-http-form-post-size: 16MB

management:
  endpoints:
    web:
      exposure:
        include: health,prometheus       # ← chỉ hai điểm cuối, không phải "*"
  endpoint:
    health:
      show-details: when-authorized
  server:
    port: 9090                           # ← cổng riêng, KHÔNG phơi ra internet

springdoc:
  swagger-ui:
    enabled: ${SWAGGER_ENABLED:false}    # mặc định tắt, chỉ bật khi phát triển
```

Bốn dòng đáng dừng lại:

- `ddl-auto: validate` — nếu để `update`, Hibernate sẽ tự sửa lược đồ theo entity. Nghe tiện, nhưng nó có thể âm thầm làm mất dữ liệu và khiến môi trường thật khác môi trường phát triển theo cách không ai kiểm soát được.
- `open-in-view: false` — mặc định của Spring Boot là `true`, và đây là một trong những mặc định gây hại nhất: nó giữ kết nối cơ sở dữ liệu mở suốt cả vòng đời một request, kể cả trong lúc dựng phản hồi JSON, làm cạn bể kết nối và giấu đi các truy vấn N+1.
- `include-stacktrace: never` — dấu vết ngăn xếp lộ tên lớp, cấu trúc thư mục và phiên bản thư viện. Với người tấn công, đó là bản đồ.
- `management.server.port: 9090` — điểm cuối số liệu giám sát chạy trên cổng riêng, chỉ mở trong mạng nội bộ Docker (FR-OPS-02).

### 4.4 Vì sao Maven

Khuyến nghị **Maven** thay vì Gradle cho dự án này: cấu hình dài dòng hơn nhưng dễ đoán, tài liệu và câu trả lời trên mạng nhiều hơn hẳn, và không có ngôn ngữ kịch bản trong tệp build để mà lạc. Gradle nhanh hơn — nhưng thời gian build không phải nút thắt của dự án này, còn thời gian gỡ lỗi build thì có.

---

## 5. Dữ liệu & Lưu trữ

| Thành phần | Chọn | Vì sao |
|---|---|---|
| Cơ sở dữ liệu | **PostgreSQL 16/17** | Giao dịch ACID, chỉ mục một phần, kiểu `JSONB` cho nhật ký kiểm toán, công cụ sao lưu đáng tin |
| Bộ nhớ tạm | **Redis 7** | Phiên + giới hạn tần suất + cache. Một thành phần cho ba nhu cầu |
| Lưu trữ ảnh (môi trường thật) | **Cloudflare R2** | Tương thích S3, **egress miễn phí**. Với 24 GB ảnh và những đợt cả lớp tải album, đây là yếu tố quyết định — xem bảng so sánh ngay dưới |
| Lưu trữ ảnh (môi trường phát triển) | **MinIO** trong Docker | Cùng giao thức S3 nên mã nguồn không khác một dòng; chạy ngoại tuyến, xoá làm lại thoải mái |
| Kho sao lưu | **Backblaze B2** (hoặc bất kỳ nhà cung cấp khác R2) | Rẻ nhất cho dữ liệu để nguội. Điều quan trọng là **khác nhà cung cấp** với kho chính |

**So sánh nhà cung cấp — với 24 GB lưu trữ và một đợt 160 GB truyền ra** (40 bạn cùng tải album 4 GB):

| Nhà cung cấp | Lưu trữ/tháng | 160 GB egress | Tổng đợt đó |
|---|---|---|---|
| **Cloudflare R2** | ~$0,21 | **$0** | **~$0,21** |
| Backblaze B2 | ~$0,14 | Miễn phí qua Cloudflare | ~$0,14 |
| AWS S3 | ~$0,55 | ~$14,40 | ~$15 (**~380.000đ**) |
| MinIO tự dựng | Chi phí đĩa VPS | Trong hạn mức băng thông VPS | Xem cảnh báo dưới |

> ⚠️ **Vì sao không dùng MinIO cho môi trường thật**: nó đặt 20 GB ảnh không phục hồi được lên đĩa của một VPS duy nhất — đúng tài sản A1 trong mô hình đe doạ. RAID không phải sao lưu. Và dù có tự dựng MinIO, bạn **vẫn phải** đẩy bản sao sang một kho đối tượng bên ngoài, tức là vẫn trả tiền cho R2 hoặc B2. Phân tích đầy đủ ở [`MemoryBook-Storage-Media.md`](MemoryBook-Storage-Media.md) mục 3.
>
> Giá ở trên là mức tham khảo tại thời điểm viết — hãy kiểm tra lại bảng giá hiện hành. Nhưng *cấu trúc* vấn đề thì không đổi: **chọn theo giá egress, không phải giá lưu trữ.**
| Sao lưu | `pg_dump` + nén + mã hoá | Đẩy sang nhà cung cấp **khác** với nơi chạy ứng dụng — sao lưu nằm cùng chỗ với bản gốc thì không phải sao lưu |

**Vì sao không lưu ảnh trong cơ sở dữ liệu:** ảnh trong `bytea` làm bản `pg_dump` phình lên hàng GB, khiến việc sao lưu và phục hồi chậm tới mức không ai làm thường xuyên nữa — và một bản sao lưu không ai chạy là một bản sao lưu không tồn tại. Cơ sở dữ liệu giữ siêu dữ liệu và khoá đối tượng; tệp nằm ở kho đối tượng.

---

## 6. Bảo mật — công cụ

> Phân tích đầy đủ ở [`MemoryBook-Security.md`](MemoryBook-Security.md). Đây chỉ là danh sách công cụ.

| Nhu cầu | Công cụ |
|---|---|
| Xác thực & phân quyền | Spring Security 6 |
| Băm mật khẩu | Argon2id (`Argon2PasswordEncoder`) |
| 2FA | Thư viện TOTP + mã QR |
| Giới hạn tần suất | Bucket4j + Redis |
| Chống bot | Cloudflare Turnstile |
| Quét mã độc tệp tải lên | ClamAV |
| Quét lỗ hổng phụ thuộc | OWASP Dependency-Check hoặc Trivy, chạy trong CI |
| Quét image Docker | Trivy |
| Cập nhật phụ thuộc | Dependabot hoặc Renovate |
| Quản lý bí mật | Biến môi trường + Docker secrets; **không bao giờ đưa vào kho mã** |
| Phát hiện bí mật lỡ commit | gitleaks, chạy như git hook và trong CI |
| Chứng chỉ TLS | Caddy tự động qua Let's Encrypt |
| Tường lửa ứng dụng | Cloudflare WAF |

---

## 7. Kiểm thử

| Mức | Công cụ | Kiểm cái gì |
|---|---|---|
| Đơn vị (Back-end) | JUnit 5 + Mockito + AssertJ | Quy tắc nghiệp vụ, chuyển trạng thái, kiểm tra dữ liệu |
| Tích hợp | **Testcontainers** (PostgreSQL, Redis, ClamAV thật) | Truy vấn thật, giao dịch thật, giới hạn tần suất thật |
| Web | `@WebMvcTest` + Spring Security Test | Phân quyền: đường dẫn quản trị phải trả `401` khi chưa đăng nhập |
| Kiến trúc | **ArchUnit** | Ép quy tắc: controller không được tham chiếu repository; không lớp nào ngoài `photo.processing` được gọi ClamAV |
| Đơn vị (Front-end) | Vitest + React Testing Library | Thành phần và hook |
| Đầu-cuối | Playwright | Ba luồng: xem trang, gửi lời nhắn, tải ảnh lên |
| Tải | k6 | NFR-PERF-08: 100 người xem đồng thời |
| Độ phủ | JaCoCo | Ngưỡng ≥ 70% cho tầng nghiệp vụ (NFR-MAIN-03) |

**Testcontainers là lựa chọn đáng chú ý nhất ở đây.** Dùng H2 trong bộ nhớ để kiểm thử thì nhanh hơn, nhưng H2 không có chỉ mục một phần, không có `JSONB`, không có `FOR UPDATE SKIP LOCKED` — tức là không kiểm thử được đúng những phần khó nhất của dự án. Testcontainers chạy PostgreSQL thật trong Docker, chậm hơn vài giây và đổi lại là kiểm thử nói thật.

---

## 8. Hạ tầng & Triển khai

### 8.1 Kiến trúc Docker Compose (một VPS)

```
┌──────────────────────────────────────────────────────────────┐
│  VPS (2 vCPU · 4 GB RAM · Ubuntu LTS)                        │
│                                                              │
│  ┌────────────┐  cổng 80/443 ra internet                     │
│  │   caddy    │  TLS · header bảo mật · phục vụ tệp tĩnh     │
│  └─────┬──────┘                                              │
│        │ mạng nội bộ Docker (không cổng nào phơi ra ngoài)   │
│  ┌─────▼──────┐  ┌────────────┐  ┌──────────┐  ┌──────────┐ │
│  │    app     │  │  postgres  │  │  redis   │  │  clamav  │ │
│  │Spring Boot │  │            │  │          │  │          │ │
│  └────────────┘  └─────┬──────┘  └──────────┘  └──────────┘ │
│                        │ ổ đĩa gắn ngoài                     │
│  ┌────────────┐  ┌─────▼──────┐  ┌──────────┐               │
│  │ prometheus │  │  grafana   │  │   loki   │               │
│  └────────────┘  └────────────┘  └──────────┘               │
│                                                              │
│  Tác vụ định kỳ: pg_dump hằng đêm → Cloudflare R2            │
└──────────────────────────────────────────────────────────────┘
```

**Nguyên tắc mạng:** chỉ container `caddy` mở cổng ra ngoài. PostgreSQL, Redis, ClamAV, Prometheus, Grafana đều chỉ nghe trên mạng nội bộ Docker. Đây là chi tiết hay bị làm sai — mở cổng 5432 ra internet "cho tiện kết nối bằng công cụ quản trị" là cách nhanh nhất để mất cơ sở dữ liệu.

### 8.2 Dockerfile nhiều tầng

```dockerfile
# ---- Tầng build ----
FROM maven:3-eclipse-temurin-21 AS build
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline -B          # tách bước này để tận dụng cache Docker
COPY src ./src
RUN mvn clean package -DskipTests -B

# ---- Tầng chạy ----
FROM eclipse-temurin:21-jre-jammy
RUN apt-get update \
 && apt-get install -y --no-install-recommends libvips-tools \
 && rm -rf /var/lib/apt/lists/*

# Chạy bằng người dùng thường, KHÔNG phải root
RUN useradd --system --uid 1001 --create-home appuser
USER appuser

WORKDIR /app
COPY --from=build --chown=appuser:appuser /build/target/*.jar app.jar

EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s \
  CMD wget -qO- http://localhost:9090/actuator/health/readiness || exit 1

ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75", "-jar", "app.jar"]
```

Ba điểm đáng chú ý: build tách khỏi chạy nên image cuối không chứa Maven và mã nguồn; chạy bằng người dùng thường nên một lỗ hổng thực thi mã cũng không cho quyền `root` trong container; `MaxRAMPercentage` để JVM biết giới hạn bộ nhớ của container thay vì đoán theo bộ nhớ của cả máy.

### 8.3 CI/CD — GitHub Actions

```
Đẩy mã lên nhánh main
   ↓
[1] Build + kiểm thử đơn vị            (Maven, Vitest)
[2] Kiểm thử tích hợp                  (Testcontainers)
[3] Kiểm tra chất lượng                (ESLint, Checkstyle, ArchUnit)
[4] Quét bảo mật                       ← lỗ hổng nghiêm trọng thì DỪNG (NFR-SEC-15)
      · Trivy quét phụ thuộc và image
      · gitleaks tìm bí mật lỡ commit
[5] Build image Docker, gắn thẻ theo mã commit
[6] Đẩy image lên registry
[7] Triển khai qua SSH: docker compose pull && docker compose up -d
[8] Kiểm tra sức khoẻ; thất bại thì quay lại thẻ trước
```

Bước [4] chặn bản dựng khi có lỗ hổng nghiêm trọng là một quyết định có chủ đích: nếu chỉ cảnh báo, cảnh báo sẽ bị bỏ qua sau đúng hai tuần.

### 8.4 Ước tính chi phí hằng tháng

> Tính cho quy mô thật: **~1.000 ảnh, ảnh gốc tới 25 MB, tổng kho ~24 GB.**

| Khoản | Lựa chọn | Chi phí ước tính |
|---|---|---|
| VPS 4 vCPU / 8 GB / 80 GB SSD | Đủ cho libvips + ClamAV chạy thường trực | 350.000 – 450.000 đ |
| *(phương án tiết kiệm)* VPS 2 vCPU / 4 GB | Kèm quét mã độc theo lô thay vì thường trực | 180.000 – 250.000 đ |
| Cloudflare R2 — 24 GB | 10 GB đầu miễn phí | ~5.000 đ |
| Cloudflare R2 — egress | **Miễn phí** | **0 đ** |
| Backblaze B2 — sao lưu 20 GB | Nhà cung cấp khác với kho chính | ~4.000 đ |
| Cloudflare (CDN, WAF, Turnstile) | Gói miễn phí | 0 đ |
| Tên miền `.com` / `.vn` | Quy đổi theo tháng | ~25.000 – 60.000 đ |
| Chứng chỉ TLS | Let's Encrypt qua Caddy | 0 đ |
| **Tổng (VPS 8 GB)** | | **~390.000 – 520.000 đ/tháng** |
| **Tổng (VPS 4 GB, quét theo lô)** | | **~215.000 – 320.000 đ/tháng** |

Hai cách cắt chi phí, theo thứ tự nên cân nhắc:

1. **Quét mã độc theo lô thay vì thường trực** — chạy `clamscan` mỗi 10 phút cho nhóm ảnh đang chờ, thay vì giữ `clamd` chạy liên tục. Tiết kiệm ~1,2 GB RAM và gần như không mất mát gì, vì ảnh vốn đã phải nằm chờ người duyệt. Chi tiết ở [`MemoryBook-Storage-Media.md`](MemoryBook-Storage-Media.md) mục 8.
2. **Bỏ hẳn ClamAV** — chỉ nên làm sau khi đã đọc phần đánh đổi ở [`MemoryBook-Security.md`](MemoryBook-Security.md) mục 4.5 và ghi lại quyết định.

Điều **không** nên cắt: chi phí sao lưu. Nó là khoản 4.000 đồng bảo vệ toàn bộ giá trị của dự án.

---

## 9. Những gì cố tình KHÔNG dùng

Phần này quan trọng ngang phần danh sách công nghệ ở trên — và là thứ hay được hỏi trong phỏng vấn.

| Không dùng | Vì sao |
|---|---|
| **Kubernetes** | Một người vận hành, một máy chủ, lưu lượng vài trăm lượt xem mỗi ngày. Kubernetes ở đây chỉ thêm chỗ để hỏng và thêm thứ phải học lúc đang cần sửa lỗi |
| **Kiến trúc microservices** | Một miền nghiệp vụ, một người phát triển. Chia nhỏ ra là tự tạo cho mình các cuộc gọi mạng, giao dịch phân tán và bài toán truy vết — trong khi chưa có vấn đề nào cần tới chúng |
| **Kafka / RabbitMQ** | Vài chục tác vụ mỗi ngày. Bảng outbox trong PostgreSQL đã đủ và bớt được một hệ thống phải vận hành, giám sát, sao lưu |
| **Elasticsearch** | Không có nhu cầu tìm kiếm. Nếu sau này cần, tìm kiếm toàn văn của PostgreSQL đủ dùng cho vài nghìn bản ghi |
| **GraphQL** | Số điểm cuối rất ít và ổn định. REST đơn giản hơn để bảo mật (giới hạn tần suất theo điểm cuối, phân quyền theo đường dẫn) và để lưu đệm |
| **JWT lưu ở `localStorage`** | Món quà cho bất kỳ lỗ hổng XSS nào, và không thu hồi được trước hạn. Xem DD2 ở HLD |
| **MongoDB** | Dữ liệu ở đây có quan hệ rõ ràng và cần giao dịch. Chọn cơ sở dữ liệu tài liệu ở đây là chọn sai công cụ |
| **WebFlux** | Xem DD9 ở HLD |
| **Next.js** | Xem mục 3.1 — SEO bị cấm trong dự án này, mà SEO là lý do chính để chọn SSR |
| **Redux** | Rất ít trạng thái toàn cục. TanStack Query đã lo phần trạng thái dữ liệu máy chủ, phần còn lại Zustand xử lý gọn |
| **Công cụ phân tích của bên thứ ba (Google Analytics…)** | Đặt cookie theo dõi trên trang chứa ảnh của người vị thành niên là điều không nên làm (NFR-PRV-02) |
| **Đăng nhập bằng mạng xã hội** | Người đóng góp không cần tài khoản (DD7). Thêm OAuth là thêm phụ thuộc và thêm dữ liệu cá nhân phải bảo vệ, đổi lấy giá trị bằng không |

---

## 10. Lộ trình nâng cấp khi quy mô tăng

Nếu một ngày dự án vượt khỏi quy mô hiện tại, đây là thứ tự nâng cấp — **và cũng là thứ tự không nên làm sớm**:

| Ngưỡng | Việc cần làm |
|---|---|
| > 50 ảnh tải lên mỗi ngày | Tách tiến trình xử lý ảnh ra container riêng, vẫn dùng chung bảng outbox |
| > 10.000 lượt xem mỗi ngày | Tăng thời gian lưu đệm ở Cloudflare; thêm bản sao chỉ đọc cho PostgreSQL |
| > 50.000 ảnh | Chuyển sang phân trang có chỉ mục phủ; cân nhắc phân vùng bảng `photos` theo thời gian |
| Cần chạy nhiều bản ứng dụng | Đã sẵn sàng: ứng dụng không giữ trạng thái, phiên nằm ở Redis (NFR-MAIN-04) |
| Cần dừng-không-gián-đoạn khi triển khai | Thêm bản thứ hai sau Caddy, triển khai luân phiên |
| Cần thay đổi tác vụ nền phức tạp hơn | Lúc đó mới cân nhắc message broker — và đổi được mà không đụng vào logic nghiệp vụ |

---

*Tài liệu này bổ trợ cho [`MemoryBook-HLD.md`](MemoryBook-HLD.md) (kiến trúc) và [`MemoryBook-LLD.md`](MemoryBook-LLD.md) (chi tiết hiện thực). Phần bảo mật được tách riêng ở [`MemoryBook-Security.md`](MemoryBook-Security.md).*
