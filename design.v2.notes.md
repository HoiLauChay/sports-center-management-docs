# Ghi chú Design v2

Tài liệu đi kèm `design.v2.md` và `api.design.md`, ghi lại thay đổi so với `design.md`, lý do, và những điểm code hiện tại chưa khớp với thiết kế.

## 1. Thay đổi so với design.md

| Phần | design.md | design.v2.md / api.design.md | Lý do |
|---|---|---|---|
| Kiến trúc | Không có | Thêm mục Kiến trúc: monorepo, Vercel một project cùng origin, Supabase pooler, Resend, SePay, Vercel Blob, timezone, bảo mật, job nền, biến môi trường | Chuyển toàn bộ sang Vercel; thiết kế sơ bộ cần mô tả hạ tầng |
| Cookie | 3 cookie (`access_token`, `refresh_token` JWT, `csrf_token`) + header `X-CSRF-Token` | 2 cookie; `refresh_token` là chuỗi ngẫu nhiên lưu hash; kiểm tra `Origin` | Chốt theo code (detail.v4 BR_G.2, BR_G.3) |
| Auth | Không có logout-all; refresh chỉ cấp lại cookie | Thêm `POST /auth/logout-all`; refresh xoay vòng + phát hiện dùng lại; logout thu hồi phía server | Như trên |
| Response lỗi | Không có `retryAfter` | Thêm `retryAfter`, quy ước `errors[].path`, danh sách mã lỗi chung | Code đã trả `retryAfter` cho `OTP_COOLDOWN` |
| Idempotency | Chỉ nêu ở checkout / nạp quầy | Quy ước chung cho mọi thao tác tiền, lỗi `IDEMPOTENCY_CONFLICT` | Khớp `orders.idempotency_key`, `wallet_transactions.idempotency_key` |
| Giỏ hàng | `/cart`, `/cart/items`, `/cart/buyer`, `/cart/coupon` lưu trên server | `POST /checkout/quote` + `POST /checkout` nhận `items`; giỏ lưu ở trình duyệt | Không ghi DB cho dữ liệu tạm; khớp detail (không bảng cart, checkout nhận items) |
| Nạp ví | VNPay/Momo: `paymentUrl` redirect, callback theo provider | SePay: `$TOP_UP` có QR + nội dung chuyển khoản, polling trạng thái, webhook `POST /payments/sepay/webhook` | SePay gói Free, không cần đăng ký doanh nghiệp |
| Đối soát | Không có | `GET /bank-transactions`, `resolve`, `ignore`; trang `/admin/bank-transactions` | Xử lý chuyển khoản sai nội dung / sai số tiền (UC_3.21) |
| Báo cáo | overview, revenue, members, facilities, courses | Thêm `reports/wallet`; overview có `unmatchedBankTransactions`; revenue có `byPaymentMethod` | Quản lý dòng tiền (UC_3.22) |
| Membership | `$MEMBERSHIP.periods` không có quyền lợi | `$BENEFITS` theo từng kỳ, `currentBenefits`, `freeSlotsUsedThisMonth` | Quyền lợi chốt theo kỳ (D07), quota slot miễn phí (D04) |
| Hủy buổi | `POST /sessions/{id}/cancel` không nói hoàn tiền | Hoàn phần phân bổ của buổi; response có `refundTotal` | D05 |
| Booking | Không có `benefit` | `$BOOKING.benefit` | D04 |
| Specialization | `note` | `reviewNote` | Khớp cột `review_note` |
| Audit log | `changes` | `oldValues`, `newValues`, `ipAddress`, `account` | Khớp cột DB |
| Notification | `data`, `body` | `referenceType`, `referenceId`, `message`; thêm type `SUPPORT` | Khớp cột DB |
| Support | Không có loại | `category` | UC_3.16 |
| Sport / Facility / Course | Thiếu `isActive`, `description`, `thumbnailUrl` | Bổ sung | Khớp cột DB |
| Settings | 7 field | Thêm `topUpMinAmount`, `topUpExpiryMinutes` | Cấu hình SePay |
| Cron | Mục "Cron" mô tả job | Mục "Cron" là các endpoint `POST /cron/*` xác thực `CRON_SECRET`, lịch gọi từ cPanel | Vercel Hobby chỉ cho cron chạy mỗi ngày một lần |
| Upload | Không có | `POST /uploads/token` (Vercel Blob) | Function serverless không lưu file; body giới hạn 4,5 MB |
| Receptionist | Chỉ member hủy booking/enrollment | Receptionist cũng hủy được (UC_2.9, UC_2.17); hủy membership hộ member | Khớp use case trong detail |
| Comment trong khối code | Có `//` | Chuyển thành gạch đầu dòng | Tài liệu chính thức không chứa comment |
| Cấu trúc file | Một file gồm Frontend + API + Cron + Note | `design.v2.md`: kiến trúc + Frontend; `api.design.md`: quy ước + API. Bỏ mục "Note" (trùng use case trong detail.v4); quy tắc nghiệp vụ trong API chỉ ghi tham chiếu `BR_*` | Giảm trùng lặp giữa các tài liệu |

