# Phân tích đề bài: Hệ thống Quản lý Trung tâm Thể thao

## 1. Tổng quan hệ thống

**Tên dự án**: Sports Center Management System
**Tech stack**: Monorepo Turborepo + Bun. Backend Express.js (TypeScript) + Prisma ORM + PostgreSQL (Supabase). Frontend React (Vite, TailwindCSS, Ant Design, TanStack Router/Query). Triển khai trên Vercel. Email qua Resend. Nạp ví qua SePay (chuyển khoản VietQR).
**Scope**: 4 Flows (F1, F2, F3 bắt buộc + F4 tùy chọn)
**Ngôn ngữ hiển thị**: Tiếng Việt
**Platform**: Responsive Web App (không mobile app, không IoT)
**Tài liệu liên quan**: `design.v2.md` (thiết kế sơ bộ), `api.design.md` (API), `db.v5.md` (thiết kế DB)

**Out of scope**: Personal Training (thuê HLV cá nhân), rút tiền từ ví, hoàn tiền về tài khoản ngân hàng, thanh toán gateway trực tiếp cho từng đơn hàng, thanh toán thẻ quốc tế, màn hình/bảng lịch sử điểm danh riêng.

---

## 2. Bối cảnh kinh doanh (Business Context)

### 2.1 Mô hình trung tâm

Trung tâm thể thao **đa bộ môn**, hỗ trợ nhiều loại hình:
- **Phòng tập gym** (thể hình, fitness)
- **Phòng tập nhóm** (yoga, boxing, zumba, dance...)
- **Sân thể thao** (cầu lông, bóng rổ, futsal, tennis...)
- Bộ môn **không cố định**, lưu trong database, Manager có thể CRUD (xem BR_1.13 khi sửa/xóa bộ môn đang có lớp)

### 2.2 Cơ sở vật chất (Facility)

Tất cả phòng/sân được gộp chung dưới khái niệm **Facility**:
- Mỗi facility có: tên, loại (gym/court/room/field), giới hạn số booking đồng thời (`capacity_per_slot`), giá mỗi slot, trạng thái hoạt động
- **Facility ↔ Sport là quan hệ nhiều-nhiều**: phòng đa năng dùng được cho yoga, zumba, dance
- Facility được quản lý theo **slot thời gian cố định** (thời lượng slot do Manager cấu hình)
- Mỗi slot có giới hạn số booking đồng thời (capacity), không phải sức chứa học viên của lớp:
  - Phòng gym: slot có thể chứa 20 người → hiển thị 14/20
  - Sân cầu lông: capacity = 1 (1 booking chiếm toàn bộ sân)
  - Sân futsal: capacity = 1
- **Lớp học cũng chiếm slot của facility** → slot có buổi học bị khóa hoàn toàn cho booking lẻ (BR_2.3)
- **Bảo trì facility theo lịch** (from → to), Manager đặt; trong khoảng đó facility không nhận booking (BR_2.19)
- Giờ hoạt động cố định (ví dụ 6:00–22:00), Manager cấu hình được

### 2.3 Các loại sản phẩm/dịch vụ

| # | Loại | Mô tả | Ví dụ |
|---|------|-------|-------|
| 1 | **Membership Package** | Gói thành viên theo thời hạn, có quyền lợi cố định (BR_1.8) | Gói Tháng 500k, Gói Năm 5M |
| 2 | **Facility Booking (lẻ)** | Đặt sân/phòng theo slot cụ thể (bao gồm vào gym 1 lần) | Sân CL 1, 18:00–19:00, Thứ 3 |
| 3 | **Facility Package (định kỳ)** | Gói đặt sân định kỳ hàng tuần | Sân CL mỗi T3–T5 18:00, 1 tháng |
| 4 | **Course Enrollment** | Đăng ký khóa học (lớp có HLV) | Boxing cơ bản, 12 buổi |

Danh mục gói hội viên là `memberships`; gói member đã mua là `member_memberships`; mỗi kỳ mua/gia hạn là một `membership_orders`.

### 2.4 Mô hình thanh toán

- **Ví (Wallet)**: Member nạp tiền vào ví bằng chuyển khoản ngân hàng qua mã VietQR (SePay xác nhận tự động) hoặc nạp tại quầy. Mua dịch vụ online → trừ ví. **Mua online chỉ thanh toán bằng ví** (không thanh toán gateway trực tiếp cho từng đơn)
- **Không rút tiền từ ví**. Ví chỉ có nạp / trả / hoàn. Mọi khoản hoàn đều về ví
- **Thanh toán tại quầy**: Lễ tân ghi nhận thanh toán (tiền mặt/chuyển khoản/thẻ), chỉ áp dụng khi khách đến trực tiếp. Member trả tại quầy nếu được hoàn thì hoàn **về ví**, không trả tiền mặt
- **Guest booking**: Chỉ thanh toán tại quầy, lễ tân nhập thông tin khách (tên, SĐT) + đặt sân + thu tiền. **Guest không được hoàn tiền và không dùng coupon**
- **Hóa đơn = Order**. `orders` là đầu hóa đơn, `order_number` là số hóa đơn; `order_items` là các dòng dịch vụ. Một order có thể gồm đặt sân + đăng ký lớp + mua membership, cùng người mua và một phương thức thanh toán. Không có bảng hóa đơn riêng.
- Mỗi `order_items.type` là MEMBERSHIP / FACILITY_BOOKING / FACILITY_PACKAGE / COURSE_ENROLLMENT. Một item là một booking lẻ, một package, một enrollment hoặc một kỳ membership; không có quantity. Package có nhiều booking con cùng item và chỉ tính tiền một lần.
- Mỗi order có ít nhất một item, tối đa một MEMBERSHIP item; tổng từng cột tiền của order bằng tổng cột tương ứng ở items. Toàn bộ đơn phải hợp lệ và thanh toán atomic, không bán một phần khi một dòng thất bại.
- Quyền lợi dùng để tính giá lấy từ membership đã có hiệu lực trước checkout; membership mới mua trong cùng order chưa giảm giá cho các dòng còn lại của order đó.
- Mọi giao dịch đều ghi lịch sử. Hủy dịch vụ không đồng nghĩa với hoàn tiền; order chỉ đổi trạng thái hoàn tiền khi có khoản thực sự được hoàn. Chính sách hoàn chỉ áp dụng cho item/component của dịch vụ bị hủy, không tự hoàn toàn bộ đơn nhiều loại.
- Tiền tính theo VND nguyên đồng. Làm tròn khoản giảm đến đồng gần nhất (0,5 làm tròn lên), tổng giảm không vượt giá gốc. Quy tắc phân bổ phần dư ở BR_3.12.

#### 2.4.1 Định nghĩa Order và Order Item

**Order** là đơn thanh toán/hóa đơn của một người mua. **Order Item** là một dòng dịch vụ cụ thể trong đơn, gồm loại dịch vụ, đối tượng được chọn, lịch/kỳ mua và số tiền của dòng. Một order đã thanh toán có **1..N order items**; mỗi item thuộc đúng một order.

| Item type | Một item đại diện cho | Thông tin chọn trước thanh toán | Ví dụ |
|---|---|---|---|
| `FACILITY_BOOKING` | Một lần đặt một slot facility | Facility, ngày, giờ/slot | Sân CL 1, ngày 20/10, 18:00–19:00 |
| `FACILITY_PACKAGE` | Một gói đặt facility định kỳ, sinh nhiều booking con | Facility, ngày bắt đầu, các thứ trong tuần, slot, số tuần | Sân CL 1 mỗi T3/T5 trong 4 tuần; một item, 8 booking con |
| `COURSE_ENROLLMENT` | Một lần đăng ký một class cụ thể của course | Class; học viên là member mua đơn | Đăng ký Boxing cơ bản — lớp A |
| `MEMBERSHIP` | Một kỳ mua mới/gia hạn membership | Membership package; kỳ được hệ thống xác định | Mua một kỳ gói 30 ngày |

Ví dụ: một booking sân và một đăng ký lớp được chọn cùng nhau tạo **một order, hai order items**. Hai booking lẻ khác slot là hai item. Booking con của package không tạo thêm dòng tính tiền.

**Đơn đang soạn (giỏ dịch vụ)** là danh sách lựa chọn tạm trước thanh toán, được phép thêm/sửa/xóa và có thể rỗng. Giỏ chỉ tồn tại phía người dùng (trình duyệt của member hoặc lễ tân), không lưu trên server, không phải record `orders`/`order_items`, chưa phát sinh hóa đơn, chưa giữ chỗ, chưa trừ ví hoặc tiêu coupon. Server tính giá và kiểm tra giỏ mỗi khi người dùng xem lại; order và dịch vụ chỉ được tạo khi checkout thành công, trạng thái order là PAID.

### 2.5 Hệ thống khuyến mãi (Coupon)

Chỉ có **Coupon Code** — mã giảm giá nhập khi thanh toán.

