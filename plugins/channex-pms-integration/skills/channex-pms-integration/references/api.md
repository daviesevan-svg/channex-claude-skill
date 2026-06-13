# Channex API reference (verified shapes)

Every endpoint, payload, and response shape below was verified by
calling a live Channex staging server (not transcribed from docs) —
request bodies, the `data`/`meta` envelope, the error envelope, and the
read-back shapes for availability and restrictions. Field *availability*
can still vary by account/plan tier, so treat unexpected 422s as a cue
to re-check the live docs. For anything not covered here, fetch the
official docs — every page at
https://docs.channex.io has a markdown variant (append `.md` to the
URL), `https://docs.channex.io/sitemap.md` lists all pages, and
`https://docs.channex.io/llms-full.txt` is the full corpus.

## Basics

- Base URLs: `https://staging.channex.io/api/v1` (sandbox),
  `https://app.channex.io/api/v1` (production)
- Auth: `user-api-key: <key>` header on every request
- Success: 2xx with `{"data": {...}}` or `{"data": [...]}` — unwrap it.
  List/feed responses also carry a sibling `"meta"` object (pagination)
- Errors: `{"errors": {"code": "...", "title": "...", "details": [...]}}`
  with 400/401/404/422. `details` is an array and is OPTIONAL — present
  on validation errors (e.g. 400 → `["restrictions is required"]`),
  absent on auth errors (401 → just `code` + `title`). Don't assume it
  exists when formatting an error
- POST/PUT bodies wrap attributes under the entity name:
  `{"property": {...}}`, `{"room_type": {...}}`, `{"rate_plan": {...}}`
- Keep request payloads under 10 MB; send availability and
  restrictions as SEPARATE messages
- Dates are ISO 8601 `YYYY-MM-DD`; rates are integers in MINOR units
  (cents) on writes, decimal strings ("120.00") on reads and in
  booking payloads

## Content entities

### Property

```
POST /properties            {"property": ATTRS}     → 201, data.id = UUID
PUT  /properties/:id        {"property": ATTRS}
GET  /properties            → data: [{id, attributes: {...}}, ...]
```

ATTRS (all optional except title + currency): `title`, `currency`
(ISO 4217), `email`, `phone`, `website`, `country` (2-letter),
`state`, `city`, `address`, `zip_code`, `timezone` (IANA), `content`.
Omit nulls rather than sending them.

`content` is an OBJECT, not a string:

```json
"content": {
  "description": "Some Property Description Text",
  "important_information": "Notes shown in booking confirmation emails",
  "photos": [
    {"url": "https://img.channex.io/<uuid>/", "position": 0,
     "description": "Room View", "author": "Author Name", "kind": "photo"}
  ]
}
```

- `description` (string), `important_information` (string, property-only)
- `photos` (array): each has `url`, `position` (int; 0 = cover photo),
  `description`, `author`, `kind` ("photo" | "ad" | "menu"). On updates
  a photo may also carry its `id` (UUID); responses add system fields
  (`id`, `property_id`).

### Room type

```
POST /room_types            {"room_type": ATTRS}    → 201
PUT  /room_types/:id
GET  /room_types?filter[property_id]=UUID
```

ATTRS: `property_id` (UUID), `title`, `count_of_rooms` (int),
`occ_adults`, `occ_children`, `occ_infants`, `default_occupancy`
(must be ≤ occ_adults), `room_kind` ("room" | "dorm"), `content`.
New room types start with availability 0 — you must push availability
after creating them.

`content` is an OBJECT (same photo shape as property, but NO
`important_information`):

```json
"content": {
  "description": "Some Room Type Description Text",
  "photos": [
    {"url": "https://img.channex.io/<uuid>/", "position": 0,
     "description": "Room View", "author": "Author Name", "kind": "photo"}
  ]
}
```

`description` (string) and `photos` (array, same fields as property
photos; 0 = cover). Responses add `id`/`property_id`/`room_type_id`
to each photo.

### Rate plan

```
POST /rate_plans            {"rate_plan": ATTRS}    → 201
PUT  /rate_plans/:id
GET  /rate_plans?filter[property_id]=UUID
DELETE /rate_plans/:id
```

ATTRS: `property_id`, `room_type_id` (one rate plan belongs to ONE
room type), `title` (unique per property), `currency`,
`sell_mode` ("per_room" | "per_person"), `rate_mode` ("manual" is the
PMS-driven mode; "derived"/"auto"/"cascade" exist),
`options: [{"occupancy": N, "is_primary": true, "rate": 0}]` — that
minimal write shape is enough to create a plan. Note the asymmetry: on
WRITE `rate` is an integer in minor units (cents); in the RESPONSE each
option comes back richer — `{id, occupancy, rate: "0.00" (decimal
string), is_primary, derived_option, rate_category_id, inherit_* …}`.
The option `id` only matters if you later target one specific occupancy
option; for a single-occupancy plan you can ignore it.

## ARI (availability, rates, restrictions)

### Availability — per room type

