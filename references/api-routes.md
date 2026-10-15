# API routes and runtime

Every route is a thin shell: parse, call one service function, map errors. The seams are wired once in
`runtime.ts`.

## Surface

| Route | Method | Who | Does |
|---|---|---|---|
| `/api/scheduling/slots?eventTypeId&from&to` | GET | Public | Lists slots, `no-store` |
| `/api/scheduling/bookings` | POST | Public | `createBooking`, then drains jobs in `after()` |
| `/api/scheduling/manage/[token]` | GET, POST | Token holder | Reads the booking; `cancel` or `reschedule` |
| `/api/scheduling/manage/[token]/ics` | GET | Token holder | The ICS download |
| `/api/scheduling/cron` | GET | `CRON_SECRET` | Jobs, watch renewal, calendar sync |
| `/api/scheduling/calendar/[provider]/connect` | GET | Signed-in host | Redirects to consent |
| `/api/scheduling/calendar/[provider]/callback` | GET | Signed-in host | Stores the connection, first sync, watches |
| `/api/scheduling/calendar/connections/[id]` | DELETE | Signed-in host | Stops the channels, deletes the connection |
| `/api/scheduling/calendar/google/notify` | POST | Google | Verifies the channel token, asks for a sync |
| `/api/scheduling/calendar/microsoft/notify` | POST | Microsoft | Validation handshake, then the same |

The host's dashboard (event types, schedules, calendars, bookings list) is the host app's own pages; see
[booking-page-ui.md](booking-page-ui.md).

## Seams

The auth seam. On Supabase Auth it is copied unchanged: it reads `NEXT_PUBLIC_SUPABASE_URL` and
`NEXT_PUBLIC_SUPABASE_ANON_KEY`, and needs no other helper or variable. Replace the body for Clerk, Auth.js or
a custom session; keep the signature.

```ts
// lib/scheduling/host.ts: the auth seam. Return the signed-in user's id, or null. The id is the host id.
// This version reads a Supabase Auth session through @supabase/ssr; replace the body for Clerk, Auth.js or
// your own session, keeping the signature.
import { createServerClient } from '@supabase/ssr';
import { cookies } from 'next/headers';

export async function currentHostId(): Promise<string | null> {
  const store = await cookies();
  const supabase = createServerClient(process.env.NEXT_PUBLIC_SUPABASE_URL ?? '', process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY ?? '', {
    cookies: {
      getAll: () => store.getAll(),
      setAll: () => {
        // Route handlers that only read the session never need to write cookies here.
      },
    },
  });
  const { data } = await supabase.auth.getUser();
  return data.user?.id ?? null;
}
```

```ts
// lib/scheduling/runtime.ts: wires the seams once. Swap supabaseStore for firestoreStore, the mailer for yours.
import { supabaseCalendarStore } from '@/lib/scheduling/calendar/supabase-calendar-store';
import { calendarWriter, type SyncDeps } from '@/lib/scheduling/calendar/sync';
import { sendgridMailer } from '@/lib/scheduling/mailer';
import type { RunnerDeps } from '@/lib/scheduling/runner';
import type { ServiceDeps } from '@/lib/scheduling/booking-service';
import { supabaseStore } from '@/lib/scheduling/supabase-store';

export function appUrl(): string {
  const url = process.env.APP_URL;
  if (!url) throw new Error('APP_URL is required, for example https://book.example.com');
  return url.replace(/\/$/, '');
}

export function serviceDeps(): ServiceDeps {
  return { store: supabaseStore(), now: () => new Date() };
}

export function syncDeps(): SyncDeps {
  return { calendars: supabaseCalendarStore(), store: supabaseStore(), now: () => new Date(), appUrl: appUrl() };
}

export function runnerDeps(): RunnerDeps {
  const sync = syncDeps();
  return { store: sync.store, mailer: sendgridMailer(), calendar: calendarWriter(sync), now: sync.now, appUrl: sync.appUrl };
}
```

