# Carrier Booking Gateway — Design Spec (v1)

Date: 2026-09-01
Status: Approved for implementation planning
Repo: `zhenghe-monorepo`
Owner: Zhenghe Logistics

## 1. Context

Zhenghe Logistics books ocean freight across seven carriers, each with its own
portal and login. The goal is a single internal booking gateway so operators log
in to one ZHL system, book once, and the gateway talks to every carrier.

This spec covers **v1**: four REST/DCSA-aligned carriers — ONE, Maersk, HMM,
Evergreen — inside a new monorepo (`zhenghe-monorepo`). The other three carriers
from the earlier analysis (OOCL EDI, ZIM, MSC) are out of scope for v1 but the
architecture must not preclude them, including EDIFACT and hybrid API+EDIFACT
carriers.

Prior art: `carrier-booking-gateway-analysis.html` / `.pdf` (7-carrier analysis).
Monorepo conventions follow Project Greenlit PRD Part I, Section 0 and Sections
73–85.

## 2. Goals and non-goals

### v1 goals

- New monorepo `zhenghe-monorepo` with a minimal skeleton (Option A): build
  tooling, `packages/{carrier-core,edifact,db,auth,config}`, `apps/booking`,
  `apps/mock-carriers`. Migration of Motus / Pluckd / Greenlit / Hive is **not**
  blocked by and **not** part of this work.
- `apps/booking`: an API-first gateway with a thin UI.
- Operations: **create booking**, **status**, **cancel**, **tracking**.
- Four carrier adapters: ONE (ONEY), Maersk (MAEU), HMM (HDMU), Evergreen (EGLV).
- Credential-agnostic: adapters run against per-carrier mock servers now; base URL
  plus secret swap in when sandbox credentials arrive. Each adapter carries a
  contract test suite.
- Status updates by **polling** in v1; architecture leaves a webhook seam
  (`handleWebhook?` on the adapter, single reconcile choke point).
- ZHL Postgres is the source of truth for internal state; carrier is the
  system-of-record for confirmation. Raw carrier exchanges are persisted.
- Encrypted per-carrier credential store in Postgres, master key from env.
- One active ZHL account per carrier (schema is account-ready but v1 uses one).
- `packages/edifact`: seam plus skeleton only (working IFTMBF reference builder
  and parser, other message types stubbed).

### Non-goals for v1 (later specs)

- Shipping Instruction (SI) submission.
- Booking amendment.
- Point-to-point schedule / sailing search (operators enter vessel/voyage/POL/POD/
  ETD manually).
- Live webhook ingestion (seam only).
- Full EDIFACT message set and `edi-transport` (SFTP/AS2).
- OOCL, ZIM, MSC adapters.
- Migration of existing apps into the monorepo.
- Dashboard, saved views, bulk actions, reporting.

## 3. Decisions log

| # | Decision |
|---|----------|
| 1 | Gateway lives in a **new standalone monorepo**, not inside Motus. |
| 2 | Build **minimal monorepo skeleton + `apps/booking`** only (Option A). |
| 3 | **No carrier credentials yet** (applications in progress). Build against mock servers + DCSA contract tests. |
| 4 | v1 operations: create + status + **cancel** + **tracking**. SI, amend, schedule search deferred. |
| 5 | Status updates: **polling in v1**; design keeps a webhook seam for later (Option C intent). |
| 6 | **ZHL Postgres is source of truth** for internal state; carrier is system-of-record for confirmation. Store raw responses. Local Postgres in dev, Supabase Postgres in production. |
| 7 | Credentials in an **encrypted Postgres table**, master key from env, `CredentialStore` interface for later swap. |
| 8 | `apps/booking` is **API-first with a thin UI** (form, list, detail, cancel). |
| 9 | **One active ZHL account per carrier** (schema account-ready). |
| 10 | **Static egress IP is available** for Evergreen IP whitelisting. |
| 11 | **Manual** vessel/voyage/POL/POD/ETD entry in v1; no schedule API. |
| 12 | `packages/edifact` is a **seam + skeleton** in v1 (working IFTMBF reference only). |
| 13 | Architecture picked: **modular monolith gateway** (Approach 1) — one backend service (API + poller in one process) + thin frontend, in-process adapters, Postgres-backed job queue. |
| 14 | Handoff to **mega-sdd** factory track for implementation. |

## 4. Monorepo layout and tooling

New git repo, pnpm workspaces.

