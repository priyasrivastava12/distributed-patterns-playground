Great question — this is **core to ticket booking system design** (like BookMyShow). Let’s break it down cleanly:

---

# 🔒 1. How to ensure *same user* completes checkout (and not others)?

You don’t “lock the user” — you **lock the seat(s)** temporarily.

### ✅ Flow

1. User selects seats
2. System creates a **temporary reservation (HOLD)**
3. Seats are marked as:

   ```
   RESERVED (by userId, expiresAt)
   ```
4. Only that user can proceed to payment using:

   ```
   bookingId + userId validation
   ```

### ✅ Key idea

* Seat is not *owned permanently* yet
* It is **soft-locked for a specific user**

---

## 🔐 Data Model Addition

In your `Ticket` or `SeatInventory`:

```
seatId
showId
status → AVAILABLE | RESERVED | BOOKED
reservedBy → userId
reservationExpiry → timestamp
```

---

## 🧠 Validation during checkout

When user tries to pay:

```
if (seat.reservedBy != currentUserId) → reject
if (currentTime > reservationExpiry) → release + reject
```

So even if another user tries:
👉 They cannot checkout because they don’t own reservation

---

# 🔒 2. How locking works (Important)

There are **3 common strategies**:

---

## 🟢 Option 1: Optimistic Locking (Recommended for scale)

* No hard lock
* Use versioning or conditional updates

### Example:

```
UPDATE seats
SET status = 'RESERVED', reservedBy = user1
WHERE seatId = X AND status = 'AVAILABLE'
```

👉 Only **one user succeeds**
👉 Others fail → retry

---

## 🔴 Option 2: Pessimistic Locking (DB lock)

```
SELECT * FROM seats WHERE seatId = X FOR UPDATE
```

* Locks row until transaction completes
* Prevents race condition

❌ Problem:

* Doesn’t scale (high contention)
* Slow under heavy traffic

---

## 🟡 Option 3: Distributed Lock (Redis)

Use:

```
SET seat:X lock:user1 NX EX 300
```

* NX → only if not exists
* EX → expiry (5 mins)

👉 Works well in distributed systems

---

## 🧠 Best Practice (Real systems)

Use combination:

* Redis → fast locking
* DB → final consistency

---

# ⏳ 3. What happens if user doesn’t pay?

You need a **TTL (expiry mechanism)**

### Example:

* Reservation valid for 5 minutes
* After that:

```
status → AVAILABLE
reservedBy → NULL
```

### Implementation:

* Background job (cron / queue)
* OR Redis expiry events

---

# 💥 Edge Cases

1. User closes app → seat auto released after TTL
2. Payment success but DB fails → need retry (idempotency key)
3. Double booking → prevented via atomic update

---

# 🔍 4. Why use Elasticsearch?

Elasticsearch is **NOT for booking or locking**

It is used for:

### ✅ Fast search use-cases

* Movies near me
* Shows by city
* Filters (time, language, rating)

---

## 🔥 Problems Elasticsearch solves

### 1. Fast search (millisecond latency)

Instead of:

```
SELECT * FROM shows WHERE city = 'Bangalore'
```

Use ES:

* Geo queries (nearest theaters)
* Full-text search

---

### 2. Complex filtering

* By time
* By language
* By price range

---

### 3. High read scalability

* Millions of users browsing

---

## ❌ What Elasticsearch should NOT do

* Seat availability (❌ eventually consistent)
* Booking transactions (❌ no ACID guarantees)

---

# ⚠️ Problem if you use ES for seats

If you store seat status in ES:

* Delay in updates
* Two users may see same seat as available
* Double booking risk 🚨

👉 Always use **DB (source of truth)** for booking

---

# 🧩 Final Architecture Thought

### Use:

* **DB (Postgres/MySQL)** → seats, bookings (strong consistency)
* **Redis** → locks + TTL
* **Elasticsearch** → search & discovery
* **Queue (Kafka/SQS)** → async tasks (payment, notifications)

---

# 🚀 Simple Flow Summary

```
User → Select Seat
     → Lock seat (Redis / DB)
     → Create reservation (TTL)
     → Payment
     → Confirm booking (DB update)
```



--------------------------------------------------------------------------------------------------------

Perfect — let’s go **full interview mode + production-level depth**. I’ll cover everything step-by-step so you can **explain confidently + handle follow-ups**.

---

# 🏗️ 1. Clean Architecture Diagram (Explain like this)

