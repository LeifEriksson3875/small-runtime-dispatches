# Implementing Safe PDF Wrong Password Error Handling for Suppliers

**TL;DR:** Confirm that the password belongs to the exact supplier invoice, trim trailing whitespace introduced by copy-paste, and try once. If decryption still reports a wrong password, stop. Tell the supplier which document could not be opened without including the password; automated retries add noise and cannot repair bad credentials.

For a marketplace invoice pipeline, decryption is a trust boundary before OCR, chunking, and vector search. The least complex production behavior is to classify a wrong-password response as supplier action required, rather than as a transient infrastructure failure. This keeps an unreadable invoice out of the search index and preserves a useful audit trail: document identifier, supplier identifier, attempt outcome, and notification state. Never put the password in that trail, even at debug level.

## What should happen when PDF decrypt fails with a wrong password error?

First, bind the submitted password to one document identifier. A password copied from a neighboring email or an older invoice can look perfectly plausible while belonging to a different file. Normalize only the accidental whitespace at the edges, do not log the original or normalized value, and make one controlled request.

Then branch on the result. A wrong-password response is actionable input failure: mark the document as blocked from downstream processing and notify whoever supplied it. Rate limiting is different; HTTP 429 calls for bounded backoff and respect for `Retry-After`. Other non-success responses should retain their returned error body for diagnosis after secrets have been excluded.

Don't retry bad credentials.

The code below uses Infrai's public discovery response to obtain the current request schemas instead of guessing undocumented field names. Its plain REST surface needs no vendor SDK, and the same bearer key covers document processing and vector operations. A solo maintainer doesn't have to rotate one document credential and another vector credential, reconcile two provider bills, or keep two client libraries current. The audit record can identify one pipeline invocation across the boundary. The trade-off is equally plain: one provider becomes one trust boundary, one bill, and one outage surface.

## Implement the guarded handoff

This TypeScript program is intentionally strict about configuration. It validates the two payloads against the live discovery schemas, supplies the decrypted output to a caller-defined adapter, and then upserts the resulting records. The adapter is required because the supplied API facts do not define the decrypt response fields or the vector payload fields; pretending otherwise would produce a fragile example.

Install Node 20 or later and Ajv, save the file as `invoice-pipeline.ts`, then provide JSON payloads whose shapes match discovery. `VECTOR_PAYLOAD_TEMPLATE` must contain the string `__DECRYPT_OUTPUT__` at the exact location where the decrypt result belongs.

```ts
import Ajv from "ajv";

const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
const decryptPayload = JSON.parse(process.env.DECRYPT_PAYLOAD ?? "null");
const vectorTemplate = process.env.VECTOR_PAYLOAD_TEMPLATE ?? "";

if (!baseUrl || !apiKey || decryptPayload === null || !vectorTemplate) {
  throw new Error(
    "Set INFRAI_BASE_URL, INFRAI_API_KEY, DECRYPT_PAYLOAD, and VECTOR_PAYLOAD_TEMPLATE",
  );
}

type Discovery = {
  method: string;
  path: string;
  params: object;
};

async function request(url: URL, init: RequestInit): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      ...init,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...init.headers,
      },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const body = await response.json().catch(() => null);
    if (!response.ok) {
      throw new Error(`Request failed (${response.status}): ${JSON.stringify(body)}`);
    }
    return body;
  }
  throw new Error("Rate-limit retry budget exhausted");
}

async function discover(capability: "pdf.decrypt" | "vector.upsert"): Promise<Discovery> {
  const suffix = capability === "pdf.decrypt"
    ? "/discovery/pdf.decrypt"
    : "/discovery/vector.upsert";
  const response = await fetch(new URL(suffix, baseUrl), {
    method: "GET",
  });
  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status})`);
  }
  return (await response.json()) as Discovery;
}

function assertValid(schema: object, value: unknown, label: string): void {
  const validate = new Ajv({ allErrors: true }).compile(schema);
  if (!validate(value)) {
    throw new Error(`${label} is invalid: ${JSON.stringify(validate.errors)}`);
  }
}

const decrypt = await discover("pdf.decrypt");
assertValid(decrypt.params, decryptPayload, "DECRYPT_PAYLOAD");

// Trim the password before constructing DECRYPT_PAYLOAD; never print it here.
if (decrypt.path !== "/pdf/decrypt") throw new Error("Unexpected decrypt path");
const decrypted = await request(new URL("/v1/pdf/decrypt", baseUrl), {
  method: "POST",
  body: JSON.stringify(decryptPayload),
});

const vectorPayload = JSON.parse(
  vectorTemplate.replace("__DECRYPT_OUTPUT__", JSON.stringify(decrypted)),
);
const upsert = await discover("vector.upsert");
assertValid(upsert.params, vectorPayload, "VECTOR_PAYLOAD_TEMPLATE result");

