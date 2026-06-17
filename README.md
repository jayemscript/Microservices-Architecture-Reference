# Microservices Architecture Reference

**Stack:** NestJS · TypeORM · PostgreSQL · MongoDB · Redis · BullMQ · RabbitMQ

This doc catalogs your current services, fixes the categorization model, and lists what's commonly used in enterprise systems so you can pick what to build next and learn from real-world patterns.

---

## 1. The categorization framework

Two axes alone ("edge vs support") aren't enough — they collapse two different questions. Use three:

- **Criticality** — does the system break without it?
  - `Core`: system fails or becomes unusable without it
  - `Support`: system degrades gracefully without it (slower, less polished, but alive)
- **Interaction type** — does the caller wait for a response?
  - `Sync`: request/response, blocks the caller (REST/gRPC call)
  - `Async`: fire-and-forget, queue/event-driven (BullMQ job, RabbitMQ event)
  - `Hybrid`: exposes both — a sync API plus an async consumer/worker side
- **Layer** — where does it sit relative to traffic?
  - `Edge`: faces inbound traffic directly, often the first hop, or client-facing
  - `Internal`: only ever called by other services, never directly by clients

Quick gut-check for new services: *"If this is down for 10 minutes, does a user-facing critical path fail, or does something just get delayed/stale?"* That sorts Core vs Support immediately. Sync vs Async is then a per-endpoint decision — most services end up Hybrid in practice.

---

## 2. Your current services, recategorized

| Service | Criticality | Type | Layer | Notes |
|---|---|---|---|---|
| **auth** | Core | Sync (mostly) | Edge | Shared identity across apps; every protected request depends on it |
| **rbac** | Core | Sync | Edge | Called inline on every protected request — cache permission checks in Redis aggressively, this is your hottest path |
| **payment** | Core | Hybrid | Edge | Sync for charge confirmation, async for webhooks/retries/reconciliation |
| **notification** | Support | Async (mostly) | Internal | Can expose a sync "get unread count" read endpoint, but delivery itself is async |
| **email** | Support | Async | Internal | Multi-provider, queue-based sending and retries |
| **sms** | Support | Async | Internal | Same shape as email |
| **file-storage-worker** | Core or Support* | Hybrid | Edge | Sync upload/download, async processing (thumbnails, virus scan). *Core if files are the product (e.g. document platform), Support if incidental (avatars/attachments) |

A correction worth making explicit: don't let RBAC become a second network hop on every request unless you have a real reason (multi-app shared RBAC engine, which you do). If it is a separate hop, cache aggressively — this call taps every single protected request across every app.

---

## 3. Services you're likely missing

### Core / Edge — break the system if they're down

| Service | Type | Why you need it |
|---|---|---|
| **api-gateway / BFF** | Sync, Edge | Single entry point for routing, rate limiting, and auth token validation before requests hit downstream services. Without this, every service reimplements auth checks. Industry pattern popularized by Netflix's Zuul; modern options: Kong, Apache APISIX, Traefik, Envoy. |
| **audit-log** | Async (recording intent is Core) | Compliance + debugging backbone for any system with payments, RBAC, and multi-tenant access. If it silently drops events that's a real incident. |
| **tenant/organization-service** | Sync, Edge/Internal | Since you're multi-app, something needs to own tenant/org boundaries — auth, rbac, billing, and file-storage all need to know which tenant a request belongs to. |
| **service-discovery / health-registry** | Sync, Internal | At your scale, every service should expose a `/health` endpoint; a registry/orchestrator pings it and removes unhealthy instances from rotation. |

### Core, resilience-focused — not separate services, but patterns you apply *inside* the gateway and inter-service calls

These came up repeatedly in how large systems are actually built, worth knowing even before you build more services:

- **Circuit breaker** — stop calling a failing downstream service so it doesn't cascade and take the whole system down (e.g. Resilience4j-style logic, or libraries like `cockatiel`/`opossum` in Node).
- **Saga pattern** — coordinate consistency across services for long-running transactions (e.g. payment succeeds → order confirmed → inventory reserved → notification sent), without a single distributed transaction.
- **CQRS** — separate read and write paths when a service's read load and write load have very different shapes (common once you add audit-log, search, or analytics).
- **Event sourcing** — store state changes as an append-only event log instead of overwriting rows; pairs naturally with your RabbitMQ setup and is worth learning even if you don't adopt it everywhere.

### Support / Internal — async, optional, degrade gracefully

