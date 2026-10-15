# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.2] - 2026-10-15

Fix release, from scoring the prompt-1 agent eval runs against 0.1.1. Apps built from 0.1.0 or 0.1.1 should
copy `booking-service.ts`, `booking-widget.tsx` and the new manage slots route.

### Fixed

- The reschedule widget listed times from the public list, which counts the booking being moved, so it hid
  every time overlapping it although `rescheduleBooking` accepts them. `listRescheduleSlots` and
  `/api/scheduling/manage/[token]/slots` list without it (`booking-flow.md`, `api-routes.md`).

### Changed

- The vitest config is `vitest.config.mts`, so Vite loads it as ESM in an app without `"type": "module"`.
- The handover in `SKILL.md` names the migration, the cron, the e-mail sender and each OAuth app with its
  redirect URI.
- `booking-page-ui.md` says the widget stays on the page after a reschedule, and why.
- 53 unit tests (booking suite 14).

## [0.1.1] - 2026-10-15

Fix release, from scoring the prompt-1 agent eval runs against 0.1.0. Apps built from 0.1.0 should copy
`runner.ts`, `calendar/writer.ts`, `calendar/sync.ts`, `booking-widget.tsx` and `cancel-form.tsx`.

### Fixed

- The host's "new booking" e-mail went out for a booking cancelled before its job ran. The host handler now
  has the invitee handler's guard; a booking-suite test fails on the 0.1.0 runner
  (`jobs-and-notifications.md`).
- The widget's time zone field threw `RangeError` from `Intl.DateTimeFormat` on a partial zone name and
  took the widget down. It now edits a draft and applies only a valid zone (`booking-page-ui.md`).
- A rejected `fetch` left the widget on "Loading times" or a disabled button, and the cancel form in
  "sending". Both now end in the error state.
- The store contract could not compile in a Supabase app without `firebase-admin` and the Firestore store.
  It is now `test/store-contract.ts`, run by one test file per store (`testing-flows.md`).

### Added

- The host's e-mail carries the ICS invite, without the manage link, unless the module writes the booking
  into a calendar the host connected. `CalendarWriter` gains `writes(hostId)`; a host's own implementation
  adds it. 52 unit tests (booking suite 13).

### Changed

- `host.ts` is used unchanged on Supabase Auth; the quick start, `adaptation.md` and `api-routes.md` say so.
- The environment names in `operations.md` are canonical, the `NEXT_PUBLIC_SUPABASE_*` pair is marked
  required on Supabase Auth, an `.env.example` lists every variable with an empty value, and the final
  message itself names every variable and the provider setup (`SKILL.md`, `operations.md`).

## [0.1.0] - 2026-10-15

First release: self-hosted scheduling links for Next.js App Router on Vercel, with Supabase or Firestore.

### Added
- The slot engine and time helpers: weekly rules, date overrides, buffers, minimum notice, rolling horizon,
  daily limits, group seats, busy time, DST in both directions (`time-and-slots.md`).
- The booking service with the `SchedulingStore` seam: book, reschedule on one chain, cancel by manage token
  (`booking-flow.md`).
- Supabase: migration with server-only tables, the per-host lock, the `slot_key` exclusion constraint and the
  atomic SQL functions; booking and calendar stores (`supabase.md`).
- Firestore: the booking store with a per-host lock document, indexes and rules (`firestore.md`).
- Jobs for e-mail with ICS, reminders, calendar writes and signed webhooks, with leases and backoff
  (`jobs-and-notifications.md`); webhook delivery only to checked public addresses, pinned against DNS
  rebinding (`webhooks.md`).
- Google Calendar and Microsoft 365 sync: OAuth with signed state and PKCE, sealed tokens, mirrored busy
  time, watches, write-back on the booking's chain, disconnect (`calendar-sync.md`).
- Routes, the public page, the widget, the manage page and the embed script (`api-routes.md`,
  `booking-page-ui.md`).
- Operations for Vercel Cron and Supabase `pg_cron`, provider setup and monitoring (`operations.md`).
- Extensions with prompts (`extensions.md`), the seam contract (`adaptation.md`), the audit ledger
  (`provenance.md`).
- Five suites: 51 unit tests and a 5-test store contract run on local Supabase and the Firestore emulator.
