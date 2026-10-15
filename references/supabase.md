# Supabase

The default store: Postgres on Supabase, reached only from the server with the service role. The atomic
writes are SQL functions; TypeScript maps rows and refusal reasons.

## Migration

Every table is server-only: row level security on with no policies, `anon` and `authenticated` revoked, and
`service_role` granted explicitly, because new tables in `public` get no grants by default, `service_role`
included. The functions are `security definer` with a pinned `search_path` and are executable by
`service_role` alone.

The constraint that stops double booking is `bookings_no_overlap`: two confirmed bookings of one host may
overlap only when they are seats of the same slot. `<>` on text inside a GiST exclusion constraint needs the
`btree_gist` extension.

```sql
-- supabase/migrations/20261015000000_scheduling.sql: scheduling links. Every table is server-only:
-- RLS on with no policies, anon and authenticated revoked, service_role granted explicitly.
create extension if not exists btree_gist;

create table public.scheduling_hosts (
  id text primary key,                       -- the host app's user id
  handle text not null unique check (handle ~ '^[a-z0-9][a-z0-9-]{1,38}$'),
  name text not null,
  email text not null,
  time_zone text not null,
  created_at timestamptz not null default now()
);

create table public.schedules (
  id uuid primary key default gen_random_uuid(),
  host_id text not null references public.scheduling_hosts(id) on delete cascade,
  name text not null,
  time_zone text not null,
  rules jsonb not null default '[]',         -- AvailabilityRule[], validated by the app
  overrides jsonb not null default '[]',     -- DateOverride[]
  updated_at timestamptz not null default now()
);

create table public.event_types (
  id uuid primary key default gen_random_uuid(),
  host_id text not null references public.scheduling_hosts(id) on delete cascade,
  schedule_id uuid not null references public.schedules(id),
  slug text not null check (slug ~ '^[a-z0-9][a-z0-9-]{0,62}$'),
  title text not null,
  description text not null default '',
  kind text not null check (kind in ('one_on_one', 'group')),
  capacity int not null check (capacity >= 1),
  duration_minutes int not null check (duration_minutes between 5 and 720),
  slot_interval_minutes int not null check (slot_interval_minutes between 5 and 720),
  buffer_before_minutes int not null default 0 check (buffer_before_minutes between 0 and 240),
  buffer_after_minutes int not null default 0 check (buffer_after_minutes between 0 and 240),
  min_notice_minutes int not null default 240 check (min_notice_minutes >= 0),
  horizon_days int not null default 60 check (horizon_days between 1 and 365),
  daily_limit int check (daily_limit is null or daily_limit >= 1),
  location_kind text not null default 'google_meet',
  location_value text,
  questions jsonb not null default '[]',
  active boolean not null default true,
  unique (host_id, slug),
  check (kind = 'group' or capacity = 1)
);

create table public.bookings (
  id uuid primary key,
  chain_id uuid not null,                    -- first booking of a reschedule chain: ICS UID, calendar event
  host_id text not null references public.scheduling_hosts(id) on delete cascade,
  event_type_id uuid not null references public.event_types(id),
  slot_key text not null,
  start_at timestamptz not null,
  end_at timestamptz not null,
  blocked_start timestamptz not null,
  blocked_end timestamptz not null,
  blocked tstzrange generated always as (tstzrange(blocked_start, blocked_end, '[)')) stored,
  status text not null check (status in ('confirmed', 'cancelled')),
  invitee_name text not null,
  invitee_email text not null,
  invitee_time_zone text not null,
  answers jsonb not null default '{}',
  manage_token_hash text not null unique,
  sequence int not null default 0,
  rescheduled_from_id uuid references public.bookings(id),
  cancel_reason text,
  external_event_id text,
  meeting_url text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  check (start_at < end_at and blocked_start <= start_at and end_at <= blocked_end),
  -- The backstop against double booking: two confirmed bookings of one host may overlap only when they are
  -- seats of the same slot. `<>` on text needs btree_gist.
  constraint bookings_no_overlap exclude using gist (
    host_id with =, blocked with &&, slot_key with <>
  ) where (status = 'confirmed')
);
create index bookings_host_start on public.bookings (host_id, start_at);
create index bookings_slot on public.bookings (slot_key) where status = 'confirmed';

create table public.calendar_connections (
  id uuid primary key default gen_random_uuid(),
  host_id text not null references public.scheduling_hosts(id) on delete cascade,
  provider text not null check (provider in ('google', 'microsoft')),
  account_email text not null,
  access_token_enc text not null,
  refresh_token_enc text not null,
  access_expires_at timestamptz not null,
  status text not null default 'active' check (status in ('active', 'needs_reauth')),
  check_calendar_ids text[] not null default '{}',
  write_calendar_id text,
  sync_requested_at timestamptz,
  synced_at timestamptz,
  sync_lease_until timestamptz,
  last_error text,
  unique (host_id, provider, account_email)
);

create table public.calendar_watches (
  id uuid primary key default gen_random_uuid(),
  connection_id uuid not null references public.calendar_connections(id) on delete cascade,
  calendar_id text not null,
  channel_id text not null unique,           -- Google channel id, or Microsoft subscription id
  resource_id text,                          -- Google only, needed to stop the channel
  expires_at timestamptz not null
);

create table public.busy_blocks (
  id bigint generated always as identity primary key,
  connection_id uuid not null references public.calendar_connections(id) on delete cascade,
  host_id text not null,
  start_at timestamptz not null,
  end_at timestamptz not null,
  check (start_at < end_at)
);
create index busy_blocks_host on public.busy_blocks (host_id, start_at);

create table public.webhook_endpoints (
  id uuid primary key default gen_random_uuid(),
  host_id text not null references public.scheduling_hosts(id) on delete cascade,
  url text not null check (url like 'https://%'),
  secret_enc text not null,
  events text[] not null,
  active boolean not null default true
);

create table public.jobs (
  id bigint generated always as identity primary key,
  kind text not null,
  payload jsonb not null,
  run_at timestamptz not null,
  dedupe_key text not null unique,
  attempts int not null default 0,
  lease_until timestamptz,
  last_error text,
  done_at timestamptz,
  booking_id uuid generated always as ((payload->>'bookingId')::uuid) stored
);
create index jobs_due on public.jobs (run_at) where done_at is null;
create index jobs_booking on public.jobs (booking_id) where done_at is null;

-- Server-only tables. New public tables get no grants by default here, service_role included.
do $$
declare t text;
begin
  foreach t in array array['scheduling_hosts','schedules','event_types','bookings','calendar_connections',
                           'calendar_watches','busy_blocks','webhook_endpoints','jobs'] loop
    execute format('alter table public.%I enable row level security', t);
    execute format('revoke all on public.%I from anon, authenticated', t);
    execute format('grant all on public.%I to service_role', t);
  end loop;
end $$;
grant usage, select on all sequences in schema public to service_role;

-- Inserts one booking and its jobs under a per-host lock. Returns 'ok' or the reason it refused.
create or replace function public.csl_insert_booking(b jsonb, capacity int, daily_limit int,
  day_start timestamptz, day_end timestamptz, jobs jsonb)
returns text language plpgsql security definer set search_path = public as $$
declare
  taken int;
  day_slots int;
begin
  -- One host books one at a time; the constraint below would also catch it, this gives a clean answer.
  perform pg_advisory_xact_lock(hashtextextended('csl:' || (b->>'hostId'), 0));
  select count(*) into taken from bookings
    where slot_key = b->>'slotKey' and status = 'confirmed';
  if taken >= capacity then return 'slot_full'; end if;
  if daily_limit is not null and taken = 0 then
    select count(distinct slot_key) into day_slots from bookings
      where event_type_id = (b->>'eventTypeId')::uuid and status = 'confirmed'
        and start_at >= day_start and start_at < day_end;
    if day_slots >= daily_limit then return 'day_full'; end if;
  end if;
  begin
    insert into bookings (id, chain_id, host_id, event_type_id, slot_key, start_at, end_at, blocked_start,
      blocked_end, status, invitee_name, invitee_email, invitee_time_zone, answers, manage_token_hash, sequence,
      rescheduled_from_id, external_event_id, meeting_url)
    values ((b->>'id')::uuid, (b->>'chainId')::uuid, b->>'hostId', (b->>'eventTypeId')::uuid, b->>'slotKey',
      (b->>'start')::timestamptz, (b->>'end')::timestamptz, (b->>'blockedStart')::timestamptz,
      (b->>'blockedEnd')::timestamptz, 'confirmed', b->>'inviteeName', b->>'inviteeEmail',
      b->>'inviteeTimeZone', coalesce(b->'answers', '{}'), b->>'manageTokenHash', (b->>'sequence')::int,
      nullif(b->>'rescheduledFromId', '')::uuid, b->>'externalEventId', b->>'meetingUrl');
  exception when exclusion_violation then
    return 'slot_taken';
  end;
  insert into jobs (kind, payload, run_at, dedupe_key)
    select j->>'kind', j->'payload', (j->>'runAt')::timestamptz, j->>'dedupeKey'
    from jsonb_array_elements(jobs) j
    on conflict (dedupe_key) do nothing;
  return 'ok';
end $$;

create or replace function public.csl_book(b jsonb, capacity int, daily_limit int,
  day_start timestamptz, day_end timestamptz, jobs jsonb)
returns text language sql security definer set search_path = public as $$
  select public.csl_insert_booking(b, capacity, daily_limit, day_start, day_end, jobs);
$$;

-- Cancels the old booking and inserts the new one in one transaction. Raising rolls both back.
create or replace function public.csl_reschedule(old_id uuid, b jsonb, capacity int, daily_limit int,
  day_start timestamptz, day_end timestamptz, jobs jsonb)
returns text language plpgsql security definer set search_path = public as $$
declare
  old_status text;
  outcome text;
begin
  perform pg_advisory_xact_lock(hashtextextended('csl:' || (b->>'hostId'), 0));
  select status into old_status from bookings where id = old_id for update;
  if old_status is null then return 'not_found'; end if;
  if old_status <> 'confirmed' then return 'not_confirmed'; end if;
  -- Free the old slot first, so moving within an overlapping range is not refused by the constraint.
  update bookings set status = 'cancelled', cancel_reason = 'rescheduled', updated_at = now() where id = old_id;
  delete from jobs where booking_id = old_id and done_at is null and kind = 'reminder.invitee';
  outcome := public.csl_insert_booking(b, capacity, daily_limit, day_start, day_end, jobs);
  if outcome <> 'ok' then
    raise exception using errcode = 'P0001', message = outcome;
  end if;
  return 'ok';
end $$;

create or replace function public.csl_cancel(booking uuid, reason text, jobs jsonb)
returns text language plpgsql security definer set search_path = public as $$
declare
  old_status text;
begin
  select status into old_status from bookings where id = booking for update;
  if old_status is null then return 'not_found'; end if;
  if old_status <> 'confirmed' then return 'not_confirmed'; end if;
  update bookings set status = 'cancelled', cancel_reason = reason, sequence = sequence + 1, updated_at = now()
    where id = booking;
  delete from jobs where booking_id = booking and done_at is null and kind = 'reminder.invitee';
  insert into jobs (kind, payload, run_at, dedupe_key)
    select j->>'kind', j->'payload', (j->>'runAt')::timestamptz, j->>'dedupeKey'
    from jsonb_array_elements(jobs) j
    on conflict (dedupe_key) do nothing;
  return 'ok';
end $$;

-- Claims due jobs: a lease, not a status flag, so a crashed run's jobs come back after the lease.
create or replace function public.csl_claim_jobs(max_jobs int, now_at timestamptz, lease_seconds int)
returns setof public.jobs language sql security definer set search_path = public as $$
  update jobs set lease_until = now_at + make_interval(secs => lease_seconds), attempts = attempts + 1
  where id in (
    select id from jobs
    where done_at is null and run_at <= now_at and (lease_until is null or lease_until < now_at)
    order by run_at
    limit max_jobs
    for update skip locked
  )
  returning *;
$$;

-- Replaces one connection's busy blocks in a single transaction, so a slot query never sees half a sync.
create or replace function public.csl_replace_busy(connection uuid, host text, blocks jsonb)
returns void language sql security definer set search_path = public as $$
  delete from busy_blocks where connection_id = connection;
  insert into busy_blocks (connection_id, host_id, start_at, end_at)
    select connection, host, (x->>'start')::timestamptz, (x->>'end')::timestamptz
    from jsonb_array_elements(blocks) x;
$$;

-- Connections that asked for a sync or went stale, and are not leased by another run.
create or replace function public.csl_due_calendar_syncs(now_at timestamptz, stale_before timestamptz, max_rows int)
returns table (id uuid) language sql security definer set search_path = public as $$
  select c.id from calendar_connections c
  where c.status = 'active'
    and (c.sync_requested_at is not null or c.synced_at is null or c.synced_at < stale_before)
    and (c.sync_lease_until is null or c.sync_lease_until < now_at)
  order by c.sync_requested_at nulls last, c.synced_at nulls first
  limit max_rows;
$$;

do $$
declare f text;
begin
  foreach f in array array['csl_insert_booking(jsonb,int,int,timestamptz,timestamptz,jsonb)',
    'csl_book(jsonb,int,int,timestamptz,timestamptz,jsonb)',
    'csl_reschedule(uuid,jsonb,int,int,timestamptz,timestamptz,jsonb)', 'csl_cancel(uuid,text,jsonb)',
    'csl_claim_jobs(int,timestamptz,int)', 'csl_replace_busy(uuid,text,jsonb)',
    'csl_due_calendar_syncs(timestamptz,timestamptz,int)'] loop
    execute format('revoke all on function public.%s from public, anon, authenticated', f);
    execute format('grant execute on function public.%s to service_role', f);
  end loop;
end $$;
```

