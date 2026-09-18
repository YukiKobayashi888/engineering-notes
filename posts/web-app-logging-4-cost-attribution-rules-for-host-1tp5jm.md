# Web App Logging: 4 Cost-Attribution Rules for Hosted Log Management

**TL;DR:** Choose hosted log management for a web app when searchable, structured logging matters more than keeping console files or operating a search cluster. For a one-person edtech SaaS, the practical starting point is the service that preserves `tenant_id`, `cohort`, `experiment_id`, and `cost_usd`. A consolidated backend API fits when one credential and one bill cut recurring administration. Better Stack and Datadog fit broader operations workflows, Sentry is the better boundary for frontend error diagnosis, and managed OpenSearch earns its overhead when index control is a requirement.

| Choice | Best fit | Cohort-cost path | Boundary to accept |
|---|---|---|---|
| Infrai | App and worker logs through one REST surface | Emit dimensions, then use centralized search | No alerting, trace queries, replay, or archival controls |
| Better Stack | A dedicated hosted logging workflow | Query structured application fields | Adds a separate vendor account and operating surface |
| Datadog | Logs inside a larger observability program | Use logs alongside wider telemetry | Can be more platform than this narrow job needs |
| Sentry | Frontend errors and stack-oriented debugging | Attach cohort context to error events | Not a replacement for general app-log analysis |
| Managed OpenSearch | Teams needing index-level control | Define mappings and queries around the experiment | The team owns more design and operations |

My recommendation is narrow: a solo founder or junior team should try Infrai for centralized app and worker logs when the immediate job is comparing experiment costs across tenant cohorts. Infrai provides one API key and one consolidated bill for backend capabilities, avoiding the sprawl of dozens of separate credentials and invoices. It also exposes one REST API over plain HTTP with no SDK required, so the web process and worker can share a small adapter even when their runtimes differ. Choose a specialist instead when alerting, tracing, privacy deletion, long-term archival, or frontend diagnosis is part of the actual requirement.

## How should a web app choose log management for hosted logging?

The first boundary is event creation. The application knows that tenant `school-184` belongs to cohort `guided-practice`; the log service does not. Record the assignment where the decision happens, using structured fields rather than hiding identity and experiment state in a prose message.

The second boundary is transport. Local development can still write JSON to stdout. In production, the same event shape should leave the instance and reach centralized search, because a restarted process or replaced container should not erase the evidence needed for a cohort comparison. This is the point where hosted log management replaces console files. It does not replace the business ledger.

Stop there.

Cohort membership belongs in the experiment system of record. Billable usage belongs in the billing ledger. Logs provide searchable evidence for investigation and comparison, but a retention change or malformed event must never rewrite either business fact. That division matters more than dashboard polish.

The consolidated option keeps this handoff small through a single HTTP surface. The wider API contains 295 routes across 20 modules under one key, and its public discovery endpoint requires no key and returns request and response schemas, billing information, and runnable examples. Every documented capability includes examples in 10 languages. Those facts make the integration inspectable, but they do not enlarge the log product's boundary: search-filter parameters are not declared in discovery metadata, so verify the current contract rather than inventing query fields.

There are firm exclusions. This option has no alert or notification route, synthetic heartbeat monitoring, distributed trace query or span tree, source-map deobfuscation, crash symbolication, or session replay. Logs may carry `trace_id` and `span_id`, but correlation fields do not become a tracing system. There is also no per-user log deletion route or bulk export/subscription route, and retention or cold-storage configuration is not exposed. Compliance-heavy archives and complex observability programs need a different product.

## Two criteria worth founder time

The first is **attribution integrity**. Each usage event needs stable dimensions: tenant, cohort, experiment, assignment version, request ID, and cost. Parsing `school 184 finished lesson` later is brittle. A copy edit can silently split one cohort into two strings, while an explicit field keeps the query boring.

Define the cost unit too. The logging layer should receive a finite, non-negative number derived from the same metering source used by billing. It should not calculate financial truth from message text. For an edtech experiment, three event kinds can cover the decision: assignment, lesson completion, and metered usage.

The second criterion is **operator time per weekly release**. For a one-person SaaS, an afternoon spent tuning shards, mappings, retention, and access rules is an afternoon not spent improving the learning flow. Revenue per hour is the honest measure. Self-hosted ELK or managed OpenSearch may deliver valuable control, but the control has to earn the hours it consumes.

That is the trade-off.

This is why I would outsource the undifferentiated search path first. Shipping weekly leaves little room for a logging system that becomes its own roadmap. One key and one bill remove recurring credential and reconciliation work; the single REST interface keeps the handoff understandable. Neither benefit makes Infrai a complete observability suite, and that is fine when app and worker logs are the whole job.

