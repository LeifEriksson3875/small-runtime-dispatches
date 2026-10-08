# Event Notification Email Deliverability: Proving Bounces and Suppression Decisions

TL;DR: For property-management event notifications, treat deliverability as an evidence problem: verify the sending domain, keep DKIM current, check suppression before every repeat send, and record why each address was allowed or blocked. A plain REST provider is attractive when a small team does not want another SDK, but polling for bounce and complaint events creates a reaction-delay trade-off. Choose the provider whose event path meets the response window you can defend.

This matters when a rent receipt, maintenance update, or inspection reminder targets the same tenant several times. Retrying a bounced address without consulting suppression state is not resilience. It is another failed delivery and a weaker compliance record.

## How should DKIM and event data drive email deliverability troubleshooting?

The useful artifact is not a dashboard screenshot. It is a chronological record connecting a business event to a domain-authentication decision, a suppression decision, a provider message identifier, and any later delivery event. That record should answer a reviewer’s basic questions: Which policy ran? What did it know at send time? Why was this recipient eligible? When did a later bounce or complaint prevent another attempt? If a support engineer cannot reconstruct that chain without guessing, the evidence design is unfinished, even if aggregate delivery looks healthy.

Keep the flow plain. A property event enters a queue. A worker reads the current suppression state, blocks opted-out or previously bounced recipients, sends eligible mail, and stores the result. A separate synchronizer fetches delivery events and advances the suppression record. Domain verification sits before this flow as a deployment gate; DKIM rotation is an operational response when authentication needs to be renewed.

Evidence beats intuition.

## Build the suppression decision before tuning delivery

The following TypeScript file makes the pre-send suppression call directly. It uses one read-only route, accepts the API base URL through the environment because this article is intentionally unlinked, and prints the untouched response for an audit adapter to validate against the current discovery schema. Run it with Node 20 after compiling, or with a TypeScript runner. The important part is less glamorous: no notification leaves the queue until this check completes and its result is durably recorded.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;
const recipient = process.argv[2];

if (!apiKey || !baseUrl || !recipient) {
  throw new Error(
    "Set INFRAI_API_KEY and INFRAI_BASE_URL, then pass the recipient email",
  );
}

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;

    const date = Date.parse(retryAfter);
    if (Number.isFinite(date)) return Math.max(0, date - Date.now());
  }
  return 500 * 2 ** attempt;
}

async function checkSuppression(email: string): Promise<unknown> {
  const path = `/v1/email/suppression/check/${encodeURIComponent(email)}`;

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(new URL(path, baseUrl), {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`Suppression check failed (${response.status}): ${JSON.stringify(body)}`);
    }
    return body;
  }

  throw new Error("Suppression check exhausted its retry budget");
}

const auditRecord = {
  notificationId: "maintenance_notice_1042",
  recipient: recipient.trim().toLowerCase(),
  checkedAt: new Date().toISOString(),
  suppressionResponse: await checkSuppression(recipient),
};