```
Client (Mobile/Web)
        ↓
API Gateway (Auth, Rate Limit)
        ↓
-------------------------------------------------
|  Browse Service     |  Booking Service        |
|  (Search)           |  (Seat Locking)         |
-------------------------------------------------
        ↓                      ↓
   Elasticsearch          Redis (Lock/TTL)
        ↓                      ↓
        --------->  DB (Postgres/MySQL)  <---------
                       ↓
                Payment Service
                       ↓
                     Kafka
                       ↓
            Notification Service
```

---

# 🔄 2. End-to-End Booking Flow (VERY IMPORTANT)

### 🎯 Step-by-step (say this clearly in interview)

### 1. Browse

* User searches movie → hits **Elasticsearch**
* Fast filtering (city, time, language)

---

### 2. Select Seats

👉 Critical step

* User selects seats (A1, A2)
* System tries to reserve:

```sql
UPDATE seat_inventory
SET status = 'RESERVED',
    reserved_by = :userId,
    expiry_time = NOW() + INTERVAL 5 MINUTE
WHERE seat_id IN (A1, A2)
  AND status = 'AVAILABLE';
```

👉 Only succeeds if seats are free

---

### 3. Create Booking

```json
{
  "bookingId": "B123",
  "userId": "U1",
  "status": "PENDING",
  "expiry": "5 mins"
}
```

---

### 4. Payment

* Call payment gateway
* Use **idempotency key = bookingId**

---

### 5. Confirm Booking

```sql
UPDATE seat_inventory
SET status = 'BOOKED'
WHERE reserved_by = :userId;
```

---

### 6. Send Ticket

* Kafka event → Notification service

---

# 🔒 3. Locking Deep Dive (Interview Favorite 🔥)

---

## 🟢 Approach 1: Optimistic Locking (Best Answer)

Add column:

```
version INT
```

Query:

```sql
UPDATE seat_inventory
SET status='RESERVED', version = version + 1
WHERE seat_id = A1
AND version = 3;
```

👉 If version changed → retry

---

## 🟡 Approach 2: Redis Distributed Lock

```
SET seat:A1 user1 NX EX 300
```

* NX → only if not exists
* EX → expiry

---

## 🔴 Approach 3: Pessimistic Lock

```sql
SELECT * FROM seats WHERE id=A1 FOR UPDATE;
```

❌ Blocks others → not scalable

---

## 💡 Best Answer

👉 “Use Redis for fast locking + DB for final consistency”

---

# ⚡ 4. Concurrency Handling (Real Thinking)

---

## Problem:

1000 users clicking same seat

---

## Solution:

* Atomic DB update
* Only 1 succeeds
* Others fail gracefully

---

## Add Retry Strategy:

* Retry with backoff
* Suggest nearby seats

---

# ⏳ 5. TTL / Expiry Handling

---

## Why?

User may abandon payment

---

## Solution:

### Option 1: Redis TTL

* Auto expiry

### Option 2: Background Job

```sql
UPDATE seats
SET status = 'AVAILABLE'
WHERE expiry_time < NOW();
```

---

# 💳 6. Payment Deep Dive

---

## Problem: Duplicate Payment

---

## Solution: Idempotency

```
POST /payment
Idempotency-Key: bookingId
```

---

## Problem: Payment success but DB fails

---

## Solution:

* Kafka retry event
* Eventually consistent update

---

# 🧩 7. Database Design (Clean + Scalable)

---

## Tables:

### SeatInventory

```
seatId
showId
status
reservedBy
expiryTime
version
```

---

### Booking

```
bookingId
userId
status (PENDING, CONFIRMED)
```

---

### Payment

```
paymentId
bookingId
status
```

---

# 🔍 8. Elasticsearch Deep Dive

---

## Used For:

* Search movies
* Filter shows
* Geo queries

---

## Example Query:

* "Movies near me after 6 PM"

---

## ❌ Not used for:

* Seat availability
* Transactions

---

## Why?

👉 Elasticsearch = eventually consistent
👉 Booking needs strong consistency

---

# 🚀 9. Scaling Strategy (Senior Level)

---

## 🔥 Problem: Avengers Release (Huge Traffic)

---

## Solutions:

### 1. Read Scaling

* Cache (Redis)
* Elasticsearch

---

### 2. Write Scaling

👉 Partition by:

```
showId
```

---

### 3. Virtual Queue

* Users enter queue
* Controlled booking rate

---

