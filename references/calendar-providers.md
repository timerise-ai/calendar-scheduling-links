# Calendar providers

Google Calendar and Microsoft Graph behind the `CalendarProvider` contract from
[calendar-sync.md](calendar-sync.md). Each file is the whole provider: consent URL, code exchange, refresh,
account address, calendar list, busy read, event upsert and delete, and push channels. The busy read ends in a
pure function (`googleEventsToBusy`, `graphEventsToBusy`) so the filtering rules have tests without a network.

## Google

```ts
// lib/scheduling/calendar/google.ts: Google Calendar over REST. Env: GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET.
import { zonedToInstant } from '@/lib/scheduling/time';
import type { Interval } from '@/lib/scheduling/types';
import { call, form, ProviderHttpError, requireEnv } from './http';
import { ReauthRequired, type CalendarProvider, type EventInput, type TokenSet, type WrittenEvent } from './types';

const API = 'https://www.googleapis.com/calendar/v3';
/** Both are sensitive scopes: Google's OAuth verification is needed before strangers can connect. */
export const GOOGLE_SCOPES = [
  'openid',
  'email',
  'https://www.googleapis.com/auth/calendar.calendarlist.readonly',
  'https://www.googleapis.com/auth/calendar.events',
];
/** Private extended property on every event the app writes, so the busy read can leave them out. */
export const OWN_EVENT_KEY = 'schedulingChainId';
/** Watch channels: Google caps the TTL per resource; ask for a week and renew a day before expiry. */
export const GOOGLE_WATCH_TTL_SECONDS = 7 * 24 * 3600;

/** Google accepts client-chosen event ids in base32hex (a to v, 0 to 9), 5 to 1024 characters. */
export function googleEventId(chainId: string): string {
  return `sl${chainId.toLowerCase().replace(/[^0-9a-f]/g, '')}`;
}

interface GTime {
  dateTime?: string;
  date?: string;
}

export interface GoogleEvent {
  id: string;
  status?: string;
  transparency?: string;
  eventType?: string;
  start?: GTime;
  end?: GTime;
  attendees?: { self?: boolean; responseStatus?: string }[];
  extendedProperties?: { private?: Record<string, string> };
  hangoutLink?: string;
}

/**
 * Pure: Google events to busy intervals. Left out: cancelled, transparent (shown as free), declined by the
 * calendar owner, working-location markers, and the app's own events. All-day events block the host's whole
 * local days, end date exclusive, so a three-day trip blocks three days.
 */
export function googleEventsToBusy(items: GoogleEvent[], hostTimeZone: string): Interval[] {
  const out: Interval[] = [];
  for (const e of items) {
    if (e.status === 'cancelled' || e.transparency === 'transparent') continue;
    if (e.eventType === 'workingLocation') continue;
    if (e.extendedProperties?.private?.[OWN_EVENT_KEY]) continue;
    if (e.attendees?.some(a => a.self && a.responseStatus === 'declined')) continue;
    const s = e.start;
    const en = e.end;
    if (s?.dateTime && en?.dateTime) {
      out.push({ start: new Date(s.dateTime), end: new Date(en.dateTime) });
    } else if (s?.date && en?.date) {
      out.push({ start: zonedToInstant(s.date, '00:00', hostTimeZone), end: zonedToInstant(en.date, '00:00', hostTimeZone) });
    }
  }
  return out.filter(i => i.start < i.end);
}

function eventBody(input: EventInput): Record<string, unknown> {
  const { booking: b, eventType: et } = input;
  const body: Record<string, unknown> = {
    summary: `${et.title}: ${b.inviteeName}`,
    description: et.questions
      .filter(q => b.answers[q.id])
      .map(q => `${q.label}: ${b.answers[q.id]}`)
      .join('\n'),
    start: { dateTime: b.start.toISOString() },
    end: { dateTime: b.end.toISOString() },
    status: 'confirmed',
    attendees: [{ email: b.inviteeEmail, displayName: b.inviteeName, responseStatus: 'accepted' }],
    extendedProperties: { private: { [OWN_EVENT_KEY]: b.chainId } },
    reminders: { useDefault: true },
  };
  if (et.locationKind === 'google_meet' && !b.meetingUrl) {
    body.conferenceData = { createRequest: { requestId: b.chainId, conferenceSolutionKey: { type: 'hangoutsMeet' } } };
  } else if (et.locationValue) {
    body.location = et.locationValue;
  }
  return body;
}

function tokens(r: { access_token: string; expires_in: number; refresh_token?: string }): TokenSet {
  return { accessToken: r.access_token, refreshToken: r.refresh_token ?? null, expiresAt: new Date(Date.now() + r.expires_in * 1000) };
}

export const google: CalendarProvider = {
  id: 'google',
  authorizeUrl({ state, codeChallenge, redirectUri }) {
    const q = new URLSearchParams({
      client_id: requireEnv('GOOGLE_CLIENT_ID'),
      redirect_uri: redirectUri,
      response_type: 'code',
      scope: GOOGLE_SCOPES.join(' '),
      access_type: 'offline',
      // Without consent Google returns no refresh token on a reconnect.
      prompt: 'consent',
      include_granted_scopes: 'true',
      state,
      code_challenge: codeChallenge,
      code_challenge_method: 'S256',
    });
    return `https://accounts.google.com/o/oauth2/v2/auth?${q}`;
  },
  async exchangeCode({ code, codeVerifier, redirectUri }) {
    const r = await call<{ access_token: string; expires_in: number; refresh_token?: string }>('https://oauth2.googleapis.com/token', {
      method: 'POST',
      form: form({ code, code_verifier: codeVerifier, redirect_uri: redirectUri, grant_type: 'authorization_code', client_id: requireEnv('GOOGLE_CLIENT_ID'), client_secret: requireEnv('GOOGLE_CLIENT_SECRET') }),
    });
    if (!r.refresh_token) throw new Error('Google returned no refresh token');
    return { ...tokens(r), refreshToken: r.refresh_token };
  },
  async refresh(refreshToken) {
    try {
      return tokens(
        await call('https://oauth2.googleapis.com/token', {
          method: 'POST',
          form: form({ refresh_token: refreshToken, grant_type: 'refresh_token', client_id: requireEnv('GOOGLE_CLIENT_ID'), client_secret: requireEnv('GOOGLE_CLIENT_SECRET') }),
        }),
      );
    } catch (err) {
      if (err instanceof ProviderHttpError && err.status === 400 && err.body.includes('invalid_grant')) throw new ReauthRequired(err.body);
      throw err;
    }
  },
  async accountEmail(token) {
    const r = await call<{ email?: string }>('https://openidconnect.googleapis.com/v1/userinfo', { token });
    if (!r.email) throw new Error('Google returned no email');
    return r.email.toLowerCase();
  },
  async listCalendars(token) {
    const r = await call<{ items?: { id: string; summary?: string; primary?: boolean; accessRole?: string }[] }>(`${API}/users/me/calendarList?minAccessRole=freeBusyReader`, { token });
    return (r.items ?? []).map(c => ({ id: c.id, name: c.summary ?? c.id, primary: c.primary === true, writable: c.accessRole === 'owner' || c.accessRole === 'writer' }));
  },
  async listBusy(token, calendarId, window, hostTimeZone) {
    const items: GoogleEvent[] = [];
    let pageToken: string | undefined;
    do {
      const q = new URLSearchParams({
        timeMin: window.start.toISOString(),
        timeMax: window.end.toISOString(),
        singleEvents: 'true',
        maxResults: '2500',
        fields: 'nextPageToken,items(id,status,transparency,eventType,start,end,attendees(self,responseStatus),extendedProperties)',
      });
      if (pageToken) q.set('pageToken', pageToken);
      const r = await call<{ items?: GoogleEvent[]; nextPageToken?: string }>(`${API}/calendars/${encodeURIComponent(calendarId)}/events?${q}`, { token });
      items.push(...(r.items ?? []));
      pageToken = r.nextPageToken;
    } while (pageToken);
    return googleEventsToBusy(items, hostTimeZone);
  },
  async upsertEvent(token, calendarId, input): Promise<WrittenEvent> {
    const id = googleEventId(input.booking.chainId);
    const base = `${API}/calendars/${encodeURIComponent(calendarId)}/events`;
    // sendUpdates=none: the app sends its own e-mail with the ICS; Google's would be a second invitation.
    const q = 'conferenceDataVersion=1&sendUpdates=none';
    let e: GoogleEvent;
    try {
      e = await call<GoogleEvent>(`${base}?${q}`, { method: 'POST', token, json: { id, ...eventBody(input) } });
    } catch (err) {
      // 409: this chain's event exists (a retry, or a reschedule). Patch it, which also revives a deleted one.
      if (!(err instanceof ProviderHttpError && err.status === 409)) throw err;
      e = await call<GoogleEvent>(`${base}/${id}?${q}`, { method: 'PATCH', token, json: eventBody(input) });
    }
    return { externalEventId: e.id, meetingUrl: e.hangoutLink ?? input.booking.meetingUrl };
  },
  async deleteEvent(token, calendarId, { chainId }) {
    try {
      await call(`${API}/calendars/${encodeURIComponent(calendarId)}/events/${googleEventId(chainId)}?sendUpdates=none`, { method: 'DELETE', token });
    } catch (err) {
      if (err instanceof ProviderHttpError && (err.status === 404 || err.status === 410)) return;
      throw err;
    }
  },
  async watch(token, calendarId, { address, token: channelToken, channelId }) {
    const r = await call<{ resourceId: string; expiration: string }>(`${API}/calendars/${encodeURIComponent(calendarId)}/events/watch`, {
      method: 'POST',
      token,
      json: { id: channelId, type: 'web_hook', address, token: channelToken, params: { ttl: String(GOOGLE_WATCH_TTL_SECONDS) } },
    });
    return { channelId, resourceId: r.resourceId, expiresAt: new Date(Number(r.expiration)) };
  },
  async stopWatch(token, { channelId, resourceId }) {
    try {
      await call(`${API}/channels/stop`, { method: 'POST', token, json: { id: channelId, resourceId } });
    } catch (err) {
      if (err instanceof ProviderHttpError && err.status === 404) return;
      throw err;
    }
  },
};
```

| Fact | Verified against |
|---|---|
| Client-chosen event ids use base32hex characters (a to v, 0 to 9), 5 to 1024 long, unique per calendar | Google Calendar API, events.insert reference |
| `sendUpdates=none` sends no Google invitation; the app's own e-mail carries the ICS instead | events.insert reference |
| Conference changes persist only with `conferenceDataVersion=1` | events.insert reference |
| `events.list` pages with `nextPageToken`; read every page | events.list reference |

## Microsoft

```ts
// lib/scheduling/calendar/microsoft.ts: Outlook and Microsoft 365 over Microsoft Graph.
// Env: MICROSOFT_CLIENT_ID, MICROSOFT_CLIENT_SECRET, optional MICROSOFT_TENANT (default 'common', which
// admits work, school and personal accounts). Plain OAuth endpoints, no MSAL: the refresh token is in the
// token response and is stored per connection, never in a process-wide cache.
import { zonedToInstant } from '@/lib/scheduling/time';
import type { Interval } from '@/lib/scheduling/types';
import { call, form, ProviderHttpError, requireEnv } from './http';
import { ReauthRequired, type CalendarProvider, type EventInput, type TokenSet, type WrittenEvent } from './types';

