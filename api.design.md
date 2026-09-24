# API Design

REST API JSON dưới prefix `/api/v1`, phục vụ cùng origin với web (kiến trúc ở `design.v2.md`). Quy tắc nghiệp vụ được tham chiếu theo mã `BR_*` / `UC_*` trong `detail.v4.md`; cấu trúc dữ liệu theo `db.v5.md`.


## Quy ước

- Prefix `/api/v1`. JSON camelCase. `Money` = số nguyên VND. `Date` = `YYYY-MM-DD`, `Time` = `HH:mm`, `DateTime` = ISO 8601.
- Response:

  ```
  Thành công
  {
    "status": true,
    "message": string,
    "result": T
  }

  Lỗi
  {
    "status": false,
    "code": string,
    "message": string,
    "errors"?: [ { "path": string, "message": string } ],
    "retryAfter"?: int
  }
  ```

  Bên dưới, phần `response` chỉ ghi nội dung `result`. `errors[].path` có dạng `body.<field>`, `query.<field>`, `params.<field>`. `retryAfter` tính bằng giây.
- Mã lỗi chung: 401 `UNAUTHORIZED` / `TOKEN_EXPIRED` / `TOKEN_INVALID`, 403 `FORBIDDEN`, 404 `NOT_FOUND`, 409 `CONFLICT`, 422 `VALIDATION_ERROR`, 429 `RATE_LIMITED`, 500 `INTERNAL_ERROR`.
- Auth bằng 2 cookie httpOnly, `SameSite=Lax`:
  - `access_token`: JWT HS256, 15 phút, path `/`.
  - `refresh_token`: chuỗi ngẫu nhiên, 30 ngày, path `/api/v1/auth`; server lưu HMAC-SHA256 của token.
- Access token bị từ chối khi account không ACTIVE hoặc `iat` (giây) nhỏ hơn `passwordChangedAt` (làm tròn xuống giây).
- `role:` là các role được gọi; `owner` = chính chủ. Không ghi = mọi role đã đăng nhập.
- Danh sách phân trang: query `page`, `limit` (mặc định 20, tối đa 100) → `{ "items": [...], "page": int, "limit": int, "total": int }`. Audit log và notifications dùng `cursor` → `{ "items": [...], "nextCursor": string | null }`.
- Xóa = xóa mềm (`deletedAt`) hoặc chuyển `CANCELLED`, không xóa vật lý.
- Thao tác tiền có `idempotencyKey` (chuỗi do client sinh, ví dụ UUID). Gửi lại cùng key và cùng nội dung → trả lại kết quả cũ; cùng key khác nội dung → 409 `IDEMPOTENCY_CONFLICT`.

Đối tượng dùng chung:

```
$ACCOUNT
{
  "id": UUID,
  "email": string,
  "fullName": string,
  "phone": string | null,
  "dateOfBirth": Date | null,
  "gender": MALE | FEMALE | OTHER | null,
  "address": string | null,
  "avatarUrl": string | null,
  "role": MANAGER | COACH | MEMBER | RECEPTIONIST,
  "status": ACTIVE | INACTIVE | BANNED,
  "emailVerifiedAt": DateTime | null,
  "createdAt": DateTime,
  "profile": $MEMBER_PROFILE | $COACH_PROFILE | $STAFF_PROFILE
}

$MEMBER_PROFILE
{
  "emergencyContact": string | null,
  "fitnessGoals": string | null,
  "healthNotes": string | null,
  "walletBalance": Money
}

$COACH_PROFILE
{
  "bio": string | null,
  "experience": string | null,
  "certifications": string | null,
  "coverImageUrl": string | null
}

$STAFF_PROFILE
{
  "staffNotes": string | null
}

$PERSON
{
  "id": UUID,
  "fullName": string
}

$REF
{
  "id": UUID,
  "name": string
}
```

- `profile` theo role: MEMBER → `$MEMBER_PROFILE`, COACH → `$COACH_PROFILE`, RECEPTIONIST / MANAGER → `$STAFF_PROFILE`.
- `healthNotes` chỉ trả cho chính chủ, Manager, Receptionist và Coach hiện tại của lớp member đang học; người khác nhận `null`.

## Upload

- POST /api/v1/uploads/token:
  - Cấp token upload trực tiếp lên Vercel Blob
  - request:

  ```
  {
    "purpose": AVATAR | COVER_IMAGE | SESSION_ATTACHMENT | COURSE_THUMBNAIL | SPORT_ICON,
    "contentType": string,
    "size": int
  }
  ```
  - Ảnh tối đa 5 MB (`image/jpeg`, `image/png`, `image/webp`); file đính kèm tối đa 20 MB
  - response: `{ "uploadUrl": string, "fileUrl": string }`

## Auth

- POST /api/v1/auth/send-otp:
  - Captcha Turnstile; gửi lại sau 60 giây. OTP 6 số, hiệu lực 10 phút, sai tối đa 5 lần
  - request:

  ```
  {
    "email": string,
    "purpose": REGISTER | PASSWORD_RESET,
    "captchaToken": string
  }
  ```
  - lỗi: 400 `CAPTCHA_FAILED`, 409 `EMAIL_TAKEN` (REGISTER), 404 `EMAIL_NOT_FOUND` (PASSWORD_RESET), 429 `OTP_COOLDOWN` (kèm `retryAfter`)

- POST /api/v1/auth/register:
  - Tạo account MEMBER + member profile (ví = 0) cùng transaction, gửi email chào mừng, set cookie đăng nhập
  - Mật khẩu 8–72 ký tự, có chữ và số
  - request:

  ```
  {
    "email": string,
    "otp": string,
    "fullName": string,
    "password": string,
    "confirmPassword": string
  }
  ```
  - response: `$ACCOUNT`
  - lỗi: 400 `OTP_INVALID` / `OTP_EXPIRED` / `OTP_MAX_ATTEMPTS`, 409 `EMAIL_TAKEN`

- POST /api/v1/auth/login:
  - request:

  ```
  {
    "email": string,
    "password": string
  }
  ```
  - response: set 2 cookie, body `$ACCOUNT`
  - lỗi: 401 `INVALID_CREDENTIALS`, 403 `ACCOUNT_INACTIVE`

- POST /api/v1/auth/refresh:
  - Dùng cookie `refresh_token`. Thu hồi token cũ, cấp cặp cookie mới
  - Token đã bị thu hồi mà vẫn được gửi → thu hồi mọi phiên của account
  - Lỗi → 401 + xóa cookie

- POST /api/v1/auth/logout:
  - Thu hồi refresh token hiện tại, xóa cookie

- POST /api/v1/auth/logout-all:
  - Thu hồi mọi refresh token của account, xóa cookie

- GET /api/v1/auth/me:
  - response: `$ACCOUNT`

