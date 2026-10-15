# Booking page and UI

What the invitee and the host see. The templates carry structure, behaviour and accessibility, not a look:
render the elements with the host app's own buttons, inputs, dialogs and layout. No class names ship here on
purpose.

## Pages

| Path | Who | Contents |
|---|---|---|
| `/[handle]/[eventSlug]` | Public | Host name, event title, duration, description, the widget |
| `/[handle]/[eventSlug]?embed=1` | Public, in an iframe | The same, reporting its height and a booked event to the parent |
| `/booking/[token]` | Invitee, from the e-mail | The booking, "Add to calendar", reschedule with the widget, cancel |
| `/dashboard/event-types` | Host | List, create, edit, activate: `eventTypeInput` |
| `/dashboard/availability` | Host | Schedules: weekly ranges per weekday, date overrides, time zone: `scheduleInput` |
| `/dashboard/calendars` | Host | Connect Google or Microsoft, choose calendars to check and one to write, reconnect when `needs_reauth` |
| `/dashboard/bookings` | Host | Upcoming and past bookings with status, answers and the invitee's zone |

The `/[handle]` segment sits at the root of the app. If the host app already uses root segments, mount it
under a prefix such as `/book/[handle]/[eventSlug]` and change the manage link in `runner.ts` with it.

Both pages read the database inside `<Suspense>` after `await connection()`. That builds with
`cacheComponents: true`, which the current `create-next-app` template turns on and which refuses
`export const dynamic`, and keeps the pages per request in apps without it. A missing handle or token still
renders the not-found page, but once the shell has streamed, the status is 200 with a `noindex` tag rather
than 404.

## Public page

```tsx
// app/[handle]/[eventSlug]/page.tsx: the public booking page, /ana/intro. Server component.
import { notFound } from 'next/navigation';
import { connection } from 'next/server';
import { Suspense } from 'react';
import { BookingWidget } from '@/components/scheduling/booking-widget';
import { serviceDeps } from '@/lib/scheduling/runtime';

type Props = { params: Promise<{ handle: string; eventSlug: string }>; searchParams: Promise<{ embed?: string }> };

export default function BookingPage(props: Props) {
  return (
    <Suspense fallback={<main aria-busy="true" />}>
      <BookingPageContent {...props} />
    </Suspense>
  );
}

async function BookingPageContent({ params, searchParams }: Props) {
  // Read per request, never prerendered or cached, with or without cacheComponents.
  await connection();
  const { handle, eventSlug } = await params;
  const { store } = serviceDeps();
  const host = await store.getHostByHandle(handle);
  const eventType = host ? await store.getEventTypeBySlug(host.id, eventSlug) : null;
  if (!host || !eventType || !eventType.active) notFound();
  return (
    <main>
      <header>
        <p>{host.name}</p>
        <h1>{eventType.title}</h1>
        <p>{eventType.durationMinutes} min</p>
        {eventType.description ? <p>{eventType.description}</p> : null}
      </header>
      <BookingWidget
        eventTypeId={eventType.id}
        questions={eventType.questions}
        horizonDays={eventType.horizonDays}
        embedded={(await searchParams).embed === '1'}
      />
    </main>
  );
}
```

## Widget

