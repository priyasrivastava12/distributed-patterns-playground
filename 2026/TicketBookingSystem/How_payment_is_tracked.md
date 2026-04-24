I want to know how payment is being processed once status in DB is changed to reserved and then how ticketbooking system know that payment is succeeded or failed and how it track that payment request

Great — now you’re asking the **real production question**:
👉 *“Once seats are RESERVED, how does payment actually work end-to-end, and how does the system track success/failure?”*

Let’s go step-by-step like a **real system (Stripe/Razorpay style)**.

---

# 🧠 1. Core Idea (Very Important)

After seats are **RESERVED**, the system creates a **Booking + Payment record** and then:

👉 Payment is handled **asynchronously**
👉 System relies on **payment gateway callbacks/webhooks**
👉 Uses **idempotency + status tracking**

---

# 🧩 2. Key Tables (Tracking Everything)

---

## 🎟️ Booking Table

```text id="p5q3tm"
bookingId
userId
status → PENDING | CONFIRMED | FAILED
amount
expiryTime
```

---

## 💳 Payment Table

```text id="r8kq4x"
paymentId
bookingId
status → INITIATED | SUCCESS | FAILED
gatewayTxnId
idempotencyKey
createdAt
```

---

# 🔄 3. End-to-End Payment Flow

---

## STEP 1: Seat Reserved

DB:

```text id="l4wpm5"
Seat → RESERVED
Booking → PENDING
```

---

## STEP 2: Create Payment Request

```http id="p6gm3v"
POST /payments
```

Payload:

```json id="1k6k9g"
{
  "bookingId": "B123",
  "amount": 500
}
```

---

## STEP 3: Create Payment Entry

DB:

```text id="9k7y2j"
paymentId: P123
status: INITIATED
bookingId: B123
```

---

## STEP 4: Call Payment Gateway

System calls gateway (e.g., Razorpay/Stripe):

```json id="lfp0gi"
{
  "amount": 500,
  "orderId": "B123"
}
```

---

## STEP 5: User Pays on Gateway UI

* Card / UPI / Netbanking
* Happens **outside your system**

---

# 🔔 4. How System Knows Payment Result?

👉 This is the MOST IMPORTANT part

---

## ✅ Method 1: Webhook (Primary)

Payment gateway calls your backend:

```http id="jv72l6"
POST /payment/webhook
```

Payload:

```json id="mx8h9h"
{
  "paymentId": "P123",
  "status": "SUCCESS"
}
```

---

## ✅ Method 2: Polling (Fallback)

Your system calls gateway:

```http id="ydx5rt"
GET /payment/status/P123
```

---

# 🔁 5. Processing Payment Result

---

## CASE 1: Payment SUCCESS

---

### Step 1: Update Payment

```sql id="b7q7wi"
UPDATE payment
SET status = 'SUCCESS'
WHERE paymentId = 'P123';
```

---

### Step 2: Confirm Booking

```sql id="1p5z0w"
UPDATE booking
SET status = 'CONFIRMED'
WHERE bookingId = 'B123';
```

---

### Step 3: Mark Seats BOOKED

```sql id="5mw5lj"
UPDATE seats
SET status = 'BOOKED'
WHERE bookingId = 'B123';
```

---

### Step 4: Send Event (Kafka)

```text id="x8yapz"
BookingConfirmed → Notification Service
```

---

## CASE 2: Payment FAILED

---

### Step 1: Update Payment

```sql id="j8d3b7"
UPDATE payment
SET status = 'FAILED'
WHERE paymentId = 'P123';
```

---

### Step 2: Release Seats

```sql id="c3f3q3"
UPDATE seats
SET status = 'AVAILABLE'
WHERE bookingId = 'B123';
```

---

### Step 3: Update Booking

```sql id="cxm2cs"
UPDATE booking
SET status = 'FAILED'
WHERE bookingId = 'B123';
```

---

# 🔐 6. How Do We Track Payment Safely?

---

## 🔑 Idempotency Key (CRITICAL)

```text id="t5q1rm"
idempotencyKey = bookingId
```

---

### Why?

If:

* User clicks “Pay” twice
* Gateway retries webhook

👉 System won’t process twice

---

# ⚠️ 7. Edge Cases (VERY IMPORTANT)

---

## ❗ Case 1: Payment success but webhook delayed

👉 Solution:

* Keep booking in PENDING
* Poll gateway

---

## ❗ Case 2: Webhook received twice

👉 Solution:

* Idempotent update (check status before updating)

---

## ❗ Case 3: Payment success but DB fails

👉 Solution:

* Kafka retry
* Or reconciliation job

---

## ❗ Case 4: User closes app

👉 Still safe:

* Webhook updates system

---

# ⏳ 8. Timeout Handling

If payment not done in 5 mins:

```sql id="8pxp7q"
UPDATE seats
SET status = 'AVAILABLE'
WHERE expiryTime < NOW();
```

---

# 🧠 9. Clean Flow Diagram

```text id="r2pf77"
User selects seat
    ↓
Seat RESERVED (DB)
    ↓
Booking = PENDING
    ↓
Payment INITIATED
    ↓
User pays (gateway)

        ↓
   Webhook received

IF SUCCESS:
    Payment → SUCCESS
    Booking → CONFIRMED
    Seat → BOOKED

IF FAILED:
    Payment → FAILED
    Booking → FAILED
    Seat → AVAILABLE
```

---

# 🎯 10. Interview-Perfect Answer

If asked:

👉 “How do you track payment status?”

Say:

> “Once the seat is reserved, we create a payment record with status INITIATED and send a request to the payment gateway. The gateway processes the payment and notifies our system asynchronously via webhook. Based on the webhook response, we update the payment status and accordingly confirm or fail the booking. We use idempotency keys to ensure duplicate events don’t cause inconsistent updates.”

---

# 💥 11. Pro-Level Insight

👉 “Payment systems are eventually consistent, so we design for retries, idempotency, and reconciliation instead of assuming a single synchronous response.”

---

