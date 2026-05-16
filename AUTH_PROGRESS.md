# Auth Feature — Progress Tracking

**Branch:** `feature/auth` (pushed to `origin` = `HuuFuoc/wash-auto`)
**Last update:** 2026-05-15

---

## 1. Tech stack đang dùng

| Layer | Lib | Mục đích |
|---|---|---|
| ODM | `@nestjs/mongoose` + `mongoose` | Schema cho `users`, `roles` |
| Auth strategy | `@nestjs/passport` + `passport-jwt` | Verify access token |
| JWT signing | `@nestjs/jwt` | Issue access (và sắp tới: refresh) token |
| Hashing | `bcrypt` | Hash password (salt rounds đọc từ `BCRYPT_SALT_ROUNDS`) |
| Validation | `class-validator` + `class-transformer` | DTO boundary check |
| Docs | `@nestjs/swagger` | Swagger UI ở `/docs` |
| Cache (sắp tới) | `ioredis` | Refresh-token blacklist |

Connection: Mongo Atlas (`mongodb+srv://...`), Redis Upstash (`rediss://...`) — config từ `.env` qua typed `registerAs`.

---

## 2. Cấu trúc thư mục

```text
src/
├── config/
│   ├── app.config.ts          # port, nodeEnv
│   ├── database.config.ts     # Mongo URI
│   └── auth.config.ts         # JWT secrets/TTL, bcrypt rounds
├── core/
│   └── database/
│       └── database.module.ts # Mongoose forRootAsync
├── shared/
│   ├── decorators/
│   │   └── current-user.decorator.ts  # @CurrentUser()
│   ├── guards/
│   │   └── jwt-auth.guard.ts          # JwtAuthGuard
│   └── types/
│       └── auth-payload.type.ts       # IAuthPayload
└── features/
    └── auth/
        ├── auth.module.ts          # OnModuleInit -> seed 5 roles
        ├── auth.controller.ts
        ├── auth.service.ts
        ├── entities/
        │   ├── role.entity.ts      # `roles` collection
        │   └── user.entity.ts      # `users` collection (snake_case DB fields)
        ├── repositories/
        │   ├── role.repository.ts
        │   └── user.repository.ts
        ├── dto/
        │   ├── register.dto.ts
        │   ├── login.dto.ts
        │   ├── auth-response.dto.ts
        │   └── user-response.dto.ts
        ├── strategies/
        │   └── jwt.strategy.ts
        └── types/
            └── role.enum.ts        # constant string codes
```

---

## 3. Quyết định kiến trúc đã chốt

| # | Quyết định | Lý do |
|---|---|---|
| D-01 | Mongoose (không phải MongoDB raw driver) | Schema validation, hooks, typing tốt. Teammate dùng raw driver trên `origin/main` — sẽ giải quyết ở PR merge. |
| D-02 | `core/database/`, không phải `database/` flat | Theo `docs/2.BE-PROJECT-RULES-neutral.md` §1 |
| D-03 | `roles` là collection riêng (không phải enum trên User) | Theo `docs/AutoWashPro_MongoDB_Collections_RutGon.md` §3.1 + §3.2 |
| D-04 | TS class property name = DB field name = snake_case | `@Prop({ name: ... })` không phải option Mongoose hợp lệ. Đành để TS class snake_case ở entity layer; DTO/service giữ camelCase. |
| D-05 | Timestamps đặt tên `created_at`/`updated_at` | Đồng bộ snake_case |
| D-06 | Public `/auth/register` luôn force `role = customer` | Không nhận `role`/`role_id` từ body |
| D-07 | 5 role auto-seed qua `AuthModule.onModuleInit` | Idempotent với `upsertByCode` |
| D-08 | `strictPropertyInitialization: false` ở tsconfig | NestJS pattern — properties được khung gán runtime |

---

## 4. Đã hoàn thành

### Step 1 — Foundation
- Cài deps: mongoose, JWT, Passport, bcrypt, swagger, class-validator
- Typed config (`registerAs`) cho app/database/auth
- `DatabaseModule` (Mongoose `forRootAsync`)
- Global `ValidationPipe` + Swagger UI ở `/docs`

