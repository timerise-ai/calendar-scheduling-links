# Firestore

The second store, for hosts on Firebase. It implements `SchedulingStore` with `firebase-admin`; the calendar
store (`CalendarStore`) is not shipped for Firestore in this release: implement it from the Supabase version
in [supabase.md](supabase.md), against the same interface in [calendar-sync.md](calendar-sync.md). Firestore
has no cascading delete, so its `deleteConnection` must delete the connection's watches and `busyBlocks`
itself, in one batch with the connection.

## How it keeps the guarantees without SQL

| Postgres | Firestore |
|---|---|
| `pg_advisory_xact_lock` per host | Every booking transaction reads and writes `schedulingLocks/{hostId}`. Firestore serializes transactions that touch one document, so writes for one host queue up. |
| Exclusion constraint | The transaction queries the host's confirmed bookings near the new one and checks overlap itself. Bookings never run past a day plus buffers here, so a two-day margin before `blockedStart` finds every candidate. |
| `on conflict (dedupe_key) do nothing` | The job document id is the base64url of its dedupe key; the transaction skips keys that already exist. |
| `for update skip locked` | `claimJobs` takes each due job in its own short transaction and skips one whose lease is live. |

Transactions read before they write, as Firestore requires: `tryInsert` reads the lock, the nearby bookings,
the day's bookings and the job keys, then writes.

