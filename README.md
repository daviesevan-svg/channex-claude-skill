# Channex skills for Claude Code

A Claude Code plugin marketplace with one (so far) skill:

## `channex-pms-integration`

Teaches Claude to integrate **any PMS codebase** — whatever the
language or framework — with the [Channex](https://channex.io) channel
manager, so availability, rates, and restrictions flow out to OTAs
(Booking.com, Airbnb, Expedia, …) and OTA bookings flow back in.

The skill encodes an architecture and operating discipline proven in a
real integration (the open-source [CorePMS](https://github.com/daviesevan-svg/CorePMS)
hotel PMS, verified end-to-end against Channex staging):

- **Discovery first** — map the PMS's room model, rate source, money
  units, and change signals before writing code
- **Four components in testable order** — API client → ID mapping +
  idempotent content sync → outbound ARI push → inbound booking feed
- **Efficient pushes** — delta scoping (only changed dates AND only
  changed fields — Channex applies partial restriction updates),
  run-length range compression, debouncing, past-date filtering
- **Robust ingestion** — revisions feed + ack-after-apply, dedupe,
  account-wide feed hygiene, cautious handling of OTA modifications,
  never dropping a booking even when it means flagging an overbooking
- **Verification discipline** — readback comparisons instead of
  trusting 200s, a doctor-style health check, and the dashboard
  workflow for test bookings (there's no injection API)
- `references/api.md` — endpoint and payload shapes verified against
  a live staging server, including the quirks that cost debugging time

### Benchmark

Tested by running identical PMS-integration tasks (Django ARI push,
Node/Prisma booking ingestion, a restrictions-clobbering debugging
case) with and without the skill, graded on 16 objective assertions:

| | Assertions passed |
|---|---|
| With skill | **16/16 (100%)** |
| Without skill | 12/16 (75%) |

Baseline failures were the classic traps: no update path after first
sync, blind auto-apply of OTA modifications, no foreign-property guard
on the account-wide feed, no readback verification.

## Install

```
/plugin marketplace add daviesevan-svg/channex-claude-skill
/plugin install channex-pms-integration@channex-skills
```

Or copy `plugins/channex-pms-integration/skills/channex-pms-integration/`
into your project's `.claude/skills/` directory — it's a plain skill
folder, no plugin machinery required.

## License

MIT
