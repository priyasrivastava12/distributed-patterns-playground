can u help me give the structure of table used in payment processing and in our system and also give all the enums 
(state) changes in different scenerios :

Absolutely — let’s design **production-level table structures** for the booking + payment flow. I’ll keep it **clean, interview-ready, and practical** (with indexes, constraints, and reasoning).

---

# 🧠 1. Core Tables Overview

We’ll define:

1. **SeatInventory** → seat state
2. **Booking** → user booking
3. **Booking_Seats (mapping)** → seats per booking
4. **Payment** → payment tracking
5. **Outbox (optional but powerful)** → event reliability

---

# 🪑 2. SeatInventory Table (CRITICAL)

```sql id="9p3l2m"
CREATE TABLE seat_inventory (
    seat_id        VARCHAR(10),
    show_id        BIGINT,
    status         VARCHAR(20), -- AVAILABLE | RESERVED | BOOKED
    reserved_by    BIGINT NULL,
    booking_id     BIGINT NULL,
    expiry_time    TIMESTAMP NULL,
    version        INT DEFAULT 0,

    PRIMARY KEY (seat_id, show_id)
);
```

---

## 🔍 Important Points

* **Composite PK (seat_id + show_id)** → same seat across different shows
* `status` → controls lifecycle
* `reserved_by` → ensures only same user can checkout
* `expiry_time` → for auto release
* `version` → optimistic locking

---

## ⚡ Indexes

```sql id="0vxqyy"
CREATE INDEX idx_show_status ON seat_inventory(show_id, status);
CREATE INDEX idx_expiry ON seat_inventory(expiry_time);
```

---

# 🎟️ 3. Booking Table

```sql id="7o2p1f"
CREATE TABLE booking (
    booking_id     BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id        BIGINT NOT NULL,
    show_id        BIGINT NOT NULL,
    status         VARCHAR(20), -- PENDING | CONFIRMED | FAILED
    total_amount   DECIMAL(10,2),
    created_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expiry_time    TIMESTAMP
);
```

---

## 🔍 Why needed?

* Tracks booking lifecycle
* Helps resume payment
* Useful for audit

---

# 🔗 4. Booking_Seats Mapping Table

---

## Why separate table?

👉 A booking can have multiple seats

---

```sql id="0m9x2s"
CREATE TABLE booking_seats (
    booking_id BIGINT,
    seat_id    VARCHAR(10),
    show_id    BIGINT,

    PRIMARY KEY (booking_id, seat_id, show_id)
);
```

---

## 🔍 Benefit:

* Avoids duplication
* Easy seat lookup per booking

---

# 💳 5. Payment Table

---

```sql id="u8n7g2"
CREATE TABLE payment (
    payment_id        BIGINT PRIMARY KEY AUTO_INCREMENT,
    booking_id        BIGINT,
    status            VARCHAR(20), -- INITIATED | SUCCESS | FAILED
    amount            DECIMAL(10,2),
    gateway_txn_id    VARCHAR(100),
    idempotency_key   VARCHAR(100),
    created_at        TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE (idempotency_key)
);
```

---

## 🔍 Key Points

* `booking_id` → link to booking
* `gateway_txn_id` → from Razorpay/Stripe
* `idempotency_key` → prevents duplicate payments

---

# 🔐 6. Optional but Powerful: Outbox Table

---

```sql id="skj3n9"
CREATE TABLE outbox (
    event_id     BIGINT PRIMARY KEY AUTO_INCREMENT,
    event_type   VARCHAR(50),
    payload      JSON,
    status       VARCHAR(20), -- NEW | PROCESSED
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## 🔍 Why?

* Reliable Kafka publishing
* Prevents data loss

---

# 🔄 7. How Tables Work Together (Flow)

---

## 🎯 Seat Reservation

```sql id="wd8x9n"
UPDATE seat_inventory
SET status = 'RESERVED',
    reserved_by = :userId,
    expiry_time = NOW() + INTERVAL 5 MINUTE
