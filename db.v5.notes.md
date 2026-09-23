# Ghi chú DB v5

Tài liệu đi kèm `db.v5.md`. `db.v5.md` chỉ chứa schema DBML (import được vào dbdiagram.io); mọi giải thích, ràng buộc liên bảng, SQL không biểu diễn được bằng DBML và hướng dẫn implement nằm ở đây. Yêu cầu nghiệp vụ gốc: `detail.v4.md`. API: `api.design.md`.

## 1. Thay đổi so với v4

| # | Thay đổi | Lý do |
|---|---|---|
| 1 | Thêm `refresh_tokens` | Phiên đăng nhập dùng refresh token ngẫu nhiên lưu hash, xoay vòng, thu hồi được (BR_G.2 trong detail.v4). Thay cho refresh JWT stateless |
| 2 | Bỏ `payment_gateway_transactions`, `gateway_provider`, `gateway_status`; thêm `wallet_top_ups`, `bank_transactions`, `top_up_status`, `bank_transaction_status` | Chuyển cổng nạp ví từ VNPay/Momo sang SePay (chuyển khoản VietQR + webhook biến động số dư) |
| 3 | `wallet_transactions`: bỏ `gateway_transaction_id`, thêm `bank_transaction_id` (unique), `idempotency_key` (unique) và `top_up_method` | Một giao dịch ngân hàng chỉ cộng ví một lần; mọi sự kiện ví (nạp quầy, thanh toán, hoàn từng component) có khóa chống lặp riêng, thay cho `uq_wtx_gateway_credit` và `uq_wtx_order_payment`. `top_up_method` ghi phương thức nạp (CASH/CARD/TRANSFER) để đối soát tiền thu tại quầy |
| 4 | `orders.idempotency_key` (unique, not null) | Checkout và auto-renew gửi lại không tạo order thứ hai |
| 5 | Tiền: `numeric(12,2)` → `numeric(12,0)` | Tiền VND nguyên đồng |
| 6 | `order_type` → `order_item_type` | Enum dùng cho `order_items.type` và `coupons.applicable_types` |
| 7 | `coach_specializations`: bỏ unique cứng `(coach_id, sport_id)`, thêm partial unique theo PENDING/APPROVED, thêm `review_note` | Unique cứng chặn đăng ký lại sau REJECTED (BR_1.12) |
| 8 | `member_evaluations`: unique cứng → partial unique `WHERE deleted_at IS NULL` | Soft-delete rồi tạo lại đánh giá (BR_4.5) |
| 9 | `membership_orders`: thêm 4 cột quyền lợi của kỳ | Kỳ đã trả giữ quyền lợi lúc mua (BR_1.8, chốt D07) |
| 10 | `facility_bookings.benefit` (`booking_benefit`) | Đếm quota slot miễn phí theo tháng và phân biệt booking gym miễn phí (chốt D04) |
| 11 | `notifications`: `is_read` → `read_at`; thêm `dedup_key` (unique); thêm type `SUPPORT`; FK account đổi `cascade` → `restrict` | Nhắc lịch/cron chỉ gửi một lần; account không bao giờ bị xóa |
| 12 | `support_requests.category` (`support_request_category`) | Ticket có loại (UC_3.16) |
| 13 | `class_attendance.note` | Ghi chú điểm danh |
| 14 | `facility_maintenances.reason` not null | Lịch bảo trì bắt buộc có lý do (UC_2.4) |
| 15 | `center_checkins`: bỏ `check_out_time` | Không có yêu cầu check-out |
| 16 | `system_settings`: thêm `top_up_min_amount`, `top_up_expiry_minutes` | Cấu hình nạp ví qua SePay |
| 17 | Cú pháp: bỏ `>?`, chuyển CHECK từ note sang khối `checks` | `>?` không phải DBML hợp lệ; nullable FK suy ra từ cột |
| 18 | Ref 1–1 viết theo chiều `bảng_được_tham_chiếu.id - bảng_con.fk` | Chiều ngược (như v4) khiến exporter đặt FK sai phía (ví dụ `accounts.id` tham chiếu `member_profile`) |

