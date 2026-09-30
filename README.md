# Short-Term Rental Booking Platform: API Design and Data Model

# Part 1: Requirements

## 1. What the product does
A marketplace where hosts list properties for short stays and guests search for, book, pay for, and review them. Bookings require host approval before they are confirmed.

## 2. Who uses it
- **Guests** search for available properties, request bookings, pay, and leave reviews.
- **Hosts** list properties, set prices, block dates, and approve or decline booking requests.
- A single person can be both a guest and a host under one account.

## 3. The five most important actions
1. **List a property.** A host creates a property with location, capacity, nightly rate, and cleaning fee.
2. **Search for stays.** A guest searches by location and dates and sees only properties that are free for the whole stay.
3. **Request a booking.** A guest asks to book a property for a date range. The price is locked in at the time of the request.
4. **Respond to a booking.** The host confirms or declines. The guest or host can cancel a booking before check-in.
5. **Pay and review.** The guest pays for a confirmed booking within the payment window. After the stay is completed, the guest can leave one review.

## 4. Business rules the design must enforce
- Two confirmed bookings for the same property can never overlap in dates. Requests do not hold dates.
- Confirmed bookings cannot overlap dates the host has blocked.
- A guest cannot book their own property.
- A booking can only be cancelled before its check-in date, and only marked completed on or after its check-out date.
- A review can only exist for a completed booking, and only one review per booking.
- A booking can have at most one active charge, so a guest is never charged twice.
- All money is stored as whole numbers in minor units (kobo for NGN) with a currency.
- A booking keeps the price and property details as they were when it was made, even if the host edits them later.

## 5. Out of scope
- Messaging between users
- Currency conversion (one currency per property)
- Payouts to hosts and split payments
- Admin and dispute tools
- Photos and media uploads
- Cancelling or shortening a stay after check-in
- Transferring a property to a different host

# Part 2: Data model

## Entities

Every table has `id` (UUID, primary key, generated), `created_at`, and `updated_at`, except `property_amenities`, which is identified by its composite key. "Req" means required.

**users**: a person on the platform, guest, host, or both.

| Field | Type | Req |
|---|---|---|
| email | text, unique | yes |
| name | text | yes |
| phone | text | no |
| deleted_at | timestamp | no |

**properties**: a place a host offers for stay. The host of a property never changes.

| Field | Type | Req |
|---|---|---|
| host_id | UUID, FK to users | yes |
| title | text | yes |
| description | text | no |
| address_line, city | text | yes |
| country_code | 2-letter text | yes |
| latitude, longitude | decimal | yes |
| max_guests | integer | yes |
| nightly_rate_minor | integer (kobo) | yes |
| cleaning_fee_minor | integer (kobo) | yes |
| currency | 3-letter text (NGN) | yes |
| status | text: draft, active, paused | yes |
| deleted_at | timestamp | no |

**amenities**: a feature a property can have (wifi, pool).

| Field | Type | Req |
|---|---|---|
| name | text, unique | yes |

**property_amenities**: the join table between properties and amenities.

| Field | Type | Req |
|---|---|---|
| property_id | UUID, FK | yes |
| amenity_id | UUID, FK | yes |

Identifier: the pair of the two IDs together (composite primary key).

**availability_blocks**: dates a host has closed off manually. `end_date` is exclusive, matching `check_out`.

| Field | Type | Req |
|---|---|---|
| property_id | UUID, FK | yes |
| start_date, end_date | date | yes |
| reason | text | no |

**bookings**: a guest's request to stay. `check_out` is the departure day and is exclusive, so a stay ending on the 5th and another starting on the 5th do not overlap.

