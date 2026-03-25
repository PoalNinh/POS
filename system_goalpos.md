# 📋 Đánh Giá Hệ Thống POS — Bảo Mật, Khả Năng Mở Rộng & Công Nghệ

> Tài liệu dành cho khách hàng — đánh giá toàn diện hệ thống quản lý POS (Point of Sale).

---

## 1. Tổng Quan Công Nghệ

| Thành phần | Công nghệ | Phiên bản | Ghi chú |
|---|---|---|---|
| **Framework** | Next.js (App Router) | 16.1.6 | Framework React hàng đầu, hỗ trợ SSR/SSG/Edge Runtime |
| **Ngôn ngữ** | TypeScript | 5.x | Type-safe, giảm lỗi runtime |
| **Database** | PostgreSQL + Prisma ORM | Prisma 6.19.2 | RDBMS  + ORM type-safe |
| **UI Library** | Shadcn UI + Radix UI | Latest | Accessible, customizable, production-grade |
| **State Management** | TanStack Query | 5.90+ | Cache, sync, pagination tự động |
| **Styling** | Tailwind CSS | 4.x | Utility-first, responsive, tree-shaking |
| **Auth** | JWT (jose) + bcrypt | jose 6.1, bcrypt 3.0 | Edge Runtime compatible |
| **Encryption** | AES-256-GCM | Node.js crypto | Mã hóa bí mật API trong database |
| **File Storage** | Cloudinary | 2.9 | CDN toàn cầu, xử lý ảnh tự động |
| **Runtime** | Node.js | ≥20.9.0 | LTS, hỗ trợ dài hạn |

---

## 2. Kiến Trúc Bảo Mật 🔒

### 2.1. Xác Thực (Authentication)

| Tính năng | Chi tiết | Mức độ |
|---|---|---|
| **Mã hóa mật khẩu** | bcrypt với salt rounds = 12 | ✅ Tiêu chuẩn ngành |
| **JWT Token** | HMAC-SHA256 (HS256) qua thư viện `jose` | ✅ Edge Runtime compatible |
| **Token hết hạn** | 1 ngày (24 giờ), tự động redirect khi expired | ✅ Giảm rủi ro token bị đánh cắp |
| **HttpOnly Cookie** | Token lưu trong cookie `httpOnly: true` | ✅ Chống XSS đánh cắp token |
| **Secure Cookie** | `secure: true` trên production (HTTPS only) | ✅ Chống MITM |
| **SameSite** | `lax` — chống CSRF cơ bản | ✅ |
| **Session tracking** | Model `PhienDangNhap` — ghi nhận thiết bị, IP, thời gian | ✅ Audit trail đăng nhập |
| **Session revocation** | Hỗ trợ thu hồi session (Revoked status) | ✅ Khóa phiên từ xa |

### 2.2. Phân Quyền (Authorization — RBAC)

