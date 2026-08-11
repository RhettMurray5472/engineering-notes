# Support KB LLM JSON Extraction: Node.js Schema Triage for Missing Fields and Nulls

Customer-support extraction fails at the boundary, not in the prose. An answer may cite the right private knowledge-base article and still be unusable because `priority` disappeared, `customer_id` became `null`, or `status: "urgent"` does not belong to the declared enum. The operational choice is to treat the model response as an untrusted candidate and enforce the JSON contract in Node.js before it reaches a queue, dashboard, or agent.

**Short answer:** keep the prompt narrow, make the schema explicit, validate the parsed object in application code, and route ambiguous records to a typed retry or review path instead of silently filling values.

That is the experiment constraint: structured output correctness matters more than a slightly nicer extraction prompt. A prompt-only fix can make a happy-path example look clean, but it cannot tell your system whether a missing field means “not present in the ticket,” “not found in the knowledge base,” or “the response was truncated.” The application has to decide that.

## What the support record is allowed to mean

Start with semantics, not syntax. For a support ticket, define each field as one of three things: required and always present, required but nullable when the source has no evidence, or optional because the downstream consumer genuinely does not need it. “Missing” and `null` are different states. Collapsing them early throws away information that is useful during triage. This is also where the contract should name the owner of each decision: product policy decides whether an unanswered question is a review case, while the extractor only reports what the ticket and retrieved documents support. A schema can describe the shape, but it cannot decide that policy for you.

Write it down.

For example, a record can use a stable contract like this:

```ts
type SupportRecord = {
  answer: string;
  sourceIds: string[];
  intent: "billing" | "access" | "technical" | "other";
  urgency: "low" | "normal" | "high" | null;
  needsHuman: boolean;
};
```

The enum is deliberately small. If the model sees “critical” in a ticket, the extractor should not invent a fourth business meaning merely because the word sounds plausible. Map it in a separate normalization step, or reject it for review. Keep the source text and retrieved document identifiers alongside the candidate record so a reviewer can explain the decision later.

The same rule applies to a field such as `urgency`. `null` can mean “the ticket contains no evidence,” while an absent key means “the extractor did not satisfy the contract.” Those states should produce different metrics and, often, different follow-up actions. A practical example is a billing ticket that says “I was charged twice” but never asks for urgency: the record can preserve `urgency: null`, while a response that omits `needsHuman` must be rejected because the routing decision is now unknowable. That distinction keeps a missing model output from masquerading as a legitimate absence in the customer's words.

## How should a Node.js extractor handle missing JSON fields, null values, and enum mismatch?

Put one explicit boundary between model output and business logic. Parse once, check the object shape, validate every required key, and reject unexpected enum values. Do not use truthiness as validation: an empty string, `false`, and `null` each have distinct meanings.

Here is a small, dependency-free gate. It does not repair a bad record. That is intentional.

```ts
type Candidate = Record<string, unknown>;

type CheckResult =
  | { ok: true; value: SupportRecord }
  | { ok: false; reason: string; raw: string };

type SupportRecord = {
  answer: string;
  sourceIds: string[];
  intent: "billing" | "access" | "technical" | "other";
  urgency: "low" | "normal" | "high" | null;
  needsHuman: boolean;
};

const intents = new Set(["billing", "access", "technical", "other"]);
const urgencies = new Set(["low", "normal", "high"]);

function isRecord(value: unknown): value is Candidate {
  return typeof value === "object" && value !== null && !Array.isArray(value);
}

function checkSupportRecord(raw: string): CheckResult {
  try {
    const parsed: unknown = JSON.parse(raw);

    if (!isRecord(parsed)) {
      return { ok: false, reason: "root is not an object", raw };
    }

    const required = ["answer", "sourceIds", "intent", "urgency", "needsHuman"];
    const missing = required.filter((key) => !(key in parsed));
    if (missing.length > 0) {
      return { ok: false, reason: `missing keys: ${missing.join(", ")}`, raw };
    }

    if (typeof parsed.answer !== "string" || parsed.answer.length === 0) {
      return { ok: false, reason: "answer must be a non-empty string", raw };
    }
    if (!Array.isArray(parsed.sourceIds) ||
        parsed.sourceIds.some((id) => typeof id !== "string")) {
      return { ok: false, reason: "sourceIds must be an array of strings", raw };
    }
    if (typeof parsed.intent !== "string" || !intents.has(parsed.intent)) {
      return { ok: false, reason: "intent is outside the enum", raw };
    }
    if (parsed.urgency !== null &&
        (typeof parsed.urgency !== "string" || !urgencies.has(parsed.urgency))) {
      return { ok: false, reason: "urgency must be null or a known enum value", raw };
    }
    if (typeof parsed.needsHuman !== "boolean") {
      return { ok: false, reason: "needsHuman must be boolean", raw };
    }

    return {
      ok: true,
      value: parsed as SupportRecord,
    };
  } catch {
    return { ok: false, reason: "response is not valid JSON", raw };
  }
}
```

The cast at the return line is safe only because the checks immediately above establish the fields this local type requires. In a larger service, a schema library can express the same contract and produce structured issues; the important design decision is the boundary, not the library choice.

Prompt wording still matters, but it has a narrower job. Tell the extractor to emit only the declared keys, to emit `null` only where the contract permits it, and to choose enum values exactly as spelled. Include one example with an absent fact and one with a known fact. Then test the prompt against tickets that contain negation, copied email threads, multiple requests, and no matching knowledge-base source.

## The failure loop should preserve evidence

A rejected response needs a reason code, not a generic “retry.” Record the model text, ticket identifier, schema version, validation reason, latency, and token counts under an access policy suitable for support data. Redact secrets before logs leave the service. A retry should receive a concise failure category and the original task context; it should not receive an ever-growing transcript of previous failed attempts.

There are three useful exits. A syntactic failure can be retried with a strict JSON reminder. A semantic failure, such as an unknown enum, should usually go to normalization or human review. A missing source match should remain an explicit null or review state, depending on the field contract. Retrying every branch hides the real defect and adds latency to the queue.

Keep these counters separate: invalid JSON, missing key, wrong type, null where forbidden, unknown enum, and review decision. A single “schema error rate” is too coarse to guide a fix. I’m not sure one retry policy fits every support queue; your mileage may vary with ticket mix and the cost of delaying a human response.

## What should be measured before changing the prompt?

The useful baseline is not just valid JSON. Measure contract pass rate, field-level completeness, allowed-null rate, enum rejection rate, source citation coverage, p95 extraction latency, retry rate, and the percentage sent to review. Break results down by ticket language, message length, and intent if those dimensions are available and governed.

A simple test matrix catches more than a large pile of ordinary examples:

| Case | Expected result | Why it matters |
| --- | --- | --- |
| All facts present | Complete record | Establishes the happy path |
| No urgency evidence | `urgency: null` | Tests nullable semantics |
| Required key omitted | Rejection with `missing keys` | Prevents silent defaults |
| `intent: "critical"` | Rejection or explicit normalization | Protects enum meaning |
| Malformed JSON | Rejection with parse reason | Separates syntax from semantics |
| Conflicting ticket messages | Review or defined precedence | Tests business policy |

Run this matrix on every schema version change. Store a few representative failing payloads as fixtures, with personal information removed, so a prompt edit cannot trade away correctness in one intent while improving another.

## The trade-off: strict gates add work

The catch is that strict validation creates visible rejects. That is a feature for a support system, but it can feel slower than accepting a plausible object and patching it with defaults. Do not choose this approach when the output is disposable prose, when a human always rewrites every field, or when the business has not defined what `null` means. In those cases, a lighter parser may be enough; define the contract before adding machinery.

For an automated answer over a private knowledge base, I would keep the gate and make review cheap. The system should be able to show the ticket, selected source IDs, candidate JSON, and exact rejection reason in one place. If the team cannot inspect that bundle, stricter validation will increase queue friction without making the failure easier to resolve.

This also limits vendor lock-in. The extraction interface can accept plain text, return a typed candidate, and leave model selection, retrieval, and retry policy behind adapters. Switching providers then changes an adapter and its evaluation fixtures, not the support workflow's meaning of `intent` or `urgency`.

The decision rule is straightforward: optimize the contract first, then the prompt, then the model. Measure each change against the same fixtures. Ship only when the record is valid, its uncertainty is represented honestly, and a rejected answer has a useful next step.

## Sources

- https://json-schema.org/specification
- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/parse
- https://platform.openai.com/docs/guides/embeddings
- https://elevenlabs.io/docs

## References

- https://json-schema.org/specification
- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/parse
- https://platform.openai.com/docs/guides/embeddings
- https://elevenlabs.io/docs
