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

---

## 6. Where does the API gateway fit, and where does domain logic live?

A common point of confusion: the gateway is **not** a domain service. It has no idea what a "student," "grade," or "enrollment" is — it's pure traffic plumbing:

1. **Single entry point** — web and mobile both call the *same* gateway URL instead of knowing the address of 7 different services.
2. **Auth token validation** — checks "is this JWT valid," but does NOT decide what the user is allowed to do — that's RBAC's job, called right after.
3. **Routing** — `/api/students/*` → student-portal-core, `/api/files/*` → file-storage, etc.
4. **Cross-cutting concerns** — rate limiting, CORS, request logging, per-client response shaping.

This is exactly why one shared auth + RBAC setup works cleanly across web and mobile: the identity layer doesn't care which client made the call.

**The missing piece in a service list like auth/rbac/sms/email/file-storage/audit/cron is the domain layer.** All seven of those are platform/infrastructure services — generic, reusable across any product, with zero knowledge of "students" or "courses." You still need one or more **domain/core services** that own your actual business logic (enrollment, grades, attendance, course catalog). That service is the thing that *calls* auth, rbac, email, sms, file-storage, and audit — it's not parallel to them, it sits in the middle and orchestrates them.

```
Platform services (generic, reusable):  auth, rbac, sms, email, file-storage, audit, cron
Domain service (your actual product):   student-portal-core  →  calls the platform services above
```

## 7. Example: student portal request flow (web + mobile, shared identity)

Scenario: a teacher posts a grade through the web app; a student checks their attendance through the mobile app. Both go through the same identity and permission layer because they're the same user base on different clients.

```
┌──────────────┐                              ┌──────────────┐
│   Web app    │                              │  Mobile app  │
└──────┬───────┘                              └──────┬───────┘
       │                                              │
       └───────────────────┬──────────────────────────┘
                            ▼
                  ┌───────────────────┐
                  │    API gateway     │   ← single entry point,
                  │  routing + rate    │     same for both clients
                  │  limit + CORS      │
                  └─────────┬──────────┘
                            ▼
                  ┌───────────────────┐
                  │   Auth service     │   ← "is this token valid?"
                  │  validates JWT,    │     shared identity model,
                  │  shared identity   │     same for web + mobile
                  └─────────┬──────────┘
                            ▼
                  ┌───────────────────┐
                  │   RBAC service     │   ← "is this user allowed
                  │  checks role +     │     to do THIS action?"
                  │  permission        │
                  └─────────┬──────────┘
                            ▼
                  ┌────────────────────────┐
                  │  student-portal-core    │   ← YOUR domain logic:
                  │  enrollment, grades,    │     enrollment, grades,
                  │  attendance, courses    │     attendance, courses
                  └───┬──────┬──────┬───┬───┘
            (queue)   │      │      │   │   (queue)
              ┌───────┘      │      │   └───────┐
              ▼              ▼      ▼           ▼
        ┌──────────┐  ┌──────────┐ ┌──────────┐ ┌──────────┐
        │  Email   │  │   SMS    │ │  File    │ │  Audit   │
        │ grade    │  │ absence  │ │ storage  │ │  log     │
        │ posted   │  │ notice   │ │ uploads  │ │ records  │
        │ alert    │  │          │ │          │ │ action   │
        └──────────┘  └──────────┘ └──────────┘ └──────────┘

   All four above are consumed via RabbitMQ / BullMQ queues,
   not direct synchronous calls — student-portal-core fires an
   event and moves on, it doesn't wait for the email to send.

                  ┌───────────────────┐
                  │   Cron-job service  │   ← runs on a schedule,
                  │  no client request  │     no client involved at all
                  │  needed              │
                  └─────────┬───────────┘
                            │
              fires jobs directly into the
              same queues as student-portal-core
                            │
              ┌─────────────┴─────────────┐
              ▼                           ▼
        nightly attendance         weekly grade
        summary → Email            digest → SMS
```

**Why this shape works for your case:**

- **Web and mobile converge at the gateway**, not before it. Neither client talks to auth/rbac/student-portal-core directly — they only know the gateway's address. This is what makes "same users, different clients" simple: there's exactly one place identity gets checked, regardless of which app made the call.
- **Auth and RBAC are two separate, sequential checks**, not one combined step. Auth answers "who is this." RBAC answers "what can they do." Keeping them separate is what lets you reuse the exact same identity model across a student portal, a teacher portal, and a parent portal, while still giving each role different permissions.
- **student-portal-core is the only service that knows what a "grade" or "course" is.** Auth, rbac, email, sms, file-storage, audit, and cron are all completely reusable — you could lift every one of them into an entirely different product (an e-commerce app, a clinic system) and they wouldn't need to change. Only student-portal-core is specific to "student portal."
- **The fan-out to email/sms/file-storage/audit is async on purpose.** When a teacher posts a grade, the request shouldn't hang while an email goes out — student-portal-core publishes an event (RabbitMQ) or enqueues a job (BullMQ) and immediately returns success to the gateway. The notification arriving 2 seconds later is fine; the teacher's UI shouldn't wait for it.
- **Cron-job is the only service in this diagram with no client in its path at all.** It wakes up on a schedule, decides "time to send the weekly digest," and pushes straight into the same queues student-portal-core uses — from the email/sms services' point of view, a cron-triggered job and a student-portal-core-triggered job look identical.

**A second domain service is normal, not a smell.** As the student portal grows, you'll likely split `student-portal-core` further — e.g. `enrollment-service`, `grading-service`, `attendance-service` — once each grows complex enough to deserve its own database schema and team ownership. The platform services (auth/rbac/email/sms/file-storage/audit/cron) stay exactly as they are; only the domain layer fans out.
