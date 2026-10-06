# Clinical Chatbot Moderation API: Safe JSON Gates Without Dedicated Endpoints

A safe in-app chatbot for private clinical knowledge has an awkward constraint: every extra API call delays an answer, while rushed basic moderation can mishandle a question about symptoms, medication, or self-harm. That constraint changes the API choice.

**TL;DR:** When there is no dedicated moderation endpoint, choose an API by testing whether its ordinary generation path can return a strictly validated JSON risk verdict, quickly enough to run before retrieval and again before display. Treat that verdict as one signal in a deterministic policy, not as permission for the model to make a clinical decision. The best fit is the API that holds the required false-negative ceiling and latency budget on your own labeled prompts; a feature checklist cannot establish either result.

The simple design is one call: retrieve private documents, ask the model for an answer and a safety label, then display the answer when the label looks acceptable. It is fast on paper. It also couples two jobs with different failure costs. A polished answer can distract reviewers from a bad label, and unsafe input reaches retrieval before the application has decided whether retrieval should happen at all.

The more defensible experiment uses two small, typed gates around the expensive answer call. That costs another decision point and some latency. I would accept that trade only after measuring it against a one-pass baseline, because a solo team cannot afford ceremony that produces no detectable safety gain.

## How should an API handle basic moderation for a safe in-app chatbot?

Start with actions, not a universal list of forbidden words. The assistant answers questions over an approved private knowledge base. It must distinguish an ordinary policy question from content that needs a refusal, an urgent escalation, or a constrained response. The application, rather than the model, owns those actions.

A compact verdict needs enough information for enforcement and review: a finite decision, one or more risk categories, a confidence value, and a short reason that must not contain private source text. JSON Schema is useful here because it narrows the allowed shape. It does not prove that the classification is correct.

This distinction matters. A parser can establish that `decision` is one of three strings. It cannot establish that a medication question was assigned the right string. **Schema validity is a transport property; safety is an evaluated behavior.**

Types are not judgment.

The threat model should include prompt injection as well as harmful content. OWASP's LLM application guidance treats prompt injection and sensitive-information disclosure as separate risks. A typed response does not neutralize instructions hidden in a retrieved document, and a well-formed answer can still disclose data. Keep document authorization, retrieval filters, output policy, and audit logging outside the model call.

## The experiment: one pass versus two narrow gates

Use a labeled set that resembles actual traffic but contains no live patient data. Include short benign questions, ambiguous symptom language, requests for clinical conclusions, attempts to override system instructions, and adversarial text embedded in knowledge-base excerpts. Split near-duplicates so that a template does not appear in both tuning and evaluation sets.

Then compare two paths. The baseline makes one generation request for an answer plus a verdict. The candidate runs input triage, retrieves only when policy allows it, generates the answer, and runs output triage before display. Keep the model, prompt fixtures, temperature policy, timeout, and concurrency fixed where the API permits. Otherwise the comparison is mostly noise.

Consider one concrete fixture: “Can I take another dose?” arrives with an instruction to ignore policy, while the authorized knowledge set contains a dosing document and an unrelated benefits document. Record whether the input gate detects the override attempt, whether authorization excludes the unrelated document, whether the answer avoids making a patient-specific dosing decision, and whether the output gate selects the expected action. Run a benign paraphrase through the same path. This single pair exercises more of the real boundary than a hundred copies of an obvious banned phrase, yet it still does not stand in for the full labeled set.

Do not summarize the result with accuracy alone. A dataset dominated by benign questions makes an always-allow classifier look good. Record false negatives by category, false positives, invalid JSON, timeouts, and p50/p95 end-to-end latency. For health-related escalation, define the maximum acceptable misses before the test; moving that line after seeing results defeats the exercise.

A useful acceptance table is small enough to argue about:

| Measure | Why it matters | Decision use |
|---|---|---|
| False negatives by risk class | Misses have unequal consequences | Hard release gate for each critical class |
| False positives | Excess blocking destroys utility | Compare against the one-pass baseline |
| Invalid or extra fields | The policy engine needs predictable input | Must fail closed into a known action |
| p95 added latency | Tail delay shapes the chat experience | Must fit the product's declared budget |
| Timeout rate | A gate that disappears under load is not a gate | Exercise the fallback path |

Do the latency arithmetic per request path, not per individual call. Two quick calls can still create a slow tail when they are serial. Conversely, the input gate can prevent retrieval and answer generation for a blocked request, so averages may hide the behavior users actually see. Segment allowed, refused, and escalated traffic.

## A verdict contract that stays boring

The contract below is deliberately small. More categories feel precise, but each category needs examples, a policy action, and enough evaluation cases to defend it. Four half-tested labels are worse than two labels whose errors are understood.

```ts
const riskVerdictSchema = {
  type: "object",
  additionalProperties: false,
  required: ["decision", "categories", "confidence", "reason"],
  properties: {
    decision: { enum: ["allow", "refuse", "escalate"] },
    categories: {
      type: "array",
      items: {
        enum: ["prompt_injection", "clinical_risk", "self_harm", "privacy"]
      },
      uniqueItems: true
    },
    confidence: { type: "number", minimum: 0, maximum: 1 },
    reason: { type: "string", minLength: 1, maxLength: 240 }
  }
} as const;

type RiskVerdict = {
  decision: "allow" | "refuse" | "escalate";
  categories: Array<
    "prompt_injection" | "clinical_risk" | "self_harm" | "privacy"
  >;
  confidence: number;
  reason: string;
};
```

A production adapter should reject malformed output before business logic sees it. It should also map transport failures to an explicit policy result. Do not silently turn a timeout into `allow`.

```ts
type GateResult =
  | { kind: "verdict"; value: RiskVerdict }
  | { kind: "unavailable"; cause: "timeout" | "invalid_json" | "transport" };

function nextAction(result: GateResult): "continue" | "refuse" | "review" {
  if (result.kind === "unavailable") return "review";

  if (result.value.decision === "allow" && result.value.confidence >= 0.9) {
    return "continue";
  }

  if (result.value.decision === "refuse") return "refuse";
  return "review";
}
```

The `0.9` value is an example configuration, not a validated clinical threshold. Tune it on held-out cases, version it with the prompt and schema, and report results across several thresholds. One number should never acquire authority merely because it appears in code.

Keep the reason short and operational. Reviewers need to understand why a route was chosen, but free-form rationales invite sensitive text into logs. Store the policy version, schema version, coarse category, action, latency, and request correlation identifier. Store raw content only under a separate, explicit retention policy.

## Where the apparently simple approach fails

The first failure is conflating refusal language with moderation. A response can sound cautious while still revealing a private passage or making an unsupported clinical claim. Evaluate the final text independently of the model's own label.

The second is retrying malformed output without a budget. An unlimited repair loop turns schema enforcement into unpredictable latency. Set one overall deadline, cap attempts, and make the exhausted path observable. Short failure paths are easier to operate.

The third is treating retrieval as harmless. If input triage happens after retrieval, a prompt-injection attempt can pull sensitive context into the generation request. Authorization must narrow the candidate documents first; input policy decides whether the request should proceed; retrieval then operates inside that boundary.

One more trap is category drift. A prompt revision can preserve aggregate accuracy while moving errors from ordinary questions into the high-consequence class. Pin an evaluation set, inspect per-category confusion counts, and require an explicit review when the model, prompt, schema, or policy mapping changes.

## How should the API decision be made?

First, eliminate any API that cannot reliably produce the contract your parser enforces. “JSON mode” is not enough evidence by itself; run adversarial prompts and verify required fields, enums, extra-field rejection, and behavior when generation is truncated. The application still validates every response locally.

Next, test quality against the latency budget at realistic concurrency. Do this with the same region, network path, and request sizes expected in production. Record the full distribution. A low median does not compensate for a p95 that repeatedly pushes the chat beyond its target.

Finally, inspect operational fit: deadline controls, cancellation behavior, usage reporting, data-handling terms, retention controls, and the ability to export enough metadata for incident review. These are selection criteria, not proof of safety. Evidence comes from the end-to-end system test.

Avoid choosing on token price alone. A cheaper call that raises false positives can create review work, while a marginally faster classifier may not change end-to-end latency once retrieval dominates. Compare total request-path cost and quality on the same corpus. Keep the adapter generic so the evaluation can be rerun against another conforming endpoint without rewriting policy code.

The limitation is direct: two serial model gates are not suitable when the measured tail latency exceeds the chat budget, when review is unavailable for uncertain cases, or when a dedicated deterministic control already covers the risk better. The one-pass design has a latency advantage. My choice would still depend on category-level misses, because extra calls without a measured quality gain are just extra cost and another failure surface.

No shortcut fixes that.

## What to measure before copying this design

Before shipping, measure the one-pass and two-gate paths on the same frozen test set. Publish the confusion counts internally, not just percentages. Include malformed output, timeouts, and policy fallbacks in the release report. Then replay the test whenever any component that can alter a verdict changes.

**The decision rule is blunt:** adopt the second gate only if it reduces the predefined high-consequence misses enough to justify its p95 latency and operating burden. If it does not, change the rubric, model, or surrounding controls and test again. Do not hide a weak classifier behind stricter JSON.

For a private clinical assistant, a schema-constrained generation API can support basic moderation when no dedicated endpoint exists. It cannot carry the whole safety case. The durable design is a measured pipeline: deterministic authorization, narrow typed judgments, local validation, explicit failure actions, and category-level regression tests. Ship the smallest version that clears those gates, then keep measuring it.

## Sources

- https://owasp.org/www-project-top-10-for-large-language-model-applications/
- https://openrouter.ai/docs
