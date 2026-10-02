# Backend Engineering — Interview Prep

Deep-dive notes on backend engineering concepts and interview questions, aimed at a **3–4 YOE backend engineer** job search. Compiled from a focused round of research (current interview-prep guides, engineering blogs, and official docs) in September 2026.

## Scope — how this fits with your other repos

Four repos form one prep pipeline. The same topic name can legitimately appear in more than one of them — the boundary is **depth**, not topic:

| Repo | Depth | What it owns |
|---|---|---|
| `HLD-learning` | Shallow, vocabulary pass | Name the concept, 1-2 line trade-off — just enough to not freeze mid system-design interview |
| **This repo** | Deep, theory-only | Mechanism internals, trade-offs, production gotchas, layered interviewer follow-up questions — no code |
| `LLD-learning` | Hands-on, timed | All coding exercises: OOP/SOLID/design patterns plus the implementable problems (rate limiter, circuit breaker, LRU cache, notification service, pub/sub, etc.) |
| `Go-Learning` | Deep, language-only | Goroutines/channels internals, GC, scheduler, memory model — a different topic domain entirely |
| Database engineering | Deep, DB-only | SQL/NoSQL internals, indexing, transactions/ACID, query optimization, replication/sharding |

What **is** covered here: the practical "glue" engineering knowledge that every backend interview at the 3–4 YOE level probes — API design, auth & security, caching, messaging, resilience, observability, testing, networking, deployment, and architectural patterns — at the deep-theory-plus-follow-ups level described above.

### Where to go deeper for the same topic

| Topic (this repo) | `HLD-learning` vocabulary section | `LLD-learning` coding exercise |
|---|---|---|
| API Design | APIs, Load Balancers & Gateway | Rate Limiter (partial overlap) |
| Networking & Protocols | — | — |
| Auth & Security | Security Basics | — |
| Testing Strategies | — | — |
| Caching | Caching | In-memory LRU Cache |
| Messaging & Event-Driven Systems | Messaging & Async Processing | Notification Service, Pub/Sub System |
| Resilience & Fault Tolerance | Reliability & Observability | Rate Limiter, Circuit Breaker |
| Observability | Reliability & Observability | — |
| Deployment, CI/CD & Cloud | — | — |
| Architectural Patterns | — | — |

A "—" means that topic stays theory-only across the whole pipeline: no coding exercise exists for it in `LLD-learning`, and it isn't part of `HLD-learning`'s vocabulary list either.

## Topics

1. [API Design](01-api-design.md) — REST principles, HTTP status codes, versioning, pagination, idempotency, rate limiting, REST vs GraphQL vs gRPC, error response design
2. [Networking & Protocols](08-networking-and-protocols.md) — TCP vs UDP, HTTP/1.1 vs HTTP/2 vs HTTP/3, TLS handshake, DNS, load balancers, reverse proxies, WebSockets vs SSE vs polling
3. [Authentication, Authorization & Security](02-auth-and-security.md) — sessions vs tokens, JWT internals, OAuth2/OIDC flows, RBAC/ABAC, OWASP Top 10 (web + API), input validation, password hashing, secrets management
4. [Testing Strategies](07-testing-strategies.md) — testing pyramid, unit/integration/contract/e2e testing, mocking/test doubles, TDD, load & performance testing
5. [Caching](03-caching.md) — cache-aside/read-through/write-through/write-behind, invalidation, cache stampede, eviction policies, HTTP caching headers, CDNs, Redis vs Memcached
6. [Messaging & Event-Driven Systems](04-messaging-and-event-driven.md) — Kafka & RabbitMQ concepts, delivery semantics, idempotent consumers, DLQs, outbox pattern, pub/sub
7. [Resilience & Fault Tolerance](05-resilience-and-fault-tolerance.md) — circuit breaker, retry with backoff/jitter, bulkhead, timeouts, fallback strategies, load shedding, graceful degradation
8. [Observability](06-observability.md) — logs/metrics/traces, RED/USE methods, distributed tracing & OpenTelemetry, SLI/SLO/SLA, alerting
9. [Deployment, CI/CD & Cloud](09-deployment-cicd-and-cloud.md) — 12-factor apps, Docker, Kubernetes basics, CI/CD pipelines, deployment strategies, config & secrets management
10. [Architectural Patterns](10-architectural-patterns.md) — monolith vs microservices vs SOA, API gateway/BFF, service discovery, sidecar/service mesh, saga pattern, strangler fig

## Suggested study order

- **Phase 1 — Request and service fundamentals:** API Design → Networking → Auth & Security
- **Phase 2 — Build confidence:** Testing → Caching → Messaging
- **Phase 3 — Production behavior:** Resilience → Observability → Deployment
- **Phase 4 — Architecture vocabulary:** Architectural Patterns

For each topic, follow the same interview loop: learn the core concept, work through a layered follow-up drilling chain, review common gotchas, answer interview questions, and close with a one-line pointer to the matching `LLD-learning` exercise or `HLD-learning` section (see the table above) if one exists.

Each file ends with a rapid-fire Q&A list for self-testing.
