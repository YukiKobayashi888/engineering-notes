# Express Feature Flag Check: How to Gate a Node.js API Route

Use one boolean flag for each billable search boundary, and evaluate it before the route touches the log store. **TL;DR:** for a nightly property pipeline, `is_enabled` should answer whether a cost center may search its structured logs; authentication and authorization must still happen first. Fail closed when the flag cannot be read, record the decision without tenant secrets, and keep flag evaluation out of the query function.

| Choice | Cost attribution | Failure behavior | Best fit |
|---|---|---|---|
| Boolean gate per cost center | Direct and easy to audit | Deny the gated route | A new search capability with a clear on/off boundary |
| Percentage rollout | Indirect unless assignment is logged | Stable only with deterministic assignment | Measuring behavior across a large user population |
| Permission in the auth model | Direct, but coupled to identity policy | Follows authorization availability | A durable entitlement rather than a temporary rollout |

The recommendation is the first row. A property manager's nightly import can produce logs for hundreds of buildings, but the useful accounting unit is usually the customer or internal cost center paying for those searches. A boolean gate makes that boundary explicit. It also gives a solo operator one fast rollback while leaving the weekly shipping cadence intact.

## How should Express middleware check a boolean feature flag?

A feature flag is not an access-control system. The request must first establish who is calling and which cost center they may access. The flag then controls whether that already-authorized scope can use the new search path. Reversing those checks can leak rollout state, and trusting a cost-center ID from a header can let one tenant test another tenant's gate.

The order is small but important: authenticate, authorize the route parameter, evaluate the flag from the trusted authorization result, then execute the search. **The flag changes availability, never identity or scope.** For this property-log route, that means a caller cannot select another company's `cost_center_id`, discover its rollout state, or use its search budget merely by changing the URL. The route parameter is input to authorization, not proof of authorization. Only the verified context reaches the evaluator. That separation also keeps the nightly pipeline independent: ingestion does not stop because an additive search feature is off.

For structured pipeline logs, attach the same trusted cost-center key to the decision event and the downstream query metrics. That creates a joinable cost trail: allowed search attempts, query duration, scanned records, and storage or compute consumption can all be aggregated on the same dimension. Do not put lease documents, access tokens, tenant names, or raw query text in the decision log. OWASP recommends excluding or masking sensitive data such as session identifiers, access tokens, passwords, and personal data from application logs.

Keep the event compact.

More fields are not automatically more observability.

## Two criteria decide whether a boolean is enough

First, choose a key that matches the invoice or budget review. `user_id` is tempting because it is readily available, but it fragments a property company's usage across employees. `building_id` can be too granular when one management company owns many buildings. A stable `cost_center_id` derived during authorization matches the question the operator will face later: who caused this search load?

Second, define unavailable flag state before deployment. For an additive and potentially expensive log-search route, the conservative result is disabled. Existing ingestion continues; only the gated search returns a temporary-unavailable response. This is a deliberate trade-off: a short loss of the new capability is preferable to unattributed queries with no owner. I would not apply that default to a flag protecting a critical existing path. There, denying service may be the larger failure, so the policy needs a separate review and an explicitly tested fallback. One universal helper default hides that distinction.

Use three states internally even though the business flag is boolean: enabled, disabled, and evaluation unavailable. Collapsing the last two in the response is fine. Collapsing them in telemetry is not. An operator needs to distinguish a deliberate rollout decision from a dependency failure.

## Implement the Express gate in TypeScript

The following example uses an injected interface, so the HTTP layer does not know where flags live. It assumes prior middleware has produced a trusted authorization context. Express middleware either ends the response or calls `next()`; Express also documents that middleware which does neither leaves the request hanging.