## Secrets and tokens

```ts
// lib/scheduling/tokens.ts: manage tokens, HMAC signatures and secret comparison. Node runtime only.
import { createHash, createHmac, randomBytes, timingSafeEqual } from 'node:crypto';

/**
 * The manage token is derived from the booking id with a server secret, so a job can rebuild the link for an
 * e-mail at send time while the database holds only its hash. Rotating SCHEDULING_TOKEN_SECRET invalidates
 * every link already sent.
 */
export function manageTokenFor(bookingId: string, secret = process.env.SCHEDULING_TOKEN_SECRET): { token: string; hash: string } {
  if (!secret || secret.length < 32) throw new Error('SCHEDULING_TOKEN_SECRET must be at least 32 characters');
  const token = createHmac('sha256', secret).update(`manage:${bookingId}`).digest('base64url');
  return { token, hash: hashToken(token) };
}

export function randomSecret(bytes = 32): string {
  return randomBytes(bytes).toString('base64url');
}

/** The database stores this, so a leaked table cannot cancel anyone's booking. */
export function hashToken(token: string): string {
  return createHash('sha256').update(token).digest('hex');
}

export function safeEqual(a: string, b: string): boolean {
  const ab = Buffer.from(a);
  const bb = Buffer.from(b);
  return ab.length === bb.length && timingSafeEqual(ab, bb);
}

export function hmacHex(secret: string, message: string): string {
  return createHmac('sha256', secret).update(message).digest('hex');
}

/**
 * Cron and internal routes: fails closed. An unset secret rejects every request instead of
 * comparing undefined with undefined.
 */
export function isAuthorizedCron(authorization: string | null, secret: string | undefined): boolean {
  if (!secret || secret.length < 16 || !authorization) return false;
  return safeEqual(authorization, `Bearer ${secret}`);
}

/** 22 base64url characters, 132 bits: short enough for Graph's 128-character clientState. */
function shortSig(secret: string, body: string): string {
  return createHmac('sha256', secret).update(body).digest('base64url').slice(0, 22);
}

/** Signed, expiring value for OAuth state and channel tokens: `${payload}.${expires}.${sig}`. */
export function signValue(secret: string, payload: string, expiresAt: Date): string {
  const body = `${payload}.${expiresAt.getTime()}`;
  return `${body}.${shortSig(secret, body)}`;
}

export function verifySignedValue(secret: string, value: string, now: Date): string | null {
  const last = value.lastIndexOf('.');
  const mid = value.lastIndexOf('.', last - 1);
  if (last < 0 || mid < 0) return null;
  const body = value.slice(0, last);
  if (!safeEqual(value.slice(last + 1), shortSig(secret, body))) return null;
  const expires = Number(value.slice(mid + 1, last));
  if (!Number.isFinite(expires) || expires < now.getTime()) return null;
  return value.slice(0, mid);
}
```

```ts
// lib/scheduling/crypto.ts: AES-256-GCM for OAuth tokens and webhook secrets at rest.
// SCHEDULING_ENCRYPTION_KEY is 32 bytes, base64. Rotate by adding a key id prefix before the first rotation.
import { createCipheriv, createDecipheriv, randomBytes } from 'node:crypto';

function key(): Buffer {
  const raw = process.env.SCHEDULING_ENCRYPTION_KEY;
  const buf = raw ? Buffer.from(raw, 'base64') : Buffer.alloc(0);
  if (buf.length !== 32) throw new Error('SCHEDULING_ENCRYPTION_KEY must be 32 bytes, base64-encoded');
  return buf;
}

/** v1.<iv>.<tag>.<ciphertext>, base64url parts. */
export function encryptSecret(plaintext: string): string {
  const iv = randomBytes(12);
  const cipher = createCipheriv('aes-256-gcm', key(), iv);
  const data = Buffer.concat([cipher.update(plaintext, 'utf8'), cipher.final()]);
  return ['v1', iv, cipher.getAuthTag(), data].map(p => (typeof p === 'string' ? p : p.toString('base64url'))).join('.');
}

export function decryptSecret(sealed: string): string {
  const [version, iv, tag, data] = sealed.split('.');
  if (version !== 'v1' || !iv || !tag || data === undefined) throw new Error('Unsupported sealed secret');
  const decipher = createDecipheriv('aes-256-gcm', key(), Buffer.from(iv, 'base64url'));
  decipher.setAuthTag(Buffer.from(tag, 'base64url'));
  return Buffer.concat([decipher.update(Buffer.from(data, 'base64url')), decipher.final()]).toString('utf8');
}
```

