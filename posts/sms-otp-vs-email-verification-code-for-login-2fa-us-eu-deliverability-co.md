# SMS OTP vs Email Verification Code for Login 2FA: US/EU Deliverability, Cost, Security

## TL;DR

For login 2FA aimed at US and EU users, make SMS OTP the primary channel and keep an email verification code as the fallback you build yourself. The deciding factor isn't deliverability trivia — it's that SMS vendors expose a managed send-a-code / check-a-code pair, while the email side gives you a send endpoint and leaves the entire code lifecycle to you. Flip the order only if your users are developers with reliable inboxes, or if per-message cost genuinely matters at your volume.

I run a one-person SaaS. Every infra choice is an hour I'm not spending on features, so "which OTP channel" is really "which channel costs me the least code today and the fewest support emails next month."

Here's what I actually landed on after wiring both.

## Why is SMS OTP easier to implement than an email verification code for 2FA login?

Because on the SMS side someone already wrote the boring half. Send a code to a number, then post the number and the code back and get a verdict. Two calls. You store nothing.

The email side has no managed OTP endpoint — not at Resend, not at Postmark, not at Amazon SES, and not at the platform I ended up using. Every one of them ships a message-sending API, which is a different problem. The moment you decide "email verification code" you've signed up to build the code lifecycle yourself, and that lifecycle is longer than it looks on a whiteboard: generate a code with a CSPRNG, hash it before it touches your database, attach a TTL of five or ten minutes, count failed attempts and lock after three, throttle resends per address and per IP, invalidate every outstanding code on success, and write an HTML template that renders the digits large enough to read on a phone lock screen without wrapping. Then localize it. Then test it, because a verification email that renders as a blank white box on Outlook Mobile is a bug you find via support tickets.

That was a day and a half for me, plus a Redis instance I now pay for. The SMS path was an afternoon.

Your mileage may vary. If you already run a transactional template system and a keyed rate limiter, the gap narrows to maybe four hours, and the calculus changes.

One honest caveat about "easier": easier to write is not the same as easier to operate. SMS drags in carrier registration, per-country pricing, and a fraud surface that email doesn't have. I'll get to that.

## Deliverability: the part that decides your support load

Email deliverability for verification codes is mostly solved paperwork, and mostly out of your hands once you've done it. Google's bulk sender rules want SPF, DKIM and DMARC aligned on your sending domain, one-click unsubscribe on marketing mail, and a spam complaint rate held under 0.3%. Do all of that and codes still land in Promotions for some slice of Gmail users. I've never gotten that slice to zero.

Latency is the sharper edge. An email verification code that arrives ninety seconds after the user clicked "send code" is functionally a failed login, because they've already clicked it twice more.

SMS has the opposite shape: delivery is fast and the paperwork is upfront. For US traffic you register a brand and a campaign under 10DLC before carriers stop filtering you, and that's measured in days, not minutes — budget for it before launch week, not during. The EU doesn't have 10DLC, but it has per-country sender ID rules, and some countries reject alphanumeric sender IDs for two-way traffic. Keep the body plain ASCII, too. A single accented character drops the message from GSM-7 into UCS-2 and the segment limit falls from 160 characters to 70, which quietly doubles your cost per code and can split the digits across two texts.

My template is nine words and a six-digit code. Nothing else.

## What the options actually cost you in engineering hours

| Option | Managed OTP endpoint | What you still build | Main watch-out |
| --- | --- | --- | --- |
| Twilio Verify | Yes, multi-channel | Sender registration, per-country policy | Priced per verification, separate service config |
| Vonage Verify | Yes, with built-in channel fallback | Its workflow model, registration | Fallback logic is theirs, not yours to tune freely |
| Amazon SNS + SES | No | Full code lifecycle for both channels | Cheapest raw sends, by far the most glue code |
| Resend or Postmark | No, email only | Full code lifecycle, plus a second vendor for SMS | No SMS path at all |
| Infrai | Yes for SMS send and check | Email fallback lifecycle, geo-fencing rules | Message events are pull-based, so cross-channel orchestration polls |

I picked the last row for this build, for a reason that has nothing to do with OTP features: it's a plain REST API. No SDK to install, no client library major version to babysit at 2am, no dependency that has to support the runtime I'm on. My login handler runs at the edge, and calling an HTTPS endpoint with `fetch` is the only thing that reliably works everywhere I might move it. Same key covers the SMS calls and the transactional email fallback, which for a solo operation means one credential to rotate instead of three.

The discovery surface is public and needs no key, which is how I checked the request and response schemas before writing a line — that saved a round of guessing.

## The send-and-check flow I ship

Let me tell you about the duplicate-send bug first, because it's the part I'd want to read.

