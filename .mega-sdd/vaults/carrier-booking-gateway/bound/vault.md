---
type: vault
doc_id: vault
vault_layout: 2
vault_version: "1.0"
project_shape: web-app
implementation_mode: new
<!-- BIND: confirmed=C-MODE-01 -->
mode_migration_trigger: "first commit on main"
prd_status: final
output_mode: compact
prd_source: "docs/superpowers/specs/2026-09-01-carrier-booking-gateway-design.md, v1, 2026-09 — FINAL (approved for implementation planning)"
locked_at: "2026-09-01"
locked_by: ["coresystem-dev"]
aliases: [Vault, Grand Design]
tags: ["vault/carrier-booking-gateway", "doc/vault"]
---

# Carrier Booking Gateway — Grand Design

> One internal system where ZHL operators book ocean freight once, and the gateway talks to every carrier for them.

## Phase context

**Phase:** 1 of 1

**This vault covers:** Single-phase project. v1 of the carrier booking gateway: four REST/DCSA-aligned carriers (ONE, Maersk, HMM, Evergreen), create + status + cancel + tracking, built as `apps/booking` plus shared packages inside the new `zhenghe-monorepo`.

## Overview

> **TL;DR**: An internal booking gateway that fronts four ocean carriers behind one canonical API and a thin UI · for ZHL operations + platform engineers · read before building any part of `apps/booking`.

### Product

The carrier booking gateway is a new internal service. ZHL books ocean freight across seven carriers, each with its own portal and login. This system replaces per-carrier logins with one internal API and a thin UI: an operator creates a booking once, and per-carrier adapters translate it to each carrier's API, submit it, and keep its status and tracking events in sync. v1 covers four carriers that all expose REST APIs broadly aligned with the DCSA Booking 2.0 standard: ONE (SCAC ONEY), Maersk (MAEU), HMM (HDMU), Evergreen (EGLV).

### Target users / personas

- **Operations controller**: books freight, cancels bookings, checks status and tracking. Maps to the `controller` role.
- **Operations manager / viewer**: reads bookings, status and tracking; no write actions. Maps to the `viewer` role.
- **Platform administrator**: configures carriers and sets/rotates carrier credentials. Maps to the `admin` role.

> Roles are named in the spec (§7); no other personas are described.

### Problem

ZHL operators log in to seven separate carrier portals to book freight. There is no single place to create a booking, no consistent status model across carriers, and no unified tracking. The spec's goal is one internal login and one booking action that the gateway fans out to every carrier.

### Success criteria

- An operator can create, cancel, and track a booking for any of the four v1 carriers from one internal system, without logging in to a carrier portal (spec §1, §2).
- Booking status is presented in one canonical model regardless of carrier (spec §5).
- The four adapters pass their contract test suites against mock carrier servers; the same suites can be re-pointed at real sandboxes when credentials arrive (spec §2, §12).

> Numeric performance / throughput targets are not stated in the spec — see OQ-CN-6, OQ-CN-7.

### Sources

- PRD §1 (Context), §2 (Goals and non-goals), §7 (Internal API surface — roles)

### Out of Scope

- Shipping Instruction (SI) submission, booking amendment, point-to-point schedule / sailing search (spec §2 non-goals; D-011).
- Live webhook ingestion — seam only in v1 (spec §2, §5; D-006).
- Full EDIFACT message set and `edi-transport` (SFTP/AS2) (spec §2, §14; D-014).
- OOCL, ZIM, MSC adapters (spec §2).
- Migration of Motus / Pluckd / Greenlit / Hive into the monorepo (spec §2).
- Dashboard, saved views, bulk actions, reporting (spec §2).

## Architecture

> **TL;DR**: modular-monolith gateway — one backend service (API + worker) + a thin React frontend + shared packages; in-process transport-agnostic carrier adapters; Postgres-backed async polling.

### System overview

