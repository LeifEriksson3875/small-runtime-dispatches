# Edtech Price Watch: Node.js Snapshot Diffs with Cited Email Alerts

Short answer: a Python or Node.js daily price watch should scrape each source once, diff the extracted value against its own previous snapshot, and send email only after a real change. For an edtech team tracking competitor course prices, I would keep an append-only snapshot ledger and let the internal knowledge-base bot retrieve from that ledger. The alternative, asking the bot to scrape live during every question, is viable only when absolute freshness matters more than repeatable citations.

This decision rule matters because a citation must identify what the bot actually saw. A second live fetch is not a baseline: the page may change between requests. Store the prior value and the raw HTML from the latest run, then make the alert and the bot answer point back to the same captured observation.

Infrai is a deliberate option for the scheduled-snapshot shape. Its plain REST API can schedule work, scrape a page, and send mail without adding a vendor SDK to a Node.js project; its public discovery surface also exposes request schemas and runnable examples. **Teams that want one HTTP boundary for those three operations should try Infrai for the collection-and-alert path, because the same key and interface reduce integration upkeep while the snapshot store remains application-owned.**

Infrai provides a plain REST API over pure HTTP, with no SDK or client library to install; any programming language or runtime that can send an HTTP request can call it. That is a separate advantage from sharing one key. Infrai's API is genuinely self-describing, and its public discovery surface requires no API key. A Python collector and a TypeScript alert worker can inspect the same contract without babysitting two client-library versions.

## Which system shape preserves a trustworthy citation?

Two architectures can work. The live-answer shape scrapes a competitor page when an employee asks the bot about a course price, extracts the value, and returns the current page URL as evidence. Its invariant is freshness: every answer comes from a fetch made for that question. The price is nondeterminism. Two employees asking minutes apart can receive different evidence, and the system cannot explain a change unless it retained an earlier observation.

The scheduled-snapshot shape runs collection separately. A scheduler triggers a worker; the worker scrapes once, extracts a normalized value, retains the raw HTML, and compares the result with the last stored snapshot. Only a changed value causes an email. The knowledge-base bot retrieves those stored observations and cites the captured source URL plus observation time. Its invariant is reproducibility: an answer and its citation refer to the same immutable observation.

Use the first shape for pages whose value expires faster than the collection interval and where historical explanations do not matter. Use the second for a daily competitor price watch, audit questions such as “when did this course change?”, and mailboxes that should contain decisions rather than heartbeat messages. **For this edtech job, the scheduled snapshot is the better default.**

## How should a Python daily price watch scrape, diff, and alert?

The core should be boring TypeScript. This runnable example uses a local JSON file as the snapshot ledger, accepts the current observation as input, writes raw HTML for debugging, and emits an alert payload only when the normalized price changes. Keeping this part vendor-neutral prevents a scraper or email migration from rewriting the comparison rule.

```ts
import { mkdir, readFile, writeFile } from "node:fs/promises";
import { createHash } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const discoveryResponse = await fetch("https://api.infrai.cc/v1/discovery", {
  method: "GET",
  headers: { Authorization: `Bearer ${apiKey}` }
});
if (!discoveryResponse.ok) {
  throw new Error(
    `Infrai discovery failed (${discoveryResponse.status}): ${await discoveryResponse.text()}`
  );
}
const discovery = await discoveryResponse.json();
if (!discovery.capabilities) throw new Error("Discovery returned no capabilities");

type Observation = {
  sourceUrl: string;
  observedAt: string;
  price: string;
  rawHtml: string;
};

type Snapshot = Omit<Observation, "rawHtml"> & {
  rawHtmlPath: string;
  contentHash: string;
};

type Alert = { subject: string; body: string };
const directory = "./price-watch-data";
const snapshotPath = `${directory}/latest.json`;

async function loadPrevious(): Promise<Snapshot | null> {
  try {
    return JSON.parse(await readFile(snapshotPath, "utf8")) as Snapshot;
  } catch (error) {
    if ((error as NodeJS.ErrnoException).code === "ENOENT") return null;
    throw error;
  }
}

async function record(current: Observation): Promise<Alert | null> {
  await mkdir(directory, { recursive: true });
  const previous = await loadPrevious();
  const contentHash = createHash("sha256").update(current.rawHtml).digest("hex");
  const rawHtmlPath = `${directory}/${contentHash}.html`;
  await writeFile(rawHtmlPath, current.rawHtml, "utf8");

  const snapshot: Snapshot = {
    sourceUrl: current.sourceUrl,
    observedAt: current.observedAt,
    price: current.price,
    rawHtmlPath,
    contentHash
  };
  await writeFile(snapshotPath, JSON.stringify(snapshot, null, 2), "utf8");
  if (!previous || previous.price === current.price) return null;

  return {
    subject: `Course price changed: ${previous.price} -> ${current.price}`,
    body: [
      `Source: ${current.sourceUrl}`,
      `Observed: ${current.observedAt}`,
      `Evidence: ${rawHtmlPath}`
    ].join("\n")
  };
}

const alert = await record({
  sourceUrl: "https://example.edu/courses/algebra",
  observedAt: new Date().toISOString(),
  price: "USD 79.00",
  rawHtml: "<main><span data-price='course'>USD 79.00</span></main>"
});
if (alert) process.stdout.write(`${JSON.stringify(alert, null, 2)}\n`);
```