```tsx
// components/scheduling/booking-widget.tsx: pick a day, pick a time, fill the form, confirm.
'use client';
// Structure and behaviour only: render the elements with the host app's own buttons, inputs and layout.
import { useCallback, useEffect, useMemo, useState } from 'react';
import type { InviteeQuestion } from '@/lib/scheduling/types';

interface SlotDto {
  start: string;
  end: string;
  seatsLeft: number;
}

type Props = {
  eventTypeId: string;
  questions: InviteeQuestion[];
  horizonDays: number;
  embedded?: boolean;
  /** Reschedule mode: the manage token, and the new start goes to the manage endpoint instead. */
  rescheduleToken?: string;
};

const DAY = 86_400_000;

/** The visitor's own zone: slots are grouped and labelled by the day they fall on for them, not the host. */
function visitorZone(): string {
  return Intl.DateTimeFormat().resolvedOptions().timeZone || 'UTC';
}

function dayKey(iso: string, timeZone: string): string {
  return new Intl.DateTimeFormat('en-CA', { timeZone, year: 'numeric', month: '2-digit', day: '2-digit' }).format(new Date(iso));
}

function post(message: Record<string, unknown>) {
  // Height may go to any parent; the booked message carries no personal data, only the start time.
  if (window.parent !== window) window.parent.postMessage({ source: 'scheduling-links', ...message }, '*');
}

export function BookingWidget({ eventTypeId, questions, horizonDays, embedded = false, rescheduleToken }: Props) {
  const [timeZone, setTimeZone] = useState('UTC');
  const [monthStart, setMonthStart] = useState(() => new Date());
  const [slots, setSlots] = useState<SlotDto[] | null>(null);
  const [day, setDay] = useState<string | null>(null);
  const [picked, setPicked] = useState<SlotDto | null>(null);
  const [state, setState] = useState<'idle' | 'sending' | 'done' | 'taken' | 'error'>('idle');

  useEffect(() => setTimeZone(visitorZone()), []);

  const load = useCallback(async () => {
    setSlots(null);
    const from = new Date(Math.max(Date.now(), monthStart.getTime()));
    const to = new Date(Math.min(from.getTime() + 31 * DAY, Date.now() + horizonDays * DAY + DAY));
    const q = new URLSearchParams({ eventTypeId, from: from.toISOString(), to: to.toISOString() });
    const res = await fetch(`/api/scheduling/slots?${q}`, { cache: 'no-store' });
    setSlots(res.ok ? ((await res.json()) as { slots: SlotDto[] }).slots : []);
  }, [eventTypeId, horizonDays, monthStart]);

  useEffect(() => {
    void load();
  }, [load]);

  useEffect(() => {
    if (!embedded) return;
    const send = () => post({ type: 'height', height: document.documentElement.scrollHeight });
    const ro = new ResizeObserver(send);
    ro.observe(document.body);
    return () => ro.disconnect();
  }, [embedded]);

  const byDay = useMemo(() => {
    const m = new Map<string, SlotDto[]>();
    for (const s of slots ?? []) {
      const k = dayKey(s.start, timeZone);
      m.set(k, [...(m.get(k) ?? []), s]);
    }
    return m;
  }, [slots, timeZone]);

  const time = useMemo(() => new Intl.DateTimeFormat(undefined, { timeZone, hour: 'numeric', minute: '2-digit' }), [timeZone]);

  async function submit(form: FormData) {
    if (!picked) return;
    setState('sending');
    const answers = Object.fromEntries(questions.map(q => [q.id, String(form.get(`q_${q.id}`) ?? '')]));
    const res = rescheduleToken
      ? await fetch(`/api/scheduling/manage/${encodeURIComponent(rescheduleToken)}`, { method: 'POST', headers: { 'content-type': 'application/json' }, body: JSON.stringify({ action: 'reschedule', start: picked.start }) })
      : await fetch('/api/scheduling/bookings', {
          method: 'POST',
          headers: { 'content-type': 'application/json' },
          body: JSON.stringify({ eventTypeId, start: picked.start, name: form.get('name'), email: form.get('email'), timeZone, answers }),
        });
    if (res.status === 409) {
      // Someone else took it between listing and booking: say so and refresh the list.
      setState('taken');
      setPicked(null);
      void load();
      return;
    }
    if (!res.ok) return setState('error');
    setState('done');
    post({ type: 'booked', start: picked.start });
  }

  if (state === 'done' && picked) {
    return (
      <section aria-live="polite">
        <h2>{rescheduleToken ? 'Rescheduled' : 'You are booked'}</h2>
        <p>
          {new Date(picked.start).toLocaleDateString(undefined, { timeZone, dateStyle: 'full' })}, {time.format(new Date(picked.start))} ({timeZone})
        </p>
        <p>A confirmation with a calendar invitation is on its way to your inbox.</p>
      </section>
    );
  }

  return (
    <section>
      <nav aria-label="Month">
        <button type="button" onClick={() => setMonthStart(new Date(Math.max(Date.now(), monthStart.getTime() - 31 * DAY)))} disabled={monthStart.getTime() <= Date.now()}>
          Earlier
        </button>
        <button type="button" onClick={() => setMonthStart(new Date(monthStart.getTime() + 31 * DAY))} disabled={monthStart.getTime() + 31 * DAY > Date.now() + horizonDays * DAY}>
          Later
        </button>
        <label>
          Time zone <input value={timeZone} onChange={e => setTimeZone(e.target.value)} aria-describedby="tz-hint" />
        </label>
        <small id="tz-hint">Times are shown in this zone.</small>
      </nav>

      {slots === null ? <p>Loading times</p> : null}
      {slots && slots.length === 0 ? <p>No open times in this period.</p> : null}
      {state === 'taken' ? <p role="alert">That time was just taken. Pick another.</p> : null}

      <ul aria-label="Days">
        {[...byDay.keys()].map(k => (
          <li key={k}>
            <button type="button" aria-pressed={day === k} onClick={() => { setDay(k); setPicked(null); }}>
              {new Date(`${k}T12:00:00Z`).toLocaleDateString(undefined, { weekday: 'short', month: 'short', day: 'numeric', timeZone: 'UTC' })}
            </button>
          </li>
        ))}
      </ul>

      {day ? (
        <ul aria-label="Times">
          {(byDay.get(day) ?? []).map(s => (
            <li key={s.start}>
              <button type="button" aria-pressed={picked?.start === s.start} onClick={() => setPicked(s)}>
                {time.format(new Date(s.start))}
                {s.seatsLeft > 1 ? ` (${s.seatsLeft} seats)` : ''}
              </button>
            </li>
          ))}
        </ul>
      ) : null}

      {picked ? (
        <form action={submit}>
          {rescheduleToken ? null : (
            <>
              <label>
                Name <input name="name" required maxLength={120} autoComplete="name" />
              </label>
              <label>
                Email <input name="email" type="email" required maxLength={254} autoComplete="email" />
              </label>
              {questions.map(q => (
                <label key={q.id}>
                  {q.label}
                  {q.type === 'textarea' ? <textarea name={`q_${q.id}`} required={q.required} maxLength={2000} /> : <input name={`q_${q.id}`} type={q.type === 'phone' ? 'tel' : 'text'} required={q.required} maxLength={2000} />}
                </label>
              ))}
            </>
          )}
          <button type="submit" disabled={state === 'sending'}>
            {rescheduleToken ? 'Move to this time' : 'Confirm'}
          </button>
          {state === 'error' ? <p role="alert">Something went wrong. Try again.</p> : null}
        </form>
      ) : null}
    </section>
  );
}
```