- PATCH /api/v1/auth/me:
  - Không nhận `email`, `role`, `status`, `walletBalance`, `staffNotes`
  - request:

  ```
  {
    "fullName"?: string,
    "phone"?: string | null,
    "dateOfBirth"?: Date | null,
    "gender"?: MALE | FEMALE | OTHER | null,
    "address"?: string | null,
    "avatarUrl"?: string | null,
    "profile"?: $MEMBER_PROFILE | $COACH_PROFILE
  }
  ```
  - response: `$ACCOUNT`

- POST /api/v1/auth/change-password:
  - Thu hồi mọi phiên, xóa cookie; mọi thiết bị phải đăng nhập lại
  - request:

  ```
  {
    "currentPassword": string,
    "password": string,
    "confirmPassword": string
  }
  ```
  - lỗi: 400 `INVALID_CREDENTIALS`

- POST /api/v1/auth/reset-password:
  - OTP purpose `PASSWORD_RESET`; thu hồi mọi phiên
  - request:

  ```
  {
    "email": string,
    "otp": string,
    "password": string,
    "confirmPassword": string
  }
  ```

## User

- GET /api/v1/users:
  - role: manager, receptionist (receptionist chỉ thấy MEMBER)
  - query: `q` (tên/email/SĐT), `role`, `status`, `page`, `limit`
  - response:

  ```
  {
    "items": [
      {
        "id": UUID,
        "email": string,
        "fullName": string,
        "phone": string | null,
        "avatarUrl": string | null,
        "role": enum,
        "status": enum,
        "createdAt": DateTime
      }
    ],
    "page": int,
    "limit": int,
    "total": int
  }
  ```

- POST /api/v1/users:
  - role: manager
  - Tạo account + profile cùng transaction; mật khẩu ngẫu nhiên; gửi email đặt mật khẩu (luồng quên mật khẩu)
  - request:

  ```
  {
    "email": string,
    "fullName": string,
    "role": COACH | RECEPTIONIST,
    "phone"?: string,
    "profile"?: $COACH_PROFILE | $STAFF_PROFILE
  }
  ```
  - response: `$ACCOUNT`

- GET /api/v1/users/{id}:
  - role: manager, receptionist, coach (coach chỉ xem member đang học lớp mình)
  - response: `$ACCOUNT`

- PATCH /api/v1/users/{id}:
  - role: manager
  - request: như `PATCH /auth/me`, thêm `profile.staffNotes`
  - response: `$ACCOUNT`

- PATCH /api/v1/users/{id}/status:
  - role: manager
  - Vô hiệu hóa: thu hồi mọi phiên của account
  - Vô hiệu hóa Coach theo BR_1.15; Coach còn lớp đang học → 409 `HAS_DEPENDENCIES`
  - request:

  ```
  {
    "status": ACTIVE | INACTIVE | BANNED,
    "reason"?: string
  }
  ```
  - response: `$ACCOUNT`

- GET /api/v1/audit-logs:
  - role: manager
  - query: `from`, `to` (bắt buộc), `entityType`, `accountId`, `action`, `cursor`, `limit`
  - response:

  ```
  {
    "items": [
      {
        "id": UUID,
        "account": $PERSON | null,
        "action": string,
        "entityType": string,
        "entityId": string,
        "oldValues": object | null,
        "newValues": object | null,
        "ipAddress": string | null,
        "createdAt": DateTime
      }
    ],
    "nextCursor": string | null
  }
  ```

## Membership

```
$PACKAGE
{
  "id": UUID,
  "name": string,
  "description": string | null,
  "price": Money,
  "durationDays": int,
  "bookingDiscountPct": int,
  "classDiscountPct": int,
  "gymAccess": bool,
  "freeBookingSlotsPerMonth": int,
  "isActive": bool
}

$BENEFITS
{
  "bookingDiscountPct": int,
  "classDiscountPct": int,
  "gymAccess": bool,
  "freeBookingSlotsPerMonth": int
}

$MEMBERSHIP
{
  "id": UUID,
  "package": $REF,
  "status": ACTIVE | EXPIRED | CANCELLED,
  "startDate": Date,
  "endDate": Date,
  "autoRenew": bool,
  "currentBenefits": $BENEFITS | null,
  "freeSlotsUsedThisMonth": int,
  "periods": [
    {
      "orderItemId": UUID,
      "periodStart": Date,
      "periodEnd": Date,
      "paidAmount": Money,
      "benefits": $BENEFITS
    }
  ]
}
```

- Mua / gia hạn: thêm dòng `MEMBERSHIP` vào đơn (xem Checkout); kỳ mua theo BR_1.7, quyền lợi theo kỳ (BR_1.8)
- `currentBenefits`: quyền lợi của kỳ đang hiệu lực; `null` khi không hiệu lực

- GET /api/v1/memberships:
  - response: `[$PACKAGE]` (không phải manager chỉ thấy `isActive = true`)

- POST /api/v1/memberships, PATCH /api/v1/memberships/{id}, DELETE /api/v1/memberships/{id}:
  - role: manager
  - request: các field của `$PACKAGE` trừ `id`
  - response: `$PACKAGE`
  - Ngừng bán (`isActive = false` hoặc xóa) → member đang bật auto-renew nhận thông báo

- GET /api/v1/me/memberships:
  - role: member
  - response:

  ```
  {
    "current": $MEMBERSHIP | null,
    "history": [$MEMBERSHIP]
  }
  ```

- PATCH /api/v1/me/memberships/{id}/auto-renew:
  - role: member (owner)
  - request:

  ```
  {
    "autoRenew": bool
  }
  ```
  - response: `$MEMBERSHIP`

- POST /api/v1/me/memberships/{id}/cancel:
  - role: member (owner)
  - Mất quyền lợi ngay, không hoàn tiền
  - response: `$MEMBERSHIP`

- GET /api/v1/users/{id}/memberships:
  - role: receptionist, manager
  - response: như `GET /me/memberships`

- POST /api/v1/users/{id}/memberships/{membershipId}/cancel:
  - role: receptionist, manager
  - response: `$MEMBERSHIP`

## Specialization (chuyên môn HLV)

```
$SPECIALIZATION
{
  "id": UUID,
  "coach": $PERSON,
  "sport": $REF,
  "status": PENDING | APPROVED | REJECTED,
  "reviewNote": string | null,
  "reviewedAt": DateTime | null,
  "createdAt": DateTime
}
```

- GET /api/v1/coach/specializations:
  - role: coach
  - response: `[$SPECIALIZATION]` của mình

- POST /api/v1/coach/specializations:
  - role: coach
  - Đã có PENDING / APPROVED cùng bộ môn → 409 `CONFLICT`. REJECTED được gửi lại
  - request:

  ```
  {
    "sportId": UUID
  }
  ```
  - response: `$SPECIALIZATION`

