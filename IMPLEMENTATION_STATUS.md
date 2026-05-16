# Implementation Status — AutoWash Pro Backend

**Last update:** 2026-05-16 (Phase 3 done)
**Default branch:** `main` on https://github.com/HuuFuoc/wash-auto
**Spec reference:** `docs/AutoWashPro_MongoDB_Collections_RutGon.md` (21 collections)

> Dùng tài liệu này khi thiết kế DB diagram để biết collection nào đã build, deviation nào so với spec, và phase nào sắp tới.

---

## 1. Trạng thái 21 collections theo spec

| # | Collection | Status | PR / branch | Notes |
|---:|---|:---:|---|---|
| 1 | `roles` | ✅ Done | merged `feature/auth` | Seed 5 code (customer/cashier/washer/manager/admin) khi app khởi động |
| 2 | `users` | ✅ Done | merged `feature/auth` + `feature/admin-user-management` | Schema đầy đủ §3.2 |
| 3 | `vehicle_types` | ✅ Done | `feature/vehicle-types` (pushed, chờ merge) | Admin CRUD; public read |
| 4 | `vehicles` | ✅ Done | `feature/vehicles` (pushed, chờ merge) | Xem deviation §2 dưới |
| 5 | `service_types` | ✅ Done | merged `feature/service-types` | Admin/manager CRUD; public read |
| 6 | `staff_shifts` | ❌ Phase 5 | — | Manager tạo ca + capacity; sẽ làm với atomic `$inc` |
| 7 | `bookings` | ❌ Phase 6 | — | Customer đặt lịch — flow trung tâm |
| 8 | `wash_sessions` | ❌ Phase sau | — | Phase 7+ (sau khi booking xong) |
| 9 | `inspections` | ❌ Phase sau | — | Phase 7+ |
| 10 | `inspection_photos` | ❌ Phase sau | — | Phase 7+ |
| 11 | `payments` | ❌ Phase sau | — | Phase 8+ |
| 12 | `cash_sessions` | ❌ Phase sau | — | Phase 8+ |
| 13 | `loyalty_accounts` | 🚧 Phase 4 (đang làm) | `feature/loyalty-accounts` | Auto-create khi customer register, default Member tier |
| 14 | `tier_configs` | ✅ Done | `feature/tier-configs` (pushed, chờ merge) | 4 tier auto-seed: Member(prio=0,win=3,pts=10) / Silver(prio=1,win=7,pts=15) / Gold(prio=2,win=14,pts=20) / Platinum(prio=3,win=30,pts=30) |
| 15 | `tier_histories` | ❌ Phase sau | — | Phase 9+ (loyalty math) |
| 16 | `point_transactions` | ❌ Phase sau | — | Phase 9+ |
| 17 | `point_ledgers` | ❌ Phase sau | — | Phase 9+ FIFO redemption |
| 18 | `promotions` | ❌ Phase sau | — | Phase 10+ |
| 19 | `promotion_services` | ❌ Phase sau | — | Phase 10+ |
| 20 | `promotion_valid_days` | ❌ Phase sau | — | Phase 10+ |
| 21 | `promotion_usages` | ❌ Phase sau | — | Phase 10+ |

**Tóm tắt:** 6/21 done, Phase 4 đang triển khai.

---

## 2. Deviations so với spec (cần xem khi vẽ DB diagram)

Spec **không sửa**. Đây là các điểm code chệch một chút, ghi nhận để bạn biết khi vẽ ERD.

### D-01 — `vehicles.model` → DB field tên `car_model`

| Field theo spec §3.4 | Field thực tế ở DB | API surface (DTO) | Lý do |
|---|---|---|---|
| `model` | **`car_model`** | `model` (giữ nguyên cho FE) | Mongoose `Model<T>` có method nội bộ `.model()`, đặt schema property tên `model` gây xung đột TS inference trên `.create()`. Đổi tên ở DB để build sạch, vẫn map về `model` ở DTO response. |

→ Khi vẽ ERD: ghi `car_model` ở column DB; nếu muốn đồng nhất với spec, có thể đặt tên `model` trong diagram nhưng phải nhớ đổi tên cột trong implementation thật. **Nên hỏi tôi trước nếu muốn rename đồng bộ về `model`.**

### D-02 — Snake_case ở mọi field (TS class + DB)

Khác với JS idiomatic. Toàn bộ entity dùng `snake_case` để đồng nhất với spec docs. DTO request/response vẫn camelCase nên FE không thấy điều này.

### D-03 — Timestamps tên `created_at` / `updated_at`

Mặc định Mongoose là `createdAt` / `updatedAt`. Mình đã đổi qua option `timestamps: { createdAt: 'created_at', updatedAt: 'updated_at' }`.

---

## 3. Endpoints đã có (16 endpoints)

### Auth (5)
| Method | Path | Access |
|---|---|---|
| POST | `/auth/register` | Public — tự register customer |
| POST | `/auth/login` | Public |
| POST | `/auth/refresh` | Public (body: refreshToken) |
| POST | `/auth/logout` | Public (body: refreshToken) |
| GET | `/auth/me` | Bearer JWT |

