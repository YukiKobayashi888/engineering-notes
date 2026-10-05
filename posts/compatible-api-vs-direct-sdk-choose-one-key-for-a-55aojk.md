# Compatible API vs Direct SDK — Choose One Key for App Chatbot

TL;DR: An app chatbot that extracts supplier invoices can place an OpenAI-compatible API behind one server-side key while keeping validation and retry policy in the app. The direct SDK alternative earns its extra integration only when a provider-specific capability measurably improves field accuracy.

For a one-person SaaS, one key and one transport path reduce weekly shipping drag. They do not remove the need to test quality and latency separately. A wrong invoice total can create support work worth far more than a fast response saves, while an in-app support agent cannot leave a user staring at a spinner through unbounded retries. The practical target is a bounded synchronous attempt with a review path.

## Should an app chatbot use a compatible API alternative?

The gateway wins here because transport is undifferentiated work. A stable request shape, one credential boundary, and normalized request metadata leave more founder-hours for the extraction workflow itself. Ship weekly. Spend those cycles on field definitions, fixtures, validation, and the review screen.

Compatibility has a limit. A shared chat-style request can carry text and request structured data, but provider-specific schemas, safety behavior, error bodies, and model capabilities may differ. Treat the compatible surface as an adapter, not as proof that every backend behaves identically. The application owns the contract. The gateway also adds a network hop and another failure domain, can expose only the common part of unlike provider APIs, and is not suitable when a native feature is essential to extraction quality. Those are real limitations, not footnotes.

Direct calls become the better choice under one concrete condition: a capability available only through a native SDK produces a repeatable improvement on the invoice test set, and that improvement matters more than the maintenance and credential overhead. Until that evidence exists, extra integrations are speculative complexity.

That is the trade-off.

The revenue-per-hour lens makes the choice less romantic. A second SDK means another authentication path, error taxonomy, dependency update stream, and set of operational checks. Those costs recur. Outsource the common transport behind an interface and keep the differentiated judgment local.

## The constraint that changed the design

The customer-support workflow needs four fields from supplier invoices: invoice number, supplier name, currency, and total. Inputs can contain missing labels, duplicated totals, tax-inclusive amounts, and text from poor OCR. A syntactically valid response can still be commercially wrong.

Four fields. Two execution paths. One retry owner.

That rules out a design where the chatbot calls a model and immediately writes the returned object into a ticket. Extraction produces a candidate. Deterministic code checks its shape and basic invariants; uncertain candidates enter review. The chat response can acknowledge receipt without pretending the invoice has been accepted.

Latency is split the same way. The user-facing request gets one bounded attempt. Work that can wait belongs on an asynchronous path with an idempotency key. OpenAI documents its Batch API as asynchronous processing, which supports the broader architectural distinction: batch work and interactive chat have different latency contracts. It is not a reason to put an interactive request on a batch endpoint.

One trap deserves emphasis. Do not hide retries inside three layers. If the SDK retries, the gateway retries, and the application retries, one click may create more work than the owner realizes. Pick one layer to own retry policy, cap attempts, and record every attempt under the same operation ID.

No invisible retry stack.

## The smallest working implementation

The useful abstraction is narrow. It accepts invoice text, returns a candidate plus request metadata, and exposes no provider-specific object to the rest of the application. The endpoint below is deliberately pseudonymous; the contract is the point.