```
pnpm-workspace.yaml
tsconfig.base.json
turbo.json                 (optional build orchestration)
.env.example
.github/workflows/ci.yml   (lint + typecheck + test)

apps/
  booking/
    src/
      api/          typed route handlers per domain: /bookings /tracking /carriers /health
      domain/       gateway services: bookingService, trackingService, statusReconciler
      adapters/     one dir per carrier: one/ maersk/ hmm/ evergreen/ (thin, use carrier-core)
      worker/       poller loop + job queue consumer + cron enqueuers
      db/           migration SQL files + query modules (no ORM)
      config/       env parsing/validation
    web/            React 19 SPA (Vite), thin UI
    test/
  mock-carriers/    small HTTP apps per carrier for dev/test (never deployed to prod)

packages/
  carrier-core/     CarrierAdapter interface, canonical types (DCSA Booking 2.0 basis),
                    zod schemas, HttpClient (auth strategies, retry/backoff, circuit
                    breaker, carrier_messages logging hook), AdapterResult/AdapterError,
                    ContractTestKit
  edifact/          translator seam + skeleton: toEdifact/fromEdifact, MessageType enum,
                    UN/EDIFACT segment primitives, working IFTMBF builder + parser
  db/               migration runner, connection pool, shared base types
  auth/             thin: session/JWT verify, RBAC middleware
  config/           shared tsconfig, eslint, prettier
```

### Tooling

| Area | Choice | Note |
|---|---|---|
| Package manager | pnpm workspaces | monorepo; other apps migrate in later |
| Backend HTTP | Fastify + TypeScript | typed routes, built-in schema validation |
| Frontend | Vite + React 19 + TypeScript | ZHL house style |
| DB access | `pg` + hand-written SQL + `node-pg-migrate` | no ORM (ZHL house style) |
| Job queue | `graphile-worker` (Postgres-native, cron support) | alternative: `jobs` table + `FOR UPDATE SKIP LOCKED` |
| Validation | zod | API boundary + canonical payload parsing |
| Logging | pino (ships with Fastify) | structured events |
| Test | vitest + Fastify `inject` | contract tests via ContractTestKit |
| CI | GitHub Actions: install, typecheck, lint, test | required green before merge |

## 5. Canonical data model

### Canonical booking status (internal enum)

```
DRAFT
PENDING_SUBMISSION
SUBMITTED
RECEIVED
PENDING_CONFIRMATION
CONFIRMED
DECLINED
CANCELLATION_REQUESTED
CANCELLED
CANCELLATION_FAILED
SUBMISSION_FAILED
```

Transitions:

```
DRAFT -> PENDING_SUBMISSION -> SUBMITTED -> RECEIVED -> PENDING_CONFIRMATION
      -> CONFIRMED | DECLINED
<cancelable set> -> CANCELLATION_REQUESTED -> CANCELLED | CANCELLATION_FAILED
any active state -> SUBMISSION_FAILED   (submit exhausted retries or non-retryable error)
```

- **active set** (polled by cron): `{ SUBMITTED, RECEIVED, PENDING_CONFIRMATION, CANCELLATION_REQUESTED }`
- **cancelable set**: `{ SUBMITTED, RECEIVED, PENDING_CONFIRMATION, CONFIRMED }`

DCSA Booking 2.0 carrier states (`RECE`, `PENU`, `PENC`, `CONF`, `REJE`, `CANC`,
`CMPL`) and per-carrier deviations are mapped to the canonical enum **inside the
adapter** (`toCanonicalStatus`). The rest of the system never sees carrier status
strings except in `bookings.carrier_status_raw` and `carrier_messages`.

### Canonical booking request (stored as `jsonb`)

Subset of DCSA Booking 2.0, sufficient for FCL in v1:

```
carrierCode                ONEY | MAEU | HDMU | EGLV
carrierAccountRef          string (v1: the single active account)
shipmentType               FCL
movementType               e.g. CY-CY, CY-DOOR, DOOR-DOOR
incoterms                  optional, e.g. FOB, CIF
bookingParty               { name, address, contact { name, email, phone } }
parties                    { shipper, consignee, notifyParty }  each { name, address }
commodities[]              { description, hsCode?, grossWeight { value, unit }, grossVolume? }
requestedEquipments[]      { isoEquipmentCode (22GP/45GP/40HC/...), units, isShipperOwned }
transport                  { vesselName, voyageNumber, carrierServiceName?,
                             pol (UN/LOCODE), pod (UN/LOCODE),
                             placeOfReceipt? (UN/LOCODE), placeOfDelivery? (UN/LOCODE),
                             requestedDepartureDate (date) }
references[]               { type (CR | PO | FF | ...), value }
flags                      { isPartialLoadAllowed, isExportDeclarationRequired }
```

Validated by a zod schema in `packages/carrier-core`. The same schema is reused
on the frontend.

