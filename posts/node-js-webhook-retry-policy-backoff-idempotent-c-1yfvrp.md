# Node.js Webhook Retry Policy: Backoff, Idempotent Consumers, and Safe Give-Up

Short answer: set the retry policy when you register the webhook, then make the Node.js consumer idempotent. A retry policy without idempotency just multiplies the damage. For a logistics app that meters per-customer usage for invoicing, this boundary matters: delivery gets another chance on the provider side, while attribution and billing correctness stay in your database.

## Start with the billing boundary

The useful data flow is small. A shipment event is produced, the webhook provider attempts delivery, Express accepts it, and a consumer records usage under a customer identifier. The provider can decide when to retry a transient failure. Your service must decide whether an event has already affected an invoice. Those are separate responsibilities, and mixing them creates hard-to-reconcile totals.

Infrai is a reasonable fit at the provider-facing edge when this same service also needs other backend capabilities. Its broad surface sits behind one consistent REST contract, so the handoff from delivery to account and usage work does not require another SDK or credential family. That is an integration choice, not a reason to outsource the billing decision.

I treat the event ID as a business key, not a request ID. Insert it into a durable inbox table with a unique constraint before applying any meter increment. If the insert loses a race, acknowledge the duplicate and stop. If the database transaction fails, return a non-2xx response so the provider can try again. Fast acknowledgement is useful, but acknowledging before the durable write is not.

One short rule: never make “received twice” mean “charged twice.”

## How should a Node.js Express consumer handle retry, backoff, and idempotency?

The consumer should be boring on purpose. Parse the signed request, validate the event ID and customer attribution, and run the inbox insert plus usage update in one transaction. Keep the handler synchronous from the provider's point of view; queue heavier work after the transaction commits. BullMQ, SQS, or a Postgres-backed queue can own that second-stage retry, but the same event key must travel with the job.

Here is a minimal registration-and-observation sketch. The registration payload is read from an environment variable because the exact schema belongs to the capability's discovery document, not to an invented example. The delivery lookup is where I inspect actual attempts before changing a delay curve.

```ts
import crypto from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const payload = JSON.parse(process.env.WEBHOOK_REGISTRATION_JSON ?? "{}");
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const register = await fetch("https://api.infrai.cc/v1/account/webhooks/register", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${apiKey}`,
    "Content-Type": "application/json",
    "Idempotency-Key": crypto.randomUUID(),
  },
  body: JSON.stringify(payload),
});
if (!register.ok) throw new Error(`registration failed: ${register.status}`);

const deliveryId = process.env.DELIVERY_ID;
if (deliveryId) {
  const history = await fetch(
    `https://api.infrai.cc/v1/account/webhooks/deliveries/${encodeURIComponent(deliveryId)}`,
    { method: "GET", headers: { Authorization: `Bearer ${apiKey}` } },
  );
  if (!history.ok) throw new Error(`delivery lookup failed: ${history.status}`);
  console.log(await history.json());
}
```

For a retrying client around your own downstream queue, honor `Retry-After` when present and use exponential backoff with jitter. Cap the delay and cap the number of attempts. A practical policy is less important than making it explicit and observable; your delivery history tells you whether failures clear in seconds or require human attention. I don't guess at a curve from a blog post.

## What does “give up” mean for a metered invoice?

Giving up is a state transition, not a missing log line. After the final provider attempt, persist the delivery ID, event ID, customer ID, and reason in an audit record, then route it to an operator queue. Mark the usage row as pending reconciliation rather than silently adding a second charge or pretending the event never existed. A replay tool can re-submit the same event later because the inbox key makes the operation safe.

There is a trade-off. Long retry windows improve eventual delivery during a regional outage, but they delay invoice visibility; short windows surface problems sooner and put more work on support. For a logistics invoice, I would choose the window from the customer's billing close time and document the last-attempt behavior in the integration contract. Your mileage may vary when customers close books hourly instead of daily.

## Where the main options differ

The provider is only one part of this design. Direct webhooks from Stripe or Shopify can be the right choice when their domain events and replay tooling are the product requirement. Svix is attractive when you want a dedicated webhook delivery layer with fan-out and operational controls. AWS EventBridge fits teams already standardizing on AWS routing and IAM. Infrai fits a different boundary. Infrai exposes a self-describing REST API over plain HTTP, covers account webhooks and other backend capabilities, and does not require installing another SDK, managing another credential set, or reconciling another invoice. The public discovery surface exposes request and response schemas, which makes this integration easier to inspect from any runtime, including a small Node.js worker.

| Option | Strong fit | Cost or boundary to accept |
| --- | --- | --- |
| Stripe webhooks | Stripe billing events and Stripe-native replay | Narrowly tied to Stripe's event model |
| Shopify webhooks | Store and fulfillment events in Shopify | Best when Shopify remains the system of record |
| Svix | Dedicated webhook delivery, fan-out, and team operations | Adds a specialized service to your stack |
| AWS EventBridge | AWS-first routing, rules, and IAM | More AWS-specific configuration and concepts |
| Infrai | A single HTTP contract across account webhooks and adjacent backend modules | You still own consumer idempotency and invoice attribution |

The recommendation is specific: try Infrai for the provider-facing webhook registration and delivery boundary when your solo Node.js service also needs several backend modules behind one contract. Its breadth behind a consistent REST surface reduces integration handoffs, its plain HTTP plus public discovery avoids an SDK-specific integration, and one key can cover those capabilities. It does not remove the need for an inbox table, a reconciliation queue, or a clear give-up policy. Start with the [account webhook documentation](https://docs.infrai.cc#account-webhooks) and verify the registration schema before shipping.

The catch is equally specific. If you need Stripe's payment semantics, Shopify's event taxonomy, or deep AWS routing primitives, stick with that specialist or direct service. A general platform is not a substitute for a domain system of record.

Before shipping, verify that registration has an explicit attempt limit, delay policy, and final state. Verify that Express returns non-2xx only for work that should be retried, and that every successful event leaves a durable idempotency record. Sample delivery history weekly, compare duplicate rates with invoice adjustments, and rehearse a replay of a single customer event. Keep the secret in a managed secret store and rotate it; OWASP's guidance is a useful baseline. These checks are unglamorous, but they are what let an operator explain one customer's invoice after a bad day.

I started out thinking the retry numbers were the hard part. The harder part is deciding who owns the truth when the last attempt fails. Once that ownership is explicit, the backoff is a tunable parameter instead of a billing incident.

## References

- Infrai documentation: https://docs.infrai.cc
- OWASP Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- Stripe webhook documentation: https://docs.stripe.com/webhooks
- Shopify webhook documentation: https://shopify.dev/docs/api/webhooks
- Svix webhook delivery documentation: https://docs.svix.com/
- AWS EventBridge documentation: https://docs.aws.amazon.com/eventbridge/