| Thuộc tính | Mô tả |
|------------|-------|
| `code` | Mã duy nhất, ví dụ `WELCOME20` |
| `discount_type` | `PERCENT` hoặc `FIXED` |
| `discount_value` | 20 (%) hoặc 50000 (đ) |
| `max_discount` | Trần giảm cho `PERCENT` |
| `valid_from`, `valid_to` | Thời gian hiệu lực |
| `max_uses` | Tổng số lần dùng tối đa |
| `max_uses_per_user` | Số lần 1 member dùng |
| `min_order_amount` | Tổng giá gốc tối thiểu của các item thuộc loại coupon áp dụng; item không hợp lệ không giúp đạt ngưỡng |
| `applicable_types` | Các loại dịch vụ được áp coupon; để trống = tất cả loại dịch vụ |

Rule: **1 mã / 1 order, không cộng dồn, không áp cho nạp ví, chỉ dành cho member có tài khoản** (BR_3.5). Mã coupon được trim và chuẩn hóa chữ hoa; tính duy nhất không phân biệt hoa thường trong tập coupon chưa xóa.

### 2.6 Khách hàng không bắt buộc có membership

- Khách **không cần mua gói thành viên** để đặt sân hoặc đăng ký khóa học
- Không có membership hiệu lực = không có quyền lợi (không có "gói basic")
- Cần tài khoản để sử dụng hệ thống online
- **Guest (khách vãng lai)** không cần tài khoản: chỉ áp dụng cho booking facility lẻ tại quầy, lễ tân thao tác

---

## 3. Actors & Roles

| Actor | Role | Quyền hạn cố định (hardcode) |
|-------|------|-----|
| **Center Manager** | `MANAGER` | Quản trị toàn bộ hệ thống, tạo/quản lý user, facility, khóa học, lớp, phân công HLV, gói, coupon, báo cáo, cấu hình hệ thống, lịch bảo trì, đối soát giao dịch ngân hàng |
| **Coach** | `COACH` | Đăng ký bộ môn dạy, đăng ký dạy lớp, điểm danh, tạo session notes, đánh giá học viên |
| **Member** | `MEMBER` | Đăng ký tài khoản, mua gói, đặt sân online, đăng ký khóa học, xem lịch, nạp ví, tạo yêu cầu hỗ trợ |
| **Receptionist** | `RECEPTIONIST` | Check-in, đặt sân tại quầy (cho member + guest), ghi nhận thanh toán, nạp ví tại quầy, xuất hóa đơn, xử lý yêu cầu hỗ trợ |

### Account & Profile Model

**`accounts` lưu thông tin chung; mỗi account có đúng một profile phù hợp role.**

| Bảng | Nội dung | Quan hệ |
|---|---|---|
| `accounts` | Email, SĐT, password hash, họ tên, avatar, role, status, mốc xác thực email/đổi mật khẩu; ngày sinh, giới tính, địa chỉ | Định danh tài khoản chung |
| `member_profile` | Liên hệ khẩn cấp, mục tiêu tập luyện, ghi chú sức khỏe, số dư ví | 1–1 với account, chỉ MEMBER |
| `coach_profile` | Giới thiệu, kinh nghiệm, chứng chỉ, ảnh bìa | 1–1 với account, chỉ COACH |
| `receptionist_profile` | Ghi chú nhân sự | 1–1 với account, chỉ RECEPTIONIST |
| `manager_profile` | Ghi chú nhân sự | 1–1 với account, chỉ MANAGER |

- Tạo account và đúng profile cùng lúc, kể cả khi các field tùy chọn còn trống. Manager đầu tiên được tạo sẵn (seed) cùng profile.
- Profile không xóa độc lập với account. Vô hiệu hóa dùng trạng thái account.
- Không đổi role của account trong workflow hiện tại.
- Ghi chú sức khỏe chỉ chính chủ, Manager, Receptionist và Coach hiện tại của lớp member đang học được đọc.
- Ghi chú nhân sự chỉ Manager được sửa. Cập nhật thông tin cá nhân không cho sửa role, status hoặc số dư ví.

### Giao diện theo Role

- **Chung layout khung**: Navbar, Body (content area), Footer
- **Mỗi role có giao diện riêng**: menu và trang nội dung khác nhau theo role
- Frontend tổ chức theo **feature-based architecture**: có components dùng chung và components riêng trong từng feature
- **Mỗi role có trang Home riêng** với nội dung phù hợp
- **Manager có trang Overview & Reports** và **System Settings**

---

## 4. Business Rules chung (Global)

### 4.0.1 Security — Password & Authentication

| Rule ID | Mô tả | Chi tiết |
|---------|-------|---------|
| BR_G.1 | **Password phải hash** | Password hash bằng bcrypt trước khi lưu. Không lưu/log/trả về plaintext, password hash, OTP hoặc token. Mọi kết nối qua HTTPS. |
| BR_G.2 | **Phiên đăng nhập bằng cookie httpOnly** | Access token (JWT, 15 phút) và refresh token (chuỗi ngẫu nhiên, 30 ngày) lưu trong cookie httpOnly. Server chỉ lưu hash của refresh token. Mỗi lần làm mới phiên, refresh token cũ bị thu hồi và cấp token mới; dùng lại một refresh token đã thu hồi → thu hồi toàn bộ phiên của account. Đăng xuất thu hồi phiên hiện tại; có thao tác đăng xuất khỏi mọi thiết bị. Đổi/đặt lại mật khẩu thu hồi toàn bộ phiên. Access token bị từ chối khi account không ACTIVE hoặc token được cấp trước lần đổi mật khẩu gần nhất. |
| BR_G.3 | **Chống CSRF** | Frontend và API cùng origin. Cookie đặt `SameSite=Lax`, API chỉ nhận body JSON và từ chối request ghi (POST/PUT/PATCH/DELETE) có header `Origin` khác origin của hệ thống. Webhook thanh toán là ngoại lệ, xác thực bằng khóa riêng (BR_3.6). |
| BR_G.4 | **Giới hạn tần suất** | Giới hạn số request theo IP cho các endpoint xác thực (gửi OTP, đăng nhập, đăng ký, đặt lại mật khẩu, làm mới phiên) |
| BR_G.5 | **Captcha** | Gửi OTP yêu cầu captcha (Cloudflare Turnstile) |

### 4.0.2 Chính sách xóa dữ liệu

> [!IMPORTANT]
> **Nguyên tắc chung**: Không có thao tác xóa vĩnh viễn trên UI. Tùy entity, thao tác là xóa mềm (`deleted_at`), hủy nghiệp vụ bằng `CANCELLED`, hoặc vô hiệu hóa bằng trạng thái. Không phải bảng nào cũng có `deleted_at`.

#### Phân biệt `deleted_at` và trạng thái

| Field | Ý nghĩa | Ví dụ |
|-------|---------|-------|
| `deleted_at` | **Vòng đời dữ liệu** — entity biến mất khỏi giao diện nhưng vẫn tồn tại để tham chiếu | Xóa facility → không còn trong danh sách, booking cũ vẫn tham chiếu được |
| `status` / `is_active` | **Trạng thái nghiệp vụ** — entity vẫn tồn tại, chỉ đổi trạng thái hoạt động | Facility ngừng hoạt động; account `INACTIVE` / `BANNED` |

- Facility ngừng hoạt động = ngừng nhận booking vô thời hạn. Bảo trì **không** dùng trạng thái mà dùng lịch bảo trì (BR_2.19)
- Account không có xóa mềm; dùng `INACTIVE` / `BANNED`.

#### Phân loại entity theo chính sách xóa

| Nhóm | Entities | Xóa mềm | Xóa vĩnh viễn | Lý do |
|------|----------|:-----------:|:-----------:|-------|
| 🔴 **Không cho phép xóa** | Accounts, 4 bảng profile, Orders, Order Items, Wallet Transactions, Wallet Top-ups, Bank Transactions, Audit Logs, Class Attendance, Center Checkins, Member Memberships, Membership Orders, Coach Specializations, Class Coach Registrations | ❌ | ❌ | Dữ liệu tài chính, kết quả xét duyệt và lịch sử; dùng trạng thái khi có workflow |
| 🟠 **Hủy nghiệp vụ, giữ lịch sử** | Facility Bookings, Facility Packages, Class Enrollments, Class Sessions (kèm session notes) | ❌ | ❌ | Hủy bằng `CANCELLED`; hoàn tiền theo chính sách riêng. Không tái sử dụng enrollment đã hủy |
| 🟡 **Dữ liệu tạm, tự xóa** | Notifications (sau 90 ngày), OTP Codes (hết hạn/đã dùng sau 7 ngày), Refresh Tokens (hết hạn/đã thu hồi sau 30 ngày) | ❌ | ✅ (job nền) | Không có giá trị thống kê |
| 🟢 **Xóa mềm, giữ vĩnh viễn** | Facilities, Sports, Courses, Classes, Membership Packages, Coupons, Support Requests, Member Evaluations, Facility Maintenances | ✅ | ❌ | Có thể cần tham chiếu hoặc khôi phục |

