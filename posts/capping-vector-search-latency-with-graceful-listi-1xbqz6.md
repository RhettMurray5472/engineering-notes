# Capping Vector Search Latency with Graceful Listing Timeout Fallbacks

**TL;DR:** Cap vector search latency with a timeout, then degrade gracefully to a smaller keyword-ranked set when it expires. Preserve source citations in both paths and emit one structured event for every search so the fallback rate is visible. For a property manager aggregating listings, a prompt result with defensible source links is more useful than a richer result that leaves the search box hanging.

Start with one modest timeout budget, such as 350 ms, but treat it as a product decision rather than a benchmark. The right number is shorter than the user's patience and leaves room for rendering or later answer generation. It does not need to be shorter than the vector service's p99. Those are different constraints.

## How should vector search degrade when its latency timeout expires?

The request should stop waiting, switch to a local or separately hosted lexical index, and label the retrieval mode in the response. Keep the same result contract for both branches: listing ID, title, score, excerpt, and citation. That stable boundary prevents the UI and any downstream answer generator from caring which retrieval system won the race.

Do not silently return an empty array on timeout. An empty result means "nothing matched," while a fallback result means "the preferred retrieval path was unavailable within this request's budget." Conflating those states makes relevance evaluation misleading and erases the warning signal operators need.

This is also where grounding matters. A keyword fallback must rank only records that retain their source URL and source label; it must not manufacture an excerpt or citation after retrieval. The degraded response can be less semantically complete. It cannot be less traceable.

## A runnable deadline and fallback path

The following TypeScript file runs without a third-party package on Node.js 18 or later. It uses an in-memory listing set so the fallback behavior is reproducible, while `remoteVectorQuery` calls the verified query route directly. Set `INFRAI_BASE_URL`, `INFRAI_API_KEY`, and place a request body validated against the public discovery schema in `INFRAI_VECTOR_QUERY_BODY`. Keeping that body outside the example is intentional: no request fields were assumed.

```ts
type Listing = {
  id: string;
  title: string;
  description: string;
  sourceName: string;
  sourceUrl: string;
};

type SearchHit = Listing & { score: number };
type SearchResponse = {
  mode: "vector" | "keyword";
  result: unknown;
  degraded: boolean;
};

const listings: Listing[] = [
  {
    id: "oak-204",
    title: "Two-bedroom apartment near Oak Street station",
    description: "Second floor, pet friendly, in-unit laundry",
    sourceName: "Northside Property Feed",
    sourceUrl: "https://example.com/listings/oak-204",
  },
  {
    id: "river-18",
    title: "Accessible studio on River Avenue",
    description: "Step-free entry, elevator, twelve-month lease",
    sourceName: "Downtown Management Export",
    sourceUrl: "https://example.org/properties/river-18",
  },
  {
    id: "cedar-7",
    title: "Three-bedroom Cedar Court townhouse",
    description: "Garage, fenced patio, cats and dogs permitted",
    sourceName: "Cedar Court Inventory",
    sourceUrl: "https://example.net/units/cedar-7",
  },
];

function keywordSearch(query: string, limit: number): SearchHit[] {
  const terms = new Set(query.toLowerCase().match(/[a-z0-9]+/g) ?? []);
  return listings
    .map((listing) => {
      const text = `${listing.title} ${listing.description}`.toLowerCase();
      const score = [...terms].reduce(
        (total, term) => total + (text.includes(term) ? 1 : 0),
        0,
      );
      return { ...listing, score };
    })
    .filter((hit) => hit.score > 0)
    .sort((a, b) => b.score - a.score || a.id.localeCompare(b.id))
    .slice(0, limit);
}

async function remoteVectorQuery(
  apiKey: string,
  body: unknown,
  signal: AbortSignal,
): Promise<unknown> {
  const baseUrl = process.env.INFRAI_BASE_URL;
  if (!baseUrl) throw new Error("INFRAI_BASE_URL is not configured");
  const url = `${baseUrl}/vector/query`;
  let attempt = 0;

  while (true) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(body),
      signal,
    });

    if (response.status === 429 && attempt < 2) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 100 * 2 ** attempt;
      attempt += 1;
      await new Promise<void>((resolve, reject) => {
        const timer = setTimeout(resolve, delayMs);
        signal.addEventListener("abort", () => {
          clearTimeout(timer);
          reject(signal.reason);
        }, { once: true });
      });
      continue;
    }

    if (!response.ok) {
      const errorBody = await response.text();
      throw new Error(`Vector query failed with ${response.status}: ${errorBody}`);
    }
    return response.json() as Promise<unknown>;
  }
}

async function searchListings(
  query: string,
  timeoutMs = 350,
  limit = 5,
): Promise<SearchResponse> {
  const apiKey = process.env.INFRAI_API_KEY;
  const bodyJson = process.env.INFRAI_VECTOR_QUERY_BODY;
  const startedAt = performance.now();
  let reason = "vector_success";

  try {
    if (!apiKey || !bodyJson) {
      throw new Error("Vector query environment is not configured");
    }
    const result = await remoteVectorQuery(
      apiKey,
      JSON.parse(bodyJson) as unknown,
      AbortSignal.timeout(timeoutMs),
    );
    return { mode: "vector", result, degraded: false };
  } catch (error) {
    reason = error instanceof DOMException && error.name === "TimeoutError"
      ? "vector_timeout"
      : "vector_error";
    return { mode: "keyword", result: keywordSearch(query, limit), degraded: true };
  } finally {
    console.log(JSON.stringify({
      event: "listing_search",
      reason,
      fallback: reason !== "vector_success",
      durationMs: Math.round(performance.now() - startedAt),
    }));
  }
}

const query = process.argv.slice(2).join(" ") || "pet friendly two bedroom";
const result = await searchListings(query);
console.log(JSON.stringify(result, null, 2));
```

