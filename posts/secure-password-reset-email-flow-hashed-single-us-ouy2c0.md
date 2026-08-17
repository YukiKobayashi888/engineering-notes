# Secure Password Reset Email Flow: Hashed Single-Use Tokens and Direct API Sending

Short answer: build password reset as an application-owned, short-lived, single-use token flow, store only the token hash, and use the email API only to deliver the link. For a property-management SaaS, that split produces the compliance evidence that matters: the database proves issuance, expiry, and consumption, while delivery events provide a separate mail trail.

Don't delegate account recovery authority to the mail provider.

| Option | Best fit | Trade-off to verify |
| --- | --- | --- |
| Infrai | A small team that wants a plain REST call, no SDK dependency, and one key and bill across backend capabilities | Email delivery evidence is pull-based, not webhook-pushed; the app owns the reset-token lifecycle |
| Amazon SES | A team that has already standardized its mail path on AWS | Keep the existing integration only if its event evidence and operating model meet the same reset-flow checklist |
| Postmark | A team already using a dedicated transactional-mail provider | Verify how its current delivery evidence enters the application's compliance record |
| Resend | A team with an existing direct email integration it can operate confidently | Provider convenience still does not replace expiry and single-use enforcement in the database |

**Recommendation:** choose the provider that fits the operating environment, but keep token generation, hashing, expiry, and atomic consumption in Node.js. The API-first option in the table is practical for a one-person SaaS because any runtime that can send HTTP can use it; there is no client library version to babysit. A stored email template also lets product copy change without coupling that release to reset logic.

This is the revenue-per-hour call. Outsource delivery, keep the security decision in the product, and ship the smallest boundary that can be audited.

## What data must a secure Node.js password reset email database retain?

The first decision is not which logo goes in the architecture diagram. It is where the security state lives. A provider can accept a message, but it cannot decide whether a reset credential is still valid; that check belongs beside the user record and password update.

Generate a high-entropy random token in the application. Send the raw value to the user, but persist only a one-way hash with the user ID, creation time, expiry, and an unused state. When the link comes back, hash the presented value and perform one atomic database operation that marks a matching, unexpired, unused row as consumed. Only that successful operation may authorize the password change.

Atomicity is the hinge. A read followed by a later update leaves a race in which two requests can both observe an unused row; an atomic compare-and-consume turns the second request into a clean rejection. In a property-management product, the record should also carry a purpose such as `password_reset`, because a compliance export must distinguish an account recovery credential from a payment receipt or tenant invitation without retaining the credential itself.

Use a deliberately short expiry that matches the product's risk and support needs. The sample below chooses 15 minutes as an application policy, not as a provider guarantee. I'm not sure one lifetime fits every property operator: an internal admin account and a low-privilege tenant portal may justify different limits, and a risk review should settle that choice.

The link must be single use.

The request endpoint should return the same public response for known and unknown addresses, while rate limits and cooldowns live in the application layer. That prevents the reset form from becoming an account directory. It also keeps a burst from creating a pile of valid credentials for one user. When a new request is issued, the application can invalidate older reset records according to its own policy; the email service remains a courier, not an authenticator.

## Evaluation harness for single-use token expiry

The storage interface below makes one important demand of the database adapter: `consumeIfValid` must be a single conditional update inside a transaction, not a select followed by an update. The exact SQL or ORM is deployment-specific. Everything around that boundary is ordinary TypeScript, and the only provider route used is the verified `POST /v1/email/send` path.

