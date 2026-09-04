# 📚 Tài liệu dự án — Phượng Hồng Memories

> Bộ tài liệu kỹ thuật cho kỷ yếu điện tử lớp 12A1, trường THPT Mạc Đĩnh Chi.
> Front-end **React** (chưa chốt) · Back-end **Spring Boot** (đã chốt) · Bảo mật ở mức triển khai được ra internet công cộng.

---

## Đọc theo thứ tự nào

```
        VÌ SAO                LÀM GÌ              PHẢI LÀM ĐƯỢC GÌ
         BRD      ────────▶    PRD     ────────▶       SRS
                                                        │
                                                        ▼
                            CHI TIẾT RA SAO   ◀────  KIẾN TRÚC
                                 LLD          ◀────     HLD
                                  │                      │
                                  └──────────┬───────────┘
                                             ▼
                       TechStack · Storage & Media · Security
                                             │
                                             ▼
                                 Workflow  →  Build Guide
                                (kế hoạch)     (gõ gì)
```

| # | Tài liệu | Trả lời câu hỏi | Đọc khi nào |
|---|---|---|---|
| 0 | [**BRD**](MemoryBook-BRD.md) | Vì sao làm dự án này? | Khi phân vân không biết có nên làm tính năng nào đó |
| 1 | [**PRD**](MemoryBook-PRD.md) | Làm cái gì, cho ai? | Khi cần biết phạm vi — cái gì trong, cái gì ngoài |
| 2 | [**SRS**](MemoryBook-SRS.md) | Hệ thống phải làm được gì, cụ thể? | Khi nghiệm thu, đối chiếu từng yêu cầu |
| 3 | [**HLD**](MemoryBook-HLD.md) | Kiến trúc trông như thế nào? | Trước khi viết dòng code đầu tiên |
| 4 | [**LLD**](MemoryBook-LLD.md) | Từng phần hoạt động ra sao? | Trong lúc code — và cập nhật ngay sau khi code |
| 5 | [**TechStack**](MemoryBook-TechStack.md) | Dùng công nghệ gì, vì sao? | Khi khởi tạo dự án, khi cân nhắc đổi công nghệ |
| 6 | [**Storage & Media**](MemoryBook-Storage-Media.md) | Lưu 20 GB ảnh ở đâu, xử lý ảnh 25 MB thế nào? | Trước khi làm tính năng ảnh — **bắt buộc** |
| 7 | [**Security**](MemoryBook-Security.md) | Chống lại ai, bằng cách nào? | Trước khi code, và dùng như checklist trước khi công bố |
| 8 | [**Workflow**](MemoryBook-Workflow-Docs.md) | Làm theo trình tự nào? | Khi lập kế hoạch, khi thấy mình đang lan man |
| 9 | [**Build Guide**](MemoryBook-Build-Guide.md) | Gõ gì, theo thứ tự nào? | **Khi ngồi xuống code** — đây là tài liệu cầm tay |

**Chưa biết bắt đầu từ đâu?** → Mở [`Build Guide`](MemoryBook-Build-Guide.md) và làm từ Bước 0.

**Muốn hiểu bức tranh lớn trước?** → [`Workflow`](MemoryBook-Workflow-Docs.md) mục 3 (lộ trình theo giai đoạn).

---

## Tóm tắt trong một trang

### Dự án là gì

Kỷ yếu điện tử cho lớp 12A1: một trang cuộn liền mạch gồm thư viện ảnh, hành trình ba năm, hồ sơ học sinh và thầy cô, bảng lời nhắn chung, biểu mẫu đóng góp ảnh, cùng một tranh động sân trường vẽ hoàn toàn bằng SVG.

### Trạng thái hiện tại

| Phần | Trạng thái |
|---|---|
| [`client/`](../client/) | Bản dựng tạm — một tệp `index.html` 3.100 dòng, thư viện nạp từ CDN, dữ liệu giả lập trong `localStorage`. **Chưa phải bản chạy thật** |
| [`server/`](../server/) | Thư mục trống. Mới có đặc tả trong `server/README.md` |
| [`docs/`](.) | Bộ tài liệu này — ở mức thiết kế, chưa có mã tương ứng |

### Sẽ xây dựng cái gì

```
React SPA (tệp tĩnh)  →  Spring Boot API  →  PostgreSQL + Redis
        │                      ↓
        │            Kho đối tượng: archive/ (bytes gốc, riêng tư)
        └──[ảnh 25MB đi thẳng]─▶ public/  (biến thể, qua CDN img.<domain>)
```

