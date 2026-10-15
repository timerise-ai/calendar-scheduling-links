# Calendar sync

Two directions, both optional for the core: busy time read from the host's calendars so slots avoid it, and
bookings written into one calendar so the host sees them. Google Calendar and Microsoft 365 or Outlook.com,
over their REST APIs with `fetch`, no SDKs.

## Shape

```
connect   /api/scheduling/calendar/<provider>/connect     signed state names the host, PKCE verifier in an
          /api/scheduling/calendar/<provider>/callback    HttpOnly cookie, callback checks both, tokens sealed
read      cron or push notice, syncConnection         lease, list events in the window, drop free,
                                                          cancelled, declined and own events, replace
                                                          busy_blocks in one transaction
write     job calendar.upsert / calendar.delete           event id derived from the chain: a retry or a
                                                          reschedule patches the same event
watch     ensureWatches after connect                     one channel per checked calendar; renewWatches
          renewWatches in the cron                        opens the new one before stopping the old
remove    DELETE /api/scheduling/calendar/connections/<id> stop the channels, then delete the connection
                                                          with its watches, busy blocks and tokens
```

Busy time is mirrored, not fetched per request: a slot query reads `busy_blocks` and never waits on Google or
Microsoft, and keeps working through their outages. A push notification only marks the connection for a sync;
the sync runs under a lease after the response, and the cron re-reads any connection not synced for 30
minutes, so a missed notification costs at most that.

## Contract

```ts
// lib/scheduling/calendar/types.ts: the provider contract and the connection records behind it.
import type { HostProfile } from '@/lib/scheduling/store';
import type { Booking, EventType, Interval } from '@/lib/scheduling/types';

export type ProviderId = 'google' | 'microsoft';

export interface TokenSet {
  accessToken: string;
  /** Google rarely returns a new one on refresh; Microsoft rotates it. Keep the old one when absent. */
  refreshToken: string | null;
  expiresAt: Date;
}

export interface ProviderCalendar {
  id: string;
  name: string;
  primary: boolean;
  writable: boolean;
}

export interface WatchResult {
  channelId: string;
  resourceId: string | null;
  expiresAt: Date;
}

export interface WrittenEvent {
  externalEventId: string;
  meetingUrl: string | null;
}

export interface CalendarProvider {
  id: ProviderId;
  authorizeUrl(input: { state: string; codeChallenge: string; redirectUri: string }): string;
  exchangeCode(input: { code: string; codeVerifier: string; redirectUri: string }): Promise<TokenSet & { refreshToken: string }>;
  refresh(refreshToken: string): Promise<TokenSet>;
  accountEmail(accessToken: string): Promise<string>;
  listCalendars(accessToken: string): Promise<ProviderCalendar[]>;
  /** Busy intervals in the window, the app's own events and free, declined and cancelled ones left out. */
  listBusy(accessToken: string, calendarId: string, window: Interval, hostTimeZone: string): Promise<Interval[]>;
  upsertEvent(accessToken: string, calendarId: string, input: EventInput): Promise<WrittenEvent>;
  deleteEvent(accessToken: string, calendarId: string, input: { chainId: string; externalEventId: string | null }): Promise<void>;
  watch(accessToken: string, calendarId: string, input: { address: string; token: string; channelId: string }): Promise<WatchResult>;
  stopWatch(accessToken: string, watch: { channelId: string; resourceId: string | null }): Promise<void>;
}

export interface EventInput {
  booking: Booking;
  eventType: EventType;
  host: HostProfile;
}

/** Thrown when the refresh token is dead: revoked consent, password change, expired grant. */
export class ReauthRequired extends Error {}

export interface Connection {
  id: string;
  hostId: string;
  provider: ProviderId;
  accountEmail: string;
  accessTokenEnc: string;
  refreshTokenEnc: string;
  accessExpiresAt: Date;
  status: 'active' | 'needs_reauth';
  checkCalendarIds: string[];
  writeCalendarId: string | null;
  syncedAt: Date | null;
}

export interface Watch {
  id: string;
  connectionId: string;
  calendarId: string;
  channelId: string;
  resourceId: string | null;
  expiresAt: Date;
}

export interface CalendarStore {
  listConnections(hostId: string): Promise<Connection[]>;
  getConnection(id: string): Promise<Connection | null>;
  /** Upsert by (hostId, provider, accountEmail): reconnecting the same account keeps its settings. */
  saveConnection(input: Omit<Connection, 'id' | 'syncedAt' | 'status' | 'checkCalendarIds' | 'writeCalendarId'> & { defaultCalendarId: string | null }): Promise<Connection>;
  /** Compare-and-set on the old expiry, so a slower refresh never overwrites a newer token. */
  updateTokens(id: string, prevExpiresAt: Date, next: { accessTokenEnc: string; refreshTokenEnc: string; accessExpiresAt: Date }): Promise<boolean>;
  markNeedsReauth(id: string, error: string): Promise<void>;
  requestSync(id: string, now: Date): Promise<void>;
  /** Connections that asked for a sync, or have not synced since staleBefore. */
  listDueSync(now: Date, staleBefore: Date, limit: number): Promise<string[]>;
  claimSync(id: string, now: Date, leaseSeconds: number): Promise<Connection | null>;
  finishSync(id: string, now: Date, error: string | null): Promise<void>;
  replaceBusy(connectionId: string, hostId: string, busy: Interval[]): Promise<void>;
  listWatches(connectionId: string): Promise<Watch[]>;
  listWatchesExpiring(before: Date, limit: number): Promise<Watch[]>;
  getWatchByChannel(channelId: string): Promise<Watch | null>;
  saveWatch(w: Omit<Watch, 'id'>): Promise<void>;
  deleteWatch(channelId: string): Promise<void>;
  /** Deletes the connection with its watches and busy blocks, sealed tokens included. */
  deleteConnection(id: string): Promise<void>;
}
```

