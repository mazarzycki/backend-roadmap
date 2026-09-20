# Backend Engineering Roadmap

*Python backend · Networking · Docker & Kubernetes · Azure · Security as a thread through everything*

**Pace:** 5–7 hours per week · **Length:** 44 weeks (about 10 months) · **Started:** ____________

---

## How to use this plan

1. **One week at a time.** Don't read ahead and don't re-plan mid-phase. If a week takes two weeks, that's fine: slide everything down. Changes to the plan happen only in buffer weeks.
2. **Apply for jobs in parallel, starting in Week 1.** This plan is for growth, not a gate you must pass before applying. Job ads describe an ideal candidate who rarely exists. Rule of thumb: if you match a posting's core stack and most of the rest, apply.
3. **"Done" means built and explained.** A week is complete when every checkbox is ticked, including "explain it out loud". Reading alone never completes a week.
4. **Everything lands in one anchor project.** Pick one real API you'll keep running (CifraGuard's paid tier is the natural choice). By Week 44 it has auth, caching, containers, a Helm chart, Azure infrastructure, dashboards and a threat model. That repository *is* your portfolio.
5. **Buffer weeks are part of the plan.** They absorb slippage and give you rest. Skipping them is how burnout comes back.
6. **Keep an evidence log** (template in Appendix C). One entry per week. When impostor syndrome says "you should know more", read the log.

### Weekly rhythm (about 6 hours)

| Block | Time | What |
|---|---|---|
| Learn | 2 h | Work through the "Learn" items. Take notes in your own words. |
| Build | 3 h | Do the "Build" task in the anchor project. |
| Explain | 30 min | Record a 5-minute voice memo explaining the week's topic as if to a junior colleague. Re-record until it's clear. |
| Recall + log | 30 min | Answer *last* week's recall question without notes, then write your evidence log entry. |

### Threads that run through every phase

- **Security — marked [SEC].** There's no separate security track. Instead, every phase has at least one week where you harden or attack what you just built. This is your differentiator: most backend engineers can't threat-model a system; you can.
- **Networking — marked [NET].** A full phase (Phase 3), plus networking weeks inside the Kubernetes and Azure phases, because every "mysterious" production bug eventually becomes a networking question.
- **System design.** One 30-minute design drill per week (Appendix A).

### Overview

| Phase | Weeks | Milestone |
|---|---|---|
| 1. FastAPI & API design | 1–7 (+8 buffer) | Production-shaped API with auth, pagination, migrations and tests |
| 2. Redis & performance | 9–12 (+13 buffer) | Performance report with before/after numbers |
| 3. Networking | 14–18 (+19 buffer) | You can trace a request from browser to database, hop by hop |
| 4. Docker & CI/CD | 20–23 (+24 buffer) | `git push` produces a tested, scanned, versioned image |
| 5. Kubernetes | 25–30 (+31 buffer) | App runs from your Helm chart, hardened and autoscaling |
| 6. Azure | 32–36 (+37 buffer) | Full stack on Azure via Terraform, zero secrets in the repo |
| 7. Observability & security ops | 38–41 (+42 buffer) | Dashboards, SLOs, a postmortem, a threat model |
| 8. Portfolio & interviews | 43–44 | Evidence pack and mock interviews |

> **Reality check for impostor syndrome:** Phases 1–4 alone (about 24 weeks) already cover what most remote backend postings ask for. Everything after that is leverage, not a prerequisite.

---

## Phase 1 — FastAPI & API design (Weeks 1–8)

**Why this phase:** FastAPI is your core employability signal. Interviewers rarely test whether you've *used* FastAPI; they test whether you understand what happens underneath it: the event loop, the database, the contract with clients.

**Main resources:** FastAPI docs (Tutorial and Advanced User Guide) · *Architecture Patterns with Python* (free at cosmicpython.com)

### Week 1 — Async Python and the event loop

**Why:** "Why is my FastAPI endpoint slow?" is the most common interview question for this stack, and the most common real answer is "something is blocking the event loop."

**Learn:**
- Coroutines, `await`, and the event loop as a single-threaded scheduler
- `async def` vs `def` endpoints: `def` runs in a threadpool (40 threads by default), `async def` runs directly on the loop
- What blocks the loop: `requests`, `time.sleep`, synchronous DB drivers, CPU-heavy work
- `asyncio.gather`, `asyncio.TaskGroup` (Python 3.11+), `asyncio.timeout`
- Moving CPU-bound work to a threadpool or process pool

**Real-world example:** A team adds a call to a partner's API using `requests.get()` inside an `async def` endpoint. Locally it's fine. In production, the partner slows to 2 seconds per call, and *every* endpoint on that worker freezes, including `/health`. The load balancer marks the instance as dead, traffic shifts to the others, and the outage spreads. The fix is one line (`httpx.AsyncClient`), but finding it requires understanding the loop.

**Build:** Create an endpoint that simulates a slow upstream (`await asyncio.sleep(2)`). Call it from two other endpoints: one using `requests`, one using `httpx.AsyncClient`. Fire 50 concurrent requests at each (a short asyncio script, or a tool like `hey` or `oha`) and record the total time.

**Done when:**
- [ ] You have measured numbers for blocking vs non-blocking
- [ ] You can explain why a `def` endpoint doesn't block the loop but an `async def` endpoint calling `requests` does
- [ ] 5-minute recorded explanation

**Recall (answer next week, no notes):** What happens to all other requests when one `async def` endpoint calls `time.sleep(5)`?

### Week 2 — FastAPI internals: dependencies, lifespan and structure

**Why:** Dependency injection is what separates a script from a maintainable, testable service. Lifespan management is where connection pools live, and pool bugs are classic production incidents.

**Learn:**
- `Depends`, sub-dependencies, and per-request dependency caching
- `yield` dependencies for resources such as DB sessions, and their cleanup semantics
- `lifespan` for startup/shutdown: create one `httpx.AsyncClient` and one DB engine for the whole app, not one per request
- Middleware (and its execution order) vs dependencies: when to use which
- Configuration with `pydantic-settings`; project layout with `APIRouter` (routers → services → repositories)

**Real-world example:** An API opens a DB session per request, but one code path returns early without closing it. Weeks later, during a traffic spike, the pool runs dry and every request hangs for 30 seconds before failing with SQLAlchemy's `QueuePool limit ... reached` error. `yield` dependencies with guaranteed cleanup prevent this entire class of bug.

**Build:** Restructure the anchor project into routers, services and repositories. Load settings from environment variables, create shared clients in `lifespan`, and provide the DB session through a `yield` dependency.

**Done when:**
- [ ] No module creates its own DB engine or HTTP client
- [ ] Settings come only from the environment
- [ ] 5-minute recorded explanation of the request lifecycle in your app

**Recall:** Why should an `httpx.AsyncClient` be created in `lifespan` rather than inside each request?

### Week 3 — Pydantic v2 and API contracts

**Why:** Your API contract is a promise to clients. Many bugs that reach customers are contract bugs: a field leaks, a type changes, an error can't be parsed.

**Learn:**
- Separate input, output and database models (`UserCreate`, `UserRead`, ORM `User`)
- `response_model` as a security boundary: it filters what leaves your API
- Field constraints, `field_validator`, `model_validator`, strict mode, `extra="forbid"`
- Serialization: aliases, `model_dump(exclude_unset=True)` for PATCH
- Consistent errors with RFC 9457 (Problem Details) and custom exception handlers
- OpenAPI as documentation clients actually rely on

**Real-world example:** Returning the ORM object directly is how password hashes, internal flags and other users' IDs end up in API responses. OWASP lists this as API3:2023 (Broken Object Property Level Authorization): the caller is allowed to use the endpoint, but it returns or accepts properties they should never see or set. The classic case is a `PATCH /users/me` that happily accepts `"is_admin": true` (mass assignment).