## Routes

```ts
// lib/scheduling/http.ts: one mapping from SchedulingError to HTTP, shared by the routes.
import { NextResponse } from 'next/server';
import { SchedulingError } from '@/lib/scheduling/booking-service';

const STATUS: Record<SchedulingError['code'], number> = {
  not_found: 404,
  invalid_input: 400,
  slot_unavailable: 409,
  slot_taken: 409,
  slot_full: 409,
  day_full: 409,
  not_confirmed: 409,
  too_late: 409,
};

export function errorResponse(err: unknown): NextResponse {
  if (err instanceof SchedulingError) return NextResponse.json({ error: err.code, detail: err.message }, { status: STATUS[err.code] });
  console.error('[scheduling]', err);
  return NextResponse.json({ error: 'internal' }, { status: 500 });
}
```

```ts
// app/api/scheduling/slots/route.ts: GET ?eventTypeId=&from=&to= (ISO instants). Public.
import { NextResponse, type NextRequest } from 'next/server';
import { listSlots } from '@/lib/scheduling/booking-service';
import { errorResponse } from '@/lib/scheduling/http';
import { serviceDeps } from '@/lib/scheduling/runtime';

// No `export const dynamic`: cacheComponents refuses it, and reading the request already keeps this per call.
export async function GET(req: NextRequest) {
  const q = req.nextUrl.searchParams;
  const from = new Date(q.get('from') ?? '');
  const to = new Date(q.get('to') ?? '');
  if (Number.isNaN(from.getTime()) || Number.isNaN(to.getTime())) return NextResponse.json({ error: 'invalid_input' }, { status: 400 });
  try {
    const slots = await listSlots(serviceDeps(), q.get('eventTypeId') ?? '', from, to);
    return NextResponse.json(
      { slots: slots.map(s => ({ start: s.start.toISOString(), end: s.end.toISOString(), seatsLeft: s.seatsLeft })) },
      // Availability changes with every booking: never cache it at the edge.
      { headers: { 'cache-control': 'no-store' } },
    );
  } catch (err) {
    return errorResponse(err);
  }
}
```

```ts
// app/api/scheduling/bookings/route.ts: POST a booking. Public; the slot is recomputed on the server.
import { after, NextResponse, type NextRequest } from 'next/server';
import { createBooking } from '@/lib/scheduling/booking-service';
import { errorResponse } from '@/lib/scheduling/http';
import { runDueJobs } from '@/lib/scheduling/runner';
import { runnerDeps, serviceDeps } from '@/lib/scheduling/runtime';

export async function POST(req: NextRequest) {
  let body: unknown;
  try {
    body = await req.json();
  } catch {
    return NextResponse.json({ error: 'invalid_input' }, { status: 400 });
  }
  try {
    const { booking, manageToken } = await createBooking(serviceDeps(), body);
    // The jobs are already committed with the booking; this only makes the e-mail arrive in seconds.
    after(() => runDueJobs(runnerDeps(), 20_000).then(() => undefined));
    return NextResponse.json({ id: booking.id, start: booking.start.toISOString(), end: booking.end.toISOString(), manageToken }, { status: 201 });
  } catch (err) {
    return errorResponse(err);
  }
}
```

