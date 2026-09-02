# Incident Evidence for Next.js Backend Routes: Error Capture Without Replay

Counting how often a route throws and reconstructing what happened to one school district on Tuesday afternoon are different jobs, and self-serve exception tracking is tuned for the first one. In short: for Next.js backend routes, a lightweight error capture API holds up fine as evidence infrastructure — as long as every event carries a tenant id, a correlation id, a build id, and a retention class you picked on purpose. Grouping is triage. Reconstruction is evidence. The two answers need two different indexes, and that, rather than the feature matrix, is what should decide the tool.

The running example here is an edtech gradebook. A district admin exported final grades three days ago, the CSV came back with 14 of 400 rows missing, the ticket landed today, and nobody can tell them what happened.

| Option | Pick it when | What you carry |
| --- | --- | --- |
| Capture endpoint you own (one POST, one table) | Failures are server-side and the questions are tenant-scoped | Schema, retention, purge, and your own alert polling |
| Full-suite error platform (Sentry, Datadog and peers) | The browser is part of the reconstruction: sourcemaps, release health, session replay | A much wider surface than backend routes need, plus a quota model to watch |
| Structured logs in a query engine | You need the lines around the throw, not only the throw | Log volume, index cost, one retention policy per stream |
| OpenTelemetry logs and traces into your own store | Spans already exist and you want one correlation id everywhere | A collector to operate and schema discipline to enforce |
| Heartbeat or absence monitor | The failure mode is "the nightly export never ran at all" | A second system — but it catches what exception tracking structurally cannot |

## What does error capture in Next.js backend routes need to retain to reconstruct one customer's incident?

Six fields, and the exception is only one of them:

- `district_id` (or whatever your tenant key is) — the axis every support ticket actually starts from
- `correlation_id` — the W3C trace-context trace id, minted at the edge when the caller doesn't send one
- `build_id` — which deployed artifact produced this stack
- `occurred_at` in UTC, plus the timezone the reporter was in
- the operation name, not just the URL path — one route often serves three jobs
- input shapes: `sectionIds: string[24]`, never the 24 values

Nothing there is exotic. What's unusual is insisting on all six at write time, because you cannot backfill evidence into an incident that already happened.

Grouping indexes by fingerprint: same error name, same top frames, same release. That is the right index for "is this getting worse". It is the wrong index for the gradebook ticket, where the query you need is closer to `WHERE district_id = '4821' AND occurred_at BETWEEN ...`. Plenty of self-serve tools store a tenant field happily and then give you no way to query by it — search is scoped to the group, and the group is scoped to the stack. Check that one thing before anything else. Open the tool, pick a customer, and try to list every event they touched in a ten-minute window. If that takes more than one query, the tool is a counter, not a record.

Sourcemaps and replay are the other half of the reader's question, and they fall away faster than people expect on the server side. Session replay records a browser; a nightly export triggered by a scheduler has no browser to record, so there is nothing for replay to reconstruct — the input row set and the code version are the entire state you need. Sourcemaps matter when your stack frames point into minified bundles, which is a client-side condition; for server output, Node has read source maps natively since the `--enable-source-maps` flag shipped, so readable frames are a runtime option rather than an upload pipeline you have to buy into. Ship the build with its maps, start the server with the flag, record the build id on every event, and your stack traces stay legible without a vendor-side symbolication step:

```bash
NODE_OPTIONS="--enable-source-maps" node .next/standalone/server.js
```

That's the whole trick.

## Five options, and the question each one is built to answer

A capture endpoint you own is the right size when failures are server-side, questions are tenant-scoped, and you can name the four or five operations that matter. One POST, one append-only table, two indexes. You'll write the alerting yourself, which is a real cost, and I'd rather pay it than reshape my incident data to fit somebody else's grouping model.

A full-suite platform earns its price the moment the browser is part of the story. Frontend reconstruction is a genuinely hard product — sourcemap ingestion, release tracking, replay storage — and a backend capture table is not a cheap version of it. It's a different category. Reach for the suite when your incidents live in the client.

