# Failed Webhook Delivery Retries in Node.js Message Queues with Rate Limits

Short answer: put webhook delivery behind a durable queue, make expiration a separate idempotent command, and retry only failures that can recover. Rate limits belong at the delivery boundary, not inside the reservation state transition. A reservation should become expired once, even if the queue redelivers the same job five times.

In an education platform, a learner may hold a seat for 10 minutes while a payment or enrollment service catches up. The awkward case is a stale hold whose expiration webhook is accepted by the queue but rejected downstream because the recipient is rate-limited. The fix is not an aggressive retry loop in the Node.js process. It is a small state machine with durable attempts, an explicit next-attempt time, and a receiver-side idempotency key.

## What should happen when the expiration webhook is rejected?

Treat the reservation expiration and the notification as two related outcomes. First, atomically mark the hold expired if its deadline has passed. Then enqueue an event containing the reservation ID, expiration version, and a stable event ID. The event can be delivered late, duplicated, or out of order; none of those conditions should reopen the seat or apply the expiration twice.

A 429 response means “try later,” assuming the receiver documents that interpretation. A timeout, connection reset, or 5xx is usually retryable. A 400 caused by an invalid payload is not fixed by repeating it. The worker must record the difference, because retrying permanent errors consumes capacity that should be serving healthy learners.

The useful unit of work is therefore `deliver-expiration-event`, not `expire-reservation-and-hope-the-webhook-works`. That separation also makes recovery boring: replay the event without running the business transition again.

Keep the command idempotent.

## A small TypeScript worker with a durable retry decision

The queue adapter below is intentionally generic. Its important contract is that claiming a job, delaying a retry, and acknowledging a successful delivery are durable operations. The in-memory map is only a boundary for the example; production state belongs in the queue and a database.

```ts
type DeliveryResult =
  | { kind: "delivered" }
  | { kind: "retry"; delayMs: number; reason: string }
  | { kind: "discard"; reason: string };

type ExpirationEvent = {
  eventId: string;
  reservationId: string;
  expirationVersion: number;
  expiresAt: string;
};

const retryDelaysMs = [30_000, 120_000, 600_000, 1_800_000];

function classifyResponse(status: number, attempt: number): DeliveryResult {
  if (status >= 200 && status < 300) return { kind: "delivered" };
  if (status === 429 || status >= 500) {
    const delayMs = retryDelaysMs[Math.min(attempt, retryDelaysMs.length - 1)];
    return { kind: "retry", delayMs, reason: `HTTP ${status}` };
  }
  return { kind: "discard", reason: `Permanent HTTP ${status}` };
}

async function deliver(event: ExpirationEvent, attempt: number): Promise<DeliveryResult> {
  const response = await fetch("https://enrollment.example.test/hooks/expiration", {
    method: "POST",
    headers: {
      "content-type": "application/json",
      "idempotency-key": event.eventId,
    },
    body: JSON.stringify(event),
    signal: AbortSignal.timeout(8_000),
  });

  return classifyResponse(response.status, attempt);
}
```

The delay is a starting point, not a promise that the receiver will be ready. Add bounded jitter so thousands of holds created during one enrollment window do not retry on the same second. Respect a documented `Retry-After` value when it is valid and within the policy limit. Keep a maximum delivery age as well; an event about a seat that expired yesterday may belong in a dead-letter stream for inspection rather than a live retry queue.

The worker should acknowledge only after a successful response or an intentional discard. A crash before acknowledgement causes redelivery, which is expected. This is why `eventId` must remain stable across attempts and why the receiving endpoint must deduplicate it before applying side effects.

## Why does rate limiting change the retry design?

A rate limit is shared state. If every worker calculates its own allowance, a process restart can reset the counters and create a burst. Put the limiter at the delivery boundary, backed by shared storage or a queue primitive that guarantees the required ordering. Reserve capacity before sending, and release or reconcile it when the request fails before reaching the receiver.

There are two clocks to keep separate: the reservation deadline and the delivery schedule. The first decides whether a seat can still be held. The second decides when a notification may be attempted. Mixing them creates a nasty bug where a delayed webhook causes the worker to call the expiration command again. The command should check the current reservation version and return success for an already-expired matching version.

For example, RabbitMQ exposes consumer acknowledgements and negative acknowledgements, so an application can choose when a message leaves the ready state; its documentation also makes clear that acknowledgements are separate from the application’s business transaction. Amazon SQS uses visibility timeouts and redelivery semantics, which shifts the design toward extending visibility for slow work and making handlers idempotent. BullMQ provides delayed jobs and attempts in a Node.js-oriented API, but the same application-level distinction still matters: a retry setting does not make a reservation transition idempotent.

Those are different operational shapes, not a ranking. Choose based on the guarantees your team can operate: acknowledgement control, visibility-timeout behavior, delayed-job support, dead-letter handling, and the quality of metrics exposed to the worker.\n+
The trade-off is concrete. A managed queue is a poor fit when the team needs broker-level acknowledgement control or local replay of every delivery attempt; an application-oriented delayed-job library is a poor fit when several services need one shared queue contract. A broker with explicit acknowledgements can require more operational ownership. A visibility-timeout queue can simplify operations but makes timeout extension part of the worker design. Pick the boundary that matches the failure you can actually observe and repair.

## Failure modes worth testing before launch

Test the sequence, not just the happy response. Freeze the clock at the hold deadline, deliver the same event twice, return 429 for three attempts, kill the worker after the receiver commits but before the queue acknowledgement, and send an invalid payload that receives 400. The expected result for the duplicated event is one receiver-side state change and two observable delivery attempts.

Also test a late retry after a reservation has been replaced. Include `expirationVersion` in the event and reject an old version at the receiver or state service. A reservation ID alone is too weak if a learner can release a seat and then acquire another hold using the same business identifier.

Watch four numbers: ready jobs, oldest job age, retry count by status class, and dead-letter age. Add structured fields for event ID, reservation ID, attempt, response status, and next-attempt time. Do not log payment data or learner details just to make a webhook easier to debug.

The deployment checklist is short. Confirm that the worker can restart without losing claimed work. Confirm that the queue's retry delay survives process loss. Confirm that the receiver stores idempotency keys for at least the maximum replay window. Confirm that a 429 path honors rate-limit guidance. Finally, page on growing oldest-job age rather than on every individual retry; transient failure is normal, unbounded lag is not.

## Further reading

- RabbitMQ consumer acknowledgements and publisher confirms: https://www.rabbitmq.com/docs/confirms
- MDN HTTP response status codes: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status
- MDN `Retry-After` header: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Retry-After
- Amazon SQS visibility timeout: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/visibility-timeout.html
- BullMQ retries and backoff: https://docs.bullmq.io/guide/retrying-failing-jobs
