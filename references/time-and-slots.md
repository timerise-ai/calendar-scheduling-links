# Time and the slot engine

`computeSlots` is the one function that decides what can be booked. The booking page lists its output, and
the booking route calls it again for the single requested start, so the page and the write cannot disagree.
It is pure: no I/O, no clock, no database. Everything it needs comes in the query.

## The rules it applies, in order

1. **Local dates, not 24-hour steps.** The engine walks calendar dates in the schedule's zone with `addDays`,
   from the day before the window to the earlier of the window's end and the horizon. Adding 24 hours to an
   instant skips a local date on a 23-hour spring-forward day whenever the walk starts late in the evening.
2. **Override or rules.** A date override replaces that date's weekly rules entirely. An override with no
   ranges closes the day.
3. **Wall clock to instants.** Each range's start and end become instants with `zonedToInstant`. A time in
   the spring-forward gap moves forward by the gap; a time in the fall-back overlap takes the earlier
   instant. Candidate starts then step by `slotIntervalMinutes` in real time, so a range that crosses a DST
   change still yields evenly spaced slots.
4. **Notice and window.** A start earlier than `now + minNoticeMinutes`, or outside `[from, to)`, is skipped.
5. **Seats.** Bookings of this event type with the same `slotKey` are seats. A full slot is skipped; a slot
   with seats left shows `seatsLeft`.
6. **Daily limit.** A local day with `dailyLimit` distinct slots booked offers no new slot, but a group slot
   that already has a seat stays open until it is full.
7. **Conflicts.** The candidate is widened by this event type's buffers and must not overlap any other
   booking's blocked range (any event type of the same host) or any busy block. Two buffers between two
   bookings add up: a 15-minute after-buffer and a 15-minute before-buffer need a 30-minute gap.
8. **Cap.** At most `MAX_CANDIDATES` starts are examined per call, so a 1-minute grid over a year cannot hang
   a request.

## Time helpers

Built on `Intl.DateTimeFormat` alone, so there is no date library to keep in step with the IANA database the
runtime already ships. Every function takes the zone explicitly; the server's `TZ` never matters.

