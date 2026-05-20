# Auto Wash Simulation — Design Proposal

**Status:** 🟡 DRAFT — chờ anh duyệt
**Date:** 2026-05-19
**Scope:** Bổ sung phần "rửa tự động" cho đồ án — mô phỏng phần mềm, không IoT thật.

> **Ý tưởng:** Sau khi cashier nhấn `checked_in → in_progress` (= bật máy), hệ thống tự động đẩy đơn rửa qua **5 công đoạn** theo thời gian định trước, không cần thao tác tay nào nữa. Hoàn tất step cuối → tự set order `completed`. FE hiển thị live progress.

---

## 1. Bảng mới — `wash_steps`

| Field | Type | Constraint | Note |
|---|---|---|---|
| `_id` | ObjectId | PK | |
| `order_id` | ObjectId | required, IX, **FK→orders** | Cha của step |
| `step_name` | String | required, enum | `pre_wash`, `soap`, `rinse`, `wax`, `dry` |
| `step_order` | Number | required, min 1, max 5 | Thứ tự chạy 1→5 |
| `duration_seconds` | Number | required, min 1 | Tổng thời lượng — tính từ `service.estimated_minutes` × % phân bố |
| `status` | String | required, IX, enum, default `pending` | `pending` → `in_progress` → `completed`; có thể `skipped` |
| `started_at` | Date | optional | Khi cron set in_progress |
| `completed_at` | Date | optional | Khi cron set completed |
| `created_at` | Date | auto | |
| `updated_at` | Date | auto | |

**Indexes:**
- `{ order_id: 1, step_order: 1 }` **UQ** — mỗi order có đúng 1 instance / step_order
- `{ status: 1, started_at: 1 }` — cho cron sweep nhanh

**Quan hệ:** `orders 1 ─── N wash_steps` (mỗi order luôn có đúng 5 step records sau khi `in_progress`).

---

## 2. Phân bổ thời lượng từ `service.estimated_minutes`

5 công đoạn chia theo tỉ lệ cố định trong code (không lưu DB):

| Step | % thời lượng | Ví dụ với service 30 phút |
|---|---:|---:|
| 1. `pre_wash` (xịt sơ bụi) | 15% | 4.5 phút |
| 2. `soap` (xà bông) | 25% | 7.5 phút |
| 3. `rinse` (xả nước) | 15% | 4.5 phút |
| 4. `wax` (đánh bóng) | 25% | 7.5 phút |
| 5. `dry` (sấy khô) | 20% | 6 phút |
| **Tổng** | **100%** | **30 phút** |

> Nếu shop sau này muốn cho phép admin chỉnh tỉ lệ → backlog. Hard-code đủ cho MVP.

---

## 3. Luồng auto

```
Cashier: PATCH /admin/orders/:id/status { in_progress }
                          │
                          ▼
   ┌─────────────────────────────────────────┐
   │ Order service hook:                      │
   │  - tạo 5 records wash_steps (pending)   │
   │  - tính duration cho mỗi step           │
   └─────────────────────────────────────────┘
                          │
                          ▼
                 [WashStepCron — mỗi 30s]
                          │
                          ▼
   ┌────────────────────────────────────────────┐
   │ Với mỗi order đang `in_progress`:           │
   │  - Tìm step `in_progress` đang chạy         │
   │       └─ nếu (now - started_at) ≥ duration │
   │             → set step.completed_at = now,  │
   │               step.status = 'completed'    │
   │  - Tìm step `pending` nhỏ nhất kế tiếp     │
   │       → set step.started_at = now,          │
   │         step.status = 'in_progress'         │
   │  - Nếu step 5 đã completed                  │
   │       → set order.status = 'completed'      │
   └────────────────────────────────────────────┘
```

**Idempotent:** cron có thể chạy chồng — chỉ touch step đúng điều kiện thời gian.

**Edge case khi đổi service giữa chừng:** không cho phép (state machine không có path `in_progress → ???`).

---

## 4. Endpoint mới

### 4.1 Customer xem live progress

```
GET /me/orders/:id/wash-steps
Role: JWT (chỉ chủ order)
Response: [
  {
    stepOrder: 1,
    stepName: 'pre_wash',
    status: 'completed',
    startedAt: '2026-05-19T10:00:00Z',
    completedAt: '2026-05-19T10:04:30Z',
    durationSeconds: 270
  },
  ...
  {
    stepOrder: 3,
    stepName: 'rinse',
    status: 'in_progress',
    startedAt: '2026-05-19T10:12:00Z',
    completedAt: null,
    durationSeconds: 270
  },
  {
    stepOrder: 4,
    stepName: 'wax',
    status: 'pending',
    ...
  }
]
```

### 4.2 Admin xem (đã có sẵn data, chỉ cần expose)

```
GET /admin/orders/:id/wash-steps
Role: cashier, manager, admin
```

Cùng response shape.

### 4.3 (Optional) Force advance/skip — chỉ admin