`apps/booking` is one backend service (Fastify API routes plus a `graphile-worker` poller in the same process) and a thin Vite/React 19 frontend. Shared packages carry the reusable parts: `carrier-core` (canonical types, the `CarrierAdapter` interface, the HTTP client, the contract-test kit), `edifact` (a translator seam, skeleton only), `db` (migration runner, pool), `auth` (session/JWT + RBAC). Each carrier adapter is an in-process module under `apps/booking/src/adapters/`. `apps/mock-carriers` simulates each carrier's REST API for dev and test. Data flows Browser → API → Domain services → Postgres; carrier calls happen only inside worker jobs, never in a request.

```
        Browser (React 19, thin UI)
                 |
        API routes (Fastify, zod, RBAC)      apps/booking/src/api
                 |
        Domain services  ── statusReconciler (single writer of booking status)
                 |                    apps/booking/src/domain
        Postgres  <── worker jobs ──> Carrier adapters ──> carrier REST API / mock
        (source of truth)   apps/booking/src/worker      apps/booking/src/adapters
                                                          packages/carrier-core
```

### Backend

| Component | Purpose | Source |
|---|---|---|
| `api/` | Typed route handlers per domain: `/bookings`, `/tracking`, `/carriers`, `/health` | spec §7 |
| `domain/bookingService` | Create booking, guard cancel, enqueue jobs | spec §9 |
| `domain/trackingService` | Read/query normalized tracking events | spec §7, §9 |
| `domain/statusReconciler` | The only writer of `bookings.status` and `booking_events`; applies the allowed-transition table | spec §9 |
| `domain/CredentialStore` + `PgCredentialStore` | Encrypted per-carrier credential get/put/rotate/list | spec §13 |
| `worker/` | `graphile-worker` tasks (`submit_booking`, `poll_booking_status`, `poll_tracking`, `submit_cancellation`) + cron enqueuers | spec §9 |
| `db/` | SQL migration files + query modules (no ORM) | spec §4, §6 |

**Tech stack (Backend)**: TypeScript, Fastify, `pg`, `node-pg-migrate`, `graphile-worker`, zod, pino (spec §4; Fastify vs Express — OQ-AR-1; queue choice — OQ-AR-2).

### Web Frontend

| Component | Purpose | Source |
|---|---|---|
| Bookings list | Table + filters (status, carrier, date range, customer ref) + pagination; default screen | spec §15 |
| New booking | Form for the canonical booking request; client-side zod validation mirrors the server | spec §15 |
| Booking detail | Header + tabs: Overview, Timeline (`booking_events`), Tracking (`shipment_events`), Carrier messages (admin only); actions Cancel + Refresh | spec §15 |
| Carriers / settings | Admin: carrier registry, capability flags, credential health, set/rotate credential form | spec §15 |

**Tech stack (Web Frontend)**: TypeScript, Vite, React 19, TanStack Query. Motus shell patterns are copied and adapted, not imported. No business logic in the frontend — it renders `status` from the API (spec §15).

### Integrations

| Component | Purpose | Source |
|---|---|---|
| `packages/carrier-core` | `CarrierAdapter` interface, canonical DCSA-based types + zod, `HttpClient` (auth strategies, retry/backoff, circuit breaker, `carrier_messages` logging hook), `AdapterResult`/`AdapterError`, `ContractTestKit` | spec §8 |
| `apps/booking/src/adapters/one` | ONE (ONEY) adapter — API key header; booking + T&T close to DCSA 2.0 | spec §10 |
| `apps/booking/src/adapters/maersk` | Maersk (MAEU) adapter — OAuth2 client-credentials + `Consumer-Key` header; separate sandbox host | spec §10 |
| `apps/booking/src/adapters/hmm` | HMM (HDMU) adapter — API key; partial DCSA adoption, possibly proprietary booking payload; Korean-language portal | spec §10 |
| `apps/booking/src/adapters/evergreen` | Evergreen (EGLV) adapter — signed JWT + mandatory IP allowlist; proprietary payloads | spec §10 |
| `apps/mock-carriers` | Small HTTP app per carrier: scripted ack / status progression / tracking / cancel + error scenarios via `x-mock-scenario`; dev/test only | spec §11 |
| `packages/edifact` | Translator seam + skeleton: `toEdifact`/`fromEdifact`, `MessageType` enum, working IFTMBF reference builder + parser, other message types stubbed | spec §14 |

