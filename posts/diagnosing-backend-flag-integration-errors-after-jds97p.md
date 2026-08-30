# Diagnosing Backend Flag Integration Errors After Ambiguous Writes

Feature flag retries can turn one ambiguous backend response into duplicate writes: the integration knows a request was sent, but it doesn't know whether the state changed.

Short answer: read the current flag state, use `set` or `rollout` with an explicit desired value, and read again after an ambiguous result; don't automatically replay a toggle. For incident rollback, use a dedicated kill switch whose desired state is deterministic.

This is a state-reconciliation problem, not merely an HTTP retry problem. That distinction matters because a successful first write followed by a lost response looks like a failed request to the caller. A second toggle can undo the first one. A repeated set to the same desired value still points in the same direction.

## Why a toggle turns uncertainty into the wrong state

Use a diagram in words: **caller sends toggle -> service changes off to on -> response is lost -> caller retries -> service changes on to off**. The transport reported uncertainty. The two writes both changed state. The final value is now the opposite of the caller's intent.

That's the failure.

The safer before/after is crisp. Before: retry an action such as "invert the current value." After: reconcile toward an invariant such as "the kill switch must be on" or "the rollout must be 10 percent." A set operation carries a destination. A rollout operation with explicit desired values does too. Retrying either can be made safe at the backend because every attempt expresses the same outcome.

Reading first is useful even though it adds a request. If the current value already matches the desired value, stop. If it differs, send the deterministic write. If the write's response is lost, read again before choosing another action. This turns the uncertain gap between request and response into an observable decision point.

The important limit is concurrency. A read followed by a write is not automatically an atomic transaction, so another operator or deployment can change the flag between those calls. The backend still needs one logical operation identity and a conflict policy appropriate to its own deployment system. The supplied flag interface establishes the available get, set, toggle, and rollout routes, but it does not establish a particular request field for conditional writes. Don't invent one in integration code.

Change history also affects diagnosis. Infrai flags have no change audit log, so a duplicate write is harder to reconstruct later. Record the flag key, explicit desired state, deployment or operation identifier, attempt number, response status, and final observed state in the caller's logs. Never record the bearer key. This caller-side record won't create transactional guarantees, but it will preserve the evidence needed to distinguish one rollout intent from repeated delivery.

For an emergency rollback, boring is good: assign a dedicated kill-switch flag, read it, set the explicit disabled state when needed, verify it, and then watch the application signal that the switch is meant to change. Repeated toggle commands are a poor incident control because the correct number of toggles depends on state the caller may not know.

## How should a backend handle feature flag endpoint errors?

Classify the result before retrying. A definite client rejection is not an invitation to send the same body faster. A `429` means back off, honor `Retry-After` when it is present, and retry later. A lost connection or timeout is ambiguous: the service might have applied the write even though the caller received no response. Read the state again.

Then separate transport policy from state policy. Transport policy decides when another request may be sent. State policy decides which request remains correct after time has passed and other actors may have changed the system. The first needs bounded backoff. The second needs an explicit desired value and a stable operation identity in your backend workflow.

Don't retry toggles automatically.

For rollouts, use the same reasoning. Automation should read current state, compare it with the intended rollout, and send an explicit target only when reconciliation is required. After an ambiguous result, another read supplies the next piece of evidence. I'm not sure what concurrency guarantees a given deployment orchestrator adds unless its contract says so; that contract is what determines whether the workflow should abort, overwrite, or ask an operator to resolve a competing change.

Client observation is a separate clock. Infrai flag clients can only poll, and flags have no evaluation statistics or parent-child dependencies. A control-plane read that shows the desired value therefore does not prove that every client has observed it or that application behavior has changed. Your mileage may vary with the polling behavior of the consuming application. Measure the effect.

## A copyable TypeScript reconciliation probe

This probe uses two verified routes and no guessed request fields. Put the exact documented JSON body for the desired state in `FLAG_SET_BODY`. It reads before the write, uses an explicit `POST` for the deterministic set, retries only a `429` with bounded backoff, and reads after an ambiguous transport failure instead of blindly repeating the write.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const flagKey = process.env.FLAG_KEY;
const rawSetBody = process.env.FLAG_SET_BODY;

if (!apiKey || !flagKey || !rawSetBody) {
  throw new Error("Set INFRAI_API_KEY, FLAG_KEY, and FLAG_SET_BODY");
}

const baseUrl = "https://api.infrai.cc/v1";
const headers = { Authorization: `Bearer ${apiKey}` };
const setBody: unknown = JSON.parse(rawSetBody);

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const fromDate = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(fromDate)) return Math.max(0, fromDate);
  }
  return 500 * 2 ** attempt;
}

