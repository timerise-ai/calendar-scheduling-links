# Provenance

This module is the scheduling core of a booking platform, written down by the engineer who builds it: the
slot generation, booking, notification and calendar-sync layers of the earlier implementation, audited end
to end and rebuilt for Next.js on Vercel with Supabase or Firestore. The earlier implementation was a
long-running API server on a document database. The scope map came from the public feature set of the
scheduling-link products the module is an alternative to; the behaviour came from the earlier
implementation and from the audit.

This file is the ledger: what the audit changed and how the templates verify it, what was kept deliberately
and why, and what was designed here and has never run in production. 38 entries.

## Fixed in the templates

### Slots

1. **Days were reached by adding 24 hours to an instant.** On a 23-hour spring-forward day, a walk that
   started
   late in the evening skipped a whole local date. The engine now iterates local dates with `addDays`
   ([time-and-slots.md](time-and-slots.md)); the spring-forward test walks from 23:45 local.
2. **The "fifth weekday of the month" rule checked the wrong length.** It returned nothing in months with a
   fifth such weekday and failed in months without one. Monthly patterns are not in this release; when one is
   added it needs a test for months with four and five occurrences.
3. **The booking-time re-check was a different function from the listing.** It overwrote a service-level
   block with the result of the host-level check, and read at most one page of blocks. The booking route now
   calls `computeSlots` for the one start ([booking-flow.md](booking-flow.md)); a test books what the page
   listed and refuses what it did not.
4. **Blocks were stored as wall-clock times dressed as UTC and read back in another zone.** Busy time from a
   calendar in a different zone shifted by the difference. Everything is stored as instants; zones appear
   only in `zonedToInstant` and `localDateOf`.
5. **Slot data lived in per-process caches fed by realtime listeners.** That does not survive serverless.
   Each request reads the window it needs; the engine is pure.

### Booking

6. **The capacity check counted bookings, not seats, and ran partly outside the transaction.** The store now
   counts seats under a per-host lock, with an exclusion constraint as the backstop
   ([supabase.md](supabase.md)); the contract suite sends four concurrent requests and expects one winner.
7. **Free-range bookings had no transactional protection at all.** There is one booking path; every booking
   has a slot key and goes through the same lock and constraint.
8. **A cancelled booking could be reopened without a capacity check.** There is no reopen transition; a
   cancelled booking stays cancelled.
9. **Cancel and reschedule needed only the booking id.** A confirm token was generated and never checked. The
   manage token is now the only credential, derived with a server secret and stored as a hash.
10. **A reschedule fired no side effects and was not atomic.** No e-mail, webhook, reminder or calendar
    change,
    and a crash between the two writes left two live bookings. `csl_reschedule` does both writes and the jobs
    in one transaction; a test checks the chain, the reminders and the calendar patch.
11. **Old reminders fired at the old time after a reschedule.** Reminders are deleted in the same transaction,
    and the reminder handler re-checks the start; a test re-inserts a stale reminder and expects silence.
12. **Plan and promotion limits were checked outside any transaction.** Not part of this module; any such
    limit added later belongs inside the store's transaction.
13. **Bookings could be cancelled after they started.** Reschedule is refused after the start, cancel after
    the end.

### Notifications

14. **One failed e-mail stopped the steps after it.** Each effect is its own job, retried on its own.
15. **Reminders were queued again on every status pass, and the claim was not atomic.** One reminder per
    booking, start and offset (the dedupe key), claimed with a lease.
16. **Reminders went out for cancelled bookings.** Deleted on cancel, and re-checked before sending.
17. **Failed reminders were never retried.** Every job retries with backoff and keeps its last error.
18. **Invitee input reached e-mail HTML unescaped, and form fields could overwrite system values such as the
    confirmation link.** `escapeHtml` on every value; answers live under their own keys and are filtered to
    the event type's questions. A test books with markup in the name.
19. **E-mail times were formatted in the server's zone.** Every format call names the zone.
20. **The ICS had a random UID, no SEQUENCE and no METHOD, so a change added a second event.** UID from the
    chain id, SEQUENCE from the booking, REQUEST or CANCEL
    ([jobs-and-notifications.md](jobs-and-notifications.md)).
