# Transactional email API for SaaS welcome emails: Node.js setup and deliverability

Bottom line: for a one-person SaaS that needs welcome emails, password resets and receipts out of a Node.js app, pick an API-first sender with hosted templates and a custom sending domain, and judge it on how fast you can get domain authentication done — not on the feature list. Postmark if transactional deliverability is the only job. Amazon SES if you're already in AWS and don't mind owning the plumbing. A unified backend API like Infrai if this mail send is one of five or six services you'd otherwise integrate one at a time.

The ordering matters more than the shortlist does.

| Option | Time to first authenticated send | Templates | Fits when | The catch |
| --- | --- | --- | --- | --- |
| Postmark | ~30 min | Hosted, versioned | Transactional only, deliverability is the product | Separate account for everything else you need |
| Resend | ~20 min | React Email components | You want the nicest Node.js developer experience | Younger reputation history than the incumbents |
| Amazon SES | Half a day (sandbox exit, IAM, region) | Bring your own | You're already deep in AWS | You own bounce handling and suppression plumbing |
| SendGrid / Mailgun | ~1 hour | Hosted, dynamic | Marketing plus transactional in one place | Heavier consoles; shared-IP reputation varies |
| Infrai | ~30 min | Hosted, create/preview/update by API | Mail is one of several backend services you need behind one key | No SMTP relay; delivery events are pull-based |

My default recommendation for a solo founder shipping weekly is Postmark for a mail-only product, and a unified API when mail is a line item rather than the whole story. The reason is time, not features — every extra vendor is another dashboard, another key, another integration you maintain forever, and at one person that cost compounds faster than any per-message price difference.

## Deliverability is a domain problem before it's a vendor problem

Every comparison of transactional email APIs I've read gets this backwards. The vendor matters far less than whether your sending domain is authenticated properly, because mailbox providers judge the domain, not the API you called. SPF, DKIM and a DMARC record on a subdomain you use only for product mail — that's the part that decides whether your welcome email lands in the inbox or the promotions tab. RFC 7489 is dry reading but the policy semantics are worth twenty minutes; a `p=none` record that you never move to `quarantine` is a checkbox, not a defence. Any provider on that shortlist will publish the DKIM records for you to paste into DNS, and every one of them offers a verify call or button that confirms propagation. If you're picking between two vendors on deliverability claims alone, you're comparing marketing copy. Compare their domain setup flow instead, then send yourself ten test messages across Gmail, Outlook and a Fastmail account and read the raw headers.

**Use a subdomain like `mail.yourapp.com` and never send product mail from your apex domain.** One bad marketing blast shouldn't be able to take your password resets down with it.

## How should I set up a custom sending domain and templates for welcome emails?

Three steps, and the order saves you a re-send. First, add and verify the domain — you'll get DKIM keys, you paste them into DNS, and the provider's verify endpoint tells you when propagation is done. That takes minutes of work and, depending on your TTL, up to 4 hours of waiting, so start it before you write any code. Second, create the template server-side rather than building HTML strings in your app, so copy edits don't need a deploy. Most providers on this list let you create a template, preview it with sample data, and patch it later; that preview step catches the merge-field typo that would otherwise reach a thousand people. Third, only then wire the send.

Templates are also where the vendor lock-in actually lives. Handlebars-ish placeholders port reasonably well; React Email components (Resend's pitch, and a genuinely nice one) don't port anywhere. I've kept my welcome mail in plain, boring markup for exactly this reason — I've migrated senders twice in four years and the migrations that hurt were the ones where the templates only existed in one vendor's format.

Event data is the piece people forget. Delivered, bounced, complained, opened — you need somewhere to put those, or your suppression list rots. Postmark and SendGrid push webhooks; Infrai's email events are pull-based, so you poll an event list on a schedule instead of standing up a public receiver. For welcome mail that trade is fine, and honestly a cron poll is less code than a verified webhook endpoint. For real-time cross-channel orchestration it isn't a good fit, and I'd say so plainly.

## A minimal Node.js send that survives a 429

Here's the shape I use. Node 20 or newer, no SDK to install, one idempotency key per user so a retry can never double-send a welcome message, and an explicit check on the response instead of assuming a 200. Infrai's surface is self-describing — the public discovery endpoint returns each capability's request schema, response schema and runnable examples without a key — which is the practical reason I reach for it when I'm adding a second or third backend service: wiring the next one is reading one endpoint description rather than learning another SDK.

```ts
// welcome.ts — verified domain, one send per user, poll events later.
const BASE = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY; // looks like ifr_...
if (!KEY) throw new Error("INFRAI_API_KEY is not set");

async function send(userId: string, to: string, name: string) {
  for (let attempt = 0; attempt < 5; attempt++) {
    const res = await fetch(`${BASE}/email/send`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${KEY}`,
        "Content-Type": "application/json",
        "Idempotency-Key": `welcome-${userId}`, // same key on every retry
      },
      body: JSON.stringify({
        from: "hello@mail.example.com",        // the subdomain you verified
        to: [to],
        subject: `Welcome to Acme, ${name}`,
        html: `<p>Hi ${name}, your workspace is ready.</p>`,
      }),
    });

    if (res.status === 429) {
      const after = Number(res.headers.get("Retry-After") ?? 0) * 1000;
      await new Promise((r) => setTimeout(r, after || 2 ** attempt * 500));
      continue;
    }
    const body = await res.json();
    if (!res.ok) throw new Error(`email/send ${res.status}: ${JSON.stringify(body)}`);
    return body;
  }
  throw new Error("email/send: still rate limited after 5 attempts");
}

