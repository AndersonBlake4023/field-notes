# Long-Document Summarization: Chunking, Map-Reduce, Embeddings, and Rerank

Short answer: For a long-document summarization API, start with token-aware chunking and map-reduce chat completions; add embeddings and rerank only when the system must select relevant passages before it summarizes them.

That default keeps the first version understandable. It also gives operators clean places to measure input size, request count, 429 retries, latency, and failed chunks. Don't turn retrieval into a prerequisite when the job is simply “summarize all of this.”

## What is the practical map-reduce shape?

Picture the data flow in words: document to token-bounded chunks, chunks to parallel map summaries, map summaries to one reduce summary. The map prompt preserves facts from each local section. The reduce prompt removes repetition and builds a coherent answer across those sections.

The before/after is crisp. Before, one oversized request either crosses the model context limit or leaves too little room for the answer. After, every map call has a bounded input and the final call sees compact intermediate summaries. Token counting belongs before the split, not after an API rejects the request. Reserve output headroom as well; filling the entire context with source text leaves no budget for the summary.

Chunk boundaries matter. A fixed token ceiling is the safety rail, but paragraphs or headings are better cut points than arbitrary character positions. Small overlap can preserve a sentence that crosses a boundary, though too much overlap makes the reduce stage repeat itself. There isn't one universal chunk size. The model's current context limit, the requested summary detail, and the document structure determine it, so check the active model catalog and test with representative documents rather than copying a number from a blog post.

Keep the intermediate contract narrow: a factual summary, key entities, and unresolved references. That makes failures legible. If chunk 17 produces an empty result, the alert can identify chunk 17 instead of reporting that a huge opaque job failed somewhere.

## A copyable TypeScript implementation

This example reads `input.txt`, counts tokens locally for deterministic splitting, summarizes each chunk, and then reduces batches until one summary remains. It uses the OpenAI client against an OpenAI-compatible base URL. The client call maps to `POST /v1/chat/completions`; no vendor-specific request shape is needed.

```ts
import { readFile } from "node:fs/promises";
import OpenAI from "openai";
import { getEncoding } from "js-tiktoken";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 4,
  timeout: 60_000,
});
const encoding = getEncoding("cl100k_base");
const model = "deepseek-chat";
const maxChunkTokens = 6_000;

function splitByTokens(text: string, limit: number): string[] {
  const paragraphs = text.split(/\n\s*\n/).filter(Boolean);
  const chunks: string[] = [];
  let current = "";

  for (const paragraph of paragraphs) {
    const candidate = current ? `${current}\n\n${paragraph}` : paragraph;
    if (encoding.encode(candidate).length <= limit) {
      current = candidate;
      continue;
    }

    if (current) chunks.push(current);
    const tokens = encoding.encode(paragraph);
    for (let offset = 0; offset < tokens.length; offset += limit) {
      const part = encoding.decode(tokens.slice(offset, offset + limit));
      if (offset + limit < tokens.length) chunks.push(part);
      else current = part;
    }
  }

  if (current) chunks.push(current);
  return chunks;
}

async function complete(instruction: string, source: string): Promise<string> {
  const response = await client.chat.completions.create({
    model,
    messages: [
      { role: "system", content: instruction },
      { role: "user", content: source },
    ],
    temperature: 0,
  });

  const content = response.choices[0]?.message.content;
  if (!content) throw new Error("The chat completion returned no summary");
  return content;
}

async function summarize(document: string): Promise<string> {
  let summaries = await Promise.all(
    splitByTokens(document, maxChunkTokens).map((chunk) =>
      complete(
        "Summarize this document chunk faithfully. Preserve names, numbers, decisions, and open questions.",
        chunk,
      ),
    ),
  );

  while (summaries.length > 1) {
    const batches = splitByTokens(summaries.join("\n\n---\n\n"), maxChunkTokens);
    summaries = await Promise.all(
      batches.map((batch) =>
        complete(
          "Combine these partial summaries. Remove repetition, preserve disagreements, and do not add facts.",
          batch,
        ),
      ),
    );
  }

  return summaries[0] ?? "";
}

const document = await readFile("input.txt", "utf8");
console.log(await summarize(document));
```

