# 2. Networking & Protocols

## Why this shows up in interviews

Networking & Protocols rarely gets its own standalone round at the 3-4 YOE bar — it shows up as rapid-fire vocabulary checks woven into an HLD conversation ("why would you pick WebSockets here?", "what is your load balancer actually doing at L7?") or as a quick sanity check early on. Nobody expects you to have configured BGP or hand-rolled a TCP stack. What's actually being probed is whether you understand what's happening *underneath* the APIs you design — because caching headers, TLS termination, rate limiting, and resilience timeouts are all decisions made at this layer, not the application layer. Fluency here, not depth of production war stories, is the bar.

---

## 1. TCP vs UDP

Both are transport-layer protocols (the layer directly below HTTP) — they exist to get bytes from one process to another across a network, but they make opposite trade-offs between reliability and speed.

| | TCP | UDP |
|---|---|---|
| Connection model | Connection-oriented — 3-way handshake before any data | Connectionless — just send |
| Reliability | Guaranteed delivery via ACKs + retransmission | Best-effort, no delivery guarantee |
| Ordering | In-order byte stream | No ordering guarantee |
| Flow/congestion control | Yes — sliding window, slow start, congestion avoidance | None |
| Header overhead | 20-60 bytes | 8 bytes (fixed) |
| Data model | Continuous byte stream | Independent messages (datagrams) |
| Typical users | HTTP/1.1, HTTP/2, FTP, SMTP, SSH | DNS, DHCP, VoIP, video streaming, HTTP/3 (via QUIC) |

```
TCP three-way handshake (connection setup):
  Client --SYN-->       Server
  Client <--SYN,ACK--   Server
  Client --ACK-->       Server    (connection established, data can flow)

TCP teardown (why TIME_WAIT exists):
  Client --FIN-->  Server --ACK-->  Client --FIN-->  Server --ACK-->  Client
  Client holds the connection in TIME_WAIT briefly afterward, to absorb any
  stray/delayed packets from the old connection before the port is reused.
```

