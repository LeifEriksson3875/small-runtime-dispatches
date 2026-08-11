# Moderation Reports: Cheap Embeddings, Semantic Rerank, and an Ask-Docs Alternative

## Short answer

Short answer: for a B2B SaaS product classifying moderation reports before human review, use embeddings for broad recall, keep reranking selective, and meter both stages per tenant before comparing providers. A cost per 1M tokens is an input to that decision, not the decision itself.

The useful unit is a reviewed report that reaches the right policy or precedent, with its retrieval quality and tenant charge visible together. That catches a common failure: a shared search service looks cheap overall while one high-volume tenant quietly consumes the budget.

The flow is plain. Policy pages, prior decisions, and escalation notes are chunked and embedded into a versioned index. A new report is embedded for recall. The service takes a capped candidate set, optionally reranks it, and hands ordered evidence to the classifier or reviewer. Usage events keep indexing, refresh, query, and rerank work separate.

Ship the accounting with the retrieval path.

## How should a B2B SaaS compare embeddings, rerank, and semantic search cost by tenant?

Start with the corpus, not the pricing page. Build a fixed evaluation set from real moderation-report shapes: short abuse reports, long appeals, policy references, and reports that need an escalation rule rather than a keyword match. Label the evidence a reviewer would need. Keep chunking, metadata filters, candidate limits, and the quality threshold constant while testing an embedding choice. If relevant evidence never enters the candidate set, a reranker cannot recover it.

Then measure reranking separately. Send only recalled candidates and vary the candidate cap. A larger cap may improve ordering, but it increases candidate tokens per report. A smaller cap may protect latency while hiding recall failure. The right setting is the smallest cap that clears the same reviewer-accepted quality gate. There is no honest universal answer without the corpus and labels.

For every trial, record four denominators: indexed tokens, changed tokens, moderation reports, and reranked reports. Add candidate tokens to the last one. Cost per 1M tokens belongs beside these counts. A mostly static policy corpus has a different cost shape from a tenant whose policy and historical decisions change each day. A tenant with repetitive reports may need less reranking after a high-confidence first pass. I'm not sure a spreadsheet can predict the steady-state bill until production telemetry replaces its assumptions. Imagine a tenant that uploads a 40-page policy revision on Monday, then sends short reports all week: the refresh spike and the query stream are different operational events, so merging them into one average makes the bill harder to explain and the tuning decision worse.

Keep US and EU traffic separate in the worksheet when location, retention, or procurement rules affect the deployment path. Compare the same quality test and accounting fields in each path. Published rates cannot establish the final result when request volume, document churn, candidate length, and rerank frequency differ.

The comparison can stay vendor-neutral:

| Integration shape | Useful test | Main trade-off |
| --- | --- | --- |
| Direct API | Quality and tenant cost on the fixed corpus | More provider-specific plumbing to own |
| Unified API | One request shape across retrieval stages | A required provider-specific capability may be unavailable |
| Self-hosted service | Predictable control over deployment and data path | Operations, capacity, and model updates become the team's work |

No row wins by default. The acceptance gate and the workload decide.

## Build the meter before tuning the model

A per-tenant event should say what work happened, not merely that a search request existed. An indexing event can carry `tenantId`, `indexVersion`, `inputTokens`, and `region`; a query event can carry `queryTokens` and `candidateCount`; a rerank event can carry `candidateTokens` and `enabled`. Store the event with a request identifier so a retry does not become an untraceable second charge.

The following TypeScript models monthly refresh and query-time reranking. Rates are inputs because they must come from the current contract or billing source at evaluation time.

