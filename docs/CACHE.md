# Cache Design - ShowTime Ticketing System

This document specifies exact Redis key formats, TTLs, invalidation triggers, and the cache-aside behavior used by the system. The goal is to maximize read throughput during a sale while never caching per-seat authoritative state that could cause double-bookings.

## Principles
- Use cache-aside (delete on write) for availability counts and event details so the DB remains the single source of truth.
- Never cache individual seat `status` (available/held/booked) as authoritative.
- Prefer targeted invalidation over blind TTL-only caching for availability counts.

---

## Cache Entries (required)

1) Event details
- Key: `event:{event_id}`
- Value: JSON object { id, name, venue_id, starts_at, status, artist, metadata }
- TTL: 3600 seconds (1 hour)
	- Why: Event metadata changes rarely; 1 hour reduces DB reads during heavy sale traffic while still allowing timely updates for event edits.
- Invalidation trigger: UPDATE or DELETE on `events` row (on event status change, name change, cancellation).
- Invalidation action: `DEL event:{event_id}` (cache-aside). Next read rebuilds cache from DB.

2) Seat availability count per event per category
- Key: `availability:{event_id}:{category}`
- Value: integer (available seats count)
- TTL: 30 seconds
	- Why: Availability numbers are high-read during sales and can tolerate short staleness. 30s balances write volume vs user perception (5s is too aggressive, 5m too stale).
- Invalidation trigger(s):
	- Any seat status change in event_id and category (available → held, held → booked, booked → refunded, held → available)
	- After successful booking commit, seat release background job, or bulk seat updates
- Invalidation action: `DEL availability:{event_id}:{category}` immediately after DB transaction commits that modified seat(s). Use targeted deletion per affected categories.
	- If many categories are affected, delete keys for each category impacted.
	- If entire event seat inventory changes (rare), delete `availability:{event_id}:*` via scanned keys or maintain a set of category keys per event.

3) Static seat map layout
- Key: `seatmap:{event_id}`
- Value: JSON object describing sections, rows, seat counts, coordinates (does not include dynamic status)
- TTL: 86400 seconds (24 hours)
	- Why: Layout rarely changes; long TTL avoids repetitive DB hits during sale. Invalidate only on venue changes or event cancellation.
- Invalidation trigger: UPDATE on `events` or `venues` that affects layout, or admin changes to seating map
- Invalidation action: `DEL seatmap:{event_id}`

---

## What we WILL NOT cache (and why)
- Individual seat `status` keyed as `seat:{event_id}:{seat_id}`: NEVER cached as authoritative.
	- Risk: Two users reading cached `available` could both proceed to hold the seat and cause double-booking unless strict, immediate invalidation is guaranteed. The hybrid Redis lock + DB confirmation strategy already covers the concurrency requirement; caching seat status adds complexity and risk.
- Short-lived per-request transient locks are stored in Redis as `seat_lock:{event_id}:{seat_id}` but these are not seat-status caches — they are ephemeral locks with TTL and are managed separately.

---

## Invalidation Strategy (overall)
- Pattern: Cache-aside with targeted deletion.
	- On writes that affect cached data, update the authoritative DB first in a transaction, then delete the affected cache keys as part of the same transaction scope (or immediately after commit).
	- Avoid write-through because it complicates transactions and increases write amplification during the hot sale.

Rationale: Cache-aside keeps DB as source-of-truth, minimizes race windows (delete after commit), and avoids heavy Redis writes that write-through would cause during peak traffic.

---

## Code-level Pseudocode

// Helper: delete keys after DB commit
function onSeatStatusChange(seatIds, eventId) {
	// 1. Update DB in transaction (source-of-truth)
	db.beginTransaction()
	try {
		// update seats table for each seatId
		db.query('UPDATE seats SET status=$1, held_by=$2, held_until=$3 WHERE id = ANY($4)', ...)

		db.commit()
	} catch (err) {
		db.rollback()
		throw err
	}

	// 2. Invalidate affected cache keys (outside DB txn)
	// Derive categories affected by querying seat rows or by input metadata
	for (const category of affectedCategories) {
		redis.del(`availability:${eventId}:${category}`)
	}

	// 3. Optionally publish an event (SNS/SQS) for downstream consumers to recompute aggregates
}

// Cache-aside read pattern for availability count
async function getAvailabilityCount(eventId, category) {
	const key = `availability:${eventId}:${category}`
	const cached = await redis.get(key)
	if (cached !== null) return parseInt(cached, 10)

	// Miss - read from DB and repopulate
	const result = await db.query('SELECT COUNT(*) FROM seats WHERE event_id=$1 AND category=$2 AND status = \'' + 'available' + '\'', [eventId, category])
	const count = parseInt(result.rows[0].count, 10)
	// Set with TTL (cache-aside)
	await redis.setex(key, 30, String(count))
	return count
}

// Event details read
async function getEventDetails(eventId) {
	const key = `event:${eventId}`
	const cached = await redis.get(key)
	if (cached) return JSON.parse(cached)
	const row = await db.query('SELECT id,name,venue_id,starts_at,status FROM events WHERE id=$1', [eventId])
	if (!row) return null
	await redis.setex(key, 3600, JSON.stringify(row))
	return row
}

---

## Edge Cases & Operational Notes
- Race window: delete-after-commit may still allow a brief window where a client reads old cache immediately after the DB commit but before `DEL` executes. Mitigation: Invalidate synchronously in the same app flow immediately after commit; use short TTL (30s) as a fallback.
- Bulk invalidation: If many categories need invalidation frequently, maintain a Redis Set `availability_keys:{event_id}` that lists keys to delete, or use pattern deletes carefully with rate limiting to avoid Redis CPU spikes.
- Monitoring: Track cache hit rate, `DEL` failure rates, and TTL expirations during sale. Add alarms for abnormal patterns.

---

## Commit
After review, I'll commit this file to the `bookmy-show` branch.