```ts
// app/api/scheduling/manage/[token]/route.ts: the invitee's own booking. GET to read, POST to change.
// The token is the only credential: it comes from the e-mail, and the database holds just its hash.
import { after, NextResponse, type NextRequest } from 'next/server';
import { z } from 'zod';
import { bookingForToken, cancelBooking, rescheduleBooking } from '@/lib/scheduling/booking-service';
import { errorResponse } from '@/lib/scheduling/http';
import { runDueJobs } from '@/lib/scheduling/runner';
import { runnerDeps, serviceDeps } from '@/lib/scheduling/runtime';

const action = z.discriminatedUnion('action', [
  z.object({ action: z.literal('cancel'), reason: z.string().max(500).optional() }),
  z.object({ action: z.literal('reschedule'), start: z.iso.datetime({ offset: true }) }),
]);

type Ctx = { params: Promise<{ token: string }> };

export async function GET(_req: NextRequest, { params }: Ctx) {
  try {
    const b = await bookingForToken(serviceDeps(), (await params).token);
    return NextResponse.json(
      { id: b.id, eventTypeId: b.eventTypeId, status: b.status, start: b.start.toISOString(), end: b.end.toISOString(), inviteeName: b.inviteeName, inviteeTimeZone: b.inviteeTimeZone, meetingUrl: b.meetingUrl },
      { headers: { 'cache-control': 'no-store', 'referrer-policy': 'no-referrer' } },
    );
  } catch (err) {
    return errorResponse(err);
  }
}

export async function POST(req: NextRequest, { params }: Ctx) {
  const parsed = action.safeParse(await req.json().catch(() => null));
  if (!parsed.success) return NextResponse.json({ error: 'invalid_input' }, { status: 400 });
  const { token } = await params;
  try {
    const deps = serviceDeps();
    const result =
      parsed.data.action === 'cancel'
        ? { booking: await cancelBooking(deps, token, parsed.data.reason ?? null), manageToken: null }
        : await rescheduleBooking(deps, token, parsed.data.start);
    after(() => runDueJobs(runnerDeps(), 20_000).then(() => undefined));
    return NextResponse.json({ id: result.booking.id, status: result.booking.status, start: result.booking.start.toISOString(), manageToken: result.manageToken });
  } catch (err) {
    return errorResponse(err);
  }
}
```

```ts
// app/api/scheduling/manage/[token]/ics/route.ts: "Add to calendar" download, behind the manage token.
import { NextResponse, type NextRequest } from 'next/server';
import { bookingForToken } from '@/lib/scheduling/booking-service';
import { errorResponse } from '@/lib/scheduling/http';
import { buildIcs, icsUid } from '@/lib/scheduling/ics';
import { manageUrl } from '@/lib/scheduling/runner';
import { appUrl, serviceDeps } from '@/lib/scheduling/runtime';

export async function GET(_req: NextRequest, { params }: { params: Promise<{ token: string }> }) {
  try {
    const deps = serviceDeps();
    const b = await bookingForToken(deps, (await params).token);
    const [et, host] = await Promise.all([deps.store.getEventType(b.eventTypeId), deps.store.getHost(b.hostId)]);
    if (!et || !host) return NextResponse.json({ error: 'not_found' }, { status: 404 });
    const body = buildIcs({
      uid: icsUid(b.chainId, new URL(appUrl()).hostname),
      sequence: b.sequence,
      method: b.status === 'cancelled' ? 'CANCEL' : 'REQUEST',
      start: b.start,
      end: b.end,
      summary: `${et.title} with ${host.name}`,
      description: `Reschedule or cancel: ${manageUrl(appUrl(), b.id)}`,
      location: b.meetingUrl,
      url: b.meetingUrl,
      organizer: { name: host.name, email: host.email },
      attendee: { name: b.inviteeName, email: b.inviteeEmail },
      now: deps.now(),
    });
    return new NextResponse(body, {
      headers: { 'content-type': 'text/calendar; charset=utf-8', 'content-disposition': 'attachment; filename="booking.ics"', 'cache-control': 'no-store' },
    });
  } catch (err) {
    return errorResponse(err);
  }
}
```

