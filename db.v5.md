Enum role {
  MANAGER
  COACH
  MEMBER
  RECEPTIONIST
}

Enum account_status {
  ACTIVE
  INACTIVE
  BANNED
}

Enum gender {
  MALE
  FEMALE
  OTHER
}

Enum otp_purpose {
  REGISTER
  PASSWORD_RESET
}

Enum specialization_status {
  PENDING
  APPROVED
  REJECTED
}

Enum facility_type {
  GYM
  COURT
  ROOM
  FIELD
}

Enum booking_status {
  CONFIRMED
  CANCELLED
}

Enum booking_benefit {
  NONE
  DISCOUNT
  GYM_ACCESS
  FREE_SLOT
}

Enum facility_package_status {
  ACTIVE
  CANCELLED
}

Enum class_status {
  DRAFT
  PENDING_APPROVAL
  OPEN
  CANCELLED
}

Enum coach_registration_status {
  PENDING
  APPROVED
  REJECTED
}

Enum coach_registration_source {
  COACH_REGISTERED
  MANAGER_ASSIGNED
}

Enum session_status {
  SCHEDULED
  CANCELLED
}

Enum enrollment_status {
  ENROLLED
  CANCELLED
}

Enum membership_status {
  ACTIVE
  EXPIRED
  CANCELLED
}

Enum order_item_type {
  MEMBERSHIP
  FACILITY_BOOKING
  FACILITY_PACKAGE
  COURSE_ENROLLMENT
}

Enum payment_method {
  WALLET
  CASH
  CARD
  TRANSFER
}

Enum order_status {
  PAID
  PARTIALLY_REFUNDED
  REFUNDED
}

Enum transaction_type {
  TOP_UP
  PAYMENT
  REFUND
}

Enum top_up_status {
  PENDING
  SUCCESS
  EXPIRED
}

Enum bank_transaction_status {
  MATCHED
  UNMATCHED
  RESOLVED
  IGNORED
}

Enum discount_type {
  PERCENT
  FIXED
}

Enum attendance_status {
  PRESENT
  ABSENT
  LATE
}

Enum notification_type {
  PAYMENT
  MEMBERSHIP
  BOOKING
  CLASS
  TRAINING
  SUPPORT
  SYSTEM
}

Enum support_request_category {
  ACCOUNT
  MEMBERSHIP
  BOOKING
  CLASS
  PAYMENT
  OTHER
}

Enum support_request_status {
  OPEN
  IN_PROGRESS
  RESOLVED
  CLOSED
}

Table accounts {
  id                  uuid           [pk, default: `gen_random_uuid()`]
  email               varchar(255)   [unique, not null]
  phone               varchar(20)    [unique]
  password_hash       varchar(255)   [not null]
  full_name           varchar(255)   [not null]
  avatar_url          text
  role                role           [not null, default: 'MEMBER']
  status              account_status [not null, default: 'ACTIVE']
  email_verified_at   timestamptz
  password_changed_at timestamptz    [not null, default: `now()`]
  date_of_birth       date
  gender              gender
  address             varchar(500)
  created_at          timestamptz    [not null, default: `now()`]
  updated_at          timestamptz    [not null, default: `now()`]

  indexes {
    (role, status) [name: 'idx_accounts_role_status']
  }
}

Table member_profile {
  account_id        uuid          [pk]
  emergency_contact varchar(255)
  fitness_goals     text
  health_notes      text
  wallet_balance    numeric(12,0) [not null, default: 0]
  created_at        timestamptz   [not null, default: `now()`]
  updated_at        timestamptz   [not null, default: `now()`]

  checks {
    `wallet_balance >= 0` [name: 'ck_member_profile_wallet']
  }
}

Ref: accounts.id - member_profile.account_id [delete: restrict]

Table coach_profile {
  account_id      uuid        [pk]
  bio             text
  experience      text
  certifications  text
  cover_image_url text
  created_at      timestamptz [not null, default: `now()`]
  updated_at      timestamptz [not null, default: `now()`]
}

Ref: accounts.id - coach_profile.account_id [delete: restrict]

Table receptionist_profile {
  account_id  uuid        [pk]
  staff_notes text
  created_at  timestamptz [not null, default: `now()`]
  updated_at  timestamptz [not null, default: `now()`]
}

Ref: accounts.id - receptionist_profile.account_id [delete: restrict]

Table manager_profile {
  account_id  uuid        [pk]
  staff_notes text
  created_at  timestamptz [not null, default: `now()`]
  updated_at  timestamptz [not null, default: `now()`]
}

