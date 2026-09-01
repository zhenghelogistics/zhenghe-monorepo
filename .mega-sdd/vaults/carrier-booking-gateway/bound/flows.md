---
type: prose
doc_id: flows
vault_version: "1.0"
aliases: [Flows, User Flows, System Flows]
tags: ["vault/carrier-booking-gateway", "doc/flows"]
---

# Flows

> **TL;DR**: operator flows (create / cancel / credential admin), worker flows (submit / poll status / poll tracking / submit cancellation), and one end-to-end lifecycle · developers per layer + QA · read when building or testing a feature.

> Flow types for `web-app`: **User flows (web)**, **Backend / system flows**, **Cross-cutting flows**. All carrier calls happen inside worker jobs; the API never calls a carrier during a request (spec §9).

## User flows (web)

### F-U-001: Create a booking
<!-- BIND: confirmed=C-FL-01 -->

**Actor / Trigger**: operations controller submits the New booking form.

```mermaid
flowchart TD
    S1["Controller fills the canonical booking form (carrier, parties, commodities, equipment, transport, references)"] --> S2["Frontend validates with the shared zod schema"]
    S2 --> D1{"Valid?"}
    D1 -- "no" --> E1(["Show field errors, stay on form"])
    D1 -- "yes" --> S3["POST /bookings with optional Idempotency-Key"]
    S3 --> D2{"Idempotency-Key already used by this created_by?"}
    D2 -- "yes" --> S4["Return the existing booking (no new row)"]
    D2 -- "no" --> S5["Insert bookings row status=PENDING_SUBMISSION"]
    S5 --> S6["Append booking_events NOTE (source=user)"]
    S6 --> S7["Enqueue submit_booking(bookingId)"]
    S7 --> S8["202 {bookingId, status: PENDING_SUBMISSION}"]
    S4 --> S9["Redirect to Booking detail"]
    S8 --> S9
```

**Definition of Done**:
- [ ] A valid submit creates exactly one `bookings` row with `status = PENDING_SUBMISSION` and one `booking_events` NOTE row.
- [ ] A `submit_booking` job is enqueued for that booking id.
- [ ] Re-POST with the same `Idempotency-Key` and `created_by` returns the first booking and creates no second row or job.
- [ ] Invalid payload returns 4xx with field-level errors and creates nothing.
- [ ] Response is `202` with `{ bookingId, status: "PENDING_SUBMISSION" }`; the UI lands on the detail screen.

**Source**: PRD §7, §9, §15.

---

### F-U-002: Cancel a booking
<!-- BIND: confirmed=C-FL-02 -->

**Actor / Trigger**: operations controller clicks Cancel on Booking detail.

```mermaid
flowchart TD
    S1["Controller clicks Cancel (optional reason)"] --> S2["POST /bookings/:id/cancel"]
    S2 --> D1{"status in cancelable set {SUBMITTED, RECEIVED, PENDING_CONFIRMATION, CONFIRMED} AND carrier.capabilities.supportsCancel?"}
    D1 -- "no" --> E1(["409 / disabled button — cancel not allowed"])
    D1 -- "yes" --> S3["reconcile(id, CANCELLATION_REQUESTED, source=user)"]
    S3 --> S4["Append booking_events CANCEL_REQUESTED"]
    S4 --> S5["Enqueue submit_cancellation(bookingId)"]
    S5 --> S6["202 {status: CANCELLATION_REQUESTED}"]
```

**Definition of Done**:
- [ ] Cancel is offered only when the status is in the cancelable set and the carrier supports cancel.
- [ ] A successful request moves the booking to `CANCELLATION_REQUESTED`, appends a `CANCEL_REQUESTED` event, and enqueues `submit_cancellation`.
- [ ] A disallowed cancel changes no state and returns a clear error.

**Source**: PRD §7, §9.

---

### F-U-003: Set or rotate a carrier credential
<!-- BIND: confirmed=C-FL-03 -->

**Actor / Trigger**: platform administrator submits the credential form on the Carriers / settings screen.

```mermaid
flowchart TD
    S1["Admin enters authType, secret fields, baseUrl (optional accountRef)"] --> S2["PUT /carriers/:code/credentials (admin only)"]
    S2 --> D1{"Caller has admin role?"}
    D1 -- "no" --> E1(["403"])
    D1 -- "yes" --> S3["Encrypt the secret blob with libsodium crypto_secretbox (master key from env)"]
    S3 --> S4["Upsert carrier_credentials row; set is_active, bump key_version on rotate"]
    S4 --> S5["Append credential_events (action=set|rotate, actor)"]
    S5 --> S6["204"]
    S6 --> S7["GET /carriers now shows configured=true, circuitState, lastAuthOk"]
```

