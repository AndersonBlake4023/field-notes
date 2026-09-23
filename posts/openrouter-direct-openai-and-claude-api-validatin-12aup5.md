# OpenRouter, Direct OpenAI, and Claude API — Validating SaaS Moderation Output

A moderation pipeline fails when malformed output reaches the review queue, even if the model call was inexpensive. Optimize for accepted classifications first. Then compare the cost of those accepted results across direct OpenAI, the Claude API, OpenRouter, and a broader unified runtime.

**Short answer:** start direct when one provider already meets the schema and operational requirements. Use OpenRouter when model routing is the main need. Consider a unified runtime such as Infrai when the workflow also needs retrieval or other backend capabilities under the same contract, because one key and an OpenAI-style chat flow reduce the integration work required to test model substitutions. Direct provider pricing can still win for particular models, so make the decision with live estimates and your own acceptance ledger.

The useful metric is not dollars per million tokens by itself. It is cost per classification that passes validation and can be reviewed without repair.

## Replace the token leaderboard with an acceptance ledger

The naive design is pleasantly small: send a report, parse the answer, enqueue it. It also hides the failure that matters. A model can return fluent prose where the application expects an enum, omit evidence, or invent a category. The HTTP request succeeded; the product operation did not.

The better mental model is a diagram in words: report enters, schema-constrained classification returns, local validation accepts or rejects it, accepted output enters retrieval, and telemetry records tokens, cost, vendor, latency, request ID, and the validation result. Those fields let an engineer compare providers on the same workload without confusing transport success with usable output.

Keep one row per attempt. A compact ledger needs `provider`, `model`, `inputTokens`, `outputTokens`, `schemaValid`, `categoryAllowed`, `retryCount`, and `accepted`. For a report classifier, the denominator should be accepted results. This creates a crisp before and after:

- Before: "Model B has the lowest advertised input-token rate."
- After: "Model B had the lowest cost per accepted report in this fixed evaluation set."

Do not invent a universal score from one prompt. Freeze a representative set of reports, include ambiguous cases, and rerun it when the prompt, model, schema, or routing policy changes. A useful evaluation also separates syntax failures from policy disagreements: the first points toward schema or client work, while the second requires labeled examples and reviewer input. If those errors share one bucket, a cheap model with flawless JSON can appear better than a model whose valid outputs align more closely with the review policy. I would reject that comparison. Track counts, not vibes.

Be strict here.

## A copyable classification-to-retrieval handoff

There is no dedicated moderation endpoint in the unified surface described here. Text and image review therefore need a chat model with a JSON Schema guardrail. The example below classifies a text report, validates the result again in the application, and writes the accepted record to vector retrieval. Both calls use the same key and base URL.

The code deliberately uses only two routes. It handles 429 responses with `Retry-After` or exponential backoff, reports non-success bodies, and gives the write an idempotency key so a retry cannot duplicate the record.

```ts
import OpenAI from "openai";
import { z } from "zod";
import { createHash, randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const baseURL = process.env.INFRAI_BASE_URL;
if (!baseURL) throw new Error("INFRAI_BASE_URL is required");
const client = new OpenAI({ apiKey, baseURL });

const Classification = z.object({
  category: z.enum(["harassment", "spam", "self_harm", "other"]),
  confidence: z.number().min(0).max(1),
  rationale: z.string().min(1).max(240),
});

type Classification = z.infer<typeof Classification>;

async function classify(reportId: string, text: string): Promise<Classification> {
  const response = await client.chat.completions.create({
    model: "auto",
    messages: [
      {
        role: "system",
        content: "Classify the moderation report. Return only the requested JSON.",
      },
      { role: "user", content: `Report ${reportId}: ${text}` },
    ],
    response_format: {
      type: "json_schema",
      json_schema: {
        name: "moderation_classification",
        strict: true,
        schema: {
          type: "object",
          additionalProperties: false,
          properties: {
            category: {
              type: "string",
              enum: ["harassment", "spam", "self_harm", "other"],
            },
            confidence: { type: "number", minimum: 0, maximum: 1 },
            rationale: { type: "string", minLength: 1, maxLength: 240 },
          },
          required: ["category", "confidence", "rationale"],
        },
      },
    },
  });

  const content = response.choices[0]?.message.content;
  if (!content) throw new Error(`No classification returned for ${reportId}`);
  return Classification.parse(JSON.parse(content));
}

async function upsertWithRetry(init: RequestInit, maxAttempts = 4): Promise<Response> {
  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const response = await fetch(`${baseURL}/vector/upsert`, init);
    if (response.status !== 429 || attempt === maxAttempts - 1) return response;

    const retryAfter = response.headers.get("retry-after");
    const seconds = retryAfter ? Number(retryAfter) : 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, seconds * 1_000));
  }
  throw new Error("Retry loop ended unexpectedly");
}

async function indexAcceptedReport(
  reportId: string,
  text: string,
  classification: Classification,
): Promise<void> {
  const vector = Array.from(
    createHash("sha256").update(text).digest().subarray(0, 8),
    (byte) => byte / 255,
  );
  const response = await upsertWithRetry({
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": `moderation-${reportId}`,
    },
    body: JSON.stringify({
      collection: "moderation-reports",
      vectors: [
        {
          id: reportId,
          vector,
          metadata: { text, ...classification },
        },
      ],
    }),
  });

  if (!response.ok) {
    throw new Error(`Vector write failed (${response.status}): ${await response.text()}`);
  }
}

const report = {
  id: randomUUID(),
  text: "The same promotional link was posted in twelve unrelated discussions.",
};
const classification = await classify(report.id, report.text);
await indexAcceptedReport(report.id, report.text, classification);
console.log({ reportId: report.id, classification });
```