WHERE seat_id = :seatId
AND show_id = :showId
AND status = 'AVAILABLE';
```

---

## 🎟️ Create Booking

```sql id="p2x7kl"
INSERT INTO booking (user_id, show_id, status, expiry_time)
VALUES (:userId, :showId, 'PENDING', NOW() + INTERVAL 5 MINUTE);
```

---

## 🔗 Map Seats

```sql id="v3d7as"
INSERT INTO booking_seats VALUES (...);
```

---

## 💳 Payment

```sql id="z9x1kl"
INSERT INTO payment (booking_id, status, amount, idempotency_key)
VALUES (:bookingId, 'INITIATED', 500, :bookingId);
```

---

## ✅ On Payment Success

```sql id="q7r2sd"
UPDATE booking SET status = 'CONFIRMED' WHERE booking_id = :id;

UPDATE seat_inventory
SET status = 'BOOKED'
WHERE booking_id = :id;
```

---

## ❌ On Payment Failure

```sql id="m3z8pl"
UPDATE booking SET status = 'FAILED' WHERE booking_id = :id;

UPDATE seat_inventory
SET status = 'AVAILABLE',
    reserved_by = NULL,
    booking_id = NULL
WHERE booking_id = :id;
```

---

# ⚠️ 8. Important Constraints

---

## Prevent double booking

```sql id="x1m9bc"
CHECK (status IN ('AVAILABLE', 'RESERVED', 'BOOKED'))
```

---

## Ensure valid state transitions (logic layer)

* AVAILABLE → RESERVED
* RESERVED → BOOKED / AVAILABLE

---

# 🚀 9. Scaling Considerations

---

## Partitioning

```text id="h7s2qp"
Partition by show_id
```

---

## Why?

* Hot shows (Avengers) → heavy traffic
* Avoid DB bottleneck

---

# 🎯 10. Interview-Perfect Summary

Say this:

> “I would design separate tables for seat inventory, booking, and payment to maintain clear separation of concerns. Seat inventory tracks real-time seat status with optimistic locking, booking maintains user-level transaction state, and payment ensures idempotent and reliable payment tracking. I would also include a mapping table for seats per booking and optionally use an outbox table for reliable event-driven processing.”

Great — this is exactly the level where your design becomes **production-grade + interview impressive**.

We’ll define **ALL state enums + valid transitions + scenarios**, not just basic ones.

---

# 🧠 1. Why State Design Matters

States help you:

* Handle failures safely
* Support retries
* Maintain consistency
* Debug production issues

👉 Think of each table as a **state machine**

---

# 🎟️ 2. Booking State Machine

---

## ✅ Booking Status ENUM

```text
INITIATED      → booking created, seats not yet locked
PENDING        → seats reserved, payment in progress
CONFIRMED      → payment success, seats booked
FAILED         → payment failed
EXPIRED        → user didn’t pay in time
CANCELLED      → user cancelled
REFUND_PENDING → refund initiated
REFUNDED       → refund completed
```

---

## 🔄 Booking Transitions

```text
INITIATED → PENDING → CONFIRMED
                 ↓
               FAILED
                 ↓
               EXPIRED

CONFIRMED → CANCELLED → REFUND_PENDING → REFUNDED
```

---

## 💡 Notes

* `INITIATED` is optional (some systems start at PENDING)
* `EXPIRED` handled by background job
* Refund states used post-booking

---

# 🪑 3. SeatInventory State Machine

---

## ✅ Seat Status ENUM

```text
AVAILABLE
RESERVED
BOOKED
BLOCKED        (admin hold / maintenance)
```

---

## 🔄 Seat Transitions

```text
AVAILABLE → RESERVED → BOOKED
     ↑         ↓
     └── EXPIRED / FAILED ──┘
