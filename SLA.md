# Asset Tracking, SLA Monitoring & Automated Billing Platform

### A Backend Engineering Project Blueprint (Java 21 · Spring Boot · PostgreSQL · Redis · Docker)

---

## 1. Project Overview

**Problem statement:** Telecom/service-based organizations manage physical and logical assets (routers, SIM cards, field equipment, service tickets) across multiple client accounts. Each client has contractual Service Level Agreements (SLAs) — e.g., "resolve P1 incidents within 4 hours" — and needs periodic, accurate billing reports tied to asset usage and SLA compliance.

Manually tracking assets, calculating SLA breaches, and generating billing reports is slow and error-prone. This platform automates all three.

**Goal:** Build a multi-tenant backend system that:
1. Tracks the lifecycle and status of assets across clients
2. Continuously monitors SLA compliance and flags breaches in near real-time
3. Automatically generates periodic billing reports tied to asset usage and SLA performance

---

## 2. Requirement Gathering

### 2.1 Stakeholders
- **Client Operations teams** — need visibility into asset status and SLA compliance
- **Billing/Finance team** — needs accurate, auditable billing reports
- **Internal Support/Ops engineers** — need to update asset status and resolve SLA breaches
- **System Admins** — manage client onboarding, SLA rule configuration, access control

### 2.2 Functional Requirements (FRs)

| ID | Requirement |
|----|-------------|
| FR1 | System shall allow CRUD operations on Assets (create, update status, retire) |
| FR2 | System shall maintain a full audit trail (history) of asset status changes |
| FR3 | System shall allow defining SLA rules per client (response time, resolution time thresholds) |
| FR4 | System shall automatically detect and flag SLA breaches based on ticket/asset event timestamps |
| FR5 | System shall notify relevant stakeholders when an SLA breach occurs or is about to occur |
| FR6 | System shall generate periodic (daily/weekly/monthly) billing reports per client |
| FR7 | System shall expose REST APIs for all above operations, secured via authentication |
| FR8 | System shall support role-based access — Admin, Ops Engineer, Client Viewer |
| FR9 | System shall support 5+ concurrent client tenants with isolated SLA configs |

### 2.3 Non-Functional Requirements (NFRs)

| ID | Requirement | Target |
|----|-------------|--------|
| NFR1 | API response time under load | p95 < 300ms |
| NFR2 | System must handle concurrent read-heavy asset status queries | 10K+ DAU |
| NFR3 | SLA breach detection latency | < 1 minute from breach condition |
| NFR4 | Data durability for billing records | No data loss (ACID compliance) |
| NFR5 | Horizontal scalability | Stateless services, container-deployable |
| NFR6 | Security | JWT auth, RBAC, encrypted secrets |

### 2.4 Assumptions & Constraints
- Assets belong to exactly one client at a time
- SLA rules are defined per client, per asset category (e.g., "Critical Equipment" vs "Standard")
- Billing periods are configurable (default monthly)

---

## 3. Requirement Analysis

### 3.1 Core Entities (Domain Model)

- **Client** — a tenant/customer organization
- **Asset** — a trackable unit (device, equipment, ticket) belonging to a Client
- **AssetStatusHistory** — append-only log of every status change (for audit + SLA calculation)
- **SLARule** — defines thresholds (e.g., response_time_minutes, resolution_time_minutes) per client + asset category
- **SLABreachEvent** — recorded when an asset's time-in-status exceeds its SLA threshold
- **BillingReport** — generated document/record summarizing asset usage + SLA compliance for a period
- **User** — internal staff or client-side viewer with a Role

### 3.2 User Stories (sample)
- *As an Ops Engineer*, I want to update an asset's status so that the system tracks its current state.
- *As a Client Admin*, I want to view SLA compliance percentage for my account so I can verify contract adherence.
- *As Finance*, I want a monthly PDF/CSV billing report auto-generated so I don't have to compile it manually.
- *As a System Admin*, I want to configure SLA thresholds per client without redeploying code.

### 3.3 Why This Domain Needs Careful Design
- **Multi-tenancy**: SLA rules and billing must never leak across clients → data isolation matters
- **High read volume**: Asset status is queried far more than it's written → caching candidate
- **Time-sensitive correctness**: SLA breach detection depends on accurate timestamp handling and needs to run continuously, not just on-demand
- **Auditability**: Billing is financial — every number must be traceable to source events