Structured logs sit underneath both. An exception record tells you a throw happened; the log lines around it tell you the export had already skipped 14 rows before anything threw, which in the gradebook case is the actual finding. If you keep only one thing, keep the logs and derive error records from them — the reverse doesn't work.

OpenTelemetry is the neutral ground, and it's mostly a data-model decision rather than a vendor one. Its logs data model gives you a place to put trace ids, severity, and resource attributes that any backend can read, so a switch of storage later is a config change instead of a rewrite. The price is a collector in your topology and the discipline to keep attribute names stable across services.

Then there's absence. No error tracker can report a job that never started, because nothing ran to report it. A heartbeat monitor is a few lines and it covers the silent half of the failure space.

## One evidence boundary, written once

Route handler to evidence boundary to append-only store, with two indexes hanging off the store: one on `incident_key` for triage, one on `(district_id, occurred_at)` for reconstruction. Draw that on a whiteboard and the code writes itself. The point of the boundary is that no route handler decides what an incident record contains — one module does, and it's the module you audit when legal asks what you're storing about students.

```ts
// lib/incident-evidence.ts — the only place that decides what an incident record contains.
type Retention = "incident-90d" | "debug-7d";

export type EvidenceRecord = {
  incident_key: string;      // stable per failing operation: operation + error name + build
  correlation_id: string;    // W3C trace-context trace id
  build_id: string;
  route: string;
  operation: "grade-export" | "roster-sync" | "assignment-submit";
  district_id: string;
  actor_role: "teacher" | "admin" | "student" | "system";
  error_name: string;
  error_message: string;
  stack_head: string;        // top frames only
  input_shape: Record<string, string>;
  occurred_at: string;       // ISO 8601, UTC
  retention: Retention;
};

const ENDPOINT = process.env.EVIDENCE_ENDPOINT;
const TOKEN = process.env.EVIDENCE_TOKEN;

// "sectionIds: string[24]" tells you what to reproduce without storing one student name.
export function shapeOf(input: Record<string, unknown>): Record<string, string> {
  const shape: Record<string, string> = {};
  for (const [key, value] of Object.entries(input)) {
    if (Array.isArray(value)) {
      shape[key] = `${value.length === 0 ? "empty" : typeof value[0]}[${value.length}]`;
      continue;
    }
    shape[key] = value === null ? "null" : typeof value;
  }
  return shape;
}

export async function recordEvidence(record: EvidenceRecord): Promise<void> {
  if (!ENDPOINT || !TOKEN) throw new Error("EVIDENCE_ENDPOINT and EVIDENCE_TOKEN are required");

  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch(ENDPOINT, {
      method: "POST",
      headers: {
        authorization: `Bearer ${TOKEN}`,
        "content-type": "application/json",
        // Same operation, same minute, same build: one record, not three.
        "idempotency-key": `${record.incident_key}:${record.occurred_at.slice(0, 16)}`,
      },
      body: JSON.stringify(record),
    });

    if (response.ok) return;
    if (attempt === 2) throw new Error(`evidence write rejected with ${response.status}`);
    await new Promise((resolve) => setTimeout(resolve, 250 * 2 ** attempt));
  }
}
```

The handler stays boring, which is the goal:

