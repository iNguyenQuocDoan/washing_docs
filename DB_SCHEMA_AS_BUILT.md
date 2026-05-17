# Database Schema — As Built

**Mục đích:** Bản schema thực tế của 9 collections đã code, để vẽ ERD/DB diagram bằng tay.

**Quy ước:**
- 🟢 = đúng spec docs §3.x
- 🟡 = deviation (xem section §Deviations dưới)
- `PK` = primary key (Mongo auto `_id` ObjectId)
- `FK→X` = foreign key reference to collection X
- `UQ` = unique index
- `IX` = single index
- `IX(a,b)` = compound index

**Convention chung cho mọi collection:**
- `_id`: ObjectId, PK, auto-generated
- `created_at`, `updated_at`: Date, auto Mongoose timestamps (đã rename từ `createdAt/updatedAt`)

---

## 1. `roles` 🟢

| Field | Type | Constraint | Note |
|---|---|---|---|
| `_id` | ObjectId | PK | |
| `code` | String | required, UQ, IX, enum | `customer`, `cashier`, `washer`, `manager`, `admin` |
| `name` | String | required | Display name (Customer, Cashier, ...) |
| `description` | String | optional | |
| `is_active` | Boolean | default `true` | |
| `created_at` | Date | auto | |
| `updated_at` | Date | auto | |

**Seed:** Auto 5 docs khi app khởi động (`AuthModule.onModuleInit`).

---

## 2. `users` 🟢

| Field | Type | Constraint | Note |
|---|---|---|---|
| `_id` | ObjectId | PK | |
| `role_id` | ObjectId | required, IX, **FK→roles** | |
| `name` | String | required | Họ tên |
| `phone` | String | required, UQ, IX | Số điện thoại |
| `email` | String | required, UQ, IX, lowercase | Login |
| `password_hash` | String | required | bcrypt hash |
| `avatar_url` | String | optional | |
| `date_of_birth` | Date | optional | |
| `is_active` | Boolean | default `true` | |
| `delete_requested_at` | Date | optional | Soft-delete request marker |
| `created_at` | Date | auto | |
| `updated_at` | Date | auto | |

---

## 3. `vehicle_types` 🟢

| Field | Type | Constraint | Note |
|---|---|---|---|
| `_id` | ObjectId | PK | |
| `name` | String | required, UQ, IX | Motorbike, Car, SUV... |
| `description` | String | optional | |
| `is_active` | Boolean | default `true`, IX | |
| `created_at` | Date | auto | |
| `updated_at` | Date | auto | |

---

## 4. `vehicles` 🟡 (D-01 — `model` → `car_model`)

| Field | Type | Constraint | Note |
|---|---|---|---|
| `_id` | ObjectId | PK | |
| `customer_id` | ObjectId | required, IX, **FK→users** | |
| `vehicle_type_id` | ObjectId | required, IX, **FK→vehicle_types** | |
| `license_plate` | String | required, UQ, IX | |
| `nickname` | String | optional | |
| `brand` | String | optional | |
| **`car_model`** 🟡 | String | optional | **Renamed from `model`** (Mongoose Model<T> conflict) |
| `color` | String | optional | |
| `is_default` | Boolean | default `false` | At most 1 = true per customer (enforced in service) |
| `is_active` | Boolean | default `true` | |
| `created_at` | Date | auto | |
| `updated_at` | Date | auto | |

**Indexes:**
- `{ license_plate: 1 }` UQ
- `{ customer_id: 1, is_active: 1 }` compound

---

## 5. `service_types` 🟢

| Field | Type | Constraint | Note |
|---|---|---|---|
| `_id` | ObjectId | PK | |
| `name` | String | required, UQ, IX | |
| `description` | String | optional | |
| `base_price` | Decimal128 | required | VND price |
| `estimated_minutes` | Number | required, min 1 | Duration |
| `points_multiplier` | Number | required, min 0 | For loyalty math |
| `is_active` | Boolean | default `true`, IX | |
| `created_at` | Date | auto | |
| `updated_at` | Date | auto | |

