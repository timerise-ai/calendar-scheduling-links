# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
