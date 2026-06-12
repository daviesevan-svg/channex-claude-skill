# Channex API reference (verified shapes)

Every shape below was verified against a live Channex staging server.
For anything not covered here, fetch the official docs — every page at
https://docs.channex.io has a markdown variant (append `.md` to the
URL), `https://docs.channex.io/sitemap.md` lists all pages, and
`https://docs.channex.io/llms-full.txt` is the full corpus.

## Basics

- Base URLs: `https://staging.channex.io/api/v1` (sandbox),
  `https://app.channex.io/api/v1` (production)
- Auth: `user-api-key: <key>` header on every request
- Success: 2xx with `{"data": {...}}` or `{"data": [...]}` — unwrap it
- Errors: `{"errors": {"code": "...", "title": "...", "details": [...]}}`
  with 400/401/404/422
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
`options: [{"occupancy": N, "is_primary": true, "rate": 0}]`.

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

## Bookings (inbound)

```
GET  /booking_revisions/feed          → unacked revisions, oldest first
POST /booking_revisions/:revision_id/ack   {}
```

Use the revisions feed, NOT `GET /bookings...` — that's the documented
PMS pattern. Acked revisions never reappear. The feed covers the WHOLE
account (every property the key can see).

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
- Unacked revisions trigger warning emails to the account owner after
  ~30 minutes.
- Channex applies availability auto-decrement on bookings only if the
  property settings enable it; a PMS-driven integration should push
  authoritative availability itself and treat Channex's counters as a
  mirror, not a source.
- Certification (required before production OTA connections):
  https://docs.channex.io/api-v.1-documentation/pms-certification-tests.md
- Webhooks (`/webhooks` CRUD) can replace feed polling later; start
  with polling — simpler and certifiable.
