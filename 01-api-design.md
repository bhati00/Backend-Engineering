# 1. API Design

## Why this shows up in interviews

API design is usually the *warm-up* topic — the first 10 minutes of an HLD round, or the whole subject of a "design a REST API for X" LLD round. It's low-depth-of-knowledge but high-frequency: almost every backend interview touches at least one of REST semantics, pagination, idempotency, or error design. The bar at 3-4 YOE is not "have you used GraphQL in production" — it's "can you justify a design choice instead of naming a technology."

---

## 1. REST Principles

REST is an architectural style, not a protocol. The parts that actually get probed in interviews:

- **Resources, not actions** — URLs name nouns (`/orders/123`), HTTP methods supply the verb. `POST /createOrder` is an interview red flag.
- **Statelessness** — each request carries everything needed to process it (auth token, etc.); the server holds no client session between requests. This is *why* REST APIs scale horizontally without sticky sessions.
- **Uniform interface** — same method semantics everywhere (a `DELETE` always means delete, regardless of resource).
- **HATEOAS** — responses include links to related/next actions (e.g., an order response links to its `cancel` action). Name-recognition only at this level; almost no real-world API fully implements it, and interviewers rarely expect you to have used it — just know what it means if asked.
- **Richardson Maturity Model** (vocabulary only): Level 0 (single endpoint, RPC-over-HTTP) → Level 1 (resources) → Level 2 (HTTP verbs + status codes, where almost all "RESTful" APIs actually live) → Level 3 (HATEOAS).

| Method | Safe (no side effect)? | Idempotent? | Typical use |
|---|---|---|---|
| GET | Yes | Yes | Read a resource |
| HEAD | Yes | Yes | GET without body (metadata only) |
| PUT | No | **Yes** | Full replace of a resource |
| DELETE | No | **Yes** | Delete (repeat = still deleted) |
| PATCH | No | Not guaranteed | Partial update — idempotent only if you design it that way |
| POST | No | **No** | Create a new resource / non-idempotent action |

The PUT/PATCH/POST row is one of the most common quick-fire interview questions — know it cold.

---

## 2. HTTP Status Codes

Don't recite the full list — interviewers probe the ones that are commonly *confused*:

| Code | Meaning | Common confusion |
|---|---|---|
| 400 | Bad Request — malformed syntax | vs 422: 400 = server can't even parse it |
| 401 | Unauthorized | Actually means **unauthenticated** — "I don't know who you are" |
| 403 | Forbidden | **Authenticated but not authorized** — "I know you, you can't do this" |
| 404 | Not Found | vs 410: use 410 Gone when a resource is *permanently* removed and you want to signal "don't retry" |
| 409 | Conflict | Request conflicts with current resource **state** (e.g., version/ETag mismatch, duplicate unique key) |
| 422 | Unprocessable Entity | Syntactically valid, **semantically invalid** (failed validation) |
| 429 | Too Many Requests | Rate limit hit — pair with a `Retry-After` header |
| 500 | Internal Server Error | Unhandled server-side failure |
| 503 | Service Unavailable | Server is deliberately not serving (overloaded/maintenance) — vs 500 which is unexpected |

Interview gotcha: a very common anti-pattern is returning **200 OK with an error payload** (`{ "success": false, ... }`). This breaks HTTP semantics — monitoring, caching, and generic HTTP clients all key off the status code, not the body.

---

## 3. API Versioning

| Strategy | Example | Pros | Cons |
|---|---|---|---|
| URI path | `/v1/orders` | Most visible/discoverable, trivial to route | "Not very RESTful" (a resource now has multiple URLs); breaks the idea that a URL identifies *one* resource |
| Custom header | `X-API-Version: 2` | Keeps URLs clean | Invisible — harder to discover/test from a browser, easy to forget |
| Content negotiation | `Accept: application/vnd.company.v2+json` | Most "correct" REST-wise | Least ergonomic for consumers; tooling support is weaker |
| Query param | `?version=2` | Simple | Easy to omit accidentally, caching gets messy |

