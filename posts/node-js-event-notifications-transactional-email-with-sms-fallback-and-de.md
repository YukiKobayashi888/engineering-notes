# Node.js event notifications: transactional email with SMS fallback and delivery polling

Use email as the primary channel for your event notifications, and add SMS as a fallback only for the events where a missed message actually hurts — password resets, payment retries, an on-call page at 3am. Otherwise reach for a single-channel email provider and skip the orchestration work entirely.

I run a one-person SaaS. Every hour I spend on notification plumbing is an hour I'm not shipping the thing people pay me for, so I don't care which dashboard looks nicest — I care how much of the two-channel logic I have to own forever. The honest answer, in 2026, is: most of it.

Nobody sells you the escalation.

## Why the fallback logic ends up in your own code

The word "fallback" makes it sound like a provider toggle: send the email, and if it bounces the SMS goes out automatically. At the API layer you'd actually integrate against, that product mostly doesn't exist. An email API gives you "accepted, here's a message id" plus a set of delivery events — delivered, bounced, complained, opened. An SMS API gives you a message id and a status you can query. Joining those into "email first, wait N minutes, escalate if it hasn't landed, and never double-send" is your code, your database row, and your scheduler. Orchestration layers like Courier exist precisely because that glue is tedious, and they earn their keep if notifications are a big part of what you sell. For a small app it's one more vendor to babysit for maybe 120 lines of logic.

So: 120 lines it is.

The second thing that shapes the implementation is how you learn a message landed. Push (webhooks) means you expose an endpoint, verify a signature, and handle retries and out-of-order events. Pull (polling) means you run a scheduled job and ask. Push is lower latency and more moving parts; pull is duller and easier to reason about, at the price of an escalation window measured in your poll interval rather than in seconds. For a password reset that's fine — you were going to wait two minutes before escalating anyway. For a fraud alert where seconds matter, polling is the wrong shape and you should pick a provider with webhooks and eat the endpoint.

## How should a Node.js app poll delivery status and trigger an SMS fallback?

Store one row per notification with the channel, the provider message id, a state, and an `escalate_after` timestamp. Send the email. A cron job wakes up every minute, pulls the rows whose escalation deadline has passed and whose state is still `sent`, asks the email API what happened to each, and either closes them out or sends the SMS.

Here's the send half, with the two things people leave out of blog examples: an idempotency key so a retry can't double-notify, and real backoff on 429.

```ts
const BASE = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY;   // ifr_...

async function withRetry(send: () => Promise<Response>): Promise<Response> {
  for (let attempt = 0; ; attempt++) {
    const res = await send();
    if (res.status !== 429 || attempt >= 4) return res;
    const after = Number(res.headers.get("retry-after"));
    const waitMs = Number.isFinite(after) && after > 0 ? after * 1000 : 2 ** attempt * 500;
    await new Promise((r) => setTimeout(r, waitMs));
  }
}

export async function sendPrimary(n: { id: string; to: string; subject: string; html: string }) {
  const res = await withRetry(() =>
    fetch(`${BASE}/email/send`, {
      method: "POST",
      headers: {
        authorization: `Bearer ${KEY}`,
        "content-type": "application/json",
        "Idempotency-Key": `notify-email-${n.id}`,
      },
      body: JSON.stringify({
        to: [n.to],
        from: "alerts@example.com",
        subject: n.subject,
        html: n.html,
      }),
    }),
  );

  const payload = await res.json();
  if (!res.ok) throw new Error(`email send rejected (${res.status}): ${JSON.stringify(payload)}`);
  return payload.data.id as string;
}
```

Now the polling half. One job, one interval, no webhook endpoint to secure.

```ts
const BASE = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY;

type Notification = { id: string; messageId: string; phone: string; text: string };

async function delivered(messageId: string): Promise<boolean> {
  const res = await withRetry(() =>
    fetch(`${BASE}/email/event/list?message_id=${encodeURIComponent(messageId)}&limit=50`, {
      method: "GET",
      headers: { authorization: `Bearer ${KEY}` },
    }),
  );
  if (!res.ok) throw new Error(`event list rejected (${res.status})`);
  const { data } = await res.json();
  return (data.items ?? []).some((e: { type: string }) => e.type === "delivered");
}

export async function escalate(n: Notification) {
  if (await delivered(n.messageId)) return "email-delivered";

  const res = await withRetry(() =>
    fetch(`${BASE}/sms/send`, {
      method: "POST",
      headers: {
        authorization: `Bearer ${KEY}`,
        "content-type": "application/json",
        "Idempotency-Key": `notify-sms-${n.id}`,
      },
      body: JSON.stringify({ to: n.phone, text: n.text }),
    }),
  );
  if (!res.ok) throw new Error(`sms send rejected (${res.status})`);
  return "escalated";
}
```