Ref: accounts.id - manager_profile.account_id [delete: restrict]

Table refresh_tokens {
  id         uuid         [pk, default: `gen_random_uuid()`]
  account_id uuid         [not null]
  token_hash varchar(64)  [unique, not null]
  expires_at timestamptz  [not null]
  revoked_at timestamptz
  user_agent varchar(500)
  ip         varchar(45)
  created_at timestamptz  [not null, default: `now()`]

  indexes {
    account_id [name: 'idx_refresh_tokens_account']
    expires_at [name: 'idx_refresh_tokens_expires']
  }
}

Ref: refresh_tokens.account_id > accounts.id [delete: restrict]

Table otp_codes {
  id          uuid         [pk, default: `gen_random_uuid()`]
  email       varchar(255) [not null]
  purpose     otp_purpose  [not null]
  code_hash   varchar(64)  [not null]
  attempts    int          [not null, default: 0]
  expires_at  timestamptz  [not null]
  consumed_at timestamptz
  created_at  timestamptz  [not null, default: `now()`]

  indexes {
    (email, purpose) [name: 'idx_otp_email_purpose']
  }

  checks {
    `attempts >= 0` [name: 'ck_otp_attempts']
  }
}

Table bank_transactions {
  id                  uuid                    [pk, default: `gen_random_uuid()`]
  sepay_id            bigint                  [unique, not null]
  bank_name           varchar(100)            [not null]
  account_number      varchar(50)             [not null]
  amount              numeric(12,0)           [not null]
  content             text                    [not null]
  payment_code        varchar(50)
  reference_code      varchar(100)
  transaction_date    timestamptz             [not null]
  status              bank_transaction_status [not null]
  resolved_account_id uuid
  handled_by          uuid
  handled_at          timestamptz
  note                varchar(500)
  raw_payload         jsonb                   [not null]
  created_at          timestamptz             [not null, default: `now()`]
  updated_at          timestamptz             [not null, default: `now()`]

  indexes {
    (status, transaction_date) [name: 'idx_bank_tx_status_date']
    payment_code [name: 'idx_bank_tx_payment_code']
  }

  checks {
    `amount > 0` [name: 'ck_bank_tx_amount']
  }
}

Ref: bank_transactions.resolved_account_id > member_profile.account_id [delete: restrict]
Ref: bank_transactions.handled_by > accounts.id [delete: restrict]

Table wallet_top_ups {
  id                  uuid          [pk, default: `gen_random_uuid()`]
  account_id          uuid          [not null]
  payment_code        varchar(20)   [unique, not null]
  amount              numeric(12,0) [not null]
  status              top_up_status [not null, default: 'PENDING']
  expires_at          timestamptz   [not null]
  bank_transaction_id uuid          [unique]
  completed_at        timestamptz
  created_at          timestamptz   [not null, default: `now()`]
  updated_at          timestamptz   [not null, default: `now()`]

  indexes {
    (account_id, created_at) [name: 'idx_top_ups_account_date']
    (status, expires_at) [name: 'idx_top_ups_status_expires']
  }

  checks {
    `amount > 0` [name: 'ck_top_ups_amount']
    `(status = 'SUCCESS') = (bank_transaction_id IS NOT NULL AND completed_at IS NOT NULL)` [name: 'ck_top_ups_success']
  }
}

Ref: wallet_top_ups.account_id > member_profile.account_id [delete: restrict]
Ref: bank_transactions.id - wallet_top_ups.bank_transaction_id [delete: restrict]

Table wallet_transactions {
  id                  uuid             [pk, default: `gen_random_uuid()`]
  account_id          uuid             [not null]
  transaction_code    varchar(50)      [unique, not null]
  idempotency_key     varchar(150)     [unique, not null]
  type                transaction_type [not null]
  top_up_method       payment_method
  amount              numeric(12,0)    [not null]
  balance_after       numeric(12,0)    [not null]
  order_id            uuid
  order_item_id       uuid
  bank_transaction_id uuid             [unique]
  created_by          uuid
  description         varchar(500)
  created_at          timestamptz      [not null, default: `now()`]

  indexes {
    (account_id, created_at) [name: 'idx_wtx_account_date']
    (type, created_at) [name: 'idx_wtx_type_date']
    order_id [name: 'idx_wtx_order']
    order_item_id [name: 'idx_wtx_order_item']
  }

  checks {
    `amount > 0 AND balance_after >= 0` [name: 'ck_wtx_amounts']
    `(type = 'TOP_UP') = (top_up_method IS NOT NULL) AND (top_up_method IS NULL OR top_up_method <> 'WALLET')` [name: 'ck_wtx_top_up_method']
    `bank_transaction_id IS NULL OR top_up_method = 'TRANSFER'` [name: 'ck_wtx_bank_transfer']
    `(type = 'TOP_UP' AND order_id IS NULL AND order_item_id IS NULL) OR (type = 'PAYMENT' AND order_id IS NOT NULL AND order_item_id IS NULL AND bank_transaction_id IS NULL) OR (type = 'REFUND' AND order_id IS NOT NULL AND order_item_id IS NOT NULL AND bank_transaction_id IS NULL)` [name: 'ck_wtx_type_refs']
  }
}

