# Jobs and notifications

Every side effect of a booking change is a row in one `jobs` table, written in the same transaction as the
change: the invitee's e-mail with its ICS, the host's e-mail, the calendar write or delete, each webhook
delivery and each reminder. A runner drains the table. Nothing is sent from inside the transaction, and
nothing is lost if a function dies after the commit.

## Why one table

| Property | How the table gives it |
|---|---|
| A failure in one effect does not stop the others | Each job runs, fails and retries on its own. |
| A retry does not send twice | Jobs are claimed with a lease; a completed job is marked done. The dedupe key stops the same job from being queued twice. |
| A moved or cancelled booking sends no stale reminder | Reminders are deleted in the reschedule and cancel transactions, and the reminder handler re-checks status and start before sending. |
| E-mails arrive in seconds without waiting for the cron | The route calls `runDueJobs` inside `after()`. The cron every five minutes is the guarantee. |
| Every effect has a record | Done jobs stay with `done_at`; failed ones keep `last_error`, and after the last attempt `gave up: ...`. |

## Jobs

```ts
// lib/scheduling/jobs.ts: every side effect is a row in one jobs table, written in the booking's transaction.
// A job is idempotent on its own: re-running it after a crash must not send, create or charge twice.

export type JobKind =
  | 'email.invitee'
  | 'email.host'
  | 'calendar.upsert'
  | 'calendar.delete'
  | 'webhook.deliver'
  | 'reminder.invitee';

export type BookingChange = 'created' | 'rescheduled' | 'cancelled';

export interface JobPayloads {
  'email.invitee': { bookingId: string; change: BookingChange };
  'email.host': { bookingId: string; change: BookingChange };
  'calendar.upsert': { bookingId: string };
  'calendar.delete': { bookingId: string; hostId: string; externalEventId: string | null };
  'webhook.deliver': { endpointId: string; event: string; bookingId: string; occurredAt: string };
  /** startIso pins the reminder to the start it was scheduled for; a moved booking makes it stale. */
  'reminder.invitee': { bookingId: string; startIso: string; offsetMinutes: number };
}

export interface JobInput<K extends JobKind = JobKind> {
  kind: K;
  payload: JobPayloads[K];
  runAt: Date;
  /** Unique across the table. A second insert with the same key is ignored, never duplicated. */
  dedupeKey: string;
}

export interface JobRecord {
  id: string;
  kind: JobKind;
  payload: unknown;
  runAt: Date;
  attempts: number;
  dedupeKey: string;
}

/** Reminder offsets before the start, in minutes. The host may change them; keep them few. */
export const REMINDER_OFFSETS_MINUTES = [24 * 60, 60] as const;
export const MAX_JOB_ATTEMPTS = 8;
export const JOB_LEASE_SECONDS = 120;

/** 1, 2, 4 ... minutes, capped at 6 hours. null once the attempts are spent. */
export function retryAt(attempts: number, now: Date): Date | null {
  if (attempts >= MAX_JOB_ATTEMPTS) return null;
  const minutes = Math.min(2 ** (attempts - 1), 360);
  return new Date(now.getTime() + minutes * 60_000);
}

export function job<K extends JobKind>(kind: K, payload: JobPayloads[K], runAt: Date, dedupeKey: string): JobInput<K> {
  return { kind, payload, runAt, dedupeKey };
}

/** The jobs one booking change implies. Reminders in the past are skipped, never sent late. */
export function jobsFor(
  change: BookingChange,
  b: { id: string; hostId: string; start: Date; sequence: number; externalEventId: string | null },
  webhookEndpointIds: string[],
  now: Date,
): JobInput[] {
  const tag = `${b.id}:${change}:${b.sequence}`;
  const jobs: JobInput[] = [
    job('email.invitee', { bookingId: b.id, change }, now, `email.invitee:${tag}`),
    job('email.host', { bookingId: b.id, change }, now, `email.host:${tag}`),
  ];
  if (change === 'cancelled') {
    jobs.push(job('calendar.delete', { bookingId: b.id, hostId: b.hostId, externalEventId: b.externalEventId }, now, `calendar.delete:${b.id}`));
  } else {
    jobs.push(job('calendar.upsert', { bookingId: b.id }, now, `calendar.upsert:${tag}`));
    for (const offsetMinutes of REMINDER_OFFSETS_MINUTES) {
      const runAt = new Date(b.start.getTime() - offsetMinutes * 60_000);
      if (runAt.getTime() <= now.getTime()) continue;
      jobs.push(
        job('reminder.invitee', { bookingId: b.id, startIso: b.start.toISOString(), offsetMinutes }, runAt, `reminder:${b.id}:${b.start.toISOString()}:${offsetMinutes}`),
      );
    }
  }
  const event = `booking.${change}`;
  for (const endpointId of webhookEndpointIds) {
    jobs.push(
      job('webhook.deliver', { endpointId, event, bookingId: b.id, occurredAt: now.toISOString() }, now, `webhook:${endpointId}:${tag}`),
    );
  }
  return jobs;
}
```