```ts
// app/api/scheduling/cron/route.ts: every 5 minutes. Drains jobs, renews watches, syncs calendars.
// Vercel Cron sends `Authorization: Bearer $CRON_SECRET`; Supabase pg_cron can call it the same way.
import { NextResponse, type NextRequest } from 'next/server';
import { runCalendarCron } from '@/lib/scheduling/calendar/sync';
import { runDueJobs } from '@/lib/scheduling/runner';
import { runnerDeps, syncDeps } from '@/lib/scheduling/runtime';
import { isAuthorizedCron } from '@/lib/scheduling/tokens';

export const maxDuration = 300;

export async function GET(req: NextRequest) {
  if (!isAuthorizedCron(req.headers.get('authorization'), process.env.CRON_SECRET)) {
    return NextResponse.json({ error: 'unauthorized' }, { status: 401 });
  }
  const jobs = await runDueJobs(runnerDeps(), 120_000);
  const calendar = await runCalendarCron(syncDeps());
  return NextResponse.json({ jobs, calendar });
}
```

```ts
// app/api/scheduling/calendar/[provider]/connect/route.ts: signed-in host only. Redirects to consent.
import { NextResponse, type NextRequest } from 'next/server';
import { isProviderId, PKCE_COOKIE, startConnect } from '@/lib/scheduling/calendar/oauth';
import { currentHostId } from '@/lib/scheduling/host';
import { appUrl } from '@/lib/scheduling/runtime';

export async function GET(_req: NextRequest, { params }: { params: Promise<{ provider: string }> }) {
  const { provider } = await params;
  if (!isProviderId(provider)) return NextResponse.json({ error: 'not_found' }, { status: 404 });
  const hostId = await currentHostId();
  if (!hostId) return NextResponse.json({ error: 'unauthorized' }, { status: 401 });
  const { url, verifier } = startConnect(appUrl(), provider, hostId, new Date());
  const res = NextResponse.redirect(url);
  res.cookies.set(PKCE_COOKIE, verifier, { httpOnly: true, secure: true, sameSite: 'lax', path: `/api/scheduling/calendar/${provider}/callback`, maxAge: 600 });
  return res;
}
```

```ts
// app/api/scheduling/calendar/[provider]/callback/route.ts: the provider returns here with code and state.
import { NextResponse, type NextRequest } from 'next/server';
import { finishConnect, isProviderId, PKCE_COOKIE } from '@/lib/scheduling/calendar/oauth';
import { ensureWatches, syncConnection } from '@/lib/scheduling/calendar/sync';
import { currentHostId } from '@/lib/scheduling/host';
import { appUrl, syncDeps } from '@/lib/scheduling/runtime';

export async function GET(req: NextRequest, { params }: { params: Promise<{ provider: string }> }) {
  const { provider } = await params;
  const q = req.nextUrl.searchParams;
  const back = (status: string) => {
    const res = NextResponse.redirect(`${appUrl()}/dashboard/calendars?connect=${status}`);
    res.cookies.delete({ name: PKCE_COOKIE, path: `/api/scheduling/calendar/${provider}/callback` });
    return res;
  };
  if (!isProviderId(provider) || q.get('error')) return back('cancelled');
  const hostId = await currentHostId();
  if (!hostId) return back('signed_out');
  try {
    const deps = syncDeps();
    const connection = await finishConnect({
      store: deps.calendars,
      appUrl: appUrl(),
      provider,
      sessionHostId: hostId,
      code: q.get('code') ?? '',
      state: q.get('state') ?? '',
      verifier: req.cookies.get(PKCE_COOKIE)?.value,
      now: new Date(),
    });
    await syncConnection(deps, connection.id).catch(() => undefined);
    await ensureWatches(deps, connection.id).catch(err => console.error('[scheduling] watch', err));
    return back('ok');
  } catch (err) {
    console.error('[scheduling] connect', err);
    return back('failed');
  }
}
```