Ref: wallet_transactions.account_id > member_profile.account_id [delete: restrict]
Ref: wallet_transactions.order_id > orders.id [delete: restrict]
Ref: wallet_transactions.order_item_id > order_items.id [delete: restrict]
Ref: bank_transactions.id - wallet_transactions.bank_transaction_id [delete: restrict]
Ref: wallet_transactions.created_by > accounts.id [delete: restrict]

Table sports {
  id          uuid         [pk, default: `gen_random_uuid()`]
  name        varchar(100) [not null]
  description text
  icon_url    text
  is_active   boolean      [not null, default: true]
  deleted_at  timestamptz
  created_at  timestamptz  [not null, default: `now()`]
  updated_at  timestamptz  [not null, default: `now()`]

  Note: 'Partial unique: uq_sports_name (name) WHERE deleted_at IS NULL'
}

Table coach_specializations {
  id          uuid                  [pk, default: `gen_random_uuid()`]
  coach_id    uuid                  [not null]
  sport_id    uuid                  [not null]
  status      specialization_status [not null, default: 'PENDING']
  review_note varchar(500)
  reviewed_by uuid
  reviewed_at timestamptz
  created_at  timestamptz           [not null, default: `now()`]

  indexes {
    (coach_id, sport_id) [name: 'idx_coach_spec_coach_sport']
    (status, created_at) [name: 'idx_coach_spec_status_date']
  }

  Note: 'Partial unique: uq_coach_spec_active (coach_id, sport_id) WHERE status IN (PENDING, APPROVED)'
}

Ref: coach_specializations.coach_id > accounts.id [delete: restrict]
Ref: coach_specializations.sport_id > sports.id [delete: restrict]
Ref: coach_specializations.reviewed_by > accounts.id [delete: restrict]

Table facilities {
  id                uuid          [pk, default: `gen_random_uuid()`]
  name              varchar(100)  [not null]
  type              facility_type [not null]
  capacity_per_slot int           [not null]
  price_per_slot    numeric(12,0) [not null]
  is_active         boolean       [not null, default: true]
  description       text
  deleted_at        timestamptz
  created_at        timestamptz   [not null, default: `now()`]
  updated_at        timestamptz   [not null, default: `now()`]

  checks {
    `capacity_per_slot >= 1 AND price_per_slot >= 0` [name: 'ck_facilities_values']
  }

  Note: 'Partial unique: uq_facilities_name (name) WHERE deleted_at IS NULL'
}

Table facility_sports {
  facility_id uuid [not null]
  sport_id    uuid [not null]

  indexes {
    (facility_id, sport_id) [pk]
    sport_id [name: 'idx_facility_sports_sport']
  }
}

Ref: facility_sports.facility_id > facilities.id [delete: restrict]
Ref: facility_sports.sport_id > sports.id [delete: restrict]

Table facility_maintenances {
  id          uuid         [pk, default: `gen_random_uuid()`]
  facility_id uuid         [not null]
  start_at    timestamptz  [not null]
  end_at      timestamptz  [not null]
  reason      varchar(500) [not null]
  created_by  uuid         [not null]
  deleted_at  timestamptz
  created_at  timestamptz  [not null, default: `now()`]
  updated_at  timestamptz  [not null, default: `now()`]

  indexes {
    (facility_id, start_at, end_at) [name: 'idx_fm_facility_range']
  }

  checks {
    `end_at > start_at` [name: 'ck_fm_range']
  }

  Note: 'Exclusion: ex_maintenance_overlap (facility_id =, tstzrange(start_at, end_at) &&) WHERE deleted_at IS NULL'
}