The interview-relevant question isn't "which is better" — it's *why would anyone choose unreliable delivery on purpose*. For latency-sensitive real-time data (a video frame, a voice packet), a stale retransmitted packet is worse than a dropped one — by the time TCP recovers it, the moment it was needed for has passed. DNS uses UDP because a query/response is small enough to fit in a single datagram and a handshake would be pure overhead for that (it falls back to TCP when a response is too large for one UDP datagram, e.g., zone transfers or a truncated response with the `TC` flag set). QUIC (HTTP/3's transport) deliberately builds its *own* reliability layer in userspace on top of UDP instead of using TCP — see section 2 for why.

---

## 2. HTTP/1.1 vs HTTP/2 vs HTTP/3

| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Message format | Text | Binary framing | Binary framing (over QUIC) |
| Transport | TCP | TCP | QUIC (over UDP) |
| Multiplexing | No — one request in flight per connection (workaround: ~6 parallel TCP connections per origin) | Yes — many streams over one TCP connection | Yes — many streams, each with independent loss recovery |
| Head-of-line blocking | Yes, per connection | App-layer HOL blocking fixed; **TCP-layer HOL blocking remains** (one lost packet stalls every stream) | Fixed — a lost packet only blocks its own stream |
| Header compression | None | HPACK | QPACK |
| Handshake cost (new connection) | TCP (1 RTT) + TLS (1-2 RTT) | Same as 1.1 | Transport + crypto handshakes combined (1-RTT, or 0-RTT on resumption) |
| Connection migration (e.g., WiFi → cellular) | No | No | Yes — QUIC connections are identified by a connection ID, not the IP/port tuple |

The single most commonly *misunderstood* fact here: **HTTP/2's multiplexing does not fully solve head-of-line blocking.** It solves it at the *application* layer (you no longer need 6 separate connections to get parallelism), but all those streams still ride on one TCP connection underneath — and TCP guarantees in-order byte delivery. If one packet is lost, TCP must recover that exact segment before *any* stream's data can be handed up to HTTP/2, even streams whose data arrived just fine. This is why HTTP/3 exists: QUIC gives every stream its own independent sequence space for loss detection and retransmission, so a lost packet only blocks the one stream it belonged to.

The cost of that fix: QUIC reimplements reliability and congestion control in userspace instead of reusing the kernel's TCP stack, which is more CPU-expensive per byte, and it runs over UDP — a protocol some middleboxes/corporate firewalls throttle or block, so clients must be able to fall back to HTTP/2 over TCP.

---

## 3. TLS Handshake

TLS runs *after* the TCP handshake completes, and its job is to let client and server agree on a cipher suite, authenticate the server (via its certificate), and derive a shared session key — all before any application data is sent.

```
TLS 1.2 (≈2 extra RTTs before app data):
  Client Hello (versions, cipher suites, client random)        -->
                                        <-- Server Hello (chosen cipher, server random)
                                        <-- Certificate, ServerHelloDone
  Client verifies cert, sends encrypted premaster secret        -->
  Both derive session keys; Client Finished                     -->
                                        <-- Server Finished
  [application data flows]

TLS 1.3 (≈1 extra RTT before app data):
  Client Hello + a guessed key-share                            -->
                 <-- Server Hello + key-share + Certificate + Finished  (one flight)
  Client verifies, derives keys, sends Finished                 -->
  [application data flows]
```

TLS 1.3 shrinks the handshake by removing static RSA key exchange entirely (mandating ephemeral Diffie-Hellman for forward secrecy) and cutting the negotiable cipher suite list down to a short, all-secure set — which lets the client *guess* the key-exchange parameters in its very first flight instead of waiting to learn the server's preference first.

**0-RTT resumption**: if the client has a session ticket from a prior connection (a PSK), it can send encrypted *application data* in its very first flight — zero extra round trips. The catch: that first flight isn't protected against replay. A network attacker who captures it can resend the exact same request, and the server can't distinguish it from a legitimate retry. Only idempotent requests should ever ride in 0-RTT data.

Also worth knowing: **SNI (Server Name Indication)** lets the client state which hostname it's connecting to *during* the handshake, before the certificate is sent — this is what allows one IP address (e.g., one load balancer) to terminate TLS for many different domains/certificates, which is exactly how most CDNs and multi-tenant load balancers work. TLS termination is commonly done at the edge (load balancer/reverse proxy) to centralize certificate management and offload CPU-heavy crypto from backend instances — trading that off against the backend network being "trusted" plaintext (or requiring a second, internal TLS/mTLS leg when it isn't).

---

## 4. DNS

DNS translates human-readable domain names into IP addresses. A cold lookup (nothing cached anywhere) involves four kinds of servers:

- **Recursive resolver** — the one your OS/ISP actually queries; does all the legwork on your behalf.
- **Root nameserver** — knows where to find the TLD servers (`.`).
- **TLD nameserver** — knows where to find the authoritative server for a given TLD (e.g., `.com`).
- **Authoritative nameserver** — holds the actual record and returns the final answer.

```
Browser cache -> OS (stub) resolver cache -> recursive resolver
   -> recursive resolver asks root nameserver            -> "ask .com TLD"
   -> recursive resolver asks .com TLD nameserver         -> "ask example.com's nameserver"
   -> recursive resolver asks example.com's authoritative  -> returns the IP
   -> recursive resolver returns the IP to the browser
   -> browser opens a TCP connection to that IP and sends the HTTP request
```

Each hop is cached (browser, OS, recursive resolver) for a duration set by that record's **TTL**, so most real-world lookups skip most of this chain. Queries are either *recursive* (client asks the resolver to fully resolve it, or return an error) or *iterative* (each server returns its best partial answer/referral, and the asker walks the chain itself) — the recursive resolver issues iterative queries to root/TLD/authoritative servers on the client's behalf.