- GET /api/v1/specializations:
  - role: manager
  - query: `status`, `coachId`, `sportId`, `page`, `limit`
  - response: `{ "items": [$SPECIALIZATION], ... }`

- POST /api/v1/specializations/{id}/approve, POST /api/v1/specializations/{id}/reject:
  - role: manager
  - request:

  ```
  {
    "reviewNote"?: string
  }
  ```
  - response: `$SPECIALIZATION`

## Sport / Facility / Settings / Maintenance

```
$SPORT
{
  "id": UUID,
  "name": string,
  "description": string | null,
  "iconUrl": string | null,
  "isActive": bool
}

$FACILITY
{
  "id": UUID,
  "name": string,
  "type": GYM | COURT | ROOM | FIELD,
  "description": string | null,
  "capacityPerSlot": int,
  "pricePerSlot": Money,
  "isActive": bool,
  "sports": [$REF]
}

$SETTINGS
{
  "openTime": Time,
  "closeTime": Time,
  "slotDurationMinutes": int,
  "maxAdvanceBookingDays": int,
  "bookingCancelDeadlineHours": int,
  "courseCancelDeadlineDays": int,
  "membershipExpiryWarningDays": int,
  "topUpMinAmount": Money,
  "topUpExpiryMinutes": int
}

$MAINTENANCE
{
  "id": UUID,
  "facility": $REF,
  "startAt": DateTime,
  "endAt": DateTime,
  "reason": string,
  "createdAt": DateTime
}
```

- GET /api/v1/sports:
  - response: `[$SPORT]` (không phải manager chỉ thấy `isActive = true`)

- POST /api/v1/sports, PATCH /api/v1/sports/{id}:
  - role: manager
  - Vô hiệu hóa (`isActive = false`) theo cùng điều kiện với xóa
  - request:

  ```
  {
    "name": string,
    "description"?: string,
    "iconUrl"?: string,
    "isActive"?: bool
  }
  ```
  - response: `$SPORT`

- DELETE /api/v1/sports/{id}:
  - role: manager
  - Theo BR_1.13. Có lớp đang học → 409 `HAS_DEPENDENCIES`. Có lớp chưa học → lần đầu trả 409 `CONFIRMATION_REQUIRED` kèm `affectedClasses`, `refundTotal`; gọi lại với `?confirm=true` để thực hiện

- GET /api/v1/facilities:
  - query: `type`, `sportId`, `isActive`
  - response: `[$FACILITY]`

- POST /api/v1/facilities, PATCH /api/v1/facilities/{id}:
  - role: manager
  - request:

  ```
  {
    "name": string,
    "type": GYM | COURT | ROOM | FIELD,
    "description"?: string,
    "capacityPerSlot": int,
    "pricePerSlot": Money,
    "isActive": bool,
    "sportIds": [UUID]
  }
  ```
  - response: `$FACILITY`
  - Gỡ bộ môn đang có buổi học tương lai tại facility → 409 `HAS_DEPENDENCIES`

- DELETE /api/v1/facilities/{id}:
  - role: manager
  - Còn booking / buổi học tương lai → 409 `HAS_DEPENDENCIES`

- GET /api/v1/facilities/{id}/schedule:
  - query: `date`
  - response:

  ```
  {
    "facility": {
      "id": UUID,
      "name": string,
      "capacityPerSlot": int
    },
    "date": Date,
    "slots": [
      {
        "startTime": Time,
        "endTime": Time,
        "status": AVAILABLE | PARTIAL | FULL | CLASS | MAINTENANCE | CLOSED,
        "booked": int,
        "capacity": int,
        "classSession"?: { "classId": UUID, "className": string },
        "maintenance"?: { "id": UUID, "reason": string }
      }
    ]
  }
  ```

- GET /api/v1/settings:
  - response: `$SETTINGS`

- PATCH /api/v1/settings:
  - role: manager
  - Đổi giờ / slot khi còn booking / buổi học tương lai lệch lưới → 409 `SCHEDULE_CONFLICT` kèm danh sách
  - request: các field của `$SETTINGS` (partial)
  - response: `$SETTINGS`

- GET /api/v1/maintenances:
  - role: manager, receptionist
  - query: `facilityId`, `from`, `to`
  - response: `[$MAINTENANCE]`

- POST /api/v1/maintenances/preview:
  - role: manager
  - Chỉ xem ảnh hưởng, không ghi gì
  - request:

  ```
  {
    "facilityId": UUID,
    "startAt": DateTime,
    "endAt": DateTime,
    "reason": string
  }
  ```
  - response:

  ```
  {
    "affectedBookings": [
      {
        "id": UUID,
        "account": $PERSON | null,
        "guestName": string | null,
        "date": Date,
        "startTime": Time,
        "endTime": Time,
        "refundAmount": Money,
        "packageId": UUID | null
      }
    ],
    "affectedSessions": [
      {
        "id": UUID,
        "classId": UUID,
        "className": string,
        "date": Date,
        "startTime": Time,
        "endTime": Time,
        "alternatives": [$REF]
      }
    ]
  }
  ```
  - `alternatives`: các facility cùng bộ môn còn trống đúng khung giờ đó

- POST /api/v1/maintenances:
  - role: manager
  - Theo BR_2.19, một transaction
  - `sessionResolutions` phải có đủ mọi buổi bị ảnh hưởng; thiếu → 422
  - request:

  ```
  {
    "facilityId": UUID,
    "startAt": DateTime,
    "endAt": DateTime,
    "reason": string,
    "sessionResolutions": [
      { "sessionId": UUID, "action": MOVE_FACILITY, "facilityId": UUID }
      | { "sessionId": UUID, "action": RESCHEDULE, "date": Date, "startTime": Time, "endTime": Time, "facilityId"?: UUID }
      | { "sessionId": UUID, "action": CANCEL }
    ]
  }
  ```
  - response:

  ```
  {
    "maintenance": $MAINTENANCE,
    "cancelledBookings": int,
    "refundedTotal": Money,
    "sessionsUpdated": int
  }
  ```

- PATCH /api/v1/maintenances/{id}, DELETE /api/v1/maintenances/{id}:
  - role: manager
  - Chỉ khi chưa bắt đầu; mở rộng khoảng thời gian đi qua cùng luồng xem trước/xử lý như tạo mới

## Booking

```
$BOOKING
{
  "id": UUID,
  "facility": $REF,
  "date": Date,
  "startTime": Time,
  "endTime": Time,
  "status": CONFIRMED | CANCELLED,
  "unitPrice": Money,
  "benefit": NONE | DISCOUNT | GYM_ACCESS | FREE_SLOT,
  "paidAmount": Money,
  "refundedAmount": Money,
  "packageId": UUID | null,
  "orderItemId": UUID,
  "account": $PERSON | null,
  "guestName": string | null,
  "guestPhone": string | null,
  "createdAt": DateTime
}

$FACILITY_PACKAGE
{
  "id": UUID,
  "facility": $REF,
  "startDate": Date,
  "endDate": Date,
  "daysOfWeek": [int],
  "startTime": Time,
  "endTime": Time,
  "status": ACTIVE | CANCELLED,
  "unitPrice": Money,
  "bookings": [$BOOKING]
}
```

