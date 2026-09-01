# Auditable Recovery Ledgers for Scheduled Cron Trigger Enqueue and Queue Worker Cleanup

Short answer: treat each reservation-expiry cycle as a durable run that can be inspected and reconciled, then let a cron trigger enqueue only that run's ID; a Node.js queue worker can claim bounded pages without making a 15-minute execution limit part of correctness.

The usual diagram, `cron -> queue -> worker`, leaves out the thing an operator needs after an interruption: a durable answer to “what remains?” For fixed-window property holds, the useful unit is not a process invocation. It is a run with a frozen cutoff, a continuation cursor, and counts that can be checked against reservation state. The timer may arrive late or twice. Recovery still has a place to begin.

Keep the clock out of the hot path.

## Failure recovery begins with a run record

A reservation should expire according to its stored `expiresAt`, not according to a worker's start time. At each scheduled tick, create or reuse a run identified by the cutoff window, then enqueue the run ID. The worker reads that record, claims it for a bounded period, processes one page, saves the next cursor, and releases or completes the claim. A separate reconciliation pass looks for incomplete runs and makes them eligible again. This reverses the common design discussion: the first question is not how frequently cron fires, but what durable evidence survives when execution stops between two writes.

Use two kinds of idempotency. The run key prevents repeated scheduler delivery from creating independent sweeps for the same cutoff. The reservation transition prevents an old run from expiring a hold that was confirmed or extended. Queue deduplication can reduce repeated delivery, and FIFO queue documentation describes ordering and deduplication behavior, but neither property can decide whether the current business state still permits expiry. That decision belongs in an atomic conditional database update.

The run also makes partial progress legible. `cursor = hold-240` says where the next page begins; `examined = 240` and `expired = 187` describe what happened without pretending every examined row needed a change. If a process disappears after the database commit but before it records the next cursor, replay may inspect that page again. Fine. Conditional expiry turns that replay into harmless work, while a transaction or an outbox can tighten the boundary where the storage system supports it.

This costs state.

That cost is justified when the cleanup spans enough reservations that interruption and replay are routine operational concerns. It isn't justified for one fast conditional update that comfortably fits the scheduler's retry model.

## How should a Node.js cron trigger recover long-running background cleanup jobs?

The cron trigger should perform a tiny, repeatable transaction: derive a stable window key, upsert the run, enqueue its ID, and return. It must not own an unbounded scan. A queue worker then advances the run in pages, with one claimant at a time and a lease that can expire. If the worker loses its lease, it stops committing progress; the reconciler can later expose the same run for another claim.

There is a subtle detail here. Capture the cutoff once. If every page calls `new Date()` and expands the eligibility window, a busy run can chase newly stale reservations forever, its result depends on page latency, and a retry no longer represents the same decision. Freezing the cutoff makes a run reproducible: it asks which holds were still `held` and due no later than one timestamp. The next scheduled run covers later deadlines.

The scheduler's own timing is only a wake-up signal. GitHub's scheduled workflow documentation, for example, notes that scheduled events can be delayed during periods of high load and that some queued jobs may be dropped; it also documents a shortest schedule interval of five minutes. Those are reasons to query durable deadlines and reconcile missed windows, not reasons to assume the timer is a precise clock. Other schedulers have their own delivery contracts, so verify those contracts before deciding how far back reconciliation must look.

For a 15-minute hold, a five-minute trigger cadence does not mean the job has a five-minute runtime budget or that every reservation expires on the exact second. It means the system should define and measure expiry lag. If the business requires sub-minute release, a coarse cron sweep is the wrong clock; use a deadline-oriented mechanism or a more frequent trusted scheduler, while retaining the conditional state transition.

## Implementation example: a runnable TypeScript recovery ledger

This example keeps storage and the queue in memory so the recovery mechanics fit in one file. Production adapters must make `claim`, `saveProgress`, and `expireIfCurrent` atomic in persistent storage. The example deliberately runs one item per page, abandons the first lease after one page, advances time, and then reconciles the run. That sequence exposes the recovery path instead of presenting only the happy path.