### 4. Rate Limiting

* Per user/IP

---

# 🔁 10. Event-Driven Design (Kafka)

---

## Events:

* Booking Created
* Payment Success
* Ticket Confirmed

---

## Why?

* Retry handling
* Loose coupling

---

# 🧠 11. Advanced Patterns (🔥 must know)

---

## Saga Pattern

Booking = multi-step transaction

Steps:

1. Reserve seat
2. Payment
3. Confirm

👉 If payment fails → rollback

---

## Outbox Pattern

* Avoid DB + Kafka inconsistency

---

## CQRS

* Reads → Elasticsearch
* Writes → DB

---

# 🎤 12. Mock Interview Questions + Answers

---

## ❓ Q1: How do you prevent double booking?

👉 Answer:

* Atomic DB update
* Optimistic locking
* Unique constraint

---

## ❓ Q2: What if Redis fails?

👉 Answer:

* DB is source of truth
* Re-validate before booking

---

## ❓ Q3: How to handle partial failure?

👉 Answer:

* Saga pattern
* Retry via Kafka

---

## ❓ Q4: Why not lock entire row?

👉 Answer:

* Reduces scalability
* High contention

---

## ❓ Q5: How do you scale to millions?

👉 Answer:

* Partitioning
* Queueing
* Caching

---

## ❓ Q6: How ensure same user completes booking?

👉 Answer:

* `reservedBy` validation
* Booking tied to userId

---

## ❓ Q7: What if user refreshes page?

👉 Answer:

* BookingId stored
* Resume flow

---

# 🧠 13. Production-Level API Design

---

## 🎟️ Select Seats

```
POST /shows/{id}/reserve
```

Request:

```json
{
  "seatIds": ["A1", "A2"]
}
```

---

## 💳 Payment

```
POST /payments
```

Headers:

```
Idempotency-Key: bookingId
```

---

## 🎫 Confirm Booking

```
POST /bookings/{id}/confirm
```

---

# 🎯 Final Interview Summary (Say This)

👉

“I’ll design the system using a microservices architecture. Elasticsearch will handle search, Redis will manage locks and caching, and a relational DB will ensure strong consistency for bookings. Seat reservation will use optimistic locking with TTL to prevent double booking. Payments will be idempotent, and Kafka will handle async workflows like notifications and retries. For scalability, I’ll partition by showId and introduce rate limiting and queues during high traffic.”



--------------------------------------------------------------

Short answer first: **you don’t *need* Elasticsearch to make booking work** — you need it to make **search fast, flexible, and scalable**. Without it, your system will struggle when users are *browsing*, not when they are *booking*.

Now let’s go deeper 👇

---

# 🎯 1. What problem are we solving?

When a user opens your app, they do things like:

* “Movies in Bangalore”
* “Shows after 7 PM”
* “Hindi movies near me”
* “Avengers in IMAX tomorrow”

This is **search + filtering + sorting + geo queries**.

👉 A relational DB (like Postgres/MySQL) is **not optimized** for this kind of workload at scale.

---

# ⚠️ 2. What goes wrong if you only use DB?

Imagine this query:

```sql
SELECT * 
FROM shows 
WHERE city = 'Bangalore'
AND language = 'Hindi'
AND time > '7 PM'
ORDER BY rating DESC;
```

### Problems:

### ❌ 1. Slow queries at scale

* Millions of rows
* Multiple filters + joins
* Latency increases

---

### ❌ 2. Poor full-text search

User types:

> “Avngr movi”

DB cannot:

* Handle typos
* Do fuzzy matching
* Rank relevance

---

### ❌ 3. No geo search

User wants:

> “theaters near me”

DB struggles with:

* distance calculations
* geo indexing

---

### ❌ 4. High read load

* Thousands of users browsing simultaneously
* DB becomes bottleneck

---

# 🚀 3. What Elasticsearch solves

---

## ✅ 1. Full-text search (Google-like)

* Handles typos
* Relevance ranking
* Tokenization

Example:

> “Avngr” → still finds *Avengers*

---

## ✅ 2. Super fast filtering

* Time
* Language
* Price
* Theater

👉 Uses inverted index (not row scan)

---

## ✅ 3. Geo search

* “near me” queries
* Distance-based ranking

---

## ✅ 4. Millisecond latency

* Designed for **read-heavy systems**

---

## ✅ 5. Aggregations

* “Top movies in Bangalore”
* “Most popular shows”

---

