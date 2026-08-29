# Node.js Password Reset API Under 429: Isolate Fintech Report Email Traffic

**Short answer:** A password reset email should never wait behind a generated financial report: put authentication mail and report attachments in separate durable queues, enforce idempotency in the application database, and retry HTTP 429 responses with bounded, randomized backoff.

That is the smallest design I would ship for a one-person fintech SaaS. It protects the urgent path without turning email into a platform project, and it leaves the generated-report workflow free to absorb throttling. The important move isn't a clever retry formula. It is refusing to let two jobs with radically different deadlines compete in one FIFO line.

## How can a Node.js email API keep password resets ahead of 429 retries?

The system has two kinds of transactional email. A password reset contains a short-lived recovery link and is requested by a person who is waiting. A generated account report can carry a PDF or CSV attachment, is larger, and usually tolerates more delay. Both may call the same email send endpoint, but they do not have the same delivery objective. One queue hides that distinction. Imagine 800 reports becoming ready after a reporting run, followed by one reset request. Even if the provider is accepting traffic normally, the reset inherits the report backlog. If the endpoint responds with 429, naive workers make the coupling worse: every job sleeps, wakes on roughly the same schedule, and contests the same capacity again. More workers can increase pressure without moving the urgent message forward. The concrete reliability budget therefore belongs to each lane — maximum useful queue age, retry window, and terminal state — rather than to email as a whole.

Split the work before tuning concurrency.

I use two logical lanes: `auth-mail` and `report-mail`. The auth lane gets a reserved concurrency budget and a strict age alarm. The report lane gets lower concurrency, wider retry windows, and attachment-size checks before enqueueing. This is still one worker implementation and one transport adapter — it isn't a miniature message-routing company.

The database also needs one durable record per business send. For a reset, the natural identity is the reset request ID. For a report, it is the report generation ID plus recipient and report version. A unique constraint on that identity prevents two HTTP handlers, two queue deliveries, or a process restart from creating two logical sends. Provider-side idempotency can be useful when a verified contract offers it, but application correctness shouldn't depend on an undocumented header.

This separation matters for security as well. The application remains responsible for reset-token generation, storage, expiration, and one-time redemption. Email delivery status must not decide whether a token is valid. The public reset route should return the same neutral response for known and unknown accounts, while rate controls cover both reset issuance and token verification. NIST SP 800-63B is the primary reference I use for authenticator and recovery requirements; the exact assurance level and threat model still need to be chosen for the product.

Treat 429 as scheduling information, not as permission to create a fresh email. The worker records the failed attempt, computes a future `nextAttemptAt`, and releases the process. It does not hold a timer for minutes inside a server instance. When the job becomes eligible again, it reuses the same outbox row and the same stable operation key.

The backoff should grow, stop growing at a cap, and include randomness. For example, a worker can choose a random delay between zero and an exponential ceiling. Randomness prevents a fleet of workers from synchronizing after the same throttle response. The ceiling prevents a late attempt from being pushed beyond any useful recovery window. These are policy values, not universal constants; I'm not sure a single set of numbers could fit both a five-minute recovery link and a report that remains useful tomorrow.

Keep the retry classifier narrow. Retry 429 and transport failures whose outcome is genuinely unknown. Do not retry a rejected address or an invalid payload as though time will repair it. If a request times out after the remote service may have accepted it, the stable operation key and the locally unique outbox identity are what stop ambiguity from becoming a duplicate. When the email API has no verified idempotency mechanism, reconcile uncertain outcomes through its documented message lookup or event mechanism before issuing another send.

No guessing.

A retry budget must be expressed in time as well as attempts. Password-reset work expires when the corresponding token is no longer useful, so the worker checks `expiresAt` before every attempt. Report delivery has a separate service target and can wait longer. Dead jobs move to an operator-visible state with the last classified reason; they do not loop forever, and they do not silently disappear.

The metrics follow the user promise. Track oldest eligible job age by lane, accepted-send latency, retry count by reason, and terminal outcome. A global throughput chart can look healthy while one reset sits behind attachment work. Lane-specific queue age exposes that failure immediately.

## Break the design with a failure matrix

I make the 429 fixture boring and deterministic. In the failure test, I force HTTP 429 twice, record every operation key, and never touch a network. A fake clock then proves that a throttled job becomes eligible later without changing identity. This gives me more confidence than manually watching logs because it checks the ambiguous boundaries on every change.

| Injected condition | Invariant to assert | Wrong outcome |
| --- | --- | --- |
| 800 report jobs, then one reset | Reserved auth capacity claims the reset without draining reports first | One FIFO backlog controls both deadlines |
| 429 on the first two attempts | The same outbox ID and operation key reach attempts two and three | A retry creates another logical email |
| Two workers claim together | One lease owner calls the transport | Both workers send the same job |
| Retry time passes token expiry | The reset job becomes terminal before another call | Expired recovery mail is accepted later |
| Process exits after remote acceptance | Reconciliation runs before an uncertain resend | Restart means “send again” |

There is one subtle test worth keeping. Enqueue the same report identity twice with different request IDs and verify that the database returns the original outbox row. HTTP request IDs describe transport attempts; they are not business identities. Mixing those concepts makes duplicate prevention disappear precisely when a caller retries after losing a response.

## The outbox state machine and its operating boundary

The code below deliberately stops at a generic `EmailTransport`. Recipient fields, attachment encoding, endpoint paths, authentication headers, idempotency headers, and response bodies vary by contract. The adapter must be implemented from the current documentation for the chosen service. The reusable part is the state transition around it.

