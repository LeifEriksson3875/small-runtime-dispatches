# Healthtech User Reminders with Delayed Queue Messages — Seven-Day Limits and DLQs

A healthtech reminder must outlive the web request that created it, and the delivery deadline matters more than API elegance. **Short answer:** publish one delayed queue message per reminder when the due time is no more than seven days away; keep later reminders in the database and let a cron promoter publish them once they enter that window.

That split removes frequent database polling from the common, short-horizon path. It does not create exactly-once delivery. Standard queues are at-least-once, so the notification worker must be idempotent, retry temporary provider failures such as HTTP 429 with backoff, and move exhausted deliveries to a dead-letter queue (DLQ) with an explicit redrive path.

The evaluation constraint is therefore simple: a retry may repeat a delivery attempt, but it must never create a second patient reminder. Keep protected reminder content in the application database and put only a small identifier plus routing metadata in the message; a queue message cannot exceed 256KB.

## Can a delayed queue message retry a Node.js user reminder notification?

It shouldn't publish a message with an eight-day delay. The maximum delay is 604,800 seconds, so the scheduling decision has two branches. A reminder due inside that boundary goes directly to the delayed queue. A reminder farther out stays in a `scheduled` database state until a cron-triggered promoter finds it within the next seven days and publishes it.

Seven days is a hard boundary.

The promoter is deliberately boring. It claims eligible rows transactionally, derives a stable message key from the reminder ID, publishes each claimed item, and records that transition. If the promoter is interrupted, another run can reclaim work, while the stable key and state transition prevent a retry from turning one reminder into two logical jobs. Cron does not backfill triggers missed while paused, and trigger timing can have seconds of jitter, so this design needs a promotion margin rather than a boundary measured to the exact second. For a daily reminder, promoting several hours before the seven-day edge is easier to reason about than betting on a precise cron tick.

Keep the cron task short. A cron execution is capped at 900 seconds; if promotion or cleanup can run longer, cron should enqueue bounded work and workers should consume it. This is the same separation that keeps the original web request short: the clock decides *when work becomes eligible*, while workers own the work itself.

No request stays open.

This is also why a single daily database poll is a useful fallback scheduler rather than a defeat. It handles the long horizon only. Publishing one delayed message per near-term reminder still avoids repeatedly scanning every reminder that is already inside the queue's delivery window.

## Operational cost lives in the adapter

The following example keeps the publish body behind a small adapter because its fields must come from the provider's live discovery schema, not from a guessed REST convention. Set `QUEUE_PUBLISH_BODY_JSON` to a request body validated against that schema. The adapter then makes the real publish call with Bearer authentication, an explicit method, an idempotency key, status checks, and bounded handling for HTTP 429. The application logic makes the seven-day choice, keeps messages small, and treats duplicate consumption as success. The two database methods must be atomic in a production implementation.

