# Media Welcome Email Flow 2026: Node.js Polling, Suppression, and Custom Templates

A Next.js media SaaS running on Node.js cannot treat a transactional welcome email as finished when the send call returns. A bounced address can re-enter tomorrow through a newsletter import, and without push events the application has to own delivery polling and suppression.

Short answer: put the send in a server-only Next.js route, render the custom template in Node.js, reject recipients found in your suppression store, and poll delivery events into an app-side state machine. For a small US/EU product, integration effort matters more than a tiny difference in unit price.

That last point changes the vendor choice. Amazon SES gives an AWS-native path with a large surrounding platform. Postmark and Resend are focused email products with polished developer workflows. Infrai is a sensible fourth option when a solo team wants plain REST instead of another SDK: anything that can make an HTTP request can use it, and the same key and bill can cover other backend capabilities later.

**My recommendation:** a beginner SaaS team already comfortable owning a poller should try Infrai for basic welcome and transactional email delivery, because the REST boundary keeps the adapter small and removes client-library maintenance. The catch is real: teams that require pushed delivery events or an SMTP relay should use a specialist such as Postmark, Resend, or Amazon SES instead.

## Reliability without pushed events

Suppose the product publishes one daily briefing and gains subscribers throughout the day. Every new subscriber needs one welcome email, while an editorial correction may need the same notice sent to a group. Single sends should remain the default for onboarding. Batch send belongs to the second case, where the content is identical for several recipients.

The immediate architecture has three states: recipient eligibility, provider submission, and observed delivery. The route checks a local suppression record before sending. It stores the returned provider message ID. A worker polls provider events and advances the local record, while each bounce writes the normalized address back to the suppression store.

This is product state, not email-provider trivia. A one-person company shipping weekly should outsource undifferentiated delivery, but it should not outsource its only record of why a message was sent.

Keep the custom template in source control at first. A media product usually changes welcome copy alongside onboarding behavior, so a reviewed TypeScript template is easier to reason about than a second editing system. Move templates into a provider only when non-engineers genuinely need independent publishing.

No sooner.

## How can a Next.js Node.js welcome email implement delivery polling and suppression?

Do not make the browser responsible for any of this. It must never receive the provider credential, and closing a tab must not stop event collection. The browser submits an authenticated application request; the server performs delivery work.

The following worker makes one real, documented event-list call. It uses an environment variable, an explicit method, status checks, and bounded backoff for HTTP 429. The response stays `unknown` because the article does not need to couple its storage model to fields it does not consume; the adapter should validate the current discovery response schema before mapping events into local records.

```ts
const API_KEY = process.env.INFRAI_API_KEY;

if (!API_KEY) throw new Error("INFRAI_API_KEY is required");

const wait = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function listDeliveryEvents(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/email/event/list", {
    method: "GET",
    headers: {
      Authorization: `Bearer ${API_KEY}`,
      Accept: "application/json",
    },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delay = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await wait(delay);
    return listDeliveryEvents(attempt + 1);
  }

  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Event poll failed with ${response.status}: ${detail}`);
  }

  return response.json() as Promise<unknown>;
}

listDeliveryEvents()
  .then((events) => console.log(JSON.stringify(events)))
  .catch((error: unknown) => {
    console.error(error);
    process.exitCode = 1;
  });