**Tech stack (Integrations)**: TypeScript. Auth strategies: `OAuth2ClientCredentials` (token cache), `ApiKeyHeader`, `JwtBearer` (spec §8, §10).

### API contracts

> All internal, JSON, timestamps ISO-8601 UTC. Mutating routes behind RBAC (`admin` / `controller` / `viewer`), server-side checks (spec §7).

| Endpoint | Method | Purpose | Auth | Notes / errors | Source |
|---|---|---|---|---|---|
| `/bookings` | POST | Create a canonical booking → `202 {bookingId, status: PENDING_SUBMISSION}` | controller | accepts `Idempotency-Key` header, unique per `created_by` | spec §7 |
<!-- BIND: confirmed=C-AR-04 -->
| `/bookings` | GET | List + filter (status, carrier, pol, pod, date range, customerRef), paginated | viewer | | spec §7 |
<!-- BIND: confirmed=C-AR-05 -->
| `/bookings/:id` | GET | Canonical record + status + `booking_events` + `shipment_events` | viewer | | spec §7 |
<!-- BIND: confirmed=C-AR-06 -->
| `/bookings/:id/cancel` | POST | Request cancellation → `202 {status: CANCELLATION_REQUESTED}` | controller | guard: status in cancelable set AND `carrier.capabilities.supportsCancel` | spec §7, §9 |
<!-- BIND: confirmed=C-AR-07 -->
| `/bookings/:id/refresh` | POST | Force a status/tracking poll now (enqueue jobs) | controller | | spec §7 |
<!-- BIND: confirmed=C-AR-08 -->
| `/bookings/:id/events` | GET | `booking_events` audit timeline | viewer | | spec §7 |
<!-- BIND: confirmed=C-AR-09 -->
| `/bookings/:id/tracking` | GET | `shipment_events` for this booking | viewer | | spec §7 |
<!-- BIND: confirmed=C-AR-10 -->
| `/tracking` | GET | Query events across bookings (equipmentRef, carrier, date range), paginated | viewer | | spec §7 |
<!-- BIND: confirmed=C-AR-11 -->
| `/carriers` | GET | Registry + capabilities + credential health (`configured`, `lastAuthOk`, `circuitState`) | viewer | never exposes secret material | spec §7, §13 |
<!-- BIND: confirmed=C-AR-12 -->
| `/carriers/:code/credentials` | PUT | Set / rotate a carrier credential | admin | appends `credential_events` | spec §7, §13 |
<!-- BIND: confirmed=C-AR-13 -->
| `/health` | GET | DB ping + worker heartbeat + per-carrier circuit state + `lastAuthOk` | viewer | | spec §7, §16 |
<!-- BIND: confirmed=C-AR-14 -->

### Sources

- PRD §4 (Monorepo layout and tooling), §7 (Internal API surface), §8 (CarrierAdapter interface and carrier-core), §9 (Async processing), §10 (Per-carrier adapters), §11 (Mock carrier servers), §13 (Credential store), §15 (Frontend), §16 (Observability)

### Out of Scope

- Webhook receiver endpoints — the adapter carries an unused `handleWebhook?` slot in v1 (spec §5, §8).
- `packages/ui` promotion of the status chip / table shell — later, not v1 (spec §15).
- `packages/edi-transport` (SFTP/AS2) (spec §14).

## Decisions

> ADR-lite. One entry per decision with an explicit source in the spec.

### D-001: Standalone monorepo, not a module inside Motus
<!-- BIND: confirmed=C-DC-01 -->

