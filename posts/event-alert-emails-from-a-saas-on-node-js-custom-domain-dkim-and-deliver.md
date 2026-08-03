# Event alert emails from a SaaS on Node.js: custom domain, DKIM, and deliverability

Use a transactional-only sender like Postmark or Amazon SES when your SaaS pushes event alert emails out of Node.js and delivery is the only thing you care about; reach for Resend or SendGrid when you'd rather buy the template editor and the dashboards than build them. The provider choice is a two-hour decision. The part that decides whether the alert actually lands in someone's inbox is the sending domain setup — a dedicated subdomain, DKIM, SPF, a DMARC record — plus the discipline to never send the same alert twice.

I run a one-person SaaS. Every hour I spend on infra is an hour not spent on the thing customers pay for, so my bias is to outsource the undifferentiated part and keep the ten lines of glue in my own repo.

Here's the setup I'd build again.

## What does DKIM setup on a custom domain actually take before the first alert emails go out?

Send from a subdomain you don't use for anything else. `mail.example.com` or `alerts.example.com` — never the bare apex you also send invoices and newsletters from. Reputation is tracked per domain, so a subdomain keeps a bad week on alerts from poisoning everything else you send.

The provider hands you three or four DNS records: a DKIM key (a CNAME pair with SES and Resend, a TXT record with some others), an SPF include on the subdomain, and usually a custom return-path/MAIL FROM record so the bounce address aligns with your domain instead of the provider's. Add a DMARC record yourself — nobody adds it for you. Start at `v=DMARC1; p=none; rua=mailto:dmarc@example.com`, read the aggregate reports for a couple of weeks, then move to `p=quarantine` once you've confirmed every legitimate sender for that subdomain passes alignment. Google's sender guidelines make SPF or DKIM the floor for everyone and require both, plus DMARC, plus one-click unsubscribe, once you cross 5,000 messages a day to Gmail. Most solo SaaS never crosses it. Do it anyway — the requirements have only ratcheted in one direction.

Verify from the outside, not from the provider's green checkmark:

```bash
dig +short TXT _dmarc.example.com
dig +short TXT mail.example.com
dig +short CNAME s1._domainkey.mail.example.com
```

Then send one message to a Gmail account and read the raw headers. You want `dkim=pass`, `spf=pass`, and `dmarc=pass` with the DKIM domain matching your subdomain. If DKIM passes but DMARC doesn't, alignment is your problem, not the key.

Budget half a day. DNS propagation is the slow part, and the provider verification poll usually takes a few minutes after the record is live.

## Picking a provider you won't rip out in three months

Everyone in this space delivers mail. What differs is how much product sits on top and how badly you'll want out later.

| Provider | Integration | Templates | Where it hurts |
| --- | --- | --- | --- |
| Amazon SES | SMTP or AWS SDK v3 | Bare-bones; render your own | Sandbox until you request production access; bounce handling arrives via SNS, so you wire more yourself |
| Postmark | SMTP or REST | Layouts + templates, good preview | Transactional focus means it's not a good fit for marketing blasts; separate streams required |
| Resend | REST, first-class React Email | JSX templates in your repo | Younger product, smaller ecosystem of third-party integrations |
| SendGrid | SMTP or REST | Handlebars dynamic templates | Broad surface with a marketing suite attached; more knobs than a one-person team needs |
| Mailgun | REST, inbound routing too | Handlebars templates | Log retention and region features are tier-dependent; read the plan table before committing |

My default for alerts is SMTP against whichever of these I'm on, because SMTP is the one interface all of them speak. Swapping providers then costs three environment variables instead of a rewrite. The catch is that you give up provider-specific niceties — scheduled sends, per-template analytics, batch endpoints — and you have to render templates yourself.

If your alert volume is genuinely tiny and your app already lives in AWS, stick with SES and skip the comparison shopping entirely.

## Templates: keep them in the repo, keep the payload dumb

Provider-hosted templates feel faster on day one. Then you're editing production copy in a web dashboard with no diff, no review, and no way to test the rendered output in CI. I've stopped doing it.

Render in-process instead. MJML if you want tables-based HTML that survives Outlook, React Email if you're already in a TypeScript codebase and want components. Both compile to a string you can snapshot-test.

Two rules that have saved me repeatedly: always send a plain-text alternative alongside the HTML, and never put anything in the subject line you can't reconstruct from the event payload. The text part matters more than people expect — some spam filters score a missing text/plain part, and half the people receiving a 2 a.m. alert are reading it on a watch.

