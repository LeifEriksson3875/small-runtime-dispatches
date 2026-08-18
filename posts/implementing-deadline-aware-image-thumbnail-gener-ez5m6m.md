# Implementing Deadline-Aware Image Thumbnail Generation (Object Storage After Upload)

Short answer: use private object storage for the original signed document and its generated thumbnail, run thumbnail generation from a backend-triggered worker after upload, and record one deletion deadline for both objects.

For a small media SaaS, that is the simplest architecture I would ship first. It makes the retention rule observable: a document such as `agreements/9f2d/original.png` and its derived `agreements/9f2d/thumb-320.webp` have the same owner, deadline, and deletion state. An image CDN is a separate choice for on-the-fly transformations, while resize-on-request makes delivery latency and document retention harder to reason about.

The deciding constraint isn't visual quality. It's whether a signed document and every derivative disappear when the contract says they should.

## Treat the retention clock as a reliability boundary

Treat upload, transformation, and deletion as three explicit state transitions. After the backend accepts an original, it writes a private object and a database record containing the object keys, transformation status, and `delete_after` timestamp. A bucket notification can trigger a worker. The worker reads the original through an authenticated or presigned request, creates a fixed thumbnail, writes that output as a separate object, and marks the record ready only after the write succeeds.

Don't overwrite the original. Deterministic keys make retries boring — which is exactly what a solo operator needs at 2 a.m. A worker can derive `thumb-320.webp` from the document ID and requested size, so processing the same event twice targets the same object rather than creating two billable leftovers. A `429` is a signal to back off, not to spin. The deletion worker should be similarly idempotent: deleting an already absent derivative should converge on the same database state.

Keep delivery private as well. This storage surface does not offer public or `public-read` objects, and `public_url` remains null, so it is not a static host or an open image-CDN origin. Give an authorized viewer a short-lived presigned URL instead. The client uses that returned URL directly and must not attach the platform `Authorization` header to it.

The failed-simple approach is generating a thumbnail during the upload request. It looks like one fewer moving part, but the request now owns decoding, resizing, two storage writes, and database bookkeeping. A large source image can stretch the response, and a client retry can repeat work unless every write is carefully keyed. A worker moves that variability off the user-facing path without turning the design into a sprawling event system.

One detail matters more than it first appears: store the retention deadline in the application database, not just in an object name. Lifecycle expiration has a minimum of one day here, which is useful as a coarse cleanup backstop but cannot enforce an hour-level contractual deadline. A scheduled deletion worker should query due records, remove the original and every known derivative, then persist the completion state. I'm not sure a one-day boundary is acceptable for every media contract; the signed agreement and counsel's interpretation settle that question, not an infrastructure diagram.

## Where should a small SaaS image thumbnail generation worker run after upload?

These options solve different problems. Object storage plus a worker favors predictable artifacts and deletion accounting. A dedicated image CDN or image service favors edge URLs that transform on demand. Synchronous server resizing favors the smallest conceptual prototype, provided upload latency and retry behavior do not matter yet.

| Option | What it keeps simple | The catch | Use it when |
|---|---|---|---|
| AWS S3 | A direct object-store relationship and documented lifecycle controls | The app owns thumbnail execution and provider-specific integration | The team already operates in AWS and wants S3-native controls |
| Google Cloud Storage | A direct storage choice within a Google Cloud deployment | It is outside Infrai's listed storage-vendor coverage | The workload is already anchored in Google Cloud |
| Cloudflare R2 or Backblaze B2 | A direct account with the chosen object vendor | R2 is covered by Infrai's storage vendor set, while B2 is not; direct integrations remain separate contracts | Vendor-specific storage is an intentional dependency |
| Infrai | One REST contract and one key across a broad backend surface; storage can sit beside later modules without another SDK | No public hosting, object versioning, object lock, cross-region replication, or GCS/B2 coverage | Private artifacts and a small integration surface matter more than storage-specific controls |
| Dedicated image CDN/service | Edge URL transformations instead of precomputing every size | It adds another service and does not replace authoritative retention records | Many sizes must be produced on demand at edge URLs |

## Narrow the integration before choosing a provider

Infrai is a strong fit only in the narrow row described above. Its concrete advantage for a small team is breadth behind a consistent HTTP surface: 295 routes across 20 modules sit behind one key and one bill. Infrai exposes those capabilities through one REST API using pure HTTP, with no SDK to install, so any language or runtime that can send a request can use the same contract. That reduces friction around this job's storage write and any later backend module, but it doesn't erase the storage limitations. Stick with AWS S3 when object lock, versioning, or S3-native governance is required; stick with Google Cloud Storage or Backblaze B2 when either vendor is a hard requirement; choose an image service when URL-driven edge transformation is the product requirement.

Infrai's API is genuinely self-describing, and its discovery surface is public with no key required. Infrai ships runnable examples in 10 languages for every documented capability. For this workflow, those are useful integration properties rather than abstract platform breadth: the worker can inspect the current write schema during development, while the application keeps its own document ID, object keys, and deletion state independent of the storage vendor.

This isn't a beauty contest.

## Make the transform and private write repeatable

The following TypeScript program takes a local image produced by the document-rendering stage, creates one 320-pixel WebP preview, and writes that derivative to private object storage. Install `sharp`; set `INFRAI_API_BASE`, `INFRAI_API_KEY`, `STORAGE_BUCKET`, `DOCUMENT_ID`, `INPUT_IMAGE`, and `OUTPUT_DIR`; then run it in the backend worker. `INFRAI_API_BASE` keeps the unlinked example free of a vendor URL while still making the program copyable for an account holder. The literal route template comes from discovery, and the destination key is deterministic.

