# Supabase stores

`SchedulingStore` and `CalendarStore` on the tables and functions from [supabase.md](supabase.md). Rows are
mapped by hand rather than generated, so the column names in the migration and here must move together.
The atomic writes are one `rpc` call each; a refusal comes back as a reason, anything else throws.

## Booking store

```ts
// lib/scheduling/supabase-store.ts: SchedulingStore on Supabase Postgres, server-only, service role.
// The atomic writes are SQL functions in the migration; this file maps rows and reasons.
import { createClient, type SupabaseClient } from '@supabase/supabase-js';
import { decryptSecret } from '@/lib/scheduling/crypto';
import type { JobKind, JobRecord } from '@/lib/scheduling/jobs';
import type { BookResult, ChangeResult, HostProfile, NewBooking, SchedulingStore, WebhookEndpoint } from '@/lib/scheduling/store';
import type { BookedInterval, Booking, EventType, Interval, Schedule } from '@/lib/scheduling/types';

type Row = Record<string, unknown>;
const str = (v: unknown): string => (typeof v === 'string' ? v : String(v ?? ''));
const strOrNull = (v: unknown): string | null => (typeof v === 'string' ? v : null);
const num = (v: unknown): number => Number(v);
const date = (v: unknown): Date => new Date(str(v));

export function supabaseAdmin(): SupabaseClient {
  const url = process.env.SUPABASE_URL ?? process.env.NEXT_PUBLIC_SUPABASE_URL;
  const key = process.env.SUPABASE_SERVICE_ROLE_KEY;
  if (!url || !key) throw new Error('SUPABASE_URL and SUPABASE_SERVICE_ROLE_KEY are required');
  return createClient(url, key, { auth: { persistSession: false, autoRefreshToken: false } });
}

function toHost(r: Row): HostProfile {
  return { id: str(r.id), handle: str(r.handle), name: str(r.name), email: str(r.email), timeZone: str(r.time_zone) };
}

function toEventType(r: Row): EventType {
  return {
    id: str(r.id),
    hostId: str(r.host_id),
    scheduleId: str(r.schedule_id),
    slug: str(r.slug),
    title: str(r.title),
    description: str(r.description),
    kind: r.kind === 'group' ? 'group' : 'one_on_one',
    capacity: num(r.capacity),
    durationMinutes: num(r.duration_minutes),
    slotIntervalMinutes: num(r.slot_interval_minutes),
    bufferBeforeMinutes: num(r.buffer_before_minutes),
    bufferAfterMinutes: num(r.buffer_after_minutes),
    minNoticeMinutes: num(r.min_notice_minutes),
    horizonDays: num(r.horizon_days),
    dailyLimit: r.daily_limit === null ? null : num(r.daily_limit),
    locationKind: str(r.location_kind) as EventType['locationKind'],
    locationValue: strOrNull(r.location_value),
    questions: (r.questions ?? []) as EventType['questions'],
    active: r.active === true,
  };
}

function toSchedule(r: Row): Schedule {
  return {
    id: str(r.id),
    hostId: str(r.host_id),
    name: str(r.name),
    timeZone: str(r.time_zone),
    rules: (r.rules ?? []) as Schedule['rules'],
    overrides: (r.overrides ?? []) as Schedule['overrides'],
  };
}

function toBooking(r: Row): Booking {
  return {
    id: str(r.id),
    chainId: str(r.chain_id),
    hostId: str(r.host_id),
    eventTypeId: str(r.event_type_id),
    slotKey: str(r.slot_key),
    start: date(r.start_at),
    end: date(r.end_at),
    blockedStart: date(r.blocked_start),
    blockedEnd: date(r.blocked_end),
    status: r.status === 'cancelled' ? 'cancelled' : 'confirmed',
    inviteeName: str(r.invitee_name),
    inviteeEmail: str(r.invitee_email),
    inviteeTimeZone: str(r.invitee_time_zone),
    answers: (r.answers ?? {}) as Record<string, string>,
    manageTokenHash: str(r.manage_token_hash),
    sequence: num(r.sequence),
    rescheduledFromId: strOrNull(r.rescheduled_from_id),
    cancelReason: strOrNull(r.cancel_reason),
    externalEventId: strOrNull(r.external_event_id),
    meetingUrl: strOrNull(r.meeting_url),
    createdAt: date(r.created_at),
    updatedAt: date(r.updated_at),
  };
}

/** Field names in the JSON the SQL functions read; dates as ISO strings. */
function bookingJson(b: NewBooking['booking']): Row {
  return { ...b, start: b.start.toISOString(), end: b.end.toISOString(), blockedStart: b.blockedStart.toISOString(), blockedEnd: b.blockedEnd.toISOString() };
}

function rpcArgs(input: NewBooking): Row {
  return {
    b: bookingJson(input.booking),
    capacity: input.capacity,
    daily_limit: input.dailyLimit,
    day_start: input.dayStart.toISOString(),
    day_end: input.dayEnd.toISOString(),
    jobs: input.jobs.map(j => ({ ...j, runAt: j.runAt.toISOString() })),
  };
}

const REFUSALS = new Set(['slot_taken', 'slot_full', 'day_full', 'not_found', 'not_confirmed']);

function must<T>(res: { data: T; error: { message: string } | null }): T {
  if (res.error) throw new Error(res.error.message);
  return res.data;
}

export function supabaseStore(db: SupabaseClient = supabaseAdmin()): SchedulingStore {
  const one = async (table: string, column: string, value: string): Promise<Row | null> =>
    must(await db.from(table).select('*').eq(column, value).maybeSingle()) as Row | null;

  const loadBooking = async (id: string): Promise<Booking> => {
    const row = await one('bookings', 'id', id);
    if (!row) throw new Error(`booking ${id} vanished after write`);
    return toBooking(row);
  };

  return {
    async getHostByHandle(handle) {
      const r = await one('scheduling_hosts', 'handle', handle.toLowerCase());
      return r ? toHost(r) : null;
    },
    async getHost(id) {
      const r = await one('scheduling_hosts', 'id', id);
      return r ? toHost(r) : null;
    },
    async getEventTypeBySlug(hostId, slug) {
      const r = must(await db.from('event_types').select('*').eq('host_id', hostId).eq('slug', slug).maybeSingle()) as Row | null;
      return r ? toEventType(r) : null;
    },
    async getEventType(id) {
      const r = await one('event_types', 'id', id);
      return r ? toEventType(r) : null;
    },
    async getSchedule(id) {
      const r = await one('schedules', 'id', id);
      return r ? toSchedule(r) : null;
    },
    async listBookedIntervals(hostId, w: Interval): Promise<BookedInterval[]> {
      const rows = must(
        await db
          .from('bookings')
          .select('id, event_type_id, slot_key, start_at, blocked_start, blocked_end')
          .eq('host_id', hostId)
          .eq('status', 'confirmed')
          .lt('blocked_start', w.end.toISOString())
          .gt('blocked_end', w.start.toISOString()),
      ) as Row[];
      return rows.map(r => ({
        id: str(r.id),
        eventTypeId: str(r.event_type_id),
        slotKey: str(r.slot_key),
        start: date(r.start_at),
        blockedStart: date(r.blocked_start),
        blockedEnd: date(r.blocked_end),
      }));
    },
    async listBusy(hostId, w) {
      const rows = must(
        await db.from('busy_blocks').select('start_at, end_at').eq('host_id', hostId).lt('start_at', w.end.toISOString()).gt('end_at', w.start.toISOString()),
      ) as Row[];
      return rows.map(r => ({ start: date(r.start_at), end: date(r.end_at) }));
    },
    async getBooking(id) {
      const r = await one('bookings', 'id', id);
      return r ? toBooking(r) : null;
    },
    async getBookingByTokenHash(hash) {
      const r = await one('bookings', 'manage_token_hash', hash);
      return r ? toBooking(r) : null;
    },
    async listWebhookEndpointIds(hostId, event) {
      const rows = must(await db.from('webhook_endpoints').select('id').eq('host_id', hostId).eq('active', true).contains('events', [event])) as Row[];
      return rows.map(r => str(r.id));
    },
    async getWebhookEndpoint(id): Promise<WebhookEndpoint | null> {
      const r = await one('webhook_endpoints', 'id', id);
      if (!r) return null;
      return { id: str(r.id), hostId: str(r.host_id), url: str(r.url), secret: decryptSecret(str(r.secret_enc)), events: (r.events ?? []) as string[], active: r.active === true };
    },
    async book(input): Promise<BookResult> {
      const outcome = str(must(await db.rpc('csl_book', rpcArgs(input))));
      if (outcome === 'ok') return { ok: true, booking: await loadBooking(input.booking.id) };
      if (outcome === 'slot_taken' || outcome === 'slot_full' || outcome === 'day_full') return { ok: false, reason: outcome };
      throw new Error(`csl_book: ${outcome}`);
    },
    async reschedule(oldId, input): Promise<ChangeResult> {
      const res = await db.rpc('csl_reschedule', { old_id: oldId, ...rpcArgs(input) });
      // A refused insert raises inside the function so the cancel of the old booking rolls back with it.
      const outcome = res.error ? res.error.message : str(res.data);
      if (outcome === 'ok') return { ok: true, booking: await loadBooking(input.booking.id) };
      if (REFUSALS.has(outcome)) return { ok: false, reason: outcome as Exclude<ChangeResult, { ok: true }>['reason'] };
      throw new Error(`csl_reschedule: ${outcome}`);
    },
    async cancel(id, reason, jobs): Promise<ChangeResult> {
      const outcome = str(must(await db.rpc('csl_cancel', { booking: id, reason, jobs: jobs.map(j => ({ ...j, runAt: j.runAt.toISOString() })) })));
      if (outcome === 'ok') return { ok: true, booking: await loadBooking(id) };
      if (outcome === 'not_found' || outcome === 'not_confirmed') return { ok: false, reason: outcome };
      throw new Error(`csl_cancel: ${outcome}`);
    },
    async setBookingExternal(id, fields) {
      const patch: Row = { updated_at: new Date().toISOString() };
      if (fields.externalEventId !== undefined) patch.external_event_id = fields.externalEventId;
      if (fields.meetingUrl !== undefined) patch.meeting_url = fields.meetingUrl;
      must(await db.from('bookings').update(patch).eq('id', id));
    },
    async claimJobs(limit, now, leaseSeconds): Promise<JobRecord[]> {
      const rows = must(await db.rpc('csl_claim_jobs', { max_jobs: limit, now_at: now.toISOString(), lease_seconds: leaseSeconds })) as Row[];
      return rows.map(r => ({ id: str(r.id), kind: str(r.kind) as JobKind, payload: r.payload, runAt: date(r.run_at), attempts: num(r.attempts), dedupeKey: str(r.dedupe_key) }));
    },
    async completeJob(id) {
      must(await db.from('jobs').update({ done_at: new Date().toISOString(), lease_until: null, last_error: null }).eq('id', id));
    },
    async failJob(id, error, at) {
      // No retry left: done_at is set with the error kept, so the row stays as a record and leaves the queue.
      const patch = at
        ? { run_at: at.toISOString(), lease_until: null, last_error: error.slice(0, 2000) }
        : { done_at: new Date().toISOString(), lease_until: null, last_error: `gave up: ${error}`.slice(0, 2000) };
      must(await db.from('jobs').update(patch).eq('id', id));
    },
  };
}
```

