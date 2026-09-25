# Thiết kế sơ bộ

Yêu cầu nghiệp vụ: `detail.v4.md`. API: `api.design.md`. Database: `db.v5.md`.

## Kiến trúc

### Cấu trúc mã nguồn

Monorepo Turborepo + Bun:

- `apps/web`: React 19 + Vite + TanStack Router/Query + Ant Design + TailwindCSS. Chứa Vercel Function `api/index.js` phục vụ API.
- `apps/api`: Express 5 + Prisma 7 (adapter `pg`). Bundle thành `dist/app.js`; chạy local bằng Bun, chạy trên Vercel bằng Node.js runtime.
- `packages/shared`: hằng số (error code, enum, giới hạn auth), kiểu API và schema zod dùng chung cho web và api.
- `packages/typescript-config`, `packages/eslint-config`: cấu hình dùng chung.

### Triển khai

- Một Vercel project, Root Directory `apps/web`. Build: `turbo run build --filter=@sports-center/web`.
- Cùng origin: SPA phục vụ mọi đường dẫn; `/api/*` rewrite tới function chạy Express app.
- Database: Supabase PostgreSQL. Runtime kết nối qua transaction pooler (port 6543, pool tối đa 5 kết nối mỗi instance); migration qua session pooler (port 5432). Migration không chạy trong build của Vercel.
- Email: Resend, gửi sau khi transaction đã commit (dùng `waitUntil` để không chặn response). Email lỗi được job dọn dẹp gửi lại dựa trên `notifications.email_sent_at`.
- Nạp ví: SePay (chuyển khoản VietQR, webhook biến động số dư).
- Rate limit: Upstash Redis (gói Free) qua thư viện `@upstash/ratelimit`.
- File (avatar, ảnh bìa, file đính kèm session notes): Vercel Blob, upload trực tiếp từ client.
- Timezone nghiệp vụ: `Asia/Ho_Chi_Minh` (server Vercel chạy UTC; mọi phép tính "hôm nay", slot, lịch dùng timezone này).

### Bảo mật

- Cookie, kiểm tra `Origin`, captcha: BR_G.2, BR_G.3, BR_G.5; chi tiết ở phần Quy ước của `api.design.md`.
- Rate limit theo IP trong API: `@upstash/ratelimit` + Upstash Redis, thuật toán sliding window, khóa = endpoint + IP client (`x-real-ip`). Vượt giới hạn → 429 `RATE_LIMITED` kèm `retryAfter`:

  | Endpoint | Giới hạn |
  |---|---|
  | `POST /api/v1/auth/send-otp` | 5 / giờ |
  | `POST /api/v1/auth/login` | 10 / 15 phút |
  | `POST /api/v1/auth/register`, `POST /api/v1/auth/reset-password` | 30 / 15 phút |
  | `POST /api/v1/auth/refresh` | 120 / 15 phút |

- Upstash không truy cập được → cho request đi qua và ghi log lỗi, để sự cố Redis không chặn đăng nhập. Local không khai báo biến Upstash → tắt rate limit.
- Vercel Firewall chỉ dùng lớp chống DDoS mặc định, không cấu hình rule rate limit.

- Security headers cho SPA (CSP, HSTS, `X-Frame-Options`, `Referrer-Policy`, `Permissions-Policy`) khai báo trong `vercel.json`; API dùng `helmet`.
- Webhook SePay xác thực bằng API key; endpoint cron xác thực bằng `CRON_SECRET`.

### Job nền

Vercel gói miễn phí chỉ cho cron chạy mỗi ngày một lần, nên lịch chạy đặt ở Cron Jobs của hosting cPanel, gọi endpoint của hệ thống:

```
curl -fsS -X POST -H "Authorization: Bearer $CRON_SECRET" https://<domain>/api/v1/cron/<job>
```

| Job | Lịch (giờ VN) | Endpoint |
|---|---|---|
| Membership | Mỗi giờ, phút 05 | `POST /api/v1/cron/memberships` |
| Sĩ số lớp | Mỗi giờ, phút 10 | `POST /api/v1/cron/class-min-students` |
| Nhắc lịch | Mỗi giờ, phút 00 | `POST /api/v1/cron/reminders` |
| Dọn dẹp | Hàng ngày, 03:00 | `POST /api/v1/cron/cleanup` |
| Đồng bộ SePay | Mỗi giờ, phút 20 | `POST /api/v1/cron/sepay-sync` |
| Điểm danh mặc định *(F4)* | Mỗi giờ, phút 15 | `POST /api/v1/cron/attendance-defaults` |

- Nội dung từng job: detail.v4 mục 4.0.5. Mỗi lần gọi xử lý theo lô để xong trong giới hạn thời gian của function (300 giây); còn việc thì lần gọi sau xử lý tiếp.

### Biến môi trường

| Biến | App | Mô tả |
|---|---|---|
| `DATABASE_URL`, `DIRECT_URL` | api | Kết nối runtime / migration |
| `JWT_SECRET`, `TOKEN_HASH_SECRET` | api | Ký access token; hash refresh token và OTP |
| `RESEND_API_KEY`, `MAIL_FROM` | api | Gửi email |
| `TURNSTILE_SECRET_KEY` | api | Xác thực captcha |
| `SEPAY_WEBHOOK_API_KEY` | api | Xác thực webhook SePay |
| `SEPAY_BANK_ACCOUNT`, `SEPAY_BANK_CODE`, `SEPAY_ACCOUNT_NAME` | api | Thông tin tài khoản nhận tiền và tạo QR |
| `SEPAY_API_TOKEN` | api | Gọi SePay API (đồng bộ bù, đối soát) |
| `CRON_SECRET` | api | Xác thực endpoint cron |
| `UPSTASH_REDIS_REST_URL`, `UPSTASH_REDIS_REST_TOKEN` | api | Rate limit (bắt buộc ở production) |
| `BLOB_READ_WRITE_TOKEN` | api | Cấp token upload Vercel Blob |
| `VITE_TURNSTILE_SITE_KEY` | web | Site key captcha |

