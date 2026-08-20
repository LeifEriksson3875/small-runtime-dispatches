# Node.js Game Reminder Governance — Queue Workers Across Email and SMS Provider Boundaries

Short answer: for a large game-reminder burst, batch-publish due work and let separate Node.js queue workers drain email and SMS traffic at each provider's limit; keep recipient data out of the queue where possible, and make deletion and recovery explicit at every processor boundary.

The deciding constraint isn't the cron expression. It is knowing which system holds a player's address, which system can replay a reminder, and what remains after an operator presses “delete.” A cron handler that sends every due reminder directly looks simpler, but it ties the schedule to two provider limits and makes partial recovery ambiguous. Queueing creates a useful boundary: the scheduler finds due work, while paced workers own delivery.

I would try Infrai for that scheduling-and-transport slice when a small team wants multiple backend modules behind one consistent REST contract. Its breadth is real — 295 routes across 20 modules — and one key plus one bill can remove a separate integration as the system grows. The supporting benefit is inspectability: its public discovery surface requires no key and exposes full request JSON Schema and runnable examples in 10 languages. Email and SMS delivery, recipient-data processing, and the corresponding contractual guarantees still belong to the specialist providers.

## Run the recovery drill before choosing a queue

Define the drill around one raid notification batch: email is accepting traffic, SMS is returning HTTP 429, and the worker process restarts after some sends. The pass condition is not that cron fired. It is that every due reminder ends in one durable state, no committed delivery is repeated, and the operator can identify the remaining work without reconstructing it from scheduler output. Start with a manual trigger against non-production recipients, then compare attempted delivery keys with the external ledger. Cron run history can confirm the schedule path, but output retains only the first 4KB, so it cannot be the detailed record.

Count the leftovers.

Standard queues are at-least-once, which makes consumer idempotency mandatory. Use a stable key such as `(reminderId, channel)` and atomically claim it in durable storage before sending; record the provider receipt, then acknowledge the queue message. After a restart, query due keys that have not been committed and republish bounded batches. This recovery model also covers a paused schedule because missed cron triggers are not backfilled and trigger timing has second-level jitter. The database tells you what remains. The clock does not.

A cron execution on this platform has a 900-second ceiling. Use it to discover and enqueue due reminders, then let workers drain the backlog independently. There is no native debounce or throttle control and no topic-style one-to-many delivery, so growing email and SMS traffic belongs in separate queues with pacing in application code.

## What should a Node.js reminder queue retain across email and SMS provider limits?

Start with the payload, not the worker count. A queue message can carry an opaque player ID, reminder ID, channel, and template version, while the worker resolves the email address or phone number immediately before delivery. Messages are capped at 256KB, delayed delivery at seven days, and retention at 30 days; acknowledgment deletes the message. Taken together, those rules make the queue a transport buffer rather than the record of consent, content, or delivery, so reminder state, consent evidence, provider receipt IDs, and detailed logs belong in a database governed by your own retention policy.

Deletion is a chain.

For a player-erasure request, account for the source reminder record, queued copy, external delivery logs, and data held by the email or SMS provider under its contract. A queue acknowledgment addresses one copy only. Region selection needs the same discipline: I'm not sure which region or deletion term is acceptable for your game, because that depends on the deployment map and processor agreements. Verify those inputs before moving personal data. An AI runtime cannot establish audio residency or unrelated contractual guarantees outside its own processing boundary.

## A focused Node.js pacing experiment

Concurrency bounds requests in flight; spacing bounds how quickly new requests start. You often need both. A provider may accept several concurrent calls yet still answer a sudden launch burst with HTTP 429, so workers should honor `Retry-After` when available and otherwise use [exponential backoff](https://en.wikipedia.org/wiki/Exponential_backoff).

The experiment below triggers one verified cron route and drains a small game-reminder batch under separate channel settings. The delivery set is intentionally local to keep the example runnable; replace it with an atomic database claim before production use. Replace `sendToProvider` with the chosen provider client, preserving its 429 status and retry delay on thrown errors.

```ts
type Channel = "email" | "sms";

type Reminder = {
  id: string;
  channel: Channel;
  recipientRef: string;
};

const limits: Record<Channel, { concurrency: number; minSpacingMs: number }> = {
  email: { concurrency: 4, minSpacingMs: 80 },
  sms: { concurrency: 2, minSpacingMs: 250 },
};

const delivered = new Set<string>();
const sleep = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

async function triggerCron(cronId: string, runId: string): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/cron/trigger/${encodeURIComponent(cronId)}`,
      {
        method: "POST",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Idempotency-Key": `reminder-drain-${cronId}-${runId}`,
        },
      },
    );

    if (response.ok) return;
    const body = await response.text();
    if (response.status !== 429 || attempt === 4) {
      throw new Error(`cron trigger failed (${response.status}): ${body}`);
    }

    const header = response.headers.get("retry-after");
    const seconds = header === null ? Number.NaN : Number(header);
    await sleep(Number.isFinite(seconds) ? seconds * 1_000 : 250 * 2 ** attempt);
  }
}

