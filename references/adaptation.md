# Adaptation

How the module lands in a host app: what the host must already have, the seams it fills, the order of work,
and the non-negotiables that do not bend.

## The host must already have

- A **Next.js App Router** app on the Node.js runtime (the module uses `node:crypto` and `node:dns`).
- **Signed-in users** who will be hosts. Any auth works behind `currentHostId()`.
- **Supabase** (Postgres) or **Firestore** for data. Supabase is the complete path in this release; Firestore
  ships the booking store, not yet the calendar store.
- A way to **send e-mail** (SendGrid ships; any provider fits behind `Mailer`).
- A **scheduler** that can call a URL every five minutes: Vercel Cron on Pro, or Supabase `pg_cron` on Hobby.

## Seam contract

| Seam | This skill ships | The host supplies |
|---|---|---|
| Domain words | host, event type, schedule, booking, invitee, handle, and a rename table in [data-model.md](data-model.md) | Its own words for each |
| Auth | `currentHostId(): Promise<string \| null>` in `lib/scheduling/host.ts`, Supabase Auth version | Its session lookup |
| Tenant scope | `hostId` on every row, checked in every dashboard mutation | An `org_id` too, if it has teams ([extensions.md](extensions.md)) |
| Data | `SchedulingStore` and `CalendarStore` with Supabase implementations, `SchedulingStore` on Firestore | Its project, and the migration applied |
| E-mail | `Mailer` with a SendGrid adapter, three render functions | Its provider and its wording, language and branding |
| Calendar | Google and Microsoft providers behind `CalendarProvider` | OAuth clients ([operations.md](operations.md)) |
| Scheduler | One cron route | Vercel Cron or `pg_cron` |
| UI | Pages and a widget with structure, behaviour and ARIA, no styling | Its components, layout and design tokens |
| Strings | English literals in the widget and e-mails | Its i18n keys, if it has i18n |
| Rate limiting | Nothing | A limit on the public booking and manage routes |

## Order of work

1. Read the host: auth, data store, e-mail, UI kit, i18n, test runner. Fill the right-hand column above and
   the rename table, and show them to the person asking before writing code.
2. Copy `time.ts`, `types.ts` and `slot-engine.ts`, then the time and engine suites from
   [testing.md](testing.md). Run them. Nothing else depends on the database yet.
3. Apply the migration ([supabase.md](supabase.md)) or set up the Firestore indexes and rules
   ([firestore.md](firestore.md)). Copy the store.
4. Copy `store.ts`, `jobs.ts`, `tokens.ts`, `crypto.ts`, `booking-service.ts`, `validation.ts`, and the
   fixtures and booking suite. Run them.
5. Copy `ics.ts`, `emails.ts`, `mailer.ts`, `webhooks.ts`, `runner.ts`; adapt the e-mail wording; run the
   edges suite.
6. Copy the routes and `runtime.ts`; write `host.ts` for the host's auth.
7. Build the public page, the widget and the manage page with the host's components
   ([booking-page-ui.md](booking-page-ui.md)), then the dashboard.
8. Calendar sync last ([calendar-sync.md](calendar-sync.md)): it needs OAuth clients and a public URL.
9. Wire the cron, set the environment, deploy ([operations.md](operations.md)).

## Non-negotiables

The same five rules as `SKILL.md` and the README, in the same order.

1. **The write recomputes the slot with the engine that listed it.** A start is booked only if `computeSlots`
   offers it at write time.
2. **Every booking write is atomic per host.** Seats, the daily limit and overlap are checked under one lock
   per host, with the overlap constraint as the backstop.
3. **Side effects are jobs, written with the change and run after it.** No e-mail, calendar call or webhook
   inside the transaction; every job idempotent and re-checking the booking.
4. **Instants in storage, zones only at the edges.** Store UTC instants; convert with an explicit IANA zone,
   iterate local dates, never add 24 hours.
5. **Secrets stay sealed and links stay unforgeable.** Manage tokens stored as hashes, OAuth tokens and
   webhook secrets encrypted, OAuth state and channel tokens signed, cron auth failing closed.

Everything else, the look, the words, the provider for e-mail, the extra fields, is the host app's.
