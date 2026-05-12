# bookmy-show

# ShowTime - BookMyShow Competitor Ticketing System

## Mission
Exclusive ticketing rights for 3 Coldplay India shows. Handle 5 lakh concurrent users at 12:00:00 noon. Zero double-bookings. $2,000/month AWS budget. 500ms max API response time.

---

## Part A: System Design (Foundation)

### Constraint Analysis

#### Constraint 1: 5 Lakh Concurrent Users at 12:00:00 Noon

**Peak RPS (Requests Per Second) Calculation:**
- 5 lakh concurrent users = 500,000 simultaneous users
- Assuming each user submits 1-2 booking requests in the first 30 seconds
- Peak load estimate: ~500,000 users × 1 booking request / 30 seconds = **~16,667 RPS at T+0**
- Sustained for ~60 seconds during the initial stampede, then taper off

**Which component hits its limit first?**
1. **Database connections** (PostgreSQL default: ~100-500 connections with PgBouncer)
   - At 16,667 RPS with 100ms DB latency per request: we need ~1,667 concurrent DB connections
   - This exceeds even a large RDS instance's max_connections
   - **Verdict: DB is the first bottleneck**

2. **API servers** (6 × t3.xlarge = 24 vCPU, ~5,000 requests/vCPU/sec = 120K RPS capacity)
   - With proper async handling, API servers can handle the load
   - **Not a bottleneck if DB offloading is done right**

3. **Redis (for seat locks)**
   - Redis can handle 100,000+ ops/second on a single node
   - 3-node cluster (our design) handles 300K+ ops/second
   - **Not a bottleneck**

**Strategy to solve:** Minimize DB connection hold time by using Redis for distributed locks (50ms) and async queues (SQS) for payment processing (don't hold DB connections for payment calls).

---

#### Constraint 2: Zero Acceptable Double-Bookings

**What is a double-booking technically?**
- Two rows in `booking_seats` table referencing the same `seat_id` with different `booking_id` values where both bookings have `status='confirmed'`
- This happens when two concurrent requests both read `seats.status='available'`, both pass validation, and both UPDATE the seat
- Result: Legal liability, contract breach with the promoter

**Mechanisms to prevent double-booking:**

| Mechanism | How It Works | Trade-offs |
|-----------|-------------|-----------|
| **PostgreSQL SELECT FOR UPDATE** | Lock the row at DB level before UPDATE | Correct but bottlenecks at 500K concurrent users (connection pool exhausted) |
| **Redis SETNX Distributed Lock** | Atomic "set if not exists" on Redis with TTL | Non-blocking, sub-millisecond, but requires Redis cluster and careful TTL tuning |
| **Optimistic Locking (version column)** | Read version, UPDATE only if version matches | No locks, but requires retry logic on conflicts |
| **Hybrid: Redis for holds + Optimistic for confirmation** | Redis lock for 10-minute seat hold, then optimistic locking for final booking | Combines throughput of Redis with ACID guarantees of optimistic locking |

**Our choice:** Hybrid approach (Redis SETNX for seat holds during booking initiation + optimistic locking with `seats.version` for final confirmation).

---

#### Constraint 3: $2,000/Month AWS Budget

**What does $2,000/month actually buy?**

| Component | Configuration | Cost/Month | Purpose |
|-----------|---|---|---|
| **API Servers** | EC2 t3.xlarge × 6 | $719 | Handle booking API requests |
| **Database (Primary)** | RDS db.r6g.xlarge | $262 | PostgreSQL, seat source of truth |
| **Database (Read Replicas)** | RDS db.r6g.large × 2 | $262 | Scale read queries for seat maps, avoid primary bottleneck |
| **Redis Cache** | ElastiCache cache.r6g.large × 3 | $359 | Distributed locks, seat availability cache |
| **Payment Workers** | ECS Fargate (0.25 vCPU × 0.5GB × 10 avg) | $73 | Async payment processing |
| **Networking** | ALB + CloudFront + SQS | $165 | Load balancing, CDN, queue |
| **TOTAL** | | **~$1,840/month** | ✅ Under budget |

**Key constraints:**
- Cannot use multi-AZ RDS (would double DB cost → $1,500+)
- Must use spot instances or reserved instances for compute
- Must cache aggressively to avoid hitting RDS beyond ~100 queries/sec

**If budget were $500/month:**
- Single t3.medium API server ($35/month)
- Single RDS db.t3.micro ($42/month)
- Single ElastiCache cache.t3.micro ($15/month)
- **Cannot support 5L concurrent users** - would need vertical scaling or accept lower SLA
- **Design changes needed:** Implement PostgreSQL row-level locking (no Redis cost), accept 2-5 second response times, or limit concurrent bookings to 10,000 users

---

### Design Decisions Summary

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Primary Key (bookings) | UUID v4 | Unpredictable - prevents user enumeration; can be generated client-side for idempotency |
| Primary Key (seats) | SERIAL INT | Compactness - event with 10,000 seats means INT is sufficient; reduces index size |
| Concurrency Strategy | Redis SETNX + Optimistic Locking | Throughput at scale without DB bottleneck; Redis failure falls back to retry |
| Seat Hold Duration | 10 minutes | Long enough for payment (avg 2-3s), short enough to release abandoned holds |
| Cache TTL for Availability | 30 seconds | Stale counts acceptable, targeted invalidation on seat changes keeps data fresh |
| Async Payment | SQS Queue | Decouples payment latency from API latency; workers scale independently |
| Async Message Retry | 3 attempts + DLQ | Failed payments moved to DLQ for manual review |
| Read Replicas | 2 replicas | Seat map reads distributed; primary handles writes |

---

## Part B: Live Panel Roast (Defense)

See [ARCHITECTURE.md](docs/ARCHITECTURE.md) for full system diagram and design defense.

---

## Repository Structure

```
bookmy-show/
├── README.md                 (this file - overview + constraints)
├── docs/
│   ├── SCHEMA.md            (PostgreSQL DDL, constraints, indexes, commentary)
│   ├── CONCURRENCY.md       (Redis vs PostgreSQL analysis + chosen strategy)
│   ├── CACHE.md             (What to cache, TTL, invalidation triggers)
│   ├── QUEUE.md             (SQS message format, worker logic, failure handling)
│   ├── ARCHITECTURE.md      (Full system diagram + component interactions)
│   ├── DESIGN-DECISIONS.md  (Detailed rationale for each choice)
│   └── DESIGN-UPDATES.md    (Changes made during Part B panel discussion)
└── assets/                  (Architecture diagrams, ERD, etc.)
```

---