```ts
type InvoiceCandidate = {
  invoiceNumber: string | null;
  supplierName: string | null;
  currency: string | null;
  total: number | null;
};

type ExtractionResult = {
  candidate: InvoiceCandidate;
  operationId: string;
  latencyMs: number;
};

const requiredKeys = [
  "invoiceNumber",
  "supplierName",
  "currency",
  "total",
] as const;

function isInvoiceCandidate(value: unknown): value is InvoiceCandidate {
  if (!value || typeof value !== "object") return false;
  const row = value as Record<string, unknown>;

  return requiredKeys.every((key) => key in row)
    && (row.invoiceNumber === null || typeof row.invoiceNumber === "string")
    && (row.supplierName === null || typeof row.supplierName === "string")
    && (row.currency === null || typeof row.currency === "string")
    && (row.total === null
      || (typeof row.total === "number" && Number.isFinite(row.total)));
}

export async function extractInvoice(text: string): Promise<ExtractionResult> {
  const operationId = crypto.randomUUID();
  const startedAt = performance.now();
  const response = await fetch("https://gateway.example/v1/chat/completions", {
    method: "POST",
    headers: {
      "authorization": `Bearer ${process.env.AI_GATEWAY_KEY ?? ""}`,
      "content-type": "application/json",
      "x-operation-id": operationId,
    },
    body: JSON.stringify({
      model: process.env.EXTRACTION_MODEL,
      messages: [{
        role: "user",
        content: `Extract invoiceNumber, supplierName, currency, and total. Use null when absent.\n\n${text}`,
      }],
      response_format: { type: "json_object" },
      temperature: 0,
    }),
    signal: AbortSignal.timeout(8_000),
  });

  if (!response.ok) {
    throw new Error(`Extraction request failed: ${response.status}`);
  }

  const payload = await response.json() as {
    choices?: Array<{ message?: { content?: string } }>;
  };
  const content = payload.choices?.[0]?.message?.content;
  if (!content) throw new Error("Extraction response had no content");

  const candidate: unknown = JSON.parse(content);
  if (!isInvoiceCandidate(candidate)) {
    throw new Error("Invoice candidate failed validation");
  }

  return {
    candidate,
    operationId,
    latencyMs: performance.now() - startedAt,
  };
}
```

Eight seconds is an application budget in this example, not a universal recommendation. Set it from the support experience the product is willing to offer, then measure timeout and review rates. The important property is that the budget is explicit.

For HTTP 429 responses, the selected retry owner should honor `Retry-After` when present and use capped exponential backoff with jitter. Any later write to a ticket or review queue needs an idempotency key derived from the invoice operation, so retrying transport cannot duplicate the business action.

This code is intentionally incomplete at the business boundary. A finite number is not proof that the total is correct. Currency should be checked against the source text, and the candidate should not trigger payment or accounting changes without the workflow's required review.

Keep credentials server-side. The browser talks to the application, and the application talks to the gateway. That preserves one place for authorization, request limits, audit metadata, and redaction before invoice text leaves the application boundary. Regional handling also belongs in deployment policy: decide where invoice content may be processed, then configure and test the selected path rather than inferring location from an API-shaped interface.

## Invoice extraction failure gates

Build a fixed evaluation set before comparing routes. Include clean invoices, OCR noise, missing currency, multiple totals, credits, and duplicate invoice numbers. Store the expected fields separately from the prompt. Never tune against the only copy of the test set.

For every candidate route, record exact-field accuracy, invalid-output rate, review rate, end-to-end latency, timeout rate, and retry count. Aggregate numbers alone are weak. Break failures down by field and document type, because a route that usually finds supplier names but regularly chooses a subtotal is not good enough for this job.

Use the same contract and fixtures for gateway and direct-call trials. Change one variable at a time. A simple decision table keeps the choice tied to the product rather than to a demo:

| Signal | Keep the gateway | Add a direct path |
| --- | --- | --- |
| Field quality | Meets the acceptance threshold on held-out invoices | Native capability shows a repeatable, material gain |
| Interactive latency | Fits the declared request budget | Gateway overhead causes measured budget failures |
| Operations | One retry owner and one audit trail | Native path can preserve equivalent controls |
| Maintenance | Adapter stays stable during model changes | Extra SDK work earns back founder-hours |

Do not combine those signals into a mystery score too early. A hard quality floor and a hard latency ceiling are easier to reason about. Among routes that pass both, prefer the one with less operational surface.

The fastest failing route still fails.

## What I would change at scale

At higher volume, separate ingestion from extraction with a durable queue, store an idempotency key per invoice, and move slow retries outside the chat request. Add a dead-letter review path. Keep raw model output for only as long as the data policy permits, while retaining enough structured telemetry to explain a rejection.

I would also replace the in-process shape guard with a versioned schema and test each adapter against recorded contract fixtures. Roll out model or routing changes to a small traffic slice, compare field-level outcomes, then expand. The application should be able to disable a route without a client release.

The decision boundary stays firm: use the compatibility gateway while it clears both quality and latency gates. Add a direct integration only after a controlled test proves that a native feature fixes a meaningful extraction failure. Cheapest is a poor primary metric here. Weekly shipping time, review workload, and wrong-field risk are the bill that matters.

## Further reading

- https://platform.openai.com/docs/guides/batch
- https://elevenlabs.io/docs
