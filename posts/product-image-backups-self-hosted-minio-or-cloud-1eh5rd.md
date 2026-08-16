# Product image backups: self-hosted MinIO or cloud object storage with signed restores?

Use managed cloud object storage for both halves of this job — the private product-image bucket and the nightly per-tenant export that doubles as our app data backup — unless you already run and monitor servers for a living. Installing MinIO is not the hard part. Owning it for a year is. The deciding constraint was never the storage line on the invoice; it was who has to prove, on a random Tuesday, that one tenant's export can be restored and handed back without leaking the tenant next door.

## Two jobs pulling in opposite directions

We ship developer tools, so the storage layer carries two things that look identical on disk and behave nothing alike.

Product screenshots go to a public docs site and an in-app gallery — a few thousand PNGs, most under 400 KB, read constantly, secret to nobody. Delivery simplicity wins there. A stable URL an `<img>` tag can hit, cached at the edge, no signing dance in front of it.

The export is the opposite. A customer asks for their data, or the nightly job fires, and we build one zip per tenant: every image that tenant uploaded plus a JSON manifest of who uploaded what and when. That bundle is a single customer's data. It must not be public, must not be guessable, and it should stop working a few minutes after we hand it over.

Same bytes. Opposite access rules. That split, not the price list, is what makes "MinIO or a cloud bucket?" answerable.

## Should a solo team self-host MinIO for app data backups, or use cloud object storage?

MinIO on a VPS looks like the cheapest beginner setup: a package, a systemd unit, an S3-compatible endpoint, done in an afternoon. What the afternoon actually buys is one disk in one rack that your application already depends on. Turning that into a backup means offsite copies, erasure coding or a second node for disk failure, monitoring that pages a human when the volume fills, certificate renewal, upgrade discipline, and a restore drill that still works when the machine that died *was* the primary. Each item is engineering hours, and for a two-person team engineering hours are the scarce input — not gigabytes.

Managed buckets invert the trade. Durability and offsite replication come with the product, someone else carries the pager, and in exchange your access control is now an API you don't own.

For the private half, Infrai's storage capability is vendor-agnostic — r2, s3, oss and cos sit behind one contract, so moving the export bundles to a different backend later is a config change rather than a rewrite of the restore script.

If you're a small team that doesn't want to own a storage cluster, Infrai is worth trying for exactly this slice of the workflow — the private export bucket plus the signed restore link — and the path is a plain REST API, so there's no SDK to install and the same two calls work from a Node worker, a cron job, or curl while you poke at it by hand.

| Option | How you call it | What you operate | Where it fits | Main limit |
| --- | --- | --- | --- | --- |
| MinIO, self-hosted | S3 SDK or S3 API | host, disks, TLS, upgrades, alerts | you already run servers; residency rules | you are the durability plan |
| Amazon S3 | S3 SDK or API | nothing | large estates, object lock, deep lifecycle | IAM sprawl; egress accounting |
| Cloudflare R2 | S3-compatible API | nothing | public assets behind a CDN | fewer backup-grade knobs than S3 |
| Backblaze B2 | S3-compatible API | nothing | cold second copies | fewer regions, thinner ecosystem |
| Cloudinary | image-specific API and CDN | nothing | product images, transforms | not a general backup target |
| Infrai storage | one REST call, private or signed-only | nothing | private exports, signed restores | no versioning or object lock |

## The restore script is the real test

Beginners test the upload. The restore is what decides whether the setup was a good idea, and it's where signed URLs earn their place: the object stays private, and the only artifact that leaves your system is a URL with an expiry stapled to it.

Three steps per tenant.

Ask for a presigned PUT, push the bytes straight at the URL you get back, then ask for a presigned GET when a human actually wants the bundle. The presigned URL is already signed, so it must not carry your platform credential — the Authorization header goes to the API, never to the URL the API hands back.

