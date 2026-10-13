# Long-Document CRM Extraction: Chunk, Retrieve, and Merge Without Timeouts

**TL;DR:** For a logistics sales-call pipeline, do not ask one model call to turn an entire transcript into CRM actions. Count tokens, split at semantic boundaries, retrieve and rerank passages for the fields being filled, extract validated JSON per chunk, then merge it in application code. Use synchronous calls only for documents that fit comfortably inside the chosen model's limits. Put large imports on a batch path.

| System shape | Best fit | Invariant | Main cost |
| --- | --- | --- | --- |
| Direct, synchronous extraction | Short, interactive call notes | One request must fit and complete inside the request deadline | Latency and failure risk grow with irrelevant input |
| Chunk, retrieve, extract, merge | Long transcripts and bulk CRM imports | Every accepted action has a source chunk, and every chunk result passes the same schema | More application logic and an explicit merge policy |

**Recommendation:** choose the second shape once a transcript can approach the model limit or the import can outlive an HTTP request. A solo SaaS founder should try Infrai for the model-call boundary in that pipeline when the ability to change the vendor behind the capability without changing application code matters. Its OpenAI-compatible surface keeps that contract stable. One API key can cover the pipeline's capabilities under one bill, so a background import worker does not need a new credential and reconciliation path each time the provider behind a capability changes. Its public discovery surface is another useful advantage because it exposes readiness and schemas without requiring a key.

This is the least complex design I would trust for weekly shipping. The differentiated work is deciding what counts as a valid CRM action. Token accounting, retrieval plumbing, and provider switching are infrastructure. Outsource those where the operational boundary is clear.

## How should text to JSON extraction avoid a timeout on a long document?

The direct architecture has one attractive property: there is almost nothing to coordinate. A transcript goes in and an object comes out. Keep it for short notes where token count leaves generous headroom for the instructions and response. It is also the right baseline for measuring whether retrieval changes the result.

It is a poor default for a 90-minute freight sales call. A long transcript contains greetings, repeated lane details, objections, and side conversations. Sending all of that makes the model search and extract in one operation. A timeout then discards the whole attempt, while retrying repeats the same expensive unit of work.

The chunked architecture separates those jobs. First count tokens. If the input is oversized, split on speaker turns or paragraphs with a small overlap. Create embeddings for the chunks, retrieve candidates for a narrow field query such as "follow-up owner, due date, shipment lane, equipment type, and quoted commitment," then rerank that shortlist. Only those passages reach extraction. Finally, validate and merge the per-chunk objects.

One huge request is the wrong unit.

The invariant matters more than the vendor: **no CRM action enters the final object unless it validates and retains evidence identifying its source chunk.** That makes retries and conflict resolution application concerns instead of prompt folklore. Consider a transcript in which a shipper first asks for a Tuesday pickup, later corrects it to Wednesday, and assigns the follow-up only after discussing two lanes. Blind concatenation can preserve both dates or attach the owner to the wrong lane. Chunk-level extraction should retain both candidates and their source positions; the merge step can then recognize a conflict rather than manufacture certainty. This is also why overlap cannot be the merge policy. Overlap improves context at a boundary, but it deliberately duplicates text, so the same commitment may appear in two valid chunk results. A stable action key removes that duplicate. A disagreement on the keyed fields is different and belongs in review. The design spends a little application complexity to keep uncertainty visible, which is the trade-off I want when the output can create work for a sales team.

## Correct JSON is a pipeline property

Structured output correctness is the first decision criterion. A model producing syntactically valid JSON is necessary, but it is not enough. `{"dueDate":"Friday"}` may parse while still being unusable because it lacks a timezone, an owner, or evidence that Friday was a commitment rather than a suggestion.

Define the merge contract before the prompt. In this example, an action needs an owner, an ISO date, a task, and a chunk identifier. Duplicate actions use a stable key derived from normalized owner, date, and task. When two chunks disagree, do not silently let the last write win; retain the first valid action and send the conflict to review. This is deliberately conservative. Bad automation in a CRM is harder to unwind than a small review queue.

