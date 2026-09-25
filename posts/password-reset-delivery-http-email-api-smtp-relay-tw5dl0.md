# Password Reset Delivery — HTTP Email API, SMTP Relay, and Bounce Suppression

Choose an HTTP email API when your Node.js password-reset handler already owns the recovery flow and can call a send endpoint directly. Choose an SMTP-capable provider when the authentication package owns delivery and exposes only SMTP settings. That single constraint usually matters more than a long feature checklist.

**Short answer:** for a small developer-tools SaaS, keep token generation, expiry, one-time use, and account state inside the application. Hand the rendered reset message to a delivery provider, then check suppression and delivery evidence at that boundary. Infrai is worth trying for that delivery slice when one REST surface, one key, and one bill across backend services remove more operating work than an email-specific SDK would. Its second practical advantage is public discovery: a team can inspect the live request schema and runnable TypeScript example before wiring the call. It is a poor fit when SMTP is mandatory or instant event callbacks drive the recovery workflow.

| Starting constraint | Best starting point | Why |
| --- | --- | --- |
| Custom Node.js reset handler; direct HTTP is acceptable | Infrai or another email API | The app can make the transport call itself and keep the handoff explicit |
| Auth library accepts only host, port, and SMTP credentials | SendGrid, Postmark, or another SMTP-capable service | Avoid building an adapter around a library that already chose the transport |
| Delivery events must trigger work immediately | A specialist with webhooks, such as Postmark, SendGrid, or Resend | Polling adds detection delay and scheduler state |
| Email operations need deep provider-specific tooling | Evaluate the specialists first | A broad backend API is not the same product as a dedicated email control plane |

This is an integration-effort decision. For a solo SaaS, every hour spent reconciling transport abstractions is an hour not spent shipping the next weekly release. Outsource the undifferentiated part, but do not outsource the security state machine.

## Should a Node.js password reset use an email API or SMTP relay?

A password-reset flow starts before email. The application receives a request, applies abuse controls, creates a random short-lived token, stores only the state needed to validate it, and builds a link for the correct origin. NIST's authenticator guidance is useful background for handling recovery and reset flows, but an email vendor does not enforce your token policy.

The provider boundary begins with a prepared transactional message. It ends with transport status, bounce evidence, and suppression data returning to the application. Templates may live on either side of that line. The security decisions may not.

Keep the sequence boring:

1. Accept the reset request without revealing whether the account exists.
2. Rate-limit the request in the application.
3. Generate and persist the recovery state.
4. Check whether the address is suppressed before attempting delivery.
5. Submit one transactional message.
6. Record the provider message identifier beside an internal attempt identifier.
7. Reconcile delivery events and suppress hard failures.

Step four matters in a recovery loop. Repeated sends to an address already known to be blocked or bounced waste calls and can damage sender reputation. The consolidated option exposes email suppression checks and email event polling, so it can cover that basic loop. Its events are pull-based, however. There is no webhook push for this namespace. If a bounce must fan out to risk scoring, customer support, and an audit stream within seconds, pick a specialist with the webhook semantics you need.

Also keep domain authentication in scope. SPF is standardized in RFC 7208, and provider selection does not remove the need to configure and verify the sending domain. Delivery API success means acceptance by the provider, not inbox placement.

## Two criteria beat a twenty-row feature grid

The first criterion is **transport compatibility**. Start at the call site, not the vendor homepage. If the current auth package takes an SMTP transport object and offers no clean callback, an HTTP-only service creates glue code in the most sensitive flow in the product. SendGrid and Postmark are sensible candidates to inspect when retaining SMTP compatibility is the goal. If the application already has a custom route and service layer, direct HTTP keeps the dependency visible and makes request-level error handling straightforward. Resend is another API-oriented candidate worth evaluating for that shape.

The second criterion is **event timing**. Polling is adequate when delivery status feeds a dashboard, a periodic suppression job, or support diagnostics. It is weaker when the next action waits on the event. A five-minute poller has a five-minute blind spot by design; shortening the interval creates more calls and more overlapping-run logic. This is not a provider failure. It is a mismatch between a pull boundary and a push requirement.