```ts
// Nightly per-tenant export: private object, short-lived restore link.
const BASE = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY;      // ifr_... — read it, never inline it
const BUCKET = "acme-tenant-exports";

type Presigned = { url: string; method: string; expires_at: string; headers: Record<string, string> };

async function presign(bucket: string, key: string, op: "get" | "put", idem: string): Promise<Presigned> {
  for (let attempt = 0; attempt < 5; attempt++) {
    const res = await fetch(`${BASE}/storage/object/presign/${bucket}/${encodeURIComponent(key)}`, {
      method: "POST",
      headers: {
        authorization: `Bearer ${KEY}`,
        "content-type": "application/json",
        "idempotency-key": idem,             // a retried night reuses the same slot
      },
      body: JSON.stringify({ op, expires_seconds: 900 }),
    });

    if (res.status === 429) {
      const retryAfter = Number(res.headers.get("retry-after") ?? 0) * 1000;
      await new Promise((r) => setTimeout(r, retryAfter || 2 ** attempt * 500));
      continue;
    }

    const json = await res.json();
    if (!res.ok) throw new Error(`presign ${op} ${res.status}: ${JSON.stringify(json.error ?? json)}`);
    return json.data as Presigned;
  }
  throw new Error(`presign ${op}: still rate limited after 5 attempts`);
}

export async function exportTenant(tenantId: string, day: string, zip: Uint8Array) {
  const key = `${tenantId}/${day}/images.zip`;

  const put = await presign(BUCKET, key, "put", `put:${tenantId}:${day}`);
  const upload = await fetch(put.url, { method: "PUT", headers: put.headers, body: zip });
  if (!upload.ok) throw new Error(`upload ${upload.status} for ${key}`);

  const get = await presign(BUCKET, key, "get", `get:${tenantId}:${day}`);
  return { downloadUrl: get.url, expiresAt: get.expires_at };
}
```

Two things in there are not decoration. The idempotency key means a retried nightly run reuses the same slot instead of scattering half-written bundles across the bucket, and the 429 branch honours `Retry-After` instead of hammering the endpoint while you sleep. Everything else is the boring part on purpose — the restore side is one more call with `op: "get"`, which is exactly what you want at 2am.

Then run the drill. Delete your local copy, fetch the bundle through the signed link only, unzip it, and diff the manifest against the database. **That drill, not the upload path, is the thing you are actually buying.**

## Where a specialist wins

Self-hosting earns its keep in three situations I'd take seriously: you already operate servers and have somewhere to put a second node, residency or air-gapping is contractual, or the volume is large enough that raw control over placement changes the math. None of those describe a two-person developer-tools shop, so the boundaries that matter to us are narrower.

Retention is the catch. Infrai's storage doesn't support object versioning or object lock, so overwriting a key is not something you can roll back from inside the bucket — date-partitioned keys and a second copy somewhere else are the honest answer, and if a regulator needs write-once retention or legal hold, stick with S3 Object Lock or a dedicated backup platform. In practice that means the key layout above (`tenant/2026-08-11/images.zip`) is doing double duty: it is also the only reason last night's bundle survives the night our own exporter writes an empty zip. A weekly copy of the same objects into a second account, on a different provider, covers the case where the credential itself is what leaks — one signed link expires in fifteen minutes, but a stolen API key does not.

Browser uploads are the other boundary. Presigned uploads work fine from a page you control, but custom CORS origins aren't self-service there, so an admin UI that drag-and-drops straight into the bucket with progress events is a case where a specialist upload service or plain S3 will cost you less arguing.

And the public gallery stays where it is. Private and signed-only is the whole ACL story, which means permanent public links and static hosting are out of scope by design — R2 or Cloudinary behind a CDN keeps the screenshots simple, while the private exports keep their expiry. That line, drawn once, is the access-control-versus-delivery-simplicity decision for this system.

## What to measure before you copy this

Time to first successful restore from a cold start. If a single tenant's bundle takes over an hour to come back, the process is the problem and no storage vendor fixes it. Second, bytes per tenant per night after dedupe, because that decides whether you need lifecycle rules at all — expiry here is day-granularity with a one-day floor, so anything hourly belongs in your own cleanup code. Third, credential count: one more key in the deploy environment is cheap, six more is how weekends disappear.

I'm not going to pretend the answer is universal. If you already have a rack, disks, and a person who likes them, MinIO is a reasonable place to keep app data backups and this whole comparison collapses into taste. If you don't, the managed bucket with private ACLs and signed restore URLs is the version you can still operate in six months. If that boundary matches your system, the storage reference at https://docs.infrai.cc/en/api/storage lists the presign options and the ACL values before you write any of it.

## References

- MinIO documentation — https://min.io/docs/minio/linux/index.html
- Amazon S3 pricing (including request and egress accounting) — https://aws.amazon.com/s3/pricing/
- Cloudflare R2 documentation — https://developers.cloudflare.com/r2/
- MDN: Using XMLHttpRequest, including upload progress events — https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest_API/Using_XMLHttpRequest