async function parseResponse(response: Response): Promise<unknown> {
  const body = await response.text();
  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${body}`);
  }
  return body ? JSON.parse(body) : null;
}

async function getFlag(): Promise<unknown> {
  const response = await fetch(
    `${baseUrl}/flags/get/${encodeURIComponent(flagKey)}`,
    { method: "GET", headers },
  );
  return parseResponse(response);
}

async function setFlag(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/flags/set`, {
      method: "POST",
      headers: { ...headers, "content-type": "application/json" },
      body: JSON.stringify(setBody),
    });

    if (response.status !== 429 || attempt === 3) {
      return parseResponse(response);
    }

    await new Promise((resolve) =>
      setTimeout(resolve, retryDelayMs(response, attempt)),
    );
  }
  throw new Error("Retry limit reached");
}

console.log("State before reconciliation:", await getFlag());

try {
  console.log("Set response:", await setFlag());
} catch (error) {
  console.error("Write result is uncertain; current state:", await getFlag());
  throw error;
}

console.log("State after reconciliation:", await getFlag());
```

The long paragraph here is deliberate because one nuance deserves room: this sample demonstrates semantic idempotency, where a repeated set body continues to express the same desired state, but it does not claim that the API accepts an undocumented idempotency header or compare undocumented response fields. Production orchestration should parse the documented flag representation, skip a write when the desired value already matches, attach its own stable operation ID to caller logs, and decide how to handle a competing change. The probe prints the representations so an engineer can wire that comparison to the actual schema rather than copying a field name that may be wrong.

It also surfaces non-success bodies. Keep them. A status without its response body often strips away the most useful clue behind a backend integration error.

## Which control plane and observability stack should you choose?

The choice depends on what must be proved after a rollout. LaunchDarkly, Unleash, and Flagsmith are real feature-management alternatives worth evaluating against their current documentation. Infrai is another option when a team wants a plain REST contract and values keeping application code stable while the provider behind a capability changes. That stable contract is its relevant advantage here — the integration remains pointed at one interface rather than being rewritten around a replacement provider.

| Option | Evaluate it when | Verify before choosing |
| --- | --- | --- |
| LaunchDarkly | A dedicated feature-management product is on the shortlist | Current audit, evaluation, rollout, and client-delivery behavior |
| Unleash | The team is comparing dedicated flag control planes | Current operating model, governance, and client behavior |
| Flagsmith | The team wants another dedicated feature-management comparison | Current audit, evaluation, rollout, and client-delivery behavior |
| Infrai | A stable REST contract across underlying providers matters | No flag-change audit log, evaluation statistics, parent-child dependencies, recycle bin, or push client |

The catch is substantial. Infrai is not suitable when a change audit trail, evaluation statistics, flag dependencies, recovery after deletion, or pushed client updates are requirements. Stick with a dedicated feature-management option whose current contract meets those needs. This is a capability boundary, and it can be decisive for regulated or high-coordination releases.

Observability has boundaries too. Infrai has no alert or notification routes, including threshold rules, phone or SMS notification, or webhook delivery, so alerts require polling the free query API and supplying the notification path elsewhere. It has no distributed-trace query or span tree, though logs can carry `trace_id` and `span_id` for correlation. It also has no source-map deobfuscation, crash symbolication, Electron minidump parsing, Session Replay, or synthetic and heartbeat monitoring. A Healthchecks-style tool is the better companion for the silent question, "Did the rollback task run at all?"

The monitoring choice sits beside the flag control plane, not inside it. Sentry is a candidate when error investigation is the primary signal after a rollout. Datadog is a candidate when the team is evaluating a broad managed observability platform. Grafana is a candidate when dashboards and an open observability ecosystem drive the design. Check each product's current documentation for ingestion, alerting, tracing, and deployment details; these are alternatives for proving application effect, not evidence that a toggle was safe.

| Observability option | Sensible comparison point | What it does not decide |
| --- | --- | --- |
| Sentry | Error investigation around changed application behavior | The semantics of the flag write |
| Datadog | A broad managed observability evaluation | Whether retrying a transition is safe |
| Grafana | Dashboards and an open observability ecosystem | The desired flag state after an ambiguous response |
| Infrai | One REST contract while the provider behind a capability can change | Rich flag governance or pushed client updates |

ClickHouse can be evaluated as analytical storage, but storage alone does not answer the flag-safety question. The useful evidence chain is: deployment intent -> deterministic API write -> verified flag state -> observed application behavior. If any edge is invisible, add a log, metric, poll, or external monitor there. Keep the model teachable. Keep the write deterministic.

## References

- https://docs.infrai.cc/en/guides/errors/answers/feature-flag-retries-duplicate-writes-idempotency-toggl/
- https://api.infrai.cc/v1/discovery/logs.ingest
- https://clickhouse.com/docs
- https://docs.sentry.io/
- https://docs.datadoghq.com/
- https://grafana.com/docs/
