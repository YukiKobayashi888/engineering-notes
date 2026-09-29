# Catalog Media Derivatives: Reusing Node.js Image Transformation Presets Safely

Short answer: store each transformation preset as versioned data, apply it in one Node.js pipeline, and keep the original asset immutable. For a marketplace catalog, that gives moderation the same pixels that buyers see while letting the storefront reuse derivatives without copying crop logic into every client.

The least complex setup is a small preset registry in your application database, an object store for originals and outputs, and a worker that runs deterministic transforms. The important design choice is not a particular image library. It is deciding which properties belong to a preset and which belong to a request.

## A choice matrix for catalog derivatives

| Approach | Good fit | Main trade-off |
| --- | --- | --- |
| Client-side transforms | Low-stakes previews and offline tools | Different browsers can produce different bytes and moderation cannot inspect the final result |
| Inline server transforms | Small catalogs with predictable traffic | Upload latency grows with every derivative and retries compete with checkout work |
| Queue-backed worker | Large catalogs, reprocessing, and moderation coverage | You must track jobs, idempotency, and stale outputs |
| CDN query transforms | Many read-only sizes with a capable image edge | Preset definitions can drift, and auditing the exact moderation input is harder |

For a headless-commerce catalog, I choose the queue-backed worker once images affect listing approval. It lets an upload event create a bounded set of named derivatives, then sends the same derivative IDs to search, product pages, and moderation. A tiny catalog can start inline and move the work behind a queue later; the preset contract should stay the same.

Ship weekly. Keep the contract boring.

## What should a reusable image transformation preset contain?

A preset is a policy, not a bag of URL parameters. Give it a stable name such as `listing-card-v2`, a schema version, an output format, a maximum width and height, a crop rule, and a background policy. Record whether the output is intended for human review, a thumbnail, or zoom. Those labels are useful when a reviewer asks why a blocked listing looked different from its source photo.

Keep source identity and preset identity separate. The derivative key can be a hash of `(sourceDigest, presetName, presetVersion)`. That makes retries idempotent and makes reuse explicit: a second product referencing the same source does not need a second transformation job.

Do not overwrite a preset in place. Publish `v3`, migrate references deliberately, and retain old outputs until moderation records no longer point at them. I have seen a one-line crop change turn an audit trail into a guessing game because the old thumbnail was silently replaced. The bug was not in the resizer; it was the missing version boundary. During a migration, I would write both the old and new derivative references to the event log, compare moderation decisions for a sample of listings, and only then switch the storefront pointer; that extra step costs a little storage but avoids a silent change in the evidence reviewers rely on.

## How do catalog derivatives, image transformation presets, and moderation fit together?

Treat moderation as a consumer of a declared derivative. The event payload should include the original digest, the preset version, and the derivative location. A reviewer can then reproduce the exact input, while a buyer-facing request can reuse the already-approved output.

Here is a deliberately small TypeScript model. The `applyPreset` function stands in for your chosen image engine; keeping that dependency behind one interface makes a library change a data migration, not a frontend rewrite.

```ts
type Crop = "center" | "attention" | "none";

type ImagePreset = {
  name: string;
  version: number;
  width: number;
  height: number;
  crop: Crop;
  format: "webp" | "jpeg" | "avif";
  purpose: "review" | "card" | "zoom";
};

type DerivativeRef = {
  sourceDigest: string;
  preset: Pick<ImagePreset, "name" | "version">;
  objectKey: string;
};

function derivativeKey(sourceDigest: string, preset: ImagePreset): string {
  return `derivatives/${sourceDigest}/${preset.name}-v${preset.version}.${preset.format}`;
}

async function buildDerivative(
  source: Uint8Array,
  sourceDigest: string,
  preset: ImagePreset,
): Promise<DerivativeRef> {
  const output = await applyPreset(source, preset);
  const objectKey = derivativeKey(sourceDigest, preset);
  await objectStore.put(objectKey, output, { contentType: `image/${preset.format}` });
  return { sourceDigest, preset, objectKey };
}

declare function applyPreset(source: Uint8Array, preset: ImagePreset): Promise<Uint8Array>;
declare const objectStore: {
  put(key: string, body: Uint8Array, metadata: { contentType: string }): Promise<void>;
};
```

The worker should emit one completion event per derivative and include a transform fingerprint in logs. Alert on missing outputs and repeated retries, not on a single transient failure. Your mileage may vary on queue settings because image dimensions, not item count alone, drive memory pressure.

That's the operational loop.

The catch is that preset versioning adds storage and lifecycle work. It is not suitable when users need arbitrary, millisecond-level edits in a design tool; use a session-oriented rendering path there. It is also a poor fit for private originals that cannot be copied into a shared derivative store. Keep those transforms inside the tenant boundary and enforce authorization before issuing a derivative reference.

If moderation coverage is the priority, do not let a CDN generate an untracked variant that reviewers never see. If latency is the priority for a handful of fixed sizes, inline transforms can be simpler. Measure queue age, derivative hit rate, bytes stored, and the percentage of listings whose moderation input matches a published preset. Those metrics map directly to revenue per engineering hour: they tell a one-person team whether another preset is worth maintaining.

## References

- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/ETag
- https://www.rfc-editor.org/rfc/rfc9110