The interview-relevant trade-off is **TTL sizing**: a high TTL caches efficiently and reduces load on authoritative/recursive infrastructure, but a changed record (e.g., failing over to a new IP during an incident) can take as long as the old TTL to be seen everywhere. A low TTL propagates changes fast but multiplies lookup volume and the blast radius of a resolver outage. This also ties directly into load balancing: **round-robin DNS** (handing out a rotating list of IPs for one domain) is a crude, health-check-blind form of load balancing — it will keep handing out a dead server's IP until the record changes and every cache's TTL expires.

---

## 5. Load Balancers

A load balancer distributes incoming traffic across a pool of servers to avoid overloading any one of them and to fail over automatically when one becomes unhealthy.

**Static vs dynamic algorithms** (does the balancer look at real-time server state?):

| | Static | Dynamic |
|---|---|---|
| Awareness of server load | None — follows a fixed plan | Yes — reacts to current state |
| Examples | Round robin, weighted round robin, IP hash | Least connections, weighted least connections, resource-based (CPU/memory via an agent) |
| Setup cost | Simple | More complex to configure correctly |
| Risk | Can still overload one server if request costs are uneven | Needs reliable, low-latency health/load signals to be accurate |

**L4 vs L7** (what layer does the balancer actually understand?):

| | L4 (transport) | L7 (application) |
|---|---|---|
| Visibility | IP + port only | Full HTTP — headers, cookies, URL path, method |
| Routing capability | Can't route by content | Content-based routing (e.g., `/api/*` → service A, `/static/*` → service B) |
| Can terminate TLS / inspect requests | No | Yes |
| Performance | Faster, less CPU per packet | Slower — has to parse the application protocol |
| Example | AWS NLB | AWS ALB, NGINX, Envoy |

**Consistent hashing** solves a specific problem: a naive `hash(key) % N` routing scheme remaps *almost every* key the moment `N` (the number of servers) changes — adding or removing one node reshuffles nearly the whole mapping, which is disastrous for a sharded cache (massive cache-miss storm) or a distributed data store. Consistent hashing places both servers and keys on a hash ring; adding or removing one node only remaps the keys that land in that node's small slice of the ring — roughly `K/N` keys, not all of them. Virtual nodes (each physical server mapped to many points on the ring) are used in practice to keep the load distribution even.

Dynamic load balancers also run continuous **health checks**, routing around (and eventually failing over from) any server that stops responding correctly — this is what separates a real load balancer from "just round-robin DNS."

---

## 6. Reverse Proxies (and Forward Proxies, and API Gateways)

| | Forward proxy | Reverse proxy | API Gateway |
|---|---|---|---|
| Sits in front of | Clients | Servers (the origin) | Servers, but logically part of the API surface |
| Hides | The client's identity from the server | The server's identity from the client | Backend topology from the client |
| Typical purpose | Content filtering, bypassing restrictions, client anonymity | Load balancing, TLS termination, caching, DDoS protection | Auth, rate limiting, routing to many microservices, request/response transformation |

