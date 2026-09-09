# Troubleshooting Node.js Polling: Disable Noisy Uptime Checks Before Retries Cascade

Short answer: use a feature flag to disable a noisy Node.js uptime-check polling client, make one scheduler own every retry, and cancel both the pending timer and active request.

The deciding constraint is control latency. If an operator can flip a flag but queued callbacks keep creating work, there is no kill switch; there is only a label on a dashboard. The safer mental model is small and concrete. Before: interval, timeout, and error handler can each schedule the next check. After: one state machine admits at most one request and one future timer, while telemetry distinguishes why a check should run from what it eventually achieved.

## Can a kill switch stop a Node.js polling retry storm without blinding uptime checks?

Yes, if the switch controls admission rather than hiding results. A disabled check should stop producing new traffic, but its disabled state, flag age, and last accepted outcome must remain visible. Otherwise the team trades a noisy target for a silent monitoring gap.

Picture the flow in words: cached flag snapshot -> generation check -> single scheduler -> one in-flight probe -> classified attempt -> bounded delay -> scheduler. The shutdown path is shorter: new flag generation -> clear timer -> abort probe -> record transition. Every arrow returns to the same owner. No error callback gets a private clock.

That generation check matters more than it first appears. Walk through the race slowly. A request begins in generation 7. An operator disables polling, which clears the visible timer and aborts that request; then the operator re-enables polling, creating generation 9 and its new timer. The generation 7 promise still has to settle. If its catch path merely sees that `enabled` is now true, it schedules another retry beside the generation 9 timer. Nothing looks broken in either callback, yet the process now owns two loops and can double its request rate after every similar race. A boolean cannot distinguish old work from current work. Carrying the generation through the timer and request gives the completion path enough context to reject stale work before it touches the clock.

One owner. One clock.

The flag read itself should not perform a network request in the hot path. Keep a local snapshot that another control-plane loop refreshes asynchronously, and report how old that snapshot is. This avoids making the permission check another source of piled-up requests. It also forces an operational decision that deserves to be explicit: after the snapshot exceeds its accepted age, should this particular monitor continue or pause? For a redundant, noisy probe, pausing may protect the target. For the only safety-critical signal, pausing could conceal danger. There isn't one honest default for both.

Don't ask a flag to compensate for unbounded retry behavior. Randomized backoff, a timeout, one in-flight request, and a ceiling on retry frequency are normal operating controls. The flag is the emergency control.

## Make the scheduler the narrow waist

Here is a compact polling client. It has one timer, one active request, and a monotonically increasing generation. The `applyFlag` method is idempotent when the value has not changed, while every real transition invalidates older asynchronous work.

```ts
type FlagSnapshot = {
  enabled: boolean;
  observedAtMs: number;
};

type PollerTelemetry = {
  count(name: string, labels?: Record<string, string>): void;
  gauge(name: string, value: number): void;
};

type PollerOptions = {
  url: URL;
  telemetry: PollerTelemetry;
  normalDelayMs: number;
  requestTimeoutMs: number;
};

export class UptimePoller {
  private enabled = false;
  private generation = 0;
  private timer?: ReturnType<typeof setTimeout>;
  private active?: AbortController;
  private failures = 0;

  constructor(private readonly options: PollerOptions) {}

  applyFlag(snapshot: FlagSnapshot): void {
    this.options.telemetry.gauge(
      "uptime_flag_age_ms",
      Math.max(0, Date.now() - snapshot.observedAtMs),
    );

    if (snapshot.enabled === this.enabled) return;

    this.enabled = snapshot.enabled;
    this.generation += 1;
    this.clearTimer();
    this.active?.abort("polling state changed");
    this.options.telemetry.gauge("uptime_polling_enabled", this.enabled ? 1 : 0);

    if (this.enabled) {
      this.failures = 0;
      this.schedule(this.jitter(1_000), this.generation);
    }
  }

  private schedule(delayMs: number, generation: number): void {
    if (!this.enabled || generation !== this.generation) return;
    this.clearTimer();
    this.timer = setTimeout(() => void this.tick(generation), delayMs);
  }

  private async tick(generation: number): Promise<void> {
    if (!this.enabled || generation !== this.generation || this.active) return;

    const controller = new AbortController();
    const timeout = setTimeout(
      () => controller.abort("request timed out"),
      this.options.requestTimeoutMs,
    );
    this.active = controller;
    this.options.telemetry.count("uptime_attempts_total");

    try {
      const response = await fetch(this.options.url, {
        method: "GET",
        signal: controller.signal,
      });

      if (response.ok) {
        this.failures = 0;
        this.options.telemetry.count("uptime_attempt_results_total", { result: "ok" });
        this.schedule(this.options.normalDelayMs, generation);
      } else {
        this.failures = Math.min(this.failures + 1, 3);
        this.options.telemetry.count("uptime_attempt_results_total", {
          result: `status_${response.status}`,
        });
        this.schedule(this.retryDelayMs(), generation);
      }
    } catch (error) {
      const result = error instanceof Error ? error.name : "unknown";
      this.failures = Math.min(this.failures + 1, 3);
      this.options.telemetry.count("uptime_attempt_results_total", { result });
      this.schedule(this.retryDelayMs(), generation);
    } finally {
      clearTimeout(timeout);
      if (this.active === controller) this.active = undefined;
    }
  }

  private retryDelayMs(): number {
    const cappedBackoffMs = 5_000 * 2 ** Math.max(0, this.failures - 1);
    return this.jitter(cappedBackoffMs);
  }

  private jitter(baseMs: number): number {
    return Math.round(baseMs * (0.75 + Math.random() * 0.5));
  }

  private clearTimer(): void {
    if (this.timer) clearTimeout(this.timer);
    this.timer = undefined;
  }
}
```