```ts
import express, { NextFunction, Request, Response } from "express";

type AuthzContext = {
  subjectId: string;
  costCenterId: string;
};

type AuthorizedRequest = Request & {
  authz?: AuthzContext;
};

interface BooleanFlagReader {
  isEnabled(flag: string, context: { costCenterId: string }): Promise<boolean>;
}

interface AuditSink {
  write(event: {
    event: "feature_flag_decision";
    flag: string;
    cost_center_id: string;
    outcome: "enabled" | "disabled" | "unavailable";
  }): void;
}

function requireFlag(
  reader: BooleanFlagReader,
  audit: AuditSink,
  flag: string,
) {
  return async (
    req: AuthorizedRequest,
    res: Response,
    next: NextFunction,
  ): Promise<void> => {
    const costCenterId = req.authz?.costCenterId;
    if (!costCenterId) {
      res.status(403).json({ error: "forbidden" });
      return;
    }

    try {
      const enabled = await reader.isEnabled(flag, { costCenterId });
      audit.write({
        event: "feature_flag_decision",
        flag,
        cost_center_id: costCenterId,
        outcome: enabled ? "enabled" : "disabled",
      });

      if (!enabled) {
        res.status(404).json({ error: "not_found" });
        return;
      }

      next();
    } catch (error: unknown) {
      audit.write({
        event: "feature_flag_decision",
        flag,
        cost_center_id: costCenterId,
        outcome: "unavailable",
      });
      res.status(503).json({ error: "temporarily_unavailable" });
    }
  };
}

const app = express();
const flags: BooleanFlagReader = {
  async isEnabled(_flag, context) {
    return process.env[`SEARCH_${context.costCenterId}`] === "true";
  },
};
const audit: AuditSink = { write: event => console.info(JSON.stringify(event)) };

app.get(
  "/cost-centers/:costCenterId/pipeline-logs",
  (req: AuthorizedRequest, res, next) => {
    // Replace this demo with authentication plus a membership check.
    req.authz = { subjectId: "operator-42", costCenterId: req.params.costCenterId };
    next();
  },
  requireFlag(flags, audit, "nightly_log_search"),
  async (req: AuthorizedRequest, res) => {
    res.json({ cost_center_id: req.authz?.costCenterId, records: [] });
  },
);

app.listen(3000);
```

The environment-backed reader is intentionally plain. It makes the example runnable, while the interface leaves room for a database, a local configuration snapshot, or a remote evaluator without changing route behavior. Production authorization must verify membership instead of copying the route parameter as the demonstration middleware does.

There is one operational trap here: request volume can turn one flag decision into one remote call per search. Put any cache inside the reader, set a bounded freshness policy, and expose cache age in metrics. Do not add an unbounded in-memory map keyed by arbitrary request input. The key comes from authorization, and the cache needs a size limit. A local snapshot lowers request-path dependency risk but accepts stale decisions; a remote read sees changes sooner but adds latency and another failure mode. Pick the stale window before rollout and include it in the rollback plan.

Test all three outcomes. Enabled must reach the handler exactly once. Disabled must not reach it. A rejected evaluation must return `503` and emit `unavailable`; the audit sink itself should be buffered or otherwise prevented from crashing the request path. These tests protect the behavior that matters during a hurried rollback.

## When is the runner-up better?

The boolean approach has a real limitation: it cannot measure a gradual population response, and one gate per cost center becomes awkward when rules multiply across roles, buildings, and plans. Choose deterministic percentage rollout when the question is behavioral rather than financial: for example, whether search latency remains acceptable across a representative population. The assignment must be stable for the same subject, and the selected variant must be logged, or a user may move between cohorts and corrupt the comparison. Percentage rollout is a poor first tool for cost attribution because percentages do not identify which budget owns the work.

Move the rule into authorization when log search becomes a purchased or contractual entitlement. Flags are good at temporary operational control; durable permissions deserve reviewable policy, lifecycle handling, and tests alongside the rest of the access model. Running both indefinitely creates two sources of truth.

This is the revenue-per-hour decision. Keep the boolean while it buys a clean rollout and a fast stop button. Once the route is stable, either remove the gate or promote the rule to the system that actually owns entitlement. Small code is easier to ship weekly, and undifferentiated evaluation storage can stay behind the narrow reader interface.

## References

- https://expressjs.com/en/guide/using-middleware.html
- https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- https://www.rfc-editor.org/rfc/rfc9110.html
