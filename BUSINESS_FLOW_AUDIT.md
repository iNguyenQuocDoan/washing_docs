# Business Flow Audit — Car Washing Booking

**Ngày audit:** 2026-05-18
**Branch audit:** `feature/wash-sessions`
**So sánh với:** `origin/main` (rev `da6557c`)
**Mức đáp ứng nghiệp vụ:** ~60–65%

> Tài liệu này tổng hợp các vấn đề tìm được khi đối chiếu code với luồng nghiệp vụ chuẩn:
> *Customer chọn gói → chọn xe → booking → Cashier verify → mở wash session → gán Washer → thực hiện rửa → hoàn tất → thanh toán → đóng đơn.*

---

## 1. Tóm tắt luồng nghiệp vụ vs code hiện tại

| # | Bước nghiệp vụ | Status | Ghi chú nhanh |
|---:|---|:---:|---|
| 1 | Chọn gói rửa | ✅ | `service_types` đủ field, public `GET /service-types` |
| 2 | Chọn loại xe + lưu xe | ✅ | `vehicle_types` + `vehicles` (license unique, ownership) |
| 3 | Tạo booking | ✅ | `POST /me/bookings`, atomic shift capacity, tier window |
| 4 | Cashier xem booking | ✅ | `GET /admin/bookings` + search phone/plate |
| 5 | Cashier verify booking ↔ xe | ⚠ | Chỉ ngầm qua "tạo wash_session"; không có flag `verified_by_cashier`, không có flow đổi gói/đổi xe tại counter |
| 6 | Chuyển vào quy trình | ✅ ý / ⚠ tên | Mở `wash_session.WAITING`, nhưng booking thiếu state `CHECKED_IN` |
| 7 | Gán Washer | ✅ | `PATCH /admin/wash-sessions/:id/assign-washer` |
| 8 | Washer thực hiện các bước rửa | ⛔ | Chỉ có `start` + `complete` — KHÔNG có sub-step (pre_wash/exterior/interior/drying/final_check) |
| 9 | Hoàn tất dịch vụ | ⚠ | Washer mark DONE → booking auto DONE; thiếu quality-check, thiếu rework loop |
| 10 | Thanh toán / đóng đơn | ⛔ | `Order` (PayOS) chạy rời, KHÔNG liên kết với `booking_id` hay `wash_session_id`; booking thiếu `payment_status`/`final_price` |

---

## 2. Vấn đề chính (theo mức ưu tiên)

### P1 — Bắt buộc fix để chạy đúng nghiệp vụ

1. **Booking không liên kết Payment.**
   - `Order` ([payment/entities/order.entity.ts](../src/features/payment/entities/order.entity.ts)) không có `booking_id`, không có `wash_session_id`.
   - `Booking` ([booking/entities/booking.entity.ts](../src/features/booking/entities/booking.entity.ts)) không có `payment_status`, `final_price`, `payment_method`, `paid_at`, `cashier_id`.
   - **Hệ quả:** Cashier không có cách "đóng đơn"; báo cáo doanh thu không truy xuất được từ booking.