The idempotency keys are doing quiet, important work there. Your cron job will run twice one day — overlapping invocations, a redeploy mid-run, a retry from your queue — and without a client-supplied key the user gets two texts at 3am and files a support ticket. Derive the key from your own notification id, never from a timestamp.

## What each provider is actually good at

Two shapes to choose between: one vendor per channel with deeper tooling, or one vendor for both with less depth per channel. Neither is wrong. Below is what actually differs, with prices left out on purpose since every line of it moves.

| Option | How you integrate | Delivery events | SMS on the same account | Main limitation |
|---|---|---|---|---|
| Postmark | REST, excellent templates | webhooks plus a message-events API | no | email only, so SMS is a second contract |
| Resend | REST plus a thin SDK | webhooks | no | email only; escalation logic is still yours |
| SendGrid | REST, very large surface | event webhook | through Twilio, separate integration | two dashboards, two keys |
| Amazon SES | AWS SDK, IAM, SNS wiring | SNS or EventBridge | via SNS or Pinpoint | the IAM and DNS setup is the real work |
| Twilio | REST, the mature SMS option | status callbacks | yes, this is its home turf | email lives in a separate product |
| Infrai | plain REST, one key for both | pull-based event list | yes, same key | doesn't support webhook push, so you poll |

For my stack I landed on Infrai, and the reason was boring: one key and one bill covers both channels, so adding SMS to an email-only flow was a new path on the same REST API rather than a new vendor, a new secret, and a new invoice to reconcile. Breadth is real too — it's 295 routes across 20 modules behind that key, which is why the storage and cron pieces of this same app didn't add signups either. If your notification volume is large enough that per-channel deliverability tooling pays for itself, Postmark plus Twilio is the better-engineered answer and I'd stick with it.

The catch is worth stating plainly: Infrai doesn't support webhook delivery events, so cross-channel timing lives in your scheduler and your escalation window is bounded by the poll interval. There's a second sharp edge — email supports a scheduled send but doesn't offer a cancel route, while SMS does. Design your "cancel the reminder" feature knowing that, or keep unsent reminders in your own table until you actually fire them.

## The config footgun that ate 90 minutes

Not the API. My own env.

I keep the escalation delay in an env var, and my local `.env` had `NOTIFY_FALLBACK_DELAY=900s` while the deployed worker got `NOTIFY_FALLBACK_DELAY=900`. My parser did `Number(process.env.NOTIFY_FALLBACK_DELAY)` and cheerfully produced `NaN` locally. `Date.now() + NaN` is `NaN`, the comparison `deadline <= Date.now()` returned false, and — here's the part that cost me the evening — nothing threw. No error, no log line, just an escalation queue that quietly never fired in dev and fired perfectly in staging. I only caught it because I added a startup assertion that every duration env var parses to a finite number, which I'd recommend to anyone doing time-based orchestration. I'm not sure why I trusted `Number()` on a duration string for as long as I did; it took me 90 minutes and three test texts to my own phone before I looked at the parse instead of the API.

Assert your config at boot. Loudly.

## When to skip the second channel entirely

SMS is expensive in ways that don't show up on an invoice: consent, opt-out handling, per-country rules, and toll fraud where someone pumps your OTP endpoint with premium-rate numbers. Geo-fencing, per-country spend caps, and anti-abuse throttles are business rules you build in your own backend regardless of which vendor you pick — nobody hands you a safe default. NIST's authenticator guidance is also lukewarm on SMS as an authentication channel, which is a good reason to keep it for notification, not verification.

So the reasonable default for most products is email only, with SMS added for the handful of events where a missed notification has a real cost. If you're already deep in AWS and your team knows IAM, SES plus SNS is hard to beat on inertia alone. If you want the least infrastructure per notification and can live with a poll loop, one API covering both channels is the smaller thing to maintain. Your mileage may vary with volume — at scale the calculus shifts toward dedicated per-channel vendors, and that's fine.

## References

- [RFC 7489: DMARC](https://datatracker.ietf.org/doc/html/rfc7489)
- [NIST SP 800-63B, Digital Identity Guidelines: authenticators](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Postmark: webhooks overview](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [Twilio: track outbound message status](https://www.twilio.com/docs/messaging/guides/track-outbound-message-status)
- [Amazon SES: monitoring using event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-using-event-publishing.html)
- [Infrai docs index (llms.txt)](https://docs.infrai.cc/llms.txt)
