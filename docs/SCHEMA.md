# Database Schema - ShowTime Ticketing System

## Overview

This document defines the complete PostgreSQL schema for the ShowTime ticketing system. Every table, constraint, index, and column is designed to support:
- **Zero double-bookings** through optimistic locking (`seats.version`)
- **High-throughput booking** through careful indexing
- **Efficient cache invalidation** through targeted queries
- **Audit trails** through timestamps and version tracking

---

## Complete DDL (Data Definition Language)

### 1. VENUES Table

```sql
CREATE TABLE venues (
  id            SERIAL PRIMARY KEY,
  name          VARCHAR(255) NOT NULL,
  city          VARCHAR(100) NOT NULL,
  capacity      INT NOT NULL CHECK (capacity > 0),
  address       VARCHAR(500),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_venues_city ON venues(city);
CREATE INDEX idx_venues_name ON venues(name);
```

**Purpose:** Defines physical venues where events occur. Referenced by the events table.

**Key fields:**
- `id`: Surrogate key for the venue
- `capacity`: Total seats at this venue (informational; actual seat counts per event may differ)
- `created_at`: When the venue was added to the system

**Indexes:**
- `idx_venues_city`: Allows filtering events by city (e.g., "Show me all events in Mumbai")
- `idx_venues_name`: Allows finding venues by name

---

### 2. EVENTS Table

```sql
CREATE TABLE events (
  id            SERIAL PRIMARY KEY,
  name          VARCHAR(255) NOT NULL,
  artist        VARCHAR(255),
  venue_id      INT NOT NULL REFERENCES venues(id) ON DELETE CASCADE,
  starts_at     TIMESTAMPTZ NOT NULL,
  ends_at       TIMESTAMPTZ,
  total_seats   INT NOT NULL CHECK (total_seats > 0),
  status        VARCHAR(20) NOT NULL DEFAULT 'upcoming'
                  CHECK (status IN ('upcoming','on_sale','sold_out','cancelled')),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_events_starts ON events(starts_at);
CREATE INDEX idx_events_status ON events(status);
CREATE INDEX idx_events_venue ON events(venue_id);
CREATE UNIQUE INDEX idx_events_venue_starts ON events(venue_id, starts_at);
```

**Purpose:** Represents ticketed events (concerts, shows, etc.). Links to venues.

**Key fields:**
- `status`: Tracks event lifecycle (upcoming → on_sale → sold_out → cancelled)
- `total_seats`: Used for sold-out detection; validated against actual seat counts
- `starts_at`: Critical for caching (e.g., upcoming events vs past events)

**Constraints:**
- `ON DELETE CASCADE`: If a venue is deleted, all its events cascade-delete (use cautiously in production)
- `CHECK (status IN (...))`: Ensures only valid status values

**Indexes:**
- `idx_events_starts`: Enables queries like "get all events in next 7 days"
- `idx_events_status`: Enables filtering by lifecycle (e.g., only show on_sale events)
- `idx_events_venue_starts`: Composite index for queries like "all events at venue X between time Y and Z"

---

### 3. SEATS Table

```sql
CREATE TABLE seats (
  id            SERIAL PRIMARY KEY,
  event_id      INT NOT NULL REFERENCES events(id) ON DELETE CASCADE,
  section       VARCHAR(50) NOT NULL,
  row_label     VARCHAR(10) NOT NULL,
  number        INT NOT NULL,
  category      VARCHAR(30) NOT NULL,
  price         DECIMAL(10,2) NOT NULL CHECK (price > 0),
  status        VARCHAR(20) NOT NULL DEFAULT 'available'
                  CHECK (status IN ('available','held','booked')),
  held_until    TIMESTAMPTZ,
  held_by       UUID,
  version       INT NOT NULL DEFAULT 0,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_seats_event_pos
  ON seats(event_id, section, row_label, number);
CREATE INDEX idx_seats_event_status
  ON seats(event_id, status);
CREATE INDEX idx_seats_held_until
  ON seats(held_until)
  WHERE status = 'held';
```

