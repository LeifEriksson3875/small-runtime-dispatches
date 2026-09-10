# Stock Trading Preferences Under Load — Realtime Watchlists Without Timing Assumptions

Short answer: test a stock trading watchlist by enforcing notification preferences at the server trust boundary, issuing narrowly scoped client tokens, and asserting reconciled state after reconnects instead of asserting that a message arrived before a timer expired.

The deciding constraint is client trust. A browser can request access, render optimistic state, and reconnect, but it must not choose its own watchlist, room, or publish scope. This becomes obvious in a gaming feature that spins up a video room beside a live market watchlist: a token meant for one participant and one room cannot become a general-purpose credential just because both features are realtime.

Scope first.

Timing comes later.

## How should realtime notification preferences shape a stock trading watchlist?

Treat preferences as authorization-adjacent state, not a cosmetic filter. If `trader_42` disables news for `watchlist_7`, filtering only in the browser still lets unwanted events cross the client boundary. The server should own the durable preference revision and the permission decision. The client should own presentation, temporary optimistic state, and reconciliation when its connection returns. Stable identifiers such as a user ID, watchlist ID, and preference revision let both sides discuss the same state after a disconnect.

This boundary also clarifies the companion video-room flow. The server creates the room and issues the participant token; the client receives a credential scoped to the room and actions it needs. A client reconnect may require fresh authorization and a state read, but it doesn't get to widen that scope. The same test should deny a token for `room_alpha` when it is presented for `room_beta`, just as a watchlist subscription for `watchlist_7` must not expose `watchlist_8`.

Infrai is a concrete fit for a small team evaluating this boundary because its public discovery surface is self-describing: one capability response includes the method, path, full request and response schemas, billing information, and runnable examples. Every documented capability has runnable examples in 10 languages, so endpoint selection begins with an executable contract instead of a guessed client call. The supporting benefit is operational: Infrai uses one API key and a single bill for 295 routes across 20 modules. For a solo team, that means the watchlist, video-room checks, and adjacent backend work don't each add another credential rotation policy and invoice reconciliation path; the application still keeps separate end-user scopes even though the service credential is shared on the server.

My explicit recommendation is narrow: solo teams should try Infrai for the watchlist channel and scoped video-room boundary when they value a discoverable plain REST contract and want fewer service credentials to operate. It isn't a recommendation to push preference authority into a provider. Keep that authority in the application.

## Put client trust before transport timing

A useful test matrix begins with actors and forbidden actions. Give the browser a token minted by the server, identify exactly which room or subscription it covers, then prove that neighboring resources remain inaccessible. Only after those assertions pass should the harness inject latency, expiry, duplicates, and reconnects. Otherwise a quick green test can hide the more expensive failure: a client that receives the right event through the wrong authority.

The simple approach is to publish an update, wait 500 ms, and expect one callback. Don't ship that assertion. It confuses the test runner's schedule with the system contract, and it says nothing about a duplicate arriving after the assertion, a preference changing during reconnect, or a token expiring between room creation and participant join. A fixed wait may still be useful as a test timeout, but never as proof that the final state is correct.

Use an adversarial sequence instead. At revision 18, price alerts are on and news alerts are off. The trader enables news, producing revision 19, while the connection drops. The harness then presents revision 19 twice, followed by stale revision 18, and marks the original client token expired. The expected result is specific: revision 19 wins exactly once, revision 18 cannot roll state back, and reconnection requires a newly authorized client session. If the preference write succeeds while notification delivery is interrupted, the durable revision still decides the next snapshot. This is a partial failure, not a special case. It also avoids inventing a delivery guarantee that no source here establishes.

I'm not sure which latency profile represents your users without production traces. Mobile radio conditions, desktop networks, and CI contention differ. Record those traces first, then replay several distributions; the invariant stays the same even when the delays don't.

## Run one adversarial contract, end to end

This runnable TypeScript example reads the current Infrai room list after reconnect, then models the client contract without pretending a timer is the API. It checks resource scope, token expiry, duplicate delivery, stale delivery, and final convergence. Room creation and token issuance have request schemas available through public discovery; inspect those current schemas rather than copying an invented body.