A **forward proxy** sits in front of a group of clients; from the server's point of view, every request appears to come from the proxy, not the real client (used for egress filtering, anonymity, or bypassing network restrictions). A **reverse proxy** sits in front of one or more origin servers; from the client's point of view, it *is* the server — the client never talks to the origin directly. This is what gives a reverse proxy its main benefits: it can load-balance across multiple origins, terminate TLS once at the edge instead of on every backend instance, cache responses closer to users, and absorb DDoS traffic before it ever reaches the origin (since the origin's real IP is never exposed).

An **API gateway** is best understood as a reverse proxy with extra application-aware logic bolted on: it's where cross-cutting concerns like authentication, rate limiting (see the API Design topic), request/response transformation, and routing to the correct one of many backend microservices typically live — one chokepoint instead of duplicating that logic in every service.

Common confusion worth clearing up: load balancing is a *feature* a reverse proxy can provide, not a synonym for what a reverse proxy fundamentally is — plenty of reverse proxies (e.g., a pure caching/SSL-termination proxy in front of a single origin) don't load balance anything at all.

---

## 7. WebSockets vs SSE vs Polling

Four ways to get server-originated data to a client, in increasing order of sophistication:

| | Short polling | Long polling | SSE | WebSocket |
|---|---|---|---|---|
| Direction | Client-initiated, repeated | Client-initiated, held open | Server → client only | Full-duplex |
| Latency | High (poll interval) | Low (server responds as soon as data exists) | Low | Lowest |
| Transport | Plain HTTP, new request each time | Plain HTTP, one held-open request | Plain HTTP (`text/event-stream`) | Upgraded HTTP connection, own framing |
| Auto-reconnect | N/A (client just polls again) | Must re-poll manually | Yes — built into `EventSource`, with `Last-Event-ID` for resuming | No — must implement manually |
| Binary data | Yes | Yes | No (text only; base64 for binary) | Yes, native |
| Proxy/firewall friendliness | High (plain HTTP) | High (plain HTTP) | High (plain HTTP) | Lower — needs `Upgrade` header support, which some corporate proxies strip |
| Typical use | Legacy/simple cases | Legacy systems that can't upgrade | Live feeds, notifications, dashboards | Chat, gaming, collaborative editing, low-latency bidirectional data |

Long polling exists because plain HTTP had no push mechanism: the client sends a request, the server holds it open until it has data (or hits a timeout), responds, and the client immediately re-polls. It looks like push but is really a disguised loop — and every poll pays full HTTP header/TLS overhead again. **SSE** fixes this for the common case where the server only ever needs to push, not receive: one long-lived HTTP connection, text events, and automatic reconnection with resumption built into the browser's `EventSource` API for free. **WebSockets** are the only option when the client must also push data back with low latency (not just the initial subscribe) — at the cost of having to build reconnection and liveness-checking (heartbeats) yourself.

---

## Follow-up Drilling

These are the layered, production-judgment questions an interviewer chains on top of the base answer. Try answering each before reading the next; they get progressively harder.

### Chain A — HTTP/2 under packet loss
1. **Baseline**: A client is downloading 6 resources multiplexed over a single HTTP/2 connection on a lossy WiFi network. One TCP packet carrying part of resource #3 is dropped. What happens to the other 5 in-flight resources, and why?
2. **Mechanism**: Why did the *old* HTTP/1.1 model (6 parallel TCP connections) actually suffer less from this specific failure, despite having its own head-of-line blocking problem?
3. **Fix layer**: HTTP/3 is designed specifically to fix this. What did it have to change at the transport layer to do so, and why couldn't that fix be retrofitted into HTTP/2-over-TCP?
4. **Cost**: QUIC reimplements reliability and congestion control in userspace over UDP instead of reusing the kernel's TCP stack. What's the real-world operational cost of that choice?

<details>
<summary>Expected reasoning (check your answer against this)</summary>

1. TCP guarantees in-order, reliable byte-stream delivery on the one underlying connection. A lost packet stalls the *entire* connection until the kernel recovers that exact segment — every other stream multiplexed on top is blocked waiting, even streams whose data arrived perfectly fine. This is TCP-layer head-of-line blocking, and HTTP/2's multiplexing (which only fixed *application*-layer HOL blocking) does nothing to prevent it.
2. Each of the 6 connections has fully independent TCP state. Losing a packet on the connection carrying resource #3 only stalls that one connection; the other 5 resources, on their own separate connections, keep flowing uninterrupted. The "worse" app-layer design (no multiplexing) accidentally bought better loss isolation.
3. HTTP/3 runs over QUIC, which gives every stream its own independent loss-detection and retransmission sequence space — a lost packet only blocks the one stream whose data it carried. This requires owning the whole reliability/retransmission layer yourself instead of relying on TCP's single shared byte-stream guarantee, which is exactly why it couldn't just be bolted onto HTTP/2-over-TCP.
4. Two costs: performance (reliability/congestion-control logic that used to be free, kernel-optimized work must now run in userspace, which is more CPU-expensive per byte) and deployability (QUIC runs over UDP, and some middleboxes/corporate firewalls throttle or block UDP, so clients/servers must support falling back to HTTP/2-over-TCP when QUIC is blocked).
</details>

### Chain B — TLS 1.3 round trips and 0-RTT risk
1. **Baseline**: A mobile client opens a brand-new HTTPS connection to your API with no prior session. Count every round trip that must complete before your API's response can be sent, under TLS 1.2 vs TLS 1.3.
2. **Mechanism**: What did TLS 1.3 actually remove to shrink the handshake down to 1-RTT?
3. **Optimization & risk**: The client has connected before and holds a session ticket. How can you skip a round trip entirely with 0-RTT, and what concrete security risk does this introduce that TLS 1.2 never had?
4. **Boundary**: Given that risk, what category of request should never be allowed to ride inside a 0-RTT payload?

<details>
<summary>Expected reasoning</summary>

1. TLS 1.2: 1 RTT for the TCP handshake + 2 RTT for the full TLS handshake (Client Hello → Server Hello/Certificate/ServerHelloDone → client sends premaster secret + both send Finished) = 3 RTT before application data. TLS 1.3: 1 RTT for TCP + 1 RTT for TLS (client guesses the key-share and sends it immediately; server replies with its own key share + certificate + Finished in a single flight) = 2 RTT total — one full round trip saved.
2. It removed static RSA key exchange entirely (mandating ephemeral Diffie-Hellman for forward secrecy on every connection) and cut the negotiable cipher suite list down to a short, all-secure set — so the client can guess the key-exchange parameters in its very first flight instead of waiting to learn the server's preference first.
3. With a resumed session (a PSK from a previous session ticket), the client can send encrypted *application data* in its very first flight, before the server has responded at all — "0-RTT." The risk: that first flight isn't protected against replay. A network attacker who captures it can resend the exact same encrypted request, and the server has no way to distinguish it from a legitimate retry.
4. Only idempotent, side-effect-free requests belong in 0-RTT data — never a payment, an order placement, or any other non-idempotent mutation, since a replayed packet would silently duplicate the effect.
</details>

### Chain C — Real-time transport at scale
1. **Baseline**: Product wants live order-status updates pushed to a web dashboard. The client never needs to send anything back after the initial subscribe. What transport do you reach for, and why not WebSockets by default?
2. **Scale**: The feature takes off — 50,000 dashboards now hold a connection open, mostly idle. What infrastructure component is most likely to silently kill those connections, and how do you prevent it?
3. **Failure mode**: Your API server restarts for a routine deploy. All 50,000 clients' `EventSource` objects fire a reconnect in the same second. What happens next if you've done nothing special, and what's the fix?
4. **Escalation**: Product now also wants the same dashboard to let a user click "cancel order," reflected in real time from the same view. Does your original transport choice still hold up?

<details>
<summary>Expected reasoning</summary>

1. Server-Sent Events — it's unidirectional (server-to-client), which matches the requirement exactly, runs over plain HTTP/1.1 (no upgrade handshake, friendlier to existing proxies/load balancers), and gets automatic reconnection with `Last-Event-ID` tracking for free from the `EventSource` API. WebSockets would add bidirectional complexity (manual reconnect logic, heartbeats, framing) for a capability not actually needed yet.
2. Load balancers, reverse proxies, and CDNs all enforce an idle-connection timeout; a connection that sends no bytes for longer than that window gets silently dropped. Fix: send periodic keepalive data — an SSE comment line, or a WebSocket ping/pong frame — at an interval shorter than the proxy's idle timeout.
3. All 50,000 clients reconnect at once — a self-inflicted thundering herd that can look exactly like a DDoS against your own servers, right as they're coming back up from a deploy. Fix: client-side exponential backoff with random jitter on reconnect (SSE's `retry` field supports this natively), plus a server-side connection-admission limit so excess reconnects are rejected gracefully instead of taking the service down again.
4. No — SSE is one-directional by design. Adding a client-to-server "cancel order" action needs either a separate channel (a plain REST call for the cancel, keeping SSE only for the read side) or switching the whole feature to WebSockets for true bidirectional push — and switching means now owning reconnection and heartbeat logic yourself instead of getting it for free.
</details>

