# SMS OTP Login for US/EU Support Reports: A Next.js Phone Verification Build Log

For a Next.js phone verification login, the SMS resend button guards an expensive action: sending a generated customer-support report as an email attachment. Two open tabs must not trigger two messages, pass independent checks, or send the report twice.

Short answer: put phone verification, SMS OTP resend policy, and report-send authorization in the Next.js backend; let the browser display a countdown derived from server time; and keep the email template in the application when the attachment and audit record must ship as one reviewed release.

That boundary fits a small product team. Outsource SMS and email transport, but own the policy that decides who may send which customer-support report. It protects revenue per hour because the replaceable plumbing stays behind narrow interfaces while the business rule remains testable.

## How should a phone verification login handle an SMS OTP countdown?

Model verification as a short-lived challenge, not as a phone number plus a timer. The server record needs an opaque challenge ID, a purpose such as `send_support_report`, a destination fingerprint, a resend eligibility time, attempt counters, and a state. The browser receives only what it needs to render the screen: the challenge ID, a masked destination, and the next eligible resend time.

The button can count down locally for responsive UI, but zero means "you may ask again," not "a send is approved." Every resend request goes back through the same server policy. This matters when two tabs reach zero together, when a device clock changes, or when a caller skips the page and invokes the endpoint directly. The backend clock wins.

Use one atomic operation to inspect the current challenge and reserve the next send. A naive read-then-write sequence has a race: request A and request B both read `resendCount: 0`, both pass, and both send. In a concurrency test, the useful expected result is concrete: one request advances the record and the other receives `429` with a retry delay. That isn't an exceptional provider incident; it is normal application policy under contention.

Verification should also be single-use. After a correct OTP, transition the challenge to `verified` and bind it to the intended report action. Don't turn verification into a general permission that can authorize unrelated exports. The subsequent email command consumes that authorization and records its own idempotency key, so refreshing the confirmation page cannot create a second attachment email.

US and EU labels are not enough to define policy. Country availability, retention, consent wording, and abuse thresholds depend on the product and its users. I'm not sure a universal country matrix exists for this workflow; legal review and the application's actual risk data must settle those choices. Put the resulting rules in server configuration rather than branching on browser locale.

## The constraint that changed the build

Template ownership sounds like an email-team preference until the template carries a generated support report. Then it affects releases, testing, and the audit trail.

I would keep this template in the application repository. Its inputs are coupled to code-owned data: report period, customer label, attachment filename, and the identity of the verified requester. A pull request can change the generator, template, and tests together. The message transport still belongs behind an adapter — undifferentiated work is a good outsourcing target — but a remote dashboard template would create a second release path for one user-visible operation.

There is another reason not to use email engagement as proof that the job completed. DKIM defines a way for a signing domain to take responsibility for a message through a cryptographic signature; it does not prove that the intended human opened an attachment. Mail Privacy Protection can also prevent senders from learning about Mail activity. An open pixel therefore cannot replace an explicit application state such as `email_accepted` or a later, authenticated download event.

Keep those states precise. `verified` means the OTP passed. `authorized` means this challenge may request this report. `email_accepted` means the transport accepted the composed message. None of them means the recipient read the report.

Small distinction. Big debugging payoff.

## A focused TypeScript backend example

The smallest useful implementation is a policy core with injected storage, clock, SMS transport, and report mailer. The store's `change` operation must be transactional or use compare-and-swap semantics. The transports expose application verbs instead of vendor URLs, so replacing either one doesn't rewrite the authorization logic.