Backoff: 1, 2, 4, 8, 16, 32, 64 minutes, then the job gives up after its eighth attempt. A reminder already in
the past when the booking is made is never queued.

## Runner

```ts
// lib/scheduling/runner.ts: drains the jobs table. Called by the cron route and, right after a booking
// change, from after() in the route that made it. Each job re-reads the booking: the row is the truth.
import type { CalendarWriter } from '@/lib/scheduling/calendar/writer';
import { renderHostEmail, renderInviteeEmail, renderReminderEmail } from '@/lib/scheduling/emails';
import { buildIcs, icsUid } from '@/lib/scheduling/ics';
import { JOB_LEASE_SECONDS, retryAt, type JobPayloads, type JobRecord } from '@/lib/scheduling/jobs';
import type { Mailer } from '@/lib/scheduling/mailer';
import type { SchedulingStore } from '@/lib/scheduling/store';
import { manageTokenFor } from '@/lib/scheduling/tokens';
import type { Booking } from '@/lib/scheduling/types';
import { deliverWebhook } from '@/lib/scheduling/webhooks';

export interface RunnerDeps {
  store: SchedulingStore;
  mailer: Mailer;
  calendar: CalendarWriter;
  now: () => Date;
  /** Absolute origin of the app, for manage links: https://book.example.com */
  appUrl: string;
}

export function manageUrl(appUrl: string, bookingId: string): string {
  return `${appUrl}/booking/${manageTokenFor(bookingId).token}`;
}

async function context(deps: RunnerDeps, bookingId: string) {
  const booking = await deps.store.getBooking(bookingId);
  if (!booking) return null;
  const [eventType, host] = await Promise.all([deps.store.getEventType(booking.eventTypeId), deps.store.getHost(booking.hostId)]);
  if (!eventType || !host) return null;
  return { booking, eventType, host };
}

/** The invitee's copy carries the private manage link; the host's copy never does. */
function ics(deps: RunnerDeps, b: Booking, title: string, host: { name: string; email: string }, cancel: boolean, forInvitee = true): string {
  return buildIcs({
    uid: icsUid(b.chainId, new URL(deps.appUrl).hostname),
    sequence: b.sequence,
    method: cancel ? 'CANCEL' : 'REQUEST',
    start: b.start,
    end: b.end,
    summary: title,
    description: cancel ? 'Cancelled' : forInvitee ? `Reschedule or cancel: ${manageUrl(deps.appUrl, b.id)}` : `Booked by ${b.inviteeName}`,
    location: b.meetingUrl,
    url: b.meetingUrl,
    organizer: host,
    attendee: { name: b.inviteeName, email: b.inviteeEmail },
    now: deps.now(),
  });
}

type Handlers = { [K in keyof JobPayloads]: (deps: RunnerDeps, p: JobPayloads[K], job: JobRecord) => Promise<void> };

const handlers: Handlers = {
  async 'email.invitee'(deps, p, job) {
    const c = await context(deps, p.bookingId);
    if (!c) return;
    const cancelled = p.change === 'cancelled';
    // A confirmation for a booking that was cancelled before this job ran would mislead; skip it.
    if (!cancelled && c.booking.status !== 'confirmed') return;
    const mail = renderInviteeEmail({ ...c, change: p.change, manageUrl: cancelled ? null : manageUrl(deps.appUrl, c.booking.id) });
    await deps.mailer.send({
      to: { email: c.booking.inviteeEmail, name: c.booking.inviteeName },
      replyTo: { email: c.host.email, name: c.host.name },
      ...mail,
      attachments: [{ filename: 'invite.ics', type: 'text/calendar', content: ics(deps, c.booking, c.eventType.title, c.host, cancelled) }],
      idempotencyKey: job.dedupeKey,
    });
  },
  async 'email.host'(deps, p, job) {
    const c = await context(deps, p.bookingId);
    if (!c) return;
    const cancelled = p.change === 'cancelled';
    // The same guard as the invitee's: a "new booking" notice for a booking already cancelled would mislead.
    if (!cancelled && c.booking.status !== 'confirmed') return;
    // A host whose calendar the module writes to gets the event there; an ICS as well would add it twice.
    const invite = (await deps.calendar.writes(c.booking.hostId)) ? [] : [{ filename: 'invite.ics', type: 'text/calendar', content: ics(deps, c.booking, c.eventType.title, c.host, cancelled, false) }];
    await deps.mailer.send({
      to: { email: c.host.email, name: c.host.name },
      replyTo: { email: c.booking.inviteeEmail, name: c.booking.inviteeName },
      ...renderHostEmail({ ...c, change: p.change, manageUrl: null }),
      attachments: invite,
      idempotencyKey: job.dedupeKey,
    });
  },
  async 'reminder.invitee'(deps, p, job) {
    const c = await context(deps, p.bookingId);
    // Moved or cancelled since the reminder was scheduled: the reminder belongs to a start that is gone.
    if (!c || c.booking.status !== 'confirmed' || c.booking.start.toISOString() !== p.startIso) return;
    await deps.mailer.send({
      to: { email: c.booking.inviteeEmail, name: c.booking.inviteeName },
      ...renderReminderEmail({ ...c, manageUrl: manageUrl(deps.appUrl, c.booking.id) }),
      idempotencyKey: job.dedupeKey,
    });
  },
  async 'calendar.upsert'(deps, p) {
    const c = await context(deps, p.bookingId);
    if (!c || c.booking.status !== 'confirmed') return;
    const written = await deps.calendar.upsert(c.booking, c.eventType, c.host);
    if (written) await deps.store.setBookingExternal(c.booking.id, written);
  },
  async 'calendar.delete'(deps, p) {
    const booking = await deps.store.getBooking(p.bookingId);
    if (booking) await deps.calendar.remove(booking);
  },
  async 'webhook.deliver'(deps, p, job) {
    const endpoint = await deps.store.getWebhookEndpoint(p.endpointId);
    if (!endpoint || !endpoint.active) return;
    const booking = await deps.store.getBooking(p.bookingId);
    if (!booking) return;
    const { manageTokenHash: _hash, answers, ...safe } = booking;
    await deliverWebhook(endpoint.url, endpoint.secret, { id: job.dedupeKey, event: p.event, occurredAt: p.occurredAt, data: { booking: { ...safe, answers } } }, deps.now());
  },
};

export interface RunReport {
  claimed: number;
  done: number;
  failed: number;
}

/** Runs until the queue is empty or the time budget is spent. Jobs run one by one; each fails alone. */
export async function runDueJobs(deps: RunnerDeps, budgetMs = 50_000, batch = 25): Promise<RunReport> {
  const report: RunReport = { claimed: 0, done: 0, failed: 0 };
  const startedAt = Date.now();
  while (Date.now() - startedAt < budgetMs) {
    const jobs = await deps.store.claimJobs(batch, deps.now(), JOB_LEASE_SECONDS);
    if (jobs.length === 0) break;
    report.claimed += jobs.length;
    for (const job of jobs) {
      try {
        const handler = handlers[job.kind] as (d: RunnerDeps, p: unknown, j: JobRecord) => Promise<void>;
        await handler(deps, job.payload, job);
        await deps.store.completeJob(job.id);
        report.done++;
      } catch (err) {
        report.failed++;
        await deps.store.failJob(job.id, err instanceof Error ? err.message : String(err), retryAt(job.attempts, deps.now()));
      }
    }
  }
  return report;
}
```