**Purpose:** Individual seat records. The most critical table - stores the seat's state (available/held/booked).

**Key fields:**

| Field | Type | Purpose |
|-------|------|---------|
| `event_id` | INT FK | Which event this seat belongs to |
| `section` | VARCHAR | Seating section (e.g., "A", "VIP-Front") |
| `row_label` | VARCHAR | Row identifier (e.g., "1", "AA") |
| `number` | INT | Seat number within the row (e.g., 12) |
| `category` | VARCHAR | Price category (e.g., "Premium", "General") |
| `price` | DECIMAL | Seat price in rupees |
| `status` | VARCHAR | Current state: available / held / booked |
| `held_until` | TIMESTAMPTZ | When this hold expires (for seat release background job) |
| `held_by` | UUID | Which user holds this seat (for audit) |
| **version** | **INT** | **Optimistic lock counter - see commentary below** |

**Constraints:**
- `UNIQUE (event_id, section, row_label, number)`: Ensures no duplicate seat positions within an event
- `CHECK (price > 0)`: No free or negative-price seats allowed
- `CHECK (status IN ...)`: Only valid states allowed

**Indexes:**

| Index | Purpose |
|-------|---------|
| `idx_seats_event_pos` | **Workhorse index**: Find a specific seat by event+section+row+number (during booking) |
| `idx_seats_event_status` | Find all seats of a given status for an event (e.g., count available seats) |
| `idx_seats_held_until` | Find held seats whose hold has expired (for background release job) - partial index keeps it small |

---

### 4. USERS Table

```sql
CREATE TABLE users (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email         VARCHAR(255) NOT NULL UNIQUE,
  phone         VARCHAR(20) UNIQUE,
  name          VARCHAR(100) NOT NULL,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
```

**Purpose:** Registered user accounts.

**Key fields:**
- `id`: UUID ensures unpredictable user IDs (prevents enumeration attacks)
- `email`, `phone`: Unique identifiers; emails especially used for confirmations and promotions
- `created_at`: User registration timestamp

**Constraints:**
- `UNIQUE(email)`: One account per email address
- `UNIQUE(phone)`: Optional, but if used, enforces one account per phone

---

### 5. BOOKINGS Table

```sql
CREATE TABLE bookings (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  event_id      INT NOT NULL REFERENCES events(id) ON DELETE CASCADE,
  status        VARCHAR(20) NOT NULL DEFAULT 'pending'
                  CHECK (status IN ('pending','confirmed','failed','refunded')),
  total_amount  DECIMAL(10,2) NOT NULL CHECK (total_amount >= 0),
  payment_ref   VARCHAR(100),
  confirmation_sent_at  TIMESTAMPTZ,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  confirmed_at  TIMESTAMPTZ,
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_bookings_user
  ON bookings(user_id, created_at DESC);
CREATE INDEX idx_bookings_event
  ON bookings(event_id);
CREATE INDEX idx_bookings_status
  ON bookings(status)
  WHERE status IN ('pending','failed');
CREATE INDEX idx_bookings_payment_ref
  ON bookings(payment_ref)
  WHERE payment_ref IS NOT NULL;
```

**Purpose:** Order records linking users to events.

**Key fields:**

| Field | Type | Purpose |
|-------|------|---------|
| `id` | UUID | Unpredictable booking ID (prevents enumeration) |
| `user_id` | UUID FK | Which user made this booking |
| `event_id` | INT FK | Which event is being booked |
| `status` | VARCHAR | Lifecycle: pending → confirmed / failed → refunded |
| `total_amount` | DECIMAL | Total rupees paid/to-be-paid |
| `payment_ref` | VARCHAR | Reference from payment gateway (for reconciliation) |
| `confirmed_at` | TIMESTAMPTZ | When payment was confirmed (null until confirmation) |

**Constraints:**
- `CHECK (status IN ...)`: Only valid booking statuses
- `CHECK (total_amount >= 0)`: Amount cannot be negative

**Indexes:**