const GRAPH = 'https://graph.microsoft.com/v1.0';
export const MICROSOFT_SCOPES = ['openid', 'email', 'offline_access', 'User.Read', 'Calendars.ReadWrite'];
/** A fixed namespace GUID for the app's single-value extended property; never change it once deployed. */
export const OWN_EVENT_PROPERTY = 'String {6f1c2a8e-3b4d-4e5f-9a1b-2c3d4e5f6a7b} Name schedulingChainId';
/** Subscription lifetime asked for; sync.ts replaces it a day before it runs out. */
export const GRAPH_SUBSCRIPTION_DAYS = 3;

const tenant = () => process.env.MICROSOFT_TENANT ?? 'common';
const authority = () => `https://login.microsoftonline.com/${tenant()}/oauth2/v2.0`;

interface GTime {
  dateTime: string;
  timeZone?: string;
}

export interface GraphEvent {
  id: string;
  isAllDay?: boolean;
  isCancelled?: boolean;
  showAs?: string;
  start?: GTime;
  end?: GTime;
  responseStatus?: { response?: string };
  singleValueExtendedProperties?: { id: string; value: string }[];
  onlineMeeting?: { joinUrl?: string } | null;
}

/**
 * Pure: Graph events (requested with Prefer outlook.timezone="UTC") to busy intervals. Left out: cancelled,
 * free and working-elsewhere, declined, and the app's own events. Tentative blocks. All-day events block
 * the host's local days, end date exclusive.
 */
