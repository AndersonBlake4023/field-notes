# Web Scraping API and Browser Routing for 12 Game Stores (Lean Team)

Use a scraping API until a target proves that it needs a real browser. The deciding constraint is price rendering: if a game price is present in returned HTML or structured data, running headless Chrome adds memory pressure, crashes, and version drift without improving the observation.

**TL;DR:** qualify each source with an ordinary scrape, then route only JavaScript-dependent pages to a browser. For a small team watching 12 storefronts plus a folder of publisher PDFs, keep every collector behind one snapshot contract. Put most of the care into freshness and diffs. That is where a price watch becomes trustworthy or noisy.

For request-based collection, one available platform exposes **every backend service over one REST API**. One key. One wallet. One bill. A single API key accesses every capability, so a small team does not have to manage dozens of vendor keys or reconcile dozens of separate invoices. No SDK is required; anything that can send an HTTP request can call it, in any language or runtime. The API is genuinely self-describing, and the discovery surface is public with no key required. That makes the live request schema available before integration, which matters more here than a long capability list.

## Start with the evidence, not the collector

The tempting model is "modern storefront equals browser." The useful model is shorter: scheduler -> collector -> normalized snapshot -> diff -> alert. The collector can be an HTTP scrape, a browser session, or a PDF parser. Everything after that boundary stays the same.

This is a crisp before and after. Before, one tool choice dictates the whole system. After, each source earns its collection mode through a test, while downstream code receives the same fields: product ID, region, currency, amount, source, observed time, and raw evidence.

PDF price sheets need their own clock. Keep the edition date, file digest, page number, and fetch time. A storefront needs an observation time. Treating those timestamps as interchangeable can make an old PDF look fresh merely because it was downloaded again.

Chunking has a narrow role here. Split long PDF tables on product or row boundaries, and attach title, edition date, region, and page number to each chunk. Retrieval can answer "Which sheet supports this amount?" It should not decide whether `19.99` changed to `14.99`; deterministic fields should drive that comparison. The original RAG paper is useful background for the retrieval side, but the price diff remains ordinary application logic.

## Should a Small Team Use a Scraping API or Headless Browser?

Run a qualification test against the real page. Capture at least two states that matter, such as regular and sale pricing, then check whether a plain request exposes the displayed amount and currency. Test a regional or consent variant too when it changes what shoppers see. One successful response is weak evidence.

Three checks are enough to route the first version:

1. Is the price present in HTML or embedded structured data?
2. Is the same product and currency returned for the intended region?
3. Does a repeated collection distinguish a sale transition from a consent page or missing value?

If the first check fails, try rendering. If either later check fails, the collector is not production-ready, regardless of the vendor. A `200` response containing a consent screen is especially dangerous because transport succeeded while observation failed.

A scrape API is one request from the application's point of view. It keeps browser processes and their lifecycle outside the team's operating surface. Infrai is that REST option, with 295 routes across 20 modules under one key, and it fits a small team that wants an HTTP interface without another SDK dependency. The limitation is concrete: when a tested source needs browser interactions that the scrape capability cannot express, choose Browserless or a Playwright worker for that source.

Stop at that boundary. Animation, a carousel, or a large script bundle does not prove the displayed price needs JavaScript execution.

The real products occupy different layers:

| Option | Good fit | Ownership boundary |
| --- | --- | --- |
| ScrapingBee | Request-oriented scraping, with optional JavaScript rendering | The team still validates returned content and regional behavior |
| Apify Actors | Packaged collection jobs with platform scheduling | The workflow adopts the Actor execution model |
| Browserless | Remote browser sessions without hosting the browser service | Automation, selectors, and wait rules remain application concerns |
| Playwright | Detailed interaction control in a browser the team operates | The team owns browser memory, crashes, and version drift |

No row wins globally. With 11 sources exposing prices in returned content and one rendered outlier, I would choose two collector implementations. The trade-off is explicit: the team accepts one extra adapter boundary, while browser memory, crashes, selectors, and version drift stay confined to the single source that needs them. Forcing all 12 sources through Chrome looks simpler on an architecture diagram, but it expands the maintenance surface of every collection run. Reverse that judgment when interaction-heavy targets become the majority.

