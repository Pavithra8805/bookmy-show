# Async Order Processing Queue - ShowTime

This document describes the asynchronous payment processing pipeline used to keep the API responsive during peak sales. It explains why async processing is required, the exact SQS message format, the payment worker logic (success, retry, and failure), edge cases, and SQS configuration (visibility timeout, DLQ settings).

---

## Section 1 — Why Async?

Synchronous payment calls block API request handlers and therefore DB connections. At extreme scale (5 lakh users tapping simultaneously) this collapses the DB connection pool.

We reuse the connection formula from `CONCURRENCY.md`:

```
Connections held = (% non-payment RPS × avg_nonpayment_time_s × RPS_total)
                 + (% payment RPS × avg_payment_hold_time_s × RPS_total)
Pool exhausted when Connections held ≥ max_connections
```

Assume:
- Peak RPS_total for the first 60s: 500,000 / 60 ≈ 8,333 RPS
- Payment attempts: 20% of requests (0.2)
- Non-payment path avg DB time: 20 ms = 0.02 s
- Synchronous payment path holds DB connection while calling gateway: 800 ms = 0.8 s
- max_connections (PgBouncer pool): 500

Connections held per second:

Connections = 8,333 × (0.8 × 0.02 + 0.2 × 0.8)
            = 8,333 × (0.016 + 0.16)
            = 8,333 × 0.176 ≈ 1,466 connections

This far exceeds 500 — pool collapses.

If we make payments asynchronous (publish to SQS) and return quickly after holding seats and creating a `pending` booking, the payment path holds the DB only briefly (e.g., 50 ms). Recompute:

Connections = 8,333 × (0.8 × 0.02 + 0.2 × 0.05)
            = 8,333 × (0.016 + 0.01)
            = 8,333 × 0.026 ≈ 217 connections

Result: Pool stays well under 500 and overall latency for the API is sub-500ms. Conclusion: synchronous payments at scale are catastrophic; use async.

---

## Section 2 — Queue Message Format (SQS)

Message body (JSON):

```json
{
  "bookingId": "<uuid>",
  "userId": "<uuid>",
  "eventId": 1234,
  "seatIds": [111,112,113],
  "totalAmount": 2999.00,
  "currency": "INR",
  "paymentToken": "<payment-provider-token-or-nonce>",
  "paymentProvider": "razorpay",
  "idempotencyKey": "<client-or-generated-key>",
  "createdAt": "2026-05-12T12:00:00Z",
  "meta": { "clientIp": "1.2.3.4", "userAgent": "..." }
}
```

Field explanations:
- `bookingId` (UUID): Primary idempotency anchor. Worker updates booking by this id.
- `userId` (UUID): For notifications and cross-checks.
- `eventId` (int): Event context — useful for downstream analytics and routing.
- `seatIds` (array[int]): Exact seats to confirm; worker uses this to update `seats` and `booking_seats` without extra DB lookups.
- `totalAmount` (decimal): The amount to charge; prevents worker-only out-of-sync pricing anomalies.
- `currency` (string): Currency code for gateway calls and reconciliation.
- `paymentToken` (string): Payment provider token/nonce needed to charge.
- `paymentProvider` (string): Which gateway integration to use.
- `idempotencyKey` (string): Ensures repeated worker attempts don't double-charge the customer. This can be generated client-side or by the API server.
- `createdAt` (timestamp): For TTL and auditing.
- `meta` (object): Optional telemetry and debugging data (IP, UA).

Design note: The message contains everything the worker needs to decide and act without additional read queries; this reduces DB load and simplifies retry semantics.

---

## Section 3 — Payment Worker Logic (numbered steps)

1. Worker receives message from SQS.
2. Immediately check message shape & signature. If malformed → delete message and log & alert.
3. Idempotency check: `SELECT status FROM bookings WHERE id = bookingId`.
   - If booking.status == `confirmed` → delete message (already processed).
   - If booking.status == `processing` and worker sees a recent processing timestamp → treat as in-flight; extend visibility timeout and continue.
   - If booking.status == `pending` (expected) → proceed.
4. Acquire application-level safety: attempt to re-check & lock seats inside DB transaction:
   - BEGIN;
   - SELECT id, status FROM seats WHERE id = ANY(seatIds) FOR UPDATE ORDER BY id;
   - If any seat.status != 'held' OR held_by != booking.userId OR held_until < NOW() then: mark booking `failed`, ROLLBACK, delete message, notify user (seat no longer held).
   - Else: mark booking status `processing` with `processing_started_at = NOW()`.
