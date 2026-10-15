# Data model

The canonical vocabulary, the records the module keeps, and the rename table a host fills in. Every other
reference uses these names.

## Vocabulary

| Term | Meaning |
|---|---|
| host | The person whose time is booked. One signed-in user of the host app; `hostId` is that user's id. |
| schedule | Weekly availability rules plus date overrides, in one IANA time zone. A host may keep several. |
| event type | One bookable offer: "Intro call, 30 min". Points at one schedule. |
| slot | A start time the engine computed. Never stored: it exists only as the engine's output. |
| booking | A taken slot, or one seat of a group slot. |
| invitee | The person who books. Not a user of the app; their only credential is the manage token. |
| chain | A booking and every booking that replaced it by rescheduling. One calendar event, one ICS UID. |
| busy block | Time a connected calendar reports as busy, mirrored into the database. |
| job | One side effect waiting to run: an e-mail, a calendar write, a webhook, a reminder. |

## Types

```ts
// lib/scheduling/types.ts: the canonical vocabulary. The host renames it once, in adaptation.md.

/** 0 is Sunday, as Date.prototype.getUTCDay. */
export type Weekday = 0 | 1 | 2 | 3 | 4 | 5 | 6;

/** Wall-clock range in the schedule's zone: start 'HH:MM', end 'HH:MM' or '24:00', start before end. */
export interface TimeRange {
  start: string;
  end: string;
}

export interface AvailabilityRule extends TimeRange {
  weekday: Weekday;
}

/** Replaces the weekly rules for one local date. No ranges means the day is off. */
export interface DateOverride {
  date: string;
  ranges: TimeRange[];
}

export interface Schedule {
  id: string;
  hostId: string;
  name: string;
  timeZone: string;
  rules: AvailabilityRule[];
  overrides: DateOverride[];
}

export type EventKind = 'one_on_one' | 'group';

export type LocationKind = 'google_meet' | 'teams' | 'phone' | 'in_person' | 'custom_link';

export interface EventType {
  id: string;
  hostId: string;
  scheduleId: string;
  slug: string;
  title: string;
  description: string;
  kind: EventKind;
  /** 1 for one_on_one; seats per slot for group. */
  capacity: number;
  durationMinutes: number;
  /** Step between candidate starts. Defaults to durationMinutes in the editor. */
  slotIntervalMinutes: number;
  bufferBeforeMinutes: number;
  bufferAfterMinutes: number;
  minNoticeMinutes: number;
  /** Rolling window in local days, today included. */
  horizonDays: number;
  /** Distinct slots of this event type per local day, or null for no limit. */
  dailyLimit: number | null;
  locationKind: LocationKind;
  locationValue: string | null;
  questions: InviteeQuestion[];
  active: boolean;
}

export interface InviteeQuestion {
  id: string;
  label: string;
  type: 'text' | 'textarea' | 'phone';
  required: boolean;
}

/** Half-open [start, end). */
export interface Interval {
  start: Date;
  end: Date;
}

export type BookingStatus = 'confirmed' | 'cancelled';

export interface Booking {
  id: string;
  hostId: string;
  eventTypeId: string;
  /** `${eventTypeId}:${start ISO}`: the seats of one group slot share it. */
  slotKey: string;
  start: Date;
  end: Date;
  /** [start - bufferBefore, end + bufferAfter), frozen at booking time. */
  blockedStart: Date;
  blockedEnd: Date;
  status: BookingStatus;
  inviteeName: string;
  inviteeEmail: string;
  inviteeTimeZone: string;
  answers: Record<string, string>;
  /** First booking of a reschedule chain: the ICS UID and the calendar event follow it, not the id. */
  chainId: string;
  /** SHA-256 of the manage token; the token itself is only ever in the invitee's e-mail. */
  manageTokenHash: string;
  /** Bumped on every change the invitee's calendar must see; the ICS SEQUENCE. */
  sequence: number;
  rescheduledFromId: string | null;
  cancelReason: string | null;
  externalEventId: string | null;
  meetingUrl: string | null;
  createdAt: Date;
  updatedAt: Date;
}

/** What the slot engine needs from a booking: any event type of the same host. */
export type BookedInterval = Pick<Booking, 'id' | 'eventTypeId' | 'slotKey' | 'start' | 'blockedStart' | 'blockedEnd'>;

export interface Slot {
  start: Date;
  end: Date;
  key: string;
  seatsLeft: number;
}

export function slotKeyOf(eventTypeId: string, start: Date): string {
  return `${eventTypeId}:${start.toISOString()}`;
}
```