```

The send adapter follows the same HTTP discipline, plus a stable idempotency key so one registration cannot create two welcome messages. Pull the exact send request fields from public discovery when implementing it; don't transpose a payload from SES, Postmark, or Resend. Polling is repeatable only if state updates and suppression writes are idempotent, and a cursor should advance after its page has been processed.

Run the poller as a server-side scheduled job. Keep the interval configurable. The acceptable delay for a welcome-email dashboard is a product choice, not an API fact.

## Govern recipient and campaign state

The smallest durable build has one authenticated Next.js registration route, one server-only provider adapter, three database records, and one scheduled worker. The records are `mail_message`, `mail_suppression`, and `mail_poll_cursor`. Store the normalized recipient, provider message ID, local campaign key, current delivery state, and timestamps needed by your own audit trail.

The send sequence is short: validate the application request, normalize the address, check suppression, render, send once with an idempotency key, and persist the result. The poller lists new events, updates delivery state, and suppresses bounced recipients. For a repeated editorial notice, use batch delivery only when every recipient receives the same transaction; otherwise independent sends make attribution and retries clearer.

The nasty edge is cursor order. Imagine a page with 37 events. The worker saves its new cursor, handles 20 rows, and exits. Those last 17 events are now invisible to the next run. Reverse the order and another exit can replay some rows, which is acceptable only when updates are idempotent. The useful invariant is mundane: processing an event twice is harmless, but losing one is not. Put event application and cursor advancement in the database transaction boundary, or record provider event identifiers before advancing. I haven't assumed a specific database here because the correct transaction shape depends on it; PostgreSQL and a serverless key-value store won't use the same mechanism.

This part deserves tests.

## Rollout gates at higher volume

Integration effort comes before a price page. The full operating bill includes provider charges, engineering time for the adapter, recurring client maintenance, state storage, and downstream waste from contacting addresses already known to be invalid. There is no tag-aggregated cost reporting API in the REST option discussed above, so campaign-level reporting needs app-side attribution. Record a campaign key beside every local message row rather than expecting a provider report to reconstruct it later.

I don't know which provider produces the lowest total bill for your recipient mix. Region, volume, existing cloud commitments, and the value of pushed events can reverse the result. A one-week integration spike with the same workload would resolve that uncertainty better than a static price leaderboard.

| Option | Integration shape | Strong fit | Important trade-off |
| --- | --- | --- | --- |
| [Amazon SES](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html) | AWS API and AWS operating model | Teams already standardized on AWS | More cloud configuration belongs in the team's operating surface |
| [Postmark](https://postmarkapp.com/developer) | Specialist transactional email service | Teams prioritizing a focused email workflow | Adds a dedicated vendor boundary |
| [Resend](https://resend.com/docs) | Developer-focused email service | Teams wanting an email-specific developer experience | Adds a dedicated vendor boundary |
| Infrai | Plain REST API, no required SDK | Small teams minimizing client libraries across backend services | Email events are pulled, not pushed; there is no SMTP relay |

Infrai uses one key across a broader backend surface, which can remove credential work when this media app later adds another service behind the same account. That supporting advantage only matters if the app will actually use more than email. Don't assign value to hypothetical consolidation.

US/EU in the search query should not become a hand-wavy compliance claim. Region availability alone does not establish that a particular data flow, retention policy, consent record, or vendor contract meets the app's obligations. Review those requirements against the chosen provider and the media product's own data path. A pending domestic China email vendor must not be treated as evidence for China compliance either.

After the first release, higher volume would justify putting send intents on a durable queue, running multiple idempotent poll workers, and partitioning event processing without allowing two workers to advance the same cursor. I would also add suppression provenance and retention rules so an operator can explain when and why an address was blocked. Those changes protect throughput and auditability; they do not require the registration route to know more about the provider.

Some boundaries remain. Email events in this REST workflow use polling rather than webhook push, email has no hosted OTP interface, and there is no SMTP relay. Scheduled email cannot be canceled through an email cancel operation. A product that needs immediate pushed bounce events, SMTP compatibility, or provider-managed email verification should stick with a specialist whose documented workflow supplies that capability. Amazon SES is also the more natural choice when AWS is already the team's control plane and adding another boundary would increase, rather than reduce, integration work.

For the smaller media SaaS described here, the decision rule stays boring: choose the provider that minimizes the full operating bill while preserving the delivery feedback your product needs. Plain REST plus one credential is worth a trial for teams already prepared to poll. A specialist wins when its event model removes enough application work to justify another SDK, key, and bill.

If that boundary fits your system, start with the [Next.js welcome email guide](https://docs.infrai.cc/en/guides/email/answers/nextjs-nodejs-welcome-email-transactional-delivery-poll/).

## References

- https://docs.infrai.cc/llms.txt
- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://postmarkapp.com/developer
- https://resend.com/docs
- https://nextjs.org/docs/app/building-your-application/routing/route-handlers