| Index | Purpose |
|-------|---------|
| `idx_bookings_user` | Query user's booking history: "Show me all bookings by user X, newest first" |
| `idx_bookings_event` | Query all bookings for an event (for event analytics) |
| `idx_bookings_status (partial)` | **Critical for payment workers**: Find all pending/failed bookings that need processing. Partial index keeps it small (only ~0.1% of bookings are unresolved). |
| `idx_bookings_payment_ref` | Reconciliation: "Find booking by payment gateway reference" |

---

### 6. BOOKING_SEATS Table (Junction)

```sql
CREATE TABLE booking_seats (
  booking_id  UUID NOT NULL REFERENCES bookings(id) ON DELETE CASCADE,
  seat_id     INT NOT NULL REFERENCES seats(id) ON DELETE CASCADE,
  PRIMARY KEY (booking_id, seat_id)
);

CREATE INDEX idx_booking_seats_seat
  ON booking_seats(seat_id);
```

**Purpose:** Maps many-to-many relationship between bookings and seats (one booking can include multiple seats).

**Key fields:**
- `booking_id`: Which booking
- `seat_id`: Which seat
- Composite PK ensures no duplicate seat entries per booking

**Indexes:**
- `idx_booking_seats_seat`: Enables reverse lookup: "In how many bookings is seat S-12 involved?" (for debugging double-bookings)

---

## Schema Design Decisions - Detailed Commentary

### 1. Why UUID for `bookings.id` Instead of SERIAL?

**Decision:** Use UUID v4 for booking IDs.

**Why NOT SERIAL integers?**
- SERIAL (1, 2, 3, ...) is **predictable**
- An attacker can enumerate all bookings: `GET /bookings/1`, `/bookings/2`, `/bookings/3`, etc.
- Exposes other users' booking details (privacy violation)

**Why UUID v4?**
- Random 128-bit identifier: **2^128 ≈ 3.4 × 10^38** possible values
- Impossible to guess or enumerate
- Can be **generated client-side** before the API call (enables idempotency on retries)

**Trade-off:**
- UUIDs are larger (16 bytes vs 4 bytes for INT)
- Slightly slower index lookups (longer key comparison)
- **Worth it:** Security + idempotency >> minor performance cost

**Idempotency benefit example:**
```
Client generates: bookingId = "7f8a3c2b-4d9e-11ec-..."
User clicks "Book" → API creates booking with this ID
User's internet drops → client retries with same bookingId
API sees bookingId exists, returns 200 (idempotent)
User never gets duplicate booking
```

---

### 2. Why Does `seats` Have a `version` Column?

**Decision:** Add `version INT DEFAULT 0` to seats table.

**Purpose:** Enable **optimistic locking** - read the row, remember the version, update only if the version still matches.

**How it prevents double-booking:**

```sql
-- Scenario: Two users both try to book seat A-12

-- USER 1: Initial read
SELECT id, status, version FROM seats WHERE id = 100;
-- Returns: id=100, status='available', version=0

-- USER 2: Initial read (happens at same time)
SELECT id, status, version FROM seats WHERE id = 100;
-- Returns: id=100, status='available', version=0

-- USER 1: Try to update
UPDATE seats SET status='booked', version=1
WHERE id=100 AND version=0;
-- Matches! version becomes 1. UPDATE succeeds (1 row).

-- USER 2: Try to update
UPDATE seats SET status='booked', version=1
WHERE id=100 AND version=0;
-- No match! version is now 1, not 0. UPDATE fails (0 rows).
-- USER 2 gets CONFLICT and must retry or fail the booking.
```

**Why not use locks?**
- Locks block during the entire UPDATE (slower)
- Locks exhaust DB connection pool at 500K concurrent users
- Optimistic locking allows reads without locks, retries on conflict

**Trade-off:**
- Requires retry logic in application code
- Under high contention (both users want same seat), many requests fail/retry
- **Worth it:** No lock contention, no connection pool exhaustion

---

### 3. Why `held_until` on Seats Instead of Application-Level Holds?

**Decision:** Store `held_until` timestamp on each seat; release via background job.