21. **The ICS download needed only the booking id and carried the invitee's details.** It now needs the
    manage token.
22. **Outgoing webhooks were unsigned and tried once.** HMAC signature with a timestamp, retries with backoff,
    a stable id per delivery.
23. **The webhook URL guard followed redirects, allowed a failed DNS lookup, and missed IPv6 private ranges
    and
    CGNAT.** `redirect: 'manual'`, refusal on lookup failure, the wider range list, and a test for each range.

### Calendar

24. **The OAuth state was an unsigned user id and the callback did not check the session.** Anyone could bind
    their calendar to another account. State signed and bound to the signed-in host, plus PKCE
    ([calendar-sync.md](calendar-sync.md)).
25. **OAuth tokens were stored in plaintext.** AES-256-GCM, with a test that a tampered value is refused.
26. **A revoked grant was never noticed: writes silently stopped.** `invalid_grant` flags the connection
    `needs_reauth` and drops its busy time.
27. **One provider's refresh token could be empty after the first refresh, and a process-wide token cache was
    shared across users.** Plain OAuth token endpoints, refresh tokens stored per connection, compare-and-set
    on refresh.
28. **The Microsoft authority admitted work and school accounts only.** `common` by default.
29. **Busy events were read without paging past the first page on one provider, and a short read then deleted
    real blocks.** Both providers page to the end before the blocks are replaced.
30. **Free, tentative-free, declined and cancelled events blocked time; multi-day all-day events blocked one
    day on one provider and nothing on the other.** `googleEventsToBusy` and `graphEventsToBusy`, each with a
    test.
31. **The app's own calendar events came back as busy time and blocked group seats.** A private marker on
    every event the app writes, left out of busy time.
32. **A retried calendar write created a second event, and a reschedule had no update path.** Event ids
    derive from the chain: Google's client id, Microsoft's `transactionId` and marker lookup; a reschedule
    patches.
33. **Watch renewal stopped the old channel before starting the new one, and a failed re-create was never
    retried.** New first, then stop the old, then a sync.
34. **Push handlers did the whole sync before answering, and a channel token was not bound to its channel.**
    The handler verifies, marks the connection and answers; the sync runs under a lease. The token names its
    channel and is checked against it.
35. **The cron route accepted any request when its secret was unset, and compared in non-constant time.**
    `isAuthorizedCron` fails closed and uses `timingSafeEqual`; a test passes an unset secret.

## Kept deliberately

36. **Busy time is mirrored into the database, not fetched per request.** Slot queries never wait on a
    provider
    and keep working through its outage. The cost is up to one sync of delay; the cron re-reads any connection
    older than 30 minutes.
37. **A reschedule is a new booking that cancels the old one**, not an edit in place: each keeps its own row,
    token and history. The chain id keeps one calendar event and one ICS UID.
38. **Small decisions that look odd:** capacity is derived from bookings, with no counter to drift; reminders
    are absolute times polled by a cron, not provider-side schedules; the ICS attachment type is bare
    `text/calendar` because SendGrid refuses a `;` in an attachment type; the channel signature is 22
    base64url characters (132 bits) so it fits Graph's 128-character `clientState`; the host's calendar
    receives the invitee as an accepted attendee with `sendUpdates=none`, because the app sends its own
    invitation.

## Designed here, not yet run in production

| Addition | Verified by |
|---|---|
| The per-host advisory lock and the `slot_key <>` exclusion constraint | The store contract on local Supabase |
| The jobs table, leases and backoff | The booking suite and the contract suite |
| The Firestore store with a per-host lock document | The store contract on the Firestore emulator |
| Daily limits, buffers that add up, group seats in one engine | The engine suite |
| Derived manage tokens | The edges suite |
| The embed script and its message checks | Reading; no browser test ships |
| Renewing Microsoft subscriptions by re-creating them | Reading against the Graph documentation; not run against a live tenant |
| The `pg_cron` schedule for Vercel Hobby | Reading against the Supabase documentation |

## If you are rebuilding the earlier implementation

Fix in this order: the OAuth state binding (24), the manage token (9), webhook signing and the URL guard (22,
23), the cron guard (35), then booking atomicity (6, 7, 10), then the calendar read (29 to 31), then the rest.
