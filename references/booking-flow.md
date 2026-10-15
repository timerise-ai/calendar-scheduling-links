# Booking flow

Listing, booking, rescheduling and cancelling, as four service functions the routes call. The service decides
what to write; the store writes it atomically. The rule that holds it together: **the write recomputes the
slot with the same `computeSlots` the page used, and the store re-checks seats, the daily limit and overlap
under one lock per host.**

## Shape

```
POST /api/scheduling/bookings
  createBooking(deps, body)
    bookingInput.safeParse                      zod: start, name, email, IANA zone, answers
    loadBookable                                event type active, host, schedule
    requireSlot                                 computeSlots for [start, start + 1 min)
    newBookingFor                               id, chainId, blocked range, token hash, day bounds
    jobsFor('created', ...)                     e-mails, calendar, reminders, webhooks
    store.book({ booking, capacity, dailyLimit, dayStart, dayEnd, jobs })   one transaction
  after(() => runDueJobs(...))                  e-mails in seconds; the cron is the guarantee
```

## The store seam

Everything the service and the runner need from a database. [supabase.md](supabase.md) and
[firestore.md](firestore.md) implement it; the in-memory version in [testing.md](testing.md) follows the same
refusals.

```ts
// lib/scheduling/store.ts: the data seam. postgres-store.ts (Supabase) and firestore-store.ts implement it.
import type { JobInput, JobRecord } from '@/lib/scheduling/jobs';
import type { BookedInterval, Booking, EventType, Interval, Schedule } from '@/lib/scheduling/types';

export interface HostProfile {
  id: string;
  handle: string;
  name: string;
  email: string;
  timeZone: string;
}

export interface WebhookEndpoint {
  id: string;
  hostId: string;
  url: string;
  /** Plaintext only in memory: the store decrypts it with decryptSecret. */
  secret: string;
  events: string[];
  active: boolean;
}

/** Everything one booking write needs, decided by the service before the store is called. */
export interface NewBooking {
  /** The service mints the id, so the jobs written with the booking can name it. */
  booking: Omit<Booking, 'createdAt' | 'updatedAt'>;
  capacity: number;
  /** null for no limit. With dayStart and dayEnd, the host's local day of the start. */
  dailyLimit: number | null;
  dayStart: Date;
  dayEnd: Date;
  /** Jobs written in the same transaction as the booking: e-mails, calendar, webhooks, reminders. */
  jobs: JobInput[];
}

export type BookResult =
  | { ok: true; booking: Booking }
  | { ok: false; reason: 'slot_taken' | 'slot_full' | 'day_full' };

export type ChangeResult =
  | { ok: true; booking: Booking }
  | { ok: false; reason: 'not_found' | 'not_confirmed' | 'slot_taken' | 'slot_full' | 'day_full' };

export interface SchedulingStore {
  getHostByHandle(handle: string): Promise<HostProfile | null>;
  getHost(hostId: string): Promise<HostProfile | null>;
  getEventTypeBySlug(hostId: string, slug: string): Promise<EventType | null>;
  getEventType(id: string): Promise<EventType | null>;
  getSchedule(id: string): Promise<Schedule | null>;
  /** Confirmed bookings of the host, every event type, whose blocked range overlaps the window. */
  listBookedIntervals(hostId: string, window: Interval): Promise<BookedInterval[]>;
  /** Busy blocks mirrored from the host's connected calendars, overlapping the window. */
  listBusy(hostId: string, window: Interval): Promise<Interval[]>;
  getBooking(id: string): Promise<Booking | null>;
  /** Active endpoints of the host subscribed to an event such as 'booking.created'. */
  listWebhookEndpointIds(hostId: string, event: string): Promise<string[]>;
  getWebhookEndpoint(id: string): Promise<WebhookEndpoint | null>;
  getBookingByTokenHash(tokenHash: string): Promise<Booking | null>;

  /** Atomic: capacity, daily limit and the overlap constraint are checked under one lock per host. */
  book(input: NewBooking): Promise<BookResult>;
  /** Atomic: the new booking is written and the old one cancelled together, or neither. */
  reschedule(oldId: string, input: NewBooking): Promise<ChangeResult>;
  /** Atomic: status, pending reminder jobs of the booking deleted, new jobs added. */
  cancel(id: string, reason: string | null, jobs: JobInput[]): Promise<ChangeResult>;
  /** Fields written back by jobs after the provider answered. */
  setBookingExternal(id: string, fields: { externalEventId?: string | null; meetingUrl?: string | null }): Promise<void>;

  /** Claims due jobs with a lease, so overlapping cron runs never run one job twice. */
  claimJobs(limit: number, now: Date, leaseSeconds: number): Promise<JobRecord[]>;
  completeJob(id: string): Promise<void>;
  failJob(id: string, error: string, retryAt: Date | null): Promise<void>;
}
```