---

## 4. High-Level Architecture

```
                          ┌─────────────────────┐
                          │   API Gateway /     │
                          │   Spring Cloud      │
                          │   Gateway           │
                          └──────────┬──────────┘
                                     │
        ┌────────────────────────────┼───────────────────────────┐
        │                            │                           │
┌───────▼────────┐         ┌─────────▼───────┐        ┌──────────▼──────┐
│ Asset Tracking │         │  SLA Monitoring │        │  Billing Report |
│ Service        │◄───────►│  Service        │◄──────►│  Service        │
│ (Spring Boot)  │  events │ (Spring Boot)   │  data  │ (Spring Boot)   │
└───────┬────────┘         └─────────┬───────┘        └──────────┬──────┘
        │                            │                           │
        │        ┌───────────────────┴────────────┐              │
        │        │                                │              │
┌───────▼────────▼───┐                  ┌─────────▼────────┐     │
│   Redis Cache      |                  │ Scheduler        │     │
│  (status cache,    │                  │ (Spring Schedule/│     │
│   rate limiting)   │                  │  Quartz)         │     │
└────────────────────┘                  └──────────────────┘     |
        │                                                        │
        └──────────────────────┬─────────────────────────────────┘
                               │
                     ┌─────────▼────────────┐
                     │     PostgreSQL       │
                     │  (Assets, SLARules,  │
                     │   Breaches, Billing) │
                     └──────────────────────┘

All services containerized via Docker, secured via Spring Security (JWT + RBAC)
```

**Why microservices split this way?**
- Asset Tracking = high write + read frequency, needs to be fast and independently scalable
- SLA Monitoring = compute-heavy (scheduled jobs comparing timestamps against rules), benefits from independent scaling of worker instances
- Billing = low frequency, batch-oriented, can scale down most of the time

This is a legitimate, defensible reason to use microservices here — **not** microservices for their own sake, which is a common interview trap ("why not just build a monolith?"). Be ready to say: *"Given the different load patterns and scaling needs of these three modules, splitting them let us scale SLA monitoring workers independently of the read-heavy asset APIs."*

---

## 5. Database Design (PostgreSQL)

### 5.1 Core Tables

```sql
-- Clients (tenants)
CREATE TABLE clients (
    client_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    billing_cycle   VARCHAR(20) DEFAULT 'MONTHLY',
    created_at      TIMESTAMP DEFAULT now()
);

-- Assets
CREATE TABLE assets (
    asset_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID REFERENCES clients(client_id),
    category        VARCHAR(50) NOT NULL,     -- e.g. CRITICAL, STANDARD
    current_status  VARCHAR(30) NOT NULL,     -- e.g. OPEN, IN_PROGRESS, RESOLVED
    created_at      TIMESTAMP DEFAULT now(),
    updated_at      TIMESTAMP DEFAULT now()
);
CREATE INDEX idx_assets_client_status ON assets(client_id, current_status);

-- Append-only status history (source of truth for SLA calculation)
CREATE TABLE asset_status_history (
    id              BIGSERIAL PRIMARY KEY,
    asset_id        UUID REFERENCES assets(asset_id),
    status          VARCHAR(30) NOT NULL,
    changed_at      TIMESTAMP NOT NULL DEFAULT now()
);
CREATE INDEX idx_history_asset_time ON asset_status_history(asset_id, changed_at);

-- SLA Rules (configurable per client + category)
CREATE TABLE sla_rules (
    sla_rule_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id              UUID REFERENCES clients(client_id),
    asset_category         VARCHAR(50) NOT NULL,
    response_time_minutes  INT NOT NULL,
    resolution_time_minutes INT NOT NULL
);

-- SLA Breach Events
CREATE TABLE sla_breach_events (
    breach_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID REFERENCES assets(asset_id),
    sla_rule_id     UUID REFERENCES sla_rules(sla_rule_id),
    breach_type     VARCHAR(20) NOT NULL,  -- RESPONSE, RESOLUTION
    detected_at     TIMESTAMP DEFAULT now(),
    resolved        BOOLEAN DEFAULT false
);

-- Billing Reports
CREATE TABLE billing_reports (
    report_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID REFERENCES clients(client_id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    total_assets    INT,
    sla_compliance_pct NUMERIC(5,2),
    total_amount    NUMERIC(12,2),
    generated_at    TIMESTAMP DEFAULT now(),
    file_path       VARCHAR(500)
);
```