| Tính năng | Chi tiết |
|---|---|
| **Vai trò hệ thống** | `QuanTriVien`, `NhanVien`, `KhachHang` — 3 cấp rõ ràng |
| **RBAC trên API** | [withAuth()](file:///e:/Goalcrm/AppGoal/pos-goal-v1/src/lib/api-handler.ts#17-86) wrapper kiểm tra `allowedRoles` trước khi thực thi |
| **Phân quyền chi tiết** | Model `PhanQuyen` — quyền theo phòng ban + chức vụ: `xem`, `them`, `sua`, `xoa` |
| **Cô lập dữ liệu** | Mỗi cửa hàng chỉ thấy dữ liệu của mình (Store Isolation via `cuaHangId`) |

### 2.3. Bảo Vệ API

| Biện pháp | Triển khai |
|---|---|
| **Rate Limiting** | In-memory rate limiter — giới hạn request/IP/endpoint (VD: 5 req/60s cho login) |
| **Input Sanitization** | [sanitizeBody()](file:///e:/Goalcrm/AppGoal/pos-goal-v1/src/lib/api-handler.ts#129-168) — loại bỏ `__proto__`, `constructor`, `prototype`, `id`, `createdAt` trước khi write DB |
| **Prototype Pollution Protection** | Recursive sanitize, chặn injection qua nested objects |
| **Prisma Error Handling** | Tự động xử lý `P2002` (trùng dữ liệu), `P2025` (không tìm thấy) |
| **Centralized Error Handler** | [withAuth()](file:///e:/Goalcrm/AppGoal/pos-goal-v1/src/lib/api-handler.ts#17-86) bắt mọi exception, trả `{ error }` chuẩn — không leak stack trace |

### 2.4. Security Headers (Middleware)

Tất cả response đều được gắn headers bảo mật qua [proxy.ts](file:///e:/Goalcrm/AppGoal/pos-goal-v1/src/proxy.ts):

| Header | Giá trị | Tác dụng |
|---|---|---|
| `X-Content-Type-Options` | `nosniff` | Chống MIME sniffing |
| `X-Frame-Options` | `DENY` | Chống Clickjacking |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Giới hạn thông tin referrer |
| `X-XSS-Protection` | `1; mode=block` | Chống XSS (legacy browsers) |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` | Vô hiệu hóa API thiết bị nhạy cảm |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains; preload` | Bắt buộc HTTPS (production) |

### 2.5. Mã Hóa Dữ Liệu Nhạy Cảm

| Tính năng | Chi tiết |
|---|---|
| **Thuật toán** | AES-256-GCM — mã hóa đối xứng mạnh nhất |
| **Áp dụng** | API Secret (Cloudinary) được mã hóa trước khi lưu DB |
| **Format** | `iv:authTag:ciphertext` (hex-encoded) |
| **Backward compatible** | Tự phát hiện dữ liệu plaintext cũ, không bị lỗi |

### 2.6. Nhật Ký Hệ Thống (Audit Log)

| Tính năng | Chi tiết |
|---|---|
| **Ghi nhận** | Mọi thao tác CÀI ĐẶT/TẠO/SỬA/XÓA/DUYỆT/NHẬP DỮ LIỆU |
| **Thông tin** | Ai thực hiện, thời gian, IP, User-Agent, nội dung thay đổi |
| **Diff chi tiết** | So sánh dữ liệu trước/sau, ghi rõ từng trường thay đổi |
| **Smart skip** | Không ghi log nếu dữ liệu không thực sự thay đổi |
| **Fire-and-forget** | Không ảnh hưởng performance của API chính |
| **Bảo vệ** | Không log mật khẩu, bỏ qua field hệ thống (`id`, `createdAt`) |

---

## 3. Khả Năng Mở Rộng (Scalability) 📈

### 3.1. Database Scalability

| Tính năng | Chi tiết |
|---|---|
| **PostgreSQL** | Hỗ trợ horizontal scaling (read replicas), partitioning, JSONB indexes |
| **Connection Pooling** | Tự động `connection_limit=10` + `pool_timeout=10` (serverless-ready) |
| **Singleton Pattern** | Prisma Client singleton — tránh connection leak |
| **Database Indexes** | 20+ index trên các trường hay query (`createdAt DESC`, `cuaHangId`, `trangThai`, v.v.) |
| **Prisma Accelerate** | Sẵn sàng tích hợp cho production (connection pooling + caching) |
| **PgBouncer** | Compatible — có thể thêm connection pooler bên ngoài |

### 3.2. Application Scalability

| Tính năng | Chi tiết |
|---|---|
| **Serverless-ready** | Next.js App Router — auto-scales trên Vercel, AWS Lambda |
| **Edge Runtime** | Middleware chạy trên Edge (nhanh hơn, gần user hơn) |
| **API Pagination** | [parsePagination()](file:///e:/Goalcrm/AppGoal/pos-goal-v1/src/lib/api-handler.ts#87-103) + [paginationMeta()](file:///e:/Goalcrm/AppGoal/pos-goal-v1/src/lib/api-handler.ts#104-115) chuẩn — xử lý dataset lớn |
| **Large Dataset** | Manual pagination + server-side filter cho >1000 rows |
| **Client-side Cache** | TanStack Query — staleTime, refetch on focus, cache invalidation |
| **Search Debounce** | 500ms debounce — giảm API calls |

### 3.3. Multi-Store Architecture

| Tính năng | Chi tiết |
|---|---|
| **Cô lập dữ liệu** | Mỗi cửa hàng = 1 tenant riêng, filter tự động qua `cuaHangId` |
| **10+ models** | `TaiKhoan`, `PhongBan`, `SanPham`, `DonHang`, v.v. đều hỗ trợ multi-store |
| **Bảo vệ cross-store** | API verify `cuaHangId` match trước khi cho phép sửa/xóa |
| **Admin toàn quyền** | QuanTriVien không gắn `cuaHangId` → xem toàn bộ dữ liệu |

### 3.4. File Storage

| Tính năng | Chi tiết |
|---|---|
| **Cloudinary CDN** | File được lưu trên CDN toàn cầu — load nhanh mọi nơi |
| **Multi-account** | Hỗ trợ nhiều tài khoản Cloudinary, tự động chọn active account |
| **Giới hạn** | Ảnh: 5MB, File: 10MB — chống abuse |

---


## 7. Luồng Hoạt Động Hệ Thống 🔄

### 7.1. Luồng Đăng Nhập & Xác Thực

```mermaid
sequenceDiagram
    actor User as 👤 Người dùng
    participant Browser as 🌐 Trình duyệt
    participant Middleware as 🛡️ Middleware (Edge)
    participant API as ⚙️ API /auth/dang-nhap
    participant DB as 🗄️ PostgreSQL

    User->>Browser: Nhập tên đăng nhập + mật khẩu
    Browser->>API: POST /api/auth/dang-nhap
    API->>API: Rate Limit check (5 req/60s)
    alt Quá giới hạn
        API-->>Browser: 429 Too Many Requests
    end
    API->>DB: Tìm TaiKhoan theo tenDangNhap
    alt Không tìm thấy
        API-->>Browser: 401 Sai thông tin
    end
    API->>API: bcrypt.compare(password, hash)
    alt Sai mật khẩu
        API-->>Browser: 401 Sai thông tin
    end
    API->>API: Kiểm tra trangThai (bị khóa?)
    API->>API: signToken(JWT) — HS256, expire 1 ngày
    API->>DB: Tạo PhienDangNhap (IP, thiết bị, sessionId)
    API-->>Browser: Set Cookie (httpOnly, secure, sameSite)
    Browser-->>User: ✅ Chuyển đến /trang-chu

    Note over Browser,Middleware: Các request tiếp theo...
    Browser->>Middleware: Request + Cookie
    Middleware->>Middleware: jwtVerify(token) — Edge Runtime
    alt Token hết hạn / không hợp lệ
        Middleware-->>Browser: 302 Redirect → /dang-nhap
    end
    Middleware->>Middleware: Gắn Security Headers (6 loại)
    Middleware-->>Browser: Response + Headers
```

### 7.2. Luồng Xử Lý API Request

```mermaid
sequenceDiagram
    actor Client as 🌐 Client (apiFetch)
    participant Auth as 🛡️ withAuth()
    participant Handler as ⚙️ API Handler
    participant Sanitize as 🧹 sanitizeBody()
    participant DB as 🗄️ PostgreSQL
    participant Audit as 📝 Audit Log

    Client->>Auth: Request + Cookie
    Auth->>Auth: getCurrentUser() — verify JWT
    alt 401 Chưa đăng nhập
        Auth-->>Client: { error } 401
    end
    Auth->>Auth: Kiểm tra allowedRoles (RBAC)
    alt 403 Không có quyền
        Auth-->>Client: { error } 403
    end
    Auth->>Handler: Truyền { user, params }

    alt GET Request
        Handler->>Handler: filter cuaHangId (Store Isolation)
        Handler->>DB: prisma.model.findMany({ where })
        DB-->>Handler: data[]
        Handler-->>Client: { data, pagination }
    end

    alt POST/PUT Request
        Handler->>Sanitize: sanitizeBody(req.json())
        Sanitize->>Sanitize: Loại bỏ __proto__, id, createdAt
        Sanitize-->>Handler: Clean body
        Handler->>Handler: Gắn cuaHangId vào data
        Handler->>DB: prisma.model.create/update()
        DB-->>Handler: record
        Handler->>Audit: ghiAuditLog (fire-and-forget)
        Note right of Audit: Không block response
        Handler-->>Client: { data: record }
    end

    alt DELETE Request
        Handler->>DB: Verify cuaHangId match
        alt Cross-store access
            Handler-->>Client: { error } 403
        end
        Handler->>DB: prisma.model.delete()
        Handler->>Audit: ghiAuditLog('XOA')
        Handler-->>Client: { success: true }
    end

    alt Lỗi xảy ra
        Auth->>Auth: Catch Prisma P2002/P2025
        Auth-->>Client: { error } — không leak stack trace
    end
```

### 7.3. Luồng Đơn Hàng POS

```mermaid
sequenceDiagram
    actor NV as 👤 Nhân viên
    participant POS as 📱 Giao diện POS
    participant API as ⚙️ API Server
    participant DB as 🗄️ PostgreSQL
    participant Kho as 📦 Quản lý Kho

    NV->>POS: Chọn bàn / Mang về
    NV->>POS: Thêm sản phẩm + size + topping
    POS->>POS: Tính tổng tiền (real-time)

    NV->>POS: Xác nhận đơn hàng
    POS->>API: POST /api/don-hang
    API->>API: withAuth() + Store Isolation
    API->>API: generateCode('DH') → mã đơn unique
    API->>DB: Tạo DonHang + ChiTietDonHang + Topping
    API->>DB: Cập nhật trạng thái Bàn → DANG_SU_DUNG

    DB-->>API: Đơn hàng đã tạo
    API-->>POS: ✅ { data: donHang }

    Note over NV,Kho: Xử lý đơn hàng...

    NV->>POS: Thanh toán
    POS->>API: PUT /api/don-hang/[id] (trangThai: DA_THANH_TOAN)
    API->>DB: Lấy oldData (cho audit)
    API->>DB: Update trạng thái đơn
    API->>DB: Cập nhật Bàn → TRONG
    API->>Kho: Trừ nguyên vật liệu theo công thức
    Kho->>DB: Update TonKho
    API->>API: ghiAuditLog('SUA', diff trước/sau)
    API-->>POS: ✅ Đơn hàng đã thanh toán
```

### 7.4. Kiến Trúc Tổng Quan

```mermaid
flowchart TB
    subgraph Client["🌐 Client (Browser)"]
        React["React 19 + Shadcn UI"]
        TQ["TanStack Query (Cache)"]
        AF["apiFetch (401/403 handler)"]
    end

    subgraph Edge["🛡️ Edge Runtime"]
        MW["Middleware (proxy.ts)"]
        SH["Security Headers"]
        JV["JWT Verify"]
    end

    subgraph Server["⚙️ Next.js Server"]
        WA["withAuth() — RBAC"]
        SB["sanitizeBody()"]
        RL["Rate Limiter"]
        API["API Route Handlers"]
        AL["Audit Log"]
    end

    subgraph Data["🗄️ Data Layer"]
        Prisma["Prisma ORM (Connection Pool)"]
        PG["PostgreSQL"]
        CL["Cloudinary CDN"]
    end

    React --> TQ --> AF
    AF --> MW
    MW --> SH --> JV
    JV --> WA
    WA --> RL --> SB --> API
    API --> AL
    API --> Prisma --> PG
    API --> CL

    style Client fill:#1e293b,color:#e2e8f0
    style Edge fill:#7c3aed,color:#e2e8f0
    style Server fill:#0369a1,color:#e2e8f0
    style Data fill:#065f46,color:#e2e8f0
```

---

> **Kết luận:** Hệ thống đã triển khai đầy đủ các biện pháp bảo mật tiêu chuẩn công nghiệp (JWT, bcrypt, AES-256-GCM, RBAC, Security Headers, Audit Log). Kiến trúc multi-store tenant + serverless-ready cho phép mở rộng linh hoạt. Tech stack sử dụng các framework/library mới nhất và được cộng đồng hỗ trợ rộng rãi.