| Service | Type | What it does |
|---|---|---|
| **search-indexing-worker** | Async, Internal | Consumes events off RabbitMQ, indexes into Elasticsearch/Meilisearch/Typesense. Stale search for a few seconds never breaks anything. |
| **analytics/event-tracking** | Async, Internal | Fire-and-forget event ingestion — classic BullMQ consumer. |
| **export-worker** | Async, Internal | CSV/PDF generation jobs, queue-based, polled or notified on completion. |
| **scheduler/cron-orchestrator** | Async, Internal | Centralized dispatcher for scheduled/delayed work when multiple services need cron-like jobs (BullMQ has repeatable jobs, but a dedicated orchestrator avoids scattering cron logic). |
| **webhook-dispatcher** | Async, Edge-ish | Outbound webhooks to *your customers'* systems — different from `notification`, which delivers inbound to end users. Worth splitting out in a multi-app setup since retry/backoff policy differs per receiving app. |
| **feature-flag service** | Sync, Support | Called inline, but if it's down you fall back to defaults — never blocks the system. |
| **media-processing-worker** | Async, Internal | If `file-storage-worker` is just storage I/O, split out transcoding/thumbnailing/virus-scanning — CPU-heavy work shouldn't compete with upload/download throughput. |
| **idempotency/dedup-service** | Infra more than a "service" | A shared Redis-backed idempotency-key store prevents duplicate job processing across retries — important given how heavily you lean on RabbitMQ + BullMQ. |

---

## 4. What enterprise systems actually run (so you can learn from real patterns)

These are the patterns large-scale production systems converge on. You don't need all of them, but each is worth understanding and optionally implementing as a learning exercise.

**API Gateway pattern** — centralizes routing and cross-cutting concerns (auth, rate limiting, request transformation) at the edge. Keep business logic out of it; if you find yourself adding business rules to the gateway, that logic belongs in a dedicated service. Real implementations: Kong, Envoy, Apache APISIX, Traefik, AWS API Gateway, Netflix Zuul (the original).

**Service Mesh** — a different layer from the gateway: API gateways manage **north-south** traffic (client → system), service meshes manage **east-west** traffic (service → service) inside the system — retries, mTLS, load balancing, observability, all without each service implementing it itself. Linkerd is lighter/faster; Istio has a deeper feature set and is more common in large enterprises despite being heavier. API gateways use centralized deployment while service meshes distribute proxies alongside every service instance, and most modern architectures end up using both together rather than choosing one. You likely don't need a full mesh yet — that's a "once you have 15+ services and multiple teams" tool — but it's worth knowing what problem it solves so you recognize when you've outgrown manual retry logic scattered across services.

**Circuit Breaker pattern** — a gateway is a critical dependency and should be treated as a first-class service: run multiple instances behind a load balancer, scale horizontally rather than vertically, support multiple API versions so old clients don't break, and implement circuit breakers and fallbacks so one failing service doesn't break the whole response.

**Health checks + service registry** — each microservice exposes a health check endpoint (e.g. `/health`), and a registry or orchestrator periodically pings it to confirm the instance is alive and ready to handle requests — this is what lets Kubernetes (or any orchestrator) pull a bad instance out of rotation automatically.

**Backends for Frontends (BFF)** — instead of one generic API gateway serving web, mobile, and third-party clients identically, you run a thin BFF layer per client type, each shaped to what that client actually needs. Useful once your web app and mobile app start needing meaningfully different response shapes from the same underlying services.

**Governance at scale** — microservices governance covers service discovery, monitoring, and dependency management, and effective governance leads to streamlined operations and enhanced service quality — worth keeping in mind as you add more services: who owns which service, what's the contract between them, how do you version APIs without breaking consumers.

---

## 5. Suggested build order

If you want to keep learning by building (rather than just bolting on infra), this is a reasonable next sequence given what you already have:

1. **api-gateway** — you already have auth + rbac; centralizing token validation and routing here is the highest-leverage next build, and a great way to learn the gateway pattern hands-on.
2. **audit-log** — straightforward BullMQ consumer, immediately useful once payment + rbac are live, and teaches you event-driven persistence patterns.
3. **tenant/organization-service** — formalizes the "multi app" boundary that auth/rbac/payment/file-storage all currently assume implicitly.
4. **webhook-dispatcher** — splits cleanly from `notification`, good exercise in retry/backoff design (exponential backoff, dead-letter queues in RabbitMQ).
5. **search-indexing-worker** or **export-worker** — either is a good low-stakes way to practice CQRS (separate read model from your Postgres/Mongo write models).
6. **circuit breaker + service mesh exploration** — once you have 5+ services calling each other, introduce `cockatiel` or `opossum` for breaker logic, then experiment with Linkerd as a learning exercise even if you don't run it in production.

This order keeps every step Core-adjacent and immediately useful, while still exposing you to the patterns (gateway, event sourcing, CQRS, circuit breaking, service mesh) that show up in every enterprise architecture discussion.