5. Commit the transaction that set `processing` status, but keep the seats `held` until payment outcome.
6. Call payment gateway API with `paymentToken`, `totalAmount`, `currency`, and pass `idempotencyKey` to the gateway.

Success path:
7a. If gateway returns success (charge captured or authorized):
   - BEGIN TRANSACTION;
   - UPDATE bookings SET status='confirmed', payment_ref=<gateway_id>, confirmed_at=NOW() WHERE id=bookingId;
   - UPDATE seats SET status='booked', held_by=NULL, held_until=NULL WHERE id = ANY(seatIds);
   - INSERT INTO booking_seats (booking_id, seat_id) VALUES ...;
   - COMMIT;
   - Release any Redis locks for affected seats (Lua script safe delete).
   - Delete SQS message.
   - Send confirmation (SMS/email) to user.

Failure path:
7b. If gateway returns permanent failure (card declined / invalid token):
   - BEGIN TRANSACTION;
   - UPDATE bookings SET status='failed', payment_ref=<provider_response>, confirmed_at=NULL WHERE id=bookingId;
   - UPDATE seats SET status='available', held_by=NULL, held_until=NULL WHERE id = ANY(seatIds);
   - COMMIT;
   - Release Redis locks.
   - Delete SQS message.
   - Send failure notification to user with next steps.

Transient errors (timeouts, 5xx):
8. If gateway call times out or returns a retryable error:
   - Do NOT immediately release seats. Instead: extend DB `processing` metadata timestamp, extend SQS message visibility timeout, and retry with exponential backoff up to `maxAttempts`.
   - Use idempotencyKey so that repeated attempts don't double-charge on the provider side.

DLQ handling:
9. If message receives > `maxReceiveCount` (configured on SQS) without definitive success/failure (e.g., repeated gateway timeouts), SQS routes the message to DLQ.
   - On DLQ: alert on-call, send admin-facing ticket with message payload, mark booking `failed` and release seats via background job if still held.

Notes on atomicity:
- All DB updates that change seats or booking status must be done in transactions to avoid partial state.
- Release Redis locks only after DB commit and notification of success/failure.

---

## Section 4 — Edge Cases

Edge: API server crashes after SQS publish but before responding to the user
- What happens: The SQS message is already in the queue. A worker will pick it up and process payment; the booking proceeds normally. The user’s browser may see a network error or timeout.
- User experience: The app should instruct users to check SMS/email for confirmation and show an idempotent booking status page where they can input `bookingId` if available.
- Booking state: Worker will set `bookings.status` to `confirmed` or `failed` as appropriate; no duplicate bookings due to `bookingId` idempotency check.

Edge: Payment gateway returns ambiguous timeout (neither success nor explicit failure)
- Behavior:
  - Treat as transient. Use idempotencyKey and retry with exponential backoff (e.g., 1s, 2s, 4s) up to `N` attempts.
  - Extend SQS visibility timeout between retries to prevent other workers from picking the same message.
  - If after `N` attempts still ambiguous, route message to DLQ for human investigation and set booking to `failed` or `manual_review`.

Decision rules for retries:
- Retry on network timeouts, 5xx gateway errors, and rate-limit responses.
- Do NOT retry on 4xx permanent failures (card declined, invalid parameters).

---

## Section 5 — SQS Configuration

- Queue type: Standard SQS (best throughput; ordering not required). If strict ordering per user is required, use FIFO for per-user queues.
- Visibility Timeout: **120 seconds**
  - Justification: Payment calls are usually 200–2,000ms, but retries and downstream webhook verification can take longer (up to multiple seconds). We choose a conservative 120s to allow worker retries and downstream processing without the message becoming visible to other workers.
  - If a worker expects processing to take longer for specific gateways, use per-message ChangeMessageVisibility while processing.
- MaxReceiveCount before DLQ: **3**
  - Justification: Allows transient network flakiness to be retried twice, then surface to DLQ for human investigation. Keeps retry storm limited.
- Dead-Letter Queue (DLQ): Separate queue with CloudWatch alarm on messages >0
  - Operator flow: Alert on-call, inspect message payload, reconcile payment provider logs, and then either manually confirm booking or refund and release seats.

---

## Operational Notes
- Idempotency: Use `bookingId` + `idempotencyKey`. Ensure both API and worker store provider responses and use provider-side idempotency where available.
- Monitoring: Track message age, approximate age-of-oldest message, number of messages in-flight, and DLQ count.
- Security: Encrypt paymentToken at rest in SQS (KMS) and only keep ephemeral tokens in messages. Rotate and audit access.
- Cost: SQS is inexpensive and decouples spikes from worker scaling. Use short-lived Fargate tasks or Lambda for workers depending on workload patterns.