**Build:** Split the models in the anchor project. Add a global exception handler returning Problem Details JSON. Add a test that fails if a sensitive field (e.g., `password_hash`) ever appears in any response.

**Done when:**
- [ ] No endpoint returns an ORM object directly
- [ ] All errors share one JSON format
- [ ] 5-minute recorded explanation

**Recall:** What is mass assignment, and which Pydantic settings prevent it?

### Week 4 — API design patterns

**Why:** Pagination, idempotency and versioning are where "it works" becomes "it works for paying customers at scale". They're also staple interview topics.

**Learn:**
- Offset vs cursor (keyset) pagination, and why `OFFSET 100000` gets slow
- Filtering and sorting without SQL injection or unindexed full scans
- Idempotency keys for POST requests; safe client retries
- Versioning strategies: URL (`/v1`), header, date-based
- Useful headers: `Retry-After`, rate-limit headers, `ETag` and conditional requests

**Real-world example:** Stripe's API is the reference design. List endpoints use cursor pagination (`starting_after`). POST requests accept an `Idempotency-Key` header, so a client that retries after a network timeout doesn't charge a card twice. The API is versioned by date, so years-old integrations keep working. Read Stripe's API reference with an engineer's eye; it's a free masterclass.

**Build:** Add cursor pagination to your main list endpoint and support `Idempotency-Key` on one POST endpoint (store key → response for 24 hours; a DB table is fine for now, and it moves to Redis in Phase 2).

**Done when:**
- [ ] A test proves that repeating a POST with the same key returns the original response and creates nothing new
- [ ] A test proves pagination doesn't skip or duplicate items when new rows are inserted between pages
- [ ] 5-minute recorded explanation

**Recall:** Why can offset pagination return duplicate or missing items when data changes between page requests?

### Week 5 — Authentication, authorization and API security [SEC]

**Why:** Most real API breaches aren't clever exploits; they're missing authorization checks. Your security background makes this week a natural strength, so make it visible in the code.

**Learn:**
- OAuth2 flows (Authorization Code with PKCE, Client Credentials) and OpenID Connect: what each one is for
- JWTs: structure, signing (HS256 vs RS256), expiry, refresh-token rotation, the limits of revocation; PyJWT
- Password hashing with Argon2
- Scopes and roles; object-level checks ("does *this* user own *this* invoice?")
- OWASP API Security Top 10 (2023), especially API1 (BOLA) and API5 (BFLA)

**Real-world example:** In 2022, Optus in Australia exposed data on nearly 10 million customers, reportedly through an API endpoint that required no authentication and used predictable identifiers. In 2021, researchers showed that Peloton's API returned private account data to unauthenticated requests, even for users with private profiles. Neither attack needed an exploit, just a missing check. For JWTs specifically, several early libraries accepted tokens with `"alg": "none"`, meaning no signature at all.

**Build:** JWT authentication with short-lived access tokens and refresh-token rotation; an ownership-check dependency; tests proving that user A cannot read or modify user B's resources by changing an ID in the URL.

**Done when:**
- [ ] Every endpoint has an explicit auth decision (public, authenticated, or role/ownership-checked)
- [ ] BOLA tests exist and pass
- [ ] 5-minute recorded explanation

**Recall:** What's the difference between authentication and authorization, and which one does BOLA break?

### Week 6 — The data layer: queries, indexes and transactions

**Why:** In most slow APIs, the bottleneck is the database, not Python. And in fintech, transaction correctness *is* the job.

**Learn:**
- SQLAlchemy 2.0 async sessions; Alembic migrations, including backward-compatible migrations for zero-downtime deploys
- The N+1 query problem; `selectinload` and `joinedload`
- Indexes (B-tree, composite, partial) and reading `EXPLAIN ANALYZE`
- Connection pooling: pool size × workers × instances vs Postgres `max_connections`
- Transactions, isolation levels, `SELECT ... FOR UPDATE`, optimistic locking with version columns

**Real-world example:** Race conditions cost real money. In 2014 the Bitcoin bank Flexcoin shut down after an attacker fired many simultaneous transfers between accounts; each request checked the balance before any of them had deducted it, and 896 BTC were withdrawn. In 2015 a researcher showed he could transfer the same Starbucks gift-card balance twice in parallel and create money. Both are the classic check-then-act race that `FOR UPDATE` or an atomic `UPDATE ... WHERE balance >= :amount` prevents.