```ts
// lib/scheduling/time.ts: wall-clock and instant conversion on Intl alone, with explicit DST rules.
// A local date is a 'YYYY-MM-DD' string and a wall-clock time is 'HH:MM' ('24:00' allowed as an end).
// Every function takes the IANA zone explicitly, so nothing depends on the server's TZ.

const formatters = new Map<string, Intl.DateTimeFormat>();

function formatter(timeZone: string): Intl.DateTimeFormat {
  let f = formatters.get(timeZone);
  if (!f) {
    f = new Intl.DateTimeFormat('en-US', {
      timeZone,
      hourCycle: 'h23',
      year: 'numeric',
      month: '2-digit',
      day: '2-digit',
      hour: '2-digit',
      minute: '2-digit',
      second: '2-digit',
    });
    formatters.set(timeZone, f);
  }
  return f;
}

export function isValidTimeZone(timeZone: string): boolean {
  try {
    formatter(timeZone);
    return true;
  } catch {
    return false;
  }
}

interface WallParts {
  year: number;
  month: number;
  day: number;
  hour: number;
  minute: number;
  second: number;
}

function wallParts(instant: Date, timeZone: string): WallParts {
  const out: Record<string, number> = {};
  for (const p of formatter(timeZone).formatToParts(instant)) {
    if (p.type !== 'literal') out[p.type] = Number(p.value);
  }
  return {
    year: out.year ?? 0,
    month: out.month ?? 0,
    day: out.day ?? 0,
    hour: out.hour ?? 0,
    minute: out.minute ?? 0,
    second: out.second ?? 0,
  };
}

/** Offset of the zone from UTC at an instant, in minutes (Europe/Warsaw in July: 120). */
export function offsetMinutes(instant: Date, timeZone: string): number {
  const w = wallParts(instant, timeZone);
  const asUtc = Date.UTC(w.year, w.month - 1, w.day, w.hour, w.minute, w.second);
  const truncated = Math.floor(instant.getTime() / 1000) * 1000;
  return Math.round((asUtc - truncated) / 60_000);
}

const DATE_RE = /^(\d{4})-(\d{2})-(\d{2})$/;
const TIME_RE = /^([01]\d|2[0-4]):([0-5]\d)$/;

export function isLocalDate(value: string): boolean {
  const m = DATE_RE.exec(value);
  if (!m) return false;
  const d = new Date(Date.UTC(Number(m[1]), Number(m[2]) - 1, Number(m[3])));
  return d.toISOString().slice(0, 10) === value;
}

export function isWallTime(value: string): boolean {
  const m = TIME_RE.exec(value);
  return m !== null && (m[1] !== '24' || m[2] === '00');
}

/** Minutes since midnight: '09:30' is 570, '24:00' is 1440. */
export function wallMinutes(time: string): number {
  if (!isWallTime(time)) throw new RangeError(`Invalid wall-clock time: ${time}`);
  const [h, m] = time.split(':');
  return Number(h) * 60 + Number(m);
}

function dateParts(date: string): [number, number, number] {
  const m = DATE_RE.exec(date);
  if (!m || !isLocalDate(date)) throw new RangeError(`Invalid local date: ${date}`);
  return [Number(m[1]), Number(m[2]), Number(m[3])];
}

/** Calendar arithmetic on local dates. Never add 24 hours to an instant to reach the next day. */
export function addDays(date: string, days: number): string {
  const [y, m, d] = dateParts(date);
  return new Date(Date.UTC(y, m - 1, d + days)).toISOString().slice(0, 10);
}

/** 0 is Sunday, as Date.prototype.getUTCDay. */
export function weekdayOf(date: string): number {
  const [y, m, d] = dateParts(date);
  return new Date(Date.UTC(y, m - 1, d)).getUTCDay();
}

/** The local date an instant falls on in a zone. */
export function localDateOf(instant: Date, timeZone: string): string {
  const w = wallParts(instant, timeZone);
  return `${String(w.year).padStart(4, '0')}-${String(w.month).padStart(2, '0')}-${String(w.day).padStart(2, '0')}`;
}

/**
 * The instant a wall-clock time on a local date names in a zone.
 * DST gap (02:30 on a spring-forward night does not exist): moves forward by the gap, to 03:30.
 * DST overlap (01:30 happens twice on a fall-back night): the earlier of the two instants.
 * These are the 'compatible' rules of Temporal and of RFC 5545 for non-existent local times.
 */
export function zonedToInstant(date: string, time: string, timeZone: string): Date {
  const [y, m, d] = dateParts(date);
  const wall = Date.UTC(y, m - 1, d, 0, wallMinutes(time));
  const before = offsetMinutes(new Date(wall - 86_400_000), timeZone);
  const after = offsetMinutes(new Date(wall + 86_400_000), timeZone);
  const candidates = [wall - before * 60_000, wall - after * 60_000].filter(
    t => wall - offsetMinutes(new Date(t), timeZone) * 60_000 === t,
  );
  if (candidates.length > 0) return new Date(Math.min(...candidates));
  return new Date(wall - before * 60_000);
}
```

## The engine