The gateway needs its own repo and deploy, and will become the home of a future monorepo (Motus, Pluckd, Zhengheinventory). **Decision**: build a new git repo `zhenghe-monorepo` with a minimal skeleton (`apps/` + `packages/`) and put the booking gateway in `apps/booking`; migration of existing apps is not part of this work. **Consequences**: clean start on one stack; the monorepo conventions from the Greenlit PRD (§0, §73–85) are adopted now, other apps migrate later. **Source**: spec §Decisions log #1–#2, §4.

### D-002: Modular-monolith gateway (Approach 1)
<!-- BIND: confirmed=C-DC-02 -->

Three architectures were weighed (modular monolith, gateway + worker pool, per-carrier microservices). **Decision**: one backend service (API + poller in one process) + a thin frontend, with in-process adapters and a Postgres-backed job queue. **Consequences**: fastest to build and operate, one deploy, all TypeScript; carrier failure is isolated by a per-carrier circuit breaker rather than by process. **Source**: spec §Decisions log #13, §3 (Three approaches).

### D-003: Canonical booking model based on DCSA Booking 2.0
<!-- BIND: confirmed=C-DC-03 -->

All four v1 carriers expose DCSA-aligned REST APIs. **Decision**: the internal canonical booking request and status enum are a trimmed subset of DCSA Booking 2.0; adapters map each carrier's states and payloads to the canonical model. **Consequences**: one model for the whole system; carrier deviations are contained in adapters; proprietary carriers (HMM, Evergreen) still map onto the same canonical shape. **Source**: spec §5.

### D-004: Transport-agnostic CarrierAdapter interface
<!-- BIND: confirmed=C-DC-04 -->

Future carriers will use EDIFACT or hybrid API+EDIFACT. **Decision**: the `CarrierAdapter` interface exposes `createBooking` / `getBookingStatus` / `cancelBooking` / `getTracking` / `toCanonicalStatus` (+ optional `handleWebhook`); how an adapter fulfils them (REST, or build EDIFACT + SFTP/AS2 + parse) is internal to the adapter. **Consequences**: the gateway and canonical model never change when a transport changes; an EDIFACT adapter later composes `carrier-core` + `edifact` + a future `edi-transport`. **Source**: spec §8, §"EDIFACT addition" (§14 seam).

### D-005: ZHL Postgres is source of truth; carrier is system-of-record for confirmation
<!-- BIND: confirmed=C-DC-05 -->

**Decision**: every booking is stored as a canonical record in ZHL Postgres (request + status + event history + raw carrier responses); the internal system reads only its own DB and never calls a carrier synchronously during a render. Carrier responses are reconciled into that record. **Consequences**: resilient to carrier API downtime; full event/audit history; raw responses aid debugging when sandboxes come online. Local Postgres in dev, Supabase Postgres in production. **Source**: spec §Decisions log #6, §6.

### D-006: Polling for status updates in v1; webhook seam retained
<!-- BIND: confirmed=C-DC-06 -->

**Decision**: a scheduler polls active bookings and confirmed-booking tracking on a configurable interval; the adapter interface keeps an unused `handleWebhook?` slot and the reconciler is written so events can later arrive from a webhook without a data-model change. **Consequences**: works for all carriers day one, latency of a few minutes; webhooks can be added per carrier later. **Source**: spec §Decisions log #5, §5, §9.

### D-007: Encrypted Postgres credential store, master key from env
<!-- BIND: confirmed=C-DC-07 -->

**Decision**: per-carrier secrets live in a `carrier_credentials` table, encrypted at rest with libsodium `crypto_secretbox`; the master key comes from `CARRIER_CRED_MASTER_KEY`; a `CredentialStore` interface allows swapping to an external secrets manager later. **Consequences**: rotation per carrier, audit via `credential_events`, everything inside the ZHL stack; libsodium Node binding is unspecified (OQ-AR-4). **Source**: spec §Decisions log #7, §13.

### D-008: Postgres-backed job queue (graphile-worker)
<!-- BIND: confirmed=C-DC-08 -->

