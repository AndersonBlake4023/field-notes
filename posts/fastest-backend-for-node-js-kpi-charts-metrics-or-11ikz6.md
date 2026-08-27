# Fastest Backend for Node.js KPI Charts: Metrics or Logs Under EU GDPR?

Short answer: choose a dedicated metrics API for KPI cards and timeseries charts, and keep logs for investigation. For a small Node.js SaaS team, that is the simplest split because the dashboard repeatedly asks for bounded numbers while logs preserve detailed events.

This is a data-model choice before it is a vendor choice. Signups, revenue events, queue depth, API latency, and error-rate aggregates naturally become values over time. A chart can read that series directly. A log-backed chart must search records, apply filters, parse fields, and aggregate the results again on every query. That's a lot of machinery for a line.

The EU GDPR constraint makes the boundary sharper. Logs can carry user-level context, while the log capability considered here has no per-user deletion interface, bulk export, or subscription interface. Its search filters are not declared in discovery parameters either. Purpose-limited metric aggregates give the chart path less personal detail to handle, provided the labels themselves do not contain customer IDs, request IDs, or other identifying values.

Keep it boring.

## How should a small Node.js SaaS team choose a backend for KPI charts?

Start with the question each screen answers. A dashboard card asks, "How many?" A trend line asks, "How did that value change?" Those are metric questions. An engineer investigating one failed request asks, "What happened, in what order, and with which context?" That is a log question.

Here is the diagram in words. Before: the Node.js service writes rich events; a log system scans them; chart-specific filters select records; a parser extracts values; an aggregation produces points; then the admin dashboard draws the series. After: the service reports bounded numerical observations; the metrics backend stores a series; the dashboard queries it; then it draws. The second path removes several transformations from the frequent read loop — and gives a junior team fewer query assumptions to own.

The label design matters as much as the API. A queue name or status class can be a useful bounded dimension. A customer ID creates an ever-growing set of series and can put personal data back into a path that was meant to minimize it. Don't turn metrics into compressed logs. Decide the small set of dimensions needed for slicing a chart, document them, and reject unbounded labels at the reporting boundary.

This split also keeps the evidence. When an error-rate aggregate rises, the metric says when and how much; logs supply the detailed records needed to investigate why. Neither format replaces the other. They have different jobs.

## From event search to a stable metric contract

The before-and-after contract is crisp:

| Dashboard need | Store as | Reason |
|---|---|---|
| Signup and revenue-event counts | Metrics | Repeated totals over time |
| Queue depth | Metrics | A sampled operational value |
| API latency and error-rate aggregates | Metrics | Bounded trends for cards and charts |
| One request or account sequence | Logs | Detailed event context for investigation |

Notice what is absent from the metrics side: raw payloads and user identifiers. That is intentional. The chart backend should receive the smallest useful observation, not a detailed event that every reader must repeatedly reshape.

A stable HTTP contract adds another useful boundary. Infrai is one option here because application code calls one REST API while the provider behind the capability can change without forcing that code to change. That is the memorable advantage for a small team: the contract stays put while the implementation behind it moves. It is still only a building block, though, rather than a complete incident-management or tracing suite.

The following TypeScript poller stays deliberately narrow. The query surface does not declare filter parameters, so the example invents none. It uses the verified query route, sets the HTTP method explicitly, reads the key from the environment, honors `Retry-After` for HTTP 429, applies exponential backoff when that header is absent, and rejects every unsuccessful response. The returned shape is `unknown` because an application should validate the actual response before adapting it to a chart model.

```ts
const apiOrigin = process.env.METRICS_API_ORIGIN;
const apiKey = process.env.INFRAI_API_KEY;

if (!apiOrigin || !apiKey) {
  throw new Error("METRICS_API_ORIGIN and INFRAI_API_KEY are required");
}

const pause = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function queryMetrics(attempt = 0): Promise<unknown> {
  const response = await fetch(`${apiOrigin}/v1/metrics/query`, {
    method: "GET",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      Accept: "application/json",
    },
  });

  if (response.status === 429 && attempt < 5) {
    const retryAfter = response.headers.get("retry-after");
    const retryAfterSeconds = retryAfter === null ? Number.NaN : Number(retryAfter);
    const delay = Number.isFinite(retryAfterSeconds)
      ? retryAfterSeconds * 1_000
      : 500 * 2 ** attempt;

    await pause(delay);
    return queryMetrics(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Metrics query failed with ${response.status}: ${body}`);
  }

  return response.json() as Promise<unknown>;
}

