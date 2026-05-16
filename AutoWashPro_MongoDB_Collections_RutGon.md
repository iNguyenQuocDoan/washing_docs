# AutoWash Pro — MongoDB Collection Design Rút Gọn

**Phiên bản:** 1.1  
**Mục tiêu:** Liệt kê collections và trường dữ liệu để vẽ MongoDB diagram / ERD-style.  
**Phạm vi:** Advance Booking, Wash Operation, Inspection, Payment, Loyalty, Promotion.  
**Quyết định:** Dùng `roles` + `users`, không dùng `user_roles`; hạn chế nhúng object nghiệp vụ quan trọng.

**Changelog:**
- **1.1** — Thêm collection `service_pricing` (#22) để tách ma trận giá theo loại xe khỏi `service_types`. `service_types.base_price` xuống vai trò giá đề xuất. Thêm BR-29 → BR-32. Cập nhật flow §5.2 và §5.3.
- **1.0** — Bản gốc, 21 collections.

---

## 1. Quy ước kiểu dữ liệu

| Type | Ý nghĩa |
|---|---|
| `ObjectId` | ID document hoặc khóa tham chiếu collection khác |
| `String` | Chuỗi |
| `Number` | Số |
| `Decimal128` | Tiền, giá, số tiền giảm |
| `Boolean` | Đúng / sai |
| `Date` | Ngày giờ |

## 2. Danh sách collections

| # | Collection | Mục đích nghiệp vụ |
|---|---|---|
| 1 | `roles` | Lưu vai trò chính của người dùng. |
| 2 | `users` | Lưu tài khoản customer, cashier, washer, manager, admin. Bản rút gọn chỉ dùng 1 role chính qua role_id. |
| 3 | `vehicle_types` | Lưu loại xe để phân loại xe và hỗ trợ cấu hình dịch vụ/giá. |
| 4 | `vehicles` | Lưu xe của customer. |
| 5 | `service_types` | Lưu gói dịch vụ rửa xe và thông tin tính điểm. |
| 6 | `staff_shifts` | Lưu ca làm việc và capacity đặt lịch. |
| 7 | `bookings` | Lưu lịch đặt trước của customer. |
| 8 | `wash_sessions` | Lưu phiên rửa thực tế, là collection trung tâm của vận hành. |
| 9 | `inspections` | Lưu kiểm tra xe trước/sau khi rửa. |
| 10 | `inspection_photos` | Lưu ảnh inspection, DB chỉ lưu URL và metadata. |
| 11 | `payments` | Lưu thanh toán của wash session. Transfer/VNPAY để field phẳng, không nhúng object. |
| 12 | `cash_sessions` | Lưu ca thu ngân và đối soát tiền mặt. |
| 13 | `loyalty_accounts` | Lưu điểm và tier hiện tại của customer. |
| 14 | `tier_configs` | Lưu cấu hình tier. |
| 15 | `tier_histories` | Lưu lịch sử upgrade/downgrade/manual tier. |
| 16 | `point_transactions` | Lưu lịch sử cộng/trừ điểm, nên immutable. |
| 17 | `point_ledgers` | Lưu từng lô điểm để trừ theo FIFO và xử lý expire. |
| 18 | `promotions` | Lưu chương trình khuyến mãi/reward. |
| 19 | `promotion_services` | Mapping promotion với service áp dụng. |
| 20 | `promotion_valid_days` | Lưu ngày trong tuần promotion được áp dụng. |
| 21 | `promotion_usages` | Lưu lịch sử sử dụng promotion. |
| 22 | `service_pricing` | Ma trận giá theo cặp (dịch vụ, loại xe). Là nguồn giá chính khi đặt lịch và check-in. |

---
## 3. Chi tiết collections

## 3.1 `roles`

**Mục đích:** Lưu vai trò chính của người dùng.

| Field | Type | Required | Reference | Mô tả |
|---|---:|:---:|---|---|
| `_id` | `ObjectId` | Yes | `—` | ID role |
| `code` | `String` | Yes | `—` | customer | cashier | washer | manager | admin |
| `name` | `String` | Yes | `—` | Tên hiển thị |
| `description` | `String` | No | `—` | Mô tả role |
| `is_active` | `Boolean` | Yes | `—` | Role còn hoạt động |
| `created_at` | `Date` | Yes | `—` | Ngày tạo |
| `updated_at` | `Date` | Yes | `—` | Ngày cập nhật |

**Enum / Giá trị hợp lệ:**

```txt
code = customer | cashier | washer | manager | admin
```

**Index đề xuất:**

```js
{ code: 1 } unique
```

---

## 3.2 `users`

**Mục đích:** Lưu tài khoản customer, cashier, washer, manager, admin. Bản rút gọn chỉ dùng 1 role chính qua role_id.

| Field | Type | Required | Reference | Mô tả |
|---|---:|:---:|---|---|
| `_id` | `ObjectId` | Yes | `—` | ID user |
| `role_id` | `ObjectId` | Yes | `roles._id` | Role chính |
| `name` | `String` | Yes | `—` | Họ tên |
| `phone` | `String` | Yes | `—` | Số điện thoại |
| `email` | `String` | Yes | `—` | Email |
| `password_hash` | `String` | Yes | `—` | Mật khẩu đã hash |
| `avatar_url` | `String` | No | `—` | Ảnh đại diện |
| `date_of_birth` | `Date` | No | `—` | Ngày sinh |
| `is_active` | `Boolean` | Yes | `—` | Tài khoản hoạt động |
| `delete_requested_at` | `Date` | No | `—` | Ngày yêu cầu xóa account |
| `created_at` | `Date` | Yes | `—` | Ngày tạo |
| `updated_at` | `Date` | Yes | `—` | Ngày cập nhật |

**Index đề xuất:**

```js
{ phone: 1 } unique; { email: 1 } unique; { role_id: 1 }
```

---

## 3.3 `vehicle_types`

**Mục đích:** Lưu loại xe để phân loại xe và hỗ trợ cấu hình dịch vụ/giá.

| Field | Type | Required | Reference | Mô tả |
|---|---:|:---:|---|---|
| `_id` | `ObjectId` | Yes | `—` | ID loại xe |
| `name` | `String` | Yes | `—` | Motorbike, Car, SUV... |
| `description` | `String` | No | `—` | Mô tả |
| `is_active` | `Boolean` | Yes | `—` | Còn dùng |
| `created_at` | `Date` | Yes | `—` | Ngày tạo |
| `updated_at` | `Date` | Yes | `—` | Ngày cập nhật |

**Index đề xuất:**

```js
{ name: 1 } unique
```

---

## 3.4 `vehicles`

**Mục đích:** Lưu xe của customer.

| Field | Type | Required | Reference | Mô tả |
|---|---:|:---:|---|---|
| `_id` | `ObjectId` | Yes | `—` | ID xe |
| `customer_id` | `ObjectId` | Yes | `users._id` | Chủ xe |
| `vehicle_type_id` | `ObjectId` | Yes | `vehicle_types._id` | Loại xe |
| `license_plate` | `String` | Yes | `—` | Biển số |
| `nickname` | `String` | No | `—` | Tên gợi nhớ |
| `brand` | `String` | No | `—` | Hãng xe |
| `model` | `String` | No | `—` | Dòng xe |
| `color` | `String` | No | `—` | Màu xe |
| `is_default` | `Boolean` | Yes | `—` | Xe mặc định |
| `is_active` | `Boolean` | Yes | `—` | Xe còn dùng |
| `created_at` | `Date` | Yes | `—` | Ngày tạo |
| `updated_at` | `Date` | Yes | `—` | Ngày cập nhật |

**Index đề xuất:**

```js
{ license_plate: 1 } unique; { customer_id: 1, is_active: 1 }
```

---

## 3.5 `service_types`

**Mục đích:** Lưu gói dịch vụ rửa xe và thông tin tính điểm.

> **Lưu ý về giá:** `base_price` ở đây chỉ là **giá đề xuất/gợi ý** dùng khi seed ma trận giá ban đầu. Giá thực tế khi đặt lịch và check-in lấy từ collection `service_pricing` (xem §3.22) — vì giá rửa xe khác nhau theo loại xe (xe máy / sedan / SUV).

| Field | Type | Required | Reference | Mô tả |
|---|---:|:---:|---|---|
| `_id` | `ObjectId` | Yes | `—` | ID dịch vụ |
| `name` | `String` | Yes | `—` | Tên dịch vụ |
| `description` | `String` | No | `—` | Mô tả |
| `base_price` | `Decimal128` | Yes | `—` | Giá đề xuất (fallback, không phải nguồn truth khi booking) |
| `estimated_minutes` | `Number` | Yes | `—` | Thời lượng dự kiến |
| `points_multiplier` | `Number` | Yes | `—` | Hệ số điểm |
| `is_active` | `Boolean` | Yes | `—` | Còn bán |
| `created_at` | `Date` | Yes | `—` | Ngày tạo |
| `updated_at` | `Date` | Yes | `—` | Ngày cập nhật |

**Index đề xuất:**

```js
{ name: 1 } unique; { is_active: 1 }
```

---

## 3.6 `staff_shifts`

**Mục đích:** Lưu ca làm việc và capacity đặt lịch.

| Field | Type | Required | Reference | Mô tả |
|---|---:|:---:|---|---|
| `_id` | `ObjectId` | Yes | `—` | ID ca |
| `staff_id` | `ObjectId` | Yes | `users._id` | Nhân viên |
| `shift_type` | `String` | Yes | `—` | cashier | washer |
| `station_name` | `String` | No | `—` | Tên khoang/khu vực rửa |
| `start_at` | `Date` | Yes | `—` | Bắt đầu ca |
| `end_at` | `Date` | Yes | `—` | Kết thúc ca |
| `status` | `String` | Yes | `—` | scheduled | active | completed | cancelled |
| `max_bookings` | `Number` | Yes | `—` | Sức chứa booking |
| `current_bookings` | `Number` | Yes | `—` | Booking hiện tại |
| `note` | `String` | No | `—` | Ghi chú |
| `created_at` | `Date` | Yes | `—` | Ngày tạo |
| `updated_at` | `Date` | Yes | `—` | Ngày cập nhật |

**Enum / Giá trị hợp lệ:**

```txt
shift_type = cashier | washer; status = scheduled | active | completed | cancelled
```

**Index đề xuất:**

```js
{ staff_id: 1, start_at: -1 }; { shift_type: 1, status: 1 }
```

---

## 3.7 `bookings`

**Mục đích:** Lưu lịch đặt trước của customer.

| Field | Type | Required | Reference | Mô tả |
|---|---:|:---:|---|---|
| `_id` | `ObjectId` | Yes | `—` | ID booking |
| `customer_id` | `ObjectId` | Yes | `users._id` | Khách đặt |
| `vehicle_id` | `ObjectId` | Yes | `vehicles._id` | Xe đặt lịch |
| `service_type_id` | `ObjectId` | Yes | `service_types._id` | Dịch vụ |
| `staff_shift_id` | `ObjectId` | Yes | `staff_shifts._id` | Ca phục vụ |
| `scheduled_at` | `Date` | Yes | `—` | Thời gian hẹn |
| `status` | `String` | Yes | `—` | pending | confirmed | in_progress | done | cancelled | no_show |
| `priority_level` | `Number` | Yes | `—` | Ưu tiên theo tier |
| `reschedule_count` | `Number` | Yes | `—` | Số lần đổi lịch |
| `cancel_reason` | `String` | No | `—` | Lý do hủy |
| `created_at` | `Date` | Yes | `—` | Ngày tạo |
| `updated_at` | `Date` | Yes | `—` | Ngày cập nhật |

**Enum / Giá trị hợp lệ:**

```txt
status = pending | confirmed | in_progress | done | cancelled | no_show
```

**Index đề xuất:**

```js
{ customer_id: 1, scheduled_at: -1 }; { scheduled_at: 1, status: 1 }; { staff_shift_id: 1 }
```

---

## 3.8 `wash_sessions`

**Mục đích:** Lưu phiên rửa thực tế, là collection trung tâm của vận hành.

| Field | Type | Required | Reference | Mô tả |
|---|---:|:---:|---|---|
| `_id` | `ObjectId` | Yes | `—` | ID phiên rửa |
| `booking_id` | `ObjectId` | No | `bookings._id` | Nullable nếu walk-in |
| `customer_id` | `ObjectId` | Yes | `users._id` | Khách hàng |
| `vehicle_id` | `ObjectId` | Yes | `vehicles._id` | Xe được rửa |
| `service_type_id` | `ObjectId` | Yes | `service_types._id` | Dịch vụ |
| `cashier_id` | `ObjectId` | Yes | `users._id` | Cashier xử lý quầy |
| `washer_id` | `ObjectId` | No | `users._id` | Washer được gán |
| `service_price_snapshot` | `Decimal128` | Yes | `—` | Giá chốt tại thời điểm rửa |
| `status` | `String` | Yes | `—` | waiting | assigned | in_progress | done | cancelled |
| `started_at` | `Date` | No | `—` | Bắt đầu rửa |
| `completed_at` | `Date` | No | `—` | Hoàn thành rửa |
| `note` | `String` | No | `—` | Ghi chú |
| `created_at` | `Date` | Yes | `—` | Ngày tạo |
| `updated_at` | `Date` | Yes | `—` | Ngày cập nhật |

**Enum / Giá trị hợp lệ:**

```txt
status = waiting | assigned | in_progress | done | cancelled
```

**Index đề xuất:**

```js
{ booking_id: 1 }; { vehicle_id: 1, created_at: -1 }; { cashier_id: 1, created_at: -1 }; { washer_id: 1, started_at: -1 }
```

---

## 3.9 `inspections`

**Mục đích:** Lưu kiểm tra xe trước/sau khi rửa.

| Field | Type | Required | Reference | Mô tả |
|---|---:|:---:|---|---|
| `_id` | `ObjectId` | Yes | `—` | ID inspection |
| `wash_session_id` | `ObjectId` | Yes | `wash_sessions._id` | Phiên rửa |
| `inspector_id` | `ObjectId` | Yes | `users._id` | Cashier/Manager kiểm tra |
| `phase` | `String` | Yes | `—` | before | after |
| `damage_notes` | `String` | No | `—` | Ghi chú hư hỏng |
| `customer_acknowledged` | `Boolean` | Yes | `—` | Khách đã xác nhận |
| `customer_signature_url` | `String` | No | `—` | URL chữ ký |
| `created_at` | `Date` | Yes | `—` | Ngày tạo |
| `updated_at` | `Date` | Yes | `—` | Ngày cập nhật |

**Enum / Giá trị hợp lệ:**

```txt
phase = before | after
```

**Index đề xuất:**

```js
{ wash_session_id: 1, phase: 1 } unique; { inspector_id: 1, created_at: -1 }
```

---

## 3.10 `inspection_photos`

**Mục đích:** Lưu ảnh inspection, DB chỉ lưu URL và metadata.

| Field | Type | Required | Reference | Mô tả |
|---|---:|:---:|---|---|
| `_id` | `ObjectId` | Yes | `—` | ID ảnh |
| `inspection_id` | `ObjectId` | Yes | `inspections._id` | Inspection liên quan |
| `photo_url` | `String` | Yes | `—` | URL ảnh |
| `mime` | `String` | Yes | `—` | image/jpeg | image/png | image/webp |
| `size` | `Number` | Yes | `—` | Dung lượng byte |
| `uploaded_at` | `Date` | Yes | `—` | Thời điểm upload |

**Enum / Giá trị hợp lệ:**

```txt
mime = image/jpeg | image/png | image/webp
```

**Index đề xuất:**

```js
{ inspection_id: 1 }
```

---

## 3.11 `payments`

**Mục đích:** Lưu thanh toán của wash session. Transfer/VNPAY để field phẳng, không nhúng object.

| Field | Type | Required | Reference | Mô tả |
|---|---:|:---:|---|---|
| `_id` | `ObjectId` | Yes | `—` | ID payment |
| `wash_session_id` | `ObjectId` | Yes | `wash_sessions._id` | Phiên rửa |
| `cash_session_id` | `ObjectId` | No | `cash_sessions._id` | Ca thu ngân |
| `received_by` | `ObjectId` | No | `users._id` | Cashier nhận/xác nhận tiền |
| `promotion_usage_id` | `ObjectId` | No | `promotion_usages._id` | Khuyến mãi áp dụng |
| `method` | `String` | Yes | `—` | cash | transfer | vnpay |
| `status` | `String` | Yes | `—` | pending | paid | failed | cancelled |
| `base_price` | `Decimal128` | Yes | `—` | Giá gốc |
| `discount_amount` | `Decimal128` | Yes | `—` | Tiền giảm |
| `final_price` | `Decimal128` | Yes | `—` | Tiền phải trả |
| `bank_reference` | `String` | No | `—` | Mã chuyển khoản thủ công |
| `proof_image_url` | `String` | No | `—` | Ảnh bằng chứng chuyển khoản |
| `transfer_verified_by` | `ObjectId` | No | `users._id` | Người verify transfer |
| `transfer_verified_at` | `Date` | No | `—` | Thời điểm verify |
| `vnp_txn_ref` | `String` | No | `—` | Mã giao dịch VNPAY nội bộ |
| `vnp_transaction_no` | `String` | No | `—` | Mã giao dịch VNPAY |
| `vnp_response_code` | `String` | No | `—` | Response code VNPAY |
| `vnp_bank_code` | `String` | No | `—` | Mã ngân hàng VNPAY |
| `vnp_pay_date` | `Date` | No | `—` | Ngày thanh toán VNPAY |
| `paid_at` | `Date` | No | `—` | Thời điểm paid |
| `created_at` | `Date` | Yes | `—` | Ngày tạo |
| `updated_at` | `Date` | Yes | `—` | Ngày cập nhật |

**Enum / Giá trị hợp lệ:**

```txt
method = cash | transfer | vnpay; status = pending | paid | failed | cancelled
```

**Index đề xuất:**

```js
{ wash_session_id: 1 } unique; { cash_session_id: 1 }; { status: 1, paid_at: -1 }; { vnp_transaction_no: 1 } sparse unique
```

---

## 3.12 `cash_sessions`

**Mục đích:** Lưu ca thu ngân và đối soát tiền mặt.

| Field | Type | Required | Reference | Mô tả |
|---|---:|:---:|---|---|
| `_id` | `ObjectId` | Yes | `—` | ID ca thu ngân |
| `cashier_id` | `ObjectId` | Yes | `users._id` | Cashier mở ca |
| `closed_by_cashier_id` | `ObjectId` | No | `users._id` | Người đóng ca |
| `opened_at` | `Date` | Yes | `—` | Mở ca |
| `closed_at` | `Date` | No | `—` | Đóng ca |
| `status` | `String` | Yes | `—` | open | closed | review_required |
| `opening_balance` | `Decimal128` | Yes | `—` | Tiền đầu ca |
| `expected_cash_total` | `Decimal128` | Yes | `—` | Cash hệ thống tính |
| `expected_transfer_total` | `Decimal128` | Yes | `—` | Transfer hệ thống ghi nhận |
| `actual_cash_total` | `Decimal128` | No | `—` | Cash thực tế |
| `variance` | `Decimal128` | No | `—` | Chênh lệch |
| `note` | `String` | No | `—` | Ghi chú |

**Enum / Giá trị hợp lệ:**

```txt
status = open | closed | review_required
```

**Index đề xuất:**

```js
{ cashier_id: 1, opened_at: -1 }; { status: 1 }
```

---

## 3.13 `loyalty_accounts`

**Mục đích:** Lưu điểm và tier hiện tại của customer.

| Field | Type | Required | Reference | Mô tả |
|---|---:|:---:|---|---|
| `_id` | `ObjectId` | Yes | `—` | ID loyalty account |
| `customer_id` | `ObjectId` | Yes | `users._id` | Customer |
| `tier_config_id` | `ObjectId` | Yes | `tier_configs._id` | Tier hiện tại |
| `points_balance` | `Number` | Yes | `—` | Điểm hiện còn |
| `visits_this_month` | `Number` | Yes | `—` | Lượt tháng này |
| `visits_last_month` | `Number` | Yes | `—` | Lượt tháng trước |
| `consecutive_low_months` | `Number` | Yes | `—` | Số tháng liên tiếp dưới ngưỡng |
| `tier_reviewed_at` | `Date` | No | `—` | Lần review tier gần nhất |
| `points_expire_at` | `Date` | No | `—` | Ngày hết hạn điểm gần nhất |
| `updated_at` | `Date` | Yes | `—` | Ngày cập nhật |

**Index đề xuất:**

```js
{ customer_id: 1 } unique; { tier_config_id: 1 }
```

---

## 3.14 `tier_configs`

**Mục đích:** Lưu cấu hình tier.

| Field | Type | Required | Reference | Mô tả |
|---|---:|:---:|---|---|
| `_id` | `ObjectId` | Yes | `—` | ID tier |
| `tier_name` | `String` | Yes | `—` | Member | Silver | Gold | Platinum |
| `min_visits_per_month` | `Number` | Yes | `—` | Lượt tối thiểu/tháng |
| `booking_window_days` | `Number` | Yes | `—` | Số ngày đặt trước |
| `priority_level` | `Number` | Yes | `—` | Độ ưu tiên |
| `points_per_wash` | `Number` | Yes | `—` | Điểm cơ bản mỗi lượt |
| `is_active` | `Boolean` | Yes | `—` | Còn dùng |
| `created_at` | `Date` | Yes | `—` | Ngày tạo |
| `updated_at` | `Date` | Yes | `—` | Ngày cập nhật |

**Enum / Giá trị hợp lệ:**

```txt
tier_name = Member | Silver | Gold | Platinum
```

**Index đề xuất:**

```js
{ tier_name: 1 } unique; { priority_level: 1 } unique
```

---

## 3.15 `tier_histories`

**Mục đích:** Lưu lịch sử upgrade/downgrade/manual tier.

| Field | Type | Required | Reference | Mô tả |
|---|---:|:---:|---|---|
| `_id` | `ObjectId` | Yes | `—` | ID history |
| `customer_id` | `ObjectId` | Yes | `users._id` | Customer |
| `old_tier_config_id` | `ObjectId` | No | `tier_configs._id` | Tier cũ |
| `new_tier_config_id` | `ObjectId` | Yes | `tier_configs._id` | Tier mới |
| `reason` | `String` | Yes | `—` | upgrade | downgrade | manual | initial |
| `triggered_by` | `ObjectId` | No | `users._id` | Admin/system thực hiện |
| `created_at` | `Date` | Yes | `—` | Ngày tạo |

**Enum / Giá trị hợp lệ:**

```txt
reason = upgrade | downgrade | manual | initial
```

**Index đề xuất:**

```js
{ customer_id: 1, created_at: -1 }
```

---

## 3.16 `point_transactions`

**Mục đích:** Lưu lịch sử cộng/trừ điểm, nên immutable.

| Field | Type | Required | Reference | Mô tả |
|---|---:|:---:|---|---|
| `_id` | `ObjectId` | Yes | `—` | ID transaction |
| `customer_id` | `ObjectId` | Yes | `users._id` | Customer |
| `wash_session_id` | `ObjectId` | No | `wash_sessions._id` | Phiên rửa liên quan |
| `promotion_usage_id` | `ObjectId` | No | `promotion_usages._id` | Promotion usage liên quan |
| `type` | `String` | Yes | `—` | earn | redeem | expire | manual_adjust | downgrade_penalty | welcome_bonus | birthday_bonus | referral_bonus |
| `points` | `Number` | Yes | `—` | Điểm thay đổi, có thể âm |
| `balance_after` | `Number` | Yes | `—` | Số dư sau giao dịch |
| `description` | `String` | No | `—` | Diễn giải |
| `created_at` | `Date` | Yes | `—` | Ngày tạo |

**Enum / Giá trị hợp lệ:**

```txt
type = earn | redeem | expire | manual_adjust | downgrade_penalty | welcome_bonus | birthday_bonus | referral_bonus
```

**Index đề xuất:**

```js
{ customer_id: 1, created_at: -1 }; { wash_session_id: 1 }; { promotion_usage_id: 1 }
```

---

## 3.17 `point_ledgers`

**Mục đích:** Lưu từng lô điểm để trừ theo FIFO và xử lý expire.

| Field | Type | Required | Reference | Mô tả |
|---|---:|:---:|---|---|
| `_id` | `ObjectId` | Yes | `—` | ID ledger |
| `customer_id` | `ObjectId` | Yes | `users._id` | Customer |
| `source_transaction_id` | `ObjectId` | Yes | `point_transactions._id` | Transaction tạo lô |
| `points_initial` | `Number` | Yes | `—` | Điểm ban đầu |
| `points_remaining` | `Number` | Yes | `—` | Điểm còn lại |
| `expire_at` | `Date` | Yes | `—` | Ngày hết hạn |
| `status` | `String` | Yes | `—` | active | expired | consumed |
| `created_at` | `Date` | Yes | `—` | Ngày tạo |

**Enum / Giá trị hợp lệ:**

```txt
status = active | expired | consumed
```

**Index đề xuất:**

```js
{ customer_id: 1, status: 1, expire_at: 1 }; { source_transaction_id: 1 }
```

---

## 3.18 `promotions`

**Mục đích:** Lưu chương trình khuyến mãi/reward.

| Field | Type | Required | Reference | Mô tả |
|---|---:|:---:|---|---|
| `_id` | `ObjectId` | Yes | `—` | ID promotion |
| `created_by` | `ObjectId` | Yes | `users._id` | Admin tạo |
| `min_tier_config_id` | `ObjectId` | No | `tier_configs._id` | Tier tối thiểu |
| `title` | `String` | Yes | `—` | Tên promotion |
| `description` | `String` | No | `—` | Mô tả |
| `code` | `String` | No | `—` | Mã khuyến mãi |
| `discount_type` | `String` | Yes | `—` | percent | fixed_amount | free_wash | addon |
| `discount_value` | `Decimal128` | Yes | `—` | Giá trị giảm |
| `min_order_amount` | `Decimal128` | No | `—` | Đơn tối thiểu |
| `max_uses_per_customer` | `Number` | No | `—` | Giới hạn mỗi khách |
| `max_total_uses` | `Number` | No | `—` | Giới hạn toàn hệ thống |
| `current_total_uses` | `Number` | Yes | `—` | Số lượt đã dùng |
| `eligibility` | `String` | Yes | `—` | any | booking_only | walk_in_only |
| `start_at` | `Date` | Yes | `—` | Bắt đầu |
| `end_at` | `Date` | Yes | `—` | Kết thúc |
| `is_active` | `Boolean` | Yes | `—` | Còn active |
| `created_at` | `Date` | Yes | `—` | Ngày tạo |
| `updated_at` | `Date` | Yes | `—` | Ngày cập nhật |

**Enum / Giá trị hợp lệ:**

```txt
discount_type = percent | fixed_amount | free_wash | addon; eligibility = any | booking_only | walk_in_only
```

**Index đề xuất:**

```js
{ code: 1 } sparse unique; { is_active: 1, start_at: 1, end_at: 1 }; { eligibility: 1 }
```

---

## 3.19 `promotion_services`

**Mục đích:** Mapping promotion với service áp dụng.

| Field | Type | Required | Reference | Mô tả |
|---|---:|:---:|---|---|
| `_id` | `ObjectId` | Yes | `—` | ID mapping |
| `promotion_id` | `ObjectId` | Yes | `promotions._id` | Promotion |
| `service_type_id` | `ObjectId` | Yes | `service_types._id` | Service |
| `created_at` | `Date` | Yes | `—` | Ngày tạo |

**Index đề xuất:**

```js
{ promotion_id: 1, service_type_id: 1 } unique; { service_type_id: 1 }
```

---

## 3.20 `promotion_valid_days`

**Mục đích:** Lưu ngày trong tuần promotion được áp dụng.

| Field | Type | Required | Reference | Mô tả |
|---|---:|:---:|---|---|
| `_id` | `ObjectId` | Yes | `—` | ID valid day |
| `promotion_id` | `ObjectId` | Yes | `promotions._id` | Promotion |
| `day_of_week` | `Number` | Yes | `—` | 1=Monday ... 7=Sunday |
| `created_at` | `Date` | Yes | `—` | Ngày tạo |

**Enum / Giá trị hợp lệ:**

```txt
day_of_week = 1 | 2 | 3 | 4 | 5 | 6 | 7
```

**Index đề xuất:**

```js
{ promotion_id: 1, day_of_week: 1 } unique
```

---

## 3.21 `promotion_usages`

**Mục đích:** Lưu lịch sử sử dụng promotion.

| Field | Type | Required | Reference | Mô tả |
|---|---:|:---:|---|---|
| `_id` | `ObjectId` | Yes | `—` | ID usage |
| `promotion_id` | `ObjectId` | Yes | `promotions._id` | Promotion |
| `customer_id` | `ObjectId` | Yes | `users._id` | Customer |
| `wash_session_id` | `ObjectId` | Yes | `wash_sessions._id` | Phiên rửa |
| `used_at` | `Date` | No | `—` | Thời điểm dùng |
| `discount_amount` | `Decimal128` | Yes | `—` | Số tiền giảm |
| `status` | `String` | Yes | `—` | reserved | used | cancelled |
| `created_at` | `Date` | Yes | `—` | Ngày tạo |
| `updated_at` | `Date` | Yes | `—` | Ngày cập nhật |

**Enum / Giá trị hợp lệ:**

```txt
status = reserved | used | cancelled
```

**Index đề xuất:**

```js
{ promotion_id: 1, customer_id: 1 }; { wash_session_id: 1 } unique
```

---

## 3.22 `service_pricing`

**Mục đích:** Ma trận giá theo cặp (dịch vụ × loại xe). Là nguồn giá chính khi customer xem giá lúc đặt lịch và khi cashier check-in. Tách khỏi `service_types` vì cùng 1 dịch vụ "Rửa cơ bản" có giá khác nhau cho xe máy / sedan / SUV.

| Field | Type | Required | Reference | Mô tả |
|---|---:|:---:|---|---|
| `_id` | `ObjectId` | Yes | `—` | ID pricing row |
| `service_type_id` | `ObjectId` | Yes | `service_types._id` | Dịch vụ |
| `vehicle_type_id` | `ObjectId` | Yes | `vehicle_types._id` | Loại xe |
| `price` | `Decimal128` | Yes | `—` | Giá thực tế cho cặp |
| `is_active` | `Boolean` | Yes | `—` | Pricing còn hiệu lực |
| `created_at` | `Date` | Yes | `—` | Ngày tạo |
| `updated_at` | `Date` | Yes | `—` | Ngày cập nhật |

**Index đề xuất:**

```js
{ service_type_id: 1, vehicle_type_id: 1 } unique; { vehicle_type_id: 1, is_active: 1 }
```

**Quy tắc:**

- Mỗi cặp `(service_type_id, vehicle_type_id)` chỉ có **1 row** (unique compound, áp dụng cho cả `is_active=false`).
- Update chỉ cho đổi `price` và `is_active`, KHÔNG cho đổi cặp ref (xóa mềm + tạo mới nếu cần).
- Soft delete dùng `is_active=false`, không xóa cứng.
- Khi `service_types` hoặc `vehicle_types` bị set inactive, các row `service_pricing` liên quan **giữ nguyên** (orphan-tolerant). Booking layer filter `is_active=true` ở cả 3 collection để ẩn pair khỏi flow đặt lịch.

---

## 4. Quan hệ chính để vẽ diagram

```txt
roles 1 ─── N users

users 1 ─── N vehicles
vehicle_types 1 ─── N vehicles

service_types 1 ─── N service_pricing
vehicle_types 1 ─── N service_pricing

users 1 ─── N bookings
vehicles 1 ─── N bookings
service_types 1 ─── N bookings
staff_shifts 1 ─── N bookings

bookings 1 ─── 0..1 wash_sessions

users 1 ─── N wash_sessions        // cashier_id
users 1 ─── N wash_sessions        // washer_id
vehicles 1 ─── N wash_sessions
service_types 1 ─── N wash_sessions

wash_sessions 1 ─── N inspections
inspections 1 ─── N inspection_photos

wash_sessions 1 ─── 1 payments
cash_sessions 1 ─── N payments

users 1 ─── 1 loyalty_accounts
tier_configs 1 ─── N loyalty_accounts

users 1 ─── N tier_histories
tier_configs 1 ─── N tier_histories

users 1 ─── N point_transactions
wash_sessions 1 ─── N point_transactions
promotion_usages 1 ─── N point_transactions

point_transactions 1 ─── N point_ledgers

tier_configs 1 ─── N promotions
promotions 1 ─── N promotion_services
service_types 1 ─── N promotion_services

promotions 1 ─── N promotion_valid_days

promotions 1 ─── N promotion_usages
users 1 ─── N promotion_usages
wash_sessions 1 ─── 0..1 promotion_usages
```

## 5. Luồng nghiệp vụ chính

### 5.1 Customer đăng ký và thêm xe

- Customer tạo tài khoản trong `users`.
- Hệ thống gán `role_id` tương ứng role `customer`.
- Customer thêm xe vào `vehicles`, liên kết với `vehicle_types`.
- Hệ thống tạo `loyalty_accounts` mặc định ở tier Member.

### 5.2 Đặt lịch trước

- Customer chọn xe, dịch vụ và thời gian.
- Hệ thống lookup giá thực tế từ `service_pricing` theo cặp `(service_type_id, vehicle_type_id)`. Nếu không có row active cho cặp đó → từ chối booking (dịch vụ chưa hỗ trợ loại xe này).
- Hệ thống kiểm tra tier từ `loyalty_accounts` + `tier_configs`.
- Hệ thống kiểm tra booking window, số booking chưa hoàn thành và capacity của `staff_shifts`.
- Tạo `bookings` với `priority_level` theo tier.

### 5.3 Check-in và tạo wash session

- Cashier tìm booking theo phone, biển số hoặc booking ID.
- Nếu có booking thì confirm; nếu walk-in thì tạo `wash_sessions.booking_id = null`. Walk-in cũng lookup giá từ `service_pricing` theo cặp `(service_type_id, vehicle_type_id)` của xe được rửa.
- Chốt `wash_sessions.service_price_snapshot` = giá lấy từ `service_pricing.price` tại thời điểm check-in (snapshot, không bị ảnh hưởng nếu admin sửa pricing sau đó).
- Cashier tạo inspection before và ảnh trong `inspection_photos`.
- Cashier verify promotion và gán washer.

### 5.4 Washer xử lý rửa xe

- Washer nhận session được gán.
- Cập nhật `wash_sessions.status = in_progress` khi bắt đầu.
- Cập nhật `wash_sessions.status = done` và `completed_at` khi hoàn thành.

### 5.5 Inspection sau và thanh toán

- Cashier tạo inspection after và ảnh after.
- Nếu damage khác before thì Manager xử lý dispute.
- Tạo/cập nhật `payments`.
- Cash gắn với `cash_sessions`; transfer có bank/proof fields; VNPAY optional theo PRD v3.

### 5.6 Cộng điểm loyalty

- Điều kiện: `wash_sessions.status = done` và `payments.status = paid`.
- Tính điểm theo `tier_configs.points_per_wash * service_types.points_multiplier`.
- Tạo `point_transactions` type `earn`.
- Tạo `point_ledgers` cho lô điểm mới có `expire_at`.
- Cập nhật `loyalty_accounts.points_balance` và `visits_this_month`.

### 5.7 Dùng promotion / redeem

- Kiểm tra promotion active, thời gian, tier, service, valid day, quota, eligibility.
- Tạo `promotion_usages` nếu hợp lệ.
- Payment áp dụng `discount_amount`.
- Nếu có redeem điểm thì tạo `point_transactions` type `redeem` và trừ `point_ledgers` theo FIFO.

### 5.8 Review tier hàng tháng

- Cron đọc `loyalty_accounts.visits_this_month`.
- So sánh với `tier_configs.min_visits_per_month`.
- Upgrade/downgrade nếu đủ điều kiện.
- Ghi `tier_histories` và cập nhật `loyalty_accounts`.

### 5.9 Đóng ca thu ngân

- Cashier mở `cash_sessions` đầu ca.
- Cash payment trong ca gắn `cash_session_id`.
- Cuối ca nhập `actual_cash_total`.
- Hệ thống tính `variance` và chuyển `review_required` nếu lệch vượt ngưỡng.

## 6. Business Rules

| # | Rule |
|---|---|
| BR-01 | `users.phone` là duy nhất. |
| BR-02 | `users.email` là duy nhất. |
| BR-03 | Bản rút gọn dùng một role chính: `users.role_id`. |
| BR-04 | `vehicles.license_plate` là duy nhất. |
| BR-05 | Customer phải có ít nhất một xe để đặt lịch. |
| BR-06 | Booking phải nằm trong booking window của tier hiện tại. |
| BR-07 | Chỉ tạo booking khi `current_bookings < max_bookings`. |
| BR-08 | Priority queue sắp theo `priority_level` giảm dần, sau đó `created_at` tăng dần. |
| BR-09 | Reschedule tối đa 2 lần. |
| BR-10 | Booking quá giờ theo rule chuyển thành `no_show`. |
| BR-11 | Walk-in session có `booking_id = null`. |
| BR-12 | Cashier/Manager mới được check-in, inspection, verify promotion, thu tiền. |
| BR-13 | Washer chỉ được cập nhật nhận xe, bắt đầu rửa, hoàn thành. |
| BR-14 | Mỗi wash session bắt buộc có inspection before và after. |
| BR-15 | `inspections.customer_acknowledged = true` mới được close/giao xe. |
| BR-16 | Mỗi inspection cần tối thiểu 3 ảnh. |
| BR-17 | Mỗi wash session có tối đa một payment. |
| BR-18 | Payment `paid` là điều kiện cộng điểm. |
| BR-19 | Cash payment phải gắn cash session đang open. |
| BR-20 | Một cashier chỉ nên có một cash session open. |
| BR-21 | Lượt hợp lệ tính tier = wash session done + payment paid. |
| BR-22 | Mỗi lần earn điểm tạo `point_transactions` và `point_ledgers`. |
| BR-23 | Điểm trừ theo FIFO từ `point_ledgers.expire_at` cũ nhất. |
| BR-24 | `point_transactions` và `tier_histories` nên immutable. |
| BR-25 | Mỗi wash session chỉ dùng tối đa một promotion. |
| BR-26 | Promotion phải đúng tier, thời gian, service, valid day, quota và eligibility. |
| BR-27 | `promotion_usages.wash_session_id` unique. |
| BR-28 | Manual tier adjust phải ghi `tier_histories`. |
| BR-29 | Mỗi cặp `(service_type_id, vehicle_type_id)` chỉ có duy nhất 1 row trong `service_pricing`. |
| BR-30 | Booking và walk-in check-in phải có row `service_pricing` active cho cặp `(service_type, vehicle_type)`; thiếu → từ chối tạo. |
| BR-31 | `wash_sessions.service_price_snapshot` chốt từ `service_pricing.price` tại thời điểm check-in, không cập nhật lại nếu pricing thay đổi sau đó. |
| BR-32 | `service_types.base_price` chỉ là giá đề xuất; nguồn truth khi giao dịch là `service_pricing`. |

## 7. Collections không đưa vào sơ đồ rút gọn

| Collection | Lý do |
|---|---|
| `user_roles` | Người dùng yêu cầu chỉ dùng `roles` + `users`. |
| `wash_stations` | Rút gọn bằng `staff_shifts.station_name`; có thể thêm lại nếu cần quản lý khoang rửa độc lập. |
| `transfer_proofs` | Gộp field phẳng vào `payments`. |
| `payment_adjustments` | Nghiệp vụ ngoại lệ/refund/void, để phase sau. |
| `promotion_rules` | Rule chính đã thể hiện qua `promotions`, `promotion_services`, `promotion_valid_days`. |
| `notifications` | Chức năng phụ, không phải core data model. |
| `audit_logs` | Log hệ thống, có thể vẽ riêng ở sơ đồ support. |
| `system_configs` | Cấu hình global, không cần trong core diagram. |

## 8. Gợi ý bố cục vẽ

```txt
[User & Vehicle]
roles → users → vehicles → vehicle_types

[Catalog & Pricing]
service_types ─┐
               ├→ service_pricing
vehicle_types ─┘

[Booking & Operation]
users → bookings → wash_sessions
vehicles → bookings
service_types → bookings
staff_shifts → bookings
wash_sessions → inspections → inspection_photos

[Payment]
wash_sessions → payments
cash_sessions → payments

[Loyalty]
users → loyalty_accounts → tier_configs
users → point_transactions → point_ledgers
tier_configs → tier_histories

[Promotion]
promotions → promotion_services → service_types
promotions → promotion_valid_days
promotions → promotion_usages → wash_sessions
```

## 9. Kết luận

Bản thiết kế này dùng **22 collections** để vẽ MongoDB collection diagram rút gọn nhưng vẫn giữ đủ nghiệp vụ chính: **Customer → Vehicle → Booking → Wash Session → Inspection → Payment → Loyalty → Promotion**, với ma trận giá theo loại xe qua `service_pricing` (thêm so với phiên bản 21-collection ban đầu để phản ánh đúng nghiệp vụ giá rửa xe khác nhau cho từng loại xe).
