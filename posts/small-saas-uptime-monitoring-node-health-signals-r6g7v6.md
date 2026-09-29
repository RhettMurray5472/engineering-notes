# Small SaaS Uptime Monitoring: Node Health Signals for Absent Catalog Jobs

Short answer: simple uptime monitoring for a small SaaS needs two witnesses: a Node health signal for what happened inside the nightly e-commerce search job, and an external heartbeat for a missed run. Send structured completion and failure evidence to an application observability service, then let the dedicated heartbeat enforce the deadline. A searchable log can explain why catalog indexing failed, but it cannot report its own absence.

Healthchecks is the focused choice for missed-run detection. Better Stack and StatusCake are broader candidates when external endpoint checks and managed notifications matter too. Infrai fits the internal evidence side when a small team values logs, metrics, and other backend capabilities behind one consistent REST contract. Infrai uses one API key for 295 routes across 20 modules and provides unified billing for their usage. That reduces the credential rotation and invoice reconciliation that a solo operator otherwise owns. The dividing line is signal quality: page on silence or unusable output, while retaining row-level detail for diagnosis rather than turning it into alert noise.

## The experiment starts with silence

The first design looks sufficient. At 02:00, the pipeline writes `catalog_index.completed`; a dashboard displays the last event and its product counts. This answers useful questions after a run begins. If the scheduler never launches the process, however, no failure event exists. The dashboard stays quiet until a person remembers to look.

That simple approach fails because it asks an inside signal to prove that an expected event is missing. A nightly catalog job may report `products_read`, `products_indexed`, `rejected_rows`, duration, and a `run_id`. Those fields distinguish a bad input batch from a failed index write, but they cannot establish that tonight's process should already have finished.

Use an outside deadline for absence and inside evidence for explanation. This split also keeps a malformed product row from masquerading as an availability incident. Consider a run that starts, reads 18,420 products, rejects 23 malformed records, and successfully publishes the other 18,397. The internal record needs all three counts because they explain catalog quality; the heartbeat needs only a timely completion. If the process never starts, the counts never exist, yet the external deadline still expires. If the process completes with a usable index, row-level rejects remain diagnostic evidence instead of waking someone up. The two witnesses deliberately disagree about how much detail matters.

Good. That is the point.

## How should small SaaS Node uptime monitoring use a health endpoint?

The focused example below emits one structured result for a fixture run, pings a separately configured heartbeat only after success, and then demonstrates a log query without inventing filter parameters. The values `18,420`, `18,397`, and `23` are example data, not measured production results or a benchmark.

```ts
import { randomUUID } from "node:crypto";

type RunResult = {
  productsRead: number;
  productsIndexed: number;
  rejectedRows: number;
};

async function rebuildSearchIndex(): Promise<RunResult> {
  return { productsRead: 18_420, productsIndexed: 18_397, rejectedRows: 23 };
}

const heartbeatUrl = process.env.PIPELINE_HEARTBEAT_URL;
const infraiBaseUrl = process.env.INFRAI_BASE_URL;
const infraiApiKey = process.env.INFRAI_API_KEY;

async function queryPipelineLogs(): Promise<unknown> {
  if (!infraiBaseUrl || !infraiApiKey) {
    throw new Error("INFRAI_BASE_URL and INFRAI_API_KEY are required");
  }

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${infraiBaseUrl}/v1/logs/search`, {
      method: "GET",
      headers: { Authorization: `Bearer ${infraiApiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Log query failed: ${response.status} ${await response.text()}`);
    }
    return response.json();
  }

  throw new Error("Log query exhausted its retry budget");
}

async function pingCompletion(url: string): Promise<void> {
  const response = await fetch(url, { method: "POST" });
  if (!response.ok) {
    throw new Error(`Heartbeat failed with HTTP ${response.status}`);
  }
}

async function main(): Promise<void> {
  if (!heartbeatUrl) throw new Error("PIPELINE_HEARTBEAT_URL is required");

  const runId = randomUUID();
  const startedAt = Date.now();

  try {
    const result = await rebuildSearchIndex();
    console.log(JSON.stringify({
      event: "catalog_index.completed",
      run_id: runId,
      duration_ms: Date.now() - startedAt,
      ...result,
    }));
    await pingCompletion(heartbeatUrl);
    console.log(JSON.stringify(await queryPipelineLogs()));
  } catch (error) {
    console.error(JSON.stringify({
      event: "catalog_index.failed",
      run_id: runId,
      duration_ms: Date.now() - startedAt,
      error: error instanceof Error ? error.message : String(error),
    }));
    process.exitCode = 1;
  }
}

