## Full Architecture Diagram

```text
Mobile / Browser
			|
			v
CloudFront CDN
	- serves: static assets, cached event pages
	- passes through: dynamic API calls to ALB
	- cache hit: event pages/assets served at edge
	- cache miss: request forwarded to origin
	annotation: edge caching reduces pressure on the API path (see CACHE.md: event details TTL 3600s, availability TTL 30s)
			|
			|  API calls only
			v
Application Load Balancer (ALB)
	- SSL termination
	- health checks every 10s
	- rate limit rule: 200 req/IP/min
	annotation: front-door throttling protects the 500k-user burst window (see CONCURRENCY.md: peak RPS and DB pool limits)
			|
			v
Node.js API Auto-Scale Group
	- handles HTTP requests
	- reads availability and event data from Redis
	- writes authoritative booking state to PostgreSQL primary
	- publishes payment jobs to SQS
	annotation: this is the async API path chosen to avoid holding DB connections during payment (see QUEUE.md: async payment prevents pool exhaustion)
			|
			+------------------------------- READ -------------------------------+
			|                                                                    |
			v                                                                    v
Redis Cluster (3 nodes, ElastiCache)                                PostgreSQL Primary
	- cache keys:                                                      - writes only
		* availability:{event_id}:{category} TTL 30s                      - booking inserts and seat updates
		* event:{event_id} TTL 3600s                                      - source of truth for seat status
		* seatmap:{event_id} TTL 86400s                                   annotation: all writes route here to preserve correctness (see SCHEMA.md: seats.version and bookings schema)
	- lock keys:                                                        
		* seat_lock:{event_id}:{seat_id}                                  
		* SETNX + lock TTL 30s                                            
	annotation: Redis is used both for hot cache reads and distributed seat holds (see CONCURRENCY.md: Redis SETNX hybrid strategy)
			|                                                                    |
			| cache miss / lock grant                                             | replication
			|                                                                    v
			|                                                              PostgreSQL Read Replicas ×2
			|                                                                - reads only
			|                                                                - event details, seat maps, user booking history
			|                                                                annotation: replica reads are safe for non-transactional views, not for booking confirmation (see CACHE.md: cache-aside and read separation)
			|
			+------------------------------ PUBLISH ----------------------------+
																	 |
																	 v
													 SQS Payment Queue
														 - message format: bookingId, userId, eventId, seatIds, totalAmount, paymentToken, idempotencyKey
														 - visibility timeout: 120s
														 - DLQ: payment-dlq after maxReceiveCount=3
														 annotation: async queue decouples payment latency from request latency (see QUEUE.md: sync payment collapses the DB pool)
																	 |
																	 | read SQS message
																	 v
												Payment Worker (ECS Fargate)
													1. read from SQS
													2. call payment gateway
													3. update PostgreSQL primary
													4. publish SNS on confirmed booking
													5. delete SQS message
													annotation: worker owns payment retries and booking finalization (see QUEUE.md: worker logic, retries, and DLQ)
																	 |
																	 v
																AWS SNS
																 /   \
																/     \
															 v       v
												SES Email     SMS Delivery
													- confirmation email   - SMS confirmation
													annotation: notifications fire only after confirmed booking (see QUEUE.md: success path sends confirmation)
```

## Component Notes

CloudFront CDN keeps static assets and cached event pages close to the user, while API requests still flow to the origin so booking correctness stays centralized.

ALB terminates SSL, applies health checks, and rate-limits abusive bursts before traffic reaches the application layer.

Node.js API servers are the control plane for booking: they consult Redis for fast reads and locks, write authoritative state to PostgreSQL, and enqueue payment work asynchronously.

Redis Cluster has two distinct roles: hot cache for read-heavy data and seat-lock coordination with SETNX for correctness during seat holds.

SQS is the durability boundary for payment work. If payment processing slows down, the queue absorbs the spike and the API still returns quickly.

The payment worker is the only component that talks to the payment gateway. It is responsible for idempotency, retries, DB finalization, and notification fan-out.

PostgreSQL Primary receives all writes because it is the source of truth for seat inventory, bookings, and booking-seat mappings.

Read replicas serve low-risk reads only. They improve browse performance, but booking confirmation always validates against the primary.

SNS fans out confirmation events to email and SMS channels after the booking is confirmed.

## Part A Traceability

- CloudFront CDN ← CACHE.md: cache event pages and static assets to reduce origin traffic.
- ALB ← CONCURRENCY.md: rate limit the sale burst before it reaches the API tier.
- Node.js API ← CONCURRENCY.md: hybrid Redis + PostgreSQL path keeps throughput high without losing correctness.
- Redis Cluster ← CONCURRENCY.md: SETNX seat locks plus short-lived cache keys.
- SQS Payment Queue ← QUEUE.md: async payment pipeline prevents DB pool collapse.
- Payment Worker ← QUEUE.md: worker owns payment retries, failure handling, and DLQ recovery.
- PostgreSQL Primary ← SCHEMA.md: authoritative seat, booking, and versioned update state.
- PostgreSQL Read Replicas ← SCHEMA.md and CACHE.md: safe separation of browse reads from write transactions.
- SNS / SES / SMS ← QUEUE.md: confirmation notifications are emitted only after successful payment.
