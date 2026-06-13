---
name: channex-pms-integration
description: Integrate a Property Management System (PMS) with the Channex channel manager API (channex.io) so availability, rates, and restrictions flow out to OTAs (Booking.com, Airbnb, Expedia, VRBO) and OTA bookings flow back in. Use this whenever the user wants to connect their PMS, hotel software, hostel system, or vacation-rental platform to Channex or to OTAs through a channel manager — including pushing ARI (availability/rates/inventory), receiving or syncing channel bookings, building a channel-manager integration, preparing for Channex PMS certification, or debugging an existing Channex integration. Apply it even when the user doesn't name Channex explicitly but mentions distributing inventory to booking channels via a channel manager API.
---

# Integrate a PMS with Channex

Channex (https://channex.io) is a channel manager: the PMS pushes
availability/rates/restrictions (ARI) to it, Channex fans them out to
OTAs, and OTA bookings come back through a revisions feed. This skill
encodes a battle-tested integration architecture and the API knowledge
to build it into ANY PMS codebase, in whatever language the project
uses.

Read `references/api.md` before writing any client code — it has the
verified endpoint shapes, payload formats, and API quirks. Don't guess
payloads from memory; between this reference and the live docs
(https://docs.channex.io — append `.md` to any page URL for clean
markdown; `https://docs.channex.io/sitemap.md` lists every page) you
can confirm everything.

## Before writing code: map their PMS

The integration is a translation layer, so first learn the PMS's shape.
Find (in code, or ask the user):

1. **Room model.** Channex sells *room types* with a `count_of_rooms`,
   not individual rooms. If the PMS tracks individual rooms, you'll
   aggregate: availability = rooms of type − rooms occupied that night,
   and inbound bookings need a room-assignment step.
2. **Rate source.** Where does "the price for room type X on date D"
   come from? A rate table? A formula (base × season × weekday)?
   Manual overrides? Channex needs a concrete number per date.
3. **Money units.** Channex rates are in MINOR units (cents). Convert
   at the API boundary only — never let two money conventions leak
   through the codebase.
4. **Booking model.** What's the minimal write path for "create a
   booking with guest, dates, price, source=OTA"? Prefer the PMS's
   existing domain functions over raw inserts so audit/eventing
   logic still runs.
5. **Change signals.** How does the PMS announce "a booking changed" /
   "a price changed" (events, hooks, pub/sub, triggers)? This drives
   real-time pushes. If there's nothing, plan a polling/diff fallback
   and tell the user about the tradeoff.

Also ask: staging or production? (staging.channex.io vs
app.channex.io) and get an API key (Channex dashboard → user settings
→ API keys). **Keys are secrets**: env vars / gitignored .env only,
and check the repo isn't public before letting any credential near a
committed file.

## The four components

Build and verify them in this order — each one is independently
testable against staging before the next exists.

### 1. API client

A thin HTTP wrapper: `user-api-key` header, JSON, unwrap the
`{"data": ...}` envelope, normalize errors (`{"errors": {code,
title, details}}`), modest retries on transient failures. Make the
transport stubbable for tests in whatever way is idiomatic for the
stack. Verify: `GET /properties` returns 200 with the key.

### 2. ID mapping + content sync

Channex assigns UUIDs to the property, room types, and rate plans.
Persist a mapping table in the PMS database:

    (kind, local_id) → channex_id      kinds: property, room_type,
                                       rate_plan, booking

Content sync upserts the property → room types → rate plans (in that
dependency order): POST when no mapping exists, PUT when it does, and
store the returned UUID. This makes the sync idempotent — safe to
re-run forever. The `booking` kind serves double duty later: dedupe of
inbound revisions and lookup for cancellations.

**Start with ONE rate plan per room type** (the PMS's standard/default
price). Multi-plan sync multiplies every downstream push and confuses
users whose PMS UI shows a single price; add plans only when the PMS
actually models them. Make the choice configurable.

Verify: read back `/room_types` and `/rate_plans` filtered by the
property and confirm titles/counts match the PMS.

### 3. Outbound ARI push

Two endpoints, separate messages: `POST /availability` (per room
type) and `POST /restrictions` (per rate plan: rate, min_stay,
stop_sell, closed_to_arrival, closed_to_departure).

Non-negotiables learned the hard way:

- **Compress ranges.** Sending one value per date for a year is
  thousands of entries. Run-length encode: consecutive dates with
  equal values become one `date_from`/`date_to` entry. (Compare the
  whole value object, not just the rate.)
- **Push deltas, not the world.** Scope every push to what changed:
  - booking created/moved/cancelled → availability only (cheap after
    compression — bookings don't change prices)
  - a price/restriction edit → restrictions for the touched
    room-type/date cells only, and **only the changed fields** —
    Channex applies partial updates, so a price edit can send `{rate}`
    alone without clobbering min-stay/closures
  - property/room-type/rate-plan config change → full push
  - plus a periodic (e.g. hourly) full push as drift correction, so
    no missed event can desynchronize you forever
- **Debounce.** Coalesce a burst of edits (one save = many events in
  most PMSs) into one push a few seconds later, through a job queue
  with retries rather than inline HTTP in a request handler.
- **Never send past dates** — Channex rejects them.
- **Availability semantics:** count holds/blocks as occupied (they're
  not sellable), exclude cancelled bookings, clamp negatives
  (overbooked) to 0.

Verify with a READBACK, not by trusting the 200: `GET /availability`
and `GET /restrictions?...&filter[restrictions]=rate` for sample dates
and compare against locally computed values. (The `filter[restrictions]`
param is required on restrictions reads — you get a 400 without it.)

### 4. Inbound booking feed

Poll `GET /booking_revisions/feed` (every minute is the conventional
cadence), apply each revision, then ack it with
`POST /booking_revisions/:id/ack`. Rules that keep this robust:

- **The feed is a 30-minute window, NOT a durable queue.** An unacked
  revision is re-served for only ~30 minutes (with a warning email),
  then it DROPS OUT of the feed for good. So the feed alone is not a
  safe system of record — anything you fail to ack within 30 minutes
  (poller outage, deploy, a bug that crashes processing) is gone from
  the feed and will never redeliver.
- **Therefore you need TWO paths, not one:**
  1. *Real-time:* drain the feed and ack-after-apply (below).
  2. *Reconciliation:* a periodic sweep (e.g. hourly, and always on
     poller startup after any downtime) that lists `GET /bookings`
     for each managed property and backfills anything missing locally.
     This is the safety net that makes a >30-min outage survivable.
     Skipping it means silently losing bookings — unacceptable for a
     channel manager. Build it from the start, not "later."
- **Drain until empty; don't fall behind the page limit.** The feed
  returns oldest-first and defaults to `limit: 10`. If more than 10
  revisions are waiting (`meta.total > meta.limit`), loop immediately
  rather than waiting for the next tick — a surge (channel backfill,
  reconnect dump) can otherwise outrun a once-per-minute poll and push
  revisions toward the 30-minute cliff.
- **Ack only after applying successfully — but a failure can't block
  the queue OR sit unhandled.** Leave a failed revision un-acked so it
  retries, but keep draining the rest (don't `break` on first error),
  and alert on it — because the 30-minute expiry means a poison
  revision you ignore becomes permanent data loss, not an indefinite
  retry. The reconciliation sweep is the backstop, but loud alerting
  is how you fix the cause before the window closes.
- **The feed is account-wide.** ONE call covers every property the key
  can see (optional `filter[property_id]` exists but you don't need it
  for normal polling). Skip-and-ack revisions for property ids that
  aren't in your mapping table, or stray test properties on the
  account will wedge the poller.
- Revision statuses: `new` → create local booking; `cancelled` → find
  via the booking mapping and cancel; `modified` → safest default is
  log + ack + notify a human, because blindly applying OTA
  modifications (date/room/price changes) to a live calendar needs
  reconciliation UX the PMS probably doesn't have yet. Say so to the
  user instead of silently auto-applying.
- **Never drop an OTA booking.** If the PMS tracks individual rooms
  and none is free, ingest anyway into a flagged/overbooked state —
  a guest with a Booking.com confirmation exists whether or not the
  PMS likes it.
- Dedupe by Channex booking id via the mapping table — feeds can
  resend.
- Map money from major-unit strings ("230.00") into the PMS's money
  type; map `ota_name` into the PMS's source field; keep
  `ota_reservation_code` — staff need it on the phone with the OTA.

Webhooks exist as a lower-latency push alternative to feed polling;
polling is simpler and certifiable, so start there. But note: whether
you poll or use webhooks, the 30-minute feed expiry is the same, so the
`GET /bookings` reconciliation sweep is mandatory either way — it is
not the thing you defer.

## Verification & ongoing health

People trust channel managers with real money — build the checker, not
just the integration:

- A **sync command** (CLI/task) for the initial content sync, runnable
  again safely.
- A **doctor command** that checks: key configured → API reachable →
  every local room type/rate plan has a mapping → sampled
  availability/rates match a live readback → feed reachable + pending
  count → last push job state. Exit non-zero on failure so CI can run
  it. This catches drift that no amount of "the push returned 200"
  can.
- **Testing inbound:** there is NO API to inject test bookings. In the
  Channex dashboard: Applications → add the "Booking CRS" app → create
  a booking manually; it arrives via the feed within a minute. Walk
  the user through this — then cancel it and confirm the cancellation
  flows too.
- For go-live, OTA channel mapping happens in the Channex dashboard
  (not the API), and Channex runs PMS certification tests:
  https://docs.channex.io/api-v.1-documentation/pms-certification-tests.md

## Common traps

- Trusting memory for payload shapes → use `references/api.md` and the
  live `.md` docs.
- Testing rate pushes by checking your own logs instead of a readback.
- One full 365-day push wired to every change event (works in a demo,
  hammers the API in production).
- Two sources of price truth: if the PMS UI shows a different number
  than what's pushed, that's a bug to fix BEFORE connecting OTAs —
  unify them so what staff see is exactly what guests pay.
- Committing the API key to a public repo template.
- Letting in-memory state (caches, unsaved overrides) feed pushes —
  anything that feeds Channex must survive a restart, or document
  loudly that it doesn't yet.