export function graphEventsToBusy(items: GraphEvent[], hostTimeZone: string): Interval[] {
  const out: Interval[] = [];
  for (const e of items) {
    if (e.isCancelled || e.showAs === 'free' || e.showAs === 'workingElsewhere') continue;
    if (e.responseStatus?.response === 'declined') continue;
    if (e.singleValueExtendedProperties?.some(p => p.value)) continue;
    if (!e.start || !e.end) continue;
    if (e.isAllDay) {
      out.push({
        start: zonedToInstant(e.start.dateTime.slice(0, 10), '00:00', hostTimeZone),
        end: zonedToInstant(e.end.dateTime.slice(0, 10), '00:00', hostTimeZone),
      });
    } else {
      // With the UTC preference Graph returns '2026-10-15T09:00:00.0000000' and no offset.
      out.push({ start: new Date(`${e.start.dateTime.slice(0, 19)}Z`), end: new Date(`${e.end.dateTime.slice(0, 19)}Z`) });
    }
  }
  return out.filter(i => i.start < i.end);
}

function tokens(r: { access_token: string; expires_in: number; refresh_token?: string }): TokenSet {
  return { accessToken: r.access_token, refreshToken: r.refresh_token ?? null, expiresAt: new Date(Date.now() + r.expires_in * 1000) };
}

