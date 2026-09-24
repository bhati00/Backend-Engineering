# Backend Engineering — Interview Prep

Deep-dive notes on backend engineering concepts and interview questions, aimed at a **3–4 YOE backend engineer** job search. Compiled from a focused round of research (current interview-prep guides, engineering blogs, and official docs) in September 2026.

## Scope — how this fits with your other repos

This repo deliberately covers the topics that sit *between* language internals, low-level design, high-level system design, and databases, so it doesn't duplicate what's already built elsewhere.

| Already covered elsewhere | NOT duplicated here |
|---|---|
| Go internals | Goroutines/channels internals, GC, scheduler, memory model, language-specific runtime behavior |
| LLD | OOP design, SOLID, GoF design patterns, class-diagram exercises |
| HLD | Full system-design case studies (design Twitter/Uber/TinyURL), whole-system scalability/sharding/CAP discussions |
| Database engineering | SQL/NoSQL internals, indexing internals, transactions/ACID, query optimization, DB replication/sharding |

What **is** covered here: the practical "glue" engineering knowledge that every backend interview at the 3–4 YOE level probes — API design, auth & security, caching, messaging, resilience, observability, testing, networking, deployment, and architectural patterns. Where a topic overlaps with system design at the *pattern* level (e.g., circuit breakers, sagas), it's covered here at the concept/pattern level; full-scale system design write-ups belong in your HLD repo.

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

For each topic, follow the same interview loop: learn the core concept, complete a small Go exercise, test edge cases, review common gotchas, answer follow-up questions, and finish with a short HLD scale-up discussion. The deeper end-to-end service comes after this first pass.

Each file ends with a rapid-fire Q&A list for self-testing.
