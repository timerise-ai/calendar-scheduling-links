# Webhooks

Outgoing webhooks for booking changes, delivered by the `webhook.deliver` job from
[jobs-and-notifications.md](jobs-and-notifications.md).

## Signing and delivery

The receiver verifies `Scheduling-Signature: t=<unix seconds>,v1=<hex HMAC-SHA256 of "<t>.<raw body>">` and
rejects a timestamp more than five minutes off. The body's `id` is the job's dedupe key, stable across
retries, so the receiver can drop a repeat.

```ts
// lib/scheduling/webhooks.ts: signed outgoing webhooks. The receiver verifies
// `Scheduling-Signature: t=<unix seconds>,v1=<hex HMAC-SHA256 of "<t>.<raw body>">` and rejects old timestamps.
import { lookup } from 'node:dns/promises';
import { isIP } from 'node:net';
import { hmacHex, safeEqual } from '@/lib/scheduling/tokens';

export const SIGNATURE_HEADER = 'Scheduling-Signature';
export const SIGNATURE_TOLERANCE_SECONDS = 300;

export function signatureHeader(secret: string, body: string, now: Date): string {
  const t = Math.floor(now.getTime() / 1000);
  return `t=${t},v1=${hmacHex(secret, `${t}.${body}`)}`;
}

/** For the receiving side and for tests. */
export function verifySignature(secret: string, body: string, header: string, now: Date): boolean {
  const parts = Object.fromEntries(header.split(',').map(p => p.split('=', 2) as [string, string]));
  const t = Number(parts.t);
  if (!Number.isFinite(t) || Math.abs(now.getTime() / 1000 - t) > SIGNATURE_TOLERANCE_SECONDS) return false;
  return typeof parts.v1 === 'string' && safeEqual(parts.v1, hmacHex(secret, `${t}.${body}`));
}

function isPrivateV4(ip: string): boolean {
  const [a = 0, b = 0] = ip.split('.').map(Number);
  return (
    a === 0 || a === 10 || a === 127 || (a === 100 && b >= 64 && b <= 127) || (a === 169 && b === 254) ||
    (a === 172 && b >= 16 && b <= 31) || (a === 192 && b === 168) || (a === 198 && (b === 18 || b === 19)) || a >= 224
  );
}

function isPrivateV6(ip: string): boolean {
  const v = ip.toLowerCase();
  if (v === '::' || v === '::1') return true;
  if (v.startsWith('::ffff:')) return isPrivateV4(v.slice(7));
  return /^(fc|fd|fe8|fe9|fea|feb|ff)/.test(v);
}

export function isPrivateAddress(ip: string): boolean {
  return isIP(ip) === 4 ? isPrivateV4(ip) : isPrivateV6(ip);
}

/**
 * Refuses non-https URLs and hosts that resolve to private, loopback, link-local or CGNAT addresses.
 * Run it when the host saves an endpoint and again before every delivery. A failed lookup refuses.
 */
export async function assertPublicUrl(raw: string): Promise<URL> {
  const url = new URL(raw);
  if (url.protocol !== 'https:') throw new Error('webhook URL must be https');
  if (url.username || url.password) throw new Error('webhook URL must not carry credentials');
  const host = url.hostname.replace(/^\[|\]$/g, '');
  const addresses = isIP(host) ? [host] : (await lookup(host, { all: true })).map(a => a.address);
  if (addresses.length === 0 || addresses.some(isPrivateAddress)) throw new Error('webhook URL resolves to a private address');
  return url;
}

export interface WebhookBody {
  id: string;
  event: string;
  occurredAt: string;
  data: Record<string, unknown>;
}

/** One attempt. Redirects are not followed: a public URL may not bounce the request inward. */
export async function deliverWebhook(url: string, secret: string, body: WebhookBody, now: Date): Promise<void> {
  const target = await assertPublicUrl(url);
  const raw = JSON.stringify(body);
  const res = await fetch(target, {
    method: 'POST',
    redirect: 'manual',
    headers: {
      'content-type': 'application/json',
      'user-agent': 'scheduling-links-webhooks/1',
      'idempotency-key': body.id,
      [SIGNATURE_HEADER]: signatureHeader(secret, raw, now),
    },
    body: raw,
    signal: AbortSignal.timeout(10_000),
  });
  if (res.status < 200 || res.status >= 300) throw new Error(`webhook ${res.status}`);
}
```

| Guard | Reason |
|---|---|
| `https:` only, no credentials in the URL | A webhook is a request the server makes on a host's behalf. |
| Every resolved address checked, refusal on a failed lookup | A hostname that resolves to `169.254.169.254` or `10.x` would reach the cloud metadata service or the private network. |
| `redirect: 'manual'` | A public URL may not answer with a redirect to a private one. |
| Check on save and before each delivery | DNS can change between the two. |

Accepted risk: a hostname can still re-resolve between the check and the connection (DNS rebinding with a
very short TTL). Closing that needs a custom agent that connects to the checked address; add it when hosts
can register endpoints without review.

The payload:

```json
{
  "id": "webhook:<endpointId>:<bookingId>:created:0",
  "event": "booking.created",
  "occurredAt": "2026-10-15T09:00:00.000Z",
  "data": { "booking": { "id": "...", "chainId": "...", "start": "...", "status": "confirmed", "answers": {} } }
}
```

Events: `booking.created`, `booking.rescheduled` (sent for the new booking), `booking.cancelled`.