---

## 6. `staff_shifts` 🟢

| Field | Type | Constraint | Note |
|---|---|---|---|
| `_id` | ObjectId | PK | |
| `staff_id` | ObjectId | required, IX, **FK→users** | role must match `shift_type` |
| `shift_type` | String | required, IX, enum | `cashier`, `washer` |
| `station_name` | String | optional | E.g. "Bay 1" |
| `start_at` | Date | required | |
| `end_at` | Date | required | start_at < end_at (service rule) |
| `status` | String | required, IX, enum, default `scheduled` | `scheduled`→`active`→`completed`; or `cancelled` |
| `max_bookings` | Number | required, min 1 | Capacity cap |
| `current_bookings` | Number | required, default 0, min 0 | Atomic `$inc` from booking flow |
| `note` | String | optional | |
| `created_at` | Date | auto | |
| `updated_at` | Date | auto | |

**Indexes:**
- `{ staff_id: 1, start_at: -1 }` compound
- `{ shift_type: 1, status: 1 }` compound

---

## 7. `bookings` 🟡 (D-04 — thêm `note`)

| Field | Type | Constraint | Note |
|---|---|---|---|
| `_id` | ObjectId | PK | |
| `customer_id` | ObjectId | required, IX, **FK→users** | |
| `vehicle_id` | ObjectId | required, IX, **FK→vehicles** | |
| `service_type_id` | ObjectId | required, IX, **FK→service_types** | |
| `staff_shift_id` | ObjectId | required, IX, **FK→staff_shifts** | |
| `scheduled_at` | Date | required | Within shift window; + estimated_minutes ≤ end_at (D-05) |
| `status` | String | required, IX, enum, default `pending` | State machine: `pending`→`confirmed`→`in_progress`→`done`; or `cancelled`/`no_show` |
| `priority_level` | Number | required, default 0, min 0 | Copied from tier.priority_level at creation |
| `reschedule_count` | Number | required, default 0, min 0 | Capped by `MAX_RESCHEDULES` env (default 2) |
| `cancel_reason` | String | optional | |
| **`note`** 🟡 | String | optional, max 500 | **Added — not in spec §3.7** — customer note for staff |
| `created_at` | Date | auto | |
| `updated_at` | Date | auto | |

**Indexes:**
- `{ customer_id: 1, scheduled_at: -1 }` compound
- `{ scheduled_at: 1, status: 1 }` compound

---

## 8. `loyalty_accounts` 🟢

| Field | Type | Constraint | Note |
|---|---|---|---|
| `_id` | ObjectId | PK | |
| `customer_id` | ObjectId | required, UQ, IX, **FK→users** | One account per customer |
| `tier_config_id` | ObjectId | required, IX, **FK→tier_configs** | Defaults to Member at creation |
| `points_balance` | Number | required, default 0, min 0 | |
| `visits_this_month` | Number | required, default 0, min 0 | |
| `visits_last_month` | Number | required, default 0, min 0 | |
| `consecutive_low_months` | Number | required, default 0, min 0 | For tier downgrade |
| `tier_reviewed_at` | Date | optional | Last cron review |
| `points_expire_at` | Date | optional | Earliest expiring batch |
| `created_at` | Date | auto | |
| `updated_at` | Date | auto | |

**Lifecycle:**
- Auto-create at `customer` register (`AuthService.register` → `LoyaltyService.ensureForCustomer`)
- Lazy-create on first `GET /me/loyalty` (cho user cũ trước feature ship)

---

## 9. `tier_configs` 🟡 (D-06 — thay points field + thêm discount)

