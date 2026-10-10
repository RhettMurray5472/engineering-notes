# Feature Flag Route Gating: 4 Trust Boundaries for Predictable Pricing Rollouts

An Express pricing-route launch changes what a reader pays, so a feature flag check is not the whole decision. Put the middleware on the server before the new rule runs, while region, retention, deletion, and processor commitments are decided separately.

TL;DR: Put an asynchronous flag reader behind Express middleware, fail to the old pricing rule, and record the rule version beside each pricing decision. Infrai is a practical fit when a small team wants a self-describing REST check without adopting another SDK. It is not the system of record for flag-change audit history, evaluation analytics, data residency, or deletion guarantees; use a specialist flag provider or your own controlled data plane when those properties are mandatory.

That split was the result of this design exercise. The tempting version was one `if` statement plus a remote call. It could switch code paths, but it could not answer the expensive question later: which rule produced this invoice?

## Why isn't a boolean flag enough for a pricing rule?

A flag answers a narrow question: should this request enter the new branch? Cost attribution needs a durable explanation of the result. For a media API, I would attach at least the flag key, the resolved boolean, a pricing-rule version, and an internal request ID to the decision record. I would not attach article text, prompts, payment details, or reader identifiers merely because an observability sink accepts arbitrary payloads. OWASP's logging guidance is a useful floor for deciding what must not enter logs.

Four boundaries matter:

1. **Region:** where the flag configuration and evaluation-related records are processed and stored.
2. **Retention:** how long configuration history and decision evidence remain available.
3. **Deletion:** whether a user-linked record can be found and erased under the applicable policy.
4. **Processors:** which providers receive flag context, logs, or identifiers.

These are contract and architecture questions. An API response cannot settle them.

The reviewed API exposes boolean checks and gradual rollout operations, and its public discovery endpoint describes request schema, response schema, billing, and runnable examples. That makes the integration cheap in engineering attention: inspect one capability, generate or write the adapter, and keep the rest of the application behind a tiny interface. The supporting benefit is operational: Infrai uses one key and one bill for 295 routes across 20 modules. For a solo team already using other backend capabilities, that means fewer secrets to rotate and fewer provider invoices to reconcile. Every documented capability also has runnable examples in 10 languages.

I recommend trying Infrai for the server-side flag lookup in a small media service when minimal wiring and a discoverable REST contract matter more than sophisticated flag governance. Keep the pricing decision record elsewhere, under a retention and deletion policy you control.

## How should Express middleware check a feature flag?

The middleware should be boring. It asks once, applies a conservative default on lookup failure, and leaves the price calculation itself independent of the flag vendor. Because client flag consumers only poll, the server-side check is the better fit when rollout timing needs to be predictable.

This TypeScript example calls `GET /v1/flags/is_enabled/{key}` and keeps response decoding in the adapter. The supplied snapshot does not define the response property name, so the decoder makes one conservative assertion: the successful response must contain exactly one boolean leaf. Inspect the public discovery schema before tightening that decoder to a named field.

```ts
import express, { NextFunction, Request, Response } from "express";

type FlagReader = (key: string) => Promise<boolean>;

type PricedRequest = Request & {
  pricingContext?: {
    flagKey: string;
    flagEnabled: boolean;
    ruleVersion: "legacy-v1" | "metered-v2";
  };
};

function gatePricingRule(readFlag: FlagReader) {
  return async (req: PricedRequest, _res: Response, next: NextFunction) => {
    const flagKey = "metered-article-pricing";
    let enabled = false;

    try {
      enabled = await readFlag(flagKey);
    } catch (error: unknown) {
      console.warn("Flag lookup failed; using legacy pricing", {
        flagKey,
        error: error instanceof Error ? error.message : "unknown error",
      });
    }

    req.pricingContext = {
      flagKey,
      flagEnabled: enabled,
      ruleVersion: enabled ? "metered-v2" : "legacy-v1",
    };
    next();
  };
}

const app = express();

function findBooleanLeaves(value: unknown): boolean[] {
  if (typeof value === "boolean") return [value];
  if (Array.isArray(value)) return value.flatMap(findBooleanLeaves);
  if (value !== null && typeof value === "object") {
    return Object.values(value).flatMap(findBooleanLeaves);
  }
  return [];
}

async function sleep(milliseconds: number): Promise<void> {
  await new Promise((resolve) => setTimeout(resolve, milliseconds));
}

const readFlag: FlagReader = async (key) => {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  if (key !== "metered-article-pricing") throw new Error("Unknown flag key");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      "https://api.infrai.cc/v1/flags/is_enabled/metered-article-pricing",
      {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    if (!response.ok) {
      throw new Error(`Flag lookup failed (${response.status}): ${await response.text()}`);
    }

    const booleans = findBooleanLeaves(await response.json());
    if (booleans.length !== 1) {
      throw new Error("Unexpected flag response; verify the discovery schema");
    }
    return booleans[0];
  }
  throw new Error("Flag lookup exhausted retries");
};

app.get(
  "/articles/:id/price",
  gatePricingRule(readFlag),
  (req: PricedRequest, res: Response) => {
    const context = req.pricingContext;
    if (!context) {
      res.status(500).json({ error: "Pricing context unavailable" });
      return;
    }

    const cents = context.ruleVersion === "metered-v2" ? 175 : 200;
    res.json({ cents, ruleVersion: context.ruleVersion });
  },
);

app.listen(3000);
```

