# Testing

Five suites, run with Vitest. The four unit suites need nothing external; the store contract runs against
the in-memory store always, against the Firestore emulator when `FIRESTORE_EMULATOR_HOST` is set, and against
Supabase when `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` are set.

| Suite | Tests | Proves |
|---|---|---|
| `test/time.test.ts` | 8 | Wall clock to instant in both DST directions, `24:00`, a 5:45 offset, date arithmetic, validation |
| `test/slot-engine.test.ts` | 13 | Rules, notice, horizon, overrides, cross-type conflicts, buffers adding up, busy time, seats, daily limit, the spring-forward walk, the fall-back day, page-and-write agreement, the candidate cap |
| `test/booking.test.ts` | 12 | Token hash only, answer filtering, off-grid refusal, one winner of two, input validation, jobs written with the booking and sent once, HTML escaping, retry with backoff, atomic reschedule on one chain, the old booking kept on refusal, cancel by token, stale reminders skipped, group seats |
| `test/edges.test.ts` | 14 | ICS UID, SEQUENCE, CANCEL, escaping and folding; webhook signatures and the private-address guard; derived tokens, fail-closed cron auth, signed values, channel token length, sealed secrets; Google and Graph normalization; Google event ids |
| `test/store-contract.test.ts` | 5 per store | Booking and refusals, four concurrent requests with one winner, group seats with a daily limit, reschedule into an overlap, job leases |

47 unit tests, plus 5 contract tests for each store that runs.

## Setup

```bash
npm i -D vitest
```

```ts
// vitest.config.ts
import { fileURLToPath } from 'node:url';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  resolve: { alias: { '@': fileURLToPath(new URL('.', import.meta.url)) } },
  test: { environment: 'node', include: ['test/**/*.test.ts'] },
});
```

Add `"test": "vitest run"` to `package.json`.

## Fixtures

The in-memory store refuses exactly what the SQL functions refuse, so service tests mean something for both
real stores.