Ref: facility_maintenances.facility_id > facilities.id [delete: restrict]
Ref: facility_maintenances.created_by > accounts.id [delete: restrict]

Table facility_packages {
  id            uuid                    [pk, default: `gen_random_uuid()`]
  account_id    uuid                    [not null]
  facility_id   uuid                    [not null]
  order_item_id uuid                    [unique, not null]
  days_of_week  "int[]"                 [not null]
  start_time    time                    [not null]
  end_time      time                    [not null]
  start_date    date                    [not null]
  end_date      date                    [not null]
  unit_price    numeric(12,0)           [not null]
  status        facility_package_status [not null, default: 'ACTIVE']
  cancelled_at  timestamptz
  created_at    timestamptz             [not null, default: `now()`]

  indexes {
    (account_id, status) [name: 'idx_fp_account_status']
  }

  checks {
    `unit_price >= 0 AND end_date >= start_date AND end_time > start_time` [name: 'ck_fp_values']
    `valid_days_of_week(days_of_week)` [name: 'ck_fp_days']
  }
}

Ref: facility_packages.account_id > accounts.id [delete: restrict]
Ref: facility_packages.facility_id > facilities.id [delete: restrict]
Ref: order_items.id - facility_packages.order_item_id [delete: restrict]

Table facility_bookings {
  id            uuid            [pk, default: `gen_random_uuid()`]
  facility_id   uuid            [not null]
  account_id    uuid
  order_item_id uuid            [not null]
  package_id    uuid
  booking_date  date            [not null]
  start_time    time            [not null]
  end_time      time            [not null]
  unit_price    numeric(12,0)   [not null]
  benefit       booking_benefit [not null, default: 'NONE']
  status        booking_status  [not null, default: 'CONFIRMED']
  cancelled_at  timestamptz
  created_at    timestamptz     [not null, default: `now()`]

  indexes {
    (facility_id, booking_date, start_time) [name: 'idx_booking_facility_date_time']
    (account_id, booking_date) [name: 'idx_booking_account_date']
    package_id [name: 'idx_booking_package']
    order_item_id [name: 'idx_booking_order_item']
  }

  checks {
    `unit_price >= 0 AND end_time > start_time` [name: 'ck_booking_values']
    `account_id IS NOT NULL OR (package_id IS NULL AND benefit = 'NONE')` [name: 'ck_booking_guest']
  }

  Note: 'Partial unique: uq_booking_single_item (order_item_id) WHERE package_id IS NULL'
}

Ref: facility_bookings.facility_id > facilities.id [delete: restrict]
Ref: facility_bookings.account_id > accounts.id [delete: restrict]
Ref: facility_bookings.order_item_id > order_items.id [delete: restrict]
Ref: facility_bookings.package_id > facility_packages.id [delete: restrict]

Table courses {
  id             uuid          [pk, default: `gen_random_uuid()`]
  name           varchar(255)  [not null]
  sport_id       uuid          [not null]
  description    text
  total_sessions int           [not null]
  price          numeric(12,0) [not null]
  thumbnail_url  text
  deleted_at     timestamptz
  created_at     timestamptz   [not null, default: `now()`]
  updated_at     timestamptz   [not null, default: `now()`]

  checks {
    `total_sessions >= 1 AND price >= 0` [name: 'ck_courses_values']
  }
}

Ref: courses.sport_id > sports.id [delete: restrict]

Table classes {
  id                    uuid         [pk, default: `gen_random_uuid()`]
  course_id             uuid         [not null]
  coach_id              uuid
  facility_id           uuid         [not null]
  name                  varchar(100) [not null]
  min_students          int          [not null, default: 1]
  min_students_override boolean      [not null, default: false]
  max_students          int          [not null]
  start_date            date
  end_date              date
  weekly_schedule       jsonb        [not null]
  status                class_status [not null, default: 'DRAFT']
  approved_by           uuid
  approved_at           timestamptz
  cancel_reason         varchar(500)
  deleted_at            timestamptz
  created_at            timestamptz  [not null, default: `now()`]
  updated_at            timestamptz  [not null, default: `now()`]

  indexes {
    (course_id, status) [name: 'idx_classes_course_status']
    coach_id [name: 'idx_classes_coach']
    (status, start_date, end_date) [name: 'idx_classes_status_dates']
  }

  checks {
    `min_students >= 1 AND max_students >= min_students` [name: 'ck_classes_students']
    `(start_date IS NULL AND end_date IS NULL) OR (start_date IS NOT NULL AND end_date IS NOT NULL AND end_date >= start_date)` [name: 'ck_classes_dates']
    `status <> 'OPEN' OR (coach_id IS NOT NULL AND start_date IS NOT NULL)` [name: 'ck_classes_open']
  }
}