## Calendar store

```ts
// lib/scheduling/calendar/supabase-calendar-store.ts: CalendarStore on the same Supabase tables.
import type { SupabaseClient } from '@supabase/supabase-js';
import { supabaseAdmin } from '@/lib/scheduling/supabase-store';
import type { Interval } from '@/lib/scheduling/types';
import type { CalendarStore, Connection, Watch } from './types';

type Row = Record<string, unknown>;

function must<T>(res: { data: T; error: { message: string } | null }): T {
  if (res.error) throw new Error(res.error.message);
  return res.data;
}

function toConnection(r: Row): Connection {
  return {
    id: String(r.id),
    hostId: String(r.host_id),
    provider: r.provider === 'microsoft' ? 'microsoft' : 'google',
    accountEmail: String(r.account_email),
    accessTokenEnc: String(r.access_token_enc),
    refreshTokenEnc: String(r.refresh_token_enc),
    accessExpiresAt: new Date(String(r.access_expires_at)),
    status: r.status === 'needs_reauth' ? 'needs_reauth' : 'active',
    checkCalendarIds: (r.check_calendar_ids ?? []) as string[],
    writeCalendarId: typeof r.write_calendar_id === 'string' ? r.write_calendar_id : null,
    syncedAt: typeof r.synced_at === 'string' ? new Date(r.synced_at) : null,
  };
}

function toWatch(r: Row): Watch {
  return {
    id: String(r.id),
    connectionId: String(r.connection_id),
    calendarId: String(r.calendar_id),
    channelId: String(r.channel_id),
    resourceId: typeof r.resource_id === 'string' ? r.resource_id : null,
    expiresAt: new Date(String(r.expires_at)),
  };
}

export function supabaseCalendarStore(db: SupabaseClient = supabaseAdmin()): CalendarStore {
  return {
    async listConnections(hostId) {
      return (must(await db.from('calendar_connections').select('*').eq('host_id', hostId).order('id')) as Row[]).map(toConnection);
    },
    async getConnection(id) {
      const r = must(await db.from('calendar_connections').select('*').eq('id', id).maybeSingle()) as Row | null;
      return r ? toConnection(r) : null;
    },
    async saveConnection(input) {
      const existing = must(
        await db.from('calendar_connections').select('id').eq('host_id', input.hostId).eq('provider', input.provider).eq('account_email', input.accountEmail).maybeSingle(),
      ) as Row | null;
      const tokens = {
        access_token_enc: input.accessTokenEnc,
        refresh_token_enc: input.refreshTokenEnc,
        access_expires_at: input.accessExpiresAt.toISOString(),
        status: 'active',
        last_error: null,
      };
      const row = existing
        ? must(await db.from('calendar_connections').update(tokens).eq('id', String(existing.id)).select('*').single())
        : must(
            await db
              .from('calendar_connections')
              .insert({
                host_id: input.hostId,
                provider: input.provider,
                account_email: input.accountEmail,
                ...tokens,
                // A new connection checks and writes its primary calendar until the host changes it.
                check_calendar_ids: input.defaultCalendarId ? [input.defaultCalendarId] : [],
                write_calendar_id: input.defaultCalendarId,
              })
              .select('*')
              .single(),
          );
      return toConnection(row as Row);
    },
    async updateTokens(id, prev, next) {
      const rows = must(
        await db
          .from('calendar_connections')
          .update({ access_token_enc: next.accessTokenEnc, refresh_token_enc: next.refreshTokenEnc, access_expires_at: next.accessExpiresAt.toISOString() })
          .eq('id', id)
          .eq('access_expires_at', prev.toISOString())
          .select('id'),
      ) as Row[];
      return rows.length === 1;
    },
    async markNeedsReauth(id, error) {
      must(await db.from('calendar_connections').update({ status: 'needs_reauth', last_error: error }).eq('id', id));
      // Busy time from a dead connection is stale; dropping it is safer than blocking on old data forever.
      must(await db.from('busy_blocks').delete().eq('connection_id', id));
    },
    async requestSync(id, now) {
      must(await db.from('calendar_connections').update({ sync_requested_at: now.toISOString() }).eq('id', id));
    },
    async listDueSync(now, staleBefore, limit) {
      const rows = must(await db.rpc('csl_due_calendar_syncs', { now_at: now.toISOString(), stale_before: staleBefore.toISOString(), max_rows: limit })) as Row[];
      return rows.map(r => String(r.id));
    },
    async claimSync(id, now, leaseSeconds) {
      const rows = must(
        await db
          .from('calendar_connections')
          .update({ sync_lease_until: new Date(now.getTime() + leaseSeconds * 1000).toISOString(), sync_requested_at: null })
          .eq('id', id)
          .or(`sync_lease_until.is.null,sync_lease_until.lt.${now.toISOString()}`)
          .select('*'),
      ) as Row[];
      return rows[0] ? toConnection(rows[0]) : null;
    },
    async finishSync(id, now, error) {
      must(
        await db
          .from('calendar_connections')
          .update({ sync_lease_until: null, last_error: error, ...(error ? {} : { synced_at: now.toISOString() }) })
          .eq('id', id),
      );
    },
    async replaceBusy(connectionId, hostId, busy: Interval[]) {
      must(
        await db.rpc('csl_replace_busy', {
          connection: connectionId,
          host: hostId,
          blocks: busy.map(b => ({ start: b.start.toISOString(), end: b.end.toISOString() })),
        }),
      );
    },
    async listWatches(connectionId) {
      return (must(await db.from('calendar_watches').select('*').eq('connection_id', connectionId)) as Row[]).map(toWatch);
    },
    async listWatchesExpiring(before, limit) {
      return (must(await db.from('calendar_watches').select('*').lt('expires_at', before.toISOString()).order('expires_at').limit(limit)) as Row[]).map(toWatch);
    },
    async getWatchByChannel(channelId) {
      const r = must(await db.from('calendar_watches').select('*').eq('channel_id', channelId).maybeSingle()) as Row | null;
      return r ? toWatch(r) : null;
    },
    async saveWatch(w) {
      must(
        await db.from('calendar_watches').upsert(
          { connection_id: w.connectionId, calendar_id: w.calendarId, channel_id: w.channelId, resource_id: w.resourceId, expires_at: w.expiresAt.toISOString() },
          { onConflict: 'channel_id' },
        ),
      );
    },
    async deleteWatch(channelId) {
      must(await db.from('calendar_watches').delete().eq('channel_id', channelId));
    },
    async deleteConnection(id) {
      // Watches and busy blocks reference the connection with on delete cascade, so one statement removes all.
      must(await db.from('calendar_connections').delete().eq('id', id));
    },
  };
}
```