### Normalized tracking event (`NormalizedEvent`)

```
carrierCode
eventType            SHIPMENT | TRANSPORT | EQUIPMENT
classifier          PLN (planned) | EST (estimated) | ACT (actual)
eventDateTime        ISO-8601 UTC
eventCode            carrier/DCSA event code, kept as-is
description          human-readable
locationUnloc        UN/LOCODE, optional
facility             optional
equipmentReference   container number, optional
transport            { vesselName?, voyageNumber?, port? }, optional
dedupeKey            carrier event id if available, else stable hash of the fields above
raw                  original carrier payload for this event
```

## 6. Database schema

`pg` + hand-written SQL, migrations in `apps/booking/src/db/migrations` run via
`node-pg-migrate`. `graphile-worker` manages its own tables via its installer.

### `carriers`

| column | type | note |
|---|---|---|
| code | text PK | SCAC: ONEY, MAEU, HDMU, EGLV |
| display_name | text | |
| transport_mode | text | REST \| EDIFACT \| HYBRID |
| dcsa_booking_version | text | e.g. `2.0.0`, null if proprietary |
| capabilities | jsonb | `{ supportsCancel, trackingByBooking, trackingByEquipment, supportsWebhook }` |
| enabled | boolean | gateway will not route to a disabled carrier |
| created_at, updated_at | timestamptz | |

Seeded with the four v1 carriers.

### `carrier_credentials`

| column | type | note |
|---|---|---|
| id | uuid PK | |
| carrier_code | text FK -> carriers.code | |
| account_ref | text | v1: one row per carrier; column present for future multi-account |
| auth_type | text | oauth2 \| api_key \| jwt |
| secret_encrypted | bytea | libsodium `crypto_secretbox` of a JSON blob (fields depend on auth_type) |
| key_version | int | for master-key rotation |
| base_url | text | mock URL in dev, sandbox/prod URL later |
| is_active | boolean | one active row per (carrier_code) enforced by partial unique index |
| rotated_at | timestamptz | |
| updated_by | text | actor |
| created_at, updated_at | timestamptz | |

Partial unique index: `unique (carrier_code) where is_active`.

### `bookings`

| column | type | note |
|---|---|---|
| id | uuid PK | internal id |
| carrier_code | text FK | |
| carrier_account_ref | text | |
| status | text | canonical status enum |
| carrier_booking_ref | text | assigned by carrier after submit/ack; nullable |
| canonical_request | jsonb | the full canonical booking request |
| pol | text | UN/LOCODE, denormalized for filtering |
| pod | text | UN/LOCODE, denormalized |
| vessel_name | text | denormalized |
| voyage_number | text | denormalized |
| requested_departure_date | date | denormalized |
| customer_reference | text | denormalized from references[] where type=CR |
| carrier_status_raw | text | last raw carrier status string |
| idempotency_key | text | from `Idempotency-Key` header; nullable |
| created_by | text | actor |
| last_carrier_sync_at | timestamptz | last successful poll |
| created_at, updated_at | timestamptz | |

Unique index: `unique (created_by, idempotency_key) where idempotency_key is not null`.
Index on `status`, on `(carrier_code, status)`, on `requested_departure_date`.

### `booking_events`

Append-only audit timeline.

| column | type | note |
|---|---|---|
| id | uuid PK | |
| booking_id | uuid FK -> bookings.id | |
| seq | bigint | monotonically increasing per booking |
| event_type | text | STATUS_CHANGE \| SUBMITTED \| CANCEL_REQUESTED \| ERROR \| NOTE |
| from_status | text | nullable |
| to_status | text | nullable |
| source | text | adapter \| poller \| user \| webhook \| system |
| actor | text | user id or `system` |
| detail | jsonb | error category, carrier message, correlation id, etc. |
| created_at | timestamptz | |

Unique `(booking_id, seq)`.

### `shipment_events`

Tracking events, upserted by `dedupe_key`.

| column | type | note |
|---|---|---|
| id | uuid PK | |
| booking_id | uuid FK | nullable (event may arrive before link) |
| carrier_code | text | |
| dedupe_key | text | unique |
| event_type | text | SHIPMENT \| TRANSPORT \| EQUIPMENT |
| classifier | text | PLN \| EST \| ACT |
| event_code | text | |
| description | text | |
| event_date_time | timestamptz | |
| location_unloc | text | nullable |
| facility | text | nullable |
| equipment_reference | text | nullable |
| transport_vessel | text | nullable |
| transport_voyage | text | nullable |
| transport_port | text | nullable |
| raw | jsonb | |
| retrieved_at | timestamptz | |
| created_at | timestamptz | |