Every handler starts by re-reading the booking. The row is the truth; the payload only says which row.

| Handler | Re-checks before acting |
|---|---|
| `email.invitee` | A confirmation for a booking cancelled in the meantime is skipped. |
| `reminder.invitee` | Booking still confirmed and still starting at `startIso`. |
| `calendar.upsert` | Booking still confirmed. Writes back the event id and meeting link. |
| `calendar.delete` | Nothing: deleting an absent event is a success. |
| `webhook.deliver` | Endpoint still active. The manage token hash is never sent. |

## E-mail

The mailer is a seam. The SendGrid adapter uses `fetch`; replace it with Resend, SES or SMTP behind the same
`Mailer` interface.

```ts
// lib/scheduling/mailer.ts: the mail seam, with a SendGrid adapter over fetch. Swap in Resend, SES or SMTP.

export interface MailAttachment {
  filename: string;
  content: string;
  /** No parameters: SendGrid rejects an attachment type containing ';', so 'text/calendar' goes bare. */
  type: string;
}

export interface MailMessage {
  to: { email: string; name?: string };
  replyTo?: { email: string; name?: string };
  subject: string;
  html: string;
  text: string;
  attachments?: MailAttachment[];
  /** Same value on a retry of the same job, so the provider or a log can tie the two together. */
  idempotencyKey: string;
}

export interface Mailer {
  send(message: MailMessage): Promise<void>;
}

export function sendgridMailer(apiKey = process.env.SENDGRID_API_KEY, from = process.env.SCHEDULING_FROM_EMAIL): Mailer {
  return {
    async send(m) {
      if (!apiKey || !from) throw new Error('SENDGRID_API_KEY and SCHEDULING_FROM_EMAIL are required');
      const res = await fetch('https://api.sendgrid.com/v3/mail/send', {
        method: 'POST',
        headers: { authorization: `Bearer ${apiKey}`, 'content-type': 'application/json' },
        body: JSON.stringify({
          personalizations: [{ to: [m.to], custom_args: { idempotency_key: m.idempotencyKey } }],
          from: { email: from },
          ...(m.replyTo ? { reply_to: m.replyTo } : {}),
          subject: m.subject,
          content: [
            { type: 'text/plain', value: m.text },
            { type: 'text/html', value: m.html },
          ],
          attachments: m.attachments?.map(a => ({
            filename: a.filename,
            type: a.type,
            disposition: 'attachment',
            content: Buffer.from(a.content, 'utf8').toString('base64'),
          })),
        }),
        signal: AbortSignal.timeout(10_000),
      });
      if (!res.ok) throw new Error(`sendgrid ${res.status}: ${(await res.text()).slice(0, 300)}`);
    },
  };
}
```