**Definition of Done**:
- [ ] Only an `admin` can call the endpoint; others get 403.
- [ ] The stored `secret_encrypted` is ciphertext; the plaintext never appears in logs or `carrier_messages`.
- [ ] Exactly one active credential row exists per carrier after the write (partial unique index holds).
- [ ] A `credential_events` row is appended with the actor and action.
- [ ] `GET /carriers` reflects the new credential health without exposing secret material.

**Source**: PRD §7, §13.

## Backend / system flows

### F-S-001: submit_booking worker job
<!-- BIND: confirmed=C-FL-04 -->

**Trigger**: `submit_booking(bookingId)` dequeued.
**Inputs**: the `bookings` row, the resolved carrier adapter, a built `AdapterContext` (decrypted credential, bound HTTP client).

```mermaid
flowchart TD
    T(["submit_booking(bookingId)"]) --> D0{"booking.status == PENDING_SUBMISSION?"}
    D0 -- "no" --> X1(["Skip — idempotent no-op"])
    D0 -- "yes" --> S1["adapter.createBooking(canonical_request, ctx)"]
    S1 --> D1{"AdapterResult"}
    D1 -- "ok" --> S2["reconcile(id, ack.acceptedStatus SUBMITTED|RECEIVED, source=adapter); store carrier_booking_ref"]
    S2 --> S3["Enqueue poll_booking_status(bookingId) with delay"]
    D1 -- "err && retryable" --> S4["Throw — graphile-worker retries with backoff (max N)"]
    S4 --> D2{"attempts exhausted?"}
    D2 -- "no" --> T
    D2 -- "yes" --> S5["reconcile(id, SUBMISSION_FAILED, source=system); booking_events ERROR"]
    D1 -- "err && non-retryable" --> S6["reconcile(id, SUBMISSION_FAILED, source=adapter); booking_events ERROR with carrier message"]
```

**Outputs**: updated `bookings.status`, `carrier_booking_ref`, `booking_events` rows; a `carrier_messages` row for the exchange; possibly a `poll_booking_status` job.

**Definition of Done**:
- [ ] Re-running the job when status is no longer `PENDING_SUBMISSION` does nothing.
- [ ] On carrier ack the booking moves to `SUBMITTED` or `RECEIVED`, `carrier_booking_ref` is stored, and a status poll is scheduled.
- [ ] A retryable error retries with backoff up to the configured max, then lands in `SUBMISSION_FAILED` with an `ERROR` event.
- [ ] A non-retryable error lands in `SUBMISSION_FAILED` immediately, surfacing the carrier validation message.
- [ ] Every attempt writes one `carrier_messages` row with the correlation id, secrets redacted.

**Source**: PRD §8, §9, §16. Retry max N — OQ-CN-7.

---

### F-S-002: poll_booking_status (cron + worker)
<!-- BIND: confirmed=C-FL-05 -->

**Trigger**: cron every `POLL_INTERVAL_BOOKING_MIN` enqueues one job per booking in the active set; also enqueued on demand by `POST /bookings/:id/refresh`.

```mermaid
flowchart TD
    C(["cron: for each booking in active set {SUBMITTED, RECEIVED, PENDING_CONFIRMATION, CANCELLATION_REQUESTED}"]) --> Q["Enqueue poll_booking_status(bookingId)"]
    Q --> W(["worker: poll_booking_status(bookingId)"])
    W --> S1["adapter.getBookingStatus(carrier_booking_ref, ctx)"]
    S1 --> S2["observed = adapter.toCanonicalStatus(view.carrierStatus)"]
    S2 --> S3["reconcile(id, observed, source=poller, raw)"]
    S3 --> D1{"observed"}
    D1 -- "CONFIRMED" --> S4["Enqueue poll_tracking(bookingId)"]
    D1 -- "DECLINED | CANCELLED" --> S5["Booking leaves the active set naturally"]
    D1 -- "other" --> S6["Update last_carrier_sync_at only if no change"]
```

**Definition of Done**:
- [ ] A changed carrier status is written via `reconcile` with `source = poller` and a `STATUS_CHANGE` event; `last_carrier_sync_at` is updated.
- [ ] An unchanged status updates only `last_carrier_sync_at`.
- [ ] Reaching `CONFIRMED` enqueues `poll_tracking`.
- [ ] `DECLINED` / `CANCELLED` drop out of the polled set.
- [ ] An illegal transition reported by the carrier is logged and dropped, not applied.

**Source**: PRD §9. Poll interval value — OQ-CN-6.

---

### F-S-003: poll_tracking (cron + worker)
<!-- BIND: confirmed=C-FL-06 -->

**Trigger**: cron every `POLL_INTERVAL_TRACKING_MIN` enqueues one job per booking with status `CONFIRMED`; also enqueued after a booking reaches `CONFIRMED` and by `POST /bookings/:id/refresh`.

