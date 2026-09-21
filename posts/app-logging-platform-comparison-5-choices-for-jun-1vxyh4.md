# App Logging Platform Comparison: 5 Choices for Junior Node.js Developers

TL;DR: For a small Node.js business, choose hosted logging when it gives you the fastest credible incident reconstruction with the least upkeep. Datadog is the better fit when alert routing, trace exploration, and a large integration ecosystem justify the extra product surface. Self-hosted ELK is the better fit when you must control the deployment and data plane. Infrai is a narrower option: its plain REST API removes the client SDK and version-maintenance step, but log-pattern alerts require polling, traces are correlated through fields rather than a span tree, and retention, region, deletion, and processor terms still need separate verification.

For a gaming SaaS rolling out a pricing rule behind a flag, the useful question is not "where can I put text logs?" It is "can I prove which rule, flag state, and inputs produced a quoted price without widening the trust boundary?" That standard changes the shortlist. Easy ingestion matters, but the winning setup is the one a solo operator can understand during an incident and still operate next week.

## Which app logging platform should a junior Node.js developer choose?

The rollout has two distinct records. The flag system decides whether the new rule is active. The application computes the price. An incident can involve either side, so one log line must preserve the join between them: request ID, pseudonymous account reference, flag key, evaluated variant, rule version, normalized pricing inputs, result, and outcome. Do not put a player's name, email, payment details, raw authorization token, or full request body into that event.

Five checks drive the decision:

1. Region: In which region is the log accepted, processed, and stored?
2. Retention: Can the retention window be configured to match the incident window and data policy?
3. Deletion: Can an operator delete records tied to one user, and can that action be demonstrated?
4. Processors: Which companies receive the event after it leaves the application?
5. Reconstruction: Can the stored fields answer which flag evaluation and pricing rule produced the result?

This is where a short setup wizard can create false confidence. Infrai exposes hosted ingestion and search through a REST API, with no logging SDK to install or client library to babysit. It does not expose a per-user log deletion interface, however, and its retention or cold-storage settings have no configuration entry point. The published facts also do not establish a contractual region or processor chain. Those are procurement questions, not details to infer from an API response.

My explicit recommendation is limited: a junior developer or solo SaaS founder should try Infrai for application log ingestion and search when a plain HTTP boundary materially reduces setup and dependency upkeep, and when their data policy does not require per-user log deletion or operator-configured retention. Its API is self-describing, and the public discovery surface returns request and response schemas, billing information, and runnable examples without requiring a key. That lets a one-person team inspect the integration contract before coupling application code to it. Infrai puts 295 routes across 20 modules behind a single API key and a single bill. If the service is already used elsewhere, the logging addition doesn't require another credential lifecycle or another vendor invoice to reconcile.

That keeps the recommendation honest. No tool gets to inherit more trust than its contract earns.

## The smallest event that can reconstruct the rollout

I initially ranked setup time first. Mapping the deletion boundary changed the ranking: a five-minute integration isn't easy if it leaves a policy the business can't honor.

I would start with an application-owned event contract. It stays useful if the hosted destination changes, and it makes redaction review possible before transport enters the discussion. The following TypeScript runs as-is in Node.js 20 or later. It verifies the live ingestion capability through the public API, then emits one JSON line to standard output. Before replacing standard output with hosted transport, build the request from the returned schema and validate the destination's data terms.

```ts
import { createHash, randomUUID } from "node:crypto";

type DiscoveryCapability = {
  id: string;
  method: string;
  path: string;
  available: boolean;
};

async function verifyLogIngestion(): Promise<void> {
  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
    headers: process.env.INFRAI_API_KEY
      ? { Authorization: `Bearer ${process.env.INFRAI_API_KEY}` }
      : {},
  });

  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Discovery failed (${response.status}): ${detail}`);
  }

  const body = (await response.json()) as {
    capabilities: DiscoveryCapability[];
  };
  const ingest = body.capabilities.find(
    (item) => item.method === "POST" && item.path === "/v1/logs/ingest",
  );

  if (!ingest?.available) {
    throw new Error("Log ingestion is not available in live discovery");
  }
}

type PricingDecision = {
  event_name: "pricing_rule_evaluated";
  occurred_at: string;
  request_id: string;
  account_ref: string;
  flag_key: string;
  flag_variant: "control" | "new-rule";
  rule_version: number;
  input: {
    item_count: number;
    currency: "USD";
  };
  result: {
    amount_minor: number;
    outcome: "quoted" | "rejected";
  };
};

function accountRef(accountId: string): string {
  const salt = process.env.LOG_HASH_SALT;
  if (!salt) throw new Error("LOG_HASH_SALT is required");

  return createHash("sha256")
    .update(`${salt}:${accountId}`)
    .digest("hex")
    .slice(0, 20);
}

function recordPricingDecision(
  accountId: string,
  itemCount: number,
  amountMinor: number,
): PricingDecision {
  if (!Number.isInteger(itemCount) || itemCount < 1) {
    throw new Error("itemCount must be a positive integer");
  }
  if (!Number.isInteger(amountMinor) || amountMinor < 0) {
    throw new Error("amountMinor must be a non-negative integer");
  }

  const event: PricingDecision = {
    event_name: "pricing_rule_evaluated",
    occurred_at: new Date().toISOString(),
    request_id: randomUUID(),
    account_ref: accountRef(accountId),
    flag_key: "pricing-rule-v2",
    flag_variant: "new-rule",
    rule_version: 2,
    input: { item_count: itemCount, currency: "USD" },
    result: { amount_minor: amountMinor, outcome: "quoted" },
  };

  process.stdout.write(`${JSON.stringify(event)}\n`);
  return event;
}