Ref: classes.course_id > courses.id [delete: restrict]
Ref: classes.coach_id > accounts.id [delete: restrict]
Ref: classes.facility_id > facilities.id [delete: restrict]
Ref: classes.approved_by > accounts.id [delete: restrict]

Table class_coach_registrations {
  id          uuid                      [pk, default: `gen_random_uuid()`]
  class_id    uuid                      [not null]
  coach_id    uuid                      [not null]
  source      coach_registration_source [not null]
  status      coach_registration_status [not null, default: 'PENDING']
  reviewed_by uuid
  reviewed_at timestamptz
  created_at  timestamptz               [not null, default: `now()`]

  indexes {
    (class_id, coach_id) [name: 'idx_ccr_class_coach']
  }

  Note: 'Partial unique: uq_ccr_pending (class_id, coach_id) WHERE status = PENDING'
}

Ref: class_coach_registrations.class_id > classes.id [delete: restrict]
Ref: class_coach_registrations.coach_id > accounts.id [delete: restrict]
Ref: class_coach_registrations.reviewed_by > accounts.id [delete: restrict]

Table class_sessions {
  id               uuid           [pk, default: `gen_random_uuid()`]
  class_id         uuid           [not null]
  facility_id      uuid           [not null]
  session_number   int            [not null]
  session_date     date           [not null]
  start_time       time           [not null]
  end_time         time           [not null]
  status           session_status [not null, default: 'SCHEDULED']
  cancel_reason    varchar(500)
  note_title       varchar(255)
  note_content     text
  note_attachments jsonb
  note_updated_at  timestamptz
  created_at       timestamptz    [not null, default: `now()`]
  updated_at       timestamptz    [not null, default: `now()`]

  indexes {
    (class_id, session_number) [unique, name: 'uq_cs_class_session_num']
    (facility_id, session_date, start_time) [name: 'idx_cs_facility_date_time']
    (session_date, start_time) [name: 'idx_cs_date_time']
  }

  checks {
    `session_number >= 1 AND end_time > start_time` [name: 'ck_cs_values']
  }

  Note: 'Exclusion: ex_session_facility_overlap (facility_id =, tsrange(session_date + start_time, session_date + end_time) &&) WHERE status = SCHEDULED'
}

Ref: class_sessions.class_id > classes.id [delete: restrict]
Ref: class_sessions.facility_id > facilities.id [delete: restrict]

Table class_enrollments {
  id            uuid              [pk, default: `gen_random_uuid()`]
  class_id      uuid              [not null]
  account_id    uuid              [not null]
  order_item_id uuid              [unique, not null]
  status        enrollment_status [not null, default: 'ENROLLED']
  enrolled_at   timestamptz       [not null, default: `now()`]
  cancelled_at  timestamptz

  indexes {
    (class_id, account_id) [name: 'idx_enrollment_class_account']
    account_id [name: 'idx_enrollment_account']
  }

  Note: 'Partial unique: uq_enrollment_active (class_id, account_id) WHERE status = ENROLLED'
}

Ref: class_enrollments.class_id > classes.id [delete: restrict]
Ref: class_enrollments.account_id > accounts.id [delete: restrict]
Ref: order_items.id - class_enrollments.order_item_id [delete: restrict]

Table memberships {
  id                           uuid          [pk, default: `gen_random_uuid()`]
  name                         varchar(255)  [not null]
  description                  text
  price                        numeric(12,0) [not null]
  duration_days                int           [not null]
  gym_access                   boolean       [not null, default: false]
  booking_discount_pct         int           [not null, default: 0]
  class_discount_pct           int           [not null, default: 0]
  free_booking_slots_per_month int           [not null, default: 0]
  is_active                    boolean       [not null, default: true]
  deleted_at                   timestamptz
  created_at                   timestamptz   [not null, default: `now()`]
  updated_at                   timestamptz   [not null, default: `now()`]

  checks {
    `price >= 0 AND duration_days >= 1 AND free_booking_slots_per_month >= 0` [name: 'ck_memberships_values']
    `booking_discount_pct BETWEEN 0 AND 100 AND class_discount_pct BETWEEN 0 AND 100` [name: 'ck_memberships_pct']
  }
}