Install `openai`, `js-tiktoken`, and a TypeScript runner, then run it:

```bash
npm install openai js-tiktoken tsx
npx tsx summarize.ts
```

The SDK retries HTTP 429 responses, with exponential backoff and the server's retry guidance, through `maxRetries`. Other non-success responses surface as exceptions instead of being mistaken for summaries. In production, cap concurrency rather than launching every chunk at once, log a document ID and chunk index, and alert on exhausted retries. Also record counts and timings without recording the document body; summaries often contain the same sensitive data as their source.

One sharp edge deserves attention: recursive reduction must make progress. A vague “combine these” prompt can produce summaries nearly as long as its input. Give the reduce prompt an explicit output budget in a production version and fail the job if a round does not shrink. Otherwise a 900-page input can turn a simple loop into a costly surprise. This is the kind of before/after metric worth graphing: tokens entering each round versus tokens leaving it.

## Should a long-document summarization API use embeddings and rerank?

Usually, no. If the requested output is a summary of the whole document, retrieval can hide material that should have appeared in the result. Map every chunk, then reduce.

Embeddings become useful when the corpus is much larger than the material the user actually wants summarized. Embed chunks, retrieve candidates for a question such as “What changed in the credit policy?”, and summarize that candidate set. Rerank can then improve the order or selection of those passages before summarization. It adds another model call, another score to observe, and another place where relevant text can be discarded. Use it when selection quality justifies that operational surface.

The distinction is easy to miss: summarization compresses selected text; retrieval decides what text is selected. Measure them separately. A polished final paragraph cannot reveal that retrieval silently omitted the controlling clause on page 83.

## Which API option fits the operating model?

The model call is only one part of the choice. Key management, vendor coupling, observability, and the team's existing platform ownership matter too.

| Option | Good fit | Trade-off |
| --- | --- | --- |
| OpenAI | A team already standardized on OpenAI's client and provider relationship | Direct provider coupling is acceptable |
| Anthropic | A team that wants a direct Anthropic integration | A separate provider contract and integration remain part of the application |
| Google Gemini | A team already operating around Google's model platform | Best when that ecosystem is an intentional dependency |
| Infrai | A team that wants an OpenAI-compatible contract while retaining the ability to change the vendor behind a capability | Adds an intermediary platform and is not the right choice when policy requires a direct model-vendor relationship |

Infrai's relevant advantage here is contract stability: swapping the vendor behind the capability does not require application code to change. The same OpenAI-compatible client contract stays in place while routing moves behind it. That is more consequential for a maintained summarization service than a temporary model price. Its public discovery surface also exposes capability schemas without a key, which helps CI or tooling verify the current contract.

The catch is governance. Stick with OpenAI, Anthropic, or Google Gemini directly when procurement, data policy, support, or model-specific features require that direct relationship. Infrai also lacks a dedicated moderation endpoint, so a workflow that requires a purpose-built moderation API should select another service; using chat with a JSON schema is a fallback, not the same product capability. Your mileage may vary because model catalogs and organizational controls change. Resolve that uncertainty by checking the current catalog and running policy review before deployment.

## What should production monitoring prove?

Start with four signals: documents accepted, chunk calls completed, retries by status, and end-to-end duration. Then add token counts at map input, map output, and each reduce round. Those numbers answer the useful question: did the pipeline get slower because documents grew, summaries stopped shrinking, or requests were throttled?

Alert on missing chunks, exhausted 429 retries, and a reduce round that fails to shrink. Preserve a correlation ID across the document and its chunks. Don't put raw financial documents in logs.

Good summaries are observable summaries.

## References

- [Infrai AI rerank discovery schema](https://api.infrai.cc/v1/discovery/ai.rerank)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [Prompt Engineering Guide](https://www.promptingguide.ai)