```ts
type Challenge = {
  id: string;
  purpose: "send_support_report";
  reportId: string;
  maskedPhone: string;
  resendAllowedAt: number;
  resendCount: number;
  maxResends: number;
  state: "pending" | "verified" | "consumed" | "locked";
};

type VerificationResult = "approved" | "rejected";

interface ChallengeStore {
  change<T>(
    id: string,
    update: (current: Challenge | null) => Promise<{ next: Challenge; result: T }>,
  ): Promise<T>;
}

interface SmsTransport {
  resend(challengeId: string, idempotencyKey: string): Promise<void>;
  verify(challengeId: string, code: string): Promise<VerificationResult>;
}

interface ReportMailer {
  send(input: {
    reportId: string;
    template: "support-report-v1";
    idempotencyKey: string;
  }): Promise<{ messageId: string }>;
}

class PolicyError extends Error {
  constructor(
    public readonly code: string,
    public readonly status: 400 | 404 | 409 | 429,
    public readonly retryAfterSeconds?: number,
  ) {
    super(code);
  }
}

type Dependencies = {
  challenges: ChallengeStore;
  sms: SmsTransport;
  mailer: ReportMailer;
  now: () => number;
};

export function createVerificationService(deps: Dependencies) {
  async function resend(challengeId: string) {
    return deps.challenges.change(challengeId, async (current) => {
      if (!current || current.state !== "pending") {
        throw new PolicyError("challenge_not_pending", 409);
      }

      const waitMs = current.resendAllowedAt - deps.now();
      if (waitMs > 0) {
        throw new PolicyError("resend_too_soon", 429, Math.ceil(waitMs / 1000));
      }
      if (current.resendCount >= current.maxResends) {
        throw new PolicyError("resend_limit_reached", 429);
      }

      const nextCount = current.resendCount + 1;
      await deps.sms.resend(current.id, `${current.id}:resend:${nextCount}`);

      const next = {
        ...current,
        resendCount: nextCount,
        resendAllowedAt: deps.now() + 60_000,
      };
      return { next, result: { retryAfterSeconds: 60 } };
    });
  }

  async function verify(challengeId: string, code: string) {
    return deps.challenges.change(challengeId, async (current) => {
      if (!current || current.state !== "pending") {
        throw new PolicyError("challenge_not_pending", 409);
      }

      const result = await deps.sms.verify(current.id, code);
      if (result !== "approved") {
        throw new PolicyError("code_rejected", 400);
      }

      return { next: { ...current, state: "verified" }, result: undefined };
    });
  }

  async function sendReport(challengeId: string) {
    return deps.challenges.change(challengeId, async (current) => {
      if (!current || current.state !== "verified") {
        throw new PolicyError("report_not_authorized", 409);
      }

      const sent = await deps.mailer.send({
        reportId: current.reportId,
        template: "support-report-v1",
        idempotencyKey: `${current.id}:report:${current.reportId}`,
      });

      return { next: { ...current, state: "consumed" }, result: sent };
    });
  }

  return { resend, verify, sendReport };
}
```

The `60_000` value is an example product policy, not a carrier or regional rule. Put it in configuration before deployment and return the server-derived value to the client. The page can calculate `Math.max(0, resendAllowedAt - Date.now())` for display, then refresh from the server after focus changes or a resend response. It can't grant itself permission.

The long paragraph in this build log belongs around the transaction boundary because there is a subtle failure mode: calling an external transport while holding a database transaction can keep a lock open, yet committing before the call can lose the intent if the process stops between those steps. For a modest system, a store that serializes per challenge plus transport idempotency may be an acceptable starting point. At higher volume, write a send intent and challenge transition atomically, then let a worker deliver the intent. The same pattern applies to the report email. Ship the simple version only after its concurrency behavior is tested; otherwise weekly releases buy speed by borrowing an outage from the next week.

## What should change at scale?

Move SMS sends and report emails to a durable job queue, preserve the same idempotency keys, and make workers safe to retry. Separate limits by challenge, destination fingerprint, account, and network signal because each catches a different abuse shape. Exact thresholds need observation; your mileage may vary.

Add structured events for challenge creation, resend allowed or denied, verification approved or rejected, report authorization consumed, and mail accepted. Keep raw OTP values and full phone numbers out of those events. A dashboard should answer whether users are stuck before verification or after report generation without turning logs into another sensitive-data store.

The catch is ownership cost. An application-owned template is not suitable when non-engineers must change regulated copy immediately without a code release; in that case, use a controlled external template system with version pinning, review, and an audit log. A fully managed verification workflow may also be the better boundary when the team cannot operate challenge storage and abuse controls. Conversely, keep the policy in the application when report authorization is product-specific and must be tested in the same release as attachment generation.

I would still start with one template, one explicit purpose, and one queue. Ship weekly. Split services only when observed load or ownership requires it.

## Sources

- https://datatracker.ietf.org/doc/html/rfc6376
- https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
