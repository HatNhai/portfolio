# Portfolio Landing Page — Phase 1 (UI Hardcode) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Dựng landing page một trang tái hiện trung thực 9 slide thiết kế, nội dung hardcode, hiệu ứng cuộn mượt (GSAP + ScrollTrigger + Lenis), chạy được ngay không cần backend.

**Architecture:** Static site đặt tại `src/Portfolio.Api/wwwroot/` (forward-compatible với Phase 2). Markup ngữ nghĩa cố định với các "hook" `data-content` / `data-src`; toàn bộ chữ + đường dẫn ảnh nằm trong một object trong `content.js`; hàm `hydrate()` đổ dữ liệu vào hook. Phase 2 chỉ đổi nguồn object sang `fetch('/api/content')` — không phải viết lại markup/animation. Ảnh được cắt ra từ 9 file spec PNG bằng Pillow.

**Tech Stack:** HTML5, CSS3 (Grid/Flexbox, custom properties, `clamp()`), JavaScript thuần, GSAP 3 + ScrollTrigger (CDN), Lenis (CDN), Google Fonts. Công cụ: Python 3.13 + Pillow (cắt ảnh + test cấu trúc).

## Global Constraints

- Không dùng framework FE (React/Vue), không bundler — chỉ HTML/CSS/JS thuần + thư viện qua CDN.
- Nền tối: `--bg: #0a0a0a`, chữ chính `--fg: #f5f5f5`, chữ phụ `--muted: #9a9a9a`, số nhạt `--faint: #6a6a6a`.
- Toàn bộ text/ảnh phải nạp qua object `content` trong `content.js` + `hydrate()` — KHÔNG viết cứng chữ trực tiếp trong thẻ (trừ nhãn cấu trúc). Mỗi phần tử hiển thị có `data-content="path.to.key"` hoặc ảnh có `data-src="path.to.img"` + `data-alt="path.to.alt"`.
- Font: display cho tiêu đề = **Manrope** (700/800), body = **Inter** (400/500). Nạp qua Google Fonts.
- Thư viện CDN pin phiên bản: GSAP `3.12.5`, ScrollTrigger `3.12.5`, Lenis `1.1.13`.
- Tôn trọng `prefers-reduced-motion: reduce` — tắt animation mạnh, giữ nội dung hiển thị đầy đủ.
- Ảnh: `loading="lazy"`, luôn có `alt`.
- Dev server chạy tại thư mục site: `cd src/Portfolio.Api/wwwroot && python -m http.server 5500` → mở `http://localhost:5500`.
- Kiểm thử: đây là UI trình bày tĩnh — logic tự động kiểm bằng `tools/check_structure.py` (Python, có sẵn); phần giao diện kiểm bằng so sánh trực quan với `spec/N.png`. Mỗi task UI có bước verify trực quan bắt buộc.
- Commit thường xuyên: mỗi task kết thúc bằng 1 commit.

## File Structure

```
tools/
  check_structure.py         # test cấu trúc: 9 section id + mọi ảnh tham chiếu tồn tại + mọi data-content có key
  crop_assets.py             # cắt ảnh con từ spec/*.png -> wwwroot/assets/images/
src/Portfolio.Api/wwwroot/
  index.html                 # markup ngữ nghĩa 9 section + nav + footer + hook data-*
  css/
    reset.css                # reset tối thiểu
    variables.css            # design tokens (màu, font, spacing)
    styles.css               # base + layout từng section + responsive
  js/
    content.js               # object `content` (data) + hàm hydrate()
    main.js                  # init Lenis + GSAP, gọi hydrate(), gọi initAnimations()
    animations.js            # ScrollTrigger cho từng section + nav active
  assets/images/             # ảnh cắt ra (Task 2 sinh)
  lib/                       # (tuỳ chọn) bản cục bộ của gsap/lenis nếu cần offline
```

---

### Task 1: Scaffold — khung trang, design tokens, smooth scroll

**Files:**
- Create: `src/Portfolio.Api/wwwroot/index.html`
- Create: `src/Portfolio.Api/wwwroot/css/reset.css`
- Create: `src/Portfolio.Api/wwwroot/css/variables.css`
- Create: `src/Portfolio.Api/wwwroot/css/styles.css`
- Create: `src/Portfolio.Api/wwwroot/js/main.js`
- Create: `src/Portfolio.Api/wwwroot/js/animations.js`
- Create: `tools/check_structure.py`
- Test: `tools/check_structure.py`

**Interfaces:**
- Produces: file `index.html` chứa 9 `<section>` với `id` = `hero`, `about-role`, `intro`, `work-features`, `springtime-bloom`, `springtime-grid`, `what-i-do`, `galleries`, `contact`. `main.js` export global `window.__hydrate` sẽ được định nghĩa ở Task 3 (Task 1 gọi có phòng vệ). `animations.js` định nghĩa `function initAnimations()` (Task 1 tạo rỗng, các task sau bổ sung).

- [ ] **Step 1: Viết test cấu trúc (thất bại trước)**

Create `tools/check_structure.py`:

```python
import re, sys, os

ROOT = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
WWW = os.path.join(ROOT, "src", "Portfolio.Api", "wwwroot")
HTML = os.path.join(WWW, "index.html")

SECTIONS = ["hero","about-role","intro","work-features","springtime-bloom",
            "springtime-grid","what-i-do","galleries","contact"]

def fail(msg):
    print("FAIL:", msg); sys.exit(1)

if not os.path.exists(HTML):
    fail("index.html không tồn tại")
html = open(HTML, encoding="utf-8").read()

# 1) đủ 9 section id
for s in SECTIONS:
    if not re.search(r'<section[^>]*\bid=["\']%s["\']' % re.escape(s), html):
        fail("thiếu section id=%s" % s)

# 2) mọi ảnh tham chiếu (data-src="/assets/images/..") phải tồn tại trên đĩa
refs = set(re.findall(r'data-src=["\'](/assets/images/[^"\']+)["\']', html))
for r in refs:
    p = os.path.join(WWW, r.lstrip("/").replace("/", os.sep))
    if not os.path.exists(p):
        fail("ảnh tham chiếu không tồn tại: %s" % r)

# 3) mọi data-content key phải có trong content.js (nếu file tồn tại)
cjs = os.path.join(WWW, "js", "content.js")
if os.path.exists(cjs):
    content = open(cjs, encoding="utf-8").read()
    keys = set(re.findall(r'data-content=["\']([^"\']+)["\']', html))
    keys |= set(re.findall(r'data-alt=["\']([^"\']+)["\']', html))
    for k in keys:
        leaf = k.split(".")[-1]
        if ('"%s"' % leaf) not in content and ("%s:" % leaf) not in content:
            fail("content.js thiếu key cho: %s" % k)

print("OK: %d sections, %d image refs" % (len(SECTIONS), len(refs)))
```

- [ ] **Step 2: Chạy test để xác nhận thất bại**

Run: `python tools/check_structure.py`
Expected: `FAIL: index.html không tồn tại`

- [ ] **Step 3: Tạo reset.css**

Create `src/Portfolio.Api/wwwroot/css/reset.css`:

```css
*,*::before,*::after{box-sizing:border-box;margin:0;padding:0}
html{-webkit-text-size-adjust:100%}
body{min-height:100vh;line-height:1.5;-webkit-font-smoothing:antialiased}
img,picture{max-width:100%;display:block}
ul{list-style:none}
a{color:inherit;text-decoration:none}
h1,h2,h3,p{overflow-wrap:break-word}
```

- [ ] **Step 4: Tạo variables.css (design tokens)**

Create `src/Portfolio.Api/wwwroot/css/variables.css`:

```css
:root{
  --bg:#0a0a0a; --bg-2:#111; --fg:#f5f5f5; --muted:#9a9a9a; --faint:#6a6a6a;
  --line:#242424;
  --font-display:"Manrope",system-ui,sans-serif;
  --font-body:"Inter",system-ui,sans-serif;
  --maxw:1200px;
  --pad-x:clamp(1.25rem,5vw,5rem);
  --sec-py:clamp(4rem,10vh,8rem);
  --h1:clamp(2.5rem,7vw,5.5rem);
  --h2:clamp(2rem,5vw,3.75rem);
  --lead:clamp(1.5rem,3.5vw,2.75rem);
  --label:.8rem;
}
```

- [ ] **Step 5: Tạo styles.css (base + tiện ích dùng chung)**

Create `src/Portfolio.Api/wwwroot/css/styles.css`:

```css
body{background:var(--bg);color:var(--fg);font-family:var(--font-body)}
.section{position:relative;min-height:100vh;display:flex;align-items:center;
  padding:var(--sec-py) var(--pad-x)}
.wrap{width:100%;max-width:var(--maxw);margin-inline:auto}
.eyebrow{font-family:var(--font-body);font-weight:600;letter-spacing:.3em;
  text-transform:uppercase;font-size:var(--label);color:var(--fg)}
.title{font-family:var(--font-display);font-weight:800;line-height:1.02;
  letter-spacing:-.01em}
.muted{color:var(--muted)}
.faint{color:var(--faint)}
/* trạng thái reveal mặc định (JS sẽ animate) */
.reveal{opacity:0;transform:translateY(30px)}
@media (prefers-reduced-motion:reduce){
  .reveal{opacity:1;transform:none}
  html{scroll-behavior:auto}
}
```

- [ ] **Step 6: Tạo index.html (khung 9 section rỗng + nav + footer)**

Create `src/Portfolio.Api/wwwroot/index.html`:

```html
<!doctype html>
<html lang="vi">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>Vũ Thị Thanh Nhài — Hồ sơ năng lực</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600&family=Manrope:wght@700;800&display=swap" rel="stylesheet">
<link rel="stylesheet" href="css/reset.css">
<link rel="stylesheet" href="css/variables.css">
<link rel="stylesheet" href="css/styles.css">
</head>
<body>
<div class="scroll-progress" id="scrollProgress"></div>
<nav class="dotnav" id="dotnav" aria-label="Section navigation"></nav>

<main>
  <section id="hero" class="section" aria-label="Trang bìa"></section>
  <section id="about-role" class="section" aria-label="Tôi là ai"></section>
  <section id="intro" class="section" aria-label="Giới thiệu bản thân"></section>
  <section id="work-features" class="section" aria-label="Đặc điểm công việc"></section>
  <section id="springtime-bloom" class="section" aria-label="Springtime Bloom"></section>
  <section id="springtime-grid" class="section" aria-label="Springtime pts 1-4"></section>
  <section id="what-i-do" class="section" aria-label="What I do"></section>
  <section id="galleries" class="section" aria-label="Find my works"></section>
  <section id="contact" class="section" aria-label="Thông tin liên hệ"></section>
</main>

<footer class="site-footer">
  <span data-content="footer.name">Vũ Thị Thanh Nhài</span>
  <span data-content="footer.year">© 2026</span>
</footer>

<script src="https://cdn.jsdelivr.net/npm/@studio-freight/lenis@1.1.13/dist/lenis.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/gsap@3.12.5/dist/gsap.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/gsap@3.12.5/dist/ScrollTrigger.min.js"></script>
<script src="js/content.js"></script>
<script src="js/animations.js"></script>
<script src="js/main.js"></script>
</body>
</html>
```

- [ ] **Step 7: Tạo animations.js (rỗng, sẽ bổ sung)**

Create `src/Portfolio.Api/wwwroot/js/animations.js`:

```js
// Được gọi sau khi hydrate xong. Các task sau thêm ScrollTrigger vào đây.
function initAnimations(){
  if (!window.gsap) return;
  gsap.registerPlugin(ScrollTrigger);
  // (các section sẽ được thêm ở task sau)
}
```

- [ ] **Step 8: Tạo main.js (Lenis smooth scroll + bootstrap)**

Create `src/Portfolio.Api/wwwroot/js/main.js`:

```js
(function(){
  var reduce = window.matchMedia("(prefers-reduced-motion:reduce)").matches;

  // Smooth scroll (Lenis) đồng bộ với GSAP ScrollTrigger
  if (window.Lenis && !reduce){
    var lenis = new Lenis({ lerp:0.1, smoothWheel:true });
    function raf(t){ lenis.raf(t); requestAnimationFrame(raf); }
    requestAnimationFrame(raf);
    if (window.ScrollTrigger){
      lenis.on("scroll", function(){ ScrollTrigger.update(); });
    }
  }

  // Nạp nội dung (Task 3 định nghĩa window.__hydrate)
  if (typeof window.__hydrate === "function") window.__hydrate();

  // Khởi tạo animation
  if (typeof initAnimations === "function") initAnimations();

  // Thanh tiến trình cuộn
  var bar = document.getElementById("scrollProgress");
  if (bar){
    window.addEventListener("scroll", function(){
      var h = document.documentElement;
      var p = h.scrollTop / (h.scrollHeight - h.clientHeight || 1);
      bar.style.transform = "scaleX(" + p + ")";
    }, { passive:true });
  }
})();
```

- [ ] **Step 9: Thêm style cho scroll-progress + footer + nav (base) vào styles.css**

Append to `src/Portfolio.Api/wwwroot/css/styles.css`:

```css
.scroll-progress{position:fixed;top:0;left:0;height:2px;width:100%;
  background:var(--fg);transform:scaleX(0);transform-origin:0 50%;z-index:50}
.site-footer{display:flex;justify-content:space-between;
  padding:2rem var(--pad-x);color:var(--muted);font-size:.85rem;border-top:1px solid var(--line)}
.dotnav{position:fixed;right:1.25rem;top:50%;transform:translateY(-50%);
  display:flex;flex-direction:column;gap:.75rem;z-index:40}
```

- [ ] **Step 10: Chạy test cấu trúc — xác nhận PASS phần section**

Run: `python tools/check_structure.py`
Expected: `OK: 9 sections, 0 image refs`

- [ ] **Step 11: Verify trực quan smooth scroll**

Run: `cd src/Portfolio.Api/wwwroot && python -m http.server 5500`
Mở `http://localhost:5500`. Expected: trang nền đen, cuộn thấy mượt (Lenis), thanh tiến trình trên cùng chạy khi cuộn, không lỗi console.

- [ ] **Step 12: Commit**

```bash
git add tools/check_structure.py src/Portfolio.Api/wwwroot
git commit -m "feat(fe): scaffold landing page — tokens, reset, smooth scroll, 9 empty sections"
```

---

### Task 2: Cắt ảnh từ spec PNG → assets

**Files:**
- Create: `tools/crop_assets.py`
- Create (sinh ra): `src/Portfolio.Api/wwwroot/assets/images/*.jpg`
- Test: kiểm tra file tồn tại + kích thước > 0

**Interfaces:**
- Produces: các file ảnh tại `/assets/images/` với tên chuẩn (dùng ở mọi section sau):
  `hero-guitar.jpg, hero-parasol.jpg, hero-tiger.jpg, about-avatar.jpg, about-brush.jpg, intro-willow.jpg, work-01.jpg, work-02.jpg, work-03.jpg, bloom-left.jpg, bloom-right.jpg, grid-1.jpg, grid-2.jpg, grid-3.jpg, grid-4.jpg, wid-square.jpg, wid-circle.jpg, gal-city.jpg, gal-gustav.jpg, gal-modern.jpg, contact-portrait.jpg`

