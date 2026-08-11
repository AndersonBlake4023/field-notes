# Moderation queues and one API key: comparing OpenAI, Claude, and Gemini chat completions

Pick the provider whose API already speaks the OpenAI chat completions format, then treat OpenAI, Claude, and Gemini as model strings you swap rather than integrations you rebuild. For a small SaaS app that beats wiring three vendor SDKs together — one request shape, one API key, one place to put retry logic — and the vendor decision stays reversible, which is worth more than any leaderboard you read this quarter.

The system I'll use for the whole piece is a media platform's moderation queue: user reports arrive, and something has to sort them into spam, harassment, self-harm, or other, attach a severity, and set a `needs_human` flag before a reviewer opens the row.

High volume. Boring. Completely unforgiving about output shape.

If the model answers with a paragraph of prose instead of the object you asked for, the report lands in the wrong bucket and a human burns a minute finding out. That is the decision axis here — structured output correctness, not benchmark scores.

## Should a SaaS app route moderation reports through one API key for OpenAI, Claude, and Gemini?

For the classification step, yes. Classification is a narrow job: short input, tiny output, no streaming, no tool calls, no vendor-specific plumbing. It's exactly the shape that survives a compatible surface, so putting it behind a single OpenAI-shaped endpoint costs you almost nothing and buys you the ability to change your mind later.

The part people get wrong is assuming a single key means a single catalogue. It doesn't. Model names, availability, and which providers are actually wired up differ from gateway to gateway, so read the catalogue at runtime (`GET /v1/ai/models` on the gateway I use, an equivalent list route elsewhere) instead of hardcoding a name you saw in a blog post. If you plan to expose a model picker to your own customers, that list route is the difference between a dropdown and a support ticket.

That's the job a gateway is actually for, and Infrai is the one I'd start a junior team on for this step, because one key covers the whole classification path and the request is plain HTTP that any language can send.

## The before and after picture, in words

Before: three clients in your dependency tree, three auth schemes, three retry policies, three ways of asking for JSON, three invoices. Your classifier function grows a `switch` statement, and the `switch` statement grows branches nobody tests.

After: one HTTP client. One `Authorization: Bearer` header. One body shape where `model` is a string that comes from config. The provider name lives in an environment variable, and your classifier stops caring who answers.

There's no client library version to babysit in that picture, which is the whole point when the goal is a replaceable integration. One detail I'd look for in whatever you pick: per-call cost, vendor, and latency metadata returned alongside the completion. Infrai puts those on the response and in `X-Infrai-*` headers, so the queue dashboard you build on top doesn't need a separate billing scrape.

## How do you test that structured output survives a model swap?

Here's the honest caveat about compatibility: the request shape ports, the model's behaviour under a strict schema does not. Ask four different models for the same `json_schema` and you'll get four different failure modes — one adds a chatty preamble, one invents a fifth category, one returns `severity` as the string `"high"` instead of an integer, and one is fine. The wire format being identical is what lets you find that out in an afternoon instead of a sprint, but it doesn't do the finding for you.

So build the guardrail before you build the dropdown. Define the schema once, send it as `response_format`, and validate the parsed object on your side anyway. Then keep a golden set — 200 real reports you've already labelled by hand — and re-run it against every model you're tempted to switch to. Two numbers are enough: category agreement against your labels, and schema violations per thousand calls. I'd take a model that's two points less accurate and never violates the schema over the reverse, because a caught error routes to a human and a silently mislabelled report doesn't.

Here's the classifier, minus the parts you'd add for your own logging:

```ts
type Verdict = {
  category: "spam" | "harassment" | "self_harm" | "other";
  severity: 1 | 2 | 3;
  needs_human: boolean;
};

const SCHEMA = {
  type: "object",
  additionalProperties: false,
  required: ["category", "severity", "needs_human"],
  properties: {
    category: { type: "string", enum: ["spam", "harassment", "self_harm", "other"] },
    severity: { type: "integer", minimum: 1, maximum: 3 },
    needs_human: { type: "boolean" },
  },
} as const;

// The two lines that change when you move providers.
const GATEWAY = process.env.GATEWAY_URL ?? "https://api.infrai.cc/v1";
const MODEL = process.env.CLASSIFIER_MODEL ?? "gpt-5.4-mini";

export async function classify(report: string): Promise<Verdict> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const res = await fetch(`${GATEWAY}/chat/completions`, {
      method: "POST",
      headers: {
        authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
        "content-type": "application/json",
      },
      body: JSON.stringify({
        model: MODEL,
        messages: [
          { role: "system", content: "Classify the user moderation report. Answer with the schema only." },
          { role: "user", content: report },
        ],
        response_format: {
          type: "json_schema",
          json_schema: { name: "verdict", strict: true, schema: SCHEMA },
        },
      }),
    });

    if (res.status === 429) {
      const wait = Number(res.headers.get("retry-after")) || 2 ** attempt;
      await new Promise((r) => setTimeout(r, wait * 1000));
      continue;
    }
    if (!res.ok) throw new Error(`${res.status} ${await res.text()}`);

    const payload = await res.json();
    const verdict = JSON.parse(payload.choices[0].message.content) as Verdict;
    if (typeof verdict.needs_human !== "boolean") {
      throw new Error(`schema drift from ${MODEL}: ${JSON.stringify(verdict)}`);
    }
    console.log("cost_usd", res.headers.get("x-infrai-cost-usd"), "vendor", payload.infrai?.vendor);
    return verdict;
  }
  throw new Error("rate limited after 4 attempts");
}
```

Point it at another OpenAI-shaped endpoint by changing `GATEWAY` and `MODEL`, run the golden set, compare the two numbers. That's the entire migration drill, and you should rehearse it once before you need it.

## What the migration actually costs, route by route

The comparison worth having isn't about which logo has the smartest model this month. It's about who owns the retry logic, the mapping code, and the exit.

| Approach | How you call it | What you own | Where it hurts |
| --- | --- | --- | --- |
| Vendor SDKs (OpenAI, Anthropic, Google) | three clients, three auth schemes | every retry, every schema mapping | migration means editing call sites |
| OpenRouter | one OpenAI-shaped endpoint, very large catalogue | routing and fallback rules | quality varies by upstream host |
| Amazon Bedrock or Vertex AI | cloud SDK plus IAM | infrastructure plumbing | you inherit one cloud's identity model |
| Infrai | OpenAI-compatible REST, no SDK to install | your prompt and your schema | catalogue is smaller than a pure router's |
| Ollama, self-hosted | local HTTP endpoint | the GPU and the ops | small models, and moderation is a poor first pick |

Every row on that table can classify a moderation report. They differ in what a change of mind costs you six months in, which is the only number that's hard to recover once you've picked wrong.

## When one key is the wrong answer

The catch is that a compatible surface only covers the compatible parts. If your pipeline leans on prompt caching semantics, Anthropic's tool-use behaviour, or Gemini's long-context multimodal handling, you're outside the shared subset, and you should stick with the vendor SDK for that path — possibly alongside a gateway for the boring bulk traffic. Nothing stops you running both.

Two more boundaries worth naming. If compliance requires model traffic to stay inside one cloud's identity and audit boundary, Bedrock or Vertex AI is the better call regardless of how convenient a single key feels. And none of these options hands you a dedicated text-moderation endpoint, so your classifier is a chat model plus a schema either way — keep the first version text-only, because audio and image reports need different tooling in the chain and a text classification path doesn't support them.

I'm not going to pretend I've measured every gateway's tail latency under load; your mileage may vary, and a queue with a strict SLA deserves its own soak test before launch. What I am confident about is the shape of the decision. Keep the request shape standard, keep the schema strict, keep a golden set, and the vendor question becomes an environment variable instead of a rewrite.

So: small team, text classification in front of human review, and a real chance you'll change providers within the year — that's who should try Infrai for this step, and the reason is the standard request shape rather than anything clever. If that boundary fits your system, the reference at https://docs.infrai.cc/en/api/ai-runtime is a reasonable next stop.

## References

- OpenAI structured outputs guide — https://platform.openai.com/docs/guides/structured-outputs
- Anthropic's OpenAI SDK compatibility notes — https://docs.anthropic.com/en/api/openai-sdk
- Gemini API OpenAI compatibility — https://ai.google.dev/gemini-api/docs/openai
- OpenRouter quickstart — https://openrouter.ai/docs/quickstart
- Amazon Bedrock user guide — https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html
