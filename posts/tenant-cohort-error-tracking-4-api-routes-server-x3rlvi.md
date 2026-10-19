# Tenant Cohort Error Tracking: 4 API Routes, Server Actions, and Edge Signals

Use one error envelope across API routes, server actions, and edge handlers, then attribute cost with four bounded signals: `tenant_cohort`, `experiment_variant`, `operation`, and `runtime`. Keep tenant IDs out of metric labels. Upload source maps as build artifacts, restrict their access, and treat symbolication as enrichment rather than a condition for recording the failure.

| Choice | Cohort comparison | Cost attribution | Operational load |
|---|---|---|---|
| One shared envelope and sink | Same dimensions everywhere | Direct by cohort and runtime | One schema to maintain |
| Runtime-specific adapters, shared envelope | Comparable after normalization | Direct after normalization | Adapter tests per runtime |
| Separate pipelines and schemas | Requires later joins | Estimates depend on join quality | Highest ongoing work |

**Recommendation:** use runtime-specific adapters behind one small TypeScript interface, and emit the same envelope to a sink you control. This preserves comparable experiment data without pretending that a long-lived server process and an edge isolate have identical capabilities. For a solo SaaS, that is the useful middle: outsource transport details, retain the attribution contract, and get back to shipping weekly.

## How should API routes and server actions capture error context?

A logistics experiment usually crosses more than one server boundary. A dispatcher might request a quote through an API route, confirm it through a server action, and trigger a lightweight validation at the edge. If those failures arrive with different field names, comparing the control cohort with the experiment cohort becomes a data-cleaning project. The expensive question is not merely "how many errors occurred?" It is "did variant B increase failed booking work for small-fleet tenants, and where did that work run?"

The event contract should answer that question without encoding an individual customer into every metric series. Record the four bounded dimensions on each event. Keep a pseudonymous tenant key in the event payload for investigation, but aggregate metrics by a deliberately small cohort vocabulary such as `small_fleet`, `regional`, and `enterprise`. Prometheus warns that labels create new time series for every label-set combination and specifically cautions against high-cardinality dimensions such as user IDs. A tenant ID is the same kind of trap.

This matters to cost attribution because cardinality is work. More series consume more storage and make queries harder to operate. The revenue-per-hour test is blunt: a dimension earns its place only if it changes an experiment decision or lets an operator assign work to the right bucket. In the logistics example, the 4 fields support a concrete review: compare `control` with `quote_v2` inside one cohort, split the result by `quote` and `book_shipment`, then check whether the runtime distribution changed. A raw tenant identifier would add investigative detail but no new experiment axis. It belongs in controlled event storage, not the metric label set. This division also keeps a single noisy tenant from expanding the metric schema while preserving a path from an aggregate regression to the relevant events. The limitation is deliberate: aggregate charts cannot answer every tenant-specific support question on their own. That investigation requires the event store and its access controls.

That is the trade-off.

Keep severity separate from business outcome. RFC 5424 defines ordered severity values from Emergency through Debug, but a rejected quote can be an expected business result rather than a system error. Conversely, a failed booking write may deserve an error event even when the handler converts it into a polite response. Capture the technical failure once, at the boundary that owns the response.

## Two criteria decide the design

The first criterion is **attribution integrity**. Every runtime must populate the same experiment fields before transport. Do not infer cohort later from mutable account state; the assignment attached to the request is the evidence for that request. Validate the vocabulary at compile time and reject unknown values in tests.

The second is failure independence. Error capture must not replace the application result, hang the response, or assume process-local flushing will finish after the handler returns. The adapter should accept the runtime's available execution mechanism and make a bounded attempt to deliver. If delivery fails, the original exception still follows the handler's normal error path.

Short paths win.

Four fields. One contract.

Source maps sit outside both criteria. They improve a stack trace after collection, but they do not establish the tenant cohort, experiment assignment, operation, or runtime. Associate maps with an immutable build identifier, keep that identifier on the event, and prevent map artifacts from being served as public application assets. A missing map should reduce debugging detail, never erase the underlying event.