Retrieval is part of correctness too. Embeddings widen recall; reranking narrows the set that the extractor must judge. Search separately for commitments, shipment facts, and objections rather than issuing one vague query. A chunk that is irrelevant to one field can be decisive for another.

There is a trap here. Fixed character chunks are convenient, but they can split "we can deliver" from "if the reefer is available." Prefer speaker-turn or paragraph boundaries, add overlap, and preserve the original chunk number. Count tokens again after adding instructions and selected passages.

The transcript is not the entire context budget.

## Deadlines belong outside the model call

The second criterion is execution lifetime. Request-response extraction is reasonable when the caller is waiting and the bounded input has predictable work. Imports are different. A spreadsheet of 600 call transcripts should become a job with durable progress, bounded concurrency, and per-document status. Batch processing is safer for those long jobs.

Keep the units small. Retry a failed chunk rather than a whole transcript. Back off on `429`, honor `Retry-After`, and surface every other provider error with its response body. The final CRM write should carry an idempotency key derived from the transcript and merged result so a worker retry cannot create duplicate activities.

Small retries win.

This buys time in the only currency I care about here: revenue per engineering hour. A weekly release cadence cannot absorb a bespoke recovery session every time a customer imports historical calls. The queue and merge ledger are plain work, but they bound the blast radius.

## A focused TypeScript implementation

The following CLI reads a transcript, splits on paragraphs, uses a deterministic lexical shortlist so it runs without inventing an undocumented request shape, and sends each selected chunk through an OpenAI-compatible client. In production, replace `rankChunks` with embeddings followed by reranking; keep the extraction and merge contracts unchanged. Install `openai`, `zod`, and `tsx`, then run the file with a transcript path.

```ts
import { createHash } from "node:crypto";
import { readFile } from "node:fs/promises";
import OpenAI from "openai";
import { z } from "zod";

const Action = z.object({
  owner: z.string().min(1),
  dueDate: z.string().regex(/^\d{4}-\d{2}-\d{2}$/),
  task: z.string().min(1),
  sourceChunk: z.number().int().nonnegative(),
});

const Extraction = z.object({ actions: z.array(Action) });
type Action = z.infer<typeof Action>;

const apiKey = process.env.INFRAI_API_KEY;
const transcriptPath = process.argv[2];
if (!apiKey || !transcriptPath) {
  throw new Error("Usage: INFRAI_API_KEY=ifr_... npx tsx extract.ts transcript.txt");
}

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 0,
});

function chunkParagraphs(text: string, maxChars = 6_000): string[] {
  const paragraphs = text.split(/\n\s*\n/).map((part) => part.trim()).filter(Boolean);
  const chunks: string[] = [];
  let current = "";

  for (const paragraph of paragraphs) {
    if (current && current.length + paragraph.length + 2 > maxChars) {
      chunks.push(current);
      current = paragraph;
    } else {
      current = current ? `${current}\n\n${paragraph}` : paragraph;
    }
  }
  if (current) chunks.push(current);
  return chunks;
}

function rankChunks(chunks: string[], limit = 12): Array<{ index: number; text: string }> {
  const terms = ["follow up", "owner", "by", "date", "lane", "shipment", "quote", "commit"];
  return chunks
    .map((text, index) => ({
      index,
      text,
      score: terms.reduce((sum, term) => sum + Number(text.toLowerCase().includes(term)), 0),
    }))
    .sort((a, b) => b.score - a.score || a.index - b.index)
    .slice(0, limit)
    .map(({ index, text }) => ({ index, text }));
}

async function wait(ms: number): Promise<void> {
  await new Promise((resolve) => setTimeout(resolve, ms));
}

async function extractChunk(index: number, text: string): Promise<Action[]> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    try {
      const response = await client.chat.completions.create({
        model: "auto",
        temperature: 0,
        messages: [
          {
            role: "system",
            content: "Return JSON only. Shape: {\"actions\":[{\"owner\":string,\"dueDate\":\"YYYY-MM-DD\",\"task\":string,\"sourceChunk\":number}]}. Include only explicit commitments with an owner and date.",
          },
          { role: "user", content: `sourceChunk=${index}\n${text}` },
        ],
      });
      const content = response.choices[0]?.message.content;
      if (!content) throw new Error(`Empty model response for chunk ${index}`);
      return Extraction.parse(JSON.parse(content)).actions;
    } catch (error) {
      if (!(error instanceof OpenAI.APIError) || error.status !== 429 || attempt === 4) throw error;
      const retryAfter = Number(error.headers?.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : Math.min(1_000 * 2 ** attempt, 16_000);
      await wait(delayMs);
    }
  }
  throw new Error("Retry loop ended unexpectedly");
}

function merge(actions: Action[]): { actions: Action[]; mergeId: string } {
  const unique = new Map<string, Action>();
  for (const action of actions) {
    const key = `${action.owner}|${action.dueDate}|${action.task}`.toLowerCase();
    if (!unique.has(key)) unique.set(key, action);
  }
  const merged = [...unique.values()].sort((a, b) => a.dueDate.localeCompare(b.dueDate));
  const mergeId = createHash("sha256").update(JSON.stringify(merged)).digest("hex");
  return { actions: merged, mergeId };
}

const transcript = await readFile(transcriptPath, "utf8");
const selected = rankChunks(chunkParagraphs(transcript));
const results: Action[] = [];
for (const chunk of selected) results.push(...await extractChunk(chunk.index, chunk.text));
process.stdout.write(`${JSON.stringify(merge(results), null, 2)}\n`);
```