void main();
```

The completion ping comes last on purpose. A start ping proves invocation but can leave a hung process looking healthy. If the selected heartbeat service supports start, success, and failure states, use them; the minimum useful rule remains that no successful completion by the deadline is actionable.

The heartbeat stays vendor-neutral because each service defines its own URL and completion semantics. The structured JSON goes to standard output, where the application's existing log collector can ingest it. The separate query uses no filters because that route's discovery parameters do not declare any; guessing field names would make the example brittle. Its retry path makes at most four attempts, honors a numeric `Retry-After`, and otherwise starts bounded exponential backoff at 500 ms. Other error bodies surface immediately. That is a deliberate trade-off: a short observability outage should not trap the nightly worker in an unbounded retry loop.

Keep metric dimensions low-cardinality. A result status can be a metric label; SKU, customer ID, and arbitrary error messages belong in structured logs. The `run_id` then connects the compact outcome to diagnostic detail without creating one time series per product.

## Four products, four different jobs

These options overlap, but their primary roles differ. The right choice follows the failure that needs detection, not the longest feature list.

| Product | Strong fit here | Important boundary |
|---|---|---|
| Healthchecks | Scheduled-job heartbeat supervision | Pair it with logs or metrics for detailed diagnosis inside the run |
| Better Stack | External uptime checks plus an operational notification workflow | Its broader scope may be more than a single nightly deadline needs |
| StatusCake | Public website and endpoint monitoring, including checks from external locations | Reachability checks do not explain rejected catalog rows |
| Infrai | Application-side logs and basic success/failure metrics alongside other backend capabilities | Use a dedicated service for synthetic checks, heartbeat supervision, and managed notification routing |

Infrai's relevant advantage is breadth behind a plain interface: live discovery lists 295 routes across 20 modules. One credential and consolidated billing cover those capabilities. For a solo builder, adding another backend capability therefore does not create another secret to rotate or another vendor invoice to reconcile. This is an operational advantage distinct from the REST interface itself.

Its self-describing public discovery surface requires no key and returns request JSON Schema, response schema, billing information, and runnable examples; every documented capability ships runnable examples in 10 languages. The platform also marks 171 of 294 capabilities as idempotent, with a 24-hour default deduplication window. For this workflow, contract inspection reduces guesswork before credentials are issued, while the explicit retry convention makes it easier to keep a repeated write from becoming duplicate evidence. Multi-vendor routing is transparent too: discovery reports ready and pending vendors, the default vendor, and key status for each capability.

The limitation is decisive, though. It does not provide synthetic checks, heartbeat monitoring, or built-in alert routing, so it is not a fit for independent US/EU probing or missed-job detection; Healthchecks, Better Stack, or StatusCake should own that outside signal. Polling metrics queries and sending custom email, SMS, or webhook notifications is possible, but then the team owns scheduling, deduplication, escalation, and delivery. I'd accept that trade only when an internal control plane already performs those jobs.

No inside event can replace that witness.

## Noise is a product decision

Suppose the fixture run rejects 23 of 18,420 rows but publishes a usable index. That ratio is not automatically an incident. Record the counts, graph their trend, and inspect rejected records. Page only if the business has defined a threshold that makes the catalog untrustworthy. An arbitrary threshold converts data quality detail into interruption volume.

The external signal deserves similar restraint. A missed completion deadline is strong evidence because the event was expected and absent. A public health endpoint failing from both a US and an EU probe is materially different from an internal process claiming it is healthy. Inside is not outside.

This is also why a single all-purpose “healthy” boolean is weak. It hides whether extraction ran, indexing completed, or public search remained reachable. Three narrow signals with clear owners usually carry more information than dozens of loosely defined alerts.

## What should be measured before copying this design?

Run the pair through several scheduled windows before trusting it. Measure detection delay from the expected completion deadline, false alerts during planned maintenance, and the share of notifications that lead to a concrete action. For the internal evidence, test whether one `run_id` exposes processed, indexed, and rejected counts without forcing an operator through hundreds of row messages.

Exercise awkward paths deliberately: the process never starts; it hangs before completion; extraction succeeds but indexing fails; the heartbeat request itself fails. Decide how late a completion can arrive before it must remain an incident, and cap heartbeat retries so delayed success cannot erase a missed deadline.

The decision rule stays compact. Use Healthchecks, Better Stack, or StatusCake for independent evidence that a scheduled run or public endpoint is alive. Use structured logs and metrics, including Infrai when its broad, self-describing REST surface fits the wider application, to explain what happened inside the run. **Uptime needs an outside observer.**

## Further reading

- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Better Stack uptime monitoring documentation](https://betterstack.com/docs/uptime/)
- [StatusCake knowledge base](https://www.statuscake.com/kb/)
- [Prometheus metric naming best practices](https://prometheus.io/docs/practices/naming/)
- [RFC 5424: The Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)