2. **Thiếu trạng thái `CHECKED_IN` ở Booking.**
   - State machine hiện tại: `pending → confirmed → in_progress → done` ([booking.state-machine.ts:3-18](../src/features/booking/booking.state-machine.ts#L3)).
   - Nghiệp vụ cần bước "khách đã đến + cashier verify xe" trước khi vào `in_progress`.

3. **Thiếu sub-step rửa xe cho Washer.**
   - Washer hiện chỉ có `start` ([washer-wash-session.controller.ts:49-67](../src/features/wash-session/washer-wash-session.controller.ts#L49)) và `complete` ([:69-87](../src/features/wash-session/washer-wash-session.controller.ts#L69)).
   - **Không có** entity / endpoint cho: `pre_wash`, `exterior`, `interior`, `drying`, `final_check`.
   - **Hệ quả:** Không truy vết được bước nào đang làm, không có lịch sử thao tác.

4. **Thiếu bước "quality check / xác nhận đóng đơn" của Cashier.**
   - Sau khi washer `complete`, hệ thống auto-set booking = `done`. Không có nút Cashier "approve" hay "rework".
   - Không có endpoint `POST /admin/bookings/:id/close`.

### P2 — Cần để nghiệp vụ rõ ràng

5. **`VehicleType` không ảnh hưởng giá.** Schema chỉ có `name`/`description`/`is_active` ([vehicle-type.entity.ts](../src/features/vehicle-type/entities/vehicle-type.entity.ts)). Thiếu `price_multiplier` hoặc bảng pricing `service_type × vehicle_type`.

6. **Không có flow xử lý mismatch tại counter** (sai gói / sai loại xe / khách đến trễ): không có endpoint `switch-service`, `switch-vehicle`. Trường hợp khách không đến tuy có `no_show` nhưng không có audit log.

7. **Inspection không gate state machine.** `BEFORE` inspection ([inspection.service.ts:37-46](../src/features/wash-session/inspection.service.ts#L37)) là optional — washer có thể `/start` mà chưa có inspection.

8. **Không có `StatusHistory` / `AuditLog`.** Mỗi lần đổi status không lưu `from → to`, `actor_id`, `reason`, `at`. Hiện chỉ log qua `Logger` console.

9. **WashSession state thiếu `QUALITY_CHECK` / `NEEDS_REWORK`.** Enum chỉ có 5 giá trị ([wash-session-status.enum.ts](../src/features/wash-session/types/wash-session-status.enum.ts)).

### P3 — Tối ưu sau

10. `WashSession` không lưu `assigned_at` riêng (dùng `updated_at` chung).
11. `Order` thiếu `REFUNDED`.
12. Wash session walk-in không tự tạo booking → khó báo cáo dồn 1 chỗ.

---

## 3. ERD đề xuất (luồng đầy đủ)

> Phần in đậm là **field/collection MỚI cần thêm**. Phần gạch ngang là entity hiện tại.

```
┌─────────────┐         ┌──────────────┐         ┌────────────────┐
│   roles     │1───→N│   users       │1───→N│   vehicles      │
└─────────────┘         └──────┬───────┘         └────────┬───────┘
                               │1                          │N
                               │N                          │1
                       ┌───────▼────────┐         ┌────────▼───────┐
                       │ loyalty_accts  │         │ vehicle_types  │
                       └───────┬────────┘         │ + price_multi★ │
                               │N                 └────────────────┘
                               │1
                       ┌───────▼────────┐
                       │  tier_configs  │
                       └────────────────┘

┌────────────────┐       ┌───────────────────────────────────┐
│  staff_shifts  │1────N│            BOOKING                 │
└────────────────┘       │  + checked_in_at★                 │
                         │  + verified_by_cashier_id★        │
                         │  + estimated_price★               │
                         │  + final_price★                   │
                         │  + payment_status★                │
                         │  + payment_method★                │
                         │  + paid_at★                       │
                         │  + closed_by_cashier_id★          │
                         │  + closed_at★                     │
                         └─┬──────────┬────────────┬─────────┘
                           │1         │1           │1
                           │1         │1           │0..1
                  ┌────────▼───┐ ┌────▼───┐  ┌─────▼─────────┐
                  │ wash_session│ │ ORDER  │  │status_history★│
                  │ + qc_at★    │ │+booking│  │(polymorphic)  │
                  │ + qc_by★    │ │_id★    │  └───────────────┘
                  └─┬───┬───────┘ └────────┘
                    │N  │1
                    │1  ▼
                    │   ┌─────────────┐
                    │   │ INSPECTION  │1───N→┌──────────────────┐
                    │   │ (before/    │      │inspection_photos │
                    │   │  after)     │      └──────────────────┘
                    │   └─────────────┘
                    │N
                    ▼
              ┌────────────────────┐
              │   wash_steps★      │
              │  - step (enum)     │
              │  - started_at      │
              │  - finished_at     │
              │  - washer_id       │
              │  - note            │
              │  - photo_url[]     │
              └────────────────────┘
```

★ = **field hoặc collection mới cần thêm**

---

## 4. Field cần thêm cho từng collection

### 4.1 `bookings` (P1)

| Field mới | Type | Nullable | Mục đích |
|---|---|:---:|---|
| `estimated_price` | Decimal128 | N | Snapshot giá lúc tạo booking (service.base_price × vehicle_type.price_multiplier - tier.discount%) |
| `final_price` | Decimal128 | Y | Giá cuối (sau khi cashier điều chỉnh / áp khuyến mãi) |
| `payment_method` | String enum {`cash`,`online`} | Y | Set lúc cashier xác nhận thanh toán |
| `payment_status` | String enum {`unpaid`,`paid`,`refunded`} | N (default `unpaid`) | |
| `paid_at` | Date | Y | Khi cashier xác nhận / webhook PayOS |
| `order_id` | ObjectId FK→orders | Y | Liên kết Order (online) — null nếu cash |
| `wash_session_id` | ObjectId FK→wash_sessions | Y | Liên kết phiên rửa (denormalized cho query nhanh) |
| `checked_in_at` | Date | Y | Cashier check-in |
| `verified_by_cashier_id` | ObjectId FK→users | Y | |
| `closed_at` | Date | Y | Đóng đơn (terminal) |
| `closed_by_cashier_id` | ObjectId FK→users | Y | |

### 4.2 `bookings.status` enum mới (P1)

```
pending → confirmed → CHECKED_IN★ → in_progress → QUALITY_CHECK★ → completed → CLOSED★
                                                          ↘
                                                           NEEDS_REWORK★ → in_progress
+ cancelled, no_show (terminal)
+ REFUNDED★ (terminal sau closed)
```

| Status mới | Khi nào |
|---|---|
| `CHECKED_IN` | Cashier verify xe khớp + mở wash_session |
| `QUALITY_CHECK` | Washer complete, chờ cashier kiểm tra |
| `NEEDS_REWORK` | Cashier không hài lòng → trả lại washer |
| `COMPLETED` | Cashier approve QC |
| `CLOSED` | Đã thanh toán + đóng đơn |
| `REFUNDED` | Hoàn tiền sau khi đã CLOSED |

### 4.3 `wash_sessions` (P1, P2)

| Field mới | Type | Nullable | Mục đích |
|---|---|:---:|---|
| `assigned_at` | Date | Y | Lưu thời điểm assign washer (P3) |
| `quality_check_at` | Date | Y | Khi cashier QC |
| `quality_check_by_id` | ObjectId FK→users | Y | |
| `quality_check_result` | String enum {`approved`,`rework`} | Y | |
| `rework_count` | Number | N (default 0) | Số lần bị trả lại |

`status` enum thêm: `QUALITY_CHECK`, `NEEDS_REWORK`.

### 4.4 `wash_steps` — **collection mới** (P1)

| Field | Type | Constraint | Mục đích |
|---|---|---|---|
| `_id` | ObjectId | PK | |
| `wash_session_id` | ObjectId FK→wash_sessions | required, IX | |
| `step` | String enum {`pre_wash`,`exterior`,`interior`,`drying`,`final_check`} | required | |
| `status` | String enum {`pending`,`in_progress`,`done`,`skipped`} | required, default `pending` | |
| `started_at` | Date | optional | |
| `finished_at` | Date | optional | |
| `washer_id` | ObjectId FK→users | required, IX | |
| `note` | String | optional, max 500 | |
| `photo_urls` | String[] | optional | Ảnh trước/sau từng bước |
| `created_at` / `updated_at` | Date | auto | |

**Indexes:** `{wash_session_id: 1, step: 1}` UQ (mỗi step duy nhất 1 record/session).

### 4.5 `orders` (P1)

| Field mới | Type | Nullable | Mục đích |
|---|---|:---:|---|
| `booking_id` | ObjectId FK→bookings | Y, **sparse UQ** | 1 booking ↔ ≤1 order online |
| `wash_session_id` | ObjectId FK→wash_sessions | Y | Cho walk-in (không có booking) |
| `paid_at` | Date | Y | Set khi webhook PayOS xác nhận PAID |

`status` enum thêm: `REFUNDED`.

### 4.6 `vehicle_types` (P2)

| Field mới | Type | Nullable | Mục đích |
|---|---|:---:|---|
| `price_multiplier` | Number | N (default 1.0) | Hệ số giá: motorbike 0.7, sedan 1.0, SUV 1.3, truck 1.5 |

### 4.7 `status_history` — **collection mới** (P2)

Polymorphic audit log cho mọi state transition của booking / wash_session / order.

| Field | Type | Constraint | Mục đích |
|---|---|---|---|
| `_id` | ObjectId | PK | |
| `entity_type` | String enum {`booking`,`wash_session`,`order`} | required, IX | |
| `entity_id` | ObjectId | required, IX | |
| `from_status` | String | required | |
| `to_status` | String | required | |
| `actor_id` | ObjectId FK→users | required, IX | |
| `actor_role` | String enum | required | Snapshot role khi đổi |
| `reason` | String | optional, max 500 | |
| `created_at` | Date | auto | |

**Indexes:** `{entity_type: 1, entity_id: 1, created_at: -1}` compound.

### 4.8 `inspections` (P2)

| Field mới | Type | Nullable | Mục đích |
|---|---|:---:|---|
| `is_required_gate` | Boolean | N (default true cho `before`) | Phải có inspection BEFORE thì washer mới `/start` được |

---

## 5. API còn thiếu (gắn với gap nghiệp vụ)

| Method | Path | Role | Mục đích |
|---|---|---|---|
| `PATCH` | `/admin/bookings/:id/check-in` | cashier+ | Xác nhận xe + khách đến, tự tạo wash_session atomic |
| `PATCH` | `/admin/bookings/:id/switch-service` | cashier+ | Đổi gói tại counter (recalc giá) |
| `PATCH` | `/admin/bookings/:id/switch-vehicle` | cashier+ | Khách đem xe khác đến |
| `PATCH` | `/me/washer/sessions/:id/steps/:step` | washer | Cập nhật bước rửa con |
| `PATCH` | `/admin/wash-sessions/:id/quality-check` | cashier+ | Approve / rework |
| `PATCH` | `/admin/bookings/:id/payment` | cashier+ | Xác nhận thanh toán cash/online |
| `PATCH` | `/admin/bookings/:id/close` | cashier+ | Đóng đơn (terminal) |
| `POST` | `/admin/bookings/:id/refund` | manager+ | Hoàn tiền |

Chi tiết request/response xem trong audit gốc (conversation).

---

## 6. Quan hệ với `origin/main`

| Vấn đề | Hiện trạng branch `feature/wash-sessions` | Hiện trạng `origin/main` | Hành động đề xuất |
|---|---|---|---|
| Mô hình DB | Tách 3 entity: Booking + WashSession + Order (PayOS) | Hợp nhất 1 entity `Order` chứa cả booking + payment | **Giữ thiết kế tách** của branch (rõ ràng hơn, hỗ trợ walk-in). KHÔNG merge thô main vào. |
| Status `CHECKED_IN` | Không có | Có sẵn (`OrderStatusEnum.CHECKED_IN`) | Copy ý tưởng, áp vào `BookingStatusEnum`. |
| Wash session | Có (untracked!) | Không | Branch là nguồn duy nhất → **phải commit ngay**. |
| CI/CD, Swagger, Vercel | Không | Có | Cherry-pick `a844e3b` (CI), `b095ae7` (Swagger), `cace6d4` (Vercel). |
| OTP / Email verify | Không | Có | Tùy chọn — không chặn audit. |

---

## 7. Roadmap fix theo phase

**Phase A — Commit & link payment (1-2 ngày)**
- Commit `src/features/wash-session/` (đang untracked).
- Thêm các field §4.1 vào `bookings`; thêm `booking_id` vào `orders`.
- Update webhook PayOS để khi PAID, set `booking.payment_status = paid` + `booking.paid_at`.

**Phase B — Status flow (1-2 ngày)**
- Mở rộng `BookingStatusEnum` (`CHECKED_IN`, `QUALITY_CHECK`, `COMPLETED`, `CLOSED`, `NEEDS_REWORK`, `REFUNDED`).
- Mở rộng `WashSessionStatusEnum` (`QUALITY_CHECK`, `NEEDS_REWORK`).
- Endpoint `check-in`, `quality-check`, `close`.

**Phase C — Wash steps (2-3 ngày)**
- Tạo collection `wash_steps` + module + 5 step enum.
- Endpoint cập nhật từng bước.
- Update washer UI flow (sau khi có FE).

**Phase D — Audit & pricing (2 ngày)**
- Collection `status_history`.
- `vehicle_types.price_multiplier`.
- Tính `estimated_price` trong booking create.

**Phase E — Polish**
- Inspection gate (`is_required_gate`).
- `REFUNDED` flow.
- Cherry-pick CI/Swagger/Vercel từ main.

---

## 8. Tham chiếu code

- Audit gốc: conversation transcript ngày 2026-05-18.
- Branch hiện tại: `feature/wash-sessions` @ commit `1820341`.
- Working tree untracked: `src/features/wash-session/` (toàn bộ module).
