Designing a high-concycle system like **BookMyShow** requires handling massive spikes in traffic (like a blockbuster movie release) while ensuring strict data consistency (no double-booked seats).

---

## 1. Requirements

### Functional Requirements
* **Search:** Browse movies by city, genre, and date.
* **Selection:** View cinema halls, showtimes, and real-time seat availability.
* **Reservation:** Temporary lock on seats for 5–10 minutes during payment.
* **Booking:** Confirm seat after payment and generate a ticket/QR code.
* **Waiting Room:** Handle high demand during "peak" releases.

### Non-Functional Requirements
* **High Availability:** The system must stay up even if a data center fails.
* **Consistency:** Strict consistency for seat selection (prevent double booking).
* **Low Latency:** Fast search and seat selection experience.
* **Scalability:** Must handle 100x traffic during major movie launches.

---

## 2. Database Selection
A **Hybrid Approach** is best here:
* **Relational DB (PostgreSQL/MySQL):** Used for **Transactions, Seat Status, and User Data**. We need ACID compliance to ensure that if a seat is sold, it stays sold.
* **NoSQL (MongoDB/ElasticSearch):** Used for **Movie Metadata and Search**. Movie details don't change often and are read-heavy.
* **In-Memory Store (Redis):** Used for **Temporary Seat Locking** and **Caching** showtimes. This is critical for speed.

---

## 3. High-Level Design (HLD)

The system is broken into microservices:

* **Search Service:** Uses ElasticSearch for fast movie discovery.
* **Booking Service:** Manages the lifecycle of a reservation.
* **Payment Service:** Integrates with third-party gateways.
* **Notification Service:** Sends SMS/Email/WhatsApp tickets.



---

## 4. Low-Level Design (LLD) & DB Schema

### DB Schema (Simplified)
* **Show:** `id, movie_id, theater_id, start_time, end_time`.
* **Seat:** `id, show_id, row, number, type (Gold/Silver), price`.
* **Booking:** `id, user_id, show_id, status (Pending/Confirmed/Cancelled), total_price`.

### How to prevent double booking? (The "Two Seat" problem)
This is the most critical part of the LLD. We use **Distributed Locking** with Redis.

1.  **Transaction Initiation:** When a user clicks "Book," the system attempts to set a key in Redis: `SET seat_id_show_id "locked" NX EX 300`.
    * `NX`: Only set the key if it doesn't exist.
    * `EX 300`: Expire in 5 minutes (auto-release lock if payment fails).
2.  **Database Constraint:** We use a **Unique Constraint** on the `SeatStatus` table: `UNIQUE(show_id, seat_id)`.
3.  **Atomic Update:**
    ```sql
    UPDATE seats SET status = 'LOCKED', user_id = 'XYZ' 
    WHERE id = '123' AND status = 'AVAILABLE';
    ```
    The database ensures that only one worker can update that specific row at a time.

---

## 5. Scalability during Peak Time

When a movie like *Avengers* or *Avatar* drops, the system faces "Thundering Herd" problems.

### Handling the Load:
* **Waiting Room (Virtual Queue):** Use a service like **Cloudflare Waiting Room** or a Kafka-based buffer. Users are redirected to a queue page and let in at a rate the DB can handle.
* **Read Replicas:** Scale PostgreSQL with multiple read replicas for movie/showtime queries.
* **CDN:** Cache all static assets (movie posters, UI components) at the edge.
* **Database Sharding:** Shard the `Booking` and `Seats` tables by `City_ID` or `Theater_ID`. This ensures that a surge in Mumbai doesn't slow down bookings in Delhi.

### Preventing Database Bottlenecks:
For seat status, we can move the "availability check" entirely to **Redis**. 
1. Check Redis for seat status (Very fast).
2. If available, lock in Redis.
3. Only then hit the heavier SQL database to persist the transaction.

---

### Summary of the "No Double Booking" Logic:
1.  **User A** and **User B** select Seat 5.
2.  Both requests hit the **Booking Service**.
3.  The service tries to acquire a **Redis Lock** for `show_123_seat_5`.
4.  User A wins the lock (first millisecond). User B gets an "Already being booked" message.
5.  User A has 5 minutes to pay. If they fail, the Redis key expires, and the seat becomes "AVAILABLE" for the next person in the queue.

 I've reviewed the document. It's a solid foundation — the core flows are right. Here's a thorough review with gaps filled in.

---
<img width="1440" height="1356" alt="image" src="https://github.com/user-attachments/assets/2c77a708-8c7c-4989-bdc7-cebee2a8e845" />

## What's good

The document correctly nails the hardest parts — Redis distributed lock with `NX EX`, atomic SQL `WHERE status = 'AVAILABLE'`, and the thundering herd mitigation with a waiting room. These are exactly what interviewers want to hear.