```ts
// app/api/grade-export/route.ts
import { recordEvidence, shapeOf } from "@/lib/incident-evidence";

function traceIdFrom(traceparent: string | null): string {
  const parts = (traceparent ?? "").split("-");
  return parts.length === 4 && parts[1].length === 32 ? parts[1] : crypto.randomUUID();
}

export async function POST(request: Request): Promise<Response> {
  const correlationId = traceIdFrom(request.headers.get("traceparent"));
  const body = (await request.json()) as { districtId: string; sectionIds: string[] };
  const buildId = process.env.BUILD_ID ?? "dev";

  try {
    const csv = await buildGradeExport(body);
    return new Response(csv, { headers: { "x-correlation-id": correlationId } });
  } catch (cause) {
    const err = cause instanceof Error ? cause : new Error(String(cause));

    await recordEvidence({
      incident_key: `grade-export:${err.name}:${buildId}`,
      correlation_id: correlationId,
      build_id: buildId,
      route: "/api/grade-export",
      operation: "grade-export",
      district_id: body.districtId,
      actor_role: "admin",
      error_name: err.name,
      error_message: err.message,
      stack_head: (err.stack ?? "").split("\n").slice(0, 6).join("\n"),
      input_shape: shapeOf(body),
      occurred_at: new Date().toISOString(),
      retention: "incident-90d",
    }).catch((writeError) => console.error("evidence write skipped", { correlationId, writeError }));

    return Response.json(
      { error: "grade export could not be completed", correlation_id: correlationId },
      { status: 500 },
    );
  }
}
```

Three decisions in there are worth defending. The capture call is wrapped in its own `catch` so a problem reaching the evidence store never swallows the response the admin is waiting for — the ticket says "export was slow and then errored", not "export hung for 45 seconds while we retried a logging call three times". The idempotency key is built from the operation and the minute rather than a fresh UUID, so a retried write collapses into one record instead of inflating the count you'll later report to the district. And `input_shape` exists because storing the roster itself turns an error table into student data with a retention obligation attached, which is a much larger conversation than error capture. Your mileage may vary on the three attempts; if the route is user-facing and latency-sensitive, hand the record to a queue and let a consumer do the retrying — same idempotency key, different process.

## The cost of keeping evidence for 90 days

Retention is the whole budget, so do the arithmetic before you pick a number. At roughly 2 KB per record and 500 failures a day, ninety days of `incident-90d` is about 90 MB — nothing. The `debug-7d` class is where the chatty stuff goes, and it's the class you sample. Sample successes, never failures: a 1% sample of a failing path is how you end up telling a district you have no record of their export.

Keep the high-cardinality fields out of your metrics. Every distinct label value is another time series, which is the oldest warning in instrumentation practice, and `district_id` will happily produce four thousand of them. Aggregates answer a different question anyway — Core Web Vitals are assessed at the 75th percentile of field data, and a p75 is a fact about a population, not about the admin who filed the ticket. Percentiles for trends, records for reconstruction.

Two operational details decide whether this holds up a year in. Someone has to own alerting: if the capture path has no notification hook, a scheduled reader queries unresolved groups and forwards to whatever already pages your team, and that reader needs its own heartbeat. And deletion has to be a single statement. If you can't purge by `district_id` in one query when a contract ends, your retention policy is a wish, not a control.

## When a capture endpoint you own is the wrong call

Stick with a full suite when the browser is part of the reconstruction. Sourcemap ingestion, release health, and session replay are real products; a table with a POST in front of it is not a smaller version of them, and you should not talk yourself into rebuilding them. Likewise, if engineers need span trees and distributed trace queries, correlation ids are a join key, not a tracing UI — that gap doesn't close with more fields.

A capture endpoint you own is also not suitable when the team can't absorb the operational tail: polling, escalation, retention jobs, schema changes. That work doesn't disappear, it just moves to you. And it's the wrong choice outright when a contract requires tamper-evident, immutable audit logs, since an append-only application table isn't the same claim as an audit log with integrity guarantees.

For the gradebook, though, the decision rule is narrow enough to state in one line: if you can answer "what happened to district 4821 between 14:05 and 14:20, and on which build" in a single query, the evidence layer is doing its job — and if you can't, no amount of grouping, replay, or dashboards will reconstruct the incident for you.

## Sources

- https://www.w3.org/TR/trace-context/
- https://nodejs.org/api/cli.html#--enable-source-maps
- https://opentelemetry.io/docs/specs/otel/logs/data-model/
- https://prometheus.io/docs/practices/instrumentation/
- https://web.dev/articles/vitals
- https://nextjs.org/docs/app/api-reference/file-conventions/route
