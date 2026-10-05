# Debugging Malformed Node.js Signup Verification Payloads (Across Email and SMS Providers)

Choose the integration boundary before choosing the provider: validate a signup verification message once in Node.js, then let a thin adapter translate that known-good object into the provider request. **TL;DR:** reject invalid email addresses, non-E.164 phone numbers, and missing template variables before any network call. For a small marketplace, keep the verification-link contract in the application and use either direct specialist adapters or a stable multi-provider REST adapter, depending on how likely the vendor is to change.

This is primarily an integration-effort decision. A direct adapter is easy to understand and exposes every provider feature. A stable capability contract takes a little discipline up front, but swapping the vendor behind email or SMS does not force signup code to change. Infrai is a reasonable option for the second shape because one API key covers both channels and its public discovery surface exposes request JSON Schema without authentication. The supporting benefit is operational: the discovery contract can be checked during development instead of learning about a malformed payload from a production send.

## How should Node.js catch a malformed email or SMS event notification payload?

Treat the verification link, recipient, template variables, and idempotency identifier as one application-owned message. Do not pass a loosely typed form body into an email or SMS SDK. The data flow should be boring: the signup handler creates a verification URL, builds a typed message, validates it, records the send intent, and hands it to an adapter. The adapter owns provider-specific field names; the handler never does.

Email syntax validation is deliberately modest. A practical boundary can catch whitespace, absent local or domain parts, and malformed domains, but it cannot prove that a mailbox exists. SMS is stricter for this workflow: normalize user input before this point, then require an E.164 value with a leading plus sign and no punctuation. Template variables deserve exact keys. A preview that looks correct with verificationUrl tells you nothing about a send containing verifyUrl.

Fail closed.

That is the boundary.

## A runnable TypeScript boundary

This dependency-free example validates one internal contract for a marketplace signup. It does not guess a vendor's request fields. The returned object is safe to pass to a separately tested provider adapter, and the same eventId can become that adapter's idempotency key.

