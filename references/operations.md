# Operations

Environment, schedules, provider setup and the first deploy, on Vercel with Supabase.

## Environment

| Variable | Required | What |
|---|---|---|
| `APP_URL` | yes | Public origin, `https://book.example.com`. Links in e-mails, OAuth redirects, push addresses. |
| `SUPABASE_URL` | yes | Project URL. |
| `SUPABASE_SERVICE_ROLE_KEY` | yes | Server only. |
| `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY` | yes, on Supabase Auth | Read by `host.ts` for the host's session; without them no host can sign in. |
| `SCHEDULING_TOKEN_SECRET` | yes | At least 32 characters. Derives manage tokens and signs OAuth state and channel tokens. Rotating it invalidates every manage link already sent. |
| `SCHEDULING_ENCRYPTION_KEY` | yes | 32 bytes, base64. Seals OAuth tokens and webhook secrets. `openssl rand -base64 32`. |
| `CRON_SECRET` | yes | At least 16 characters. The cron route refuses every request without it. |
| `SENDGRID_API_KEY`, `SCHEDULING_FROM_EMAIL` | for e-mail | Or the host's own mailer behind `Mailer`. |
| `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` | for Google | OAuth web client. |
| `MICROSOFT_CLIENT_ID`, `MICROSOFT_CLIENT_SECRET` | for Microsoft | Entra app registration. |
| `MICROSOFT_TENANT` | no | Default `common`: work, school and personal accounts. |

Set them with `vercel env add`. Supabase from the Vercel Marketplace sets the Supabase variables itself.

These names are the module's; the templates read them, so they are never renamed. A tracked `.env.example`
lists every variable in the table, each with an empty value, and a `.env*` ignore rule gets a
`!.env.example` exception. Real values go only into the deployment's environment or an ignored `.env.local`.
The handover names every variable in this table in the final message itself, with what it is for.

## Schedules

The cron route drains jobs, renews watches and syncs calendars. Run it every five minutes.

**Vercel Pro**, in `vercel.ts`:

```ts
// vercel.ts
import type { VercelConfig } from '@vercel/config/v1';

export const config: VercelConfig = {
  crons: [{ path: '/api/scheduling/cron', schedule: '*/5 * * * *' }],
};
```

Vercel sends `Authorization: Bearer $CRON_SECRET` to cron paths when `CRON_SECRET` is set.

**Vercel Hobby** allows cron jobs once a day only, which is too slow for reminders. Schedule from Supabase
instead, with `pg_cron` and `pg_net`, keeping the secret in Vault:

```sql
-- Run once in the Supabase SQL editor, with your URL and secret.
select vault.create_secret('<CRON_SECRET>', 'scheduling_cron_secret');
select cron.schedule('scheduling-cron', '*/5 * * * *', $$
  select net.http_get(
    url := 'https://book.example.com/api/scheduling/cron',
    headers := jsonb_build_object('authorization', 'Bearer ' ||
      (select decrypted_secret from vault.decrypted_secrets where name = 'scheduling_cron_secret')),
    timeout_milliseconds := 290000
  );
$$);
```

Without any schedule the module still works, more slowly: e-mails go out from `after()` right after each
change, but retries, reminders and calendar renewal wait for the next run.

## Google Cloud

1. Create a project, enable the Google Calendar API.
2. OAuth consent screen: external, the app's name and domain, scopes `openid`, `email`,
   `calendar.calendarlist.readonly`, `calendar.events`.
3. Credentials, OAuth client ID, web application. Authorized redirect URI:
   `<APP_URL>/api/scheduling/calendar/google/callback`.
4. Until the app is verified, only test users added on the consent screen can connect, and their refresh
   tokens expire after seven days while the app is in testing. Calendar scopes are sensitive: submit for
   verification before strangers connect. Plan for days to weeks.
5. Push notifications need a public HTTPS `APP_URL`; a preview URL behind Vercel Authentication cannot receive
   them. The cron's 30-minute re-read covers a deployment without push.

## Microsoft Entra

1. App registrations, new registration, "Accounts in any organizational directory and personal Microsoft
   accounts".
2. Redirect URI, web: `<APP_URL>/api/scheduling/calendar/microsoft/callback`.
3. Certificates and secrets, new client secret. Note its expiry date: an expired secret stops every refresh.
4. API permissions, delegated: `User.Read`, `Calendars.ReadWrite`, `offline_access`, `openid`, `email`.
5. Organizations whose admins require consent will see "Need admin approval"; their admin grants it once.

## First deploy

1. `supabase link` and `supabase db push` with the migration from [supabase.md](supabase.md).
2. Set the environment above on Vercel, for Production and Preview.
3. `vercel deploy --prod`.
4. Create a host row, a schedule and an event type from the dashboard, or with SQL for a first test.
5. Book the page from a private window. Check the e-mail, the ICS, the manage link, a reschedule and a cancel.
6. Connect a calendar, add a busy event in it, and watch the slot disappear within a minute (push) or by the
   next cron run.

## Running it

| Watch | Where | Act when |
|---|---|---|
| Jobs that gave up | `select kind, last_error from jobs where last_error like 'gave up%'` | Any row: a provider is down or a secret expired |
| Connections needing reauth | `calendar_connections.status = 'needs_reauth'` | Tell the host; the dashboard shows a reconnect button |
| Sync errors | `calendar_connections.last_error` | Repeated errors on one connection |
| Watch expiry | `calendar_watches.expires_at < now()` | Rows in the past mean the cron is not running |
| Queue age | `select min(run_at) from jobs where done_at is null` | Older than ten minutes: the cron is not running |

Done jobs can be pruned after 90 days: `delete from jobs where done_at < now() - interval '90 days'`.