> [!TIP]
> **Quy tắc vàng**: Entity được hóa đơn hoặc dữ liệu thống kê tham chiếu → **không bao giờ** xóa vĩnh viễn. Báo cáo, hóa đơn, audit và kiểm tra xung đột lịch luôn tính cả dữ liệu đã xóa mềm.

- Tên facility, tên bộ môn, mã coupon và cặp đánh giá (buổi, học viên) là duy nhất trong tập chưa xóa mềm. Khôi phục bản cũ phải kiểm tra trùng với bản đang tồn tại.
- Lớp chỉ được xóa mềm sau khi đã hủy lịch còn giữ chỗ và xử lý quyền lợi học viên; xóa mềm không tự giải phóng slot.

### 4.0.3 Audit Log

| Rule ID | Mô tả | Chi tiết |
|---------|-------|---------|
| BR_G.6 | **Một nơi lưu audit** | Mọi audit log lưu chung, ghi cùng lúc với thao tác nghiệp vụ |
| BR_G.7 | **Truy vấn có khoảng thời gian** | Mọi truy vấn audit log bắt buộc có khoảng thời gian. UI mặc định 7 ngày gần nhất |
| BR_G.8 | **Phân trang** | Audit log luôn phân trang theo cursor |
| BR_G.9 | **Những gì cần log** | Tạo/sửa/xóa quan trọng trên account và profile, facility, bộ môn, khóa học, lớp, buổi học, gói thành viên, coupon, cấu hình hệ thống, lịch bảo trì; xét duyệt chuyên môn/phân công Coach; bật/tắt auto-renew; chỉnh sửa điểm danh (F4); đối soát giao dịch ngân hàng; nạp ví tại quầy. Không log thao tác đọc, check-in, notification |
| BR_G.10 | **Không xóa** | Audit log không bao giờ bị xóa |

Audit chỉ ghi các field được phép; không ghi mật khẩu, OTP, token, khóa bí mật hoặc ghi chú sức khỏe.

### 4.0.4 Cơ chế thông báo

| Loại | Cơ chế | Ví dụ |
|------|--------|-------|
| **Tức thời (theo sự kiện)** | Thông báo in-app được tạo cùng lúc với thao tác; email gửi sau khi thao tác đã lưu thành công | Thay đổi lịch lớp, hoàn tiền, nạp ví thành công |
| **Định kỳ (job nền)** | Job nền quét và gửi thông báo | Nhắc lịch (trước 1h), gói sắp hết hạn, auto-renew |

Mỗi thông báo định kỳ chỉ gửi một lần cho mỗi người/sự kiện, kể cả khi job chạy lại.

### 4.0.5 Job nền

| Job | Tần suất | Việc |
|-----|----------|------|
| Membership | Hàng giờ | Xử lý gói đã đến hạn: auto-renew hoặc chuyển EXPIRED (BR_1.6); nhắc gói sắp hết hạn |
| Sĩ số lớp | Hàng giờ | Lớp sắp đến ngày bắt đầu mà thiếu sĩ số (BR_2.20) |
| Nhắc lịch | Hàng giờ | Booking/buổi học bắt đầu trong 1 giờ tới; thời điểm gửi nằm trong cửa sổ, không cam kết đúng 60 phút |
| Dọn dẹp | Hàng ngày | Xóa nhóm 🟡; chuyển yêu cầu nạp ví quá hạn sang EXPIRED |
| Đồng bộ SePay | Hàng giờ | Lấy từ SePay các giao dịch tiền vào mà hệ thống chưa nhận được và xử lý như giao dịch mới (BR_3.6) |
| Điểm danh mặc định *(F4)* | Hàng giờ | Buổi học không bị hủy đã kết thúc: member thuộc buổi mà chưa có điểm danh → `ABSENT`; không ghi đè điểm danh đã có |

- Mọi job chạy lại nhiều lần không gây xử lý lặp (không thu tiền, hoàn tiền, gửi thông báo hai lần).
- Job trễ không cấp thêm quyền lợi hoặc hoàn tiền sai: xử lý dựa trên dữ liệu và ngày thực tế.
- **Trạng thái suy ra từ ngày, không lưu**: lớp đang học / đã kết thúc, booking / buổi học / enrollment đã diễn ra.

### 4.0.6 Thời gian, validation và đồng thời

- Toàn trung tâm dùng một timezone thống nhất (Asia/Ho_Chi_Minh). Khoảng lịch là `[bắt đầu, kết thúc)`: hai khoảng tiếp giáp không trùng. Không hỗ trợ ca qua đêm; giờ mở cửa < giờ đóng cửa, giờ bắt đầu < giờ kết thúc.
- Giá, số ngày, deadline, quota không âm; thời lượng slot và số buổi dương; `1 <= min_students <= max_students`; phần trăm trong 0–100; tập thứ trong tuần không rỗng, không trùng, giá trị 0–6. Booking/buổi học phải khớp lưới giờ hoạt động; buổi học có thể chiếm nhiều slot liên tiếp.
- Hai thao tác đồng thời không được tạo trạng thái sai: vượt capacity/sĩ số/lượt coupon, trùng lịch member/Coach/facility, số dư ví âm hoặc lệch lịch sử giao dịch. Điều kiện luôn được kiểm tra lại tại thời điểm ghi, không dựa vào kết quả xem trước.
- Request hoặc webhook gửi lặp không được nạp/trừ/hoàn tiền hai lần.
- Đổi lịch phải kiểm tra lại facility, Coach và toàn bộ member bị ảnh hưởng. Package phải kiểm tra cả chuỗi lịch trước khi thanh toán. Đổi cấu hình hoặc xóa/vô hiệu hóa đối tượng có lịch phải qua cùng kiểm tra xung đột.

---

## 5. Phân tích chi tiết từng Flow

---

### 🔴 Flow 1: User & Membership Management (REQUIRED)

#### 5.1.1 Mô tả
Quản lý vòng đời người dùng: đăng ký, xác thực, thông tin cá nhân (Account + Profile), quản lý user, gói thành viên, chuyên môn HLV và audit log.

#### 5.1.2 Use Cases

| # | Use Case | Actor(s) | Mô tả |
|---|----------|----------|-------|
| UC_1.1 | Đăng ký tài khoản | Member | Nhập email + captcha → nhận OTP qua email → nhập OTP, họ tên, mật khẩu. Tạo account MEMBER và member profile có số dư ví 0 cùng lúc, đăng nhập luôn |
| UC_1.2 | Đăng nhập / Đăng xuất | All | Phiên đăng nhập theo BR_G.2; đăng xuất phiên hiện tại hoặc mọi thiết bị |
| UC_1.3 | Quên mật khẩu | All | Đặt lại mật khẩu qua OTP email |
| UC_1.4 | Đổi mật khẩu | All | Đổi mật khẩu, thu hồi mọi phiên |
| UC_1.5 | Cập nhật thông tin cá nhân | All | Sửa thông tin cá nhân và field profile được phép; ghi chú nhân sự chỉ Manager sửa. Không tự sửa role/status/số dư ví |
| UC_1.6 | Xem thông tin thành viên | Receptionist, Coach, Manager | Tìm kiếm & xem thông tin member. **Coach chỉ xem member trong lớp mình** |
| UC_1.7 | Quản lý danh sách người dùng | Manager | Tạo/sửa/vô hiệu hóa Coach, Receptionist; xem/sửa/vô hiệu hóa Member. Tài khoản Coach/Receptionist mới nhận email đặt mật khẩu |
| UC_1.8 | Quản lý gói thành viên | Manager | CRUD danh mục gói (tên, giá, thời hạn, quyền lợi) |
| UC_1.9 | Mua / Gia hạn gói thành viên | Member, Receptionist | Chọn gói, thêm một dòng MEMBERSHIP vào đơn đang soạn (UC_3.18), xem lại rồi thanh toán bằng ví hoặc tại quầy. Kỳ mua do hệ thống xác định (BR_1.7) |
| UC_1.10 | Kiểm tra trạng thái gói | Receptionist, Member | Xem gói hiện tại, ngày hết hạn, trạng thái, bật/tắt auto-renew |
| UC_1.11 | Hủy gói thành viên | Member, Receptionist | Chuyển gói sang `CANCELLED`, không hoàn tiền |
| UC_1.12 | Xem lịch sử thao tác | Manager | Audit log theo khoảng thời gian, phân trang |
| UC_1.13 | Đăng ký bộ môn chuyên môn | Coach | HLV đăng ký bộ môn mình có thể dạy |
| UC_1.14 | Duyệt chuyên môn HLV | Manager | Duyệt/từ chối bộ môn HLV đăng ký, kèm ghi chú |

#### 5.1.3 Business Rules

