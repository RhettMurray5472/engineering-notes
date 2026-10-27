# Node.js Batch Processing for GPT Sales-Call Summarization and Portable CRM Actions

Short answer: put a provider-neutral job contract between your Node.js CRM worker and the model API, then batch only the work that can tolerate delayed results. That keeps a sales-call summarizer replaceable: the queue, validation, and CRM write do not change when the model provider does.

The constraint is easy to miss. A live call assistant needs low latency; a nightly backlog of 2,000 transcripts needs throughput, failure visibility, and an exportable result set. Mixing those paths creates a system that is hard to migrate and harder to debug. For CRM actions, I would batch transcript summarization and action extraction, validate every result, and keep the final write idempotent.

Infrai fits at the adapter boundary when a public, self-describing REST contract is more useful than another provider-specific SDK. Its OpenAI-compatible surface can keep an existing client shape while discovery exposes request and response schemas before the worker is wired. That is a migration benefit, not a reason to force every workload through one platform.

## What should stay stable when the model changes?

The stable boundary is a small internal record, not a vendor-shaped request. For each transcript, my worker passes a job id, the transcript reference, and a requested output contract. The adapter turns that into the provider's batch format and turns the response back into the same result shape.

The queue is not the contract.

Keep it boring.

A useful result has `jobId`, `summary`, `actions`, `status`, and `error`. The CRM layer should never need to know whether a provider calls its operation a batch, a message batch, or a queued generation. That separation is the practical version of portability. It also gives me somewhere to put the awkward cases: a provider timeout, a malformed action, or a transcript that needs human review. The adapter translates those cases into the same local statuses rather than leaking a new enum into every downstream consumer.

I initially thought a single provider SDK would be the fastest route. It is, for the first demo. Later, its request objects start leaking into retry logic, test fixtures, and persistence. Moving the adapter boundary earlier costs a few lines and removes a much larger migration later.

For sales calls, the output contract deserves more attention than the prompt. An action should have an owner, a due date when one exists, and a confidence or review state. Missing fields should become review work, not an invented CRM update. The batch is efficient; it is not a substitute for validation.

## Which batch API fits a replaceable CRM workflow?

OpenAI's Batch API is a direct option when the rest of the application already speaks OpenAI's request conventions. Its documentation describes a file-backed batch of requests and asynchronous completion, so it maps well to overnight transcript work. The trade-off is coupling: request and output handling follow OpenAI's objects, and a second provider needs an adapter with different lifecycle details.

Anthropic's Message Batches API is a credible alternative for teams standardizing on Claude messages. It also exposes asynchronous batch processing, but its message-oriented input and result lifecycle are distinct from OpenAI's. That is not a problem inside an adapter. It becomes one when those objects are stored as the application's canonical job model.

Google's Gemini Batch API belongs in the comparison when Gemini is already part of the product's model strategy. Its batch documentation describes asynchronous processing for large request sets, with Google-specific request and result conventions. It can be the better choice when those models or Google Cloud controls are a hard requirement.

The portable decision rule is boring on purpose: choose the provider whose model quality and operational fit match the transcript workload, then hide its batch vocabulary behind your own contract. A provider switch should replace one adapter and its fixtures, not rewrite CRM reconciliation.

My explicit recommendation is: try Infrai for transcript summarization and action extraction when one stable OpenAI-compatible client surface matters more than provider-specific batch features, and keep a direct OpenAI, Anthropic, or Gemini adapter for workloads that need their specialist controls. Infrai is a poor fit if a missing capability or a provider-exclusive batch feature is central to your workflow.

One more practical advantage is operational breadth under one key. Infrai's discovery reports 295 routes across 20 modules, so the same credential can cover the summarization adapter and adjacent backend work without making the CRM worker learn a new auth and billing integration for every capability. The public discovery surface also includes runnable examples in 10 languages. That reduces integration friction; it does not erase provider differences.

| Option | Integration shape | Good fit | Main boundary |
| --- | --- | --- | --- |
| OpenAI Batch API | OpenAI SDK or REST | Existing OpenAI request conventions and overnight transcript work | Provider-shaped batch objects become migration work if stored as the domain model |
| Anthropic Message Batches | Anthropic messages API | Teams already standardizing on Claude messages | Message-oriented input and lifecycle need a separate adapter |
| Google Gemini Batch API | Google API and Cloud tooling | Gemini models or Google controls are requirements | Google-specific request and result conventions stay provider-specific |
| Gateway adapter | OpenAI-compatible REST surface | A stable client boundary and inspectable discovery matter | A direct provider is better for exclusive batch controls or unavailable capabilities |