# 🧠 4. Key Concept (Very Important)

👉 Elasticsearch is **NOT source of truth**

It is:

> “Search index optimized for fast reads”

---

# ⚠️ 5. Why NOT use Elasticsearch for booking?

This is where many candidates fail.

---

## ❌ Problem 1: Eventual Consistency

* Delay in updates
* Seat may appear AVAILABLE but already BOOKED

---

## ❌ Problem 2: No ACID guarantees

* No transactions
* Risk of double booking

---

## ❌ Problem 3: No strong locking

* Cannot enforce concurrency control

---

# 🧩 6. Final Architecture Thinking

---

## Use DB for:

* Seats
* Bookings
* Payments
  👉 Needs **strong consistency**

---

## Use Elasticsearch for:

* Movies search
* Shows discovery
  👉 Needs **fast reads**

---

# 🔄 7. Real Data Flow

1. Data stored in DB
2. Synced to Elasticsearch (via Kafka / pipeline)
3. User searches → Elasticsearch
4. User books → DB

---

# 🎤 8. Interview-Ready Answer

If interviewer asks:

👉 **“Why Elasticsearch?”**

You say:

> “Elasticsearch is used to handle fast and flexible search queries like filtering shows by city, time, language, and proximity. A relational database is not optimized for full-text search and geo queries at scale. Elasticsearch provides low-latency search using inverted indexing, but since it is eventually consistent, it is not used for critical operations like booking, where we rely on a strongly consistent database.”

---

# 🧠 Simple Analogy

* **Database = Bank account (accurate, strict)**
* **Elasticsearch = Google search (fast, approximate)**

---
------------------------------------------------------------------------

Great — now let’s go **one level deeper (this is where strong candidates stand out)**:

We’ll cover:

1. **How data syncs from DB → Elasticsearch**
2. **Real Elasticsearch queries (movie app use-case)**
3. **Tricky interview cross-questions (with sharp answers)**

---

# 🔄 1. How data flows: DB → Elasticsearch

Remember:
👉 DB = **source of truth**
👉 Elasticsearch = **search index**

---

## ❌ Wrong Approach (common mistake)

* Directly writing to both DB and Elasticsearch in same request

👉 Problem:

* If ES write fails → inconsistency
* Hard to retry safely

---

## ✅ Correct Approach: Event-Driven Sync

```id="d3aw6j"
DB (Postgres)
   ↓
Outbox Table
   ↓
Kafka (event stream)
   ↓
Consumer Service
   ↓
Elasticsearch Index
```

---

## 🧠 Step-by-step flow

### 1. Write to DB

```sql id="6s67q9"
INSERT INTO shows (...)
```

---

### 2. Write event in Outbox (same transaction)

```sql id="fl4x8h"
INSERT INTO outbox (event_type, payload)
VALUES ('SHOW_CREATED', {...});
```

👉 Ensures **no data loss**

---

### 3. Kafka Producer reads outbox

* Publishes event:

```json id="6ehf63"
{
  "event": "SHOW_CREATED",
  "data": {...}
}
```

---

### 4. Consumer updates Elasticsearch

```json id="45cjkn"
PUT /shows/_doc/123
{
  "movie": "Avengers",
  "city": "Bangalore",
  "time": "7 PM"
}
```

---

## 💡 Why this is powerful?

* Retry safe
* Scalable
* Decoupled
* Industry standard (Outbox Pattern)

---

# 🔍 2. Real Elasticsearch Queries (Movie App)

---

## 🎬 1. Search movie by name (fuzzy)

```json id="u3zqg3"
GET /movies/_search
{
  "query": {
    "match": {
      "title": {
        "query": "avngr",
        "fuzziness": "AUTO"
      }
    }
  }
}
```

👉 Handles typos

---

## 📍 2. Find theaters near user

```json id="p2g9hp"
GET /theaters/_search
{
  "query": {
    "geo_distance": {
      "distance": "5km",
      "location": {
        "lat": 12.97,
        "lon": 77.59
      }
    }
  }
}
```

---

## 🎟️ 3. Filter shows

```json id="dwgmf3"
GET /shows/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "city": "Bangalore" }},
        { "range": { "time": { "gte": "18:00" }}}
      ]
    }
  }
}
```

---

## 📊 4. Aggregation (popular movies)

```json id="dqj93x"
GET /shows/_search
{
  "aggs": {
    "top_movies": {
      "terms": { "field": "movie.keyword" }
    }
  }
}
```