- [ ] **Step 1: Viết script cắt ảnh với bảng toạ độ**

Create `tools/crop_assets.py`:

```python
import os
from PIL import Image

ROOT = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
SPEC = os.path.join(ROOT, "spec")
OUT  = os.path.join(ROOT, "src", "Portfolio.Api", "wwwroot", "assets", "images")
os.makedirs(OUT, exist_ok=True)

# (spec_file, out_name, (left, top, right, bottom))  — toạ độ trên ảnh 1366x768
JOBS = [
    ("1.png","hero-guitar.jpg",   (55,385,430,730)),
    ("1.png","hero-parasol.jpg",  (480,40,870,730)),
    ("1.png","hero-tiger.jpg",    (915,20,1345,390)),
    ("2.png","about-avatar.jpg",  (45,110,290,355)),
    ("2.png","about-brush.jpg",   (785,105,1345,710)),
    ("3.png","intro-willow.jpg",  (785,55,1345,715)),
    ("4.png","work-01.jpg",       (160,118,575,302)),
    ("4.png","work-02.jpg",       (160,350,575,535)),
    ("4.png","work-03.jpg",       (160,585,575,767)),
    ("5.png","bloom-left.jpg",    (45,210,470,712)),
    ("5.png","bloom-right.jpg",   (490,210,915,712)),
    ("6.png","grid-1.jpg",        (452,78,878,378)),
    ("6.png","grid-2.jpg",        (895,78,1320,378)),
    ("6.png","grid-3.jpg",        (452,398,878,698)),
    ("6.png","grid-4.jpg",        (895,398,1320,698)),
    ("7.png","wid-square.jpg",    (600,145,945,490)),
    ("7.png","wid-circle.jpg",    (985,140,1315,470)),
    ("8.png","gal-city.jpg",      (40,165,430,535)),
    ("8.png","gal-gustav.jpg",    (495,160,870,535)),
    ("8.png","gal-modern.jpg",    (940,160,1325,535)),
    ("9.png","contact-portrait.jpg",(453,205,912,712)),
]

for spec_file, name, box in JOBS:
    im = Image.open(os.path.join(SPEC, spec_file)).convert("RGB")
    im.crop(box).save(os.path.join(OUT, name), "JPEG", quality=90)
    print("saved", name, box)

print("TOTAL", len(JOBS))
```

- [ ] **Step 2: Chạy script**

Run: `python tools/crop_assets.py`
Expected: in ra 21 dòng `saved ...` và `TOTAL 21`.

- [ ] **Step 3: Test — mọi ảnh tồn tại & > 0 byte**

Run:
```bash
python -c "import os;d='src/Portfolio.Api/wwwroot/assets/images';n=[f for f in os.listdir(d) if f.endswith('.jpg')];assert len(n)==21,len(n);assert all(os.path.getsize(os.path.join(d,f))>0 for f in n);print('OK',len(n),'images')"
```
Expected: `OK 21 images`

- [ ] **Step 4: Verify trực quan crop**

Mở thư mục `src/Portfolio.Api/wwwroot/assets/images/` xem 21 ảnh. Với ảnh nào bị cắt lẹm tranh/mặt người, chỉnh lại `box` tương ứng trong `crop_assets.py` (±10–20px) rồi chạy lại Step 2. Đối chiếu với slide gốc trong `spec/`.

- [ ] **Step 5: Commit**

```bash
git add tools/crop_assets.py src/Portfolio.Api/wwwroot/assets/images
git commit -m "feat(fe): extract 21 image assets from spec slides"
```

---

### Task 3: Content data + hydration engine

**Files:**
- Create: `src/Portfolio.Api/wwwroot/js/content.js`
- Modify: `src/Portfolio.Api/wwwroot/index.html` (thêm hook cho footer đã có sẵn)
- Test: `tools/check_structure.py`

**Interfaces:**
- Consumes: các hook `data-content` / `data-src` / `data-alt` do các section tạo.
- Produces: `window.content` (object dữ liệu) và `window.__hydrate()` (đổ dữ liệu vào mọi `[data-content]`, `[data-src]`, `[data-alt]`). Hàm tra cứu theo path chấm (vd `"hero.name"`). Được `main.js` gọi ở bootstrap.

- [ ] **Step 1: Viết test — hydrate không để sót hook**

Tạm thời kiểm bằng runtime assertion. Thêm vào cuối `content.js` (Step 3) đoạn `data-check`. Test tự động ở Step 5 dùng `check_structure.py` (đã kiểm key tồn tại trong content.js).

- [ ] **Step 2: Tạo content.js với object dữ liệu đầy đủ (đúng nội dung 9 ảnh)**

Create `src/Portfolio.Api/wwwroot/js/content.js`:

```js
window.content = {
  hero: {
    name: "VŨ THỊ THANH NHÀI",
    subtitle: "HỒ SƠ NĂNG LỰC",
    imgGuitar: "/assets/images/hero-guitar.jpg",  altGuitar: "Người phụ nữ chơi guitar",
    imgParasol:"/assets/images/hero-parasol.jpg", altParasol:"Người phụ nữ cầm ô",
    imgTiger:  "/assets/images/hero-tiger.jpg",   altTiger:  "Con hổ trong rừng"
  },
  aboutRole: {
    eyebrow: "TÔI LÀ AI",
    title: "LẬP TRÌNH VIÊN .NET",
    body: "Lập trình viên Công nghệ thông tin với nền tảng về phát triển phần mềm bằng C#, .NET, Blazor và cơ sở dữ liệu. Có tinh thần trách nhiệm, khả năng học hỏi nhanh và làm việc hiệu quả trong môi trường nhóm. Mong muốn phát triển chuyên môn, đóng góp vào các dự án thực tế và gắn bó lâu dài với doanh nghiệp.",
    imgAvatar:"/assets/images/about-avatar.jpg", altAvatar:"Ảnh đại diện",
    imgBrush: "/assets/images/about-brush.jpg",  altBrush: "Bàn tay cầm cọ vẽ"
  },
  intro: {
    title: "GIỚI THIỆU BẢN THÂN",
    eduHead: "Education",
    edu: ["Bachelor of Fine Arts, Trempleway Academy","MFA in Painting, Grayerville University"],
    workHead: "Work Experience",
    work: ["Associate Professor, Trempleway Academy","Guest Curator, Grayerville Fine Art Gallery","Guest Speaker, Trempleway Art Conference"],
    exhibitsTitle: "EXHIBITS",
    soloHead: "Solo Shows",
    solo: ["Splash of Ambiguity, Grayerville Fine Art Gallery, 2024","Unknown Corners of Still Life, Eximus Studios, 2024","Paint It True, Trempleway Art Conference, 2023"],
    groupHead: "Group Shows",
    group: ["Art Maven Picks, Trempleway Art Conference, 2023","Trempleway Artists' Forum, Grayerville Fine Art Gallery, 2022"],
    imgWillow:"/assets/images/intro-willow.jpg", altWillow:"Tranh hàng liễu"
  },
  workFeatures: {
    eyebrow: "ĐẶC ĐIỂM CÔNG VIỆC",
    items: [
      { no:"01", title:"GARDENS IN BLUE", desc:"A quick painting of one of my favorite parks in France", img:"/assets/images/work-01.jpg", alt:"Gardens in Blue" },
      { no:"02", title:"PLATTER #2",       desc:"Peaches and apples on a holiday table",                 img:"/assets/images/work-02.jpg", alt:"Platter #2" },
      { no:"03", title:"THE FIRST MEETING",desc:"A painting based on a scene in a book",                 img:"/assets/images/work-03.jpg", alt:"The First Meeting" }
    ]
  },
  bloom: {
    title: "SPRINGTIME BLOOM",
    imgLeft:"/assets/images/bloom-left.jpg",  altLeft:"Springtime Bloom trái",
    imgRight:"/assets/images/bloom-right.jpg", altRight:"Springtime Bloom phải",
    desc: "A diptych of a park in spring, based off my imagination and a beautiful scene in a book. This is currently hanging in the San Dias City Gallery.",
    meta: ["Dimensions: 24\" x 14\" (each)","Concept: Springtime greenery","Year: 2020"]
  },
  grid: {
    eyebrow: "Springtime, pts. 1-4",
    imgs: [
      {img:"/assets/images/grid-1.jpg", alt:"Springtime pt.1"},
      {img:"/assets/images/grid-2.jpg", alt:"Springtime pt.2"},
      {img:"/assets/images/grid-3.jpg", alt:"Springtime pt.3"},
      {img:"/assets/images/grid-4.jpg", alt:"Springtime pt.4"}
    ]
  },
  whatIDo: {
    eyebrow: "WHAT I DO",
    lead: "I believe in showing slices of life. I paint in oils, watercolors, and acrylics. I also draw in graphite and ink.",
    imgSquare:"/assets/images/wid-square.jpg", altSquare:"Chân dung nghệ sĩ",
    imgCircle:"/assets/images/wid-circle.jpg", altCircle:"Bàn tay cầm cọ"
  },
  galleries: {
    eyebrow: "FIND MY WORKS",
    items: [
      { name:"CITY GALLERY",          city:"San Dias City",  img:"/assets/images/gal-city.jpg",   alt:"City Gallery" },
      { name:"GUSTAV SHAFFER MUSEUM",  city:"San Dias City",  img:"/assets/images/gal-gustav.jpg", alt:"Gustav Shaffer Museum" },
      { name:"MODERN ART MUSEUM",      city:"El Fantigo City", img:"/assets/images/gal-modern.jpg", alt:"Modern Art Museum" }
    ]
  },
  contact: {
    title: "THÔNG TIN LIÊN HỆ",
    img:"/assets/images/contact-portrait.jpg", alt:"Chân dung liên hệ",
    addressHead:"Address", address:"123 Anywhere St., Any City State, Country 12345",
    phoneHead:"Phone", phone:"(123) 456 7890",
    emailHead:"Email", email:"hello@reallygreatsite.com"
  },
  footer: { name:"Vũ Thị Thanh Nhài", year:"© 2026 · Portfolio" }
};

// Tra cứu path chấm: get(content, "hero.name")
function __get(obj, path){
  return path.split(".").reduce(function(o,k){ return (o==null)?undefined:o[k]; }, obj);
}

window.__hydrate = function(){
  var c = window.content;
  document.querySelectorAll("[data-content]").forEach(function(el){
    var v = __get(c, el.getAttribute("data-content"));
    if (v != null) el.textContent = v;
  });
  document.querySelectorAll("[data-src]").forEach(function(el){
    var v = __get(c, el.getAttribute("data-src"));
    if (v != null) el.setAttribute("src", v);
    var a = el.getAttribute("data-alt");
    if (a){ var av = __get(c, a); if (av != null) el.setAttribute("alt", av); }
  });
};
```

- [ ] **Step 3: Chạy test cấu trúc**

Run: `python tools/check_structure.py`
Expected: `OK: 9 sections, 0 image refs` (chưa có section nào dùng ảnh — sẽ tăng ở các task sau).

- [ ] **Step 4: Verify footer đã hydrate**

Reload `http://localhost:5500`. Expected: footer hiển thị "Vũ Thị Thanh Nhài" và "© 2026 · Portfolio".

- [ ] **Step 5: Commit**

```bash
git add src/Portfolio.Api/wwwroot/js/content.js
git commit -m "feat(fe): content data model + hydration engine"
```

---

### Task 4: Section 1 — Hero / Cover

**Files:**
- Modify: `src/Portfolio.Api/wwwroot/index.html` (nội dung `#hero`)
- Modify: `src/Portfolio.Api/wwwroot/css/styles.css` (append `.hero-*`)
- Modify: `src/Portfolio.Api/wwwroot/js/animations.js` (thêm reveal hero)
- Test: `tools/check_structure.py` + đối chiếu `spec/1.png`

**Interfaces:**
- Consumes: `content.hero.*`, ảnh `hero-guitar/parasol/tiger.jpg`.

- [ ] **Step 1: Markup #hero (thay `<section id="hero" ...></section>`)**

```html
<section id="hero" class="section hero" aria-label="Trang bìa">
  <div class="wrap hero-grid">
    <div class="hero-text">
      <h1 class="title hero-name" data-content="hero.name">VŨ THỊ THANH NHÀI</h1>
      <p class="eyebrow hero-sub" data-content="hero.subtitle">HỒ SƠ NĂNG LỰC</p>
    </div>
    <figure class="hero-art hero-art--guitar reveal">
      <img data-src="hero.imgGuitar" data-alt="hero.altGuitar" loading="lazy" alt="">
    </figure>
    <figure class="hero-art hero-art--parasol reveal">
      <img data-src="hero.imgParasol" data-alt="hero.altParasol" loading="lazy" alt="">
    </figure>
    <figure class="hero-art hero-art--tiger reveal">
      <img data-src="hero.imgTiger" data-alt="hero.altTiger" loading="lazy" alt="">
    </figure>
  </div>
</section>
```

- [ ] **Step 2: CSS hero (append vào styles.css)**

```css
.hero-grid{display:grid;grid-template-columns:repeat(12,1fr);
  grid-template-rows:auto auto;gap:1.5rem;align-items:center}
.hero-text{grid-column:1/4;grid-row:1/3;z-index:2}
.hero-name{font-size:var(--h1)}
.hero-sub{margin-top:1.5rem;color:var(--fg)}
.hero-art{overflow:hidden}
.hero-art img{width:100%;height:100%;object-fit:cover}
.hero-art--guitar{grid-column:1/4;grid-row:2;align-self:end;aspect-ratio:3/4;max-width:230px}
.hero-art--parasol{grid-column:5/9;grid-row:1/3;aspect-ratio:3/4}
.hero-art--tiger{grid-column:9/13;grid-row:1;aspect-ratio:4/3}
```
(Nếu chồng lấn với text, đẩy `.hero-art--guitar` xuống dưới text bằng grid-row như trên.)

- [ ] **Step 3: Animation hero (trong initAnimations của animations.js, thêm vào thân hàm)**

```js
  gsap.from("#hero .hero-art", {
    x: function(i){ return i===0 ? -80 : i===2 ? 80 : 0; },
    y: 60, opacity: 0, duration: 1, ease: "power3.out", stagger: 0.15,
    scrollTrigger: { trigger: "#hero", start: "top 80%" }
  });
  gsap.from("#hero .hero-text > *", {
    y: 40, opacity: 0, duration: 0.9, ease: "power3.out", stagger: 0.12,
    scrollTrigger: { trigger: "#hero", start: "top 80%" }
  });
```
Đồng thời bỏ class `reveal` gây ẩn cứng cho `.hero-art` (GSAP `from` tự xử lý) — hoặc để `reveal` và dùng `gsap.to(...{opacity:1,y:0})`. Chọn 1 cách nhất quán: **dùng `gsap.from` và bỏ `reveal`** trên các `.hero-art` ở Step 1 markup (xoá chữ `reveal`).

- [ ] **Step 4: Chạy test cấu trúc**

Run: `python tools/check_structure.py`
Expected: `OK: 9 sections, 3 image refs`

- [ ] **Step 5: Verify trực quan vs spec/1.png**