Table member_memberships {
  id           uuid              [pk, default: `gen_random_uuid()`]
  account_id   uuid              [not null]
  package_id   uuid              [not null]
  start_date   date              [not null]
  end_date     date              [not null]
  auto_renew   boolean           [not null, default: false]
  status       membership_status [not null, default: 'ACTIVE']
  cancelled_at timestamptz
  created_at   timestamptz       [not null, default: `now()`]
  updated_at   timestamptz       [not null, default: `now()`]

  indexes {
    (account_id, status) [name: 'idx_mm_account_status']
    (status, end_date) [name: 'idx_mm_status_end']
  }

  checks {
    `end_date > start_date` [name: 'ck_mm_dates']
  }

  Note: 'Partial unique: uq_mm_one_active (account_id) WHERE status = ACTIVE'
}

Ref: member_memberships.account_id > accounts.id [delete: restrict]
Ref: member_memberships.package_id > memberships.id [delete: restrict]

Table membership_orders {
  id                           uuid          [pk, default: `gen_random_uuid()`]
  membership_id                uuid          [not null]
  order_item_id                uuid          [unique, not null]
  period_start                 date          [not null]
  period_end                   date          [not null]
  gym_access                   boolean       [not null]
  booking_discount_pct         int           [not null]
  class_discount_pct           int           [not null]
  free_booking_slots_per_month int           [not null]
  created_at                   timestamptz   [not null, default: `now()`]

  indexes {
    (membership_id, period_start) [unique, name: 'uq_membership_order_period']
  }

  checks {
    `period_end > period_start` [name: 'ck_mo_period']
    `booking_discount_pct BETWEEN 0 AND 100 AND class_discount_pct BETWEEN 0 AND 100 AND free_booking_slots_per_month >= 0` [name: 'ck_mo_benefits']
  }

  Note: 'Exclusion: ex_membership_period_overlap (membership_id =, daterange(period_start, period_end) &&)'
}

Ref: membership_orders.membership_id > member_memberships.id [delete: restrict]
Ref: order_items.id - membership_orders.order_item_id [delete: restrict]

Table orders {
  id                         uuid           [pk, default: `gen_random_uuid()`]
  order_number               varchar(50)    [unique, not null]
  idempotency_key            varchar(150)   [unique, not null]
  account_id                 uuid
  created_by                 uuid
  invoice_snapshot           jsonb          [not null]
  subtotal                   numeric(12,0)  [not null]
  membership_discount_amount numeric(12,0)  [not null, default: 0]
  coupon_discount_amount     numeric(12,0)  [not null, default: 0]
  total_amount               numeric(12,0)  [not null]
  refunded_amount            numeric(12,0)  [not null, default: 0]
  coupon_id                  uuid
  payment_method             payment_method [not null]
  status                     order_status   [not null, default: 'PAID']
  guest_name                 varchar(255)
  guest_phone                varchar(20)
  created_at                 timestamptz    [not null, default: `now()`]
  updated_at                 timestamptz    [not null, default: `now()`]

  indexes {
    account_id [name: 'idx_orders_account']
    (created_at, status) [name: 'idx_orders_date_status']
    (coupon_id, account_id) [name: 'idx_orders_coupon_account']
    guest_phone [name: 'idx_orders_guest_phone']
  }

  checks {
    `subtotal >= 0 AND membership_discount_amount >= 0 AND coupon_discount_amount >= 0 AND total_amount >= 0` [name: 'ck_orders_non_negative']
    `total_amount = subtotal - membership_discount_amount - coupon_discount_amount` [name: 'ck_orders_total']
    `refunded_amount BETWEEN 0 AND total_amount` [name: 'ck_orders_refunded']
    `(status = 'PAID' AND refunded_amount = 0) OR (status = 'PARTIALLY_REFUNDED' AND refunded_amount > 0 AND refunded_amount < total_amount) OR (status = 'REFUNDED' AND total_amount > 0 AND refunded_amount = total_amount)` [name: 'ck_orders_status']
    `jsonb_typeof(invoice_snapshot) = 'object' AND invoice_snapshot ? 'schema_version'` [name: 'ck_orders_snapshot']
    `account_id IS NOT NULL OR (length(btrim(coalesce(guest_name, ''))) > 0 AND length(btrim(coalesce(guest_phone, ''))) > 0 AND created_by IS NOT NULL AND payment_method <> 'WALLET' AND coupon_id IS NULL AND membership_discount_amount = 0 AND refunded_amount = 0)` [name: 'ck_orders_guest']
  }
}

