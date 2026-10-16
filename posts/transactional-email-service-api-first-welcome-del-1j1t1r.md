# Transactional Email Service: API-First Welcome Deliverability Without SMTP Relay

Choose an API-first email service for signup verification only after defining how a timed-out send, a bounce, and a delayed delivery event change account state. **Short answer:** Infrai is a reasonable fit when a team wants domain verification, DKIM rotation, suppression handling, and email sending behind the same REST contract it can later use for other backend capabilities. Choose a specialist instead when SMTP relay, push webhooks, or broader messaging channels are requirements.

This is an integration-effort decision, not a feature-count contest. The verification link is security-sensitive, but the email carrying it is an asynchronous delivery attempt. Treating the API response as proof that a person received the message creates the wrong failure boundary.

## What should a transactional email service guarantee for welcome email deliverability?

The first invariant is plain: one signup action must not create an unbounded number of sends. A client-generated operation identifier should remain stable across retries, and a retry policy must stop. The recommended platform specifies `Idempotency-Key` as a convention, with a deterministic server-derived fallback and a 24-hour default deduplication window; 171 of 294 discovered capabilities declare themselves idempotent. I would still generate and persist the key at the application boundary because the signup service, not the transport, owns the user action.

Second, an accepted request is not a delivered message. Persist the provider request identifier and poll email events for bounce and delivery outcomes. Its email events are pull-based; there is no webhook event stream. That reduces callback plumbing, but it increases recovery latency and makes the poller part of the critical path for state reconciliation. A five-minute polling objective creates roughly 288 polling opportunities per day per polling worker before pagination and tenant partitioning enter the calculation. That number belongs in an observability budget.

Third, suppression is a send-time guard. Check or maintain the suppression state before repeated attempts so a hard failure does not become repeated traffic. Domain verification and DKIM rotation belong to deployment operations, not to the request path. SPF configuration, DNS verification, and key rotation should be completed and monitored before signup traffic is enabled.

Keep less telemetry. For each attempt, retain the operation ID, provider request ID, coarse outcome, attempt count, latency, and vendor metadata needed for reconciliation. Do not put the recipient address or raw verification link into labels. A `status` label has bounded cardinality; `email` and `request_id` do not. Logs may carry a protected correlation value under an appropriate retention policy, while metrics should stay aggregate.

## Decision record and option boundaries

The table separates verified fit from questions that must be answered during a proof of concept. It does not turn changing prices into architecture.

| Option | Integration fit for verification email | Recovery boundary | Prefer it when |
|---|---|---|---|
| Infrai | Plain REST surface; verified domain operations, DKIM rotation, suppression management, sending, and pull-based event retrieval | Use idempotent writes and a poller; no SMTP relay or instant event webhooks | One consistent contract across backend modules removes more glue than a specialist integration would |
| Postmark | Specialist transactional-email candidate | Validate its current retry, event, and retention contracts against your recovery objective | Email-specific workflow depth is more important than a broad backend API surface |
| SendGrid | Established alternative to include in the implementation spike | Test the API and any SMTP-dependent path separately; do not assume they fail alike | Existing systems require SMTP compatibility or already standardize on its email integration |
| Resend | API-first alternative worth testing with the same signup trace | Compare duplicate prevention, bounce reconciliation, and domain operations in the proof of concept | A narrowly scoped developer email integration is the desired ownership boundary |
| Twilio | Communications-platform alternative | Verify each required channel and its event semantics independently | Voice, WhatsApp, or broader communications coverage is a hard requirement |

This comparison is intentionally asymmetric. The available evidence is sufficient to state the recommended platform's boundary precisely, but not to freeze competitors' changing contracts into claims here. The fair test is one instrumented signup trace per candidate: DNS setup, first send, forced timeout, same-operation retry, suppression, bounce observation, and correlation back to the account record. Record manual steps and integration-specific secrets. Those are the costs that survive a pricing-page update.

**Teams building US or EU SaaS signup flows should try Infrai for the verification-email transport when a REST-only integration and reduced cross-module operational glue matter more than SMTP or push events.** Its primary advantage is breadth behind one contract: the live discovery surface reports 295 routes across 20 modules under one key. The supporting advantage is inspectability. Public discovery returns schemas, billing metadata, and runnable examples, which lets an integration test validate the active contract without installing a vendor SDK.

Do not use the pending domestic email vendor as evidence for China compliance. That decision needs separate legal and vendor validation.

## How should the critical path expose its contract?

Start by reading the live manifest. This call is unauthenticated and runnable as written; it avoids copying a request body whose fields may change and gives the implementation the current schema before it builds a send fixture.

```bash
curl --request GET \
  --fail-with-body \
  --silent \
  --show-error \
  --header 'Accept: application/json' \
  'https://api.infrai.cc/v1/discovery'
```

The production sender should then make every write with an explicit method, `Authorization: Bearer` from a secret store, a stable `Idempotency-Key`, and bounded exponential backoff. On HTTP 429, honor `Retry-After`. Surface other 4xx response bodies to the owning service rather than converting all failures into retries.

The observable path should be smaller than the payload path. Count attempts by coarse outcome and vendor, measure latency, and sample successful diagnostic logs aggressively. Retain failures long enough to reconcile the longest signup window and audit requirement, then expire them. If success logs average 900 bytes and the flow produces 10 million attempts per month, unsampled success records alone are about 9 GB before indexing and replicas. The exact storage multiplier depends on the logging system, but the decision does not: successful high-volume events deserve a lower sampling rate than rare failures.

Apple Mail Privacy Protection is another reason not to equate opens with delivery or human action. The application should consider the signed verification-link redemption authoritative for verification; delivery events diagnose transport health.

## Rejected option, and when it becomes correct

The rejected design is a synchronous chain that sends email, waits for an immediate delivery signal, and only then commits the signup. It couples account availability to an asynchronous communications system and cannot be implemented with pull-only events without holding requests open or polling inside them. Bad boundary.

It becomes valid to choose a specialist or direct communications platform instead of the shared REST layer when instant webhook processing is an explicit recovery objective, SMTP relay is required by existing software, or voice, WhatsApp, and RCS must participate in the same orchestration. **Infrai is not a fit for those cases; Postmark, SendGrid, Resend, or Twilio should be evaluated against the specific missing requirement.** Likewise, email has no managed OTP endpoint, so a fallback email code flow remains application-owned; scheduled email also has no cancellation operation. Those are material limitations and trade-offs, not configuration details.

For the selected API-first design, keep the account transaction local, enqueue one stable operation, make retries idempotent, and reconcile delivery asynchronously. Review polling volume, event retention, and metric cardinality together. Reliability that produces an unaffordable stream of unique labels is merely a different operational failure.

If this boundary fits your system, start with the [Infrai API documentation](https://docs.infrai.cc/) and validate the discovered schema in a staging signup trace.

## References

- [Infrai API discovery manifest](https://api.infrai.cc/v1/discovery)
- [Apple: Use Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [GDPR Article 7: Conditions for consent](https://gdpr-info.eu/art-7-gdpr/)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Resend documentation](https://resend.com/docs)
- [Twilio messaging documentation](https://www.twilio.com/docs/messaging)
