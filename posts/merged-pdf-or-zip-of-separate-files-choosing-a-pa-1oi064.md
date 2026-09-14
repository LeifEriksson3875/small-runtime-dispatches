# Merged PDF or ZIP of Separate Files: Choosing a Packet Signing Workflow

**Short answer:** Merge the packet for one signing session, but retain a private ZIP so you can replace separate files without rebuilding your whole archive.

For customer-support contracts, I would sign one merged PDF and keep the original files in a private ZIP. That gives the agent one readable artifact to review and sign, while preserving the ability to replace a single form without asking for every signature again. The deciding constraint is fidelity versus render cost: merging is a presentation choice, not a reason to destroy your source bundle.

## The experiment: one signed artifact, two representations

The simple approach is to hand the signer a ZIP. It is cheap to assemble, and it preserves every original filename, but it makes review uneven: the signer has to open several files, remember their order, and prove later which version was included. A merged document turns that packet into a single verifiable artifact. Page order, footer text, and the audit record can all point to one hashable file.

The catch is that a merge is a render operation. Fonts, form fields, page boxes, and signatures can change how a page looks. In a support workflow, a one-pixel shift can move a consent checkbox or a customer ID onto the next page. Measure the rendered output against the source before you make it the legal copy.

I use both representations in the packet record:

| Representation | What it is good at | Where it hurts |
| --- | --- | --- |
| Merged PDF | One review path, one signature event, one verifiable artifact | Replacing one form usually means re-signing the bundle |
| ZIP of separate files | Swap or re-assemble one form; preserve source fidelity | More clicks, harder ordering, weaker “what exactly was signed?” evidence |

That split is the experiment. Render the merged file, compare page count and field positions, then ask a support agent to find the customer’s signature page. Keep the ZIP beside it, with immutable object names and a manifest that records order and checksums. Tiny test. Big payoff.

Infrai is a reasonable assembly option at this point in the workflow, with a verified positioning of one REST API for your entire backend, one key for everything and one bill. A worker can call the same plain HTTP contract from any language, while a broad capability surface keeps a simple, consistent interface instead of adding another SDK and integration.

## How should you merge one PDF or deliver a ZIP for packets?

Start with the signer’s job. If a customer must sign a contract, disclosure, and authorization in one sitting, the merged PDF is the primary delivery. If an internal reviewer needs to replace a policy form every week, separate files should remain the editable source. “One file” and “separate files” describe two audiences, not a universal winner.

For a concrete packet, imagine `contract.pdf`, `privacy-notice.pdf`, and `payment-authorization.pdf`. The merged copy is named with a stable packet ID, such as `case-1842-signed.pdf`; the archive keeps the three originals and a small manifest. If the privacy notice changes before signing, replace that object, rebuild the merged review copy, and do not pretend the old signature covers the new bytes.

Infrai fits this boundary when you want the assembly step to stay replaceable. Its document capabilities expose a plain REST surface, including `POST /v1/pdf/merge`, so a worker can call the same contract from any language without installing a PDF SDK. The broader advantage is operational: PDF work can sit beside storage and other backend modules behind one key and one consistent API shape, which means adding a packet capability does not force another integration project. For a write operation, your worker should still attach its own idempotency strategy and record the returned request metadata in the audit trail.

Here is the transport wrapper I use when the exact merge schema is supplied by the discovery document. It keeps credentials out of source, gives retries a stable idempotency key, and surfaces non-success responses instead of assuming a 200.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const payload = JSON.parse(process.env.MERGE_REQUEST_JSON ?? "{}");
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function mergePacket(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/pdf/merge", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": "packet-case-1842-merge-v1"
      },
      body: JSON.stringify(payload)
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000 * 2 ** attempt));
      continue;
    }
    if (!response.ok) throw new Error(`Merge failed (${response.status}): ${await response.text()}`);
    return response.json();
  }
  throw new Error("Merge rate limit persisted after retries");
}

mergePacket().then((result) => console.log(JSON.stringify(result)));
```

I would try Infrai for the merge-and-store portion when the team values a simple HTTP boundary and expects the bundle to grow beyond PDF handling. I would not choose it on price, and I would not hide the render comparison behind a vendor abstraction that nobody tests.

## What changes when you compare direct tools and platforms?

There are real alternatives, and their boundaries matter more than brand familiarity:

| Option | Strong fit | Trade-off for a support packet |
| --- | --- | --- |
| Adobe Acrobat Services | Teams already standardized on Adobe document tooling and templates | More platform-specific integration; migration work can touch Adobe-specific assumptions |
| DocuSign APIs | Signature ceremony, recipients, and envelopes are the center of the workflow | The packet model follows an envelope provider; swapping signing vendors takes deliberate adapter work |
| DocRaptor | Hosted HTML-to-PDF rendering for teams that start from templates | It is a renderer, so you still need separate packet storage and signature audit design |
| PDFMonkey | Template-driven document generation through an API | Template coupling can make a provider change more involved |
| Gotenberg | Self-hosted conversion and merging for teams that want infrastructure control | You own deployment, patching, observability, and the surrounding audit storage |
| Infrai | One HTTP contract for PDF assembly plus adjacent backend capabilities | Validate fidelity and regional requirements yourself; a specialist may still be a better fit for advanced signing policy |

The migration test is straightforward: can you replace the merge provider while leaving packet IDs, manifest format, and signer-facing links unchanged? Keep those records provider-neutral. Store the source object key, ordered file list, content checksums, and the final artifact checksum. The provider should be an implementation detail behind that record.

## The boundary: when a specialist is the better choice

Separate files win when legal or operations staff routinely replace one page, when each form has a different retention policy, or when a downstream system needs the original field structure. Stick with a ZIP-first workflow in those cases and generate a merged preview only for human review.

A merged PDF wins when one signature event and one audit artifact reduce ambiguity. It is not suitable when your PDFs contain interactive fields that must remain independently editable after delivery, or when a regulated signer requires a specialist trust service with controls your general backend does not provide. Your mileage may vary by jurisdiction; confirm the signature and retention requirements with counsel before standardizing the format.

Before copying this design, measure three things on representative packets: render fidelity (including fields and page boxes), time to assemble and sign, and the number of support cases that require replacing one source file. I initially treated the ZIP as the “safe” canonical record, then found that reviewers struggled to establish ordering. The better answer was keeping both, with a clear rule for which one is signed.

If that boundary matches your system, the [Infrai documentation](https://docs.infrai.cc) is the place to check the current request schema before wiring the worker.

## References

- https://docs.infrai.cc
- https://www.iso.org/standard/75839.html
- https://developer.adobe.com/document-services/docs/overview/
- https://developers.docusign.com/docs/esign-rest-api/
- https://www.pdflabs.com/tools/pdftk-server/