- `daysOfWeek`: 0 = Chủ nhật … 6 = Thứ bảy
- `paidAmount`: phần tiền phân bổ cho booking lúc mua
- Mua booking / gói định kỳ: thêm dòng `FACILITY_BOOKING` / `FACILITY_PACKAGE` vào đơn (xem Checkout)

- GET /api/v1/me/bookings:
  - role: member
  - query: `from`, `to`, `status`, `page`, `limit`
  - response: `{ "items": [$BOOKING], ... }`

- GET /api/v1/bookings:
  - role: receptionist, manager
  - query: như trên, thêm `facilityId`, `accountId`, `guestPhone`, `date`
  - response: `{ "items": [$BOOKING], ... }`

- GET /api/v1/bookings/{id}:
  - role: owner, receptionist, manager
  - response: `$BOOKING`

- POST /api/v1/bookings/{id}/cancel:
  - role: member (owner), receptionist
  - Hoàn theo BR_2.6. Đã diễn ra → 409 `INVALID_STATE`
  - response:

  ```
  {
    "booking": $BOOKING,
    "refund": {
      "amount": Money,
      "reason": FULL | PAST_DEADLINE | GUEST | FREE_ITEM
    }
  }
  ```

- POST /api/v1/facility-packages/preview:
  - role: member
  - Bất kỳ slot nào đã có booking / lớp / bảo trì → cả gói không hợp lệ
  - request:

  ```
  {
    "facilityId": UUID,
    "startDate": Date,
    "daysOfWeek": [int],
    "startTime": Time,
    "endTime": Time,
    "weeks": int
  }
  ```
  - response:

  ```
  {
    "bookings": [
      {
        "date": Date,
        "startTime": Time,
        "endTime": Time,
        "available": bool,
        "conflict"?: BOOKED | CLASS | MAINTENANCE | CLOSED
      }
    ],
    "isValid": bool,
    "unitPrice": Money,
    "basePrice": Money
  }
  ```

- GET /api/v1/me/facility-packages:
  - role: member
  - response: `[$FACILITY_PACKAGE]`

- POST /api/v1/facility-packages/{id}/cancel:
  - role: member (owner), receptionist
  - Theo BR_2.18
  - response:

  ```
  {
    "package": $FACILITY_PACKAGE,
    "cancelledBookings": int,
    "refundTotal": Money
  }
  ```

- GET /api/v1/me/schedule:
  - role: member
  - query: `from`, `to`
  - response: booking + buổi học, sắp theo thời gian

  ```
  [
    {
      "kind": BOOKING,
      "id": UUID,
      "date": Date,
      "startTime": Time,
      "endTime": Time,
      "facility": $REF,
      "status": CONFIRMED | CANCELLED
    }
    | {
      "kind": CLASS_SESSION,
      "id": UUID,
      "date": Date,
      "startTime": Time,
      "endTime": Time,
      "facility": $REF,
      "class": { "id": UUID, "name": string, "coach": $PERSON | null },
      "status": SCHEDULED | CANCELLED
    }
  ]
  ```

## Course / Class / Session / Enrollment

```
$COURSE
{
  "id": UUID,
  "name": string,
  "description": string | null,
  "sport": $REF,
  "totalSessions": int,
  "price": Money,
  "thumbnailUrl": string | null
}

$CLASS
{
  "id": UUID,
  "name": string,
  "course": $COURSE,
  "status": DRAFT | PENDING_APPROVAL | OPEN | CANCELLED,
  "derivedStatus": UPCOMING | ONGOING | COMPLETED | null,
  "startDate": Date | null,
  "endDate": Date | null,
  "weeklySchedule": [ { "dayOfWeek": int, "startTime": Time, "endTime": Time } ],
  "facility": $REF,
  "coach": $PERSON | null,
  "minStudents": int,
  "maxStudents": int,
  "enrolledCount": int,
  "minStudentsOverride": bool,
  "cancelReason": string | null
}

$SESSION
{
  "id": UUID,
  "classId": UUID,
  "sessionNumber": int,
  "date": Date,
  "startTime": Time,
  "endTime": Time,
  "facility": $REF,
  "status": SCHEDULED | CANCELLED,
  "cancelReason": string | null
}

$COACH_REGISTRATION
{
  "id": UUID,
  "classId": UUID,
  "coach": $PERSON,
  "status": PENDING | APPROVED | REJECTED,
  "source": COACH_REGISTERED | MANAGER_ASSIGNED,
  "createdAt": DateTime
}

$ENROLLMENT
{
  "id": UUID,
  "class": $REF,
  "account": $PERSON,
  "status": ENROLLED | CANCELLED,
  "paidAmount": Money,
  "refundedAmount": Money,
  "orderItemId": UUID,
  "enrolledAt": DateTime
}
```

- `derivedStatus` chỉ có khi `status = OPEN`
- Vòng đời lớp: bảng chuyển trạng thái BR_2.10
- Đăng ký lớp: thêm dòng `COURSE_ENROLLMENT` vào đơn (xem Checkout)

- GET /api/v1/courses:
  - response: `[$COURSE]`

- POST /api/v1/courses, PATCH /api/v1/courses/{id}, DELETE /api/v1/courses/{id}:
  - role: manager
  - request:

  ```
  {
    "name": string,
    "description"?: string,
    "sportId": UUID,
    "totalSessions": int,
    "price": Money,
    "thumbnailUrl"?: string
  }
  ```
  - response: `$COURSE`

- GET /api/v1/classes:
  - query: `status`, `courseId`, `sportId`, `coachId`, `needsCoach`, `openForEnrollment`, `page`, `limit`
  - Member: lớp nhận đăng ký (BR_2.9) + lớp mình đã mua. Coach với `needsCoach=true`: chỉ bộ môn đã duyệt
  - response: `{ "items": [$CLASS], ... }`

- POST /api/v1/classes:
  - role: manager
  - Sinh đúng `course.totalSessions` buổi từ `startDate` theo lịch tuần; slot trùng → 409 `SCHEDULE_CONFLICT`
  - request:

  ```
  {
    "courseId": UUID,
    "name": string,
    "facilityId": UUID,
    "startDate": Date,
    "weeklySchedule": [ { "dayOfWeek": int, "startTime": Time, "endTime": Time } ],
    "minStudents": int,
    "maxStudents": int
  }
  ```
  - response: `$CLASS` + `"sessions": [$SESSION]`

- GET /api/v1/classes/{id}:
  - response: `$CLASS` + `"sessions": [$SESSION]` (+ `"coachRegistrations": [$COACH_REGISTRATION]` cho manager)

