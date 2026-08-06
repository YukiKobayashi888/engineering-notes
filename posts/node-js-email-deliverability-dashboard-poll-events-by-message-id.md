# Node.js Email Deliverability Dashboard: Poll Events by Message ID

**Store every transactional email under its message ID, poll only unfinished messages, and show sent, delivered, and bounced as operational states rather than proof that a person read the email.**

That is the smallest dashboard I can justify in a one-person SaaS. I ship weekly, so I want support evidence without adopting a second product inside my product. My working version has one message table, an append-only event table, a provider adapter, and a scheduled reconciliation job. The adapter owns vendor-specific details; the dashboard never does.

Keep it dull.

## What should a Node.js transactional email deliverability dashboard track by message ID?

The join key is the provider message ID returned when I send. I save it beside my own immutable ID, tenant ID, recipient, template name, creation time, and current normalized state. The provider ID answers a transport question. My ID answers a product question. Keeping both means a support search can start from an account or invoice and still reach the transport history without making an email address the primary key.

I use a short state vocabulary: `pending`, `sent`, `delivered`, `bounced`, and `unknown`. Those labels need careful copy. `sent` means the sending system accepted the job. `delivered` means the receiving mail system accepted it. Neither means the recipient saw, opened, or trusted the message. A bounce is a delivery outcome, but its original response still belongs in an append-only event row because a normalized badge can't preserve enough evidence for later debugging.

The dashboard itself gets four views: unfinished messages, recent bounces, a message timeline, and daily counts by final state. I don't put opens in the primary health score. The customer question is usually "where is my login email?", and a delivery timeline is more useful than a decorative engagement percentage.

Authentication health sits next to event health, not inside it. Google's sender guidelines call for SPF or DKIM for all senders to personal Gmail accounts, and SPF, DKIM, and DMARC for bulk senders. They also cover TLS, valid forward and reverse DNS, message formatting, and subscription-message handling. Those are domain-level controls; polling a message ID cannot diagnose them alone. I review them separately after DNS or sending-domain changes.

That distinction keeps the screen honest: events describe individual attempts, while sender configuration describes the conditions around those attempts.

## Why poll unfinished sent, delivered, and bounced events instead of every email?

Polling is reconciliation. The worker selects only rows whose state is unfinished, whose next check is due, and whose age is still inside the retention policy I chose for my own database. Each run asks the provider adapter for events by message ID, inserts unseen events idempotently, advances the summary state, and schedules another check only when the message remains unfinished.

Callbacks can reduce latency when a sending service offers them, but I don't let callback delivery determine whether the dashboard can repair itself. The same event ingestion function handles callback-shaped input and polling results. That gives me one normalization boundary and one idempotency rule. It also lets me change the transport without rewriting the dashboard — exactly the kind of undifferentiated work I prefer to outsource.

I learned the selection rule the expensive way. One early job scanned 1,237 messages, launched 16 requests at once, and met a `429` halfway through. A retry loop quietly swallowed the final rejection and returned an empty array, so the run looked green while several rows stayed `sent`. I found it from a customer ticket, not an alert. Now the adapter treats rate limiting as a rescheduling signal, honors a server-provided delay when available, and throws after its retry budget. The job records failure. It never turns missing data into an empty event history.

I'm not sure why I once considered a successful process exit meaningful when the work was incomplete. Revenue per hour makes the answer obvious: a loud failed job takes minutes to inspect; a falsely reassuring dashboard consumes a support thread and customer trust.

The interval is workload-specific, so your mileage may vary. I start with a modest delay, apply bounded backoff with jitter, cap work per run, and measure unfinished-row age. If that age rises, I know reconciliation is losing ground before a user tells me.

## The smallest working TypeScript implementation

The provider boundary below is deliberate. `fetchEvents` is the only function that knows how a chosen email service represents a lookup by message ID. I keep its concrete URL, authorization header, response schema, and rate-limit policy in that adapter, based on the service's current documentation. The rest is ordinary Node.js code and can be tested with an in-memory store.

