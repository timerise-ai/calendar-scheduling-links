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
import type { LookupAddress } from 'node:dns';
import { lookup } from 'node:dns/promises';
import { request } from 'node:https';
import { BlockList, isIP, type LookupFunction } from 'node:net';
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

/**
 * Node's BlockList matches an IPv4-mapped IPv6 address (`::ffff:127.0.0.1`, which `new URL` rewrites to
 * `::ffff:7f00:1`) against the IPv4 rules, in either spelling. Do not add `::ffff:0:0/96` here: it would
 * match every IPv4 address. Prefixes that carry an IPv4 address inside (NAT64, 6to4, Teredo, the old
 * IPv4-compatible form) are refused whole, because the embedded address is not checked.
 */
const BLOCKED = new BlockList();
for (const [net, prefix] of [
  ['0.0.0.0', 8], ['10.0.0.0', 8], ['100.64.0.0', 10], ['127.0.0.0', 8], ['169.254.0.0', 16], ['172.16.0.0', 12],
  ['192.0.0.0', 24], ['192.0.2.0', 24], ['192.168.0.0', 16], ['198.18.0.0', 15], ['198.51.100.0', 24],
  ['203.0.113.0', 24], ['224.0.0.0', 3],
] as const) {
  BLOCKED.addSubnet(net, prefix, 'ipv4');
}
for (const [net, prefix] of [
  ['::', 96], ['64:ff9b::', 96], ['64:ff9b:1::', 48], ['100::', 64], ['2001::', 32], ['2001:db8::', 32],
  ['2002::', 16], ['fc00::', 7], ['fe80::', 10], ['fec0::', 10], ['ff00::', 8],
] as const) {
  BLOCKED.addSubnet(net, prefix, 'ipv6');
}

export function isPrivateAddress(ip: string): boolean {
  const family = isIP(ip);
  if (family === 0) return true;
  return BLOCKED.check(ip, family === 4 ? 'ipv4' : 'ipv6');
}

export interface PublicTarget {
  url: URL;
  /** The checked addresses. The connection is made to these, never to a second lookup. */
  addresses: LookupAddress[];
}

export type Resolver = (hostname: string) => Promise<LookupAddress[]>;

const systemResolver: Resolver = hostname => lookup(hostname, { all: true });

/**
 * Refuses non-https URLs and hosts that resolve to private, loopback, link-local or CGNAT addresses, and
 * returns the addresses it checked. Run it when the host saves an endpoint and before every delivery.
 * A failed lookup refuses.
 */
export async function resolvePublicUrl(raw: string, resolve: Resolver = systemResolver): Promise<PublicTarget> {
  const url = new URL(raw);
  if (url.protocol !== 'https:') throw new Error('webhook URL must be https');
  if (url.username || url.password) throw new Error('webhook URL must not carry credentials');
  const host = url.hostname.replace(/^\[|\]$/g, '');
  const family = isIP(host);
  const addresses: LookupAddress[] = family === 0 ? await resolve(host) : [{ address: host, family }];
  if (addresses.length === 0 || addresses.some(a => isPrivateAddress(a.address))) {
    throw new Error('webhook URL resolves to a private address');
  }
  return { url, addresses };
}

/** For the endpoint form: the same check, without keeping the addresses. */
export async function assertPublicUrl(raw: string, resolve: Resolver = systemResolver): Promise<URL> {
  return (await resolvePublicUrl(raw, resolve)).url;
}

/**
 * Answers every lookup with the addresses already checked. Without it the request resolves the name again,
 * and a hostname with a very short TTL can answer the check with a public address and the connection with
 * a private one (DNS rebinding). TLS still verifies the certificate against the hostname.
 */
export function pinnedLookup(addresses: LookupAddress[]): LookupFunction {
  return (_hostname, options, callback) => {
    const first = addresses[0];
    if (options.all) callback(null, addresses);
    else if (first) callback(null, first.address, first.family);
    else callback(Object.assign(new Error('no checked address'), { code: 'ENOTFOUND' }), '', 0);
  };
}

export interface WebhookBody {
  id: string;
  event: string;
  occurredAt: string;
  data: Record<string, unknown>;
}

/**
 * One POST to the checked addresses. `node:https`, not `fetch`: fetch has no way to pin the connection to
 * an address. It never follows redirects, so a public URL cannot bounce the request inward, and the
 * response body is discarded unread.
 */
function postPinned(target: PublicTarget, body: string, headers: Record<string, string>): Promise<number> {
  return new Promise((resolve, reject) => {
    const req = request(
      target.url,
      {
        method: 'POST',
        headers: { ...headers, 'content-length': String(Buffer.byteLength(body)) },
        lookup: pinnedLookup(target.addresses),
        signal: AbortSignal.timeout(10_000),
      },
      res => {
        res.on('error', reject);
        res.on('end', () => resolve(res.statusCode ?? 0));
        res.resume();
      },
    );
    req.on('error', reject);
    req.end(body);
  });
}

/** One attempt. The jobs table retries a failure with backoff. */
export async function deliverWebhook(url: string, secret: string, body: WebhookBody, now: Date): Promise<void> {
  const target = await resolvePublicUrl(url);
  const raw = JSON.stringify(body);
  const status = await postPinned(target, raw, {
    'content-type': 'application/json',
    'user-agent': 'scheduling-links-webhooks/1',
    'idempotency-key': body.id,
    [SIGNATURE_HEADER]: signatureHeader(secret, raw, now),
  });
  if (status < 200 || status >= 300) throw new Error(`webhook ${status}`);
}
```

| Guard | Reason |
|---|---|
| `https:` only, no credentials in the URL | A webhook is a request the server makes on a host's behalf. |
| Every resolved address checked, refusal on a failed lookup | A hostname that resolves to `169.254.169.254` or `10.x` would reach the cloud metadata service or the private network. |
| `BlockList`, not prefix matching on the string | `new URL('https://[::ffff:127.0.0.1]/')` gives the host `[::ffff:7f00:1]`; a check on the dotted spelling lets it through to loopback. |
| The connection goes to the checked addresses (`pinnedLookup`) | A second lookup at connect time can answer differently: DNS rebinding with a very short TTL. |
| Redirects are not followed | A public URL may not answer with a redirect to a private one. |
| Check on save and before each delivery | DNS can change between the two. |

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