- PATCH /api/v1/classes/{id}:
  - role: manager
  - Chỉ trước ngày bắt đầu; `maxStudents` không nhỏ hơn số đang học
  - request:

  ```
  {
    "name"?: string,
    "minStudents"?: int,
    "maxStudents"?: int
  }
  ```
  - response: `$CLASS`

- PATCH /api/v1/classes/{id}/min-students-override:
  - role: manager
  - Chỉ trước ngày bắt đầu
  - request:

  ```
  {
    "override": bool
  }
  ```
  - response: `$CLASS`

- POST /api/v1/classes/{id}/approve, POST /api/v1/classes/{id}/reject:
  - role: manager
  - request:

  ```
  {
    "note"?: string
  }
  ```
  - response: `$CLASS`

- POST /api/v1/classes/{id}/cancel:
  - role: manager
  - Hoàn theo BR_2.7b
  - request:

  ```
  {
    "reason": string
  }
  ```
  - response:

  ```
  {
    "class": $CLASS,
    "refundTotal": Money
  }
  ```

- GET /api/v1/classes/{id}/sessions:
  - response: `[$SESSION]`

- PATCH /api/v1/sessions/{id}:
  - role: manager
  - Trùng lịch facility / coach / học viên → 409 `SCHEDULE_CONFLICT`
  - request:

  ```
  {
    "date"?: Date,
    "startTime"?: Time,
    "endTime"?: Time,
    "facilityId"?: UUID
  }
  ```
  - response: `$SESSION`

- POST /api/v1/sessions/{id}/cancel:
  - role: manager
  - Theo BR_2.23. Hủy buổi cuối cùng còn lại → 409 `INVALID_STATE` (dùng hủy lớp)
  - request:

  ```
  {
    "reason": string
  }
  ```
  - response: `{ "session": $SESSION, "refundTotal": Money }`

- GET /api/v1/coach/classes:
  - role: coach
  - response: `[$CLASS]` đang dạy

- GET /api/v1/coach/schedule:
  - role: coach
  - query: `from`, `to`
  - response: `[$SESSION + "class": $REF]`

- POST /api/v1/classes/{id}/coach-registrations:
  - role: coach
  - Bộ môn chưa duyệt → 403; trùng lịch → 409 `SCHEDULE_CONFLICT`
  - response: `$COACH_REGISTRATION`

- GET /api/v1/classes/{id}/coach-registrations:
  - role: manager
  - response: `[$COACH_REGISTRATION]`

- POST /api/v1/classes/{id}/assign-coach:
  - role: manager
  - Chọn 1 đăng ký (APPROVED, các PENDING còn lại REJECTED) hoặc phân công trực tiếp coach
  - request:

  ```
  { "registrationId": UUID }
  hoặc
  { "coachId": UUID }
  ```
  - response: `$CLASS`

- POST /api/v1/classes/{id}/withdraw:
  - role: coach hiện tại của lớp
  - Lớp đang học → 409 `INVALID_STATE`

- GET /api/v1/classes/{id}/enrollments:
  - role: coach (lớp mình), manager, receptionist
  - response: `[$ENROLLMENT]`

- GET /api/v1/me/enrollments:
  - role: member
  - response: `[$ENROLLMENT + "class": $CLASS]`

- POST /api/v1/enrollments/{id}/cancel:
  - role: member (owner), receptionist
  - Hoàn theo BR_2.7
  - response:

  ```
  {
    "enrollment": $ENROLLMENT,
    "refund": {
      "amount": Money,
      "reason": FULL | PAST_DEADLINE | FREE_ITEM
    }
  }
  ```

## Checkout / Order

- Đơn đang soạn lưu ở trình duyệt (member: `localStorage`; lễ tân: theo tab soạn đơn), server không lưu (BR_3.18)
- Giá và điều kiện: BR_2.5, BR_1.8, BR_3.5; kiểm tra khi checkout: BR_3.16–BR_3.19; hóa đơn bất biến: BR_3.4
- Hoàn tiền không có API riêng, là hệ quả của các API hủy dịch vụ (BR_3.13)

```
$CHECKOUT_ITEM_INPUT
  { "type": FACILITY_BOOKING,  "facilityId": UUID, "date": Date, "startTime": Time, "endTime": Time }
| { "type": FACILITY_PACKAGE,  "facilityId": UUID, "startDate": Date, "daysOfWeek": [int], "startTime": Time, "endTime": Time, "weeks": int }
| { "type": COURSE_ENROLLMENT, "classId": UUID }
| { "type": MEMBERSHIP,        "packageId": UUID }

$BUYER
  { "accountId": UUID }
| { "guest": { "name": string, "phone": string } }

$QUOTE
{
  "items": [
    {
      "lineNumber": int,
      "type": FACILITY_BOOKING | FACILITY_PACKAGE | COURSE_ENROLLMENT | MEMBERSHIP,
      "selection": $CHECKOUT_ITEM_INPUT,
      "valid": bool,
      "error"?: { "code": string, "message": string },
      "snapshot": object,
      "subtotal": Money,
      "membershipDiscount": Money,
      "couponDiscount": Money,
      "total": Money
    }
  ],
  "coupon": { "code": string, "valid": bool, "discount": Money, "error"?: string } | null,
  "subtotal": Money,
  "membershipDiscount": Money,
  "couponDiscount": Money,
  "total": Money,
  "walletBalance": Money | null,
  "canCheckout": bool
}
```

- Tối đa một dòng `MEMBERSHIP` mỗi đơn; guest chỉ có `FACILITY_BOOKING`
- `buyer` bắt buộc khi receptionist gọi; member luôn mua cho chính mình (bỏ qua `buyer`)
- `lineNumber` theo thứ tự `items` gửi lên, bắt đầu từ 1

Ví dụ `$QUOTE.items`:

```
[
  {
    "lineNumber": 1,
    "type": "FACILITY_BOOKING",
    "selection": { "type": "FACILITY_BOOKING", "facilityId": "f2…", "date": "2026-09-17", "startTime": "18:00", "endTime": "20:00" },
    "snapshot": {
      "facilityName": "Sân cầu lông 2",
      "sportName": "Cầu lông",
      "date": "2026-09-17",
      "startTime": "18:00",
      "endTime": "20:00",
      "slots": 2,
      "pricePerSlot": 120000
    },
    "valid": true,
    "subtotal": 240000,
    "membershipDiscount": 60000,
    "couponDiscount": 0,
    "total": 180000
  },
  {
    "lineNumber": 2,
    "type": "COURSE_ENROLLMENT",
    "selection": { "type": "COURSE_ENROLLMENT", "classId": "c9…" },
    "snapshot": {
      "className": "Boxing cơ bản - Lớp A",
      "courseName": "Boxing cơ bản",
      "coachName": "Nguyễn Văn Minh",
      "facilityName": "Phòng Boxing 1",
      "startDate": "2026-10-01",
      "endDate": "2026-11-07",
      "totalSessions": 12,
      "price": 1500000
    },
    "valid": true,
    "subtotal": 1500000,
    "membershipDiscount": 0,
    "couponDiscount": 150000,
    "total": 1350000
  }
]
```