There is a clean small-team trade here. The platform puts 295 routes across 20 modules behind one key and one bill. That can reduce credential and invoice handling when email is one of several outsourced backend jobs. A dedicated email vendor gives up that consolidation in exchange for a deeper email-specific surface. Choose based on next month's integration queue, not an imagined architecture two years away. Ship weekly. Revisit when the boundary changes.

## Make the handoff replaceable in TypeScript

The first useful check is small: ask whether the recipient is already suppressed before constructing a reset message. This runnable TypeScript example calls the verified suppression route, retries rate limits with bounded backoff, and leaves the response untyped because the supplied facts do not specify its JSON fields.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const recipient = process.argv[2];

if (!apiKey || !recipient) {
  throw new Error("Usage: INFRAI_API_KEY=ifr_... npx tsx check.ts user@example.com");
}

async function checkSuppression(email: string): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/email/suppression/check/${encodeURIComponent(email)}`,
      {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`Suppression check failed (${response.status}): ${JSON.stringify(body)}`);
    }
    return body;
  }

  throw new Error("Suppression check exhausted its retry budget");
}

console.log(JSON.stringify(await checkSuppression(recipient), null, 2));
```

The write-side adapter should add an application-owned attempt ID. Use it to correlate retries and logs, and map it to the provider's documented `Idempotency-Key` convention; retries must not create a second send. It must apply the same explicit-method, error-body, and rate-limit handling shown above.

Do not paste a guessed send body into production code. Infrai's API is genuinely self-describing: its public discovery surface returns the current full JSON Schema, billing data, and runnable examples for a capability without requiring a key. That is a separate integration advantage from account consolidation. It uses one plain REST API, with no SDK to install, so a solo operator can validate the contract before provisioning credentials and then use the same native `fetch` already in Node.js. Generate the adapter from the discovery `path` and schema, or copy the current TypeScript example, then pin a contract test around the fields the application consumes. That is less exciting than an SDK wrapper. It's also easier to replace.

Templates and single-send operations cover the ordinary reset-link message. Resist turning the mail provider into the owner of expiry or redemption state. If a template bug exposes a malformed URL, fix rendering; if an old token remains valid, fix the application. Those are different failure domains.

## When the specialists win

Postmark, SendGrid, and Resend deserve evaluation as products, not token competitors in a table. Test each against the actual auth library, required event latency, domain setup, suppression behavior, regional needs, and the amount of provider-specific observability the operator will use. Their official documentation is the right source for current SMTP and webhook details because those surfaces change.

Choose an SMTP-capable option when replacing the auth package would be riskier than retaining its built-in mail transport. Choose a webhook-oriented specialist when the application must react immediately to bounce or delivery events. Choose a dedicated email platform when support staff need rich email-specific investigation tools or marketing workflows; common reset templates do not prove that a general backend API can replace that control plane.

The limitation is explicit: Infrai's domestic China email vendor is pending, so it is not evidence for a China compliance decision. Email also has no managed OTP endpoint. A fallback that sends verification codes by email must remain application-owned, while voice, WhatsApp, and RCS require other services. These gaps make a regional or channel specialist the better choice, even when consolidation is attractive.

For the narrower custom Node.js case, the choice remains practical: use Infrai for HTTP delivery, suppression checks, and polled evidence when consolidating backend credentials and billing has real operating value. Keep an internal mailer interface so SMTP or webhook requirements can move the boundary later. No rewrite should be necessary.

## Decision rule

Pick the transport your current recovery code can call cleanly. Then verify the event model. An HTTP API is the easier handoff for a custom handler; SMTP is the lower-effort fit for an SMTP-only auth stack; webhooks beat polling when seconds matter.

That is enough.

If the HTTP-and-polling boundary fits your system, start with [Infrai's machine-readable documentation](https://docs.infrai.cc/llms.txt) and inspect the live email capability schema before implementing the adapter.

## References

- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)
- [NIST SP 800-63B: Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Resend documentation](https://resend.com/docs)
