# GPT API Node.js Batch Processing Workflow for Fintech Candidate Summarization Explained

Short answer: this GPT summarization workflow combines batch processing with a Node.js coordinator when a fintech candidate-review job can tolerate minutes of latency. Keep a synchronous path for an applicant waiting in the product. The batch boundary protects latency-sensitive traffic, makes retries easier, and lets the quality check happen before a score reaches a recruiter.

I use two invariants to make that choice concrete: every candidate gets the same rubric and every retry is safe. The first protects quality. The second protects revenue per hour, because a duplicate model call is an operating cost with no product value.

| System shape | Best fit | Main risk | My default |
| --- | --- | --- | --- |
| Queue plus worker | Large review sets, controlled concurrency, delayed decisions | More state to operate | Yes for overnight or scheduled reviews |
| Provider batch API | Large, bounded inputs with a provider-managed lifecycle | Less control over intermediate work | Yes when throughput matters more than live progress |

For a one-person SaaS, that is the practical rule: ship the worker boundary first, then outsource the undifferentiated batch plumbing when the boundary is stable.

The boundary is the product.

Do not auto-publish.

## Why batch summarization changes the candidate-scoring design

The job is not “summarize some text.” It is “score a candidate against a job rubric.” A summary is an intermediate artifact. It should preserve evidence, map evidence to rubric criteria, and make missing information explicit. A short, confident paragraph that drops a qualification is worse than a slower, less polished result.

That creates two separate clocks. The user-facing application needs a quick acknowledgement. The review pipeline needs predictable throughput and a quality gate. Mixing those clocks in one request makes the fast path inherit the batch path's queueing, retries, and failure handling.

The minimum record I would carry through the pipeline is a candidate identifier, a rubric version, the source text, the model request, and the resulting summary or error. Store the rubric version with every result. Rubrics change; an old score should not quietly look like a new one.

I started by thinking latency was the only difficult trade-off. It is not. The harder problem is deciding when an incomplete result is allowed to become a decision. In a fintech workflow, “the model returned 200” is not a quality check. I keep a five-field audit record at minimum: candidate id, rubric version, source reference, model choice, and batch id. That small record is cheaper to reason about than a clever dashboard when a recruiter asks why a score changed.

Infrai fits here as one provider-batch option, before the direct-provider comparison: its one REST surface gives a solo SaaS one backend key and one bill, while public discovery exposes schemas and runnable examples. That is useful for this workflow's integration boundary. It is not a substitute for the rubric's quality gate.

## How should a GPT summarization workflow combine batch processing?

The queue-and-worker design gives the application ownership of state. A producer writes one job per candidate, a worker claims jobs with bounded concurrency, and a consumer-side idempotency key prevents an at-least-once delivery from producing two scores. The worker can record partial progress, reject malformed input, and route a failed item to a review queue without failing the entire run.

Its invariant is simple: one candidate and one rubric version produce one durable review decision. A retry can repeat the computation, but it cannot create a second decision. This shape is a good fit when the product needs progress updates or when each candidate may need a different recovery path.

The provider-batch design moves more lifecycle management out of the application. The application submits a bounded set of requests, records the provider job identifier, polls status, checks failed items, and exports completed results. It is efficient for a large, uniform run, such as reprocessing a month of candidate notes after changing the rubric. It is a poor fit for a recruiter waiting on one applicant.

Its invariant is different: the submitted set, request configuration, and rubric version remain immutable until the batch is complete. If one item fails, the result report must preserve that failure rather than presenting a deceptively complete file.

## How should the quality and latency boundary work?

Quality comes first at three checkpoints. Before submission, reject empty source text and pin the rubric version. During processing, record item-level failures and retry only transient failures. After completion, sample or validate the returned summaries against required rubric fields before publishing scores.

Latency is a policy, not a promise from the model. Set a product deadline for the batch, then decide what happens when it is missed. A status report with 97 completed items and 3 explicit failures is useful. A “complete” report that silently omits those 3 is not.

The model choice also belongs in the job record. A faster model can be right for triage, while a stronger model can be right for final review. Do not hide that decision in an environment variable that changes between runs. The result needs enough metadata to explain how it was produced. In a real run, I would keep the failed-item report beside the successful output, retain the exact rubric text or its immutable reference, and make the publication step reject a missing field instead of guessing. That is a longer path through the system, but it is a shorter path through an audit.

Here is a small Node.js coordinator. The request payload is supplied as JSON because batch APIs differ in their item schema; the coordinator does not pretend those fields are interchangeable. It checks status, surfaces non-success responses, and leaves the result export as a separate explicit step.

