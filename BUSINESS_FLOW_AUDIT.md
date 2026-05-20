# Business Flow Audit — Car Washing Booking (Lean MVP)

> ⚠️ **ARCHIVED — 2026-05-19**
>
> Tài liệu này audit branch `feature/wash-sessions` (Booking + WashSession + Order rời nhau) và đề xuất link `Booking ↔ Order`.
>
> **`main` đã giải quyết khác:** gộp Booking + Payment vào 1 entity `Order` duy nhất. Gap P1 đề cập dưới đây **không còn hiệu lực**.
>
> Đọc [BOOKING_FLOW.md](./BOOKING_FLOW.md) cho luồng nghiệp vụ hiện hành. Giữ file này lại làm context lịch sử về quyết định kiến trúc.

---

**Ngày audit:** 2026-05-18
**Branch:** `feature/wash-sessions`
**So sánh với:** `origin/main` (rev `da6557c`)
**Scope:** Project sem-8 / shop nhỏ, không enterprise.

> Bản này tập trung **chỉ thứ thật sự cần để chạy đúng nghiệp vụ MVP**. Phiên bản đầy đủ (status_history, wash_steps, quality_check loop, refund flow, vehicle_type multiplier...) là backlog enterprise, không bắt buộc cho project hiện tại.

---

## 1. Luồng nghiệp vụ và trạng thái code

| # | Bước | Status | Ghi chú |
|---:|---|:---:|---|
| 1 | Chọn gói rửa | ✅ | `service_types`, public read |
| 2 | Chọn loại xe + lưu xe | ✅ | `vehicle_types` + `vehicles` |
| 3 | Tạo booking | ✅ | `POST /me/bookings` đầy đủ validate |
| 4 | Cashier xem booking | ✅ | `GET /admin/bookings` + search phone/plate |
| 5 | Cashier mở phiên rửa (verify + check-in) | ✅ | `POST /admin/wash-sessions/from-booking` — ngầm là check-in |
| 6 | Gán Washer | ✅ | `PATCH /admin/wash-sessions/:id/assign-washer` |
| 7 | Washer rửa xe | ✅ | start / complete (đủ cho MVP) |
| 8 | Hoàn tất dịch vụ | ✅ | washer complete → booking auto = `done` |
| 9 | **Thanh toán + đóng đơn** | ⛔ | **GAP DUY NHẤT P1** |

---

## 2. Vấn đề duy nhất phải fix (P1)

**`Order` (PayOS) và `Booking` đang chạy rời nhau.**