| Rule ID | Mô tả | Chi tiết |
|---------|-------|---------|
| BR_1.1 | **Tài khoản duy nhất** | Email được trim và chuẩn hóa chữ thường ở mọi luồng (đăng ký, đăng nhập, OTP, đặt lại mật khẩu) và là duy nhất. SĐT nếu có phải chuẩn hóa và duy nhất; không dùng chuỗi rỗng thay cho giá trị trống |
| BR_1.2 | **RBAC cố định** | 4 role cố định, quyền hardcode. Manager tạo Coach/Receptionist. Manager đầu tiên được seed |
| BR_1.3 | **Account + Profile** | Mỗi account có đúng một profile phù hợp role, tạo cùng lúc với account. Ngày sinh/giới tính/địa chỉ thuộc account; số dư ví thuộc member profile |
| BR_1.4 | **Gói thành viên có thời hạn** | Hiệu lực `[ngày bắt đầu, ngày kết thúc)`: có quyền lợi khi ACTIVE và ngày bắt đầu ≤ hôm nay < ngày kết thúc. Job trễ không cấp thêm quyền lợi. Ngày kết thúc = ngày bắt đầu + thời hạn gói |
| BR_1.5 | **Trạng thái gói** | Mỗi member tối đa một gói ACTIVE. Khi mua/gia hạn, gói ACTIVE đã quá hạn được chuyển trạng thái trước khi tạo gói mới. Mỗi kỳ chỉ được ghi khi order đã thanh toán; giữ lịch sử kể cả EXPIRED/CANCELLED |
| BR_1.6 | **Auto-renew** | Member tự bật/tắt, mặc định tắt. Khi gói đến hạn: nếu bật auto-renew, gói vẫn đang bán và ví đủ tiền → tạo order riêng có một dòng MEMBERSHIP, trừ ví một lần, thêm kỳ mới nối từ ngày kết thúc cũ, áp giá và quyền lợi hiện hành của gói, thông báo. Ngược lại → EXPIRED + thông báo. Job không truy thu các kỳ đã qua: nếu xử lý trễ, chỉ gia hạn một kỳ nối tiếp; member đã hết hạn thì mua lại từ hôm nay |
| BR_1.7 | **Gia hạn thủ công** | Gói còn hiệu lực: kỳ mới nối từ ngày kết thúc, là một dòng MEMBERSHIP trong order mới (có thể cùng booking/lớp). Gói hết hạn/đã hủy: tạo gói mới từ hôm nay. Không ghi đè kỳ cũ |
| BR_1.8 | **Quyền lợi membership** | Mỗi gói có: % giảm giá booking, % giảm giá đăng ký lớp, quyền vào phòng gym miễn phí, số slot booking miễn phí mỗi tháng. Quyền lợi được chốt theo **từng kỳ đã mua**: kỳ đã thanh toán giữ nguyên quyền lợi lúc mua dù Manager sửa gói sau đó. Mức giảm = giá gốc × tỷ lệ / 100, làm tròn theo mục 2.4, tính trước coupon. Vào gym miễn phí vẫn phải có booking để đếm capacity và không tính vào quota slot miễn phí. Quota slot miễn phí tính theo tháng lịch của ngày booking, theo kỳ đang hiệu lực tại ngày đó; booking bị hủy trước giờ bắt đầu trả lại quota. Không có gói hiệu lực thì không có quyền lợi |
| BR_1.9 | **Hủy gói không hoàn tiền** | Gói `CANCELLED` → mất quyền lợi ngay, không hoàn tiền |
| BR_1.10 | **Ví điện tử** | Mỗi Member có một ví, số dư ≥ 0, không rút tiền. Mọi thay đổi số dư đi kèm một giao dịch ví. Cập nhật profile không được sửa số dư |
| BR_1.11 | **Audit trail** | Mọi thao tác quan trọng ghi log (BR_G.9). Lịch sử check-in lưu riêng |
| BR_1.12 | **Coach đăng ký chuyên môn** | Mỗi lần đăng ký là một bản ghi riêng. REJECTED được đăng ký lại; mỗi (coach, bộ môn) chỉ có tối đa một đăng ký PENDING hoặc APPROVED. Chỉ dạy bộ môn đã APPROVED; kết quả xét duyệt cũ được giữ |
| BR_1.13 | **Quản lý bộ môn** | Không cho xóa/vô hiệu hóa bộ môn đang có lớp đang học. Có lớp `DRAFT` / `PENDING_APPROVAL` / `OPEN` chưa bắt đầu → cho phép xóa nhưng phải hủy các lớp đó + hoàn 100% cho member đã trả + thông báo Coach và Member liên quan |
| BR_1.14 | **Mỗi lớp tối đa một Coach hiện tại** | DRAFT/PENDING_APPROVAL có thể chưa có Coach; OPEN bắt buộc có đúng một Coach. Đăng ký được duyệt là quyết định lịch sử, không tự xác định người đang dạy |
| BR_1.15 | **Vô hiệu hóa Coach đang có lớp** | Coach bị vô hiệu hóa → mọi lớp `OPEN`/`PENDING_APPROVAL` chưa bắt đầu của coach đó về `PENDING_APPROVAL` không coach, thông báo Manager + member. Coach có lớp đang học → Manager phải phân công coach mới trước khi vô hiệu hóa |

---

### 🔴 Flow 2: Class Booking & Schedule Management (REQUIRED)

#### 5.2.1 Mô tả
Quản lý cơ sở vật chất, cấu hình slot, lịch bảo trì, đặt sân/phòng theo slot, gói sân định kỳ, khóa học, lớp học, buổi học, phân công HLV, đăng ký khóa học và lịch trình.

#### 5.2.2 Phân biệt khái niệm

| Khái niệm | Mô tả | Ví dụ |
|-----------|-------|-------|
| **Sport** | Bộ môn thể thao | Cầu lông, Boxing, Yoga, Gym |
| **Facility** | Cơ sở vật chất (sân/phòng), nhiều-nhiều với Sport | Sân CL 1, Phòng Yoga A, Phòng Gym |
| **Slot** | Khung giờ cố định trên facility (không lưu, tính từ giờ hoạt động + thời lượng slot) | 18:00–19:00 |
| **Facility Booking** | Đặt 1 slot trên 1 facility | Sân CL 1, 18:00–19:00, ngày 20/10 |
| **Facility Package** | Gói booking định kỳ hàng tuần, sinh ra N Facility Booking | Sân CL 1, T3+T5 18:00, 4 tuần → 8 booking |
| **Facility Maintenance** | Lịch bảo trì 1 facility: từ, đến, lý do | Sân CL 1 bảo trì 20/10 → 22/10 |
| **Course** | Khóa học (template): bộ môn, số buổi, mô tả, giá | Boxing cơ bản, 12 buổi |
| **Class** | Lớp học của course: lịch tuần, ngày bắt đầu, facility, coach, min/max học viên | Boxing CB - Lớp A (T3-T5 18:00, HLV Minh) |
| **Class Session** | 1 buổi học cụ thể: ngày, giờ, facility. Sinh tự động khi tạo lớp | Lớp A — buổi 3 — T5 20/10 18:00 — Phòng Boxing 1 |
| **Class Coach Registration** | Coach đăng ký dạy 1 lớp, chờ Manager duyệt | Coach Minh đăng ký Lớp A — PENDING |
| **Class Enrollment** | Member đăng ký lớp | Member A đăng ký Lớp A |

#### 5.2.3 Use Cases