**Why NOT just hold seats in application memory?**
- If API server crashes, held seats are lost (users can re-book already-held seats)
- No audit trail of who held what and when
- Difficult to implement seat release at exact timeout

**Why `held_until` in the database?**
- **Durability:** Survives server crashes
- **Audit trail:** See when seat was held and by whom (`held_by` = user UUID)
- **Background release job:** Every 60 seconds, release seats where `held_until < NOW()`

**Example flow:**
```
1. User selects seats, clicks "Proceed to Payment"
   → API: UPDATE seats SET status='held', held_until=NOW()+10min, held_by=$userId
   
2. User's payment takes 30 seconds, then fails
   → API: UPDATE seats SET status='available', held_until=NULL, held_by=NULL
   → Other users can now book these seats
   
3. User abandons checkout without completing payment
   → Background job (runs every 60s):
      DELETE FROM seats WHERE status='held' AND held_until < NOW();
   → Seats auto-release after 10 minutes
```

**Index optimization:**
- Partial index `idx_seats_held_until WHERE status='held'` keeps index small
- Only held seats are in the index (small % of total seats)
- Background release job scans this small index, not entire seats table

---

### 4. Why Partial Index on `bookings.status`?

**Decision:** Create index only on pending/failed bookings, not on all bookings.

```sql
CREATE INDEX idx_bookings_status
  ON bookings(status)
  WHERE status IN ('pending','failed');
```

