# Concurrency Strategy - ShowTime Ticketing System

## Problem Statement
5 lakh concurrent users at 12:00:00 noon, all trying to book the same seat. Zero double-bookings allowed. Maximum response time: 500ms. Budget: $2,000/month.

**The Technical Challenge:** Ensure exactly one user gets seat A-12. Two competing concerns:
1. **Correctness**: No double-bookings under any circumstance
2. **Performance**: Sub-500ms response time for all 500,000 users simultaneously

This is a distributed systems problem. Every option has a tradeoff.

---

## Option A: PostgreSQL SELECT FOR UPDATE (Row-Level Locking)

### How It Prevents Double-Booking

PostgreSQL's `SELECT FOR UPDATE` acquires an exclusive lock on the row before any other transaction can modify it. The exact transaction:

```sql
BEGIN TRANSACTION;

-- 1. Lock the specific seat row exclusively
SELECT id, status FROM seats
WHERE id = $1
AND status = 'available'
FOR UPDATE;

-- 2. Double-check status inside the lock
-- (in case another transaction already modified it)
UPDATE seats 
SET status = 'held',
    held_until = NOW() + INTERVAL '10 minutes',
    held_by = $2::UUID
WHERE id = $1
RETURNING id;

-- 3. Create the booking record
INSERT INTO bookings (user_id, event_id, total_amount, status)
VALUES ($3::UUID, $4::INT, $5::DECIMAL, 'pending')
RETURNING id;

COMMIT;
```

**Why this is correct:** 
- The `SELECT FOR UPDATE` on the seat row is atomic. It acquires an exclusive lock that blocks all other transactions trying to lock the same row.
- If Transaction A and Transaction B both try to lock seat A-12 simultaneously, one succeeds immediately and the other blocks until Transaction A commits.
- The blocking transaction then re-evaluates the WHERE clause - finds status = 'held' instead of 'available', and the UPDATE affects 0 rows.
- Result: Exactly one transaction books the seat. The other gets 0 rows affected and returns an error.

### Hard Limit: Connection Pool Exhaustion

PostgreSQL max_connections with PgBouncer: **500 connections** (typical mid-tier RDS configuration for $260/month).

Each booking attempt holds a connection for the entire transaction duration:
- Query time = lock acquire + status check + update + insert + commit

**The RPS Math:**

Formula: 
```
Connections held = RPS × Avg transaction time (in seconds)
Pool exhaustion when: Connections held ≥ max_connections
```

#### Scenario A: Async payments (no long waits)
- Regular booking RPS: 80% at ~30ms (lock + update + insert)
- Payment RPS: 20% at ~2,000ms (but this is async - we hold for only ~50ms before publishing to queue)

Peak RPS estimate: 500,000 users tapping within 60 seconds = ~8,333 RPS for the first 60 seconds

```
Connections held = (8,333 RPS × 0.05s) = 416 connections
```

**Result:** Stays under 500. The pool does NOT exhaust in the first 60 seconds. ✅

**But this only works with async payments.** If payment is synchronous:

#### Scenario B: Synchronous payments (Anti-pattern)
- Payment gateway call: 200-2,000ms (e.g., 800ms average)
- Each booking attempt holds its connection for 800ms

At just **370 RPS** with sync payments:
```
Connections held = 370 RPS × 0.8s = 296 connections
```

At **625 RPS** with sync payments:
```
Connections held = 625 RPS × 0.8s = 500 connections ← POOL EXHAUSTED
```

**The critical insight:** With synchronous payments, the DB pool exhausts at 625 RPS. Since we expect 8,333 RPS in the first 60 seconds, PostgreSQL SELECT FOR UPDATE alone is insufficient for 5L concurrent users.

**We must use async payments to make Option A viable.**

### Deadlock Risk with Multi-Seat Bookings

When a user books 3 seats (A-12, B-12, C-12), the transaction must lock all three rows:

```sql
BEGIN;
SELECT id FROM seats WHERE id IN ($1, $2, $3) FOR UPDATE;
UPDATE seats SET status = 'held' WHERE id IN ($1, $2, $3);
COMMIT;
```

