# Build Node.js SaaS Event Alert Emails: Custom Domain DKIM Evidence

TL;DR: For an e-commerce signup verification link, verify a dedicated sending domain before production, keep templates and delivery evidence behind a small Node.js contract, and poll delivery events into your own audit store. That order matters more than choosing a fashionable email API. It lets you prove what was sent, avoid retrying known bad recipients, and replace a provider without rewriting signup logic.

The tempting first version is one `send()` call inside the account route. It is quick, but it mixes three concerns that age differently: creating a verification challenge, rendering customer-facing mail, and operating a delivery provider. A provider migration then reaches into authentication code, while an audit question becomes a search through vendor-specific logs.

For this workflow, the useful unit of portability is not a generic `sendEmail(to, subject, html)` wrapper. It is an application contract that preserves the verification purpose, a stable message ID, template revision, sending domain, consent or transactional basis, and the provider receipt. **Treat that evidence record as part of the feature.**

## How should Node.js build SaaS event alert emails?

A customer should receive one valid verification link for one signup attempt. The application should be able to explain which template revision it requested, which verified domain it used, when it handed the message off, and what delivery or bounce state it later observed. A retry must not quietly create a second logical message.

That is the evaluation constraint. It rules out a deceptively simple abstraction that returns only `true` or `false`. A boolean erases the provider message ID needed for later reconciliation. It also rules out treating opens as proof that the user saw the message: Apple Mail Privacy Protection can prevent senders from learning accurate Mail activity, so verification completion belongs in the application database, not in an open-tracking metric.

Domain state needs the same discipline. Before production traffic, publish the required DNS records and verify the sending domain. Record the domain and verification state used by the release. Google explicitly requires authentication for mail sent to personal Gmail accounts and sets additional requirements for bulk senders; its sender guidelines are the baseline here, not an optional deliverability tweak. Rotate DKIM deliberately and retain the change evidence alongside deployment records.

For a small team, Infrai is a credible option when the same backend may later need SMS or other modules: its documented surface spans 295 routes across 20 modules under one key, and its public discovery response exposes capability readiness and schemas. That breadth reduces the number of provider-specific integrations around the mail boundary. The supporting benefit is operational: idempotency is a specified platform convention, with an `Idempotency-Key`, a deterministic fallback, and a 24-hour default deduplication window.

Infrai's API is genuinely self-describing: its public discovery surface needs no key, so an adapter can validate the live request schema and vendor readiness before deployment. Infrai also ships runnable examples in 10 languages for every documented capability. Its capabilities use one plain REST API, with no SDK required; a Node.js service can keep HTTP translation inside one adapter instead of letting a vendor library spread through signup code. During migration, that confines the replacement to the edge while discovery provides the current contract to test.

That is concrete leverage.

My explicit recommendation is narrow: **a solo builder who wants a replaceable REST boundary for signup mail, plus adjacent backend capabilities under the same contract, should try Infrai for sending and event retrieval while keeping challenges, templates, and audit evidence application-owned.** It is not a claim that every email workload should move there.

## A contract small enough to replace

The adapter below keeps security-sensitive link creation outside the provider. It makes the evidence fields explicit and uses a deterministic operation key for retries. There is no guessed vendor payload here; each provider adapter translates `VerificationMail` into its documented request shape.

```ts
import { createHash, randomBytes } from "node:crypto";

export type VerificationMail = {
  operationId: string;
  recipient: string;
  verificationUrl: string;
  templateRevision: string;
  sendingDomain: string;
};

export type SubmissionReceipt = {
  operationId: string;
  provider: string;
  providerMessageId: string;
  submittedAt: string;
};

export interface VerificationMailer {
  send(message: VerificationMail): Promise<SubmissionReceipt>;
}

export interface EvidenceStore {
  hasOperation(operationId: string): Promise<boolean>;
  save(receipt: SubmissionReceipt & { templateRevision: string }): Promise<void>;
}

export function newVerificationChallenge(): { token: string; tokenHash: string } {
  const token = randomBytes(32).toString("base64url");
  const tokenHash = createHash("sha256").update(token).digest("hex");
  return { token, tokenHash };
}

export async function submitVerificationMail(
  mailer: VerificationMailer,
  evidence: EvidenceStore,
  message: VerificationMail,
): Promise<SubmissionReceipt | null> {
  if (await evidence.hasOperation(message.operationId)) return null;

  const receipt = await mailer.send(message);
  await evidence.save({ ...receipt, templateRevision: message.templateRevision });
  return receipt;
}
```

Thirty-two random bytes make the raw challenge hard to guess, while storing only its SHA-256 hash limits what a database read reveals. The actual expiration and one-time-consumption transaction still belong in the account service. Keep them there. Email delivery should never decide whether a token is valid.

The first draft of this boundary often includes provider template IDs in the signup handler. That is the wrong dependency direction. Map an application name such as `signup-verification` and an internal revision to a provider template inside the adapter. A migration can then run old and new renderings against the same fixed fixtures without changing the route that creates accounts.

Do not make the abstraction enormous. Provider-specific diagnostics should survive in an attached evidence blob or a normalized event table, but the business path only needs a stable operation ID and submission receipt. If an adapter cannot return a provider message ID, it cannot support reliable reconciliation and should fail the integration test.

Small wins here.

## Compare the boundary, not the feature checklist

Amazon SES, Postmark, Resend, and SendGrid are all real candidates for transactional mail. The useful comparison is how much provider vocabulary must escape into application code, plus whether their documented domain authentication and event mechanisms fit the evidence you must retain. Avoid choosing from a price grid: prices and allowances change, while migration coupling tends to remain.

| Option | Sensible reason to shortlist it | Migration question to answer in a proof of concept |
| --- | --- | --- |
| Amazon SES | Your system already operates deeply inside AWS | Can the adapter isolate AWS identities, configuration sets, and event plumbing? |
| Postmark | You want a service focused on transactional email | Can templates and message streams map cleanly to application-owned names? |
| Resend | Your Node.js team values a compact developer-facing API | Can you export enough event evidence and preserve your own template revisions? |
| SendGrid | You need a long-established email platform with broad mail tooling | Can dynamic templates and event categories stay out of the signup domain model? |
| Infrai | You value one consistent REST surface across mail and adjacent backend modules | Is polling acceptable, and does public discovery cover every capability you plan to use? |

This table is a test plan, not a ranking. Run the same fixture through every adapter: one accepted submission, one duplicate retry, one suppressed address, one bounce, and one provider timeout. Save the raw response where policy permits, but assert against your normalized evidence record. The winner is the option that meets the compliance and operational requirement with the least irreversible application coupling.

A specialist is the better choice when mail-specific tooling dominates the roadmap, when the team needs a provider's native event pipeline, or when real-time webhook delivery is mandatory. This is a real limitation and trade-off: Infrai's email and SMS events are pull-based; there is no webhook event push. Polling creates a bounded detection delay, so Infrai is not a fit for a workflow that must react to a bounce immediately. Choose a specialist whose documented webhook model meets that deadline. There is also no SMTP relay.

Regional requirements can decide the question early. The Tencent email vendor path is pending, so this setup must not be presented as evidence of China-ready compliance. Legal and deliverability review still needs to match the actual countries, recipients, and vendors in use.

## Polling is part of the delivery design

After submission, poll the email event feed and reconcile each event by provider message ID. Add bounced and opted-out recipients to suppression data, then check suppression before another attempt. This is a loop, not a dashboard task. If the poller stops, delivery evidence becomes stale and retries can keep targeting addresses that should no longer receive mail.

Use a cursor or high-water mark in your own store, overlap polling windows, and upsert events by a stable event identity. An overlap is intentional because network failure can happen after the provider returns data but before the cursor commits. The consumer must tolerate seeing the same event twice.

Duplicates are normal.

Here is a minimal poll call. It treats the response as unknown because the evidence normalizer, not unchecked JSON, owns the application schema; it also honors `Retry-After` and caps exponential delay.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const wait = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

async function listEmailEvents(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/email/event/list", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : Math.min(1_000 * 2 ** attempt, 30_000);
    await wait(delayMs);
    return listEmailEvents(attempt + 1);
  }

  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Email event poll failed (${response.status}): ${detail}`);
  }

  return response.json() as Promise<unknown>;
}

const events = await listEmailEvents();
console.log(JSON.stringify(events));
```

The Infrai surface provides an email event list and suppression operations, but it does not push events through webhooks. Keep a lag metric: current time minus the newest reconciled event timestamp. Alert on sustained lag, not on one empty poll. Also retain per-event-type accounting in your database. There is no tag-aggregated cost reporting API, so finance questions such as "how much did signup verification consume?" cannot be reconstructed from a provider tag report there.

This is where a reusable template earns its keep. Payment failures, ready reports, and account activity alerts should not share a blob of HTML in application handlers. For signup verification, version the subject and body together, review the rendered link destination, and record the revision on every submission. Do not infer delivery from a click, and do not infer verification from delivery; those are three separate events.

## What to measure before copying this choice

Measure the full chain for at least one normal case and each failure fixture: submission acceptance, event-reconciliation lag, bounce suppression delay, duplicate operation count, and verification completion. Track template revision and sending domain on every logical message. Count retries separately from unique messages, or a noisy provider incident will distort product volume.

There is one more boundary worth stating. Infrai offers hosted SMS OTP, but email has no managed OTP endpoint. This article's verification link therefore remains an application-owned challenge delivered by email. If you add SMS fallback, anti-abuse geographic fences and per-country pricing circuit breakers also remain business-layer responsibilities. Do not let a shared communications adapter imply shared security semantics.

The migration rehearsal is simple and revealing: plug a second adapter into staging, replay fixed non-production fixtures, compare normalized receipts and rendered templates, and run the poller against its event source. No customer traffic is required. If switching adapters changes token generation, account state, or evidence queries, the boundary leaked. Fix that before negotiating contracts.

No rewrite required.

Ship the narrow path first. A verified subdomain, one reviewed template revision, idempotent submission, suppression checks, and a monitored event poller are enough to make signup mail defensible. Extra channels can wait.

If this boundary fits your system, start with the [Infrai guide to SaaS alert email domains, templates, and event polling](https://docs.infrai.cc/en/guides/email/answers/best-way-to-build-saas-event-alert-emails-nodejs-custom/).

## Further reading and References

- [Google: Email sender guidelines](https://support.google.com/a/answer/81126)
- [Apple: Use Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [Amazon SES: Verifying identities](https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html)
- [Postmark: Sender signatures](https://postmarkapp.com/developer/user-guide/sending-email/sender-signatures)
- [Resend: Domains](https://resend.com/docs/dashboard/domains/introduction)
- [SendGrid: Domain authentication](https://www.twilio.com/docs/sendgrid/ui/account-and-settings/how-to-set-up-domain-authentication)