**Why not index all bookings?**
- 99% of bookings are in 'confirmed' or 'refunded' state (historical)
- These historical bookings are rarely queried (except for user's "My Bookings" list)
- Indexing them wastes disk space and slows INSERT/UPDATE on bookings table

**Why index only pending/failed?**
- Payment workers query: `SELECT * FROM bookings WHERE status='pending'` **every second**
- This query runs continuously during the sale
- Partial index is **tiny** (only 0.1% of bookings) but covers this critical query
- Reduces index scan time from O(n) to O(log n) on small set

**Index size impact:**
- Full index on 5M bookings: ~200MB
- Partial index on 5K pending bookings: ~200KB
- **1000x smaller, same query speed**

---

### 5. Why Is `category` VARCHAR Instead of Foreign Key to a Price Category Table?

**Decision:** Store category directly as VARCHAR on seats (denormalized).

**Why not normalize to a separate `seat_categories` table?**
```sql
-- Normalized (bad for booking):
CREATE TABLE seat_categories (
  id SERIAL PRIMARY KEY,
  name VARCHAR(30),
  price DECIMAL(10,2)
);
CREATE TABLE seats (
  ...
  category_id INT REFERENCES seat_categories(id)
);
-- Requires JOIN on every booking query
-- Adds ~5-10ms latency
```

**Why denormalized?**
- Booking queries access seats table directly
- Seat price must be readable in single SELECT
- No need for JOIN; no extra latency

**Trade-off:**
- Slight data duplication (price stored on each seat)
- If price changes mid-sale, existing seats keep old price ✅ (correct behavior - they already sold)
- If we redesigned, new event uses new price ✅

---

### 6. Why `ON DELETE CASCADE` for Foreign Keys?

**Decision:** Foreign keys use `ON DELETE CASCADE`.

```sql
REFERENCES events(id) ON DELETE CASCADE
REFERENCES venues(id) ON DELETE CASCADE
REFERENCES users(id) ON DELETE CASCADE
REFERENCES bookings(id) ON DELETE CASCADE
```

**Implications:**
- Delete an event → all seats for that event are deleted → all bookings are deleted
- Delete a user → all their bookings are deleted

**Caution in production:**
- This makes delete operations very destructive
- Real systems typically use soft deletes (add `deleted_at` column, never delete)
- But for this design exercise, CASCADE demonstrates the relationships clearly

**Alternative:** Use `ON DELETE RESTRICT` to prevent accidental deletions:
```sql
REFERENCES events(id) ON DELETE RESTRICT
-- Event cannot be deleted if it has seats
```

---

## Sample Data & Usage Patterns

### Create a Concert Event

```sql
INSERT INTO venues (name, city, capacity, address)
VALUES ('Aadmittal Mumbai', 'Mumbai', 15000, 'NESCO, Goregaon');

INSERT INTO events (name, artist, venue_id, starts_at, total_seats, status)
VALUES ('Coldplay Tour 2024', 'Coldplay', 1, '2024-05-12 12:00:00 IST', 15000, 'upcoming');

-- Insert seats for the event (simplified - normally bulk inserted)
INSERT INTO seats (event_id, section, row_label, number, category, price, status)
VALUES 
  (1, 'A', '1', 1, 'General', 5000.00, 'available'),
  (1, 'A', '1', 2, 'General', 5000.00, 'available'),
  ...
  (1, 'VIP', '1', 1, 'Premium', 15000.00, 'available');
```

### Create a User & Booking

```sql
INSERT INTO users (email, phone, name)
VALUES ('user@example.com', '9876543210', 'John Doe')
RETURNING id;

INSERT INTO bookings (user_id, event_id, status, total_amount)
VALUES ('7f8a3c2b-4d9e-11ec-...', 1, 'pending', 10000.00)
RETURNING id;

-- Link seats to the booking
INSERT INTO booking_seats (booking_id, seat_id)
VALUES ('9c2f8a4e-5b3c-11ec-...', 100),
       ('9c2f8a4e-5b3c-11ec-...', 101);
```

### Query Patterns for Critical Paths

**Find available seats in a section:**
```sql
SELECT id, row_label, number, price
FROM seats
WHERE event_id = 1 AND section = 'A' AND status = 'available'
ORDER BY row_label, number;
-- Uses: idx_seats_event_status
```

**Hold multiple seats (optimistic locking):**
```sql
UPDATE seats
SET status = 'held', held_until = NOW() + INTERVAL '10 minutes', held_by = $1, version = version + 1
WHERE id = ANY($2) AND status = 'available' AND version = $3
RETURNING id, version;
-- If result is empty or fewer rows than requested, conflict occurred - retry
```

**Find pending bookings for payment processing:**
```sql
SELECT id, user_id, total_amount, payment_ref
FROM bookings
WHERE status = 'pending'
LIMIT 100;
-- Uses: idx_bookings_status (partial, small scan)
```

**Release expired holds (background job):**
```sql
UPDATE seats
SET status = 'available', held_until = NULL, held_by = NULL
WHERE status = 'held' AND held_until < NOW();
-- Uses: idx_seats_held_until
```

---

## Database Capacity & Scalability Notes

### Estimated Sizes (5L users, 500K concurrent, 15K total seats)

| Table | Rows | Size | Queries/sec |
|-------|------|------|-------------|
| venues | 10 | <1MB | <10 |
| events | 50 | 5MB | ~100 |
| **seats** | **15,000** | **5MB** | **16,667** (during sale) |
| users | 500,000 | 200MB | ~1,000 |
| **bookings** | **50,000** | **20MB** | **10/s** (sustained) |
| booking_seats | 150,000 | 20MB | With bookings |

### Index Sizes

| Index | Size | Purpose |
|-------|------|---------|
| idx_seats_event_status | 500KB | Workhorse - hit thousands of times during sale |
| idx_seats_held_until (partial) | 50KB | Only ~500 held seats, very small |
| idx_bookings_status (partial) | 200KB | Only ~5K pending bookings |

### Connection Pool Needs

- **Without Redis locking:** 1,667 DB connections needed for 16,667 RPS at 100ms latency → **not feasible**
- **With Redis locking:** 50ms seat hold time → 833 DB connections needed → **still over default limit**
- **With async queue:** API holds connection for only 10ms → 166 DB connections needed → **feasible**

---

## Next Steps

1. Implement this schema in development PostgreSQL
2. Load test with seat booking simulation
3. Monitor query latencies and connection pool utilization
4. Proceed to [CONCURRENCY.md](CONCURRENCY.md) for locking strategy details
