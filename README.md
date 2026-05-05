<div align="center">

# 🎮 LF Launcher System
### Hệ sinh thái Launcher Minecraft đa nền tảng — 7,700+ người dùng

[![Node.js](https://img.shields.io/badge/Node.js-20+-339933?logo=node.js&logoColor=white)](https://nodejs.org/)
[![Next.js](https://img.shields.io/badge/Next.js-16-000000?logo=next.js&logoColor=white)](https://nextjs.org/)
[![Tauri](https://img.shields.io/badge/Tauri-2_(Rust)-FFC131?logo=tauri&logoColor=white)](https://tauri.app/)
[![Android](https://img.shields.io/badge/Android-Java%2FC++-3DDC84?logo=android&logoColor=white)](https://developer.android.com/)
[![Prisma](https://img.shields.io/badge/Prisma%20ORM-MySQL-2D3748?logo=prisma&logoColor=white)](https://www.prisma.io/)
[![Redis](https://img.shields.io/badge/Redis-Cache-DC382D?logo=redis&logoColor=white)](https://redis.io/)

</div>

---

## 🚀 Dự án là gì?

Một **hệ sinh thái hoàn chỉnh tự xây dựng** — giúp người dùng khám phá, cài đặt và chơi Minecraft trên cả Android lẫn Windows, dưới một lớp xác thực thống nhất.

Tự thiết kế, tự lập trình và vận hành từ tháng 08/2023 đến nay.

---

## 📈 Kết quả thực tế

| Chỉ số | Con số |
|--------|--------|
| 🎮 Tổng người dùng | **7,700+** (Discord Auth) |
| 📦 Người dùng mới mỗi lần release | **1,000 – 2,000** |
| 🌐 Nền tảng hỗ trợ | Android + Windows |
| ⏱️ Thời gian vận hành | Production từ 2023 đến nay |

##changelog
<img width="1277" height="52" alt="image" src="https://github.com/user-attachments/assets/d7eac4e7-053b-4672-9bc1-8244405daf3b" />
<img width="1264" height="50" alt="image" src="https://github.com/user-attachments/assets/93f58ea6-efe4-43cd-befa-311580918a39" />
<img width="1270" height="60" alt="image" src="https://github.com/user-attachments/assets/1a538fd7-8ba8-40db-b0ca-a412fa572e6c" />
<img width="1290" height="49" alt="image" src="https://github.com/user-attachments/assets/7c491df1-c355-45bd-ab09-75bb5fcff89e" />



---

## 🧩 Tổng quan hệ thống

```
┌────────────────────────────────────────────────────────┐
│  Web (Next.js)     ──► REST API ──► MySQL + Redis      │
│  Desktop (Tauri)   ──► deep-link ──► REST API          │
│  Android (Java/JNI)──► Retrofit  ──► REST API          │
│                                                        │
│  Tất cả dùng chung một lớp auth (Firebase + JWT)       │
└────────────────────────────────────────────────────────┘
```

| Thành phần | Stack | Vai trò |
|-----------|-------|---------|
| **Backend API** | Node.js · Express 5 · Prisma · MySQL · Redis | Auth, packages, tương tác xã hội, cập nhật launcher |
| **Website** | Next.js 16 · React 19 · NextAuth v5 | Cổng thông tin SEO-first, nội dung cộng đồng |
| **Desktop Launcher** | Tauri 2 · Rust · React 19 | Chạy Minecraft Java/Bedrock trên Windows |
| **Android App** | Java · C++20 JNI · CMake | Launcher + quản lý nội dung cho Android |
| **OTP Service** | PHP · Nginx · Docker | Xác minh email OTP tách biệt |

---

## 🧠 Điểm kỹ thuật nổi bật

### 1. Yggdrasil API tùy biến (Tương thích Minecraft 1.21+)
Tự triển khai toàn bộ giao thức Yggdrasil — chuẩn mà Minecraft dùng để xác thực người chơi và tải skin.

```
Client kết nối server game
  → POST /sessionserver/session/minecraft/join    (lưu session)
  → GET  /sessionserver/session/minecraft/hasJoined  (server verify)
  → GET  /minecraft/profile  (lấy skin / textures)
```

- Textures được ký bằng **RSA SHA-256** (bắt buộc từ MC 1.21+)
- Hỗ trợ `authlib-injector` qua header `X-Authlib-Injector-API-Location`
- Xử lý offline token `"0"` để tránh crash phía client

---

### 2. Deep-link Auth Flow (Web → Desktop)
Đăng nhập một lần duy nhất trên trình duyệt, Desktop tự nhận token — không cần hệ thống credential riêng biệt.

```
Người dùng đăng nhập trên website (Email / Google / Discord)
  → Backend cấp token + profile
  → Website redirect: lflauncher://auth?token=...&name=...&avatar=...
  → Desktop app (Tauri) bắt deep-link
  → Token lưu an toàn trong Rust core (auth.db)
  → Dashboard mở khóa
```

Xử lý **single-instance**: nếu app đang mở, instance thứ hai forward URL về process đang chạy.

---

### 3. Lưu trữ auth an toàn phía Rust
Token không bao giờ lưu dạng plain text. Rust core quản lý toàn bộ credential.

```rust
// Kiểm tra tính toàn vẹn mỗi lần đọc
fn calculate_checksum(data: &UserData) -> String {
    SHA-256( JSON(data) + SECRET_SALT )
}
// Checksum sai → force logout + hiện SecurityWarningModal
```

Khi khởi động: đọc `auth.db` → xác minh checksum → đồng bộ profile từ `GET /api/me` → cập nhật local store.

---

### 4. Redis Cache với chiến lược invalidation rõ ràng

| Tài nguyên | TTL | Khi nào xóa cache |
|------------|-----|-------------------|
| Danh sách package | 10 phút | Tạo / Cập nhật / Xóa |
| Chi tiết package | 30 phút | Cập nhật / Xóa |

Key dùng wildcard delete (`packages-all:*`) để xử lý mọi biến thể query. Tự động bỏ qua nếu Redis offline.

---

### 5. Luồng kích hoạt tài khoản (Pending → Active)

```
POST /signup
  → Tạo Firebase user (disabled=true)  ← chặn đăng nhập trước khi xác minh
  → Tạo user DB (status=0)
  → Ký JWT OTP (TTL 5 phút) → gửi sang PHP service → gửi email

POST /verify-otp
  → PHP service xác minh mã
  → DB: status = 1
  → Firebase: disabled = false
  → Tài khoản được kích hoạt

POST /login (nếu status=0)
  → Chặn đăng nhập + tự gửi lại OTP
  → Trả 403 kèm thông báo rõ ràng
```

---

### 6. Engine khởi động Minecraft (Rust)
Desktop launcher **thực sự chạy Minecraft** — không chỉ quản lý nội dung.

- Parse manifest JSON chính thức của Minecraft
- Xây dựng classpath đầy đủ cho vanilla + mod loader (Fabric/Forge/NeoForge — Maven format)
- Thay thế toàn bộ biến manifest: `${auth_uuid}`, `${classpath}`, `${natives_directory}`...
- Convert đường dẫn sang Windows short-path để tránh lỗi khoảng trắng trong path
- Tự phát hiện Java runtime từ `%APPDATA%/.minecraft/runtime/` hoặc PATH hệ thống
- Stream `stdout`/`stderr` về giao diện qua Tauri event theo thời gian thực

---

## 🔑 Các phương thức xác thực

| Phương thức | Cơ chế |
|-------------|--------|
| Email + Password | Firebase REST API → kiểm tra status → JWT |
| Google | Firebase ID Token → Admin SDK verify → đồng bộ DB |
| Discord OAuth2 | Code exchange → lấy `/users/@me` → sync Firebase + DB → Custom Token |
| Refresh Token | Firebase `securetoken` API → cấp `idToken` mới |
| Đăng xuất | `revokeRefreshTokens` (Firebase Admin) — thu hồi toàn bộ phiên |

Mọi endpoint có xác thực đều qua middleware `checkAuth`: verify Firebase `idToken` bằng Admin SDK, gắn `req.user`.

---

## 🗄️ Mô hình dữ liệu (MySQL + Prisma)

```
User ──────── Package ──── Image, Url, Version
                     └──── Comment (lồng nhau: self-relation parentId)
                     └──── Rating  (unique userId+packageId, upsert)
                     └──── Report

User ──────── Follow   (self-relation, unique follower+following)
Category ──── Package  (phân cấp cha-con: self-relation parentId)
AppUpdate ─── platform, versionCode, isForce, downloadUrl, changelog
```

Slug package theo dạng `{tiêu-đề-slugified}-{id}` — thân thiện SEO, không trùng lặp.  
Package luôn tạo ở `status=0` (chờ duyệt), publish ở `status=1`.

---

## ⚡ Khởi động nhanh

```bash
# Backend
cd backend && cp .env.example .env && npm install && npm run dev

# Frontend
cd frontend && npm install && npm run dev

# Desktop (cần Rust + Tauri CLI v2)
cd lflauncher_pc && npm install && npm run tauri dev

# OTP Service
cd OTP_PHP && docker-compose up -d
```

**Biến môi trường chính (backend):** `DATABASE_URL` · `FIREBASE_API_KEY` · `JWT_OTP_SECRET` · `OTP_SERVICE_URL` · `REDIS_URL` · `DISCORD_CLIENT_ID/SECRET`

---

## 🛠️ Tech Stack tóm tắt

```
Backend:  Node.js 20 · Express 5 · Prisma ORM · MySQL · Redis · Firebase Admin · JWT
Web:      Next.js 16 · React 19 · TypeScript · NextAuth v5 · Tailwind CSS 4
Desktop:  Tauri 2 · Rust (tokio, reqwest, sha2, serde) · React 19 · Framer Motion
Android:  Java · C++20/CMake/JNI · Retrofit 2 · Room · Glide · Security-Crypto
DevOps:   Docker · Nginx (OTP service) · Firebase Auth
```

---

<div align="center">

**Fullstack Developer · Backend-first · 08/2023 – Hiện tại**

</div>
