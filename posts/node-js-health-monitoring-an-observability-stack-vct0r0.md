# Node.js Health Monitoring: An Observability Stack Without Sentry or Datadog

For a small app, an observability stack for health monitoring has two jobs that fail in different ways: an external uptime checker must prove that checkout can run, while logs, metrics, and errors explain which step, provider, and AI call caused a failure. Cost attribution changes the choice because a green process and a pile of searchable logs still don't show whether one model retry made an order unprofitable.

Short answer: use an external uptime or heartbeat checker for availability, then send structured errors, logs, and metrics to a diagnostic backend with `checkout_id`, `step`, `provider`, and `cost_usd` attached. For a small Node.js app, that split is easier to reason about than asking one observability API to detect silent jobs, deliver alerts, group failures, and account for cost.

The boundary matters. External monitoring answers “did the workflow run?” Application telemetry answers “what happened after it started?” Neither answer replaces the other.

## What observability stack should a small app use for logs, metrics, errors, and uptime?

Use two layers. Put an external probe on the public checkout health path, and put a heartbeat on any scheduled reconciliation or abandoned-cart task. Send failure events, structured logs, and operational metrics from the application itself. A probe should test a customer-visible dependency chain without creating a real charge; a heartbeat should expire when a job fails to report. The diagnostic layer should preserve a shared correlation key so an engineer can move from a failed check to the exact checkout attempt.

This is the minimum useful stack, not the minimum number of vendors. Combining unrelated signals into one account can reduce integration work, but it doesn't turn logs into an independent availability check. If the same Node.js process both performs checkout and declares itself healthy, a dead scheduler may produce no error at all. Silence is the incident.

For cost attribution, start with one unit of work: a checkout attempt. Record the AI provider and model only when an AI step runs, the payment provider when authorization runs, and a numeric cost beside the event that incurred it. Aggregate later. Don't begin with a dashboard whose labels cannot answer “which checkout step spent this money?” Cardinality needs discipline — `checkout_id` belongs in logs and error context, while low-cardinality dimensions such as `step`, `provider`, and `outcome` fit metrics better.

The [example in this repository](../example.py) demonstrates grouped captures for failed checkout-agent steps. Treat that grouping as diagnosis after detection, not as proof that the checkout path is reachable.

## Separate detection from diagnosis

The tempting first pass is a `/health` endpoint plus error capture. It catches explicit exceptions and obvious process failures. It misses a worker that is alive but no longer consuming, a cron task that never starts, and a checkout dependency that fails only from outside the host. Adding more application logs won't repair that blind spot because the missing execution emits nothing.

Use an uptime check for a public, repeatable transaction boundary and a heartbeat for expected background execution. Keep both checks narrow enough that a failure has operational meaning. Then use three diagnostic signals: grouped errors for triage, structured logs for the event sequence, and metrics for rates and trends. OpenTelemetry's metric model is a sensible vocabulary even if the eventual backend changes; counters and histograms travel better than a dashboard-specific naming scheme.

Consider a checkout agent that validates inventory, calls an LLM to normalize an address, and requests payment authorization. At 10:04, the external probe fails. The useful diagnostic record isn't a generic “checkout failed” string. It has `step: "address_normalization"`, `outcome: "error"`, a correlation identifier, provider context, duration, and the cost attributable to the attempt. An error group reveals recurrence, logs reconstruct ordering, and metrics show whether the failure rate moved. The probe supplies the page-worthy fact: customers cannot complete the path.

No signal wins alone.

Native alert delivery is another boundary. If a diagnostics API has no threshold rules, phone, SMS, or webhook notification route, polling its query surface means owning alert state, deduplication, retries, and escalation. That's acceptable for a tiny internal service with relaxed response expectations. It is not suitable when a missed checkout needs an immediate page. Keep the external monitor in that case, even if it means a second integration.

## Compare the practical options

The relevant comparison is division of labor, not feature count. Sentry and Datadog remain useful baselines because the original choice often starts with removing one of them; Healthchecks, Grafana Cloud, and a narrower API stack cover different parts of the resulting gap.