## 2. Quyết định thiết kế

- **Một Vercel project cùng origin**: cookie first-party, không cần CORS, FE và API deploy cùng lúc. Hai project riêng trên `*.vercel.app` sẽ là cross-site (Public Suffix List) và cookie `SameSite=Lax` không được gửi.
- **Bundle API bằng Bun**: Vercel không đọc `paths` của tsconfig và không import được file `.ts` từ workspace; bundle gom code nội bộ + `@sports-center/shared`, để các package npm cho Vercel tự trace.
- **Giỏ phía client**: mỗi thao tác thêm/sửa/xóa không tốn lượt gọi function và lượt ghi DB. Lễ tân mở nhiều tab để soạn nhiều đơn song song. Mất giỏ khi đổi thiết bị là chấp nhận được vì giỏ không giữ chỗ.
- **Polling trạng thái nạp ví** mỗi 3 giây thay vì WebSocket/SSE: function serverless không giữ kết nối lâu; SePay báo gần như tức thì nên số lượt gọi nhỏ.
- **Giới hạn 3 yêu cầu nạp PENDING mỗi member**: chống tạo mã rác; mã chưa dùng tự hết hạn.
- **Rate limit bằng Upstash Redis thay vì Vercel WAF**: gói Hobby của Vercel chỉ cho 1 rule rate limit mỗi project và cửa sổ tối đa 10 phút, không đủ cho 4 mức giới hạn (có cửa sổ 15 phút và 1 giờ). Upstash gói Free (500K lệnh/tháng) đủ vì mỗi lần kiểm tra chỉ tốn 1–2 lệnh và chỉ áp cho endpoint xác thực. Chọn cho qua khi Redis lỗi (fail-open) vì các endpoint này còn được bảo vệ bởi captcha, cooldown OTP, giới hạn nhập sai OTP và bcrypt.
- **Webhook trả 5xx khi lỗi hệ thống** để SePay retry (tối đa 7 lần trong 5 giờ); `bank_transactions.sepay_id` unique nên retry an toàn.
- **Đồng bộ bù và đối soát với SePay API**: webhook chỉ được SePay gửi lại tối đa 7 lần trong 5 giờ; job `sepay-sync` kéo các giao dịch bị lỡ bằng `since_id`, đối soát theo ngày so tổng với SePay. Báo cáo và biểu đồ vẫn tính từ DB vì SePay API chỉ trả dữ liệu giao dịch thô, không có dữ liệu thống kê.
- **Cron qua cPanel**: lệnh `curl` gọi endpoint; mọi job idempotent và xử lý theo lô, nên cPanel gọi trễ hay gọi lặp đều không sai dữ liệu.
- **Quota SePay**: gói Free 50 giao dịch tiền vào/tháng. Chỉ nạp ví đi qua SePay (mua hàng trừ ví), nên số giao dịch = số lần nạp; vượt thì tính phí theo lượt hoặc nâng gói.

## 3. Việc cần làm ở code để khớp thiết kế

Code hiện tại (sau refactor monorepo) mới có auth tối giản. Các điểm lệch:

| Điểm lệch | Hiện tại | Cần đổi |
|---|---|---|
| Schema DB | Bảng `users`, `refresh_tokens`, `otp_codes` | Migration theo `db.v5.md`: `accounts` + 4 profile; giữ `refresh_tokens`, `otp_codes` |
| Đăng ký | Không nhận `fullName` | Thêm `fullName` vào schema dùng chung và service; tạo member profile cùng account |
| Enum trạng thái account | `ACTIVE`, `INACTIVE` | Thêm `BANNED` (`packages/shared` và Prisma) |
| Route sau đăng nhập | `/app` | `/dashboard` |
| Kiểm tra `Origin` | Chưa có | Middleware cho request ghi, bỏ qua `/payments/sepay/webhook` và `/cron/*` |
| Chỉ nhận JSON | `express.json()` bỏ qua body khác nhưng không từ chối | Từ chối request ghi có `Content-Type` không phải `application/json` (415) |
| Email chào mừng khi đăng ký | Chưa có | Gửi qua Resend sau commit |
| Dọn dẹp refresh token | Chưa có | Job cleanup |
| Lỗi mã `CONFLICT`, `IDEMPOTENCY_CONFLICT`, … | Chưa có trong `ERROR_CODE` | Bổ sung dần theo tính năng |