## The Node.js send path: dedupe first, retry second

This is where my cost surprise came from, so let me get specific.

My alert bill for one month came in at $214 against a normal run rate of about $12. Cause: a customer's health check was flapping, my dedupe key was the rendered subject line, and a timestamp had crept into that subject during a refactor — so every check produced a "new" alert. Roughly 18,000 messages in six hours, all to four people. Nobody read them. I paid for all of them, and two of those recipients hit the spam button, which cost me more in reputation than in money.

The fix was to make the database decide who gets to send, keyed on the incident and its version — not on anything rendered:

```ts
import nodemailer from "nodemailer";
import { db } from "./db.js";
import { renderAlert } from "./templates/alert.js";

const mailer = nodemailer.createTransport({
  host: process.env.SMTP_HOST!,   // swap the host, keep the code
  port: 587,
  auth: { user: process.env.SMTP_USER!, pass: process.env.SMTP_PASS! },
});

type Alert = { incidentId: string; version: number; to: string; project: string; detail: string };

export async function sendAlert(a: Alert): Promise<"sent" | "duplicate" | "suppressed"> {
  if (await db.isSuppressed(a.to)) return "suppressed";

  // unique index on (incident_id, version) — the insert is the lock
  const claimed = await db.claimAlertSend(a.incidentId, a.version);
  if (!claimed) return "duplicate";

  const { subject, html, text } = renderAlert(a);
  await mailer.sendMail({
    from: '"Example alerts" <alerts@mail.example.com>',
    to: a.to,
    subject,
    html,
    text,
    messageId: `<${a.incidentId}.${a.version}@mail.example.com>`,
    headers: {
      "List-Unsubscribe": "<https://example.com/alerts/unsub?t=TOKEN>, <mailto:unsub@mail.example.com>",
      "List-Unsubscribe-Post": "List-Unsubscribe=One-Click",
    },
  });
  return "sent";
}
```

Call that from a queue worker, not from the request handler that detected the event. Any real queue will do — BullMQ if you already run Redis, a `pending_alerts` table with `SELECT ... FOR UPDATE SKIP LOCKED` if you don't. Retry on connection errors and provider rate limits with exponential backoff, give up after four attempts, and log the failure somewhere you'll actually see it. Because the claim row already exists, a retry can't turn into a second delivery.

Suppression is the other half. Every provider posts bounce and complaint events to a webhook; normalize them into one table and check it before every send, as in the guard above:

```ts
app.post("/hooks/email", express.json(), async (req, res) => {
  const e = normalize(req.body);   // event shape differs per provider
  if (e.kind === "hard_bounce" || e.kind === "complaint") {
    await db.suppress(e.recipient, e.kind);
  }
  res.sendStatus(204);
});
```

Sending to an address that hard-bounced last week is the single easiest way to wreck a young domain's reputation.

## After launch: which numbers are worth watching

Open rates are close to useless now. Apple's Mail Privacy Protection pre-fetches remote images for Mail users who turn it on, which inflates opens and makes the "last opened" timestamp meaningless. I'm not sure what the real share of affected traffic is on my list — the reports don't tell you — so I stopped putting opens on any dashboard.

Watch bounce rate, complaint rate, and delivery latency instead. Google Postmaster Tools gives you the spam-rate number Gmail actually judges you by; the guidance is to stay under 0.3%, and you want to be well under it. Set an alert on your alerting, which sounds silly until the day your sender gets throttled and you find out from a customer.

One more trade-off worth flagging: a dedicated IP is the wrong call at low volume. Below a steady few thousand messages a day, a dedicated IP has too little traffic to build a warm reputation and you're better off on the provider's shared pool. Your mileage may vary if you send bursts — a fleet-wide incident that fans out to every customer at once looks a lot like a spam run to a filter, and that's the one case where a warmed dedicated IP earns its keep.

## References

- Google: Email sender guidelines — https://support.google.com/a/answer/81126
- Apple: Use Mail Privacy Protection — https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- RFC 6376: DomainKeys Identified Mail (DKIM) Signatures — https://datatracker.ietf.org/doc/html/rfc6376
- RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC) — https://datatracker.ietf.org/doc/html/rfc7489
- RFC 8058: One-Click Unsubscribe — https://datatracker.ietf.org/doc/html/rfc8058
- Amazon SES: Creating and verifying identities — https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html
- Nodemailer documentation — https://nodemailer.com/