```

---

## 💡 Scenarios

| Scenario          | Transition           |
| ----------------- | -------------------- |
| User selects seat | AVAILABLE → RESERVED |
| Payment success   | RESERVED → BOOKED    |
| Payment failed    | RESERVED → AVAILABLE |
| Timeout           | RESERVED → AVAILABLE |
| Admin block       | AVAILABLE → BLOCKED  |

---

# 💳 4. Payment State Machine

---

## ✅ Payment Status ENUM

```text
CREATED        → record created internally
INITIATED      → request sent to gateway
PENDING        → waiting for user action
SUCCESS        → payment completed
FAILED         → payment failed
TIMEOUT        → no response from gateway
CANCELLED      → user aborted payment
REFUND_INITIATED
REFUNDED
```

---

## 🔄 Payment Transitions

```text
CREATED → INITIATED → PENDING → SUCCESS
                               ↓
                             FAILED
                               ↓
                             TIMEOUT

SUCCESS → REFUND_INITIATED → REFUNDED
```

---

## 💡 Important Notes

* `PENDING` = user still on payment page
* `SUCCESS` comes via webhook
* `TIMEOUT` handled via polling

---

# 🔗 5. Combined Flow (Cross-Table State Sync)

---

## 🎯 SUCCESS FLOW

```text
Seat       : AVAILABLE → RESERVED → BOOKED
Booking    : INITIATED → PENDING → CONFIRMED
Payment    : CREATED → INITIATED → SUCCESS
```

---

## ❌ PAYMENT FAILURE

```text
Seat       : RESERVED → AVAILABLE
Booking    : PENDING → FAILED
Payment    : INITIATED → FAILED
```

---

## ⏳ TIMEOUT / ABANDONED

```text
Seat       : RESERVED → AVAILABLE
Booking    : PENDING → EXPIRED
Payment    : INITIATED → TIMEOUT
```

---

## 🔁 REFUND FLOW

```text
Booking    : CONFIRMED → CANCELLED → REFUND_PENDING → REFUNDED
Payment    : SUCCESS → REFUND_INITIATED → REFUNDED
Seat       : BOOKED (usually stays booked, depends on policy)
```

---

# ⚠️ 6. Critical Rules (Interview GOLD)

---

## 🔐 Rule 1: Only RESERVED seats can be BOOKED

```text
INVALID: AVAILABLE → BOOKED ❌
```

---

## 🔐 Rule 2: Only SUCCESS payment confirms booking

---

## 🔐 Rule 3: Booking must match Payment

```text
Payment SUCCESS ⇒ Booking CONFIRMED
```

---

## 🔐 Rule 4: Expiry must clean system

* Seats released
* Booking expired
* Payment marked timeout

---

# 🧠 7. Edge Case Handling via States

---

## ❗ Duplicate webhook

* Payment already SUCCESS → ignore

---

## ❗ Late payment after expiry

* Booking = EXPIRED
* Reject confirmation

---

## ❗ User retries payment

* Same bookingId
* New payment record OR reuse idempotency

---

# 🚀 8. Production-Level ENUM Definition (SQL)

---

## Booking ENUM

```sql
ENUM('INITIATED','PENDING','CONFIRMED','FAILED','EXPIRED','CANCELLED','REFUND_PENDING','REFUNDED')
```

---

## Seat ENUM

```sql
ENUM('AVAILABLE','RESERVED','BOOKED','BLOCKED')
```

---

## Payment ENUM

```sql
ENUM('CREATED','INITIATED','PENDING','SUCCESS','FAILED','TIMEOUT','CANCELLED','REFUND_INITIATED','REFUNDED')
```

---

# 🎯 9. Interview-Perfect Answer

If asked:

👉 “What states would you design?”

Say:

> “I would model booking, seat, and payment as independent state machines. Seat transitions from AVAILABLE → RESERVED → BOOKED, booking moves from PENDING to CONFIRMED or FAILED, and payment transitions from INITIATED to SUCCESS or FAILED. These states allow us to handle retries, failures, and asynchronous workflows reliably while ensuring consistency across the system.”

---

# 💥 10. Pro Insight (Very Strong Signal)

👉 “Designing explicit state machines simplifies recovery, debugging, and ensures correctness in distributed systems.”

---