---

## Gotchas & Edge Cases

- TCP `TIME_WAIT`/ephemeral port exhaustion under very high connection churn (e.g., a service opening a new short-lived connection per request instead of pooling/keep-alive) — a real production capacity cliff, not just trivia.
- Assuming HTTP/2 multiplexing fully solves head-of-line blocking — it only fixes it at the application layer; a single dropped TCP packet still stalls every stream on that connection, which is exactly what HTTP/3 exists to fix.
- Long-polling endpoints silently exhausting a thread-per-request server's thread pool, starving unrelated requests (e.g., payments) sharing that same pool — needs async I/O or a dedicated event loop, not a bigger thread pool.
- A load balancer's or proxy's idle timeout killing WebSocket/SSE connections that go quiet — a mismatch between your app's heartbeat interval and the infra's idle timeout is a classic "why do connections silently die after exactly 60 seconds" bug.
- Forgetting WebSocket heartbeats (ping/pong) — the server can be left thinking a connection is alive long after the client vanished, leaking file descriptors and memory indefinitely.
- Reconnection storms: a server restart or deploy causing every client to reconnect in the same instant — needs exponential backoff plus jitter, not a naive immediate retry.
- DNS TTL trade-off: a high TTL caches efficiently but slows failover/propagation during an incident (a changed IP can take as long as the old TTL to fully propagate); a low TTL propagates fast but multiplies lookup load/latency.
- Treating round-robin DNS as a real load balancer — it has no health-check awareness, so it keeps handing out a dead server's IP until the record changes and every cache's TTL expires.
- 0-RTT TLS 1.3 resumption is replayable — sending a non-idempotent request (like a payment) inside 0-RTT data risks an attacker-replayed packet silently executing it twice.
- Confusing "reverse proxy" with "load balancer" — load balancing is one feature a reverse proxy *can* provide, not what a reverse proxy fundamentally is.