Text, with every invitee value escaped before it meets HTML and every time formatted with an explicit zone:

```ts
// lib/scheduling/emails.ts: message text. Every value from an invitee is escaped before it meets HTML, and
// times are formatted with an explicit zone, so the server's TZ never leaks into an e-mail.
import type { BookingChange } from '@/lib/scheduling/jobs';
import type { HostProfile } from '@/lib/scheduling/store';
import type { Booking, EventType } from '@/lib/scheduling/types';

export function escapeHtml(value: string): string {
  return value.replace(/[&<>"']/g, c => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' })[c] ?? c);
}

export function formatWhen(start: Date, end: Date, timeZone: string, locale = 'en-US'): string {
  const day = new Intl.DateTimeFormat(locale, { timeZone, weekday: 'long', day: 'numeric', month: 'long', year: 'numeric' }).format(start);
  const time = new Intl.DateTimeFormat(locale, { timeZone, hour: 'numeric', minute: '2-digit' });
  return `${day}, ${time.format(start)} to ${time.format(end)} (${timeZone})`;
}

export interface EmailContext {
  booking: Booking;
  eventType: EventType;
  host: HostProfile;
  change: BookingChange;
  /** Absolute URL of the manage page, with the token; only in the invitee's copy. */
  manageUrl: string | null;
}

export interface RenderedEmail {
  subject: string;
  html: string;
  text: string;
}

const VERB: Record<BookingChange, string> = { created: 'confirmed', rescheduled: 'rescheduled', cancelled: 'cancelled' };

export function renderInviteeEmail(c: EmailContext): RenderedEmail {
  const when = formatWhen(c.booking.start, c.booking.end, c.booking.inviteeTimeZone);
  const subject = `${VERB[c.change][0]?.toUpperCase()}${VERB[c.change].slice(1)}: ${c.eventType.title} with ${c.host.name}`;
  const lines = [
    `Hi ${c.booking.inviteeName},`,
    `Your ${c.eventType.title} with ${c.host.name} is ${VERB[c.change]}.`,
    `When: ${when}`,
    ...(c.booking.meetingUrl && c.change !== 'cancelled' ? [`Join: ${c.booking.meetingUrl}`] : []),
    ...(c.manageUrl && c.change !== 'cancelled' ? [`Reschedule or cancel: ${c.manageUrl}`] : []),
  ];
  return {
    subject,
    text: lines.join('\n\n'),
    html: lines.map(l => `<p>${escapeHtml(l)}</p>`).join(''),
  };
}

export function renderHostEmail(c: EmailContext): RenderedEmail {
  const when = formatWhen(c.booking.start, c.booking.end, c.host.timeZone);
  const answers = c.eventType.questions
    .filter(q => c.booking.answers[q.id])
    .map(q => `${q.label}: ${c.booking.answers[q.id]}`);
  const lines = [
    `${c.booking.inviteeName} <${c.booking.inviteeEmail}>`,
    `${c.eventType.title}, ${VERB[c.change]}`,
    `When: ${when}`,
    ...answers,
    ...(c.booking.cancelReason ? [`Reason: ${c.booking.cancelReason}`] : []),
  ];
  return {
    subject: `${c.eventType.title} ${VERB[c.change]}: ${c.booking.inviteeName}`,
    text: lines.join('\n'),
    html: lines.map(l => `<p>${escapeHtml(l)}</p>`).join(''),
  };
}

export function renderReminderEmail(c: Omit<EmailContext, 'change'>): RenderedEmail {
  const when = formatWhen(c.booking.start, c.booking.end, c.booking.inviteeTimeZone);
  const lines = [
    `Hi ${c.booking.inviteeName},`,
    `A reminder: ${c.eventType.title} with ${c.host.name}.`,
    `When: ${when}`,
    ...(c.booking.meetingUrl ? [`Join: ${c.booking.meetingUrl}`] : []),
    ...(c.manageUrl ? [`Reschedule or cancel: ${c.manageUrl}`] : []),
  ];
  return { subject: `Reminder: ${c.eventType.title} with ${c.host.name}`, text: lines.join('\n\n'), html: lines.map(l => `<p>${escapeHtml(l)}</p>`).join('') };
}
```

