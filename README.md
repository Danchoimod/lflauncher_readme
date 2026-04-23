<div align="center">

# 🎮 LF Launcher System

**Hệ sinh thái Minecraft đa nền tảng — Website · REST API · Desktop Launcher (Windows) · Android App**

[![Node.js](https://img.shields.io/badge/Node.js-20+-339933?logo=node.js&logoColor=white)](https://nodejs.org/)
[![Next.js](https://img.shields.io/badge/Next.js-16.1.5-000000?logo=next.js&logoColor=white)](https://nextjs.org/)
[![Tauri](https://img.shields.io/badge/Tauri-2-FFC131?logo=tauri&logoColor=white)](https://tauri.app/)
[![Android](https://img.shields.io/badge/Android-API%2026%2B-3DDC84?logo=android&logoColor=white)](https://developer.android.com/)
[![Prisma](https://img.shields.io/badge/Prisma-6.x-2D3748?logo=prisma&logoColor=white)](https://www.prisma.io/)
[![Redis](https://img.shields.io/badge/Redis-5.x-DC382D?logo=redis&logoColor=white)](https://redis.io/)
[![Docker](https://img.shields.io/badge/Docker-OTP%20Service-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)

*Built by LASSCMONE STUDIO — "Built from zero, reaching for miracles."*

</div>

---

## 📖 Giới thiệu

**LF Launcher** là hệ sinh thái launcher Minecraft tự xây dựng, hỗ trợ đa nền tảng (Android & Windows). Dự án tích hợp đầy đủ từ website người dùng, REST API backend, desktop launcher (Tauri/Rust), đến mobile app Android — tất cả dùng chung một lớp xác thực Firebase và cùng một API backend.

Điểm nổi bật:
- **Yggdrasil API** với chữ ký RSA SHA-256, tương thích xác thực profile/skin Minecraft 1.21+
- **Secure Auth** lưu trữ token phía Rust core (integrity checksum SHA-256 + salt)
- **Deep-link protocol** `lflauncher://auth` để đăng nhập từ web về desktop
- **Redis cache** với chiến lược invalidation rõ ràng cho truy vấn đọc cao
- **OTP Service** tách biệt (PHP) để xác minh email khi đăng ký

---

## 📁 Cấu trúc dự án

```
LFLauncher_system/
├── frontend/          # Next.js 16 – Website người dùng (SSR/ISR, SEO-first)
├── backend/           # Node.js + Express 5 – REST API (routes/controllers/services)
├── lflauncher_pc/     # Tauri 2 + React 19 – Desktop launcher Windows
├── mobile/            # Android (Java/Kotlin + C++20 JNI) – Mobile app
├── OTP_PHP/           # PHP OTP Service – Docker + Nginx + MySQL
├── admin/             # Next.js – Trang quản trị nội bộ
├── cdn/               # Dữ liệu tĩnh: patchnotes.json, metadata phiên bản
├── core_logic/        # Logic dùng chung
├── old_website/       # Website legacy (React + Vite) – đang migrate dần
├── logion/            # Module xác thực bổ sung
├── tunnel_ba/         # Tunneling / proxy phụ trợ
└── debug/             # Scripts debug và kiểm tra hệ thống
```

---

## 🧩 Các thành phần chính

### 1. `frontend/` — Website người dùng
> **Stack:** Next.js 16.1.5 · React 19 · TypeScript · NextAuth v5 · Tailwind CSS 4 · Firebase · SweetAlert2

Website SEO-first với App Router (Server Components mặc định). Hỗ trợ các trang:

| Route | Chức năng |
|-------|-----------|
| `/` | Home — Carousel + Packages grid + FAQ (structured data) |
| `/download` | Tải launcher theo platform |
| `/[slug]` | Trang chi tiết package/mod |
| `/login`, `/signup` | Đăng nhập / Đăng ký |
| `/profile` | Trang cá nhân, quản lý skin, avatar |
| `/user/[slug]` | Profile tác giả |
| `/search` | Tìm kiếm package |
| `/blog` | Blog và patch notes |
| `/about-us`, `/contact` | Thông tin Studio |
| `/android-install-guide`, `/windows-install-guide` | Hướng dẫn cài đặt |
| `/privacy-policy`, `/terms` | Pháp lý |
| `/project` | Danh sách dự án Studio |
| `/auth/*` | Auth callback (Discord, success redirect) |

Luồng đăng nhập từ launcher:
```
Mở trình duyệt với ?launcher=true
  → Đăng nhập qua NextAuth (Email / Google / Discord)
  → Lấy profile + token từ backend
  → Redirect: lflauncher://auth?name=...&avatar=...&token=...&type=...
  → Desktop app nhận deep-link → đăng nhập hoàn tất
```

---

### 2. `backend/` — REST API
> **Stack:** Node.js · Express 5 · Prisma ORM 6 · MySQL · Redis · Firebase Admin 13 · JWT · Axios

#### Kiến trúc
Tổ chức theo mô hình `routes → controllers → services`, với error handling tập trung qua `AppError` + `catchAsync`.

**Entry point:** `index.js` — khởi động Express, kết nối Prisma + Redis, xử lý graceful shutdown (SIGTERM/SIGINT).

**Middleware:**
- `checkAuth` — xác thực Firebase ID Token từ header `Authorization: Bearer <token>`
- `errorHandler` — xử lý lỗi tập trung, trả về response chuẩn

#### API Endpoints

**Auth** (`/api/auth`)

| Method | Endpoint | Mô tả | Auth |
|--------|----------|-------|------|
| POST | `/login` | Đăng nhập email/password qua Firebase REST API | - |
| POST | `/signup` | Đăng ký → tạo Firebase user (disabled) → gửi OTP | - |
| POST | `/verify-otp` | Xác minh OTP → kích hoạt tài khoản | - |
| POST | `/google` | Đăng nhập bằng Google ID Token | - |
| GET | `/discord/url` | Lấy Discord OAuth2 URL | - |
| GET | `/discord/callback` | Discord callback → redirect về frontend | - |
| POST | `/discord` | Đăng nhập Discord (code exchange) | - |
| POST | `/refresh` | Làm mới Firebase ID Token | - |
| POST | `/logout` | Thu hồi toàn bộ refresh token Firebase | ✅ |

**Packages** (`/api/packages`)

| Method | Endpoint | Mô tả | Auth |
|--------|----------|-------|------|
| GET | `/` | Danh sách packages (phân trang, filter, search) | - |
| GET | `/search` | Tìm kiếm theo title/shortSummary | - |
| GET | `/:id` | Chi tiết package (có thể dùng slug) | - |
| GET | `/me` | Packages của tôi | ✅ |
| POST | `/me` | Tạo package mới (status=0, chờ duyệt) | ✅ |
| GET | `/me/:id` | Chi tiết package của tôi | ✅ |
| PATCH | `/me/:id` | Cập nhật package (reset về status=0) | ✅ |
| DELETE | `/me/:id` | Xóa package | ✅ |
| POST | `/:id/rate` | Chấm điểm package (upsert) | ✅ |
| GET | `/:id/my-rate` | Rating của tôi cho package | ✅ |

**Các nhóm khác**

| Nhóm | Prefix | Các endpoint chính |
|------|--------|-------------------|
| Categories | `/api/categories` | GET all, GET `/:slug/packages` |
| Comments | `/api/comments` | GET by package, POST (create), DELETE /:id |
| Users | `/api/users` | GET following, PATCH profile, POST follow, GET `/:slug/profile` |
| App Updates | `/api/app-updates` | GET all, GET `/latest/:platform`, POST |
| Carousels | `/api/carousels` | GET all |
| Versions | `/api/versions` | GET all |
| Reports | `/api/reports` | POST (tạo báo cáo) |
| Yggdrasil | `/api/yggdrasil` | Xem mục Yggdrasil bên dưới |
| General | `/api` | GET `/me` (profile tôi), GET `/xinchao` |

#### Database Schema (Prisma + MySQL)

```
User ─────────┬── Package ───┬── Image
              │              ├── Comment (self-relation: parent/replies)
              │              ├── Rating (unique: userId+packageId)
              │              ├── Url
              │              ├── Version
              │              ├── Carousel
              │              └── Report
              ├── Follow (self-relation: follower/following)
              ├── Carousel
              └── Report (FiledReports / ReceivedReports)

Category ─────┬── Package (many)
              └── Carousel (many)
              (self-relation: parent/children hierarchy)

AppUpdate ─── platform, versionName, versionCode, isForce, downloadUrl, changelog
```

#### Cache Strategy (Redis)
- Package list: cache 10 phút, key `lfl-cache-prod:packages-all:{query}`
- Package detail: cache 30 phút, key `lfl-cache-prod:package-detail:{id|slug}`
- Invalidation khi create/update/delete: xóa cache bằng wildcard

---

### 3. `lflauncher_pc/` — Desktop Launcher (Windows)
> **Stack:** Tauri 2 · Vite 7 · React 19 · TypeScript · Tailwind CSS 4 · Framer Motion · skinview3d

#### Frontend (React/TS)
Cấu trúc module hóa:
```
src/
├── App.tsx           # Entry: deep-link listener, auth state, splash screen
├── pages/
│   ├── LoginPage.tsx # Màn hình đăng nhập
│   └── Dashboard.tsx # Màn hình chính sau đăng nhập
└── modules/
    ├── auth/         # UI xác thực
    ├── launcher/     # UI launcher, play, download
    └── shared/       # Components dùng chung (LoadingOverlay, SplashScreen, ...)
```

**App.tsx** xử lý:
- Lắng nghe deep-link `lflauncher://auth?name=...&avatar=...&token=...` qua Tauri plugin
- Single-instance event (nếu app đã mở, forward deep-link về instance hiện tại)
- Kiểm tra auth persistence qua Rust secure store khi khởi động
- Lazy load `LoginPage` và `Dashboard` để giảm bundle size
- Gọi `invoke("get_all_version_sources")` để fetch danh sách Minecraft versions từ Rust

#### Backend (Rust / Tauri)
```
src-tauri/src/
├── main.rs           # Tauri setup, single-instance plugin, commands register
├── lib.rs            # Tauri builder
├── models.rs         # Struct: Profile, MinecraftVersion, Library, ...
└── modules/
    ├── auth/
    │   └── auth.rs   # save_secure_user, get_secure_user, logout_secure_user, sync_user_from_backend
    └── game/
        ├── game.rs   # MinecraftLauncher struct
        ├── launch.rs # Build JVM args, classpath, launch process
        ├── install.rs# Tải về và cài đặt Minecraft (libraries, assets, natives)
        ├── java.rs   # Phát hiện / tải Java runtime
        ├── versions.rs # Fetch version manifest từ Mojang
        └── commands.rs # Tauri commands expose ra frontend
```

**Secure Auth (Rust):**
- Lưu user data vào `auth.db` trong AppData dir
- Tính SHA-256 checksum (data + secret salt) để chống giả mạo
- Nếu checksum sai khi đọc → force logout + hiện `SecurityWarningModal`

**Game Launch (Rust):**
- Parse `version_data.json` của Minecraft
- Xây dựng classpath (vanilla + mod loader Maven) và JVM arguments đầy đủ
- Thay thế biến: `${auth_player_name}`, `${auth_uuid}`, `${auth_access_token}`, `${classpath}`, ...
- Detect Java: theo profile setting → theo `javaVersion.component` trong manifest → fallback hệ thống
- Convert đường dẫn sang short-path Windows để tránh lỗi khoảng trắng
- Stream stdout/stderr về frontend qua Tauri event `game-log`

---

### 4. `mobile/` — Android App
> **Stack:** Java · Android SDK (API 26+ / target 36) · CMake/C++20 · Retrofit 2 · Room · Glide · Security-Crypto

**Package:** `com.lasscmone.lflauncher` · Version: `1.1.4.0`

```
app/src/main/java/com/lasscmone/lflauncher/
├── view/activities/
│   ├── SplashActivity.java   # Màn hình khởi động
│   ├── MainActivity.java     # Activity chính (23KB — logic lớn)
│   └── BaseActivity.java     # Base class chung
├── api/
│   ├── ApiService.java       # Retrofit interface
│   ├── ApiResponse.java      # Wrapper response
│   └── NativeConfig.java     # Config cho JNI
├── manager/
│   ├── DownloadManager.java  # Quản lý tải file
│   └── SessionManager.java  # Quản lý phiên đăng nhập
├── model/                   # Data models
├── repository/              # Repository pattern
├── database/                # Room database
├── dto/                     # Data Transfer Objects
├── utils/                   # Utilities
├── adapter/                 # RecyclerView adapters
└── view/adapter/fragment/custom/  # UI components
```

**AndroidManifest:**
- Quyền: INTERNET, READ/WRITE_EXTERNAL_STORAGE, MANAGE_EXTERNAL_STORAGE, REQUEST_INSTALL_PACKAGES
- Deep-link: `lflauncher://auth` (singleTask, BROWSABLE)
- FileProvider để chia sẻ file APK/MCPACK
- `queries`: detect xem Minecraft PE (`com.mojang.minecraftpe`) đã cài chưa

**Native C++ (CMake/C++20):**
```
app/src/main/cpp/lasscmone/
```
Build cho `arm64-v8a`, `armeabi-v7a`, `x86_64`.

---

### 5. `OTP_PHP/` — Dịch vụ OTP
> **Stack:** PHP · Nginx · Docker · MySQL

Dịch vụ độc lập nhận JWT payload từ backend:
- **Gửi OTP:** nhận JWT (`action: REGISTER_VERIFY`) → sinh mã → gửi email
- **Xác minh OTP:** nhận JWT (`action: VERIFY_OTP`) + `code` → trả `{status: "success"}`

JWT OTP được ký bằng `JWT_OTP_SECRET`, có thời hạn 5 phút.

---

### 6. `OTP_PHP/` — Yggdrasil API
> Được mount tại `/api/yggdrasil` trong backend

| Endpoint | Mô tả |
|----------|-------|
| `GET /api/yggdrasil/` | Metadata: serverName, skinDomains, publicKey. Trả header `X-Authlib-Injector-API-Location` |
| `GET /api/yggdrasil/sessionserver/session/minecraft/profile/:uuid` | Profile + textures theo UUID |
| `POST /api/yggdrasil/sessionserver/session/minecraft/join` | Lưu session khi client vào server game |
| `GET /api/yggdrasil/sessionserver/session/minecraft/hasJoined` | Server game verify player |
| `GET /api/yggdrasil/minecraft/profile` | Profile của player đang đăng nhập (Bearer token) |

- Textures: base64 encode JSON → ký RSA SHA-256 → trả `signature`
- UUID dạng undashed (32 ký tự) được normalize về dạng hyphenated trước khi tìm DB
- Token `"0"` (offline mode) được xử lý riêng, trả profile rỗng thay vì 401
- Session map lưu tạm trong RAM (30s timeout) — production nên migrate sang Redis

---

## 🔐 Luồng xác thực chi tiết

### Đăng ký
```
POST /api/auth/signup
  → Tạo Firebase user (disabled=true)
  → Tạo user nội bộ (status=0, pending)
  → Ký JWT OTP (5 phút) → gửi sang OTP_PHP service
  → Trả về thông tin user + idToken

POST /api/auth/verify-otp  { email, code }
  → Ký JWT verification → gọi OTP_PHP để verify code
  → Cập nhật user.status = 1 trong DB
  → Bật lại Firebase user (disabled=false)
  → Tài khoản kích hoạt thành công
```

### Đăng nhập
```
POST /api/auth/login  { email, password }
  → Kiểm tra status trong DB (nếu status=0 → gửi lại OTP → trả 403)
  → Gọi Firebase REST API để xác thực
  → Trả idToken + refreshToken

POST /api/auth/refresh  { refreshToken }
  → Gọi Firebase securetoken API
  → Trả idToken mới
```

### Discord OAuth2
```
GET  /api/auth/discord/url     → lấy OAuth2 URL
GET  /api/auth/discord/callback → code exchange → sync Firebase + DB → redirect về frontend
POST /api/auth/discord          → code exchange → trả idToken (dùng cho SPA)
```

---

## 🏗️ Kiến trúc tổng thể

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                             │
│  [Browser / Web (Next.js)]   [Desktop (Tauri/Rust)]   [Android]│
└──────────┬────────────────────────┬──────────────┬─────────────┘
           │ HTTPS                  │ deep-link    │ HTTP (local)
           │                        │ lflauncher:// │
           ▼                        ▼               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    BACKEND API (Express 5)                      │
│   routes/  →  controllers/  →  services/                        │
│   checkAuth middleware (Firebase Admin verifyIdToken)           │
├──────────────┬────────────────┬────────────────┬───────────────┤
│   MySQL      │     Redis      │  Firebase Auth │  OTP_PHP      │
│  (Prisma ORM)│  (Cache 10-30m)│  (Admin SDK)   │  (Docker/Nginx)│
└──────────────┴────────────────┴────────────────┴───────────────┘
```

---

## ⚡ Quick Start

### Yêu cầu
| Tool | Version |
|------|---------|
| Node.js | 20+ |
| npm | 10+ |
| Docker | any (cho OTP_PHP) |
| Android Studio | any (cho mobile) |
| Rust + Tauri CLI | v2 (cho desktop) |
| MySQL | 8+ |
| Redis | 6+ |

### Backend
```bash
cd backend
cp .env.example .env   # Điền DATABASE_URL, FIREBASE_API_KEY, JWT_OTP_SECRET, REDIS_URL, OTP_SERVICE_URL...
npm install
npm run dev            # node --watch index.js → http://localhost:3000
```

### Frontend
```bash
cd frontend
# Tạo .env.local với NEXT_PUBLIC_API_URL, NEXTAUTH_SECRET, NEXTAUTH_URL, Firebase config...
npm install
npm run dev            # http://localhost:3001 (hoặc 3000 nếu backend chạy port khác)
```

### Desktop Launcher
```bash
cd lflauncher_pc
npm install
npm run tauri dev      # Cần Rust + Tauri CLI v2
```

### OTP Service
```bash
cd OTP_PHP
docker-compose up -d
```

---

## 🔧 Biến môi trường

### `backend/.env`
```env
DATABASE_URL="mysql://user:password@localhost:3306/lflauncher"
FIREBASE_API_KEY=          # Firebase Web API Key
JWT_OTP_SECRET=            # Secret ký JWT gửi sang OTP service
OTP_SERVICE_URL=           # URL PHP OTP endpoint
REDIS_URL=                 # Redis connection string
DISCORD_CLIENT_ID=
DISCORD_CLIENT_SECRET=
DISCORD_REDIRECT_URI=
FRONTEND_URL=              # URL frontend (dùng cho redirect)
PORT=3000
NODE_ENV=development
```

### `frontend/.env.local`
```env
NEXT_PUBLIC_API_URL=       # URL backend API
NEXTAUTH_URL=              # URL của chính frontend
NEXTAUTH_SECRET=           # Secret cho NextAuth
# Firebase client config...
# Google / Discord OAuth credentials...
```

> ⚠️ Không commit `.env` thực tế. Tham khảo `.env.example` trong từng thư mục.

---

## 📦 Cấu trúc Database chi tiết

| Model | Trường quan trọng | Ghi chú |
|-------|-------------------|---------|
| `User` | uid (UUID), firebaseUid, slug, status (0/1) | status=0: pending, =1: active |
| `Package` | slug (`title-id`), status (0=pending/1=published), ratingCount, ratingAvg | Luôn tạo ở status=0 |
| `Comment` | parentId (self-relation) | Hỗ trợ comment lồng nhau |
| `Rating` | unique(userId, packageId) | Dùng upsert |
| `Follow` | unique(followerId, followingId) | Tự quan hệ trên User |
| `Category` | parentId (self-relation) | Hỗ trợ category cha-con |
| `AppUpdate` | platform, versionCode, isForce | Android + Windows |
| `Carousel` | imageUrl, catId, packageId | Banner homepage |

---

## 🗺️ Roadmap

- [ ] Unit test cho backend service layer
- [ ] CI/CD pipeline (frontend / backend / mobile)
- [ ] API documentation bằng OpenAPI/Swagger
- [ ] Migrate session map Yggdrasil từ RAM sang Redis
- [ ] Observability: logs, metrics, tracing (production)
- [ ] Hỗ trợ Fabric / Quilt / NeoForge trong desktop launcher
- [ ] Linux build cho desktop launcher

---

## 🛠️ Tech Stack tóm tắt

| Layer | Công nghệ |
|-------|-----------|
| **Backend** | Node.js 20, Express 5, Prisma ORM 6, MySQL, Redis, JWT, Firebase Admin 13, Axios |
| **Frontend** | Next.js 16.1.5, React 19, TypeScript, NextAuth v5, Tailwind CSS 4, Firebase |
| **Desktop (UI)** | React 19, TypeScript, Tailwind CSS 4, Framer Motion, skinview3d |
| **Desktop (Core)** | Rust (Tauri 2), tokio, reqwest, serde, sha2, hex |
| **Mobile** | Java (Android API 26+), C++20 JNI/CMake, Retrofit 2, Room, Glide, Security-Crypto |
| **OTP Service** | PHP, Nginx, MySQL, Docker |
| **Infrastructure** | Firebase Auth, Redis Cache, Docker, REST API |

---

## 📄 License

Dự án hiện dùng nội bộ bởi **LASSCMONE STUDIO**. Sẽ bổ sung LICENSE chính thức nếu mở source.

---

<div align="center">

*Phát triển từ 08/2023 · Fullstack Developer (Backend-first, Đa nền tảng)*

*Chi tiết kỹ thuật để phỏng vấn: xem [`CV_PROJECT_DESCRIPTION.md`](./CV_PROJECT_DESCRIPTION.md)*

</div>