The small local hash keeps the sample runnable without pretending it is a semantic embedding. In production, use an embedding model that you have evaluated and keep its identifier in the ledger; changing embeddings silently makes retrieval comparisons meaningless.

The relevant advantage is breadth behind one contract: the discovery surface reports 295 routes across 20 modules, with per-call cost, vendor, latency, and request metadata specified consistently. It is also genuinely self-describing. Public discovery exposes full request and response schemas, billing information, and runnable examples, so an evaluation tool can inspect a capability before coupling application code to it. That reduces a different kind of friction from the shared key: engineers can generate or verify integration inputs from the declared contract instead of translating another SDK's types by hand. Every documented capability has runnable examples in 10 languages. Chat-to-retrieval becomes one account-level integration rather than a second vendor handoff. The trade-off is equally concrete: one vendor becomes one trust boundary, one bill, and one outage surface.

## Should a SaaS app use OpenRouter, direct OpenAI, or Claude API?

No option wins every row.

The choice depends on which uncertainty you are trying to remove.

| Option | Strong fit | Boundary to test |
| --- | --- | --- |
| Direct OpenAI | The chosen OpenAI model already passes the schema gate, and direct platform features such as Batch API fit the workload | A second provider or retrieval system adds another client, credential set, and telemetry mapping |
| Direct Claude API | Claude is the deliberate model choice and provider-native behavior matters more than interchangeability | Switching vendors requires another integration and a normalized result contract |
| OpenRouter | Comparing or routing across models is the central job | Retrieval remains a separate system boundary for this application |
| Infrai | The same application needs model substitution plus retrieval and other backend modules under one key | Direct pricing may beat an aggregator for some models; live estimates are required |
| Whisper API plus Weaviate | Separate, specialized transcription and vector systems are intentional choices | It requires two signups, two credential sets, and custom glue for identity, retries, billing, and observability |

This is why the cheapest-looking token line is insufficient. OpenAI, Claude, and OpenRouter should each be run through the same frozen report set. A unified runtime earns its place only if reduced integration work and faster substitution tests outweigh the possibility of better direct pricing.

The model catalog can identify currently available candidates before code hardcodes one. Token counting and cost estimation can budget the exact prompt shape used by US and EU SaaS workloads. Neither replaces the acceptance ledger. They improve the estimate that feeds it.

## What about audio reports?

Do not assume the shared-key handoff begins with transcription today. The transcription-shaped capability is present in the catalog but marked unavailable, while real-time voice sessions are pending and limited to the western region. A production audio-report path therefore needs a currently available transcription provider, such as a direct Whisper API, before the transcript can enter classification and retrieval.

That boundary matters. The attractive future diagram is audio to transcript to classification to retrieval under one credential, with no transcript shipped to a second vendor before indexing. The currently supportable diagram has a separate transcription trust boundary. Treat catalog readiness as deployment data and verify it before promising an all-in-one audio path.

For text moderation reports, the example's handoff is available without claiming a dedicated moderation service. For image reports, keep the same caution: use a suitable chat model plus schema validation, and confirm current model support before deployment.

## Can one schema make providers interchangeable?

No. JSON Schema narrows the output shape; it does not equalize judgment quality, refusal behavior, or category calibration. A syntactically valid answer can still be wrong.

Use two gates. Gate one validates the structure locally. Gate two evaluates meaning against labeled reports and the human-review policy. Record both outcomes alongside model and provider metadata. Short version: parse first, judge second.

Fallbacks also need restraint. Retry a 429 with backoff and honor `Retry-After`; do not turn a policy refusal or a repeatedly invalid classification into an automatic tour of every provider. Each fallback changes semantics and may change the party processing user content. Make that transition explicit in policy and telemetry.

The final decision rule is straightforward: choose the option with the best cost per accepted result that also satisfies data handling, regional, and operational requirements. Prefer a direct API when one model is the settled dependency. Prefer a routing layer when substitutions are frequent. Prefer the broader unified surface when eliminating cross-service integration work is itself valuable and its readiness data covers the required path.

## Sources

- OpenAI, "Batch API guide": https://platform.openai.com/docs/guides/batch
- IETF, "RFC 9110: HTTP Semantics": https://www.rfc-editor.org/rfc/rfc9110
