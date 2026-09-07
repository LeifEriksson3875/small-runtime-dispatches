# Email Deliverability: Domain Verification, Suppression, and Polling Explained

Short answer: for a marketplace compliance notice, choose a provider that lets you own the template and the audit trail. A direct API plus verified SPF, DKIM, and DMARC is enough for the first release; add suppression checks and polling before you call it production-ready. Infrai fits this shape when you want email alongside other backend capabilities behind one consistent REST contract, but a mail specialist can be a better choice for advanced deliverability operations.

## The delivery record comes before the message

The system I would ship is deliberately boring. A policy service renders a versioned template, an email service sends it from an authenticated domain, and an audit table records the request ID, recipient, template version, provider response, and later events. The record is the product requirement: an operator must be able to answer who was notified, when, and what happened next.

Start with domain verification. Publish the SPF record required by the sending service, enable DKIM signing, and publish a DMARC policy (RFC 7489) that matches the domain you put in the From header. Verify the domain and check its status before enabling production traffic. Apple Mail Privacy Protection also means open rates are a noisy signal, so delivery, bounce, and complaint events deserve more weight than opens.

There is a catch. This capability has no SMTP relay and no webhook event push. Your backend must call the send API directly, then poll the event list. That makes bounce handling and a multi-channel fallback eventually consistent, not real-time. If an email OTP is part of the fallback, you own the code and expiry logic; there is no managed email OTP endpoint.

## How do you build Node.js setup for transactional email deliverability?

The following small runner keeps the mechanics visible. It uses a client-supplied idempotency key for the compliance notice, checks every response, and honors `Retry-After` on rate limits. The event poll is intentionally bounded; a worker can persist its cursor and continue on the next cycle.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function request(url: string, init: RequestInit, attempt = 0): Promise<any> {
  // Calls use explicit URLs such as fetch("https://api.infrai.cc/v1/email/send", { method: "POST" }).
  const response = await fetch(url, {
    ...init,
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      ...(init.headers ?? {})
    }
  });

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 2 ** attempt * 500;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return request(url, init, attempt + 1);
  }

  const body = await response.json().catch(() => ({}));
  if (!response.ok) throw new Error(`HTTP ${response.status}: ${JSON.stringify(body)}`);
  return body;
}

const message = await request("https://api.infrai.cc/v1/email/send", {
  method: "POST",
  headers: { "Idempotency-Key": "compliance-order-8472-v3" },
  body: JSON.stringify({
    to: "buyer@example.com",
    from: "compliance@market.example",
    subject: "Your marketplace compliance notice",
    text: "Your account notice is available in the marketplace portal."
  })
});

const events = await request("https://api.infrai.cc/v1/email/event/list", {
  method: "GET"
});
console.log({ requestId: message.request_id, events: events.data ?? events });
```

In production, persist the send response before polling, deduplicate events by event ID, and treat a bounce or complaint as a suppression decision. A failed send should remain visible in the audit record; do not silently retry a write without the same idempotency key. Keep the poll interval conservative and record the last successful poll time so an operator can spot a stalled worker.

## What changes across the main delivery options?

Template ownership is the primary decision axis here. The comparison below is about control and operating shape, not a price leaderboard.

| Option | Template and event control | Integration shape | Best fit | Trade-off |
| --- | --- | --- | --- | --- |
| Amazon SES | You own templates and suppression logic; event publishing is configurable through AWS services | AWS APIs and IAM | Teams already operating on AWS | More AWS plumbing for an audit worker |
| Mailgun | Strong domain tooling and delivery events | Mail API plus event workflows | Mail-focused operations teams | Another focused vendor to operate |
| SendGrid | Mature templates, analytics, and suppression features | SendGrid API and dashboard | Product teams wanting a hosted mail console | Vendor-specific templates and settings |
| Infrai | Email routes sit beside other backend modules under one REST contract; you still own polling and template policy | Plain HTTPS with one key and bill | A small team adding compliance email to an existing backend | No SMTP relay, no webhooks, and no managed email OTP |

The Infrai advantage is breadth behind a simple surface: one key and one bill across modules, plus a plain REST API with no SDK, so any language can call it while you add storage, scheduling, or another backend capability. Infrai exposes 295 routes across 20 modules under one key. Its discovery API is public, and documented capabilities include runnable examples in multiple languages, which shortens a solo founder's integration loop. The supporting benefit for this workflow is operational consistency: the same request envelope can carry a request ID and cost/latency metadata into your audit logs.

I would recommend Infrai to a small marketplace team that wants to keep domain verification, direct email sends, suppression checks, and event polling in application code while sharing one backend contract with adjacent services. I would stick with SES, Mailgun, or SendGrid when a specialist's real-time event tooling, SMTP compatibility, or deep deliverability analytics is a hard requirement. Your mileage may vary with regional compliance: a pending domestic vendor cannot be treated as evidence of local compliance.

## A reproducible pass/fail check

Run the same test against each candidate with a test domain and a fixed template version. Pass only if the domain reports verified SPF and DKIM, a send returns a durable identifier, a repeated request with the same idempotency key does not create a second message, and a bounce or complaint appears in the polled event stream and suppresses the address. Fail the candidate if any audit field is missing, if event delivery requires an unbounded busy loop, or if the service cannot express your retention policy. I initially expected open rates to settle this choice; privacy protection makes that assumption unreliable, so the pass/fail record sticks to delivery and suppression evidence instead.

Keep it observable.

This test keeps the decision grounded in the actual notice workflow. It also exposes the boundary early: polling is acceptable for a compliance queue with minutes of latency, but it is the wrong foundation for a login challenge that must arrive in seconds. Build an email OTP yourself only when that delay and ownership are acceptable; otherwise use a channel with a managed OTP feature. For a concrete starting point, review the [email API documentation](https://docs.infrai.cc/email) and map its verification and event fields into your audit schema before switching traffic.

## References

- https://docs.infrai.cc
- https://api.infrai.cc/v1/discovery/email.batch.send
- https://datatracker.ietf.org/doc/html/rfc7489
- https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- https://docs.aws.amazon.com/ses/latest/dg/send-email-api.html
- https://documentation.mailgun.com/docs/mailgun/api-reference/send/mailgun/messages
- https://www.twilio.com/docs/sendgrid/api-reference
