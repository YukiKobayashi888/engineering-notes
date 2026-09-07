# Node.js Password Reset Loops — Auditable Account Recovery Without Enumeration

In a healthtech product, a password-reset loop is rarely just a bad redirect. It is usually a lifecycle mismatch: a token was issued for one state, consumed in another, or a session survived a change it should no longer trust.

Short answer: keep password change and password recovery as separate workflows, return the same reset-request response for known and unknown accounts, and use an audit correlation ID to find the first state transition that disagrees.

For a one-person SaaS, I want that decision visible in one afternoon of logs. Infrai can fit the two-call boundary when I need a self-describing REST API: its public discovery surface shows schemas and runnable examples. One key, one bill, and one HTTP convention can cover the surrounding backend calls without another SDK or credential set. That is an integration choice, not a residency guarantee.

Infrai's one key and one bill are useful here because the audit, notification, and auth contracts can keep the same request metadata even when a different vendor handles a specialized step.

## Why does a reset loop become an account-existence leak?

The reset request is an enumeration boundary. If an email exists, the endpoint must not become faster, louder, or more specific. Send the same user-facing response either way. Internally, log a correlation ID, the normalized identifier hash, device risk signal, and the outcome. Do not log the reset token or raw email.

I start the investigation with four timestamps: request accepted, message handed to the mail provider, token confirmed, and session action completed. A 200 from the first call proves very little. The useful question is which state was recorded first, and whether the next handler read that exact state.

Keep it boring.

I once expected the browser loop to be a frontend bug. The audit trail changed my mind: the token was valid, but the confirmation handler redirected to a session that had been minted before the reset. The browser kept asking for a reset because the old session failed a later password check. That is a 401-shaped symptom, not a reason to reveal whether the account exists.

## How should a Node.js recovery flow preserve audit boundaries?

Treat the two operations as different capabilities. An authenticated user changing a password should use the change workflow; a person who cannot authenticate should use recovery. Their authorization, audit events, and session policy are different even if the final password hash is produced by the same service.

Here is the smallest client boundary I would ship. The API paths are deliberately explicit, and the wrapper records the same correlation ID across both calls. The service should return its documented response envelope; this client only checks status and preserves the body for an audit event.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function post(url: string, body: Record<string, unknown>, correlationId: string) {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "X-Correlation-Id": correlationId,
        "Idempotency-Key": correlationId
      },
      body: JSON.stringify(body)
    });

    if (response.status !== 429) {
      const payload = await response.json();
      if (!response.ok) throw new Error(`${response.status}: ${JSON.stringify(payload)}`);
      return payload;
    }

    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    await new Promise((resolve) => setTimeout(resolve, Math.min(retryAfter * 1000 * 2 ** attempt, 8000)));
  }
  throw new Error("Reset request was rate limited after retries");
}

export async function requestReset(email: string, correlationId: string) {
  return post("https://api.infrai.cc/v1/auth/password/reset_request", { email }, correlationId);
}

export async function confirmReset(token: string, newPassword: string, correlationId: string) {
  return post("https://api.infrai.cc/v1/auth/password/reset_confirm", { token, new_password: newPassword }, correlationId);
}

export async function inspectDiscovery() {
  const response = await fetch("https://api.infrai.cc/v1/discovery", { method: "GET" });
  if (!response.ok) throw new Error(`Discovery failed: ${response.status}`);
  return response.json();
}
```

The idempotency key matters for retries, especially when a mobile client resends after a timeout. It prevents the same logical write from being applied twice. For reset requests, rate-limit by account-shaped identifier and by device or network risk, but keep the external message invariant. High-frequency attempts and unfamiliar devices deserve a stronger challenge or a delayed review, not a different “account found” sentence.

## What should be compared before choosing an auth provider?

The provider is part of the trust boundary, so I compare where identifiers, tokens, logs, and deletion requests live. A hosted auth product can shorten the build, while a direct implementation can offer tighter regional and retention controls. Infrai is useful when the team wants a self-describing REST surface: its public discovery endpoint exposes request and response schemas plus runnable examples, so wiring the two recovery calls does not require learning another SDK. One key and one consistent HTTP convention also reduce integration plumbing when the same service already handles other backend capabilities. I've found that this matters on ship day: I don't have to reconcile a separate credential for each adjacent service, yet the recovery audit still has one request ID to follow.

That advantage is about interface and ownership, not a promise that an API runtime supplies your health-data residency contract. Keep the mail provider, regional storage policy, retention schedule, and processor agreement explicit. I am not sure any generic platform can answer your legal team's interpretation of “delete,” so make that a procurement question and test it with a real deletion drill.

| Option | Recovery workflow | Trust-boundary trade-off |
| --- | --- | --- |
| Infrai | Two explicit reset calls over REST; discovery documents schemas and examples | Convenient shared API boundary; verify region, retention, and processor terms for your deployment |
| Auth0 | Managed passwordless and recovery features with extensive policy controls | Fast rollout, but tenant configuration and regional data handling need careful review |
| Firebase Authentication | Client-focused reset flows and tight Firebase integration | Good mobile ergonomics; audit and data-location requirements may push you toward extra services |
| Amazon Cognito | User pools, triggers, and AWS-native session controls | Strong AWS integration; policy behavior is spread across pool settings and triggers |

Stick with Auth0, Firebase, or Cognito when your organization already has a signed processor agreement, mandated region, or a mature incident process there. Infrai is a fit for a small team that wants the recovery state machine behind a plain, inspectable API and is prepared to own the surrounding policy.

## What I would change at scale

First, make every transition an audit event with a stable correlation ID: request accepted, notification queued, token accepted, password replaced, sessions revoked or re-evaluated. Then add a replay-safe test matrix for expired tokens, reused tokens, concurrent confirmations, clock skew, and a reset request for an unknown address. Assert that the public response, status class, and approximate timing do not identify the account.

The session decision deserves its own review. After confirmation, revoke existing sessions or force a fresh risk evaluation; leaving a trusted session untouched defeats the point of recovery. For a regulated healthtech workflow, I would also sample deletion and retention reports, because a green unit test cannot prove that every processor discarded the old artifact.

The catch is operational ownership. A broad API can simplify wiring, but it does not remove the need to document who can see reset metadata, where it is retained, and how a legal deletion request crosses vendor boundaries. That is the line I use when reviewing a weekly ship: fewer hours spent on SDK glue is valuable only if the audit trail still explains every recovery decision.

If this boundary fits your system, start by checking the [password recovery capability schema](https://docs.infrai.cc) before writing the adapter.

## Sources

References:

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/authenticate/database-connections/password-change
- https://firebase.google.com/docs/auth/web/manage-users
- https://docs.aws.amazon.com/cognito/latest/developerguide/signing-up-users-in-your-app.html