Khi checkout, snapshot được lưu nguyên vào hóa đơn và không đổi nữa: sau này đổi tên sân hay giá gói, hóa đơn cũ vẫn in đúng như lúc mua.

```
$ORDER
{
  "id": UUID,
  "orderNumber": string,
  "account": $PERSON | null,
  "guestName": string | null,
  "guestPhone": string | null,
  "createdBy": $PERSON | null,
  "status": PAID | PARTIALLY_REFUNDED | REFUNDED,
  "paymentMethod": WALLET | CASH | CARD | TRANSFER,
  "coupon": { "code": string, "discount": Money } | null,
  "subtotal": Money,
  "membershipDiscount": Money,
  "couponDiscount": Money,
  "totalAmount": Money,
  "refundedAmount": Money,
  "items": [
    {
      "id": UUID,
      "lineNumber": int,
      "type": FACILITY_BOOKING | FACILITY_PACKAGE | COURSE_ENROLLMENT | MEMBERSHIP,
      "snapshot": object,
      "subtotal": Money,
      "membershipDiscount": Money,
      "couponDiscount": Money,
      "totalAmount": Money,
      "refundedAmount": Money,
      "refId": UUID
    }
  ],
  "paidAt": DateTime
}
```

- `refId`: id booking / facility package / enrollment / membership tương ứng với dòng

- POST /api/v1/checkout/quote:
  - role: member, receptionist
  - Tính giá và kiểm tra đơn, không ghi gì
  - request:

  ```
  {
    "buyer"?: $BUYER,
    "items": [$CHECKOUT_ITEM_INPUT],
    "couponCode"?: string
  }
  ```
  - response: `$QUOTE`

- POST /api/v1/checkout:
  - role: member, receptionist
  - Tính lại dưới lock; tổng khác `expectedTotal` → 409 `PRICE_CHANGED` kèm `$QUOTE` mới
  - `paymentMethod`: member chỉ `WALLET`; receptionist `CASH | CARD | TRANSFER`
  - request:

  ```
  {
    "buyer"?: $BUYER,
    "items": [$CHECKOUT_ITEM_INPUT],
    "couponCode"?: string,
    "paymentMethod": WALLET | CASH | CARD | TRANSFER,
    "expectedTotal": Money,
    "idempotencyKey": string
  }
  ```
  - response: `$ORDER`
  - lỗi: 422 `CART_EMPTY`, 409 `CART_ITEM_INVALID` (kèm `$QUOTE`), 409 `INSUFFICIENT_BALANCE`, 409 `COUPON_INVALID`, 409 `PRICE_CHANGED`, 409 `IDEMPOTENCY_CONFLICT`

- GET /api/v1/me/orders:
  - role: member
  - query: `from`, `to`, `page`, `limit`
  - response: `{ "items": [$ORDER], ... }`

- GET /api/v1/orders:
  - role: receptionist, manager
  - query: `accountId`, `guestPhone`, `status`, `orderNumber`, `from`, `to`, `page`, `limit`
  - response: `{ "items": [$ORDER], ... }`

- GET /api/v1/orders/{id}:
  - role: owner, receptionist, manager
  - response: `$ORDER`

- GET /api/v1/orders/{id}/invoice:
  - role: owner, receptionist, manager
  - response: file PDF, dựng từ snapshot lúc mua

## Wallet

```
$WALLET_TX
{
  "id": UUID,
  "transactionCode": string,
  "type": TOP_UP | PAYMENT | REFUND,
  "amount": Money,
  "balanceAfter": Money,
  "orderId": UUID | null,
  "orderItemId": UUID | null,
  "source": BANK_TRANSFER | COUNTER | null,
  "method": CASH | CARD | TRANSFER | null,
  "createdBy": $PERSON | null,
  "description": string | null,
  "createdAt": DateTime
}

$TOP_UP
{
  "id": UUID,
  "amount": Money,
  "paymentCode": string,
  "status": PENDING | SUCCESS | EXPIRED,
  "bankAccount": {
    "bankCode": string,
    "accountNumber": string,
    "accountName": string
  },
  "transferContent": string,
  "qrImageUrl": string,
  "expiresAt": DateTime,
  "completedAt": DateTime | null,
  "createdAt": DateTime
}
```

- `source` và `method` chỉ có với `TOP_UP`: `BANK_TRANSFER` (qua SePay, kể cả Manager gán; `method = TRANSFER`) hoặc `COUNTER` (nạp tại quầy; `method` là phương thức lễ tân thu)
- `transferContent` = `paymentCode`; `qrImageUrl` = `https://qr.sepay.vn/img?acc={accountNumber}&bank={bankCode}&amount={amount}&des={paymentCode}`
- `EXPIRED` chỉ là hết thời gian hiển thị mã; tiền đến sau đó vẫn được cộng nếu đúng mã và số tiền

- GET /api/v1/me/wallet:
  - role: member
  - query: `type`, `page`, `limit`
  - response:

  ```
  {
    "balance": Money,
    "transactions": {
      "items": [$WALLET_TX],
      "page": int,
      "limit": int,
      "total": int
    }
  }
  ```

- GET /api/v1/users/{id}/wallet:
  - role: receptionist, manager
  - response: như trên

- POST /api/v1/wallet/top-ups:
  - role: member
  - `amount` ≥ `topUpMinAmount`; mỗi member tối đa 3 yêu cầu PENDING còn hạn, quá → 409 `TOO_MANY_PENDING_TOP_UPS`
  - request:

  ```
  {
    "amount": Money
  }
  ```
  - response: `$TOP_UP`

- GET /api/v1/me/wallet/top-ups:
  - role: member
  - query: `status`, `page`, `limit`
  - response: `{ "items": [$TOP_UP], ... }`

- GET /api/v1/wallet/top-ups/{id}:
  - role: owner
  - FE gọi lại mỗi 3 giây khi đang ở trang QR cho tới khi `SUCCESS` hoặc rời trang
  - response: `$TOP_UP`

- POST /api/v1/users/{id}/wallet/top-ups:
  - role: receptionist
  - Nạp tại quầy
  - request:

  ```
  {
    "amount": Money,
    "method": CASH | CARD | TRANSFER,
    "note"?: string,
    "idempotencyKey": string
  }
  ```
  - response: `$WALLET_TX`

## Payment (SePay)