**Build:** Add a "transfer credits" operation (or your project's equivalent). Write a test that fires 20 concurrent transfers and proves the balance never goes negative. Then find and fix one N+1 query, keeping `EXPLAIN ANALYZE` output from before and after.

**Done when:**
- [ ] The concurrency test passes reliably
- [ ] You can read an `EXPLAIN ANALYZE` plan and point to the expensive node
- [ ] 5-minute recorded explanation

**Recall:** Why doesn't "read the balance, check it, then write" work under concurrency, even inside a transaction at READ COMMITTED?

### Week 7 — Testing and background work

**Why:** Tests let you move fast without fear, and a portfolio repo with a real test suite reads very differently to a reviewer. Background jobs are where many "it worked locally" bugs hide.

**Learn:**
- pytest fixtures and parametrization; `httpx.AsyncClient` with `ASGITransport`
- `app.dependency_overrides` for test doubles
- Testcontainers for real Postgres and Redis in tests (don't mock your database)
- FastAPI `BackgroundTasks` vs a real task queue (Celery, or an async-native option such as Taskiq): retries, persistence, visibility
- Coverage as a signal, not a target

**Real-world example:** `BackgroundTasks` runs in the same process after the response is sent. If the container restarts during a deploy, the task is simply gone: no retry, no record. Teams discover this when a welcome email, or worse, a payment-confirmation webhook, silently never goes out. Rule of thumb: if losing the job would cost money or trust, it goes in a queue.

**Build:** Integration tests running against Postgres in Testcontainers. Move one background job into a proper queue with retries.

**Done when:**
- [ ] `pytest` runs green against real Postgres
- [ ] One job survives a worker restart and retries correctly
- [ ] 5-minute recorded explanation

**Recall:** When is `BackgroundTasks` acceptable, and when is it a liability?

### Week 8 — Buffer and Phase 1 checkpoint

- [ ] Finish anything that slipped
- [ ] Update the anchor project README: what it does, how to run it, key design decisions
- [ ] **Phase 1 milestone:** API with auth, cursor pagination, migrations and integration tests
- [ ] Answer these out loud, as in an interview:
  1. Your FastAPI endpoint's p95 went from 80 ms to 2 s after a deploy. Walk me through your investigation.
  2. How would you design versioning for a public API?
  3. How do you prevent a user from reading another user's data?
  4. How would you make a payment endpoint idempotent?
  5. Explain `async def` vs `def` in FastAPI and when you'd use each.
- [ ] Rest. Seriously.

---

## Phase 2 — Redis & performance (Weeks 9–13)

**Why this phase:** Redis appears in a large share of backend job postings, and "how would you scale this?" questions almost always end at caching, rate limiting or queues. Performance engineering is the skill that turns "I think it's slow" into "p95 is 1.2 s because of X, and here's the fix."

**Main resources:** Redis documentation (data types and patterns) · Martin Kleppmann, "How to do distributed locking" (2016)

### Week 9 — Redis fundamentals and caching patterns

**Why:** Caching is the cheapest performance win and the easiest way to serve stale or wrong data. Interviewers probe the second part.

**Learn:**
- Data types: strings, hashes, lists, sets, sorted sets, and when each fits
- Cache-aside, read-through, write-through; choosing TTLs
- Invalidation: TTL-only, delete-on-write, versioned keys
- Cache stampede (thundering herd): locking, request coalescing, TTL jitter, stale-while-revalidate
- Serialization cost and key naming conventions

**Real-world example:** On 23 September 2010 Facebook was down for about two and a half hours. An automated system meant to repair invalid cached configuration values saw an invalid value and had every client query the database to fix it. The flood of queries overwhelmed the database, which caused more errors, which caused more queries: a feedback loop that only stopped when engineers took the site offline. That is a cache stampede at planetary scale.

**Build:** Add cache-aside with TTL and jitter to your most expensive read endpoint, invalidate on write, and add stampede protection with a short-lived Redis lock.

**Done when:**
- [ ] A test proves a write invalidates the cached read
- [ ] 100 concurrent requests on a cold cache trigger only one database query
- [ ] 5-minute recorded explanation

**Recall:** Name three ways to prevent a cache stampede.

### Week 10 — Rate limiting and atomic operations [SEC]

**Why:** Rate limiting protects you from abuse and from your own customers' bugs. It's a favourite design question because it tests atomicity and distributed thinking in one go.

**Learn:**
- Fixed window, sliding window log, sliding window counter, token bucket, leaky bucket: trade-offs of each
- The `INCR`-then-`EXPIRE` race, and why Lua scripts or `MULTI` fix it
- Where to limit: gateway vs application; per user, per IP, per API key
- `429 Too Many Requests`, `Retry-After`, and rate-limit headers
- OWASP API4:2023 (Unrestricted Resource Consumption)

**Real-world example:** Stripe's engineering post "Scaling your API with rate limiters" describes the four kinds of limiters they run, including a token-bucket request limiter and a concurrent-request limiter, and how they shed low-priority traffic during incidents. GitHub's API tells every client exactly where it stands through `x-ratelimit-remaining` and `x-ratelimit-reset` headers. Both are designs worth copying.

**Build:** Implement a token-bucket limiter as an atomic Lua script, applied per API key through a dependency. Test it with concurrent requests from several simulated clients.

**Done when:**
- [ ] The limiter holds exactly under concurrency (no over-admission)
- [ ] Clients receive correct `429` and `Retry-After` responses
- [ ] 5-minute recorded explanation

**Recall:** Why is "GET the counter; if below the limit, INCR it" wrong under concurrency?

### Week 11 — Concurrency and messaging: locks, Pub/Sub and Streams

**Why:** "How do you handle race conditions across multiple instances?" is a senior-level question. Event-style thinking (streams, consumer groups, at-least-once delivery) is how modern backends decouple work.

**Learn:**
- Distributed locks with `SET key value NX PX`, lock expiry, fencing tokens
- Pub/Sub: fire-and-forget, nothing is stored
- Streams: `XADD`, consumer groups, `XREADGROUP`, `XACK`, pending entries, claiming stuck messages
- Delivery guarantees: at-most-once, at-least-once, and "effectively once" through idempotent consumers

**Real-world example:** In 2016 Martin Kleppmann published "How to do distributed locking", arguing that Redis's Redlock algorithm isn't safe when correctness matters: a process can pause (for example, during garbage collection) after acquiring a lock, lose the lock without knowing it, and keep writing. The creator of Redis published a reply. Read both. The lesson: a Redis lock is an efficiency tool unless you add fencing tokens. A simpler everyday failure: a notification service subscribed through Pub/Sub restarts for 30 seconds during a deploy, and every message sent in that window is gone forever. Streams would have kept them.

**Build:** Publish domain events (e.g., `transfer.completed`) to a Redis Stream and process them with a consumer group. Kill the consumer mid-run and prove the pending messages are picked up again, and that the idempotent handler makes reprocessing harmless.

**Done when:**
- [ ] No event is lost when the consumer crashes
- [ ] Reprocessing an event twice has no double effect
- [ ] 5-minute recorded explanation

**Recall:** Why does Pub/Sub lose messages when a consumer restarts, and how do Streams avoid it?

### Week 12 — Performance engineering and Redis in production [SEC]

**Why:** This week turns everything into numbers, which is the language of senior engineers and of strong interview answers.

**Learn:**
- Latency percentiles (p50, p95, p99) and why averages lie
- Load testing with Locust or k6: ramp-up, steady state, finding the point where latency bends upward
- Profiling a running process with py-spy (flame graphs)
- Little's Law: concurrency = throughput × latency (for sizing workers and pools)
- Redis in production: persistence (RDB vs AOF), eviction policies, memory limits, replication, Sentinel vs Cluster
- Redis security: ACLs, authentication, never exposing port 6379 publicly, disabling dangerous commands

**Real-world example:** Redis servers exposed to the internet without authentication are routinely compromised. A well-known technique uses `CONFIG SET` to make Redis write an attacker's SSH key or a cron job to disk. In 2023 the P2PInfect worm spread through exposed Redis servers. Your cache is part of your attack surface, not just your performance story.

**Build:** Load-test the anchor API before and after the caching and rate limiting from Weeks 9–10. Capture a py-spy flame graph of the slowest endpoint. Write a one-page performance report in the repo. Lock down your Redis with ACLs and a dedicated user for the app.

**Done when:**
- [ ] `docs/performance.md` shows p50/p95/p99 and throughput, before and after
- [ ] Redis requires authentication and the app user can't run `CONFIG` or `FLUSHALL`
- [ ] 5-minute recorded explanation

**Recall:** By Little's Law, if you need 200 requests/second at 250 ms average latency, how many requests are in flight at once?

### Week 13 — Buffer and Phase 2 checkpoint

- [ ] Finish anything that slipped
- [ ] **Phase 2 milestone:** performance report with before/after numbers
- [ ] Answer these out loud:
  1. One Redis instance isn't enough anymore. How do you scale it?
  2. Design a rate limiter for a public API with 10,000 customers.
  3. How do you invalidate a cache without serving stale data?
  4. Two instances of your service receive the same event. How do you make that safe?
  5. Your API is slow only at p99. What does that tell you, and where do you look?
- [ ] Rest

---

## Phase 3 — Networking for backend engineers (Weeks 14–19) [NET]

**Why this phase:** You're genuinely interested in it, and it pays off everywhere else in this plan: container networking, Kubernetes Services, Azure VNets, TLS errors, and those "works on my machine" timeouts. The engineer who can read a packet capture ends most debugging arguments.

**Main resources:** *High Performance Browser Networking* by Ilya Grigorik (free at hpbn.co) · Julia Evans' networking zines · Wireshark

### Week 14 — The packet's journey: IP, TCP and routing [NET]

**Why:** Every higher-level concept in this plan (load balancers, Kubernetes networking, VNets) is built on these foundations.

**Learn:**
- The TCP/IP model vs OSI, and why engineers mostly think in terms of L3, L4 and L7
- IPv4 addressing, CIDR, subnetting by hand; private ranges (RFC 1918)
- Routing tables and default gateways; BGP conceptually (how networks announce which addresses they can reach)
- TCP: three-way handshake, sequence numbers, retransmission, windows, `TIME_WAIT`; UDP and when it's used
- Tools: `ip`, `ss`, `ping`, `traceroute`/`mtr`, `tcpdump`, Wireshark

**Real-world example:** On 4 October 2021, Facebook, Instagram and WhatsApp vanished for about six hours. A maintenance command accidentally disconnected Facebook's backbone network. Facebook's DNS servers are designed to withdraw their BGP route announcements when they can't reach the data centers, and they did exactly that, so the rest of the internet could no longer find facebook.com. Engineers reportedly struggled even to get into buildings, because internal tools depended on the same network.

**Build:** Subnet by hand: split `10.20.0.0/16` into subnets for app, data and management tiers, and check your answers with a calculator afterward. Capture a request to your anchor API with `tcpdump`, open it in Wireshark, and identify the SYN, SYN-ACK, ACK, the HTTP request and the FIN.

**Done when:**
- [ ] 10 subnetting exercises done correctly without tools
- [ ] Annotated screenshot of your first packet capture in your notes
- [ ] 5-minute recorded explanation of the TCP handshake

**Recall:** How many usable host addresses are in a /26, and why?

### Week 15 — DNS and HTTP in depth [NET] [SEC]

**Why:** "It's always DNS" is a joke because it's so often true. And HTTP versions and connection reuse directly shape your API's latency.

**Learn:**
- DNS resolution: stub resolver → recursive resolver → root → TLD → authoritative; caching and TTLs
- Record types: A, AAAA, CNAME, MX, TXT, SRV, NS; using `dig` and `dig +trace`
- HTTP/1.1 keep-alive and head-of-line blocking; HTTP/2 multiplexing; HTTP/3 over QUIC (UDP)
- Connection pooling in clients; ephemeral port exhaustion
- Timeouts: connect vs read vs total, and why every outbound call needs them
- HTTP/2 "Rapid Reset" attacks (CVE-2023-44487)

**Real-world example:** On 20 October 2025, much of AWS's us-east-1 region failed for many hours. The root cause was a latent race condition in DynamoDB's automated DNS management, which left the regional DynamoDB endpoint with an empty DNS record. Many AWS services and customer applications depended on that endpoint and failed in turn. It is a DNS incident and a race condition at once, two themes of this plan. Separately, in 2023 attackers abused HTTP/2's stream-cancellation feature ("Rapid Reset") to launch record-breaking DDoS attacks, which is why web servers and proxies shipped emergency patches.

**Build:** Run `dig +trace` on your own domain and explain each step. Then compare 100 sequential requests using a new `httpx` client per request vs one shared client. Measure the difference and explain it using what you saw in last week's packet capture.

**Done when:**
- [ ] You can explain every line of `dig +trace` output
- [ ] Measured numbers for connection reuse vs no reuse
- [ ] 5-minute recorded explanation

**Recall:** Why is creating a new HTTP client for every request slow, even when calling the same host?

### Week 16 — TLS, certificates and mTLS [NET] [SEC]

**Why:** TLS failures cause some of the most embarrassing outages, and mTLS is the backbone of zero-trust service-to-service security. This is where networking and security meet, which suits your background.

**Learn:**
- The TLS 1.3 handshake; certificates, chains, intermediates and CAs; SNI
- Debugging certificate errors with `openssl s_client`
- ACME and Let's Encrypt; automated renewal
- mTLS: both sides present certificates; where service meshes fit (conceptually only)
- HSTS and secure defaults

**Real-world example:** Expired certificates take down giants. In February 2020, Microsoft Teams went down for hours because an authentication certificate expired. In December 2018, an expired certificate in Ericsson software knocked out mobile data for millions of O2 customers in the UK. In September 2021, an old Let's Encrypt root certificate expired, and older devices and clients started rejecting perfectly valid websites. The lesson: certificate expiry is an operational risk that must be automated and monitored, not remembered.

**Build:** Put Caddy or nginx with automatic HTTPS in front of the anchor API on a small VPS (or locally with your own CA). Then set up mTLS between two local services so that a client without a valid certificate is rejected.

**Done when:**
- [ ] `openssl s_client` shows the full chain for your endpoint
- [ ] A request without a client certificate is refused by the mTLS service
- [ ] 5-minute recorded explanation

**Recall:** What does a client check when it validates a server's certificate?

### Week 17 — Load balancers, reverse proxies and NAT [NET] [SEC]

**Why:** Every production request passes through several of these. Misconfigured timeouts, headers or NAT are behind many "random" 502s, 504s and connection failures.

**Learn:**
- L4 vs L7 load balancing; algorithms (round robin, least connections); health checks
- Reverse proxies (nginx, Traefik, Caddy): timeouts, buffering, WebSockets
- `X-Forwarded-For` and `Forwarded`; Uvicorn's `--proxy-headers` and `--forwarded-allow-ips`
- NAT and SNAT; ephemeral ports; connection tracking
- The Azure equivalents: Load Balancer (L4), Application Gateway (L7), Front Door (global L7)

**Real-world example:** A very common Azure support case: an app makes many outbound calls to the same external API and starts failing intermittently under load. The cause is SNAT port exhaustion: outbound connections share a limited pool of source ports on the public IP. The fixes are connection reuse (Week 15) and a NAT Gateway. On the security side: if your rate limiter trusts `X-Forwarded-For` from anyone, an attacker sends a new random IP in that header with every request and walks straight past the limit.

**Build:** nginx in front of two instances of the anchor API with health checks. Kill one instance and watch traffic shift. Configure proxy headers correctly and verify that your rate limiter sees real client IPs, and that a forged `X-Forwarded-For` from outside is ignored.

**Done when:**
- [ ] Failover works with no errors visible to a running load test
- [ ] Forged forwarding headers don't bypass the rate limiter
- [ ] 5-minute recorded explanation

**Recall:** Why can't an L4 load balancer route requests by URL path?

### Week 18 — Linux networking and firewalls: what Docker really does [NET] [SEC]

**Why:** This week removes the magic from containers before Phase 4. Docker networking is Linux network namespaces, virtual Ethernet pairs, bridges and iptables rules, nothing more.

**Learn:**
- Network namespaces, veth pairs, Linux bridges
- iptables/nftables basics: chains, NAT, connection tracking
- MTU and fragmentation; why VPNs and overlay networks reduce the MTU
- WireGuard VPN fundamentals
- Network segmentation and default-deny firewalls

**Real-world example:** Docker writes its own iptables rules for published ports, and on Ubuntu those rules are evaluated before UFW's. Many people have exposed a database to the internet with `-p 5432:5432` while `ufw status` claimed the port was blocked. A related classic: small requests work over a VPN, but TLS handshakes or large responses hang forever. That symptom almost always means an MTU problem.

**Build:** By hand, create two network namespaces, connect them to a Linux bridge with veth pairs, assign IPs and ping between them. You've just rebuilt Docker's default network. Then set up WireGuard between your laptop and the VPS from Week 16.

**Done when:**
- [ ] Namespaces ping each other through your bridge
- [ ] WireGuard tunnel works, and you can explain its handshake at a high level
- [ ] 5-minute recorded explanation

**Recall:** What's the difference between `-p 5432:5432` and `-p 127.0.0.1:5432:5432`?

### Week 19 — Buffer and Phase 3 checkpoint

- [ ] Finish anything that slipped
- [ ] **Phase 3 milestone:** on paper, draw every hop of a request from a browser to your API's database: DNS, TCP, TLS, proxy, app, connection pool, database. Put the diagram in the repo.
- [ ] Answer these out loud:
  1. What happens when you type a URL and press Enter? Go deep on DNS, TCP, TLS and HTTP.
  2. Your service returns intermittent 502s behind a load balancer. How do you debug it?
  3. Explain L4 vs L7 load balancing and when you'd use each.
  4. What is mTLS, and why would you use it between internal services?
  5. Outbound calls fail under load but never in tests. What do you suspect?
- [ ] Rest

---

## Phase 4 — Docker & CI/CD (Weeks 20–24)

**Why this phase:** "Dockerized production environments" appears on nearly every backend posting. With Phase 3 behind you, containers are no longer magic; they're namespaces and iptables rules you've built by hand.

**Main resources:** Docker docs (building best practices) · pythonspeed.com articles on Python Docker images · GitHub Actions docs

### Week 20 — Images done right

**Why:** Image size and build speed affect deploy time, cost and attack surface. A clean Dockerfile is also one of the first things a reviewer notices in your repo.

**Learn:**
- Layers and cache ordering (dependencies before source code)
- Multi-stage builds: build wheels in one stage, copy them into a slim runtime stage
- Base images: `python:3.x-slim` vs Alpine vs distroless; pinning by digest
- `.dockerignore`; non-root `USER`; reproducible installs from a lock file (`uv` or `pip-tools`)

**Real-world example:** Many teams switch to Alpine for "smaller images" and then watch Python builds become dramatically slower. Alpine uses musl instead of glibc, and when a package has no musl-compatible wheel, pip compiles it from source. (musllinux wheels have improved this, but not for every package.) Itamar Turner-Trauring's widely cited article on this makes the point: measure, don't assume.

**Build:** Rewrite the anchor project's Dockerfile: multi-stage, slim, non-root, pinned base image. Record image size and rebuild time (after a one-line code change) before and after.

**Done when:**
- [ ] Container runs as a non-root user
- [ ] Before/after numbers for size and rebuild time in your notes
- [ ] 5-minute recorded walkthrough of every line of the Dockerfile

**Recall:** Why does copying the lock file before the source code speed up rebuilds?

### Week 21 — Containers at runtime, and Compose [NET]

**Why:** Most container bugs are runtime bugs: signals, health, configuration and networking.

**Learn:**
- PID 1 and signals: exec-form `CMD`, SIGTERM handling, graceful shutdown in Uvicorn
- Healthchecks, restart policies and resource limits
- Logging to stdout/stderr; configuration through environment variables
- Worker processes: `uvicorn --workers` vs one process per container (and why Kubernetes prefers the latter)
- Docker networking: default bridge vs user-defined networks (with built-in DNS), host mode, port publishing
- Compose: services, networks, volumes, `depends_on` with `condition: service_healthy`, profiles

**Real-world example:** "Why does our container take exactly 10 seconds to stop?" Because the app is launched through a shell script without `exec`, so the shell is PID 1 and never forwards SIGTERM to the app. Docker waits out its 10-second grace period, then sends SIGKILL, and every in-flight request is dropped on every single deploy.

**Build:** A Compose stack with the API, Postgres, Redis and nginx on a user-defined network, with healthchecks and graceful shutdown. Verify with `docker stop` during a load test that in-flight requests complete.

**Done when:**
- [ ] `docker stop` completes in well under 10 seconds with no dropped requests
- [ ] Services start only when their dependencies are healthy
- [ ] 5-minute recorded explanation

**Recall:** Why can containers on a user-defined bridge network reach each other by name, while on the default bridge they can't?

### Week 22 — CI/CD pipelines

**Why:** Remote teams trust engineers whose changes arrive tested and reproducible. This pipeline is also how you'll deploy to Kubernetes and Azure later.

**Learn:**
- GitHub Actions: workflows, jobs, matrices, caching
- Pipeline stages: lint (ruff) → type-check (mypy) → tests (with service containers) → build → push
- Image tagging: semantic version plus git SHA; never deploy `latest`
- Environments, required reviews and deployment protection rules
- Trunk-based development and small pull requests

**Real-world example:** On 1 August 2012, Knight Capital lost about $440 million in 45 minutes. New trading code was deployed manually to eight servers, but one server was missed. A repurposed feature flag then activated old, long-dead code on that server, which sent a flood of unintended orders into the market. An automated, verified deployment would have caught the inconsistent server before the market opened.

**Build:** A GitHub Actions pipeline for the anchor project: lint → type-check → tests (with Postgres and Redis service containers) → build → push to GitHub Container Registry, tagged with version and SHA.

**Done when:**
- [ ] A broken test blocks the image from being built
- [ ] Every image in the registry maps to exactly one commit
- [ ] 5-minute recorded explanation

**Recall:** Why is deploying the `latest` tag dangerous?

### Week 23 — Software supply chain security [SEC]

**Why:** Your pipeline has access to your secrets, your registry and eventually your cloud. Attackers know CI is often the softest target in a company.

**Learn:**
- Least-privilege `GITHUB_TOKEN` permissions; pinning third-party actions to a full commit SHA
- OIDC federation from GitHub to Azure, so there are no long-lived cloud secrets in CI
- Dependency scanning (pip-audit, Dependabot), image scanning (Trivy), secret scanning (gitleaks)
- SBOMs (Syft) and image signing (cosign): what they are and why regulators increasingly care

**Real-world example:** In 2021, attackers modified Codecov's Bash Uploader script, which thousands of CI pipelines downloaded and ran on every build. For about two months, it quietly sent those pipelines' environment variables, including credentials, to the attackers. In March 2025, the popular GitHub Action `tj-actions/changed-files`, used by more than 23,000 repositories, was compromised: its version tags were repointed to malicious code that dumped CI secrets into build logs. Pinning actions to commit SHAs would have blocked the second attack.

**Build:** Add pip-audit, Trivy and gitleaks to the pipeline, failing on high or critical findings. Pin all actions to SHAs, set minimal `permissions:` on every workflow, and generate an SBOM as a build artifact.

**Done when:**
- [ ] A deliberately vulnerable dependency makes the pipeline fail
- [ ] A fake secret committed on a branch is caught
- [ ] 5-minute recorded explanation

**Recall:** Why is pinning an action to `@v4` weaker than pinning it to a commit SHA?

### Week 24 — Buffer and Phase 4 checkpoint

- [ ] Finish anything that slipped
- [ ] **Phase 4 milestone:** `git push` produces a linted, type-checked, tested, scanned and versioned image with no manual steps
- [ ] Answer these out loud:
  1. Walk me through your Dockerfile and justify each decision.
  2. How do you handle secrets in containers and in CI?
  3. A container gets OOM-killed in production but never locally. Why might that be?
  4. How do you roll back a bad release?
  5. A library you depend on is reported compromised. What do you do?
- [ ] Rest. You're now past the point that covers most remote backend postings.

---

## Phase 5 — Kubernetes (Weeks 25–31)

**Why this phase:** Kubernetes shows up in a large share of remote backend postings, often as "nice to have". Knowing it well, especially debugging and networking, moves you from "can write the API" to "can run the system".

**Main resources:** *Kubernetes: Up and Running* · Kubernetes docs (Concepts and Tasks) · k3d or kind for a local cluster

### Week 25 — Core objects and the mental model

**Why:** Everything in Kubernetes is a controller reconciling *desired* state with *actual* state. Once that clicks, the rest is details.

**Learn:**
- Control plane components at a high level: API server, etcd, scheduler, controllers, kubelet
- Pods, ReplicaSets, Deployments; labels and selectors; namespaces
- Services as stable addresses for ever-changing pods
- `kubectl get`, `describe`, `logs`, `exec`, and `events`

**Real-world example:** Delete a pod managed by a Deployment and a replacement appears within seconds. Nobody restarted it; the controller noticed the difference and fixed it. The flip side: engineers "fix" a pod by editing it by hand, the controller replaces it, and the fix silently vanishes. Declarative configuration in Git is the only source of truth.

**Build:** Start a local cluster. Deploy the anchor API, Postgres and Redis with plain YAML (no Helm yet). Kill pods and watch them recover.

**Done when:**
- [ ] The app works end to end in the cluster
- [ ] You can explain what happened, step by step, when you deleted a pod
- [ ] 5-minute recorded explanation

**Recall:** What's the relationship between a Deployment, a ReplicaSet and a Pod?

### Week 26 — Configuration, secrets and health probes

**Why:** Many self-inflicted Kubernetes outages come from badly designed probes and careless config handling.

**Learn:**
- ConfigMaps and Secrets (Secrets are base64-encoded, not encrypted, by default)
- Liveness vs readiness vs startup probes, and what each one should check
- Graceful shutdown: `preStop` hooks, `terminationGracePeriodSeconds`, readiness during rollouts

**Real-world example:** A classic cascade: the liveness probe calls `/health`, which checks the database. The database slows down for a minute; every pod fails liveness at once; Kubernetes restarts all of them; they all reconnect at the same moment, making the database even slower. A small database hiccup becomes a full outage. Liveness should answer "is this process stuck?", never "are my dependencies up?"

**Build:** Separate `/livez` and `/readyz` endpoints, configure all three probes, move configuration into ConfigMaps and Secrets, and prove zero failed requests during a rollout with a load test running.

**Done when:**
- [ ] Rollout under load produces zero errors
- [ ] Slowing the database doesn't trigger restarts
- [ ] 5-minute recorded explanation

**Recall:** What should a liveness probe check, and what should it never check?

### Week 27 — Kubernetes networking [NET] [SEC]

**Why:** This is where your Phase 3 investment pays off. Kubernetes networking questions separate people who have deployed to a cluster from people who have operated one.

**Learn:**
- The model: every pod gets its own IP, and pods talk to each other without NAT
- CNI plugins (Calico, Cilium) and what they actually do
- Services: ClusterIP, NodePort, LoadBalancer; kube-proxy (iptables/IPVS) or eBPF-based replacements
- CoreDNS, service names, and the `ndots:5` default that multiplies DNS lookups
- Ingress and the Gateway API (the community ingress-nginx controller has been retired; the Gateway API is the direction to learn)
- NetworkPolicies: default-deny, then allow explicitly

**Real-world example:** On Pi Day (14 March) 2023, Reddit was down for over five hours during a Kubernetes upgrade from 1.23 to 1.24. Their Calico networking configuration selected route reflectors using a node label (`master`) that the new version no longer applied (it became `control-plane`), so pod networking broke across the cluster. Reddit's public postmortem, "You Broke Reddit: The Pi-Day Outage", is one of the best Kubernetes networking reads available.

**Build:** Expose the API through an Ingress or a Gateway. Apply a default-deny NetworkPolicy, then allow only API → Postgres and API → Redis. Verify with a debug pod that everything else is blocked. Make sure your local cluster actually enforces NetworkPolicies (k3d does by default; with kind, install Calico or Cilium first). An unenforced policy looks fine and silently does nothing.

**Done when:**
- [ ] A debug pod cannot reach Postgres; the API can
- [ ] You can trace how `redis:6379` resolves and routes to a pod
- [ ] 5-minute recorded explanation

**Recall:** When a pod connects to `http://redis:6379`, how does the request reach the actual Redis pod?

### Week 28 — Resources, scaling and rollouts

**Why:** Getting resources wrong either wastes money or causes mysterious slowness and restarts.

**Learn:**
- Requests vs limits; QoS classes; OOMKilled
- CPU throttling under limits (the CFS quota per 100 ms period)
- Horizontal Pod Autoscaler; PodDisruptionBudgets
- Rolling updates (`maxSurge`, `maxUnavailable`), `kubectl rollout status/undo`

**Real-world example:** A service has p99 latency spikes even though its average CPU usage looks low. The cause is CPU throttling: with a CPU limit, the container gets a fixed CPU quota per 100 ms period, and bursty request handling uses it up early in the period, so requests wait for the next one. Many teams have published postmortems about this, and it's why some organizations drop CPU limits entirely and rely on requests.

**Build:** Set requests and limits based on your Phase 2 load-test data. Configure an HPA. Run a load test and watch it scale. Then deploy a deliberately broken version and roll back.

**Done when:**
- [ ] HPA scales up under load and back down afterward
- [ ] Rollback restores service within a minute
- [ ] 5-minute recorded explanation

**Recall:** What happens when a container exceeds its memory limit, and what happens when it hits its CPU limit?

### Week 29 — Helm and stateful workloads

**Why:** Helm is how most teams package and deploy applications. Understanding StatefulSets explains why managed databases are usually the better choice in production.

**Learn:**
- Helm charts: templates, values, per-environment values files; `helm upgrade --install` and rollback
- StatefulSets, PersistentVolumeClaims, storage classes
- Jobs and CronJobs for migrations and scheduled tasks

**Real-world example:** Database migrations in Kubernetes are a common trap. Put `alembic upgrade head` in the app's startup and three replicas race to migrate the same database at once. Running migrations as a Helm pre-upgrade hook Job runs them exactly once, before new pods start.

**Build:** A Helm chart for the anchor project with dev and prod values files, migrations as a pre-upgrade Job, and a CronJob for one periodic task.

**Done when:**
- [ ] `helm upgrade --install` deploys everything from scratch
- [ ] Migrations run once per release
- [ ] 5-minute recorded explanation

**Recall:** Why run migrations as a separate Job instead of in the application's startup?

### Week 30 — Kubernetes security [SEC]

**Why:** Kubernetes defaults are permissive. Hardening a cluster is a visible and relatively rare skill for a backend engineer.

**Learn:**
- RBAC: Roles, ClusterRoles, ServiceAccounts; least privilege; disabling token automount where unused
- Pod Security Standards (baseline, restricted); `securityContext`: `runAsNonRoot`, `readOnlyRootFilesystem`, dropping capabilities
- Secrets management options: external secret stores, the Secrets Store CSI driver, encryption at rest
- Scanning a cluster with kubescape

**Real-world example:** In 2018, security researchers discovered that Tesla's Kubernetes administrative console was accessible without a password. Attackers had used it to run cryptocurrency-mining workloads, and cloud credentials were exposed inside it. No zero-day was needed, only an unprotected control plane.

**Build:** Enforce the `restricted` Pod Security Standard on your namespace and fix everything that breaks. Give the app a dedicated ServiceAccount with minimal RBAC. Scan with kubescape and fix the top findings.

**Done when:**
- [ ] Your pods run non-root with a read-only root filesystem
- [ ] kubescape findings are fixed or consciously documented as accepted
- [ ] 5-minute recorded explanation

**Recall:** Why is a pod running as root with default capabilities dangerous, even inside a container?

### Week 31 — Buffer and Phase 5 checkpoint

- [ ] Finish anything that slipped
- [ ] **Phase 5 milestone:** the app runs from your Helm chart with correct probes, resources, HPA, NetworkPolicies and restricted pod security
- [ ] Answer these out loud:
  1. A pod is in CrashLoopBackOff. Walk me through your debugging.
  2. Service A can't reach Service B in the cluster. What do you check, in order?
  3. How do you achieve zero-downtime deployments on Kubernetes?
  4. Give a bad example of a liveness probe and a bad example of a readiness probe.
  5. How would you secure a cluster shared by several teams?
- [ ] Rest

---

## Phase 6 — Azure (Weeks 32–37)

**Why this phase:** You already know Azure from banking, which is a real asset: many EU companies, especially in finance, run on it. This phase turns that experience into the specific skill hiring managers look for: deploying and operating containerized applications with infrastructure as code.

**Main resources:** Microsoft Learn (AKS, Azure networking, managed identities) · *Terraform: Up & Running*

### Week 32 — Terraform fundamentals

**Why:** Infrastructure as code is expected at any serious company. Terraform works across clouds, so it keeps your CV portable beyond Azure.

**Learn:**
- Providers, resources, data sources, variables, outputs
- State: what it is, remote state in Azure Storage with locking, and why state can contain secrets
- Modules; `terraform plan` as a review artifact; drift
- Separate state per environment

**Real-world example:** Two engineers run `terraform apply` at the same time from laptops with local state files. Each overwrites the other's view of reality, and infrastructure gets duplicated or deleted. Remote state with locking exists precisely because this has happened to so many teams.

**Build:** A Terraform project with remote state that creates a resource group, an Azure Container Registry and a Log Analytics workspace, run from CI with a plan step visible in pull requests.

**Done when:**
- [ ] No Terraform state exists on your laptop
- [ ] Pull requests show the `plan` output
- [ ] 5-minute recorded explanation

**Recall:** Why must Terraform state be treated as sensitive?

### Week 33 — Azure networking [NET]

**Why:** Most Azure security and connectivity problems are networking problems. This is where your interest in networking becomes cloud architecture skill.

**Learn:**
- VNets, subnets and address planning (reuse Week 14), VNet peering
- NSGs vs Azure Firewall
- Private Endpoints and Private DNS zones vs Service Endpoints
- NAT Gateway for outbound traffic; Load Balancer vs Application Gateway vs Front Door
- Hub-and-spoke topology, conceptually

**Real-world example:** A team adds a Private Endpoint to their database, yet the app still connects over the public endpoint, or fails outright once public access is disabled. The Private DNS zone was never linked to the app's VNet, so the database hostname still resolves to the public IP. Private networking in Azure is half routing and half DNS.

**Build:** Use Terraform to create a VNet with subnets for AKS, data and private endpoints, plus NSGs and a NAT Gateway. Draw the network diagram and commit it to the repo.

**Done when:**
- [ ] The diagram matches what Terraform actually created
- [ ] You can explain every NSG rule
- [ ] 5-minute recorded explanation

**Recall:** Why does a Private Endpoint need a Private DNS zone to work as expected?

### Week 34 — Identity and secrets: no passwords anywhere [SEC]

**Why:** Leaked credentials are among the most common causes of cloud breaches. Managed identities remove the credential entirely.

**Learn:**
- Microsoft Entra ID basics; Azure RBAC, scopes and least privilege
- Managed identities (system-assigned vs user-assigned); AKS Workload Identity
- Key Vault, and consuming secrets without storing them in code or CI
- The instance metadata service (IMDS), and why SSRF attacks target it
- SAS tokens and their risks

**Real-world example:** In 2023, Microsoft AI researchers shared training data on GitHub using an Azure Storage SAS token that accidentally granted full-control access to the entire storage account: about 38 TB of internal data, including backups of employees' workstations. In 2019, an attacker exploited a server-side request forgery (SSRF) through a misconfigured web application firewall at Capital One, queried the cloud metadata service for credentials, and stole data on around 100 million people. That was AWS, but Azure's IMDS works on the same principle, which is why every identity must be scoped narrowly.

**Build:** The anchor API on AKS reads its secrets from Key Vault through Workload Identity, and connects to Postgres with Entra authentication where possible. Search the repo and CI configuration and confirm that zero secrets remain.

**Done when:**
- [ ] No connection strings or keys exist in the repo, CI variables or container images
- [ ] Each identity has only the roles it needs
- [ ] 5-minute recorded explanation

**Recall:** Why is a managed identity safer than a connection string in an environment variable?

### Week 35 — Deploying to AKS (and knowing when not to)

**Why:** This is the end-to-end deployment story interviewers will ask you to walk through.

**Learn:**
- AKS with Terraform: node pools, ACR integration, cluster autoscaler
- Ingress on AKS (Application Gateway for Containers, or another Gateway API implementation)
- GitHub Actions → Azure via OIDC → Helm deploy
- Azure Container Apps as a simpler alternative, and when AKS is overkill
- Cost control: budgets and alerts, stopping dev clusters, right-sizing node pools

**Real-world example:** Surprise cloud bills are a rite of passage: a forgotten test cluster or load balancer left running for a month. A budget alert takes five minutes to set up on day one and prevents an expensive lesson. For your own products, remember that Container Apps can scale to zero, while an AKS cluster costs money every hour it exists.

**Build:** Deploy the full anchor stack to AKS from CI using OIDC and your Helm chart. Set a budget alert. Write half a page on what the same deployment would look like on Container Apps, and which you'd choose for a small SaaS product.

**Done when:**
- [ ] A merge to `main` deploys to AKS with no manual steps
- [ ] Budget alert configured
- [ ] 5-minute recorded explanation

**Recall:** When would you choose Container Apps over AKS?

### Week 36 — Managed data services and resilience

**Why:** A backup that has never been restored is a hope, not a backup.

**Learn:**
- Azure Database for PostgreSQL Flexible Server: high availability, backups, point-in-time restore
- Azure Managed Redis (Microsoft's successor to Azure Cache for Redis)
- Availability zones; RPO and RTO
- Disaster recovery drills

**Real-world example:** On 31 January 2017, a GitLab engineer accidentally deleted data from the production database while fighting an unrelated incident. When the team reached for backups, they found that none of their five backup and replication methods worked as expected. They lost about six hours of data and live-streamed the recovery. Their public postmortem is required reading for anyone who runs a database.

**Build:** Perform a point-in-time restore of your Postgres to a new server and verify the data. Then `terraform destroy` the whole environment and rebuild it from scratch, timing yourself.

**Done when:**
- [ ] Restore drill completed and documented
- [ ] Full rebuild from nothing in under an hour
- [ ] 5-minute recorded explanation

**Recall:** What's the difference between RPO and RTO?

### Week 37 — Buffer and Phase 6 checkpoint

- [ ] Finish anything that slipped
- [ ] **Phase 6 milestone:** full stack on Azure, deployed through Terraform and CI, zero secrets in the repo, rebuildable from scratch in under an hour
- [ ] Answer these out loud:
  1. Design the network for an Azure-hosted API with a private database.
  2. How does your app authenticate to Key Vault without any secret?
  3. Walk me through your deployment pipeline end to end.
  4. How would you cut this setup's monthly cost by 40%?
  5. The database's region goes down. What happens, and what are your RPO and RTO?
- [ ] Destroy any cloud resources you don't need running
- [ ] Rest

---

## Phase 7 — Observability & security operations (Weeks 38–42)

**Why this phase:** "How do you debug performance in production?" and "How do you structure logs and observability?" are core questions for remote roles, because remote teams can't debug by walking over to your desk. The security weeks close the loop: you attack your own system.

**Main resources:** *Site Reliability Engineering* (Google, free at sre.google) · OpenTelemetry docs · OWASP API Security Top 10 (2023) · PortSwigger Web Security Academy (free)

### Week 38 — Structured logging and audit trails [SEC]

**Why:** Logs are the first thing you open during an incident, and a surprisingly common source of data leaks.

**Learn:**
- Structured JSON logs (structlog); log levels; request/correlation IDs propagated across services
- What never to log: passwords, tokens, full card numbers, unnecessary personal data (GDPR applies to logs too)
- Redaction at the logger level, not by developer discipline
- Audit logs for security-relevant events: logins, permission changes, money movement

**Real-world example:** In 2018, Twitter asked all its users to change their passwords after discovering that a bug had written passwords, unmasked, into an internal log. In 2019, Facebook disclosed that hundreds of millions of user passwords had been stored in plain text in internal logs. Neither was an attack. Both were just logging.

**Build:** Structured logging with request IDs in the anchor project, a redaction processor, tests proving tokens and passwords never appear in logs, and an audit log for sensitive actions.

**Done when:**
- [ ] One request ID lets you find every log line for that request
- [ ] Redaction tests pass
- [ ] 5-minute recorded explanation

**Recall:** How does a correlation ID help when one user request passes through three services?

### Week 39 — Metrics and distributed tracing

**Why:** Metrics tell you *that* something is wrong; traces tell you *where*.

**Learn:**
- RED (rate, errors, duration) for services; USE (utilization, saturation, errors) for resources
- Prometheus: counters, gauges, histograms; why histograms beat averages; label cardinality
- OpenTelemetry: traces, spans, context propagation; auto-instrumentation for FastAPI, SQLAlchemy, Redis and httpx
- Grafana dashboards

**Real-world example:** In March 2024, Microsoft engineer Andres Freund noticed that SSH logins on a test machine were using unusual amounts of CPU and taking about half a second longer than normal. Investigating that tiny performance anomaly, he uncovered a backdoor planted in the widely used xz compression library, on its way into major Linux distributions. Paying attention to performance is also a security skill.

**Build:** Prometheus metrics and a Grafana RED dashboard for the anchor API. OpenTelemetry traces showing API → Redis → Postgres spans. Find your slowest span and explain it.

**Done when:**
- [ ] The dashboard shows rate, errors and p95/p99 latency per endpoint
- [ ] One trace shows the full path of a request
- [ ] 5-minute recorded explanation

**Recall:** Why is average latency misleading, and what do you use instead?

### Week 40 — SLOs, alerting and incident response

**Why:** Mature remote teams run on SLOs and blameless postmortems. Showing that you think this way signals seniority.

**Learn:**
- SLIs, SLOs and error budgets
- Alerting on symptoms (users affected) rather than causes (CPU at 80%); burn-rate alerts
- Runbooks, on-call basics, blameless postmortems

**Real-world example:** Google's SRE book describes error budgets: if a service's SLO is 99.9% availability, the team has about 43 minutes of downtime per month to "spend". When the budget runs out, feature launches pause and reliability work takes priority. It turns the eternal "features vs stability" argument into arithmetic.

**Build:** Define two SLOs for the anchor API, write alert rules for them, and write a runbook. Then break something on purpose (kill Redis during a load test), handle it using only your dashboards and runbook, and write a blameless postmortem.

**Done when:**
- [ ] The alert fired during your simulated incident
- [ ] `docs/postmortems/` contains your first postmortem
- [ ] 5-minute recorded explanation

**Recall:** Why alert on SLO burn rate instead of on CPU usage?

### Week 41 — Threat modeling and attacking your own API [SEC]

**Why:** This week turns your security background into a portfolio artifact few backend candidates have: a threat model and a security test of a real system you built.

**Learn:**
- Threat modeling with STRIDE; data-flow diagrams; trust boundaries
- Walking through the OWASP API Security Top 10 against your own endpoints
- Dynamic scanning with ZAP; manual testing with Burp Suite Community Edition
- Security headers and CORS pitfalls
- Patch management and vulnerability triage

**Real-world example:** In 2017, Equifax was breached through a vulnerability in Apache Struts for which a patch had been available for about two months; data on roughly 147 million people was stolen. When Log4Shell hit in December 2021, the teams that patched within hours were the ones that already knew exactly where Log4j was running. That's what the SBOMs and dependency scanning from Week 23 buy you.

**Build:** Write a STRIDE threat model for the anchor project (one page plus a data-flow diagram). Run ZAP against it, test each OWASP API Top 10 item manually, fix what you find, and add a `SECURITY.md` describing your controls.

**Done when:**
- [ ] Threat model and `SECURITY.md` committed
- [ ] Every ZAP finding is fixed or documented as accepted, with a reason
- [ ] 5-minute recorded explanation

**Recall:** Name the six STRIDE categories and give one example of each in your API.

### Week 42 — Buffer and Phase 7 checkpoint

- [ ] Finish anything that slipped
- [ ] **Phase 7 milestone:** dashboards, SLOs with alerts, one postmortem, a threat model and fixed scan findings
- [ ] Answer these out loud:
  1. How do you debug a production performance problem you can't reproduce locally?
  2. How do you structure logs so they're both useful and safe?
  3. Tell me about an incident and what you changed afterward. (Use Week 40.)
  4. How would you threat-model a new payments endpoint?
  5. A critical CVE is announced in a library you use. What do you do in the first hour?
- [ ] Rest

---

## Phase 8 — Portfolio & interview readiness (Weeks 43–44)

### Week 43 — Build the evidence pack

**Why:** Everything you've built is invisible until it's presented well. For remote roles especially, a clear repository does part of the interviewing for you.

- [ ] README with an architecture diagram, local setup instructions and key design decisions
- [ ] `docs/adr/` with 5–8 Architecture Decision Records (e.g., "Why cursor pagination", "Why Redis Streams over Pub/Sub", "Why Container Apps for the product, AKS for learning")
- [ ] Performance report, network diagram, threat model and postmortem linked from the README
- [ ] CV and LinkedIn updated with concrete, measured outcomes from your own reports (e.g., "reduced p95 latency from X ms to Y ms through caching and query fixes")
- [ ] Two short write-ups or blog posts from your most interesting weeks (the race-condition test and the packet capture make good stories)

### Week 44 — Mock interviews and next steps

- [ ] Two system design mocks, 45 minutes each, recorded (use Appendix A)
- [ ] One debugging-scenario mock ("the API is slow", "pods can't reach Redis")
- [ ] One 10-minute "walk me through your project" rehearsal, without notes
- [ ] Review Appendix B (the "not now" list) and decide what, if anything, comes next

---

## Appendix A — Weekly design drills (30 minutes)

Each week, pick the next drill for your current phase. Sketch the components, data flow, failure modes and scaling limits, then explain the design out loud as if to an interviewer. Repeat earlier drills in later phases: your answers should visibly improve as you learn more.

| Phase | Drills |
|---|---|
| 1 — FastAPI | URL shortener · Idempotent payments API · Multi-tenant SaaS API with per-tenant data isolation · Paginated activity feed |
| 2 — Redis | Rate limiter for a public API · Real-time leaderboard · Session store for 1M users · Distributed job scheduler |
| 3 — Networking | Low-latency API for users across the EU · Webhook delivery system (retries, signatures, backoff) · Service-to-service authentication with mTLS |
| 4 — Docker & CI/CD | CI/CD for a repository with five services · Blue-green vs canary deployment system |
| 5 — Kubernetes | Platform for several teams on one cluster · Zero-downtime database migration during rolling deploys |
| 6 — Azure | Hub-and-spoke network for three teams · Disaster recovery plan for a fintech API |
| 7 — Observability & security | Audit-logging system for a bank · Event pipeline for fraud detection · Notification system (email, SMS, push) |

---

## Appendix B — Not now (parking lot)

When a job post makes one of these feel urgent, add it here instead of starting it. Review this list only in Week 44.

- Go or Rust
- Service meshes (Istio, Linkerd)
- A second cloud provider (AWS, GCP)
- Kafka in depth (Redis Streams teaches the core concepts)
- ML engineering
- Certifications
- ______________________
- ______________________

---

## Appendix C — Evidence log

Copy this block once per week. Keep entries short; honesty matters more than polish.

```markdown
## Week __ — <topic>  (date: ______)
Built:            <what exists now that didn't last week>
Measured:         <any number: latency, image size, test count, rebuild time>
Can now explain:  <one concept, in one sentence>
Was hard:         <what confused me, and what finally made it click>
Job applications: <how many sent this week>
```

### When impostor syndrome shows up

Open a job posting that made you feel "not enough" and go through its requirements one by one:

1. Which of these have I **built** (link to the evidence log entry)?
2. Which can I **explain**, even if I haven't built them at production scale?
3. Which are genuinely new? Are they in this plan, or in the "not now" list?

If items 1 and 2 cover the core stack, apply. The feeling of "I should know more" never fully goes away for good engineers; the evidence log is how you argue with it using facts.

---

## Appendix D — Core reading list (only these)

Resist adding more. Depth in a few sources beats skimming many.

| Resource | Use it for |
|---|---|
| FastAPI documentation (Tutorial + Advanced User Guide) | Phase 1 |
| *Architecture Patterns with Python* (cosmicpython.com, free) | Phase 1 project structure |
| *Designing Data-Intensive Applications* (Martin Kleppmann) | System design; one or two chapters per phase |
| Redis documentation and Kleppmann's "How to do distributed locking" | Phase 2 |
| *High Performance Browser Networking* (hpbn.co, free) | Phase 3 |
| Julia Evans' networking zines | Phase 3, for quick mental models |
| *Kubernetes: Up and Running* | Phase 5 |
| *Terraform: Up & Running* | Phase 6 |
| *Site Reliability Engineering* (sre.google, free) | Phase 7 |
| OWASP API Security Top 10 (2023) and PortSwigger Web Security Academy (free) | [SEC] weeks throughout |

**Postmortems worth reading** (one per phase is plenty): Reddit's "You Broke Reddit: The Pi-Day Outage" (2023) · GitLab's database incident report (2017) · Facebook's explanation of the October 2021 outage · AWS's summary of the October 2025 us-east-1 event · Knight Capital (2012, via the SEC's findings)