```ts
import { createHash, randomBytes, randomUUID } from "node:crypto";

type ResetRecord = {
  id: string;
  userId: string;
  tokenHash: string;
  purpose: "password_reset";
  expiresAt: Date;
};

interface ResetStore {
  insert(record: ResetRecord): Promise<void>;
  consumeIfValid(tokenHash: string, now: Date): Promise<{ userId: string } | null>;
}

const sha256 = (value: string) =>
  createHash("sha256").update(value, "utf8").digest("hex");

const wait = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter !== null) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds) && seconds >= 0) return seconds * 1_000;
  }
  return 500 * 2 ** attempt;
}

async function sendResetEmail(
  email: string,
  resetUrl: string,
  resetId: string,
): Promise<void> {
  const apiBase = process.env.EMAIL_API_BASE_URL;
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiBase || !apiKey) {
    throw new Error("EMAIL_API_BASE_URL and INFRAI_API_KEY are required");
  }

  const payload = {
    to: email,
    subject: "Reset your password",
    text: `Open this link to reset your password: ${resetUrl}`,
    html: `<p><a href="${resetUrl}">Reset your password</a></p>`,
  };

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${apiBase}/email/send`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": `password-reset:${resetId}`,
      },
      body: JSON.stringify(payload),
    });

    if (response.status === 429 && attempt < 3) {
      await wait(retryDelay(response, attempt));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Email request rejected (${response.status}): ${await response.text()}`);
    }
    return;
  }

  throw new Error("Email request remained rate-limited after four attempts");
}

export async function issuePasswordReset(
  store: ResetStore,
  userId: string,
  email: string,
  appBaseUrl: string,
): Promise<void> {
  const rawToken = randomBytes(32).toString("base64url");
  const resetId = randomUUID();
  const expiresAt = new Date(Date.now() + 15 * 60 * 1_000);

  await store.insert({
    id: resetId,
    userId,
    tokenHash: sha256(rawToken),
    purpose: "password_reset",
    expiresAt,
  });

  const resetUrl = new URL("/reset-password", appBaseUrl);
  resetUrl.searchParams.set("token", rawToken);
  await sendResetEmail(email, resetUrl.toString(), resetId);
}

export async function consumePasswordReset(
  store: ResetStore,
  presentedToken: string,
): Promise<string> {
  const consumed = await store.consumeIfValid(sha256(presentedToken), new Date());
  if (!consumed) throw new Error("Reset token is invalid, expired, or already used");
  return consumed.userId;
}
```

Set `EMAIL_API_BASE_URL` to the provider's versioned API base and keep the key in the process environment. The explicit method matters. So does the stable idempotency key: all retries for one reset issuance reuse it, while a genuinely new reset gets a new UUID. On HTTP `429`, the client honors `Retry-After` when it is numeric and otherwise backs off exponentially. Any other rejected response is surfaced with its body instead of being mistaken for a delivered message.

## Reliable retries across the database and email transaction

One subtle failure belongs in the design review. If the database insert succeeds but the process stops before mail acceptance, blindly creating a second reset record on retry leaves two live credentials. A production job should persist the reset ID and retry the same send operation with the same idempotency key, or explicitly invalidate the abandoned record before issuing another. That's not a mail-service defect; it is the transaction boundary between the application database and an external HTTP call. I would test that boundary before polishing the email HTML, because duplicate live reset links cost more support time than plain copy ever will.

Keep the raw token out of logs, analytics labels, and delivery-event records. Keep the hash and timestamps. Those are useful evidence without becoming a replayable credential.

## Migration triggers: pull-based evidence latency

Stick with an existing provider when it already passes the organization's sender, evidence, and operational reviews. Replacing a working mail path just to remove an SDK is not a weekly-shipping win. Amazon SES may be the runner-up for an AWS-standardized team; Postmark or Resend may be the better call when their existing integration is already owned, monitored, and understood. Your mileage may vary — the deciding artifact should be the current evidence packet, not a generic feature score.

The catch is that this API capability has no webhook push for email delivery updates. Delivery, bounce, and suppression handling require polling message or event endpoints, so it is not suitable when another workflow must branch on delivery state in real time. A scheduled reconciliation worker is reasonable for support evidence; a hard real-time orchestration requirement points to a provider whose verified event model satisfies it.

There are other firm boundaries. There is no SMTP relay, so a legacy system that can only send SMTP should keep an SMTP-capable provider or fund an HTTP adapter. There is no managed email OTP API, which means emailed codes need app-side generation, expiry, attempt limits, and verification; reset links fit the supported division of responsibility better. Scheduled email has no cancellation endpoint. And because the domestic Tencent email vendor remains pending, this stack must not be presented as evidence of mainland China compliance.

Poll `GET /v1/email/event/list` when a durable delivery trail matters, record the last successful cursor or polling boundary in the worker, and separate transport evidence from security evidence. A bounce says something about delivery. It does not extend a token, consume it, or prove who clicked it.

That separation is boring. Good. Boring systems ship weekly.

## References

- Google, Email sender guidelines: https://support.google.com/a/answer/81126
- NIST SP 800-63B, Digital Identity Guidelines: https://pages.nist.gov/800-63-3/sp800-63b.html