| Method | Must hold |
|---|---|
| `book` | Under one lock per host: refuse `slot_full` when the slot's seats are taken, `day_full` when a new slot would pass the daily limit, `slot_taken` when the blocked range overlaps another slot's booking. Booking and jobs commit together or not at all. |
| `reschedule` | Cancel the old booking, delete its pending reminders, insert the new one and its jobs, in one transaction. Any refusal leaves the old booking confirmed. The old booking's own range does not block the new one. |
| `cancel` | Refuse anything not confirmed. Bump `sequence`, delete pending reminders, add jobs. |
| `claimJobs` | Lease due jobs so two overlapping runs never hold the same job. A crashed run's jobs come back when the lease ends. |

## The service

```ts
// lib/scheduling/booking-service.ts: list slots, book, reschedule, cancel. Routes call only these.
// The same computeSlots call decides what the page lists and what the write accepts.
import { randomUUID } from 'node:crypto';
import { z } from 'zod';
import { jobsFor } from '@/lib/scheduling/jobs';
import { computeSlots } from '@/lib/scheduling/slot-engine';
import type { HostProfile, NewBooking, SchedulingStore } from '@/lib/scheduling/store';
import { addDays, isValidTimeZone, localDateOf, zonedToInstant } from '@/lib/scheduling/time';
import { hashToken, manageTokenFor } from '@/lib/scheduling/tokens';
import { slotKeyOf, type Booking, type EventType, type Schedule, type Slot } from '@/lib/scheduling/types';

const DAY = 86_400_000;
/** Widest window one slots request may ask for. The page asks for one month at a time. */
export const MAX_WINDOW_DAYS = 62;

export class SchedulingError extends Error {
  constructor(
    public readonly code:
      | 'not_found'
      | 'invalid_input'
      | 'slot_unavailable'
      | 'slot_taken'
      | 'slot_full'
      | 'day_full'
      | 'not_confirmed'
      | 'too_late',
    message?: string,
  ) {
    super(message ?? code);
  }
}

export interface ServiceDeps {
  store: SchedulingStore;
  now: () => Date;
}

interface Bookable {
  host: HostProfile;
  eventType: EventType;
  schedule: Schedule;
}

async function loadBookable(deps: ServiceDeps, eventTypeId: string): Promise<Bookable> {
  const eventType = await deps.store.getEventType(eventTypeId);
  if (!eventType || !eventType.active) throw new SchedulingError('not_found');
  const [host, schedule] = await Promise.all([
    deps.store.getHost(eventType.hostId),
    deps.store.getSchedule(eventType.scheduleId),
  ]);
  if (!host || !schedule) throw new SchedulingError('not_found');
  return { host, eventType, schedule };
}

async function slotsIn(deps: ServiceDeps, b: Bookable, from: Date, to: Date, ignoreBookingId?: string): Promise<Slot[]> {
  // Widen the read by a day each side: buffers and ranges that cross midnight reach outside the window.
  const window = { start: new Date(from.getTime() - DAY), end: new Date(to.getTime() + DAY) };
  const [bookings, busy] = await Promise.all([
    deps.store.listBookedIntervals(b.host.id, window),
    deps.store.listBusy(b.host.id, window),
  ]);
  return computeSlots({
    eventType: b.eventType,
    schedule: b.schedule,
    // A reschedule must not be blocked by the booking it replaces.
    bookings: ignoreBookingId ? bookings.filter(x => x.id !== ignoreBookingId) : bookings,
    busy,
    now: deps.now(),
    from,
    to,
  });
}

function checkWindow(from: Date, to: Date): void {
  if (!(from < to) || to.getTime() - from.getTime() > MAX_WINDOW_DAYS * DAY) {
    throw new SchedulingError('invalid_input', 'window');
  }
}

export async function listSlots(deps: ServiceDeps, eventTypeId: string, from: Date, to: Date): Promise<Slot[]> {
  checkWindow(from, to);
  return slotsIn(deps, await loadBookable(deps, eventTypeId), from, to);
}

/**
 * The times a reschedule can move to: the same engine as rescheduleBooking, without the booking being moved.
 * The public listing counts that booking, so it would hide every time that overlaps it.
 */
export async function listRescheduleSlots(deps: ServiceDeps, token: string, from: Date, to: Date): Promise<Slot[]> {
  checkWindow(from, to);
  const old = await bookingForToken(deps, token);
  if (old.status !== 'confirmed') throw new SchedulingError('not_confirmed');
  return slotsIn(deps, await loadBookable(deps, old.eventTypeId), from, to, old.id);
}

export const bookingInput = z.object({
  eventTypeId: z.string().min(1).max(64),
  start: z.iso.datetime({ offset: true }),
  name: z.string().trim().min(1).max(120),
  email: z.email().max(254),
  timeZone: z.string().refine(isValidTimeZone, 'Unknown time zone'),
  answers: z.record(z.string().max(64), z.string().max(2000)).default({}),
});
export type BookingInput = z.infer<typeof bookingInput>;

function newBookingFor(
  b: Bookable,
  slot: Slot,
  invitee: Pick<BookingInput, 'name' | 'email' | 'timeZone' | 'answers'>,
  id: string,
  extra: { sequence: number; rescheduledFrom: Booking | null },
): Omit<NewBooking, 'jobs'> {
  const et = b.eventType;
  const day = localDateOf(slot.start, b.schedule.timeZone);
  const old = extra.rescheduledFrom;
  return {
    booking: {
      id,
      chainId: old?.chainId ?? id,
      hostId: b.host.id,
      eventTypeId: et.id,
      slotKey: slot.key,
      start: slot.start,
      end: slot.end,
      blockedStart: new Date(slot.start.getTime() - et.bufferBeforeMinutes * 60_000),
      blockedEnd: new Date(slot.end.getTime() + et.bufferAfterMinutes * 60_000),
      status: 'confirmed',
      inviteeName: invitee.name,
      inviteeEmail: invitee.email.toLowerCase(),
      inviteeTimeZone: invitee.timeZone,
      answers: invitee.answers,
      manageTokenHash: manageTokenFor(id).hash,
      sequence: extra.sequence,
      rescheduledFromId: old?.id ?? null,
      cancelReason: null,
      // The host's calendar event and its meeting link move with the booking instead of being re-created.
      externalEventId: old?.externalEventId ?? null,
      meetingUrl: old?.meetingUrl ?? null,
    },
    capacity: et.capacity,
    dailyLimit: et.dailyLimit,
    dayStart: zonedToInstant(day, '00:00', b.schedule.timeZone),
    dayEnd: zonedToInstant(addDays(day, 1), '00:00', b.schedule.timeZone),
  };
}

function onlyKnownAnswers(et: EventType, answers: Record<string, string>): Record<string, string> {
  const out: Record<string, string> = {};
  for (const q of et.questions) {
    const v = answers[q.id]?.trim() ?? '';
    if (q.required && !v) throw new SchedulingError('invalid_input', `answer:${q.id}`);
    if (v) out[q.id] = v;
  }
  return out;
}

async function requireSlot(deps: ServiceDeps, b: Bookable, start: Date, ignoreBookingId?: string): Promise<Slot> {
  const slots = await slotsIn(deps, b, start, new Date(start.getTime() + 60_000), ignoreBookingId);
  const slot = slots.find(s => s.key === slotKeyOf(b.eventType.id, start));
  if (!slot) throw new SchedulingError('slot_unavailable');
  return slot;
}

/** Returns the booking and the manage token, which the caller shows once and the e-mail carries. */
export async function createBooking(deps: ServiceDeps, raw: unknown): Promise<{ booking: Booking; manageToken: string }> {
  const parsed = bookingInput.safeParse(raw);
  if (!parsed.success) throw new SchedulingError('invalid_input', parsed.error.issues[0]?.path.join('.'));
  const input = parsed.data;
  const b = await loadBookable(deps, input.eventTypeId);
  const slot = await requireSlot(deps, b, new Date(input.start));
  const id = randomUUID();
  const base = newBookingFor(b, slot, { ...input, answers: onlyKnownAnswers(b.eventType, input.answers) }, id, {
    sequence: 0,
    rescheduledFrom: null,
  });
  const endpoints = await deps.store.listWebhookEndpointIds(b.host.id, 'booking.created');
  const result = await deps.store.book({ ...base, jobs: jobsFor('created', base.booking, endpoints, deps.now()) });
  if (!result.ok) throw new SchedulingError(result.reason);
  return { booking: result.booking, manageToken: manageTokenFor(id).token };
}

export async function bookingForToken(deps: ServiceDeps, token: string): Promise<Booking> {
  const booking = await deps.store.getBookingByTokenHash(hashToken(token));
  if (!booking) throw new SchedulingError('not_found');
  return booking;
}

/** A reschedule is a new booking that replaces the old one: its own id, history and manage token. */
export async function rescheduleBooking(
  deps: ServiceDeps,
  token: string,
  newStartIso: string,
): Promise<{ booking: Booking; manageToken: string }> {
  const old = await bookingForToken(deps, token);
  if (old.status !== 'confirmed') throw new SchedulingError('not_confirmed');
  if (old.start.getTime() <= deps.now().getTime()) throw new SchedulingError('too_late');
  const start = new Date(newStartIso);
  if (Number.isNaN(start.getTime())) throw new SchedulingError('invalid_input', 'start');
  const b = await loadBookable(deps, old.eventTypeId);
  const slot = await requireSlot(deps, b, start, old.id);
  const id = randomUUID();
  const base = newBookingFor(
    b,
    slot,
    { name: old.inviteeName, email: old.inviteeEmail, timeZone: old.inviteeTimeZone, answers: old.answers },
    id,
    { sequence: old.sequence + 1, rescheduledFrom: old },
  );
  const endpoints = await deps.store.listWebhookEndpointIds(b.host.id, 'booking.rescheduled');
  // calendar.upsert on the new booking patches the event the old one created: same chain, same event.
  const jobs = jobsFor('rescheduled', base.booking, endpoints, deps.now());
  const result = await deps.store.reschedule(old.id, { ...base, jobs });
  if (!result.ok) throw new SchedulingError(result.reason);
  return { booking: result.booking, manageToken: manageTokenFor(id).token };
}

export async function cancelBooking(deps: ServiceDeps, token: string, reason: string | null): Promise<Booking> {
  const booking = await bookingForToken(deps, token);
  if (booking.status !== 'confirmed') throw new SchedulingError('not_confirmed');
  if (booking.end.getTime() <= deps.now().getTime()) throw new SchedulingError('too_late');
  const endpoints = await deps.store.listWebhookEndpointIds(booking.hostId, 'booking.cancelled');
  const jobs = jobsFor('cancelled', { ...booking, sequence: booking.sequence + 1 }, endpoints, deps.now());
  const result = await deps.store.cancel(booking.id, reason?.slice(0, 500) ?? null, jobs);
  if (!result.ok) throw new SchedulingError(result.reason);
  return result.booking;
}
```

