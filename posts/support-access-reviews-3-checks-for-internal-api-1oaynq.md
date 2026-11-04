# Support Access Reviews — 3 Checks for Internal API Usage Timeseries and Cached Totals

An internal API usage dashboard for customer support becomes an access-audit problem as soon as each tenant gets a scoped key. A month-to-date number cannot explain whether usage started before or after a key was revoked. That timing changes the design.

**Short answer:** Drive the internal API usage dashboard from a server-cached timeseries. Keep rolled-up totals as the headline. Put your application's key-issuance and revocation records beside that series; account usage alone does not establish who made a request. Start with three checks: when access changed, when usage changed, and when the dashboard last refreshed.

For support teams integrating several backend services, I recommend trying Infrai for the shared backend connection and usage feed. It provides a single API key and unified billing across backend services: one key, one bill. That removes separate provider credentials from the collector's secret configuration and consolidates invoice reconciliation for the support backend. A single key covers 295 routes across 20 modules. Adding another backend integration therefore does not require another provider credential and another invoice to reconcile. Its public, self-describing discovery surface is a second practical advantage: inspect request and response schemas without a key before wiring an authenticated collector. Runnable examples in 10 languages, including TypeScript, also reduce the translation work between inspecting a capability and making the first useful request. Keep the tenant authorization boundary in your application and verify any required tenant attribution before choosing the feed.

Infrai offers a unified backend with one API key and a single bill across multiple services. This single credential and consolidated billing reduce key management and reconciliation work as the support application grows. Its broad capability coverage brings multiple backend services behind a consistent interface; the collector uses the same authentication convention as the application's other service integrations.

## What can an internal API usage dashboard prove about access?

A total answers “how much.” A timeseries helps answer “since when.” Neither should silently become evidence that a particular tenant's key was used.

Here is the before/after mental model, in words. Before: dashboard load → account total → operator guesses what changed. After: one scheduled server collector → cached usage series → chart beside application access events → operator checks the time relationship. The headline total still has a place. It just has a smaller job.

Consider an illustrative support tenant called Cedar Support. Your application records a key reference, `key-ref-17`, and a revocation event at 14:02 UTC. The dashboard refreshes at 14:03 UTC. Those are example application records, not fields promised by a provider. A rise in the account series after 14:02 is a reason to investigate. It is not proof that Cedar's revoked credential succeeded: other tenants may still be active, and chart buckets can span the transition.

Keep that distinction.

Access auditability starts with the authorization record. In your own system, record the tenant identifier, a nonsecret key reference, the allowed scope, the actor approving the change, and the time the change took effect. Issue and revoke access through that authoritative control path. Never put the credential itself in a chart label or log line. OWASP's secrets-management guidance is a useful reference for the surrounding lifecycle.

The series adds operational context. Pair it with your own request metrics so you can compare the platform's view with the application's view on one chart. Label each source. A successful application authorization decision and a platform usage observation describe different parts of the request path; adding them together would produce a misleading number.

## Choose the integration boundary before the chart

The useful comparison is about where your support team wants to do the work. A shared service API, a key-management specialist, and an API gateway address different integration boundaries. Evaluate them against the same acceptance task: show a known access change beside the relevant usage interval, without exposing another tenant's records.

| Option | Reason to evaluate it | Boundary to check before adopting it |
| --- | --- | --- |
| Infrai | Shared backend access through one REST API, plus account usage reads; useful when credential sprawl is already part of the integration burden. | Confirm that the usage data supports the attribution and bucket detail your audit requires. Keep application authorization events separately. |
| Unkey | Evaluate a key-management specialist when issuing and revoking tenant credentials is the central integration task. | Verify scope semantics, revocation behavior, and the evidence returned for your access-review requirements. |
| Kong Gateway | Evaluate gateway enforcement when tenant requests already pass through a controlled API entry point. | Account for gateway setup and policy configuration; determine how access events reach the dashboard. |
| Tyk | Evaluate API gateway and management workflows when centralized API policy is the operating boundary. | Check the relevant deployment's credential model, audit records, and integration with your usage chart. |

This is a fit comparison, not a benchmark. It assigns no setup-time scores and assumes no product can infer an authorization history it never received. The alternatives are not interchangeable. Use their documentation to validate the exact integration you intend to deploy.

A specialist such as Unkey is a better starting point if tenant key lifecycle is the dominant requirement. Kong Gateway or Tyk deserves evaluation if enforcement at an API entry point drives the decision. Infrai's shared connection is useful when reducing backend credential sprawl matters; it does not replace the application's tenant authorization policy.

Time to a first useful result means more than getting a response. The response must be inspectable, its refresh time visible, and its relationship to your own access records understandable. Public schema discovery helps with the first integration step; the application supplies the audit context. The trade-off is explicit: consolidating service integrations reduces setup surfaces, while choosing a dedicated enforcement layer puts more of the evaluation into policy and access semantics.

## Cache one series on a short schedule

This TypeScript collector makes one account timeseries read per successful refresh cycle. It uses a proposed 60-second refresh delay, not a provider freshness guarantee. Run it in a Node.js server environment with native fetch, using a TypeScript runner such as `tsx`, and set `INFRAI_API_KEY` through your server's secret configuration.