**Design rationale to know cold:**
- `asset_status_history` is **append-only** — this is the audit trail and the single source of truth for SLA time calculations. Never UPDATE it, only INSERT.
- Composite index on `(client_id, current_status)` supports the most common query pattern: "show me all OPEN assets for Client X"
- `sla_rules` decoupled from `assets` so thresholds can change without touching asset data — supports FR3 (configurable per client)

---

## 6. Design Patterns Used

| Pattern | Where Used | Why |
|---------|-----------|-----|
| **Strategy Pattern** | SLA breach calculation | Different asset categories (CRITICAL vs STANDARD) may need different breach-detection logic. Strategy interface `SLABreachCalculator` with implementations per category avoids messy if-else chains. |
| **Repository Pattern** | All data access (via Spring Data JPA) | Abstracts persistence logic from service layer; enables easier testing with mocks |
| **Observer/Event-Driven** | Asset status change → triggers SLA check | When status changes, an `AssetStatusChangedEvent` is published (Spring's `ApplicationEventPublisher`); SLA module listens and reacts asynchronously — decouples the two modules |
| **Builder Pattern** | Constructing `BillingReport` objects with many optional fields | Cleaner than a telescoping constructor; used with Lombok `@Builder` |
| **Template Method** | Report generation (PDF vs CSV export) | Abstract `ReportGenerator` defines the skeleton (fetch data → compute → format → save); subclasses `PdfReportGenerator`/`CsvReportGenerator` override formatting step |
| **Circuit Breaker** (Resilience4j, often bundled with Spring Cloud) | Calls between Asset service and SLA service | Prevents cascading failure if SLA service is slow/down |
| **Cache-Aside Pattern** | Redis usage for asset status reads | Application checks cache first, falls back to DB on miss, populates cache — standard pattern, easy to explain in interviews |

---

## 7. Technology Stack — What Each Piece Does

### Java 21
- **Records** for immutable DTOs (e.g., `AssetStatusDTO`) — reduces boilerplate vs Lombok in some cases
- **Virtual Threads** (Project Loom) — used in the SLA monitoring scheduler to handle many concurrent breach-check tasks without the overhead of platform threads; explain this as: *"Since SLA checks are I/O-bound (DB queries per asset), virtual threads let us run thousands of lightweight checks concurrently without exhausting the OS thread pool."*
- **Pattern matching for switch** — used in status-transition validation logic (e.g., validating that an asset can move from OPEN → IN_PROGRESS but not OPEN → CLOSED directly)

### Spring Boot
- Core framework for all three services (Asset, SLA, Billing) — auto-configuration, embedded Tomcat, starter dependencies
- `@RestController` for exposing APIs, `@Service`/`@Repository` for layering

### Spring Security
- JWT-based stateless authentication — each request carries a signed JWT validated by a filter (`OncePerRequestFilter`)
- RBAC via `@PreAuthorize("hasRole('ADMIN')")` on sensitive endpoints (e.g., SLA rule configuration is Admin-only)
- Rate limiting at the gateway level (paired with Redis, see below) to prevent abuse

### Spring Cloud (optional, if you want the "microservices" story to be airtight)
- **Spring Cloud Gateway** — single entry point routing to the 3 services
- **Eureka / Service Discovery** — services register themselves, gateway routes dynamically
- **Config Server** — centralizes SLA thresholds' default configs and DB credentials outside code

### Spring Data JPA / Hibernate
- Entity mapping for all tables above
- `@Query` / Specification API for dynamic filtering (e.g., filter assets by client + status + date range)
- Pagination (`Pageable`) for asset listing APIs to avoid loading huge result sets

### PostgreSQL
- Primary transactional store — chosen for ACID guarantees (billing data cannot be "eventually consistent")
- Indexes as shown above for the hot query paths
- Partitioning consideration (mention if asked about scale): `asset_status_history` could be partitioned by month once it grows large, since it's append-only and time-ordered

### Redis
- **Cache-aside** for asset current-status lookups (high read frequency, changes infrequently relative to reads)
- **Rate limiting** — token bucket counters per client/API key using Redis `INCR` + `EXPIRE`
- **Distributed lock** (via Redisson or `SETNX`) — prevents duplicate SLA breach events from being written if two scheduler instances process the same asset simultaneously (important once you scale SLA service horizontally)

### Docker
- Each service (Asset, SLA, Billing) has its own `Dockerfile`, built as a separate image
- `docker-compose.yml` for local dev: spins up Postgres, Redis, and all 3 services together
- Enables consistent environments between local dev and production, and is the natural stepping stone toward Kubernetes if asked "how would you deploy this at scale?"

---

## 8. Module Deep-Dive

### 8.1 Asset Tracking Module
**What it does:** CRUD + status transitions for assets, with full history logging.

**Key flow:**
1. Ops engineer calls `PATCH /assets/{id}/status` with new status
2. Service validates transition is legal (state machine check)
3. Writes new row to `asset_status_history`, updates `assets.current_status`
4. Publishes `AssetStatusChangedEvent`
5. Invalidates/updates the Redis cache entry for that asset

**Interview-ready explanation:** *"Asset status changes are the source events for everything downstream — SLA monitoring listens to these to know when a clock starts or stops."*

### 8.2 SLA Monitoring Module
**What it does:** Continuously (via scheduled job, e.g., every 1 minute) scans open assets and checks elapsed time against `sla_rules` thresholds.

**Key flow:**
1. Scheduled job (`@Scheduled` + virtual threads) queries all assets with status ≠ RESOLVED
2. For each, computes elapsed time since last status change (from `asset_status_history`)
3. Compares against the applicable `sla_rules` entry (matched by client + category)
4. If breached and not already recorded → insert into `sla_breach_events`, trigger notification (e.g., publish to a queue or send email)

**Interview-ready explanation:** *"Rather than checking SLA compliance on-demand when someone views a dashboard, we run a background scheduler so breaches are detected within a minute of occurring — that's the NFR3 requirement."*

### 8.3 Billing Report Module
**What it does:** Periodically (monthly, via scheduled job) aggregates asset usage + SLA compliance per client and generates a report file.

**Key flow:**
1. Scheduled job runs on the 1st of each month (or triggered manually via API)
2. Queries total assets, SLA compliance % (breaches / total-tracked-assets), and computes billed amount (business rule, e.g., per-asset rate × count, with penalty deductions for SLA breaches)
3. Uses Template Method pattern to generate PDF/CSV
4. Saves file reference + summary row to `billing_reports`

**Interview-ready explanation:** *"Billing reports tie directly back to SLA breach data, so if a client had 3 breaches in a category with a penalty clause, that's reflected in the final billed amount — the report is auditable back to the individual breach events."*

---

## 9. Sample REST API Design

```
POST   /api/v1/clients                        (Admin)
POST   /api/v1/assets                          (Ops)
PATCH  /api/v1/assets/{assetId}/status          (Ops)
GET    /api/v1/assets?clientId=&status=&page=   (All roles, filtered by RBAC)
GET    /api/v1/assets/{assetId}/history

POST   /api/v1/sla-rules                        (Admin)
GET    /api/v1/sla-rules?clientId=

GET    /api/v1/sla-breaches?clientId=&resolved=

GET    /api/v1/billing-reports?clientId=&period=
POST   /api/v1/billing-reports/generate         (Admin/Finance, manual trigger)
```

All endpoints secured behind JWT; RBAC enforced via method-level `@PreAuthorize`.

---

## 10. How to Talk About This in Interviews

**If asked "walk me through the system":**
Start top-down: problem → 3 modules → why microservices (load pattern difference) → data flow (status change → event → SLA check → billing rollup) → then drop into one module's detail if they want depth.

**If asked "why Redis specifically":**
Two distinct uses — caching (read-heavy asset status) and rate limiting (protecting APIs). Don't just say "for caching," name the cache-aside pattern and what would go wrong without it (DB gets hammered on every dashboard refresh).

**If asked "how do you avoid race conditions in SLA checking at scale":**
Talk about the distributed lock via Redis when running multiple scheduler instances — this is a strong, specific answer that shows you've thought about horizontal scaling.

**If asked "why not just use cron + a script instead of all this":**
Acknowledge it's a fair question for a small scale — but at 5+ clients with different SLA rules, audit requirements, and the need for API-driven configurability (non-engineers changing SLA thresholds without a deploy), a proper service with a data model and APIs pays off.

**Be ready for:** "What would you do differently at 10x scale?" — good answer: partition `asset_status_history` by month, move SLA breach detection to a Kafka-based event stream instead of polling every minute, add read replicas for PostgreSQL for the reporting queries.

---

## 11. Suggested Build Order (if you want to actually implement this)

1. PostgreSQL schema + Spring Boot project skeleton with JPA entities
2. Asset CRUD + status transition API + status history logging
3. Dockerize (Postgres + app via docker-compose) — get this working early
4. Add Redis caching for asset reads
5. SLA rules API + scheduled breach-detection job
6. Spring Security (JWT + RBAC) layered in
7. Billing report generation (start with CSV, add PDF later)
8. (Optional, for the full microservices story) Split into 3 services + Spring Cloud Gateway

Building even a scaled-down version of this (skip the full microservices split, keep it as one well-structured Spring Boot app) would give you enough hands-on depth to answer almost any follow-up question confidently.


## Queries

1. What is the role of scheduler and how it is used?
2. how to ensure only-append schema for asset status history?
3. What is data flow in all the microservices? 
4. what kind of events are produced?
5. what are the different API calls between microservices?
6. can we use Spring AI in this?

## Resume summary 

### Asset Tracking, SLA Monitoring & Automated Billing Platform | Personal/Professional Project

**Tech Stack: Java 21, Spring Boot, Spring Security, Spring Cloud Gateway, Kafka, PostgreSQL, Redis, Docker**

- Architected a multi-tenant microservices platform (Asset Tracking, SLA Monitoring, Billing) supporting 5+ client accounts with isolated SLA configurations and audit-compliant data flow.
- Designed an event-driven architecture using Apache Kafka, publishing AssetStatusChanged, SLABreachDetected, and BillingReportGenerated events to decouple services, enabling independent scaling and full event replay/audit capability
- Built an append-only asset status history schema enforced at the database level (PostgreSQL GRANT/REVOKE + triggers), ensuring tamper-proof audit trails for SLA and billing calculations
- Implemented a dual-mode SLA breach detection system — reactive (Kafka event listeners) and proactive (scheduled sweeps using Java 21 virtual threads) — reducing breach-detection latency to under 1 minute
- Used Redis for cache-aside asset status lookups and distributed locking (via Shedlock/Redis) to prevent duplicate SLA breach processing across horizontally scaled scheduler instances
- Secured all REST and internal service-to-service APIs with Spring Security (JWT authentication, RBAC) and Kafka topic-level ACLs for inter-service communication
- Applied Resilience4j (Circuit Breaker, Retry) on synchronous internal API calls between services to prevent cascading failures during partial outages
- Automated monthly billing report generation by aggregating asset usage and SLA compliance data from Kafka event streams, cutting manual report preparation effort by ~85%
- Containerized all services with Docker and orchestrated local multi-service environments via Docker Compose, mirroring production deployment topology

A few notes on using this well:

This reads as a strong, cohesive project — the Kafka + event-driven + append-only + distributed locking combination signals real systems thinking, not just "I used Spring Boot."
If this goes on your resume as a "Project" (not "Experience"), that's actually safer and more honest than folding it into your TCS bullets, since it didn't happen at TCS. Recruiters and interviewers treat personal/side projects as fair game for deep technical questions too — but there's no ambiguity about where it happened.
Before you use this, I'd strongly repeat what I flagged earlier: build at least the core of this (Asset + SLA modules with Kafka wired up) so you can open your IDE and show real code if asked. A project bullet with Kafka, virtual threads, and distributed locks will draw exactly that kind of deep-dive question at a product company.