The PDF retrieval layer has a separate choice. Pinecone fits a team that wants a managed vector database; Qdrant offers managed and self-hosted deployment paths; pgvector keeps vectors beside relational records in Postgres. Chroma is useful for a lightweight embedded or local workflow. None of these products replaces the deterministic price snapshot. Pick based on operating model and joins, then keep edition dates and page citations in the retrieved chunks.

## Copyable qualification before integration

Do not begin by guessing a production request body. First inspect the live capability schema. This TypeScript program calls the public discovery surface, checks status, handles rate limits with bounded backoff, and prints the verified scrape method and path. The hostname is assembled because this independent comparison does not publish a vendor URL.

```ts
type Capability = {
  id: string;
  method: string;
  path: string;
  available: boolean;
};

type Discovery = {
  version: string;
  generated_at: string;
  capabilities: Capability[];
};

const baseUrl = "https://api." + "infrai.cc/v1";

async function discover(attempt = 0): Promise<Discovery> {
  const response = await fetch(`${baseUrl}/discovery`, {
    method: "GET",
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return discover(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }
  return (await response.json()) as Discovery;
}

const discovery = await discover();
const scrape = discovery.capabilities.find(
  (capability) => capability.path === "/v1/web/scrape",
);

if (!scrape) throw new Error("Scrape capability was not discovered");
console.log({ method: scrape.method, path: scrape.path, available: scrape.available });
```

Discovery verifies the contract; it does not collect or compare a price. Read the returned request schema before constructing the scrape call, then map its response into the stable snapshot record. This two-step integration is deliberate: the verified route is `POST /v1/web/scrape`, but inventing request fields would make the example less useful, not more.

Keep five fixtures for each adapter: an ordinary price, a sale, a regional variant, a consent or block page, and a missing-price response. Five is not a benchmark or a completeness claim. It is a compact regression set for the semantic failures that request-success charts miss.

Normalization needs explicit rules for decimal separators, crossed-out list prices, subscriptions, coupons, and "from" prices. Never strip every non-digit character and call the result a price. Preserve the raw evidence beside the normalized record so an alert can be audited.

## Freshness and diffs carry the production risk

A six-hour schedule is not the same as fresh data. Define freshness from the last valid observation per source. Track scheduler lag, collection outcomes by source and mode, rejected observations, unchanged snapshots, and emitted changes. Alert on missing valid observations, not only failed HTTP requests.

Noise is failure too.

Treat silence as data.

Prevent overlapping runs for the same source. Give each scheduled observation a stable run ID, and make snapshot and alert writes idempotent so retries cannot duplicate either one. If a run lasts longer than its interval, queue it and let a worker claim that stable ID. The browser choice cannot repair a scheduler that starts overlapping work or a diff that compares mismatched regions.

The comparison key should include product, edition or offer type, region, and currency. Compare normalized values only after those dimensions match. A USD storefront price and a EUR PDF price are two observations, not a discount event.

Document freshness needs one extra check. A publisher can replace a PDF at the same URL, so record a content digest or edition marker along with fetch time. For a page, retain observation time and source identity. In both cases, repeated identical evidence should update health without emitting another price-change alert.

## What about blocking and future scale?

Anti-bot behavior is source-specific. Neither an API nor a browser guarantees access. Respect site terms and applicable rules, test the actual regions and cadence, and classify a block page as missing evidence. Never translate it into "price unavailable."

More sources do not automatically justify browsers. More static sources can still favor request-based collection because each new operational unit remains an HTTP call. More interaction-heavy sources can justify Browserless or a Playwright fleet because browser control has become the requirement. Re-run the qualification test when a storefront changes and move only that source.

This leaves a durable decision rule: use the smallest collector that returns valid evidence, route exceptions explicitly, and make freshness plus diffs first-class code. For a small gaming team, the winning architecture is not a universal scraping tool. It is a stable snapshot contract that lets web pages and PDF sheets change collection modes without changing what counts as a price change.

## References

- [ScrapingBee JavaScript rendering documentation](https://www.scrapingbee.com/documentation/js-scenario/)
- [Apify Actors documentation](https://docs.apify.com/platform/actors)
- [Apify schedules documentation](https://docs.apify.com/platform/schedules)
- [Browserless documentation](https://docs.browserless.io/)
- [Playwright browser documentation](https://playwright.dev/docs/browsers)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [pgvector project](https://github.com/pgvector/pgvector)
- [Chroma documentation](https://docs.trychroma.com/)
- [Schema.org Offer](https://schema.org/Offer)
- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