- `Order` ([payment/entities/order.entity.ts](../src/features/payment/entities/order.entity.ts)) không có `booking_id`.
- `Booking` ([booking/entities/booking.entity.ts](../src/features/booking/entities/booking.entity.ts)) không có `payment_status`, không có `paid_at`.
- Webhook PayOS ([payment.service.ts:172-220](../src/features/payment/payment.service.ts#L172)) chỉ update Order.status, không động vào Booking.
- Cashier không có nút "đã thu tiền mặt" cho booking.

**Hệ quả:** Sau khi washer `complete`, booking ở status `done` nhưng không biết đã thanh toán chưa. Không có cách "đóng đơn".

---

## 3. Fix tối thiểu (MVP)

### 3.1 Thêm 4 field vào `bookings`

| Field | Type | Default | Mục đích |
|---|---|---|---|
| `payment_status` | enum `'unpaid'` \| `'paid'` | `'unpaid'` | Trạng thái thanh toán |
| `payment_method` | enum `'cash'` \| `'online'` | null | Set khi cashier xác nhận |
| `paid_at` | Date | null | Thời điểm thanh toán |
| `final_price` | Decimal128 | null | Snapshot giá lúc đóng đơn (phòng admin đổi giá service sau) |

> **Không thêm:** `checked_in_at`, `verified_by_cashier_id`, `closed_at`, `closed_by_cashier_id`, `order_id` denormalized. Không cần cho MVP — query qua `Order.booking_id` đủ.

### 3.2 Thêm 1 field vào `orders` (PayOS)

| Field | Type | Constraint | Mục đích |
|---|---|---|---|
| `booking_id` | ObjectId FK→bookings | sparse UQ | 1 booking ↔ ≤1 order online |

> Cho phép null vì có Order khởi tạo trước khi link (FE chưa truyền booking_id) hoặc walk-in không booking.

### 3.3 Thêm 1 endpoint cho cashier

```
PATCH /admin/bookings/:id/payment
Role: cashier, manager, admin
Request body:
  {
    method: 'cash' | 'online',
    finalPrice: number,         // VND, integer
    orderId?: string            // bắt buộc nếu method=online
  }
Response: BookingResponseDto (with payment_status=paid)
Business rule:
  - Booking phải ở status 'done'
  - Booking.payment_status phải = 'unpaid'
  - Nếu method=online: Order.status phải = 'PAID' và Order.booking_id phải match
  - Set: payment_status='paid', payment_method, paid_at=now, final_price
```

### 3.4 Update PayOS webhook

Trong [payment.service.ts:handleWebhook](../src/features/payment/payment.service.ts#L172):
- Khi nhận `PAID` event, nếu `Order.booking_id` có → auto set `Booking.payment_status='paid'`, `Booking.payment_method='online'`, `Booking.paid_at=now`, `Booking.final_price=Order.amount`.

### 3.5 Update `POST /me/orders` (CreateOrderDto)

Thêm optional `bookingId` để FE truyền khi tạo payment link cho 1 booking đã có. Lưu vào `Order.booking_id`.

---

## 4. Status flow sau khi fix

**Booking:**
```
pending → confirmed → in_progress → done
                                    └─ payment_status: unpaid → paid (terminal)
+ cancelled / no_show (terminal)
```

→ **Không thêm `CHECKED_IN` / `QUALITY_CHECK` / `CLOSED` status.** Việc check-in đã ngầm qua tạo wash_session. Đóng đơn = `status='done'` AND `payment_status='paid'`.

**WashSession:** giữ nguyên (`waiting → assigned → in_progress → done`).

**Order (PayOS):** giữ nguyên (`PENDING → PAID/CANCELLED/EXPIRED`).

---

## 5. Backlog (KHÔNG làm cho MVP)

Nếu sau này shop scale lớn, các thứ sau có thể thêm — nhưng **không cần cho project hiện tại**:

- `wash_steps` collection (5 sub-step). MVP: washer bấm 1 nút done là đủ.
- `status_history` polymorphic audit log. MVP: dùng `Logger.log` (đã có).
- `QUALITY_CHECK` + `NEEDS_REWORK` flow. MVP: cashier tự kiểm tra trước khi nhấn payment.
- `REFUNDED` status. MVP: hiếm gặp, xử lý tay qua Mongo Atlas khi cần.
- `vehicle_types.price_multiplier`. MVP: tạo nhiều service khác nhau ("Rửa xe máy" / "Rửa sedan") thay vì multiplier.
- Endpoint `switch-service` / `switch-vehicle` tại counter. MVP: cancel booking cũ + tạo mới.
- Inspection BEFORE gate (bắt buộc trước khi `/start`). MVP: optional như hiện tại.
- Walk-in tự tạo Booking sau lưng. MVP: walk-in = wash_session, không cần booking.

---

## 6. Roadmap fix

**1 phase duy nhất — ước ~1 ngày:**

1. Add 4 field vào `Booking` entity + migration mặc định `payment_status='unpaid'` cho records cũ.
2. Add `booking_id` (sparse UQ) vào `Order` entity.
3. Add `bookingId?` optional vào `CreateOrderDto`.
4. Update `payment.service.ts.createOrder` để lưu `booking_id`.
5. Update `payment.service.ts.handleWebhook` để auto-sync `Booking.payment_status` khi PayOS PAID.
6. Add endpoint `PATCH /admin/bookings/:id/payment` ở `admin-booking.controller.ts`.
7. Update `BookingResponseDto` để expose 4 field mới.

Hết.

---

## 7. Quan hệ với `origin/main`

Main hợp nhất Booking + Payment vào 1 entity `Order` duy nhất với status flow đầy đủ (`pending_payment → confirmed → checked_in → in_progress → completed`).

**Branch hiện tại KHÔNG đi theo hướng đó** — giữ Booking + WashSession + Order tách rời. Lý do:
- Cho phép walk-in (wash_session không cần booking).
- Tách rõ vai trò: Booking = lịch hẹn, WashSession = vận hành, Order = thanh toán online.

**Khuyến nghị:** không merge thô `origin/main` vào. Có thể cherry-pick:
- `a844e3b` (CI/CD workflow)
- `b095ae7` (Swagger setup)
- `cace6d4` (Vercel serverless)

---

## 8. Tham chiếu

- Audit conversation: 2026-05-18.
- Commit wash-session: `31e67a2 feat(wash-session): cashier↔washer handoff with inspections + photos`.
- Schema as-built: [DB_SCHEMA_AS_BUILT.md](./DB_SCHEMA_AS_BUILT.md).