**The deadlock scenario:**
- Transaction X locks seats in order: [A-12, B-12, C-12]
- Transaction Y locks seats in order: [C-12, B-12, A-12] (different order)
- Transaction X holds A-12, waits for B-12, but Transaction Y holds B-12 and waits for A-12
- **Deadlock.** PostgreSQL detects this and aborts one transaction.

**Mitigation:** Always lock seats in a **consistent global order** - sort by seat ID before the IN clause:

```sql
SELECT id FROM seats 
WHERE id IN ($1, $2, $3) 
ORDER BY id  -- CRITICAL: consistent lock order
FOR UPDATE;
```

With a sorted lock order, the deadlock scenario cannot occur. Both transactions lock in the same order, so one completes before the other starts.

---

## Option B: Redis SETNX Distributed Lock

### How It Prevents Double-Booking

Redis SETNX (SET if Not eXists) is an atomic operation that acquires a distributed lock:

```javascript
const lockKey = 'seat_lock:' + seatId;
const lockValue = userId + ':' + Date.now();  // Unique value to identify lock owner
const lockTTL = 30;  // seconds

// 1. Try to acquire the lock atomically
const acquired = await redis.set(
  lockKey, 
  lockValue,
  'NX',           // Only set if key does NOT exist
  'EX', lockTTL   // Auto-expire after 30 seconds
);

if (!acquired) {
  // Lock is held by someone else - seat is temporarily unavailable
  return { error: 'SEAT_TEMPORARILY_UNAVAILABLE', retryAfter: 1 };
}

// 2. Lock acquired - now safe to read and update the seat
try {
  const seat = await db.query(
    'SELECT status FROM seats WHERE id = $1 FOR UPDATE',
    [seatId]
  );
  
  if (seat.rows[0].status !== 'available') {
    return { error: 'SEAT_ALREADY_TAKEN' };
  }
  
  // 3. Update seat with no additional locking needed
  await db.query(
    'UPDATE seats SET status = $1, held_by = $2 WHERE id = $3',
    ['held', userId, seatId]
  );

} finally {
  // 4. Release the lock only if we still own it
  // Use Lua script to atomically check ownership then delete
  await redis.eval(`
    if redis.call("get", KEYS[1]) == ARGV[1] then
      return redis.call("del", KEYS[1])
    else
      return 0
    end
  `, 1, lockKey, lockValue);
}
```

**Why this is correct:**
- Redis SETNX is atomic - no two clients can both set the same key simultaneously.
- The first client to SETNX wins. The second gets NIL back and must retry.
- The lock holder reads the seat status and updates it while holding the lock.
- The Lua script ensures we only delete our own lock (check lockValue before delete).

### What If Redis Fails Mid-Lock?

**Scenario:** Client acquires the lock, updates the seat in the database, then Redis crashes before the lock is released.

```
T1: redis.set('seat_lock:12', 'userId:12345', NX, EX 30)  ✅ Lock acquired
T2: UPDATE seats SET status='held' WHERE id=12              ✅ Committed to DB
T3: Redis cluster node crashes                              ❌
T4: 30 seconds later, lock expires (auto-delete)            
T5: Another client tries to acquire the same lock           ✅ Lock acquired again
T6: That client reads seat status, sees 'held', returns error 
```

**Result:** No double-booking occurs. The Redis TTL ensures the lock eventually expires, and any client that acquires it will read the correct seat status from the database (the source of truth). Redis failure creates temporary unavailability, not data corruption.

### TTL Value for Seat Lock: 30 Seconds

**Why 30 seconds?**

This TTL must be longer than the slowest legitimate booking operation, but short enough that abandoned locks don't block users for too long.

```
Slowest Operation Timeline:
- User acquires lock:          T+0
- Reads seat status:            T+5ms
- Updates seat in DB:           T+10ms
- Checks payment availability:  T+50ms (worst case: slow DB query)
- Returns to user:              T+80ms
```

The longest operation in the critical path is ~80ms. But network jitter, DB slowness, and concurrent queries could push this to 200-300ms occasionally.