```
POST /admin/orders/:id/wash-steps/:stepOrder/skip
```

Phòng case máy "kẹt" (mô phỏng) — admin force step done sớm. Không bắt buộc cho MVP.

---

## 5. State machine — vẫn giữ nguyên

Order: `pending_payment → confirmed → checked_in → in_progress → completed` (không đổi).

Khác biệt duy nhất: từ `in_progress → completed` giờ **do cron tự làm**, không phải cashier nhấn tay.

> Cashier vẫn có thể `PATCH /admin/orders/:id/status { completed }` để force xong sớm (state machine cho phép). Trường hợp khẩn cấp.

---

## 6. Mở rộng tương lai — `wash_bays` (KHÔNG build MVP)

Nếu sau này muốn mô phỏng nhiều máy rửa song song:

| Field | Type | Note |
|---|---|---|
| `_id` | ObjectId | PK |
| `name` | String, UQ | "Bay 1", "Bay 2" |
| `status` | enum | `idle`, `busy`, `maintenance` |
| `current_order_id` | ObjectId, sparse UQ, FK→orders | Bay đang phục vụ order nào |

Order thêm: `wash_bay_id` (optional FK→wash_bays).

Cần thêm `BayDispatcherCron` để match order ↔ bay rảnh.

→ **Để backlog, scope MVP chỉ cần 1 máy ngầm (= không có bay entity).**

---

## 7. Tổng kết DB diagram update

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
   │ staff_shifts │◀──────▶│    orders     │──── 1:N ───▶ ┌─────────────┐
   └──────────────┘        └───────┬───────┘              │ wash_steps  │  🆕
                                   │                       └─────────────┘
   ┌──────────────┐                │
   │service_types │◀───────────────┤
   └──────────────┘                │
                          ┌────────▼────┐
                          │   users     │ (customer_id)
                          └─────────────┘
                          ┌─────────────┐
                          │  vehicles   │ (vehicle_id)
                          └─────────────┘

   payment_transactions N─1 orders  (đã có)
```

---

## 8. Implementation plan (sau khi anh duyệt)

| Task | Effort | File |
|---|---|---|
| Tạo entity `WashStep` + schema | 15p | `src/features/order/entities/wash-step.entity.ts` |
| Repo `WashStepRepository` | 20p | `src/features/order/repositories/wash-step.repository.ts` |
| Service hook: khi order vào `in_progress` → tạo 5 steps | 20p | `order.service.ts` (sửa `adminUpdateStatus`) |
| Cron `WashStepCron` mỗi 30s advance step | 40p | `src/features/order/jobs/wash-step.cron.ts` |
| Endpoint `GET /me/orders/:id/wash-steps` + admin variant | 30p | `order.controller.ts`, `admin-order.controller.ts` |
| Response DTO `WashStepResponseDto` | 15p | `dto/wash-step-response.dto.ts` |
| Register entity + cron trong `OrderModule` | 5p | `order.module.ts` |
| Update [BOOKING_FLOW.md](./BOOKING_FLOW.md) | 15p | docs |
| Update [DB_SCHEMA_AS_BUILT.md](./DB_SCHEMA_AS_BUILT.md) | 10p | docs |

**Tổng:** ~3 tiếng. Không phải đụng PayOS, không phải đụng auth, không phải đụng existing endpoints.

---

## 9. Test plan

1. Tạo cash order → `confirmed` → `checked_in` → `in_progress`.
2. Verify 5 wash_steps được tạo, status=`pending`, đúng `step_order`+`duration_seconds`.
3. Đợi 30s → step 1 chuyển `in_progress`, `started_at` set.
4. Đợi qua `duration_seconds` của step 1 → step 1 = `completed`, step 2 = `in_progress`.
5. Lặp cho đến step 5 → order tự set `completed`.
6. `GET /me/orders/:id/wash-steps` ở mỗi giai đoạn → trả đúng snapshot.
7. Negative: gọi GET với JWT khác chủ → 404.

**Demo trên FE:** progress bar 5 đoạn, đoạn đang chạy animated, đếm ngược thời gian còn lại của step.

---

## 10. Câu hỏi chờ anh quyết

1. **Tỉ lệ thời lượng các step** (15/25/15/25/20) — OK chưa? Hay anh muốn khác?
2. **Có cần endpoint admin force-skip step không?** (§4.3) — em đang đề xuất skip, có thể thêm sau.
3. **Cron interval 30s** — OK hay muốn 10s/60s? 30s là cân bằng giữa độ mượt demo và load DB.
4. **Có lưu `progress_percent` denormalized trên Order không?** — Em đang đề xuất không (FE tự tính từ steps). Nếu cần realtime push qua WebSocket sau này thì lưu sẽ tiện hơn.

---

**Sau khi anh duyệt → em sẽ:**
- Code theo plan §8
- Update DB_SCHEMA_AS_BUILT.md thêm §15 `wash_steps`
- Update BOOKING_FLOW.md thêm §3.7 Auto wash progression
