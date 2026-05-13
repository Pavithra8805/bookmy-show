## Decision: Redis SETNX for Seat Holds

**Context:**
5 lakh concurrent users at sale start, zero double-bookings allowed, and a 500ms API response target. The seat hold path has to be fast enough for the burst while still preserving correctness under contention.

**Options considered:**
1. PostgreSQL `SELECT FOR UPDATE` - strong correctness and no extra infrastructure, but it keeps database connections occupied during the lock and update path. At peak load this pushes the pool toward exhaustion, especially once payment handling is involved.
2. Optimistic locking only with `seats.version` - simple and correct on conflict, but high-contention seat selection creates many retries and a thundering herd against the primary database.
3. Redis `SETNX` distributed lock with PostgreSQL confirmation - chosen because Redis absorbs the high-frequency lock attempts while PostgreSQL remains the source of truth for final booking state.

**Why chosen:**
Redis `SETNX` gives sub-millisecond lock acquisition and can sustain 100K+ lock operations per second, which is materially better than pushing all contention through PostgreSQL. The design keeps final confirmation on PostgreSQL so correctness is preserved. This balances the 8,333 RPS burst from the sale window against the database pool limit described in `CONCURRENCY.md`.

**Tradeoffs accepted:**
Redis adds operational complexity and a failure mode where seat holds become temporarily unavailable if the cache layer is unhealthy. We accept that availability degradation because we do not accept double-bookings.

**Revision trigger:**
If sustained booking traffic stays well below a few thousand RPS, or if the Redis operational overhead becomes harder to justify than the performance gain, PostgreSQL row locks alone would be simpler.

## Decision: Event-Driven Cache Invalidation with Short TTL Fallback

**Context:**
Availability counts and event metadata are read far more often than they are written, but stale cache entries must not become authoritative. The system needs fast reads without turning Redis into a second source of truth.

**Options considered:**
1. TTL-only caching - easiest to operate, but it leaves stale data in place for the whole TTL window and makes fresh writes visible too slowly during a sale.
2. Write-through caching - keeps cache current, but every write now has to update Redis in the hot path, increasing write amplification and coupling cache health to booking correctness.
3. Event-driven invalidation plus TTL fallback - chosen because writes update PostgreSQL first, then the affected keys are deleted immediately after commit, with TTLs only acting as a safety net.

**Why chosen:**
Targeted invalidation gives the freshest possible cache behavior without making Redis part of the correctness boundary. A 30-second availability TTL and 1-hour event TTL are short enough to bound staleness but long enough to reduce repeated DB reads during the sale surge.

**Tradeoffs accepted:**
There is still a small race window between DB commit and cache deletion, and stale reads can happen briefly. That is acceptable because booking confirmation always revalidates against PostgreSQL primary.

**Revision trigger:**
If stale availability becomes a meaningful user-experience problem, the design should move toward stronger cache coherence or more explicit publish/subscribe invalidation for the affected keys.

## Decision: UUID for Booking IDs

**Context:**
Booking identifiers are exposed through user-facing APIs and must not be guessable. The ID also needs to support retries and idempotency for transient failures.

**Options considered:**
1. SERIAL integer IDs - compact and efficient, but predictable and enumerable. A user could probe `/bookings/1`, `/bookings/2`, and so on.
2. UUID v4 - chosen because it is non-enumerable, safe to expose, and can be generated client-side before the API call.

**Why chosen:**
UUID v4 gives 128 bits of randomness, which is effectively impossible to enumerate. That makes it appropriate for public booking references and lets retrying clients reuse the same booking ID without creating duplicates.

**Tradeoffs accepted:**
UUIDs are larger than integers and slightly more expensive for index lookups. The storage and index cost is acceptable because the system prioritizes security and idempotency over the minor space savings of SERIAL.

**Revision trigger:**
If booking IDs ever become internal-only and never appear in user-facing APIs, a compact integer key could be revisited for a small storage and index-size gain.

## Decision: SQS Visibility Timeout of 120 Seconds

**Context:**
Payment processing is asynchronous, but workers still need enough time to finish retries and downstream calls without the same message being processed twice.

**Options considered:**
1. Short timeout such as 20-30 seconds - improves retry speed, but risks redelivery while a slow payment attempt is still legitimately in flight.
2. Longer timeout such as 120 seconds - chosen because it safely covers gateway latency, retry logic, and downstream bookkeeping while keeping the queue from becoming invisible for too long.

**Why chosen:**
Most payment calls are far below 120 seconds, but the extra headroom protects against transient gateway slowness and allows the worker to extend visibility if needed. This keeps the queue durable without forcing the API to hold the user request open.

**Tradeoffs accepted:**
A longer visibility timeout means a failed worker can delay redelivery longer than ideal. That is acceptable because the system favors preventing duplicate payment processing over immediate retry speed.

**Revision trigger:**
If the payment provider and worker timings become consistently much faster, the timeout could be reduced. If some gateways need more time, per-message visibility extension can supplement the baseline.