Reload trang. Expected: tên lớn bên trái, tranh guitar dưới trái, tranh cầm ô ở giữa (cao), tranh hổ trên phải; khi cuộn tới thấy ảnh trượt vào lệch nhịp. Đối chiếu bố cục với `spec/1.png`.

- [ ] **Step 6: Commit**

```bash
git add src/Portfolio.Api/wwwroot
git commit -m "feat(fe): hero/cover section with staggered image reveal"
```

---

### Task 5: Section 2 — Tôi là ai (.NET Developer)

**Files:** Modify `index.html` (`#about-role`), `styles.css`, `animations.js`. Test: `check_structure.py` + `spec/2.png`.

**Interfaces:** Consumes `content.aboutRole.*`, ảnh `about-avatar.jpg`, `about-brush.jpg`.

- [ ] **Step 1: Markup #about-role**

```html
<section id="about-role" class="section about" aria-label="Tôi là ai">
  <div class="wrap about-grid">
    <div class="about-text">
      <p class="eyebrow" data-content="aboutRole.eyebrow">TÔI LÀ AI</p>
      <figure class="about-avatar"><img data-src="aboutRole.imgAvatar" data-alt="aboutRole.altAvatar" loading="lazy" alt=""></figure>
      <h2 class="title about-title" data-content="aboutRole.title">LẬP TRÌNH VIÊN .NET</h2>
      <p class="about-body muted" data-content="aboutRole.body"></p>
    </div>
    <figure class="about-photo"><img data-src="aboutRole.imgBrush" data-alt="aboutRole.altBrush" loading="lazy" alt=""></figure>
  </div>
</section>
```

- [ ] **Step 2: CSS (append)**

```css
.about-grid{display:grid;grid-template-columns:1fr 1fr;gap:clamp(2rem,6vw,5rem);align-items:center}
.about-avatar{width:120px;aspect-ratio:1;border-radius:50%;overflow:hidden;margin:1.5rem 0}
.about-avatar img{width:100%;height:100%;object-fit:cover}
.about-title{font-size:var(--h2);margin:.5rem 0 1.5rem}
.about-body{max-width:46ch;font-size:1.05rem}
.about-photo{aspect-ratio:4/5;overflow:hidden}
.about-photo img{width:100%;height:100%;object-fit:cover}
```

- [ ] **Step 3: Animation (append trong initAnimations)**

```js
  gsap.from("#about-role .about-text > *", {
    x:-50, opacity:0, duration:.9, ease:"power3.out", stagger:.12,
    scrollTrigger:{ trigger:"#about-role", start:"top 70%" }
  });
  gsap.from("#about-role .about-photo", {
    x:80, opacity:0, duration:1, ease:"power3.out",
    scrollTrigger:{ trigger:"#about-role", start:"top 70%" }
  });
```

- [ ] **Step 4: Test** — `python tools/check_structure.py` → `OK: 9 sections, 5 image refs`
- [ ] **Step 5: Verify vs spec/2.png** — nhãn trên, avatar tròn, tiêu đề ".NET", đoạn mô tả trái; ảnh cọ vẽ phải; chữ trượt trái, ảnh trượt phải.
- [ ] **Step 6: Commit** — `git add ... && git commit -m "feat(fe): about-role section (.NET developer)"`

---

### Task 6: Section 3 — Giới thiệu bản thân (Education/Work/Exhibits)

**Files:** Modify `index.html` (`#intro`), `styles.css`, `animations.js`. Test: `check_structure.py` + `spec/3.png`.

**Interfaces:** Consumes `content.intro.*` (mảng edu/work/solo/group — hardcode số phần tử đúng như dữ liệu), ảnh `intro-willow.jpg`.

- [ ] **Step 1: Markup #intro** (list item hardcode đúng số lượng, mỗi dòng có `data-content` chỉ vào phần tử mảng theo chỉ số)

```html
<section id="intro" class="section intro" aria-label="Giới thiệu bản thân">
  <div class="wrap intro-grid">
    <div class="intro-col">
      <h2 class="title intro-title" data-content="intro.title">GIỚI THIỆU BẢN THÂN</h2>
      <h3 class="intro-head" data-content="intro.eduHead">Education</h3>
      <ul class="intro-list">
        <li data-content="intro.edu.0"></li>
        <li data-content="intro.edu.1"></li>
      </ul>
      <h3 class="intro-head" data-content="intro.workHead">Work Experience</h3>
      <ul class="intro-list">
        <li data-content="intro.work.0"></li>
        <li data-content="intro.work.1"></li>
        <li data-content="intro.work.2"></li>
      </ul>
      <h2 class="title intro-title intro-title--exhibits" data-content="intro.exhibitsTitle">EXHIBITS</h2>
      <h3 class="intro-head" data-content="intro.soloHead">Solo Shows</h3>
      <ul class="intro-list intro-list--italic">
        <li data-content="intro.solo.0"></li>
        <li data-content="intro.solo.1"></li>
        <li data-content="intro.solo.2"></li>
      </ul>
      <h3 class="intro-head" data-content="intro.groupHead">Group Shows</h3>
      <ul class="intro-list intro-list--italic">
        <li data-content="intro.group.0"></li>
        <li data-content="intro.group.1"></li>
      </ul>
    </div>
    <figure class="intro-photo"><img data-src="intro.imgWillow" data-alt="intro.altWillow" loading="lazy" alt=""></figure>
  </div>
</section>
```

Lưu ý: `__get` đã hỗ trợ path chấm nên `intro.edu.0` hoạt động (mảng truy cập bằng chỉ số).

- [ ] **Step 2: CSS (append)**

```css
.intro{align-items:flex-start}
.intro-grid{display:grid;grid-template-columns:1.1fr .9fr;gap:clamp(2rem,6vw,5rem);align-items:start}
.intro-title{font-size:clamp(1.8rem,4vw,3rem)}
.intro-title--exhibits{margin-top:2.5rem}
.intro-head{font-weight:600;color:var(--fg);margin:1.25rem 0 .4rem}
.intro-list{display:grid;gap:.35rem;color:var(--muted);padding-left:1.1rem}
.intro-list li{list-style:disc}
.intro-list--italic li{font-style:italic}
.intro-photo{position:sticky;top:6rem;aspect-ratio:3/4;overflow:hidden}
.intro-photo img{width:100%;height:100%;object-fit:cover}
```

- [ ] **Step 3: Animation (append)**

```js
  gsap.from("#intro .intro-col > *", {
    y:30, opacity:0, duration:.7, ease:"power2.out", stagger:.06,
    scrollTrigger:{ trigger:"#intro", start:"top 75%" }
  });
  gsap.from("#intro .intro-photo", {
    clipPath:"inset(0 0 100% 0)", duration:1.1, ease:"power3.out",
    scrollTrigger:{ trigger:"#intro", start:"top 75%" }
  });
```

- [ ] **Step 4: Test** — `OK: 9 sections, 6 image refs`
- [ ] **Step 5: Verify vs spec/3.png** — 2 cột: trái là các danh mục, phải là tranh liễu (sticky); dòng fade-up, ảnh reveal từ trên xuống.
- [ ] **Step 6: Commit** — `git commit -m "feat(fe): intro section — education/work/exhibits"`

---

### Task 7: Section 4 — Đặc điểm công việc (01/02/03)

**Files:** Modify `index.html` (`#work-features`), `styles.css`, `animations.js`. Test: `check_structure.py` + `spec/4.png`.

**Interfaces:** Consumes `content.workFeatures.items[0..2]`.

- [ ] **Step 1: Markup #work-features** (3 hàng hardcode, mỗi hàng trỏ vào `items.N.*`)