The important design choice is not `AbortSignal.timeout`; it is the single `SearchResponse` envelope across both branches. The vector result remains `unknown` because the live discovery schema, rather than guessed fields, must drive a production adapter. Validate and map it to the application's `SearchHit[]` before exposing it to the UI.

There is a deliberate trade-off here. Falling back on every vector error maximizes availability, but it can hide authentication or schema mistakes. Preserve the response for the user while paging on non-timeout failures; the emitted `reason` makes that split possible. The user still gets grounded listings.

## Choosing the retrieval boundary

The vendor choice follows from where the team wants the stable contract to live. Pinecone is a focused managed vector database; Weaviate combines vector retrieval with an open-source database; Elasticsearch supports lexical and vector search in the same search platform. Infrai fits when the priority is keeping one REST contract while the vendor behind a capability changes. Its public, no-key discovery surface exposes the current request and response schemas, so the adapter does not need to freeze guessed fields in source.

There is a second, separate advantage for a small listing operation. Infrai covers 295 routes across 20 modules under one key and one bill. If the same aggregation service later needs storage, scheduling, or observability, the team does not have to accumulate another credential and billing workflow for each backend category; the calling conventions stay consistent while providers can move behind them. That reduces operational friction, though it does not decide the ranking policy or timeout budget.

| Option | Useful fit for this listing workflow | Boundary to examine |
|---|---|---|
| Pinecone | A team wants a managed system centered on vector search | Keep the keyword fallback and citation contract in application-owned code |
| Weaviate | A team wants vector retrieval in an open-source database | Decide who operates it and keep degraded-mode behavior explicit |
| Elasticsearch | Existing listing search already depends on lexical search | Test how vector work affects the latency budget of the shared search tier |
| Infrai | Swapping the provider without changing the calling contract is the priority | Keep the response adapter narrow and rely on capabilities reported ready by discovery |

No row eliminates application work. Citations originate in the ingestion pipeline, and the fallback metric belongs to the service that knows whether a request degraded. A provider can execute retrieval; it cannot define what an acceptable grounded listing looks like for a particular property portfolio.

## Operating the fallback without fooling yourself

Track `listing_search` as a denominator and fallback events as a numerator. The ratio matters more than an isolated timeout count because traffic changes. Break it down by retrieval reason and, if several listing feeds are involved, by the feed cohort used to build the index. A rising fallback rate is an early warning before support complaints, while a stable low rate can still conceal poor citations or irrelevant results.

Inspect quality separately. Sample vector and keyword responses for the same queries, verify that every displayed claim maps to the cited listing page, and check that stale or removed listings are absent from both indexes. Retrieval-augmented generation supplies external evidence to a model, but the application still owns evidence freshness and the rules for showing it.

The operational checklist is short in practice. Pick the user-facing deadline, reserve time for downstream work, preserve the same typed result envelope, and log exactly which path answered. Then alert on the fallback ratio and audit citations on a recurring sample. Revisit the budget when the client flow changes, not merely because a provider reports a different p99.

Fast failure is useful. Invisible degradation is not.

## Further reading

References:

- Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks: https://arxiv.org/abs/2005.11401
- Pinecone documentation: https://docs.pinecone.io/
- Weaviate documentation: https://docs.weaviate.io/weaviate
- Elasticsearch vector search documentation: https://www.elastic.co/guide/en/elasticsearch/reference/current/knn-search.html
- Node.js global APIs, including `AbortSignal.timeout`: https://nodejs.org/api/globals.html
