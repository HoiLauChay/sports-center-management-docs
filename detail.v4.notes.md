# Ghi chú Detail v4

Tài liệu đi kèm `detail.v4.md`, ghi lại thay đổi so với `detail.v3.md` và lý do.

## 1. Quyết định đã chốt

| ID v3 | Quyết định | Vị trí trong v4 |
|---|---|---|
| D01 | Sửa điểm danh ghi vào audit log; không bảng/màn hình lịch sử riêng | BR_G.9, BR_4.2 |
| D02 | Guest không dùng coupon; coupon chỉ cho member có tài khoản | 2.4, 2.5, BR_2.17, BR_3.5 |
| D03 | Giá gốc gói sân = tổng giá booking con lúc mua; quyền lợi xét từng booking con; coupon một lần cấp order rồi phân bổ | BR_2.18, BR_3.12 |
| D04 | Quota slot miễn phí theo tháng lịch của ngày booking, theo kỳ hiệu lực tại ngày đó; hủy trước giờ bắt đầu trả quota; gym miễn phí không tính quota | BR_1.8 |
| D05 | Manager hủy riêng buổi không dạy bù → hoàn phần phân bổ của buổi; dời/đổi phòng không hoàn; không hoàn trùng khi hủy cả lớp | BR_2.23, BR_2.7b, BR_3.13 |
| D06 | Không truy thu nhiều kỳ; job trễ chỉ gia hạn một kỳ nối tiếp. Job membership chạy **hàng giờ** thay vì hàng ngày để giảm rủi ro trễ khi lịch kích hoạt từ bên ngoài lỡ một lần | BR_1.6, 4.0.5 |
| D07 | Kỳ đã trả giữ quyền lợi lúc mua; kỳ gia hạn áp giá/quyền lợi hiện hành; gói ngừng bán thì dừng auto-renew và thông báo | BR_1.6, BR_1.8 |

Mục 8 "Quyết định nghiệp vụ còn mở" của v3 được bỏ; mục 9 "Kiểm tra nghiệm thu" thành mục 8.

## 2. Thay đổi khác

| Nội dung | v3 | v4 | Lý do |
|---|---|---|---|
| Tech stack | Express + React + PostgreSQL (Prisma) | Thêm monorepo Turborepo + Bun, Supabase, Vercel, Resend, SePay | Chuyển toàn bộ sang Vercel để giảm chi phí |
| Ngôn ngữ hiển thị | Tiếng Anh | Tiếng Việt | Khớp với code hiện tại và thị trường (VND, ngân hàng VN) |
| Phiên đăng nhập (BR_G.2, UC_1.2) | Refresh JWT stateless, không thu hồi phía server | Refresh token ngẫu nhiên lưu hash, xoay vòng, phát hiện dùng lại, thu hồi được; có đăng xuất mọi thiết bị | Đã implement trong code; thu hồi phiên thật sự; chi phí chỉ một truy vấn mỗi lần làm mới phiên |
| CSRF (BR_G.3) | CSRF token cho mọi request ghi | Cùng origin + `SameSite=Lax` + chỉ nhận JSON + kiểm tra `Origin` | Đủ chống CSRF khi FE và API cùng origin; không cần lưu token, hợp với serverless |
| Captcha | Nằm trong mô tả API | BR_G.5 | Là yêu cầu bảo mật |
| Cổng nạp ví | VNPay / Momo callback | SePay: chuyển khoản VietQR + webhook biến động số dư; thêm đối soát (UC_3.21) và báo cáo dòng tiền ví (UC_3.22, BR_3.15) | SePay gói Free đủ cho project (50 giao dịch tiền vào/tháng, tài khoản cá nhân); không cần đăng ký doanh nghiệp như VNPay/Momo |
| Giỏ hàng (2.4.1, BR_3.18) | Không có bảng cart (chưa nói giỏ ở đâu) | Giỏ nằm phía người dùng; server tính lại khi xem và khi checkout | Không phát sinh ghi DB cho dữ liệu tạm; khớp với BR_3.16/3.17 nhận danh sách dòng từ payload |
| Job nền | Membership, sĩ số hàng ngày | Membership, sĩ số hàng giờ; dọn dẹp thêm refresh token và yêu cầu nạp quá hạn | Chạy lặp an toàn nên chạy dày hơn để giảm trễ |
| Yêu cầu hỗ trợ | Có "loại" trong UC nhưng không định nghĩa | Liệt kê loại ở BR_3.10 | Thiếu định nghĩa |
| Tiền | "Phép tính trung gian dùng decimal" | VND nguyên đồng | Chi tiết kiểu dữ liệu nằm ở DB |
| Mục 2.3 | Đoạn "đổi tên từ bản trước" | Bỏ, chỉ giữ tên bảng hiện hành | Changelog không thuộc tài liệu yêu cầu |
| Dòng meta | "Trạng thái tài liệu", "33 bảng" | Bỏ | Không phải yêu cầu |
| Số rule | BR_G.5–G.10 (audit) | BR_G.6–G.10; BR_G.5 là captcha; thêm BR_2.23, BR_3.15, BR_3.19 (BR_3.18 v3 → BR_3.19) | Đánh số lại |

## 3. Nội dung implement đã chuyển khỏi detail

Các đoạn dưới đây của v3 là chi tiết kỹ thuật, không thuộc phân tích yêu cầu. Nội dung tương đương nằm trong `db.v5.notes.md`, `design.v2.md` hoặc `api.design.md`.

| Nội dung v3 | Chuyển tới |
|---|---|
| Deferred constraint trigger cho account/profile | db.v5.notes §3 |
| Prisma client extension lọc `deleted_at`, partial unique index | db.v5.notes §2, §8 |
| Composite index của audit log (BR_G.6 cũ) | db.v5.md (`audit_logs`) |
| Thứ tự khóa (accounts → member_profile → …), lock theo class/facility/coupon/order | db.v5.notes §6 |
| Ghi chú `iat` và `password_changed_at` cùng đơn vị | api.design (Quy ước) |
| `entity_id` dạng text, `system_settings.id = 1` | db.v5.md |
| Chi tiết cột `invoice_snapshot` / `item_snapshot` | db.v5.notes §3, §7 |
| Chi tiết cột ledger (`order_id`, `order_item_id`, `gateway_transaction_id`) | db.v5.md (`ck_wtx_type_refs`) |
| `max_uses = 0` biểu thị không còn lượt | db.v5.md (`ck_coupons_uses` cho phép 0) |
