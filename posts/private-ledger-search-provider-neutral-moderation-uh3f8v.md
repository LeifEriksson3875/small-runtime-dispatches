# Private Ledger Search: Provider-Neutral Moderation API Error Handling

TL;DR: Treat a content-screening rate limit as a queueing event, not a verdict on the user’s text. Persist the request, retry only transient failures with bounded exponential backoff, keep a per-tenant budget, and make the moderation decision explicit before retrieval or answer generation. A small adapter around the screening provider keeps the rest of a fintech knowledge-base service portable.

The important trade-off is delay versus trust. In a private finance knowledge base, dropping a question because screening capacity is temporarily exhausted is a poor failure mode, but silently letting it through is worse. I use a durable pending state, a retry deadline, and a conservative final action when that deadline expires. This keeps provider changes local and gives operations a decision they can inspect.

The example below assumes moderate user content, one screening call before retrieval, and a Node.js worker. The provider is deliberately represented by an interface. The application should not know which HTTP API performs the check.

## How should a Node.js moderation API handle rate-limit errors?

The queue is not the fallback.

A rate limit answers an operational question: can this request be admitted now? It does not answer the content question. That distinction matters because a rejected screening call has no useful safety meaning.

For each question, I store an idempotency key, tenant ID, normalized text hash, attempt count, next-attempt time, and a screening state. The request enters `pending_screening`, waits in a durable queue, and moves to `approved` or `blocked` only after a successful screening result. Retrieval is downstream of that transition.

There are three paths. A successful response produces a decision. A transient response gets retried within a deadline. A permanent input or authentication failure is recorded and sent to an operator-facing error path. A request that reaches the deadline is not retried forever; it receives the service’s documented conservative outcome and an event with enough context to explain why.

That is the whole control loop. Small enough to test. In a real request, the sequence is visible and boring: a customer asks whether a transfer policy covers a particular account type, the API writes the question with its idempotency key, and the worker claims it once. If the screening call is admitted, the worker records the decision before the retrieval job can read the question. If capacity is exhausted, the record keeps its pending state and its next eligible time; a second delivery sees that state instead of starting a second screening call. If the deadline arrives, the system records a terminal screening failure and returns a status that says the safety check is unavailable. Retrieval never has to guess whether an absent decision means approved content, blocked content, or an outage. That explicit state machine is more useful than a clever retry helper because it remains inspectable during a busy incident and testable without a live provider.

## The portable worker

The adapter needs to expose semantics, not a vendor-shaped response. The rest of the service needs to know whether the text was accepted, rejected, or could not be screened yet. It should not parse a particular JSON field or infer retryability from a string.

```ts
type ScreeningDecision =
  | { kind: "allow" }
  | { kind: "block"; reason: string };

class TemporaryScreeningError extends Error {
  constructor(
    message: string,
    public readonly retryAfterMs?: number,
  ) {
    super(message);
    this.name = "TemporaryScreeningError";
  }
}

interface ContentScreener {
  screen(input: { text: string; idempotencyKey: string }): Promise<ScreeningDecision>;
}

interface PendingQuestion {
  id: string;
  tenantId: string;
  text: string;
  idempotencyKey: string;
  attempts: number;
  deadlineMs: number;
}

interface QuestionStore {
  markApproved(id: string): Promise<void>;
  markBlocked(id: string, reason: string): Promise<void>;
  reschedule(id: string, nextAttemptMs: number, error: string): Promise<void>;
  recordTerminalFailure(id: string, error: string): Promise<void>;
}

const MAX_ATTEMPTS = 5;
const BASE_DELAY_MS = 250;

function backoffMs(attempt: number, retryAfterMs?: number): number {
  const exponential = BASE_DELAY_MS * 2 ** Math.max(0, attempt - 1);
  const jitter = Math.floor(Math.random() * 100);
  return Math.max(retryAfterMs ?? 0, exponential + jitter);
}

async function processQuestion(
  item: PendingQuestion,
  screener: ContentScreener,
  store: QuestionStore,
  nowMs = Date.now(),
): Promise<void> {
  if (nowMs >= item.deadlineMs) {
    await store.recordTerminalFailure(item.id, "screening deadline exceeded");
    return;
  }

  try {
    const result = await screener.screen({
      text: item.text,
      idempotencyKey: item.idempotencyKey,
    });

    if (result.kind === "allow") {
      await store.markApproved(item.id);
    } else {
      await store.markBlocked(item.id, result.reason);
    }
  } catch (error) {
    if (!(error instanceof TemporaryScreeningError)) {
      await store.recordTerminalFailure(item.id, String(error));
      return;
    }

    const attempts = item.attempts + 1;
    const delay = backoffMs(attempts, error.retryAfterMs);
    const nextAttemptMs = nowMs + delay;

    if (attempts >= MAX_ATTEMPTS || nextAttemptMs >= item.deadlineMs) {
      await store.recordTerminalFailure(item.id, error.message);
      return;
    }

    await store.reschedule(item.id, nextAttemptMs, error.message);
  }
}
```