function eventBody(input: EventInput): Record<string, unknown> {
  const { booking: b, eventType: et } = input;
  const body: Record<string, unknown> = {
    subject: `${et.title}: ${b.inviteeName}`,
    body: {
      contentType: 'text',
      content: et.questions
        .filter(q => b.answers[q.id])
        .map(q => `${q.label}: ${b.answers[q.id]}`)
        .join('\n'),
    },
    start: { dateTime: b.start.toISOString().slice(0, 19), timeZone: 'UTC' },
    end: { dateTime: b.end.toISOString().slice(0, 19), timeZone: 'UTC' },
    attendees: [{ emailAddress: { address: b.inviteeEmail, name: b.inviteeName }, type: 'required' }],
    singleValueExtendedProperties: [{ id: OWN_EVENT_PROPERTY, value: b.chainId }],
  };
  if (et.locationKind === 'teams') {
    body.isOnlineMeeting = true;
    body.onlineMeetingProvider = 'teamsForBusiness';
  } else if (et.locationValue) {
    body.location = { displayName: et.locationValue };
  }
  return body;
}

async function findOwnEvent(token: string, calendarId: string, chainId: string): Promise<string | null> {
  const filter = `singleValueExtendedProperties/Any(ep: ep/id eq '${OWN_EVENT_PROPERTY}' and ep/value eq '${chainId.replace(/'/g, "''")}')`;
  const r = await call<{ value?: { id: string }[] }>(`${GRAPH}/me/calendars/${encodeURIComponent(calendarId)}/events?$filter=${encodeURIComponent(filter)}&$select=id&$top=1`, { token });
  return r.value?.[0]?.id ?? null;
}

