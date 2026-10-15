# Testing the flows

The three suites that run the module end to end: the booking service with the runner, the edges (ICS,
webhooks, tokens, provider normalization), and the store contract that runs on every store. Setup and
fixtures are in [testing.md](testing.md).

## Booking and jobs

```ts
// test/booking.test.ts: the service and the runner together, against the in-memory store.
import { describe, expect, it } from 'vitest';
import { cancelBooking, createBooking, listSlots, rescheduleBooking, SchedulingError } from '@/lib/scheduling/booking-service';
import { runDueJobs } from '@/lib/scheduling/runner';
import { hashToken } from '@/lib/scheduling/tokens';
import { clock, eventType, memoryStore, recordingCalendar, recordingMailer } from './fixtures';

// Monday 2026-10-12, 08:00 in Warsaw.
const NOW = '2026-10-12T06:00:00Z';
const input = (start: string, over: Record<string, unknown> = {}) => ({
  eventTypeId: 'et1',
  start,
  name: 'Iza <b>Guest</b>',
  email: 'Iza@Example.com',
  timeZone: 'America/New_York',
  answers: { topic: 'Pricing', injected: 'ignored' },
  ...over,
});

function setup(et = eventType()) {
  const c = clock(NOW);
  const m = memoryStore([et]);
  const mail = recordingMailer();
  const cal = recordingCalendar();
  const deps = { store: m.store, now: c.now };
  const runner = { store: m.store, mailer: mail.mailer, calendar: cal.calendar, now: c.now, appUrl: 'https://book.example.com' };
  return { c, m, mail, cal, deps, runner };
}

async function code(p: Promise<unknown>): Promise<string> {
  try {
    await p;
    return 'ok';
  } catch (e) {
    return e instanceof SchedulingError ? e.code : String(e);
  }
}

describe('booking', () => {
  it('books a listed slot, stores only the token hash and known answers', async () => {
    const { deps, m } = setup();
    const slots = await listSlots(deps, 'et1', new Date('2026-10-12T00:00:00Z'), new Date('2026-10-13T00:00:00Z'));
    const first = slots[0];
    expect(first?.start.toISOString()).toBe('2026-10-12T07:00:00.000Z');
    const { booking, manageToken } = await createBooking(deps, input('2026-10-12T07:00:00Z'));
    expect(booking.inviteeEmail).toBe('iza@example.com');
    expect(booking.answers).toEqual({ topic: 'Pricing' });
    expect(booking.chainId).toBe(booking.id);
    expect(booking.manageTokenHash).toBe(hashToken(manageToken));
    expect(JSON.stringify([...m.bookings.values()])).not.toContain(manageToken);
  });

  it('refuses a start that is not a slot: off the grid, inside notice, or outside hours', async () => {
    const { deps } = setup(eventType({ minNoticeMinutes: 120 }));
    expect(await code(createBooking(deps, input('2026-10-12T07:10:00Z')))).toBe('slot_unavailable');
    expect(await code(createBooking(deps, input('2026-10-12T07:00:00Z')))).toBe('slot_unavailable');
    expect(await code(createBooking(deps, input('2026-10-12T16:00:00Z')))).toBe('slot_unavailable');
  });

  it('lets exactly one of two simultaneous requests for the same slot through', async () => {
    const { deps } = setup();
    const results = await Promise.all([code(createBooking(deps, input('2026-10-12T08:00:00Z'))), code(createBooking(deps, input('2026-10-12T08:00:00Z', { email: 'other@example.com' })))]);
    expect(results.sort()).toEqual(['ok', 'slot_full']);
  });

  it('rejects invalid input before touching the store', async () => {
    const { deps } = setup();
    expect(await code(createBooking(deps, input('2026-10-12T08:00:00Z', { timeZone: 'Mars/Base' })))).toBe('invalid_input');
    expect(await code(createBooking(deps, input('2026-10-12T08:00:00Z', { email: 'nope' })))).toBe('invalid_input');
    const required = setup(eventType({ questions: [{ id: 'phone', label: 'Phone', type: 'phone', required: true }] }));
    expect(await code(createBooking(required.deps, input('2026-10-12T08:00:00Z')))).toBe('invalid_input');
  });

  it('writes e-mails, the calendar job and reminders with the booking, and sends them once', async () => {
    const { deps, runner, mail, cal, m } = setup();
    await createBooking(deps, input('2026-10-14T08:00:00Z'));
    expect(m.jobs.map(j => j.kind).sort()).toEqual(['calendar.upsert', 'email.host', 'email.invitee', 'reminder.invitee', 'reminder.invitee']);
    const first = await runDueJobs(runner, 1000);
    expect(first.done).toBe(3);
    expect(mail.sent.map(s => s.to.email).sort()).toEqual(['ana@example.com', 'iza@example.com']);
    expect(cal.calls).toHaveLength(1);
    expect((await runDueJobs(runner, 1000)).claimed).toBe(0);
  });

  it('escapes invitee input in HTML and formats the time in the invitee zone', async () => {
    const { deps, runner, mail } = setup();
    await createBooking(deps, input('2026-10-14T08:00:00Z'));
    await runDueJobs(runner, 1000);
    const invitee = mail.sent.find(s => s.to.email === 'iza@example.com');
    expect(invitee?.html).toContain('Iza &lt;b&gt;Guest&lt;/b&gt;');
    expect(invitee?.html).not.toContain('<b>Guest</b>');
    expect(invitee?.text).toContain('4:00 AM');
    expect(invitee?.attachments?.[0]?.type).toBe('text/calendar');
    expect(invitee?.text).toContain('https://book.example.com/booking/');
  });

  it('sends the host an invite without the manage link, and no notice for a booking cancelled before it ran', async () => {
    const { deps, runner, mail } = setup();
    const a = await createBooking(deps, input('2026-10-14T08:00:00Z'));
    await runDueJobs(runner, 1000);
    // The module writes this host's calendar, so the event is there and an ICS would add it a second time.
    expect(mail.sent.find(s => s.to.email === 'ana@example.com')?.attachments).toEqual([]);
    const bare = recordingMailer();
    const offline = { ...runner, mailer: bare.mailer, calendar: recordingCalendar(false).calendar };
    await cancelBooking(deps, a.manageToken, null);
    await createBooking(deps, input('2026-10-15T08:00:00Z', { email: 'b@example.com' }));
    await runDueJobs(offline, 1000);
    const host = bare.sent.filter(s => s.to.email === 'ana@example.com');
    expect(host).toHaveLength(2);
    // Unfold the ICS lines first, so a link split across a fold is still found.
    const ics = host.map(s => s.attachments?.[0]?.content.replace(/\r\n /g, '') ?? '');
    expect(ics.every(c => c.includes('BEGIN:VCALENDAR') && !c.includes('/booking/'))).toBe(true);
    // a was cancelled after its jobs ran; b is cancelled before they do, so the host hears only of the cancel.
    const late = setup();
    const b = await createBooking(late.deps, input('2026-10-14T08:00:00Z'));
    await cancelBooking(late.deps, b.manageToken, null);
    await runDueJobs(late.runner, 1000);
    expect(late.mail.sent.filter(s => s.to.email === 'ana@example.com').map(s => s.text.includes(', cancelled'))).toEqual([true]);
  });

  it('retries a failed e-mail with backoff, and does not resend what already went out', async () => {
    const { deps, runner, m } = setup();
    const flaky = recordingMailer(1);
    await createBooking(deps, input('2026-10-14T08:00:00Z'));
    const r = await runDueJobs({ ...runner, mailer: flaky.mailer }, 1000);
    expect(r.failed).toBe(1);
    const failed = m.jobs.find(j => j.lastError);
    expect(failed?.runAt.toISOString()).toBe('2026-10-12T06:01:00.000Z');
    runner.now = () => new Date('2026-10-12T06:02:00Z');
    await runDueJobs({ ...runner, mailer: flaky.mailer }, 1000);
    expect(flaky.sent).toHaveLength(2);
  });

  it('reschedules atomically: new booking on the same chain, old one cancelled, old reminders gone', async () => {
    const { deps, runner, m, cal } = setup();
    const { booking, manageToken } = await createBooking(deps, input('2026-10-14T08:00:00Z'));
    await runDueJobs(runner, 1000);
    const moved = await rescheduleBooking(deps, manageToken, '2026-10-15T09:00:00Z');
    expect(moved.booking.chainId).toBe(booking.id);
    expect(moved.booking.sequence).toBe(1);
    expect(moved.booking.externalEventId).toBe(`ev-${booking.id}`);
    expect(m.bookings.get(booking.id)?.status).toBe('cancelled');
    const reminders = m.jobs.filter(j => j.kind === 'reminder.invitee').map(j => (j.payload as { bookingId: string }).bookingId);
    expect(new Set(reminders)).toEqual(new Set([moved.booking.id]));
    await runDueJobs(runner, 1000);
    expect(cal.calls.at(-1)).toBe(`upsert:${booking.id}:2026-10-15T09:00:00.000Z`);
    expect(await code(rescheduleBooking(deps, manageToken, '2026-10-16T09:00:00Z'))).toBe('not_confirmed');
  });

  it('keeps the old booking when the new start is taken', async () => {
    const { deps, m } = setup();
    const a = await createBooking(deps, input('2026-10-14T08:00:00Z'));
    await createBooking(deps, input('2026-10-14T09:00:00Z', { email: 'b@example.com' }));
    expect(await code(rescheduleBooking(deps, a.manageToken, '2026-10-14T09:00:00Z'))).toBe('slot_unavailable');
    expect(m.bookings.get(a.booking.id)?.status).toBe('confirmed');
  });

  it('cancels with the token only, removes reminders, and refuses after the end', async () => {
    const { deps, m, c } = setup();
    const a = await createBooking(deps, input('2026-10-14T08:00:00Z'));
    expect(await code(cancelBooking(deps, 'not-the-token', null))).toBe('not_found');
    await cancelBooking(deps, a.manageToken, 'Conflict');
    expect(m.jobs.some(j => j.kind === 'reminder.invitee')).toBe(false);
    expect(m.jobs.some(j => j.kind === 'calendar.delete')).toBe(true);
    const b = await createBooking(deps, input('2026-10-12T07:00:00Z', { email: 'c@example.com' }));
    c.advance(120);
    expect(await code(cancelBooking(deps, b.manageToken, null))).toBe('too_late');
  });

  it('skips a reminder whose booking moved since it was scheduled', async () => {
    const { deps, runner, mail, m, c } = setup();
    const { manageToken } = await createBooking(deps, input('2026-10-14T08:00:00Z'));
    await runDueJobs(runner, 1000);
    const stale = m.jobs.find(j => j.kind === 'reminder.invitee');
    await rescheduleBooking(deps, manageToken, '2026-10-20T08:00:00Z');
    // Put the stale reminder back, as a crash between steps could have left it.
    if (stale) m.jobs.push({ ...stale, id: 'stale', done: false, runAt: new Date(NOW) });
    const before = mail.sent.length;
    c.advance(1);
    await runDueJobs(runner, 1000);
    expect(mail.sent.slice(before).some(s => s.subject.startsWith('Reminder'))).toBe(false);
  });

  it('lets a group slot fill seat by seat', async () => {
    const { deps } = setup(eventType({ kind: 'group', capacity: 2 }));
    expect(await code(createBooking(deps, input('2026-10-14T08:00:00Z')))).toBe('ok');
    expect(await code(createBooking(deps, input('2026-10-14T08:00:00Z', { email: 'b@example.com' })))).toBe('ok');
    expect(await code(createBooking(deps, input('2026-10-14T08:00:00Z', { email: 'c@example.com' })))).toBe('slot_unavailable');
  });
});
```