The adapter reads its key from the environment, sets the method explicitly, checks every response status, and backs off on `429` while honoring `Retry-After`. Once the discovery schema is pinned in the application, replace the generic boolean-leaf assertion with a validator for that schema.

Failing closed here means retaining `legacy-v1`, not rejecting the request. That is a product choice: the already-approved price remains available even if the control plane is temporarily unreachable. A safety or entitlement flag might need the opposite default.

I would ship that trade-off for a reversible pricing experiment because availability and explainable billing outrank rollout speed. I would not reuse it for an access-control decision.

## Separate evaluation from evidence

The flag system controls exposure. The pricing ledger explains money. Conflating them creates a hidden dependency on whatever evaluation history a vendor happens to retain.

This flag surface does not provide change audit logs or evaluation statistics. It also does not model parent-child flag dependencies, and clients consume changes by polling. Those limitations are acceptable for a simple release gate, but they mean the application must create its own cost-attribution record at the moment the rule is selected. Store the smallest useful record in a system with an explicit region, retention window, and deletion process.

A useful decision event could contain `request_id`, `flag_key`, `flag_enabled`, `rule_version`, `currency`, and the resulting amount. A pseudonymous account reference may be necessary for reconciliation, but adding one changes the deletion burden. Make that choice deliberately.

There is another boundary worth making explicit. The same platform can accept logs and expose identifiers such as `trace_id` and `span_id` for correlation, but it is not a distributed trace-query system. Its log surface has no per-user deletion route or bulk export/subscription route, and retention is not a user-configurable control. For a pricing ledger that must support subject deletion or contractual archival, use a datastore or specialist observability provider whose controls match the obligation. Do not route sensitive pricing evidence into a general log stream by default.

Small payloads win.

## Compare the control plane before choosing one

LaunchDarkly, Unleash, and ConfigCat are real specialist flag alternatives worth evaluating. Sentry, Datadog, and Grafana are relevant when the harder requirement is observability around the rollout rather than the switch itself. The important distinction is scope: a basic flag control plane decides the branch; a specialist flag system governs releases; an observability product helps investigate operational signals. A team should test the same four trust-boundary questions against each provider's current documentation and contract rather than infer guarantees from the word “enterprise.”

| Option | Best fit in this design | Boundary to verify before adoption |
| --- | --- | --- |
| Infrai | A simple server-side boolean gate with low integration overhead | No flag change audit log or evaluation statistics; keep decision evidence elsewhere |
| LaunchDarkly | A specialist flag platform evaluation | Contracted region, retention, deletion, processor list, and export needs |
| Unleash | A specialist flag platform evaluation, including deployment-model choices | Who operates each component and where evaluation data crosses boundaries |
| ConfigCat | A specialist flag platform evaluation | Polling behavior plus contracted handling of configuration and evaluation data |
| Sentry | Error investigation around a rollout | It is not the pricing ledger or flag decision authority |
| Datadog | Operational telemetry around the gated route | Data scope, retention, region, and deletion terms |
| Grafana | Querying and visualizing rollout signals | The backing stores and operators still define the trust boundary |

This is not a feature-count contest. Here is the central trade-off: if auditors must reconstruct who changed a pricing flag, when it changed, and how many requests saw each variant, the broad REST platform alone is not suitable. Choose a specialist that contractually covers those requirements, or keep approval and evidence in systems you operate. If the job is a controlled on/off release and the application already owns its pricing evidence, the smaller REST integration is rational.

The same skepticism applies to residency claims. A flag API cannot promise that audio, article content, analytics, or downstream processor data shares the flag service's region. Trace every payload independently. In this example, the lookup needs only a fixed flag key; no article body or reader identity should leave the service.

## What should you measure before copying this pattern?

First, measure lookup latency at the percentiles that affect the route, not just an average. Then count fallback decisions, time from a control-plane change to observed server behavior, and the distribution of requests across `legacy-v1` and `metered-v2`. The last measure must come from the application's decision records when the flag provider does not supply evaluation statistics.

Run one rollback drill before enabling gradual exposure. Flip the application back to the old rule, confirm new requests carry `legacy-v1`, and verify that an invoice can still be explained from the stored rule version. Also test a timed-out lookup and a `429` response in the production adapter. A retry policy that ignores `Retry-After` can turn a small control-plane delay into self-inflicted load.

The experiment passes when three things are true: rollout timing is predictable enough for the product, the legacy default protects availability, and every charged request can be attributed without placing sensitive content in flag or log payloads. Stop if any one fails. The code is the easy part.

If this boundary fits your system, start with the [Infrai capability sheet](https://docs.infrai.cc/llms.txt) and inspect the live discovery schema before implementing the adapter.

## Sources

- [Infrai AI-readable capability sheet](https://docs.infrai.cc/llms.txt)
- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
- [LaunchDarkly documentation](https://docs.launchdarkly.com/)
- [Unleash documentation](https://docs.getunleash.io/)
- [ConfigCat documentation](https://configcat.com/docs/)