console.log(JSON.stringify(auditRecord, null, 2));
```

The notification ID ties the evidence to the property workflow. The response remains `unknown` on purpose: validate it against the current published schema instead of freezing an assumed field name into compliance logic. Those are application-level controls, not claims about a provider’s retention or legal sufficiency. Set retention, access control, and deletion rules with counsel for the jurisdictions and data classes involved.

One trap deserves emphasis: an opt-out should enter the same decision path even though it did not originate as a delivery event. Otherwise two stores can disagree at the exact moment a scheduled lease reminder runs, and a clean suppression response can be mistaken for permission to send. Map the validated API response and the application’s consent record into one policy decision; persist the raw inputs alongside the outcome so a later schema or policy change does not erase what the worker actually knew.

That race is real.

## Authentication fixes placement, not recipient policy

SPF and DKIM answer authentication questions; they don’t tell the application whether a tenant address has bounced or opted out. DKIM associates a message with a signing domain through a cryptographic signature. Domain verification and DKIM rotation therefore belong in deployment and operations checks, while suppression lookup belongs on the hot path for every repeat notification.

I would make verified domain state a release condition rather than a warning buried in an admin page. When authentication fails, stop and repair it. When a bounce arrives, change recipient eligibility. Combining those into one generic “deliverability” flag loses the reason, the owner, and the evidence needed to reverse the state safely.

Rotation also needs a recorded change: domain, requested time, observed verification state, and the deployment or operator responsible. Do not infer successful authentication from a provider accepting a send request. Acceptance and inbox delivery are different events.

## How quickly must a bounce stop the next message?

Polling determines the worst-case freshness of the application’s local suppression view. If the synchronizer runs every five minutes, a bounce just after one poll can remain unseen until the next. That five-minute example is a design choice, not a provider guarantee. Pick the interval from the shortest legitimate gap between property notifications, then monitor poll completion and the age of the newest processed event.

This is the sharpest boundary in the REST aggregation option described here: email and SMS events are fetched by polling, not pushed by webhook. Frequent synchronization reduces the exposure window but consumes more requests and creates more scheduler work. A slow schedule is simpler, yet it can permit a repeat notification before the prior bounce or complaint is reflected locally.

The trade-off is blunt.

Do not pretend the race disappears. Close it operationally: serialize sends for the same recipient where practical, ingest before releasing queued follow-ups, and store both the provider event time and ingestion time. Those two timestamps show whether a questionable send happened because the event did not yet exist or because the poller lagged. A five-minute polling target also needs a missed-run rule; without one, “scheduled every five minutes” says nothing about how stale the data became during a scheduler outage.

## Compare products on evidence flow, not feature count

All four options can be sensible, but they produce different integration work. Confirm current regional availability and contract terms directly; this table is about the documented control surface, not a universal ranking.

| Option | Integration and event path | Best fit | Boundary to plan for |
|---|---|---|---|
| Infrai | Plain REST API under one key, with no client SDK required; email delivery events use polling | A small backend that values one consistent HTTP interface and can operate a scheduled synchronizer | No webhook event push; the Tencent email vendor is pending, so this is not evidence for a China compliance solution |
| Amazon SES | Domain authentication plus sending notifications that can be published through Amazon SNS | Teams already operating AWS controls and evidence collection | Requires assembling and governing the surrounding AWS resources |
| Twilio SendGrid | Event Webhook plus suppression mechanisms in an email-focused product | Teams that want pushed email events and a dedicated email control plane | Adds a product-specific event schema and integration to maintain |
| Postmark | Webhooks for delivery activity, including bounce events | Transactional-email systems that want event callbacks | The application still needs durable policy and audit records beyond callback receipt |

The REST option has a real ship-first advantage: anything that can issue an HTTP request can integrate, and there is no library version to babysit. Its self-describing discovery surface also publishes request and response schemas, billing data, and runnable examples. For this property-management case, those conveniences matter only after the polling window is acceptable. Compliance evidence is the deciding axis.

Channel scope matters too. This API should not be selected on the assumption that email can fall back to a hosted email OTP endpoint, or that it supplies SMTP relay, voice, WhatsApp, or RCS. Email OTP fallback must be built by the application. Scheduled email also has no cancellation interface, while scheduled SMS does. If those are core workflow requirements, score them before consolidating providers.

## The operational sign-off

Before release, I would ask an operator to produce one evidence bundle for a synthetic property notification: current domain verification, the DKIM change record if rotation occurred, the pre-send suppression decision, the accepted message identifier, the polled delivery event, and the resulting recipient state. Then repeat the exercise with a bounced address and show that a later notification is blocked.

The ongoing check is short but concrete. Watch authentication state, event-poll freshness, unprocessed-event age, and repeated-send decisions. Alert on a stalled poller before notification volume makes the gap hard to reconstruct. Review who can remove suppressions, because deletion changes future eligibility and deserves the same audit care as addition.

For US and EU transactional notifications, this design provides a workable technical foundation; it does not replace legal review or make the application compliant by itself. For China, the pending Tencent email vendor is a clear stop sign for treating this stack as the compliance answer. A fair decision can still favor it for other regions, provided the team accepts polling and owns the evidence pipeline.

## Sources

- [RFC 6376: DomainKeys Identified Mail (DKIM)](https://datatracker.ietf.org/doc/html/rfc6376)
- [Amazon SES: Monitoring email sending using event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
- [Twilio SendGrid: Event Webhook reference](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Postmark: Webhooks overview](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