```
POST /availability
{"values": [
  {"property_id": UUID, "room_type_id": UUID,
   "date_from": "2026-07-01", "date_to": "2026-07-14",
   "availability": 2},
  {"property_id": UUID, "room_type_id": UUID,
   "date": "2026-07-15", "availability": 0}
]}
```

Single `date` or `date_from`/`date_to` ranges (inclusive) both work.

### Restrictions — per rate plan

```
POST /restrictions
{"values": [
  {"property_id": UUID, "rate_plan_id": UUID,
   "date_from": "2026-07-01", "date_to": "2026-07-14",
   "rate": 13800,                  // cents
   "min_stay_arrival": 2,
   "stop_sell": false,
   "closed_to_arrival": false,
   "closed_to_departure": false}
]}
```

**Partial updates are applied as partial**: a value containing only
`{rate}` changes the rate and leaves min-stay/closures untouched. This
is what makes field-level delta pushes possible — exploit it.

**Past dates are rejected** — filter `date >= today` before sending.

### Per-person (occupancy-based) rates

For a `sell_mode: "per_person"` rate plan, restrictions carry a `rates`
ARRAY keyed by occupancy — NOT a scalar `rate`, and NOT an object keyed
by occupancy:

```json
POST /restrictions
{"values": [
  {"property_id": UUID, "rate_plan_id": UUID,
   "date_from": "2026-07-01", "date_to": "2026-07-14",
   "rates": [
     {"occupancy": 1, "rate": 9000},
     {"occupancy": 2, "rate": 11000},
     {"occupancy": 3, "rate": 13000}
   ],
   "min_stay_arrival": 2, "stop_sell": false}
]}
```

min-stay/closures stay single-valued alongside the array. A rate-only
delta sends just the `rates` key. (per_room plans keep the scalar
`rate` shown above.)

### Reading ARI back (for verification)

```
GET /availability?filter[property_id]=UUID
    &filter[date][gte]=2026-07-01&filter[date][lte]=2026-07-14
→ {"data": {ROOM_TYPE_UUID: {"2026-07-01": 2, ...}}}

GET /restrictions?filter[property_id]=UUID
    &filter[date][gte]=...&filter[date][lte]=...
    &filter[restrictions]=rate,min_stay_arrival,stop_sell
→ {"data": {RATE_PLAN_UUID: {"2026-07-01": {"rate": "138.00", ...}}}}
```

`filter[restrictions]` is REQUIRED on the restrictions read — without
it the API returns 400 "restrictions is required". Note rates read
back as decimal strings in major units, not the cents you wrote.

## Channel connection (the Channel API)

Connecting/mapping OTA channels via the API (vs the dashboard) may
require **Channel API access** (historically Whitelabel-gated) — the
first `test_connection` call confirms it. Shapes verified against
staging with Booking.com.

```
GET  /channels/list                       → OTAs available to connect
GET  /channels                            → connected channels
GET  /channels/:id
POST /channels/test_connection            {"channel","settings":{"hotel_id"}}
POST /channels/mapping_details            {"channel","settings":{"hotel_id"}}
POST /channels                            {"channel": ATTRS}        → 201
PUT  /channels/:id                        {"channel": ATTRS}
POST /channels/:id/activate               {}      → go live
POST /channels/:id/deactivate             {}      → pause
DELETE /channels/:id                              → must be inactive first
GET  /groups                              → groups the key can access
```

**`mapping_details` response (Booking.com)** — after unwrapping `data`:

```json
{"rooms": [
  {"id": 651942003, "title": "Double Room", "max_children": null,
   "rates": [
     {"id": 18527581, "title": "standard rate", "pricing": "OBP",
      "max_persons": 2, "occupancies": [1,2], "readonly": false}
   ]}
]}
```

Top key is `rooms` (not `room_types`), rates under `rates` (not
`rate_plans`). Room/rate `id`s are the **integer** `room_type_code` /
`rate_plan_code` you send on create.

**Create a channel:**

```json
POST /channels
{"channel": {
  "channel": "BookingCom",
  "group_id": "<group that owns the property>",   // REQUIRED
  "is_active": false,                              // created inactive
  "title": "Booking.com — My Hotel",
  "properties": ["<property UUID>"],
  "rate_plans": [
    {"rate_plan_id": "<your rate plan UUID>",
     "settings": {"room_type_code": 651942003, "rate_plan_code": 18527581,
                  "occupancy": 2, "pricing_type": "OBP", "primary_occ": true,
                  "readonly": false, "occ_changed": false}}
  ],
  "settings": {"hotel_id": "6519420"}
}}
```

Gotchas (all hit on staging):
- `room_type_code`/`rate_plan_code` must be **integers** — strings →
  the mapping lands under "removed rates", OTA side "Not mapped".
- Missing/inaccessible `group_id` → `422 {"code":
  "unprocessable_entity", "details": "You not have access to requested
  group"}`. Resolve via `GET /groups` (the group whose
  `relationships.properties` includes your property UUID).
- `DELETE` on an active channel → `422 {"channel": ["is active"]}` —
  `POST /channels/:id/deactivate` first.
- Activate/deactivate/load_and_save_ari return
  `{"data": {"meta": {"message": "Success"}}}`.