| Field | Type | Constraint | Note |
|---|---|---|---|
| `_id` | ObjectId | PK | |
| `tier_name` | String | required, UQ, IX, enum | `Member`, `Silver`, `Gold`, `Platinum` |
| `min_visits_per_month` | Number | required, min 0 | For tier upgrade math |
| `booking_window_days` | Number | required, min 0 | Customer can book up to N days ahead |
| `priority_level` | Number | required, UQ, min 0 | Booking queue priority |
| ~~`points_per_wash`~~ 🟡 | — | **REMOVED** | Was in spec §3.14; replaced |
| **`points_per_1000_vnd`** 🟡 | Number | required, min 0 | **NEW** — Points per 1,000 VND spent (FE: "1.5 điểm / 1.000đ") |
| **`discount_percent`** 🟡 | Number | required, default 0, min 0, max 100 | **NEW** — % off per wash (FE: "Giảm 5% mỗi lần rửa") |
| `is_active` | Boolean | default `true`, IX | |
| `created_at` | Date | auto | |
| `updated_at` | Date | auto | |

**Seed values (auto on app start, idempotent):**

| tier_name | min_visits | window_days | priority | points/1000đ | discount % |
|---|---:|---:|---:|---:|---:|
| Member | 0 | 7 | 0 | 1.0 | 0 |
| Silver | 2 | 10 | 1 | 1.5 | 5 |
| Gold | 5 | 12 | 2 | 2.0 | 10 |
| Platinum | 10 | 14 | 3 | 3.0 | 15 |

---

## Quan hệ chính (để vẽ diagram)

```
roles  1 ── N  users

users  1 ── N  vehicles  N ── 1  vehicle_types

users  1 ── 1  loyalty_accounts  N ── 1  tier_configs

users  1 ── N  staff_shifts        (staff_id, with role=cashier|washer)

bookings:
  customer_id      → users
  vehicle_id       → vehicles
  service_type_id  → service_types
  staff_shift_id   → staff_shifts
```

```
        ┌──────┐
        │roles │
        └──┬───┘
           │ 1:N
           ▼
       ┌───────┐ 1:N   ┌──────────┐
       │ users │──────▶│ vehicles │
       └───┬───┘       └────┬─────┘
           │ 1:1            │ N:1
           ▼                ▼
   ┌──────────────┐   ┌──────────────┐
   │loyalty_accts │   │vehicle_types │
   └──────┬───────┘   └──────────────┘
          │ N:1
          ▼
   ┌──────────────┐
   │ tier_configs │
   └──────────────┘

   ┌──────────────┐ 1:N    ┌───────────────┐
   │ staff_shifts │◀──────▶│   bookings    │
   └──────────────┘        └───────┬───────┘
                                   │
   ┌──────────────┐                │
   │service_types │◀───────────────┤
   └──────────────┘                │
                                   │
                          ┌────────▼────┐
                          │   users     │ (customer_id)
                          └─────────────┘
                          ┌─────────────┐
                          │  vehicles   │ (vehicle_id)
                          └─────────────┘
```

---

## Deviations summary (cho ERD chú thích)

| ID | Deviation | Lý do |
|---|---|---|
| D-01 | `vehicles.model` → `vehicles.car_model` | Conflict với Mongoose `Model.model()` |
| D-02 | TS class properties dùng `snake_case` thay vì camelCase | Đồng nhất với DB field |
| D-03 | Timestamps `created_at` / `updated_at` (thay vì `createdAt/updatedAt`) | Đồng nhất với snake_case spec |
| D-04 | `bookings.note` (thêm field optional) | Customer note for staff (UX) |
| D-05 | Validation `scheduledAt + service.estimated_minutes ≤ shift.end_at` | Tránh booking tràn quá shift |
| D-06 | `tier_configs`: bỏ `points_per_wash`, thêm `points_per_1000_vnd` + `discount_percent` | Match FE landing page |

---

## Collections còn lại theo spec (CHƯA build, sẽ vẽ riêng nếu cần)

10. `inspections`
11. `inspection_photos`
12. `wash_sessions`
13. `payments`
14. `cash_sessions`
15. `tier_histories`
16. `point_transactions`
17. `point_ledgers`
18. `promotions`
19. `promotion_services`
20. `promotion_valid_days`
21. `promotion_usages`

Khi vẽ ERD lần đầu, có thể tạm bỏ 12 collection này hoặc vẽ placeholder, focus vào 9 collection trên đã chạy được.