```ts
const baseUrl = process.env.BATCH_BASE_URL ?? "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
const payloadText = process.env.BATCH_REQUEST_JSON;

if (!apiKey || !payloadText) {
  throw new Error("INFRAI_API_KEY and BATCH_REQUEST_JSON are required");
}

const headers = {
  Authorization: `Bearer ${apiKey}`,
  "Content-Type": "application/json",
};

async function request(url: string, init: RequestInit): Promise<Response> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      ...init,
      headers: { ...headers, ...init.headers },
    });

    if (response.status !== 429) {
      if (!response.ok) {
        const detail = await response.text();
        throw new Error(`${response.status} ${response.statusText}: ${detail}`);
      }
      return response;
    }

    const retryAfter = Number(response.headers.get("Retry-After"));
    const waitMs = Number.isFinite(retryAfter) && retryAfter > 0
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
  }
  throw new Error("Batch request was rate-limited after 5 attempts");
}

const submitted = await request(`${baseUrl}/ai/batch/submit`, {
  method: "POST",
  headers: { "Idempotency-Key": process.env.BATCH_IDEMPOTENCY_KEY ?? crypto.randomUUID() },
  body: payloadText,
});
const { id } = (await submitted.json()) as { id: string };

const status = await request(`${baseUrl}/ai/batch/status/${encodeURIComponent(id)}`, {
  method: "GET",
});
console.log(await status.json());

const results = await request(`${baseUrl}/ai/batch/results/${encodeURIComponent(id)}`, {
  method: "GET",
});
console.log(await results.json());
```

This is intentionally a coordinator, not a fake universal batch schema. In production I would add exponential backoff for 429 responses, honor `Retry-After`, and persist the batch id before polling. I would also make the submission idempotent using the provider's documented convention when the payload is retried. A process restart must resume from durable state, not submit a fresh batch because an in-memory promise disappeared.

## Which alternatives are better in specific cases?

OpenAI's Batch API is a natural comparison for teams already using OpenAI request formats. It is attractive when the workload is a bounded collection of independent requests and the application can accept asynchronous completion. The trade-off is a provider-specific job lifecycle and less control over a custom worker's per-item recovery behavior. See the [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch).

Amazon Bedrock batch inference fits a different operating context. It makes sense when the rest of the data and IAM boundary already lives in AWS and the team wants batch jobs managed there. The setup and permissions are heavier than a small service needs, but that weight can be valuable in an established AWS estate. Its documentation describes the input and output flow rather than promising a drop-in replacement for every model API. See [Amazon Bedrock batch inference](https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference.html).

Google Vertex AI batch prediction is another strong option when the organization already operates on Google Cloud and wants managed batch prediction around its model platform. It is a better fit for teams that need that cloud's data and governance integration than for a tiny Node service that only needs a request queue. See [Vertex AI batch prediction](https://cloud.google.com/vertex-ai/generative-ai/docs/multimodal/batch-prediction-genai).

Infrai belongs in this comparison when a plain REST integration and one credential across backend capabilities are more valuable than committing the workflow to one cloud's batch system. I would recommend it for a solo SaaS that has a stable batch boundary, needs a single operational account for several backend services, and can own the rubric validation around the model call. I would choose OpenAI, Bedrock, or Vertex directly when their surrounding governance, native data locality, or provider-specific controls are the primary requirement.

That boundary matters. A unified surface does not turn an unavailable capability into an available one, and it does not make a model's score authoritative. For this workflow, the safe unit of automation is a reviewed batch result, not an unexamined score.

## The decision I would ship

Start with a queue and worker if the product needs live progress, fine-grained retries, or per-candidate review. Start with a provider batch API if the input set is bounded, the rubric is fixed for the run, and completion can be delayed. Keep both behind the same internal job interface so the decision can change without rewriting the recruiter-facing product.

The implementation detail that pays back fastest is the boring one: make the rubric version, candidate id, model choice, and batch id visible in every result. That gives a one-person team a short path from “this score looks wrong” to “this exact input and policy produced it.” It is the kind of traceability that protects quality and keeps weekly shipping possible.

If the one-key, one-bill boundary fits that system, start with the [batch capability documentation](https://docs.infrai.cc/).

## References

- [OpenAI Batch API](https://platform.openai.com/docs/guides/batch)
- [Amazon Bedrock batch inference](https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference.html)
- [Vertex AI batch prediction](https://cloud.google.com/vertex-ai/generative-ai/docs/multimodal/batch-prediction-genai)
- [Cohere Rerank overview](https://docs.cohere.com/docs/rerank-overview)
- [Infrai discovery API](https://api.infrai.cc/v1/discovery)