```ts
// lib/scheduling/firestore-store.ts: SchedulingStore on Firestore (firebase-admin). Every booking write runs
// in a transaction that reads and writes the host's lock document, so writes for one host serialize the way
// the per-host advisory lock does on Postgres. Collections: schedulingHosts, schedules, eventTypes, bookings,
// webhookEndpoints, busyBlocks, jobs, schedulingLocks.
import { FieldValue, getFirestore, Timestamp, type DocumentData, type Firestore, type Transaction } from 'firebase-admin/firestore';
import { decryptSecret } from '@/lib/scheduling/crypto';
import type { JobInput, JobKind, JobRecord } from '@/lib/scheduling/jobs';
import type { BookResult, ChangeResult, NewBooking, SchedulingStore } from '@/lib/scheduling/store';
import type { Booking, EventType, Schedule } from '@/lib/scheduling/types';

const toDate = (v: unknown): Date => (v instanceof Timestamp ? v.toDate() : new Date(String(v)));
const DAY = 86_400_000;

function toBooking(id: string, d: DocumentData): Booking {
  return {
    id,
    chainId: d.chainId,
    hostId: d.hostId,
    eventTypeId: d.eventTypeId,
    slotKey: d.slotKey,
    start: toDate(d.start),
    end: toDate(d.end),
    blockedStart: toDate(d.blockedStart),
    blockedEnd: toDate(d.blockedEnd),
    status: d.status,
    inviteeName: d.inviteeName,
    inviteeEmail: d.inviteeEmail,
    inviteeTimeZone: d.inviteeTimeZone,
    answers: d.answers ?? {},
    manageTokenHash: d.manageTokenHash,
    sequence: d.sequence ?? 0,
    rescheduledFromId: d.rescheduledFromId ?? null,
    cancelReason: d.cancelReason ?? null,
    externalEventId: d.externalEventId ?? null,
    meetingUrl: d.meetingUrl ?? null,
    createdAt: toDate(d.createdAt),
    updatedAt: toDate(d.updatedAt),
  };
}

function bookingDoc(b: NewBooking['booking']): DocumentData {
  const { id: _id, start, end, blockedStart, blockedEnd, ...rest } = b;
  return {
    ...rest,
    start: Timestamp.fromDate(start),
    end: Timestamp.fromDate(end),
    blockedStart: Timestamp.fromDate(blockedStart),
    blockedEnd: Timestamp.fromDate(blockedEnd),
    createdAt: FieldValue.serverTimestamp(),
    updatedAt: FieldValue.serverTimestamp(),
  };
}

/** Job ids are their dedupe keys, hashed into a valid document id; create() refuses a second insert. */
function jobRef(db: Firestore, dedupeKey: string) {
  return db.collection('jobs').doc(Buffer.from(dedupeKey).toString('base64url').slice(0, 1400));
}

function writeJobs(db: Firestore, tx: Transaction, jobs: JobInput[], existing: Set<string>) {
  for (const j of jobs) {
    if (existing.has(j.dedupeKey)) continue;
    tx.set(jobRef(db, j.dedupeKey), {
      kind: j.kind,
      payload: j.payload,
      runAt: Timestamp.fromDate(j.runAt),
      dedupeKey: j.dedupeKey,
      bookingId: (j.payload as { bookingId?: string }).bookingId ?? null,
      attempts: 0,
      leaseUntil: Timestamp.fromMillis(0),
      done: false,
      lastError: null,
    });
  }
}

export function firestoreStore(db: Firestore = getFirestore()): SchedulingStore {
  const lockRef = (hostId: string) => db.collection('schedulingLocks').doc(hostId);

  /** Reads everything the checks need inside the transaction, then writes. Firestore wants reads first. */
  async function tryInsert(tx: Transaction, input: NewBooking, ignoreId: string | null = null): Promise<'ok' | 'slot_taken' | 'slot_full' | 'day_full'> {
    const b = input.booking;
    await tx.get(lockRef(b.hostId));
    // Bookings never exceed a day plus buffers in this skill, so a two-day margin finds every overlap.
    const near = await tx.get(
      db
        .collection('bookings')
        .where('hostId', '==', b.hostId)
        .where('status', '==', 'confirmed')
        .where('start', '>=', Timestamp.fromMillis(b.blockedStart.getTime() - 2 * DAY))
        .where('start', '<', Timestamp.fromDate(b.blockedEnd)),
    );
    const dayDocs = input.dailyLimit === null
      ? null
      : await tx.get(
          db
            .collection('bookings')
            .where('eventTypeId', '==', b.eventTypeId)
            .where('status', '==', 'confirmed')
            .where('start', '>=', Timestamp.fromDate(input.dayStart))
            .where('start', '<', Timestamp.fromDate(input.dayEnd)),
        );
    const keys = input.jobs.map(j => jobRef(db, j.dedupeKey));
    const jobSnaps = keys.length ? await tx.getAll(...keys) : [];

    // A reschedule's own old booking is still confirmed inside this transaction: leave it out.
    const live = near.docs.filter(d => d.id !== ignoreId).map(d => toBooking(d.id, d.data()));
    const taken = live.filter(x => x.slotKey === b.slotKey).length;
    if (taken >= input.capacity) return 'slot_full';
    if (dayDocs && input.dailyLimit !== null && taken === 0) {
      const slots = new Set(dayDocs.docs.filter(d => d.id !== ignoreId).map(d => String(d.data().slotKey)));
      if (slots.size >= input.dailyLimit) return 'day_full';
    }
    const clash = live.some(x => x.slotKey !== b.slotKey && x.blockedStart < b.blockedEnd && b.blockedStart < x.blockedEnd);
    if (clash) return 'slot_taken';

    tx.set(db.collection('bookings').doc(b.id), bookingDoc(b));
    writeJobs(db, tx, input.jobs, new Set(jobSnaps.filter(s => s.exists).map(s => String(s.data()?.dedupeKey))));
    tx.set(lockRef(b.hostId), { touchedAt: FieldValue.serverTimestamp() });
    return 'ok';
  }

  const get = async (col: string, id: string): Promise<DocumentData | null> => {
    const s = await db.collection(col).doc(id).get();
    return s.exists ? { ...(s.data() as DocumentData), id: s.id } : null;
  };

  const loadBooking = async (id: string): Promise<Booking> => {
    const s = await db.collection('bookings').doc(id).get();
    if (!s.exists) throw new Error(`booking ${id} vanished after write`);
    return toBooking(s.id, s.data() as DocumentData);
  };

  return {
    async getHostByHandle(handle) {
      const q = await db.collection('schedulingHosts').where('handle', '==', handle.toLowerCase()).limit(1).get();
      const d = q.docs[0];
      return d ? { id: d.id, handle: d.data().handle, name: d.data().name, email: d.data().email, timeZone: d.data().timeZone } : null;
    },
    async getHost(id) {
      const d = await get('schedulingHosts', id);
      return d ? { id, handle: d.handle, name: d.name, email: d.email, timeZone: d.timeZone } : null;
    },
    async getEventTypeBySlug(hostId, slug) {
      const q = await db.collection('eventTypes').where('hostId', '==', hostId).where('slug', '==', slug).limit(1).get();
      const d = q.docs[0];
      return d ? ({ ...(d.data() as Omit<EventType, 'id'>), id: d.id } as EventType) : null;
    },
    async getEventType(id) {
      const d = await get('eventTypes', id);
      return d ? (d as unknown as EventType) : null;
    },
    async getSchedule(id) {
      const d = await get('schedules', id);
      return d ? (d as unknown as Schedule) : null;
    },
    async listBookedIntervals(hostId, w) {
      const q = await db
        .collection('bookings')
        .where('hostId', '==', hostId)
        .where('status', '==', 'confirmed')
        .where('start', '>=', Timestamp.fromMillis(w.start.getTime() - 2 * DAY))
        .where('start', '<', Timestamp.fromDate(w.end))
        .get();
      return q.docs
        .map(d => toBooking(d.id, d.data()))
        .filter(b => b.blockedStart < w.end && w.start < b.blockedEnd)
        .map(b => ({ id: b.id, eventTypeId: b.eventTypeId, slotKey: b.slotKey, start: b.start, blockedStart: b.blockedStart, blockedEnd: b.blockedEnd }));
    },
    async listBusy(hostId, w) {
      // Busy blocks are capped at the sync window; a 40-day lookback covers a long all-day absence.
      const q = await db.collection('busyBlocks').where('hostId', '==', hostId).where('start', '<', Timestamp.fromDate(w.end)).where('start', '>=', Timestamp.fromMillis(w.start.getTime() - 40 * DAY)).get();
      return q.docs.map(d => ({ start: toDate(d.data().start), end: toDate(d.data().end) })).filter(i => i.end > w.start);
    },
    async getBooking(id) {
      const s = await db.collection('bookings').doc(id).get();
      return s.exists ? toBooking(s.id, s.data() as DocumentData) : null;
    },
    async getBookingByTokenHash(hash) {
      const q = await db.collection('bookings').where('manageTokenHash', '==', hash).limit(1).get();
      const d = q.docs[0];
      return d ? toBooking(d.id, d.data()) : null;
    },
    async listWebhookEndpointIds(hostId, event) {
      const q = await db.collection('webhookEndpoints').where('hostId', '==', hostId).where('active', '==', true).where('events', 'array-contains', event).get();
      return q.docs.map(d => d.id);
    },
    async getWebhookEndpoint(id) {
      const d = await get('webhookEndpoints', id);
      return d ? { id, hostId: d.hostId, url: d.url, secret: decryptSecret(d.secretEnc), events: d.events ?? [], active: d.active === true } : null;
    },
    async book(input): Promise<BookResult> {
      const r = await db.runTransaction(tx => tryInsert(tx, input));
      if (r !== 'ok') return { ok: false, reason: r };
      return { ok: true, booking: await loadBooking(input.booking.id) };
    },
    async reschedule(oldId, input): Promise<ChangeResult> {
      const oldRef = db.collection('bookings').doc(oldId);
      const r = await db.runTransaction(async tx => {
        const old = await tx.get(oldRef);
        if (!old.exists) return 'not_found' as const;
        if (old.data()?.status !== 'confirmed') return 'not_confirmed' as const;
        const reminders = await tx.get(db.collection('jobs').where('bookingId', '==', oldId).where('kind', '==', 'reminder.invitee').where('done', '==', false));
        const outcome = await tryInsert(tx, input, oldId);
        if (outcome !== 'ok') return outcome;
        tx.update(oldRef, { status: 'cancelled', cancelReason: 'rescheduled', updatedAt: FieldValue.serverTimestamp() });
        for (const j of reminders.docs) tx.delete(j.ref);
        return 'ok' as const;
      });
      if (r !== 'ok') return { ok: false, reason: r };
      return { ok: true, booking: await loadBooking(input.booking.id) };
    },
    async cancel(id, reason, jobs): Promise<ChangeResult> {
      const ref = db.collection('bookings').doc(id);
      const r = await db.runTransaction(async tx => {
        const s = await tx.get(ref);
        if (!s.exists) return 'not_found' as const;
        if (s.data()?.status !== 'confirmed') return 'not_confirmed' as const;
        const reminders = await tx.get(db.collection('jobs').where('bookingId', '==', id).where('kind', '==', 'reminder.invitee').where('done', '==', false));
        const snaps = jobs.length ? await tx.getAll(...jobs.map(j => jobRef(db, j.dedupeKey))) : [];
        tx.update(ref, { status: 'cancelled', cancelReason: reason, sequence: FieldValue.increment(1), updatedAt: FieldValue.serverTimestamp() });
        for (const j of reminders.docs) tx.delete(j.ref);
        writeJobs(db, tx, jobs, new Set(snaps.filter(x => x.exists).map(x => String(x.data()?.dedupeKey))));
        return 'ok' as const;
      });
      if (r !== 'ok') return { ok: false, reason: r };
      return { ok: true, booking: await loadBooking(id) };
    },
    async setBookingExternal(id, f) {
      const patch: DocumentData = { updatedAt: FieldValue.serverTimestamp() };
      if (f.externalEventId !== undefined) patch.externalEventId = f.externalEventId;
      if (f.meetingUrl !== undefined) patch.meetingUrl = f.meetingUrl;
      await db.collection('bookings').doc(id).update(patch);
    },
    async claimJobs(limit, now, leaseSeconds): Promise<JobRecord[]> {
      const due = await db.collection('jobs').where('done', '==', false).where('runAt', '<=', Timestamp.fromDate(now)).orderBy('runAt').limit(limit * 2).get();
      const claimed: JobRecord[] = [];
      for (const d of due.docs) {
        if (claimed.length >= limit) break;
        // One transaction per job: a job another worker leased in the meantime is skipped, never run twice.
        const job = await db.runTransaction(async tx => {
          const s = await tx.get(d.ref);
          const x = s.data();
          if (!x || x.done || toDate(x.leaseUntil).getTime() > now.getTime()) return null;
          tx.update(d.ref, { leaseUntil: Timestamp.fromMillis(now.getTime() + leaseSeconds * 1000), attempts: FieldValue.increment(1) });
          return { id: s.id, kind: x.kind as JobKind, payload: x.payload, runAt: toDate(x.runAt), attempts: Number(x.attempts) + 1, dedupeKey: String(x.dedupeKey) };
        });
        if (job) claimed.push(job);
      }
      return claimed;
    },
    async completeJob(id) {
      await db.collection('jobs').doc(id).update({ done: true, leaseUntil: Timestamp.fromMillis(0), lastError: null });
    },
    async failJob(id, error, at) {
      await db.collection('jobs').doc(id).update(
        at
          ? { runAt: Timestamp.fromDate(at), leaseUntil: Timestamp.fromMillis(0), lastError: error.slice(0, 2000) }
          : { done: true, leaseUntil: Timestamp.fromMillis(0), lastError: `gave up: ${error}`.slice(0, 2000) },
      );
    },
  };
}
```

## Indexes

Create these composite indexes before the first booking; the emulator does not ask for them, production does.

| Collection | Fields |
|---|---|
| `bookings` | `hostId` ASC, `status` ASC, `start` ASC |
| `bookings` | `eventTypeId` ASC, `status` ASC, `start` ASC |
| `bookings` | `manageTokenHash` ASC |
| `busyBlocks` | `hostId` ASC, `start` ASC |
| `jobs` | `done` ASC, `runAt` ASC |
| `jobs` | `bookingId` ASC, `kind` ASC, `done` ASC |
| `webhookEndpoints` | `hostId` ASC, `active` ASC, `events` ARRAY |
| `eventTypes` | `hostId` ASC, `slug` ASC |

## Security rules

All collections are server-only. Deny every client read and write in `firestore.rules`; the Admin SDK
bypasses rules.

```text
rules_version = '2';
service cloud.firestore {
  match /databases/{db}/documents {
    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

## Verified

The store contract suite in [testing.md](testing.md) runs against the Firestore emulator: booking, the
refusals, four concurrent requests for one slot with exactly one winner, group seats with a daily limit,
rescheduling into an overlapping time, and job leases.