---

## Interview Questions

**Conceptual**
- What's the practical difference between application-layer and transport-layer head-of-line blocking?
- Why does DNS mostly use UDP, and when does it fall back to TCP?
- What's the difference between a forward proxy and a reverse proxy?
- Why is TLS 1.3's handshake faster than TLS 1.2's, and what did it have to give up to get there?
- L4 vs L7 load balancing — what can an L7 balancer do that an L4 balancer fundamentally cannot?

**Debugging / scenario**
- Users report your WebSocket-based chat silently drops connections after almost exactly 60 seconds of inactivity. What do you check first?
- A service is timing out intermittently under moderate load, and a connection dump shows thousands of sockets stuck in `TIME_WAIT`. What's your hypothesis?
- After updating a DNS record to point your API at a new IP, some users still hit the old server an hour later. Why, and what would you do differently next time?

**Trade-off**
- You need to choose between SSE and WebSockets for a live-notifications feature. What tips the decision one way or the other?
- Where would you terminate TLS — at the edge load balancer, or all the way at each backend instance (mTLS)? What are you trading off?
- Consistent hashing vs a simple `hash(key) % N` for routing requests to a pool of cache servers — when is the simpler approach actually fine?

---

## Review

- TCP vs UDP is a reliability/speed trade-off, not a "better/worse" choice — know *why* a protocol would deliberately give up reliability (latency-sensitive data, small single-packet exchanges).
- The HTTP/1.1 → HTTP/2 → HTTP/3 progression is really a progression of fixing head-of-line blocking at one layer at a time: app-layer (HTTP/2's multiplexing), then transport-layer (HTTP/3/QUIC's per-stream loss recovery).
- TLS 1.3 is faster than TLS 1.2 by removing insecure/legacy options, not just by "optimizing" — and its 0-RTT optimization trades a round trip for a real replay-attack surface that only idempotent requests should be exposed to.
- DNS resolution is a caching chain (browser → OS → recursive resolver → root → TLD → authoritative), and TTL sizing is a genuine incident-response trade-off, not just a config detail.
- Load balancing vocabulary comes in two overlapping framings — static/dynamic (does it see live server state?) and L4/L7 (does it see application content?) — plus consistent hashing for the specific problem of minimizing remap churn when the server pool changes.
- Reverse proxy, forward proxy, and API gateway are often used loosely — anchor each on *who it sits in front of* and *whose identity it's hiding*.
- WebSocket vs SSE vs polling is a direction-of-data question first (does the client need to push anything back?) and a production-readiness question second (heartbeats, idle timeouts, reconnection storms).
- Unlike some other topics in this repo, Networking & Protocols has no dedicated `HLD-learning` vocabulary section or `LLD-learning` coding exercise mapped to it — it stays pure theory-plus-follow-ups across the whole prep pipeline.
- Next up per the suggested study order: **Authentication, Authorization & Security**.

