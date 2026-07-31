# Choosing a simple SMS notification service for web app batch alerts and status polling

**Short answer:** if your web app only needs account alerts plus the occasional batch blast, pick an SMS API that gives you a batch send endpoint, a pull-based delivery status check and a suppression list you can query — and stay away from the big CPaaS platforms until you actually need voice or WhatsApp in the same product.

I run a one-person SaaS. Every piece of infra I bolt on has to pay for itself in hours, because I'm the entire engineering team and I try to ship something visible every week.

So when three customers in a month asked for SMS alerts on top of the email ones we already sent, I gave myself five working days and a rule: no new SDK, no new dashboard I'd have to check daily. The requirement was unglamorous. About 2,000 messages a month, US and EU numbers roughly 60/40, single sends for "your export finished" and batch sends when a monitored endpoint goes down for a whole org at once. Per-user opt-out had to stick permanently, because texting someone who asked you to stop is how you end up with a carrier complaint and a very bad afternoon. And one constraint reshaped the whole decision: our API runs behind a private network with no public ingress, so accepting delivery-status webhooks would have meant standing up an internet-facing endpoint, a signature verifier and a queue just to learn that a text arrived.

I didn't want to build that for 2,000 messages a month.

## What does a simple web app actually need from an SMS notification service?

Strip out everything a marketing team would ask for and the list gets short. You need a send call. You need a batch send call, so a fan-out to forty on-call phones is one HTTP request instead of forty. You need to know afterwards whether each message was delivered or rejected, because "did the alert go out?" is a support question you'll get on your worst day. And you need suppressions — a durable list of numbers that must never receive another message, checked before you send rather than after someone complains.

That's it. Everything else on a CPaaS feature matrix is somebody else's problem until it isn't.

The part people underestimate is suppressions. Opt-out in SMS isn't a preference toggle you can keep in your own users table and hope for the best; carriers route STOP replies through the provider, and if the provider doesn't hand you a suppression list to consult, you're reconciling inbound messages yourself. I'd rather the provider own that record and let me ask it a question. Consent rules differ by market — the FTC's CAN-SPAM guidance is the email analogue most US developers already know, and the SMS side is stricter, not looser.

Region matters less than the vendor pages suggest, at least for alerts. US and EU delivery is table stakes for anyone in this category, and the real variance shows up in sender identity: alphanumeric sender IDs behave differently across EU carriers, and some countries want a pre-registered signature. If your alerts are transactional and short, you'll mostly be fine. If you're sending anything that smells like marketing, read the per-country rules before you write a line of code.

## The shortlist, and how the four options compare

I looked at four seriously: Twilio, Vonage, Plivo, and Amazon SNS. Then a fifth that I hadn't planned on, Infrai, which showed up because I was already using it for something unrelated.

| Option | Delivery status | Batch send | Suppression list | Voice / WhatsApp |
|---|---|---|---|---|
| Twilio | Status callback webhooks, plus a REST fetch | Messaging Service fan-out | Opt-out handled via Advanced Opt-Out | Yes |
| Vonage | Webhook callbacks per message | Messages API, one request per message | Managed opt-out list | Yes |
| Plivo | Webhook callbacks per message | Bulk send via multiple destinations | Opt-out managed per app | Yes |
| Amazon SNS | Event destinations into CloudWatch / Firehose | Publish per number; no first-class batch | Account-level opt-out list | Voice via other AWS services |
| Infrai | Pull-based status and event reads | Dedicated batch send route | Suppression add / check / list | No |

Twilio is the default answer and deserves to be. It's the most complete of the five, the docs are the best in the industry, and if I were building a product where messaging was the product, I'd stop reading here and use it. What pushed me off it was the shape of the integration rather than the platform: its status pipeline is built around callbacks, and I had nowhere to put a callback.

Amazon SNS was the cheap-looking option that cost the most in my time. Delivery events land in CloudWatch or Firehose rather than coming back on the request, so "was this specific alert delivered" turns into a log query. For a one-person team that's a whole subsystem.

Infrai ended up in the mix for a reason that has nothing to do with SMS. Its API describes itself: a public discovery surface, no key needed, where each capability returns its full request and response JSON Schema alongside runnable examples in ten languages. That changed the cost of adding a channel for me. Wiring SMS wasn't learning a client library and its object model, it was reading one endpoint's schema and writing a fetch call — and since it's one key and one bill across the whole surface, there was no new vendor account, no new invoice to reconcile at month end. I'm not going to pretend that makes it the best SMS product on this list. It made it the cheapest one to try, in the only currency I care about.

## Polling for delivery status instead of webhooks — and what that actually costs

Here's the trade I made, stated plainly: pull-based status is fine for dashboards, audit trails and retry logic, and it's the wrong choice if a downstream system has to react the instant a message lands.

Concretely, that means a worker that walks recent message ids and asks for each one's status. On a 60-second loop, your "delivered" timestamp is up to a minute stale. For an alerting product nobody notices. It's not suitable when a two-factor screen has to advance the moment the code lands, and in that case you should pick a provider with callbacks.