| Behaviour | Reason |
|---|---|
| Groups and labels slots in the visitor's zone, which they can change | The visitor thinks in their own day. The host's zone is the engine's business. |
| Asks for one month at a time and never past the horizon | Matches `MAX_WINDOW_DAYS`; no request for slots that cannot exist. |
| A 409 says the time was just taken and reloads the list | Two people on one page at once is normal; the second gets a clear next step. |
| `aria-pressed` on day and time buttons, `aria-live` on the result, `role="alert"` on errors | Keyboard and screen-reader users can book. |
| In an embed, posts its height and `booked` with the start time only | The parent page gets no personal data. |

## Manage page

```tsx
// app/booking/[token]/page.tsx: the invitee's manage page from the e-mail link. No login: the token is it.
import type { Metadata } from 'next';
import { notFound } from 'next/navigation';
import { connection } from 'next/server';
import { Suspense } from 'react';
import { BookingWidget } from '@/components/scheduling/booking-widget';
import { CancelForm } from '@/components/scheduling/cancel-form';
import { bookingForToken } from '@/lib/scheduling/booking-service';
import { serviceDeps } from '@/lib/scheduling/runtime';

// Never indexed, and the token never leaves in a Referer header.
export const metadata: Metadata = { robots: { index: false, follow: false }, referrer: 'no-referrer' };

type Props = { params: Promise<{ token: string }> };

export default function ManagePage(props: Props) {
  return (
    <Suspense fallback={<main aria-busy="true" />}>
      <ManagePageContent {...props} />
    </Suspense>
  );
}

async function ManagePageContent({ params }: Props) {
  // The booking changes on every cancel and reschedule: never serve a cached copy.
  await connection();
  const { token } = await params;
  const deps = serviceDeps();
  const booking = await bookingForToken(deps, token).catch(() => null);
  if (!booking) notFound();
  const eventType = await deps.store.getEventType(booking.eventTypeId);
  if (!eventType) notFound();
  const when = new Intl.DateTimeFormat('en-US', { timeZone: booking.inviteeTimeZone, dateStyle: 'full', timeStyle: 'short' }).format(booking.start);
  if (booking.status !== 'confirmed') {
    return (
      <main>
        <h1>{eventType.title}</h1>
        <p>This booking is {booking.cancelReason === 'rescheduled' ? 'moved: use the link in your newest e-mail' : 'cancelled'}.</p>
      </main>
    );
  }
  return (
    <main>
      <h1>{eventType.title}</h1>
      <p>
        {when} ({booking.inviteeTimeZone})
      </p>
      <p>
        <a href={`/api/scheduling/manage/${encodeURIComponent(token)}/ics`}>Add to calendar</a>
      </p>
      <h2>Reschedule</h2>
      <BookingWidget eventTypeId={eventType.id} questions={[]} horizonDays={eventType.horizonDays} rescheduleToken={token} />
      <h2>Cancel</h2>
      <CancelForm token={token} />
    </main>
  );
}
```

