---
type: prose
doc_id: model
vault_version: "1.0"
aliases: [Data Model, DBML, Schema]
tags: ["vault/carrier-booking-gateway", "doc/data-model"]
---

# Data Model

> **TL;DR**: Postgres schema for the booking gateway — carriers, encrypted credentials, canonical bookings, append-only event/audit tables, tracking events, raw carrier message log · BE developer, DBA · read when writing a migration or a query.

> Hand-written SQL via `pg`; migrations are SQL files run by `node-pg-migrate` (spec §4, §6). No ORM. `graphile-worker` manages its own tables via its installer — not modelled here.

## Entities (DBML)

```dbml
// Purpose: registry of supported carriers and their capabilities; the gateway will not route to a disabled carrier
<!-- BIND: confirmed=C-DM-01 -->
Table carriers {
  code text [pk, note: 'SCAC: ONEY, MAEU, HDMU, EGLV']
  display_name text [not null]
  transport_mode text [not null, note: 'REST | EDIFACT | HYBRID']
  dcsa_booking_version text [note: 'e.g. 2.0.0; null if proprietary']
  capabilities jsonb [not null, note: '{ supportsCancel, trackingByBooking, trackingByEquipment, supportsWebhook }']
  enabled boolean [not null, default: true]
  created_at timestamptz [not null, default: `now()`]
  updated_at timestamptz [not null, default: `now()`]
}

// Purpose: encrypted per-carrier credentials; one active row per carrier in v1, account_ref present for future multi-account
<!-- BIND: confirmed=C-DM-02 -->
Table carrier_credentials {
  id uuid [pk, default: `gen_random_uuid()`]
  carrier_code text [not null, ref: > carriers.code]
  account_ref text [not null, note: 'v1: one row per carrier; column reserved for multi-account']
  auth_type text [not null, note: 'oauth2 | api_key | jwt']
  secret_encrypted bytea [not null, note: 'libsodium crypto_secretbox of a JSON blob; fields depend on auth_type']
  key_version int [not null, default: 1, note: 'for master-key rotation']
  base_url text [not null, note: 'mock URL in dev, sandbox/prod URL later']
  is_active boolean [not null, default: true]
  rotated_at timestamptz
  updated_by text [not null]
  created_at timestamptz [not null, default: `now()`]
  updated_at timestamptz [not null, default: `now()`]

  indexes {
    (carrier_code) [unique, name: 'uq_carrier_credentials_active', note: 'partial: WHERE is_active']
  }
}

// Purpose: canonical booking record; ZHL source of truth for internal state, carrier is system-of-record for confirmation
<!-- BIND: confirmed=C-DM-03 -->
Table bookings {
  id uuid [pk, default: `gen_random_uuid()`]
  carrier_code text [not null, ref: > carriers.code]
  carrier_account_ref text [not null]
  status text [not null, note: 'canonical status enum: DRAFT|PENDING_SUBMISSION|SUBMITTED|RECEIVED|PENDING_CONFIRMATION|CONFIRMED|DECLINED|CANCELLATION_REQUESTED|CANCELLED|CANCELLATION_FAILED|SUBMISSION_FAILED']
  carrier_booking_ref text [note: 'assigned by the carrier after submit/ack; nullable']
  canonical_request jsonb [not null, note: 'full canonical booking request (trimmed DCSA Booking 2.0 subset)']
  pol text [note: 'UN/LOCODE, denormalized from canonical_request for filtering']
  pod text [note: 'UN/LOCODE, denormalized']
  vessel_name text
  voyage_number text
  requested_departure_date date
  customer_reference text [note: 'denormalized from references[] where type = CR']
  carrier_status_raw text [note: 'last raw carrier status string']
  idempotency_key text [note: 'from the Idempotency-Key header; nullable']
  created_by text [not null]
  last_carrier_sync_at timestamptz [note: 'last successful poll']
  created_at timestamptz [not null, default: `now()`]
  updated_at timestamptz [not null, default: `now()`]

  indexes {
    status
    (carrier_code, status)
    requested_departure_date
    (created_by, idempotency_key) [unique, name: 'uq_bookings_idempotency', note: 'partial: WHERE idempotency_key IS NOT NULL']
  }
}

// Purpose: append-only audit timeline for a booking; every status change and error is one row
<!-- BIND: confirmed=C-DM-04 -->
Table booking_events {
  id uuid [pk, default: `gen_random_uuid()`]
  booking_id uuid [not null, ref: > bookings.id]
  seq bigint [not null, note: 'monotonically increasing per booking']
  event_type text [not null, note: 'STATUS_CHANGE | SUBMITTED | CANCEL_REQUESTED | ERROR | NOTE']
  from_status text
  to_status text
  source text [not null, note: 'adapter | poller | user | webhook | system']
  actor text [not null, note: 'user id or "system"']
  detail jsonb [note: 'error category, carrier message, correlation id, etc.']
  created_at timestamptz [not null, default: `now()`]

  indexes {
    (booking_id, seq) [unique]
  }
}

// Purpose: normalized tracking events (shipment / transport / equipment), upserted by dedupe_key
<!-- BIND: confirmed=C-DM-05 -->
Table shipment_events {
  id uuid [pk, default: `gen_random_uuid()`]
  booking_id uuid [ref: > bookings.id, note: 'nullable: event may arrive before the link is known']
  carrier_code text [not null]
  dedupe_key text [not null, unique, note: 'carrier event id if available, else a stable hash of the event fields']
  event_type text [not null, note: 'SHIPMENT | TRANSPORT | EQUIPMENT']
  classifier text [not null, note: 'PLN | EST | ACT']
  event_code text
  description text
  event_date_time timestamptz
  location_unloc text
  facility text
  equipment_reference text [note: 'container number']
  transport_vessel text
  transport_voyage text
  transport_port text
  raw jsonb [not null]
  retrieved_at timestamptz [not null, default: `now()`]
  created_at timestamptz [not null, default: `now()`]

  indexes {
    booking_id
    equipment_reference
  }
}

// Purpose: raw request/response log for every carrier exchange; audit + debugging when sandboxes go live
<!-- BIND: confirmed=C-DM-06 -->
Table carrier_messages {
  id uuid [pk, default: `gen_random_uuid()`]
  booking_id uuid [ref: > bookings.id, note: 'nullable (e.g. auth calls)']
  carrier_code text [not null]
  direction text [not null, note: 'out | in']
  operation text [not null, note: 'create | status | cancel | track | auth']
  http_status int
  request_body jsonb [note: 'redacted of auth headers and secret fields before storage']
  response_body jsonb
  latency_ms int
  error_text text
  correlation_id text
  created_at timestamptz [not null, default: `now()`]

  indexes {
    (carrier_code, created_at)
    booking_id
  }
}

// Purpose: audit of credential set/rotate/disable actions
<!-- BIND: confirmed=C-DM-07 -->
Table credential_events {
  id uuid [pk, default: `gen_random_uuid()`]
  carrier_code text [not null]
  action text [not null, note: 'set | rotate | disable']
  actor text [not null]
  created_at timestamptz [not null, default: `now()`]
}

Ref: carrier_credentials.carrier_code > carriers.code
Ref: bookings.carrier_code > carriers.code
Ref: booking_events.booking_id > bookings.id
Ref: shipment_events.booking_id > bookings.id
Ref: carrier_messages.booking_id > bookings.id
```

