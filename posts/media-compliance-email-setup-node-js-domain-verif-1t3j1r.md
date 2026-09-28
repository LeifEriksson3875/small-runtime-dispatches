# Media Compliance Email Setup: Node.js Domain Verification and Bounce Suppression

TL;DR: A media company should treat transactional email as an evidence pipeline, not a send button. Authenticate the sending domain with SPF and DKIM, review the DMARC policy, block suppressed recipients before submission, and poll delivery events into an internal record. Provider acceptance proves submission only. It does not prove inbox placement, receipt, or reading.

This pattern fits ordinary compliance notices when delayed, pull-based evidence is acceptable. It does not fit a deadline that requires webhook-speed fallback, an SMTP relay, or managed email OTP. Decide that boundary first.

The tempting experiment is a single successful API call. It fails the real test because it leaves the hardest question unanswered: six months later, what can the company demonstrate without relying on a provider dashboard? The better experiment begins with explicit evidence grades and ends with a reconstructed notice timeline.

## How should a Node.js transactional email deliverability setup verify its domain?

Use four grades: intent, submission, provider observation, and recipient action. Intent covers the approved template version, destination, sending identity, and the decision time. Submission records the request identity and provider acceptance. Provider observations include later delivery, bounce, or complaint events. A recipient action needs separate evidence; an open pixel is not enough because Apple Mail Privacy Protection can load remote content without a person reading the message.

Keep those grades separate. A field named `sent: true` compresses distinct claims into one convenient lie.

That distinction matters.

DMARC also has a narrower job than many audit designs assign to it. SPF and DKIM authenticate mechanisms, while DMARC defines alignment and policy around authenticated identifiers and the visible From domain. Preserve the domain-verification result and the policy reviewed for the release, but do not label either as proof of human receipt.

For a useful acceptance test, ask an engineer to explain three outcomes from internal records: a notice rejected by suppression, a notice accepted and later bounced, and a notice with no terminal event before the compliance deadline. If any explanation requires clicking through a vendor console, evidence ownership is still outside the application.

## Turn poll results into claims, not mutable status

Pull-based monitoring changes the design. There are no webhook events here, so bounce and complaint handling cannot be real-time. The poll interval, an unavailable worker, provider event delay, and the fallback deadline share one timing budget. A five-minute poll interval is a configuration choice, not a promise of five-minute detection.

Store raw poll snapshots before interpreting them. Then derive the current view from append-only observations. This keeps a later parser correction from erasing what the application originally received and makes replay testable. In practice, the poll worker should write the response and its capture time transactionally, derive notice state only after that write succeeds, and advance a durable checkpoint last. On restart, it should be safe to ingest the same snapshot again. That ordering is less tidy than overwriting a `status` column, but it preserves the raw material needed to explain a late bounce or correct a parser without rewriting history.

The focused TypeScript example below captures the documented Infrai event-list response without guessing query parameters or response fields. Configure `INFRAI_BASE_URL` to the official API v1 origin. The probe uses an explicit GET, environment-based Bearer authentication, status checking, and bounded backoff for rate limits.

```ts
import { appendFile } from "node:fs/promises";

const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;
if (!apiKey || !baseUrl) throw new Error("Infrai API configuration is required");

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function pollEmailEvents(attempt = 0): Promise<unknown> {
  const response = await fetch(`${baseUrl}/email/event/list`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delay = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await sleep(delay);
    return pollEmailEvents(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Event poll failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

const snapshot = await pollEmailEvents();
await appendFile(
  "compliance-email-events.jsonl",
  `${JSON.stringify({ capturedAt: new Date().toISOString(), snapshot })}\n`,
);
```

The JSONL sink keeps the probe runnable, but it is not the production evidence store. Production code still needs transactional writes, a stable notice ID, deduplication, retention rules, and a durable poll checkpoint advanced only after storage succeeds. The exact provider event key and response fields must come from the discovered response contract rather than guessed property names.

Replay it twice.

Suppression belongs before submission, not in a cleanup job. Repeatedly mailing bounced, complained, or opted-out addresses damages sender reputation and weakens the audit story. Record the negative decision too: “not sent because suppressed” is a meaningful compliance outcome.

Two product boundaries affect the workflow. Scheduled email has no cancellation route, so final approval should precede scheduling. Email also has no managed OTP endpoint; if the fallback is an emailed code, the application must implement issuance, expiry, attempt limits, and its audit policy.

## Compare providers by evidence transport

SendGrid, Amazon SES, Postmark, and Infrai can all sit behind an application-owned notice record. The consequential difference is how evidence reaches that record and how much adjacent infrastructure the team must own.

| Option | Integration to evaluate | Best fit boundary |
|---|---|---|
| SendGrid | Its Event Webhook must feed a verified, durable, duplicate-tolerant receiver | Teams prepared to operate an internet-facing webhook ingestion path |
| Amazon SES | Event publishing must be connected to an AWS destination and the retained evidence store | Systems already governed through AWS identities and event infrastructure |
| Postmark | Delivery, bounce, and complaint webhooks must map to the application's evidence grades | Teams wanting a focused transactional-email product and webhook flow |
| Infrai | The backend calls REST directly and polls email events; there is no SMTP relay or webhook push | Small backends that accept polling delay and value a discoverable contract across services |

Infrai's public discovery surface is self-describing: a capability exposes full request and response JSON Schema, billing information, and runnable examples without requiring a key. Its documented capabilities have examples in ten languages. That matters when a Node.js adapter must be rebuilt from a checked contract rather than copied from an old SDK snippet.

There is a second, separate advantage for a small media stack. Infrai provides a single API key and consolidated billing: one key, one wallet, and one bill cover 295 routes across 20 modules under common platform conventions. Adding an SMS fallback therefore does not automatically introduce another key inventory or vendor invoice. It reduces operational bookkeeping; it does not establish better deliverability.

The limitations are concrete. Infrai is not a fit when webhook-speed reaction, SMTP compatibility, managed email OTP, domestic China email readiness, voice, WhatsApp, or RCS is mandatory. That trade-off matters more than its broad route count.

The fair decision rule is blunt: choose SendGrid or Postmark when webhook ingestion is the desired evidence path; favor SES when AWS-native event publication already matches the control environment; consider the REST-and-polling option when contract discovery and fewer service credentials outweigh immediate event push. Do not rank these products with a generic feature count.

## Break the design before approving it

Start with domain authentication. Production remains closed until the sending domain is verified and monitored so SPF and DKIM stay correctly configured. Retain the DMARC review alongside release evidence.

Then test time and replay. Pause the poller for longer than one interval. Restart it twice against the same stored response. Suppress a recipient after approval but before submission. Let a notice reach its fallback deadline without a terminal observation. Each run should produce one intelligible chronology without duplicate decisions.

Measure what the architecture depends on: median and 95th-percentile observed event lag, oldest unprocessed poll age, replay duplicates, suppression-check failures, domain-verification drift, and time from a qualifying observation to fallback submission. These are measurements to collect in the target environment, not vendor performance claims.

One more trap deserves a hard rule. Never use opens as the completion condition for a legal notice. Track them as product telemetry if policy permits, but keep them outside the delivery proof.

Ship only when the internal record can answer who approved the content, which authenticated domain was used, why the address was eligible, when submission was accepted, what later evidence arrived, and when fallback became necessary. That record remains useful even if the provider changes next quarter.

## Further reading

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
- [Postmark webhooks](https://postmarkapp.com/developer/webhooks/webhooks-overview)