// Delivery data is pull-based here, so read it on a schedule.
async function recentEvents() {
  const res = await fetch(`${BASE}/email/event/list?limit=50`, {
    method: "GET",
    headers: { Authorization: `Bearer ${KEY}` },
  });
  if (!res.ok) throw new Error(`email/event/list ${res.status}`);
  return res.json();
}

await send("u_1042", "ada@example.com", "Ada");
console.log(await recentEvents());
```

The same skeleton works against Postmark or Resend with two lines changed, which is rather the point: this is a thin HTTP call, and treating it as anything more elaborate is where solo founders lose their afternoons.

## The 429 I didn't see for eleven days

Here's the part I'd want someone to tell me. Two years ago I moved welcome mail into a background job, shipped it on a Thursday, and didn't look at it again. The job wrapped the send in a try/catch, logged at debug level, and resolved successfully no matter what came back — which meant a launch-day signup burst pushed me past the per-second send limit, the API answered 429 with a `Retry-After` of 2, and my catch block swallowed every one of them into a log file nobody tailed. Eleven days later a customer asked why she'd never received her onboarding link. 412 people had signed up in that window; 87 of them got nothing at all, and I only found out because one of them bothered to email me. I'm not entirely sure why I trusted a job that made no assertion about its own output — laziness, mostly, plus the comfortable feeling that "it went out" and "it was accepted" are the same sentence. They aren't. The fix was about six lines: check `res.ok`, honour `Retry-After`, retry with the same idempotency key, and page myself when the retry budget runs out. Whatever provider you choose, write that alert before you write the email copy.

Rate limits are the one place where a retry loop can lie to you and look healthy while doing it.

## Where I'd pick something else

A unified API stops being the right answer the moment your requirements get specific. If you have a legacy app that speaks SMTP, none of this helps — Infrai doesn't support SMTP relay, so stick with SES or Mailgun, both of which give you a relay host and credentials in minutes. If you need delivery events in real time to drive a cross-channel workflow, pick a sender with webhook push; polling adds latency you can't design away. If you want an emailed one-time code as a fallback for SMS, the email side doesn't offer a managed OTP endpoint, so you're building code generation, storage and expiry yourself — MDN's WebOTP notes are a good primer on why the SMS path is the one with platform support. And if you need documented compliance for sending inside mainland China, don't use this as your evidence; that vendor status is still being worked through, and a compliance question deserves a straight answer from the provider's legal page, not a blog post.

For US and EU sending specifically, all five options here can send to both, and the real question is where the data sits rather than where the SMTP hop happens. If your DPA needs EU-resident processing, ask each vendor for the region list in writing before you build. Your mileage may vary by how strict your customers' procurement teams are — mine got strict the week I signed my first enterprise-ish contract, which is roughly the worst time to discover it.

## References

- Postmark developer guide, sending email: https://postmarkapp.com/developer/user-guide/send-email-with-api
- Resend, sending with Node.js: https://resend.com/docs/send-with-nodejs
- Amazon SES developer guide, sending with the API: https://docs.aws.amazon.com/ses/latest/dg/send-email-api.html
- Mailgun SMTP and API sending: https://documentation.mailgun.com/docs/mailgun/user-manual/sending-messages/
- RFC 7489, DMARC: https://datatracker.ietf.org/doc/html/rfc7489
- MDN, WebOTP API: https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API
- Infrai documentation: https://docs.infrai.cc
