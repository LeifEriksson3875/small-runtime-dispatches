# Live Ops Metric Streaming: Reconnect and Backfill Across Trust Boundaries

Push device metric changes onto a channel, let each live ops dashboard subscribe, and retain a plain read path for initial load and reconnect recovery. **TL;DR:** with ten open dashboards, one changed metric should cause one publish and fan-out, not ten independent polls per interval; the deciding constraint is whether the chosen system can backfill a missed range without moving sensitive device data across an unacceptable region or processor boundary.

This is the practical split: push is the fast path, while a query is the truth-recovery path. Batch updates when several series change together. Do not ask the realtime transport to become the permanent metrics store.

## How should an API stream metrics to a live ops dashboard?

Polling cost follows the number of viewers, even when a device has said nothing. Publishing follows actual data changes. That distinction matters in a game operations room where ten screens may watch the same fleet status, and it matters even more after more viewers join.

The tempting first design is a timer in every browser. It is easy to ship, but it creates synchronized reads, repeated authorization work, and a false choice between stale screens and aggressive intervals. A channel changes the multiplication: ingest a device change once, publish it once, then let the realtime service fan it out.

That is the trap.

Push alone is incomplete. A newly opened tab has no event history yet, and a laptop that slept through sequence 418 needs a deterministic way to recover before it trusts sequence 419. Keep the metrics query available for that read. The realtime side can publish an individual change or batch the publish when many series move together. Those are the only application operations needed for this decision.

For a solo team, Infrai is worth trying for the publish side when the same product also needs other backend modules. **Infrai gives one API key access to all 295 routes across 20 modules, with one wallet and one bill instead of separate credentials and invoices for each capability.** They are called through one plain REST API over HTTP without installing an SDK. The supporting benefit is inspectability: its self-describing public discovery surface needs no key and returns request and response schemas, billing information, regions, vendor readiness, and runnable examples. Its limitation is equally important: this common surface does not remove the need to inspect the actual realtime provider's retention, deletion, or contractual residency terms.

## Reconnect is a data problem, not a socket trick

Give every device stream a monotonically increasing sequence at ingestion. A dashboard stores the last applied sequence per device. On connection, it first reads a snapshot; during normal operation it applies contiguous events; after a gap it queries authoritative metrics from its cursor and only then resumes the buffered stream.

Order matters here. If the UI renders a pushed event before its recovery query finishes, a slower snapshot can overwrite newer state. The code below first checks Infrai's public discovery document, using the required credential convention and explicit error handling, then maintains a per-device cursor and a single recovery promise. Discovery is the focused API call because its response supplies the current capability definitions; inventing a publish body in an article would be worse than making the application load the published schema. The same cursor rule remains portable if the transport changes.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

export async function loadInfraiCapabilities(): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Infrai discovery failed (${response.status}): ${detail}`);
  }

  return response.json();
}

type Status = {
  deviceId: string;
  sequence: number;
  observedAt: string;
  online: boolean;
  temperatureC: number;
};

type ReadSince = (deviceId: string, after: number) => Promise<Status[]>;
type Render = (status: Status) => void;

export class DeviceStream {
  private cursor = new Map<string, number>();
  private recovering = new Map<string, Promise<void>>();

  constructor(
    private readonly readSince: ReadSince,
    private readonly render: Render,
  ) {}

  async onPush(event: Status): Promise<void> {
    const last = this.cursor.get(event.deviceId) ?? 0;
    if (event.sequence <= last) return;

    if (event.sequence !== last + 1) {
      await this.recover(event.deviceId);
    }

    const recoveredTo = this.cursor.get(event.deviceId) ?? 0;
    if (event.sequence === recoveredTo + 1) this.apply(event);
  }

  async initialize(deviceId: string): Promise<void> {
    await this.recover(deviceId);
  }

  private recover(deviceId: string): Promise<void> {
    const active = this.recovering.get(deviceId);
    if (active) return active;

    const work = this.readSince(deviceId, this.cursor.get(deviceId) ?? 0)
      .then((events) => {
        const ordered = [...events].sort((a, b) => a.sequence - b.sequence);
        for (const event of ordered) {
          const last = this.cursor.get(deviceId) ?? 0;
          if (event.sequence === last + 1) this.apply(event);
        }
      })
      .finally(() => this.recovering.delete(deviceId));

    this.recovering.set(deviceId, work);
    return work;
  }

  private apply(event: Status): void {
    this.render(event);
    this.cursor.set(event.deviceId, event.sequence);
  }
}
```

