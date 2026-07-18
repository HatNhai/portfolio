# Portfolio Landing Page — Vũ Thị Thanh Nhài

**Ngày:** 2026-07-18
**Loại:** Landing page portfolio (single-page) + Backend Clean Architecture
**Nguồn thiết kế:** 9 ảnh slide trong `spec/1.png` … `spec/9.png`

---

## 1. Mục tiêu

Dựng một **landing page một trang** (single-page, cuộn dọc) tái hiện trung thực 9 slide đã thiết kế: nền đen sang trọng, tranh trường phái ấn tượng, typography lớn. Hiệu ứng chuyển động mượt, chuyên nghiệp (ảnh trượt vào, reveal, parallax, smooth-scroll).

Chia 2 giai đoạn:
- **Phase 1 — UI hardcode:** toàn bộ nội dung (chữ + ảnh) hardcode, chạy được ngay không cần backend.
- **Phase 2 — Backend Clean Architecture:** ASP.NET Core + EF Core SQLite cho phép sửa nội dung/ảnh qua trang admin (như "tham số hệ thống"); frontend chuyển sang nạp nội dung động từ API.

## 2. Quyết định đã chốt

| Hạng mục | Quyết định |
|---|---|
| Nội dung | Hardcode phần ảnh/slide trước (tái hiện đúng 9 ảnh); đổi được sau qua BE |
| Kiến trúc tổng | ASP.NET Core phục vụ landing page tĩnh trong `wwwroot` + Web API |
| Backend | Clean Architecture, 4 project: Api → Application → Infrastructure → Domain |
| Database | SQLite qua EF Core |
| Thư viện animation | GSAP + ScrollTrigger + Lenis (smooth-scroll) |
| Cấu trúc thư mục | `src/` chứa 4 project; bỏ 2 folder `Core/`, `UI/` trống |
| Font | Google Fonts — display cho tiêu đề + sans (Inter/Manrope) cho body |

## 3. Cấu trúc 9 section (map từ 9 ảnh)

Mỗi section chiếm gần full viewport, nền đen (`#0a0a0a`/`#000`), chữ trắng, khoảng trắng rộng.

| # | Section id | Nội dung (theo ảnh) | Hiệu ứng |
|---|---|---|---|
| 1 | `hero` | Tên "VŨ THỊ THANH NHÀI", "HỒ SƠ NĂNG LỰC" + 3 tranh collage (guitar / lady with parasol / tiger) | Ảnh trượt vào lệch nhịp (stagger) + parallax nhẹ khi cuộn |
| 2 | `about-role` | Nhãn "TÔI LÀ AI", tiêu đề "LẬP TRÌNH VIÊN .NET", đoạn mô tả C#/.NET/Blazor/CSDL, avatar tròn + ảnh cọ vẽ | Chữ reveal từ trái, ảnh slide từ phải |
| 3 | `intro` | "GIỚI THIỆU BẢN THÂN": Education, Work Experience, "EXHIBITS" (Solo Shows, Group Shows) + tranh willow | List fade-up từng dòng (stagger), ảnh clip-path reveal |
| 4 | `work-features` | "ĐẶC ĐIỂM CÔNG VIỆC": 01 Gardens in Blue / 02 Platter #2 / 03 The First Meeting — số lớn + ảnh + mô tả | Mỗi hàng trượt ngang xen kẽ, số đếm hiện dần |
| 5 | `springtime-bloom` | "SPRINGTIME BLOOM": diptych 2 tranh vuông + mô tả (Dimensions/Concept/Year) | 2 ảnh scale-in, text fade-up |
| 6 | `springtime-grid` | "Springtime, pts. 1-4": lưới 4 tranh (Seurat) | Grid stagger reveal |
| 7 | `what-i-do` | Nhãn "WHAT I DO", câu "I believe in showing slices of life…" + ảnh vuông + ảnh tròn | Ảnh parallax ngược chiều nhau, chữ reveal |
| 8 | `galleries` | "FIND MY WORKS": 3 card — City Gallery / Gustav Shaffer Museum / Modern Art Museum + địa điểm | 3 card fade-up tuần tự |
| 9 | `contact` | "THÔNG TIN LIÊN HỆ": ảnh chân dung + Address / Phone / Email | Ảnh reveal, thông tin fade-up |

Bổ sung ngoài 9 ảnh:
- **Nav mảnh** cố định (dot indicator hoặc thanh mỏng) để nhảy section.
- **Scroll progress** indicator.
- **Footer** nhỏ (tên + năm).
- **Responsive**: desktop giữ đúng layout ảnh; mobile xếp dọc 1 cột.

## 4. Tài sản ảnh (assets)

- Cắt các tranh/ảnh con ra khỏi 9 file spec PNG → lưu `wwwroot/assets/images/`.
- Công cụ cắt: ImageMagick hoặc Python Pillow (kiểm tra ở đầu Phase 1; nếu không có, báo người dùng cấp ảnh gốc hoặc dùng crop thủ công).
- Đặt tên theo section, ví dụ: `hero-guitar.jpg`, `hero-parasol.jpg`, `hero-tiger.jpg`, `about-avatar.jpg`, `work-01-gardens.jpg`, v.v.
- Ảnh chân dung người (ảnh stock trong template) giữ nguyên vai trò placeholder; người dùng thay ảnh thật sau qua admin ở Phase 2.

## 5. Frontend — chi tiết kỹ thuật (Phase 1)

**Vị trí:** `src/Portfolio.Api/wwwroot/`

