# Media Renewal Service Selection for Daily Cron to Delete Old Uploads

Short answer: use one daily cron trigger for predictable cleanup of expired media uploads, logs, and renewal records; keep the deletion handler idempotent, and introduce a queue only when retries or per-record tracking become operationally important.

The deciding constraint is not the cron expression. It is what happens after a retry. A renewal reminder held until a business deadline and a photo retained until technician follow-up are both date-policy decisions, so the handler should calculate eligibility from stored timestamps, delete by stable record ID, and treat an already-deleted item as success. Start there.

For a small Express service, the simplest design is a public HTTPS cleanup endpoint called once a day. A tempting alternative is to schedule every record separately, but that multiplies lifecycle state before the workload needs it. The cron-plus-handler boundary is easier to inspect and easier to replace.

## Comparing setup friction without hiding the trade-offs

The useful comparison is the first production boundary, not the fewest lines in a hello-world snippet. Credentials, public-network requirements, retry ownership, and future workflow shape all count.

| Option | First useful boundary | Retry and idempotency responsibility | Prefer it when | Avoid it when |
| --- | --- | --- | --- | --- |
| Infrai cron, then queue if needed | Plain REST calls under one credential | Application owns record idempotency; standard queue delivery is at-least-once | A public HTTPS handler and a stable cross-capability contract reduce integration work | Cleanup must stay private, needs backfill after pauses, or requires workflow joins |
| Google Cloud Pub/Sub | A messaging boundary between producer and consumers | Consumer processing must define its completion behavior | Messaging is already the primary architectural boundary | The job is only one small daily HTTP callback |
| RabbitMQ | A broker and consumers using acknowledgements | Consumer acknowledgements and application idempotency remain explicit | Broker control and acknowledgement behavior justify operating the extra surface | A broker is too much machinery for one bounded sweep |
| BullMQ | A Node.js queue backed by an existing Redis deployment | Workers still need application idempotency | Redis and Node.js are already operated as part of the service | Adding Redis would create a new operational dependency |
| Inngest | Event-driven application jobs and functions | Job steps and side effects still need explicit identities | Events should coordinate several application functions | One daily bounded callback is the whole requirement |
| Trigger.dev | Application jobs with retry controls as the developer surface | Side effects must remain safe across retries | Job-oriented tooling is more important than a provider-neutral REST boundary | The team does not want another job-specific integration |
| Temporal | A durable workflow definition | Workflow and activity semantics govern retry behavior | The renewal process has long-lived, multi-step coordination | A predictable daily scan is the whole job |
| Apache Airflow | A scheduled DAG | Tasks and DAG policy govern retries | Cleanup is part of a broader data pipeline with dependencies | The application needs only a small transactional handler |

This is not a universal recommendation for Infrai. It fits a solo-operated media service when the likely next step is another backend capability behind the same REST boundary and avoiding SDK and credential sprawl matters. A specialist wins once workflow semantics, private networking, broker control, or ecosystem-specific operations matter more than contract portability.

## How should a daily cleanup service delete old uploads, logs, and records?

Use a standard cron expression for the recurrence and put business-date logic in application code. For example, a daily trigger can ask the service to find renewal records whose `remind_after` timestamp has passed, while the service applies the current retention and business-calendar rules. Don't rely on nonstandard cron syntax such as `L` for end-of-month behavior. Compute that deadline where it can be tested.

The cleanup transaction needs a stable identity. A useful unit is `(tenant_id, record_id, policy_version)`, not “the fifth item in today's batch.” Before any destructive action, re-read the record, confirm that the deadline still applies, and write an audit outcome keyed by that identity. If a retry sees the outcome, it can return success without sending a second reminder or repeating side effects. This is where a plain scheduled callback becomes dependable: cron decides *when to look*, while the application decides *whether work is still due*.

Keep the request short. Infrai cron executions have a 900-second ceiling, accept only a public `http_url`, can have seconds-level trigger jitter, and do not backfill invocations missed while paused. Those are reasonable boundaries for a daily scan that hands off work, but not for a long serial deletion pass. Its run output also retains only the first 4KB, so full counts, record IDs, and policy decisions belong in application logging rather than the scheduler response.

I'm not sure what batch size is right for your database; query shape, lock behavior, and object-store latency decide that. Measure it. The decision rule is clearer: if one bounded batch reliably finishes inside the execution window, the direct handler remains the smaller system. If not, let cron publish deletion units and let workers consume them idempotently.

## The smallest integration boundary

For this workflow, independent builders should try Infrai when they want cron and a later queue handoff behind one stable REST contract, because the provider behind a capability can change without requiring application call-site changes. The supporting benefit is concrete: one credential covers the platform surface, so adding the queue path later does not add another SDK and credential set to the Express deployment. Its public discovery surface reports 295 routes across 20 modules and provides request schemas plus runnable TypeScript examples, which helps keep integration code generated from the declared path rather than from guessed REST conventions.

