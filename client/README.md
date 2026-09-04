# Phượng Hồng Memories — Front-end

The client application: a single-page digital yearbook for class 12A1, Mạc Đĩnh Chi High School, built as one self-contained HTML file with no build step.

Ứng dụng phía người dùng: trang kỷ yếu điện tử một trang của lớp 12A1, trường THPT Mạc Đĩnh Chi, gói gọn trong một file HTML duy nhất, không cần build.

**[English](#english) · [Tiếng Việt](#tiếng-việt)** · [Project index](../README.md) · [Back-end](../server/README.md) · [Engineering docs](../docs/README.md)

---

> ### ⚠️ Status — please read first / Đọc trước
>
> **EN** — This is a **working preview build**, not the production application. It is one hand-written HTML file with no build step, all data simulated in the browser, and photographs that are temporary AI-generated placeholders. The production front-end will be rewritten in React against a Spring Boot API; that design lives in [`../docs/`](../docs/README.md). This file documents what exists **today**.
>
> **VI** — Đây là **bản dựng tạm đang chạy được**, chưa phải bản chính thức. Toàn bộ là một file HTML viết tay, không có bước build, dữ liệu mô phỏng trong trình duyệt, và ảnh là ảnh tạm do AI sinh. Bản chính thức sẽ viết lại bằng React nối vào API Spring Boot — thiết kế nằm ở [`../docs/`](../docs/README.md). File này mô tả những gì **hiện có**.

---

# English

## Screenshots

<!--
Add screenshots before publishing. A visual project with no images in its README
makes a poor first impression on GitHub.

Suggested: capture the home hero, the schoolyard SVG scene, the gallery lightbox,
and the message board. Save them under client/docs-assets/ and uncomment:

![Home](docs-assets/home.png)
![Schoolyard scene](docs-assets/scene.png)
![Gallery](docs-assets/gallery.png)
-->

> 📸 **Not added yet.** See the comment in the source of this section for what to capture and where to put it.

## Overview

Phượng Hồng Memories is a client-side web application that presents a high-school yearbook as a continuous, scrollable page. It combines a photo archive, a three-year timeline, student and teacher profiles, a community message board, and a photo-contribution form, wrapped in an editorial "digital keepsake" aesthetic defined in [`DESIGN.md`](DESIGN.md).

The entire front-end lives in a single file — [`index.html`](index.html) (about 3,100 lines, 184 KB) — with markup, styles and behaviour co-located. There is no bundler, package manager, or compilation step: open the file and it runs.

## Features

| Section | Anchor | Description |
| --- | --- | --- |
| Trang chủ (Home) | `#home` | Hero banner, animated statistics, bento photo grid, quotation block, and an animated SVG schoolyard scene |
| Hành trình (Journey) | `#journey` | Three-year timeline (Grade 10–12) with a progress line drawn on scroll |
| Thư viện (Gallery) | `#gallery` | 3D coverflow carousel (12 photos) plus a masonry polaroid wall with a full-screen lightbox |
| Lớp mình (Our Class) | `#lop-minh` | Twelve flip cards for students and a teacher tribute block |
| Lời nhắn (Messages) | `#messages` | Typewriter poem on ruled paper and a sticky-note message board persisted in the browser |
| Đóng góp (Contribute) | `#contribute` | Photo upload form with drag-and-drop, live preview, category chips, and a success dialog |

### The schoolyard scene

The centrepiece of the home section is a hand-authored SVG illustration (no raster assets) that runs a looping story:

1. Two students in áo dài chat through comic-style speech bubbles.
2. They walk off to the left.
3. A group of six friends runs across the frame; a second group of four runs the opposite way, crossing mid-frame.
4. The two students walk back in, and the loop repeats.

Running behind them, a couple shares a bicycle and exchanges its own dialogue. The scene reproduces the school's real architecture — terracotta roof, red columns, blue window frames, white balcony railings, and the granite gate bearing the school name — alongside a flamboyant tree (*Delonix regia*) drawn with fern-like compound leaves and orange-red flower clusters.

All scene animation is gated to the viewport: nothing moves until the illustration scrolls into view, everything freezes when it scrolls away, and the story restarts from the beginning on re-entry.

### Interaction details

- **Ambient music** — a music-box melody synthesised at runtime with the Web Audio API; no audio file is required.
- **Visitor name** — stored locally and used to pre-fill the message form.
- **PDF export** — a dedicated print stylesheet hides navigation, particles and dialogs, and prevents cards from splitting across pages.
- **Falling petals** — a canvas particle layer that reduces its particle count on small screens and pauses when the tab is hidden.
- **Reduced motion** — every animation is disabled when `prefers-reduced-motion: reduce` is set.

## Tech stack

No build tooling. All dependencies are loaded from CDNs at pinned versions.

| Dependency | Version | Source | Purpose |
| --- | --- | --- | --- |
| Tailwind CSS | Play CDN | `cdn.tailwindcss.com` | Utility classes with an inline theme configuration |
| GSAP | 3.12.5 | cdnjs | Timeline animation engine |
| GSAP ScrollTrigger | 3.12.5 | cdnjs | Scroll-driven reveals and the viewport gate |
| GSAP MotionPathPlugin | 3.12.5 | cdnjs | Paper-plane flight path |
| Swiper | 11.1.14 | cdnjs | Coverflow photo carousel |
| Typed.js | 2.1.0 | cdnjs | Typewriter effect for the poem |
| canvas-confetti | 1.9.4 | jsDelivr | Confetti bursts |
| Google Fonts | — | fonts.googleapis.com | Noto Serif, Plus Jakarta Sans, Dancing Script, Material Symbols |

> canvas-confetti is loaded from jsDelivr rather than cdnjs because the cdnjs artefact is a CommonJS build that throws `module is not defined` in the browser.

### CDN loading will not survive to production

Every dependency above is fetched from a third-party CDN at runtime. That is fine for a preview build and unacceptable for the deployed site, for two reasons: each CDN is a party that can inject arbitrary code into the page, and loading remote scripts makes a strict `Content-Security-Policy` impossible. The production build bundles all of these instead. See [`../docs/MemoryBook-TechStack.md`](../docs/MemoryBook-TechStack.md) §3.4 for the migration table, and [`../docs/MemoryBook-Security.md`](../docs/MemoryBook-Security.md) §6 for the CSP this enables.

## Project structure

```
MemoryBook/
├── README.md                 Repository index
├── docs/                     Engineering documents (design for the production build)
├── server/                   Back-end — not implemented yet
└── client/                   ← you are here
    ├── README.md             This file
    ├── index.html            The entire application (~3,100 lines)
    ├── DESIGN.md             Design system specification
    └── backup/
        ├── README.md         How to roll back
        ├── index-v2-goc.html Earlier revision
        └── index-v3.html     Snapshot of the current revision
```

The back-end that will eventually replace the browser-only storage is specified in [`../server/README.md`](../server/README.md); its full design is in [`../docs/`](../docs/README.md).

## Getting started

The page requires an internet connection for the CDN libraries, fonts and photographs.

Opening `index.html` directly from disk works. Serving it over HTTP is recommended, because `file://` origins restrict clipboard and Web Share access:

```bash
# Python 3
python -m http.server 5173

# or Node.js
npx serve .
```

Then open `http://localhost:5173`.

## Customisation

### Class roster and teachers

Near the top of the `<script>` block in `index.html`:

```js
var DANH_SACH_LOP = [
  { ten: 'Nguyễn Bảo An', nick: 'An Mèo', emoji: '🐱', loi: 'Ngủ gật trong giờ là nghệ thuật…' },
  // …
];

var DANH_SACH_THAY_CO = [
  { ten: 'Cô Nguyễn Thu Hà', mon: 'Chủ nhiệm · Ngữ Văn', emoji: '🌺', loi: '"Các em cứ bay đi…"' },
  // …
];
```

Cards are generated from these arrays; add or remove entries freely. Avatar colours cycle through a six-tone palette automatically.

### Photo archive links

**Tải bộ ảnh gốc** in the gallery header opens a dialog listing the archive parts. It reads from one object at the top of the `<script>` block:

```js
var KHO_ANH = {
  xem: '',        // optional: an external viewer, e.g. a Google Photos album
  thuMuc: '',     // link to wherever all the parts live
  cacPhan: [      // the individual parts
    { ten: 'KyYeu-12A1.part1.rar', dungLuong: '10 GB', link: '' },
    …
  ]
};
```

The archive is planned to live on **Cloudflare R2**, served from its own subdomain — see [`../docs/MemoryBook-Storage-Media.md`](../docs/MemoryBook-Storage-Media.md). The field only holds URLs, so any host works.

Leaving fields empty is safe and degrades in three different ways, deliberately:

| Empty field | What happens |
| --- | --- |
| every `link` in `cacPhan` | The dialog says the archive is being prepared; the parts list, the multi-part warning and the folder button all hide |
| `thuMuc` | The "open the folder" button hides |
| `xem` | The **Xem album ngoài** button is removed from the page entirely, rather than left as a dead control |

Parts with no `link` are skipped, so they can be published one at a time as uploads finish.

`xem` is normally left empty. It exists only for the case where photographs are *also* published somewhere with its own browsing interface — R2 is object storage, not a gallery, and the Thư viện section of this page is the viewer.

One warning worth keeping in the dialog: a multi-part archive needs **every part in one folder** before it will extract. Missing one part breaks the whole set, and that is the failure people hit most often. Phones generally cannot extract RAR at all without an extra app, which is why viewing on the page comes first and downloading is for people on a computer.

### Dialogue and text

| Variable | Controls |
| --- | --- |
| `CHAT` | Dialogue between the two standing students |
| `BIKE_CHAT` | Dialogue between the couple on the bicycle |
| `VERSES` | Verses shown by the typewriter effect on the ruled-paper page |

The running group's dialogue is authored directly in the SVG markup under the identifiers `group-bubble-1`, `group-bubble-2` and `group-bubble-3`.

### School identity

The gate sign lives in the SVG group `school-gate`. The class and cohort appear in the hero badge and in the statistics row of the home section.

### Photographs

All 32 photographs currently reference `lh3.googleusercontent.com/aida-public/…`. These are temporary AI-generated placeholders, and their contents can change without notice — one has already been observed to change into an unrelated image. Replace them with the class's own photographs before publishing.

### Theme

Colours, radii and font families are declared in the inline `tailwind.config` object in the document head. The palette follows the Material-inspired tonal system documented in `DESIGN.md`.

### Illustration assets

Four reusable SVG symbols are defined at the top of `<body>`: `phuong-flower` (flowering branch), `phuong-bloom` (single blossom), `phuong-frond` (compound leaf) and `phuong-cluster` (flower cluster). Reference them anywhere with `<use href="#phuong-bloom" />`.

## Data and privacy

The application has no backend. Two keys are written to `localStorage`:

| Key | Content |
| --- | --- |
| `pkm-name` | The visitor's display name |
| `pkm-notes` | The most recent 40 messages submitted from the board |

Nothing is transmitted to a server. Messages are visible only in the browser that created them, and uploaded photographs are previewed locally and never leave the device. All storage access is wrapped in `try/catch` so the page still works in private-browsing modes.

## Browser support and accessibility

Targets current versions of Chrome, Edge, Firefox and Safari. Required platform features include CSS 3D transforms, `backdrop-filter`, `IntersectionObserver`, the Web Audio API and SVG `<use href>`.

Accessibility measures in place:

- Descriptive `alt` text on content images; decorative images use `alt=""` and decorative SVGs use `aria-hidden="true"`.
- Full support for `prefers-reduced-motion: reduce`.
- Keyboard access for the gallery lightbox (Enter, Escape, arrow keys) and the student flip cards.
- Message content is inserted with `textContent` rather than `innerHTML`, so user input cannot inject markup.

## Known limitations

- Messages persist per browser only; a shared, permanent board requires the back-end specified in [`../server/README.md`](../server/README.md).
- The photograph URLs are temporary placeholders (see [Photographs](#photographs)).
- The photo archive is hosted on object storage (Cloudflare R2) and linked from the gallery; the page does not serve the files itself. PDF export is handled through the browser's print dialog.
- On viewports narrower than 768 px the scene switches to a cropped `viewBox`, so the school building sits outside the visible area.

## Versioning

`backup/` holds previous revisions. To roll back:

```powershell
Copy-Item .\backup\index-v2-goc.html .\index.html -Force
```

See `backup/README.md` for details.

## Roadmap

This preview build is superseded by a production front-end. The plan, in order:

| Phase | Work | Reference |
| --- | --- | --- |
| 1 | Wire the message board to a real API | [`docs` §Build Guide step 4](../docs/MemoryBook-Build-Guide.md) |
| 2 | Wire photo contribution to presigned direct upload | [`docs` §Build Guide step 7](../docs/MemoryBook-Build-Guide.md) |
| 3 | Rewrite in React + TypeScript + Vite, bundle all dependencies | [`docs` §TechStack 3.4](../docs/MemoryBook-TechStack.md) |
| 4 | Replace all 32 placeholder photographs with the class's own | — |

The front-end technology is **not final**. Everything the back-end guarantees is expressed as an API contract, so the framework can change without touching the server.

## Credits

Design system: [`DESIGN.md`](DESIGN.md). Photographs are placeholders pending replacement with the class's own images. Fonts are served by Google Fonts under the SIL Open Font License.

## License

See [`../LICENSE`](../LICENSE). Code and content are treated differently here, deliberately:

- **Code** — `index.html`, the stylesheets and the hand-authored SVG scene are MIT licensed.
- **Content** — photographs of students, personal names, nicknames and the school's identity are **excluded** and all rights are reserved. These are personal data belonging to real people, most of whom were minors when photographed.

⚠️ The licence file still has a placeholder for the copyright holder's name — fill it in before publishing. Privacy obligations are in [`../docs/MemoryBook-Security.md`](../docs/MemoryBook-Security.md) §8.

---

# Tiếng Việt

## Ảnh chụp màn hình

<!--
Thêm ảnh chụp màn hình trước khi công bố. Một dự án về giao diện mà README
không có ảnh nào thì gây ấn tượng rất kém trên GitHub.

Nên chụp: phần hero trang chủ, tranh động sân trường, khung xem ảnh,
và bảng lời nhắn. Lưu vào client/docs-assets/ rồi bỏ dấu chú thích:

![Trang chủ](docs-assets/home.png)
![Tranh sân trường](docs-assets/scene.png)
![Thư viện ảnh](docs-assets/gallery.png)
-->

> 📸 **Chưa có.** Xem phần chú thích trong mã nguồn của mục này để biết cần chụp gì và đặt ở đâu.

## Giới thiệu

Phượng Hồng Memories là một ứng dụng web chạy hoàn toàn phía trình duyệt, trình bày cuốn kỷ yếu cấp ba dưới dạng một trang cuộn liền mạch. Trang gồm thư viện ảnh, hành trình ba năm, hồ sơ học sinh và thầy cô, bảng lời nhắn, cùng biểu mẫu đóng góp ảnh — tất cả theo phong cách "cuốn sổ lưu niệm số" được mô tả trong [`DESIGN.md`](DESIGN.md).

Toàn bộ phần front-end nằm trong một file duy nhất — [`index.html`](index.html) (khoảng 3.100 dòng, 184 KB) — chứa cả HTML, CSS và JavaScript. Không dùng bundler, không cần cài package, không có bước biên dịch: mở file là chạy.

## Tính năng

| Mục | Neo | Mô tả |
| --- | --- | --- |
| Trang chủ | `#home` | Banner mở đầu, dải thống kê đếm số, lưới ảnh bento, khối trích dẫn và tranh động sân trường vẽ bằng SVG |
| Hành trình | `#journey` | Dòng thời gian ba năm (lớp 10–12), đường kẻ tự vẽ dần theo thao tác cuộn |
| Thư viện | `#gallery` | Băng chuyền ảnh 3D (12 ảnh) và tường ảnh polaroid có chế độ xem toàn màn hình |
| Lớp mình | `#lop-minh` | Mười hai thẻ học sinh lật được và khối tri ân thầy cô |
| Lời nhắn | `#messages` | Trang thơ gõ chữ trên giấy kẻ ô và bảng lời nhắn lưu ngay trên trình duyệt |
| Đóng góp | `#contribute` | Biểu mẫu tải ảnh có kéo thả, xem trước, chọn phân loại và hộp thoại xác nhận |

### Tranh động sân trường

Điểm nhấn của trang chủ là một tranh SVG vẽ tay hoàn toàn (không dùng ảnh bitmap), chạy theo một kịch bản lặp:

1. Hai bạn nữ mặc áo dài trò chuyện qua bong bóng thoại kiểu truyện tranh.
2. Hai bạn đi bộ sang trái rồi khuất khỏi khung.
3. Một nhóm sáu bạn chạy ngang sân; nhóm thứ hai gồm bốn bạn chạy ngược chiều, hai nhóm gặp nhau giữa sân.
4. Hai bạn nữ đi bộ trở lại và kịch bản lặp lại.

Phía sau, một đôi bạn chở nhau trên xe đạp và có phần thoại riêng. Bối cảnh tái hiện kiến trúc thật của trường — mái ngói cam, cột đỏ, cửa sổ xanh, lan can trắng và cổng đá granit mang tên trường — cùng cây phượng vĩ (*Delonix regia*) với lá kép lông chim và các chùm hoa cam đỏ.

Toàn bộ hoạt ảnh của tranh chỉ chạy khi tranh nằm trong khung nhìn: chưa lướt tới thì đứng yên, lướt qua chỗ khác thì dừng lại, và quay lại thì kể lại từ đầu.

### Chi tiết tương tác

- **Nhạc nền** — giai điệu hộp nhạc được tổng hợp trực tiếp bằng Web Audio API, không cần file âm thanh.
- **Tên người xem** — lưu tại máy và tự điền sẵn vào biểu mẫu gửi lời nhắn.
- **Xuất PDF** — có style riêng cho bản in, ẩn thanh điều hướng, hiệu ứng hạt và hộp thoại, đồng thời tránh cắt ngang các thẻ nội dung.
- **Hoa phượng rơi** — lớp hạt vẽ trên canvas, tự giảm số lượng trên màn hình nhỏ và tạm dừng khi người dùng chuyển tab.
- **Giảm chuyển động** — mọi hiệu ứng tự tắt khi hệ điều hành bật `prefers-reduced-motion: reduce`.

## Công nghệ sử dụng

Không có công cụ build. Mọi thư viện đều nạp từ CDN với phiên bản cố định.

| Thư viện | Phiên bản | Nguồn | Vai trò |
| --- | --- | --- | --- |
| Tailwind CSS | Play CDN | `cdn.tailwindcss.com` | Lớp tiện ích, cấu hình theme đặt ngay trong trang |
| GSAP | 3.12.5 | cdnjs | Bộ máy hoạt ảnh theo dòng thời gian |
| GSAP ScrollTrigger | 3.12.5 | cdnjs | Hiệu ứng theo cuộn và cổng chặn theo khung nhìn |
| GSAP MotionPathPlugin | 3.12.5 | cdnjs | Đường bay của máy bay giấy |
| Swiper | 11.1.14 | cdnjs | Băng chuyền ảnh kiểu coverflow |
| Typed.js | 2.1.0 | cdnjs | Hiệu ứng gõ chữ cho trang thơ |
| canvas-confetti | 1.9.4 | jsDelivr | Hiệu ứng pháo giấy |
| Google Fonts | — | fonts.googleapis.com | Noto Serif, Plus Jakarta Sans, Dancing Script, Material Symbols |

> canvas-confetti nạp từ jsDelivr thay vì cdnjs, vì bản trên cdnjs là bản CommonJS và sẽ báo lỗi `module is not defined` khi chạy trên trình duyệt.

### Cách nạp từ CDN sẽ không giữ được ở bản chính thức

Mọi thư viện ở trên đều tải từ CDN của bên thứ ba lúc chạy. Với bản dựng tạm thì không sao, nhưng với bản triển khai thật thì không chấp nhận được, vì hai lý do: mỗi CDN là một bên có thể chèn mã tuỳ ý vào trang, và nạp mã từ xa khiến không thể đặt được `Content-Security-Policy` nghiêm ngặt. Bản chính thức đóng gói tất cả cùng ứng dụng. Bảng đối chiếu chuyển đổi ở [`../docs/MemoryBook-TechStack.md`](../docs/MemoryBook-TechStack.md) mục 3.4; chính sách CSP tương ứng ở [`../docs/MemoryBook-Security.md`](../docs/MemoryBook-Security.md) mục 6.

## Cấu trúc dự án

```
MemoryBook/
├── README.md                 Chỉ mục kho mã
├── docs/                     Tài liệu kỹ thuật (thiết kế cho bản chính thức)
├── server/                   Back-end — chưa hiện thực
└── client/                   ← bạn đang ở đây
    ├── README.md             File này
    ├── index.html            Toàn bộ ứng dụng (~3.100 dòng)
    ├── DESIGN.md             Đặc tả hệ thống thiết kế
    └── backup/
        ├── README.md         Hướng dẫn quay lui phiên bản
        ├── index-v2-goc.html Phiên bản cũ
        └── index-v3.html     Bản lưu của phiên bản hiện tại
```

Phần back-end sẽ thay thế cơ chế lưu tạm trên trình duyệt được đặc tả tại [`../server/README.md`](../server/README.md); thiết kế đầy đủ nằm ở [`../docs/`](../docs/README.md).

## Chạy dự án

Trang cần kết nối internet để tải thư viện CDN, phông chữ và ảnh.

Mở thẳng `index.html` là chạy được. Tuy nhiên nên phục vụ qua HTTP, vì giao thức `file://` hạn chế quyền truy cập clipboard và Web Share:

```bash
# Python 3
python -m http.server 5173

# hoặc Node.js
npx serve .
```

Sau đó mở `http://localhost:5173`.

## Tuỳ chỉnh nội dung

### Danh sách lớp và thầy cô

Nằm ở đầu khối `<script>` trong `index.html`:

```js
var DANH_SACH_LOP = [
  { ten: 'Nguyễn Bảo An', nick: 'An Mèo', emoji: '🐱', loi: 'Ngủ gật trong giờ là nghệ thuật…' },
  // …
];

var DANH_SACH_THAY_CO = [
  { ten: 'Cô Nguyễn Thu Hà', mon: 'Chủ nhiệm · Ngữ Văn', emoji: '🌺', loi: '"Các em cứ bay đi…"' },
  // …
];
```

Các thẻ được sinh ra từ hai mảng này; thêm hoặc bớt tuỳ ý. Màu avatar tự luân phiên theo bảng sáu tông.

### Liên kết kho ảnh

Nút **Tải bộ ảnh gốc** ở đầu mục Thư viện mở một hộp thoại liệt kê các phần của bộ ảnh. Nó đọc từ một đối tượng duy nhất đặt ở đầu khối `<script>`:

```js
var KHO_ANH = {
  xem: '',        // tuỳ chọn: trang xem ảnh bên ngoài, ví dụ album Google Photos
  thuMuc: '',     // link tới chỗ chứa tất cả các phần
  cacPhan: [      // từng phần
    { ten: 'KyYeu-12A1.part1.rar', dungLuong: '10 GB', link: '' },
    …
  ]
};
```

Kho ảnh dự kiến đặt trên **Cloudflare R2**, phục vụ qua một tên miền phụ riêng — xem [`../docs/MemoryBook-Storage-Media.md`](../docs/MemoryBook-Storage-Media.md). Ở đây chỉ nhận URL nên dùng nguồn nào cũng được.

Để trống vẫn an toàn, và ứng xử khác nhau tuỳ trường hợp — đây là chủ đích:

| Trường để trống | Điều gì xảy ra |
| --- | --- |
| tất cả `link` trong `cacPhan` | Hộp thoại báo kho ảnh đang được chuẩn bị; danh sách phần, cảnh báo nhiều phần và nút mở thư mục đều ẩn |
| `thuMuc` | Nút "Mở thư mục chứa tất cả" ẩn đi |
| `xem` | Nút **Xem album ngoài** bị gỡ hẳn khỏi trang, thay vì để lại một nút bấm không làm gì |

Phần nào chưa có `link` thì bị bỏ qua, nên công bố dần từng phần theo tiến độ tải lên cũng được.

`xem` thường để trống. Nó chỉ dùng khi bạn *đồng thời* đăng ảnh ở một nơi có giao diện xem riêng — R2 là kho đối tượng chứ không phải trang xem ảnh, và mục Thư viện ngay trên trang này mới là nơi xem.

Một cảnh báo nên giữ trong hộp thoại: bộ nén nhiều phần cần **đủ tất cả các phần trong cùng một thư mục** mới giải nén được. Thiếu một phần là hỏng cả bộ, và đây là lỗi người ta hay mắc nhất. Điện thoại nhìn chung không giải nén được RAR nếu không cài thêm ứng dụng — đó là lý do việc xem ngay trên trang được đặt lên trước, còn tải về là dành cho người dùng máy tính.

### Lời thoại và câu chữ

| Biến | Điều khiển |
| --- | --- |
| `CHAT` | Lời thoại của hai bạn nữ đứng trò chuyện |
| `BIKE_CHAT` | Lời thoại của đôi bạn đi xe đạp |
| `VERSES` | Các khổ thơ hiện ra theo hiệu ứng gõ chữ |

Lời thoại của nhóm bạn chạy được viết trực tiếp trong SVG, tại các nhóm mang định danh `group-bubble-1`, `group-bubble-2` và `group-bubble-3`.

### Thông tin trường lớp

Bảng tên trên cổng nằm trong nhóm SVG `school-gate`. Tên lớp và niên khóa xuất hiện ở huy hiệu đầu trang và dải thống kê của trang chủ.

### Hình ảnh

Cả 32 ảnh hiện dùng đường dẫn `lh3.googleusercontent.com/aida-public/…`. Đây là ảnh tạm do AI sinh ra, nội dung có thể bị thay đổi bất cứ lúc nào — thực tế đã có một ảnh bị đổi thành ảnh không liên quan. Hãy thay bằng ảnh thật của lớp trước khi công bố.

### Giao diện

Màu sắc, bo góc và bộ phông được khai báo trong đối tượng `tailwind.config` đặt ở phần đầu tài liệu. Bảng màu tuân theo hệ tông màu kiểu Material được mô tả trong `DESIGN.md`.

### Hoạ tiết vẽ sẵn

Bốn mẫu SVG dùng lại được khai báo ở đầu thẻ `<body>`: `phuong-flower` (cành hoa), `phuong-bloom` (một bông), `phuong-frond` (lá kép) và `phuong-cluster` (chùm hoa). Gọi lại ở bất kỳ đâu bằng `<use href="#phuong-bloom" />`.

## Dữ liệu và quyền riêng tư

Ứng dụng không có máy chủ. Hai khoá được ghi vào `localStorage`:

| Khoá | Nội dung |
| --- | --- |
| `pkm-name` | Tên hiển thị của người xem |
| `pkm-notes` | 40 lời nhắn gần nhất gửi từ bảng lời nhắn |

Không có dữ liệu nào được gửi lên máy chủ. Lời nhắn chỉ hiển thị trên chính trình duyệt đã tạo ra chúng, và ảnh tải lên chỉ được xem trước tại máy, không rời khỏi thiết bị. Mọi thao tác lưu trữ đều bọc trong `try/catch` để trang vẫn chạy bình thường ở chế độ ẩn danh.

## Trình duyệt hỗ trợ và khả năng tiếp cận

Hỗ trợ các phiên bản hiện hành của Chrome, Edge, Firefox và Safari. Trang cần các tính năng: CSS 3D transform, `backdrop-filter`, `IntersectionObserver`, Web Audio API và SVG `<use href>`.

Các biện pháp tiếp cận đã áp dụng:

- Ảnh nội dung có `alt` mô tả; ảnh trang trí dùng `alt=""` và SVG trang trí dùng `aria-hidden="true"`.
- Hỗ trợ đầy đủ `prefers-reduced-motion: reduce`.
- Điều khiển bằng bàn phím cho khung xem ảnh (Enter, Escape, phím mũi tên) và thẻ học sinh lật.
- Nội dung lời nhắn được chèn bằng `textContent` thay vì `innerHTML`, nên người dùng không thể chèn mã HTML.

## Hạn chế hiện tại

- Lời nhắn chỉ lưu trên từng trình duyệt; muốn có bảng lời nhắn chung và lâu dài thì cần back-end được đặc tả tại [`../server/README.md`](../server/README.md).
- Đường dẫn ảnh là ảnh tạm (xem mục [Hình ảnh](#hình-ảnh)).
- Kho ảnh đặt trên kho đối tượng (Cloudflare R2) và chỉ liên kết từ mục Thư viện; trang không tự phục vụ tệp. Việc xuất PDF đi qua hộp thoại in của trình duyệt.
- Với màn hình hẹp dưới 768 px, tranh động chuyển sang `viewBox` cắt hẹp nên dãy nhà trường nằm ngoài vùng nhìn thấy.

## Quản lý phiên bản

Thư mục `backup/` lưu các phiên bản trước. Cách quay lui:

```powershell
Copy-Item .\backup\index-v2-goc.html .\index.html -Force
```

Chi tiết xem `backup/README.md`.

## Lộ trình

Bản dựng tạm này sẽ được thay bằng bản chính thức. Thứ tự làm:

| Giai đoạn | Việc | Tham chiếu |
| --- | --- | --- |
| 1 | Nối bảng lời nhắn vào API thật | [`docs` — Build Guide bước 4](../docs/MemoryBook-Build-Guide.md) |
| 2 | Nối đóng góp ảnh vào luồng tải lên bằng đường dẫn ký sẵn | [`docs` — Build Guide bước 7](../docs/MemoryBook-Build-Guide.md) |
| 3 | Viết lại bằng React + TypeScript + Vite, đóng gói mọi thư viện | [`docs` — TechStack 3.4](../docs/MemoryBook-TechStack.md) |
| 4 | Thay toàn bộ 32 ảnh tạm bằng ảnh thật của lớp | — |

Công nghệ front-end **chưa chốt cuối**. Mọi thứ back-end đảm bảo đều được diễn đạt qua hợp đồng API, nên đổi framework không phải đụng vào máy chủ.

## Ghi nhận

Hệ thống thiết kế: [`DESIGN.md`](DESIGN.md). Ảnh hiện tại là ảnh tạm, cần thay bằng ảnh thật của lớp. Phông chữ do Google Fonts cung cấp theo giấy phép SIL Open Font License.

## Giấy phép

Xem [`../LICENSE`](../LICENSE). Mã nguồn và nội dung được đối xử khác nhau, một cách có chủ đích:

- **Mã nguồn** — `index.html`, phần style và tranh SVG vẽ tay theo giấy phép MIT.
- **Nội dung** — ảnh học sinh, tên và biệt danh cá nhân, danh tính nhà trường **nằm ngoài phạm vi** giấy phép và giữ toàn bộ quyền. Đây là dữ liệu cá nhân của người thật, phần lớn còn ở tuổi vị thành niên tại thời điểm chụp.

⚠️ File giấy phép vẫn còn chỗ trống cho tên chủ sở hữu bản quyền — nhớ điền trước khi công bố. Các nghĩa vụ về quyền riêng tư nằm ở [`../docs/MemoryBook-Security.md`](../docs/MemoryBook-Security.md) mục 8.