Unique on `dedupe_key`. Index on `booking_id`, on `equipment_reference`.

### `carrier_messages`

Every raw request/response for audit and debugging (especially once sandboxes go
live).

| column | type | note |
|---|---|---|
| id | uuid PK | |
| booking_id | uuid FK | nullable |
| carrier_code | text | |
| direction | text | out \| in |
| operation | text | create \| status \| cancel \| track \| auth |
| http_status | int | nullable |
| request_body | jsonb | redacted of secrets |
| response_body | jsonb | |
| latency_ms | int | |
| error_text | text | nullable |
| correlation_id | text | |
| created_at | timestamptz | |

Index on `(carrier_code, created_at)`, on `booking_id`. Archival job is a later
concern, not v1.

### `credential_events`

| column | type | note |
|---|---|---|
| id | uuid PK | |
| carrier_code | text | |
| action | text | set \| rotate \| disable |
| actor | text | |
| created_at | timestamptz | |

## 7. Internal API surface

Fastify, zod schemas on every route. All mutating routes behind RBAC middleware
from `packages/auth`. Roles: `admin`, `controller` (create / cancel / refresh),
`viewer` (read only). Permission checks are server-side; the frontend is never
trusted.

```
POST /bookings
     body: canonical booking request
     header: Idempotency-Key (optional, unique per created_by)
     -> 202 { bookingId, status: "PENDING_SUBMISSION" }

GET  /bookings
     query: status, carrier, pol, pod, dateFrom, dateTo, customerRef, page, pageSize
     -> 200 { items: BookingSummary[], page, pageSize, total }

GET  /bookings/:id
     -> 200 { booking: CanonicalBooking, status, carrierBookingRef,
              events: BookingEvent[], tracking: ShipmentEvent[] }

POST /bookings/:id/cancel
     body: { reason?: string }
     guard: status allows cancel AND carrier.capabilities.supportsCancel
     -> 202 { status: "CANCELLATION_REQUESTED" }

POST /bookings/:id/refresh
     -> 202 { enqueued: ["poll_booking_status", "poll_tracking"] }

GET  /bookings/:id/events      -> 200 { events: BookingEvent[] }
GET  /bookings/:id/tracking    -> 200 { events: ShipmentEvent[] }

GET  /tracking
     query: equipmentRef, carrier, dateFrom, dateTo, page, pageSize
     -> 200 { items: ShipmentEvent[], ... }

GET  /carriers
     -> 200 { items: [{ code, displayName, transportMode, capabilities, enabled,
                        credential: { configured, lastAuthOk, circuitState } }] }

PUT  /carriers/:code/credentials       (admin)
     body: { authType, secret: {...}, baseUrl, accountRef? }
     -> 204

GET  /health
     -> 200 { db: "ok", worker: { lastHeartbeatAt }, carriers: [{ code, circuitState, lastAuthOk }] }
```

All timestamps in responses are ISO-8601 UTC.

## 8. CarrierAdapter interface and `carrier-core`

Adapters are transport-agnostic and never touch the database. They take canonical
input, return normalized output.

```ts
interface CarrierAdapter {
  readonly code: string;                          // ONEY, MAEU, HDMU, EGLV
  readonly transport: 'REST' | 'EDIFACT' | 'HYBRID';
  readonly capabilities: CarrierCapabilities;

  createBooking(req: CanonicalBookingRequest, ctx: AdapterContext): Promise<AdapterResult<BookingAck>>;
  getBookingStatus(carrierRef: string, ctx: AdapterContext): Promise<AdapterResult<BookingStatusView>>;
  cancelBooking(carrierRef: string, reason: string | undefined, ctx: AdapterContext): Promise<AdapterResult<CancelAck>>;
  getTracking(q: TrackingQuery, ctx: AdapterContext): Promise<AdapterResult<NormalizedEvent[]>>;

  toCanonicalStatus(carrierStatus: string): CanonicalBookingStatus;
  handleWebhook?(raw: RawInbound, ctx: AdapterContext): Promise<NormalizedInbound>;  // v2 seam, unused in v1
}

interface CarrierCapabilities {
  supportsCancel: boolean;
  trackingByBooking: boolean;
  trackingByEquipment: boolean;
  supportsWebhook: boolean;
}

type AdapterResult<T> =
  | { ok: true; data: T; raw: unknown }
  | { ok: false; error: AdapterError; retryable: boolean; raw?: unknown };

interface AdapterError {
  category: 'AUTH' | 'VALIDATION' | 'NOT_FOUND' | 'RATE_LIMIT'
          | 'CARRIER_UNAVAILABLE' | 'TIMEOUT' | 'UNKNOWN';
  message: string;               // carrier message surfaced to the UI where safe
  carrierCode?: string;
}

interface BookingAck {
  carrierBookingRef?: string;    // some carriers assign on ack, some later
  acceptedStatus: 'SUBMITTED' | 'RECEIVED';
}

interface CancelAck {
  accepted: boolean;
  effectiveStatus?: 'CANCELLATION_REQUESTED' | 'CANCELLED';
}
```