## Decisions in this file

| Decision | Reason |
|---|---|
| The service mints the booking id | The jobs written in the same transaction name it. No placeholder to patch later. |
| The manage token is HMAC(secret, booking id) | A reminder job three weeks later can rebuild the link. The database holds only the SHA-256, so a leaked table cancels nothing. |
| A reschedule is a new booking | Each booking keeps its own row, token and history; the chain id keeps one calendar event and one ICS UID across them. |
| Reschedule and cancel are refused once the booking has started or ended | A reschedule after the start, or a cancel after the end, would rewrite what already happened. |
| Answers are filtered to the event type's questions | Unknown keys are dropped before they reach a template or a webhook. |
| `MAX_WINDOW_DAYS = 62` | One request lists at most two months; the widget asks for one at a time. |

## Errors

`SchedulingError.code` is the whole contract with the routes; [api-routes.md](api-routes.md) maps it to HTTP.

| Code | When | HTTP |
|---|---|---|
| `invalid_input` | Body fails `bookingInput`, a required answer is empty, the window is wrong | 400 |
| `not_found` | Unknown or inactive event type, unknown token | 404 |
| `slot_unavailable` | The engine does not offer that start now | 409 |
| `slot_taken`, `slot_full`, `day_full` | The store refused under the lock: someone else was faster | 409 |
| `not_confirmed` | Cancel or reschedule of a booking that is already cancelled or moved | 409 |
| `too_late` | Reschedule after the start, cancel after the end | 409 |