```ts
type TenantUsage = {
  tenantId: string;
  corpusTokens: number;
  monthlyChangedFraction: number;
  monthlyReports: number;
  rerankedFraction: number;
  candidatesPerReport: number;
  tokensPerCandidate: number;
};

type Rates = {
  embeddingUsdPerMillionTokens: number;
  rerankUsdPerMillionTokens: number;
};

const estimateTenantCost = (usage: TenantUsage, rates: Rates) => {
  if (usage.monthlyChangedFraction < 0 || usage.monthlyChangedFraction > 1) {
    throw new Error("monthlyChangedFraction must be between 0 and 1");
  }
  if (usage.rerankedFraction < 0 || usage.rerankedFraction > 1) {
    throw new Error("rerankedFraction must be between 0 and 1");
  }

  const refreshTokens =
    usage.corpusTokens * usage.monthlyChangedFraction;
  const rerankTokens =
    usage.monthlyReports *
    usage.rerankedFraction *
    usage.candidatesPerReport *
    usage.tokensPerCandidate;
  const perMillion = (tokens: number, rate: number) =>
    (tokens / 1_000_000) * rate;
  const refreshUsd = perMillion(
    refreshTokens,
    rates.embeddingUsdPerMillionTokens,
  );
  const rerankUsd = perMillion(
    rerankTokens,
    rates.rerankUsdPerMillionTokens,
  );

  return {
    tenantId: usage.tenantId,
    refreshTokens,
    rerankTokens,
    refreshUsd,
    rerankUsd,
    totalUsd: refreshUsd + rerankUsd,
  };
};
```

This is a budget estimate, not a relevance benchmark. Run it once per tenant and region, then compare it with actual usage events. The useful output is a distribution that shows which tenants need a quota, a different candidate cap, or a different refresh schedule.

Keep the first version boring.

## Where semantic retrieval fails in moderation workflows

The first failure is recall disguised as a ranking problem. A report uses local language, a policy was split at a bad boundary, or filtering removed the only relevant version. Reranking the survivors can make the list look cleaner while leaving the reviewer without governing evidence. Test filtered and unfiltered retrieval separately, and retain the index version in every evaluation record.

The second failure is evidence overload. Passing a large candidate set to a classifier raises processing work and can put contradictory policy versions beside the current rule. Limit evidence by tenant and policy version before reranking. If a reviewer needs the source passage, preserve its identifier and text span rather than logging only a similarity score.

The third failure is retry accounting. A timeout at the application boundary does not tell you whether the upstream operation ran. Count attempts and completed work as distinct fields, attach an idempotency key where the integration supports one, and reconcile usage events against billing. Don't charge twice because a queue redelivered a message; don't erase a completed operation merely because its response was lost.

The fourth failure is cost blindness in the happy path. A global monthly average can look fine while one tenant sends long reports or triggers reranking on every case. Emit tenant-level counters for query tokens, candidate tokens, embedding refreshes, rerank decisions, and accepted evidence. Alert on rate of change as well as absolute spend. Three numbers are enough to start: reports, candidate tokens, and reranked reports.

## What should the deployment contract and review loop contain?

Treat retrieval configuration as a versioned artifact. Record chunking rules, embedding configuration, index version, region, candidate cap, rerank policy, and quality labels. A changed embedding configuration means an index rebuild and a new evaluation; it should not silently mutate results used by reviewers.

Keep the classifier decision separate from the retrieval decision. The workflow should be able to say that evidence was missing, evidence conflicted, or evidence passed the retrieval gate but the classifier remained uncertain. That distinction gives reviewers a useful queue and engineering a useful failure category.

The trade-off is real. Selective reranking is not suitable when the product requires a consistent second-stage score for every report. A unified integration is not suitable when procurement requires a specific direct contract or a provider-specific capability. Stick with a direct integration when its measured retrieval result, contract, or required feature earns the additional plumbing. Use a broader abstraction when the same quality gate passes and its common interface materially reduces credential and billing administration.

My handoff checklist is one paragraph because this is operational work: verify current rates and data-handling terms for each deployment path, replay the labeled moderation set, inspect recall before rerank lift, cap candidate tokens, separate refresh from query charges, exercise retry behavior, and sample the tenant report against billing. Then set a review date. The corpus will change.

## References

- https://platform.openai.com/docs/guides/function-calling
- https://elevenlabs.io/docs
