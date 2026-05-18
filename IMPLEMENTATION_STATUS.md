# Implementation Status — AutoWash Pro Backend

**Last update:** 2026-05-18 (audit luồng nghiệp vụ E2E, bổ sung wash_sessions/inspections/orders)
**Default branch:** `main` on https://github.com/HuuFuoc/wash-auto
**Current branch:** `feature/wash-sessions`
**Spec reference:** `docs/AutoWashPro_MongoDB_Collections_RutGon.md` (21 collections)

> 📋 **Audit nghiệp vụ E2E (lean MVP):** xem [BUSINESS_FLOW_AUDIT.md](./BUSINESS_FLOW_AUDIT.md). Gap duy nhất P1: liên kết Booking↔Order PayOS + endpoint cashier xác nhận thanh toán. Fix = 5 field mới + 1 endpoint, ước ~1 ngày.

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
| 6 | `staff_shifts` | ✅ Done | `feature/staff-shifts` (pushed, chờ merge) | Manager CRUD + atomic `$inc` cho capacity, state machine scheduled→active→completed |
| 7 | `bookings` | ✅ Done | merged `feature/bookings` | Main flow — atomic capacity, tier window, state machine, cashier search phone/plate, customer `note` (D-04), service-duration fit (D-05). ⚠ Audit lean: thiếu 4 field payment (`payment_status`, `payment_method`, `paid_at`, `final_price`) |
| 8 | `wash_sessions` | ✅ Done | merged `feature/wash-sessions` (commit `31e67a2`) | Cashier mở phiên / walk-in; assign washer; washer start/complete; auto-sync booking |
| 9 | `inspections` | ✅ Done | merged `feature/wash-sessions` | Before/after phase, unique per session, customer acknowledgement + signature URL |
| 10 | `inspection_photos` | ✅ Done | merged `feature/wash-sessions` | URL-based (Cloudinary/S3), max 10MB |
| 11 | `orders` (PayOS) | ⚠ Built nhưng rời | branch local | PayOS checkout + webhook. ⚠ Audit P1: thiếu `booking_id` link |
| 12 | `payment_transactions` | ✅ Built | branch local | Audit log PayOS webhook |
| 13 | `cash_sessions` | ❌ Phase sau | — | Phase 8+ (quản lý ca thu ngân) — optional cho MVP |
| 13 | `loyalty_accounts` | ✅ Done | `feature/loyalty-accounts` (pushed, chờ merge) | Auto-create khi customer register (Member tier), lazy-create cho user cũ |
| 14 | `tier_configs` | ✅ Done | `feature/tier-configs` (pushed, chờ merge) | 4 tier auto-seed: Member(prio=0,win=3,pts=10) / Silver(prio=1,win=7,pts=15) / Gold(prio=2,win=14,pts=20) / Platinum(prio=3,win=30,pts=30) |
| 15 | `tier_histories` | ❌ Phase sau | — | Phase 9+ (loyalty math) |
| 16 | `point_transactions` | ❌ Phase sau | — | Phase 9+ |
| 17 | `point_ledgers` | ❌ Phase sau | — | Phase 9+ FIFO redemption |
| 18 | `promotions` | ❌ Phase sau | — | Phase 10+ |
| 19 | `promotion_services` | ❌ Phase sau | — | Phase 10+ |
| 20 | `promotion_valid_days` | ❌ Phase sau | — | Phase 10+ |
| 21 | `promotion_usages` | ❌ Phase sau | — | Phase 10+ |

**Tóm tắt:** 9/21 done. Booking flow MVP HOÀN TẤT — customer register → book → cashier manage end-to-end. Còn lại là feature post-booking (wash sessions, payments, loyalty math, promotions).

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

### D-04 — `bookings.note` (thêm ngoài spec §3.7)

Spec §3.7 không có field `note`. Mình thêm để customer ghi yêu cầu cho staff (vd "hút bụi kỹ", "đỗ ở bãi trước"). Trường optional, `MaxLength(500)`, stored as `bookings.note`. Cashier xem qua `GET /admin/bookings/:id`. Khi vẽ ERD có thể thêm cột hoặc bỏ qua tùy quy ước.

### D-05 — Booking validation: `scheduledAt + service.estimated_minutes ≤ shift.end_at`

Spec §5.2 không quy định check này. Mình thêm để booking không tràn quá shift end. Nếu service mất 60 phút mà khách book lúc còn 10 phút thì 400 BadRequest. Đây là rule nghiệp vụ hợp lý, không phá vỡ schema spec.