Tổng: 35 bảng (32 bảng khi chưa triển khai F4: `center_checkins`, `class_attendance`, `member_evaluations`).

## 2. SQL không biểu diễn được bằng DBML

DBML không hỗ trợ partial index, exclusion constraint và function. Các đối tượng dưới đây phải thêm bằng custom migration (Prisma: `prisma migrate dev --create-only` rồi sửa SQL). `Note` trong `db.v5.md` chỉ ghi tên và điều kiện để đối chiếu.

```sql
CREATE UNIQUE INDEX uq_sports_name ON sports(name) WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX uq_facilities_name ON facilities(name) WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX uq_coupons_code ON coupons(code) WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX uq_coach_spec_active ON coach_specializations(coach_id, sport_id)
  WHERE status IN ('PENDING', 'APPROVED');
CREATE UNIQUE INDEX uq_ccr_pending ON class_coach_registrations(class_id, coach_id)
  WHERE status = 'PENDING';
CREATE UNIQUE INDEX uq_enrollment_active ON class_enrollments(class_id, account_id)
  WHERE status = 'ENROLLED';
CREATE UNIQUE INDEX uq_mm_one_active ON member_memberships(account_id)
  WHERE status = 'ACTIVE';
CREATE UNIQUE INDEX uq_order_one_membership ON order_items(order_id)
  WHERE type = 'MEMBERSHIP';
CREATE UNIQUE INDEX uq_booking_single_item ON facility_bookings(order_item_id)
  WHERE package_id IS NULL;
CREATE UNIQUE INDEX uq_eval_session_account ON member_evaluations(session_id, account_id)
  WHERE deleted_at IS NULL;

CREATE EXTENSION IF NOT EXISTS btree_gist;
ALTER TABLE facility_maintenances ADD CONSTRAINT ex_maintenance_overlap
  EXCLUDE USING gist (facility_id WITH =, tstzrange(start_at, end_at, '[)') WITH &&)
  WHERE (deleted_at IS NULL);
ALTER TABLE class_sessions ADD CONSTRAINT ex_session_facility_overlap
  EXCLUDE USING gist (facility_id WITH =,
    tsrange(session_date + start_time, session_date + end_time, '[)') WITH &&)
  WHERE (status = 'SCHEDULED');
ALTER TABLE membership_orders ADD CONSTRAINT ex_membership_period_overlap
  EXCLUDE USING gist (membership_id WITH =, daterange(period_start, period_end, '[)') WITH &&);

CREATE FUNCTION valid_days_of_week(v integer[]) RETURNS boolean
LANGUAGE sql IMMUTABLE STRICT AS $$
  SELECT CASE WHEN array_ndims(v) IS DISTINCT FROM 1 THEN false ELSE
    COALESCE(cardinality(v) BETWEEN 1 AND 7
      AND array_position(v, NULL) IS NULL
      AND v <@ ARRAY[0,1,2,3,4,5,6]
      AND cardinality(v) = (SELECT count(DISTINCT d) FROM unnest(v) AS x(d)), false) END
$$;
```

- Function `valid_days_of_week` phải tạo trước constraint `ck_fp_days`.
- Không thêm exclusion cho `facility_bookings` theo facility/giờ: sẽ chặn multi-booking hợp lệ khi `capacity_per_slot > 1`.
- Exclusion của session chỉ chặn trùng facility giữa các session; không bảo vệ lịch member/Coach và không chặn chéo với booking/maintenance.
- Không dùng `now()`/today trong partial index.
- `tsrange` của session là giờ địa phương theo timezone trung tâm (`Asia/Ho_Chi_Minh`).

## 3. Invariant liên bảng (service hoặc trigger đảm bảo)

CHECK chỉ kiểm tra được dữ liệu trong cùng row. Các invariant dưới đây phải được service đảm bảo trong cùng transaction; nếu cần DB tự enforce thì dùng deferred constraint trigger (custom migration).

**Account/profile**
- Mỗi account có đúng một profile đúng role; tạo account và profile cùng transaction, kể cả khi mọi field tùy chọn trống. Manager đầu tiên seed cùng `manager_profile`.
- Không cập nhật `accounts.role` riêng lẻ. Ví chỉ thuộc `member_profile`; ledger/top-up/bank transaction tham chiếu `member_profile.account_id`.
- Không ghi `health_notes` vào audit; `staff_notes` chỉ Manager sửa.