Real-world precedents worth naming in an interview:
- **Stripe** uses *dated* versions resolved via a header (e.g., `2024-06-20`) — every account is pinned to a version at signup and upgrades explicitly. Considered the gold standard for not breaking anyone silently.
- **GitHub's REST API** historically used `Accept: application/vnd.github.v3+json` header-based media-type versioning.

**Breaking vs non-breaking change** is the real concept being tested, not the URL mechanics:
- Non-breaking: adding a new optional field, adding a new endpoint, adding a new enum value a client can ignore.
- Breaking: removing/renaming a field, changing a field's type, changing validation to reject previously-valid input, changing status code semantics.

**Deprecation**, done well: announce ahead of time, run old + new in parallel for a grace period, signal it with `Deprecation` / `Sunset` response headers, and give a hard removal date in docs/changelog — never a silent cutover.

---

## 4. Pagination

| | Offset/limit | Cursor (keyset) |
|---|---|---|
| Query shape | `?page=3&size=20` → `OFFSET 40 LIMIT 20` | `?cursor=eyJpZCI6MTIzfQ==` → `WHERE (created_at, id) > (?, ?) ORDER BY ... LIMIT 20` |
| Random access ("jump to page 7") | Yes | No — inherently sequential |
| Performance at scale | `OFFSET n` still scans and discards `n` rows in most SQL engines — cost grows with page depth | Index seek — flat cost regardless of depth |
| Stability under concurrent writes | **Drifts**: an insert/delete shifts every row after it, so users can see duplicates or skip rows while paging | Stable: cursor encodes a position relative to actual row identity, not row count |
| Typical use | Admin tables, "jump to page N" UIs | Public APIs, infinite scroll, feeds, anything with high write volume |

A cursor must encode a **unique, stable sort key tuple** — e.g., `(created_at, id)`, not `created_at` alone. If two rows share the same timestamp and you only key on that, ties cause skipped or duplicated rows at the page boundary. This is a very common bug in home-grown cursor implementations.

Practitioner note worth repeating in an interview (echoed in r/webdev and r/node discussions on this): **API pagination is a contract, not an implementation detail** — you can expose a cursor externally while still translating it to an offset-style query internally, or vice versa. Don't assume the two are the same thing.

---

## 5. Idempotency

**Definition**: an operation is idempotent if performing it N times has the same effect as performing it once. Retries are the whole reason this matters — timeouts, load-balancer retries, and at-least-once message queues all cause the *same logical request* to arrive more than once.

```
Idempotent:      SET balance = 100        (repeat 5x → still 100)
Not idempotent:  balance += 100           (repeat 5x → balance +500!)
                 POST /orders {items:[]}  (repeat 5x → 5 orders created!)
```

**The fix — an idempotency key.** Client generates one unique key per *logical* operation (not per HTTP attempt) and sends it on every retry of that same operation:

```
Attempt 1:  POST /payments
            Idempotency-Key: pay_a1b2c3
            → server: key unseen → charge card → store {key -> response} → return 200

Retry (same key, after a timeout):
            POST /payments
            Idempotency-Key: pay_a1b2c3     <- identical key
            → server: key found → return the STORED response, no new charge
```

Implementation sketch (atomicity is the whole point — see Follow-up Drilling below):

```
function handle(request):
    key = request.header["Idempotency-Key"]
    existing = store.get(key)
    if existing: return existing.response

    claimed = store.claim(key)          # atomic insert / SETNX — must not race
    if not claimed: return 409          # another in-flight request owns this key

    result = do_business_logic(request)
    store.save(key, result, ttl = 24h)
    return result
```