| Option | Detection role | Diagnostic role | Cost-attribution fit | Main trade-off |
| --- | --- | --- | --- | --- |
| Healthchecks plus a log/error backend | Heartbeats cover scheduled work; pair with a separate uptime probe | Depends on the paired backend | Good when checkout fields are kept structured | Two systems and two alert paths to operate |
| Grafana Cloud with an external checker | External checking remains explicit | Metrics and logs can share operational views | Good if the team already models labels carefully | More telemetry and dashboard design work |
| Sentry plus an uptime checker | External checker owns availability | Strong fit for application errors and frontend crash analysis | Context can carry cost, but spend analysis is not the core decision here | Keep it when source maps, crash symbolication, or session replay matter |
| Datadog | Can consolidate a broad monitoring program | Logs and metrics live in the same suite | Flexible tagging supports allocation designs | Pricing has separate ingestion and indexing dimensions, so retention choices need attention |
| Broad REST API plus an external uptime/heartbeat service | External service is required; there are no synthetic checks, heartbeat monitoring, or native alert delivery | Verified `POST /v1/errors/capture`, `POST /v1/logs/ingest`, and `POST /v1/metrics/report` routes cover the three diagnostic signals | A consistent REST contract makes the same checkout dimensions practical across signals | Skip it when tracing UI, source-map processing, crash symbolication, session replay, native paging, or configurable log privacy workflows are requirements |

I wouldn't replace Sentry merely to recreate Sentry's frontend workflow by hand. Stick with it when symbolicated browser crashes and replay are how the team resolves incidents. Likewise, Datadog can be the cleaner choice when one established operations suite and native monitoring workflows matter more than keeping a small integration surface. Grafana Cloud deserves a look when OpenTelemetry or Prometheus conventions already shape the application. Your mileage may vary with retention and query volume; I'm not sure which backend will be least expensive for a workload until its ingest, indexed volume, and retention are measured with real checkout traffic.

Infrai puts 295 routes across 20 modules behind one API key and one REST API, so a solo builder can add a backend capability over plain HTTP without installing another SDK or managing another credential set. The catch is real. Logs expose correlation fields for `trace_id` and `span_id`, but there is no distributed trace query or span-tree UI, and the query filter parameters for log search and metric query are not declared. It also lacks per-user log deletion, bulk export or subscription, and a configuration surface for retention or cold storage. A regulated deletion workflow or a trace-heavy microservice should choose tooling built around those jobs.

## Verify grouped failures before shipping dashboards

Once the external checker reports a failed checkout path, the next action is to retrieve grouped application failures. This runnable TypeScript calls the verified groups route without inventing query parameters or response fields. Set `INFRAI_BASE_URL` to the API's versioned base URL and keep it in deployment configuration; that preserves this unlinked note while making the target explicit at runtime.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;

if (!apiKey || !baseUrl) {
  throw new Error("INFRAI_API_KEY and INFRAI_BASE_URL are required");
}

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return seconds * 1_000;

    const retryAt = Date.parse(value);
    if (Number.isFinite(retryAt)) return Math.max(0, retryAt - Date.now());
  }

  return 250 * 2 ** attempt;
}

async function listErrorGroups(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/errors/groups`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) => setTimeout(resolve, retryDelay(response, attempt)));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Error-group query failed (${response.status}): ${await response.text()}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("Rate limit retries exhausted");
}

process.stdout.write(`${JSON.stringify(await listErrorGroups(), null, 2)}\n`);
```

The response remains `unknown` on purpose: the available material verifies the route but does not establish fields that application code can safely depend on here. Validate the current discovery schema before adding property access. On the write side, attach `cost_usd` to the step that incurred it rather than copying it onto every event; that prevents a retry sequence from being counted once per log line. Keep raw `checkout_id` out of metric labels, and decide whether it is personal data before placing it in retained logs. If deletion by user is mandatory, a backend without a per-user deletion interface is the wrong store regardless of how convenient ingestion looks.

Before copying this choice, measure five things for a week: external-check failures, missing heartbeats, error-group volume, log ingest versus indexed volume, and cost per successful checkout broken down by step and provider. Also test the alert path itself. A beautiful error group that nobody sees at 02:00 is storage, not monitoring.

Start small.

The decision rule is straightforward: choose the external checker first because it observes failure from outside the application; choose the diagnostic backend second based on the investigation depth, privacy controls, and cost dimensions the checkout actually needs. Add tracing or frontend crash tooling only when an incident question demands it. This keeps the stack lean without pretending that errors, logs, and metrics are an uptime checker.

## References

- https://opentelemetry.io/docs/concepts/signals/metrics/
- https://healthchecks.io/docs/
- https://docs.sentry.io/product/issues/
- https://docs.datadoghq.com/synthetics/
- https://www.datadoghq.com/pricing/
- https://grafana.com/docs/grafana-cloud/testing/synthetic-monitoring/