```html
<section id="work-features" class="section wf" aria-label="Đặc điểm công việc">
  <div class="wrap">
    <p class="eyebrow" data-content="workFeatures.eyebrow">ĐẶC ĐIỂM CÔNG VIỆC</p>
    <div class="wf-list">
      <article class="wf-row reveal">
        <span class="wf-no faint" data-content="workFeatures.items.0.no">01</span>
        <figure class="wf-img"><img data-src="workFeatures.items.0.img" data-alt="workFeatures.items.0.alt" loading="lazy" alt=""></figure>
        <div class="wf-body">
          <h3 class="title wf-title" data-content="workFeatures.items.0.title"></h3>
          <p class="muted" data-content="workFeatures.items.0.desc"></p>
        </div>
      </article>
      <article class="wf-row reveal">
        <span class="wf-no faint" data-content="workFeatures.items.1.no">02</span>
        <figure class="wf-img"><img data-src="workFeatures.items.1.img" data-alt="workFeatures.items.1.alt" loading="lazy" alt=""></figure>
        <div class="wf-body">
          <h3 class="title wf-title" data-content="workFeatures.items.1.title"></h3>
          <p class="muted" data-content="workFeatures.items.1.desc"></p>
        </div>
      </article>
      <article class="wf-row reveal">
        <span class="wf-no faint" data-content="workFeatures.items.2.no">03</span>
        <figure class="wf-img"><img data-src="workFeatures.items.2.img" data-alt="workFeatures.items.2.alt" loading="lazy" alt=""></figure>
        <div class="wf-body">
          <h3 class="title wf-title" data-content="workFeatures.items.2.title"></h3>
          <p class="muted" data-content="workFeatures.items.2.desc"></p>
        </div>
      </article>
    </div>
  </div>
</section>
```

- [ ] **Step 2: CSS (append)** — bỏ `.reveal` opacity vì dùng gsap.from; hoặc giữ và animate. Ở đây dùng gsap.from → xoá `reveal` khỏi `.wf-row` markup.

```css
.wf-list{display:grid;gap:clamp(1.5rem,4vw,3rem);margin-top:3rem}
.wf-row{display:grid;grid-template-columns:auto 300px 1fr;gap:clamp(1rem,3vw,2.5rem);align-items:center}
.wf-no{font-family:var(--font-display);font-size:clamp(2rem,4vw,3.5rem);line-height:1}
.wf-img{aspect-ratio:2/1;overflow:hidden}
.wf-img img{width:100%;height:100%;object-fit:cover}
.wf-title{font-size:clamp(1.75rem,4vw,3rem);font-weight:700}
.wf-body p{margin-top:.5rem}
```

- [ ] **Step 3: Animation (append)**

```js
  gsap.utils.toArray("#work-features .wf-row").forEach(function(row, i){
    gsap.from(row, {
      x: i % 2 ? 80 : -80, opacity:0, duration:.9, ease:"power3.out",
      scrollTrigger:{ trigger:row, start:"top 82%" }
    });
  });
```

- [ ] **Step 4: Test** — `OK: 9 sections, 9 image refs`
- [ ] **Step 5: Verify vs spec/4.png** — 3 hàng số lớn + ảnh ngang + tiêu đề/mô tả; hàng trượt ngang xen kẽ.
- [ ] **Step 6: Commit** — `git commit -m "feat(fe): work-features 01/02/03 rows"`

---

### Task 8: Section 5 — Springtime Bloom (diptych)

**Files:** Modify `index.html` (`#springtime-bloom`), `styles.css`, `animations.js`. Test: `check_structure.py` + `spec/5.png`.

**Interfaces:** Consumes `content.bloom.*` (meta là mảng 3 dòng).

- [ ] **Step 1: Markup #springtime-bloom**

```html
<section id="springtime-bloom" class="section bloom" aria-label="Springtime Bloom">
  <div class="wrap">
    <h2 class="title bloom-title" data-content="bloom.title">SPRINGTIME BLOOM</h2>
    <div class="bloom-grid">
      <figure class="bloom-img"><img data-src="bloom.imgLeft" data-alt="bloom.altLeft" loading="lazy" alt=""></figure>
      <figure class="bloom-img"><img data-src="bloom.imgRight" data-alt="bloom.altRight" loading="lazy" alt=""></figure>
      <div class="bloom-text">
        <p data-content="bloom.desc"></p>
        <ul class="bloom-meta faint">
          <li data-content="bloom.meta.0"></li>
          <li data-content="bloom.meta.1"></li>
          <li data-content="bloom.meta.2"></li>
        </ul>
      </div>
    </div>
  </div>
</section>
```

- [ ] **Step 2: CSS (append)**

```css
.bloom-title{font-size:var(--h2)}
.bloom-grid{display:grid;grid-template-columns:1fr 1fr 1fr;gap:clamp(1rem,3vw,2rem);margin-top:2rem;align-items:start}
.bloom-img{aspect-ratio:1;overflow:hidden}
.bloom-img img{width:100%;height:100%;object-fit:cover}
.bloom-text{align-self:end}
.bloom-text p{color:var(--muted);max-width:34ch}
.bloom-meta{margin-top:1.5rem;display:grid;gap:.25rem}
```

- [ ] **Step 3: Animation (append)**

```js
  gsap.from("#springtime-bloom .bloom-img", {
    scale:.9, opacity:0, duration:1, ease:"power3.out", stagger:.15,
    scrollTrigger:{ trigger:"#springtime-bloom", start:"top 75%" }
  });
  gsap.from("#springtime-bloom .bloom-text", {
    y:30, opacity:0, duration:.9, ease:"power2.out", delay:.2,
    scrollTrigger:{ trigger:"#springtime-bloom", start:"top 75%" }
  });
```

- [ ] **Step 4: Test** — `OK: 9 sections, 11 image refs`
- [ ] **Step 5: Verify vs spec/5.png** — tiêu đề trên, 2 tranh vuông + cột mô tả/kích thước; ảnh scale-in.
- [ ] **Step 6: Commit** — `git commit -m "feat(fe): springtime bloom diptych"`

---

### Task 9: Section 6 — Springtime pts. 1-4 (grid)

**Files:** Modify `index.html` (`#springtime-grid`), `styles.css`, `animations.js`. Test: `check_structure.py` + `spec/6.png`.

**Interfaces:** Consumes `content.grid.imgs[0..3]`.

- [ ] **Step 1: Markup #springtime-grid**

```html
<section id="springtime-grid" class="section sgrid" aria-label="Springtime pts 1-4">
  <div class="wrap sgrid-wrap">
    <p class="eyebrow faint sgrid-label" data-content="grid.eyebrow">Springtime, pts. 1-4</p>
    <div class="sgrid-grid">
      <figure class="sgrid-img"><img data-src="grid.imgs.0.img" data-alt="grid.imgs.0.alt" loading="lazy" alt=""></figure>
      <figure class="sgrid-img"><img data-src="grid.imgs.1.img" data-alt="grid.imgs.1.alt" loading="lazy" alt=""></figure>
      <figure class="sgrid-img"><img data-src="grid.imgs.2.img" data-alt="grid.imgs.2.alt" loading="lazy" alt=""></figure>
      <figure class="sgrid-img"><img data-src="grid.imgs.3.img" data-alt="grid.imgs.3.alt" loading="lazy" alt=""></figure>
    </div>
  </div>
</section>
```

- [ ] **Step 2: CSS (append)**

```css
.sgrid-wrap{display:grid;grid-template-columns:1fr 2fr;gap:2rem;align-items:start}
.sgrid-label{margin-top:.5rem}
.sgrid-grid{display:grid;grid-template-columns:1fr 1fr;gap:clamp(1rem,2vw,1.5rem)}
.sgrid-img{aspect-ratio:4/3;overflow:hidden}
.sgrid-img img{width:100%;height:100%;object-fit:cover}
```