```ts
type DeliveryStatus = "pending" | "sent" | "delivered" | "bounced" | "unknown";

type DeliveryEvent = {
  id: string;
  messageId: string;
  status: DeliveryStatus;
  occurredAt: string;
  raw: unknown;
};

type Message = {
  messageId: string;
  status: DeliveryStatus;
  nextPollAt: string | null;
};

interface EventSource {
  fetchEvents(messageId: string): Promise<DeliveryEvent[]>;
}

interface MessageStore {
  due(limit: number): Promise<Message[]>;
  insertEventOnce(event: DeliveryEvent): Promise<boolean>;
  setStatus(messageId: string, status: DeliveryStatus): Promise<void>;
  schedule(messageId: string, nextPollAt: string | null): Promise<void>;
}

const FINAL = new Set<DeliveryStatus>(["delivered", "bounced"]);

function latestStatus(events: DeliveryEvent[]): DeliveryStatus {
  const ordered = [...events].sort((a, b) =>
    Date.parse(a.occurredAt) - Date.parse(b.occurredAt)
  );
  return ordered.at(-1)?.status ?? "unknown";
}

export async function reconcile(
  source: EventSource,
  store: MessageStore,
  now = new Date(),
): Promise<void> {
  const messages = await store.due(50);

  for (const message of messages) {
    const events = await source.fetchEvents(message.messageId);
    for (const event of events) await store.insertEventOnce(event);

    const status = latestStatus(events);
    if (status !== "unknown") await store.setStatus(message.messageId, status);

    const nextPollAt = FINAL.has(status)
      ? null
      : new Date(now.getTime() + 5 * 60_000).toISOString();
    await store.schedule(message.messageId, nextPollAt);
  }
}
```

`insertEventOnce` should enforce uniqueness on the provider event ID, or on a documented stable composite when no event ID exists. Run the worker with low concurrency first. Alert on thrown runs, oldest unfinished age, and repeated polling attempts. I also test out-of-order input, duplicate input, an empty response, and a rate-limit rejection; those cases matter more than the happy path.

One caveat: a provider may expose only a current status rather than event history. In that case I synthesize no history. I store each observed status with the observation time and label it as an observation, because inventing an event timestamp makes the timeline look more precise than it is.

## What changes when this simple SaaS example reaches scale?

My first change would be a queue-backed poll scheduler, not a larger dashboard. A queue gives each message its own due time, attempt count, and retry policy. I would partition work by tenant, enforce a global request budget inside the adapter, and add a dead-letter path that creates an operator task. The read model can remain the same.

The trade-offs are fairly plain:

| Approach | Good fit | Main cost |
| --- | --- | --- |
| Provider dashboard only | Low volume and rare support cases | No product-level join from a user or invoice to a message |
| Scheduled database poller | A small SaaS with modest unfinished volume | Repeated reads and delayed updates |
| Callback ingestion plus polling | Faster status changes with repair coverage | Signature validation and two ingestion paths |
| Queue-backed reconciliation | High volume or strict freshness targets | More infrastructure and operational ownership |

The catch is that a custom dashboard isn't suitable when nobody acts on its data. Stick with the sending service's own console when support can resolve the occasional case there. Use callback ingestion without frequent polling when latency matters and the callback path is already well operated. Move to a queue when database scans or shared request budgets become measurable constraints. I won't add that machinery merely because it looks mature; every extra worker competes with the feature I meant to ship Friday.

SMS also needs a separate cost and payload view even if it shares the same status vocabulary. Twilio's explanation of SMS segmentation says a GSM-7 message allows 160 characters in one segment and 153 in each segment when concatenated. UCS-2 allows 70 in one segment and 67 when concatenated. A single non-GSM character can change the encoding, so the dashboard should retain encoding and segment count when the transport reports them. Same timeline. Different economics.

## References

- Google, Email sender guidelines: https://support.google.com/a/answer/81126
- Twilio, SMS character limits and segmentation: https://www.twilio.com/docs/glossary/what-sms-character-limit