### Step 2 — Register
- `User` schema (đã refactor ở dưới)
- `UserRepository`: `findByEmail`, `existsByEmail`, `existsByPhone`, `createUser`
- `POST /auth/register`: hash bcrypt, force `role = customer`

### Step 3 — Login + JWT access
- `JwtStrategy` verify bearer token bằng `JWT_ACCESS_SECRET`
- `JwtAuthGuard` shared
- `@CurrentUser()` decorator extract `IAuthPayload` từ request
- `POST /auth/login`: verify password, sign access token với `{ sub, email, role }`
- `GET /auth/me` (protected) — sanity check guard

### Refactor — Schema theo docs
- Tạo `Role` entity + `RoleRepository`
- 5 role auto-seed (customer/cashier/washer/manager/admin)
- `User` đầy đủ field theo `docs/AutoWashPro_MongoDB_Collections_RutGon.md` §3.2: `role_id`, `name`, `phone` (required+unique), `email`, `password_hash`, `avatar_url`, `date_of_birth`, `is_active`, `delete_requested_at`, `created_at`, `updated_at`
- DB snake_case, DTO/service camelCase

---

### Step 4 — Refresh token rotation + Logout (with Redis blacklist)
- Cài `ioredis`, thêm `src/config/cache.config.ts`
- `CacheModule` (`@Global`) cung cấp Redis client qua `REDIS_CLIENT` symbol token; tự đóng connection ở `onModuleDestroy`
- `RefreshTokenService`: sign refresh JWT (jti = UUID), verify, check/set blacklist trên Redis với TTL = `exp - now`
- `POST /auth/login` giờ trả `{ accessToken, refreshToken, user }`
- `POST /auth/refresh`: verify refresh → check blacklist → blacklist jti cũ → issue cặp mới (rotation)
- `POST /auth/logout`: blacklist jti hiện tại, trả 204
- Redis key pattern: `refresh:bl:<jti>` → `1`

## 5. Sẽ làm tiếp (in-progress)

*(no active work — see backlog)*

---

## 6. Backlog (làm sau)

| # | Việc | Tại sao |
|---|---|---|
| F-01 | Admin endpoint tạo staff user (cashier/washer/manager) | Hiện chỉ có customer tự đăng ký |
| F-02 | `RolesGuard` + `@Roles(...)` decorator | Áp dụng phân quyền role-based cho các endpoint sau |
| F-03 | Endpoint reset password / forgot password | Phải có cho production |
| F-04 | Refresh token *reuse detection* (token family) | Hiện chỉ blacklist JTI — nâng cấp để invalidate cả family nếu token cũ bị replay |
| F-05 | Rate limit `/auth/login` + `/auth/register` (`@nestjs/throttler`) | Chống brute force |
| F-06 | Update `users.delete_requested_at` flow + cron xóa user | Theo spec docs §3.2 |
| F-07 | E-mail/phone verification | Bảo mật account |
| F-08 | Audit log cho login/logout/register | Theo `audit-log` feature ở docs §1 |
| F-09 | Unit + integration tests | Coverage targets ở docs §7 |
| F-10 | Hợp nhất kiến trúc với teammate trên `main` | Hiện divergent: chúng ta Mongoose, teammate raw driver |

---

## 7. API endpoints hiện có

| Method | Path | Auth | Mô tả |
|---|---|---|---|
| POST | `/auth/register` | Public | Đăng ký tài khoản customer |
| POST | `/auth/login` | Public | Login, trả `{ accessToken, refreshToken, user }` |
| POST | `/auth/refresh` | Public (body: refreshToken) | Rotate refresh token, trả cặp mới |
| POST | `/auth/logout` | Public (body: refreshToken) | Blacklist refresh token (204) |
| GET | `/auth/me` | Bearer JWT | Trả payload của token (sanity) |

Swagger UI: `http://localhost:3000/docs`

---

## 8. Lưu ý vận hành

- **JWT secrets trong `.env` đang là placeholder**. Trước khi deploy phải replace bằng chuỗi random ≥ 32 chars:
  ```powershell
  node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
  ```
- **Drop collection `users` cũ** trong Atlas nếu schema thay đổi giữa các phiên (đã làm khi refactor).
- **Roles** tự seed mỗi lần app start — không cần làm gì thủ công.
- **Staff users** hiện phải insert manual vào Mongo (chưa có admin endpoint).