- [ ] **Step 3: Animation (append)**

```js
  gsap.from("#springtime-grid .sgrid-img", {
    y:50, opacity:0, duration:.8, ease:"power3.out", stagger:.12,
    scrollTrigger:{ trigger:"#springtime-grid", start:"top 78%" }
  });
```

- [ ] **Step 4: Test** — `OK: 9 sections, 15 image refs`
- [ ] **Step 5: Verify vs spec/6.png** — nhãn nhỏ trái, lưới 2×2 tranh phải; ảnh reveal stagger.
- [ ] **Step 6: Commit** — `git commit -m "feat(fe): springtime 4-image grid"`

---

### Task 10: Section 7 — What I do

**Files:** Modify `index.html` (`#what-i-do`), `styles.css`, `animations.js`. Test: `check_structure.py` + `spec/7.png`.

**Interfaces:** Consumes `content.whatIDo.*`.

- [ ] **Step 1: Markup #what-i-do**

```html
<section id="what-i-do" class="section wid" aria-label="What I do">
  <div class="wrap wid-grid">
    <div class="wid-text">
      <p class="eyebrow" data-content="whatIDo.eyebrow">WHAT I DO</p>
      <p class="wid-lead" data-content="whatIDo.lead"></p>
    </div>
    <figure class="wid-square"><img data-src="whatIDo.imgSquare" data-alt="whatIDo.altSquare" loading="lazy" alt=""></figure>
    <figure class="wid-circle"><img data-src="whatIDo.imgCircle" data-alt="whatIDo.altCircle" loading="lazy" alt=""></figure>
  </div>
</section>
```

- [ ] **Step 2: CSS (append)**

```css
.wid-grid{display:grid;grid-template-columns:1.1fr .9fr .6fr;gap:clamp(1.5rem,4vw,3rem);align-items:center}
.wid-lead{font-family:var(--font-body);font-size:var(--lead);line-height:1.15;margin-top:1.5rem;max-width:14ch}
.wid-square{aspect-ratio:1;overflow:hidden}
.wid-square img{width:100%;height:100%;object-fit:cover}
.wid-circle{aspect-ratio:1;border-radius:50%;overflow:hidden}
.wid-circle img{width:100%;height:100%;object-fit:cover}
```

- [ ] **Step 3: Animation (append)** — parallax ngược chiều 2 ảnh

```js
  gsap.from("#what-i-do .wid-text > *", {
    y:40, opacity:0, duration:.9, ease:"power3.out", stagger:.12,
    scrollTrigger:{ trigger:"#what-i-do", start:"top 75%" }
  });
  gsap.to("#what-i-do .wid-square", { yPercent:-8, ease:"none",
    scrollTrigger:{ trigger:"#what-i-do", start:"top bottom", end:"bottom top", scrub:true }});
  gsap.to("#what-i-do .wid-circle", { yPercent:8, ease:"none",
    scrollTrigger:{ trigger:"#what-i-do", start:"top bottom", end:"bottom top", scrub:true }});
```

- [ ] **Step 4: Test** — `OK: 9 sections, 17 image refs`
- [ ] **Step 5: Verify vs spec/7.png** — câu lớn trái, ảnh vuông giữa, ảnh tròn phải; 2 ảnh trôi ngược chiều nhẹ khi cuộn.
- [ ] **Step 6: Commit** — `git commit -m "feat(fe): what-i-do section with parallax"`

---

### Task 11: Section 8 — Find my works (galleries)

**Files:** Modify `index.html` (`#galleries`), `styles.css`, `animations.js`. Test: `check_structure.py` + `spec/8.png`.

**Interfaces:** Consumes `content.galleries.items[0..2]`.

- [ ] **Step 1: Markup #galleries**

```html
<section id="galleries" class="section gal" aria-label="Find my works">
  <div class="wrap">
    <p class="eyebrow" data-content="galleries.eyebrow">FIND MY WORKS</p>
    <div class="gal-grid">
      <article class="gal-card">
        <figure class="gal-img gal-img--circle"><img data-src="galleries.items.0.img" data-alt="galleries.items.0.alt" loading="lazy" alt=""></figure>
        <h3 class="title gal-name" data-content="galleries.items.0.name"></h3>
        <p class="muted" data-content="galleries.items.0.city"></p>
      </article>
      <article class="gal-card">
        <figure class="gal-img"><img data-src="galleries.items.1.img" data-alt="galleries.items.1.alt" loading="lazy" alt=""></figure>
        <h3 class="title gal-name" data-content="galleries.items.1.name"></h3>
        <p class="muted" data-content="galleries.items.1.city"></p>
      </article>
      <article class="gal-card">
        <figure class="gal-img"><img data-src="galleries.items.2.img" data-alt="galleries.items.2.alt" loading="lazy" alt=""></figure>
        <h3 class="title gal-name" data-content="galleries.items.2.name"></h3>
        <p class="muted" data-content="galleries.items.2.city"></p>
      </article>
    </div>
  </div>
</section>
```

- [ ] **Step 2: CSS (append)**

```css
.gal-grid{display:grid;grid-template-columns:repeat(3,1fr);gap:clamp(1.5rem,4vw,3rem);margin-top:2.5rem;align-items:start}
.gal-img{aspect-ratio:4/3;overflow:hidden;margin-bottom:1.25rem}
.gal-img--circle{aspect-ratio:1;border-radius:50%}
.gal-img img{width:100%;height:100%;object-fit:cover}
.gal-name{font-size:clamp(1.25rem,2.5vw,1.75rem);font-weight:700}
.gal-card p{margin-top:.4rem}
```

- [ ] **Step 3: Animation (append)**

```js
  gsap.from("#galleries .gal-card", {
    y:60, opacity:0, duration:.85, ease:"power3.out", stagger:.18,
    scrollTrigger:{ trigger:"#galleries", start:"top 78%" }
  });
```

- [ ] **Step 4: Test** — `OK: 9 sections, 20 image refs`
- [ ] **Step 5: Verify vs spec/8.png** — 3 card, card đầu ảnh tròn, 2 card sau ảnh chữ nhật; tên + thành phố; card fade-up tuần tự.
- [ ] **Step 6: Commit** — `git commit -m "feat(fe): galleries / find my works"`

---

### Task 12: Section 9 — Thông tin liên hệ

**Files:** Modify `index.html` (`#contact`), `styles.css`, `animations.js`. Test: `check_structure.py` + `spec/9.png`.

**Interfaces:** Consumes `content.contact.*`.

- [ ] **Step 1: Markup #contact**

```html
<section id="contact" class="section contact" aria-label="Thông tin liên hệ">
  <div class="wrap contact-grid">
    <h2 class="title contact-title" data-content="contact.title">THÔNG TIN LIÊN HỆ</h2>
    <figure class="contact-photo"><img data-src="contact.img" data-alt="contact.alt" loading="lazy" alt=""></figure>
    <div class="contact-info">
      <div class="contact-item"><span class="faint" data-content="contact.addressHead">Address</span>
        <p data-content="contact.address"></p></div>
      <div class="contact-item"><span class="faint" data-content="contact.phoneHead">Phone</span>
        <p data-content="contact.phone"></p></div>
      <div class="contact-item"><span class="faint" data-content="contact.emailHead">Email</span>
        <p data-content="contact.email"></p></div>
    </div>
  </div>
</section>
```

- [ ] **Step 2: CSS (append)**