```ts
const MAX_DELAY_SECONDS = 7 * 24 * 60 * 60;

function requiredEnv(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return value;
}

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  const retryAfterSeconds = retryAfter === null ? Number.NaN : Number(retryAfter);
  return Number.isFinite(retryAfterSeconds)
    ? retryAfterSeconds * 1000
    : Math.min(500 * 2 ** attempt, 8_000);
}

async function publishWithInfrai(
  publishBody: unknown,
  idempotencyKey: string,
): Promise<void> {
  const apiKey = requiredEnv("INFRAI_API_KEY");
  const apiBaseUrl = requiredEnv("INFRAI_API_BASE_URL").replace(/\/$/, "");
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`${apiBaseUrl}/queue/publish`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(publishBody),
    });

    if (response.ok) return;
    const responseBody = await response.text();
    if (response.status !== 429 || attempt === 4) {
      throw new Error(`Queue publish failed (${response.status}): ${responseBody}`);
    }
    await new Promise((resolve) =>
      setTimeout(resolve, retryDelayMs(response, attempt)),
    );
  }
}

type Reminder = {
  id: string;
  patientId: string;
  dueAt: Date;
  channel: "sms" | "email";
  status: "scheduled" | "queued" | "sent";
};

type ReminderMessage = {
  reminderId: string;
  channel: Reminder["channel"];
};

interface ReminderStore {
  saveScheduled(reminder: Reminder): Promise<void>;
  claimForPromotion(cutoff: Date): Promise<Reminder[]>;
  markQueued(id: string): Promise<void>;
  get(id: string): Promise<Reminder>;
  markSentOnce(id: string): Promise<boolean>;
}

interface DelayedQueue {
  publish(input: {
    body: ReminderMessage;
    delaySeconds: number;
    idempotencyKey: string;
  }): Promise<void>;
  ack(receipt: string): Promise<void>;
  retryOrDeadLetter(receipt: string, reason: string): Promise<void>;
}

interface NotificationProvider {
  send(reminder: Reminder): Promise<void>;
}

function secondsUntil(dueAt: Date, now: Date): number {
  return Math.max(0, Math.floor((dueAt.getTime() - now.getTime()) / 1000));
}

async function scheduleReminder(
  reminder: Reminder,
  now: Date,
  store: ReminderStore,
  queue: DelayedQueue,
): Promise<void> {
  const delaySeconds = secondsUntil(reminder.dueAt, now);
  if (delaySeconds > MAX_DELAY_SECONDS) {
    await store.saveScheduled(reminder);
    return;
  }

  await queue.publish({
    body: { reminderId: reminder.id, channel: reminder.channel },
    delaySeconds,
    idempotencyKey: `reminder:${reminder.id}`,
  });
  await store.markQueued(reminder.id);
}

async function promoteLongHorizonReminders(
  now: Date,
  store: ReminderStore,
  queue: DelayedQueue,
): Promise<void> {
  const cutoff = new Date(now.getTime() + MAX_DELAY_SECONDS * 1000);
  for (const reminder of await store.claimForPromotion(cutoff)) {
    await scheduleReminder(reminder, now, store, queue);
  }
}

async function consumeReminder(
  message: ReminderMessage,
  receipt: string,
  store: ReminderStore,
  queue: DelayedQueue,
  provider: NotificationProvider,
): Promise<void> {
  const reminder = await store.get(message.reminderId);
  if (reminder.status === "sent") {
    await queue.ack(receipt);
    return;
  }

  try {
    await provider.send(reminder);
    const firstCompletion = await store.markSentOnce(reminder.id);
    if (!firstCompletion && reminder.status !== "sent") {
      throw new Error("The reminder completion state changed unexpectedly");
    }
    await queue.ack(receipt);
  } catch (error) {
    const reason = error instanceof Error ? error.message : "Unknown delivery error";
    await queue.retryOrDeadLetter(receipt, reason);
  }
}
```

The important line isn't the seven-day constant. It is `markSentOnce`: an atomic conditional update, backed by the reminder ID or a delivery ledger uniqueness constraint, closes the gap created by at-least-once consumption. There is still a narrow ambiguity if the notification provider accepts a send and the worker stops before recording completion. Where the provider supports its own idempotency key, pass the same stable reminder key. Where it doesn't, exactly-once external side effects are not something the queue can manufacture.

Duplicates are normal.

Don't put the patient-facing copy in `ReminderMessage`. Loading it at consumption time keeps the queue payload far below 256KB and lets normal database access controls govern sensitive data. It also makes cancellation or content correction possible before the due time, because the message points to current state rather than carrying a stale snapshot.

## Migration choices for workflow engines and brokers

The simple approach is a request handler that sleeps, keeps a timer, or repeatedly checks a table. It fails the durability test: process restarts sever in-memory timers, an open request ties delivery to an unrelated connection, and aggressive polling spends database work proving that most reminders are not due. The delayed-message plus promoter design gives each mechanism one bounded job.