The host's language, branding and wording go in these three functions. Keep the escaping and the explicit
zones when you rewrite them.

## ICS

The invitee's calendar must replace its copy on a reschedule and remove it on a cancel. That needs the same
`UID` across the chain, a rising `SEQUENCE`, and `METHOD:REQUEST` or `METHOD:CANCEL`. The host's e-mail
carries the same event without the manage link, unless the module writes the booking into a calendar the host
connected: that calendar already has it, and an ICS as well would add it twice.

```ts
// lib/scheduling/ics.ts: one iTIP event (RFC 5545 and 5546). The UID is stable per booking chain and SEQUENCE
// rises with every change, so a calendar updates or removes the event it already has instead of adding one.

export interface IcsEvent {
  uid: string;
  sequence: number;
  method: 'REQUEST' | 'CANCEL';
  start: Date;
  end: Date;
  summary: string;
  description: string;
  location: string | null;
  url: string | null;
  organizer: { name: string; email: string };
  attendee: { name: string; email: string };
  now: Date;
}

function utc(d: Date): string {
  return d.toISOString().replace(/[-:]/g, '').replace(/\.\d{3}/, '');
}

/** TEXT escaping: backslash first, then ; , and newlines. */
export function escapeText(value: string): string {
  return value.replace(/\\/g, '\\\\').replace(/;/g, '\;').replace(/,/g, '\\,').replace(/\r?\n/g, '\\n');
}

/** Quoted parameter values may not contain DQUOTE; strip it and control characters. */
function param(value: string): string {
  return value.replace(/["\u0000-\u001f]/g, '');
}

/** Fold at 75 octets, never inside a UTF-8 sequence, continuation lines start with one space. */
export function fold(line: string): string {
  const bytes = Buffer.from(line, 'utf8');
  if (bytes.length <= 75) return line;
  const parts: string[] = [];
  let start = 0;
  let limit = 75;
  while (start < bytes.length) {
    let end = Math.min(start + limit, bytes.length);
    // Step back off UTF-8 continuation bytes (10xxxxxx).
    while (end < bytes.length && ((bytes[end] ?? 0) & 0xc0) === 0x80) end--;
    parts.push(bytes.subarray(start, end).toString('utf8'));
    start = end;
    limit = 74;
  }
  return parts.join('\r\n ');
}

export function buildIcs(e: IcsEvent): string {
  const lines = [
    'BEGIN:VCALENDAR',
    'VERSION:2.0',
    'PRODID:-//scheduling-links//EN',
    'CALSCALE:GREGORIAN',
    `METHOD:${e.method}`,
    'BEGIN:VEVENT',
    `UID:${e.uid}`,
    `SEQUENCE:${e.sequence}`,
    `DTSTAMP:${utc(e.now)}`,
    `DTSTART:${utc(e.start)}`,
    `DTEND:${utc(e.end)}`,
    `SUMMARY:${escapeText(e.summary)}`,
    `DESCRIPTION:${escapeText(e.description)}`,
    ...(e.location ? [`LOCATION:${escapeText(e.location)}`] : []),
    ...(e.url ? [`URL:${e.url}`] : []),
    `ORGANIZER;CN="${param(e.organizer.name)}":mailto:${e.organizer.email}`,
    `ATTENDEE;CN="${param(e.attendee.name)}";ROLE=REQ-PARTICIPANT;PARTSTAT=ACCEPTED;RSVP=FALSE:mailto:${e.attendee.email}`,
    `STATUS:${e.method === 'CANCEL' ? 'CANCELLED' : 'CONFIRMED'}`,
    'TRANSP:OPAQUE',
    'END:VEVENT',
    'END:VCALENDAR',
  ];
  return `${lines.map(fold).join('\r\n')}\r\n`;
}

/** One UID for a booking and every booking that replaced it, so a reschedule moves the invitee's event. */
export function icsUid(rootBookingId: string, domain: string): string {
  return `${rootBookingId}@${domain}`;
}
```

## Webhooks

Signed deliveries and the URL guard are in [webhooks.md](webhooks.md).