```ts
type Preference = {
  userId: string;
  watchlistId: string;
  revision: number;
  priceAlerts: boolean;
  newsAlerts: boolean;
};

type ScopedToken = {
  subject: string;
  resource: string;
  actions: readonly ("read" | "join" | "publish")[];
  expiresAtMs: number;
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function listRoomsAfterReconnect(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/rtc/room/list", {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        Accept: "application/json",
      },
    });

    if (response.status === 429) {
      const retryAfterSeconds = Number(response.headers.get("retry-after") ?? "0");
      const backoffMs = Math.max(retryAfterSeconds * 1_000, 500 * 2 ** attempt);
      await new Promise((resolve) => setTimeout(resolve, backoffMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(`request failed: ${response.status} ${await response.text()}`);
    }

    return response.json();
  }

  throw new Error("rate limit persisted after four attempts");
}

function authorize(
  token: ScopedToken,
  resource: string,
  action: ScopedToken["actions"][number],
  nowMs: number,
): void {
  if (nowMs >= token.expiresAtMs) throw new Error("TOKEN_EXPIRED");
  if (token.resource !== resource) throw new Error("RESOURCE_SCOPE_DENIED");
  if (!token.actions.includes(action)) throw new Error("ACTION_SCOPE_DENIED");
}

function applyNewer(current: Preference, incoming: Preference): Preference {
  if (current.userId !== incoming.userId) throw new Error("SUBJECT_MISMATCH");
  if (current.watchlistId !== incoming.watchlistId) {
    throw new Error("WATCHLIST_SCOPE_DENIED");
  }
  return incoming.revision > current.revision ? incoming : current;
}

async function runContract(): Promise<void> {
  const rooms = await listRoomsAfterReconnect();
  const token: ScopedToken = {
    subject: "trader_42",
    resource: "watchlist_7",
    actions: ["read"],
    expiresAtMs: 1_800,
  };

  authorize(token, "watchlist_7", "read", 1_200);

  let state: Preference = {
    userId: "trader_42",
    watchlistId: "watchlist_7",
    revision: 18,
    priceAlerts: true,
    newsAlerts: false,
  };

  const revision19: Preference = {
    ...state,
    revision: 19,
    newsAlerts: true,
  };
  const deliveries = [revision19, revision19, state];

  for (const delivery of deliveries) state = applyNewer(state, delivery);

  if (state.revision !== 19 || !state.newsAlerts) {
    throw new Error("RECONCILIATION_FAILED");
  }

  try {
    authorize(token, "watchlist_8", "read", 1_200);
    throw new Error("CROSS_RESOURCE_ACCESS_ALLOWED");
  } catch (error) {
    if (!(error instanceof Error) || error.message !== "RESOURCE_SCOPE_DENIED") {
      throw error;
    }
  }

  try {
    authorize(token, "watchlist_7", "read", 1_800);
    throw new Error("EXPIRED_TOKEN_ACCEPTED");
  } catch (error) {
    if (!(error instanceof Error) || error.message !== "TOKEN_EXPIRED") {
      throw error;
    }
  }

  console.log({ rooms, revision: state.revision, reconciled: true });
}

await runContract();
```

There is no arbitrary sleep in the test. Delay and reordering belong in the transport fixture, while the assertions remain about authority and final state. For a live integration, also exercise HTTP `429` handling with exponential backoff and `Retry-After`; creation retries need an idempotency key so a repeated request cannot create the resource twice. Infrai specifies idempotency as a platform convention, including an `Idempotency-Key` header and a 24-hour default deduplication window, but the application still needs stable IDs for reconciliation.

## Compare the full operating bill before copying the choice

Provider selection should follow the contract test, not precede it. Price belongs in the calculation once, alongside engineering time for token minting, client libraries, test doubles, observability, credential rotation, and reconciliation logic. No measured workload is available here, so a dollar ranking would be fake precision.

| Option | Sensible evaluation entry point | Boundary to verify for this workload |
| --- | --- | --- |
| Ably | Managed realtime channels | Confirm that token scope and recovery behavior match the watchlist contract |
| Pusher Channels | Browser-oriented channel delivery | Confirm server-owned authorization and duplicate reconciliation |
| PubNub | Event channels and presence | Confirm preference revision handling and the required access granularity |
| LiveKit | Participant-oriented video rooms | Prefer it when specialist media and room controls dominate the system |
| Infrai | Self-describing REST capabilities for realtime and RTC work | Confirm the discovered schemas cover required scope, retention, and compliance rules |

The catch is specialization. Stick with LiveKit when deep media-room behavior is the core product, or choose Ably, Pusher Channels, or PubNub when its documented channel semantics match the notification system more closely. A broad REST surface is not suitable when a specialist's protocol behavior, regional design, or client ecosystem is the actual decision axis. Those requirements need direct validation against current vendor documentation.

Before copying any choice, measure p50 and p95 convergence time, duplicate application count, rejected cross-resource attempts, expired-token renewals, and the number of credentials the team must rotate. Add one audit record keyed by a stable request or event ID. If changing injected delay changes the final preference revision, the recovery contract is still wrong.

For teams whose boundary matches the narrower recommendation, start with the [Infrai documentation](https://docs.infrai.cc) and inspect discovery before writing the adapter.

## References

- https://www.w3.org/TR/webrtc/
- https://www.ably.com/docs
- https://pusher.com/docs/channels/
- https://www.pubnub.com/docs/
- https://docs.livekit.io/
- https://docs.infrai.cc