```ts
type ReservationStatus = "held" | "confirmed" | "expired";

type Reservation = {
  id: string;
  status: ReservationStatus;
  expiresAt: string;
};

type Run = {
  id: string;
  cutoff: string;
  cursor?: string;
  examined: number;
  expired: number;
  state: "ready" | "leased" | "complete";
  leaseUntil?: number;
};

const reservations: Reservation[] = [
  { id: "hold-101", status: "held", expiresAt: "2026-08-21T12:15:00Z" },
  { id: "hold-102", status: "held", expiresAt: "2026-08-21T12:35:00Z" },
  { id: "hold-103", status: "confirmed", expiresAt: "2026-08-21T12:10:00Z" },
  { id: "hold-104", status: "held", expiresAt: "2026-08-21T12:20:00Z" },
];

const runs = new Map<string, Run>();
const ready: string[] = [];

function schedule(cutoff: string): string {
  const id = `reservation-expiry:${cutoff}`;
  if (!runs.has(id)) {
    runs.set(id, {
      id,
      cutoff,
      examined: 0,
      expired: 0,
      state: "ready",
    });
    ready.push(id);
  }
  return id;
}

function claim(runId: string, nowMs: number, leaseMs: number): Run | undefined {
  const run = runs.get(runId);
  if (!run || run.state === "complete") return undefined;
  if (run.state === "leased" && (run.leaseUntil ?? 0) > nowMs) return undefined;

  run.state = "leased";
  run.leaseUntil = nowMs + leaseMs;
  return run;
}

function expireIfCurrent(id: string, expectedExpiresAt: string): boolean {
  const reservation = reservations.find((candidate) => candidate.id === id);
  if (
    !reservation ||
    reservation.status !== "held" ||
    reservation.expiresAt !== expectedExpiresAt
  ) {
    return false;
  }

  reservation.status = "expired";
  return true;
}

function processPage(runId: string, nowMs: number, pageSize: number): void {
  const run = claim(runId, nowMs, 30_000);
  if (!run) return;

  const eligible = reservations
    .filter(
      (reservation) =>
        reservation.expiresAt <= run.cutoff &&
        (!run.cursor || reservation.id > run.cursor),
    )
    .sort((left, right) => left.id.localeCompare(right.id));

  const page = eligible.slice(0, pageSize);
  for (const reservation of page) {
    run.examined += 1;
    if (expireIfCurrent(reservation.id, reservation.expiresAt)) run.expired += 1;
    run.cursor = reservation.id;
  }

  if (eligible.length <= pageSize) {
    run.state = "complete";
    delete run.leaseUntil;
  }
}

function reconcile(nowMs: number): void {
  for (const run of runs.values()) {
    const leaseExpired = run.state === "leased" && (run.leaseUntil ?? 0) <= nowMs;
    if (run.state === "ready" || leaseExpired) {
      run.state = "ready";
      delete run.leaseUntil;
      if (!ready.includes(run.id)) ready.push(run.id);
    }
  }
}

const runId = schedule("2026-08-21T12:30:00Z");
processPage(ready.shift()!, Date.parse("2026-08-21T12:30:00Z"), 1);

reconcile(Date.parse("2026-08-21T12:30:31Z"));
processPage(ready.shift()!, Date.parse("2026-08-21T12:30:31Z"), 10);

console.log(runs.get(runId));
console.log(reservations);
```

The expected result is narrow. `hold-101` and `hold-104` become `expired`; `hold-102` remains `held` because its deadline is after the frozen cutoff; `hold-103` remains `confirmed`. The final run is `complete`, with three examined reservations and two transitions. An adapter backed by a real database should express the transition as the equivalent of “update where ID, status, and expected expiry all match,” then use the affected-row count as the boolean result. The read-then-write implementation above demonstrates the predicate, not a production concurrency guarantee.

Don't copy the in-memory lease as infrastructure.