## A small TypeScript boundary

The application code needs a contract, not a vendor SDK spread through every handler. This example normalizes unknown thrown values, captures only bounded attribution fields, and preserves the original control flow. The transport can be implemented with the APIs available in each runtime.

```ts
type TenantCohort = "small_fleet" | "regional" | "enterprise";
type Variant = "control" | "quote_v2";
type Runtime = "server" | "edge";

type ErrorContext = {
  tenantCohort: TenantCohort;
  experimentVariant: Variant;
  operation: "quote" | "book_shipment";
  runtime: Runtime;
  buildId: string;
  tenantKey?: string;
};

type ErrorEnvelope = ErrorContext & {
  name: string;
  message: string;
  stack?: string;
  occurredAt: string;
};

type ErrorTransport = {
  send(event: ErrorEnvelope): Promise<void>;
};

function normalizeError(error: unknown): Pick<ErrorEnvelope, "name" | "message" | "stack"> {
  if (error instanceof Error) {
    return { name: error.name, message: error.message, stack: error.stack };
  }
  return { name: "UnknownThrownValue", message: String(error) };
}

async function captureServerError(
  error: unknown,
  context: ErrorContext,
  transport: ErrorTransport,
): Promise<void> {
  const event: ErrorEnvelope = {
    ...context,
    ...normalizeError(error),
    occurredAt: new Date().toISOString(),
  };

  try {
    await transport.send(event);
  } catch {
    // Capture must not replace the application's original failure.
  }
}

async function withErrorCapture<T>(
  context: ErrorContext,
  transport: ErrorTransport,
  work: () => Promise<T>,
): Promise<T> {
  try {
    return await work();
  } catch (error: unknown) {
    await captureServerError(error, context, transport);
    throw error;
  }
}
```

An API route can wrap its request handler with `withErrorCapture`. A server action can wrap the mutation it owns. An edge handler uses the same envelope but supplies an edge-compatible transport. The wrapper rethrows after capture, so framework error handling remains responsible for the response. Do not wrap and record the same exception again at every layer; choose the boundary where the failure becomes an externally visible failed operation.

Test the contract at three levels. A unit test checks normalization for `Error`, strings, and rejected promises. A contract test feeds each runtime adapter the same context and compares the resulting envelope. A deployment smoke test throws a synthetic, non-customer error tagged with a fixed test cohort and confirms that its build ID can be symbolicated. Delete or isolate synthetic events before experiment analysis.

## When separate pipelines are the better runner-up

Separate pipelines are reasonable when data residency, access control, or retention requirements differ by runtime or tenant class. They can also fit when edge traffic is sampled under a distinct policy and the central pipeline cannot enforce that policy before ingestion. In those cases, keep the envelope versioned and identical, then reconcile at a controlled analytics boundary. The shared-envelope approach has a real limitation here: a common schema does not create common retention, residency, or access policy. If those controls must diverge, choose separate sinks and accept the reconciliation work.

The cost is delayed confidence. A cohort comparison now depends on both pipelines arriving, retaining compatible fields, and applying matching severity and sampling rules. Budget for reconciliation tests. Without them, a lower error count can mean a healthier variant, a stricter sampler, or a broken join.

Avoid a shared pipeline when sharing would violate an actual isolation requirement. Avoid separate schemas because two runtimes happen to expose different APIs. Those are different decisions.

## The operating rule

Ship the shared envelope first. Review four breakdowns for each experiment: cohort, variant, operation, and runtime. Track counts and rates with bounded labels; retain pseudonymous tenant detail in controlled event storage only when investigation requires it. Version the envelope when a field changes meaning, and keep old readers working through the comparison window.

This design does not make runtime differences disappear. It confines them to transport adapters, where they can be tested, while the logistics decision stays stable: compare failed work across tenant cohorts and assign its operational cost to the experiment that caused it. That is enough structure to make a weekly shipping decision without building an observability platform as a side business.

## References

- https://prometheus.io/docs/practices/instrumentation/
- https://datatracker.ietf.org/doc/html/rfc5424
