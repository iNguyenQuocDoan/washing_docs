# Booking Flow — End-to-End

**Phạm vi:** Mô tả đầy đủ luồng nghiệp vụ Booking trên kiến trúc hiện tại — từ lúc Customer chọn gói đến khi Cashier đóng đơn — và map sang code thực tế.
**Branch tham chiếu:** `main` (entity `Order` thống nhất Booking + Payment)
**Last update:** 2026-05-19

> Tài liệu này dùng để onboard người mới, làm spec test, và làm chuẩn so sánh khi build FE. Scope: **MVP shop nhỏ**.

---

## 1. Kiến trúc — 1 entity duy nhất

Trên branch `main`, **Booking và Payment hợp nhất thành 1 entity `Order`**. Không có collection `bookings` rời, không có `wash_sessions`, không có `inspections`. Mọi state của 1 lần rửa xe đều nằm trên `orders`.

| Khái niệm cũ | Mapping trên main |
|---|---|
| `Booking` (lịch hẹn) | Cùng document `Order` |
| `Order` (PayOS) | Cùng document `Order` (`payos_order_code`, `payos_checkout_url`) |
| `WashSession` | Status trên `Order` (`checked_in` → `in_progress` → `completed`) |
| `payment_status` ngoài | Cùng document `Order` (`payment_status`) |

→ Khi cashier "đóng đơn" = `status='completed'` AND `payment_status='paid'`.

---

## 2. Actors

| Role | Code | Ghi chú |
|---|---|---|
| Customer | `customer` | End user, tự đăng ký qua `POST /auth/register` |
| Cashier | `cashier` | Quản lý order tại counter: confirm, check-in, mark-paid (cash), update status |
| Manager | `manager` | Có toàn quyền cashier |
| Admin | `admin` | Toàn quyền |
| ~~Washer~~ | — | **Không có washer role flow** ở phiên bản hiện tại. Cashier điều hướng order qua state thay tay. |

Roles seed tự động khi app khởi động ([auth.module.ts](../src/features/auth/auth.module.ts)).

---

## 3. Happy path — Online payment (PayOS)

### Bước 1 — Customer chọn gói + xe

| Hành động | Endpoint |
|---|---|
| Xem danh sách gói rửa | `GET /service-types` |
| Xem loại xe | `GET /vehicle-types` |
| Quản lý xe của mình | `GET/POST/PATCH /me/vehicles` |

**Quy tắc:**
- `service_types.is_active=false` → không hiển thị public.
- Vehicle ownership-protected: customer chỉ thấy xe của chính mình.
- Xe mới đầu tiên auto-set `is_default=true`.

### Bước 2 — Customer tạo order online

```
POST /me/orders
Role: JWT bất kỳ (default customer)
Headers: Idempotency-Key (optional, 8-128 chars [A-Za-z0-9_-:.])
Body: { vehicleId, serviceTypeId, scheduledAt, paymentMethod: 'online', note? }
```