The code has one intentional boundary: `TemporaryScreeningError` is created by the adapter after it has classified the HTTP response. A `429`-style response can be transient, and a `Retry-After` value can inform the delay when the service supplies one. A malformed request, invalid credential, or other permanent configuration error should not join the same retry storm. RFC 6585 documents 429 as a rate-limiting response, while the HTTP semantics specification defines `Retry-After` for communicating a delay.

The worker is also idempotent at the state transition. A queue can deliver the same item twice. `markApproved` and `markBlocked` therefore need a conditional update such as “only transition from pending screening,” backed by a unique idempotency key. The exact database syntax is less important than making duplicate delivery harmless.

## How do you stop one tenant from consuming the queue?

A global concurrency limit protects the provider, but it does not protect fairness. One noisy tenant can occupy every worker and make a quiet tenant wait. For a fintech application, that can turn a harmless batch import into a customer-facing outage.

I keep a small token bucket per tenant and a separate global ceiling. The scheduler picks eligible tenants in round-robin order, skips tenants whose next-attempt time is in the future, and reserves a token before dispatch. On a transient failure, the token is not immediately returned to the same tenant’s head of line. Otherwise the retry loop can starve fresh work.

The numbers are policy, not facts about a particular provider. For example, a service might allow 4 in-flight screening calls per tenant, 32 globally, and a 30-second retry deadline for interactive search. Those values belong in configuration and load tests. They should not be hidden inside an SDK wrapper.

Track queue age by tenant, oldest pending item, attempts per decision, and the share of deadline expirations. An average latency chart will miss the tenant that has been waiting for 28 seconds.

## What should the fallback say to the user?

The UI needs a distinct state for “screening is temporarily unavailable.” It should not display “content blocked,” because that claims a decision the system did not make. It should also avoid automatically sending the text to retrieval while an earlier screening attempt is pending.

For interactive search, I return a stable request ID and a short status response, then let the client poll or receive an event. For a batch workflow, I leave the item pending and surface the deadline outcome to an operator. The wording can be plain: “This question is waiting for a safety check. Try again shortly.”

That response policy is part of the threat model. If the knowledge base contains account procedures, exposing a retry mechanism must not leak the original text into logs, traces, or error messages. Log the tenant, request ID, state, attempt count, provider-neutral error class, and timing. Redact the text unless a separately controlled audit record is required.

## Provider portability is an operational property

Portability is often reduced to swapping a URL. That misses the hard parts: response classification, timeout behavior, category mapping, data retention assumptions, and how a provider signals capacity.

Keep those differences in one adapter. Map provider-specific categories into a small internal policy vocabulary, and version that mapping. If a replacement screener has different labels, the policy review should show the change before production traffic moves. The retrieval and answer layers should remain unaware of it.

Run contract tests against a fake screener and a small recorded corpus owned by the application team. Test allow, block, timeout, rate limit with a delay, malformed response, duplicate delivery, and deadline expiry. Do not use a live provider as the only test oracle; it makes failure handling slow and nondeterministic.

This is also where vendor comparisons belong: in an internal decision record, against measured dimensions such as maximum concurrency behavior, regional data handling, category stability, and adapter effort. A public article should not turn those dimensions into a product ranking without current, attributable evidence.

The deployment checklist is short enough to keep beside the worker: confirm the queue is durable, confirm state transitions are conditional, cap retries and total time, honor a server-provided retry delay when it is valid, add jitter, isolate tenant budgets, redact content from telemetry, and alert on queue age rather than only request failures. Then test a burst that exceeds screening capacity before launch.

A screening rate limit is a normal distributed-systems condition. Give it a state, a deadline, and a fair scheduler. The rest of the application can keep its provider-independent contract, and the user gets an honest answer about what happened.

## Sources

- OpenAI, Moderation guide: https://platform.openai.com/docs/guides/moderation
- IETF, RFC 6585, Additional HTTP Status Codes: https://www.rfc-editor.org/rfc/rfc6585
- IETF, HTTP Semantics, `Retry-After`: https://www.rfc-editor.org/rfc/rfc9110.html#name-retry-after
- MDN, 429 Too Many Requests: https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/429
- LangChain, ChatOpenAI integration documentation: https://python.langchain.com/docs/integrations/chat/openai/