Store choice: Redis (`SET key val NX EX 86400` — fast, but can lose data on restart/failover) vs a DB table with a unique constraint on the key (`INSERT ... ON CONFLICT DO NOTHING` — durable, slightly slower). Production payment systems often use the DB for durability and only use Redis as a cache in front of it.

Real-world precedent: **Stripe** requires an `Idempotency-Key` header on POST requests (24h retention). **PayPal** uses `PayPal-Request-Id`. **AWS** uses client tokens (e.g., `ClientToken` on EC2 `RunInstances`).

Ties back to Messaging & Event-Driven topic: **at-least-once delivery + an idempotent consumer = exactly-once semantics** in practice — this is the standard way distributed systems fake "exactly-once."

---

## 6. Rate Limiting

| Algorithm | Mechanism | Allows bursts? | Memory cost | Weakness |
|---|---|---|---|---|
| Fixed window | Count requests in the current fixed interval, reset at boundary | Yes, at window edges | O(1) | Boundary burst: up to 2x the limit right across a window edge |
| Sliding window (log) | Store every request timestamp, count those within the trailing window | No | O(requests in window) | Memory grows with traffic |
| Token bucket | Tokens refill at a steady rate into a capped bucket; each request consumes a token | Yes, up to bucket capacity | O(1) | Needs tuned capacity/refill rate |
| Leaky bucket | Requests queue and drain at a constant rate | No — smooths to a constant rate | O(queue size) | Adds queueing delay; doesn't reward idle periods with burst capacity |

**Where to enforce it**: at the edge/API gateway (one chokepoint, simplest to reason about) vs per-service (more granular, but see the distributed-counter gotcha in Follow-up Drilling).

**Response design**: `429 Too Many Requests` + `Retry-After` header is mandatory for well-behaved clients to back off correctly. The de facto convention is `X-RateLimit-Limit` / `X-RateLimit-Remaining` / `X-RateLimit-Reset` headers (GitHub, Twitter use this shape); an IETF draft is working to standardize this as `RateLimit-*`.

---

## 7. REST vs GraphQL vs gRPC

| | REST | GraphQL | gRPC |
|---|---|---|---|
| Data format | JSON/XML, text-based | JSON | Protocol Buffers (binary) |
| Performance | Good | Good, but query cost varies | Fastest (binary + HTTP/2) |
| Caching | Easy (HTTP caching, per-URL) | Hard (single endpoint, client-shaped queries) | Manual |
| Over/under-fetching | Common problem | Solved — client asks for exact fields | N/A (fixed contract) |
| Schema/contract | Informal (OpenAPI optional) | Strongly typed schema | Strongly typed `.proto` contract, codegen |
| Browser support | Universal | Universal | Needs a grpc-web proxy |
| Streaming | No (polling/webhooks/SSE instead) | Subscriptions (via extra transport) | Native (unary/client/server/bidi streaming) |
| Best for | Public APIs, simple CRUD, caching-sensitive | Client-driven complex/nested data, mobile (bandwidth-constrained) | Internal service-to-service, high-performance, streaming |

Known pain points worth naming proactively:
- **GraphQL's N+1 problem**: a naive resolver-per-field design issues one DB query per item in a list (e.g., 1 query for 50 posts + 50 queries for each post's author). Fixed with request-scoped batching (the "DataLoader" pattern) — GraphQL moves the over-fetching problem from the client to the backend if you're not careful.
- **gRPC and protobuf compatibility**: compatibility is about field **numbers/tags**, not field names — renaming a field is safe, reusing or changing a field number is a wire-breaking change. A common gotcha for anyone used to JSON's name-based compatibility assumptions.

Practitioner sentiment (from r/webdev and r/programming discussion threads on this comparison): GraphQL is genuinely loved by *frontend consumers* of an API, but the backend complexity it adds (resolvers, batching, query-cost limiting to prevent abuse) is a real cost — several engineers describe defaulting to REST unless there's a concrete, compelling reason (highly nested/client-varying data, many client types with different data needs) to pay that cost.