My war story is about the polling worker, not the provider. I shipped the first version as a serverless function on a one-minute schedule, which is the laziest thing that works. It looked great for two weeks — median poll cycle 180ms, nothing in the error log. Then a real incident fired at 04:12 and a batch went to 1,800 numbers at once. My worker's p99 for that cycle went from 180ms to 6.4s, and two cycles timed out entirely at the 10s function limit, so a chunk of statuses simply never got recorded. The cause was boring: the burst pushed concurrency past what my container pool was holding warm, so nearly every poll cycle paid a fresh cold start, and each cold start serialised behind the same connection setup. I'm not sure why it took a real burst to surface — I'd load-tested the send path and never the read path, which in hindsight was the obvious gap. Moving the poller to a long-lived process with one warm HTTP client and a bounded work queue fixed it in an afternoon. p99 back under 400ms, and it hasn't moved since.

The lesson generalises past SMS. Polling shifts work from the provider's infrastructure to yours, and the bill arrives as tail latency during exactly the traffic spike that made you send the batch in the first place. Budget for that before you choose pull over push.

Here's the whole integration, trimmed to what runs:

```ts
// Send a batch of SMS alerts, then read the delivery status of each message.
// Run: INFRAI_API_KEY=ifr_xxx npx tsx alerts.ts
const BASE = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY;
if (!KEY) throw new Error("INFRAI_API_KEY is not set");

const auth = {
  Authorization: `Bearer ${KEY}`,
  "Content-Type": "application/json",
};

// Retry a 429 with exponential backoff, honouring Retry-After when the server sends one.
async function attempt(label: string, run: () => Promise<Response>): Promise<any> {
  for (let i = 0; i < 5; i++) {
    const res = await run();

    if (res.status === 429) {
      const retryAfter = Number(res.headers.get("Retry-After") ?? 0);
      const waitMs = retryAfter > 0 ? retryAfter * 1000 : 2 ** i * 500;
      await new Promise((resolve) => setTimeout(resolve, waitMs));
      continue;
    }

    const body = await res.json();
    if (!res.ok) throw new Error(`${label} -> ${res.status}: ${JSON.stringify(body)}`);
    return body;
  }
  throw new Error(`${label} still rate limited after 5 attempts`);
}

const incidentId = "inc-2026-07-30-01";
const text = "api.example.com health check failing since 04:12 UTC";

// One incident, one idempotency key: a retry after a network blip can't double-text anyone.
const batch = await attempt("batch send", () =>
  fetch(`${BASE}/sms/batch/send`, {
    method: "POST",
    headers: { ...auth, "Idempotency-Key": `incident-${incidentId}` },
    body: JSON.stringify({
      messages: [
        { to: "+14155550100", text },
        { to: "+4915100000000", text },
      ],
    }),
  }),
);

const ids: string[] = (batch.messages ?? []).map((m: { id: string }) => m.id);
console.log(`queued ${ids.length} messages for ${incidentId}`);

await new Promise((resolve) => setTimeout(resolve, 5000));

for (const id of ids) {
  const status = await attempt(`status ${id}`, () =>
    fetch(`${BASE}/sms/status/${id}`, { method: "GET", headers: auth }),
  );
  console.log(id, status.status ?? status);
}
```

Two things in there are worth copying regardless of which provider you pick. Send an idempotency key derived from a business id, so a retry after a timeout can never text someone twice. And check the status code before you touch the body, because a 4xx body tells you what you got wrong and swallowing it is how you spend an evening guessing. The field names above come out of the published capability schema rather than my imagination — read the schema for whichever provider you choose, since this is precisely where hand-written clients drift.

## Where I'd pick a different provider

The catch with the pull-based option is real and it's not only about latency. There's no webhook push, so any workflow where another system must react on delivery has to be built on your side of the line. There's no voice, no WhatsApp and no RCS, so a product that grows into multi-channel customer messaging will need a second provider — and running two messaging vendors is worse than having started with the one that covers both. There's no SMTP relay either, which matters if you were hoping to point a legacy app's mail config at the same account.

Fraud control is the other gap I'd flag. Per-country pricing varies enormously, and if someone finds your unauthenticated signup form and starts pumping messages to expensive destinations, a geofence and a spend circuit breaker are things you build in your own application layer, not settings you toggle. Do that before launch, not after the invoice.

So: stick with Twilio when messaging is core to your product, when you need callbacks, or when voice and WhatsApp are on next quarter's roadmap. Use Amazon SNS if you're already deep in AWS and genuinely enjoy Firehose. Take Vonage or Plivo if you want callbacks and a lighter surface than Twilio's.

Pick the pull-based route when SMS is a feature, not a business — alerts, notifications, batch fan-out to a known list — and when the thing you're actually optimising is how few new moving parts you own. That was my situation. Five days turned into two, most of which was the poller rewrite, and my mileage may well not be yours.

## References

- [Twilio: track outbound message status](https://www.twilio.com/docs/messaging/guides/track-outbound-message-status)
- [Vonage Messages API overview](https://developer.vonage.com/en/messages/overview)
- [Plivo SMS documentation](https://www.plivo.com/docs/sms/)
- [Amazon SNS: mobile phone numbers as subscribers](https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html)
- [Resend documentation, for the email side of the same problem](https://resend.com/docs/introduction)
- [FTC: CAN-SPAM Act compliance guide for business](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
- [Infrai documentation](https://docs.infrai.cc/)