## Edges

```ts
// test/edges.test.ts: ICS, webhook signatures and the SSRF guard, tokens, sealing, provider normalization,
// calendar disconnect.
import type { LookupAddress } from 'node:dns';
import { isIP } from 'node:net';
import { afterEach, describe, expect, it, vi } from 'vitest';
import { googleEventId, googleEventsToBusy, OWN_EVENT_KEY } from '@/lib/scheduling/calendar/google';
import { graphEventsToBusy } from '@/lib/scheduling/calendar/microsoft';
import { PROVIDERS } from '@/lib/scheduling/calendar/oauth';
import { channelToken, disconnectCalendar, readChannelToken, type SyncDeps } from '@/lib/scheduling/calendar/sync';
import { ReauthRequired, type Connection, type Watch } from '@/lib/scheduling/calendar/types';
import { decryptSecret, encryptSecret } from '@/lib/scheduling/crypto';
import { buildIcs, escapeText, fold } from '@/lib/scheduling/ics';
import { isAuthorizedCron, manageTokenFor, signValue, verifySignedValue } from '@/lib/scheduling/tokens';
import {
  assertPublicUrl,
  isPrivateAddress,
  pinnedLookup,
  resolvePublicUrl,
  signatureHeader,
  verifySignature,
  type Resolver,
} from '@/lib/scheduling/webhooks';
import { memoryCalendars, memoryStore, TZ } from './fixtures';

const now = new Date('2026-10-12T06:00:00Z');

describe('ics', () => {
  const base = {
    uid: 'chain-1@book.example.com',
    sequence: 2,
    method: 'REQUEST' as const,
    start: new Date('2026-10-14T08:00:00Z'),
    end: new Date('2026-10-14T08:30:00Z'),
    summary: 'Intro; call, with Ana',
    description: 'Line one\nLine two',
    location: null,
    url: null,
    organizer: { name: 'Ana "Host"', email: 'ana@example.com' },
    attendee: { name: 'Iza', email: 'iza@example.com' },
    now,
  };

  it('writes a stable UID, the SEQUENCE, UTC times and CRLF line ends', () => {
    const ics = buildIcs(base);
    expect(ics).toContain('UID:chain-1@book.example.com\r\n');
    expect(ics).toContain('SEQUENCE:2\r\n');
    expect(ics).toContain('DTSTART:20261014T080000Z\r\n');
    expect(ics).toContain('ORGANIZER;CN="Ana Host":mailto:ana@example.com');
    expect(ics.split('\r\n').every(l => Buffer.byteLength(l) <= 75)).toBe(true);
  });

  it('cancels with METHOD:CANCEL and STATUS:CANCELLED', () => {
    const ics = buildIcs({ ...base, method: 'CANCEL' });
    expect(ics).toContain('METHOD:CANCEL');
    expect(ics).toContain('STATUS:CANCELLED');
  });

  it('escapes text and folds without splitting a UTF-8 character', () => {
    expect(escapeText('a;b,c\\d\ne')).toBe('a\;b\\,c\\\\d\\ne');
    const folded = fold(`DESCRIPTION:${'\u017c'.repeat(60)}`);
    for (const line of folded.split('\r\n')) expect(Buffer.from(line, 'utf8').toString('utf8')).not.toContain('\uFFFD');
  });
});

describe('webhooks', () => {
  it('signs and verifies, and refuses a tampered body or an old timestamp', () => {
    const body = '{"a":1}';
    const header = signatureHeader('whsec', body, now);
    expect(verifySignature('whsec', body, header, now)).toBe(true);
    expect(verifySignature('whsec', '{"a":2}', header, now)).toBe(false);
    expect(verifySignature('whsec', body, header, new Date(now.getTime() + 10 * 60_000))).toBe(false);
  });

  it('treats private, loopback, link-local, CGNAT, mapped and embedding addresses as private', () => {
    const blocked = [
      '10.0.0.1', '127.0.0.1', '169.254.169.254', '172.16.0.1', '192.168.1.1', '100.64.0.1', '0.0.0.0', '::', '::1',
      'fd00::1', 'fe80::1', '::ffff:127.0.0.1', '::ffff:7f00:1', '::ffff:a9fe:a9fe', '64:ff9b::7f00:1', '2002:7f00:1::1', 'not-an-ip',
    ];
    for (const ip of blocked) expect(isPrivateAddress(ip), ip).toBe(true);
    for (const ip of ['93.184.216.34', '1.1.1.1', '2606:4700:4700::1111', '::ffff:5db8:d822']) expect(isPrivateAddress(ip), ip).toBe(false);
  });

  it('refuses http, credentials and private hosts in every spelling, before any request', async () => {
    await expect(assertPublicUrl('http://example.com/hook')).rejects.toThrow('https');
    await expect(assertPublicUrl('https://user:pw@example.com/hook')).rejects.toThrow('credentials');
    // new URL rewrites these to [::ffff:7f00:1], [::ffff:a9fe:a9fe] and 127.0.0.1 before the check sees them.
    for (const url of ['https://169.254.169.254/latest', 'https://[::1]/x', 'https://[::ffff:127.0.0.1]/x', 'https://[::ffff:169.254.169.254]/x', 'https://2130706433/x', 'https://0x7f.1/x']) {
      await expect(assertPublicUrl(url), url).rejects.toThrow('private');
    }
  });

  it('refuses a hostname when any of its addresses is private, and keeps the ones it checked', async () => {
    const resolveTo = (...ips: string[]): Resolver => async () => ips.map(address => ({ address, family: isIP(address) }));
    await expect(resolvePublicUrl('https://hook.example.com/x', resolveTo('93.184.216.34', '10.0.0.5'))).rejects.toThrow('private');
    await expect(resolvePublicUrl('https://hook.example.com/x', resolveTo())).rejects.toThrow('private');
    const target = await resolvePublicUrl('https://hook.example.com/x', resolveTo('93.184.216.34'));
    expect(target.addresses).toEqual([{ address: '93.184.216.34', family: 4 }]);
  });

  it('connects to the checked address, whatever a second lookup would answer', async () => {
    const lookup = pinnedLookup([{ address: '93.184.216.34', family: 4 }]);
    const one = await new Promise(resolve => lookup('rebind.example.com', {}, (err, address, family) => resolve({ err, address, family })));
    expect(one).toEqual({ err: null, address: '93.184.216.34', family: 4 });
    const all = await new Promise<string | LookupAddress[]>(resolve => lookup('rebind.example.com', { all: true }, (_err, address) => resolve(address)));
    expect(all).toEqual([{ address: '93.184.216.34', family: 4 }]);
  });
});

describe('calendar disconnect', () => {
  const HOUR = 3_600_000;
  const connection = (over: Partial<Connection> = {}): Connection => ({
    id: 'c1',
    hostId: 'h1',
    provider: 'google',
    accountEmail: 'ana@example.com',
    accessTokenEnc: encryptSecret('access-1'),
    refreshTokenEnc: encryptSecret('refresh-1'),
    accessExpiresAt: new Date(now.getTime() + HOUR),
    status: 'active',
    checkCalendarIds: ['primary'],
    writeCalendarId: 'primary',
    syncedAt: null,
    ...over,
  });
  const watch: Watch = { id: 'w1', connectionId: 'c1', calendarId: 'primary', channelId: 'ch1', resourceId: 'r1', expiresAt: new Date(now.getTime() + 24 * HOUR) };
  const depsFor = (m: ReturnType<typeof memoryCalendars>): SyncDeps => ({ calendars: m.calendars, store: memoryStore().store, now: () => now, appUrl: 'https://book.example.com' });

  afterEach(() => {
    vi.restoreAllMocks();
  });

  it('stops the channels, then deletes the connection with its watches and busy time', async () => {
    const m = memoryCalendars([connection()], [watch]);
    await m.calendars.replaceBusy('c1', 'h1', [{ start: now, end: new Date(now.getTime() + HOUR) }]);
    const stop = vi.spyOn(PROVIDERS.google, 'stopWatch').mockResolvedValue(undefined);
    expect(await disconnectCalendar(depsFor(m), 'h1', 'c1')).toBe(true);
    expect(stop).toHaveBeenCalledWith('access-1', expect.objectContaining({ channelId: 'ch1', resourceId: 'r1' }));
    expect(await m.calendars.getConnection('c1')).toBeNull();
    expect(await m.calendars.getWatchByChannel('ch1')).toBeNull();
    expect(m.busy).toHaveLength(0);
  });

  it("refuses another host's connection, and still removes one whose grant is dead", async () => {
    const m = memoryCalendars([connection({ accessExpiresAt: new Date(now.getTime() - 1) })], [watch]);
    const stop = vi.spyOn(PROVIDERS.google, 'stopWatch').mockResolvedValue(undefined);
    vi.spyOn(PROVIDERS.google, 'refresh').mockRejectedValue(new ReauthRequired('invalid_grant'));
    expect(await disconnectCalendar(depsFor(m), 'h2', 'c1')).toBe(false);
    expect(await m.calendars.getConnection('c1')).not.toBeNull();
    expect(await disconnectCalendar(depsFor(m), 'h1', 'c1')).toBe(true);
    expect(stop).not.toHaveBeenCalled();
    expect(await m.calendars.getConnection('c1')).toBeNull();
  });
});

describe('tokens and secrets', () => {
  it('derives the same manage token for a booking and a different one for another', () => {
    expect(manageTokenFor('b1').token).toBe(manageTokenFor('b1').token);
    expect(manageTokenFor('b1').token).not.toBe(manageTokenFor('b2').token);
  });

  it('fails closed on cron auth when the secret is unset or short', () => {
    expect(isAuthorizedCron(null, undefined)).toBe(false);
    expect(isAuthorizedCron('Bearer undefined', undefined)).toBe(false);
    expect(isAuthorizedCron('Bearer short', 'short')).toBe(false);
    expect(isAuthorizedCron('Bearer a-long-enough-secret-123', 'a-long-enough-secret-123')).toBe(true);
  });

  it('signs values that expire and cannot be altered', () => {
    const v = signValue('k'.repeat(32), 'google:h1:n', new Date(now.getTime() + 60_000));
    expect(verifySignedValue('k'.repeat(32), v, now)).toBe('google:h1:n');
    expect(verifySignedValue('k'.repeat(32), v.replace('h1', 'h2'), now)).toBeNull();
    expect(verifySignedValue('k'.repeat(32), v, new Date(now.getTime() + 120_000))).toBeNull();
  });

  it('keeps channel tokens under the 128-character Graph clientState limit', () => {
    const t = channelToken('6f1c2a8e-3b4d-4e5f-9a1b-2c3d4e5f6a7b', '0b5e7f2a-1c3d-4e5f-8a9b-0c1d2e3f4a5b', new Date(now.getTime() + 8 * 86_400_000));
    expect(t.length).toBeLessThanOrEqual(128);
    expect(readChannelToken(t, now)?.channelId).toBe('0b5e7f2a-1c3d-4e5f-8a9b-0c1d2e3f4a5b');
  });

  it('seals secrets with AES-GCM and refuses a tampered one', () => {
    const sealed = encryptSecret('refresh-token');
    expect(decryptSecret(sealed)).toBe('refresh-token');
    const parts = sealed.split('.');
    parts[3] = Buffer.from('other-token').toString('base64url');
    expect(() => decryptSecret(parts.join('.'))).toThrow();
  });
});

describe('provider normalization', () => {
  it('Google: leaves out free, cancelled, declined, own and working-location events; all-day spans its days', () => {
    const busy = googleEventsToBusy(
      [
        { id: '1', start: { dateTime: '2026-10-14T10:00:00+02:00' }, end: { dateTime: '2026-10-14T11:00:00+02:00' } },
        { id: '2', transparency: 'transparent', start: { dateTime: '2026-10-14T12:00:00Z' }, end: { dateTime: '2026-10-14T13:00:00Z' } },
        { id: '3', status: 'cancelled' },
        { id: '4', attendees: [{ self: true, responseStatus: 'declined' }], start: { dateTime: '2026-10-14T14:00:00Z' }, end: { dateTime: '2026-10-14T15:00:00Z' } },
        { id: '5', extendedProperties: { private: { [OWN_EVENT_KEY]: 'c1' } }, start: { dateTime: '2026-10-14T16:00:00Z' }, end: { dateTime: '2026-10-14T17:00:00Z' } },
        { id: '6', eventType: 'workingLocation', start: { date: '2026-10-15' }, end: { date: '2026-10-16' } },
        { id: '7', start: { date: '2026-10-19' }, end: { date: '2026-10-22' } },
      ],
      TZ,
    );
    expect(busy.map(b => [b.start.toISOString(), b.end.toISOString()])).toEqual([
      ['2026-10-14T08:00:00.000Z', '2026-10-14T09:00:00.000Z'],
      ['2026-10-18T22:00:00.000Z', '2026-10-21T22:00:00.000Z'],
    ]);
  });

  it('Graph: reads UTC times without offset, leaves out free, cancelled, declined and own events', () => {
    const busy = graphEventsToBusy(
      [
        { id: '1', showAs: 'busy', start: { dateTime: '2026-10-14T08:00:00.0000000' }, end: { dateTime: '2026-10-14T09:00:00.0000000' } },
        { id: '2', showAs: 'tentative', start: { dateTime: '2026-10-14T10:00:00.0000000' }, end: { dateTime: '2026-10-14T10:30:00.0000000' } },
        { id: '3', showAs: 'free', start: { dateTime: '2026-10-14T11:00:00.0000000' }, end: { dateTime: '2026-10-14T12:00:00.0000000' } },
        { id: '4', isCancelled: true, start: { dateTime: '2026-10-14T12:00:00.0000000' }, end: { dateTime: '2026-10-14T13:00:00.0000000' } },
        { id: '5', responseStatus: { response: 'declined' }, start: { dateTime: '2026-10-14T13:00:00.0000000' }, end: { dateTime: '2026-10-14T14:00:00.0000000' } },
        { id: '6', singleValueExtendedProperties: [{ id: 'x', value: 'c1' }], start: { dateTime: '2026-10-14T14:00:00.0000000' }, end: { dateTime: '2026-10-14T15:00:00.0000000' } },
        { id: '7', isAllDay: true, start: { dateTime: '2026-10-19T00:00:00.0000000' }, end: { dateTime: '2026-10-20T00:00:00.0000000' } },
      ],
      TZ,
    );
    expect(busy.map(b => b.start.toISOString())).toEqual(['2026-10-14T08:00:00.000Z', '2026-10-14T10:00:00.000Z', '2026-10-18T22:00:00.000Z']);
  });

  it('makes a valid Google event id from a chain id', () => {
    const id = googleEventId('0b5e7f2a-1c3d-4e5f-8a9b-0c1d2e3f4a5b');
    expect(id).toMatch(/^[a-v0-9]{5,1024}$/);
  });
});
```