| Field | Type | Req |
|---|---|---|
| guest_id | UUID, FK to users | yes |
| property_id | UUID, FK to properties | yes |
| host_id | UUID (snapshot of the property's host) | yes |
| check_in, check_out | date | yes |
| guest_count | integer | yes |
| status | text: requested, confirmed, declined, expired, cancelled, completed | yes |
| property_title | text (snapshot) | yes |
| host_name | text (snapshot) | yes |
| nightly_rate_minor, cleaning_fee_minor, total_minor | integer, kobo (snapshot) | yes |
| currency | 3-letter text (snapshot) | yes |
| expires_at | timestamp: when an unanswered request lapses | yes |
| decided_at | timestamp: when the host confirmed or declined | no |
| payment_due_at | timestamp: deadline to pay after confirmation | no |
| cancelled_at | timestamp | no |
| cancelled_by | text: guest, host, system | no |

**payments**: money moving for a booking.

| Field | Type | Req |
|---|---|---|
| booking_id | UUID, FK | yes |
| type | text: charge, refund | yes |
| amount_minor | integer (kobo) | yes |
| currency | 3-letter text | yes |
| status | text: pending, succeeded, failed | yes |
| idempotency_key | text, unique | yes |
| provider_reference | text, unique | no |

**reviews**: a guest's rating of a completed stay.

| Field | Type | Req |
|---|---|---|
| booking_id | UUID, FK, unique | yes |
| rating | integer, 1 to 5 | yes |
| comment | text | no |
| deleted_at | timestamp | no |

## Relationships

| Relationship | Cardinality | How it is stored |
|---|---|---|
| user to properties (as host) | one to many | `properties.host_id` |
| user to bookings (as guest) | one to many | `bookings.guest_id` |
| property to bookings | one to many | `bookings.property_id` |
| property to availability_blocks | one to many | `availability_blocks.property_id` |
| booking to payments | one to many | `payments.booking_id` |
| booking to review | one to one (at most) | `reviews.booking_id`, unique |
| properties to amenities | many to many | join table `property_amenities` |

## Diagram

<img width="3983" height="2705" alt="image" src="https://github.com/user-attachments/assets/001cb950-fcc7-4809-a85b-4bee45c82b61" />


# Part 3: Design decisions

## 1. Normalisation: where does each fact live?

Every fact lives in one place unless copying it is deliberate and justified. The host's name lives in `users`, the property's title in `properties`. Bookings deliberately copy some values.

**Denormalisation 1: price snapshot.** `bookings` copies `nightly_rate_minor`, `cleaning_fee_minor`, `currency`, and `total_minor` from the property. A guest agreed to a specific price. If the host raises the rate next week, the existing booking must not change, or the guest's receipt, the refund amount, and the host's earnings would all shift silently.

**Denormalisation 2: descriptive snapshot.** `bookings` copies `property_title` and `host_name`. A guest's booking history must still read correctly if the host renames the listing or changes their name.

**Denormalisation 3: `bookings.host_id`.** It is derivable through the property, but the copy lets the database enforce "a guest cannot book their own property" with a plain check on one row, and it lets the host's pending-requests query use a single index without joining to properties. It cannot drift, because a composite foreign key `(property_id, host_id)` referencing `properties (id, host_id)` forces it to match the property's real host, and a property's host never changes.

**Why the copies are safe.** Snapshot fields are written once, at booking time, and a trigger rejects any later change to them. Copies only cause trouble when they can drift.

**Snapshots and erasure.** `host_name` is personal data, and a user can ask to be erased. Erasure wins. The erasure procedure blanks the user's own record and sets the `host_name` snapshot on their bookings to "Deleted user". This is the only update the snapshot trigger permits. Guest personal data is never snapshotted on bookings, because bookings reference the guest by `guest_id` only.

## 2. Money

**Rule:** every monetary value is a whole number in minor units (kobo for NGN), stored beside a 3-letter `currency` column. No decimals or floats anywhere.

**Reason:** decimal arithmetic on computers produces rounding errors (0.1 + 0.2 is not exactly 0.3), and those errors compound across thousands of payments. Integers are exact.

**Applied to:** `nightly_rate_minor`, `cleaning_fee_minor`, `total_minor`, `amount_minor`, each with `CHECK (... >= 0)`. The only decimals in the model are `latitude` and `longitude`, which are coordinates, not money.

**Currency copies.** Currency is stored on properties, bookings, and payments. The copy on the booking is a snapshot, and a composite foreign key `(booking_id, currency)` on payments referencing `bookings (id, currency)` guarantees a payment can never be in a different currency from its booking.

## 3. Status and state

### Booking state machine

Statuses: `requested`, `confirmed`, `declined`, `expired`, `cancelled`, `completed`.

**Allowed transitions:**

| From | To | Triggered by | Condition |
|---|---|---|---|
| requested | confirmed | host | before `expires_at`, and dates free of other confirmed bookings and blocks |
| requested | declined | host | before `expires_at` |
| requested | expired | scheduled job | `expires_at` has passed |
| requested | cancelled | guest | guest withdraws the request |
| confirmed | cancelled | guest, host, or scheduled job | today is before `check_in` |
| confirmed | completed | scheduled job | today is on or after `check_out` |

`declined`, `expired`, `cancelled`, and `completed` are terminal. Nothing leaves them.

**Forbidden transitions:**
- `completed` to anything. A finished stay is history. Later money problems are recorded as new rows in `payments`, not as status changes.
- `cancelled`, `declined`, or `expired` back to `confirmed`. The dates may have been booked by someone else since, so the guest must create a new request.
- `requested` to `completed`. It skips host approval and payment.
- `confirmed` to `declined`. Declining answers a request. Withdrawing a confirmed booking is a cancellation, which carries different consequences and is recorded in `cancelled_by`.
- `confirmed` to `cancelled` on or after `check_in`. Cancelling mid-stay is out of scope. Without this rule, a booking could be cancelled on day 3 of a 5-day stay and the dates re-opened while the guest is still there.
- `confirmed` to `completed` before `check_out`. Marking a future stay completed would unlock reviews for a stay that has not happened.
- `requested` to `expired` before `expires_at`.

**State machine image**
<img width="3078" height="3235" alt="image" src="https://github.com/user-attachments/assets/cf20d175-e5c6-4487-89e9-0f055373d1fe" />


**What enforces it.** A `CHECK` constraint sees only the row's new values, so it cannot know the old status. A `BEFORE UPDATE` trigger on `bookings` compares the old and new status, rejects any pair not in the allowed table, and applies the date conditions. A `CHECK (status IN (...))` restricts the column to the six valid values. The application validates too, but the trigger is what makes the rules unbreakable.

**Who moves bookings automatically.** Nothing changes status on its own. A scheduled job marks unanswered requests `expired`, cancels confirmed bookings that were not paid by `payment_due_at` (with `cancelled_by = 'system'`), and marks stays `completed` after check-out.

### Payment lifecycle

A payment starts `pending` and moves to `succeeded` or `failed`. Both are terminal, enforced by the same style of trigger. A retry after a failure is a new payment row with a new `idempotency_key`, never an edit of the failed row.

## 4. Time and deletion

Every table has `created_at` and `updated_at`. Delete strategy per entity:

| Entity | Delete style | Reason |
|---|---|---|
| users | Soft delete (`deleted_at`), plus anonymise personal fields | Bookings and payments must survive as financial records, but personal data must be erasable on request. Anonymising the row and keeping it satisfies both. The anonymised email becomes `deleted-<user id>@invalid`, which stays unique. |
| properties | Soft delete | Past bookings reference it |
| bookings, payments | Never deleted | They are the financial audit trail. A refund is a new payment row, not an edit. |
| reviews | Soft delete | Allows moderation and reversal |
| availability_blocks, property_amenities, amenities | Hard delete | No history value and nothing depends on them |

## 5. Identifiers

Every primary key is a generated UUID, never a sequential integer. Sequential IDs let anyone enumerate records by counting and reveal business volume. UUIDs are not guessable. An unguessable ID is a second layer of protection, not the only one: the API still checks that the caller is allowed to see each record.

## 6. Constraints

### Per table

| Table | Unique | Foreign keys | Checks |
|---|---|---|---|
| users | `email` | none | none |
| properties | `(id, host_id)` | `host_id` to users | `max_guests > 0`, money fields `>= 0`, `length(currency) = 3`, `status IN ('draft','active','paused')` |
| amenities | `name` | none | none |
| property_amenities | primary key `(property_id, amenity_id)` | `property_id` to properties, `amenity_id` to amenities | none |
| availability_blocks | none | `property_id` to properties | `end_date > start_date`; exclusion: no two blocks overlap on one property |
| bookings | `(id, currency)` | `guest_id` to users, `(property_id, host_id)` to properties `(id, host_id)` | `check_out > check_in`, `guest_id <> host_id`, `guest_count > 0`, money fields `>= 0`, `total_minor = (check_out - check_in) * nightly_rate_minor + cleaning_fee_minor`, status in the six values, `cancelled_at` and `cancelled_by` present when status is `cancelled`; exclusion: no two confirmed or completed bookings overlap on one property |
| payments | `idempotency_key`, `provider_reference`; partial unique on `booking_id` where type is `charge` and status is `pending` or `succeeded` | `(booking_id, currency)` to bookings `(id, currency)` | `amount_minor >= 0`, type and status in allowed values |
| reviews | `booking_id` | `booking_id` to bookings | `rating BETWEEN 1 AND 5` |

### Invalid states that become impossible

| Invalid state | What prevents it |
|---|---|
| Two confirmed bookings overlapping on one property | Exclusion constraint on `bookings`: `EXCLUDE USING gist (property_id WITH =, daterange(check_in, check_out) WITH &&) WHERE (status IN ('confirmed','completed'))` (needs the `btree_gist` extension) |
| Check-out on or before check-in | `CHECK (check_out > check_in)` |
| Guest booking their own property | `CHECK (guest_id <> host_id)`, made reliable by the composite foreign key that forces `host_id` to match the property's real host |
| A booking claiming a host who does not own the property | Composite foreign key `(property_id, host_id)` to `properties (id, host_id)` |
| Wrong total on a booking | `CHECK` tying `total_minor` to nights, rate, and cleaning fee |
| Illegal status jump, or a status change on the wrong date | `BEFORE UPDATE` trigger on `bookings` |
| Snapshot fields edited after booking | `BEFORE UPDATE` trigger on `bookings` |
| Two reviews for one booking | `UNIQUE (booking_id)` on reviews |
| A review on a booking that is not completed | `BEFORE INSERT` trigger on `reviews` checks the booking status. It is enough because `completed` is terminal, so a booking can never leave that state afterwards. |
| Rating outside 1 to 5 | `CHECK (rating BETWEEN 1 AND 5)` |
| Two users with one email | `UNIQUE (email)` |
| The same amenity linked to a property twice | Composite primary key `(property_id, amenity_id)` |
| Overlapping blocks on one property | Exclusion constraint on `availability_blocks` |
| A confirmed booking on blocked dates, or a block over a confirmed booking | Triggers on `bookings` and `availability_blocks` (see limits below) |
| A guest charged twice for one booking | Partial unique index on `payments (booking_id)` where type is `charge` and status is `pending` or `succeeded` |
| A payment retry creating a duplicate | `UNIQUE (idempotency_key)` and `UNIQUE (provider_reference)` |
| A payment in a different currency from its booking | Composite foreign key `(booking_id, currency)` |
| Negative money, zero guests, bad currency code | `CHECK` constraints |
| A booking pointing at a nonexistent property or user | Foreign keys on every reference |

### Decision: requests do not hold dates

The overlap constraint applies only to `confirmed` and `completed` bookings, so several guests can request the same dates and the host picks one. When the host tries to confirm a second overlapping booking, the database rejects it. The alternative was to let requests hold the dates. It was rejected because one guest could tie up a property by sending speculative requests, and hosts would lose bookings to requests that were never paid for.

### Known limits

- **Blocks and bookings live in different tables**, so an exclusion constraint cannot compare them. Triggers do it instead, and each trigger first takes a row lock on the property so two concurrent transactions cannot both pass the check. This is weaker than a single declarative constraint, and it is the one overlap rule that depends on trigger code being correct.
- **Payment before completion is enforced by process, not by schema.** The scheduled job cancels unpaid confirmed bookings, so a completed booking has been paid in practice, but no constraint proves it.

## 7. Indexes

One per query the five actions run:

| Action | Query | Index |
|---|---|---|
| 1. List property | Host views their properties | `properties (host_id)` |
| 2. Search | Active properties in a city | `properties (city) WHERE status = 'active' AND deleted_at IS NULL` (partial) |
| 2. Search | Exclude properties with overlapping bookings | The GiST index created by the bookings exclusion constraint. It covers only confirmed and completed rows, so the search query must repeat that same status filter to use it. |
| 2. Search | Exclude blocked dates | The GiST index created by the blocks exclusion constraint |
| 3. Request booking | Guest's own bookings | `bookings (guest_id, created_at DESC)` |
| 4. Respond | Host's pending requests across all their properties | `bookings (host_id, status)` |
| 5. Pay and review | Payments for a booking | `payments (booking_id)` |
| 5. Pay and review | Reviews for a property | Join from `bookings (property_id, status)` to the unique index on `reviews (booking_id)` |
| Scheduled job | Requests past `expires_at` | `bookings (expires_at) WHERE status = 'requested'` (partial) |
| Scheduled job | Confirmed bookings past `payment_due_at` | `bookings (payment_due_at) WHERE status = 'confirmed'` (partial) |

Primary keys and unique constraints create their own indexes, so those are not listed.

# Part 4: Alternatives considered

- **Instant booking instead of host approval.** Simpler, with no `requested` state. Rejected because hosts need control over who stays, and approval produces the richer and more realistic lifecycle.
- **Separate guest and host tables instead of one users table.** Rejected because one person is often both, and two tables would duplicate identity data and split login.
- **Paying at request time instead of after confirmation.** Rejected because declined and expired requests would each need a refund. Payment after confirmation, within 24 hours of `decided_at`, avoids refunding money that never needed to move.
- **Requests holding dates.** Rejected, as explained in the constraints section.
- **A `checked_in` status to allow mid-stay cancellation.** Rejected as out of scope. Cancellation stops at check-in instead, which keeps the state machine small.

# Part 5: API Design

## 1. Conventions

**Base path.** Every path starts with `/api/v1`. Versioning starts on day one, so a breaking change ships as `/api/v2` without breaking existing clients.

**Format.** JSON in and out. IDs are UUIDs. Dates are `YYYY-MM-DD`. Timestamps are ISO 8601 in UTC. Field names are camelCase. Money is an integer in minor units (kobo) beside a `currency` field.

**Authentication.** Endpoints marked "auth" need a bearer token that identifies the caller. How tokens are issued is out of scope. The caller's user id always comes from the token, never from the request body, so a client can never act as another user.

**Success envelope.**

```json
{ "data": { }, "meta": { } }
```

`meta` appears on list responses only.

**Error envelope.** Every error, on every endpoint, has this shape:

```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "Request body has invalid fields",
    "details": [ { "field": "checkOut", "issue": "must be after checkIn" } ]
  }
}
```

`details` is optional. `code` is a stable machine-readable string. `message` is for humans and can change.

**Status codes used.**

| Status | Meaning here |
|---|---|
| 200, 201, 204 | Success (read or update, created, no body) |
| 400 | Malformed request: bad JSON, bad query parameter, missing header |
| 401 | No valid token |
| 402 | Payment provider declined the payment |
| 403 | Authenticated but not allowed |
| 404 | Not found, malformed id, or a record the caller is not allowed to know exists |
| 409 | Valid request that conflicts with current state (illegal transition, overlapping dates, duplicate) |
| 422 | Well-formed request whose values fail validation |
| 429 | Rate limit exceeded. Includes a `Retry-After` header. |
| 502 | Payment provider unreachable |

**Visibility rule.** A booking is visible only to its guest and its host. Anyone else gets 404, not 403, so the API does not confirm that a booking id exists.

**Idempotency.** Mutations that create money or bookings take an `Idempotency-Key` header (a client-generated UUID). The server stores the key with a hash of the request body and the response. Repeating the same key with the same body returns the original response with the header `Idempotent-Replayed: true` and creates nothing. The same key with a different body returns 422 `IDEMPOTENCY_KEY_MISMATCH`. Keys expire after 24 hours. The `idempotency_keys` table this needs is described in section 9.

**Standard errors on every endpoint.** 401 `UNAUTHENTICATED` (auth endpoints only) and 429 `RATE_LIMITED`. They are not repeated in each table below.

## 2. Action 1: List a property

`POST /api/v1/properties` (auth)

**Request body**

| Field | Type | Req |
|---|---|---|
| title | string, 3 to 120 chars | yes |
| description | string, up to 5000 chars | no |
| addressLine | string | yes |
| city | string | yes |
| countryCode | string, 2 letters | yes |
| latitude | number, -90 to 90 | yes |
| longitude | number, -180 to 180 | yes |
| maxGuests | integer, at least 1 | yes |
| nightlyRateMinor | integer, at least 0 | yes |
| cleaningFeeMinor | integer, at least 0 | yes |
| currency | string, 3 letters | yes |
| amenityIds | array of UUID | no |

The new property starts with status `draft`. The host id is taken from the token.

**Response 201**

```json
{
  "data": {
    "id": "b1f2c6e0-7a54-4c1e-9d2a-3f6a1c0b8e11",
    "hostId": "0c5d9a3e-2b7f-4e6a-8f10-5d4c7b9a1e22",
    "title": "Lekki Beach House",
    "description": null,
    "addressLine": "12 Admiralty Way",
    "city": "Lagos",
    "countryCode": "NG",
    "latitude": 6.4474,
    "longitude": 3.4723,
    "maxGuests": 6,
    "nightlyRateMinor": 4500000,
    "cleaningFeeMinor": 500000,
    "currency": "NGN",
    "status": "draft",
    "amenities": [],
    "createdAt": "2026-09-29T10:15:00Z",
    "updatedAt": "2026-09-29T10:15:00Z"
  }
}
```

**Errors**

| Status | Code | When |
|---|---|---|
| 400 | INVALID_JSON | Body is not valid JSON |
| 422 | VALIDATION_FAILED | Any field fails its rule. `details` names each field. |
| 422 | UNKNOWN_AMENITY | An id in `amenityIds` does not exist |

**Idempotent?** Not by nature: sending it twice creates two properties. It accepts an optional `Idempotency-Key`, and a client that retries after a timeout should send one.

## 3. Action 2: Search for stays

`GET /api/v1/properties` (public)

**Query parameters**

| Parameter | Type | Meaning |
|---|---|---|
| city | string | Exact match, case-insensitive |
| countryCode | string | 2-letter code |
| checkIn, checkOut | date | Must be sent together. Excludes properties with a confirmed or completed booking, or a block, overlapping the range. |
| guests | integer | Property must have `maxGuests` at least this |
| minPriceMinor, maxPriceMinor | integer | Range on `nightlyRateMinor` |
| amenityIds | comma-separated UUIDs | Property must have all of them |
| mine | boolean (auth) | Only the caller's own properties, in every status. Without it, only `active` non-deleted properties are returned. |
| sort | `createdAt` or `price` | Default `createdAt` |
| order | `asc` or `desc` | Default `desc` |
| limit | integer | Default 20, maximum 100. Larger values are clamped to 100. |
| cursor | string | Opaque value from the previous response's `meta.nextCursor` |

**Response 200**

```json
{
  "data": [
    {
      "id": "b1f2c6e0-7a54-4c1e-9d2a-3f6a1c0b8e11",
      "title": "Lekki Beach House",
      "city": "Lagos",
      "countryCode": "NG",
      "latitude": 6.4474,
      "longitude": 3.4723,
      "maxGuests": 6,
      "nightlyRateMinor": 4500000,
      "cleaningFeeMinor": 500000,
      "currency": "NGN",
      "stayTotalMinor": 14000000
    }
  ],
  "meta": { "limit": 20, "hasMore": true, "nextCursor": "eyJ2IjoxNDAwMDAwMCwiaWQiOiJiMWYyIn0" }
}
```

`stayTotalMinor` appears only when `checkIn` and `checkOut` are given. Amenities and description are left out of the list on purpose. They are on the detail endpoint.

**Errors**

| Status | Code | When |
|---|---|---|
| 400 | INVALID_QUERY | Unparseable value, negative limit, only one of `checkIn` and `checkOut`, `checkOut` not after `checkIn`, or dates in the past. `details` names the parameter. |
| 400 | INVALID_SORT | `sort` is not an allowed field |
| 400 | INVALID_CURSOR | Cursor is malformed or does not match the current sort |
| 401 | UNAUTHENTICATED | `mine=true` without a token |

**Idempotent?** Yes. It is a read with no side effects.

## 4. Action 3: Request a booking

`POST /api/v1/bookings` (auth)

**Headers:** `Idempotency-Key` (required)

**Request body**

| Field | Type | Req |
|---|---|---|
| propertyId | UUID | yes |
| checkIn | date | yes |
| checkOut | date | yes |
| guestCount | integer, at least 1 | yes |

The server looks up the property, copies its price, title, currency, and host onto the booking, computes `totalMinor`, sets `status` to `requested`, and sets `expiresAt` to 48 hours from now.

```json
{ "propertyId": "b1f2c6e0-7a54-4c1e-9d2a-3f6a1c0b8e11", "checkIn": "2026-11-10", "checkOut": "2026-11-13", "guestCount": 4 }
```

**Response 201**

```json
{
  "data": {
    "id": "7e3a9f10-5c2b-4d88-a1f4-9b6c0d2e5a33",
    "guestId": "5a8c1d4f-9e20-4b77-b3a6-1f0e2d7c9b44",
    "propertyId": "b1f2c6e0-7a54-4c1e-9d2a-3f6a1c0b8e11",
    "hostId": "0c5d9a3e-2b7f-4e6a-8f10-5d4c7b9a1e22",
    "status": "requested",
    "checkIn": "2026-11-10",
    "checkOut": "2026-11-13",
    "guestCount": 4,
    "propertyTitle": "Lekki Beach House",
    "hostName": "Amaka O.",
    "nightlyRateMinor": 4500000,
    "cleaningFeeMinor": 500000,
    "totalMinor": 14000000,
    "currency": "NGN",
    "expiresAt": "2026-10-01T10:20:00Z",
    "decidedAt": null,
    "paymentDueAt": null,
    "cancelledAt": null,
    "cancelledBy": null,
    "createdAt": "2026-09-29T10:20:00Z",
    "updatedAt": "2026-09-29T10:20:00Z"
  }
}
```

**Errors**

| Status | Code | When |
|---|---|---|
| 400 | INVALID_JSON | Body is not valid JSON |
| 400 | IDEMPOTENCY_KEY_REQUIRED | Header missing |
| 404 | PROPERTY_NOT_FOUND | Property does not exist, is deleted, or is not `active` |
| 409 | DATES_UNAVAILABLE | A confirmed booking or block already covers part of the range. This is a courtesy check for the guest. The authoritative check happens when the host confirms. |
| 422 | VALIDATION_FAILED | `checkOut` not after `checkIn`, dates in the past, `guestCount` above the property's `maxGuests`, or a field missing |
| 422 | CANNOT_BOOK_OWN_PROPERTY | The caller is the property's host |
| 422 | IDEMPOTENCY_KEY_MISMATCH | Key reused with a different body |

**Idempotent?** Yes, through the `Idempotency-Key`. A guest whose phone drops the connection can safely resend and gets the original booking back instead of a second one.

## 5. Action 4: Respond to a booking

Status is not a writable field. A client cannot `PATCH` a booking's status, because that would let anyone attempt any transition. Each transition is a separate command, and the state machine trigger has the final say.

### Confirm

`POST /api/v1/bookings/:id/confirm` (auth, the booking's host only). No body.

Sets `status` to `confirmed`, `decidedAt` to now, and `paymentDueAt` to 24 hours from now. **Response 200** returns the booking.

| Status | Code | When |
|---|---|---|
| 404 | NOT_FOUND | Booking does not exist, or the caller is not its host or guest |
| 403 | FORBIDDEN | The caller is the booking's guest, not its host |
| 409 | INVALID_STATE_TRANSITION | Booking is `declined`, `expired`, `cancelled`, or `completed` |
| 409 | REQUEST_EXPIRED | `expiresAt` has passed and the job has not yet run |
| 409 | DATES_UNAVAILABLE | Another confirmed booking or a block overlaps. Raised by the exclusion constraint or the block trigger and translated by the API. |

**Idempotent?** Yes. Confirming an already confirmed booking returns 200 with the current state and changes nothing.

### Decline

`POST /api/v1/bookings/:id/decline` (auth, host only). Optional body `{ "reason": "string, up to 500 chars" }`.

Sets `status` to `declined` and `decidedAt` to now. **Response 200** returns the booking.

| Status | Code | When |
|---|---|---|
| 404 | NOT_FOUND | As above |
| 403 | FORBIDDEN | The caller is the guest |
| 409 | INVALID_STATE_TRANSITION | Booking is not `requested`. A confirmed booking cannot be declined. The host cancels it instead. |
| 409 | REQUEST_EXPIRED | `expiresAt` has passed |

**Idempotent?** Yes. Declining an already declined booking returns 200 unchanged.

### Cancel

`POST /api/v1/bookings/:id/cancel` (auth, guest or host). Optional body `{ "reason": "string, up to 500 chars" }`.

Sets `status` to `cancelled`, `cancelledAt` to now, and `cancelledBy` to `guest` or `host`. If a charge has succeeded, a full refund payment is created. Refund tiers are out of scope. **Response 200** returns the booking.

| Status | Code | When |
|---|---|---|
| 404 | NOT_FOUND | Booking does not exist, or the caller is neither its guest nor its host |
| 409 | INVALID_STATE_TRANSITION | Booking is `declined`, `expired`, or `completed`. Also: the host tries to cancel a `requested` booking, which should be a decline. |
| 409 | CANCELLATION_WINDOW_CLOSED | Today is on or after `checkIn` |

**Idempotent?** Yes. Cancelling an already cancelled booking returns 200 unchanged, and no second refund is created.

## 6. Action 5: Pay and review

### Pay

`POST /api/v1/bookings/:id/payments` (auth, the booking's guest only)

**Headers:** `Idempotency-Key` (required). It is stored as `payments.idempotency_key`.

**Request body**

| Field | Type | Req |
|---|---|---|
| paymentMethodToken | string, issued by the payment provider | yes |

The amount is never accepted from the client. It is always the booking's `totalMinor`, and the currency is the booking's currency.

**Response 201**

```json
{
  "data": {
    "id": "d4b7e1a9-3c60-4f52-8a1d-6e9f0b2c7d55",
    "bookingId": "7e3a9f10-5c2b-4d88-a1f4-9b6c0d2e5a33",
    "type": "charge",
    "amountMinor": 14000000,
    "currency": "NGN",
    "status": "succeeded",
    "createdAt": "2026-09-29T11:02:00Z"
  }
}
```

**Errors**

| Status | Code | When |
|---|---|---|
| 400 | IDEMPOTENCY_KEY_REQUIRED | Header missing |
| 404 | NOT_FOUND | Booking not visible to the caller |
| 403 | FORBIDDEN | The caller is the host |
| 409 | BOOKING_NOT_PAYABLE | Booking status is not `confirmed` |
| 409 | PAYMENT_WINDOW_CLOSED | `paymentDueAt` has passed |
| 409 | PAYMENT_ALREADY_EXISTS | A pending or succeeded charge exists. Raised by the partial unique index. |
| 422 | VALIDATION_FAILED | Token missing or malformed |
| 422 | IDEMPOTENCY_KEY_MISMATCH | Key reused with a different body |
| 402 | PAYMENT_DECLINED | The provider declined. A `failed` payment row is stored, and the guest may try again with a new key. |
| 502 | PROVIDER_UNAVAILABLE | The provider did not answer. The payment stays `pending`, and the client reads `GET /api/v1/bookings/:id/payments` to see the outcome. |

**Idempotent?** Yes, and it matters most here. The idempotency key plus the unique index on charges mean a retry can never charge the guest twice.

### Review

`POST /api/v1/bookings/:id/review` (auth, the booking's guest only)

**Request body:** `rating` (integer 1 to 5, required), `comment` (string, up to 2000 chars, optional).

**Response 201** returns `{ id, bookingId, rating, comment, createdAt }`.

| Status | Code | When |
|---|---|---|
| 404 | NOT_FOUND | Booking not visible to the caller |
| 403 | FORBIDDEN | The caller is the host |
| 409 | BOOKING_NOT_COMPLETED | Status is not `completed` |
| 409 | REVIEW_ALREADY_EXISTS | The booking already has a review |
| 422 | VALIDATION_FAILED | Rating out of range or comment too long |

**Idempotent?** Safe against duplicates. A repeated request cannot create a second review, because of the unique constraint on `booking_id`. The repeat returns 409 instead of replaying the original response.

## 7. Basic operations for every entity

Same conventions, error envelope, and pagination contract as above. Unless noted, an unknown or malformed id returns 404 `NOT_FOUND`, a bad body returns 422 `VALIDATION_FAILED`, and a non-owner caller gets 403 `FORBIDDEN`.

| Method and path | Auth | Purpose | Success | Specific errors | Idempotent |
|---|---|---|---|---|---|
| `POST /users` | none | Register. Body: email, name, phone. | 201 | 409 EMAIL_TAKEN | No |
| `GET /users/me` | yes | Read own profile | 200 | none | Yes |
| `PATCH /users/me` | yes | Change name or phone | 200 | none | Yes |
| `DELETE /users/me` | yes | Erase account: soft delete and anonymise | 204 | 409 HAS_UPCOMING_BOOKINGS (confirmed bookings not yet completed, as guest or host) | Yes. Repeat gives 401 because the token is dead. |
| `GET /properties/:id` | none | Full property with amenities and host name and id | 200 | 404 for drafts and deleted properties unless the caller is the host | Yes |
| `PATCH /properties/:id` | host | Partial update. `hostId` cannot be changed. Existing bookings keep their snapshots. | 200 | 422 for an attempt to set `hostId` | Yes |
| `DELETE /properties/:id` | host | Soft delete | 204 | 409 HAS_ACTIVE_BOOKINGS (confirmed bookings with a future check-out) | Yes |
| `PUT /properties/:id/amenities` | host | Replace the property's amenity set. Body: `amenityIds`. | 200 | 422 UNKNOWN_AMENITY | Yes |
| `GET /amenities` | none | List amenities. Sort by `name`. | 200 | none | Yes |
| `GET /properties/:id/blocks` | host | List blocks. Sort by `startDate`. | 200 | none | Yes |
| `POST /properties/:id/blocks` | host | Block dates. Body: startDate, endDate (exclusive), reason. | 201 | 409 BLOCK_OVERLAPS (another block or a confirmed booking), 422 for `endDate` not after `startDate` | No |
| `DELETE /properties/:id/blocks/:blockId` | host | Remove a block (hard delete) | 204 | none | Yes |
| `GET /bookings` | yes | List the caller's bookings | 200 | 400 INVALID_QUERY | Yes |
| `GET /bookings/:id` | yes | One booking | 200 | 404 for non-participants | Yes |
| `GET /bookings/:id/payments` | yes | Payments for a booking | 200 | 404 for non-participants | Yes |
| `GET /properties/:id/reviews` | none | Reviews for a property. Sort by `createdAt` or `rating`. | 200 | none | Yes |

`GET /bookings` takes the same pagination parameters as search, plus `role` (`guest` or `host`, default `guest`), `status`, `propertyId` (host role), and `checkInFrom` and `checkInTo` (dates). Sort is `createdAt` or `checkIn`.

## 8. Pagination, filtering, and sorting contract

Every list endpoint follows this contract.

**Pagination is cursor-based.** A request sends `limit` and, after the first page, `cursor`. The response sends `meta.limit`, `meta.hasMore`, and `meta.nextCursor` (null on the last page). The cursor is an opaque string that encodes the last row's sort value and id, so ordering is stable even when ties exist.

**Why cursor and not offset.** Bookings and listings change while a user scrolls. With offset, a row inserted on page 1 shifts everything down and page 2 shows a duplicate. A cursor continues after the last row seen, so nothing is skipped or repeated, and the database seeks straight to that point instead of counting past thousands of rows. The cost is that a client cannot jump to page 50, and there is no `total` count, which is expensive to compute on large tables. Offset would be the better choice for an admin table where jumping to a page matters.

**Limits.** Default 20, maximum 100, clamped. A negative or non-numeric limit is 400 `INVALID_QUERY`.

**Filtering.** Each endpoint lists the filters it accepts. An unknown filter name is ignored, and a known filter with a bad value is 400 `INVALID_QUERY`.

**Sorting.** `sort` accepts only the fields the endpoint lists, and `order` accepts `asc` or `desc`. Anything else is 400 `INVALID_SORT`. Every sort adds `id` as a tiebreaker.

## 9. Schema addition for idempotency

The idempotency rules above need one table that Part 2 does not have yet.

**idempotency_keys**

| Field | Type | Req |
|---|---|---|
| user_id | UUID, FK to users | yes |
| key | text | yes |
| request_hash | text | yes |
| response_status | integer | yes |
| response_body | JSON | yes |
| created_at | timestamp | yes |

Primary key `(user_id, key)`, so keys are scoped per user, and a cleanup job deletes rows older than 24 hours. `payments.idempotency_key` stays as a second line of defence.

## 10. Over-fetching: REST versus GraphQL

**The idea.** When a client asks a REST API for information, the endpoint returns the whole resource in a fixed shape, even if the client needs only a few fields. This is over-fetching. It costs bandwidth and processing: more data crosses the network and more JSON must be parsed, and as the data grows and clients run on mobile connections, the app becomes slower. GraphQL lets the client ask for specific fields and returns only those, and it can gather related data in the same request. The standard approach is to start with REST for an MVP and move to GraphQL when REST has made the app slow because of this.

Two points sharpen this. First, the cost is payload size and request count, not storage: both styles read from the same database, and nothing extra is stored. Second, REST over-fetches by default, not by necessity. It can return smaller responses through summary shapes in lists and a `fields` parameter, so those fixes are tried before switching.

**The screen.** A mobile "My trips" list needs, for each booking, only the booking id, status, check-in, check-out, total, and the property title and city.

**What REST returns.** `GET /api/v1/bookings` returns the full booking object for every row, 21 fields each. This is one item of 20:

```json
{
  "id": "7e3a9f10-5c2b-4d88-a1f4-9b6c0d2e5a33",
  "guestId": "5a8c1d4f-9e20-4b77-b3a6-1f0e2d7c9b44",
  "propertyId": "b1f2c6e0-7a54-4c1e-9d2a-3f6a1c0b8e11",
  "hostId": "0c5d9a3e-2b7f-4e6a-8f10-5d4c7b9a1e22",
  "status": "confirmed",
  "checkIn": "2026-11-10",
  "checkOut": "2026-11-13",
  "guestCount": 4,
  "propertyTitle": "Lekki Beach House",
  "hostName": "Amaka O.",
  "nightlyRateMinor": 4500000,
  "cleaningFeeMinor": 500000,
  "totalMinor": 14000000,
  "currency": "NGN",
  "expiresAt": "2026-10-01T10:20:00Z",
  "decidedAt": "2026-09-29T14:00:00Z",
  "paymentDueAt": "2026-09-30T14:00:00Z",
  "cancelledAt": null,
  "cancelledBy": null,
  "createdAt": "2026-09-29T10:20:00Z",
  "updatedAt": "2026-09-29T14:00:00Z"
}
```

The screen uses 6 of these 21 fields. The city is not there at all, because the booking stores the title but not the city, so the client would need a second call per property to show it.

**The same need in GraphQL.**

```graphql
query {
  myBookings(first: 20) {
    id
    status
    checkIn
    checkOut
    totalMinor
    property { title city }
  }
}
```

Returns:

```json
{
  "data": {
    "myBookings": [
      {
        "id": "7e3a9f10-5c2b-4d88-a1f4-9b6c0d2e5a33",
        "status": "confirmed",
        "checkIn": "2026-11-10",
        "checkOut": "2026-11-13",
        "totalMinor": 14000000,
        "property": { "title": "Lekki Beach House", "city": "Lagos" }
      }
    ]
  }
}
```

One request, only the fields asked for, and the city arrives in the same round trip.

**Decision.** Stay on REST for the MVP. The over-fetching is real but small: each booking is about 600 bytes of JSON, so a page of 20 is roughly 12 KB, and most of it goes unused by this screen. That is minor next to a single photo. REST is also simpler to cache, rate-limit, document, and test, and one client and one team do not justify a second API layer. Two cheap REST fixes cover most of the gap: a `fields` parameter for sparse responses, and an `include=property` option that embeds the property summary.

**When to switch.** Switch when REST has made the app slow. Made specific: first confirm that the slowness comes from response size or from too many requests, and not from missing indexes or unpaginated lists, which GraphQL would not fix. The trigger is not a user count, because 100,000 users with one app still fit REST. Switch when at least two of these are true: three or more client surfaces (web, iOS, Android, partners) need different shapes of the same data; a single screen needs three or more sequential REST calls; measured 95th-percentile load time on mobile data for a main screen stays above 2 seconds after `fields` and `include` are in place; or the frontend team is repeatedly blocked waiting for new backend endpoints. GraphQL would then sit in front of the same services, not replace them.

## 11. Real-time: watching a booking's status

**The need.** After a guest requests a booking, they wait up to 48 hours for the host's answer. When the host confirms, the guest must learn immediately, because the 24-hour payment window has started. Polling `GET /bookings/:id` every few seconds wastes requests and still lags.

**Choice: Server-Sent Events.**

`GET /api/v1/bookings/:id/events` (auth, guest or host of that booking)

The server keeps the connection open and pushes an event whenever the booking changes:

```
id: 42
event: booking.status_changed
data: {"bookingId":"7e3a9f10-5c2b-4d88-a1f4-9b6c0d2e5a33","status":"confirmed","paymentDueAt":"2026-09-30T14:00:00Z"}
```

**Why SSE and not WebSockets.** Data flows in one direction only: server to client. The guest never sends anything back through this channel, because responding to a booking already goes through normal REST commands. WebSockets are bidirectional, which is more than this needs, and they cost more to run: a separate protocol upgrade, custom reconnection logic, and trickier behaviour behind proxies. SSE runs over plain HTTP, and browsers reconnect automatically and resume from the last event id through the `Last-Event-ID` header, so a guest on a weak mobile connection does not miss the confirmation.

**Errors:** 404 `NOT_FOUND` for non-participants, 401 `UNAUTHENTICATED`. On reconnect, a client that missed events is sent the current state first, so it never shows a stale status.

**When WebSockets would be right.** Host and guest messaging, which is out of scope. Both sides send and receive continuously, so the channel needs to be bidirectional.

**Fallback.** If a client cannot hold a connection, it polls `GET /bookings/:id` no more often than every 30 seconds.

## Picture Evidence

**1. Index search** 
<img width="2096" height="960" alt="image" src="https://github.com/user-attachments/assets/02d051ca-1d86-4e15-a838-a7617c11847e" />
<img width="2031" height="628" alt="image" src="https://github.com/user-attachments/assets/8dd41972-948c-4627-bcf8-c577424c090d" />


**2. Violations**

Tested some violations and the screenshots are below;
<img width="1583" height="1101" alt="image" src="https://github.com/user-attachments/assets/1a59e702-1e12-4bfe-86b2-ecb95e590ea6" />


