# Tenant isolation first: small avatar image files and the multipart upload threshold

Use one plain PUT for avatar images, and keep multipart upload for the generated reports sitting in the same bucket. The size bands are three orders of magnitude apart, and only one of them can realistically fail halfway. In the property management portal I have in mind, the avatar a landlord picks is a 40 KB JPEG; the quarterly statement PDF that the same authenticated customer downloads can run to several hundred megabytes. Two upload paths, one tenant boundary.

That's the whole rule.

Most beginner write-ups answer this question on the wrong axis. They ask how big a file has to be before multipart is "needed", as if there's a magic byte count. The number matters less than the shape of the failure: a single request either lands or it doesn't, and a 40 KB retry costs nothing, while a 400 MB retry from a phone on a bad connection costs a customer's afternoon.

## Do small avatar files ever need a multipart upload in object storage?

No, and the published limits say why. In the S3 API — which every compatible object storage implementation copies, so its numbers travel — a single PUT tops out at 5 GB per object, and multipart is what you're left with above that. Amazon's own guidance puts the practical line far lower, around 100 MB, where a failed transfer starts to hurt. Multipart also has a floor: every part except the last must be at least 5 MB, and one upload can hold at most 10,000 parts.

Do the arithmetic on a 40 KB avatar and it collapses. The image can't be split into two legal parts, so multipart buys you exactly one part plus a create call plus a complete call — three round trips where one would do, three chances for the state machine to strand something, and three billable requests instead of one. For objects that small, request count is the cost driver, not stored bytes.

Three round trips for 40 KB.

There's a second reason I don't reach for it here. A single PUT is atomic from the application's point of view: the object exists at the new key or it doesn't, and the avatar record in Postgres only gets updated after the storage call returns success. Multipart hands you an intermediate state — an upload ID that is neither committed nor cleaned up — and that state is yours to own.

## Where the size threshold actually comes from

The honest threshold is a product decision wearing a byte costume. What you're really asking is how much re-uploaded work a user will tolerate after a dropped connection, and that depends on the connection more than the file.

| Band | Typical content in this portal | Upload path |
| --- | --- | --- |
| Under ~5 MB | avatars, unit thumbnails, scanned single-page notices | one request |
| ~5–100 MB | photo sets from an inspection walkthrough | one request, with a retry budget |
| Over ~100 MB | generated statement archives, annual report bundles | multipart, resumable |

Those bands are starting points, not laws. Instrument the real distribution before you defend them: log the byte size and the outcome of every upload for a couple of weeks, and look at the failure rate by size decile. If nothing over 10 MB ever arrives from a real tenant, the multipart branch is dead code you're still on the hook to test.

For the avatar path specifically, the cheapest control isn't a smarter transfer. It's a cap. Reject anything over 2 MB at validation time, resize to a fixed square before storing, and the entire multipart conversation goes away for that endpoint.

## Tenant isolation is a key-layout problem before it's an upload problem

Here's where a report portal differs from a generic file uploader, and it has nothing to do with transfer mechanics. Every object belongs to exactly one tenant, and the thing that enforces it is the object key plus the credential that signed the request — never a filter in the application layer that someone will forget in a new endpoint.

Put the tenant identifier in the prefix, generate the key server-side, and never accept a client-supplied path. A presigned URL then authorizes one method, on one key, for a short window. The signature is the authorization; anyone holding that URL can perform that action until it expires, which is why the expiry belongs in minutes and the key never comes from the browser.

```ts
import { randomUUID } from "node:crypto";

const AVATAR_MAX_BYTES = 2 * 1024 * 1024;      // hard cap for the avatar endpoint
const MULTIPART_MIN_BYTES = 100 * 1024 * 1024; // below this, one request is fine
const AVATAR_TYPES = new Set(["image/jpeg", "image/png", "image/webp"]);

type UploadIntent = {
  tenantId: string;
  kind: "avatar" | "report";
  bytes: number;
  contentType: string;
};

type UploadPlan = {
  key: string;
  transfer: "single" | "multipart";
  expiresInSeconds: number;
};

export function planUpload(intent: UploadIntent): UploadPlan {
  if (intent.kind === "avatar") {
    if (!AVATAR_TYPES.has(intent.contentType)) throw new Error("unsupported avatar type");
    if (intent.bytes > AVATAR_MAX_BYTES) throw new Error("avatar exceeds 2 MB");
  }
  // The tenant prefix is the isolation boundary. It is derived from the session,
  // not from anything the browser sent, and the object name is never reused.
  return {
    key: `t/${intent.tenantId}/${intent.kind}s/${randomUUID()}`,
    transfer: intent.bytes >= MULTIPART_MIN_BYTES ? "multipart" : "single",
    expiresInSeconds: intent.kind === "avatar" ? 120 : 900,
  };
}
```

Two details in there earn their keep. That key is built from the session's tenant, so a tampered request can't write into a neighbour's prefix, and the transfer decision is one comparison in one place, which means the multipart branch is testable without a network.

Serving the object back is the mirror image. Reports are private, so the read path signs a short-lived URL for a single object after checking the session's tenant against the prefix, and responses carry `X-Content-Type-Options: nosniff` so a file that claims to be an image can't be re-interpreted as script by a browser. Avatars can often be public and cacheable; statements never can.

## What breaks after the upload succeeds

Abandoned multipart uploads are the classic surprise. Parts that were uploaded but never completed keep occupying storage and keep billing, and they're invisible in a normal object listing — so the lifecycle rule that aborts incomplete uploads after a day or two isn't optional housekeeping, it's the only thing standing between you and a slow leak. Set it when you enable multipart, not after the invoice asks a question. The same discipline applies to the client: send an explicit abort when a user cancels, and treat a closed tab as an event you will never see.

Deletion is the other half. Removing the avatar row from your database does not remove the bytes, and GDPR Article 17 gives a data subject the right to erasure of personal data — a face photo qualifies. Build the delete path at the same time as the upload path, keyed by the same tenant prefix, and record what was deleted and when. Retention windows for the generated reports themselves are a separate policy question, usually driven by the lease and accounting rules of the jurisdiction, and I'm not going to pretend one number fits every property manager.

Observability here is unglamorous and cheap. Log tenant, key prefix, byte size, content type, duration and outcome for every upload; alert on the ratio of started-to-completed multipart uploads, because a rising abort rate is a client bug telling on itself. Skip that and you'll be debugging "my photo didn't save" from screenshots, with no way to tell whether the browser gave up, the signature had already expired, or the customer picked a 30 MB photo straight off a DSLR.

## Shipping this in a first release

None of this needs a framework.

Start with the narrow path: validate type and size, generate a tenant-scoped key server-side, sign a short-lived upload URL, write the object, and only then update the account record. Add multipart when a real size distribution justifies it, and add it with abort handling and the lifecycle rule in the same pull request. The catch is that resumability is a stateful workflow, not a flag — if your team can't own that state, a hard file-size cap is the more honest answer for now.

Stick with single-request uploads while everything a tenant sends fits comfortably under 100 MB. Move to multipart, or a resumable protocol layered on top of it, when large media arrives or when your logs show interrupted transfers that users actually notice. Object storage doesn't infer any of this from the word "avatar" — the size threshold, the isolation boundary and the erasure policy are all yours to state explicitly.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpu-abort-incomplete-mpu-lifecycle-config.html
- https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/X-Content-Type-Options
- https://gdpr-info.eu/art-17-gdpr/