---

# ⚠️ 3. Key Design Insight (VERY IMPORTANT)

👉 Elasticsearch data is **denormalized**

Example:

```json id="b4oxq0"
{
  "showId": 1,
  "movie": "Avengers",
  "theater": "PVR",
  "city": "Bangalore",
  "time": "7 PM"
}
```

👉 No joins → faster queries

---

# 🧠 4. Tricky Interview Questions (with strong answers)

---

## ❓ Q1: What if Elasticsearch is down?

👉 Answer:

> “System should fall back to DB with limited functionality or cached results. Booking is unaffected since it uses DB.”

---

## ❓ Q2: How do you handle stale data in Elasticsearch?

👉 Answer:

> “We accept eventual consistency for search. Critical operations like booking always validate against DB.”

---

## ❓ Q3: Why not directly query DB for search?

👉 Answer:

> “DB is not optimized for full-text, geo, and high-scale read queries. Elasticsearch provides inverted indexing and low latency.”

---

## ❓ Q4: How do you handle re-indexing?

👉 Answer:

* Create new index
* Backfill data
* Switch alias

👉 Zero downtime

---

## ❓ Q5: What happens if Kafka fails?

👉 Answer:

* Retry mechanism
* Outbox ensures no data loss

---

## ❓ Q6: How fresh should search data be?

👉 Answer:

* Few seconds delay acceptable
* Trade-off between performance vs consistency

---

## ❓ Q7: How do you scale Elasticsearch?

👉 Answer:

* Sharding
* Replicas
* Horizontal scaling

---

# 🚀 5. Advanced Insight (Senior-level thinking)

---

## 🧠 Index Design Matters

Example:

* Index by:

```id="oeg5xj"
city + movie + time
```

👉 Faster filtering

---

## 🧠 Hot Partitions Problem

* Popular movie → heavy traffic

👉 Solution:

* Shard smartly
* Cache popular queries

---

## 🧠 Caching Layer

* Redis cache for:

  * Popular searches
  * Trending movies

---

# 🎯 Final Interview Answer (Polished)

If asked:

👉 “How does Elasticsearch fit in your system?”

You say:

> “Elasticsearch is used as a secondary data store optimized for search and discovery. Data is written to the primary database and then asynchronously synced to Elasticsearch using an event-driven pipeline with Kafka and the Outbox pattern. It allows us to support fast full-text search, filtering, and geo queries at scale. Since it is eventually consistent, it is not used for critical operations like booking, where we rely on a strongly consistent database.”

---
----------------------------------------------------------------

Absolutely — here’s a **clean, interview-ready HLD (High-Level Design)** for a movie ticket booking system, but with **all the small, practical details included** so you can confidently handle follow-ups.

---

# 🏗️ 1. High-Level Architecture (HLD)

```text
                ┌───────────────┐
                │   Client      │
                │ (App/Web)     │
                └──────┬────────┘
                       ↓
              ┌──────────────────┐
              │  API Gateway     │
              │ Auth, Rate Limit │
              └──────┬───────────┘
                     ↓
 ┌─────────────────────────────────────────────┐
 │                Services                     │
 │---------------------------------------------│
 │  Browse Service   |  Booking Service        │
 │  (Search)         |  (Seat Locking)         │
 │---------------------------------------------│
 │  Payment Service  |  User Service           │
 └──────────┬──────────────────────────────────┘
            ↓
 ┌─────────────────────────────────────────────┐
 │                Data Layer                   │
 │---------------------------------------------│
 │  DB (Postgres/MySQL) → Source of Truth     │
 │  Redis → Cache + Locks + TTL              │
 │  Elasticsearch → Search Index             │
 │  Kafka → Event Streaming                  │
 └─────────────────────────────────────────────┘
```

---

# 🎯 2. Core Design Principles (say this early)

* **Strong consistency for booking**
* **Eventual consistency for search**
* **Separation of concerns (CQRS)**
* **Fail-safe + retry mechanisms**
* **Scalable read-heavy system**

---

# 🧩 3. Component-Level Deep Dive

---

## 🔍 A. Browse Service

### Responsibility:

* Movie search
* Show discovery
* Filtering (city, time, language)

### Tech:

* Elasticsearch + Redis cache

### Small but important points:

* Cache **popular searches**
* Use **pagination**
* Use **denormalized ES documents**
* Handle **typos (fuzzy search)**

---

## 🎟️ B. Booking Service (CORE)