Ref: orders.account_id > accounts.id [delete: restrict]
Ref: orders.created_by > accounts.id [delete: restrict]
Ref: orders.coupon_id > coupons.id [delete: restrict]

Table order_items {
  id                         uuid            [pk, default: `gen_random_uuid()`]
  order_id                   uuid            [not null]
  line_number                int             [not null]
  type                       order_item_type [not null]
  item_snapshot              jsonb           [not null]
  subtotal                   numeric(12,0)   [not null]
  membership_discount_amount numeric(12,0)   [not null, default: 0]
  coupon_discount_amount     numeric(12,0)   [not null, default: 0]
  total_amount               numeric(12,0)   [not null]
  refunded_amount            numeric(12,0)   [not null, default: 0]
  created_at                 timestamptz     [not null, default: `now()`]
  updated_at                 timestamptz     [not null, default: `now()`]

  indexes {
    (order_id, line_number) [unique, name: 'uq_order_item_line']
    (type, created_at) [name: 'idx_order_items_type_date']
  }

  checks {
    `line_number >= 1 AND subtotal >= 0 AND membership_discount_amount >= 0 AND coupon_discount_amount >= 0 AND total_amount >= 0` [name: 'ck_order_items_non_negative']
    `total_amount = subtotal - membership_discount_amount - coupon_discount_amount` [name: 'ck_order_items_total']
    `refunded_amount BETWEEN 0 AND total_amount` [name: 'ck_order_items_refunded']
    `type <> 'MEMBERSHIP' OR (refunded_amount = 0 AND membership_discount_amount = 0)` [name: 'ck_order_items_membership']
    `jsonb_typeof(item_snapshot) = 'object' AND item_snapshot ? 'schema_version'` [name: 'ck_order_items_snapshot']
  }

  Note: 'Partial unique: uq_order_one_membership (order_id) WHERE type = MEMBERSHIP'
}

Ref: order_items.order_id > orders.id [delete: restrict]

Table coupons {
  id                uuid              [pk, default: `gen_random_uuid()`]
  code              varchar(50)       [not null]
  name              varchar(255)      [not null]
  discount_type     discount_type     [not null]
  discount_value    numeric(12,0)     [not null]
  max_discount      numeric(12,0)
  min_order_amount  numeric(12,0)
  max_uses          int
  max_uses_per_user int               [not null, default: 1]
  applicable_types  "order_item_type[]"
  valid_from        timestamptz       [not null]
  valid_to          timestamptz       [not null]
  is_active         boolean           [not null, default: true]
  deleted_at        timestamptz
  created_at        timestamptz       [not null, default: `now()`]
  updated_at        timestamptz       [not null, default: `now()`]

  checks {
    `discount_value > 0 AND (discount_type <> 'PERCENT' OR discount_value <= 100)` [name: 'ck_coupons_value']
    `(max_discount IS NULL OR max_discount >= 0) AND (min_order_amount IS NULL OR min_order_amount >= 0)` [name: 'ck_coupons_amounts']
    `(max_uses IS NULL OR max_uses >= 0) AND max_uses_per_user >= 1` [name: 'ck_coupons_uses']
    `valid_to > valid_from` [name: 'ck_coupons_validity']
    `code = upper(btrim(code))` [name: 'ck_coupons_code']
  }

  Note: 'Partial unique: uq_coupons_code (code) WHERE deleted_at IS NULL'
}

Table center_checkins {
  id            uuid        [pk, default: `gen_random_uuid()`]
  account_id    uuid        [not null]
  checked_by    uuid        [not null]
  check_in_time timestamptz [not null, default: `now()`]

  indexes {
    (account_id, check_in_time) [name: 'idx_checkin_account_time']
  }
}

Ref: center_checkins.account_id > accounts.id [delete: restrict]
Ref: center_checkins.checked_by > accounts.id [delete: restrict]

Table class_attendance {
  id         uuid              [pk, default: `gen_random_uuid()`]
  session_id uuid              [not null]
  account_id uuid              [not null]
  status     attendance_status [not null]
  note       varchar(500)
  updated_by uuid
  created_at timestamptz       [not null, default: `now()`]
  updated_at timestamptz       [not null, default: `now()`]

  indexes {
    (session_id, account_id) [unique, name: 'uq_attendance_session_account']
    account_id [name: 'idx_attendance_account']
  }
}

Ref: class_attendance.session_id > class_sessions.id [delete: restrict]
Ref: class_attendance.account_id > accounts.id [delete: restrict]
Ref: class_attendance.updated_by > accounts.id [delete: restrict]