```tsx
// components/scheduling/cancel-form.tsx: cancel with an optional reason.
'use client';
import { useState } from 'react';

export function CancelForm({ token }: { token: string }) {
  const [state, setState] = useState<'idle' | 'sending' | 'done' | 'error'>('idle');
  async function cancel(form: FormData) {
    setState('sending');
    const res = await fetch(`/api/scheduling/manage/${encodeURIComponent(token)}`, {
      method: 'POST',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify({ action: 'cancel', reason: String(form.get('reason') ?? '') || undefined }),
    });
    setState(res.ok ? 'done' : 'error');
  }
  if (state === 'done') return <p aria-live="polite">Cancelled. The host has been told.</p>;
  return (
    <form action={cancel}>
      <label>
        Reason (optional) <textarea name="reason" maxLength={500} />
      </label>
      <button type="submit" disabled={state === 'sending'}>
        Cancel booking
      </button>
      {state === 'error' ? <p role="alert">Could not cancel. Try again.</p> : null}
    </form>
  );
}
```

The page is `noindex` and sends no Referer: the token is the invitee's only credential, so it must not leak
into a search index or into the logs of a site the invitee clicks to next.

## Embed

One script tag on any site. The script checks the message origin and source window before it acts.

```js
// public/scheduling-embed.js: <div data-scheduling-url="https://book.example.com/ana/intro"></div>
// <script src="https://book.example.com/scheduling-embed.js" async></script>
(function () {
  var nodes = document.querySelectorAll('[data-scheduling-url]');
  for (var i = 0; i < nodes.length; i++) {
    var node = nodes[i];
    if (node.getAttribute('data-scheduling-ready')) continue;
    node.setAttribute('data-scheduling-ready', '1');
    var src = new URL(node.getAttribute('data-scheduling-url'));
    src.searchParams.set('embed', '1');
    var frame = document.createElement('iframe');
    frame.src = src.toString();
    frame.title = node.getAttribute('data-scheduling-title') || 'Book a time';
    frame.style.width = '100%';
    frame.style.border = '0';
    frame.style.minHeight = '640px';
    frame.setAttribute('loading', 'lazy');
    node.appendChild(frame);
    (function (frame, origin) {
      window.addEventListener('message', function (e) {
        // Only messages from this frame's own origin and window count.
        if (e.origin !== origin || e.source !== frame.contentWindow || !e.data || e.data.source !== 'scheduling-links') return;
        if (e.data.type === 'height' && typeof e.data.height === 'number') frame.style.height = Math.max(400, e.data.height) + 'px';
        if (e.data.type === 'booked') frame.dispatchEvent(new CustomEvent('scheduling:booked', { bubbles: true, detail: { start: e.data.start } }));
      });
    })(frame, src.origin);
  }
})();
```

The parent can listen for bookings:

```html
<div data-scheduling-url="https://book.example.com/ana/intro"></div>
<script src="https://book.example.com/scheduling-embed.js" async></script>
<script>
  document.addEventListener('scheduling:booked', e => console.log('booked', e.detail.start));
</script>
```

If the host app sends `X-Frame-Options` or a `frame-ancestors` policy, allow the sites that embed the page on
the booking routes only.

## Dashboard

Build it with the host app's own forms and tables; the rules are in the schemas.

| Screen | Saves | Must |
|---|---|---|
| Availability | `scheduleInput` | Offer the zone list from `Intl.supportedValuesOf('timeZone')`, default the browser's. Show overlapping ranges as a field error. |
| Event type | `eventTypeInput` | Default `slotIntervalMinutes` to the duration. Show the public link and the embed snippet. Keep `capacity` at 1 for one-on-one. |
| Calendars | connection settings | List calendars with `listCalendars`; save `check_calendar_ids` and `write_calendar_id`; call `ensureWatches` and `requestSync` after a change. |
| Bookings | read only | Times in the host's zone, with the invitee's zone beside them. |

Every dashboard mutation runs on the server and checks `currentHostId()` against the row's `host_id`.