---

### Responsibilities:

* Seat selection
* Locking
* Reservation
* Booking lifecycle

---

### 💡 Seat State Machine

```text
AVAILABLE → RESERVED → BOOKED
             ↓
         (timeout)
             ↓
         AVAILABLE
```

---

### 🔒 Locking Strategy

#### Primary:

* **Optimistic Locking (DB)**

#### Secondary:

* **Redis distributed lock (fast)**

---

### 🔁 Reservation Flow

1. User selects seats
2. Atomic DB update (only if AVAILABLE)
3. Add:

   * `reservedBy`
   * `expiryTime (5 mins)`

---

### ⚠️ Small but critical points:

* Validate `userId` at checkout
* Prevent seat tampering from frontend
* Store `bookingId` early
* Use **idempotency for retry**

---

## 💳 C. Payment Service

---

### Responsibilities:

* Payment processing
* Idempotency
* Status tracking

---

### Key details:

* Use `Idempotency-Key = bookingId`
* Handle:

  * Success
  * Failure
  * Timeout

---

### Edge cases:

* Payment success but booking fails → retry via Kafka
* Duplicate request → ignore via idempotency

---

## 🧠 D. Seat Inventory (Very Important)

---

### Schema:

```text
seatId
showId
status
reservedBy
expiryTime
version
```

---

### Small details:

* Index on `showId`
* Partition by `showId`
* Use `version` for optimistic locking
* Avoid joins → keep it simple

---

## 🔄 E. Kafka (Event System)

---

### Events:

* Booking Created
* Payment Success
* Booking Confirmed
* Seat Released

---

### Why:

* Retry failed operations
* Decouple services
* Ensure reliability

---

## 🔍 F. Elasticsearch (Search Layer)

---

### Solves:

* Full-text search
* Filtering
* Geo queries

---

### Small details:

* Denormalized documents
* Index by:

  * city
  * movie
  * time
* Use **aliases for reindexing**
* Accept **eventual consistency**

---

## ⚡ G. Redis

---

### Use cases:

* Seat locks
* Caching
* TTL expiry

---

### Small details:

* Use **EX (expiry)** always
* Avoid stale locks
* Use fallback to DB validation

---

# 🔄 4. Complete Booking Flow (Step-by-Step)

---

### Step 1: Search

* Hits Elasticsearch

---

### Step 2: Select Seats

* Validate availability from DB
* Lock via Redis / DB

---

### Step 3: Reserve Seats

* Update DB with:

  * RESERVED
  * expiryTime

---

### Step 4: Payment

* Call payment service
* Use idempotency

---

### Step 5: Confirm Booking

* Update seats → BOOKED
* Update booking status

---

### Step 6: Notification

* Kafka → Email/SMS

---

# ⚠️ 5. Edge Cases (Interview GOLD)

---

## ❗ Case 1: User abandons payment

* TTL expires
* Seats released

---

## ❗ Case 2: Double click booking

* Idempotency prevents duplicate

---

## ❗ Case 3: Two users same seat

* Only one DB update succeeds

---

## ❗ Case 4: Redis failure

* DB validation saves consistency

---

## ❗ Case 5: Payment success, DB failure

* Kafka retry event

---

# 🚀 6. Scalability Strategy

---

## 🔹 Read Scaling

* Elasticsearch
* Redis cache

---

## 🔹 Write Scaling

* Partition DB by `showId`
* Use sharding

---

## 🔹 Traffic Control

* Rate limiting
* Virtual queue for hot shows

---

# 🔐 7. Security

* JWT authentication
* Validate `userId` at booking
* Prevent API misuse
* Rate limit bots

---

# 🧠 8. Advanced Concepts (Add if senior interview)

---

## Saga Pattern

* Handles multi-step booking

---

## Outbox Pattern

* Ensures DB + Kafka consistency

---

## CQRS

* Read → ES
* Write → DB

---

# 🎤 9. How to Explain in Interview (Golden Answer)

Say this clearly:

> “I would design the system using a microservices architecture. Elasticsearch will handle search queries with low latency, while a relational database will act as the source of truth for bookings. Seat reservation will be implemented using optimistic locking with a TTL to prevent double booking. Redis will be used for fast locking and caching. Payment will be idempotent to handle retries safely, and Kafka will be used for asynchronous processing like notifications and recovery. The system will scale via partitioning by showId and rate limiting during peak traffic.”

---

-----------------------------------------------------------------------