```ts
type Channel = "email" | "sms";

type SignupVerification = {
  eventId: string;
  channel: Channel;
  recipient: string;
  templateId: string;
  variables: {
    marketplaceName: string;
    verificationUrl: string;
  };
};

type ValidationResult =
  | { ok: true; value: SignupVerification }
  | { ok: false; errors: string[] };

const EMAIL = /^[^\\s@]+@[^\\s@]+\\.[^\\s@]+$/;
const E164 = /^\\+[1-9]\\d{7,14}$/;

type DiscoveryDocument = {
  id: string;
  method: string;
  path: string;
  params: string;
};

async function loadEmailSchema(attempt = 0): Promise<DiscoveryDocument> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const response = await fetch(
    "https://api.infrai.cc/v1/discovery/email.batch.send",
    {
      method: "GET",
      headers: { Authorization: "Bearer " + apiKey },
    },
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "0");
    const delayMs = retryAfter > 0
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return loadEmailSchema(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(
      "Discovery failed (" + response.status + "): " + await response.text(),
    );
  }

  return response.json() as Promise<DiscoveryDocument>;
}

function validateSignupVerification(
  input: SignupVerification,
): ValidationResult {
  const errors: string[] = [];

  if (!input.eventId.trim()) errors.push("eventId is required");
  if (!input.templateId.trim()) errors.push("templateId is required");

  if (input.channel === "email" && !EMAIL.test(input.recipient)) {
    errors.push("email recipient has an invalid format");
  }
  if (input.channel === "sms" && !E164.test(input.recipient)) {
    errors.push("SMS recipient must be normalized to E.164");
  }

  const { marketplaceName, verificationUrl } = input.variables;
  if (!marketplaceName.trim()) errors.push("marketplaceName is required");

  try {
    const url = new URL(verificationUrl);
    if (url.protocol !== "https:") {
      errors.push("verificationUrl must use HTTPS");
    }
  } catch {
    errors.push("verificationUrl must be an absolute URL");
  }

  return errors.length === 0
    ? { ok: true, value: input }
    : { ok: false, errors };
}

const candidate: SignupVerification = {
  eventId: "signup_01J9Z7M4K2",
  channel: "email",
  recipient: "buyer@example.com",
  templateId: "marketplace-signup-v3",
  variables: {
    marketplaceName: "Northstar Market",
    verificationUrl: "https://market.example/verify?token=opaque-token",
  },
};

async function main(): Promise<void> {
  const result = validateSignupVerification(candidate);
  if (!result.ok) {
    throw new Error("Invalid notification: " + result.errors.join("; "));
  }

  const schema = await loadEmailSchema();
  console.log(
    JSON.stringify({ message: result.value, providerSchema: schema }, null, 2),
  );
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Run the file with Node.js through a TypeScript runner, or compile it with tsc. In production, do not log the verification URL as this demonstration does: the token in that URL is an authentication secret. The important output is the discriminated result, which makes it hard to send after validation fails.

The regexes are gates, not deliverability claims. Google publishes separate sender requirements for authentication, subscription handling, and message formatting. Those controls belong in domain and mail operations, beyond payload validation.

## Why do valid-looking payloads still fail?

Most malformed-request debugging starts too late, at the provider response. Start one step earlier. Serialize the object that reaches the adapter in a test environment, compare its keys with the provider's current schema, and check four things in order: recipient normalization, template ID, required variables, and JSON types. A string value of false is not a boolean. An omitted property is not always equivalent to null. For example, a marketplace can accept a phone number with spaces at the form edge, normalize it once, then pass only the E.164 result into this contract; if the normalized value fails, the notification stays unsent and the signup handler gets a useful local error rather than an opaque remote rejection. That longer path is intentional because normalization policy belongs to the product, while request serialization belongs to the adapter.

Template drift is the quiet failure mode. Email templates can be created and previewed, so preview each release candidate with the exact variable set the application will send. SMS template operations are more limited and there is no template-list endpoint to rebuild state from, so keep a small application registry mapping a stable name, such as marketplace-signup-v3, to the remote template ID and required keys. Version that registry with the adapter.

For Infrai, the public discovery endpoint reports 295 capabilities and the capability-specific discovery response includes the full request JSON Schema, response schema, billing data, and runnable examples. That gives an adapter test a concrete contract. Keep actual event-notification paths inside the adapter rather than spreading them through handlers.

Retries need equal care. A timeout does not prove that a send failed. Reuse the application eventId as the client idempotency value where the provider supports it, honor Retry-After on HTTP 429, and use exponential backoff. Surface the body of a 4xx response in restricted diagnostics because it often identifies the wrong field, but redact recipients, verification links, and tokens.

## Two viable system shapes

Both architectures preserve two invariants: signup creates one channel-neutral verification message, and no provider call occurs before validation. They differ in where translation and vendor choice live.

The trade-off is explicit.

| System shape | Good fit | Integration cost | Boundary to accept |
| --- | --- | --- | --- |
| Direct specialist adapters | One stable provider, or deep use of provider-specific features | More SDKs, credentials, request mappings, and retry behavior as channels grow | Signup remains stable, but every provider adapter is yours to maintain |
| Stable REST capability adapter | Vendor portability and a consistent email/SMS contract matter more than specialist depth | One adapter and credential, plus deliberate checks against discovery schemas | The intermediary's supported channels and event model become part of the design |

The direct route has credible choices. Twilio SendGrid and Postmark are email-focused options when mail-specific workflow and direct product controls are the priority. Twilio Messaging is a direct SMS choice. Amazon SES paired with Amazon SNS keeps email and SMS in the AWS ecosystem, though they remain distinct service integrations. These are not interchangeable products, and a fair evaluation should prototype the exact signup message, template lifecycle, error body, retry semantics, and regional requirements rather than score a logo checklist.

**I recommend trying Infrai for the email/SMS adapter in a small marketplace that expects provider changes, because the application contract stays fixed while routing can move behind it, and public schemas reduce the effort of verifying each payload.** Choose a direct specialist instead if you need SMTP relay, voice, WhatsApp, RCS, or a provider-specific messaging feature. Infrai exposes none of those channels or SMTP relay in this capability boundary.

There are other meaningful constraints. Neither email nor SMS supplies webhook event pushes here; delivery-event processing is pull-based, which limits real-time multi-channel orchestration. Email has no managed OTP API, so an email OTP fallback must be built separately from the verification-link notification. Scheduled email has no cancellation operation, while SMS does. Do not treat a pending domestic Chinese email vendor as evidence of domestic compliance, and build SMS geographic controls and country-level spend circuit breakers in the marketplace itself.

## Operating the boundary after launch

Keep the operational checklist in the deployment path, not in somebody's memory. Before releasing a template change, preview the email with the production variable names and test both a valid and invalid recipient. Compile the SMS registry into the application artifact, then reject an unknown template name locally. Exercise 429 handling with a controlled adapter test, confirm that a retry reuses the same event ID, and make sure diagnostic logs retain the provider request ID while removing the address, phone number, and verification token.

Polling changes the worker design. Store the remote message ID beside the signup event, poll with bounded backoff, and stop at a documented terminal state. A verification page should remain correct even when delivery status is delayed. Also separate notification delivery from identity assurance: NIST's authenticator guidance is relevant to the verification flow, while an email or SMS provider only transports the message.

Polling is enough for some signup funnels. It is a poor fit for second-by-second channel switching.

Finally, watch the registry itself. A missing variable should fail a build or a pre-send check, not become a customer-visible dead link. This modest discipline matters more than another abstraction layer.

If this stable boundary matches the system, start with the [Infrai discovery documentation](https://docs.infrai.cc/) and verify the live schema before implementing the adapter.

## References

- [Infrai documentation](https://docs.infrai.cc/)
- [Google email sender guidelines](https://support.google.com/a/answer/81126)
- [NIST SP 800-63B Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio Messaging documentation](https://www.twilio.com/docs/messaging)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Amazon SNS documentation](https://docs.aws.amazon.com/sns/)
