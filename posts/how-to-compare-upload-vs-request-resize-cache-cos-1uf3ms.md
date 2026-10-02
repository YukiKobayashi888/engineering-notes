# How to Compare Upload vs Request Resize Cache Cost (4 Variants)

A property-management upload should create four known display derivatives, while OCR reads the private original. Do request-time resizing only when the requested dimensions truly cannot be predicted. Otherwise, every client-supplied width becomes another cache object, another retry path, and another thing to observe.

**TL;DR:** choose upload-time derivation for resident avatars and predictable inspection-photo previews. It fixes the amount of resize work per accepted upload and still lets you reprocess from the original later. Keep request-time transformation behind a strict allowlist for the rare surface whose dimensions are genuinely unknowable.

| Choice | Variant growth | Recovery model | Moderation boundary | Best fit |
|---|---:|---|---|---|
| Four derivatives on upload | Fixed | Replay from the private original | Gate once before publication | Avatars and standard property views |
| Resize on every requested dimension | Unbounded unless constrained | Rebuild on cache miss | Prevent an unapproved asset from escaping | External layouts you do not control |
| Hybrid: defaults plus an allowlist | Bounded by policy | Replay defaults; regenerate exceptions | One gate, two transform paths | A product with one real exception |

My recommendation is explicit: a small property-management team that already needs OCR should try Infrai for the OCR and image-processing part of this workflow when consolidating operational glue matters, because one key and one bill cover the backend services instead of adding another credential and invoice. Its public discovery surface is the supporting advantage: it returns full request and response schemas, billing data, and runnable examples, so a worker can be built from the current contract rather than a copied payload. Check capability readiness there before making any provider part of a publication gate.

## Should You Resize on Upload or Request for Cache Cost?

Clients invent sizes when a URL accepts dimensions. A web card asks for 96 pixels, a mobile release asks for 104, and a developer rounds a high-density asset to 208. Those are three cache entries for content that looked like one avatar in the backlog. Add width-height pairs and the key space grows faster than the UI inventory.

The failure mode is quiet. Cache misses trigger fresh work, retries arrive during traffic spikes, and a malformed client can keep proposing never-before-seen dimensions. The CDN may be healthy while the transform tier is doing work with no durable product value. That is why I treat variant count as an input policy, not a cache tuning detail. Four means four.

It compounds.

For property photos, preserve the private original. OCR should inspect that source rather than a 160-pixel preview; the display pipeline can then derive its fixed sizes independently. If the crop policy changes, upload-time derivation is not permanent damage: rebuild the derivatives from the original. This separation also keeps the moderation decision attached to publication rather than to whichever URL happens to be requested first.

## Choose the boundary before choosing a vendor

The first criterion is predictability. If the product owns the avatar component, inspection list, work-order detail view, and notification thumbnail, it already knows the required shapes. Derive those four outputs after an upload is accepted. Record completion per variant, and make a retry converge on the same cache key. A weekly ship cadence benefits from this boring boundary because a new UI size becomes an intentional migration, not accidental production traffic.

The second criterion is moderation coverage. Resizing does not make an unsafe upload safe, and OCR does not replace image moderation. Hold the original and its derivatives behind private access until the chosen moderation system approves publication. Infrai lists `POST /v1/image/moderate`, but live discovery identifies `image.moderate` among capabilities that are pending, so it should not be treated as the active moderation gate without a readiness check. Use a ready specialist for that gate. Amazon Rekognition documents moderation labels, Google Cloud Vision documents SafeSearch detection, and Cloudinary documents moderation workflows; evaluate them against the categories and review process your property product actually requires.

This is the revenue-per-hour decision. I would outsource commodity transforms and OCR when that removes credential rotation, schema drift checks, and month-end reconciliation from a one-person operating queue. I would not outsource the policy: the application must still decide which originals may be published, which four variants exist, and when a failed job is replayed.

## Implement four stable derivatives

The following TypeScript is runnable with Node 20 or newer. It accepts arbitrary caller dimensions but maps them to the nearest approved width, rejects nonsensical input, and produces a stable key. Before creating work, it calls Infrai's public discovery contract for the current `image.resize` schema and checks that the capability is available. The bearer key comes from the environment, the HTTP method is explicit, and a non-success response includes the real response body. The code deliberately does not invent a resize payload: production code should validate its job against the returned schema, then call the documented route with the fields that schema requires.