---

## 8. Error Response Design

Goals: consistent shape across every endpoint, machine-readable (clients can branch on a `type`/`code`) *and* human-readable, and **never leak internals** (stack traces, SQL fragments, file paths) — that's an information-disclosure issue under the OWASP API Security Top 10's security-misconfiguration category, not just a style nit.

**Current standard: RFC 9457 "Problem Details for HTTP APIs"** (obsoletes the older RFC 7807), content-type `application/problem+json`:

```json
HTTP/1.1 403 Forbidden
Content-Type: application/problem+json

{
  "type": "https://example.com/probs/out-of-credit",
  "title": "You do not have enough credit.",
  "detail": "Your current balance is 30, but that costs 50.",
  "instance": "/account/12345/msgs/abc",
  "balance": 30
}
```

Core members: `type` (URI identifying the problem category), `title` (stable short summary), `status`, `detail` (specific to this occurrence), `instance` (URI of this specific occurrence) — plus free-form extension fields for domain data. Frameworks increasingly generate this automatically (ASP.NET Core, Spring Boot 3+). For validation errors specifically, extend it with a field-level `errors` array (each entry a `detail` + a JSON Pointer to the offending field).

Separately, design for **retryability**: a well-designed error response lets a client's retry logic decide safely — 5xx and 429 are generally retryable (with backoff), 4xx (other than 429) generally are not, since retrying an unmodified bad request will just fail again.

---

## Follow-up Drilling

These are the layered, production-judgment questions an interviewer chains on top of the base answer — this is what actually separates a junior answer from a mid-level one. Try answering each before reading the next; they get progressively harder.

### Chain A — Idempotent payment API
1. **Baseline**: `POST /payments` times out client-side; the client retries the exact same request. How do you guarantee the customer isn't charged twice?
2. **Race condition**: Two identical retries land at the same instant. What breaks in a naive "check the store, then insert" implementation, and how do you fix it?
3. **Partial failure**: the server crashes *after* charging the card but *before* saving the idempotency record. The client retries. What happens, and how do you prevent a double charge?
4. **Misuse/conflict**: the same idempotency key arrives again, but with a *different* request body (different amount). What should the server do?
5. **Boundary**: how long do you retain idempotency keys, and what's the actual risk the moment after they expire?

<details>
<summary>Expected reasoning (check your answer against this)</summary>

1. Client-generated `Idempotency-Key` header per logical operation; server stores the key → response mapping and returns the stored response on a repeat.
2. A plain "SELECT then INSERT" has a TOCTOU gap — both retries can pass the check before either writes. Fix: make the *claim* atomic (DB unique constraint + `INSERT ... ON CONFLICT`, or Redis `SETNX`) *before* doing the business logic, not after.
3. This is the genuinely hard part: the charge and the idempotency-record write must be atomically consistent. Either (a) do both in one DB transaction, (b) use the outbox pattern, or (c) push the idempotency key down to the payment processor itself so *it* dedupes (what Stripe/card networks actually do).
4. Compare a hash of the stored request body against the incoming one; if they differ, return `409 Conflict` or `422` — never silently process either version. This is a client bug, not a retry.
5. Typically 24h (Stripe's choice) — balances storage cost against a realistic retry window. After expiry, a very late retry is treated as brand new — an explicit, accepted limitation, not a solved edge case.
</details>

### Chain B — Cursor pagination under concurrent writes
1. **Baseline**: with `?page=3&size=20` (offset pagination), a new row is inserted while a user pages through results. What do they experience, and why?
2. **Mechanism**: how does cursor-based pagination avoid this, and what exactly does the cursor encode?
3. **Boundary**: what breaks if the cursor is built from a single non-unique column (e.g., `created_at` alone, no tiebreaker)?
4. **Real constraint**: product wants a "jump to page 7" control. Can cursor pagination support that cheaply?

<details>
<summary>Expected reasoning</summary>

1. Rows shift under them — `OFFSET` recomputes position from the table's current order every query, so a user can see a duplicate row (if a row was inserted before their position) or silently miss one (if a row was deleted before their position).
2. The cursor encodes the last-seen sort key tuple; the query becomes a keyset lookup (`WHERE (created_at, id) > (?, ?)`), which doesn't depend on row count at all — stable regardless of concurrent writes.
3. Rows sharing the same `created_at` tie-break in an undefined order, causing skipped or duplicated rows exactly at that boundary — must add a unique column (e.g., `id`) as a tiebreaker in both the `ORDER BY` and the cursor.
4. No — cursor pagination is inherently sequential/no-random-access. Either accept offset's drift trade-off for that one admin-style view, or question whether the product need is really "jump to page N" (usually it's actually "infinite scroll," which cursors handle perfectly).
</details>

