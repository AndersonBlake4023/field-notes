# Node.js Course Material Question Answering Explained (One API, Observable Retrieval)

Chunking more aggressively can improve retrieval precision, but it also creates more embedding work, more candidates, and more latency. For a property-management question-answering API, start with structure-aware chunks, retrieve a small candidate set, and measure answer evidence rather than trusting similarity scores alone. One API key can simplify access, but it should not collapse ingestion, retrieval, and generation into an operation nobody can inspect.

**TL;DR:** preserve document boundaries, attach property and source metadata to every chunk, separate indexing from queries, and trace the retrieval stages. Choose the simplest chunking method that meets a labeled-question test set and a latency budget. Semantic splitting earns its complexity only when it produces better evidence often enough to justify the extra processing and operational surface.

| Approach | Pick this when | Retrieval trade-off | Latency trade-off |
| --- | --- | --- | --- |
| Fixed token windows with overlap | Files are uniform and headings are unreliable | Predictable, but may split a rule from its exception | Few parsing decisions; index size grows with overlap |
| Structure-aware sections | Leases, policies, and manuals have usable headings or page structure | Keeps related clauses together; oversized sections still need caps | Moderate ingestion work; query path stays simple |
| Semantic boundaries | Topic shifts occur inside long, weakly formatted passages | Can isolate coherent evidence; thresholds require evaluation | More computation during ingestion and more behavior to observe |
| Parent-child retrieval | Answers need a precise match plus surrounding clauses | Retrieves a small child, then returns its larger parent context | Adds a lookup and can enlarge generation context |

## How Should One API Retrieve Course Material for Question Answering?

Use fixed windows as the baseline. They are easy to reproduce, and reproducibility matters when a resident asks why two versions of a pet policy produced different answers. The danger is visible in a common property document: one paragraph states the rule, while the next lists exceptions. A blind boundary can separate them. Overlap reduces that risk, but duplicated text can crowd the candidate set. Picture a course for newly hired property managers: lesson 4 quotes a standard entry-notice rule, the next paragraph explains an emergency exception, and a later quiz asks when staff may enter without the usual notice. A chunk ending after the rule looks topically perfect yet cannot support the right answer. A very large chunk keeps the exception but may also pull in unrelated keys, inspections, and lockout procedures. This is the quality-versus-latency trade-off in concrete form: enough local context to preserve the condition, but not so much that retrieval and generation must process an entire lesson for one clause.

Measure both.

Structure-aware splitting is the practical next step for documents with headings, paragraphs, and page references. Keep a stable source identifier, document version, section path, page, and access scope beside the text. Those fields are not decoration. They let the answer cite its evidence, let an evaluator distinguish two policy revisions, and prevent a query for one managed property from wandering into another property's material.

Semantic splitting is useful when layout does not express meaning: exported notes, flattened scans after text extraction, or long sections that change topic without headings. Do not adopt it because it sounds smarter. Compare it with the fixed and structural baselines on the same questions. A threshold that creates tidy-looking chunks may still separate a fee from the sentence that defines when it applies.

Parent-child retrieval fits a different failure mode. Search over compact child passages, then expand a winning result to a bounded parent section before answer generation. This can recover context without making every indexed unit large. The extra lookup is real, so include it in the trace.

## Make the pipeline observable before tuning it

The system is easier to reason about as a diagram in words: document version enters; parsing emits ordered blocks; chunking emits identifiable evidence units; embedding writes vectors and metadata; a question embeds once; filtered search returns candidates; optional reranking reorders them; generation receives the selected text and source labels.

Instrument those boundaries. For ingestion, record document count, extracted character count, chunk count, chunk-size distribution, rejected chunks, and indexing duration. For a query, record filter scope, candidate count, stage durations, selected chunk IDs, and the sources ultimately cited. Do not put lease text, resident questions, or credentials into telemetry attributes. Identifiers should be opaque and access-controlled.

Trace the handoffs.

Latency alone is a weak success signal. A fast response with the wrong lease clause is still wrong. Pair a latency distribution with retrieval measures from a labeled set: whether an expected source appears in the first *k* candidates, whether the final context contains the required clause, and whether the answer's citation points to that context. The original RAG work describes combining retrieval with generation for knowledge-intensive tasks; it does not remove the need to evaluate the retriever used by a specific application.

Start with a small, reviewed set of questions that reflect the real corpus: “When may a maintenance worker enter?”, “Is a satellite dish allowed?”, and “Which move-out charges require notice?” Label acceptable source sections, including cases where two sections are required. Add adversarial pairs from different properties and superseded revisions. This is where metadata filtering gets tested, not assumed.

**A useful decision rule:** keep the simpler splitter unless the alternative raises evidence recall on that set without breaching the query latency objective or creating an ingestion backlog. The exact threshold belongs to the service owner because corpus shape, update rate, and answer risk differ.

## A minimal TypeScript implementation

The following core stays independent of a specific embedding model or vector database. Every external operation has an interface, every chunk carries traceable metadata, and the query path returns evidence rather than an uninspectable answer. The example deliberately chooses a `1,800`-character target and `8` candidates. Those numbers are starting hypotheses for an experiment, not universal defaults; recording them with each evaluation run makes the quality and latency trade-off reviewable.