- POST /api/v1/payments/sepay/webhook:
  - public; xác thực header `Authorization: Apikey {SEPAY_WEBHOOK_API_KEY}`; sai → 401
  - Không qua kiểm tra `Origin` và rate limit
  - request (SePay gửi):

  ```
  {
    "id": int,
    "gateway": string,
    "transactionDate": string,
    "accountNumber": string,
    "subAccount": string | null,
    "code": string | null,
    "content": string,
    "transferType": "in" | "out",
    "transferAmount": int,
    "accumulated": int,
    "referenceCode": string,
    "description": string
  }
  ```
  - Xử lý theo BR_3.6 (thuật toán khớp: `db.v5.notes.md` §5)
  - response: HTTP 200, body `{ "success": true }` (không theo định dạng response chung). Lỗi hệ thống → 5xx để SePay gửi lại

```
$BANK_TX
{
  "id": UUID,
  "sepayId": int,
  "bankName": string,
  "accountNumber": string,
  "amount": Money,
  "content": string,
  "paymentCode": string | null,
  "referenceCode": string | null,
  "transactionDate": DateTime,
  "status": MATCHED | UNMATCHED | RESOLVED | IGNORED,
  "topUp": { "id": UUID, "account": $PERSON } | null,
  "resolvedAccount": $PERSON | null,
  "handledBy": $PERSON | null,
  "handledAt": DateTime | null,
  "note": string | null
}
```

- GET /api/v1/bank-transactions:
  - role: manager
  - query: `status`, `from`, `to`, `q` (nội dung / mã tham chiếu), `page`, `limit`
  - response: `{ "items": [$BANK_TX], ... }`

- POST /api/v1/bank-transactions/{id}/resolve:
  - role: manager
  - Chỉ `UNMATCHED`; cộng đúng `amount` vào ví member, ghi audit, thông báo member
  - request:

  ```
  {
    "accountId": UUID,
    "note": string
  }
  ```
  - response: `$BANK_TX`

- POST /api/v1/bank-transactions/{id}/ignore:
  - role: manager
  - Chỉ `UNMATCHED`; ghi audit
  - request:

  ```
  {
    "note": string
  }
  ```
  - response: `$BANK_TX`

- GET /api/v1/bank-transactions/reconciliation:
  - role: manager
  - query: `from`, `to` (tối đa 31 ngày)
  - So tổng giao dịch tiền vào theo ngày (giờ VN) của tài khoản nhận tiền: số liệu SePay lấy qua `GET https://my.sepay.vn/userapi/transactions/list`, số liệu hệ thống từ các giao dịch ngân hàng đã ghi nhận
  - response:

  ```
  {
    "days": [
      {
        "date": Date,
        "sepay": { "count": int, "amount": Money },
        "system": { "count": int, "amount": Money },
        "matched": bool,
        "missingSepayIds": [int]
      }
    ]
  }
  ```
  - `missingSepayIds`: giao dịch có trên SePay nhưng hệ thống chưa ghi nhận (job `sepay-sync` sẽ bổ sung)
  - SePay API lỗi hoặc quá giới hạn tần suất → 503 `UPSTREAM_UNAVAILABLE`

## Coupon

```
$COUPON
{
  "id": UUID,
  "code": string,
  "name": string,
  "discountType": PERCENT | FIXED,
  "discountValue": int,
  "maxDiscount": Money | null,
  "validFrom": DateTime,
  "validTo": DateTime,
  "maxUses": int | null,
  "maxUsesPerUser": int,
  "minOrderAmount": Money | null,
  "applicableTypes": [FACILITY_BOOKING | FACILITY_PACKAGE | COURSE_ENROLLMENT | MEMBERSHIP] | null,
  "isActive": bool,
  "usedCount": int
}
```

- `applicableTypes = null`: áp mọi loại dịch vụ. `maxUses = null`: không giới hạn
- `code` được trim và chuyển chữ hoa khi lưu và khi tra cứu
- Áp mã cho đơn: qua `couponCode` của `POST /checkout/quote` và `POST /checkout`

- GET /api/v1/coupons:
  - role: manager
  - response: `[$COUPON]`

- POST /api/v1/coupons, PATCH /api/v1/coupons/{id}, DELETE /api/v1/coupons/{id}:
  - role: manager
  - request: các field của `$COUPON` trừ `id`, `usedCount`
  - response: `$COUPON`

## Report

- role: manager (tất cả)

- GET /api/v1/reports/overview:
  - response:

  ```
  {
    "revenueToday": Money,
    "newMembersToday": int,
    "bookingsToday": int,
    "ongoingClasses": int,
    "activeMemberships": int,
    "unmatchedBankTransactions": int
  }
  ```

- GET /api/v1/reports/revenue:
  - query: `from`, `to`, `granularity` (day | week | month)
  - Không tính nạp ví
  - response:

  ```
  {
    "buckets": [
      {
        "period": string,
        "revenue": Money,
        "refunds": Money,
        "net": Money,
        "byType": {
          "MEMBERSHIP": Money,
          "FACILITY_BOOKING": Money,
          "FACILITY_PACKAGE": Money,
          "COURSE_ENROLLMENT": Money
        },
        "byPaymentMethod": {
          "WALLET": Money,
          "CASH": Money,
          "CARD": Money,
          "TRANSFER": Money
        }
      }
    ]
  }
  ```

- GET /api/v1/reports/wallet:
  - query: `from`, `to`, `granularity` (day | week | month)
  - response:

  ```
  {
    "totalBalance": Money,
    "unmatched": { "count": int, "amount": Money },
    "buckets": [
      {
        "period": string,
        "topUpBankTransfer": Money,
        "topUpCounter": { "CASH": Money, "CARD": Money, "TRANSFER": Money },
        "payments": Money,
        "refunds": Money,
        "netChange": Money
      }
    ]
  }
  ```
  - `totalBalance`: tổng số dư ví của mọi member tại thời điểm gọi
  - `netChange` = `topUpBankTransfer` + tổng `topUpCounter` − `payments` + `refunds`

- GET /api/v1/reports/members:
  - query: `from`, `to`
  - response:

  ```
  {
    "total": int,
    "byStatus": { "ACTIVE": int, "INACTIVE": int, "BANNED": int },
    "activeMemberships": int,
    "expiringSoon": int,
    "renewalRate": float,
    "newByPeriod": [ { "period": string, "count": int } ]
  }
  ```

- GET /api/v1/reports/facilities:
  - query: `from`, `to`
  - response:

  ```
  {
    "utilizationRate": float,
    "byFacility": [
      {
        "facilityId": UUID,
        "name": string,
        "bookings": int,
        "occupancyPct": float,
        "revenue": Money
      }
    ]
  }
  ```

- GET /api/v1/reports/courses:
  - query: `from`, `to`
  - response:

  ```
  {
    "byClass": [
      {
        "classId": UUID,
        "name": string,
        "enrolled": int,
        "max": int,
        "fillRate": float,
        "attendanceRate": float
      }
    ],
    "topCoaches": [ { "coach": $PERSON, "students": int } ]
  }
  ```
  - `attendanceRate`: số lượt PRESENT + LATE / tổng lượt điểm danh của lớp