Table member_evaluations {
  id         uuid        [pk, default: `gen_random_uuid()`]
  session_id uuid        [not null]
  account_id uuid        [not null]
  coach_id   uuid        [not null]
  rating     int         [not null]
  comment    text
  deleted_at timestamptz
  created_at timestamptz [not null, default: `now()`]
  updated_at timestamptz [not null, default: `now()`]

  indexes {
    (session_id, account_id) [name: 'idx_eval_session_account']
    account_id [name: 'idx_eval_account']
  }

  checks {
    `rating BETWEEN 1 AND 5` [name: 'ck_eval_rating']
  }

  Note: 'Partial unique: uq_eval_session_account (session_id, account_id) WHERE deleted_at IS NULL'
}

Ref: member_evaluations.session_id > class_sessions.id [delete: restrict]
Ref: member_evaluations.account_id > accounts.id [delete: restrict]
Ref: member_evaluations.coach_id > accounts.id [delete: restrict]

Table notifications {
  id             uuid              [pk, default: `gen_random_uuid()`]
  account_id     uuid              [not null]
  type           notification_type [not null]
  title          varchar(255)      [not null]
  message        text              [not null]
  reference_type varchar(50)
  reference_id   uuid
  dedup_key      varchar(150)      [unique]
  read_at        timestamptz
  send_email     boolean           [not null, default: false]
  email_sent_at  timestamptz
  created_at     timestamptz       [not null, default: `now()`]

  indexes {
    (account_id, created_at) [name: 'idx_noti_account_date']
    (account_id, read_at) [name: 'idx_noti_account_read']
    created_at [name: 'idx_noti_date']
  }
}

Ref: notifications.account_id > accounts.id [delete: restrict]

Table audit_logs {
  id          uuid        [pk, default: `gen_random_uuid()`]
  account_id  uuid
  action      varchar(50) [not null]
  entity_type varchar(50) [not null]
  entity_id   text        [not null]
  old_values  jsonb
  new_values  jsonb
  ip_address  varchar(45)
  created_at  timestamptz [not null, default: `now()`]

  indexes {
    (entity_type, created_at) [name: 'idx_audit_entity_date']
    (account_id, created_at) [name: 'idx_audit_account_date']
  }
}

Ref: audit_logs.account_id > accounts.id [delete: restrict]

Table support_requests {
  id              uuid                     [pk, default: `gen_random_uuid()`]
  account_id      uuid                     [not null]
  handled_by      uuid
  category        support_request_category [not null, default: 'OTHER']
  subject         varchar(255)             [not null]
  description     text                     [not null]
  resolution_note text
  status          support_request_status   [not null, default: 'OPEN']
  resolved_at     timestamptz
  deleted_at      timestamptz
  created_at      timestamptz              [not null, default: `now()`]
  updated_at      timestamptz              [not null, default: `now()`]

  indexes {
    (account_id, status) [name: 'idx_sr_account_status']
    (status, created_at) [name: 'idx_sr_status_date']
  }
}

Ref: support_requests.account_id > accounts.id [delete: restrict]
Ref: support_requests.handled_by > accounts.id [delete: restrict]

Table system_settings {
  id                             int         [pk, default: 1]
  open_time                      time        [not null, default: '06:00']
  close_time                     time        [not null, default: '22:00']
  slot_duration_minutes          int         [not null, default: 60]
  max_advance_booking_days       int         [not null, default: 7]
  booking_cancel_deadline_hours  int         [not null, default: 24]
  course_cancel_deadline_days    int         [not null, default: 3]
  membership_expiry_warning_days int         [not null, default: 7]
  top_up_min_amount              numeric(12,0) [not null, default: 10000]
  top_up_expiry_minutes          int         [not null, default: 30]
  updated_by                     uuid
  updated_at                     timestamptz [not null, default: `now()`]

  checks {
    `id = 1` [name: 'ck_settings_single_row']
    `open_time < close_time AND slot_duration_minutes > 0` [name: 'ck_settings_hours']
    `max_advance_booking_days >= 0 AND booking_cancel_deadline_hours >= 0 AND course_cancel_deadline_days >= 0 AND membership_expiry_warning_days >= 0` [name: 'ck_settings_deadlines']
    `top_up_min_amount > 0 AND top_up_expiry_minutes > 0` [name: 'ck_settings_top_up']
  }
}

Ref: system_settings.updated_by > accounts.id [delete: restrict]