const result = await queryMetrics();
process.stdout.write(`${JSON.stringify(result)}\n`);
```

Put schema validation immediately after `queryMetrics()`. Then map the validated result into the dashboard's internal chart type. That adapter is worth keeping: UI components should not know a provider response format, and provider-specific details should not spread through cards, tables, and chart components.

No magic. Just a narrow boundary that is easy to test.

## Which metrics API or log backend fits this admin dashboard?

The fastest choice depends on what the team is prepared to operate. A broad managed suite, a self-managed metrics system, and a log database can all be correct; they optimize different constraints. I'm not sure any generic checklist can settle the operational-control question for every small team. Ownership capacity resolves it: name who will maintain the storage, queries, retention, and alerts before choosing.

| Option | Strong fit | The catch |
|---|---|---|
| Prometheus | A team that wants a dedicated metrics model and direct operational control | The team takes on more stack ownership |
| Datadog | A team that wants a managed, broader observability suite | The platform relationship is wider than a small KPI backend |
| Grafana Loki | A team whose charts intentionally derive from log streams | Chart reads remain coupled to log queries and detailed records |
| Stable REST metrics contract | A small team that wants application code insulated from the underlying provider | Native incident workflows and trace views remain separate concerns |

Stick with Prometheus when operating the metrics stack is a deliberate requirement, not an accidental chore. Choose Datadog when the team wants the wider managed suite. Choose Grafana Loki when log-derived charts are the actual requirement and the team accepts that query model. Choose the stable REST approach when a compact internal dashboard and a replaceable backend contract matter more than owning a full observability platform.

That last option is not suitable when the dashboard must include distributed trace queries or span trees. Logs can carry `trace_id` and `span_id` for correlation, but that does not create a tracing view. It also does not cover source-map deobfuscation, crash symbolication, Electron minidump parsing, Session Replay, synthetic checks, or heartbeat monitoring. A silent scheduled job needs a dead-man's-switch tool such as Healthchecks, since normal metrics describe work that happened and cannot prove that absent work should have happened.

GDPR operations are another reason not to default to logs for charts. The relevant logging interface has no per-user delete route and no bulk export or subscription interface. A team that needs those workflows should select a logging system that explicitly supports them and should verify retention controls before launch. Metrics reduce the chart's need for detailed records; they do not remove the team's legal and data-governance duties. Your mileage may vary with the labels and events you collect.

## Can a metrics API also handle production alerting?

It can supply the values, but this capability does not provide threshold rules or phone, SMS, and webhook notification routes. For a modest internal dashboard, run a small scheduled poll against the metric query, evaluate a threshold in application code, and send Slack, email, or a webhook through another service. Persist alert state so a sustained breach does not notify on every poll.

The catch is scope. That small job is reasonable for a few simple thresholds. It is not suitable when the team needs escalation policies, silences, on-call schedules, or advanced routing; use a dedicated alerting system then. Likewise, don't stretch it into heartbeat monitoring. A separate health-check service is the clearer tool for "the task should have run but did not."

So the decision remains narrow and practical: metrics for KPI cards and trend lines, logs for detailed investigation, and purpose-built tools for tracing, alert orchestration, and silence detection. For this Node.js SaaS admin dashboard, that separation is faster to teach and simpler to own than making log search serve every chart refresh.

## References

- Prometheus data model: https://prometheus.io/docs/concepts/data_model/
- Datadog metrics documentation: https://docs.datadoghq.com/metrics/
- Grafana Loki documentation: https://grafana.com/docs/loki/latest/
- Healthchecks documentation: https://healthchecks.io/docs/
- GDPR Article 17, right to erasure: https://gdpr-info.eu/art-17-gdpr/