```mermaid
flowchart TD
    C(["cron: for each booking with status CONFIRMED"]) --> Q["Enqueue poll_tracking(bookingId)"]
    Q --> W(["worker: poll_tracking(bookingId)"])
    W --> S1["adapter.getTracking({carrierRef | equipmentRefs}, ctx)"]
    S1 --> S2["For each NormalizedEvent: upsert shipment_events by dedupe_key"]
    S2 --> D1{"dedupe_key already present?"}
    D1 -- "yes" --> S3["Leave the existing row untouched"]
    D1 -- "no" --> S4["Insert the event row"]
    S4 --> S5["Log tracking.events_ingested with the count"]
    S3 --> S5
```

**Definition of Done**:
- [ ] New tracking events are inserted; events whose `dedupe_key` already exists are ignored.
- [ ] Tracking never changes `bookings.status` in v1.
- [ ] Each poll writes one `carrier_messages` row and logs the ingested count.

**Source**: PRD §9. No delivered-state stop condition — OQ-FL-1.

---

### F-S-004: submit_cancellation worker job
<!-- BIND: confirmed=C-FL-07 -->

**Trigger**: `submit_cancellation(bookingId)` dequeued after F-U-002.

```mermaid
flowchart TD
    T(["submit_cancellation(bookingId)"]) --> S1["adapter.cancelBooking(carrier_booking_ref, reason, ctx)"]
    S1 --> D1{"AdapterResult"}
    D1 -- "ok" --> S2["reconcile(id, ack.effectiveStatus ?? CANCELLATION_REQUESTED, source=adapter)"]
    S2 --> S3["Final CANCELLED confirmed by the next poll_booking_status if not immediate"]
    D1 -- "err" --> S4["reconcile(id, CANCELLATION_FAILED, source=adapter); booking_events ERROR"]
```

**Definition of Done**:
- [ ] A successful cancel moves the booking to `CANCELLED` (or stays `CANCELLATION_REQUESTED` until a poll confirms).
- [ ] A failed cancel moves the booking to `CANCELLATION_FAILED` with an `ERROR` event and a carrier message.
- [ ] The job is safe to re-run (guarded by current status).

**Source**: PRD §9.

## Cross-cutting flows

### F-C-001: End-to-end booking lifecycle
<!-- BIND: confirmed=C-FL-08 -->

**Actor**: operations controller.
**Layers involved**: Web Frontend, Backend API, Backend worker, Integrations (adapter + `carrier-core`), Carrier (mock in v1).

```mermaid
flowchart TD
    subgraph FE["Web Frontend"]
        A1["New booking form"] --> A2["Booking detail — polls GET /bookings/:id"]
    end
    subgraph API["Backend API"]
        B1["POST /bookings → insert PENDING_SUBMISSION, enqueue submit_booking"]
        B2["GET /bookings/:id → canonical record + events + tracking"]
    end
    subgraph WRK["Backend worker"]
        C1["submit_booking → adapter.createBooking"]
        C2["poll_booking_status → adapter.getBookingStatus → reconcile"]
        C3["poll_tracking → adapter.getTracking → upsert shipment_events"]
    end
    subgraph INT["Integrations"]
        D1["carrier adapter + carrier-core HttpClient (auth, retry, breaker, carrier_messages)"]
    end
    subgraph CAR["Carrier (mock in v1)"]
        E1["REST endpoints: create / status / tracking / cancel"]
    end
    A1 -- "canonical booking request (HTTP JSON)" --> B1
    B1 -- "job (Postgres queue)" --> C1
    C1 -- "carrier-shaped request" --> D1
    D1 -- "HTTPS" --> E1
    E1 -- "ack + carrier booking ref" --> D1
    D1 -- "AdapterResult" --> C1
    C1 -- "reconcile()" --> C2
    C2 -- "status changes" --> C3
    C3 -- "shipment_events" --> B2
    B2 -- "JSON" --> A2
```

**Definition of Done**:
- [ ] A booking created in the UI reaches `CONFIRMED` in the UI without any carrier portal login, driven entirely by worker polling against the mock.
- [ ] Every handoff (form → API → queue → adapter → carrier → reconcile → API → UI) succeeds on the happy path.
- [ ] A carrier failure at the adapter hop surfaces as a `SUBMISSION_FAILED` booking visible in the UI, not a lost request.
- [ ] Tracking events appear on the detail screen after the booking is `CONFIRMED`.
- [ ] All carrier exchanges are recorded in `carrier_messages` with correlation ids.

**Source**: PRD §7, §8, §9, §11, §15.

---

## Sources

- PRD §7 (Internal API surface), §8 (CarrierAdapter interface), §9 (Async processing), §11 (Mock carrier servers), §15 (Frontend), §16 (Observability)

## Out of Scope

- Webhook-driven status ingestion (seam only — spec §5, §8).
- Shipping Instruction, amendment, schedule-search flows (v2 — spec §2).
- Bulk create / bulk cancel (spec §2).