```css
.contact-grid{display:grid;grid-template-columns:auto 1fr;grid-template-rows:auto 1fr;
  gap:clamp(1.5rem,4vw,3rem);align-items:center}
.contact-title{grid-column:1;grid-row:1;font-size:var(--h2);align-self:start}
.contact-photo{grid-column:1;grid-row:2;aspect-ratio:4/5;max-width:420px;overflow:hidden}
.contact-photo img{width:100%;height:100%;object-fit:cover}
.contact-info{grid-column:2;grid-row:2;display:grid;gap:1.75rem;align-self:end}
.contact-item span{display:block;margin-bottom:.25rem;font-size:.85rem}
.contact-item p{font-size:1.15rem}
```

- [ ] **Step 3: Animation (append)**

```js
  gsap.from("#contact .contact-photo", {
    clipPath:"inset(0 100% 0 0)", duration:1.1, ease:"power3.out",
    scrollTrigger:{ trigger:"#contact", start:"top 75%" }
  });
  gsap.from("#contact .contact-item", {
    y:24, opacity:0, duration:.7, ease:"power2.out", stagger:.15,
    scrollTrigger:{ trigger:"#contact", start:"top 75%" }
  });
```

- [ ] **Step 4: Test** — `OK: 9 sections, 21 image refs`
- [ ] **Step 5: Verify vs spec/9.png** — tiêu đề trên trái, ảnh chân dung, cột Address/Phone/Email phải; ảnh reveal ngang, info fade-up.
- [ ] **Step 6: Commit** — `git commit -m "feat(fe): contact section"`

---

### Task 13: Dot navigation + active state

**Files:** Modify `index.html` (nav rỗng `#dotnav` đã có), `js/animations.js` (build dots + active), `css/styles.css`. Test: visual.

**Interfaces:** Consumes 9 section id đã có. Produces `buildDotNav()` gọi trong `initAnimations()`.

- [ ] **Step 1: Thêm buildDotNav vào animations.js (gọi ở đầu initAnimations)**

```js
function buildDotNav(){
  var nav = document.getElementById("dotnav");
  if (!nav) return;
  var secs = document.querySelectorAll("main > section");
  secs.forEach(function(s){
    var a = document.createElement("a");
    a.href = "#" + s.id; a.className = "dot";
    a.setAttribute("aria-label", s.getAttribute("aria-label") || s.id);
    nav.appendChild(a);
    if (window.ScrollTrigger){
      ScrollTrigger.create({ trigger:s, start:"top center", end:"bottom center",
        onToggle:function(self){ a.classList.toggle("is-active", self.isActive); }});
    }
  });
}
```
Và trong `initAnimations()` thêm dòng đầu: `buildDotNav();`

- [ ] **Step 2: CSS dot (append)**

```css
.dot{width:9px;height:9px;border-radius:50%;background:#444;transition:.3s;display:block}
.dot.is-active{background:var(--fg);transform:scale(1.4)}
.dot:hover{background:var(--muted)}
@media (max-width:768px){ .dotnav{display:none} }
```

- [ ] **Step 3: Verify** — reload; thấy cột chấm bên phải, chấm sáng theo section đang xem, click chấm nhảy tới section (Lenis cuộn mượt).
- [ ] **Step 4: Commit** — `git commit -m "feat(fe): dot navigation with active state"`

---

### Task 14: Responsive mobile pass

**Files:** Modify `css/styles.css` (append media queries cho mọi section). Test: visual ở ≤768px.

**Interfaces:** none mới.

- [ ] **Step 1: Append media queries**

```css
@media (max-width:900px){
  .about-grid,.intro-grid,.sgrid-wrap,.contact-grid{grid-template-columns:1fr}
  .wid-grid{grid-template-columns:1fr 1fr}
  .contact-title{grid-row:auto} .contact-photo{grid-row:auto} .contact-info{grid-column:1;grid-row:auto}
}
@media (max-width:768px){
  .hero-grid{grid-template-columns:1fr}
  .hero-text{grid-column:1;grid-row:1}
  .hero-art--guitar,.hero-art--parasol,.hero-art--tiger{grid-column:1;max-width:none;aspect-ratio:16/10}
  .wf-row{grid-template-columns:1fr;gap:.75rem}
  .wf-img{aspect-ratio:16/9}
  .bloom-grid,.gal-grid,.sgrid-grid{grid-template-columns:1fr}
  .wid-grid{grid-template-columns:1fr}
  .intro-photo{position:static}
  .section{min-height:auto}
}
```

- [ ] **Step 2: Verify** — DevTools responsive 375px & 768px: mọi section xếp 1 cột gọn gàng, không tràn ngang, chữ không vỡ.
- [ ] **Step 3: Commit** — `git commit -m "feat(fe): responsive mobile layout"`

---

### Task 15: Motion accessibility, lazy-load & final QA

**Files:** Modify `css/styles.css`, `js/animations.js` (guard reduced-motion). Test: `check_structure.py` + QA toàn trang vs 9 ảnh.

- [ ] **Step 1: Guard prefers-reduced-motion trong initAnimations (bọc đầu hàm)**

```js
  if (window.matchMedia("(prefers-reduced-motion:reduce)").matches){
    buildDotNav();      // vẫn dựng nav
    return;             // bỏ qua mọi hiệu ứng chuyển động
  }
```
(đặt ngay sau `gsap.registerPlugin(ScrollTrigger);`)

- [ ] **Step 2: Chạy toàn bộ test cấu trúc**

Run: `python tools/check_structure.py`
Expected: `OK: 9 sections, 21 image refs`

- [ ] **Step 3: QA toàn trang**

Mở `http://localhost:5500`, cuộn từ trên xuống dưới. Checklist:
- [ ] 9 section hiển thị đúng thứ tự & khớp bố cục `spec/1..9.png`.
- [ ] Không lỗi console.
- [ ] Ảnh không vỡ (21 ảnh tải).
- [ ] Cuộn mượt (Lenis), thanh tiến trình chạy, dot nav active đúng.
- [ ] Bật "Reduce motion" trong OS/DevTools → nội dung vẫn hiển thị đầy đủ, không animation mạnh.
- [ ] Mobile 375px OK.

- [ ] **Step 4: Commit**

```bash
git add src/Portfolio.Api/wwwroot
git commit -m "feat(fe): reduced-motion guard + final QA polish"
```

- [ ] **Step 5: Push**

```bash
git push
```

---

## Self-Review (đã thực hiện)

- **Spec coverage:** 9 section của spec → Task 4–12; smooth scroll/animation lib → Task 1; assets từ spec PNG → Task 2; tách content khỏi markup (chuẩn bị Phase 2) → Task 3; nav/progress/footer → Task 1+13; responsive → Task 14; reduced-motion/lazy → Task 1+15. Vị trí `wwwroot` forward-compatible với BE → Task 1.
- **Placeholder scan:** không có TODO/TBD; mọi bước có code/lệnh thật, nội dung chữ copy nguyên văn từ 9 ảnh.
- **Type consistency:** tên ảnh trong `crop_assets.py` (Task 2) khớp path trong `content.js` (Task 3) và `data-src` các section; `initAnimations()` / `buildDotNav()` / `window.__hydrate` nhất quán giữa Task 1/3/13/15.
- **Ghi chú `reveal`:** class `.reveal` (Task 1) chỉ dùng cho phần tử KHÔNG animate bằng `gsap.from`. Các section dùng `gsap.from` phải bỏ `reveal` khỏi phần tử đó (đã nêu rõ ở Task 4 Step 3 & Task 7 Step 2) để tránh ẩn cứng.

## Ngoài phạm vi (để Phase 2)
- ASP.NET Core project, Clean Architecture, EF Core SQLite, API `/api/content`, trang `/admin`, seed từ content hardcode, đổi `content.js` sang `fetch`. → Plan riêng.