## What each field is for

| Field | Why it exists |
|---|---|
| `EventType.slotIntervalMinutes` | Candidate starts step by this, not by the duration: a 45-minute call on a 15-minute grid offers 9:00, 9:15, 9:30. |
| `EventType.minNoticeMinutes` | Nothing starts sooner than this after now. Defaults to 240 in the migration. |
| `EventType.horizonDays` | Counted in the schedule's local days, today included, so "60 days" ends at a local midnight. |
| `EventType.dailyLimit` | Distinct booked slots of this event type per local day. A group slot that already has a seat counts once. |
| `Booking.slotKey` | `${eventTypeId}:${start ISO}`. The seats of one group slot share it; the overlap constraint lets equal keys overlap. |
| `Booking.blockedStart`, `blockedEnd` | The booking widened by its own buffers, frozen when it is made. A later change to the buffers does not move existing bookings. |
| `Booking.chainId` | The first booking of a reschedule chain. The host's calendar event id and the invitee's ICS UID derive from it. |
| `Booking.sequence` | Rises with every change along the chain. It is the ICS `SEQUENCE`, which tells the invitee's calendar to replace its copy. |
| `Booking.manageTokenHash` | SHA-256 of the manage token. The token itself is derived from the booking id with a server secret, so a job can rebuild the link at send time. |
| `Booking.externalEventId`, `meetingUrl` | Written back by the calendar job. A reschedule copies them to the new booking, so the event and the link move with it. |

## Records

| Record | Postgres table | Firestore collection | Notes |
|---|---|---|---|
| Host profile | `scheduling_hosts` | `schedulingHosts` | `handle` is the public URL segment, `/ana/intro`. |
| Schedule | `schedules` | `schedules` | Rules and overrides as JSON, validated by `scheduleInput`. |
| Event type | `event_types` | `eventTypes` | Unique `(host_id, slug)`. |
| Booking | `bookings` | `bookings` | Never deleted: a cancel or reschedule changes `status`. |
| Calendar connection | `calendar_connections` | not shipped | Tokens sealed with AES-256-GCM. |
| Watch | `calendar_watches` | not shipped | One Google channel or Microsoft subscription per checked calendar. |
| Busy block | `busy_blocks` | `busyBlocks` | Replaced per connection in one transaction on each sync. |
| Webhook endpoint | `webhook_endpoints` | `webhookEndpoints` | Secret sealed like the tokens. |
| Job | `jobs` | `jobs` | Unique `dedupe_key`; leased, retried, kept as a record when done. |

## Rename table

Fill the right column from the host before writing code, and use it everywhere. Identifiers in the templates
(`computeSlots`, `SchedulingStore`, `jobsFor`) keep their names; the words below are what users see and
what the host's own code calls things.

| Canonical | Host term (example) | Where it shows |
|---|---|---|
| host | coach, advisor, consultant, doctor | Dashboard, e-mails |
| event type | service, meeting type, session | Dashboard, booking page title |
| schedule | working hours, availability | Dashboard |
| booking | appointment, meeting, session | E-mails, manage page, webhooks |
| invitee | client, guest, patient, candidate | E-mails, host notifications |
| handle | username, slug, team name | Public URL |

Table and column names may follow the host's convention too. When they change, change the SQL functions in
[supabase.md](supabase.md) with them: they name columns in their bodies.