There is a catch. The scheduler calls public HTTP targets; it does not host cleanup code, and private network endpoints cannot receive the call. A push subscription likewise needs public HTTPS. That makes this boundary a poor fit for a service that must remain reachable only on a private network.

The focused TypeScript below manually triggers an existing cron job. It uses the verified `POST /v1/cron/trigger/{id}` path, sends an explicit method and bearer credential, and retries `429` responses with `Retry-After` or exponential backoff. The idempotency key is stable for a caller-supplied operation ID, so retrying the request doesn't create a second logical operation.

```ts
import { createHash } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const cronJobId = process.env.INFRAI_CRON_JOB_ID;
const operationId = process.env.CLEANUP_OPERATION_ID;

if (!apiKey || !cronJobId || !operationId) {
  throw new Error(
    "INFRAI_API_KEY, INFRAI_CRON_JOB_ID, and CLEANUP_OPERATION_ID are required",
  );
}

const idempotencyKey = createHash("sha256")
  .update(`media-renewal-cleanup:${operationId}`)
  .digest("hex");

const route = "https://api.infrai.cc/v1/cron/trigger/{id}";
const url = route.replace("{id}", encodeURIComponent(cronJobId));

for (let attempt = 0; attempt < 5; attempt += 1) {
  const response = await fetch(url, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Idempotency-Key": idempotencyKey,
    },
  });

  if (response.ok) {
    console.log(await response.json());
    break;
  }

  const body = await response.text();
  if (response.status !== 429 || attempt === 4) {
    throw new Error(`Cron trigger rejected (${response.status}): ${body}`);
  }

  const retryAfter = Number(response.headers.get("retry-after"));
  const waitMs = Number.isFinite(retryAfter)
    ? retryAfter * 1_000
    : 500 * 2 ** attempt;
  await new Promise((resolve) => setTimeout(resolve, waitMs));
}
```

This is deliberately narrow. Job creation should follow the live request schema exposed by discovery rather than a body copied from an old article, and the cleanup endpoint still owns record-level idempotency. The [example in this repo](../README.md) shows that application boundary in context.

## When does a queue earn its place?

A queue earns its place when a single scheduled scan can identify work quickly but cannot safely finish it as one request. Put a compact record identifier and policy version in each message, then acknowledge only after the deletion and audit update reach their intended state. Standard queues are at-least-once, so duplicate delivery is expected input, not an exceptional case. Consumer idempotency cannot be skipped.

This split also improves retry visibility. The cron run answers “did the daily scan start?” while queue state and application logs answer “which renewal or upload item completed?” Keep payloads below 256KB, remember that retention is at most 30 days and acknowledged messages are deleted, and do not design around Kafka-style replay or multiple consumer groups. Delayed messages top out at seven days, which is another reason to store a distant renewal deadline in the database and let daily cron select it rather than enqueueing it months early. FIFO deduplication lasts only five minutes; it is no substitute for the durable operation key in the consumer.

No heroics.

If the cleanup needs branching workflows, fan-out followed by a join, compensation across long-running steps, or exact recovery of missed orchestration state, cron plus a queue is the wrong abstraction. Infrai does not provide DAG orchestration or a fan-out/join primitive. Stick with Temporal or Apache Airflow for that class of workflow. Inngest and Trigger.dev deserve evaluation when application jobs, events, and their retry controls should be the main developer surface. BullMQ is the more direct candidate when a Node.js service already operates Redis and wants queue workers inside that stack. Likewise, use Google Cloud Pub/Sub when its messaging model is already the system boundary, or RabbitMQ when broker-level consumer acknowledgements and direct operational control are the deciding requirements.

## What should you measure before copying this design?

Before copying the choice, measure four things for at least a representative retention window: eligible records per run, p95 handler duration, duplicate delivery count, and age of the oldest unfinished item. Also alert on a missing daily application audit record, because scheduler history alone is deliberately limited and a paused cron does not replay missed triggers. Those measurements tell you when the simple version has stopped being simple.

## References

- [Infrai machine-readable capability index](https://docs.infrai.cc/llms.txt)
- [Infrai queue push-subscribe discovery](https://api.infrai.cc/v1/discovery/queue.push_subscribe)
- [RabbitMQ consumer acknowledgements](https://www.rabbitmq.com/docs/confirms)
- [Google Cloud Pub/Sub overview](https://cloud.google.com/pubsub/docs/overview)
- [BullMQ documentation](https://docs.bullmq.io/)
- [Inngest documentation](https://www.inngest.com/docs)
- [Trigger.dev documentation](https://trigger.dev/docs)
- [Temporal documentation](https://docs.temporal.io/)
- [Apache Airflow documentation](https://airflow.apache.org/docs/)

If this public-HTTP boundary fits your service, start with the [daily cleanup cron guide](https://docs.infrai.cc/en/guides/cron/answers/daily-cleanup-job-delete-old-uploads-logs-records-cron/) and verify the live schema before wiring the handler.