- One OTA room+rate pair should map to at most one local rate plan.

## Bookings (inbound)

```
GET  /booking_revisions/feed          → unacked revisions, oldest first
GET  /booking_revisions/:id           → one revision by id (use after a webhook)
POST /booking_revisions/:id/ack       {}
POST /webhooks                        {"webhook": {...}}  → register a callback
GET  /bookings?filter[property_id]=UUID    → full booking list (reconciliation)
GET  /booking_revisions?filter[...]        → revision list (not the feed)
```

Two delivery mechanisms, same apply→ack core:

**Webhook (push).** Register:

```json
POST /webhooks
{"webhook": {
  "callback_url": "https://you/api/channex",
  "event_mask": "booking_new;booking_modification;booking_cancellation",
  "property_id": null,          // null = all properties on the account
  "is_active": true,
  "send_data": true
}}
```

Channex then POSTs your callback (events: `booking_new` /
`booking_modification` / `booking_cancellation`, or `"*"`):

```json
{"event": "booking",
 "payload": {"booking_id": "...", "property_id": "...", "revision_id": "..."},
 "user_id": null, "timestamp": "2026-..."}
```

Take `revision_id` → `GET /booking_revisions/:id` → apply → ack. The
webhook is a notification; the pull is the source of truth. `user_id`
is the actor (null/your own id ⇒ skip events you caused).

**Feed (poll).** `GET /booking_revisions/feed` (below) — also the
backstop for missed webhooks within the 30-minute window.

The feed is the real-time path: `GET /booking_revisions/feed` covers
the WHOLE account in one call (every property the key can see; optional
`filter[property_id]` exists but isn't needed). It's paginated —
oldest-first, `meta: {total, limit (default 10), page, order_by,
order_direction}`. Drain it until `meta.total` is 0 rather than
grabbing one page per tick.

**Critical: the feed is a 30-minute window, not a durable queue.** An
unacked revision is re-served for ~30 minutes (with a warning email to
the account owner), then it DROPS OUT of the feed permanently — it does
NOT redeliver forever. Acked revisions also never reappear. Either way,
anything not acked within 30 minutes is gone *from the feed*.

So a feed-only integration loses bookings on any outage > 30 min. The
recovery path is the durable booking LIST endpoint — but use it as a
MANUAL, time-scoped, after-an-outage tool, not a cron. A periodic
full-list sweep re-pulls the same bookings forever and is heavy on both
sides for no benefit while the poller is healthy. After a known gap:
`GET /bookings?filter[inserted_at][gte]=<outage_start>` (paginated,
default `limit` 10, newest-first; also supports `filter[arrival_date]`
/`filter[departure_date]` `[gte]`/`[lte]`) and backfill anything the PMS
is missing, deduped by Channex booking id.

`filter[inserted_at]` is what keeps recovery cheap — scope to bookings
created since the outage. (`GET /booking_revisions` lists revisions
similarly. An unacked filter on that list would be the ideal scoped
recovery source, but it is NOT in the public docs — the feed is
presented as the only unacked view — so confirm it exists on your
account before depending on it.)

Revision item: `{"id": REVISION_UUID, "attributes": {...}}` with
attributes:

- `booking_id` (stable across revisions of one booking), `status`
  ("new" | "modified" | "cancelled")
- `property_id` — check it's yours before applying
- `ota_name` ("Booking.com", "Airbnb", "Expedia", ...),
  `ota_reservation_code`
- `arrival_date`, `departure_date`, `amount` (string, major units),
  `currency`, `payment_collect` ("ota" | "property")
- `customer`: `{name, surname, mail, phone, country, address, ...}`
- `rooms`: list of
  `{checkin_date, checkout_date, room_type_id, rate_plan_id, amount,
    occupancy: {adults, children, infants}, days: {date: price, ...},
    guests: [...]}`

Be defensive: treat every field as possibly missing/null. Map
`room_type_id` back through your mapping table; a booking can span
multiple rooms (one local stay/segment per entry in `rooms`).

There is no API to create test bookings. Channex dashboard →
Applications → add "Booking CRS" → create a booking manually, or use
their Booking.com test channel (see the certification doc).

## Operational notes

- No published hard rate limits, but batch via ranges and debounce —
  hundreds of small requests per minute is abuse-shaped.
- Unacked revisions trigger a warning email to the account owner and
  expire out of the feed after ~30 minutes (see the feed section) —
  pair the feed with `GET /bookings` reconciliation so an outage can't
  silently lose a booking.
- Channex applies availability auto-decrement on bookings only if the
  property settings enable it; a PMS-driven integration should push
  authoritative availability itself and treat Channex's counters as a
  mirror, not a source.
- Certification (required before production OTA connections):
  https://docs.channex.io/api-v.1-documentation/pms-certification-tests.md
- Webhooks (`/webhooks` CRUD) are the low-latency push path (see the
  bookings section); feed polling is simpler to start with and doubles
  as the backstop for missed webhooks. They complement each other —
  webhooks don't remove the 30-minute window or the ack requirement.