---

## Rapid-fire Q&A

1. **Is TCP connection-oriented or connectionless?**<br>
   **A:** Connection-oriented — a 3-way handshake happens before any data flows.
2. **Why does DNS prefer UDP?**<br>
   **A:** A query/response is small enough for one packet and a handshake would be pure overhead; it falls back to TCP when a response is too large for one datagram.
3. **What problem does HTTP/2 multiplexing solve?**<br>
   **A:** Removes the need for ~6 parallel TCP connections per origin by letting many streams share one connection (fixes app-layer head-of-line blocking).
4. **What head-of-line blocking does HTTP/2 NOT solve?**<br>
   **A:** TCP-layer HOL blocking — one lost packet still stalls every multiplexed stream on that connection.
5. **What transport does HTTP/3 use, and why?**<br>
   **A:** QUIC over UDP — gives each stream independent loss recovery, fixing TCP-layer HOL blocking.
6. **How many extra round trips does a fresh TLS 1.3 handshake need before app data flows?**<br>
   **A:** 1 RTT (vs 2 RTT for TLS 1.2).
7. **What's the risk of TLS 1.3 0-RTT data?**<br>
   **A:** It's replayable — only idempotent requests should ever be sent inside it.
8. **What are the four DNS server types in a cold lookup?**<br>
   **A:** Recursive resolver, root nameserver, TLD nameserver, authoritative nameserver.
9. **What's the trade-off of a low DNS TTL?**<br>
   **A:** Faster failover/propagation, at the cost of more frequent lookups and load.
10. **L4 vs L7 load balancer — which can route based on a URL path or cookie?**<br>
    **A:** L7 — it understands HTTP; L4 only sees IP/port.
11. **Why does consistent hashing beat `hash(key) % N` for a cache cluster?**<br>
    **A:** Adding/removing a node remaps only ~K/N keys instead of nearly all of them.
12. **Forward proxy vs reverse proxy — which hides the client's identity from the server?**<br>
    **A:** The forward proxy.
13. **What makes SSE simpler than WebSockets for a notification feed?**<br>
    **A:** Plain HTTP (no upgrade handshake) and automatic reconnect with `Last-Event-ID`, built into `EventSource`.
14. **Why do long-polling servers risk thread-pool exhaustion?**<br>
    **A:** Each held-open request ties up a thread (in thread-per-request servers) for the whole poll duration.
15. **What must always pair with a persistent WebSocket/SSE connection in production?**<br>
    **A:** A heartbeat/keepalive sent more often than the load balancer's idle timeout.
