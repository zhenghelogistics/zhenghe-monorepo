---
type: prose
doc_id: constraints
vault_version: "1.0"
aliases: [Constraints, NFR, Non-Functional Requirements]
tags: ["vault/carrier-booking-gateway", "doc/constraints"]
---

# Constraints & Open Questions

> **TL;DR**: stack and infra locks, carrier-onboarding blockers, reliability/observability NFRs, and the one authored home of every Open Question · architect, tech lead, ops management, legal · read when choosing a solution or reviewing readiness.

## Technical constraints

- **Stack lock**: TypeScript from the first commit, Vite + React 19 frontend, Fastify backend with normal API routes (not serverless), `pg` + `node-pg-migrate` (no ORM), pnpm workspaces (spec §4).
- **Persistence**: Postgres — local in development, Supabase Postgres in production; ZHL Postgres is the source of truth for internal state (spec §6, D-005).
- **Job processing**: Postgres-backed queue; `graphile-worker` recommended, hand-rolled `FOR UPDATE SKIP LOCKED` table as the fallback (spec §4, §9; OQ-AR-2).
- **Deployment**: self-hosted, single deployable service (API + worker in one process); a static public egress IP (or fixed-IP reverse proxy) is required for Evergreen's IP allowlist (spec §4, §10, §Decisions #10).
- **Credential encryption**: libsodium `crypto_secretbox` at rest, master key from `CARRIER_CRED_MASTER_KEY` env; decrypted only in memory while building an `AdapterContext`, never logged (spec §13).
- **Carrier API alignment**: ONE, Maersk broadly DCSA Booking 2.0; HMM partial DCSA (possibly proprietary booking payload); Evergreen proprietary (not DCSA). Deviations are contained in adapters (spec §5, §10).
- **Time**: timestamps stored in UTC, API returns ISO-8601 UTC, frontend displays `Asia/Singapore` (spec §13, §16).
- **Files**: operational documents on the NAS; operational records use soft delete / archive, no hard deletes (spec §13).

## Business constraints

- **No carrier credentials yet**: applications are in progress for all four carriers; no end-to-end test against a real carrier is possible until at least one sandbox is granted. Build proceeds against mock servers (spec §2, §18.1; OQ-CN-1).
- **HMM onboarding barrier**: appears to require a Korean business registration number and a partly Korean-language portal (spec §10, §18.2; OQ-CN-2).
- **Evergreen onboarding**: the static egress IP must be registered on Evergreen's allowlist during onboarding (spec §10, §18.3).
- **Legal review**: each carrier's API terms and conditions must be reviewed before go-live (spec §18.4; OQ-CN-3).
- **Build-direct vs aggregator**: this spec builds direct adapters; a later decision to use an aggregator for some carriers affects cost and timeline (spec §18.6; OQ-CN-5).

## Non-functional requirements

| Category | Requirement | Source |
|---|---|---|
| Reliability | Email/job processing is idempotent — `submit_booking` and `submit_cancellation` act only if the booking is still in the expected status; `poll_tracking` upserts by `dedupe_key`; any job is safe to re-run | PRD §9, §16 |
| Reliability | No silent failure — a job that exhausts retries writes an `ERROR` `booking_events` row, logs, and increments a metric; the booking is visibly parked in `SUBMISSION_FAILED` / `CANCELLATION_FAILED` | PRD §9, §16 |
| Observability | Structured logging (pino): every job and carrier exchange logs `{ correlationId, bookingId, carrierCode, operation, outcome, latencyMs }`; correlation id generated at API ingress and threaded through job payloads | PRD §16 |
| Observability | Named log events: `booking.created/submitted/submit_failed/status_changed/cancel_requested/cancelled`, `tracking.events_ingested`, `carrier.auth_failed`, `carrier.circuit_opened`, `credential.rotated` | PRD §16 |
| Availability | `GET /health` checks DB, worker heartbeat, per-carrier circuit state and `lastAuthOk`; the system reads its own DB and never calls a carrier during a request render | PRD §7, §6, §16 |
| Security | RBAC roles `admin` / `controller` / `viewer` with server-side permission checks; frontend checks are cosmetic; TLS on all transport; `carrier_messages` request bodies redacted of auth headers and secret fields | PRD §7, §13 |
| Auditability | Append-only `booking_events` per state change (actor, source, from/to, correlation id); `credential_events` per credential action; raw carrier exchanges in `carrier_messages` | PRD §6, §9, §13 |
| Testability | CI runs unit + contract + domain + API tests on every PR with an ephemeral Postgres; each adapter passes `ContractTestKit` against its mock | PRD §12 |

> Numeric latency / throughput / concurrency targets are not stated in the spec — see OQ-CN-6, OQ-CN-7. Do not invent SLO values.

---

## Sources