## Frontend

Chưa đăng nhập: chỉ Landing + các trang auth, route khác redirect về `/login`. Sau đăng nhập về `/dashboard`; sidebar và nội dung theo role. Giao diện tiếng Việt.

### Public

- Landing page (`/`): giới thiệu, bộ môn, gói; nút Đăng nhập / Đăng ký
- Login page (`/login`): email, mật khẩu
- Register page (`/register`): email + captcha → gửi OTP → OTP, họ tên, mật khẩu, xác nhận mật khẩu, đồng ý điều khoản
- Forgot password page (`/forgot-password`): email + captcha → OTP → mật khẩu mới

### Chung (mọi role)

- Dashboard (`/dashboard`): trang chủ theo role
- Profile (`/profile`): thông tin cá nhân + profile theo role, đổi mật khẩu, đăng xuất mọi thiết bị
- Notifications (`/notifications`): danh sách thông báo, đánh dấu đã đọc

### Member

- Wallet (`/wallet`): số dư, lịch sử giao dịch, lịch sử yêu cầu nạp
- Top-up (`/wallet/top-up`): nhập số tiền → tạo yêu cầu nạp
- Top-up detail (`/wallet/top-ups/{id}`): mã QR, số tài khoản, số tiền, nội dung chuyển khoản, thời gian còn lại; tự cập nhật khi nhận được tiền
- Membership packages (`/memberships`): danh sách gói, "Thêm vào đơn"
- My membership (`/memberships/mine`): gói hiện tại, các kỳ đã mua và quyền lợi từng kỳ, bật/tắt auto-renew, hủy gói
- Classes (`/classes`): lớp đang mở đăng ký (lọc bộ môn/HLV)
- Class detail (`/classes/{id}`): thông tin lớp, lịch buổi, HLV, "Thêm vào đơn"
- Bookings (`/bookings`): chọn bộ môn → facility → ngày → lưới slot (trống / đã đặt / lớp / bảo trì / đang chọn); tab gói định kỳ (thứ + slot + số tuần → xem trước); "Thêm vào đơn"; danh sách lượt đặt của tôi + hủy
- Schedule (`/schedule`): lịch booking + buổi học theo tuần/tháng
- Cart (`/cart`): các dòng đang soạn (lưu ở trình duyệt), sửa/xóa, nhập coupon, tổng tiền từ server, thanh toán bằng ví
- Orders (`/orders`), Order detail (`/orders/{id}`): hóa đơn, tải PDF
- Enrollments (`/enrollments`): lớp đã đăng ký, hủy
- Support (`/support`): tạo ticket (loại, tiêu đề, nội dung), xem trạng thái
- Training (`/training`) *(F4)*: điểm danh, session notes, đánh giá của HLV

### Coach

- Classes (`/coach/classes`): lớp đang dạy, danh sách học viên
- Schedule (`/coach/schedule`): buổi dạy theo tuần
- Open classes (`/coach/open-classes`): lớp cần HLV (đúng bộ môn đã duyệt) → đăng ký dạy
- Specializations (`/coach/specializations`): đăng ký bộ môn, trạng thái duyệt
- Session (`/coach/sessions/{id}`) *(F4)*: điểm danh, session notes, đánh giá học viên, gửi thông báo lớp

### Receptionist

- Members (`/reception/members`), Member detail (`/reception/members/{id}`): tìm & xem member, ví, gói, lịch sử
- Counter order (`/reception/order`): chọn member hoặc nhập guest (tên, SĐT) → thêm booking / lớp / gói → thu tiền (tiền mặt / thẻ / chuyển khoản)
- Top-up (`/reception/top-up`): nạp tiền vào ví member tại quầy
- Bookings (`/reception/bookings`): lịch facility theo ngày, danh sách booking
- Orders (`/reception/orders`): tra cứu hóa đơn, xuất PDF
- Support (`/reception/support`): tiếp nhận, cập nhật ticket
- Check-in (`/reception/checkin`) *(F4)*: tìm member / quét QR → check-in

### Manager

- Reports (`/admin/reports`): overview, doanh thu, dòng tiền ví, thành viên, sân, khóa học, export
- Bank transactions (`/admin/bank-transactions`): giao dịch tiền vào từ SePay, lọc chưa khớp → gán cho member hoặc bỏ qua; tab đối soát theo ngày với SePay
- Users (`/admin/users`), User detail (`/admin/users/{id}`): tạo Coach/Receptionist, sửa, đổi trạng thái
- Audit logs (`/admin/audit-logs`): lịch sử thao tác theo khoảng thời gian
- Specializations (`/admin/specializations`): duyệt/từ chối bộ môn HLV
- Sports (`/admin/sports`), Facilities (`/admin/facilities`), Courses (`/admin/courses`): CRUD
- Classes (`/admin/classes`), Class detail (`/admin/classes/{id}`): tạo lớp (sinh buổi), đăng ký HLV, phân công, duyệt mở lớp, hủy lớp, sửa/dời/hủy buổi
- Memberships (`/admin/memberships`), Coupons (`/admin/coupons`): CRUD
- Maintenances (`/admin/maintenances`): đặt lịch bảo trì, xem trước ảnh hưởng, chọn cách xử lý buổi học
- Settings (`/admin/settings`): giờ mở/đóng, slot, giới hạn đặt trước, deadline hủy, nhắc hết hạn gói, cấu hình nạp ví