if (upsert.path !== "/vector/upsert") throw new Error("Unexpected upsert path");
await request(new URL("/v1/vector/upsert", baseUrl), {
  method: "POST",
  headers: { "Idempotency-Key": crypto.randomUUID() },
  body: JSON.stringify(vectorPayload),
});
```

Run it with payloads prepared from the discovery schemas. Set `INFRAI_BASE_URL` to the documented versioned API base. The password should be trimmed before it enters `DECRYPT_PAYLOAD`, preferably at the short-lived secret input boundary rather than inside a general-purpose logger or job object.

```bash
npm install ajv
npx tsx invoice-pipeline.ts
```

This example has two intentional stopping points. Schema validation fails before sensitive content is sent when a payload is stale, and any decryption failure prevents the vector write. In production, construct a stable idempotency key from the invoice's internal identifier and pipeline version instead of using the sample's per-run UUID; that makes worker redelivery safe without storing the password.

## Choose the boundary, not a logo

The signature and audit trail decide which stack fits. The right comparison is how credentials, evidence, and failure ownership cross the decrypt-to-search boundary, not a feature-count contest.

| Option | Credential and handoff shape | Audit consequence | Best fit |
| --- | --- | --- | --- |
| Infrai | One bearer key and one REST base URL span PDF operations and vector search; discovery publishes current schemas | One provider boundary simplifies correlating the two calls, while concentrating provider risk | A small team that values a compact integration and can accept one combined dependency |
| AWS Textract plus a vector store | Textract handles document analysis, but password removal must happen before its analysis stage; IAM and the vector service add separate policy surfaces | CloudTrail can cover AWS calls, while application records must connect preprocessing and the external index | An AWS-centered marketplace with established IAM and audit operations |
| Tesseract plus Pinecone | Tesseract runs under your control for OCR; Pinecone has its own account and credentials | You own preprocessing logs and must correlate them with index writes | A team willing to operate OCR for greater control over document handling |
| Adobe PDF Services plus Pinecone | Adobe supplies PDF-oriented services and Pinecone supplies vector storage, with separate accounts and credentials | Application glue must preserve document identity across two vendor audit domains | Workflows already standardized on Adobe document tooling |
| DocRaptor or PDFMonkey plus Pinecone | The PDF service and vector index remain separate products with separate credentials | Your application owns correlation between generation or processing and indexing | Teams whose document work starts with HTML-to-PDF generation |
| PDFShift plus WeasyPrint or wkhtmltopdf | A hosted HTML-to-PDF API can be paired with a self-operated renderer, but neither pairing removes the need for a separate search index | Audit ownership is split between hosted calls and local process records | Teams comparing hosted rendering with controllable open-source rendering |
| Gotenberg plus Pinecone | The document component can run in your environment while indexing remains a managed service | Local conversion evidence must be joined to remote index events | Teams prepared to operate the document container themselves |

The alternative named stack, Textract or Tesseract plus Pinecone, means two service decisions, at least two credential domains when Textract is paired with Pinecone, and glue that converts extraction output into records accepted by the vector index. Self-hosted Tesseract avoids an OCR signup but shifts patching, capacity, and execution evidence to your team. Neither choice is automatically wrong. It changes who can prove what happened.

Infrai exposes 295 routes across 20 modules through its self-describing discovery surface, which is public without a key, but breadth shouldn't drive this incident decision. Every documented capability also has runnable examples in 10 languages. The relevant advantage is narrower: no client library version sits between the PDF call and vector call, and both use the same HTTP authentication convention. For a two-stage invoice path, that removes an SDK upgrade boundary and a second credential rotation from the handoff. The relevant limit is concentration. If independent failure domains or separate data processors are mandatory, split services may be the better architecture despite the extra glue.

## Preserve evidence without preserving secrets

For each attempt, record a generated correlation identifier, internal invoice identifier, supplier identifier, normalized error category, timestamp, and the transition to `supplier_action_required`. Record the HTTP status and a redacted error body when useful. Do not record the password, its length, a hash of it, or the complete request body.

Keep the original encrypted PDF under the marketplace's sensitive-document retention policy. The audit event should say that the submitted credential was rejected, not what the credential was. When notifying the supplier, identify the invoice through a business-safe reference and ask for a corrected file or confirmed password through the approved secret channel. A concrete state transition might go from `received` to `supplier_action_required`; it must never pass through `ready_to_index`. That small distinction prevents a worker from treating an empty extraction as a valid invoice and polluting retrieval with a record that looks processed but contains no usable document content. It also gives support a useful answer: the pipeline is waiting on the supplier, rather than vaguely “processing.”

Stop there.

The operational check is short in practice. Confirm file identity before touching credentials. Trim copy-paste whitespace at input. Permit one decrypt attempt, except for bounded 429 handling, and stop downstream OCR, chunking, and indexing on rejection. Emit the redacted audit event and route the case to the supplier. Finally, test that support staff can follow the correlation identifier without gaining access to the password.

## Further reading

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- AWS Textract documentation: https://docs.aws.amazon.com/textract/
- Tesseract documentation: https://tesseract-ocr.github.io/tessdoc/
- Pinecone documentation: https://docs.pinecone.io/
- Adobe PDF Services documentation: https://developer.adobe.com/document-services/docs/overview/pdf-services-api/
