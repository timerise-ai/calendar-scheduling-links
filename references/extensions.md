# Extensions

The core is one host, one-on-one and group event types, availability, calendars, e-mail, reminders and
webhooks. What a team scheduling product adds on top is listed here with the shape it should take, so an
agent asked for one extends the core instead of working around it. **Nothing in this file has templates in
this release**: each row says where the change goes and what must keep holding.

| Extension | Where it goes | Must keep holding |
|---|---|---|
| Round robin | New `event_type_hosts (event_type_id, host_id, priority)`. Slots: union of each host's `computeSlots`. Booking: pick the eligible host with the fewest bookings in the window (or by priority), then `book` for that host. | The per-host lock and the overlap constraint stay per host. Never assign a host the engine did not offer for that start. |
| Collective (all hosts attend) | Same table. Slots: intersection of each host's slots. Booking: one booking row per host with the same `chainId`, all in one transaction that takes every host's lock in a fixed order (sorted host ids). | Locks in sorted order, or two collective bookings deadlock. |
| Routing form | A form before the booking page whose answers pick an event type or a host. Store the answers on the booking as `answers`. | The routing runs on the server; the client never chooses a host the form did not allow. |
| Paid bookings | Sibling skill `bookable-events` for Stripe Checkout, holds and refunds, or `stripe-connect-subscriptions` for a marketplace. The booking is created `pending_payment` and confirmed by the webhook. | A slot held for payment needs an expiry job that releases it. The seat count must include held seats. |
| Meeting polls | Separate entity: proposed times, votes, then a normal booking for the winning time. | The final booking still goes through `createBooking`. |
| Cancellation policy | A minimum notice for cancel and reschedule on the event type, checked in `cancelBooking` and `rescheduleBooking`. | The host can always cancel from the dashboard. |
| SMS reminders | A new job kind beside `reminder.invitee`, with the invitee's phone from `answers`. | Same re-check of status and start before sending; consent to SMS stored with the booking. |
| Host notifications in Slack | Sibling skill `slack-ai-bot`, or a webhook endpoint pointing at a Slack workflow. | Through the jobs table, never from the booking transaction. |
| Who opened a booking link | Sibling skill `visit-logger`. | |
| Walk-up booking on a tablet | Sibling skill `booking-kiosk`, calling this module's service as its backend. | |
| Invoices for paid sessions | Sibling skill `invoicing`. | |
| Several schedules per host | Already supported: an event type points at one schedule. The dashboard only needs a schedule picker. | Busy time and bookings stay per host across schedules. |
| Teams and organizations | Add `org_id` to every table and check it in each dashboard mutation. | `hostId` keeps meaning one person: availability is per person. |

## Prompts that build them

What to tell an agent with this skill installed. Each prompt names the invariants so the agent keeps them.

```text
Add round robin event types: an event type can list several hosts; slots are the union of their free slots,
and a booking goes to the eligible host with the fewest bookings that week. Keep the per-host lock and the
overlap constraint; never assign a host the slot engine did not offer for that start.
```

```text
Add a routing form before the booking page: three questions, and the answers choose which event type the
visitor sees. The routing runs on the server and the answers are saved on the booking.
```

```text
Make the "consultation" event type paid with Stripe Checkout using the bookable-events skill. Hold the slot
for 30 minutes while the visitor pays, release it if they do not, and confirm the booking from the webhook.
```