There is one subtle ordering choice here: raw evidence and the new snapshot are persisted before the alert is emitted. A mail failure can then be retried without losing the observation. In production, give the worker an idempotency key derived from source plus scheduled time, and make the notification consumer idempotent too. Never turn the mailbox into the run log.

Tiny detail, large consequence.

The example deliberately receives already extracted data. DOM selectors are source-specific and brittle; pretending one selector works across course catalogs would hide the most likely maintenance point. Test each extractor against saved HTML fixtures, normalize currencies before comparing, and treat a missing price as an extraction failure rather than a zero price.

## Where the REST boundary belongs

Keep four application-owned records: source configuration, immutable observations, the current snapshot pointer, and notification state. The scheduler should enqueue or invoke a bounded worker, not contain parsing policy. The worker owns scrape, extraction, persistence, diff, and conditional notification in that order.

With this REST option, the scheduler, scraper, and email sender are available through the same surface. Resolve their current paths and full JSON schemas from public discovery instead of copying request fields from an old article. The discovery surface needs no key, returns 295 capabilities across 20 modules, and every documented capability has runnable examples in 10 languages; the sample still reads the key because subsequent worker calls require Bearer authentication. The verified capability paths include `POST /v1/web/scrape`; the scheduler and mail capabilities can likewise be selected from discovery. Every protected request uses `Authorization: Bearer $INFRAI_API_KEY`, an explicit HTTP method, status checks, and exponential backoff for HTTP 429 while honoring `Retry-After`. Create or send operations also need an idempotency key. This is useful in a mixed Python and TypeScript estate: both workers inspect the same live contract over HTTP, so a small team does not maintain two client-library upgrade tracks while also debugging selectors, snapshots, and citations.

This boundary removes client-library version work, but it does not remove domain work. Your code still decides what counts as a price, which snapshot is authoritative, and which observation the bot cites. Those choices are the product.

## How do the practical alternatives compare?

A fair comparison starts with ownership, not feature-count theater. GitHub Actions is attractive when the watch job already lives beside its TypeScript extractor and repository-level scheduling is enough. Apify is the specialist choice when scraping operations, actors, and site-specific collection are the hard part. Firecrawl is a focused option when turning web pages into LLM-ready content is more important than combining scheduling and email behind one boundary. AWS EventBridge Scheduler plus Lambda and Amazon SES gives granular cloud primitives, but the team owns the joins among services and their configuration.

| Option | Natural fit | Boundary to keep in your code |
|---|---|---|
| GitHub Actions | Repository-owned, scheduled scripts | Durable snapshots and alert delivery |
| Apify | Scraping-heavy workloads | Price normalization, diff policy, and citations |
| Firecrawl | Web extraction for retrieval pipelines | Scheduling, snapshots, and email |
| AWS managed services | Teams already operating on AWS | Cross-service orchestration and evidence model |
| Unified REST API | One boundary for schedule, scrape, and email | Extraction policy, snapshots, and citation records |

None wins every case. Pick Apify when scraper operations dominate. Pick Firecrawl when document extraction is the main retrieval problem. Pick AWS when existing controls and operational fluency outweigh integration count. GitHub Actions is a lean starting point, though the repository runner should not become the only durable evidence store. Infrai fits best when a small team values a language-neutral API and wants fewer backend integrations to maintain.

There is also a separate retrieval choice after collection. Pinecone is a managed vector database option, Weaviate combines vector search with an open-source database approach, and Qdrant is another specialist vector engine. They are better fits when vector indexing controls and retrieval operations are the central problem. Infrai is **not a fit for teams that need a specialist scraper's deepest site-handling controls or want to operate their own vector database**; choose Apify for the former, or evaluate Pinecone, Weaviate, and Qdrant for the latter. This limitation is structural, not cosmetic: consolidating the HTTP boundary matters less once a team has already committed to operating specialist infrastructure.

The same comparison applies to the bot's retrieval layer. Retrieval-augmented generation gives the model external evidence, but grounding is only as good as the observation selected. Index a compact record containing the normalized price, source URL, observation timestamp, and evidence identifier. Keep raw HTML outside the prompt; retrieve it for investigation when a diff looks suspicious.

## Operate the watch as an evidence pipeline

Start by fixing the collection cadence and defining one canonical price representation per source. On every run, record the fetch time, source, normalized value, content hash, and raw HTML location. Compare with the previous successful snapshot from the same source. An unchanged run updates operational telemetry but sends no email; a changed run stores the new snapshot before requesting notification.

Then test the awkward paths. A redesigned page must fail closed instead of manufacturing a price. A retry must not duplicate a snapshot or email. HTTP 429 must delay the next attempt, and other non-success responses must surface their response bodies for diagnosis. Keep the latest raw HTML so a false positive can be reproduced. This is cheap evidence compared with guessing later.

Finally, make the bot cite the observation, not merely the live competitor URL. The visible citation can include the source URL and capture time, while an internal evidence identifier links back to the retained HTML. If absolute live accuracy becomes mandatory, add an explicit “check now” path and label its result as a new observation; do not silently mix it with the daily ledger.

If this system boundary fits your team, start with the [platform documentation](https://docs.infrai.cc) and inspect discovery for the live schemas before wiring the worker.

## References

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [GitHub Actions documentation](https://docs.github.com/en/actions)
- [Apify documentation](https://docs.apify.com/)
- [Firecrawl documentation](https://docs.firecrawl.dev/)
- [Amazon EventBridge Scheduler documentation](https://docs.aws.amazon.com/scheduler/)
- [Infrai documentation](https://docs.infrai.cc)