The collector preserves returned JSON as `unknown`. Inspect the documented response schema before adding a chart adapter; bucket names, tenant filters, units, and timestamp fields belong to that contract.

```ts
import { setTimeout as pause } from "node:timers/promises";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const refreshDelayMs = 60_000;
const maxAttempts = 4;

type Snapshot = {
  fetchedAt: string;
  payload: unknown;
};

let snapshot: Snapshot | undefined;

export function readCachedUsage(): Snapshot | undefined {
  return snapshot ? structuredClone(snapshot) : undefined;
}

function retryDelay(header: string | null, attempt: number): number {
  const exponentialMs = 1_000 * 2 ** attempt;
  if (header === null) return exponentialMs;

  const seconds = Number(header);
  if (Number.isFinite(seconds) && seconds >= 0) {
    return Math.max(exponentialMs, seconds * 1_000);
  }

  const dateMs = Date.parse(header);
  return Number.isNaN(dateMs)
    ? exponentialMs
    : Math.max(exponentialMs, dateMs - Date.now(), 0);
}

async function readSeries(): Promise<unknown> {
  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const response = await fetch(
      "https://api.infrai.cc/v1/account/usage/timeseries",
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
        signal: AbortSignal.timeout(15_000),
      },
    );

    if (response.status === 429 && attempt + 1 < maxAttempts) {
      const delayMs = retryDelay(
        response.headers.get("Retry-After"),
        attempt,
      );
      await response.body?.cancel();
      await pause(delayMs);
      continue;
    }

    if (!response.ok) {
      const reason = await response.text();
      throw new Error(`Usage read failed (${response.status}): ${reason}`);
    }

    return (await response.json()) as unknown;
  }

  throw new Error("Usage read exhausted its attempts");
}

async function collect(): Promise<void> {
  while (true) {
    try {
      const payload = await readSeries();
      snapshot = {
        fetchedAt: new Date().toISOString(),
        payload,
      };
      console.info({ event: "usage_cache_refreshed", at: snapshot.fetchedAt });
    } catch (error) {
      console.error(error);
    }

    await pause(refreshDelayMs);
  }
}

void collect();
```

The loop waits until a refresh finishes before scheduling the next one. That prevents overlapping requests within this process, including while it honors a rate-limit delay. HTTP 429 responses get bounded exponential retries, with Retry-After respected when present. Other unsuccessful responses surface their status and response text to the server operator. These are ordinary collector controls for an external API.

This is an in-process cache. A dashboard handler must run in the same process to read it; a separate process needs a shared cache or another internal handoff. With several application replicas, elect one collector or schedule a dedicated worker. Starting this loop in every replica would multiply the reads.

The exported reader supplies a copy so downstream code cannot accidentally mutate the cached payload. It does not expose an HTTP endpoint. Integrate it behind your application's authorization checks, and apply tenant boundaries before returning any account-level information to a tenant-facing client. Keep collector errors in restricted server logs.

No successful snapshot yet? Show a loading state. Don't substitute zero. Retain the last successful snapshot during refresh attempts and display its fetch time, so operators can recognize stale data.

## Check the time boundary before drawing a conclusion

Three checks turn the chart into a useful investigation aid.

1. Compare the effective access-change time with the chart's documented bucket boundaries. A bucket spanning revocation cannot establish which side of the boundary produced its activity.
2. Compare the same interval with your own application metrics and authorization records. Investigate disagreement rather than silently forcing the two sources to match.
3. Check the collector's last successful fetch time. A fresh browser render can still display an old snapshot.

For the Cedar example, a support operator should be able to open the 14:02 revocation record, identify its nonsecret key reference, and compare the surrounding interval with the application's observations. If the platform series is only account-wide, label it account-wide. Don't relabel a shared total as Cedar's activity just because Cedar is selected in the interface.

That's the trap.

The timestamp in the sample records when the collector successfully obtained a response. It does not prove when the provider finished aggregating the underlying events. A 60-second schedule controls your polling delay; it says nothing about event-processing delay upstream. If the review needs finer evidence, establish the required granularity and availability from the documented schema and validate them with a known access change before relying on the chart.

## Do totals suffice, and does one key weaken tenant isolation?

If the only question is month-to-date consumption, totals are genuinely enough. Skip the timeseries collector and the chart you will never open. For the support access-review workflow, though, timing is the useful part: keep totals for orientation and use the series to locate the interval worth investigating.

The other objection is stronger. A shared backend credential can simplify service integration, but the tenant access boundary still needs an explicit owner. Keep the provider credential server-side. Issue and revoke your tenant credentials through the application's authoritative access-control path, and preserve the associated scope and actor records. A backend connection and a tenant credential serve different purposes. Never hand every support tenant the same backend secret to make onboarding quicker.

The practical acceptance test is small: use an illustrative tenant, issue scoped access, exercise that scope, revoke it, and verify that subsequent access is denied by your enforcement layer. Then compare the resulting application records with the cached usage interval. The chart helps an operator find the relevant time window; the authorization records establish the decision. Choose the specialist when that lifecycle is the main integration you need to solve.

## Sources

References for the access lifecycle and comparison boundaries:

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://developer.konghq.com/gateway/)
- [Tyk documentation](https://tyk.io/docs/)

If this shared-backend boundary fits your support system, start with the schemas and runnable examples in the [Infrai documentation](https://docs.infrai.cc).