There is a deliberate asymmetry here. Enabling waits for a jittered delay; disabling acts immediately. That prevents a fleet from synchronizing when a rollout begins or resumes. The backoff exponent is capped in the example, so repeated failures cannot drive the delay calculation without bound, and jitter spreads attempts across time. Adapt the constants to the target's traffic budget and detection objective rather than copying them as universal values.

Test the awkward transitions, not merely the happy request. With fake timers, prove that disable before start leaves zero callbacks, disable during a request aborts it, disable during backoff clears the timer, and disable/re-enable does not let the old generation schedule work. Then advance the clock through three failed attempts and assert that concurrent requests never exceed one. A focused test should also return a `429` response and verify that the result classification remains bounded; never place a raw URL, request ID, or exception message in metric labels.

One caveat: `response.ok` only classifies the transport attempt. It does not prove that the expected monitoring state changed.

That's the trap.

## Observe intent, attempt, and outcome

Use three signal layers because they answer three different incident questions. Intent says whether the client is supposed to run: enabled state, flag snapshot age, and generation changes. Attempt says what the client did: starts, latency, response class, aborts, current backoff, and in-flight count. Outcome says whether monitoring still provides value: age of the last accepted heartbeat and alert-state transitions.

| Layer | Low-cardinality examples | Troubleshooting question |
| --- | --- | --- |
| Intent | enabled gauge, snapshot age, transition count | Should this process be polling? |
| Attempt | request count, duration, result class, retry level | Is the loop amplifying traffic? |
| Outcome | heartbeat age, alert transition | Is the monitoring objective still being met? |

This split prevents a common category error. A `200` response says the endpoint accepted an HTTP exchange. It cannot, by itself, show that a heartbeat was recorded, a freshness window advanced, or an alert evaluated. Conversely, an aborted request after the flag changed is a control action, not a target failure. Keep those event meanings separate in logs and metrics.

The four golden signals provide a useful review frame. Latency is probe duration. Traffic is attempts across the fleet, not just in one process. Errors are classified outcomes rather than an unbounded collection of message strings. Saturation includes active probes, scheduled callbacks, and pressure on the checked service. Add retry level and flag age because they expose the control loop directly.

Page on user- or operator-relevant symptoms. A stale monitoring outcome combined with elevated attempts is much more actionable than one timeout. During a rollout, put enabled instances, aggregate attempt rate, abort count, target saturation, and outcome freshness on the same view. The before/after should be readable at a glance.

Desktop processes add another boundary. Electron's `crashReporter` collects reports for native crashes and uses minidumps, according to its API documentation. That evidence can help when a process terminates, but it does not replace application-level state-transition logs or freshness metrics for a live polling loop. Use each signal for the failure class it can actually observe.

## Roll out the control and troubleshoot recovery

Start with polling disabled in the production configuration, enable one representative process, and observe for longer than one normal poll interval plus its request timeout. Expand to a small cohort, then a broader cohort, then the fleet only when the intent, attempt, and outcome graphs agree. Write stop conditions before starting: unexpected aggregate traffic, rising in-flight work, or stale outcomes should halt expansion. The exact cohort sizes depend on fleet topology and target capacity; I'm not sure a fixed percentage sequence is defensible without those two inputs.

Recovery deserves the same care as shutdown. Keep jitter on the first run after enable, because releasing every client at once can recreate the burst that the switch was meant to contain. Watch outcome freshness while traffic returns. Fast traffic recovery with no outcome recovery is a reason to pause, inspect the monitored workflow, and avoid adding more probes.

The catch is that a remote feature flag is not suitable when its cached state cannot be trusted or when configuration propagation is slower than the required shutdown. In that situation, use a local scheduler or deployment control that can stop the workload directly. A remote switch is also the wrong sole defense for a safety-critical monitor whose absence is more dangerous than extra load; bounded admission and target-side protection must carry that case.

Troubleshoot in a fixed order. First inspect the local enabled value, snapshot age, and generation. Next count scheduled callbacks and active requests. Then compare fleet-wide attempts with accepted outcomes. Finally inspect retry levels and target saturation. If attempts continue after disable, another scheduling owner exists. If attempts fall to zero but the enabled gauge remains one, flag delivery or local evaluation is stale. If successful HTTP responses rise while heartbeat age keeps rising, the transport is healthy and the monitored workflow is not producing the expected outcome.

Keep it boring. One owner, one clock, one active request, and three layers of evidence make the kill switch predictable under pressure.

## References

- Google SRE Book, "Monitoring Distributed Systems": https://sre.google/sre-book/monitoring-distributed-systems/
- Electron documentation, "crashReporter": https://www.electronjs.org/docs/latest/api/crash-reporter