**Decision**: async work (submit, poll, cancel) runs on a Postgres-backed queue; `graphile-worker` is the recommended implementation (built-in cron), with a hand-rolled `jobs` table + `FOR UPDATE SKIP LOCKED` as the fallback. **Consequences**: no extra infrastructure; retries with backoff are built in; final choice still open (OQ-AR-2). **Source**: spec §4 (tooling), §9.

### D-009: Single status-reconcile choke point
<!-- BIND: confirmed=C-DC-09 -->

**Decision**: `domain/statusReconciler.ts` is the only module that writes `bookings.status` or appends `booking_events`; submit, poll, cancel and future webhook paths all call `reconcile(bookingId, observedStatus, source, raw)`, which applies an allowed-transition table. **Consequences**: illegal transitions are logged and dropped; one place to reason about state; mirrors the Greenlit "one layer decides" rule. **Source**: spec §9.

### D-010: Build against per-carrier mock servers until sandbox credentials arrive
<!-- BIND: confirmed=C-DC-10 -->

No carrier credentials exist yet (applications in progress). **Decision**: build against `apps/mock-carriers` (one scripted mock per carrier) plus a DCSA contract-test suite per adapter; base URL + secret swap in when a sandbox is granted, and the same contract suite runs against it. **Consequences**: development is not blocked on carrier onboarding; end-to-end carrier verification is deferred (OQ-CN-1). **Source**: spec §Decisions log #3, §2, §11, §12.

### D-011: v1 scope is create + status + cancel + tracking
<!-- BIND: confirmed=C-DC-11 -->

**Decision**: v1 supports creating a booking, polling its status, cancelling it, and pulling tracking events. Shipping Instruction submission, booking amendment, and point-to-point schedule search are deferred to v2; operators enter vessel/voyage/POL/POD/ETD manually. **Consequences**: smaller surface per carrier; tracking adds a third DCSA API surface (Track & Trace 2.x) with more per-carrier variation than booking. **Source**: spec §Decisions log #4, #11, #12, §2.

### D-012: One active ZHL account per carrier; schema account-ready
<!-- BIND: confirmed=C-DC-12 -->

**Decision**: v1 uses one set of credentials and one booking party per carrier; `carrier_credentials` carries an `account_ref` column so multiple accounts per carrier can be added later without a schema change. **Consequences**: simplest booking form and credential model now; a partial unique index enforces one active credential row per carrier. **Source**: spec §Decisions log #9, §6.

### D-013: TypeScript + React 19 + Fastify + pnpm workspaces, no ORM
<!-- BIND: confirmed=C-DC-13 -->

**Decision**: the monorepo is TypeScript from the first commit; frontend is Vite + React 19; backend is Fastify with normal API routes (not serverless); database access is hand-written SQL via `pg` with `node-pg-migrate`; package manager is pnpm workspaces. **Consequences**: matches the Greenlit PRD conventions; Express is offered as a Fastify alternative "to match Motus" (OQ-AR-1). **Source**: spec §4.

### D-014: `packages/edifact` is a seam plus skeleton in v1
<!-- BIND: confirmed=C-DC-14 -->

**Decision**: build the `edifact` package with the translator interface (`toEdifact` / `fromEdifact`), the `MessageType` enum, segment primitives, and a single working IFTMBF reference builder + parser with a round-trip test; other message types are stubs that throw `NotImplemented`. **Consequences**: the architecture stays correct for EDIFACT carriers without building a full parser before one is needed; the full message set + `edi-transport` is a later spec. **Source**: spec §Decisions log #12, §14.

### D-015: Static egress IP for Evergreen IP allowlist
<!-- BIND: confirmed=C-DC-15 -->

**Decision**: the self-hosted deployment uses a static public egress IP (or a reverse proxy with a fixed IP) so it can be registered on Evergreen's allowlist; the other three adapters do not need it. **Consequences**: the Evergreen adapter can run normally; the IP must be registered with Evergreen during onboarding (OQ-CN-2 area). **Source**: spec §Decisions log #10, §10, §18.3.

## Glossary

Product-specific terms (standard terms — ADR, DBML, DoD, FK, NFR, OQ, RTO, RPO, SLO — live in `_meta/ai-consumer-guide.md` §Standard terms):

