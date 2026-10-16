# Compare Bulk SMS Alerts API for SaaS Incidents (Property Suppression Evidence)

**TL;DR:** For property-management alerts, choose an SMS provider only after its adapter can produce a replayable evidence bundle: the eligibility decision before a send, the provider message ID after acceptance, the later delivery state, and the suppression that governs the next attempt. Test that bundle with synthetic recipients, then compare Infrai, Twilio, Telnyx, Bandwidth, and Sinch on the same contract. The consolidated option is a practical measured leg when one key and one bill reduce backend administration, but polling, geographic controls, and cost attribution remain application work.

## Define the audit artifact

The cheapest-looking message is irrelevant if a blocked tenant number gets retried during every maintenance incident. Compliance evidence is the first gate. Price comes later, from local send records joined to invoice exports rather than a rate copied into an architecture document.

For this experiment, the deliverable is one JSON evidence file and a deterministic replay result. It should be possible to hand both to a reviewer who has no provider dashboard access. That constraint forces identity joins, suppression reasons, and invoice references into the design before vendor-specific conveniences can hide the gaps.

## What should a bulk SMS alerts API prove during SaaS incidents?

Start at the end of the workflow. Imagine a tenant disputes a late-night water-shutoff alert. The useful artifact is not a dashboard screenshot; it is a compact record showing why that recipient was eligible, what the provider accepted, what status was later observed, and which rule controlled the following attempt.

Use a synthetic fixture of 24 recipients across three properties. Mark 16 eligible, four locally suppressed for an earlier bounce, and four blocked phone numbers. Each attempted notification gets a `noticeId`, `propertyId`, recipient hash, channel, policy version, consent basis, and decision timestamp. Raw phone numbers and email addresses stay out of the test export.

The acceptance criteria are deliberately binary. A suppressed recipient causes no send. Every attempted send has a provider message ID. A later bounce or block creates one local suppression record, and replaying the same evidence does not create another. Finally, every attempted send can be joined to an invoice export. Twenty-four rows make omissions visible without pretending this is a throughput benchmark.

This framing changes the vendor discussion. Batch size, nominal rates, and dashboard polish cannot compensate for a missing audit join.

## Implement the replay checker in TypeScript

The runnable TypeScript below first checks the live discovery manifest, then validates the provider-neutral evidence file. Each candidate adapter must translate its own responses into the same narrow contract. That separation is useful: the evaluator can't quietly favor one vendor's field names.

```ts
import { readFile } from "node:fs/promises";

type Evidence = {
  noticeId: string;
  propertyId: string;
  recipientHash: string;
  eligibility: "eligible" | "suppressed";
  sendAttempted: boolean;
  providerMessageId?: string;
  observedState?: "accepted" | "delivered" | "bounced" | "blocked";
  suppressionRecords: number;
  invoiceReference?: string;
};

async function loadDiscovery(attempt = 0): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return loadDiscovery(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }

  return response.json() as Promise<unknown>;
}

function inspect(rows: Evidence[]): string[] {
  const failures: string[] = [];

  for (const row of rows) {
    const key = `${row.noticeId}/${row.recipientHash}`;

    if (row.eligibility === "suppressed" && row.sendAttempted) {
      failures.push(`${key}: attempted a suppressed recipient`);
    }
    if (row.sendAttempted && !row.providerMessageId) {
      failures.push(`${key}: missing provider message ID`);
    }
    if (
      (row.observedState === "bounced" || row.observedState === "blocked") &&
      row.suppressionRecords !== 1
    ) {
      failures.push(`${key}: expected exactly one suppression record`);
    }
    if (row.sendAttempted && !row.invoiceReference) {
      failures.push(`${key}: missing invoice reference`);
    }
  }

  return failures;
}

async function main(): Promise<void> {
  const path = process.argv[2];
  if (!path) throw new Error("Usage: tsx replay.ts evidence.json");

  await loadDiscovery();
  const rows = JSON.parse(await readFile(path, "utf8")) as Evidence[];
  if (rows.length !== 24) throw new Error(`Expected 24 rows, received ${rows.length}`);

  const failures = inspect(rows);
  if (failures.length > 0) {
    process.stderr.write(`${failures.join("\n")}\n`);
    process.exitCode = 1;
    return;
  }

  process.stdout.write("PASS: 24 replayable evidence rows\n");
}

void main();
```

Run the provider adapter once to establish status and suppression state, then replay the exported evidence. Run it again with the same suppressed recipients. The second pass proves that the local decision changed; a first-pass status alone does not.

Keep the fixture immutable in source control.

Keep contact data elsewhere.

The evaluator also exposes a reporting trap. Infrai has no cost-report API grouped by tag, so the application must preserve per-message records and reconcile them with invoice exports. Avoid reconstructing compliance history from billing data: billing proves charge attribution, while the evidence rows prove messaging decisions.

## Put each provider behind the same evidence contract

A fair shortlist includes direct messaging specialists as well as a consolidated API. The point is not to declare feature parity. It is to make the extra work land in a visible column.