```
wwwroot/
├─ index.html
├─ css/
│  ├─ reset.css
│  ├─ variables.css      (màu, spacing, font tokens)
│  └─ styles.css         (layout từng section)
├─ js/
│  ├─ main.js            (khởi tạo Lenis + GSAP)
│  ├─ animations.js      (ScrollTrigger cho từng section)
│  └─ content.js         (Phase 1: object hardcode; Phase 2: fetch /api/content)
└─ assets/images/…
```

**Nguyên tắc:**
- HTML ngữ nghĩa: mỗi slide là `<section>` có `id`, `aria-label`.
- CSS: custom properties cho token; layout bằng CSS Grid/Flexbox; `clamp()` cho typography responsive.
- Thư viện nạp qua CDN (hoặc file cục bộ trong `wwwroot/lib` để self-contained): GSAP, ScrollTrigger, Lenis.
- **Tách nội dung khỏi markup ngay từ Phase 1:** toàn bộ chữ + đường dẫn ảnh nằm trong `content.js` (một object JS). Phase 1 render từ object hardcode; Phase 2 chỉ đổi nguồn object này sang `fetch('/api/content')` — markup và animation không phải viết lại.
- Ưu tiên hiệu năng: ảnh `loading="lazy"`, kích thước hợp lý; tôn trọng `prefers-reduced-motion` (tắt animation mạnh).

**Tiêu chí hoàn thành Phase 1:** mở trang trong trình duyệt thấy đủ 9 section khớp bố cục 9 ảnh, animation trượt/reveal chạy mượt khi cuộn, responsive desktop + mobile, không lỗi console.

## 6. Backend — Clean Architecture (Phase 2)

### 6.1 Cấu trúc project

```
Portfolio_NV.sln
└─ src/
   ├─ Portfolio.Api/             (Presentation — ngoài cùng)
   │     Program.cs, Controllers/, wwwroot/ (landing page + /admin), DI
   │     refs: Application + Infrastructure
   ├─ Portfolio.Application/     (Use cases, interfaces, DTOs)
   │     refs: Domain
   ├─ Portfolio.Infrastructure/  (EF Core SQLite, Repositories, Migrations, Seed)
   │     refs: Application + Domain
   └─ Portfolio.Domain/          (Entities, value objects — không phụ thuộc)
```

**Quy tắc phụ thuộc (hướng vào trong, về Domain):**
- `Api` → `Application`, `Infrastructure` (chỉ để đăng ký DI)
- `Infrastructure` → `Application` (implement interfaces), `Domain`
- `Application` → `Domain`
- `Domain` → không tham chiếu gì

Đảo phụ thuộc: `Application` định nghĩa interface (VD `IContentRepository`), `Infrastructure` implement bằng EF Core. `Application` không biết `Infrastructure`.

### 6.2 Domain (mô hình dữ liệu sơ bộ)

- `PortfolioSection` — Id, Key (hero/about-role/…), Order, Title, Subtitle, Body, IsVisible.
- `MediaAsset` — Id, SectionId, Slot (tên vị trí ảnh trong section), FilePath, AltText, Order.
- `ContactInfo` — Address, Phone, Email (có thể là 1 section đặc biệt hoặc entity riêng).
- (Tinh chỉnh chi tiết ở bước writing-plans.)

### 6.3 Application

- Interfaces: `IContentRepository`, `IMediaStorage` (lưu file ảnh upload).
- DTOs: `SectionDto`, `MediaDto`, `ContentDto` (nguyên trang cho FE nạp 1 lần).
- Services/Use cases: `GetContent`, `UpdateSection`, `UploadMedia`, `ReorderMedia`.

### 6.4 Infrastructure

- `AppDbContext` (EF Core, SQLite, file `portfolio.db`).
- Repositories implement interface của Application.
- Migrations + **seed dữ liệu = đúng nội dung hardcode Phase 1** (để bật BE không mất nội dung).
- `IMediaStorage` lưu ảnh vào `wwwroot/assets/uploads/`.

### 6.5 Api

- `GET /api/content` → trả `ContentDto` cho landing page.
- Nhóm `/api/admin/*` (CRUD section, upload/xóa/sắp xếp ảnh) — có xác thực đơn giản (tối thiểu 1 lớp bảo vệ; chi tiết chốt ở writing-plans).
- Trang admin tối giản tại `/admin` (form sửa chữ + upload ảnh + preview).
- Phục vụ static file landing page từ `wwwroot`.

**Tiêu chí hoàn thành Phase 2:** landing page nạp nội dung từ `GET /api/content`; sửa 1 tiêu đề / thay 1 ảnh trong `/admin` → reload trang thấy đổi; dữ liệu lưu SQLite; kiến trúc đúng hướng phụ thuộc.

## 7. Ngoài phạm vi (YAGNI)

- Không đa ngôn ngữ i18n (nội dung tiếng Việt + Anh như ảnh).
- Không CMS phức tạp, không phân quyền nhiều vai trò (chỉ 1 admin).
- Không SSR/framework FE (React/Vue) — giữ HTML/CSS/JS thuần theo yêu cầu.
- Không CI/CD, không Docker ở giai đoạn này.

## 8. Rủi ro & lưu ý

- **Ảnh nguồn:** cần cắt ra từ spec PNG; nếu công cụ cắt không sẵn có sẽ cần người dùng hỗ trợ ảnh gốc.
- **Mâu thuẫn nội dung template:** slide là template họa sĩ nhưng chủ nhân là lập trình viên .NET — Phase 1 giữ nguyên như ảnh; điều chỉnh nội dung thật để sau (đổi qua admin).
- **Chưa phải git repo:** nên `git init` để commit spec và code (tùy người dùng).