## Store contract

One contract in `test/store-contract.ts`, and one small test file per store that hands it a backend. Copy
the contract, the in-memory file, and the file for the host's store: `store-contract.supabase.test.ts` on
Supabase, `store-contract.firestore.test.ts` on Firestore. The other store's file is the only part of the
suites a host leaves out; nothing inside a file is cut. `npx vitest run test/store-contract` runs every
store whose file is present and whose environment is set.

```ts
// test/store-contract.ts: one contract, every store. Not a test file itself: each store's test file below
// passes its backend in, so a Supabase app never compiles the Firestore seed and a Firestore app never the
// Supabase one.
import { describe, expect, it } from 'vitest';
import { cancelBooking, createBooking, rescheduleBooking, SchedulingError } from '@/lib/scheduling/booking-service';
import type { SchedulingStore } from '@/lib/scheduling/store';
import type { EventType } from '@/lib/scheduling/types';
import { clock, eventType } from './fixtures';

export interface Backend {
  name: string;
  enabled: boolean;
  /** Seeds a fresh host with the fixture schedule and the given event types; returns a store and the ids. */
  seed: (ets: EventType[]) => Promise<{ store: SchedulingStore; ids: Record<string, string> }>;
}

const person = (eventTypeId: string, start: string, email = 'iza@example.com') => ({ eventTypeId, start, name: 'Iza', email, timeZone: 'Europe/Warsaw', answers: {} });

async function code(p: Promise<unknown>): Promise<string> {
  try {
    await p;
    return 'ok';
  } catch (e) {
    return e instanceof SchedulingError ? e.code : String(e);
  }
}

export function storeContract(backend: Backend): void {
  describe.skipIf(!backend.enabled)(`store contract: ${backend.name}`, () => {
    const ets = [eventType({ id: 'et-a', slug: 'a' }), eventType({ id: 'et-b', slug: 'b', kind: 'group', capacity: 2, durationMinutes: 60, slotIntervalMinutes: 60, dailyLimit: 1 })];

    it('books, refuses the same slot and an overlapping slot of another event type', async () => {
      const { store, ids } = await backend.seed(ets);
      const deps = { store, now: clock('2026-10-12T06:00:00Z').now };
      const a = ids.a as string;
      const b = ids.b as string;
      const first = await createBooking(deps, person(a, '2026-10-14T08:00:00Z'));
      expect(first.booking.status).toBe('confirmed');
      expect(await code(createBooking(deps, person(a, '2026-10-14T08:00:00Z', 'x@example.com')))).toBe('slot_unavailable');
      expect(await code(createBooking(deps, person(b, '2026-10-14T08:00:00Z', 'y@example.com')))).toBe('slot_unavailable');
    });

    it('lets exactly one of several concurrent requests win', async () => {
      const { store, ids } = await backend.seed(ets);
      const deps = { store, now: clock('2026-10-12T06:00:00Z').now };
      const results = await Promise.all([0, 1, 2, 3].map(i => code(createBooking(deps, person(ids.a as string, '2026-10-15T10:00:00Z', `p${i}@example.com`)))));
      expect(results.filter(r => r === 'ok')).toHaveLength(1);
    });

    it('fills a group slot seat by seat and holds the daily limit', async () => {
      const { store, ids } = await backend.seed(ets);
      const deps = { store, now: clock('2026-10-12T06:00:00Z').now };
      const b = ids.b as string;
      expect(await code(createBooking(deps, person(b, '2026-10-16T08:00:00Z', 'a@example.com')))).toBe('ok');
      expect(await code(createBooking(deps, person(b, '2026-10-16T08:00:00Z', 'b@example.com')))).toBe('ok');
      expect(await code(createBooking(deps, person(b, '2026-10-16T08:00:00Z', 'c@example.com')))).toBe('slot_unavailable');
      expect(await code(createBooking(deps, person(b, '2026-10-16T12:00:00Z', 'd@example.com')))).toBe('slot_unavailable');
    });

    it('reschedules into an overlapping time and cancels with the token', async () => {
      const { store, ids } = await backend.seed(ets);
      const deps = { store, now: clock('2026-10-12T06:00:00Z').now };
      const first = await createBooking(deps, person(ids.a as string, '2026-10-14T08:00:00Z'));
      const moved = await rescheduleBooking(deps, first.manageToken, '2026-10-14T08:30:00Z');
      expect(moved.booking.chainId).toBe(first.booking.id);
      expect((await store.getBooking(first.booking.id))?.status).toBe('cancelled');
      const cancelled = await cancelBooking(deps, moved.manageToken, 'Sick');
      expect(cancelled.status).toBe('cancelled');
      expect(cancelled.sequence).toBe(2);
    });

    it('claims a due job once while its lease runs', async () => {
      const { store, ids } = await backend.seed(ets);
      const now = new Date('2026-10-12T06:00:00Z');
      await createBooking({ store, now: () => now }, person(ids.a as string, '2026-10-20T08:00:00Z'));
      const claimed = await store.claimJobs(100, now, 60);
      const again = await store.claimJobs(100, now, 60);
      const mine = claimed.filter(j => !again.some(x => x.id === j.id));
      expect(mine.length).toBeGreaterThan(0);
    });
  });
}
```

