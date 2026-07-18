# Portfolio — Vũ Thị Thanh Nhài

Hồ sơ năng lực (portfolio) dạng landing page một trang, nền tối sang trọng, hiệu ứng cuộn mượt bằng GSAP + Lenis. Backend theo Clean Architecture (ASP.NET Core + EF Core SQLite) cho phép sửa nội dung/ảnh qua trang admin.

## Công nghệ
- **Frontend:** HTML, CSS, JavaScript thuần + GSAP (ScrollTrigger) + Lenis
- **Backend:** ASP.NET Core Web API, Clean Architecture (Api → Application → Infrastructure → Domain)
- **Database:** SQLite qua EF Core

## Lộ trình
- **Phase 1 — UI hardcode:** dựng 9 section landing page, chạy được ngay.
- **Phase 2 — Backend:** API + trang admin để đổi nội dung/ảnh động.

## Cấu trúc dự kiến
```
src/
├─ Portfolio.Api/            (Presentation + wwwroot landing page + /admin)
├─ Portfolio.Application/    (Use cases, interfaces, DTOs)
├─ Portfolio.Infrastructure/ (EF Core SQLite, repositories, migrations)
└─ Portfolio.Domain/         (Entities)
```

Thiết kế chi tiết: [docs/superpowers/specs/2026-07-18-portfolio-landing-page-design.md](docs/superpowers/specs/2026-07-18-portfolio-landing-page-design.md)