| # | Use Case | Actor(s) | Mô tả |
|---|----------|----------|-------|
| UC_2.1 | Quản lý bộ môn | Manager | CRUD bộ môn (tên, mô tả, icon, trạng thái) |
| UC_2.2 | Quản lý cơ sở vật chất | Manager | CRUD facility (tên, loại, capacity, giá slot, các bộ môn, trạng thái) |
| UC_2.3 | Cấu hình hệ thống | Manager | Giờ mở/đóng cửa, thời lượng slot, giới hạn đặt trước (ngày), deadline hủy booking (giờ), deadline hủy khóa học (ngày), số ngày nhắc gói sắp hết hạn, số tiền nạp tối thiểu và thời hạn mã nạp ví. Trang System Settings riêng |
| UC_2.4 | Đặt lịch bảo trì facility | Manager | Chọn facility + từ/đến + lý do; xem trước booking/buổi học bị ảnh hưởng và chọn cách xử lý. Khi xác nhận, hệ thống kiểm tra lại rồi lưu bảo trì + hủy/hoàn/dời lịch cùng lúc |
| UC_2.5 | Xử lý buổi học bị ảnh hưởng | Manager | Đổi facility phù hợp bộ môn, dời ngày giờ hoặc hủy buổi; kiểm tra lịch Coach/member/facility. Hủy buổi hoàn tiền theo BR_2.23 |
| UC_2.6 | Đặt sân/phòng online (lẻ) | Member | Chọn facility + ngày + slot, thêm dòng FACILITY_BOOKING vào đơn đang soạn (UC_3.18), xem lại và thanh toán bằng ví. Booking chỉ được xác nhận khi checkout thành công |
| UC_2.7 | Đặt sân tại quầy | Receptionist | Chọn member hoặc nhập tên/SĐT guest; chọn facility + ngày + slot, thêm vào đơn (UC_3.18), xem lại rồi thu tiền (UC_3.4). Guest chỉ được đặt booking lẻ |
| UC_2.8 | Mua gói sân định kỳ | Member | Chọn facility + ngày bắt đầu + các thứ trong tuần + slot + số tuần. Khoảng sinh lịch `[ngày bắt đầu, ngày bắt đầu + 7 × số tuần)`; hiển thị các booking con dự kiến và thêm một dòng FACILITY_PACKAGE vào đơn. Checkout kiểm tra lại cả chuỗi lịch |
| UC_2.9 | Hủy booking | Member, Receptionist | Hủy booking lẻ hoặc cả gói định kỳ theo chính sách hoàn tiền |
| UC_2.10 | Xem lịch facility | Member, Receptionist | Trạng thái slot từng facility theo ngày (trống / còn x chỗ / đầy / lớp học / bảo trì / ngoài giờ) |
| UC_2.11 | Tạo khóa học | Manager | Tạo course (bộ môn, số buổi, mô tả, giá, ảnh) |
| UC_2.12 | Chia lớp cho khóa học | Manager | Chọn course, ngày bắt đầu, lịch tuần (thứ + giờ bắt đầu + giờ kết thúc), facility, min/max học viên; hệ thống sinh đủ số buổi theo thứ tự ngày và giữ slot ngay từ DRAFT. Giờ khớp lưới slot |
| UC_2.13 | HLV đăng ký dạy lớp | Coach | Xem lớp cần HLV (chỉ bộ môn đã duyệt), đăng ký dạy nếu không trùng lịch |
| UC_2.14 | Phân công HLV | Manager | Chọn 1 trong các coach đã đăng ký, hoặc phân công trực tiếp coach chưa đăng ký (đúng bộ môn, không trùng lịch) |
| UC_2.15 | Duyệt mở lớp | Manager | Lớp có đủ lịch + HLV → duyệt → `OPEN`. Từ chối → về `DRAFT` |
| UC_2.16 | Đăng ký lớp | Member, Receptionist | Chọn lớp và thêm dòng COURSE_ENROLLMENT vào đơn; kiểm tra sơ bộ OPEN/chưa bắt đầu/còn chỗ/không trùng lịch. Enrollment chỉ được tạo khi checkout thành công. Đăng ký lại sau khi hủy tạo đơn và enrollment mới |
| UC_2.17 | Hủy đăng ký lớp | Member, Receptionist | Hủy theo chính sách hoàn tiền |
| UC_2.18 | Hủy lớp | Manager | Chuyển lớp sang `CANCELLED`, hoàn tiền theo BR_2.7b, thông báo |
| UC_2.19 | Xem lịch tập cá nhân | Member | Xem tất cả booking + buổi học đã đăng ký |
| UC_2.20 | Xem lịch dạy | Coach | Xem các buổi học được phân công |
| UC_2.21 | Xem danh sách học viên | Coach | Xem member đã đăng ký lớp mình |
| UC_2.22 | Thông báo thay đổi lịch | System | Lịch lớp / buổi học / booking thay đổi hoặc bị hủy, bảo trì facility → tự động thông báo cho member & coach liên quan |

#### 5.2.4 Business Rules

| Rule ID | Mô tả | Chi tiết |
|---------|-------|---------|
| BR_2.1 | **Capacity theo slot** | Mỗi facility + slot có giới hạn capacity. Phòng gym: nhiều booking (14/20). Sân CL: một booking (1/1) |
| BR_2.2 | **Không vượt capacity** | Capacity = 1: không có booking thứ hai cùng slot. Capacity > 1: kiểm tra còn chỗ |
| BR_2.3 | **Lớp học chiếm slot facility** | Slot có buổi học → **khóa hoàn toàn** cho booking lẻ. Slot bị giữ từ khi lớp ở `DRAFT`, giải phóng khi lớp `CANCELLED` hoặc buổi bị hủy |
| BR_2.4 | **Giới hạn đặt trước** | Booking lẻ chỉ đặt trước tối đa X ngày (Manager cấu hình). **Không áp dụng cho gói sân định kỳ và lớp học** |
| BR_2.5 | **Giá booking** | Giá booking đồng nhất mọi khung giờ theo facility, Manager thay đổi được. Giá cuối = giá gốc → quyền lợi membership (BR_1.8) → coupon (BR_3.5) |
| BR_2.6 | **Chính sách hủy booking** | Hoàn 100% phần đã trả về ví nếu hủy trước giờ bắt đầu slot ≥ N giờ (Manager cấu hình). Quá deadline → không hoàn. **Guest: không hoàn**. Booking đã diễn ra không hủy được |
| BR_2.7 | **Chính sách hủy khóa học (member hủy)** | Hoàn 100% phần đã trả về ví nếu hủy trước ngày buổi học đầu tiên ≥ N ngày (Manager cấu hình). Sau mốc đó → không hoàn |
| BR_2.7b | **Lớp bị hủy bởi Manager/hệ thống** | Chưa bắt đầu → hoàn toàn bộ phần đã trả chưa hoàn của dòng đăng ký lớp, không hoàn các dòng khác trong order. Đang học → hoàn phần phân bổ của các buổi chưa diễn ra và chưa được hoàn (BR_3.12). Danh sách và số tiền được chốt trước khi đổi trạng thái lớp |
| BR_2.8 | **Course + Classes + Sessions** | Course là template. Buổi học sinh từ ngày bắt đầu, lịch tuần và số buổi. Sau khi sinh, buổi học là nguồn lịch thật; sửa template không ảnh hưởng lớp đã tạo. Facility của từng buổi phải hỗ trợ bộ môn của course |
| BR_2.9 | **Mở lớp cần Manager duyệt** | Danh mục nhận đăng ký chỉ hiển thị lớp OPEN chưa đến ngày bắt đầu, có buổi học và có Coach. Member đã mua vẫn xem được lịch sử/lịch lớp của mình khi lớp đổi trạng thái, kết thúc hoặc bị hủy |
| BR_2.10 | **Trạng thái lớp và ngày suy ra** | Lưu DRAFT / PENDING_APPROVAL / OPEN / CANCELLED. Ngày bắt đầu/kết thúc của lớp = ngày sớm nhất/muộn nhất của các buổi chưa hủy (kể cả buổi đã diễn ra). OPEN và ngày bắt đầu ≤ hôm nay ≤ ngày kết thúc → đang học; OPEN và ngày kết thúc < hôm nay → đã kết thúc. Lớp không còn buổi nào thì không được OPEN; dùng luồng hủy lớp. CANCELLED ưu tiên hơn trạng thái suy ra |
| BR_2.11 | **Sĩ số và điều kiện đăng ký** | Chỉ nhận khi lớp OPEN, chưa đến ngày bắt đầu, còn chỗ và không trùng lịch. Hủy rồi đăng ký lại tạo enrollment mới trong order mới; mỗi (lớp, member) chỉ có một enrollment đang hiệu lực, giữ các bản đã hủy |
| BR_2.12 | **Không trùng lịch HLV** | Coach không dạy 2 buổi trùng giờ |
| BR_2.13 | **Không trùng lịch Member** | Member không có 2 booking / buổi học trùng giờ |
| BR_2.14 | **HLV chỉ dạy bộ môn đã duyệt** | Coach chỉ đăng ký / được phân công lớp thuộc bộ môn đã được duyệt (BR_1.12) |
| BR_2.15 | **Phân công và lịch sử đăng ký Coach** | Manager chọn một Coach, từ chối các đăng ký PENDING còn lại; phân công trực tiếp cũng tạo bản ghi đăng ký nguồn MANAGER_ASSIGNED. Kiểm tra chuyên môn và xung đột lịch. Đăng ký APPROVED cũ là lịch sử; thay Coach không đổi quyết định cũ. Phân công không thay cho thao tác duyệt mở lớp |
| BR_2.16 | **Coach rút khỏi lớp** | Coach rút (hoặc bị vô hiệu hóa) khi lớp `OPEN` chưa bắt đầu → lớp về `PENDING_APPROVAL` không coach, thông báo member "đang tìm HLV thay thế", không hoàn tự động. Lớp đang học → chỉ Manager phân công coach mới, coach không tự rút |
| BR_2.17 | **Guest booking** | Guest chỉ mua booking lẻ tại quầy; bắt buộc tên và SĐT, người tạo là Receptionist. Không dùng ví, không quyền lợi membership, không coupon, không hoàn tiền kể cả khi bảo trì |
| BR_2.18 | **Gói sân định kỳ** | Sinh N booking theo UC_2.8. Bất kỳ slot nào đã có booking/lớp/bảo trì đều từ chối cả gói, kể cả facility còn capacity. Giá gốc của gói = tổng giá các booking con tại thời điểm mua; quyền lợi membership xét cho từng booking con; coupon áp một lần ở cấp order rồi phân bổ về gói và từng booking con. Hủy gói hủy các booking tương lai còn hiệu lực, hoàn phần phân bổ của các booking đủ deadline BR_2.6, không tính theo giá hiện tại |
| BR_2.19 | **Bảo trì facility** | Xem trước booking/buổi học bị ảnh hưởng; khi xác nhận kiểm tra lại. Booking của member bị hủy và hoàn 100% phần đã trả chưa hoàn (kể cả booking thuộc gói); guest không hoàn. Buổi học phải đổi phòng / dời / hủy (BR_2.23). Không sửa/xóa bảo trì đang diễn ra; không cho hai lịch bảo trì cùng facility chồng thời gian |
| BR_2.20 | **Lớp không đủ sĩ số** | Trước buổi đầu, lớp thiếu `min_students` và không được Manager bỏ qua → hủy lớp + hoàn toàn bộ phần đã trả + thông báo. Manager có thể bật bỏ qua kiểm tra sĩ số trước ngày bắt đầu (có audit), không sửa mất ngưỡng gốc. Nếu job trễ mà lớp đã có buổi diễn ra thì không tự hủy; Manager xử lý theo BR_2.7b |
| BR_2.21 | **Đổi cấu hình slot / giờ hoạt động** | Chỉ được đổi khi không còn booking / buổi học tương lai lệch lưới mới. Nếu có → liệt kê và từ chối |
| BR_2.22 | **Thông báo tự động** | Mọi thay đổi lịch (lớp, buổi, booking, bảo trì) → hệ thống tự thông báo, không có thao tác gửi thủ công |
| BR_2.23 | **Hủy riêng một buổi học** | Manager hủy một buổi chưa diễn ra mà không dạy bù → hoàn cho từng member đang học phần phân bổ của buổi đó, không phụ thuộc deadline member hủy. Dời buổi hoặc đổi phòng không hoàn tiền. Buổi đã hoàn không được hoàn lại lần nữa khi sau đó hủy cả lớp |