```ts
// lib/scheduling/calendar/http.ts: one fetch wrapper for both providers: timeout, JSON, typed errors.
export class ProviderHttpError extends Error {
  constructor(
    public readonly status: number,
    public readonly body: string,
    url: string,
  ) {
    super(`${status} from ${new URL(url).hostname}: ${body.slice(0, 300)}`);
  }
}

export async function call<T>(url: string, init: RequestInit & { token?: string; form?: Record<string, string>; json?: unknown } = {}): Promise<T> {
  const headers = new Headers(init.headers);
  if (init.token) headers.set('authorization', `Bearer ${init.token}`);
  let body = init.body;
  if (init.form) {
    headers.set('content-type', 'application/x-www-form-urlencoded');
    body = new URLSearchParams(init.form).toString();
  } else if (init.json !== undefined) {
    headers.set('content-type', 'application/json');
    body = JSON.stringify(init.json);
  }
  const res = await fetch(url, { ...init, headers, body, signal: AbortSignal.timeout(15_000) });
  const text = await res.text();
  if (!res.ok) throw new ProviderHttpError(res.status, text, url);
  return (text ? JSON.parse(text) : {}) as T;
}

export function form(values: Record<string, string | undefined>): Record<string, string> {
  const out: Record<string, string> = {};
  for (const [k, v] of Object.entries(values)) if (v !== undefined) out[k] = v;
  return out;
}

export function requireEnv(name: string): string {
  const v = process.env[name];
  if (!v) throw new Error(`${name} is required`);
  return v;
}
```

```ts
// lib/scheduling/calendar/writer.ts: what the runner needs from calendar sync. calendar/sync.ts implements it.
import type { HostProfile } from '@/lib/scheduling/store';
import type { Booking, EventType } from '@/lib/scheduling/types';

export interface CalendarWriter {
  /** Creates or patches the host's event for the booking's chain. Returns what to store, or null if no calendar. */
  upsert(booking: Booking, eventType: EventType, host: HostProfile): Promise<{ externalEventId: string; meetingUrl: string | null } | null>;
  remove(booking: Booking): Promise<void>;
}

export const noCalendar: CalendarWriter = {
  async upsert() {
    return null;
  },
  async remove() {},
};
```

