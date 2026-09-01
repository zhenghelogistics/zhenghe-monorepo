# Project Constitution

**Status**: Active
**Version**: 1.0
**Last reviewed**: 2026-09-01
**Sign-off**: Tech Lead (pending) — review before bolts begin

---

## §A. Coding standards (Non-negotiable)

- A-001: TypeScript from the first commit across every package and app; no JavaScript source files (source: PRD §4).
- A-002: Database access is hand-written SQL via `pg`; no ORM. Schema changes are SQL migration files run by `node-pg-migrate` (source: PRD §4, §6).
- A-003: Every internal API route validates its request body and params with a zod schema at the boundary; the canonical booking schema in `packages/carrier-core` is the single definition and is reused on the frontend (source: PRD §5, §7, §8).
- A-004: Backend uses normal API routes (Fastify), not serverless functions (source: PRD §4).

## §B. Security baselines

- B-001: Carrier secrets are stored only in `carrier_credentials`, encrypted at rest with libsodium `crypto_secretbox`; the master key comes from the `CARRIER_CRED_MASTER_KEY` environment variable and is never committed (source: PRD §13).
- B-002: Decrypted credentials exist only in memory while building an `AdapterContext`; they are never logged and never written to `carrier_messages` (source: PRD §13).
- B-003: `carrier_messages` request bodies are redacted of auth headers and secret fields before they are stored (source: PRD §6, §13).
- B-004: All permission checks are server-side (RBAC roles `admin` / `controller` / `viewer`); frontend role checks are cosmetic only (source: PRD §7, §13).
- B-005: All carrier and client transport is over TLS (source: PRD §13).
- B-006: `.env` is never committed; `.env.example` lists every key with a placeholder value (source: PRD §13, §16).

## §C. Architecture invariants

- C-001: Carrier adapters are transport-agnostic and never touch the database. An adapter takes canonical input and returns normalized output; persistence and the state machine live in `domain/` (source: PRD §8, vault.md D-004).
- C-002: `domain/statusReconciler.ts` is the only module that writes `bookings.status` or appends `booking_events`. Submit, poll, cancel and any future webhook path call `reconcile(bookingId, observedStatus, source, raw)` (source: PRD §9, vault.md D-009).
- C-003: The frontend renders `status` from the API and never derives or computes it (source: PRD §9, §15).
- C-004: Illegal status transitions are logged and dropped, never applied; the allowed-transition table in the reconciler is authoritative (source: PRD §5, §9).
- C-005: ZHL Postgres is the source of truth for internal state. The internal system never calls a carrier synchronously during a request render; all carrier calls run inside worker jobs (source: PRD §2, §6, §9, vault.md D-005).
- C-006: Carrier status strings never leak past the adapter — only `bookings.carrier_status_raw` and `carrier_messages` hold them; everything else uses the canonical enum (source: PRD §5).

## §D. Anti-patterns (never replicate)

- D-001: Never add a carrier integration as a synchronous call in the request path (source: PRD §9).
- D-002: Never build one flat booking form when the canonical request has repeatable groups (commodities, equipment); model them as repeatable inputs (source: PRD §5, §15).
- D-003: No new third-party dependency without review; the aggregator-vs-direct decision stays open (OQ-CN-5) (source: PRD §18.6).
- D-004: Do not implement the full EDIFACT message set or `edi-transport` in v1 — `packages/edifact` is a seam with a single working IFTMBF reference (source: PRD §14, vault.md D-014).
- D-005: Do not let tracking events change `bookings.status` in v1 — tracking and booking status are kept separate (source: PRD §9).

## §E. Performance constraints

<!-- The spec states no numeric latency / throughput / concurrency targets. Every such value is an
     Open Question (OQ-CN-6, OQ-CN-7), never invented here. The clauses below are the reliability
     invariants the spec does state. -->

- E-001: A job that exhausts its retries must not fail silently — it writes an `ERROR` `booking_events` row, logs, and increments a metric; the booking stays visibly parked in `SUBMISSION_FAILED` / `CANCELLATION_FAILED` (source: PRD §9, §16).
- E-002: Status polling runs only for bookings in the active set; tracking polling runs only for `CONFIRMED` bookings. Poll cadence is configured via env and is pending tuning (OQ-CN-6) (source: PRD §9, §16).
- E-003: Every job and carrier exchange is safe to re-run (idempotent): `submit_*` jobs guard on current status, `poll_tracking` upserts by `dedupe_key` (source: PRD §9, §16).

## §F. Compliance

- F-001: Every booking state change is appended to `booking_events` with actor, source, from/to status and a correlation id — an append-only audit timeline (source: PRD §6, §9).
- F-002: Credential `set` / `rotate` / `disable` actions are recorded in `credential_events` with actor and timestamp (source: PRD §6, §13).
- F-003: Operational records use soft delete / archive; no hard deletes (source: PRD §13).
- F-004: Timestamps are stored in UTC and displayed in `Asia/Singapore` (source: PRD §13, §16).
- F-005: Legal review of each carrier's API terms is required before go-live (OQ-CN-3) (source: PRD §18.4).