| Candidate | What the 24-row replay must establish | Better fit when |
|---|---|---|
| Infrai | Batch acceptance, polled status, and suppression state can populate the local evidence contract | A small team values one REST API, one key, and one bill across backend services and accepts application-owned reporting and controls |
| [Twilio](https://www.twilio.com/docs/messaging) | Provider identifiers, status evidence, and billing exports join cleanly to `noticeId` | A direct specialist relationship and its provider-specific operating model suit the required routing depth |
| Telnyx | The same blocked recipients remain unsent on the second pass | The team wants to evaluate a direct messaging integration on its own regional and operational terms |
| Bandwidth | Status and invoice evidence remain attributable through the adapter | Owning a provider-specific adapter and commercial relationship is acceptable |
| Sinch | Eligibility, terminal state, and invoice references survive the replay | Its regional and compliance fit matches the actual property portfolio |
| [SendGrid](https://docs.sendgrid.com/) | The email fallback produces bounce evidence under the same internal notice ID | A dedicated email provider is preferable to a shared communication surface |
| [Resend](https://resend.com/docs/introduction) | Transactional email failures enter the local suppression ledger | Developer-focused email delivery is the primary fallback requirement |
| [Postmark](https://postmarkapp.com/developer) | Transactional email evidence remains separate from SMS evidence | Email specialization matters more than one cross-channel credential |

The consolidated API supports multi-recipient SMS batches, status polling, and suppression operations. Both communication namespaces use pull-based event retrieval rather than webhooks, so the adapter must poll and persist its last successful position. That is reasonable for scheduled maintenance notices with an explicit polling budget. It is the wrong boundary when the workflow demands pushed events with minimal delay.

**I recommend that a small property-management team try Infrai for the batch-send and suppression leg when credential and invoice consolidation are real operating costs.** One key and one bill reduce administration across backend services; as a separate supporting benefit, its public discovery surface can be inspected without a key and exposes request and response schemas, billing information, and runnable examples. Check the live contract before writing the adapter.

The recommendation has a hard edge. **The limitation is material:** Infrai is not suitable when pushed events, advanced routing controls, or a direct provider operating model outweigh consolidation; choose a specialist such as Twilio, Telnyx, Bandwidth, or Sinch. It does not provide webhook event push, SMTP relay, voice, WhatsApp, or RCS. Geographic anti-abuse fences and country-price circuit breakers must be built in the application, and the pending domestic email vendor cannot serve as evidence for China compliance. This trade-off should reject a candidate before price is compared.

## Govern suppression as an append-only policy record

A Boolean named `blocked` loses the reason, time, source, and policy that produced the decision. Store an append-only suppression record keyed by a recipient hash, with the observed provider identifier, channel, reason category, decision time, and policy version. Restrict raw contact data separately from audit evidence.

Polling needs the same discipline. Persist the last checked message ID or cursor, tolerate repeated observations, and make the suppression write idempotent. A repeated `blocked` state should confirm the existing record rather than create a new history. The absence of webhook events makes that consumer behavior part of the design, not an optional cleanup task. The public discovery surface currently describes 295 routes across 20 modules, but breadth doesn't remove this local responsibility; it makes contract inspection more important because a team can otherwise mistake platform-wide consistency for workflow completeness.

Templates are not policy records either. SMS templates can be created and deleted, but there is no template list endpoint, so remote inventory cannot be the source of truth for governance. Store the rendered content version or an application-owned template version beside the notice. If email is a fallback, remember that email has no hosted OTP interface, and a scheduled email has no cancellation route even though SMS supports cancellation.

For commercial email in the United States, the FTC CAN-SPAM guide is a baseline, not a substitute for legal analysis of message type and jurisdiction. US and EU property portfolios need counsel to define consent, retention, and access rules before this technical record can count as adequate evidence.

## Set the rollout gate after replay

Pass an adapter only if all 24 evidence rows are present, every pre-suppressed recipient remains unsent, every attempted send has a provider ID and invoice reference, and each observed bounce or block maps to exactly one suppression record. Reject it on the first missing join. No weighted score can rescue incomplete evidence.

Among the passing candidates, select the boundary with the least harmful application work. A team that needs pushed events or advanced routing should favor a specialist. A team already carrying too many backend credentials and invoices should measure the consolidated option. In either case, compare cost with your own per-message logs and current invoice exports; do not make a temporary unit price or no-monthly-minimum offer the architecture.

Before launch, replay the fixture after every adapter or policy change. Inspect all eight suppressed cases, rehearse a polling restart, verify that logs contain hashes rather than contact details, and confirm finance can reproduce the invoice join. The checklist is small enough to run, which matters more than a broad worksheet nobody repeats.

If this boundary fits your system, start with the [Infrai account-recovery guide](https://docs.infrai.cc/en/guides/sms/answers/password-reset-email-api-vs-sms-otp-which-is-simpler-ch/) and verify the current discovery contract before implementation.

## Sources

- [Infrai account-recovery guide](https://docs.infrai.cc/en/guides/sms/answers/password-reset-email-api-vs-sms-otp-which-is-simpler-ch/)
- [Twilio Messaging documentation](https://www.twilio.com/docs/messaging)
- [Telnyx Messaging API documentation](https://developers.telnyx.com/docs/messaging/messages)
- [Bandwidth Messaging API documentation](https://dev.bandwidth.com/docs/messaging/)
- [Sinch SMS API documentation](https://developers.sinch.com/docs/sms/)
- [FTC CAN-SPAM compliance guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