**Order / order item**
- Order có ≥1 item; mọi cột tiền của order = SUM cột tương ứng của items (kể cả `refunded_amount`).
- Mỗi item có đúng một đích nghiệp vụ theo type: MEMBERSHIP → một `membership_orders`; COURSE_ENROLLMENT → một `class_enrollments`; FACILITY_PACKAGE → một `facility_packages` (booking con dùng chung `order_item_id`); FACILITY_BOOKING → một booking lẻ (`package_id IS NULL`).
- Guest order chỉ có FACILITY_BOOKING items; `created_by` là Receptionist.
- Sau commit: `orders` chỉ đổi `status`, `refunded_amount`, `updated_at`; `order_items` chỉ đổi `refunded_amount`, `updated_at`; không INSERT item vào order đã commit, không DELETE. Enforce bằng trigger BEFORE UPDATE/DELETE so sánh `to_jsonb(NEW)` với `OLD` trừ các key được phép.
- `wallet_transactions`, `membership_orders`, `bank_transactions.raw_payload` append-only.

**Lịch**
- `classes.start_date/end_date` = MIN/MAX `session_date` của session SCHEDULED (kể cả buổi đã qua); recompute trong mọi đường ghi session (sinh, dời, hủy, hủy lớp, maintenance).
- `classes.coach_id` là nguồn duy nhất về Coach hiện tại; registration là lịch sử xét duyệt.
- Facility của session phải hỗ trợ `courses.sport_id` qua `facility_sports`.
- Một session SCHEDULED khóa toàn bộ slot facility với booking lẻ; `capacity_per_slot` là số booking, không phải sĩ số lớp.

**Ví**
- `member_profile.wallet_balance` = `balance_after` của ledger mới nhất của account.
- PAYMENT: tối đa một ledger mỗi order, `amount = orders.total_amount` (order tổng 0 không tạo ledger).
- REFUND: `order_item_id` thuộc `order_id`, account là chủ order, `amount` bằng phần tăng `order_items.refunded_amount`.

## 4. Khóa chống lặp (idempotency)

`wallet_transactions.idempotency_key` và `orders.idempotency_key` là unique, not null. Quy ước giá trị:

| Sự kiện | Khóa |
|---|---|
| Checkout của member | `checkout:{accountId}:{clientKey}` |
| Checkout tại quầy | `counter:{receptionistId}:{clientKey}` |
| Auto-renew | `renew:{membershipId}:{periodStart}` |
| PAYMENT ví | `payment:{orderId}` |
| Nạp ví qua SePay | `sepay:{bankTransactionId}` |
| Nạp ví tại quầy | `cash:{receptionistId}:{clientKey}` |
| Hoàn booking | `refund:booking:{bookingId}` |
| Hoàn một buổi học của một enrollment | `refund:session:{sessionId}:{enrollmentId}` |
| Hoàn khi hủy enrollment/lớp | `refund:enrollment:{enrollmentId}` |

- Request lặp với cùng khóa: trả lại kết quả đã lưu nếu payload khớp, trả 409 nếu payload khác.
- Hủy lớp đang học hoàn allocation các buổi chưa diễn ra **trừ các buổi đã có** `refund:session:*` của cùng enrollment, nên hủy riêng buổi (D05) rồi hủy cả lớp không hoàn trùng.

## 5. SePay

**Tạo yêu cầu nạp**
- `payment_code` = `SCTU` + 8 ký tự `[A-Z0-9]` ngẫu nhiên (không dùng ký tự dễ nhầm `0/O`, `1/I`). Cấu hình tiền tố `SCTU` trong SePay (Công ty → Cấu hình chung → Cấu trúc mã thanh toán) để SePay tự tách vào trường `code`.
- `expires_at = now() + system_settings.top_up_expiry_minutes`. Hết hạn chỉ ẩn QR; tiền đến muộn vẫn được khớp.
- QR: `https://qr.sepay.vn/img?acc={SEPAY_BANK_ACCOUNT}&bank={SEPAY_BANK_CODE}&amount={amount}&des={payment_code}`.