The two providers are in [calendar-providers.md](calendar-providers.md).

## Connecting

```ts
// lib/scheduling/calendar/oauth.ts: connect a calendar. The state is signed and names the signed-in host;
// the callback refuses a state whose host is not the session's, so nobody can bind their account to yours.
import { createHash, randomBytes } from 'node:crypto';
import { encryptSecret } from '@/lib/scheduling/crypto';
import { signValue, verifySignedValue } from '@/lib/scheduling/tokens';
import { google } from './google';
import { microsoft } from './microsoft';
import type { CalendarProvider, CalendarStore, Connection, ProviderId } from './types';

export const PROVIDERS: Record<ProviderId, CalendarProvider> = { google, microsoft };
export const OAUTH_STATE_TTL_MS = 10 * 60_000;
/** HttpOnly cookie holding the PKCE verifier between start and callback, scoped to the callback path. */
export const PKCE_COOKIE = 'scheduling_pkce';

function secret(): string {
  const s = process.env.SCHEDULING_TOKEN_SECRET;
  if (!s || s.length < 32) throw new Error('SCHEDULING_TOKEN_SECRET must be at least 32 characters');
  return s;
}

export function isProviderId(v: string): v is ProviderId {
  return v === 'google' || v === 'microsoft';
}

export function redirectUri(appUrl: string, provider: ProviderId): string {
  return `${appUrl}/api/scheduling/calendar/${provider}/callback`;
}

export function startConnect(appUrl: string, provider: ProviderId, hostId: string, now: Date): { url: string; verifier: string } {
  const verifier = randomBytes(32).toString('base64url');
  const challenge = createHash('sha256').update(verifier).digest('base64url');
  const nonce = randomBytes(12).toString('base64url');
  const state = signValue(secret(), `${provider}:${hostId}:${nonce}`, new Date(now.getTime() + OAUTH_STATE_TTL_MS));
  return { url: PROVIDERS[provider].authorizeUrl({ state, codeChallenge: challenge, redirectUri: redirectUri(appUrl, provider) }), verifier };
}

export async function finishConnect(input: {
  store: CalendarStore;
  appUrl: string;
  provider: ProviderId;
  sessionHostId: string;
  code: string;
  state: string;
  verifier: string | undefined;
  now: Date;
}): Promise<Connection> {
  const payload = verifySignedValue(secret(), input.state, input.now);
  const [provider, hostId] = payload?.split(':') ?? [];
  if (!payload || provider !== input.provider || hostId !== input.sessionHostId) throw new Error('state does not match this session');
  if (!input.verifier) throw new Error('missing PKCE verifier');
  const p = PROVIDERS[input.provider];
  const t = await p.exchangeCode({ code: input.code, codeVerifier: input.verifier, redirectUri: redirectUri(input.appUrl, input.provider) });
  const [email, calendars] = await Promise.all([p.accountEmail(t.accessToken), p.listCalendars(t.accessToken)]);
  const primary = calendars.find(c => c.primary) ?? calendars[0];
  const connection = await input.store.saveConnection({
    hostId: input.sessionHostId,
    provider: input.provider,
    accountEmail: email,
    accessTokenEnc: encryptSecret(t.accessToken),
    refreshTokenEnc: encryptSecret(t.refreshToken),
    accessExpiresAt: t.expiresAt,
    defaultCalendarId: primary?.id ?? null,
  });
  await input.store.requestSync(connection.id, input.now);
  return connection;
}
```

## Sync, watches and the writer