```ts
// test/store-contract.test.ts: the contract against the in-memory store. Always runs, on every stack.
import { memoryStore } from './fixtures';
import { storeContract } from './store-contract';

storeContract({
  name: 'memory',
  enabled: true,
  seed: async ets => ({ store: memoryStore(ets).store, ids: Object.fromEntries(ets.map(e => [e.slug, e.id])) }),
});
```

```ts
// test/store-contract.supabase.test.ts: Supabase apps only. Runs when SUPABASE_URL and
// SUPABASE_SERVICE_ROLE_KEY are set, against a database with the migration applied; it seeds test rows.
import { randomUUID } from 'node:crypto';
import { host, schedule } from './fixtures';
import { storeContract } from './store-contract';

storeContract({
  name: 'supabase',
  enabled: Boolean(process.env.SUPABASE_URL && process.env.SUPABASE_SERVICE_ROLE_KEY),
  seed: async ets => {
    const { supabaseAdmin, supabaseStore } = await import('@/lib/scheduling/supabase-store');
    const db = supabaseAdmin();
    const hostId = `h-${randomUUID()}`;
    const ok = <T>(r: { data: T; error: { message: string } | null }) => {
      if (r.error) throw new Error(r.error.message);
      return r.data;
    };
    ok(await db.from('scheduling_hosts').insert({ id: hostId, handle: `t${randomUUID().slice(0, 8)}`, name: host.name, email: host.email, time_zone: host.timeZone }));
    const s = ok(await db.from('schedules').insert({ host_id: hostId, name: schedule.name, time_zone: schedule.timeZone, rules: schedule.rules, overrides: schedule.overrides }).select('id').single()) as { id: string };
    const ids: Record<string, string> = {};
    for (const e of ets) {
      const r = ok(
        await db
          .from('event_types')
          .insert({
            host_id: hostId, schedule_id: s.id, slug: e.slug, title: e.title, kind: e.kind, capacity: e.capacity,
            duration_minutes: e.durationMinutes, slot_interval_minutes: e.slotIntervalMinutes,
            buffer_before_minutes: e.bufferBeforeMinutes, buffer_after_minutes: e.bufferAfterMinutes,
            min_notice_minutes: e.minNoticeMinutes, horizon_days: e.horizonDays, daily_limit: e.dailyLimit,
            location_kind: e.locationKind, questions: e.questions,
          })
          .select('id')
          .single(),
      ) as { id: string };
      ids[e.slug] = r.id;
    }
    return { store: supabaseStore(db), ids };
  },
});
```