`AdapterContext` is built per call by the domain layer and injected:

```ts
interface AdapterContext {
  http: HttpClient;              // bound with base URL, auth, retry, breaker, carrier_messages logging
  credential: ResolvedCredential;
  logger: Logger;               // carries correlationId
  clock: () => Date;
}
```

`packages/carrier-core` provides:

- **`HttpClient` factory** — base URL, auth strategy, timeout, retry with
  exponential backoff + jitter (only on `retryable`), per-carrier circuit breaker
  (opens after `CIRCUIT_FAIL_THRESHOLD` consecutive failures, half-open probe),
  and a hook that writes every exchange to `carrier_messages` with the
  correlation id (secrets redacted).
- **Auth strategies**:
  - `OAuth2ClientCredentials` — token endpoint, client id/secret, in-memory token
    cache with expiry and refresh; supports an extra static header (Maersk
    `Consumer-Key`).
  - `ApiKeyHeader` — header name + value (ONE, HMM).
  - `JwtBearer` — signs a JWT with the configured key/claims per request or
    caches until expiry (Evergreen).
- **`ContractTestKit`** — given an adapter and a base URL, runs a standard suite:
  `create -> poll status to CONFIRMED -> getTracking -> cancel`, asserting the
  canonical output shapes and that `toCanonicalStatus` covers every state the
  mock emits. Reused by each adapter test and later re-pointed at the real
  sandbox.

Adapter registry: `apps/booking/src/adapters/index.ts` exports a
`Map<string, CarrierAdapter>` keyed by carrier code. The domain layer resolves an
adapter by `booking.carrier_code`.

## 9. Async processing

### Flow

```
POST /bookings
  validate (zod)
  insert bookings (status = PENDING_SUBMISSION)
  insert booking_events (event_type = NOTE, source = user)
  enqueue submit_booking(bookingId)
  return 202

worker: submit_booking(bookingId)
  guard: booking.status == PENDING_SUBMISSION  (idempotent; skip otherwise)
  build AdapterContext (resolve credential, http client)
  adapter.createBooking(canonical_request)
    ok:
      reconcile(bookingId, ack.acceptedStatus, source=adapter, raw)
      store carrier_booking_ref if present
      enqueue poll_booking_status(bookingId) with delay = POLL_INTERVAL_BOOKING_MIN
    err && retryable:
      throw -> graphile-worker retries with backoff, max attempts N
      on final failure: reconcile(bookingId, SUBMISSION_FAILED, source=system, error)
    err && !retryable:
      reconcile(bookingId, SUBMISSION_FAILED, source=adapter, error)
      booking_events ERROR with carrier validation message

cron every POLL_INTERVAL_BOOKING_MIN:
  enqueue poll_booking_status(bookingId) for every booking whose status is in the
  active set: { SUBMITTED, RECEIVED, PENDING_CONFIRMATION, CANCELLATION_REQUESTED }

worker: poll_booking_status(bookingId)
  adapter.getBookingStatus(carrier_booking_ref)
  observed = adapter.toCanonicalStatus(view.carrierStatus)
  reconcile(bookingId, observed, source=poller, raw)
  if observed == CONFIRMED: enqueue poll_tracking(bookingId)
  if observed in { DECLINED, CANCELLED }: booking leaves the active set naturally

cron every POLL_INTERVAL_TRACKING_MIN:
  enqueue poll_tracking(bookingId) for bookings with status CONFIRMED
  (a stop condition once cargo is delivered is a later concern; v1 keeps polling
   confirmed bookings until an operator archives them)

worker: poll_tracking(bookingId)
  adapter.getTracking({ carrierRef | equipmentRefs })
  upsert shipment_events by dedupe_key (existing rows untouched)
  tracking does NOT change booking status in v1

POST /bookings/:id/cancel
  guard: status in cancelable set { SUBMITTED, RECEIVED, PENDING_CONFIRMATION,
         CONFIRMED } AND carrier.capabilities.supportsCancel
  reconcile(bookingId, CANCELLATION_REQUESTED, source=user)
  enqueue submit_cancellation(bookingId)

worker: submit_cancellation(bookingId)
  adapter.cancelBooking(carrier_booking_ref, reason)
    ok: reconcile(bookingId, ack.effectiveStatus ?? CANCELLATION_REQUESTED, source=adapter)
        (final CANCELLED may be confirmed by the next poll)
    err: reconcile(bookingId, CANCELLATION_FAILED, source=adapter, error)
```