**Webhook** (`POST /api/v1/payments/sepay/webhook`)
1. Xác thực header `Authorization: Apikey {SEPAY_WEBHOOK_API_KEY}` (so sánh timing-safe). Sai → 401.
2. Bỏ qua `transferType != 'in'` (trả `{"success": true}`).
3. INSERT `bank_transactions` với `sepay_id = payload.id`; trùng unique → đã xử lý, trả `{"success": true}`.
4. Lấy mã từ `payload.code`, fallback regex `SCTU[A-Z0-9]{8}` trên `content`. Tìm `wallet_top_ups` theo `payment_code`.
5. Khớp khi: tìm thấy, status ≠ SUCCESS, `transferAmount = amount`. Khi khớp, trong cùng transaction: lock account → member_profile → top-up; set top-up SUCCESS + `bank_transaction_id` + `completed_at`; credit ví; insert TOP_UP ledger (`bank_transaction_id`, `top_up_method = TRANSFER`, khóa `sepay:{id}`); `bank_transactions.status = MATCHED`; insert notification.
6. Không khớp (không có mã, mã không tồn tại, sai số tiền, top-up đã SUCCESS) → `status = UNMATCHED`, notify Manager.
7. Luôn trả HTTP 200 `{"success": true}` trong < 30 giây sau khi đã commit. Lỗi hệ thống → 5xx để SePay retry (tối đa 7 lần trong 5 giờ); nhờ `sepay_id` unique nên retry an toàn.

**Đối soát** (Manager)
- UNMATCHED → RESOLVED: chọn member, credit đúng `amount` vào ví (ledger TOP_UP có `bank_transaction_id`, `top_up_method = TRANSFER`, khóa `sepay:{id}`), ghi `resolved_account_id`, `handled_by`, `handled_at`, `note`, audit.
- UNMATCHED → IGNORED: tiền không thuộc hệ thống (hoàn trả ngoài hệ thống nếu cần); bắt buộc `note`, audit.
- Một `bank_transactions` chỉ credit tối đa một lần (unique `wallet_transactions.bank_transaction_id`).

**Giới hạn gói Free**: 50 giao dịch tiền vào/tháng (vượt tính phí theo lượt), 11 ngân hàng, tài khoản cá nhân hoặc doanh nghiệp. SePay không có API hoàn tiền: mọi khoản hoàn đi về ví, không hoàn về ngân hàng.

## 6. Concurrency protocol

- Isolation READ COMMITTED; mọi kiểm tra điều kiện thực hiện **sau** khi có lock.
- Mọi mutation lịch/capacity lấy trước `SELECT pg_advisory_xact_lock(74001, 1)`: booking/package, enrollment/hủy, tạo/dời/hủy class/session, assign/rút Coach, maintenance, đổi giờ/slot/capacity, gỡ `facility_sports`, vô hiệu hóa sport/facility/Coach. Chỉ dùng advisory lock cấp transaction (tương thích Supabase transaction pooler).
- Thứ tự row lock: `system_settings` (SHARE; UPDATE khi sửa) → `accounts` theo UUID tăng dần → `member_profile` → `classes` → `facilities` → `member_memberships` → `coupons` → `orders` → `order_items` → `wallet_top_ups` → `bank_transactions`. Nhiều ID cùng nhóm sort trước khi lock; bỏ qua nhóm không dùng, không quay ngược.
- Luồng chỉ liên quan tiền (nạp ví, webhook) không lấy advisory lock lịch.
- Không gọi mạng/gửi email trong transaction. Notification insert trong transaction; email gửi sau commit.
- Deadlock / serialization failure: retry toàn bộ transaction, không retry riêng bước debit.

**Booking**: lock member (nếu có) + facility; kiểm tra booking CONFIRMED cùng slot so với `capacity_per_slot`, session SCHEDULED (kể cả lớp DRAFT/PENDING_APPROVAL), maintenance chưa xóa. Package: bất kỳ slot nào có booking/session/maintenance → từ chối cả series.