Quy mô thật: **~1.000 ảnh, ảnh gốc tới 25 MB, tổng kho ~24 GB.** Con số này chi phối ba quyết định lớn — ảnh tải lên **không đi qua máy chủ ứng dụng**, xử lý ảnh dùng **libvips** chứ không phải thư viện Java thuần, và kho lưu trữ chọn theo **giá egress** chứ không phải giá lưu trữ. Chi tiết ở [`Storage & Media`](MemoryBook-Storage-Media.md).

Ba đặc điểm định hình toàn bộ thiết kế:

1. **Người đóng góp không cần đăng nhập** — bắt 40 học sinh tạo tài khoản là cách chắc chắn để không ai gửi gì cả. Đổi lại, chi phí chống lạm dụng dồn hết sang máy chủ.
2. **Mọi nội dung phải qua duyệt trước khi hiển thị** — trang mang tên thật của trường và của học sinh.
3. **Đây là ảnh của người vị thành niên** — quyền riêng tư và cơ chế gỡ nội dung là yêu cầu bắt buộc, không phải tính năng thêm.

### Công nghệ

| Tầng | Chọn | Trạng thái |
|---|---|---|
| Front-end | React + TypeScript + Vite | Định hướng, **chưa chốt** |
| Back-end | Java LTS + Spring Boot | Đã chốt |
| Cơ sở dữ liệu | PostgreSQL | Đã chốt |
| Phiên / cache / giới hạn tần suất | Redis | Đã chốt |
| Xử lý ảnh | libvips (qua tiến trình con) | Đã chốt |
| Lưu trữ ảnh — thật | Cloudflare R2 (egress miễn phí) | Đã chốt |
| Lưu trữ ảnh — dev | MinIO trong Docker | Đã chốt |
| Sao lưu | Backblaze B2 (nhà cung cấp khác) | Đã chốt |
| Biên / CDN / WAF | Cloudflare | Đã chốt |
| Triển khai | Docker Compose trên một VPS | Đã chốt |

Chi phí vận hành ước tính: **~215.000 – 520.000 đ/tháng** tuỳ cấu hình VPS (xem [`TechStack`](MemoryBook-TechStack.md) mục 8.4).

---

## Về việc front-end chưa chốt công nghệ

Bản `client/index.html` hiện tại chỉ để xem giao diện, và công nghệ front-end có thể đổi sau. Bộ tài liệu này được viết để chịu được điều đó:

- **Hợp đồng API** ([LLD](MemoryBook-LLD.md) mục 5) là phần ổn định — đổi front-end không đụng tới nó.
- **Toàn bộ logic nghiệp vụ và mọi quyết định bảo mật nằm ở máy chủ.** Front-end được coi là môi trường không đáng tin (xem [HLD](MemoryBook-HLD.md) mục 2, ranh giới B3).
- Yêu cầu ở [PRD](MemoryBook-PRD.md) và [SRS](MemoryBook-SRS.md) viết ở mức **hành vi sản phẩm**, không ràng buộc framework.
- Phần phụ thuộc vào React chỉ nằm ở [TechStack](MemoryBook-TechStack.md) mục 3 — đổi công nghệ thì viết lại đúng mục đó.

---

## Quy ước trong tài liệu

| Ký hiệu | Nghĩa |
|---|---|
| `FR-<MODULE>-<số>` | Yêu cầu chức năng (định nghĩa ở [SRS](MemoryBook-SRS.md) mục 3) |
| `NFR-<NHÓM>-<số>` | Yêu cầu phi chức năng (SRS mục 4) |
| `DD<số>` | Quyết định thiết kế kèm đánh đổi ([HLD](MemoryBook-HLD.md) mục 5) |
| `[CHƯA HIỆN THỰC]` | Phần đã thiết kế nhưng chưa có mã |

---

## Liên kết ngoài thư mục này

- [`../README.md`](../README.md) — chỉ mục toàn kho mã
- [`../client/README.md`](../client/README.md) — hướng dẫn bản dựng tạm hiện tại
- [`../client/DESIGN.md`](../client/DESIGN.md) — hệ thống thiết kế (màu, phông, bo góc)
- [`../server/README.md`](../server/README.md) — đặc tả API viết từ nhu cầu thực tế của front-end
- [`../LICENSE`](../LICENSE) — mã nguồn theo MIT; ảnh và tên cá nhân giữ toàn bộ quyền