```ts
// app/api/scheduling/calendar/connections/[id]/route.ts: the host disconnects a calendar. DELETE, not a GET
// link: a cross-site form cannot send it, and a cross-site fetch needs a preflight this route never grants.
import { NextResponse, type NextRequest } from 'next/server';
import { disconnectCalendar } from '@/lib/scheduling/calendar/sync';
import { currentHostId } from '@/lib/scheduling/host';
import { syncDeps } from '@/lib/scheduling/runtime';

const UUID = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;

export async function DELETE(_req: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const hostId = await currentHostId();
  if (!hostId) return NextResponse.json({ error: 'unauthorized' }, { status: 401 });
  // Another host's connection answers like a missing one, so ids cannot be probed.
  if (!UUID.test(id) || !(await disconnectCalendar(syncDeps(), hostId, id))) {
    return NextResponse.json({ error: 'not_found' }, { status: 404 });
  }
  return new NextResponse(null, { status: 204 });
}
```

```ts
// app/api/scheduling/calendar/google/notify/route.ts: Google push. Verify, ask for a sync, answer at once.
import { after, NextResponse, type NextRequest } from 'next/server';
import { readChannelToken, syncConnection } from '@/lib/scheduling/calendar/sync';
import { syncDeps } from '@/lib/scheduling/runtime';

export async function POST(req: NextRequest) {
  const channelId = req.headers.get('x-goog-channel-id') ?? '';
  const state = req.headers.get('x-goog-resource-state');
  const claims = readChannelToken(req.headers.get('x-goog-channel-token') ?? '', new Date());
  // The token must name this very channel: a token copied from another channel is refused.
  if (!claims || claims.channelId !== channelId) return new NextResponse(null, { status: 401 });
  if (state === 'sync') return new NextResponse(null, { status: 200 });
  const deps = syncDeps();
  const watch = await deps.calendars.getWatchByChannel(channelId);
  if (!watch || watch.connectionId !== claims.connectionId) return new NextResponse(null, { status: 404 });
  await deps.calendars.requestSync(watch.connectionId, new Date());
  after(() => syncConnection(deps, watch.connectionId).then(() => undefined, () => undefined));
  return new NextResponse(null, { status: 200 });
}
```

```ts
// app/api/scheduling/calendar/microsoft/notify/route.ts: Graph change notifications.
import { after, NextResponse, type NextRequest } from 'next/server';
import { readChannelToken, syncConnection } from '@/lib/scheduling/calendar/sync';
import { syncDeps } from '@/lib/scheduling/runtime';

export async function POST(req: NextRequest) {
  // Subscription creation handshake: echo the token as text/plain within 10 seconds.
  const validationToken = req.nextUrl.searchParams.get('validationToken');
  if (validationToken) return new NextResponse(validationToken, { status: 200, headers: { 'content-type': 'text/plain' } });
  const body = (await req.json().catch(() => null)) as { value?: { subscriptionId?: string; clientState?: string }[] } | null;
  const deps = syncDeps();
  const now = new Date();
  const toSync = new Set<string>();
  for (const n of body?.value ?? []) {
    const claims = readChannelToken(n.clientState ?? '', now);
    if (!claims || !n.subscriptionId) continue;
    const watch = await deps.calendars.getWatchByChannel(n.subscriptionId);
    if (watch && watch.connectionId === claims.connectionId) toSync.add(watch.connectionId);
  }
  for (const id of toSync) await deps.calendars.requestSync(id, now);
  after(async () => {
    for (const id of toSync) await syncConnection(deps, id).catch(() => undefined);
  });
  return new NextResponse(null, { status: 202 });
}
```

## Abuse

The booking route is public. Put the host app's rate limit in front of `POST /api/scheduling/bookings` and the
manage routes (per IP and per invitee e-mail); the module does not ship one because every host already has
its own (Vercel Firewall rules, a Redis counter, a Postgres counter). A bot that books every slot is a real
attack on a public scheduling page.
