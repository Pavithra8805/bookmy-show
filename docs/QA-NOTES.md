## Q1: What happens if Redis crashes mid-lock?

**Answer notes:** Redis failure stops new lock acquisition. API returns 503 instead of proceeding without a lock. PostgreSQL primary still rechecks seat state with `version` / row update so the system degrades to slower contention, not double-booking.

## Q2: At what concurrent users does the DB become the bottleneck?

**Answer notes:** DB is the bottleneck once the booking path starts holding connections too long. Synchronous payment pushes pool exhaustion around a few hundred RPS; the async SQS path keeps the hot request under the limit for the 5 lakh-user burst.

## Q3: What stops one user from holding 200 seats?

**Answer notes:** Per-user Redis counter capped at 8 holds, checked before lock acquisition. Seat holds also have a short TTL and are tied to the same session token, so tabs share the same limit.

## Q4: Your bill spikes to $3,200 this month - what's your plan?

**Answer notes:** Treat the spike as event-specific peak cost, not steady-state. Use spot instances for workers, tighten CloudFront cache hit rate, and tune scale-in cooldown so idle capacity disappears faster after the sale.

## Q5: Why not PostgreSQL row locking instead of Redis?

**Answer notes:** Row locking is correct but holds DB connections longer and becomes the throughput bottleneck at sale scale. Redis SETNX gives sub-ms locks and absorbs contention; PostgreSQL remains the source of truth for final confirmation.