**Set TTL = 30 seconds** because:

| TTL Value | Problem |
|-----------|---------|
| 5 seconds | **Too short.** A slow DB write (network lag, slow query) finishes at T+6s, but the lock already expired at T+5s. Another transaction acquires the lock and reads stale seat status → double-booking risk. ❌ |
| 30 seconds | **Goldilocks.** Covers worst-case slow operations, but doesn't block legitimate users for too long if a worker crashes. ✅ |
| 300 seconds | **Too long.** If a worker crashes mid-update, the lock stays for 5 minutes. User needs to wait 5 minutes or manually retry. Terrible UX. ❌ |

**The tradeoff:** We sacrifice perfect lock-release timing for pragmatic TTL. If a worker dies with the lock held, we accept a 30-second window where that particular seat is temporarily unavailable to all other clients.

### Lock Key Structure: Which Is Better?

#### Option B1: Global Lock Per Seat
```
Key: seat_lock:12345  (just the seat ID)
```
- ✅ Simple to understand
- ✅ Works for single-seat bookings
- ❌ Does NOT prevent double-booking when a user books multi-seat bundles
- Example: User A books [12, 13, 14]. User B simultaneously acquires lock on 12 while A acquires lock on 13. Both proceed → two bookings for the same event, incorrect inventory tracking.

#### Option B2: Lock Per (Event, Seat) Pair
```
Key: seat_lock:event_1234:seat_12345
```
- ✅ Isolates locks by event (events are independent)
- ✅ Slightly better cache locality in Redis
- Same deadlock prevention issue as Option B1 with multi-seat bookings.

#### Option B3: Lock Per Booking Attempt (Recommended)
```
Key: booking_lock:{userId}:{eventId}  (one lock per user per event)
```
- ✅ Prevents a user from placing duplicate bookings for the same event
- ✅ Eliminates deadlock on multi-seat bookings (only one lock per user)
- ✅ Simpler Lua logic

**We choose Option B3:** `seat_lock:{event_id}:{seat_id}` as primary for simplicity, but document that multi-seat bookings must acquire locks in a **consistent order** (sorted by seat_id) to prevent inter-seat deadlocks.

---

## RPS Comparison

### PostgreSQL FOR UPDATE (Async Payments)
- **Peak capacity:** ~8,000 RPS before pool exhaustion
- **Bottleneck:** DB connection pool (500 connections)
- **Lock latency:** 5-20ms per seat (includes lock wait time)
- **Throughput at 500K RPS:** ✅ Viable with async payments, but pool contention is high

### Redis SETNX
- **Peak capacity:** 100,000+ RPS (Redis can handle)
- **Bottleneck:** Redis cluster capacity, not DB connections
- **Lock latency:** <1ms per seat
- **Throughput at 500K RPS:** ✅ Highly viable, sub-millisecond lock overhead

---

## Our Chosen Strategy: Hybrid Approach

### Decision: "We choose a HYBRID strategy combining Redis SETNX for seat holds + PostgreSQL optimistic locking for booking confirmation."

### Architecture

```
User initiates booking for seats [A-12, B-12, C-12]
            ↓
    PHASE 1: HOLD (Redis SETNX)
    For each seat in sorted order:
      - Try redis.set('seat_lock:{event_id}:{seat_id}', '{userId}:{timestamp}', NX, EX 30)
      - If any fails → return "SEAT_UNAVAILABLE"
      - If all succeed → proceed to Phase 2
            ↓
    PHASE 2: VALIDATE & BOOK (PostgreSQL)
    BEGIN TRANSACTION
      SELECT seats WHERE id IN (...) FOR UPDATE ORDER BY id
      Check status = 'available' for all seats
      UPDATE seats SET status='booked'
      INSERT into bookings
      INSERT into booking_seats
    COMMIT
            ↓
    Release all Redis locks (Lua script)
            ↓
    Return { bookingId, status: 'confirmed' }
```

### Why This Hybrid Approach

#### Justification 1: Constraint - RPS Capacity