```ts
// lib/scheduling/calendar/sync.ts: access tokens, the busy mirror, watch channels and the CalendarWriter.
// Busy time is mirrored into busy_blocks, so a slot query never waits on Google or Microsoft and keeps
// working through their outages. A push notification only asks for a sync; the cron runs it under a lease.
import { randomUUID } from 'node:crypto';
import { decryptSecret, encryptSecret } from '@/lib/scheduling/crypto';
import type { HostProfile, SchedulingStore } from '@/lib/scheduling/store';
import { signValue, verifySignedValue } from '@/lib/scheduling/tokens';
import type { Booking, EventType, Interval } from '@/lib/scheduling/types';
import { PROVIDERS } from './oauth';
import type { CalendarWriter } from './writer';
import { ReauthRequired, type CalendarStore, type Connection, type Watch } from './types';

/** How far ahead busy time is mirrored. Keep it at least the longest horizonDays of the host's event types. */
export const SYNC_WINDOW_DAYS = 120;
export const SYNC_LEASE_SECONDS = 120;
/** A connection with no push or a silent channel is still re-read this often by the cron. */
export const STALE_SYNC_MINUTES = 30;
/** Watches are renewed when less than this is left. */
export const WATCH_RENEW_BEFORE_HOURS = 24;
const ACCESS_REFRESH_MARGIN_MS = 5 * 60_000;
const DAY = 86_400_000;

export interface SyncDeps {
  calendars: CalendarStore;
  store: SchedulingStore;
  now: () => Date;
  appUrl: string;
}

function secret(): string {
  const s = process.env.SCHEDULING_TOKEN_SECRET;
  if (!s || s.length < 32) throw new Error('SCHEDULING_TOKEN_SECRET must be at least 32 characters');
  return s;
}

/** A usable access token, refreshed when it expires within five minutes. Dead grants flag the connection. */
export async function accessToken(deps: SyncDeps, c: Connection): Promise<string> {
  if (c.accessExpiresAt.getTime() - deps.now().getTime() > ACCESS_REFRESH_MARGIN_MS) return decryptSecret(c.accessTokenEnc);
  try {
    const t = await PROVIDERS[c.provider].refresh(decryptSecret(c.refreshTokenEnc));
    const next = {
      accessTokenEnc: encryptSecret(t.accessToken),
      refreshTokenEnc: t.refreshToken ? encryptSecret(t.refreshToken) : c.refreshTokenEnc,
      accessExpiresAt: t.expiresAt,
    };
    // Another worker may have refreshed first; its token is as good as ours, so losing the race is fine.
    await deps.calendars.updateTokens(c.id, c.accessExpiresAt, next);
    return t.accessToken;
  } catch (err) {
    if (err instanceof ReauthRequired) await deps.calendars.markNeedsReauth(c.id, 'Calendar access was revoked. Reconnect it.');
    throw err;
  }
}

/**
 * Channel token: signed `${connectionId}:${channelId}`, checked on every notification. About 110 characters:
 * Graph caps clientState at 128, so the calendar id (long on Outlook) stays in the watch row, not here.
 */
export function channelToken(connectionId: string, channelId: string, expiresAt: Date): string {
  return signValue(secret(), `${connectionId}:${channelId}`, expiresAt);
}

export function readChannelToken(token: string, now: Date): { connectionId: string; channelId: string } | null {
  const payload = verifySignedValue(secret(), token, now);
  const [connectionId, channelId, extra] = payload?.split(':') ?? [];
  if (!connectionId || !channelId || extra !== undefined) return null;
  return { connectionId, channelId };
}

/** Re-reads every checked calendar of one connection and replaces its busy blocks in one transaction. */
export async function syncConnection(deps: SyncDeps, connectionId: string): Promise<'synced' | 'busy' | 'skipped'> {
  const now = deps.now();
  const c = await deps.calendars.claimSync(connectionId, now, SYNC_LEASE_SECONDS);
  if (!c) return 'busy';
  if (c.status !== 'active') {
    await deps.calendars.finishSync(c.id, now, 'needs reauth');
    return 'skipped';
  }
  try {
    const host = await deps.store.getHost(c.hostId);
    if (!host) throw new Error('host not found');
    const token = await accessToken(deps, c);
    const window: Interval = { start: new Date(now.getTime() - DAY), end: new Date(now.getTime() + SYNC_WINDOW_DAYS * DAY) };
    const busy: Interval[] = [];
    for (const calendarId of c.checkCalendarIds) {
      busy.push(...(await PROVIDERS[c.provider].listBusy(token, calendarId, window, host.timeZone)));
    }
    await deps.calendars.replaceBusy(c.id, c.hostId, busy);
    await deps.calendars.finishSync(c.id, deps.now(), null);
    return 'synced';
  } catch (err) {
    await deps.calendars.finishSync(c.id, deps.now(), err instanceof Error ? err.message : String(err));
    throw err;
  }
}

/** Opens a watch on every checked calendar that has none. Run after the host changes calendar settings. */
export async function ensureWatches(deps: SyncDeps, connectionId: string): Promise<void> {
  const c = await deps.calendars.getConnection(connectionId);
  if (!c || c.status !== 'active') return;
  const existing = new Set((await deps.calendars.listWatches(c.id)).map(w => w.calendarId));
  const token = await accessToken(deps, c);
  for (const calendarId of c.checkCalendarIds.filter(id => !existing.has(id))) {
    await openWatch(deps, c, token, calendarId);
  }
}

async function openWatch(deps: SyncDeps, c: Connection, token: string, calendarId: string): Promise<Watch> {
  const channelId = randomUUID();
  const expires = new Date(deps.now().getTime() + 8 * DAY);
  const w = await PROVIDERS[c.provider].watch(token, calendarId, {
    address: `${deps.appUrl}/api/scheduling/calendar/${c.provider}/notify`,
    token: channelToken(c.id, channelId, expires),
    channelId,
  });
  const watch = { connectionId: c.id, calendarId, channelId: w.channelId, resourceId: w.resourceId, expiresAt: w.expiresAt };
  await deps.calendars.saveWatch(watch);
  return { id: w.channelId, ...watch };
}

/**
 * Replaces watches close to expiry: open the new one first, then stop the old one, so no change falls between
 * them. Google channels cannot be extended; Microsoft subscriptions could be, but a PATCH keeps the old
 * clientState, whose signature expires, so both providers take the same path. A sync is requested after,
 * because a notification may have been missed.
 */
export async function renewWatches(deps: SyncDeps, limit = 50): Promise<number> {
  const now = deps.now();
  const due = await deps.calendars.listWatchesExpiring(new Date(now.getTime() + WATCH_RENEW_BEFORE_HOURS * 3_600_000), limit);
  let renewed = 0;
  for (const w of due) {
    const c = await deps.calendars.getConnection(w.connectionId);
    if (!c || c.status !== 'active' || !c.checkCalendarIds.includes(w.calendarId)) {
      await deps.calendars.deleteWatch(w.channelId);
      continue;
    }
    try {
      const token = await accessToken(deps, c);
      await openWatch(deps, c, token, w.calendarId);
      await deps.calendars.deleteWatch(w.channelId);
      // Already expired or removed on the provider's side is fine: the new watch is in place.
      await PROVIDERS[c.provider].stopWatch(token, w).catch(() => undefined);
      await deps.calendars.requestSync(c.id, now);
      renewed++;
    } catch (err) {
      if (!(err instanceof ReauthRequired)) throw err;
    }
  }
  return renewed;
}

/** Writes bookings into the host's chosen calendar, on the first active connection that has one. */
export function calendarWriter(deps: SyncDeps): CalendarWriter {
  async function target(hostId: string): Promise<Connection | null> {
    const list = await deps.calendars.listConnections(hostId);
    return list.find(c => c.status === 'active' && c.writeCalendarId) ?? null;
  }
  return {
    async upsert(booking: Booking, eventType: EventType, host: HostProfile) {
      const c = await target(booking.hostId);
      if (!c?.writeCalendarId) return null;
      const token = await accessToken(deps, c);
      return PROVIDERS[c.provider].upsertEvent(token, c.writeCalendarId, { booking, eventType, host });
    },
    async remove(booking: Booking) {
      const c = await target(booking.hostId);
      if (!c?.writeCalendarId) return;
      const token = await accessToken(deps, c);
      await PROVIDERS[c.provider].deleteEvent(token, c.writeCalendarId, { chainId: booking.chainId, externalEventId: booking.externalEventId });
    },
  };
}

/**
 * Disconnects one of the host's calendars: stops its watches at the provider, then deletes the connection,
 * which takes its watches, busy blocks and sealed tokens with it. Returns false when the connection is not
 * this host's, so the route answers the same as for a missing one.
 *
 * Stopping is best effort. A revoked grant has no token to stop with, and must not keep the host from
 * removing the connection. A channel left running expires on its own within days, and its notifications
 * are refused, because its watch row is gone.
 */
export async function disconnectCalendar(deps: SyncDeps, hostId: string, connectionId: string): Promise<boolean> {
  const c = await deps.calendars.getConnection(connectionId);
  if (!c || c.hostId !== hostId) return false;
  const watches = await deps.calendars.listWatches(c.id);
  if (watches.length > 0 && c.status === 'active') {
    try {
      const token = await accessToken(deps, c);
      for (const w of watches) await PROVIDERS[c.provider].stopWatch(token, w).catch(() => undefined);
    } catch {
      // No usable token: the channels expire on their own.
    }
  }
  await deps.calendars.deleteConnection(c.id);
  return true;
}

/** The cron's calendar pass: renew watches, then sync what asked for it or went stale. */
export async function runCalendarCron(deps: SyncDeps, limit = 20): Promise<{ renewed: number; synced: number; failed: number }> {
  const renewed = await renewWatches(deps);
  const now = deps.now();
  const ids = await deps.calendars.listDueSync(now, new Date(now.getTime() - STALE_SYNC_MINUTES * 60_000), limit);
  let synced = 0;
  let failed = 0;
  for (const id of ids) {
    try {
      if ((await syncConnection(deps, id)) === 'synced') synced++;
    } catch {
      failed++;
    }
  }
  return { renewed, synced, failed };
}
```