### The reconcile choke point

`domain/statusReconciler.ts` exposes:

```ts
reconcile(bookingId: string, observed: CanonicalBookingStatus,
          source: EventSource, raw?: unknown): Promise<void>
```

It is the **only** module that writes `bookings.status` and appends
`booking_events`. It:

- loads current status, applies the allowed-transition table (illegal
  transitions are logged and dropped, not applied),
- on a real change: updates `bookings.status`, `carrier_status_raw`,
  `last_carrier_sync_at`, appends a `STATUS_CHANGE` event with `from`/`to`/
  `source`/`correlationId`,
- on no change: updates `last_carrier_sync_at` only.

Submit results, poll results, cancel results and (later) webhook handlers all go
through `reconcile`. Mirrors Greenlit's "one layer decides" rule.

### Idempotency and reliability

- `submit_booking` and `submit_cancellation` guard on current status; re-running
  is a no-op.
- `poll_tracking` upserts by `dedupe_key`.
- A job that exhausts retries writes an `ERROR` `booking_events` row, logs, and
  bumps a metric. The booking is visibly parked in `SUBMISSION_FAILED` /
  `CANCELLATION_FAILED`, never silently lost (Greenlit §82, §83).

## 10. Per-carrier adapters

Directory per adapter:

```
apps/booking/src/adapters/<carrier>/
  index.ts        implements CarrierAdapter
  mapRequest.ts   canonical -> carrier payload
  mapStatus.ts    carrier status string -> CanonicalBookingStatus
  mapTracking.ts  carrier events -> NormalizedEvent[]
  auth.ts         selects a carrier-core auth strategy + carrier header quirks
  types.ts        carrier wire types, hand-written from docs
  README.md       endpoints, quirks, open questions pending sandbox
  __tests__/      unit tests for the map* functions + ContractTestKit vs the mock
```

Known quirks (contained inside the adapter, never leaked to the gateway):

| Carrier | SCAC | Auth | Notes |
|---|---|---|---|
| ONE | ONEY | API key header | booking + T&T reasonably close to DCSA 2.0; rate limits apply |
| Maersk | MAEU | OAuth2 client-credentials **plus `Consumer-Key` header** | separate sandbox host; DCSA-aligned; tracking via Track & Trace events API |
| HMM | HDMU | API key | portal and some docs in Korean; onboarding needs a Korean business registration number (management blocker); partial DCSA adoption, booking payload may be proprietary |
| Evergreen | EGLV | signed JWT **plus mandatory IP allowlist** | proprietary payloads (not DCSA); separate booking vs SI portals |

Each adapter's `README.md` records open questions to resolve when its sandbox
comes online; contract tests are re-pointed at the sandbox at that time.

## 11. Mock carrier servers

`apps/mock-carriers` — one small HTTP app per carrier (or one app, routes
namespaced by carrier). Dev and test only; never deployed to production.

Each mock:

- accepts the carrier-shaped booking request and returns a carrier-shaped ack
  with a fake booking ref,
- `GET status` returns a scripted progression (`RECEIVED -> PENDING_CONFIRMATION
  -> CONFIRMED`) driven by elapsed time or a control endpoint,
- `GET tracking` returns a scripted event list,
- `cancel` flips the booking to cancelled,
- simulates error modes via `x-mock-scenario: auth-fail | validation-fail |
  timeout | rate-limit | carrier-500`,
- performs a stubbed but present auth check (expects the right header/token shape
  so auth wiring is exercised).

`.env` switches each adapter's base URL between the mock and (later) the sandbox.

## 12. Testing strategy

- **Unit** — `mapRequest` / `mapStatus` / `mapTracking` per adapter against
  fixture payloads taken from published carrier docs and DCSA samples.
- **Contract** — `ContractTestKit` runs the standard `create -> status ->
  tracking -> cancel` suite against each mock server. When a sandbox arrives, the
  same kit runs against it and any drift is immediate.
- **Domain lifecycle** — full flow (create -> poll -> confirm -> cancel) using a
  `FakeAdapter` against a real Postgres (test database), asserting `bookings`,
  `booking_events`, `shipment_events` end state.
- **API integration** — Fastify `inject`, covering RBAC roles, idempotency,
  validation errors, `/health`.
- **HttpClient** — retry, circuit breaker open/half-open, OAuth2 token cache and
  refresh.
- **edifact** — IFTMBF reference round-trip (`toEdifact` then `fromEdifact`).