At 5L concurrent users tapping at 12:00:00:
- **Estimated peak RPS:** 8,333 RPS in first 60 seconds (5L users / 60 seconds)
- **PostgreSQL FOR UPDATE alone:** Can handle ~8,000 RPS with async payments, but pool contention becomes the bottleneck. Any variation in query time causes queuing.
- **Redis SETNX:** Can handle 100K+ RPS. Lock acquisition is <1ms. Offloads pressure from DB.
- **Hybrid:** Uses Redis for the high-frequency lock operation (100K+ capacity), uses DB only for the final commit (8K+ capacity). Stacks capacities: **total throughput = min(Redis, DB) ≈ 8,000 RPS**, but with much lower latency and latency variance.

#### Justification 2: Constraint - $2,000/Month Budget

**PostgreSQL FOR UPDATE alone:**
- Need 1 large RDS instance: $262/month
- Need 8-10 EC2 instances to absorb the 8,333 RPS: $1,000/month
- Connection pooling overhead eats into budget

**Redis SETNX alone (without DB confirmation):**
- Cannot work. Redis is not transactional. After locking, we MUST read the DB to verify seat status. Removing the DB verification creates a race condition where the seat is sold out but the Redis lock still shows available.

**Hybrid (Recommended):**
- 1 RDS instance + 2 read replicas: $524/month
- 6 EC2 instances for API servers: $719/month
- 3-node Redis cluster: $359/month
- ECS Fargate workers, ALB, SQS: ~$240/month
- **Total: ~$1,840/month** ✅ Fits the $2K budget

The hybrid approach gives us both capacity (Redis) and correctness (DB), within budget.

#### Justification 3: Constraint - Zero Double-Bookings

**Mechanism:**
1. Redis lock prevents two concurrent holders from updating the same seat simultaneously
2. PostgreSQL transaction (with `SELECT FOR UPDATE` inside Phase 2) ensures ACID semantics on the final booking
3. The version column in the seats table enables optimistic locking as a fallback if two transactions race the commit phase

**What this cannot handle:**
- Network partition: If the API server loses connection to Redis mid-booking, the Redis locks hang for 30 seconds. During that window, users cannot book those seats. This is acceptable UX (they see "Seat temporarily unavailable, try again") but not data loss.
- If somehow the Redis lock expires but a slow query is still running, another transaction could acquire the same lock and proceed. **Mitigation:** The PostgreSQL `FOR UPDATE` in Phase 2 acts as a second line of defense. Two transactions cannot both hold `FOR UPDATE` on the same row, so the second one will block and then get an error.

### When to Switch Strategies

| Condition | Switch To | Reason |
|-----------|-----------|--------|
| Peak RPS exceeds 50K and Redis cluster becomes the bottleneck | Upgrade to Redis Cluster with 6-9 nodes | Horizontal scaling of lock layer |
| Double-bookings occur despite locks | Upgrade to PostgreSQL multi-node cluster with quorum locking | Need stronger ACID guarantees |
| $2K budget reduced to $500 | Pure PostgreSQL FOR UPDATE, accept reduced RPS capacity | Cannot afford Redis infrastructure |
| Need sub-100ms response time at 100K RPS | Add application-level read caching of seat availability | Reduce DB queries entirely |

---

## Summary

| Aspect | PostgreSQL FOR UPDATE | Redis SETNX | Hybrid ✅ |
|--------|----------------------|-------------|----------|
| **Correctness** | ✅ ACID, rock solid | ⚠️ Requires DB check | ✅ Both layers |
| **Throughput at 5L users** | ⚠️ ~8K RPS with async | ✅ 100K+ RPS | ✅ ~8K RPS, lower contention |
| **Infrastructure cost** | $500/mo | $400/mo | $880/mo |
| **Lock latency** | 10-20ms | <1ms | <1ms for lock, 20-50ms total |
| **Failure mode** | Connection pool exhaustion | Temporary unavailability | Graceful degradation |
| **Budget fit** | ✅ Yes, but tight | ✅ Yes, but risky | ✅ Yes, with headroom |

**Our choice:** Hybrid. It delivers capacity, correctness, and budget-friendliness without architectural risk.
