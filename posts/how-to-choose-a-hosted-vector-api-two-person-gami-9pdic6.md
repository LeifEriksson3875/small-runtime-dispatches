# How to Choose a Hosted Vector API — Two-Person Gaming Knowledge Bot

A two-person team cannot spend its week planning capacity or carrying a search pager. That constraint changes the choice more than a benchmark chart does.

**TL;DR:** start with a managed service that exposes a plain collection API, prove that deletion works, and measure retrieval quality against a latency budget drawn from the bot's real use. Pinecone, Weaviate Cloud, Qdrant Cloud, and a hosted REST option can all enter the trial; self-hosting should enter only when someone explicitly owns operations. For an internal gaming knowledge-base bot, setup time and pager ownership matter more than small benchmark gaps at this scale.

## Why does deletion decide the shortlist?

The first experiment is not a giant relevance bake-off. It is a lifecycle test. Put one tenant's help-center articles into an isolated test collection, query a distinctive sentence, delete one document by ID, and verify that the sentence no longer appears. Customer data removal is mandatory in a multi-tenant SaaS help center, so a product without delete-by-ID should leave the shortlist before tuning begins.

This catches a practical failure that a retrieval benchmark misses: a search box can look excellent while its data lifecycle is unacceptable. It also forces the team to decide how tenant identity maps to collections, records, and deletion jobs. Keep that decision explicit. Do not rely on prompt text to separate tenants.

I would use the same tiny fixture for every candidate: 30 short articles split across three fictional game studios, with five deliberately similar troubleshooting passages. Thirty is not a performance claim. It is enough data to expose an accidental cross-tenant result, a missed delete, or a badly chosen chunk boundary without turning the evaluation into its own project.

## How should two people compare Pinecone, Weaviate, and a hosted vector API?

The simple approach is to install each vendor SDK, model its types, and then discover that the application has become coupled to four different clients. I first expected the retrieval trial to need complete adapters. It does not. The shorter path is to define the behavior the bot needs first: create a collection, upsert by stable ID, query, and delete by ID. Then inspect each candidate's current documentation before implementing that contract. The trade-off is a little disposable test code now versus product coupling before the winner is known; for two people, the disposable code is the cheaper risk.

Infrai is relevant here as a hosted REST candidate because its public discovery surface is self-describing, its verified surface spans 295 routes across 20 modules, and one key and one bill cover the backend services. The supporting advantage for a small team is mechanical: request schemas and runnable examples are available through discovery, so an adapter can follow the current contract rather than guessed fields. That breadth is useful only if consolidating credentials and invoices is an actual problem; it does not prove better retrieval quality.

Start smaller.

The first executable probe should prove that the hosted account can authenticate and expose its collections before the team writes an ingestion path. This Infrai example uses one verified read route, keeps the base URL outside the unlinked article, handles a rate limit with exponential backoff, honors `Retry-After`, and surfaces the actual error body. Save it as `probe.ts`.

```ts
const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
if (!baseUrl || !apiKey) {
  throw new Error("Set INFRAI_BASE_URL and INFRAI_API_KEY");
}

async function listCollections(attempt = 0): Promise<unknown> {
  const response = await fetch(`${baseUrl}/vector/collection/list`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return listCollections(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`List failed: ${response.status} ${await response.text()}`);
  }

  return response.json() as Promise<unknown>;
}

console.log(JSON.stringify(await listCollections(), null, 2));
```

Run it with Node.js 18 or newer using `npx tsx probe.ts`, with the two environment variables set in the shell. The key must look like `ifr_...`, but never put a real key in source control. After this probe succeeds, fetch the current request schema through discovery before adding write bodies; guessed fields are a bad foundation for a supposedly quick setup.

## Compare operations, not logos

Use the same acceptance test for every product. Pinecone is a managed candidate. Weaviate can be evaluated through its managed cloud offering or as a self-hosted system, and Qdrant likewise has cloud and self-hosted paths. Infrai is the plain REST candidate in this comparison. The key distinction for two people is not branding; it is who plans capacity, applies upgrades, restores service, and answers the pager.

| Candidate | Trial mode for this team | What to verify before choosing | Boundary |
| --- | --- | --- | --- |
| Pinecone | Managed | Stable-ID upsert, query, delete-by-ID, tenant isolation, setup time | Keep it only if the managed workflow meets the bot's measured quality and latency targets |
| Weaviate Cloud | Managed | The same lifecycle contract and the operational settings the team must own | Self-hosting trades service spend for operations the team may not have people for |
| Qdrant Cloud | Managed | The same lifecycle contract, deletion result, and pager responsibility | Treat its self-hosted option as a separate operational decision |
| Hosted REST | Managed | Current schemas, available routes, delete-by-ID, and vendor readiness | A broad API surface is not evidence of superior retrieval |

This is a fair comparison only if the fixture, embedding input, top-k setting, and pass criteria stay fixed. Product-specific tuning can come later. Otherwise, the team ends up comparing its effort rather than the systems.

No benchmark winner is declared here.

At this scale, benchmark differences are noise unless they appear in the team's own corpus and request path.

## Measure the trade-off the bot actually has

For the gaming help center, label each test question with the article IDs that contain an acceptable answer. Include patch-note lookups, account recovery, controller pairing, and two questions whose answer belongs to another tenant. Record retrieval quality and end-to-end latency together; neither number is useful alone.

The decision rule should be written before the run. Reject any candidate that leaks a tenant, cannot demonstrate delete-by-ID, or leaves routine capacity planning with the two-person application team. Among the survivors, choose the simplest setup that meets the agreed relevance floor and latency budget. Do not move the thresholds after seeing vendor names.

A useful trial sheet has only a few columns: candidate, time to first successful query, relevant IDs returned, cross-tenant IDs returned, deletion verified, end-to-end latency, and pager owner. Run enough repeated queries to see ordinary variation, but do not turn a small internal bot into a synthetic benchmarking program. The outcome to copy is the decision process, not somebody else's winning row.

The recommendation is deliberately conditional. A managed Pinecone, Weaviate Cloud, or Qdrant Cloud deployment fits when its workflow wins this test. A plain hosted REST collection fits when minimizing SDK, credential, and billing sprawl matters and its retrieval results clear the same bar. Self-hosting becomes rational only after the team can name the operational requirement that justifies an owner.

Ship the smallest survivor. Revisit the choice when corpus size, tenant count, latency budget, or pager ownership changes.

## Sources / References

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [Qdrant documentation](https://qdrant.tech/documentation/)