| Term | Definition |
|---|---|
| DCSA Booking 2.0 | Digital Container Shipping Association standard data model and API for ocean freight booking; the basis for the canonical booking model (spec §5). |
| Track & Trace (T&T) 2.x | DCSA standard for shipment / transport / equipment tracking events; the third carrier API surface in v1 (spec §2, §D-011). |
| Canonical booking request | The internal, carrier-neutral booking payload (a trimmed DCSA subset) stored as `jsonb` on `bookings` (spec §5). |
| Canonical booking status | The internal status enum (`PENDING_SUBMISSION` … `CONFIRMED` / `DECLINED` / `CANCELLED` …); adapters map carrier states to it (spec §5). |
| Adapter | A per-carrier module implementing `CarrierAdapter`; transport-agnostic, never touches the DB (spec §8, §D-004). |
| Reconcile / choke point | `domain/statusReconciler.ts` — the single writer of `bookings.status` and `booking_events` (spec §9, §D-009). |
| Active set | Bookings with status `SUBMITTED`, `RECEIVED`, `PENDING_CONFIRMATION`, or `CANCELLATION_REQUESTED` — the set the status poller runs for (spec §5, §9). |
| Cancelable set | Bookings with status `SUBMITTED`, `RECEIVED`, `PENDING_CONFIRMATION`, or `CONFIRMED` — states from which cancel is allowed (spec §5, §9). |
| Contract test | `ContractTestKit` suite (`create → status → tracking → cancel`) run against a mock now and a real sandbox later (spec §8, §12). |
| SCAC | Standard Carrier Alpha Code — carrier identifier (ONEY, MAEU, HDMU, EGLV) used as `carriers.code` (spec §6, §10). |
| IFTMBF | UN/EDIFACT firm booking request message; the one working reference in `packages/edifact` (spec §14). |
| Booking party | The shipper/contract account on the carrier side that a booking is placed under; one per carrier in v1 (spec §6, §D-012). |

## Auto-Classification Review

> Total classified: 16 OQs. Auto-resolution active: 0 (no codebase context yet — every tech OQ is `blocking`). Manual review recommended: 6 (all tech OQs).

| OQ-ID | Question | Auto-tagged | Confidence | Action |
|---|---|---|---|---|
| OQ-AR-1 | Fastify vs Express for the backend | tech / blocking | medium | needs eng-lead decision at monorepo skeleton (chunk 1) |
| OQ-AR-2 | `graphile-worker` vs hand-rolled job table | tech / blocking | medium | needs eng-lead decision at chunk 1–2 |
| OQ-AR-3 | Node.js + pnpm versions not pinned | tech / blocking | low | decide at monorepo skeleton (chunk 1) |
| OQ-AR-4 | libsodium Node binding for `crypto_secretbox` | tech / blocking | low | decide at `carrier-core` / credential store (chunk 3–4) |
| OQ-AR-5 | internal-user auth implementation details | tech / blocking | low | decide against the Greenlit auth plan (chunk 5) |
| OQ-AR-6 | CI ephemeral Postgres provisioning | tech / blocking | low | decide at CI setup (chunk 1) |

> No codebase exists yet, so no tech OQ can be auto-resolved by scan or grounded for a recommendation; all are `blocking` pending a human decision at the named build chunk.

## Source documents

- **PRD**: `docs/superpowers/specs/2026-09-01-carrier-booking-gateway-design.md`, v1, 2026-09 — FINAL (approved for implementation planning). Committed as `d6f2d5a`.
- Referenced: Project Greenlit PRD Part I (§0 Build Approach, §73–85) for monorepo conventions; `carrier-booking-gateway-analysis.html` / `.pdf` (7-carrier analysis).

## Changelog

### v1.0 (2026-09-01)
- Initial vault generated from the approved design spec (v1, 2026-09).
- Mode: new (greenfield). Project shape: web-app. Output mode: compact.

## Last updated

2026-09-01