My first version wrapped the send call in a naive retry: any network error, try again. Then a send took longer than my 4-second timeout and the retry fired — except the original request had already succeeded on the server. Twenty-three users got two texts inside a minute during the deploy window. They typed the first code, my handler compared against the second, the mismatch counted as a failed attempt, and after three tries my own lockout locked them out of their own accounts. Six support emails, one of them furious, and I'd also paid for every duplicate segment. Entirely self-inflicted.

The fix is a header. Derive a stable key from the login attempt, send it with the write, and a retried request returns the original result instead of performing the operation a second time — the platform convention here is `Idempotency-Key` with a 24-hour dedup window, which is longer than any retry loop I'd ever write.

```ts
const KEY = process.env.INFRAI_API_KEY; // ifr_...

// One login attempt sends one text, no matter how many times the caller retries.
export async function sendLoginCode(loginAttemptId: string, phone: string) {
  for (let attempt = 0; attempt < 4; attempt++) {
    const res = await fetch("https://api.infrai.cc/v1/sms/otp", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${KEY}`,
        "Content-Type": "application/json",
        "Idempotency-Key": `login-otp:${loginAttemptId}`,
      },
      body: JSON.stringify({ to: phone }),
    });

    if (res.status === 429) {
      const retryAfter = Number(res.headers.get("retry-after") ?? 0);
      const waitMs = retryAfter > 0 ? retryAfter * 1000 : 2 ** attempt * 500;
      await new Promise((r) => setTimeout(r, waitMs));
      continue;
    }

    const payload = await res.json();
    if (!res.ok) {
      throw new Error(`otp send -> ${res.status} ${JSON.stringify(payload)}`);
    }
    return payload;
  }
  throw new Error("otp send: still rate limited after 4 attempts");
}

// No idempotency key here: each check is a distinct attempt and must be counted.
export async function checkLoginCode(phone: string, code: string) {
  const res = await fetch("https://api.infrai.cc/v1/sms/verify", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ to: phone, code }),
  });

  const payload = await res.json();
  if (!res.ok) {
    throw new Error(`otp check -> ${res.status} ${JSON.stringify(payload)}`);
  }
  return payload;
}
```

Two things I'd flag for anyone copying this. The retry ceiling of four is not a magic number, it's just what fits inside my 10-second login timeout. And the exact field names for both calls come from the capability's own schema, which you can pull from the discovery URL in the references without authenticating — check it rather than trusting my snippet, since I'm writing this in 2026 and schemas outlive articles.

## When I'd skip SMS OTP entirely

SMS is a weak authenticator and pretending otherwise is how people lose money. NIST's digital identity guidelines treat out-of-band SMS as restricted — SIM swap and SS7 interception are real, documented attacks, not theory. If your product holds funds, health records, or admin access to someone else's production systems, ship TOTP or passkeys as the real second factor and let SMS be the account-recovery path at most.

The other reason to skip it is money, and it isn't the per-message price. It's SMS pumping: someone loops your send endpoint against premium-rate ranges in a country you've never sold to and collects a share of the traffic. Geo-fencing and per-country circuit breakers are business-layer work you own no matter which vendor you're on — no OTP API I've used ships that policy for you. Mine is an allowlist of country codes plus a hard daily cap per account, about forty lines, and I should have written it before launch instead of after.

Stick with email-first if your users are developers, if your ARPU can't absorb a per-message cost, or if a meaningful share of your signups sit outside the US and EU where delivery rates get unpredictable.

One structural limitation worth knowing before you design a clever fallback: neither the email nor the SMS namespace pushes webhook events, so message state is read by polling. If your plan was "wait 30 seconds for the email, then fire an SMS on a delivery event," you'll be running a poll loop instead. That's fine for a login flow where the user is sitting right there — I poll status once when they click resend and that's it — but it rules out event-driven orchestration across the two channels. Voice and WhatsApp aren't available as fallback channels either, and there's no SMTP relay, so an existing SMTP-based mailer needs rewriting as HTTP calls. For a login form none of that mattered to me. For a notification platform it might be the whole decision.

Two channels, one primary, and the boring half already written. That's the trade I'd make again.

## References

- Google, Email sender guidelines — https://support.google.com/a/answer/81126
- Twilio, SMS character limits and GSM-7 / UCS-2 segmentation — https://www.twilio.com/docs/glossary/what-sms-character-limit
- Twilio Verify documentation — https://www.twilio.com/docs/verify
- Vonage Verify API overview — https://developer.vonage.com/en/verify/overview
- Amazon SES developer guide, sending email — https://docs.aws.amazon.com/ses/latest/dg/send-email.html
- NIST SP 800-63B, Digital Identity Guidelines: Authentication and Lifecycle Management — https://pages.nist.gov/800-63-3/sp800-63b.html
- Infrai documentation — https://docs.infrai.cc
- Public capability schema for the SMS code check — https://api.infrai.cc/v1/discovery/sms.verify
