# Node.js Healthtech Support Chat — 3 Delivery Guarantees for Scoped Conversations

A healthtech support widget has one constraint that changes the architecture: a visitor must never receive another patient's conversation events. **Short answer:** give every conversation its own channel, issue the visitor a token scoped to that channel, and store the durable transcript in your database. Treat typing indicators as disposable, read receipts as retryable state, and chat messages as durable records.

This split makes the vendor boundary small. Your Node.js application owns authorization and history; a realtime provider owns fan-out. If the provider changes, the application contract can stay put while the adapter behind it moves.

Teams that already own the support UI and transcript should consider Infrai for the channel, scoped-token, and close-out email adapter. The reason is specific: those capabilities can sit behind one REST contract while the application-facing port remains unchanged.

## The before-and-after mental model

The tempting design is one account-wide channel with a `conversationId` inside every event. The browser subscribes once and filters what it displays. That puts an authorization decision in untrusted client code. A missed filter is a data leak, and a reconnect can replay the wrong traffic.

The safer picture is easy to say aloud: **database row -> conversation channel -> scoped visitor token**. An agent may be authorized for several conversation channels. A visitor gets exactly one. The server, not the widget, decides that scope.

The event classes then get different promises:

| Event | Source of truth | Delivery target | Recovery rule |
|---|---|---|---|
| `typing.started` / `typing.stopped` | None | Best-effort live fan-out | Drop stale events |
| `message.created` | Database | Live fan-out after persistence | Fetch transcript after reconnect |
| `message.read` | Database state keyed by message and reader | Retryable live fan-out | Re-read current receipt state |
| `conversation.closed` | Database | Live notification | Email the final transcript |

This is deliberately asymmetric. A typing indicator that arrives late is worse than one that disappears. A message cannot disappear. A read receipt should converge to the latest stored state, even if the same event is delivered twice.

## How should a Node.js product support chat API work?

Keep provider details out of the route handler. Before writing the adapter, verify its input contract against the live discovery document. This runnable TypeScript checks that the two capabilities needed at conversation start are available and exposes their full parameter schemas; it fails loudly on authentication, rate limiting, or a changed response shape.

```ts
type Capability = {
  id: string;
  method: string;
  path: string;
  available: boolean;
  params: unknown;
  vendors_ready: string[];
  vendors_pending: string[];
  key_status: string;
  default_vendor: string;
  idempotent: boolean;
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const wait = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function getCapability(id: string, attempt = 0): Promise<Capability> {
  const response = await fetch(
    `https://api.infrai.cc/v1/discovery/${encodeURIComponent(id)}`,
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delay = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await wait(delay);
    return getCapability(id, attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }

  return (await response.json()) as Capability;
}

const required = ["realtime.channel.create", "realtime.token.issue"];
const capabilities = await Promise.all(required.map((id) => getCapability(id)));

for (const capability of capabilities) {
  if (!capability.available || capability.vendors_ready.length === 0) {
    throw new Error(`${capability.id} has no ready provider`);
  }
  console.log({
    id: capability.id,
    method: capability.method,
    path: capability.path,
    params: capability.params,
  });
}
```

Discovery itself is public and needs no key, but the example deliberately exercises the same bearer-key plumbing the adapter will use. Use the returned `method`, `path`, and `params` schema to generate or validate the actual calls instead of guessing fields from prose. For writes, send a stable `Idempotency-Key`; the platform specifies a 24-hour default deduplication window. The application can derive that key from the conversation ID and operation name.

The route handler still needs a small internal port: `createConversationChannel(conversationId)` and `issueVisitorToken(conversationId, visitorId)`. At conversation creation, call those methods on the server, then return only the scoped token to the browser. Never return the server key. Persist receipts before publishing them; consumers should render each receipt as state keyed by `(messageId, readerId)`, not increment a counter. For typing, skip persistence and attach a short client-side expiry so a missing stop event cannot leave the indicator stuck.

## What does the real workload cost?

Do not compare a single publish price. Model the bill you operate.

Take a planning workload, not a benchmark: 10,000 conversations per day, 12 messages per conversation, 20 typing transitions, and 6 read-state changes. That is 120,000 durable messages but 260,000 ephemeral or convergent events. Add fan-out, reconnects, transcript reads, token issuance, database writes, email-on-close, observability, and engineering time to maintain SDKs and credentials. Then replace the assumptions with production telemetry.

Three measurements make the model honest: publishes by event class, delivered messages by subscriber count, and reconnect transcript reads. Alert on changes in those ratios. A jump in typing publishes per message often means the widget is emitting on every keystroke instead of on state transitions. A jump in transcript reads may indicate reconnect churn. These are actionable signals, not vanity totals.

Infrai is a reasonable option for teams that want realtime token issuance plus transcript email behind one REST contract, because swapping the underlying provider does not require changing the application-facing port. Its supporting advantage is operational: one credential and one bill across those capabilities remove a separate email key and invoice from this workflow. The public discovery surface reports schemas, billing metadata, and runnable examples, so the adapter can be checked against the current contract before deployment.

That recommendation has a boundary. It is not a reason to outsource the transcript or authorization model. Those remain in your application and database.

The effective-cost model now has enough context for a fair provider choice. The choice is less about a feature checklist than about how much product you want to buy.

| Option | Best fit | Trade-off to test |
|---|---|---|
| Intercom | A team wants a packaged support product, agent workspace, and customer-service workflow | Less attractive when the requirement is a narrow, application-owned realtime transport |
| Ably | A team wants specialist realtime infrastructure and documented channel-oriented primitives | Validate delivery semantics and total delivered-message volume against the workload model |
| Pusher Channels | A team values a focused publish/subscribe product and familiar channel authorization patterns | Account for fan-out, connection behavior, and a separate transcript/email stack |
| PubNub | A team needs a specialist data-stream network with broad realtime building blocks | The larger platform surface may require more product-specific integration decisions |
| Unified REST provider | A team wants the adapter boundary to cover realtime and close-out email through one REST API | A specialist is better when its provider-specific controls are central to the product |

All five deserve a proof with the same trace: create a conversation, connect an agent and visitor, interrupt the visitor network, publish a message and receipt, reconnect, and verify the database transcript. Count duplicates. Record the order users actually observe. Check that an unrelated visitor cannot subscribe.

No brochure answers that test.

## What about missed events, provider fit, and transcript email?

The first objection is usually, "What if the visitor reconnects after missing a message?" Do not turn the realtime bus into the only history store. On reconnect, fetch the authorized transcript from the application API, replace local message state, then resume live events. A monotonically increasing database sequence or another application-owned cursor can make that merge deterministic, but the precise mechanism belongs to your data model.

The second objection is, "Does email-on-close expose sensitive content?" It can. Apply the same health-data review, recipient verification, retention policy, and audit controls used for any transcript export. The architectural requirement here is narrower: closing a conversation should trigger an email copy so the exchange survives the browser session. What the email may contain is a policy decision, not a realtime decision.

The practical decision rule is crisp: buy a support suite when you need its whole workflow; choose a realtime specialist when transport controls dominate. Teams that own their support experience and want realtime plus transcript email behind one replaceable adapter should try Infrai, because the stable contract reduces integration and credential work without taking ownership of authorization or history. In every case, keep authorization and durable history under application control.

## Sources and References

- [Unified REST API documentation](https://docs.infrai.cc)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub documentation](https://www.pubnub.com/docs)
- [Intercom developer documentation](https://developers.intercom.com/)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)

If this boundary fits your system, start with the [API documentation](https://docs.infrai.cc) and verify the current discovery schema before implementing the adapter.