## Rules

| Rule | Reason |
|---|---|
| The OAuth state is signed and names the signed-in host; the callback refuses any other host | Otherwise anyone can send a host a consent link and bind their own calendar to the host's account. |
| PKCE on both providers | The code is useless without the verifier from the host's own cookie. |
| Tokens sealed with AES-256-GCM before they are stored | A database export does not hand out calendar access. |
| A dead refresh token flags the connection `needs_reauth` and drops its busy blocks | The host sees a reconnect prompt instead of silently stale availability. |
| Token refresh is compare-and-set on the old expiry | Two workers refreshing at once both get a working token; neither overwrites a newer one with an older one. |
| All-day events block the host's local days, end date exclusive | A three-day trip blocks three days, in the zone the slots are computed in. |
| Free, cancelled and declined events do not block; tentative and out-of-office do | What the host marked as free should stay bookable. |
| The app's own events carry a private marker and are left out of busy time | Otherwise a group slot blocks itself after its first seat is written to the calendar. |
| Channel tokens are signed and bound to their channel id | A token copied from another channel is refused. |
| Renewal opens the new watch before stopping the old one | No change falls into a gap between them. Overlap only means a duplicate notice, and the lease absorbs it. |
| Notification routes answer at once and sync after | Graph expects an answer within seconds, and Google retries a slow one. |
| Disconnecting stops the channels first, then deletes the connection, and does not fail on a dead grant | A deleted row with live channels leaves the provider calling for days; a host who cannot remove a revoked connection is stuck with it. |
| Disconnecting does not revoke the grant at the provider | Google's revoke ends the grant for the whole account, so it would also cut off another host who connected the same account. The tokens are deleted, so nothing usable is left. The host can remove the app in their Google or Microsoft account settings. |
| Events already written stay in the host's calendar | The host may still rely on them. Later changes to those bookings no longer reach that calendar. |

## Provider setup

[operations.md](operations.md) has the console steps, redirect URIs, scopes and the verification Google
requires before strangers can connect sensitive calendar scopes.