- GET /api/v1/reports/export:
  - query: `report` (overview | revenue | wallet | members | facilities | courses), `format` (pdf | xlsx), + filter của report
  - response: file

## Support

```
$SUPPORT
{
  "id": UUID,
  "account": $PERSON,
  "category": ACCOUNT | MEMBERSHIP | BOOKING | CLASS | PAYMENT | OTHER,
  "subject": string,
  "description": string,
  "status": OPEN | IN_PROGRESS | RESOLVED | CLOSED,
  "resolutionNote": string | null,
  "handledBy": $PERSON | null,
  "resolvedAt": DateTime | null,
  "createdAt": DateTime,
  "updatedAt": DateTime
}
```

- POST /api/v1/support-requests:
  - role: member
  - request:

  ```
  {
    "category": ACCOUNT | MEMBERSHIP | BOOKING | CLASS | PAYMENT | OTHER,
    "subject": string,
    "description": string
  }
  ```
  - response: `$SUPPORT`

- GET /api/v1/me/support-requests:
  - role: member
  - response: `[$SUPPORT]`

- GET /api/v1/support-requests:
  - role: receptionist, manager
  - query: `status`, `category`, `accountId`, `q`, `page`, `limit`
  - response: `{ "items": [$SUPPORT], ... }`

- GET /api/v1/support-requests/{id}:
  - role: owner, receptionist, manager
  - response: `$SUPPORT`

- PATCH /api/v1/support-requests/{id}:
  - role: receptionist, manager
  - Chỉ tiến theo thứ tự OPEN → IN_PROGRESS → RESOLVED → CLOSED; thông báo member
  - request:

  ```
  {
    "status"?: IN_PROGRESS | RESOLVED | CLOSED,
    "resolutionNote"?: string
  }
  ```
  - response: `$SUPPORT`

## Notification

- Thông báo chỉ do hệ thống tạo. Không có API gửi tay (trừ thông báo lớp của Coach ở Training)

- GET /api/v1/me/notifications:
  - query: `unreadOnly`, `cursor`, `limit`
  - response:

  ```
  {
    "items": [
      {
        "id": UUID,
        "type": PAYMENT | MEMBERSHIP | BOOKING | CLASS | TRAINING | SUPPORT | SYSTEM,
        "title": string,
        "message": string,
        "referenceType": string | null,
        "referenceId": UUID | null,
        "readAt": DateTime | null,
        "createdAt": DateTime
      }
    ],
    "nextCursor": string | null,
    "unreadCount": int
  }
  ```
  - `referenceType` / `referenceId` dùng để điều hướng (ví dụ `ORDER`, `BOOKING`, `CLASS`, `TOP_UP`, `SUPPORT_REQUEST`, `BANK_TRANSACTION`)

- PATCH /api/v1/me/notifications/{id}/read
- POST /api/v1/me/notifications/read-all

## Training

```
$ATTENDANCE
{
  "account": $PERSON,
  "status": PRESENT | ABSENT | LATE | null,
  "note": string | null,
  "updatedBy": $PERSON | null,
  "updatedAt": DateTime | null
}

$SESSION_NOTE
{
  "title": string,
  "content": string,
  "attachments": [string],
  "updatedAt": DateTime
}

$EVALUATION
{
  "id": UUID,
  "session": $SESSION,
  "account": $PERSON,
  "coach": $PERSON,
  "rating": int,
  "comment": string | null,
  "createdAt": DateTime
}
```

- `rating`: 1–5

- POST /api/v1/checkins:
  - role: receptionist
  - Cần booking hôm nay / buổi học hôm nay / membership có quyền vào gym; không có → 409 `CHECKIN_NOT_ALLOWED`
  - request:

  ```
  {
    "accountId": UUID
  }
  ```

- GET /api/v1/me/checkins:
  - role: member
  - query: `from`, `to`

- GET /api/v1/sessions/{id}/attendance:
  - role: coach (lớp mình), manager
  - response: `[$ATTENDANCE]`

- PUT /api/v1/sessions/{id}/attendance:
  - role: coach (lớp mình), manager
  - Mỗi bản ghi thay đổi được ghi audit log
  - request:

  ```
  {
    "records": [
      {
        "accountId": UUID,
        "status": PRESENT | ABSENT | LATE,
        "note"?: string
      }
    ]
  }
  ```

- GET /api/v1/me/attendance:
  - role: member
  - query: `classId`
  - response: `[ { "session": $SESSION, "status": PRESENT | ABSENT | LATE | null, "note": string | null } ]`

- GET /api/v1/sessions/{id}/notes:
  - role: coach (lớp mình), manager, member đang học lớp
  - response: `$SESSION_NOTE | null`

- PUT /api/v1/sessions/{id}/notes:
  - role: coach (lớp mình)
  - 1 note / buổi
  - request:

  ```
  {
    "title": string,
    "content": string,
    "attachments"?: [string]
  }
  ```

- GET /api/v1/sessions/{id}/evaluations:
  - role: coach (lớp mình), manager
  - response: `[$EVALUATION]`

- POST /api/v1/sessions/{id}/evaluations:
  - role: coach (lớp mình)
  - 1 đánh giá / (buổi, học viên) chưa xóa
  - request:

  ```
  {
    "accountId": UUID,
    "rating": int,
    "comment"?: string
  }
  ```
  - response: `$EVALUATION`

- PATCH /api/v1/evaluations/{id}, DELETE /api/v1/evaluations/{id}:
  - role: coach tác giả, manager

- GET /api/v1/me/evaluations:
  - role: member
  - query: `classId`
  - response: `[$EVALUATION]`

- POST /api/v1/classes/{id}/announcements:
  - role: coach (lớp mình), manager
  - Tạo thông báo `TRAINING` cho học viên đang học
  - request:

  ```
  {
    "title": string,
    "body": string
  }
  ```
  - response: `{ "recipients": int }`

## Cron

- Header `Authorization: Bearer {CRON_SECRET}`; sai → 401. Không qua kiểm tra `Origin` và rate limit
- Response: `{ "processed": int, "remaining": int }`

- POST /api/v1/cron/memberships:
  - BR_1.6

- POST /api/v1/cron/class-min-students:
  - BR_2.20

- POST /api/v1/cron/reminders:
  - detail.v4 mục 4.0.5

- POST /api/v1/cron/cleanup:
  - detail.v4 mục 4.0.2 (nhóm 🟡) và 4.0.5; gửi lại email chưa gửi được

- POST /api/v1/cron/sepay-sync:
  - BR_3.6 (thuật toán: `db.v5.notes.md` §5)

- POST /api/v1/cron/attendance-defaults *(F4)*:
  - BR_4.2