---

##  Addon's improvements

### 1. : Seat map service

The document doesn't mention how the seat map is rendered in real time. This is a separate concern:

- The seat map is a **read-heavy snapshot** — serve it from Redis, not Postgres
- Every lock/unlock/confirm event should publish to a **WebSocket or SSE channel** so all users watching the same show see seats turn grey/red in real time
- Without this, two users on the same screen both see Seat 5 as available

```
User views seat map
       ↓
Redis HGETALL show:123:seats   ← O(1), entire seat state in one call
       ↓
WebSocket subscription: channel = show:123
       ↓
On any seat state change → broadcast to all subscribers
```

---

### 2.  Payment failure & rollback flow

The document mentions the Redis TTL as the rollback mechanism, but the full failure path needs to be explicit:

```
Payment fails / times out
       ↓
Payment Service publishes → "payment.failed" event (Kafka)
       ↓
Booking Service consumes:
  1. UPDATE booking SET status = 'CANCELLED'
  2. UPDATE seat   SET status = 'AVAILABLE'
  3. DEL Redis key  (don't wait for TTL — release immediately)
  4. Notify next person in waiting queue
```

Relying solely on Redis TTL expiry means the seat is unavailable for up to 5 minutes after a failed payment. Explicit rollback releases it in milliseconds.

---

### 3.  Overbooking guard (race condition edge case)

The document uses Redis lock + SQL constraint, but misses the scenario where Redis is partitioned or restarts. The database must be the **final source of truth**:

```sql
-- Optimistic locking with a version column
UPDATE seats
SET status = 'LOCKED', user_id = 'XYZ', version = version + 1
WHERE id = '123'
  AND status = 'AVAILABLE'
  AND version = <last_seen_version>;

-- If rows_affected = 0 → another transaction won, return conflict
```

This makes the DB the last line of defence even if Redis gives a false positive.

---

### 4.  DB Schema

Several critical tables are missing:

| Table | Missing columns |
|---|---|
| `Booking` | `payment_id`, `locked_until`, `version`, `created_at` |
| `Seat` | `status (AVAILABLE/LOCKED/BOOKED)`, `locked_by`, `locked_until` |
| `Payment` | `id`, `booking_id`, `gateway`, `amount`, `status`, `gateway_txn_id` |
| `User` | `id`, `phone`, `email`, `created_at` |
| `Ticket` | `id`, `booking_id`, `qr_code`, `issued_at` |

The `Payment` table is entirely absent — this is a critical table for reconciliation and refunds.

---

### 5. Ticket generation service

After payment confirmation, ticket generation is a distinct concern:

```
Payment confirmed
       ↓
Booking Service → publishes "booking.confirmed" (Kafka)
       ↓
Ticket Service consumes:
  - Generate unique QR code (booking_id + HMAC signature)
  - Store in S3 / object store
  - Write ticket record to DB
       ↓
Notification Service consumes same event:
  - Send SMS + Email with QR link
```

The QR code should be **signed** (HMAC or JWT) so it can be verified offline at the venue without a DB call.

---

### 6. API design

No API contracts are mentioned. Key endpoints:

```
GET  /shows?city=Mumbai&movie_id=X&date=2026-04-12   → available shows
GET  /shows/{show_id}/seats                           → seat map snapshot
POST /bookings                                        → initiate lock
     { show_id, seat_ids[], user_id }
     → returns { booking_id, locked_until }
POST /bookings/{booking_id}/confirm                   → after payment
DELETE /bookings/{booking_id}                         → explicit cancel
GET  /bookings/{booking_id}/ticket                    → fetch QR ticket
```

---

### 7. Monitoring & alerting

No observability story. For a system at this scale you need:

- **Redis lock contention rate** — spikes = thundering herd happening
- **Seat lock → confirm conversion rate** — if low, TTL is too long, seats are being wasted
- **Payment gateway latency** — p99 matters here, not just average
- **DLQ depth** (Kafka dead letter queue) — booking events that failed to process

---


## Summary 

| Area | First Draft | Addon's |
|---|---|---|
| Seat map | Static fetch | Redis snapshot + WebSocket real-time push |
| Payment failure | Redis TTL only | Explicit Kafka rollback consumer |
| Double booking guard | Redis + SQL constraint | 4 layers: Redis → optimistic lock → constraint → rollback |
| Schema | 3 tables | +`Payment`, `Ticket`, `User` tables with missing columns |
| Ticket generation | Mentioned briefly | Separate service, HMAC-signed QR, S3 storage |
| API design | Not mentioned | Full REST contract with all endpoints |
| Observability | Not mentioned | Lock contention, conversion rate, p99 latency, DLQ depth |