CI runs unit + contract + domain + API on every PR with an ephemeral Postgres.

## 13. Credential store and security

```ts
interface CredentialStore {
  get(carrierCode: string, accountRef?: string): Promise<ResolvedCredential>;
  put(input: PutCredentialInput): Promise<void>;
  rotate(carrierCode: string): Promise<void>;
  list(): Promise<CredentialHealth[]>;
}
```

v1 implementation `PgCredentialStore`:

- one row per `(carrier_code, account_ref)` in `carrier_credentials`,
- `secret_encrypted` = libsodium `crypto_secretbox` of a JSON blob whose fields
  depend on `auth_type` (client id/secret + token URL, or api key, or JWT signing
  key + claims),
- master key from env `CARRIER_CRED_MASTER_KEY`; `key_version` supports rotating
  the master key,
- decrypt only in memory while building `AdapterContext`; never logged, never in
  `carrier_messages`,
- `PUT /carriers/:code/credentials` (admin) writes it and appends a
  `credential_events` row,
- `GET /carriers` reports `configured`, `lastAuthOk`, `circuitState` without
  exposing secret material.

Other security (Greenlit §79, §80):

- RBAC with server-side permission checks; frontend checks are cosmetic only.
- Encrypted transport (TLS) everywhere.
- Secrets never committed; `.env.example` lists keys with placeholder values.
- `carrier_messages` request bodies are redacted of auth headers and secret
  fields before storage.
- Soft delete / archive for operational records; no hard deletes.
- Timestamps stored UTC, displayed `Asia/Singapore` (Greenlit §78).

## 14. `packages/edifact` seam

Skeleton only in v1.

```
packages/edifact/
  src/
    index.ts        export { toEdifact, fromEdifact, MessageType }
    types.ts        MessageType = 'IFTMBF' | 'IFTMBC' | 'IFTSTA' | 'CODECO'
                                | 'CONTRL' | 'APERAK'
                    (booking cancellation is IFTMBF carrying a cancellation
                     message-function code in BGM, not a distinct message type)
    segments.ts     minimal UN/EDIFACT segment builder/parser primitives
                    (UNH, BGM, DTM, LOC, EQD, NAD, RFF, ...)
    messages/
      iftmbf.ts        working reference: CanonicalBookingRequest -> IFTMBF interchange
      iftmbf.parse.ts  working reference: IFTMBF -> partial canonical
      iftmbc.ts        stub: throws NotImplemented, doc comment describes intent
      iftsta.ts        stub
      codeco.ts        stub
  __tests__/iftmbf.test.ts    round-trips the reference sample
  README.md        scope note: this is a seam. Full message set + edi-transport
                   (SFTP/AS2) is a later spec.
```

`toEdifact(canonical, type)` and `fromEdifact(raw): { type, canonicalPartial }`
are pure, no I/O. A future `HYBRID` adapter imports this plus a future
`packages/edi-transport` (not built in v1).

## 15. Frontend (`apps/booking/web`)

Vite + React 19 + TypeScript. Motus shell patterns are copied and adapted, not
imported. TanStack Query against the internal API. No business logic: the UI
renders `status` from the API and never derives it.

Screens:

1. **Bookings list** (default) — table: carrier, booking ref, POL -> POD,
   vessel/voyage, requested ETD, status chip, updated. Filters: status, carrier,
   date range, customer ref. Pagination.
2. **New booking** — form for the canonical request: carrier select (only
   `enabled` and `configured` carriers from `/carriers`), parties, commodities
   (repeatable), equipment (repeatable), transport (vessel/voyage/POL/POD/ETD),
   references, flags. Client-side zod validation mirrors the server. Submit ->
   202 -> redirect to detail.
3. **Booking detail** — header (status chip, carrier ref, key fields). Tabs:
   Overview (canonical request), Timeline (`booking_events`), Tracking
   (`shipment_events` grouped by equipment), Carrier messages (admin only, raw
   exchange). Actions: Cancel (guarded by status + capability), Refresh now.
4. **Carriers / settings** (admin) — registry list, capability flags, credential
   health, set/rotate credential form.

Auth: `packages/auth` client (session/JWT, route guard by role). Local users in
v1, mirroring the Greenlit plan. Status chip and table shell are candidates for
`packages/ui` later, not now.

## 16. Observability, config, error handling

- **Structured logging** (pino): every job and carrier exchange logs
  `{ correlationId, bookingId, carrierCode, operation, outcome, latencyMs }`. The
  correlation id is generated at API ingress and threaded through job payloads
  into `AdapterContext`.