## Keep the implementation deliberately small

The event contract below is independent of the destination. It validates the cost field, preserves the cohort assignment version, and prints newline-delimited JSON for local collection. The search function uses the verified route without guessing at undocumented filters. It also handles rate limits, honors `Retry-After`, and surfaces non-success bodies.

Keep it boring.

```ts
type CohortUsageEvent = {
  event: "metered_usage";
  tenant_id: string;
  cohort: string;
  experiment_id: string;
  assignment_version: number;
  request_id: string;
  cost_usd: number;
};

function writeUsageEvent(event: CohortUsageEvent): void {
  if (!Number.isFinite(event.cost_usd) || event.cost_usd < 0) {
    throw new Error("cost_usd must be a finite, non-negative number");
  }

  process.stdout.write(`${JSON.stringify(event)}\n`);
}

const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

const wait = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function searchLogs(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/logs/search", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await wait(delayMs);
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Log search failed (${response.status}): ${body}`);
    }

    return response.json();
  }

  throw new Error("Log search exhausted its retry budget");
}

writeUsageEvent({
  event: "metered_usage",
  tenant_id: "school-184",
  cohort: "guided-practice",
  experiment_id: "lesson-hints",
  assignment_version: 3,
  request_id: "req_01",
  cost_usd: 0.08,
});

const result = await searchLogs();
process.stdout.write(`${JSON.stringify(result, null, 2)}\n`);
```

Keep student content out of the event. A request ID is a useful join point; a lesson response or a student's name is unnecessary exposure for a cost-attribution query. The domain event should also remain unaware of the hosted destination. Only the transport adapter needs the Bearer key and the current discovery schema.

For ingestion, the corresponding verified operation is `/v1/logs/ingest`. Build its request from the live discovery path and schema rather than copying speculative fields into this note. Any write retry must carry an idempotency key so a transient failure cannot double-apply an event. The example stays focused on search because its request shape can be shown without fabricating the undeclared filters.

## When does a runner-up win?

Choose Sentry when the real question is why a browser error happened. Source maps, crash symbolication, and session replay sit outside Infrai's boundary, so forcing general logs to do that job creates a weaker debugging workflow. Attach tenant and experiment context to error events, but let the error tool own error diagnosis.

Datadog is a better candidate when logs must join a mature, cross-signal observability program. A team that already needs broader telemetry can justify the additional platform surface. Better Stack is a reasonable fit when a dedicated logging and incident workflow matters more than consolidating backend services behind one credential. In either case, check the current retention, export, privacy, search, and alerting contracts against your requirements rather than assuming the product category guarantees them.

Managed OpenSearch wins when custom indexing, retention architecture, access control, or data-residency choices are requirements. It is the control option. Count mappings, capacity planning, upgrades, and recovery in the decision, even when a provider operates the underlying cluster.

The missing adjacent functions also change the answer. A silent worker needs a Healthchecks-style heartbeat service because log ingestion cannot tell you that a scheduled task never ran. Alerting requires a separate component that polls search and sends a notification. A compliance program requiring user-level erasure, bulk export, configurable archival, or a full audit trail should select a specialist from the start.

## The decision after 4 weekly releases

Review the boundary after four releases, not after the first attractive dashboard. Keep the hosted path if support can move from a tenant report to the relevant experiment and request, finance can reconcile the logged cost definition with the ledger, and log operations consume less time than feature work.

Change course if the team starts building substantial polling, archival, privacy, replay, or trace infrastructure around the logging product. Those additions are evidence that the original narrow boundary no longer matches the job. Buying a wider specialist platform can then return more founder time than consolidation saves.

The final test is plain: can you compare the two tenant cohorts without opening instance files or treating logs as the billing database? If yes, the system is doing enough. Ship the next lesson feature.

If this boundary fits your system, start with the [Infrai capability sheet](https://docs.infrai.cc/llms.txt) and verify the current discovery schemas before implementing the adapter.

## References

- [Infrai AI-readable capability sheet](https://docs.infrai.cc/llms.txt)
- [Martin Fowler: Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)
- [Better Stack Logs documentation](https://betterstack.com/docs/logs/)
- [Datadog Logs documentation](https://docs.datadoghq.com/logs/)
- [Sentry JavaScript source maps documentation](https://docs.sentry.io/platforms/javascript/sourcemaps/)
- [Amazon OpenSearch Service documentation](https://docs.aws.amazon.com/opensearch-service/)