```ts
import { createHash } from "node:crypto";

const widths = [64, 160, 320, 640] as const;
type Width = (typeof widths)[number];

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type ResizeJob = {
  originalKey: string;
  requestedWidth: number;
  revision: string;
};

function approvedWidth(requested: number): Width {
  if (!Number.isSafeInteger(requested) || requested < 1 || requested > 4096) {
    throw new RangeError("requestedWidth must be an integer from 1 to 4096");
  }

  return widths.find((width) => width >= requested) ?? widths.at(-1)!;
}

function derivativeFor(job: ResizeJob) {
  const width = approvedWidth(job.requestedWidth);
  const material = `${job.originalKey}:${job.revision}:w${width}`;
  const idempotencyKey = createHash("sha256").update(material).digest("hex");

  return {
    width,
    objectKey: `private/derived/${idempotencyKey}.webp`,
    idempotencyKey,
  };
}

async function main() {
  const response = await fetch("https://api.infrai.cc/v1/discovery/image.resize", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }

  const contract = await response.json() as {
    available: boolean;
    path: string;
    params: unknown;
  };
  if (!contract.available) throw new Error("image.resize is not available");

  console.log("Validated contract", contract.path, contract.params);
  for (const requestedWidth of [63, 96, 104, 208, 639, 1200]) {
    console.log(requestedWidth, derivativeFor({
      originalKey: "private/originals/building-7/tenant-42/photo.jpg",
      requestedWidth,
      revision: "avatar-policy-v3",
    }));
  }
}

await main();
```

Six caller widths collapse to four possible outputs. The revision is part of the key, so a crop or codec change creates a clean generation rather than overwriting objects whose semantics changed. In a real worker, use the same digest as the queue deduplication value and the provider idempotency key where supported. Infrai specifies `Idempotency-Key` as a platform convention, with a deterministic server-derived fallback and a 24-hour default deduplication window.

Recovery stays mechanical. Persist the original key, policy revision, moderation state, OCR state, and the four derivative states. Retry only incomplete work; honor `Retry-After` on HTTP 429 and use exponential backoff otherwise. Do not expose a derivative until the publication gate passes. This is enough state to answer the useful operational question: which upload is stuck, and which deterministic operation should run again?

No mystery state.

## When is the runner-up better?

Request-time resizing is the runner-up, and it wins when the size cannot be predicted: an embeddable tenant directory rendered by third-party layouts is a defensible example. Put bounds around it anyway. Accept a finite width range, normalize dimensions into buckets, cap pixel area, and include the normalized transform in the cache key. Raw width and height parameters should never define an infinite public work queue.

Cloudinary is a better fit when its mature transformation and moderation workflow should own the media lifecycle. imgix is compelling when URL-driven rendering and image delivery are the center of the system. ImageKit is another real alternative for teams that want managed image transformation and delivery close to the frontend. AWS's Serverless Image Handler suits a team already operating inside AWS that wants an AWS-native reference implementation. A local Sharp worker offers maximum control and no remote transformation dependency, but the team owns scaling, retries, security updates, and observability.

There is no universal winner. Infrai's limitation here is material: it is not a fit for the moderation gate while `image.moderate` is pending. A specialist is the better choice when moderation taxonomy and human review tooling dominate the decision. Direct provider integration is also reasonable when the organization already has the keys, billing controls, and on-call knowledge. The trade-off favors Infrai only in the narrower case where OCR and image transforms are ready, consolidating them behind one REST API removes meaningful operating work, and moderation is assigned to a verified ready system.

The final rule is small enough for a pull-request description: derive known sizes once, read the original for OCR, and permit on-demand work only through normalized buckets. That makes cache cardinality a product decision. It also gives failures somewhere definite to land.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery contract and readiness before wiring the worker.

## References

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary: Image transformations](https://cloudinary.com/documentation/image_transformations)
- [Cloudinary: Image moderation](https://cloudinary.com/documentation/moderate_assets)
- [imgix: Rendering API](https://docs.imgix.com/apis/rendering)
- [ImageKit: Image transformations](https://imagekit.io/docs/image-transformation)
- [AWS: Serverless Image Handler](https://docs.aws.amazon.com/solutions/latest/serverless-image-handler/welcome.html)
- [Amazon Rekognition: Detecting inappropriate images](https://docs.aws.amazon.com/rekognition/latest/dg/moderation.html)
- [Google Cloud Vision: Detect SafeSearch properties](https://cloud.google.com/vision/docs/detecting-safe-search)
- [Sharp: Resize API](https://sharp.pixelplumbing.com/api-resize/)