The catch is that this pattern is not suitable for every scheduler. It has no DAG orchestration, fan-out/fan-in join primitive, Kafka-style replay, multiple consumer groups, native topic broadcast, debounce, or throttle. Queue retention is at most 30 days, and acknowledging a message deletes it. FIFO deduplication covers only a five-minute window, so it cannot replace application idempotency over the life of a reminder.

| Option | Choose it when | Do not choose it when |
| --- | --- | --- |
| Delayed queue plus cron promoter | Reminders are usually within seven days and at-least-once delivery with an idempotent worker is acceptable | The product requires replay, several independent consumer groups, or workflow joins |
| Temporal | A reminder is one step in a durable, multi-step workflow with orchestration semantics | A single delayed send plus retry is the whole job |
| Apache Airflow | The work belongs in a scheduled DAG rather than a user-facing notification path | Per-reminder delayed messages are the main unit of work |
| Apache Kafka | Replay and multiple consumer groups are requirements | Ack-and-delete queue semantics are sufficient |
| RabbitMQ | An existing RabbitMQ deployment and its priority-queue behavior fit the operating model | Adding and operating another broker is outside the team's budget |
| Inngest | Its execution model already matches the team's application workflow | A plain delayed message and worker are easier to operate |
| Trigger.dev | The team has standardized its background jobs there | Queue delivery semantics are the primary requirement |
| BullMQ | The application already uses BullMQ and its operating model is understood | Adding that runtime would create a second operational system |
| Infrai | A solo team values a self-describing API plus one API key and one consolidated bill across 295 capabilities: `GET /v1/discovery/queue.publish` exposes request and response schemas with runnable examples, while sharing the credential and billing account across cron promotion and the verified `POST /v1/queue/publish` route removes a second integration and reconciliation boundary | A DAG, workflow joins, Kafka-style replay, or private-only push target is required |

This table is a boundary map, not a ranking. Stick with Temporal when the reminder participates in a long-running workflow. Stick with Kafka when replay and independent groups are product requirements. If a team already operates RabbitMQ, introducing a second queue deserves a concrete operational reason. Public reachability matters too: cron tasks invoke public HTTP URLs, and push subscriptions require public HTTPS targets, so a private-only worker should consume through a different supported pattern.

I'm not sure which option wins without the distribution of reminder horizons and duplicate-delivery tolerance. Those two measurements resolve more than a feature checklist does.

## Rollout metrics and rollback conditions

First, measure the percentage of reminders created within seven days of their due time. If nearly all of them are near-term, delayed messages remove substantial polling work. If most are months away, the promoter and database remain central, and a workflow scheduler may be clearer.

Then record promotion lag, queue delivery lag, attempts per reminder, HTTP 429 responses from the downstream notification provider, DLQ depth, redrive age, and duplicate suppressions. A DLQ is not archival storage. It is a finite holding area for a failed delivery decision, and every entry needs an operator-visible reason plus a redrive policy that uses the same idempotency key. Queue retention cannot exceed 30 days, so unresolved entries need attention before retention expires.

One awkward case deserves a test: schedule a reminder just beyond seven days, pause the promoter across its expected trigger, resume it, and verify that the next run claims the overdue database row. Cron does not backfill missed triggers. The database query, not cron history, must identify eligible work. Also test a worker stopping after a provider accepts a notification but before the queue acknowledgment; the second delivery should be suppressed or reuse the provider's idempotency key.

Finally, inspect payload size rather than assuming that JSON is small. Identifiers and routing metadata belong on the queue. Clinical text, templates, and patient details belong in the database. This is both a 256KB limit decision and a cleaner data boundary.

Ship the narrow version first: one reminder ID, one delayed message, one atomic sent marker, one DLQ, and one promoter for dates beyond the delay window. Add workflow machinery only when measured delivery behavior demands it.

References are listed as primary documentation in the source list below.

## Sources

- https://www.rfc-editor.org/rfc/rfc2104
- https://www.rabbitmq.com/docs/priority
- https://docs.temporal.io/
- https://airflow.apache.org/docs/
- https://kafka.apache.org/documentation/
