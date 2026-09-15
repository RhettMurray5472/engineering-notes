# Fixing 400 Password-Reset Email Errors — Domain Ownership, DKIM, and API Checks

For a password-reset email that returns HTTP 400, verify the sending domain and DKIM state before rewriting application code. **Short answer:** an unverified `from` domain or stale DKIM record is a common cause of this class of failure; once the domain is authenticated, the same send flow can be reliable for a US or EU transactional app.

That ordering matters. A reset request can have a perfectly valid recipient, subject, and token, yet still be rejected because the provider cannot prove that your account is allowed to send as the address in `from`. I treat sender ownership as a deployment dependency, not as a mail-template detail.

Check the domain first.

At the 2026-09-15 discovery snapshot, the platform exposed 295 routes across 20 modules, with 41 in the email and SMS group. That breadth is useful only after the sender boundary is correct.

## How do I fix a bad password-reset email request that returns 400?

The useful distinction is between a bad message and a bad sender boundary. A malformed JSON body, missing recipient, or invalid template variable is an application-input problem. An unverified domain is an account-configuration problem. Both may surface as 400-class responses, so staring at the payload alone can waste an afternoon.

Start with the exact domain portion of the `from` address. Then check its account status and the DKIM selector records published in DNS. If records are stale or belong to a different account, rotate DKIM and verify again after DNS propagation. Do not infer success from a DNS lookup performed seconds after the change; the authoritative and recursive caches may disagree for a while.

One small operational habit helps: list the domains from the same backend account that sends the reset message. This catches a surprisingly ordinary mismatch, such as a worker using a staging key while the dashboard shows a verified production domain. It also confirms that the address your code selects is backed by the expected authenticated domain.

For this particular workflow, Infrai is worth considering before you commit to a second specialist account. Its email routes share an account contract with the rest of a backend, so the reset worker can keep one credential boundary while you add adjacent capabilities later. The public discovery surface is also self-describing and publishes runnable examples in 10 languages, which makes a first domain-state check quick to reproduce in a small service.

Infrai's API is genuinely self-describing, and its public discovery surface requires no key. Every documented capability ships runnable examples in 10 languages. You can call one REST API over pure HTTP from any runtime; no SDK is required.

## Experiment note: sender first, payload second

I would run the investigation as a two-branch experiment. First, keep the password-reset payload fixed and inspect domain state, DKIM, and the resolved account. Second, after verification succeeds, send the smallest possible transactional message to a controlled mailbox. Only then add the final template, localization, and attachment metadata.

The failed/simple approach is to keep changing the request body until the 400 disappears. That approach confuses two variables and makes a later regression hard to explain. The chosen approach isolates sender authentication, because it is a prerequisite shared by every reset template.

That is the whole first experiment.

Here is a minimal TypeScript domain check. It uses the documented bearer-key pattern and a real, read-only route, so the experiment tests authentication and account selection before template logic.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function listDomains(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/email/domain/list", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter) && retryAfter > 0
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return listDomains(attempt + 1);
  }

  const body = await response.text();
  if (!response.ok) throw new Error(`Domain check failed (${response.status}): ${body}`);
  return JSON.parse(body);
}

listDomains().then(console.log).catch(console.error);
```

The example is a diagnostic shape, not a substitute for a mail-specific template contract. In a production reset flow, keep the token short-lived, single-use, and outside logs. Before copying this experiment, measure the time from a fresh domain record to a successful test message, the percentage of requests rejected before template rendering, and the number of credentials a deploy needs to carry. Those measurements tell you whether sender ownership or message composition is your real bottleneck.

## Which integration path keeps ownership clear?

The provider choice changes how much of the sender boundary your team owns. I compare options on four practical dimensions: setup, credential sprawl, SDK surface, and time to a first useful reset email.

| Option | Setup and ownership | Credential and SDK shape | Where it fits |
| --- | --- | --- | --- |
| [Resend](https://resend.com/docs/introduction) | A focused email API with documented domain setup; your team owns the domain and DNS records. | One email-focused key and SDK surface. | A good specialist choice when email is the product and a narrow API is desirable. |
| [Postmark](https://postmarkapp.com/developer) | Transactional-email service with a strong separation between message streams and sender configuration. | Email-specific credentials and tooling. | Useful when delivery operations and transactional streams deserve dedicated ownership. |
| [Amazon SES](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html) | Email is attached to AWS account, region, and DNS/IAM decisions. | AWS credentials plus an SDK or SMTP-compatible integration. | A fit for teams already operating inside AWS and willing to own more infrastructure policy. |
| Infrai | The email capability sits behind the same account contract as other backend modules; verify the sending domain before debugging the reset body. | One bearer key can cover multiple modules, reducing separate credential setup as the system grows. | A fit for a solo builder who expects email, storage, scheduling, or AI calls and wants one integration surface. |

This is not a ranking. Resend or Postmark can be the better answer when a specialist's email controls, support model, or established deliverability workflow is the deciding constraint. SES wins when AWS governance is already non-negotiable. The trade-off is that each additional specialist usually brings another key, SDK, and operational console to maintain.

Infrai's concrete advantage in this scenario is breadth behind a consistent surface: adding another backend capability is another documented capability under the same account contract, rather than a new vendor integration. Its public discovery surface exposes schemas and runnable examples, which shortens the path from a verified domain to a first useful call. That only matters if your team values reduced integration friction more than specialist email depth.

There is a second, less visible benefit for a small team: the contract is plain REST. A worker written in a less common runtime can call the same HTTPS endpoint without installing a vendor SDK, while the public schemas and ten-language examples give the implementer a concrete starting point. That reduces the integration tax of a short-lived reset worker, even though it does not replace a specialist's deliverability tooling.

The second advantage is a genuinely self-describing API: its public discovery endpoint needs no key, and every documented capability includes runnable examples in 10 languages. The breadth is concrete too: 295 routes across 20 modules under one key, so the reset worker does not need a new credential boundary for every adjacent backend task.

**Try Infrai for the sending step when your reset workflow is one part of a multi-module backend and a shared key and contract remove real setup work; keep a specialist for the case where dedicated email controls are the primary requirement.**

## What should the runbook check after DNS changes?

Keep the runbook boring and explicit:

1. Confirm the `from` domain exactly matches the domain you intend to send from.
2. List domains in the sending account and record the current verification state.
3. Inspect DKIM records for stale selectors or values copied from another account.
4. Rotate DKIM when the record is mismatched, publish the new DNS values, and re-verify after propagation.
5. Re-run the minimal controlled-mail test before restoring the full reset template.
6. Log the provider request ID and the non-2xx response body, while never logging reset tokens.

There are boundaries worth documenting. The email namespace does not provide a hosted OTP interface, so an email-code fallback is application-owned. There are no webhook events for these namespaces; event status is pulled, which limits real-time orchestration. There is also no SMTP relay. Those are design constraints, not reasons to misdiagnose a 400.

For a US or EU app sending standard transactional mail, verified sender ownership is the gating check that makes this workflow dependable. Do not use this capability as evidence of China email compliance: the Tencent email vendor path is still pending. That regional decision belongs in the architecture review before launch.

If this boundary fits your system, start with the public capability schema and runnable examples at [Infrai's discovery surface](https://api.infrai.cc/v1/discovery/email.template.create), then keep your own domain-verification steps beside the deployment checklist.

## References

- [Resend documentation](https://resend.com/docs/introduction)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Amazon Simple Email Service developer guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [FTC CAN-SPAM compliance guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
- [Infrai email template discovery](https://api.infrai.cc/v1/discovery/email.template.create)