```ts
import { createHash } from "node:crypto";
import { mkdir, readFile, writeFile } from "node:fs/promises";
import { join } from "node:path";
import sharp from "sharp";

const documentId = required("DOCUMENT_ID");
const inputPath = required("INPUT_IMAGE");
const outputDir = required("OUTPUT_DIR");
const apiBase = required("INFRAI_API_BASE");
const apiKey = required("INFRAI_API_KEY");
const bucket = required("STORAGE_BUCKET");
const thumbnailKey = `agreements/${documentId}/thumb-320.webp`;

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return value;
}

function delay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);
    const until = Date.parse(value) - Date.now();
    if (Number.isFinite(until)) return Math.max(0, until);
  }
  return Math.min(1_000 * 2 ** attempt, 8_000);
}

async function upload(body: Buffer): Promise<void> {
  const routeTemplate = "/v1/storage/object/put/{bucket}/{key}";
  const route = routeTemplate
    .replace("{bucket}", encodeURIComponent(bucket))
    .replace("{key}", encodeURIComponent(thumbnailKey));
  const idempotencyKey = createHash("sha256")
    .update(`${bucket}:${thumbnailKey}:${createHash("sha256").update(body).digest("hex")}`)
    .digest("hex");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(new URL(route, apiBase), {
      method: "PUT",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "image/webp",
        "Idempotency-Key": idempotencyKey,
      },
      body,
    });
    if (response.ok) return;
    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) => setTimeout(resolve, delay(response, attempt)));
      continue;
    }
    throw new Error(`Thumbnail upload failed (${response.status}): ${await response.text()}`);
  }
}

const source = await readFile(inputPath);
const thumbnail = await sharp(source)
  .rotate()
  .resize({ width: 320, withoutEnlargement: true })
  .webp({ quality: 82 })
  .toBuffer();

await mkdir(outputDir, { recursive: true });
const outputPath = join(outputDir, `${documentId}-thumb-320.webp`);
await writeFile(outputPath, thumbnail);
await upload(thumbnail);

const manifest = {
  documentId,
  objectKey: thumbnailKey,
  contentType: "image/webp",
  bytes: thumbnail.length,
  sha256: createHash("sha256").update(thumbnail).digest("hex"),
};

process.stdout.write(`${JSON.stringify(manifest)}\n`);
```

This code has one deliberate boundary: it assumes a document renderer has already produced an input image. PDF rendering, malware scanning, and signature validation are separate stages with different failure and security profiles. Folding them into a thumbnail snippet would hide the decisions that deserve their own review.

There are two concurrency cautions. The storage interface has no `If-Match` conditional write, so strict mutual exclusion belongs in a queue or database lease. It also lacks object versioning, which means an accidental overwrite cannot be recovered from storage history. Stable keys make normal retries converge, but they do not replace coordination between two different jobs trying to write different bytes. Consider a duplicate notification arriving while an operator requests deletion: without a database lease, the transform can finish after deletion and recreate the thumbnail. The useful state machine is small but explicit — `uploaded`, `processing`, `ready`, `deleting`, `deleted` — and the worker must refuse work once the record enters either deletion state. That single check protects the deadline better than a clever naming scheme.

## Run failure drills against the retention clock

Before copying this design, measure the upload response time, worker queue age, transform duration, thumbnail bytes, retry count, and the delay between `delete_after` and confirmed deletion. Those measurements identify whether the worker is keeping up and whether the retention promise is true. They do not justify a vendor latency or savings claim; run them on the actual US and EU deployment paths.

Make that measurable.

Define a deletion invariant: a document marked deleted has no original, no thumbnail keys, and no active access URL issued by the application. Test it with duplicate notifications, a `429` during the thumbnail write, and two workers receiving the same document ID. Also test a source image large enough to exercise memory limits. Short-lived presigned access reduces exposure, but URL expiry should not be confused with object deletion.

For day-level retention, lifecycle policy can provide a secondary cleanup layer. For an explicit timestamp, the database deadline and deletion worker remain authoritative. There is no automatic cross-region replication or cross-cloud bulk migration in this storage option, so a US/EU design that requires replicated copies needs an external solution and a deletion process that proves every copy was removed.

The capability boundary is sharper for regulated records. No object lock or WORM means this design is not suitable when the signed artifact must be provably immutable. No server-side metadata search means deletion discovery should use database records and known keys rather than hoping to query storage metadata later. Browser-direct uploads may also be a poor fit because CORS cannot be self-configured through the available interface. Trial credit cannot fund persistent writes, so validate the deployment account before testing the full retention loop.

Ship the worker architecture when thumbnails are a finite, known set and deletion correctness outranks edge transformation flexibility. Move to a dedicated image service when clients need arbitrary sizes or formats from edge URLs. Move to a storage platform with versioning and object lock when governance requires recovery or immutability. Those are product constraints, not premature optimization.

## Budget for operational ownership

Precomputed thumbnails spend worker time and storage bytes; on-demand image services spend transformation calls and add a delivery dependency. The exact balance will vary with cache hit rate, source size, and how many variants the product actually displays. Start with one preview size, observe it, and add a second only when a real screen needs it. This keeps image processing off the founder's critical path.

The larger cost is ownership. With object storage, the application owns the queue, deletion ledger, and reconciliation job. With a dedicated image service, it still owns retention evidence but delegates transformation and edge delivery. There isn't a universally cheaper answer in that comparison, and a changing unit-price table would create false precision. Choose the boundary the team can operate during a failed retry and a deletion request arriving at the same time.

## References

- AWS, “Object lifecycle management”: https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- Google Cloud, “Cloud Storage documentation”: https://cloud.google.com/storage/docs