## Host-side validation

The engine trusts the shapes it is given. These schemas are what the host's dashboard may save.

```ts
// lib/scheduling/validation.ts: what the host may save. The slot engine trusts these shapes.
import { z } from 'zod';
import { isLocalDate, isValidTimeZone, isWallTime, wallMinutes } from '@/lib/scheduling/time';

const wall = z.string().refine(isWallTime, 'HH:MM, 00:00 to 24:00');
const range = z
  .object({ start: wall, end: wall })
  .refine(r => wallMinutes(r.start) < wallMinutes(r.end), 'start must be before end');

function noOverlap(ranges: { start: string; end: string }[]): boolean {
  const sorted = [...ranges].sort((a, b) => wallMinutes(a.start) - wallMinutes(b.start));
  return sorted.every((r, i) => i === 0 || wallMinutes((sorted[i - 1] as { end: string }).end) <= wallMinutes(r.start));
}

export const scheduleInput = z
  .object({
    name: z.string().trim().min(1).max(80),
    timeZone: z.string().refine(isValidTimeZone, 'Unknown time zone'),
    rules: z.array(range.and(z.object({ weekday: z.number().int().min(0).max(6) }))).max(70),
    overrides: z
      .array(z.object({ date: z.string().refine(isLocalDate, 'YYYY-MM-DD'), ranges: z.array(range).max(10) }))
      .max(366),
  })
  .refine(s => [0, 1, 2, 3, 4, 5, 6].every(d => noOverlap(s.rules.filter(r => r.weekday === d))), 'Ranges on one weekday overlap')
  .refine(s => s.overrides.every(o => noOverlap(o.ranges)), 'Ranges in an override overlap')
  .refine(s => new Set(s.overrides.map(o => o.date)).size === s.overrides.length, 'One override per date');

export const eventTypeInput = z
  .object({
    scheduleId: z.string().min(1),
    slug: z.string().regex(/^[a-z0-9][a-z0-9-]{0,62}$/),
    title: z.string().trim().min(1).max(120),
    description: z.string().max(2000).default(''),
    kind: z.enum(['one_on_one', 'group']),
    capacity: z.number().int().min(1).max(500),
    durationMinutes: z.number().int().min(5).max(720),
    slotIntervalMinutes: z.number().int().min(5).max(720),
    bufferBeforeMinutes: z.number().int().min(0).max(240),
    bufferAfterMinutes: z.number().int().min(0).max(240),
    minNoticeMinutes: z.number().int().min(0).max(60 * 24 * 60),
    horizonDays: z.number().int().min(1).max(365),
    dailyLimit: z.number().int().min(1).max(100).nullable(),
    locationKind: z.enum(['google_meet', 'teams', 'phone', 'in_person', 'custom_link']),
    locationValue: z.string().max(500).nullable(),
    questions: z
      .array(z.object({ id: z.string().regex(/^[a-z0-9_]{1,40}$/), label: z.string().min(1).max(200), type: z.enum(['text', 'textarea', 'phone']), required: z.boolean() }))
      .max(10),
    active: z.boolean(),
  })
  .refine(e => e.kind === 'group' || e.capacity === 1, 'A one-on-one event has capacity 1');
```
