# Booking Business Flow — End-to-End

**Phạm vi:** Mô tả đầy đủ luồng nghiệp vụ Booking — từ lúc Customer chọn gói đến khi Cashier đóng đơn — và map sang code thực tế.
**Branch tham chiếu:** `feature/wash-sessions` (`31e67a2`)
**Last update:** 2026-05-18

> Tài liệu này dùng để onboard người mới, làm spec test, và làm chuẩn so sánh khi build FE. Scope: **MVP shop nhỏ** — không over-engineer.

---

## 1. Actors

| Role | Code | Ghi chú |
|---|---|---|
| Customer | `customer` | End user, tự đăng ký qua `POST /auth/register` |
| Cashier | `cashier` | Thu ngân, mở phiên rửa, gán washer, xác nhận thanh toán |
| Washer | `washer` | Người rửa xe, có queue riêng |
| Manager | `manager` | Quản lý ca, có quyền cashier |
| Admin | `admin` | Toàn quyền |

Roles seed tự động khi app khởi động ([auth.module.ts:22-48](../src/features/auth/auth.module.ts#L22)).

---

## 2. Happy path — Booking đặt trước (có lịch hẹn)

### Bước 1 — Customer chọn gói + xe

| Hành động | Endpoint | Bằng chứng |
|---|---|---|
| Xem danh sách gói rửa | `GET /service-types` | [service-type.controller.ts:11-17](../src/features/service-type/service-type.controller.ts#L11) |
| Xem loại xe | `GET /vehicle-types` | [vehicle-type.controller.ts:11-20](../src/features/vehicle-type/vehicle-type.controller.ts#L11) |
| Quản lý xe của mình | `GET/POST/PATCH /me/vehicles` | [vehicle.controller.ts](../src/features/vehicle/vehicle.controller.ts) |

**Quy tắc:**
- `service_types.is_active=false` → không hiển thị public.
- Vehicle ownership-protected: customer chỉ thấy xe của chính mình.
- Xe mới đầu tiên auto-set `is_default=true`.

### Bước 2 — Customer chọn ca + tạo booking

```
POST /me/bookings
Role: JWT bất kỳ (default customer)
Body: { vehicleId, serviceTypeId, staffShiftId, scheduledAt, note? }
```

**Validate** ([booking.service.ts:54-158](../src/features/booking/booking.service.ts#L54)):
1. Vehicle thuộc customer + còn active.
2. Service active.
3. Shift còn `scheduled` (chưa active/completed/cancelled).
4. `scheduledAt` nằm trong `[shift.start_at, shift.end_at]`.
5. `scheduledAt + service.estimated_minutes ≤ shift.end_at` (không tràn cuối ca — D-05).
6. `scheduledAt` ≥ hiện tại - 60s.
7. `scheduledAt` ≤ `now + tier.booking_window_days` (BR-06).
8. Customer chưa có quá `MAX_ACTIVE_PER_CUSTOMER` booking active (BR-08, default 3).
9. Shift còn slot — atomic `$inc current_bookings` (BR-07).

**Tạo:** `Booking{ status: 'pending', priority_level: tier.priority_level }`.

**Rollback:** nếu create lỗi → release slot (`$inc -1`).

### Bước 3 — Cashier xem + xác nhận booking

```
GET /admin/bookings?status=pending&customerPhone=...&vehicleLicensePlate=...
Role: cashier, manager, admin
```

[admin-booking.controller.ts:33-43](../src/features/booking/admin-booking.controller.ts#L33). Search theo `customerPhone` (LIKE) hoặc `vehicleLicensePlate` (LIKE) — đáp ứng UX cashier tại counter ([booking.service.ts:298-324](../src/features/booking/booking.service.ts#L298)).

```
PATCH /admin/bookings/:id/status
Body: { status: 'confirmed' }
```

State machine cho phép `pending → confirmed` ([booking.state-machine.ts:3-18](../src/features/booking/booking.state-machine.ts#L3)).

### Bước 4 — Khách đến shop, Cashier mở phiên rửa

```
POST /admin/wash-sessions/from-booking
Body: { bookingId, note? }
Role: cashier, manager, admin
```

[admin-wash-session.controller.ts:40-55](../src/features/wash-session/admin-wash-session.controller.ts#L40), [wash-session.service.ts:48-95](../src/features/wash-session/wash-session.service.ts#L48).

**Quy tắc:**
- Booking phải ở `pending` hoặc `confirmed`.
- 1 booking ↔ ≤1 wash_session (check trong service, return 409 nếu trùng).
- Snapshot `service.base_price` vào `wash_session.service_price_snapshot` (phòng admin đổi giá sau).

**Kết quả:** `wash_session` được tạo với `status='waiting'`, `cashier_id=actor`, `booking_id` link tới booking.

> Bước này NGẦM hiểu là "cashier đã verify xe khớp booking + check-in". Không có endpoint check-in riêng — MVP không cần.

### Bước 5 — (Tùy chọn) Cashier ghi nhận tình trạng xe (Inspection BEFORE)

```
POST /admin/wash-sessions/:sessionId/inspections
Body: { phase: 'before', damageNotes?, customerSignatureUrl? }
```

[admin-inspection.controller.ts:35-57](../src/features/wash-session/admin-inspection.controller.ts#L35). Chỉ tạo được khi session ở `waiting` hoặc `assigned`.

```
POST /admin/inspections/:inspectionId/photos
Body: { photoUrl, mime, size }
```

[admin-inspection-photo.controller.ts:34-48](../src/features/wash-session/admin-inspection-photo.controller.ts#L34). FE upload lên Cloudinary/S3 trước, truyền URL về.

```
PATCH /admin/inspections/:id/acknowledge
Body: { customerSignatureUrl? }
```

Đánh dấu khách đã xác nhận tình trạng xe trước rửa.

### Bước 6 — Cashier gán Washer

```
PATCH /admin/wash-sessions/:id/assign-washer
Body: { washerId }
```

[admin-wash-session.controller.ts:97-114](../src/features/wash-session/admin-wash-session.controller.ts#L97), [wash-session.service.ts:134-162](../src/features/wash-session/wash-session.service.ts#L134).

**Validate:**
- Session ở `waiting`.
- `washerId` là user có role=`washer` + `is_active=true`.

**Kết quả:** session → `assigned`, lưu `washer_id`.

### Bước 7 — Washer bắt đầu rửa

```
GET /me/washer/sessions?status=assigned
PATCH /me/washer/sessions/:id/start
```

[washer-wash-session.controller.ts:34-67](../src/features/wash-session/washer-wash-session.controller.ts#L34).

**Validate:**
- Session phải đang assigned cho washer này (kiểm tra `session.washer_id === user.sub`, return 403 nếu khác).
- Session phải ở `assigned`.

**Kết quả:**
- `wash_session.status = 'in_progress'`, `started_at = now`.
- Auto-sync `booking.status = 'in_progress'` ([booking.service.ts:407-428](../src/features/booking/booking.service.ts#L407)).

### Bước 8 — Washer hoàn tất

```
PATCH /me/washer/sessions/:id/complete
```

**Validate:** session ở `in_progress`.

**Kết quả:**
- `wash_session.status = 'done'`, `completed_at = now`.
- Auto-sync `booking.status = 'done'` ([booking.service.ts:434-453](../src/features/booking/booking.service.ts#L434)).
- Auto-release shift slot.

### Bước 9 — (Tùy chọn) Inspection AFTER

```
POST /admin/wash-sessions/:sessionId/inspections
Body: { phase: 'after', damageNotes? }
```

Chỉ tạo được khi session ở `done`. Để khách xác nhận xe ra không hư thêm gì.

### Bước 10 — Cashier xác nhận thanh toán + đóng đơn ⚠ **CẦN BUILD**

```
PATCH /admin/bookings/:id/payment
Body:
  {
    method: 'cash' | 'online',
    finalPrice: number,            # VND, integer
    orderId?: string               # bắt buộc nếu method=online
  }
Role: cashier, manager, admin
```

**Validate:**
- Booking ở `status='done'`.
- `booking.payment_status='unpaid'`.
- Nếu `method='online'`: `Order.status='PAID'` và `Order.booking_id === bookingId`.

**Kết quả:**
- `booking.payment_status='paid'`
- `booking.payment_method=method`
- `booking.paid_at=now`
- `booking.final_price=finalPrice`

→ Đóng đơn = `status='done'` AND `payment_status='paid'`. KHÔNG cần state `CLOSED` riêng.

---

## 3. Happy path — Walk-in (không booking trước)

Khách đến shop trực tiếp, không đặt qua app.

### Bước 1 — Customer đã có account + đã thêm xe vào hệ thống

(Hoặc cashier tự tạo user qua `POST /admin/users` rồi thêm xe.)

### Bước 2 — Cashier mở wash session trực tiếp

```
POST /admin/wash-sessions/walk-in
Body: { customerId, vehicleId, serviceTypeId, note? }
Role: cashier+
```

[admin-wash-session.controller.ts:57-74](../src/features/wash-session/admin-wash-session.controller.ts#L57), [wash-session.service.ts:97-132](../src/features/wash-session/wash-session.service.ts#L97).

**Validate:** vehicle thuộc customer + active, service active.

**Kết quả:** `wash_session{ booking_id: null, status: 'waiting', ... }`.

### Bước 3+ — Giống flow đặt trước

Assign washer → washer start → washer complete → cashier confirm payment.

**Khác biệt khi đóng đơn:** walk-in không có booking → cần endpoint riêng hoặc tạo Booking ngầm. **MVP:** chấp nhận chỉ tạo `Order` (PayOS) hoặc ghi nhận tay; nếu cần báo cáo doanh thu thì cần backlog "auto-create booking từ walk-in session".

---

## 4. Alternative flows / Edge cases

### 4.1 Customer hủy booking

```
PATCH /me/bookings/:id/cancel
Body: { reason? }
```

Chỉ cho phép từ `pending` hoặc `confirmed` ([booking.state-machine.ts:32-37](../src/features/booking/booking.state-machine.ts#L32)). Tự release shift slot.

### 4.2 Customer đổi giờ booking

```
PATCH /me/bookings/:id/reschedule
Body: { staffShiftId, scheduledAt }
```

[booking.service.ts:170-258](../src/features/booking/booking.service.ts#L170). Cap `MAX_RESCHEDULES=2`. Atomic: reserve slot mới trước, rồi release slot cũ.

### 4.3 Cashier đánh dấu khách không đến

```
PATCH /admin/bookings/:id/status
Body: { status: 'no_show', reason? }
```

Cho phép từ `pending` hoặc `confirmed`. Tự release slot.

### 4.4 Cashier hủy wash session đang chạy

```
PATCH /admin/wash-sessions/:id/cancel
Body: { reason? }
```

[admin-wash-session.controller.ts:116-130](../src/features/wash-session/admin-wash-session.controller.ts#L116). Cho phép từ `waiting`/`assigned`/`in_progress`. **LƯU Ý:** không tự cancel booking — cashier phải làm tay qua `PATCH /admin/bookings/:id/status`.

### 4.5 Khách đến trễ, muốn đổi gói tại counter

**MVP:** Cashier hủy booking cũ + tạo wash_session walk-in mới với gói khác. Không cần endpoint `switch-service` riêng.

### 4.6 Khách đem xe khác đến

**MVP:** Tương tự — hủy + walk-in mới. Hoặc cashier sửa `booking.vehicle_id` qua Mongo Atlas khi cần (hiếm).

### 4.7 PayOS online payment

```
POST /me/orders
Body: { vehicleId, serviceTypeId, bookingId?, notes? }
```

[payment.controller.ts:36-53](../src/features/payment/payment.controller.ts#L36). Trả về `checkoutUrl` để FE redirect sang PayOS.

```
POST /payments/webhook   (PayOS gọi, không auth)
```

[payment.controller.ts:99-108](../src/features/payment/payment.controller.ts#L99). Verify signature, update `Order.status`. **CẦN BUILD:** nếu Order có `booking_id`, auto-set `booking.payment_status='paid'`.

---

## 5. State machines

### Booking ([booking.state-machine.ts](../src/features/booking/booking.state-machine.ts))
```
pending ──┬─→ confirmed ──┬─→ in_progress ──→ done (terminal)
          │               │
          ├─→ cancelled (terminal)
          └─→ no_show (terminal)
```

Transitions tự release shift slot khi đi từ active → terminal.

### WashSession ([wash-session.state-machine.ts](../src/features/wash-session/wash-session.state-machine.ts))
```
waiting ──→ assigned ──→ in_progress ──→ done (terminal)
   │           │              │
   └─→ cancelled (any of 3 above)
```

Có hỗ trợ un-assign: `assigned → waiting`.

### Order PayOS ([payment/entities/order.entity.ts:6-11](../src/features/payment/entities/order.entity.ts#L6))
```
PENDING ──┬─→ PAID
          ├─→ CANCELLED
          └─→ EXPIRED
```

### Payment terminal indicator (sau khi build fix MVP)
```
booking.status = 'done' AND booking.payment_status = 'paid'  →  ĐÓNG ĐƠN
```

---

## 6. Database entities

Chi tiết schema xem [DB_SCHEMA_AS_BUILT.md](./DB_SCHEMA_AS_BUILT.md).

```
       ┌────────┐
       │ users  │
       └───┬────┘
           │1:N
           ├────────────────┬─────────────────┐
           ▼                ▼                 ▼
       ┌──────┐        ┌──────────┐      ┌─────────────┐
       │vehic.│        │staff_shft│      │loyalty_acct │
       └───┬──┘        └────┬─────┘      └──────┬──────┘
           │N:1             │1:N                │N:1
           ▼                ▼                   ▼
       ┌──────────┐    ┌──────────┐       ┌────────────┐
       │veh_types │    │ booking  │◀──────│tier_configs│
       └──────────┘    └────┬─────┘       └────────────┘
                            │1:1
                            ▼
                     ┌──────────────┐ N:1  ┌────────────┐
                     │ wash_session │─────▶│ vehicle    │
                     └──┬───────┬───┘      └────────────┘
                        │1:N    │1:N
                        ▼       ▼
                ┌──────────┐ ┌────────┐ 1:N  ┌──────────────┐
                │service_ty│ │inspect │─────▶│inspect_photos│
                └──────────┘ └────────┘      └──────────────┘

orders (PayOS) — chạy song song:
  customer_id → users
  service_type_id → service_types
  vehicle_id → vehicles
  ⚠ THIẾU booking_id (gap MVP P1 — xem §7)
```

---

## 7. MVP gap — Phải build

**Vấn đề:** `Order` (PayOS) hiện không link `Booking` → cashier không có cách "đóng đơn" sau khi washer xong.

**Fix tối thiểu — ước 1 ngày:**

### 7.1 Schema changes

**`Booking` thêm 4 field** ([booking/entities/booking.entity.ts](../src/features/booking/entities/booking.entity.ts)):
```ts
@Prop({ type: String, enum: ['unpaid','paid'], default: 'unpaid', index: true })
payment_status: 'unpaid' | 'paid';

@Prop({ type: String, enum: ['cash','online'] })
payment_method?: 'cash' | 'online';

@Prop()
paid_at?: Date;

@Prop({ type: Types.Decimal128 })
final_price?: Types.Decimal128;
```

**`Order` thêm 1 field** ([payment/entities/order.entity.ts](../src/features/payment/entities/order.entity.ts)):
```ts
@Prop({ type: Types.ObjectId, ref: 'Booking', sparse: true, unique: true, index: true })
booking_id?: Types.ObjectId;
```

### 7.2 DTO changes

`CreateOrderDto` thêm optional `bookingId?: string`. Khi tạo, lưu vào `Order.booking_id` nếu có.

### 7.3 Endpoint mới

```
PATCH /admin/bookings/:id/payment
Role: cashier, manager, admin
Body:
  {
    method: 'cash' | 'online',
    finalPrice: number,
    orderId?: string             # required if method='online'
  }
Response: BookingResponseDto (includes payment_status, payment_method, paid_at, final_price)

Business rules:
  - booking.status must be 'done'
  - booking.payment_status must be 'unpaid'
  - if method='online': Order.status must be 'PAID' AND Order.booking_id === bookingId
  - SET: payment_status='paid', payment_method=method, paid_at=now, final_price=finalPrice
```

### 7.4 PayOS webhook auto-sync

Trong [payment.service.ts:handleWebhook](../src/features/payment/payment.service.ts#L172), khi `code === '00'` (PAID), nếu `order.booking_id` có → tự gọi internal method để set `booking.payment_status='paid'`, `payment_method='online'`, `paid_at=now`, `final_price=order.amount`.

### 7.5 Test plan

1. Tạo booking → confirm → mở wash_session → assign washer → start → complete.
2. Cashier gọi `/payment` với `method=cash` → booking đóng đơn.
3. Lặp lại nhưng `method=online`: customer thanh toán PayOS trước → webhook → cashier verify orderId.
4. Negative: gọi `/payment` khi booking chưa done → 400.

---

## 8. Backlog (không build cho MVP)

| Thứ | Khi nào cần |
|---|---|
| `wash_steps` collection (pre_wash/exterior/interior/drying/final_check) | Khi shop muốn track productivity từng bước |
| `status_history` polymorphic audit log | Khi cần báo cáo audit / compliance |
| Booking status `QUALITY_CHECK`, `NEEDS_REWORK` | Khi gặp khách phàn nàn nhiều, cần loop rework |
| Booking status `REFUNDED` | Khi có chính sách hoàn tiền |
| `vehicle_types.price_multiplier` | Khi muốn 1 service áp nhiều loại xe với giá khác |
| Endpoint `switch-service` / `switch-vehicle` tại counter | Khi case đổi gói/đổi xe xảy ra thường xuyên |
| Inspection BEFORE bắt buộc gate (washer không start được nếu chưa có) | Khi có vấn đề pháp lý về damage claim |
| Walk-in tự tạo Booking ngầm | Khi cần báo cáo doanh thu thống nhất |
| Cash session quản lý ca thu ngân | Khi nhiều cashier làm việc cùng lúc |

---

## 9. Tham chiếu

- Schema: [DB_SCHEMA_AS_BUILT.md](./DB_SCHEMA_AS_BUILT.md)
- Status: [IMPLEMENTATION_STATUS.md](./IMPLEMENTATION_STATUS.md)
- Audit gốc: [BUSINESS_FLOW_AUDIT.md](./BUSINESS_FLOW_AUDIT.md)
- Branch: `feature/wash-sessions` @ commit `31e67a2`