### Chain C — Rate limiting at scale
1. **Baseline**: you rate-limit 100 req/min/user with an in-memory counter, running 5 service instances behind a load balancer. What's the *actual* effective limit?
2. **Fix**: how do you enforce the limit globally instead?
3. **New cost**: that fix now puts a shared store on the critical path of every request. What's the trade-off, and what do real systems do about it?
4. **Fairness**: how do you rate-limit fairly when one user's requests vary wildly in cost (a cheap `GET` vs. an expensive bulk-export `POST`)?

<details>
<summary>Expected reasoning</summary>

1. Up to 500 req/min — each instance counts independently, so the limit effectively multiplies by instance count. Classic distributed rate-limiting bug.
2. Centralize counter state (Redis `INCR`+`EXPIRE`, or a Lua script to keep token-bucket check-and-decrement atomic), or enforce once at a single chokepoint like the API gateway/edge instead of per-service.
3. Adds a network round-trip to every request and couples API availability to the store's availability. Mitigations real systems use: local approximate counters with periodic async sync ("eventually consistent" limiting), and an explicit fail-open vs. fail-closed decision for when the store itself is down (usually fail-open, accepting temporary over-limit risk, to protect availability).
4. Weighted/cost-based limiting — each request consumes a variable number of tokens based on estimated cost rather than a flat 1-request-1-unit model (this is how GitHub's GraphQL API and Twitter's API rate limits actually work).
</details>

---

## Gotchas & Edge Cases