### Why the functions look like this

| Line | Reason |
|---|---|
| `pg_advisory_xact_lock(hashtextextended('csl:' || host, 0))` | Writes for one host queue up, so the seat count and the daily limit are read and acted on without a race. The exclusion constraint would catch an overlap anyway; the lock gives a clean `slot_full` or `day_full` instead of an error. |
| `exception when exclusion_violation then return 'slot_taken'` | Turns the constraint into a reason the route can show. |
| `raise exception ... message = outcome` in `csl_reschedule` | A refused insert must roll back the cancel of the old booking in the same transaction. The store reads the message back as the reason. |
| `delete from jobs where booking_id = ... and kind = 'reminder.invitee'` | Reminders belong to a start time. Cancelling or moving a booking removes the ones not yet sent. |
| `on conflict (dedupe_key) do nothing` | Inserting the same job twice is harmless. |
| `csl_claim_jobs` with `for update skip locked` and a lease | Overlapping cron runs split the queue instead of both taking it. A run that dies leaves its jobs to the next one after the lease. |
| `csl_replace_busy` | One connection's busy blocks are swapped in one statement pair, so a slot query never sees half a sync. |
| `jobs.booking_id` generated from the payload | Lets the reschedule and cancel functions find a booking's reminders with an index. |

Apply it with `supabase db push`, or `supabase migration up` against a local stack. Check afterwards that
`anon` gets `42501 permission denied` on `/rest/v1/bookings` and on `/rest/v1/rpc/csl_claim_jobs`.

## Stores

The TypeScript side, `supabaseStore` and `supabaseCalendarStore`, is in
[supabase-stores.md](supabase-stores.md).

## Notes

- `SUPABASE_SERVICE_ROLE_KEY` never reaches the browser. The public booking page reads through the routes,
  which run on the server.
- `NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_ANON_KEY` are used only by `host.ts` to read the host's
  session; they grant nothing on these tables.
- Connection pooling: supabase-js talks to PostgREST over HTTP, so serverless functions do not hold Postgres
  connections.