- PRD §2 (Goals and non-goals), §4 (Monorepo layout and tooling), §5 (Canonical data model), §6 (Database schema), §9 (Async processing), §10 (Per-carrier adapters), §12 (Testing strategy), §13 (Credential store and security), §16 (Observability, config, error handling), §18 (Open questions and blockers for management)

## Out of Scope

- GDPR / EU data residency — no stated EU scope in the spec.
- Numeric performance SLAs — none stated; captured as Open Questions, not defaulted.
- Webhook signature verification per carrier — deferred with the webhook seam (spec §5, §8).

## Open Questions

> The one authored OQ surface. Sorted P1 → P2 → P3. `[origin: ...]` names where the question arose; the `[business]` / `[tech / <mode>]` bracket is mandatory.

- [ ] **OQ-CN-1** [P1] [business]: Carrier credential applications are pending for all four carriers — no end-to-end test against a real carrier is possible until at least one sandbox is granted. Which carrier is expected first, and by when? — resolve: management / carrier onboarding
- [ ] **OQ-CN-2** [P1] [business]: HMM onboarding appears to require a Korean business registration number and a partly Korean-language portal. Can ZHL satisfy this directly, or is a local partner / agent required? — resolve: management
- [ ] **OQ-CN-3** [P1] [business]: Legal review of each carrier's API terms and conditions has not been done. Who owns it and what is the timeline before go-live? — resolve: legal
- [ ] **OQ-AR-1** [P2] [tech / blocking] [conf: medium] [origin: vault.md#Architecture]: Backend HTTP framework — the spec recommends Fastify but offers Express "to match Motus". Confirm the pick before the monorepo skeleton (chunk 1). — resolve: eng lead
- [ ] **OQ-AR-2** [P2] [tech / blocking] [conf: medium] [origin: vault.md#Architecture]: Job queue — `graphile-worker` (recommended) vs a hand-rolled `jobs` table with `FOR UPDATE SKIP LOCKED`. Confirm before chunk 2. — resolve: eng lead
- [ ] **OQ-AR-3** [P2] [tech / blocking] [conf: low] [origin: vault.md#Architecture]: Node.js and pnpm versions are not pinned. — resolve: decide at the monorepo skeleton (chunk 1)
- [ ] **OQ-AR-4** [P2] [tech / blocking] [conf: low] [origin: vault.md#Architecture]: The libsodium Node binding for `crypto_secretbox` (`sodium-native` vs `libsodium-wrappers`) is not specified. — resolve: eng at `carrier-core` / credential store (chunk 3–4)
- [ ] **OQ-AR-5** [P2] [tech / blocking] [conf: low] [origin: vault.md#Architecture]: Internal-user auth implementation (password hashing, session store, JWT library) is described only as "local users mirroring the Greenlit plan". — resolve: eng, against the referenced Greenlit auth plan (chunk 5)
- [ ] **OQ-DM-1** [P2] [business] [origin: model.md#Constraints]: The `Idempotency-Key` uniqueness window has no expiry — keys are unique per `created_by` indefinitely. Is a TTL or retention window wanted? — resolve: eng / product
- [ ] **OQ-DM-2** [P2] [business] [origin: model.md#Entities]: HMM and Evergreen use proprietary (non-DCSA) booking payloads; the exact request/response shapes are unknown until their sandboxes are available. — resolve: deferred to sandbox onboarding (ties to OQ-CN-1)
- [ ] **OQ-CN-4** [P2] [business]: Who in operations owns bookings parked in `SUBMISSION_FAILED` / `CANCELLATION_FAILED`, and what is the response-time expectation? — resolve: ops management
- [ ] **OQ-CN-5** [P2] [business]: Build-direct vs aggregator for some carriers — the spec builds direct adapters; confirm no aggregator is planned for v1. — resolve: management
- [ ] **OQ-CN-6** [P2] [business]: Poll interval values (`POLL_INTERVAL_BOOKING_MIN`, `POLL_INTERVAL_TRACKING_MIN`) are not specified. — resolve: ops + eng tuning
- [ ] **OQ-AR-6** [P3] [tech / blocking] [conf: low] [origin: vault.md#Architecture]: CI ephemeral-Postgres provisioning mechanism (service container, Docker, Testcontainers, etc.) is not specified. — resolve: eng at CI setup (chunk 1)
- [ ] **OQ-CN-7** [P3] [business]: `HTTP_TIMEOUT_MS`, `CIRCUIT_FAIL_THRESHOLD`, and the `submit_booking` retry max-attempts N are not specified. — resolve: eng tuning
- [ ] **OQ-FL-1** [P3] [business] [origin: flows.md#F-S-003]: Tracking polling has no stop condition — `CONFIRMED` bookings are polled until an operator archives them; a delivered-state stop is explicitly deferred to a later phase. Confirm this is acceptable for v1. — resolve: product