**Enrollment**: lock member + class; lớp OPEN, chưa đến `start_date`, đếm ENROLLED < `max_students`, không trùng lịch member với booking CONFIRMED và session của enrollment khác.

**Coach**: assign/sửa session lock account Coach; kiểm tra mọi session của các lớp có `coach_id` đó.

**Coupon**: lock coupon rồi kiểm tra hạn, `is_active`, `deleted_at`, type, minimum, `COUNT(orders WHERE coupon_id)` (kể cả REFUNDED) và theo `account_id`.

**Ví**: lock `accounts` rồi `member_profile` trước khi đọc số dư; update số dư + insert ledger cùng transaction.

## 7. Nghiệp vụ tính toán

**Membership**
- Hiệu lực `[start_date, end_date)`: có quyền lợi khi ACTIVE và `start_date <= today < end_date`.
- Quyền lợi dùng để tính giá lấy từ `membership_orders` có `period_start <= today < period_end` của membership ACTIVE (không đọc `memberships` hiện tại).
- Gia hạn còn hiệu lực: thêm kỳ `[end_date, end_date + duration_days)`, nối `end_date`. Hết hạn/hủy: membership mới từ today.
- Cron: tìm ACTIVE có `end_date <= today`; lock rồi đọc lại. `auto_renew` + package còn bán + đủ ví → order riêng (khóa `renew:*`) + kỳ mới từ `end_date` cũ; ngược lại → EXPIRED + notify. Không truy thu nhiều kỳ.
- Package ngừng bán (`is_active = false` hoặc soft-delete): không auto-renew, notify member.

**Quota slot miễn phí**
- Tháng lịch theo `booking_date` (timezone trung tâm).
- Đã dùng = COUNT `facility_bookings` CONFIRMED có `benefit = 'FREE_SLOT'` của account trong tháng. Booking bị hủy (member hủy đúng hạn, bảo trì) không còn CONFIRMED nên tự trả quota.
- Quota tháng = `free_booking_slots_per_month` của kỳ hiệu lực tại ngày booking.
- Booking gym miễn phí nhờ `gym_access` dùng `benefit = 'GYM_ACCESS'`, không tính quota.

**Giá và phân bổ**
- Thứ tự: giá gốc → quyền lợi membership (theo từng booking/dòng) → coupon (một lần cấp order, phân bổ về dòng).
- Làm tròn đến đồng, 0,5 làm tròn lên. Phân bổ: floor từng phần, phát từng đồng dư theo phần lẻ giảm dần, tie-break `line_number` (dòng) hoặc lịch rồi ID (component).
- `item_snapshot` của package lưu booking IDs + `allocated_amount` từng booking; của enrollment lưu session IDs + `allocated_amount` từng buổi (chia đều). Tổng allocation = `order_items.total_amount`.
- Dời buổi/đổi giá không đổi allocation.

## 8. Prisma / migration

- `prisma-client` generator, adapter `@prisma/adapter-pg`, `moduleFormat = "esm"`.
- `DATABASE_URL`: Supabase transaction pooler (port 6543) cho runtime; `DIRECT_URL`: session pooler (port 5432) cho `prisma migrate`.
- Tiền `numeric(12,0)` map `Decimal` trong Prisma; không tính tiền bằng JS `number` float.
- Partial unique/exclusion/function/trigger: custom SQL trong migration; kiểm tra lại predicate sau mỗi `migrate diff`. Không thay partial unique bằng `@@unique`.
- `findUnique`/`upsert` không dùng được partial key: query theo status + xử lý lỗi unique violation.
- `SELECT ... FOR UPDATE` và advisory lock dùng raw SQL tham số hóa trên cùng transaction client.
- Soft-delete filter (client extension) chỉ áp cho danh mục đang hoạt động; báo cáo, hóa đơn, audit, kiểm tra xung đột lịch không lọc mù.
- Schema Prisma hiện tại (`users`, `refresh_tokens`, `otp_codes`) là bản tối giản của F1; migration sang v5 là migration mới (DB chưa có dữ liệu thật): đổi `users` → `accounts` + 4 profile, giữ `refresh_tokens` và `otp_codes`.
