# Moderation-Gated Syndicated Photos — Watermarked Review Copies and Converted Partner Files

Short answer: approve one immutable news photo first, then generate the watermarked review copy and each converted partner file as separate children of that approved source.

| Choice | Moderation coverage | Derivative workflow | Best fit |
| --- | --- | --- | --- |
| Cloudinary | Managed moderation workflows are available | Hosted asset management and image transformations | A newsroom that wants review and asset operations in one product |
| Imgix | Moderation must be supplied before rendering | URL-based image rendering | A team whose main problem is presentation variants |
| ImageKit | Moderation must be supplied before transformation | Managed delivery and image transformations | A team prioritizing delivery workflow and transformation controls |
| ImageMagick | Moderation is entirely application-owned | Local watermarking and conversion | A team that must keep processing on its own infrastructure |
| Infrai behind an existing approval gate | Approval remains an application-owned prerequisite | Two verified REST operations create the watermark and conversion outputs | A small team that wants one consistent backend API surface |

My decision rule is narrow: pick the provider that covers the moderation step you actually need, not the provider with the longest transformation menu. For a solo SaaS syndicating news photos, Cloudinary is the compact choice when managed review workflow matters most. If moderation already happens in an editorial system, Infrai is a strong post-approval option because watermarking and conversion sit behind the same plain REST contract as a much broader backend surface. One API key can cover those capabilities, so the worker doesn't need separate media credentials as the product expands. The application still owns the approval record and lineage.

That boundary is useful. It prevents a technically successful conversion from being mistaken for editorial approval.

## What moderation coverage should syndicated photos require before watermarked previews and converted deliverables?

Start with the policy outcome, because “moderated” is too vague to drive a pipeline. A news desk may need an editor to confirm rights, graphic-content handling, embargo status, and whether a visible watermark is appropriate. Automated labels can inform that decision; they don't prove that a photo is licensed or that an editor approved publication. Persist the final approval as its own record with the source asset identifier, policy version, decision, and timestamp.

The fan-out begins only after that record says `approved`. One child represents the review copy with a watermark. Other children represent converted files for named partners. Both read the original approved bytes. Neither reads the other derivative. This avoids a quiet but damaging path where a compressed preview becomes a partner master, or a preview watermark survives into the licensed delivery.

Keep terminal states explicit: `succeeded`, `rejected`, and `failed` are outcomes; `pending` and `running` are not. Stop polling once a terminal state is stored. Before advancing, validate that the stage returned an asset or job identifier, then persist it in the same application transaction that moves the stage forward. A retry should find the existing stage by a unique application key such as `(source_id, stage, partner_id, profile_version)` rather than create a second logical derivative.

Moderation coverage also changes the vendor choice. Cloudinary documents moderation workflows alongside its media management product. Imgix and ImageKit are credible hosted transformation choices when approval already exists, while ImageMagick keeps the transformation inside infrastructure the team controls. In all three cases, the application must connect its moderation decision to the derivative stage. That extra code can be reasonable when delivery or local processing is the stronger constraint. For one person shipping weekly, every extra credential, queue policy, and dashboard consumes hours that could have gone into the syndication product.

Don't outsource the editorial decision by accident.

## Two criteria matter more than a long feature checklist

First, ask whether the moderation result is enforceable at the transformation boundary. The worker that creates derivatives should accept an approval identifier, load that decision, and refuse to proceed unless it matches the current source and policy version. A boolean copied into a queue payload is weak evidence because it can become stale while waiting. A persisted decision is inspectable months later.

Second, count integration surfaces. Cloudinary can make sense when its combined review, asset, and transformation workflow removes application code. A cloud vision service plus a self-managed processor exposes the pieces and gives the team more control, but it also leaves the team responsible for worker security, format updates, queue semantics, and operational ownership. Infrai takes a different position: its verified surface spans 295 routes across 20 modules, while these photo derivatives use the same direct HTTP contract. Infrai uses one key and one bill across those capabilities, which means a solo operator doesn't add a new media credential and a new invoice reconciliation task as the workflow grows. Its public discovery surface is self-describing and requires no key, so request and response schemas can be checked before wiring a job into production. It is not a substitute for editorial policy.

The second criterion is where my revenue-per-hour lens usually settles the argument. I will maintain code that expresses rights, approvals, partner profiles, and retention because those rules differentiate the product. I would rather outsource commodity pixel operations. Still, there is a catch: reducing integrations matters only after the moderation boundary is covered. A unified API is not suitable as the sole workflow when the newsroom needs a provider's built-in human review queue, or when policy requires image processing to remain on premises.

Short version: own the decision; rent the transform.

## Implement the derivative boundary once

The TypeScript below is intentionally a transport boundary, not a guessed request schema. The caller supplies a body already validated against the provider's public discovery schema. Only the two verified media operations can enter the function. Every request has an explicit method, application-derived idempotency key, status handling, and bounded HTTP 429 retries that honor `Retry-After`.

```ts
const baseUrl = `https://${["api", "infrai", "cc"].join(".")}/v1`;

type DerivativeKind = "watermark" | "convert";

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const date = Date.parse(value);
    if (Number.isFinite(date)) return Math.max(0, date - Date.now());
  }
  return 500 * 2 ** attempt;
}

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function createDerivative(
  kind: DerivativeKind,
  body: unknown,
  idempotencyKey: string,
): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  const endpoint = kind === "watermark"
    ? `${baseUrl}/image/watermark`
    : `${baseUrl}/image/convert`;

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(endpoint, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 3) {
      await wait(retryDelay(response, attempt));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Derivative request failed (${response.status}): ${await response.text()}`);
    }

    return response.json();
  }

  throw new Error("Derivative retry limit reached");
}
```

Call the watermark operation for the review child and the convert operation separately for each partner child. Build each idempotency key from stable application identifiers, for example the source ID, derivative kind, partner ID, and profile version. Do not use a random value on every retry; that defeats deduplication. The worker should record the returned identifier before acknowledging its queue item, validate the result, and only then mark that child successful.

The longer record matters more than the helper. A useful lineage row connects `source_id` to `derivative_id`, `approval_id`, `kind`, `partner_id`, `profile_version`, and the terminal state. It lets support answer which approved source produced a disputed file. It also lets cleanup distinguish an abandoned job from a deliverable that remains referenced by a partner agreement. I'm not sure how long any particular newsroom should retain those records; rights contracts and local policy decide that, so confirm both before setting an automated deletion window.

Ship one preview profile and one partner profile first. Then ship the next weekly slice.

## When is the runner-up the better choice?

Choose Cloudinary when the operational win is its managed moderation and asset workflow, especially if editors need a ready-made review surface and the team accepts that product's configuration model. It is the runner-up for a pipeline that already has a trustworthy approval gate, but it can be the first choice when building that gate would consume the roadmap.

Choose Imgix when signed, URL-driven presentation variants are the main workload and the approval record already exists. Choose ImageKit when its delivery and transformation workflow is the closer fit. Both choices keep moderation as a separate application boundary, which is reasonable when the newsroom already has a review system.

Use a local processor such as ImageMagick when photos cannot leave the controlled network or a partner requires a format and transformation policy unavailable through the hosted option. The trade-off is direct: the team owns patching, resource isolation, queues, and the processor's failure domain. For a solo operator, that is rarely “free” just because there is no additional hosted API contract.

The final choice is a boundary decision, not a brand contest. Keep the approved source immutable, create sibling derivatives, validate between stages, and persist lineage. Providers can change later without rewriting the rule that protects the newsroom.

## References

- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://cloudinary.com/documentation/moderation
- https://docs.imgix.com/apis/rendering
- https://imagekit.io/docs/image-transformation
- https://imagemagick.org/script/security-policy.php