- **Log events** (Greenlit §84): `booking.created`, `booking.submitted`,
  `booking.submit_failed`, `booking.status_changed`, `booking.cancel_requested`,
  `booking.cancelled`, `tracking.events_ingested`, `carrier.auth_failed`,
  `carrier.circuit_opened`, `credential.rotated`.
- **No silent failure** (Greenlit §83): failed jobs surface as `ERROR`
  `booking_events` rows plus logs plus a metric.
- **Health**: `GET /health` checks DB, worker heartbeat, per-carrier circuit
  state and `lastAuthOk`.
- **Config** via `.env` per app, `.env.example` committed:
  `DATABASE_URL`, `CARRIER_CRED_MASTER_KEY`, `POLL_INTERVAL_BOOKING_MIN`,
  `POLL_INTERVAL_TRACKING_MIN`, `HTTP_TIMEOUT_MS`, `CIRCUIT_FAIL_THRESHOLD`,
  `<CARRIER>_BASE_URL` per carrier, `LOG_LEVEL`. No secrets in code.
- **Migrations**: `node-pg-migrate` SQL files run on deploy and in test setup.

## 17. Build phases (decomposition)

Roughly eight PR-sized chunks in dependency order.

| # | Chunk | Acceptance |
|---|---|---|
| 1 | Monorepo skeleton: pnpm workspaces, `tsconfig.base`, shared `config/`, empty `packages/{carrier-core,edifact,db,auth}`, `apps/booking` scaffold (Fastify + Vite), CI (lint + typecheck + test), `.env.example` | `pnpm i && pnpm build && pnpm test` green on the empty scaffold |
| 2 | `packages/db` + schema: migration runner, connection pool, all tables (`carriers`, `carrier_credentials`, `bookings`, `booking_events`, `shipment_events`, `carrier_messages`, `credential_events`), `graphile-worker` install, carrier seed | migrate up/down clean; generated types compile |
| 3 | `packages/carrier-core`: `CarrierAdapter` interface, canonical types + zod schemas, `HttpClient` (auth strategies, retry, circuit breaker, `carrier_messages` hook), `AdapterResult`/`AdapterError`, `ContractTestKit` | unit tests for HttpClient (retry, breaker, OAuth2 token cache) + a `FakeAdapter` passes the kit |
| 4 | Domain + async core: `bookingService`, `statusReconciler` (choke point + transition table), worker tasks (`submit_booking`, `poll_booking_status`, `poll_tracking`, `submit_cancellation`), cron enqueuers, `CredentialStore` + `PgCredentialStore` | full lifecycle test with `FakeAdapter` against a real Postgres |
| 5 | Internal API: all routes, zod validation, RBAC middleware (`packages/auth` thin), idempotency, `/health` | API integration tests covering roles, idempotency, validation |
| 6 | Four carrier adapters + mock servers: `apps/mock-carriers`, then ONE, Maersk, HMM, Evergreen (each can be a sub-PR) | each adapter green against its mock via `ContractTestKit`; end-to-end via the API with mock base URLs |
| 7 | Frontend: shell + four screens, TanStack Query, auth guard | component tests + a full manual run of create / cancel / track against the mock carriers |
| 8 | `packages/edifact` seam: skeleton + working IFTMBF reference + round-trip test | round-trip test green; independent, can land any time after chunk 1 |

Chunks 1 to 5 are the spine and mostly sequential. Chunk 6 parallelizes four
ways. Chunks 7 and 8 are independent of each other.

## 18. Open questions and blockers for management

Carried from the 7-carrier analysis, still to resolve outside engineering:

1. **Carrier credential applications** are in progress for all four. Nothing can
   be tested end-to-end against real carriers until at least one sandbox is
   granted. Build proceeds against mocks in the meantime.
2. **HMM onboarding** appears to require a Korean business registration number
   and involves a partly Korean-language portal. Confirm whether ZHL can satisfy
   this directly or needs a local partner.
3. **Evergreen IP allowlist** — the static egress IP is available; it must be
   registered with Evergreen during onboarding.
4. **Legal review** of each carrier's API terms and conditions before go-live.
5. **Exception ownership** — who in operations owns bookings parked in
   `SUBMISSION_FAILED` / `CANCELLATION_FAILED`, and the response-time
   expectation.
6. **Build-direct vs aggregator** — this spec builds direct adapters. If an
   aggregator is later preferred for some carriers, the adapter seam still holds
   but the decision affects cost and timeline.

## 19. Handoff

Implementation runs through the **mega-sdd** factory track:
`bind-codebase` -> `generate-units` (bolts roughly matching the eight chunks in
Section 17) -> `execute-bolts` with the review panel.

This spec is the input to `bind-codebase`.