```ts
// test/fixtures.ts: an in-memory SchedulingStore with the same refusals as the SQL functions, a recording
// mailer and calendar, and a movable clock.
import type { CalendarWriter } from '@/lib/scheduling/calendar/writer';
import type { JobInput, JobRecord } from '@/lib/scheduling/jobs';
import type { MailMessage, Mailer } from '@/lib/scheduling/mailer';
import type { HostProfile, NewBooking, SchedulingStore, WebhookEndpoint } from '@/lib/scheduling/store';
import type { Booking, EventType, Interval, Schedule, Weekday } from '@/lib/scheduling/types';

export const TZ = 'Europe/Warsaw';
process.env.SCHEDULING_TOKEN_SECRET = 'test-secret-test-secret-test-secret-0123';
process.env.SCHEDULING_ENCRYPTION_KEY = Buffer.alloc(32, 7).toString('base64');

export function clock(iso: string) {
  const c = { t: new Date(iso).getTime(), now: () => new Date(c.t), advance: (min: number) => (c.t += min * 60_000) };
  return c;
}

export const host: HostProfile = { id: 'h1', handle: 'ana', name: 'Ana Host', email: 'ana@example.com', timeZone: TZ };

export const schedule: Schedule = {
  id: 's1',
  hostId: 'h1',
  name: 'Working hours',
  timeZone: TZ,
  rules: ([1, 2, 3, 4, 5] as Weekday[]).map(weekday => ({ weekday, start: '09:00', end: '17:00' })),
  overrides: [],
};

export function eventType(over: Partial<EventType> = {}): EventType {
  return {
    id: 'et1',
    hostId: 'h1',
    scheduleId: 's1',
    slug: 'intro',
    title: 'Intro call',
    description: '',
    kind: 'one_on_one',
    capacity: 1,
    durationMinutes: 30,
    slotIntervalMinutes: 30,
    bufferBeforeMinutes: 0,
    bufferAfterMinutes: 0,
    minNoticeMinutes: 0,
    horizonDays: 30,
    dailyLimit: null,
    locationKind: 'google_meet',
    locationValue: null,
    questions: [{ id: 'topic', label: 'What should we cover?', type: 'textarea', required: false }],
    active: true,
    ...over,
  };
}

const overlaps = (a: Interval, b: Interval) => a.start < b.end && b.start < a.end;

export function memoryStore(eventTypes: EventType[] = [eventType()]) {
  const bookings = new Map<string, Booking>();
  const jobs: (JobRecord & { done: boolean; leaseUntil: number; lastError: string | null })[] = [];
  const busy: Interval[] = [];
  const endpoints = new Map<string, WebhookEndpoint>();
  let seq = 0;

  function addJobs(list: JobInput[]) {
    for (const j of list) {
      if (jobs.some(x => x.dedupeKey === j.dedupeKey)) continue;
      jobs.push({ id: String(++seq), kind: j.kind, payload: structuredClone(j.payload), runAt: j.runAt, attempts: 0, dedupeKey: j.dedupeKey, done: false, leaseUntil: 0, lastError: null });
    }
  }

  function insert(input: NewBooking): 'ok' | 'slot_taken' | 'slot_full' | 'day_full' {
    const b = input.booking;
    const live = [...bookings.values()].filter(x => x.status === 'confirmed');
    const taken = live.filter(x => x.slotKey === b.slotKey).length;
    if (taken >= input.capacity) return 'slot_full';
    if (input.dailyLimit !== null && taken === 0) {
      const day = new Set(live.filter(x => x.eventTypeId === b.eventTypeId && x.start >= input.dayStart && x.start < input.dayEnd).map(x => x.slotKey));
      if (day.size >= input.dailyLimit) return 'day_full';
    }
    const blocked = { start: b.blockedStart, end: b.blockedEnd };
    if (live.some(x => x.hostId === b.hostId && x.slotKey !== b.slotKey && overlaps(blocked, { start: x.blockedStart, end: x.blockedEnd }))) return 'slot_taken';
    bookings.set(b.id, { ...structuredClone(b), createdAt: new Date(), updatedAt: new Date() });
    addJobs(input.jobs);
    return 'ok';
  }

  const store: SchedulingStore = {
    async getHostByHandle(h) { return h === host.handle ? host : null; },
    async getHost(id) { return id === host.id ? host : null; },
    async getEventTypeBySlug(_h, slug) { return eventTypes.find(e => e.slug === slug) ?? null; },
    async getEventType(id) { return eventTypes.find(e => e.id === id) ?? null; },
    async getSchedule(id) { return id === schedule.id ? schedule : null; },
    async listBookedIntervals(hostId, w) {
      return [...bookings.values()]
        .filter(b => b.hostId === hostId && b.status === 'confirmed' && overlaps({ start: b.blockedStart, end: b.blockedEnd }, w))
        .map(b => ({ id: b.id, eventTypeId: b.eventTypeId, slotKey: b.slotKey, start: b.start, blockedStart: b.blockedStart, blockedEnd: b.blockedEnd }));
    },
    async listBusy(_h, w) { return busy.filter(i => overlaps(i, w)); },
    async getBooking(id) { const b = bookings.get(id); return b ? structuredClone(b) : null; },
    async getBookingByTokenHash(h) { const b = [...bookings.values()].find(x => x.manageTokenHash === h); return b ? structuredClone(b) : null; },
    async listWebhookEndpointIds(hostId, event) { return [...endpoints.values()].filter(e => e.hostId === hostId && e.active && e.events.includes(event)).map(e => e.id); },
    async getWebhookEndpoint(id) { return endpoints.get(id) ?? null; },
    async book(input) {
      const r = insert(input);
      return r === 'ok' ? { ok: true, booking: structuredClone(bookings.get(input.booking.id) as Booking) } : { ok: false, reason: r };
    },
    async reschedule(oldId, input) {
      const old = bookings.get(oldId);
      if (!old) return { ok: false, reason: 'not_found' };
      if (old.status !== 'confirmed') return { ok: false, reason: 'not_confirmed' };
      const snapshot = structuredClone(old);
      const before = jobs.length;
      old.status = 'cancelled';
      old.cancelReason = 'rescheduled';
      const removed = jobs.filter(j => !j.done && j.kind === 'reminder.invitee' && (j.payload as { bookingId: string }).bookingId === oldId);
      for (const j of removed) jobs.splice(jobs.indexOf(j), 1);
      const r = insert(input);
      if (r !== 'ok') {
        bookings.set(oldId, snapshot);
        jobs.splice(before - removed.length);
        jobs.push(...removed);
        return { ok: false, reason: r };
      }
      return { ok: true, booking: structuredClone(bookings.get(input.booking.id) as Booking) };
    },
    async cancel(id, reason, list) {
      const b = bookings.get(id);
      if (!b) return { ok: false, reason: 'not_found' };
      if (b.status !== 'confirmed') return { ok: false, reason: 'not_confirmed' };
      b.status = 'cancelled';
      b.cancelReason = reason;
      b.sequence += 1;
      for (const j of jobs.filter(j => !j.done && j.kind === 'reminder.invitee' && (j.payload as { bookingId: string }).bookingId === id)) jobs.splice(jobs.indexOf(j), 1);
      addJobs(list);
      return { ok: true, booking: structuredClone(b) };
    },
    async setBookingExternal(id, f) {
      const b = bookings.get(id);
      if (!b) return;
      if (f.externalEventId !== undefined) b.externalEventId = f.externalEventId;
      if (f.meetingUrl !== undefined) b.meetingUrl = f.meetingUrl;
    },
    async claimJobs(limit, now, lease) {
      const due = jobs.filter(j => !j.done && j.runAt <= now && j.leaseUntil < now.getTime()).sort((a, b) => a.runAt.getTime() - b.runAt.getTime()).slice(0, limit);
      for (const j of due) { j.leaseUntil = now.getTime() + lease * 1000; j.attempts++; }
      return due.map(j => ({ id: j.id, kind: j.kind, payload: j.payload, runAt: j.runAt, attempts: j.attempts, dedupeKey: j.dedupeKey }));
    },
    async completeJob(id) { const j = jobs.find(x => x.id === id); if (j) { j.done = true; j.leaseUntil = 0; } },
    async failJob(id, error, at) {
      const j = jobs.find(x => x.id === id);
      if (!j) return;
      j.lastError = error;
      j.leaseUntil = 0;
      if (at) j.runAt = at; else j.done = true;
    },
  };
  return { store, bookings, jobs, busy, endpoints };
}

export function recordingMailer(failTimes = 0) {
  const sent: MailMessage[] = [];
  let failures = failTimes;
  const mailer: Mailer = {
    async send(m) {
      if (failures-- > 0) throw new Error('provider down');
      sent.push(m);
    },
  };
  return { mailer, sent };
}

export function recordingCalendar() {
  const calls: string[] = [];
  const calendar: CalendarWriter = {
    async upsert(b) { calls.push(`upsert:${b.chainId}:${b.start.toISOString()}`); return { externalEventId: `ev-${b.chainId}`, meetingUrl: 'https://meet.example/abc' }; },
    async remove(b) { calls.push(`remove:${b.chainId}`); },
  };
  return { calendar, calls };
}
```