```ts
type Lane = "auth-mail" | "report-mail";
type OutboxStatus = "pending" | "sending" | "accepted" | "dead";

interface OutboxJob {
  id: string;
  operationKey: string;
  lane: Lane;
  status: OutboxStatus;
  attempts: number;
  nextAttemptAt: Date;
  expiresAt: Date;
  payload: {
    to: string;
    template: "password-reset" | "generated-report";
    variables: Record<string, string>;
    attachment?: {
      filename: string;
      contentType: "application/pdf" | "text/csv";
      objectKey: string;
    };
  };
}

interface SendResult {
  acceptedId: string;
}

interface EmailTransport {
  send(job: OutboxJob): Promise<SendResult>;
}

class RateLimited extends Error {
  constructor(readonly retryAfterMs?: number) {
    super("Email endpoint rate limited the request");
  }
}

interface OutboxStore {
  claimNext(lane: Lane, now: Date): Promise<OutboxJob | null>;
  markAccepted(id: string, acceptedId: string): Promise<void>;
  reschedule(id: string, attempts: number, nextAttemptAt: Date): Promise<void>;
  markDead(id: string, reason: string): Promise<void>;
}

const MAX_ATTEMPTS = 6;
const MAX_BACKOFF_MS = 60_000;

function fullJitterBackoff(attempt: number): number {
  const ceiling = Math.min(MAX_BACKOFF_MS, 1_000 * 2 ** attempt);
  return Math.floor(Math.random() * ceiling);
}

function nextDelay(error: RateLimited, attempt: number): number {
  if (error.retryAfterMs !== undefined) {
    return Math.min(MAX_BACKOFF_MS, Math.max(0, error.retryAfterMs));
  }
  return fullJitterBackoff(attempt);
}

async function runOne(
  lane: Lane,
  store: OutboxStore,
  transport: EmailTransport,
  now = new Date(),
): Promise<void> {
  const job = await store.claimNext(lane, now);
  if (!job) return;

  if (job.expiresAt <= now) {
    await store.markDead(job.id, "delivery window expired");
    return;
  }

  try {
    const result = await transport.send(job);
    await store.markAccepted(job.id, result.acceptedId);
  } catch (error: unknown) {
    const attempts = job.attempts + 1;
    if (!(error instanceof RateLimited) || attempts >= MAX_ATTEMPTS) {
      await store.markDead(job.id, "non-retryable or retry budget exhausted");
      return;
    }

    const delayMs = nextDelay(error, attempts);
    const nextAttemptAt = new Date(now.getTime() + delayMs);
    if (nextAttemptAt >= job.expiresAt) {
      await store.markDead(job.id, "next attempt exceeds delivery window");
      return;
    }

    await store.reschedule(job.id, attempts, nextAttemptAt);
  }
}
```

There are two details outside this snippet that I would make database invariants. First, `operationKey` is unique, so enqueueing the same reset or generated report twice returns the existing row. Second, `claimNext` uses an atomic lease or row lock, so only one worker owns a job at a time. The `sending` lease must have a recovery path after process termination, but recovery restores the same row; it never invents a new operation.

The transport adapter is also where a documented server delay is normalized into `retryAfterMs`. I've kept that parsing at the edge so the scheduling policy stays testable without HTTP fixtures. Unit tests can inject a `RateLimited(12_000)` result, advance a fake clock, and assert that the same job ID becomes eligible later. A concurrency test should attempt to enqueue one operation key twice and prove that exactly one outbox row exists.

For attachments, the queue payload stores an object key rather than a large base64 value. The worker reads the immutable report object just before sending, verifies the expected content type and size against the chosen API's documented limits, then hands it to the adapter. A short object-retention window must exceed the report retry window. Otherwise the email job can remain valid after its attachment has vanished.

**At higher volume**, I would preserve the two-lane contract and replace fixed worker counts with per-lane admission control. Authentication mail keeps reserved capacity. Report traffic consumes the remainder and slows first. If multiple tenants share a sending domain, fair scheduling by tenant prevents one bulk report run from occupying the entire report lane.

The catch is operational cost. Two lanes mean two age alarms, two concurrency settings, and more queue states to inspect. This design is not suitable when the application sends only a handful of messages and reports are generated one at a time; a single durable queue with auth priority may be enough. Stick with separate providers or separate sending domains when compliance boundaries, reputation isolation, or organizational ownership require hard separation rather than scheduler policy. A solo product should not take on that extra integration burden without a concrete boundary to enforce.

At larger scale, delivery evidence belongs in its own append-only event stream keyed by the internal outbox ID and the accepted remote ID. That supports audits without letting a late event mutate security state. The reset token is consumed by the authentication transaction, not by an email event. Report delivery can update a user-facing report status, but it should retain the distinction between “accepted for delivery” and evidence observed later.

Domain authentication deserves the same deliberate treatment. DMARC builds policy and reporting on identifiers aligned with SPF or DKIM. It helps domain owners publish handling policy and receive reports; it does not prove that an individual message reached an inbox. Keep DMARC aggregate reports, API acceptance, later delivery evidence, and user-visible completion as separate signals. Collapsing them into one `sent: true` field creates confident dashboards and weak incident diagnosis.

My decision rule is revenue per engineering hour: own the queue semantics because they encode the product promise, and outsource the undifferentiated mail transport behind a narrow adapter. Ship the first version weekly, observe lane age and retry reasons, then change capacity from evidence. Don't build automatic provider failover until duplicate suppression, identity alignment, attachment handling, and event reconciliation are defined across both paths. Failover adds a second ambiguous-outcome boundary; it isn't free reliability.

## References

- NIST SP 800-63B, Digital Identity Guidelines: https://pages.nist.gov/800-63-3/sp800-63b.html
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489

## Further reading

The NIST guidance above is the starting point for authenticator recovery controls. RFC 7489 defines DMARC policy, identifier alignment, and reporting; read it before treating domain authentication as a delivery receipt.