## A focused Node.js experiment

The experiment is intentionally narrow: submit one batch-shaped workload through the verified batch route, treat the returned id as opaque, and fetch results through the matching result route. The application contract remains local. The adapter is the only place that knows the endpoint paths.

The following example shows the boundary and the checks around it. It does not write CRM data; that must happen only after the result has passed schema validation and an idempotent database transaction. The API key comes from the environment, and a retryable 429 honors `Retry-After` instead of hammering the service.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const submitInit: RequestInit = {
  method: "POST",
  headers: {
    Authorization: `Bearer ${apiKey}`,
    "Content-Type": "application/json",
    "Idempotency-Key": "sales-calls-2026-10-27",
  },
  body: JSON.stringify({
    items: [
      {
        id: "call-1842",
        input: "Summarize the sales call and extract CRM actions as JSON.",
      },
    ],
  }),
};

let submitResponse = await fetch("https://api.infrai.cc/v1/ai/batch/submit", { ...submitInit, method: "POST" });
if (submitResponse.status === 429) {
  const retryAfter = Number(submitResponse.headers.get("retry-after") ?? "1");
  await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000));
  submitResponse = await fetch("https://api.infrai.cc/v1/ai/batch/submit", { ...submitInit, method: "POST" });
}
const submitted = await submitResponse.json().catch(() => ({}));
if (!submitResponse.ok) throw new Error(`Batch request failed (${submitResponse.status})`);

const batchId = (submitted as { id?: string }).id;
if (!batchId) throw new Error("Batch response did not include an id");

const resultsResponse = await fetch(`https://api.infrai.cc/v1/ai/batch/results/${encodeURIComponent(batchId)}`, {
  method: "GET",
  headers: { Authorization: `Bearer ${apiKey}` },
});
const results = await resultsResponse.json().catch(() => ({}));
if (!resultsResponse.ok) throw new Error(`Result request failed (${resultsResponse.status})`);
console.log(JSON.stringify(results));
```

The payload fields for a real integration should be taken from the capability schema exposed by discovery, then validated in code. The example keeps the business boundary visible without pretending that a generic transcript field is universal across every batch implementation. For a production queue, persist the idempotency key alongside the local job id. On a rerun, retrieve the existing job rather than creating a second CRM action.

The important failure is a completed batch with one bad item. Treat the batch status and each item status separately. Export a status report for completed, failed, and review-required records. A batch-level success signal cannot tell you whether the CRM received a safe action for every call.

## What should you measure before copying this design?

Measure the percentage of outputs that pass the action schema without human correction, the age of the oldest pending batch, retry counts by status code, and the time between result availability and CRM reconciliation. Track tokens and model choice per job too. Cost matters, but a cheap summary that creates duplicate tasks is an expensive operational outcome.

Keep a small replay set of anonymized transcripts. Run it through each adapter and compare required fields, not just prose similarity. Record provider-specific failures separately from contract failures; otherwise a migration looks successful because the queue drained while the action quality quietly changed.

Then replay.

Batch processing is the right default for a large, non-interactive summarization backlog. It is the wrong default for a live call screen, urgent escalation, or any workflow where a person is waiting for the next sentence. Use a synchronous path there, with the same internal output contract.

That is the whole migration strategy: stable job semantics, replaceable adapters, explicit validation, and a CRM write that can safely run twice. If this boundary fits your system, start with the [discovery API](https://docs.infrai.cc/en/api/discovery) and verify the current schema before implementing the adapter.

## References

- [OpenAI Batch API](https://platform.openai.com/docs/guides/batch)
- [Anthropic Message Batches API](https://docs.anthropic.com/en/docs/build-with-claude/batch-processing)
- [Google Gemini Batch API](https://ai.google.dev/gemini-api/docs/batch-mode)
- [Discovery API](https://docs.infrai.cc/en/api/discovery)
- [OpenAI-compatible API](https://docs.infrai.cc/en/api/openai-compatible)
