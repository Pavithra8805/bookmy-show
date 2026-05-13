## Update 1: Per-User Seat Hold Cap

**Triggered by:** Panel Question 3 - what stops one user from holding 200 seats?

**What changed:**
Added a Redis counter keyed by user ID, `holds:{userId}:count`, with a max of 8 concurrent active holds. The API checks this counter before acquiring any seat lock. The counter is incremented when a hold is successfully created and decremented when the hold expires, is released, or the booking is confirmed. During active sale windows, hold TTL is shortened from 10 minutes to 3 minutes so abandoned seats return to inventory faster.

**Why this is necessary:**
The original design prevented double-booking, but it still allowed one user to monopolize a large number of seats by opening many tabs or devices. This update limits abuse from a single account/session while reducing the time seats remain unavailable if the user abandons checkout.

**What it costs:**
One extra Redis read and counter update on the hot path, plus a small amount of bookkeeping when a hold is released or expires. Operationally, it also requires the counter and the hold lifecycle to stay in sync.

**What it still doesn't solve:**
A determined attacker using many accounts can still distribute abuse across identities. Fully solving that requires stronger account verification or payment-method preauthorization, which is a product decision beyond the booking lock itself.

## Update 2: SQS Publish Circuit Breaker with Booking Status Polling

**Triggered by:** Panel Question 4 - what happens if SQS is down during booking?

**What changed:**
Added a circuit breaker around SQS publish calls. If publish attempts fail for 60 seconds during an active sale window, the API stops pretending the async path is healthy and falls back to a synchronous reduced-throughput booking path. The API also exposes `GET /bookings/{id}` so the client can poll booking state and see `pending`, `confirmed`, or `failed` while the payment workflow is still in progress.

**Why this is necessary:**
The original design assumed the queue would absorb the spike, but it did not make failure visible enough to the user or operator when SQS itself was unavailable. The circuit breaker prevents a silent dead-end where the API returns success but nothing is actually being processed, and the status endpoint gives the client a way to track in-flight bookings.

**What it costs:**
The fallback path adds more application logic and a slower degraded mode during queue outages. The polling endpoint also adds a small amount of read traffic, though the status lookups are cheap compared with a full booking flow.

**What it still doesn't solve:**
If both SQS and the synchronous fallback are saturated for a prolonged period, throughput still drops. The system prefers slower confirmed processing over silently dropping bookings, but it does not eliminate the impact of a major queue outage.