[order.controller.ts:37-63](../src/features/order/order.controller.ts#L37). Idempotency cache 24h theo header — retry trả về response cũ + header `Idempotent-Replayed=true`.

**Validate** ([order.service.ts:69-217](../src/features/order/services/order.service.ts#L69)):
1. Vehicle thuộc customer + còn active.
2. Service active.
3. `scheduledAt` ≥ hiện tại - 60s.
4. `scheduledAt` ≤ `now + tier.booking_window_days` (BR-06).
5. Customer chưa có quá `MAX_ACTIVE_BOOKINGS_PER_CUSTOMER` order ở status active (default 3, BR-08).
6. **Server auto-pick shift** chứa `scheduledAt` và còn slot — atomic `$inc current_bookings`. Loop qua candidates phòng race condition.

> ⚠ **Khác wash-sessions branch:** FE **KHÔNG** truyền `staffShiftId` khi tạo order. BE tự match.

**Tạo Order:**
- `status = 'pending_payment'`
- `payment_status = 'unpaid'`
- `payment_method = 'online'`
- `amount = round(service.base_price)` — VND integer
- `payos_order_code` = mã unique (Redis INCR + timestamp)
- Gọi PayOS API → lưu `payos_checkout_url` + `payos_payment_link_id`

**Response:** `OrderResponseDto` với `payosCheckoutUrl` để FE redirect.

**Rollback** nếu PayOS lỗi → cancel order + release shift slot.

**Email:** KHÔNG gửi confirmation ở bước này. Đợi webhook PAID.

### Bước 3 — Customer thanh toán PayOS

FE redirect sang `payos_checkout_url`. Customer thanh toán xong → PayOS gọi webhook về.

### Bước 4 — Webhook PayOS xử lý

```
POST /payments/webhook   (PayOS gọi, public)
```

[order.controller.ts:117-127](../src/features/order/order.controller.ts#L117), [order.service.ts:376-483](../src/features/order/services/order.service.ts#L376).

**Idempotency 4 lớp:**
1. Verify chữ ký PayOS.
2. Redis SETNX trên `payos:txn:<reference>` TTL 30 ngày — dedup webhook replay.
3. Redis per-order lock `lock:order:<orderCode>` TTL 10s.
4. DB unique sparse index trên `payos_transaction_id`.

**Khi `code === '00'` (PAID):**
- `Order.status` → `'confirmed'`
- `Order.payment_status` → `'paid'`
- Gửi email confirmation (await, không fire-and-forget — Vercel serverless).
- Insert `payment_transactions` record.

**Khi code khác (FAILED/CANCELLED):**
- Release shift slot.
- `Order.status` → `'cancelled'`, `cancel_reason = 'Payment failed'`.

### Bước 5 — Khách đến shop, Cashier check-in

```
PATCH /admin/orders/:id/status
Body: { status: 'checked_in' }
Role: cashier, manager, admin
```

[admin-order.controller.ts:58-72](../src/features/order/admin-order.controller.ts#L58).

**Quy tắc:** Order phải ở `confirmed`. State machine ([order.state-machine.ts:11-15](../src/features/order/order.state-machine.ts#L11)).

> Bước này NGẦM hiểu là "cashier đã verify xe khớp order + check-in". Không có endpoint check-in riêng.

### Bước 6 — Bắt đầu rửa

```
PATCH /admin/orders/:id/status
Body: { status: 'in_progress' }
```

Cashier nhấn khi nhân viên bắt đầu rửa. State machine cho phép `checked_in → in_progress`.

### Bước 7 — Hoàn tất

```
PATCH /admin/orders/:id/status
Body: { status: 'completed' }
```

Terminal state. Shift slot tự release (vì `completed` không nằm trong `ACTIVE_ORDER_STATUSES`).

→ **Đóng đơn** = `status='completed'` AND `payment_status='paid'`. Với online flow đã `paid` từ Bước 4, không cần làm gì thêm.

---

## 4. Happy path — Cash payment

### Bước 1-2 giống online, nhưng:

```
POST /me/orders
Body: { vehicleId, serviceTypeId, scheduledAt, paymentMethod: 'cash', note? }
```

**Khác online:**
- `status = 'confirmed'` ngay (không qua `pending_payment` vì không cần đợi PayOS).
- `payment_status = 'unpaid'`.
- KHÔNG tạo PayOS link.
- Email confirmation gửi NGAY ở bước create.

### Bước 3 — Khách đến, cashier check-in → in_progress → completed

Giống Bước 5-7 ở §3.

### Bước 4 — Cashier thu tiền mặt và mark paid

```
POST /admin/orders/:id/mark-paid
Role: cashier, manager, admin
```

[admin-order.controller.ts:74-87](../src/features/order/admin-order.controller.ts#L74), [order.service.ts:592-612](../src/features/order/services/order.service.ts#L592).

**Validate:**
- `payment_method === 'cash'` (online không được mark tay).
- Idempotent — nếu đã `paid` trả về luôn không lỗi.

**Kết quả:** `payment_status = 'paid'`.

→ **Đóng đơn** = `status='completed'` AND `payment_status='paid'`.

> 💡 **Thời điểm mark-paid linh hoạt:** có thể gọi trước, sau, hoặc song song với chuỗi `checked_in → in_progress → completed`. Không có ràng buộc status để gọi `/mark-paid`.

---

## 5. Alternative flows / Edge cases

### 5.1 Customer hủy order

```
PATCH /me/orders/:id/cancel
Body: { reason? }
```

[order.service.ts:326-365](../src/features/order/services/order.service.ts#L326).

Chỉ cho phép từ `pending_payment` hoặc `confirmed` ([order.state-machine.ts:39-44](../src/features/order/order.state-machine.ts#L39)).

**Side effects:**
- Nếu online + còn `pending_payment` → cancel PayOS link best-effort.
- Release shift slot (nếu status còn consume).
- `status = 'cancelled'`, `cancel_reason`.

### 5.2 Customer đổi giờ order

```
PATCH /me/orders/:id/reschedule
Body: { staffShiftId, scheduledAt }
```

[order.service.ts:245-324](../src/features/order/services/order.service.ts#L245).

**Khác create:** ở đây FE **PHẢI** truyền `staffShiftId` (vì user chủ động chọn shift mới qua UI).

**Quy tắc:**
- Chỉ cho phép từ `pending_payment` hoặc `confirmed`.
- Cap `MAX_RESCHEDULES=2` (env).
- Atomic: reserve slot mới trước, rồi release slot cũ. Rollback nếu fail giữa chừng.
- `scheduledAt + service.estimated_minutes ≤ newShift.end_at`.

### 5.3 Cashier đánh dấu khách không đến (no-show)

```
PATCH /admin/orders/:id/status
Body: { status: 'no_show', reason? }
```

Cho phép từ `confirmed` hoặc `checked_in`. Tự release shift slot.

### 5.4 Cashier hủy order (trước khi rửa)

```
PATCH /admin/orders/:id/status
Body: { status: 'cancelled', reason? }
```

**Chỉ cho phép từ `pending_payment`, `confirmed`, hoặc `checked_in`** ([order.state-machine.ts:6-25](../src/features/order/order.state-machine.ts#L6)).

⚠ **Không cancel được order đang `in_progress`** — state machine chỉ cho `in_progress → completed`. Nếu thật sự cần dừng giữa chừng, hiện tại phải sửa tay qua Mongo Atlas hoặc backlog cho phép `in_progress → cancelled` (xem §8).

Tự release shift slot khi vào terminal.

### 5.5 PayOS payment timeout

Cron `OrderExpiryCron` ([jobs/order-expiry.cron.ts](../src/features/order/jobs/order-expiry.cron.ts)) chạy **mỗi phút**:
- Tìm order `status='pending_payment'` có `created_at < now - paymentTimeoutMinutes` (default 15 phút).
- Set `cancelled` + `cancel_reason='Payment timeout'`.
- Release shift slot + cancel PayOS link best-effort.

### 5.5b Cash no-show timeout

Cron `CashNoShowCron` ([jobs/cash-no-show.cron.ts](../src/features/order/jobs/cash-no-show.cron.ts)) chạy **mỗi phút**:
- Tìm order `status='confirmed'` AND `payment_method='cash'` AND `payment_status='unpaid'` có `scheduled_at < now - cashArrivalGraceMinutes` (default 30 phút sau giờ hẹn).
- Set `no_show` + `cancel_reason='No arrival within grace window'`.
- Release shift slot.

→ Đóng gap "cash order chiếm slot vĩnh viễn nếu khách không đến". Cashier vẫn có thể chủ động chuyển `no_show` sớm hơn qua `PATCH /admin/orders/:id/status` trước khi cron sweep.

**Env**: `CASH_ARRIVAL_GRACE_MINUTES` (5-240, default 30).

### 5.6 Webhook đến sau khi cron đã expire

Webhook check `order.status === 'pending_payment'` trước khi update ([order.service.ts:430-452](../src/features/order/services/order.service.ts#L430)). Nếu đã `cancelled` (do cron timeout), bỏ qua state update — chỉ ghi `payment_transactions` audit log.

⚠ **Edge case nghiêm trọng:** customer click thanh toán đúng lúc cron 15 phút cancel order. Webhook PAID đến muộn → tiền đã trừ nhưng order đã `cancelled` → **không tự refund**. Hiện tại phải xử lý tay:
1. Kiểm tra `payment_transactions` để xác nhận đã nhận tiền (`status` từ webhook PayOS).
2. Refund tay qua PayOS dashboard, hoặc tạo lại order rồi mark-paid + đánh dấu nội bộ.

Cách giảm rủi ro: tăng `BOOKING_PAYMENT_TIMEOUT_MINUTES` (default 15) — nhưng cũng giữ shift slot lâu hơn. Hoặc backlog: webhook PAID đến muộn → auto un-cancel + paid + flag.

→ Trade-off này KHÔNG ghi trong code, không có warning ở runtime. FE nên hiển thị countdown để khách thấy.

### 5.7 Cashier search tại counter

```
GET /admin/orders
  ?customerPhone=0901
  &vehicleLicensePlate=51A
  &status=confirmed
  &paymentStatus=unpaid
  &paymentMethod=cash
  &scheduledFrom=2026-06-01
  &scheduledTo=2026-06-30
  &page=1&limit=20
```

[admin-order.controller.ts:39-48](../src/features/order/admin-order.controller.ts#L39), [order.service.ts:487-535](../src/features/order/services/order.service.ts#L487).

Search `customerPhone` / `vehicleLicensePlate` là substring case-insensitive — phù hợp UX cashier gõ vài số đầu của SĐT hoặc biển số.

### 5.8 Khách walk-in (không booking trước)

**Hiện tại KHÔNG hỗ trợ trực tiếp.** Tất cả order phải do customer tạo qua `POST /me/orders`. Workaround:
1. Cashier giúp khách register (`POST /auth/register`) hoặc tạo user qua admin endpoint.
2. Khách tự (hoặc cashier dùng JWT khách) tạo order với `scheduledAt = now`.
3. Cashier confirm + check-in ngay.

→ Nếu use case walk-in nhiều, cần build endpoint `POST /admin/orders/walk-in` riêng (xem §8 Backlog).

### 5.9 Khách đổi gói/đổi xe tại counter

Hiện tại không có endpoint `switch-service` / `switch-vehicle`. Workaround: cashier set `cancelled` order cũ, customer tạo lại order mới.

### 5.10 Idempotent order creation

`POST /me/orders` chấp nhận header `Idempotency-Key` (8-128 chars [A-Za-z0-9_-:.]):
- Lần đầu: tạo order như bình thường, cache response 24h.
- Lần sau cùng key: trả response cache + header `Idempotent-Replayed=true`. Không tạo order trùng.

Phòng trường hợp FE retry do mạng chập chờn, đặc biệt khi đã có PayOS link.

---

## 6. State machine ([order.state-machine.ts](../src/features/order/order.state-machine.ts))

```
                                        ┌────────────┐
                                        ▼            │
pending_payment ──→ confirmed ──→ checked_in ──→ in_progress ──→ completed (terminal)
       │                │              │
       └──→ cancelled   └──→ cancelled └──→ cancelled              (terminal)
       └─ (auto-expire  └──→ no_show   └──→ no_show                (terminal)
           15 min →
           cancelled)
```

**Transitions hợp lệ (tổng kết):**

| Từ | Sang |
|---|---|
| `pending_payment` | `confirmed`, `cancelled` |
| `confirmed` | `checked_in`, `cancelled`, `no_show` |
| `checked_in` | `in_progress`, `cancelled`, `no_show` |
| `in_progress` | `completed` (KHÔNG cho cancel/no_show — xem §5.4) |
| `completed` / `cancelled` / `no_show` | terminal |

**Active statuses (consume shift capacity):** `pending_payment`, `confirmed`, `checked_in`, `in_progress`. Khi transition ra terminal → atomic `$inc current_bookings -1`.

**Terminal:** `completed`, `cancelled`, `no_show`.

**Payment status (orthogonal):**
```
unpaid ──→ paid  (online: via webhook; cash: via /mark-paid)
       ──→ refunded (chưa có flow, để backlog)
```

---

## 7. Entity `orders` — Field reference

[order.entity.ts](../src/features/order/entities/order.entity.ts). Tất cả field đều trên 1 document duy nhất:

### Relationships
- `customer_id` → users (required, IX)
- `vehicle_id` → vehicles (required, IX)
- `service_type_id` → service_types (required, IX)
- `staff_shift_id` → staff_shifts (required, IX)

### Scheduling
- `scheduled_at` Date (required)
- `status` enum (required, IX, default `pending_payment`)
- `priority_level` Number (default 0) — snapshot từ `tier.priority_level` lúc tạo
- `reschedule_count` Number (default 0)
- `cancel_reason` String (optional)
- `note` String (optional) — customer note cho staff

### Payment
- `payment_method` enum `'online' | 'cash'` (required)
- `payment_status` enum `'unpaid' | 'paid' | 'refunded'` (required, IX, default `unpaid`)
- `amount` Number (required, VND integer)
- `payos_order_code` Number (unique sparse, IX) — chỉ có với online
- `payos_checkout_url` String (optional)
- `payos_payment_link_id` String (optional)

### Indexes
- `{ customer_id: 1, scheduled_at: -1 }` compound
- `{ scheduled_at: 1, status: 1 }` compound
- `{ customer_id: 1, status: 1 }` compound

---

## 8. Backlog (không build cho MVP)

| Thứ | Khi nào cần |
|---|---|
| Walk-in endpoint `POST /admin/orders/walk-in` | Khi shop có nhiều khách không đặt trước |
| Washer role flow + assign-washer endpoint | Khi cần track productivity từng washer |
| Inspection (before/after photos + signature) | Khi cần damage claim defense |
| Status `refunded` + refund flow | Khi có chính sách hoàn tiền |
| Tier `discount_percent` áp dụng vào `amount` | Hiện chưa apply discount vào pricing |
| Loyalty points earn khi `paid` | Hiện chưa award points sau payment |
| `visits_this_month` increment | Hiện chưa cập nhật loyalty visits |
| ~~Auto-expire `confirmed` cash orders quá hạn~~ | ✅ Đã build — `CashNoShowCron` sweep mỗi phút, env `CASH_ARRIVAL_GRACE_MINUTES` |
| Switch-service / switch-vehicle tại counter | Khi case đổi xảy ra thường xuyên |
| Cash session quản lý ca thu ngân | Khi nhiều cashier làm việc cùng lúc |

---

## 9. Gap nghiệp vụ cần lưu ý

1. **Loyalty chưa wire vào Order flow.** `tier_configs` đã có `points_per_1000_vnd` và `discount_percent`, nhưng `Order.amount` lấy thẳng `base_price` không trừ discount, và không có hook nào award points khi `payment_status='paid'`. Khi build Phase Loyalty, hook vào webhook + `/mark-paid`.

2. ~~Cash order không có timeout.~~ ✅ Đã build (2026-05-19) — `CashNoShowCron` sweep `confirmed` + `cash` + `unpaid` có `scheduled_at < now - cashArrivalGraceMinutes` → `no_show` + release slot.

3. **Status transition không gửi notification.** Khi cashier check-in / completed, customer không nhận thông báo. Chỉ có 1 email duy nhất ở thời điểm `confirmed`.

4. **Không có audit log status transition.** Khi cần báo cáo sau, không trace được ai/khi nào thay đổi status. Backlog: `order_status_history` collection.

---

## 10. Test plan (manual)

### Cash flow
1. `POST /me/orders` với `paymentMethod=cash` → 201, status=`confirmed`, payment_status=`unpaid`, email gửi.
2. `PATCH /admin/orders/:id/status` { checked_in } → 200.
3. `PATCH /admin/orders/:id/status` { in_progress } → 200.
4. `PATCH /admin/orders/:id/status` { completed } → 200. Shift slot release.
5. `POST /admin/orders/:id/mark-paid` → 200, payment_status=`paid`.

### Online flow
1. `POST /me/orders` với `paymentMethod=online` → 201, status=`pending_payment`, payosCheckoutUrl có.
2. Mở `payosCheckoutUrl`, thanh toán test.
3. Đợi webhook → order chuyển `confirmed`+`paid`, email gửi.
4. Lặp các step admin → `completed`.

### Negative
- Tạo order khi đã có 3 active → 400 "limit 3".
- Reschedule lần 3 → 400 "Reschedule limit reached".
- Cancel order ở `in_progress` (customer) → 400.
- Webhook signature sai → silently dropped (log warn).
- `mark-paid` order online → 400.

---

## 11. Tham chiếu

- Schema: [DB_SCHEMA_AS_BUILT.md](./DB_SCHEMA_AS_BUILT.md) — ⚠ phần Booking/wash_sessions đã obsolete, đọc §13 (`orders`).
- Audit cũ trên wash-sessions: [BUSINESS_FLOW_AUDIT.md](./BUSINESS_FLOW_AUDIT.md) — đã ARCHIVED, gap MVP đó giải quyết khác trên main.
- Booking config: [booking.config.ts](../src/config/booking.config.ts).
- Branch: `main` @ commit `046bc66` (hotfix/cors-wave-wash dựa trên main).