In a deployed system, compare lease ownership as well as lease time when saving the cursor, or an older worker can wake up and overwrite a newer worker's progress. Use a unique claim token and require it in the conditional progress update. Also decide whether page selection and reservation updates share a transaction. If they do not, replay is expected, so the conditional reservation update remains mandatory. The exact transaction design depends on the database; I'm not sure which isolation and locking strategy is right without the chosen store and its contention profile. A concurrency test with two claimants and a deliberately expired lease resolves more than an abstract promise about “exactly once.”

## Measure the 15-minute recovery budget

Once progress is durable, a runtime ceiling becomes a capacity constraint. Set the page size and lease so one page normally completes with room for acknowledgement and shutdown. Before the host ends execution, stop claiming new pages. A later worker resumes from the stored cursor. No single invocation must finish the portfolio.

Watch the age of the oldest incomplete run, expiry lag, pages per run, lease expirations, and conditional-update outcomes. Queue depth alone is misleading: ten messages might represent ten tiny pages or ten portfolios with millions of holds. Expiry lag connects the machinery to the property manager's actual concern, which is how long an unavailable hold continues to block inventory after its deadline.

Reports fit the same ledger only when their output is partitionable or replaceable under a stable period key. A report for `building-17` and a closed reporting window can store a cursor and write deterministic partitions before publishing a final manifest. A single rendering call that cannot resume is different. Wrapping it in a queue changes where it times out; it doesn't create recovery. Choose a persistent workflow or batch runner that records step state for that shape of work.

There is no magic batch size.

Larger pages reduce queue operations and checkpoint writes, but they expand the replay unit, hold claims longer, and create burstier database load. Smaller pages recover with less repeated work while paying more coordination overhead. Start from the host limit, database latency under production-shaped load, and the allowed expiry lag. Then measure. A solo team should resist building an elaborate controller before those measurements show a need; one run table, one conditional transition, and one reconciler are already meaningful operational surface area.

## Compare operational limits before deployment

Test recovery before testing throughput. Create two schedule deliveries for the same cutoff and verify that only one run exists. Pause the worker until its lease expires, claim the run from another worker, then let the old worker attempt to save progress; the claim token should reject that stale write. Confirm one reservation and extend another after they are selected but before expiry commits. Both conditional updates should become successful no-ops. Finally, remove a queued wake-up while leaving its run `ready`; reconciliation should enqueue the durable run again.

Deployment should be incremental. Run the scanner in observation mode first, recording which reservation IDs would qualify without changing them, and compare that set with the current application rules. Then enable a small property cohort, watch expiry lag and rejected conditional updates, and expand only after a full hold window. Keep a direct way to pause new claims while leaving run records intact. Recovery controls that require deleting history are not recovery controls.

The catch is the ledger needs ownership, retention, indexes, alerts, and a reconciliation schedule. For a small installation where cleanup is one bounded SQL statement and the database scheduler already provides visible retry history, stick with the direct scheduled statement. For strict per-reservation timing, use a deadline-oriented facility rather than accepting sweep cadence. For indivisible multi-step reports, use a persistent execution engine. The scheduled enqueue plus queue worker pattern is suitable when work can be divided, state changes can be made idempotent, and operators need to resume a specific run instead of rerunning an opaque process.

| Work shape | Recovery boundary | Reason to reject it |
| --- | --- | --- |
| One bounded conditional update | Direct scheduled statement | A queue and run ledger add more state than recovery value |
| Many independently expiring holds | Run ledger with queue workers | Unsuitable if transitions cannot be made idempotent |
| One indivisible report | Persistent workflow or batch run | Page cursors cannot resume an opaque all-or-nothing call |
| Strict per-item deadline | Deadline-oriented execution | A periodic sweep permits timing drift up to its operating lag |

The final operational check is plain prose because it should be read as a sequence. Verify that the scheduler creates a stable run, the worker can lose a lease without losing its cursor, an old claimant cannot commit, and reconciliation can recover a ready run whose wake-up vanished. Confirm that dashboards show oldest-run age and domain expiry lag, not just successful cron invocations. Document the cutoff rule, claim duration, retention period, and the person responsible for replay. Then rehearse the interruption with production-shaped data before the first large turnover window.

## References

- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fifo-queues.html
- https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows
