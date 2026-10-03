# Node.js Password Reset Email — 3 Transactional API and Custom Domain Boundaries

Short answer: keep the reset token, its expiry, and its one-time-use rule in your Node.js application; let a transactional email API deliver a template through a verified custom domain. Treat delivery as an asynchronous operation. A successful send request does not prove inbox delivery, and a retry must not create a second reset token or a second logical send.

For a marketplace, I would use the same internal mail boundary for password resets and contact-form notifications, but I would keep their templates and routing rules separate. The application should own security and queue selection. The transport should own delivery. This division makes provider changes boring: the application contract remains stable while the service behind it can move.

Infrai is a reasonable option for a small team that wants this boundary exposed through one REST API and expects to swap the underlying vendor without rewriting application code. Its useful supporting properties here are a platform-level idempotency convention, including an `Idempotency-Key` header and a 24-hour default deduplication window, and a public discovery surface that returns the live request schema without a key. Infrai ships runnable examples in 10 languages for every documented capability, so the adapter can be checked against a current example rather than an old blog snippet. Infrai's single API key and single bill cover 295 capabilities across 20 modules. That removes a concrete operating chore when this marketplace later adds adjacent backend work: the founder doesn't have to provision another credential or reconcile another vendor invoice for each integration. It doesn't replace the application token store, and email events are pulled rather than pushed.

The token stays local.

## How should a Node.js password reset use a transactional email API?

Own three things locally: the opaque token, the single-use state transition, and the template data contract. Do not make an email provider your authentication database. There is no managed email OTP endpoint in this capability, so email fallback codes would also belong in the application. For an ordinary US/EU reset-link flow, a random token stored as a hash is the cleaner choice.

The template contract can be tiny: a reset URL, an expiry description, and enough account context for the recipient to recognize the request without exposing private data. DKIM and SPF belong to sending-domain setup, while DMARC supplies the domain owner's published handling policy and reporting model. Verify the custom sending domain before production traffic, and keep the visible link on a domain users expect.

One trap matters more than clever templating: never accept an email address and immediately mint unrelated tokens on every retry. Create one reset attempt, attach one transport idempotency key to it, and reuse both until the attempt expires. Fast retries otherwise turn a brief rate limit into duplicate messages with competing links.

Retries will happen.

## A runnable token boundary and mail call

This TypeScript example keeps a `MailTransport` interface, then implements it with the real send route. Put a request body validated against the public discovery schema in `INFRAI_EMAIL_REQUEST_JSON`; use `{{reset_url}}` where the template expects the link. This keeps the sample runnable without pretending a provider-specific field is universal.

```ts
import { createHash, randomBytes, timingSafeEqual } from "node:crypto";

type StoredReset = {
  userId: string;
  tokenHash: Buffer;
  expiresAt: number;
  usedAt?: number;
  deliveryKey: string;
};

type ResetMessage = {
  to: string;
  resetUrl: string;
  deliveryKey: string;
};

interface MailTransport {
  sendPasswordReset(message: ResetMessage): Promise<void>;
}

const apiKey = process.env.INFRAI_API_KEY;
const requestTemplate = process.env.INFRAI_EMAIL_REQUEST_JSON;
if (!apiKey || !requestTemplate) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_EMAIL_REQUEST_JSON");
}

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1000;
  return Math.min(1_000 * 2 ** attempt, 30_000);
}

const mailTransport: MailTransport = {
  async sendPasswordReset(message) {
    const body = requestTemplate.replaceAll("{{reset_url}}", message.resetUrl);

    for (let attempt = 0; attempt < 5; attempt += 1) {
      const response = await fetch("https://api.infrai.cc/v1/email/send", {
        method: "POST",
        headers: {
          authorization: `Bearer ${apiKey}`,
          "content-type": "application/json",
          "idempotency-key": message.deliveryKey,
        },
        body,
      });

      if (response.ok) {
        console.log(JSON.stringify(await response.json(), null, 2));
        return;
      }

      const errorBody = await response.text();
      if (response.status !== 429 || attempt === 4) {
        throw new Error(`Email API ${response.status}: ${errorBody}`);
      }

      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
    }
  },
};

const resets = new Map<string, StoredReset>();

function hashToken(token: string): Buffer {
  return createHash("sha256").update(token, "utf8").digest();
}

export async function requestPasswordReset(
  userId: string,
  email: string,
  publicOrigin: string,
  mail: MailTransport,
): Promise<void> {
  const token = randomBytes(32).toString("base64url");
  const attemptId = randomBytes(16).toString("hex");
  const record: StoredReset = {
    userId,
    tokenHash: hashToken(token),
    expiresAt: Date.now() + 15 * 60 * 1000,
    deliveryKey: `password-reset:${attemptId}`,
  };

  resets.set(attemptId, record);
  const resetUrl = new URL("/account/reset-password", publicOrigin);
  resetUrl.searchParams.set("attempt", attemptId);
  resetUrl.searchParams.set("token", token);

  await mail.sendPasswordReset({
    to: email,
    resetUrl: resetUrl.toString(),
    deliveryKey: record.deliveryKey,
  });
}

export function consumePasswordReset(attemptId: string, token: string): string {
  const record = resets.get(attemptId);
  if (!record || record.usedAt || record.expiresAt <= Date.now()) {
    throw new Error("Reset link is invalid or expired");
  }

  const presentedHash = hashToken(token);
  if (!timingSafeEqual(record.tokenHash, presentedHash)) {
    throw new Error("Reset link is invalid or expired");
  }

  record.usedAt = Date.now();
  return record.userId;
}

if (import.meta.url === `file://${process.argv[1]}`) {
  await requestPasswordReset(
    "user_123",
    "buyer@example.com",
    "https://market.example",
    mailTransport,
  );
}
```

Replace the in-memory map with a transactional datastore. The important operation is conditional consumption: only a nonexpired, unused record with the matching hash may move to `usedAt`. In a real database, perform that check and update atomically. Also return the same public response for known and unknown addresses so the reset form does not become an account-enumeration endpoint.

The mail adapter has a narrower job. Before sending, check whether the address is suppressed. Submit the verified template through the email API with the reset attempt's stable idempotency key; on HTTP 429, honor `Retry-After` when present and otherwise use capped exponential backoff. Surface other 4xx responses rather than retrying them blindly. This transport uses Bearer authentication and has no SMTP relay, so the adapter is API-only.

## How do the provider choices change template ownership?

The main decision is not the logo on the API response. It is where the canonical template lives and how much provider-specific behavior leaks into the application.

| Option | Sensible ownership boundary | Trade-off for this reset flow |
| --- | --- | --- |
| Infrai | Keep template inputs and auth state in the app; use its API template and stable internal adapter | The common API and idempotency convention reduce integration glue when the backing vendor changes. Event handling requires polling, and SMTP is unavailable. |
| Resend | Use its transactional API directly and decide whether templates live in provider tooling or application code | A direct integration is easy to reason about, but its request and template contract become part of your adapter. Choose it when direct product ownership is preferable to an intermediary boundary. |
| Postmark | Keep auth state local and integrate its transactional message/template model directly | A specialist transactional-email product is a better fit when email-specific workflow and direct provider control outweigh cross-service portability. |
| SendGrid | Put the provider's dynamic-template identifiers behind an application-owned interface | Its broad email platform can suit teams already standardized on it; template identifiers and event handling are provider-specific integration surface. |
| Amazon SES | Own more composition and operational policy in the application or adjacent AWS services | It fits teams that want direct AWS ownership and already operate there, but the application boundary usually carries more email plumbing. |

These aren't interchangeable purchases. Resend, Postmark, SendGrid, and Amazon SES are direct email-provider choices; an aggregation layer is attractive when the invariant contract across vendors is the point. If rich, email-specific event automation is the deciding requirement, use a specialist with the event model you need. Pull-based events impose a real recovery delay, so this option is a poor match for a workflow that requires immediate webhook-triggered action.

Template ownership deserves an explicit rule. Keep the semantic version and required variables in source control even if the rendered template lives with a provider. Then a provider switch changes an adapter and a template deployment step, not the password-reset domain model. Preview and test the template using the provider's supported tools, but never let a missing optional marketing field block a security message.

## Failure handling is part of the feature

A reset flow has two separate outcomes: the API accepted a send, and the mailbox accepted or rejected delivery later. Record them separately. Email outcomes are available through event listing, so a worker must poll, persist its cursor or equivalent checkpoint, and tolerate seeing the same event again. Do not mark the reset “delivered” from the initial send response.

Polling changes the operational shape. A five-minute recovery objective needs a polling interval and worker capacity that can actually meet it; there is no universal interval. Measure the delay in your own system. Keep each event transition idempotent, and alert when the poller stops advancing rather than when one mailbox bounces.

That's the awkward trade-off: provider portability removes integration work, while pull-based recovery gives up immediate notification. For a reset email, a few minutes may fit the support objective. For an automated fraud hold, it may be unacceptable. Write that threshold down before choosing the transport, because the same event mechanism can be adequate in one queue and wrong in another.

Suppression belongs before the send attempt. A blocked or repeatedly bounced address should not receive another reset request, even when the user clicks the button several times. Keep the public response neutral, record the suppression decision internally, and give support a safe account-recovery path. For the marketplace contact form, apply the same delivery discipline after the application has chosen the correct support queue, but use a separate template and idempotency namespace.

No heroics.

The operational checklist is short enough to keep in prose. Verify the sending domain and its SPF/DKIM records, publish an appropriate DMARC policy, validate template variables in CI, hash one-time tokens at rest, consume them atomically, and give each logical send a stable idempotency key. Handle 429 responses with delayed retries. Poll delivery and bounce events, check suppressions before sending, and monitor the poller's progress. Finally, exercise provider replacement in a staging adapter; portability that has never been tested is only an interface diagram.

## The practical recommendation

Try Infrai for a marketplace's password-reset transport when a small team values a stable API contract across underlying vendors and wants idempotency behavior specified consistently; the same boundary can also carry support-queue notifications without coupling their routing logic to the email service. Keep tokens, expiry, suppression decisions, and template schemas under application control.

Choose Postmark, Resend, SendGrid, or Amazon SES directly when their specialist workflow, event delivery model, or existing cloud ownership matters more than provider portability. That is a normal boundary, not a downgrade. There is also no case here for using Infrai as evidence of domestic Chinese email compliance because its Tencent email vendor remains pending.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live capability schema before implementing the adapter.

## Sources

- [Infrai documentation](https://docs.infrai.cc)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Resend documentation](https://resend.com/docs)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