- Returning `200 OK` with `{"success": false}` in the body instead of a real 4xx/5xx — breaks monitoring, caching, and any generic HTTP tooling that keys off status codes.
- Treating `PATCH` as automatically idempotent — it's only idempotent if you design it as absolute field assignment (`{"status": "cancelled"}`), not a delta (`{"quantity_delta": -1}`).
- Leaking stack traces/SQL/file paths in error bodies — an information-disclosure vulnerability, not just an aesthetic issue.
- Shipping a breaking change to a public API without a deprecation window — breaks every integrator simultaneously (the common cautionary example here is Twitter's abrupt v1.1 API cutover).
- Omitting rate-limit headers — clients can't implement correct backoff, which leads to retry storms that make an overload event worse.
- Deep offset pagination (`OFFSET 1000000`) — the DB still scans and discards a million rows even though it returns 20; a real performance cliff under growth, not just a theoretical one.
- Scoping idempotency keys globally instead of per-user/per-API-key — a key collision across tenants can leak or misroute another user's cached response.
- Assuming protobuf field *names* control wire compatibility in gRPC — it's field *numbers* that matter; reusing a retired field number is a silent, dangerous breaking change.

---

## Interview Questions

**Conceptual**
- What's the practical difference between `PUT` and `PATCH`?
- Why is `POST` not idempotent by default, and how do you make a POST endpoint safe to retry?
- What's the difference between `401` and `403`? Between `409` and `422`?
- When would you reach for GraphQL instead of REST, and what does that choice cost you operationally?

**Debugging / scenario**
- Customers occasionally report duplicate orders under flaky network conditions. Where do you look first, and what do you add?
- Your rate limiter behaves correctly in staging (1 instance) but users report far exceeding the limit in production (10 instances). Diagnose it.
- A partner integration says your cursor-based pagination intermittently skips records. What's your hypothesis, and how would you confirm it?

**Trade-off**
- Offset vs. cursor pagination — is there a situation where you'd still choose offset, despite its weaknesses?
- URI-path vs. header-based versioning — which would you pick for a public API with thousands of third-party integrators, and why?
- Token bucket vs. sliding window — which fits a billing-sensitive rate limit (e.g., a metered paid API tier), and why?

---

## Review

- REST principles and status codes are vocabulary — know them cold, but they rarely carry the interview on their own.
- Versioning and pagination are "pick a strategy and defend the trade-off" questions, not "name the right answer" questions.
- Idempotency and rate limiting are where interviewers actually probe production judgment — expect a drilling chain like the ones above, not a single question.
- REST vs. GraphQL vs. gRPC is a trade-off/judgment question — anchor your answer in *what the client needs* (simple CRUD vs. client-shaped queries vs. internal high-performance streaming), not technology preference.
- Error response design is small but easy to get wrong in a way that's actually a security issue (information disclosure), not just a style issue.
- The shallow vocabulary pass for this whole topic lives in `HLD-learning`'s "APIs, Load Balancers & Gateway" section. The one implementable overlap in `LLD-learning` is the **Rate Limiter** exercise (token bucket / sliding window implementation) — do that there if you want hands-on practice; this repo stays theory-only.
- Next up per the suggested study order: **Networking & Protocols**.

---

## Rapid-fire Q&A

1. **Is `PUT` idempotent?**<br>
   **A:** Yes — repeating it produces the same end state.
2. **Is `POST` idempotent by default?**<br>
   **A:** No — needs an idempotency key to make retries safe.
3. **401 vs 403?**<br>
   **A:** 401 = not authenticated ("who are you"); 403 = authenticated but not authorized ("I know you, you can't").
4. **409 vs 422?**<br>
   **A:** 409 = conflicts with current resource state; 422 = well-formed request, failed validation.
5. **Status code for a rate-limited request?**<br>
   **A:** `429`, with a `Retry-After` header.
6. **Offset pagination's main weakness at scale?**<br>
   **A:** `OFFSET n` still scans and discards `n` rows in most SQL engines, and results drift under concurrent writes.
7. **What must a cursor encode to avoid skip/duplicate bugs?**<br>
   **A:** A unique, stable sort-key tuple — not a non-unique column alone.
8. **What problem does an `Idempotency-Key` header solve?**<br>
   **A:** Makes retried non-idempotent requests (like POST) safe to repeat.
9. **Why must the idempotency "claim" be atomic?**<br>
   **A:** To avoid a race where two concurrent retries both pass a check before either has recorded its claim.
10. **Token bucket vs leaky bucket — which allows bursts?**<br>
    **A:** Token bucket (unused tokens accumulate); leaky bucket smooths to a constant rate.
11. **Why does per-instance in-memory rate limiting fail at scale?**<br>
    **A:** Each instance counts independently, so the effective global limit multiplies by instance count.
12. **GraphQL's classic backend performance pitfall?**<br>
    **A:** The N+1 query problem from naive per-field resolvers — fixed with batching (e.g., DataLoader).
13. **Why avoid stack traces in error responses?**<br>
    **A:** Information disclosure — leaks internals attackers can use.
14. **What does RFC 9457 standardize?**<br>
    **A:** A consistent `type`/`title`/`status`/`detail`/`instance` JSON shape for HTTP API errors.
15. **Best practice for deprecating a public API version?**<br>
    **A:** Announce ahead of time, run old + new in parallel, signal via `Deprecation`/`Sunset` headers — never a silent cutover.