## Suites

```ts
// test/time.test.ts: DST and calendar arithmetic, against the IANA database the runtime ships.
import { describe, expect, it } from 'vitest';
import {
  addDays,
  isLocalDate,
  isValidTimeZone,
  isWallTime,
  localDateOf,
  offsetMinutes,
  weekdayOf,
  zonedToInstant,
} from '@/lib/scheduling/time';

describe('time', () => {
  it('converts a plain wall-clock time', () => {
    expect(zonedToInstant('2026-07-15', '09:00', 'Europe/Warsaw').toISOString()).toBe('2026-07-15T07:00:00.000Z');
    expect(zonedToInstant('2026-01-15', '09:00', 'America/New_York').toISOString()).toBe('2026-01-15T14:00:00.000Z');
  });

  it('moves a time inside the spring-forward gap forward', () => {
    // Europe/Warsaw jumps from 02:00 to 03:00 on 2026-03-29.
    expect(zonedToInstant('2026-03-29', '02:30', 'Europe/Warsaw').toISOString()).toBe('2026-03-29T01:30:00.000Z');
  });

  it('picks the earlier instant for a time inside the fall-back overlap', () => {
    // Europe/Warsaw goes back from 03:00 to 02:00 on 2026-10-25; 02:30 happens twice.
    expect(zonedToInstant('2026-10-25', '02:30', 'Europe/Warsaw').toISOString()).toBe('2026-10-25T00:30:00.000Z');
  });

  it('reads 24:00 as midnight of the next day', () => {
    expect(zonedToInstant('2026-07-15', '24:00', 'Europe/Warsaw').toISOString()).toBe('2026-07-15T22:00:00.000Z');
  });

  it('handles zones with non-hour offsets', () => {
    expect(offsetMinutes(new Date('2026-07-15T00:00:00Z'), 'Asia/Kathmandu')).toBe(345);
    expect(zonedToInstant('2026-07-15', '09:00', 'Asia/Kolkata').toISOString()).toBe('2026-07-15T03:30:00.000Z');
  });

  it('does calendar arithmetic on dates, across month, year and DST boundaries', () => {
    expect(addDays('2026-03-28', 1)).toBe('2026-03-29');
    expect(addDays('2026-03-29', 1)).toBe('2026-03-30');
    expect(addDays('2026-12-31', 1)).toBe('2027-01-01');
    expect(addDays('2028-03-01', -1)).toBe('2028-02-29');
    expect(weekdayOf('2026-10-15')).toBe(4);
  });

  it('reads the local date of an instant in the zone, not in UTC', () => {
    expect(localDateOf(new Date('2026-10-15T23:30:00Z'), 'Europe/Warsaw')).toBe('2026-10-16');
    expect(localDateOf(new Date('2026-10-15T02:00:00Z'), 'America/Los_Angeles')).toBe('2026-10-14');
  });

  it('validates input', () => {
    expect(isValidTimeZone('Europe/Warsaw')).toBe(true);
    expect(isValidTimeZone('Mars/Olympus')).toBe(false);
    expect(isLocalDate('2026-02-30')).toBe(false);
    expect(isWallTime('24:00')).toBe(true);
    expect(isWallTime('24:30')).toBe(false);
    expect(isWallTime('9:00')).toBe(false);
  });
});
```

