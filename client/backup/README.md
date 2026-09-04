# Các bản lưu của giao diện

Thư mục này giữ lại các phiên bản cũ của `client/index.html` để có thể quay lại bất cứ lúc nào.

| File | Nội dung |
|---|---|
| `index-v2-goc.html` | Bản "gộp + nâng cấp lần 1": đã gộp 6 trang rời thành 1 trang, có GSAP, hoa rơi, mục Lớp mình, nhạc nền, lightbox. |
| `../index.html` | Bản mới nhất đang dùng (v3): thêm tranh động sân trường áo dài, album trượt 3D, trang thơ đánh máy, pháo giấy, hiệu ứng cuộn hai chiều. |

## Cách quay lại bản cũ

Mở PowerShell tại thư mục `client` rồi chạy:

```powershell
Copy-Item .\backup\index-v2-goc.html .\index.html -Force
```

Muốn giữ lại bản mới trước khi quay lui thì sao chép nó ra trước:

```powershell
Copy-Item .\index.html .\backup\index-v3.html -Force
```

## So sánh nhanh hai bản

Chỉ cần mở song song hai file trong trình duyệt:

- Bản cũ: `client/backup/index-v2-goc.html`
- Bản mới: `client/index.html`