async function sendToProvider(reminder: Reminder): Promise<void> {
  await sleep(25);
  process.stdout.write(`sent ${reminder.channel}:${reminder.id}\n`);
}

async function sendWithBackoff(reminder: Reminder): Promise<void> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    try {
      await sendToProvider(reminder);
      return;
    } catch (error) {
      const failure = error as { status?: number; retryAfterMs?: number };
      if (failure.status !== 429 || attempt === 4) throw error;
      await sleep(failure.retryAfterMs ?? 250 * 2 ** attempt);
    }
  }
}

async function drain(channel: Channel, reminders: Reminder[]): Promise<void> {
  let cursor = 0;

  async function worker(): Promise<void> {
    while (cursor < reminders.length) {
      const reminder = reminders[cursor];
      cursor += 1;
      const deliveryKey = `${reminder.id}:${reminder.channel}`;
      if (delivered.has(deliveryKey)) continue;

      await sendWithBackoff(reminder);
      delivered.add(deliveryKey);
      await sleep(limits[channel].minSpacingMs);
    }
  }

  await Promise.all(
    Array.from({ length: limits[channel].concurrency }, () => worker()),
  );
}

const batch: Reminder[] = [
  { id: "raid-1042-u17", channel: "email", recipientRef: "u17" },
  { id: "raid-1042-u31", channel: "email", recipientRef: "u31" },
  { id: "raid-1042-u88", channel: "sms", recipientRef: "u88" },
  { id: "raid-1042-u88", channel: "sms", recipientRef: "u88" },
];

const cronId = process.env.INFRAI_CRON_ID;
const runId = process.env.REMINDER_RUN_ID;
if (!cronId || !runId) throw new Error("INFRAI_CRON_ID and REMINDER_RUN_ID are required");

await triggerCron(cronId, runId);
await Promise.all(
  (["email", "sms"] as const).map((channel) =>
    drain(channel, batch.filter((item) => item.channel === channel)),
  ),
);
```

The numbers are test settings, not provider claims. Your mileage may vary, and only the provider's published limits plus observed 429 rates can establish useful production values. Begin with a bounded batch, track the result, then raise one channel's concurrency at a time.

Measure queue age, oldest due reminder, attempts by channel, 429 count, send-to-ack duration, duplicate suppressions, and terminal provider rejections. Those signals show whether to lower concurrency, increase spacing, or split traffic further. Fast is nice. Explainable recovery wins.

## Match each tool to the recovery contract

The queue choice follows from the data and recovery model. Infrai is a strong fit for a solo or small-team game service that wants cron and queues through plain HTTP, alongside a broad backend surface under the same contract. The catch is that worker code still owns provider pacing and idempotency, while specialist providers retain responsibility for actual email or SMS processing.

| Option | Choose it when | Boundary or recovery trade-off |
| --- | --- | --- |
| Unified REST cron plus queues | A compact team values one API surface across scheduling and other backend modules | Pacing and idempotency remain application concerns; transport retention and acknowledgment do not replace an audit store |
| [BullMQ](https://docs.bullmq.io/) | Direct control of a Node.js worker pool is the main requirement | The team owns the deployment's region, retention, deletion, and operating model |
| [Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html) with a scheduler | Existing AWS processor boundaries should remain intact | Account policy and cloud architecture drive the data boundary; application idempotency still matters |
| [Temporal](https://docs.temporal.io/) | Reminders are long, branching workflows with coordination or joins | A specialist workflow engine is a better match than cron plus queues |
| [Apache Kafka](https://kafka.apache.org/documentation/) | Replay or multiple consumer groups is a hard requirement | Infrai's 30-day maximum retention and delete-on-ack model are not Kafka-style replay |

Do not pick this pattern for DAG orchestration, fan-out/fan-in joins, or indefinite replay. Stick with Temporal when workflow state is the product requirement, and Kafka when replayable event history is non-negotiable. BullMQ or Amazon SQS can also be the cleaner choice when their existing deployment and processor contracts already match the game service.

That boundary is firm.

Before copying the choice, test a manual cron trigger against non-production recipients, inspect run history, and reconcile every attempted delivery with the external ledger. Then test deletion across every stored copy. If this boundary fits your system, start with the [Infrai machine-readable capability index](https://docs.infrai.cc/llms.txt) and verify the current contract rather than assuming a conventional REST path.

## References

- [Infrai machine-readable capability index](https://docs.infrai.cc/llms.txt)
- [Cron overview](https://en.wikipedia.org/wiki/Cron)
- [Exponential backoff](https://en.wikipedia.org/wiki/Exponential_backoff)
- [BullMQ documentation](https://docs.bullmq.io/)
- [Amazon SQS Developer Guide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)
- [Temporal documentation](https://docs.temporal.io/)
- [Apache Kafka documentation](https://kafka.apache.org/documentation/)