```ts
// test/slot-engine.test.ts: what can be booked, with an injected clock.
import { describe, expect, it } from 'vitest';
import { computeSlots, type SlotQuery } from '@/lib/scheduling/slot-engine';
import { slotKeyOf, type BookedInterval, type Weekday } from '@/lib/scheduling/types';

const TZ = 'Europe/Warsaw';
const weekdays = [1, 2, 3, 4, 5] as Weekday[];

function query(overrides: Partial<SlotQuery> = {}, et: Partial<SlotQuery['eventType']> = {}): SlotQuery {
  return {
    eventType: {
      id: 'et1',
      capacity: 1,
      durationMinutes: 30,
      slotIntervalMinutes: 30,
      bufferBeforeMinutes: 0,
      bufferAfterMinutes: 0,
      minNoticeMinutes: 0,
      horizonDays: 60,
      dailyLimit: null,
      ...et,
    },
    schedule: { timeZone: TZ, rules: weekdays.map(weekday => ({ weekday, start: '09:00', end: '12:00' })), overrides: [] },
    bookings: [],
    busy: [],
    // Monday 2026-10-12, 06:00 in Warsaw.
    now: new Date('2026-10-12T04:00:00Z'),
    from: new Date('2026-10-12T00:00:00Z'),
    to: new Date('2026-10-13T00:00:00Z'),
    ...overrides,
  };
}

function booking(start: string, minutes: number, et = 'et1', before = 0, after = 0): BookedInterval {
  const s = new Date(start);
  return {
    id: `b-${start}-${et}`,
    eventTypeId: et,
    slotKey: slotKeyOf(et, s),
    start: s,
    blockedStart: new Date(s.getTime() - before * 60_000),
    blockedEnd: new Date(s.getTime() + (minutes + after) * 60_000),
  };
}

const starts = (q: SlotQuery) => computeSlots(q).map(s => s.start.toISOString());

describe('computeSlots', () => {
  it('lays slots on the weekly rule in the schedule zone', () => {
    const s = starts(query());
    expect(s).toHaveLength(6);
    expect(s[0]).toBe('2026-10-12T07:00:00.000Z');
    expect(s[5]).toBe('2026-10-12T09:30:00.000Z');
  });

  it('honours minimum notice', () => {
    const s = starts(query({ now: new Date('2026-10-12T07:10:00Z') }, { minNoticeMinutes: 60 }));
    expect(s[0]).toBe('2026-10-12T08:30:00.000Z');
  });

  it('stops at the rolling horizon, counted in local days', () => {
    const q = query({ from: new Date('2026-10-12T00:00:00Z'), to: new Date('2026-12-31T00:00:00Z') }, { horizonDays: 2 });
    const days = new Set(computeSlots(q).map(s => s.start.toISOString().slice(0, 10)));
    expect([...days]).toEqual(['2026-10-12', '2026-10-13']);
  });

  it('lets a date override replace the weekly rule, and an empty one close the day', () => {
    const open = query({ schedule: { ...query().schedule, overrides: [{ date: '2026-10-12', ranges: [{ start: '15:00', end: '16:00' }] }] } });
    expect(starts(open)).toEqual(['2026-10-12T13:00:00.000Z', '2026-10-12T13:30:00.000Z']);
    const closed = query({ schedule: { ...query().schedule, overrides: [{ date: '2026-10-12', ranges: [] }] } });
    expect(starts(closed)).toEqual([]);
  });

  it('blocks a slot overlapped by a booking of any event type of the host', () => {
    const s = starts(query({ bookings: [booking('2026-10-12T07:30:00Z', 60, 'other')] }));
    expect(s).toEqual(['2026-10-12T07:00:00.000Z', '2026-10-12T08:30:00.000Z', '2026-10-12T09:00:00.000Z', '2026-10-12T09:30:00.000Z']);
  });

  it('adds buffers on both sides, and two buffers add up', () => {
    // Existing booking 08:15 to 08:45 Warsaw with 15 min after (blocked to 09:00); the 09:00 candidate's
    // own 15 min before reaches back to 08:45, so the two buffers together need a 30-minute gap.
    const q = query({ bookings: [booking('2026-10-12T06:15:00Z', 30, 'et1', 0, 15)] }, { bufferBeforeMinutes: 15 });
    const s = starts(q);
    expect(s).not.toContain('2026-10-12T07:00:00.000Z');
    expect(s[0]).toBe('2026-10-12T07:30:00.000Z');
  });

  it('blocks slots during connected-calendar busy time', () => {
    const s = starts(query({ busy: [{ start: new Date('2026-10-12T08:00:00Z'), end: new Date('2026-10-12T08:15:00Z') }] }));
    expect(s).not.toContain('2026-10-12T08:00:00.000Z');
    expect(s).toContain('2026-10-12T08:30:00.000Z');
  });

  it('keeps a group slot open until its seats are gone, without blocking itself', () => {
    const b = booking('2026-10-12T07:00:00Z', 30);
    const q = query({ bookings: [b, b] }, { capacity: 3 });
    const slot = computeSlots(q).find(s => s.key === b.slotKey);
    expect(slot?.seatsLeft).toBe(1);
    expect(computeSlots(query({ bookings: [b, b, b] }, { capacity: 3 })).find(s => s.key === b.slotKey)).toBeUndefined();
  });

  it('applies the daily limit per local day, but lets a started group slot fill', () => {
    const b = booking('2026-10-12T07:00:00Z', 30);
    const s = computeSlots(query({ bookings: [b] }, { capacity: 2, dailyLimit: 1 }));
    expect(s.map(x => x.key)).toEqual([b.slotKey]);
  });

  it('keeps generating the right dates across the spring-forward night', () => {
    // Late-evening clock and a 23-hour day: no local date may be skipped.
    const q = query(
      {
        schedule: { timeZone: TZ, rules: [0, 1, 2, 3, 4, 5, 6].map(w => ({ weekday: w as Weekday, start: '10:00', end: '10:30' })), overrides: [] },
        now: new Date('2026-03-27T22:45:00Z'),
        from: new Date('2026-03-27T22:45:00Z'),
        to: new Date('2026-03-31T00:00:00Z'),
      },
      {},
    );
    expect(starts(q)).toEqual(['2026-03-28T09:00:00.000Z', '2026-03-29T08:00:00.000Z', '2026-03-30T08:00:00.000Z']);
  });

  it('produces one slot per start on the fall-back day, with stable keys', () => {
    const q = query({
      schedule: { timeZone: TZ, rules: [{ weekday: 0, start: '01:00', end: '04:00' }], overrides: [] },
      now: new Date('2026-10-24T00:00:00Z'),
      from: new Date('2026-10-24T00:00:00Z'),
      to: new Date('2026-10-26T00:00:00Z'),
    }, { durationMinutes: 60, slotIntervalMinutes: 60 });
    // 01:00 CEST to 04:00 CET is four real hours: four one-hour slots.
    expect(starts(q)).toEqual([
      '2026-10-24T23:00:00.000Z',
      '2026-10-25T00:00:00.000Z',
      '2026-10-25T01:00:00.000Z',
      '2026-10-25T02:00:00.000Z',
    ]);
  });

  it('answers the booking route the same way as the page, for one start', () => {
    const start = new Date('2026-10-12T08:00:00Z');
    const one = computeSlots(query({ from: start, to: new Date(start.getTime() + 60_000) }));
    expect(one.map(s => s.start.toISOString())).toEqual([start.toISOString()]);
    const off = new Date('2026-10-12T08:10:00Z');
    expect(computeSlots(query({ from: off, to: new Date(off.getTime() + 60_000) }))).toEqual([]);
  });

  it('caps the candidate count', () => {
    const q = query({ to: new Date('2027-10-12T00:00:00Z'), schedule: { timeZone: TZ, rules: [0, 1, 2, 3, 4, 5, 6].map(w => ({ weekday: w as Weekday, start: '00:00', end: '24:00' })), overrides: [] } }, { durationMinutes: 1, slotIntervalMinutes: 1, horizonDays: 365 });
    expect(computeSlots(q).length).toBeLessThanOrEqual(20_000);
  });
});
```

The booking, edges and store contract suites are in [testing-flows.md](testing-flows.md).

## Running against real stores

```bash
# Firestore emulator (needs Java). firebase.json: {"emulators":{"firestore":{"port":8098}}}
firebase emulators:exec --only firestore --project demo-scheduling "npx vitest run test/store-contract.test.ts"

# Supabase, local stack with the migration applied
supabase start
SUPABASE_URL=http://127.0.0.1:54321 SUPABASE_SERVICE_ROLE_KEY=<service_role key from supabase status> \
  npx vitest run test/store-contract.test.ts
```

Then check by hand that the anon key is refused: `curl <SUPABASE_URL>/rest/v1/bookings -H "apikey: <anon>"`
must answer `42501`.