#### Bảng chuyển trạng thái lớp (BR_2.10)

| Từ → Đến | Ai / cái gì | Điều kiện |
|---|---|---|
| (tạo) → `DRAFT` | Manager (UC_2.12) | Có lịch tuần + ngày bắt đầu + facility; buổi học sinh xong, slot được giữ |
| `DRAFT` → `PENDING_APPROVAL` | Coach đăng ký (UC_2.13) hoặc Manager phân công (UC_2.14) | Coach đúng bộ môn, không trùng lịch |
| `PENDING_APPROVAL` → `OPEN` | Manager duyệt (UC_2.15) | Có đúng 1 coach |
| `PENDING_APPROVAL` → `DRAFT` | Manager từ chối | Coach nhận thông báo |
| `OPEN` → `PENDING_APPROVAL` | Hệ thống | Coach rút / bị vô hiệu hóa (BR_2.16) |
| `OPEN` → `CANCELLED` | Job kiểm tra sĩ số | Thiếu sĩ số, chưa học và không được bỏ qua (BR_2.20) |
| `OPEN` → *(đang học)* | Không chuyển — suy ra khi ngày bắt đầu ≤ hôm nay ≤ ngày kết thúc | — |
| *(đang học)* → *(đã kết thúc)* | Không chuyển — suy ra khi ngày kết thúc < hôm nay | — |
| `DRAFT` / `PENDING_APPROVAL` / `OPEN` (kể cả đang học) → `CANCELLED` | Manager (UC_2.18) hoặc hệ thống (BR_1.13) | Hoàn theo BR_2.7b + thông báo |

---

### 🔴 Flow 3: Payment & Report Management (REQUIRED)

#### 5.3.1 Mô tả
Quản lý ví điện tử, nạp ví qua chuyển khoản (SePay), giao dịch, hóa đơn, coupon, đối soát dòng tiền, báo cáo thống kê và yêu cầu hỗ trợ.

#### 5.3.2 Use Cases

| # | Use Case | Actor(s) | Mô tả |
|---|----------|----------|-------|
| UC_3.1 | Nạp tiền vào ví (chuyển khoản) | Member | Nhập số tiền → hệ thống tạo yêu cầu nạp có mã thanh toán riêng và mã VietQR (số tài khoản, số tiền, nội dung chuyển khoản) → member chuyển khoản bằng app ngân hàng → SePay báo giao dịch → hệ thống khớp mã + số tiền và cộng ví tự động → thông báo |
| UC_3.2 | Nạp tiền vào ví (tại quầy) | Receptionist | Ghi nhận tiền mặt/thẻ/chuyển khoản tại quầy → cộng ví |
| UC_3.3 | Thanh toán qua ví | Member | Sau UC_3.20, server kiểm tra lại toàn bộ đơn rồi trừ ví một lần theo tổng đã xác nhận; tạo order PAID, các dòng và dịch vụ cùng lúc. Đơn có dòng hợp lệ tổng 0 vẫn thành công, không trừ ví |
| UC_3.4 | Thanh toán tại quầy | Receptionist | Sau UC_3.20, kiểm tra lại các dòng cho cùng người mua; thu tiền một lần bằng một phương thức (tiền mặt/thẻ/chuyển khoản) rồi ghi order + dịch vụ. Guest chỉ có booking lẻ |
| UC_3.5 | Hoàn tiền | System | Hoàn về ví khi: hủy booking / gói sân / lớp (theo chính sách), lớp hoặc buổi bị hủy, bảo trì facility. Guest và membership không hoàn |
| UC_3.6 | Xuất hóa đơn | Receptionist, Member | Xuất PDF từ thông tin đã chốt lúc thanh toán (người mua, coupon, từng dòng, lịch/kỳ, đơn giá, giảm giá); không lấy tên/giá hiện tại |
| UC_3.7 | Xem số dư & lịch sử giao dịch | Member, Receptionist | Xem số dư, tất cả giao dịch ví (nạp, trừ, hoàn) và các yêu cầu nạp |
| UC_3.8 | Quản lý coupon | Manager | CRUD coupon (mục 2.5) |
| UC_3.9 | Áp dụng coupon | Member, Receptionist | Nhập một mã cho đơn đang soạn của member; xem giảm giá theo các dòng hợp lệ. Thêm/sửa/xóa dòng phải tính lại coupon. Chỉ tiêu lượt khi checkout thành công |
| UC_3.10 | Manager Overview | Manager | Doanh thu hôm nay, member mới, booking hôm nay, lớp đang diễn ra, giao dịch ngân hàng chưa khớp |
| UC_3.11 | Báo cáo doanh thu | Manager | Doanh thu theo ngày/tuần/tháng, theo loại dịch vụ |
| UC_3.12 | Báo cáo thành viên | Manager | Số lượng member, trạng thái gói, tỷ lệ gia hạn |
| UC_3.13 | Báo cáo sử dụng sân | Manager | Tỷ lệ lấp đầy, sân nào đặt nhiều nhất |
| UC_3.14 | Báo cáo khóa học | Manager | Số học viên/lớp, tỷ lệ lấp đầy lớp, HLV nhiều học viên nhất. (Tỷ lệ điểm danh: chỉ khi có F4) |
| UC_3.15 | Xuất báo cáo | Manager | Export PDF/Excel |
| UC_3.16 | Tạo yêu cầu hỗ trợ | Member | Tạo ticket (loại, tiêu đề, nội dung) |
| UC_3.17 | Xử lý yêu cầu hỗ trợ | Receptionist, Manager | Tiếp nhận, cập nhật trạng thái, phản hồi ticket |
| UC_3.18 | Thêm dịch vụ vào đơn đang soạn | Member, Receptionist | Chọn một dịch vụ và đủ thông tin theo mục 2.4.1, thêm một dòng vào đơn của cùng người mua. Hệ thống kiểm tra sơ bộ và tính lại tạm tính/ưu đãi, chưa thu tiền hoặc giữ chỗ |
| UC_3.19 | Sửa/xóa dòng trong đơn đang soạn | Member, Receptionist | Sửa lựa chọn hoặc xóa dòng trước thanh toán; kiểm tra lại và tính lại tổng/ưu đãi. Xóa dòng cuối → đơn rỗng, không thể thanh toán |
| UC_3.20 | Xem lại và xác nhận đơn | Member, Receptionist | Xem tất cả dòng, lịch/kỳ, giá gốc, giảm giá và tổng phải trả; chọn phương thức thanh toán. Server tính và kiểm tra lại toàn bộ; giá hoặc điều kiện thay đổi thì yêu cầu xem lại trước khi thanh toán |
| UC_3.21 | Đối soát giao dịch ngân hàng | Manager | Xem các giao dịch tiền vào do SePay báo về. Giao dịch không khớp yêu cầu nạp nào (sai nội dung, sai số tiền, chuyển trùng) → gán cho một member để cộng ví, hoặc đánh dấu bỏ qua kèm ghi chú. Xem đối soát tổng tiền vào theo từng ngày giữa SePay và hệ thống; ngày lệch được đánh dấu |
| UC_3.22 | Báo cáo dòng tiền ví | Manager | Theo khoảng thời gian: tổng nạp (chuyển khoản / tại quầy), tổng thanh toán bằng ví, tổng hoàn về ví, tổng số dư ví của member, giao dịch ngân hàng chưa xử lý |

#### 5.3.3 Business Rules