### Service Types (5)
| Method | Path | Access |
|---|---|---|
| GET | `/service-types` | Public (active only) |
| GET | `/service-types/:id` | Public |
| GET | `/admin/service-types` | Admin / Manager |
| POST | `/admin/service-types` | Admin / Manager |
| PATCH | `/admin/service-types/:id` | Admin / Manager |
| PATCH | `/admin/service-types/:id/status` | Admin / Manager |

### Vehicle Types (5)
| Method | Path | Access |
|---|---|---|
| GET | `/vehicle-types` | Public (active only) |
| GET | `/vehicle-types/:id` | Public |
| GET | `/admin/vehicle-types` | Admin / Manager |
| POST | `/admin/vehicle-types` | Admin / Manager |
| PATCH | `/admin/vehicle-types/:id` | Admin / Manager |
| PATCH | `/admin/vehicle-types/:id/status` | Admin / Manager |

### Vehicles (8)
| Method | Path | Access |
|---|---|---|
| GET | `/me/vehicles` | Customer (own) |
| GET | `/me/vehicles/:id` | Customer (ownership-protected, 404 nếu khác chủ) |
| POST | `/me/vehicles` | Customer |
| PATCH | `/me/vehicles/:id` | Customer (own) |
| PATCH | `/me/vehicles/:id/set-default` | Customer |
| DELETE | `/me/vehicles/:id` | Customer (soft delete) |
| GET | `/admin/vehicles` | Admin / Manager (paginated + filter) |
| GET | `/admin/vehicles/:id` | Admin / Manager |

### Tier Configs (5)
| Method | Path | Access |
|---|---|---|
| GET | `/tier-configs` | Public (active only, sorted by priority_level) |
| GET | `/tier-configs/:id` | Public |
| GET | `/admin/tier-configs` | Admin (incl. inactive) |
| PATCH | `/admin/tier-configs/:id` | Admin (tune knobs, 409 on priority_level conflict) |
| PATCH | `/admin/tier-configs/:id/status` | Admin (soft toggle) |

### Admin Users (8)
| Method | Path | Access |
|---|---|---|
| GET | `/admin/users` | Admin |
| GET | `/admin/users/:id` | Admin |
| POST | `/admin/users` | Admin |
| PATCH | `/admin/users/:id` | Admin |
| PATCH | `/admin/users/:id/role` | Admin |
| PATCH | `/admin/users/:id/status` | Admin (self-protection) |
| POST | `/admin/users/:id/reset-password` | Admin |
| DELETE | `/admin/users/:id` | Admin (self-protection) |

### Demo (1)
| GET | `/auth/admin-only` | Admin only — RolesGuard sanity check |

---

## 4. Roadmap còn lại (Phase 4 → 6 cho main booking flow)

Theo `C:\Users\HP\.claude\plans\b-y-gi-th-radiant-hearth.md`:

| Phase | Branch | Trạng thái | Mục tiêu |
|:---:|---|:---:|---|
| 3 | `feature/tier-configs` | ✅ | Catalog 4 tier loyalty, admin CRUD + seed |
| 4 | `feature/loyalty-accounts` | 🚧 | Tự tạo loyalty_account khi customer register; `GET /me/loyalty` |
| 5 | `feature/staff-shifts` | ⏳ | Manager tạo ca, atomic `$inc` cho capacity check |
| 6 | `feature/bookings` | ⏳ | Main flow: vehicle + service + shift → booking, kiểm tra tier window + capacity |

Sau Phase 6, booking flow MVP chạy được end-to-end. Các phase 7-10 (wash_sessions, inspections, payments, loyalty math, promotions) là backlog không chặn việc demo flow chính.

---

## 5. Test accounts đang dùng

Staff (đã insert manual trong Mongo Atlas — không có endpoint public để tạo, dùng `POST /admin/users` nếu cần thêm):

| Role | Email | Password |
|---|---|---|
| admin | `admin@washauto.local` | `Admin@123` |
| manager | `manager@washauto.local` | `Manager@123` |
| cashier | `cashier@washauto.local` | `Cashier@123` |
| washer | `washer@washauto.local` | `Washer@123` |

Customer: tự đăng ký qua `POST /auth/register`.

---

## 6. Notes vận hành

- **JWT secrets**: trong `.env` đang placeholder. Trước deploy thay bằng `node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"`.
- **Default field**: vehicle đầu tiên của customer auto-default. Set-default endpoint unset xe khác cùng owner trong 1 `updateMany`.
- **Soft delete**: vehicles và users dùng `is_active=false` (+ `delete_requested_at` cho users).
- **Ownership leak prevention**: `/me/vehicles/:id` trả 404 (không phải 403) khi truy cập xe người khác — không leak thông tin tồn tại.
- **Roles auto-seed**: 5 role tự upsert mỗi lần app khởi động qua `AuthSeederService.seedRoles()` (idempotent).