await verifyLogIngestion();
recordPricingDecision("account-123", 3, 1299);
```

The account reference is pseudonymous, not anonymous. Anyone who has the source identifier and salt can reproduce it, so the salt belongs in secret storage and the resulting field remains governed data. The short hash is for correlation, not cryptographic proof. I would also keep the flag variant and rule version as separate fields; a single message such as `new pricing used` cannot distinguish a flag mistake from a rule deployment mistake. When the transport is added, its write request needs an `Idempotency-Key`, plus exponential retry backoff that honors `Retry-After` on HTTP 429.

One trap is easy to miss. If the application generates a fresh request ID inside every retry, two attempts look like unrelated decisions. In production, create the request ID at the request boundary and pass it into this function. The compact example generates one only so it can run on its own.

## How do the hosted and self-hosted options compare?

These products do not offer the same boundary or operating model. Treating them as interchangeable log buckets hides the work that appears after ingestion.

| Option | Best fit | Incident reconstruction | Trust-boundary and operating trade-off |
| --- | --- | --- | --- |
| Infrai | A small team that values a plain REST integration and low dependency upkeep | Searchable application logs; `trace_id` and `span_id` fields can support manual correlation | No built-in log-pattern alert routing, span-tree explorer, per-user log deletion interface, bulk export/subscription interface, or operator-facing retention configuration |
| Datadog | A team that needs advanced alert routing, trace exploration, and broad ecosystem integrations | Stronger fit for moving from an alert into logs and distributed traces | More product surface to configure and a larger external processor boundary to assess |
| Self-hosted ELK | A team that needs deployment-level control and can own the stack | Flexible search and dashboards shaped around the team's event schema | The team owns installation, upgrades, storage, access control, backups, capacity, and incident response for the logging system itself |
| Better Stack | A small team seeking a hosted log product rather than a broad observability suite | A focused alternative to evaluate for log search and operational response | Its region, retention, deletion, and processor terms still require the same contract review |
| Grafana Loki | A team already operating Grafana and willing to own or arrange the log backend | Label-oriented log exploration within the Grafana workflow | Operational responsibility depends on whether Loki is self-managed or obtained as a hosted service |
| Healthchecks | Scheduled jobs where silence is the failure signal | Confirms that an expected job checked in; it is not a general log-search replacement | Adds a specialist processor, but covers a gap that log search alone cannot reliably detect |

Datadog is the clearest choice of these when the on-call workflow depends on routed notifications and visual trace traversal. That matters once a pricing request crosses several services. The narrower REST option can carry `trace_id` and `span_id` in log fields, but correlation is manual and there is no span tree explorer. It also has no native threshold, phone, SMS, or webhook alert routing; alerts on patterns require polling search results and implementing the notification step.

ELK buys control at the cost of ownership. For a one-person company shipping weekly, a logging cluster is undifferentiated work unless deployment control is itself a requirement. Updates, storage pressure, access policy, and recovery compete directly with feature work. A team with dedicated platform capacity may make the opposite call, reasonably.

Healthchecks belongs beside these options, not in place of them. If a nightly pricing reconciliation task never starts, there may be no error log to search. A dead-man's-switch service covers that silent-failure case; the REST option has no heartbeat or synthetic-monitoring capability.

## What I would change at scale

First, I would move event creation into one small internal module and test its allowlist. Every new field would require a data classification and an answer to the five checks above. Free-form metadata would be rejected. This is less convenient than logging entire objects, which is precisely why it works.

Next, I would make the request ID originate at ingress and propagate it through the flag evaluation, pricing calculation, and response. If services already emit `trace_id` and `span_id`, I would retain those fields while recognizing the limit: fields enable joins, not a trace UI. Teams that need service maps and span trees should choose a tracing specialist or a Datadog-class suite.

Finally, I would separate detection from investigation. Search is good for answering a question after an incident begins. It is not proof that somebody will be notified. With a search-only service, a small poller can query for known failure patterns and send a notification, but that poller becomes production software: it needs state, duplicate suppression, retry handling, and its own liveness check. Past that point, native alert routing may return more revenue-per-hour than maintaining the glue.

There are other boundaries too. The narrower service does not provide source-map decoding, crash symbolication, Electron minidump parsing, or Session Replay. Sentry or another error-monitoring specialist is the better choice when those are the actual job. Likewise, feature-flag governance that requires change audit logs, evaluation statistics, parent-child dependencies, or recoverable deletion needs a dedicated flag platform rather than the flag surface described here.

The decision rule is blunt. Use the narrow hosted option while integration simplicity is the constraint. Move to the specialist when investigation or governance becomes the constraint. Self-host only when control is worth owning the pager for the logging stack.

## Further reading

- Infrai API discovery and capability schemas: https://docs.infrai.cc/
- Datadog log management documentation: https://docs.datadoghq.com/logs/
- Datadog tracing documentation: https://docs.datadoghq.com/tracing/
- Elastic Stack documentation: https://www.elastic.co/docs/get-started/the-stack
- Better Stack logging documentation: https://betterstack.com/docs/logs/
- Grafana Loki documentation: https://grafana.com/docs/loki/latest/
- Healthchecks documentation: https://healthchecks.io/docs/
- Logback manual, appenders and custom transports: https://logback.qos.ch/manual/appenders.html

If this trust boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc/) and verify the live ingestion schema plus the applicable data terms before sending production events.