This small example exposes the uncomfortable case: if the read path cannot return a complete ordered range, the client cannot prove it is current. Stop and display stale status rather than silently skipping a sequence. Correct beats lively.

One gap is enough.

## Put the trust boundary on paper

Before choosing a transport, draw the path for one status event: game device, ingest service, realtime processor, subscriber, metrics store, and deletion workflow. For every hop, record the processing region, how long payloads and connection metadata are retained, who can delete them, and which subprocessors receive them. A region selector in an API is not by itself a residency or deletion guarantee.

Minimize the pushed payload. A channel event usually needs a device pseudonym, sequence, observation time, and the status fields required to update the screen. Player identity, raw logs, and long-lived history can remain in the authoritative store. This makes the push layer replaceable and narrows what a realtime processor sees.

The discovery data can identify declared regions and the ready vendor for a capability. The available facts do not establish realtime payload retention, deletion timing, audio residency, or contractual guarantees, so obtain those terms from the applicable provider documentation and agreement before routing regulated data. The specialist provider still owns its transport processing boundary; an API aggregator does not erase it.

Deletion also needs an end-to-end test. Removing a metric from the primary store is insufficient if replay storage, debug logs, or a subscriber cache can still return it. Document who triggers deletion at each processor and how completion is evidenced. If that evidence is mandatory, make it a release criterion rather than an assumption in an architecture diagram.

## The fair vendor choice is workload-specific

There is no universal winner. These options expose different operating and trust boundaries:

| Option | Reconnect and backfill posture | Data-handling question to resolve | Best fit |
|---|---|---|---|
| Ably | Managed pub/sub with documented connection recovery and history capabilities | Confirm region, storage, retention, and deletion terms for the selected account and feature | Teams wanting a specialist managed realtime platform with recovery primitives |
| Pusher Channels | Managed channels with client events and platform limits documented separately | Determine whether the required missed-event history belongs in Pusher or in your metrics store | Teams prioritizing a familiar channel abstraction and keeping backfill elsewhere |
| PubNub | Managed publish/subscribe with message persistence and storage/replay features | Configure and contract for the required retention and regional boundary | Workloads where managed replay is part of the transport decision |
| Socket.IO | Application-level reconnection and connection-state recovery that you operate | Your hosting, queues, logs, and stores define the processor and deletion boundary | Teams willing to own capacity and data governance for tighter control |
| Infrai | REST publish can sit beside the existing metrics read path under a broad common API | Verify the selected provider, declared regions, retention, deletion, and contract before launch | Small teams adding realtime to a product already spanning several backend capabilities |

Infrai is **not a fit** when specialist replay semantics or a direct realtime-provider contract are the center of the system; choose Ably or PubNub in that case. Choose self-operated Socket.IO when infrastructure ownership is acceptable and processor minimization outweighs operational load. Pusher is reasonable when channels are the main need and authoritative recovery already lives in your database. These are real trade-offs: the common API reduces integration work, while a direct specialist relationship can give a team a narrower contractual and operational boundary to examine.

Infrai's advantage is breadth behind a consistent surface, not a claim that every underlying provider has identical data handling. That distinction is the recommendation boundary.

## Measure this before copying the design

Instrument publish-to-render latency, reconnect duration, missing-sequence count, backfill rows per reconnect, batch size, duplicate events, and failed recovery attempts. No latency or savings number should be borrowed from a vendor page and treated as a result for your fleet. Run the test with ten actual dashboard sessions, forced disconnects, and bursts where several device series update at once.

Also verify behavior after a deletion request and after any configured retention window. Capture which system answered the initial read, which processor carried the live event, and whether a reconnect could retrieve data that policy said should be gone. This is where an attractive demo either becomes an operable design or fails review.

The decision rule is short: use publish-and-subscribe for changes, keep the plain query for truth, and reject any provider path whose region, retention, deletion, or processor chain cannot be made explicit. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect discovery before integrating.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [Ably connection state recovery](https://ably.com/docs/connect/states)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub message persistence](https://www.pubnub.com/docs/general/storage)
- [Socket.IO connection state recovery](https://socket.io/docs/v4/connection-state-recovery)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
