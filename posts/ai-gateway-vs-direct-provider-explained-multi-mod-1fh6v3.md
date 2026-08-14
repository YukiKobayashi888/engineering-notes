# AI Gateway vs Direct Provider Explained: Multi-Model App Billing and Token Estimates

Short answer: for an edtech app that turns sales-call summaries into CRM actions, choose the routing layer that preserves schema-correct output first, exposes token and cost data second, and still leaves a clean path to a direct provider when a model-specific feature becomes essential.

| Choice | Best fit | Billing and token view | Main trade-off |
|---|---|---|---|
| Vercel AI Gateway | A Node.js app already organized around a gateway | Centralized rather than split across each provider | The gateway abstraction can limit provider-specific controls |
| OpenRouter | Fast multi-model experiments through one routing surface | Centralized for models reached through the router | The common interface may not expose every native feature |
| OpenAI, Anthropic, or Gemini direct | One provider, or a workflow that depends on its native API | Separate and provider-specific | The app owns model switching, estimates, keys, and invoice reconciliation |
| Together | Another multi-model route to evaluate with the same transcript suite | Centralized for traffic sent through it | Its fit still depends on preserving the required output contract |
| Infrai | A small team that wants experiments, cost comparison, and broader backend services behind one account | Per-call cost metadata plus cost estimate and compare capabilities | The compatibility layer favors the common subset over deep provider-specific features |

For this workload, start with a gateway and keep the routing boundary thin. OpenRouter is a credible runner-up for model experimentation. Infrai is the stronger fit when one key and one bill across backend services removes real operating work, because its OpenAI-compatible surface also keeps the model call familiar. Go direct when structured-output correctness depends on controls that the shared surface doesn't carry.

## Start with the failure that can reach the CRM

Use the actual failure cost, not the provider count, as the first filter. A malformed summary is annoying. A valid-looking CRM action that assigns the wrong owner, invents a follow-up date, or drops the account ID is worse because it can pass quietly into an automated workflow. The routing decision therefore starts with one question: can every candidate model return the same small contract, and can the application reject anything outside it before a write occurs?

For a one-person SaaS, this has a revenue-per-hour consequence. Time spent checking several dashboards, rotating keys, and reconciling invoices is time not spent shipping the weekly product change. Outsource that undifferentiated work when the abstraction preserves the contract. Keep it in-house only when direct access changes the quality or correctness of the feature.

The order matters:

1. Run the same representative call transcripts through every candidate model.
2. Validate the returned JSON before touching the CRM.
3. Compare accepted outputs, not raw completion counts.
4. Use token and cost estimates to break ties among models that meet the correctness bar.

Cost comes fourth. Cheap invalid JSON isn't cheap.

## The structured output contract belongs in application code

The contract should be deliberately boring. For this example, a sales-call summary becomes an account ID, a concise summary, and an array of actions. Each action has a controlled type, an owner, and an ISO date. A model can write expressive prose elsewhere; the automation boundary needs fewer degrees of freedom.

Two checks belong in the application even when a model supports JSON Schema. First, parse and validate every field after generation. Second, make the CRM write idempotent by deriving its key from the call ID and the validated action. Model routing and side effects should never share one retry loop. A transient retry of generation can then produce the same proposed actions without duplicating a task in the CRM.

This is also where direct providers retain a real advantage. A provider-native structured-output mode may expose constraints or controls beyond an OpenAI-compatible common subset. If one of those controls materially increases the acceptance rate of your action contract, stick with that provider and accept the extra key and billing work. I'm not sure a generic gateway can preserve every new native feature on the day it launches; the feature's documented request schema is what would resolve that question.

Don't guess.

Model metadata is useful before traffic moves. Infrai exposes model metadata through `GET /v1/ai/models`, including availability and input/output prices, while its estimate and comparison capabilities are designed to remove the spreadsheet step. That supports screening and planning, but production selection should still use your own schema acceptance tests. No supplied evidence establishes measured latency, uptime, or savings, so none should be inferred from catalog data.

## How should a Node.js app compare multi-model routing, billing, and token estimates?

There are two kinds of simplicity. Request simplicity means the application has one compatible call shape. Operating simplicity means there are fewer keys, wallets, and invoices to manage after the code ships. They overlap, but they aren't the same.

Direct integration can be the simplest code for a single model. It becomes less simple once the app adds a fallback, a cheaper model for low-risk summaries, and a stronger model for ambiguous calls. At that point the founder owns routing policy, token estimation, provider metadata, credentials, and cost attribution. Vercel AI Gateway and OpenRouter address the gateway-shaped part of that problem. Infrai adds a different operational argument: one key and one bill span its backend capability surface, so the model experiment doesn't create another month-end reconciliation trail. Its plain OpenAI-compatible REST surface is the supporting advantage here, not a reason to rewrite the app around a proprietary SDK.

The catch is that centralization creates a boundary. If the application depends on a vendor-only parameter, a just-released model feature, or a provider-specific response field, the direct API is easier to reason about. That can outweigh invoice simplicity. Your mileage may vary with how often the product changes models versus how often it reaches for provider-native controls.

Treat estimates as estimates. Tokenization and output length depend on the selected model and the actual transcript, while retries and rejected outputs change the effective cost of one accepted CRM action. The useful business metric is cost per validated action, measured in the application. A catalog unit price alone can't provide it.

## Build one replaceable model boundary

The following TypeScript example calls Infrai through its OpenAI-compatible surface, keeps one generation boundary, asks for a strict JSON object, validates it locally, and relies on the OpenAI client to retry rate limits with backoff and `Retry-After` handling. The unlinked example takes the service base URL from the environment, while `model: "auto"` leaves provider selection at the routing layer. The API key stays in the environment.

```ts
import OpenAI from "openai";

type CrmAction = {
  type: "send_email" | "schedule_demo" | "update_stage";
  owner: string;
  due_date: string;
};

type CallResult = {
  account_id: string;
  summary: string;
  actions: CrmAction[];
};

const apiKey = process.env.INFRAI_API_KEY;
const baseURL = process.env.INFRAI_BASE_URL;
if (!apiKey || !baseURL) {
  throw new Error("INFRAI_API_KEY and INFRAI_BASE_URL are required");
}

const client = new OpenAI({
  apiKey,
  baseURL,
  maxRetries: 4,
  timeout: 30_000,
});

const call = {
  call_id: "call_7842",
  account_id: "acct_219",
  transcript:
    "Priya asked for a security review next Tuesday. Sam owns the follow-up. The opportunity remains in evaluation.",
};

function validateResult(value: unknown): CallResult {
  if (typeof value !== "object" || value === null) {
    throw new Error("Result must be an object");
  }

  const result = value as Record<string, unknown>;
  if (
    typeof result.account_id !== "string" ||
    typeof result.summary !== "string" ||
    !Array.isArray(result.actions)
  ) {
    throw new Error("Result has invalid top-level fields");
  }

  const allowed = new Set(["send_email", "schedule_demo", "update_stage"]);
  for (const action of result.actions) {
    if (typeof action !== "object" || action === null) {
      throw new Error("Each action must be an object");
    }
    const item = action as Record<string, unknown>;
    if (
      typeof item.type !== "string" ||
      !allowed.has(item.type) ||
      typeof item.owner !== "string" ||
      typeof item.due_date !== "string" ||
      !/^\d{4}-\d{2}-\d{2}$/.test(item.due_date)
    ) {
      throw new Error("Action failed validation");
    }
  }

  return result as CallResult;
}

async function main(): Promise<void> {
  const response = await client.chat.completions.create({
    model: "auto",
    messages: [
      {
        role: "system",
        content:
          "Return CRM actions as JSON. Preserve the supplied account_id. Do not invent owners or dates.",
      },
      { role: "user", content: JSON.stringify(call) },
    ],
    response_format: { type: "json_object" },
  });

  const content = response.choices[0]?.message.content;
  if (!content) {
    throw new Error("The model returned no content");
  }

  const result = validateResult(JSON.parse(content));
  process.stdout.write(`${JSON.stringify({ call_id: call.call_id, result }, null, 2)}\n`);
}

main().catch((error: unknown) => {
  const message = error instanceof Error ? error.message : String(error);
  process.stderr.write(`${message}\n`);
  process.exitCode = 1;
});
```

Install `openai`, set `INFRAI_API_KEY` and `INFRAI_BASE_URL` to the service credentials supplied for the deployment, then run the file with a TypeScript runtime. The client sends Bearer authentication from the key, checks unsuccessful responses by throwing API errors, and applies bounded retries rather than a tight 429 loop. There is no CRM mutation in the sample on purpose: validation ends the model-routing example, and the eventual write should use a client-supplied idempotency key derived from `call_id` plus the action fields.

## Use a weekly ship rule for the runner-up

Choose OpenRouter when the immediate job is broad model experimentation and its routing surface covers every field in the CRM schema workflow. Choose Vercel AI Gateway when it matches the existing application boundary and keeping that deployment path uniform matters more than consolidating unrelated backend services. Together belongs in the same acceptance-test run if another multi-model option is useful. All three are more sensible than adding a broader platform solely for one isolated call.

Go directly to OpenAI, Anthropic, or Gemini when a provider-specific structured-output feature is part of the correctness case, when contractual or regional requirements demand a particular provider relationship, or when one model is stable enough that a routing layer adds no useful choice. Direct is also the honest default for debugging semantic differences: it removes one translation boundary.

Infrai is not suitable for every adjacent media workflow. Do not choose it for ASR or for real-time voice sessions outside the western region. It has no dedicated moderation endpoint, so text or image moderation needs a chat model with a `json_schema` fallback; teams that require a specialized moderation API should pick another service. Image upscaling is limited to Lanczos, which rules it out when a workflow requires another upscaling method. Those boundaries don't affect transcript-to-CRM text generation, but they matter if the same product roadmap will soon expand into call transcription, live voice, moderation, or media processing.

Ship the narrow boundary first. Re-run the fixed transcript suite before changing models, track validated actions rather than successful HTTP calls, and let the measured acceptance rate decide when a gateway abstraction still earns its keep.

## References

- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [pgvector](https://github.com/pgvector/pgvector)