| Rule ID | Mô tả | Chi tiết |
|---------|-------|---------|
| BR_3.1 | **Ví không âm** | Số dư ≥ 0. Không cho phép giao dịch khi không đủ tiền |
| BR_3.2 | **Giao dịch atomic** | Nạp ví: xác nhận + cộng ví + ghi giao dịch cùng lúc. Mua: kiểm tra toàn bộ đơn (kể cả xung đột giữa các dòng), thu/trừ tiền một lần + order + dòng + dịch vụ; một dòng lỗi thì không ghi gì. Hoàn: cập nhật dịch vụ + số tiền đã hoàn của dòng và order + cộng ví + ghi giao dịch cùng lúc. Nạp ví không tạo order; hoàn tiền không tạo lần mua mới; thu tại quầy không trừ ví |
| BR_3.3 | **Loại giao dịch ví** | NẠP (TOP_UP): không gắn order; gắn giao dịch ngân hàng khi nạp qua chuyển khoản. THANH TOÁN (PAYMENT): gắn order, tối đa một lần mỗi order. HOÀN (REFUND): gắn order và đúng một dòng của order đó; một dòng có thể được hoàn nhiều lần cho các phần khác nhau. Số tiền luôn dương; đơn miễn phí không tạo giao dịch 0 đồng |
| BR_3.4 | **Hóa đơn bất biến** | Order được tạo ở trạng thái PAID cùng ≥1 dòng. Thông tin người mua, coupon và từng dòng (tên, lịch/kỳ, đơn giá, quyền lợi, phân bổ) được chốt lúc thanh toán. Không thêm/xóa/sửa dòng hoặc giá của hóa đơn đã thanh toán; chỉ số tiền đã hoàn tăng dần |
| BR_3.5 | **Coupon cho order nhiều loại** | Một mã/order, chỉ cho member có tài khoản; quota đếm theo order (kể cả order đã hoàn không trả lượt). Chỉ các dòng thuộc loại áp dụng được giảm; ngưỡng tối thiểu xét tổng giá gốc các dòng hợp lệ. Giảm FIXED/PERCENT (có trần) tính một lần trên tổng sau quyền lợi membership của các dòng hợp lệ, không vượt tổng đó, rồi phân bổ về từng dòng theo tỷ trọng (BR_3.12). Không tính lại coupon khi hoàn tiền |
| BR_3.6 | **Nạp ví qua SePay** | Yêu cầu nạp có mã thanh toán duy nhất, số tiền ≥ mức tối thiểu cấu hình, thời hạn hiển thị mã. SePay gửi giao dịch tiền vào kèm khóa xác thực; hệ thống chỉ chấp nhận khi khóa đúng. Mỗi giao dịch ngân hàng được ghi nhận đúng một lần dù SePay gửi lại nhiều lần. Chỉ cộng ví tự động khi nội dung chứa đúng mã của một yêu cầu chưa thành công và số tiền khớp chính xác; tiền đến sau khi mã hết hạn vẫn được cộng. Mọi trường hợp khác → giao dịch chưa khớp, chờ Manager đối soát (UC_3.21). Mỗi giao dịch ngân hàng cộng ví tối đa một lần. Giao dịch mà hệ thống không nhận được thông báo (SePay đã hết lượt gửi lại) được job đồng bộ lấy bổ sung và xử lý như trên |
| BR_3.7 | **Trạng thái order và hủy dịch vụ** | PAID khi chưa hoàn; PARTIALLY_REFUNDED khi đã hoàn một phần; REFUNDED khi tổng > 0 và đã hoàn toàn bộ. Hủy một dịch vụ không tự hủy các dòng khác. Membership không hoàn, nhưng booking cùng order vẫn có thể hoàn. Guest/quá deadline/dòng miễn phí có thể hủy mà không hoàn tiền |
| BR_3.8 | **Chỉ Manager xem báo cáo** | Overview, báo cáo và đối soát chỉ Manager truy cập |
| BR_3.9 | **Báo cáo theo thời gian** | Lọc theo ngày, tuần, tháng, năm, khoảng tùy chọn |
| BR_3.10 | **Yêu cầu hỗ trợ** | Loại: tài khoản, gói thành viên, booking, lớp học, thanh toán, khác. Trạng thái chỉ tiến: `OPEN` → `IN_PROGRESS` → `RESOLVED` → `CLOSED` |
| BR_3.11 | **Ưu tiên báo cáo** | Overview > Doanh thu > Dòng tiền ví > Thành viên > Sân > Khóa học > Export |

#### 5.3.4 Phân bổ tiền và báo cáo

| Rule ID | Mô tả | Chi tiết |
|---------|-------|---------|
| BR_3.12 | **Phân bổ tiền theo dòng và thành phần** | Tổng order = tổng các dòng. Coupon phân bổ về dòng theo tỷ trọng giá sau quyền lợi. Gói sân phân bổ tổng tiền của dòng về từng booking con theo giá sau quyền lợi; lớp học chia đều cho từng buổi. Làm tròn xuống đến đồng rồi phát phần dư từng đồng theo phần lẻ lớn nhất, hòa thì theo thứ tự dòng / lịch. Phân bổ được chốt lúc mua, không đổi khi dời buổi hoặc đổi giá |
| BR_3.13 | **Hoàn tiền từng dòng** | Chỉ hoàn phần đã trả chưa hoàn của component đủ điều kiện; không hoàn vượt dòng hoặc order; không hoàn dòng MEMBERSHIP. Mỗi component (booking, buổi học của một học viên, enrollment) chỉ được hoàn một lần. Hoàn nhiều dòng trong một thao tác tạo nhiều giao dịch hoàn. Buổi đã bắt đầu coi như đã diễn ra |
| BR_3.14 | **Báo cáo tiền** | Doanh thu theo order tại thời điểm thanh toán; theo loại dịch vụ tính từ tiền từng dòng. Nạp ví không tính là doanh thu. Khoản hoàn tính tại thời điểm hoàn, phân loại theo dòng được hoàn. Bao gồm dữ liệu đã xóa mềm |
| BR_3.15 | **Dòng tiền ví** | Tổng số dư ví của member = tổng nạp − tổng thanh toán bằng ví + tổng hoàn về ví. Tổng tiền nạp qua chuyển khoản = tổng giao dịch ngân hàng đã khớp hoặc đã gán cho member |

#### 5.3.5 Soạn đơn và điều kiện thanh toán

| Rule ID | Mô tả | Chi tiết |
|---------|-------|---------|
| BR_3.16 | **Một order có nhiều dòng** | Một order có 1..N dòng, cùng người mua và một phương thức thanh toán; tối đa một dòng MEMBERSHIP. Guest chỉ có booking lẻ. Không tạo dòng cho nạp ví hoặc cho từng booking con của gói |
| BR_3.17 | **Không thanh toán đơn rỗng** | Checkout phải có ít nhất một dòng hợp lệ. Server từ chối danh sách dòng thiếu, rỗng hoặc sai định dạng mà không thay đổi tiền, coupon hay dữ liệu. Bất kỳ dòng không hợp lệ làm thất bại cả đơn. Đơn có dòng hợp lệ tổng 0 không phải đơn rỗng |
| BR_3.18 | **Đơn đang soạn** | Chỉ tồn tại trước thanh toán, phía người dùng; không giữ chỗ, không tạo enrollment/membership, không tiêu quyền lợi/quota/coupon. Hóa đơn đã thanh toán là bất biến; muốn mua thêm tạo đơn mới, muốn hủy dùng luồng hủy/hoàn |
| BR_3.19 | **Kiểm tra lại khi checkout** | Server tự xác định sản phẩm, lịch/kỳ và giá; không tin giá/tổng/role từ client. Kiểm tra chỗ trống, điều kiện từng loại dịch vụ, coupon, quyền lợi, số dư và xung đột giữa các dòng lẫn dữ liệu đã có. Tổng khác với tổng người dùng đã xác nhận → yêu cầu xem lại, không tự thu số tiền mới. Gửi lại cùng một yêu cầu thanh toán không tạo order thứ hai |

---

### 🟡 Flow 4: Training & Attendance Management (OPTIONAL)

#### 5.4.1 Mô tả
Quản lý check-in trung tâm, điểm danh buổi học, session notes và đánh giá học viên. **Không rule nào ở F1–F3 phụ thuộc dữ liệu của flow này.**

#### 5.4.2 Use Cases

| # | Use Case | Actor(s) | Mô tả |
|---|----------|----------|-------|
| UC_4.1 | Check-in khi đến trung tâm | Receptionist | Tìm member (tên/email/SĐT) hoặc quét QR → ghi nhận check-in |
| UC_4.2 | Điểm danh buổi học | Coach, Manager | Điểm danh member trong buổi học (có mặt / vắng / muộn) kèm ghi chú. Sửa được sau đó |
| UC_4.3 | Tạo Session Notes | Coach | Ghi nội dung buổi học (tiêu đề, nội dung, bài tập, file đính kèm) cho cả lớp |
| UC_4.4 | Đánh giá học viên | Coach | Nhận xét + điểm 1–5 cho từng member, gắn với buổi học |
| UC_4.5 | Gửi thông báo lớp | Coach | Gửi thông báo hoặc bài tập cho member trong lớp |
| UC_4.6 | Xem lịch sử check-in | Member | Xem danh sách ngày đã đến trung tâm |
| UC_4.7 | Xem điểm danh lớp | Member | Xem điểm danh các buổi trong lớp đã đăng ký |
| UC_4.8 | Xem session notes & đánh giá | Member | Xem nội dung buổi học, nhận xét từ HLV |
| UC_4.9 | Nhận thông báo | Member | Nhận thông báo về lịch, bài tập, thay đổi |