### D-06 — `tier_configs` schema align với FE (`points_per_1000_vnd` + `discount_percent`)

Spec §3.14 có field `points_per_wash` (flat points per wash, tính qua `tier.points_per_wash × service.points_multiplier`). FE landing page (`wave-wash.vercel.app`) mô tả tier khác hoàn toàn:

| Tier | FE: đặt trước | FE: điểm/1000đ | FE: discount |
|---|---|---|---|
| Member | 7 ngày | 1 | 0% |
| Silver | 10 ngày | 1.5 | 5% |
| Gold | 12 ngày | 2 | 10% |
| Platinum | 14 ngày | 3 | 15% |

Đã align BE theo FE:
- Bỏ field `tier_configs.points_per_wash`, thay bằng **`points_per_1000_vnd`** (number, can fractional)
- Thêm field **`discount_percent`** (0–100)
- Cập nhật `booking_window_days` seed: 7 / 10 / 12 / 14

Logic dùng các field này sẽ implement ở Phase 8 (payment với discount) và Phase 9 (loyalty math: `points = floor(amount / 1000) × tier.points_per_1000_vnd × service.points_multiplier` — cần xác nhận lại công thức khi làm).

**MIGRATION NOTE**: tier_configs đã seed trước đây (3/7/14/30) sẽ KHÔNG tự update qua `upsertByName` vì dùng `$setOnInsert`. Phải làm tay:
1. Mở Mongo Atlas → `wdp301.tier_configs` → **drop collection** (hoặc xóa từng doc)
2. Cũng drop `loyalty_accounts` vì tier_config_id references sẽ orphan (lazy-recreate trên `/me/loyalty`)
3. Restart app → reseed 4 tier với schema mới

Hoặc dùng admin `PATCH /admin/tier-configs/:id` để update từng tier qua API (giữ _id, không break loyalty_accounts references).

---

## 3. Endpoints đã có (32 endpoints)

> **Booking flow MVP hoàn tất**: customer register → add vehicle → pick service → pick shift → book → cashier confirm → state machine.

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

### Loyalty Accounts (1)
| Method | Path | Access |
|---|---|---|
| GET | `/me/loyalty` | JWT (lazy-creates if missing) |

### Bookings (8) — main flow
| Method | Path | Access | Mô tả |
|---|---|---|---|
| POST | `/me/bookings` | Customer | Tạo booking — validate ownership/active service/shift window/tier window/capacity atomic/service-duration fit |
| GET | `/me/bookings` | Customer | List own bookings |
| GET | `/me/bookings/:id` | Customer | Get own (404 if not owner) |
| PATCH | `/me/bookings/:id/reschedule` | Customer | Đổi shift + giờ (cap MAX_RESCHEDULES=2) |
| PATCH | `/me/bookings/:id/cancel` | Customer | Cancel (chỉ pending/confirmed) |
| GET | `/admin/bookings` | Cashier / Manager / Admin | Paginated + filter status/customerId/scheduled-range + search `customerPhone` + search `vehicleLicensePlate` (per docs §5.3) |
| GET | `/admin/bookings/:id` | Cashier / Manager / Admin | Get any |
| PATCH | `/admin/bookings/:id/status` | Cashier / Manager / Admin | State machine: pending→confirmed→in_progress→done; pending\|confirmed→cancelled/no_show; tự release shift slot khi vào terminal |

### Staff Shifts (6)
| Method | Path | Access |
|---|---|---|
| GET | `/shifts/available` | JWT (any role) — filter `from`, `to`, optional `shiftType` |
| GET | `/admin/shifts` | Manager / Admin — paginated + filter |
| GET | `/admin/shifts/:id` | Manager / Admin |
| POST | `/admin/shifts` | Manager / Admin — validates staff role matches shiftType |
| PATCH | `/admin/shifts/:id` | Manager / Admin |
| PATCH | `/admin/shifts/:id/status` | Manager / Admin — state machine |

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
| 3 | `feature/tier-configs` | ✅ merged | Catalog 4 tier loyalty, admin CRUD + seed |
| 4 | `feature/loyalty-accounts` | ✅ pushed | Tự tạo loyalty_account khi customer register; `GET /me/loyalty` |
| 5 | `feature/staff-shifts` | ✅ pushed | Manager tạo ca, atomic `$inc` cho capacity check |
| 6 | `feature/bookings` | 🚧 next | Main flow: vehicle + service + shift → booking, kiểm tra tier window + capacity |

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