```ts
// test/store-contract.firestore.test.ts: Firestore apps only. Runs when FIRESTORE_EMULATOR_HOST is set.
import { randomUUID } from 'node:crypto';
import { host, schedule } from './fixtures';
import { storeContract } from './store-contract';

storeContract({
  name: 'firestore',
  enabled: Boolean(process.env.FIRESTORE_EMULATOR_HOST),
  seed: async ets => {
    const { initializeApp, getApps } = await import('firebase-admin/app');
    const { getFirestore } = await import('firebase-admin/firestore');
    const { firestoreStore } = await import('@/lib/scheduling/firestore-store');
    if (getApps().length === 0) initializeApp({ projectId: 'demo-scheduling' });
    const db = getFirestore();
    const hostId = `h-${randomUUID()}`;
    await db.collection('schedulingHosts').doc(hostId).set({ handle: hostId, name: host.name, email: host.email, timeZone: host.timeZone });
    const scheduleId = `s-${randomUUID()}`;
    await db.collection('schedules').doc(scheduleId).set({ ...schedule, id: scheduleId, hostId });
    const ids: Record<string, string> = {};
    for (const e of ets) {
      const id = `et-${randomUUID()}`;
      ids[e.slug] = id;
      await db.collection('eventTypes').doc(id).set({ ...e, id, hostId, scheduleId });
    }
    return { store: firestoreStore(db), ids };
  },
});
```