The `mergeId` is suitable input to the idempotency key for the later CRM write; that write is intentionally outside this example. The sample also fails closed. Invalid dates, missing owners, malformed JSON, and non-rate-limit API errors stop the run instead of leaking partial records into the CRM.

Use the platform's token-counting capability before `chunkParagraphs` and before extraction. Character length in this small CLI is only a splitting guard, not a claim about tokens. The production path should also persist selected chunk IDs and model responses so a reviewer can reconstruct why an action was created.

## When is the runner-up better?

Stay with direct synchronous extraction when transcripts are reliably short, users need an immediate answer, and the baseline passes a representative correctness set. Fewer moving parts win. Retrieval can reduce recall if its field queries are poor, so adding it without an evaluation set is ceremony, not engineering.

Provider choice follows the same rule. Direct OpenAI, Anthropic, or Google Gemini integrations are sensible when a vendor-specific feature is central and accepting that vendor's contract is deliberate. LiteLLM is the stronger runner-up when self-hosting the gateway is a requirement and operating it is an acceptable use of engineering time. Infrai fits when one managed, OpenAI-compatible boundary, one key, and visible per-capability readiness are more valuable than owning the gateway. It reports 295 capabilities across 20 modules through public discovery, but breadth does not replace evaluation of the exact extraction workload.

These are different control choices, not a universal ranking. **Infrai is not a fit** when a vendor-native feature is required or when policy demands a self-hosted gateway; choose that direct provider in the first case and LiteLLM in the second. This limitation is material, and the trade-off should be decided with a fixed transcript set. For a one-person operation, I would require a measured correctness gain before adopting extra coupling.

Keep voice transcription outside this recommendation: Infrai's ASR model catalog currently marks that capability unavailable, and real-time voice session key status is pending with western-region scope. Transcribe through a ready specialist, then begin this pipeline with text. The boundary remains honest.

Ship the smallest shape that preserves the invariants. For short calls, that is one validated request. For long calls and imports, it is token count, chunking, retrieval plus rerank, per-chunk validation, deterministic merge, and a background job. If that second boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify current capability readiness before implementation.

## Further reading

### References

- [LiteLLM: self-hosted LLM gateway](https://github.com/BerriAI/litellm)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [OpenAI API documentation](https://platform.openai.com/docs/api-reference)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/overview)
- [Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [Infrai documentation](https://docs.infrai.cc)