## Constraints

- **Uniqueness**: `carrier_credentials(carrier_code)` unique WHERE `is_active` (one active credential row per carrier — spec §6, D-012). `bookings(created_by, idempotency_key)` unique WHERE `idempotency_key IS NOT NULL` (spec §6, §7). `booking_events(booking_id, seq)` unique. `shipment_events.dedupe_key` unique (spec §6).
- **Indexes**: `bookings(status)`, `bookings(carrier_code, status)`, `bookings(requested_departure_date)` for list filtering (spec §6). `shipment_events(booking_id)`, `shipment_events(equipment_reference)` (spec §6). `carrier_messages(carrier_code, created_at)`, `carrier_messages(booking_id)` (spec §6).
- **Soft delete**: operational records use soft delete / archive; no hard deletes (spec §13).
- **Derived / denormalized**: `bookings.pol`, `pod`, `vessel_name`, `voyage_number`, `requested_departure_date`, `customer_reference` are denormalized copies of fields inside `canonical_request` for filtering — `canonical_request` is authoritative (spec §6).
- **Archival**: `carrier_messages` grows unbounded; a periodic archival job is a later concern, not v1 (spec §6).

---

## Sources

- PRD §5 (Canonical data model), §6 (Database schema), §7 (Idempotency), §13 (Credential store, soft delete, UTC)

## Out of Scope

- `graphile-worker` internal tables (managed by its installer — spec §6).
- Any schema for Shipping Instruction, amendment, or schedule search (v2 — spec §2).
- `packages/db` shared base types for other monorepo apps beyond what `apps/booking` needs (spec §4).