#### 5.4.3 Business Rules

| Rule ID | Mô tả | Chi tiết |
|---------|-------|---------|
| BR_4.1 | **Check-in cần booking hoặc membership** | Cần booking CONFIRMED hôm đó, hoặc enrollment hợp lệ vào buổi học không bị hủy hôm đó, hoặc membership hiệu lực có quyền vào gym. Check-in không thay booking và không giữ capacity |
| BR_4.2 | **Điểm danh theo buổi** | Tối đa một bản ghi cho mỗi (member, buổi). Chỉ điểm danh member thuộc buổi học không bị hủy, xác định theo lịch sử enrollment tại thời điểm buổi học. Lưu người sửa và thời điểm sửa; mỗi lần sửa ghi audit log, không có bảng/màn hình lịch sử riêng. Job sau buổi chỉ thêm `ABSENT` khi chưa có bản ghi |
| BR_4.3 | **Coach chỉ điểm danh lớp mình** | Coach chỉ thao tác trên lớp được phân công. Manager thao tác mọi lớp |
| BR_4.4 | **Session Notes theo buổi** | Mỗi buổi học có tối đa 1 session notes (tiêu đề, nội dung rich text, file đính kèm) |
| BR_4.5 | **Đánh giá gắn với buổi và member** | Điểm 1–5 + nhận xét. Mỗi cặp (buổi, member) chỉ có một đánh giá chưa xóa; xóa mềm rồi tạo mới giữ bản cũ |
| BR_4.6 | **Check-in lưu riêng** | Lịch sử check-in là dữ liệu nghiệp vụ, không ghi vào audit log |
| BR_4.7 | **Check-in và điểm danh độc lập** | Check-in trung tâm và điểm danh buổi học là 2 việc riêng, không suy cái này từ cái kia |

---

## 6. Dependency giữa các Flow

- **Flow 1 (User & Membership)** → Flow 2, Flow 3, Flow 4 (mọi flow phụ thuộc auth & user)
- **Flow 2 (Facility & Class)** → Flow 3 (booking/enrollment tạo order & thanh toán), Flow 4 (điểm danh cần buổi học)
- **Flow 3 (Payment & Report)** phụ thuộc Flow 1 + Flow 2
- **Flow 4 (Training & Attendance)** phụ thuộc Flow 1 + Flow 2
- **Ràng buộc**: Không rule nào ở F1–F3 phụ thuộc dữ liệu F4 (hoàn tiền tính theo thời điểm, không theo check-in/điểm danh)

→ Thứ tự phát triển: **F1 → F2 → F3 → F4**

---

## 7. Notification Mapping

| Sự kiện | In-app | Email | Cơ chế |
|---------|:------:|:-----:|--------|
| Đăng ký tài khoản thành công | — | ✅ | Sự kiện |
| Nạp tiền ví thành công | ✅ | ✅ | Sự kiện |
| Có giao dịch ngân hàng chưa khớp (gửi Manager) | ✅ | — | Sự kiện |
| Mua gói / Đăng ký lớp / Đặt sân thành công | ✅ | ✅ | Sự kiện |
| Gói sắp hết hạn | ✅ | ✅ | Job nền |
| Auto-renew thành công | ✅ | ✅ | Job nền |
| Auto-renew thất bại (ví không đủ / gói ngừng bán) | ✅ | ✅ | Job nền |
| Gói đã hết hạn | ✅ | ✅ | Job nền |
| Manager duyệt / từ chối chuyên môn HLV | ✅ | — | Sự kiện |
| Manager duyệt / từ chối lớp (gửi Coach) | ✅ | — | Sự kiện |
| Lớp được mở đăng ký | ✅ | — | Sự kiện |
| Coach rút / lớp về PENDING_APPROVAL | ✅ | — | Sự kiện |
| Lớp bị hủy (thiếu sĩ số / Manager hủy / xóa bộ môn) | ✅ | ✅ | Sự kiện |
| Buổi học bị đổi phòng / dời / hủy | ✅ | ✅ | Sự kiện |
| Thay đổi / Hủy booking | ✅ | ✅ | Sự kiện |
| Bảo trì facility ảnh hưởng booking / lớp | ✅ | ✅ | Sự kiện |
| Coach tạo session notes mới | ✅ | — | Sự kiện |
| Coach gửi đánh giá | ✅ | — | Sự kiện |
| Hoàn tiền thành công | ✅ | ✅ | Sự kiện |
| Yêu cầu hỗ trợ được cập nhật | ✅ | — | Sự kiện |
| Nhắc lịch tập (trong 1h trước buổi) | ✅ | — | Job nền |

---

## 8. Kiểm tra nghiệm thu trọng tâm

- Hai request tranh booking/chỗ học/lượt coupon cuối: chỉ số request hợp lệ thành công, không vượt giới hạn.
- Booking và enrollment đồng thời của cùng member; phân công/dời buổi trùng lịch Coach; bảo trì chen vào lúc đang booking: không tạo lịch xung đột.
- SePay gửi lại cùng một giao dịch, checkout hoặc hoàn tiền gửi lại: không cộng/trừ/hoàn lần thứ hai; lỗi giữa chừng không để lại order/dịch vụ/số dư lệch nhau.
- Chuyển khoản sai nội dung, sai số tiền hoặc chuyển hai lần cho cùng một mã: không tự cộng ví, xuất hiện trong danh sách chưa khớp; Manager gán cho member thì cộng đúng một lần.
- Chuyển khoản sau khi mã hết hạn nhưng đúng mã và số tiền: vẫn được cộng ví.
- Hệ thống không nhận được thông báo của SePay cho một giao dịch: job đồng bộ bổ sung giao dịch và cộng ví đúng một lần; chạy đồng bộ nhiều lần không cộng thêm. Đối soát theo ngày khớp giữa SePay và hệ thống sau khi đồng bộ.
- Gia hạn sớm giữ từng kỳ; mua lúc job trễ không vướng gói ACTIVE cũ; job và mua đồng thời không tạo hai gói hoặc thu hai lần. Manager sửa quyền lợi gói không đổi quyền lợi của kỳ đã mua.
- Hủy rồi đăng ký lại lớp giữ hai enrollment thuộc các đơn tương ứng; chuyên môn REJECTED được gửi lại nhưng APPROVED không tạo yêu cầu trùng; F4 xóa mềm đánh giá rồi tạo lại vẫn giữ lịch sử.
- Dời/hủy buổi cuối cập nhật ngày lớp và trạng thái đúng; lớp đã kết thúc không nhận đăng ký mới; lớp mất Coach vẫn hiện lịch cho member đã mua.
- Hủy riêng một buổi rồi hủy cả lớp: buổi đã hoàn không được hoàn lần hai.
- Đổi tên/giá sản phẩm không đổi hóa đơn cũ; tổng phân bổ con bằng tiền của dòng, tổng các dòng bằng order; hủy nhiều lần không hoàn quá component/dòng/order; đơn miễn phí không tạo giao dịch 0 đồng.
- Guest thiếu tên/SĐT, dùng ví, dùng coupon hoặc mua loại khác booking lẻ bị từ chối; guest hủy không được hoàn.
- Quota slot miễn phí: dùng hết quota thì booking tiếp theo tính tiền; hủy booking miễn phí trước giờ bắt đầu trả lại quota; vào gym bằng quyền gym không trừ quota.
- Audit không chứa password/OTP/token/khóa bí mật/ghi chú sức khỏe; F4 không là điều kiện chạy F1–F3.
- Tạo account của mỗi role có đúng một profile tương ứng; lỗi tạo profile thì không có account. Không sửa được số dư qua API profile.
- Một order gồm booking + đăng ký lớp, trừ ví một lần; coupon chỉ giảm đúng loại dòng và tiêu một lượt. Hủy booking chỉ hoàn dòng booking. Membership trong đơn không giảm giá hồi tố cho các dòng cùng đơn.
- Một dòng hết chỗ, booking trùng lịch với lớp trong cùng đơn, hoặc dòng sai loại/chủ sở hữu làm checkout thất bại toàn bộ. Gói sân chỉ có một dòng tính tiền dù có nhiều booking con.
- Gọi API checkout trực tiếp với danh sách dòng thiếu/rỗng/sai định dạng bị từ chối, không thay đổi tiền/dữ liệu/quota; đơn có dòng hợp lệ miễn phí vẫn checkout được.
- Đăng xuất rồi dùng lại refresh token cũ bị từ chối; dùng lại refresh token đã xoay vòng làm thu hồi mọi phiên; đổi mật khẩu làm mọi thiết bị phải đăng nhập lại.
- Request ghi từ origin khác bị từ chối.