```ts
type Block = { text: string; heading?: string; page?: number };

type Chunk = {
  id: string;
  text: string;
  propertyId: string;
  documentId: string;
  version: string;
  section?: string;
  page?: number;
};

type Match = { chunk: Chunk; score: number };

interface Embedder {
  embed(texts: string[], signal?: AbortSignal): Promise<number[][]>;
}

interface VectorIndex {
  upsert(items: Array<{ chunk: Chunk; vector: number[] }>): Promise<void>;
  search(input: {
    vector: number[];
    propertyId: string;
    limit: number;
  }): Promise<Match[]>;
}

interface Telemetry {
  time<T>(name: string, attributes: Record<string, string | number>, run: () => Promise<T>): Promise<T>;
  count(name: string, value: number, attributes: Record<string, string>): void;
}

const normalize = (value: string) => value.replace(/\s+/g, " ").trim();

function chunkBlocks(
  blocks: Block[],
  metadata: Pick<Chunk, "propertyId" | "documentId" | "version">
): Chunk[] {
  const targetCharacters = 1_800;
  const chunks: Chunk[] = [];
  let pending: Block[] = [];
  let size = 0;

  const flush = () => {
    if (pending.length === 0) return;
    const index = chunks.length;
    chunks.push({
      ...metadata,
      id: `${metadata.documentId}:${metadata.version}:${index}`,
      text: pending.map((block) => normalize(block.text)).join("\n\n"),
      section: pending[0].heading,
      page: pending[0].page,
    });
    pending = [];
    size = 0;
  };

  for (const block of blocks) {
    const text = normalize(block.text);
    if (!text) continue;
    if (size > 0 && size + text.length > targetCharacters) flush();
    pending.push({ ...block, text });
    size += text.length;
  }
  flush();
  return chunks;
}

async function ingest(
  blocks: Block[],
  metadata: Pick<Chunk, "propertyId" | "documentId" | "version">,
  embedder: Embedder,
  index: VectorIndex,
  telemetry: Telemetry
): Promise<void> {
  const chunks = chunkBlocks(blocks, metadata);
  await telemetry.time("rag.ingest", { chunks: chunks.length }, async () => {
    const vectors = await embedder.embed(chunks.map((chunk) => chunk.text));
    if (vectors.length !== chunks.length) throw new Error("Embedding count mismatch");
    await index.upsert(chunks.map((chunk, i) => ({ chunk, vector: vectors[i] })));
  });
  telemetry.count("rag.chunks", chunks.length, { documentId: metadata.documentId });
}

async function retrieve(
  question: string,
  propertyId: string,
  embedder: Embedder,
  index: VectorIndex,
  telemetry: Telemetry,
  signal?: AbortSignal
): Promise<Match[]> {
  return telemetry.time("rag.retrieve", { propertyId }, async () => {
    const [vector] = await embedder.embed([question], signal);
    const matches = await index.search({ vector, propertyId, limit: 8 });
    telemetry.count("rag.candidates", matches.length, { propertyId });
    return matches;
  });
}
```

The sample uses character count because it keeps the example focused; production chunk limits should match the tokenizer and context constraints of the chosen embedding and generation models. It also omits retries. Add bounded retries around transient external calls, make upserts idempotent with stable chunk IDs, and fail an ingestion version as a unit if vector counts do not match. Never silently publish a partial version.

Keep credentials at the adapter boundary. “One key” should mean a straightforward integration contract, not one credential shared by browsers, build jobs, and the query service. Server-side configuration, rotation, least privilege, and separate environments still apply. Node's `AbortSignal` gives the query adapter a standard cancellation mechanism; propagate it so abandoned requests do not keep consuming the latency budget.

## Test the evidence, then watch the drift

Run the labeled questions against each candidate strategy and store the configuration with the result: parser version, chunker version, embedding identifier, retrieval limit, filters, and corpus snapshot. Without that record, a before/after chart cannot explain what changed. Use warm-up requests and report distributions rather than a single latency value.

Deployment needs two tracks. The offline track blocks a release when evidence retrieval regresses. The online track watches latency, empty-result rate, candidate counts, ingestion lag, and citation coverage. Alert on sustained service-objective violations, not isolated slow calls. Trace sampling can be increased for errors and empty results while ordinary successful traffic remains sampled conservatively.

Corpus changes deserve their own alarm. A parser update can suddenly create many tiny chunks even though the query service is healthy. Watch chunk-size percentiles and chunks per document by parser version. Sharp movement should pause publication of the new index version until the sample documents are inspected.

There is a clean rollback shape: build a new version beside the active one, evaluate it, then switch an alias or version pointer atomically. Keep the prior version long enough to reverse the pointer. Queries should log the selected corpus version, which turns “the answer changed” into a traceable investigation.

## Limits worth accepting

Vector similarity does not prove an answer. It proposes evidence. Generation must be allowed to abstain when the retrieved passages do not support a response, and high-impact lease or safety questions may still require human review.

The sample also assumes text extraction has already preserved reading order. Scanned tables, multi-column pages, and appendices can violate that assumption. Evaluate extraction separately before tuning embeddings; better vectors cannot reconstruct text that never reached the index correctly.

Finally, do not optimize chunking in isolation. Parser quality, metadata boundaries, embedding behavior, filters, reranking, and context assembly all influence retrieval. Change one variable at a time, keep the observable before/after, and choose the least complicated pipeline that returns the right property document evidence within the required time.

## Sources and References

- Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks: https://arxiv.org/abs/2005.11401
- OpenTelemetry semantic conventions: https://opentelemetry.io/docs/specs/semconv/
- OpenTelemetry trace specification: https://opentelemetry.io/docs/specs/otel/trace/
- Node.js `AbortController` and `AbortSignal`: https://nodejs.org/api/globals.html#class-abortcontroller
- OWASP Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