```ts
// lib/scheduling/slot-engine.ts: the one function that decides what can be booked.
// Pure: no I/O, no clock. The booking page lists its output and the booking route recomputes it for the
// requested start, so the page and the write can never disagree about a slot.
import { addDays, localDateOf, wallMinutes, weekdayOf, zonedToInstant } from '@/lib/scheduling/time';
import {
  slotKeyOf,
  type BookedInterval,
  type EventType,
  type Interval,
  type Schedule,
  type Slot,
  type TimeRange,
} from '@/lib/scheduling/types';

const MINUTE = 60_000;
/** Upper bound on candidate starts per call, so a 1-minute interval over a year cannot hang a request. */
export const MAX_CANDIDATES = 20_000;

export interface SlotQuery {
  eventType: Pick<
    EventType,
    | 'id'
    | 'capacity'
    | 'durationMinutes'
    | 'slotIntervalMinutes'
    | 'bufferBeforeMinutes'
    | 'bufferAfterMinutes'
    | 'minNoticeMinutes'
    | 'horizonDays'
    | 'dailyLimit'
  >;
  schedule: Pick<Schedule, 'timeZone' | 'rules' | 'overrides'>;
  /** Confirmed bookings of the same host, every event type, overlapping the window plus buffers. */
  bookings: BookedInterval[];
  /** Busy time from connected calendars, with the app's own events already removed. */
  busy: Interval[];
  now: Date;
  /** Window to list, half-open. The booking route passes [start, start + 1 minute). */
  from: Date;
  to: Date;
}

export function overlaps(aStart: Date, aEnd: Date, bStart: Date, bEnd: Date): boolean {
  return aStart.getTime() < bEnd.getTime() && bStart.getTime() < aEnd.getTime();
}

/** Ranges for one local date: the override when there is one, otherwise the weekday's rules. */
export function rangesFor(schedule: SlotQuery['schedule'], date: string): TimeRange[] {
  const override = schedule.overrides.find(o => o.date === date);
  if (override) return override.ranges;
  const weekday = weekdayOf(date);
  return schedule.rules.filter(r => r.weekday === weekday);
}

export function computeSlots(q: SlotQuery): Slot[] {
  const { eventType: et, schedule, now } = q;
  const tz = schedule.timeZone;
  const duration = et.durationMinutes * MINUTE;
  const step = et.slotIntervalMinutes * MINUTE;
  if (duration <= 0 || step <= 0 || et.capacity < 1) return [];

  const earliest = Math.max(q.from.getTime(), now.getTime() + et.minNoticeMinutes * MINUTE);
  const today = localDateOf(now, tz);
  const lastDate = addDays(today, et.horizonDays - 1);

  // Seats taken per slot of this event type, and distinct booked slots per local day for dailyLimit.
  const seatsTaken = new Map<string, number>();
  const slotsPerDay = new Map<string, Set<string>>();
  for (const b of q.bookings) {
    if (b.eventTypeId !== et.id) continue;
    seatsTaken.set(b.slotKey, (seatsTaken.get(b.slotKey) ?? 0) + 1);
    const day = localDateOf(b.start, tz);
    const set = slotsPerDay.get(day) ?? new Set<string>();
    set.add(b.slotKey);
    slotsPerDay.set(day, set);
  }

  const out = new Map<string, Slot>();
  let candidates = 0;
  // Start one day early: a range on the previous local date can still end inside the window.
  let date = addDays(localDateOf(new Date(earliest), tz), -1);
  const endDate = localDateOf(q.to, tz) < lastDate ? localDateOf(q.to, tz) : lastDate;

  for (; date <= endDate; date = addDays(date, 1)) {
    const dayFull = et.dailyLimit !== null && (slotsPerDay.get(date)?.size ?? 0) >= et.dailyLimit;
    for (const range of rangesFor(schedule, date)) {
      if (wallMinutes(range.end) <= wallMinutes(range.start)) continue;
      const rangeStart = zonedToInstant(date, range.start, tz).getTime();
      const rangeEnd = zonedToInstant(date, range.end, tz).getTime();
      for (let t = rangeStart; t + duration <= rangeEnd; t += step) {
        if (++candidates > MAX_CANDIDATES) return sorted(out);
        if (t < earliest || t >= q.to.getTime()) continue;
        const start = new Date(t);
        const key = slotKeyOf(et.id, start);
        if (out.has(key)) continue;
        const taken = seatsTaken.get(key) ?? 0;
        // A day at its limit still shows slots that already have seats, so a group can fill up.
        if (dayFull && taken === 0) continue;
        if (taken >= et.capacity) continue;
        const end = new Date(t + duration);
        const blockedStart = new Date(t - et.bufferBeforeMinutes * MINUTE);
        const blockedEnd = new Date(t + duration + et.bufferAfterMinutes * MINUTE);
        // Another slot's booking (any event type of this host) blocks; seats of this same slot do not.
        const clash = q.bookings.some(
          b => b.slotKey !== key && overlaps(blockedStart, blockedEnd, b.blockedStart, b.blockedEnd),
        );
        if (clash) continue;
        if (q.busy.some(i => overlaps(blockedStart, blockedEnd, i.start, i.end))) continue;
        out.set(key, { start, end, key, seatsLeft: et.capacity - taken });
      }
    }
  }
  return sorted(out);
}

function sorted(map: Map<string, Slot>): Slot[] {
  return [...map.values()].sort((a, b) => a.start.getTime() - b.start.getTime());
}
```

## Questions an implementer asks

| Question | Answer |
|---|---|
| Which zone groups slots into days on the page? | The invitee's. The engine returns instants; the widget groups by the visitor's zone, so a 23:30 host slot can show on the next day for a visitor further east. |
| Why read bookings a day beyond the window? | Buffers and ranges that cross midnight reach outside it. `slotsIn` in [booking-flow.md](booking-flow.md) widens the read. |
| Can a range end at midnight? | Yes, as `24:00`. A range may not wrap past midnight; split it into two days. |
| What if a host edits buffers after bookings exist? | Existing bookings keep the blocked range frozen when they were made. New candidates use the new buffers. |
| Where do external calendar events come in? | As `busy`, already stripped of the app's own events. See [calendar-sync.md](calendar-sync.md). |

The tests for both files are in [testing.md](testing.md): DST in both directions, a non-hour offset zone,
the late-evening walk across a spring-forward night, buffers adding up, seats, the daily limit, and the
page-and-write agreement.