export const microsoft: CalendarProvider = {
  id: 'microsoft',
  authorizeUrl({ state, codeChallenge, redirectUri }) {
    const q = new URLSearchParams({
      client_id: requireEnv('MICROSOFT_CLIENT_ID'),
      redirect_uri: redirectUri,
      response_type: 'code',
      response_mode: 'query',
      scope: MICROSOFT_SCOPES.join(' '),
      prompt: 'select_account',
      state,
      code_challenge: codeChallenge,
      code_challenge_method: 'S256',
    });
    return `${authority()}/authorize?${q}`;
  },
  async exchangeCode({ code, codeVerifier, redirectUri }) {
    const r = await call<{ access_token: string; expires_in: number; refresh_token?: string }>(`${authority()}/token`, {
      method: 'POST',
      form: form({ client_id: requireEnv('MICROSOFT_CLIENT_ID'), client_secret: requireEnv('MICROSOFT_CLIENT_SECRET'), code, code_verifier: codeVerifier, redirect_uri: redirectUri, grant_type: 'authorization_code', scope: MICROSOFT_SCOPES.join(' ') }),
    });
    if (!r.refresh_token) throw new Error('Microsoft returned no refresh token: is offline_access granted?');
    return { ...tokens(r), refreshToken: r.refresh_token };
  },
  async refresh(refreshToken) {
    try {
      return tokens(
        await call(`${authority()}/token`, {
          method: 'POST',
          form: form({ client_id: requireEnv('MICROSOFT_CLIENT_ID'), client_secret: requireEnv('MICROSOFT_CLIENT_SECRET'), refresh_token: refreshToken, grant_type: 'refresh_token', scope: MICROSOFT_SCOPES.join(' ') }),
        }),
      );
    } catch (err) {
      if (err instanceof ProviderHttpError && err.status === 400 && err.body.includes('invalid_grant')) throw new ReauthRequired(err.body);
      throw err;
    }
  },
  async accountEmail(token) {
    const r = await call<{ mail?: string | null; userPrincipalName?: string }>(`${GRAPH}/me?$select=mail,userPrincipalName`, { token });
    const email = r.mail ?? r.userPrincipalName;
    if (!email) throw new Error('Microsoft returned no address');
    return email.toLowerCase();
  },
  async listCalendars(token) {
    const r = await call<{ value?: { id: string; name: string; canEdit?: boolean; isDefaultCalendar?: boolean }[] }>(`${GRAPH}/me/calendars?$select=id,name,canEdit,isDefaultCalendar&$top=100`, { token });
    return (r.value ?? []).map(c => ({ id: c.id, name: c.name, primary: c.isDefaultCalendar === true, writable: c.canEdit === true }));
  },
  async listBusy(token, calendarId, window, hostTimeZone) {
    const items: GraphEvent[] = [];
    const q = new URLSearchParams({
      startDateTime: window.start.toISOString(),
      endDateTime: window.end.toISOString(),
      $select: 'id,isAllDay,isCancelled,showAs,start,end,responseStatus',
      $expand: `singleValueExtendedProperties($filter=id eq '${OWN_EVENT_PROPERTY}')`,
      $top: '500',
    });
    let url: string | undefined = `${GRAPH}/me/calendars/${encodeURIComponent(calendarId)}/calendarView?${q}`;
    while (url) {
      const r: { value?: GraphEvent[]; '@odata.nextLink'?: string } = await call(url, { token, headers: { prefer: 'outlook.timezone="UTC"' } });
      items.push(...(r.value ?? []));
      url = r['@odata.nextLink'];
    }
    return graphEventsToBusy(items, hostTimeZone);
  },
  async upsertEvent(token, calendarId, input): Promise<WrittenEvent> {
    const known = input.booking.externalEventId ?? (await findOwnEvent(token, calendarId, input.booking.chainId));
    let e: GraphEvent;
    if (known) {
      try {
        e = await call<GraphEvent>(`${GRAPH}/me/events/${encodeURIComponent(known)}`, { method: 'PATCH', token, json: eventBody(input) });
        return { externalEventId: e.id, meetingUrl: e.onlineMeeting?.joinUrl ?? input.booking.meetingUrl };
      } catch (err) {
        if (!(err instanceof ProviderHttpError && err.status === 404)) throw err;
      }
    }
    // transactionId makes a retried create return the event the first attempt made.
    e = await call<GraphEvent>(`${GRAPH}/me/calendars/${encodeURIComponent(calendarId)}/events`, {
      method: 'POST',
      token,
      json: { ...eventBody(input), transactionId: input.booking.chainId },
    });
    return { externalEventId: e.id, meetingUrl: e.onlineMeeting?.joinUrl ?? input.booking.meetingUrl };
  },
  async deleteEvent(token, calendarId, { chainId, externalEventId }) {
    const id = externalEventId ?? (await findOwnEvent(token, calendarId, chainId));
    if (!id) return;
    try {
      await call(`${GRAPH}/me/events/${encodeURIComponent(id)}`, { method: 'DELETE', token });
    } catch (err) {
      if (err instanceof ProviderHttpError && err.status === 404) return;
      throw err;
    }
  },
  async watch(token, calendarId, { address, token: clientState }) {
    const r = await call<{ id: string; expirationDateTime: string }>(`${GRAPH}/subscriptions`, {
      method: 'POST',
      token,
      json: {
        changeType: 'created,updated,deleted',
        notificationUrl: address,
        resource: `/me/calendars/${calendarId}/events`,
        expirationDateTime: new Date(Date.now() + GRAPH_SUBSCRIPTION_DAYS * 86_400_000).toISOString(),
        clientState,
      },
    });
    return { channelId: r.id, resourceId: null, expiresAt: new Date(r.expirationDateTime) };
  },
  async stopWatch(token, { channelId }) {
    try {
      await call(`${GRAPH}/subscriptions/${encodeURIComponent(channelId)}`, { method: 'DELETE', token });
    } catch (err) {
      if (err instanceof ProviderHttpError && err.status === 404) return;
      throw err;
    }
  },
};
```

| Fact | Verified against |
|---|---|
| `clientState` is at most 128 characters | Graph subscription resource |
| Outlook event subscriptions last at most 10,080 minutes | Graph subscription resource, subscription lifetime table |
| `transactionId` lets a retried create return the first attempt's event | Graph event resource |
| `showAs` is one of free, tentative, busy, oof, workingElsewhere, unknown | Graph event resource |
| Times come back in UTC without an offset when asked with `Prefer: outlook.timezone="UTC"` | Graph calendarView |
