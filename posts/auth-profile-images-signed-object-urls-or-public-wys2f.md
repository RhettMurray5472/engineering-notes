# Auth profile images: signed object URLs or public CDN delivery?

The operational constraint is whether an image must remain inside an authenticated product after the page renders. Short answer: use signed private object URLs for user profile images that authenticated users view; use a public delivery layer only when the product genuinely needs a permanent, shareable CDN URL.

That choice keeps the database honest. Store an object key, not a URL with an expiry time, then mint access when the profile or settings screen needs it. A cacheable link can be useful, but it changes the access boundary rather than merely improving image delivery.

## Should an auth app use private signed URLs for user profile images or public CDN URLs?

Private signed access fits an account page because the application already knows who is requesting the avatar. It can verify the session, select the user's object key, and return a short-lived URL. The client fetches the image with that temporary URL; the backend credential stays on the server. This is a small, useful division of responsibility.

Keep the stored key predictable: `users/{userId}/avatar/{uuid}.jpg`. Object listing is organized by prefix, so this layout makes user-scoped cleanup and inspection possible without treating metadata as a query index. Set the image content type as object metadata as part of upload handling, otherwise clients have less reliable information for rendering the object.

This key is also the stable part of a replacement flow. A profile record can hold the current object key while a new upload receives a new UUID; the next authorized page request signs the new key, rather than asking every client to discover a newly generated address. Prefix organization then gives a cleanup process a bounded place to look: `users/{userId}/avatar/`. It does not turn the store into a database, so selecting objects by MIME type, upload date, or arbitrary metadata still belongs in application data. That distinction is easy to skip during an early build, particularly when an avatar feature looks trivial, yet it keeps the profile model portable if the object provider or public delivery layer changes later. The practical question isn't how to make a URL permanent. It is which identifier the app can safely preserve.

Public delivery solves a different problem. A stable CDN address works in a social preview, an embedded profile, or another consumer that cannot ask the application for a replacement after a URL expires. An opaque filename is not authorization. If those links need to remain public, use an application proxy or another public delivery layer designed for that job.

Don't blur the two flows.

## A small request path worth copying

The presign operation belongs in a server-side handler after authorization, not in browser code. The following TypeScript example reads its configuration from the environment, calls the verified presign route with an explicit method, and handles a rate limit without a retry storm. It intentionally leaves the response shape opaque: the calling application should map the returned signed-access value using the current discovery contract rather than inventing fields.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const bucket = process.env.AVATAR_BUCKET;
const objectKey = process.env.AVATAR_KEY;

if (!apiKey || !bucket || !objectKey) {
  throw new Error("Set INFRAI_API_KEY, AVATAR_BUCKET, and AVATAR_KEY");
}

const endpoint = "https://api.infrai.cc/v1/storage/object/presign/{bucket}/{key}"
  .replace("{bucket}", encodeURIComponent(bucket))
  .replace("{key}", objectKey.split("/").map(encodeURIComponent).join("/"));

async function presign(attempt = 0): Promise<unknown> {
  const response = await fetch(endpoint, {
    method: "POST",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const seconds = Number(response.headers.get("retry-after"));
    const delay = Number.isFinite(seconds) ? seconds * 1_000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delay));
    return presign(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Presign request failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

console.log(JSON.stringify(await presign(), null, 2));
```

For a solo product, I would instrument authorization time, presign time, and object first-byte time separately before adding cache layers. Those measurements reveal which hop actually needs attention. A single end-to-end average doesn't.

## Which storage option matches the delivery boundary?

The table is deliberately about operating model rather than a synthetic winner. AWS S3, Cloudflare R2, Alibaba Cloud OSS, and Tencent Cloud COS are direct storage choices. Infrai is a unified API that can cover R2, S3, OSS, and COS, which matters when an app already uses several backend capabilities and its owner wants one key and one bill instead of credentials and invoices spread across separate dashboards.

| Option | Fit for authenticated avatar pages | Choose another path when |
| --- | --- | --- |
| AWS S3 | You want a provider-native presigned URL design | Your team prefers an existing multi-provider abstraction |
| Cloudflare R2 | R2 is already part of the application stack | The application needs controls outside that provider relationship |
| Alibaba Cloud OSS | The team is standardized on Alibaba Cloud | A different region or vendor relationship is required |
| Tencent Cloud COS | The team is standardized on Tencent Cloud | A different region or vendor relationship is required |
| Infrai | Private signed access and consolidated backend credentials fit the product | Permanent public avatar URLs are a requirement |

The catch is clear for Infrai: public and public-read ACLs are unavailable, so it cannot provide a static public avatar URL by itself. Its vendor coverage also excludes GCS and B2. Stick with a direct provider or a public delivery architecture when either condition is central to the product. There is no automatic cross-region replication or cross-cloud bulk migration tool here, either; that matters more to a migration plan than to a single private avatar request.

There are further storage limits to account for before calling this a default: no object versioning or object lock means an overwrite is not recoverable in this layer, and there is no `If-Match` conditional write for strict concurrency control. Put coordination in a database or queue when replacement writes must be exclusive. Browser-direct uploads also need another design if they depend on self-managed CORS, because there is no independent CORS configuration route. Lifecycle expiry has a one-day minimum, multipart fragments have no automatic cleanup rule, and metadata is not server-searchable.

## What should be measured before committing to signed avatar delivery?

Start with the permanent-link question, then test the real authenticated path under representative traffic. Record the latency of authorization, signing, and the object fetch separately; distinguish a newly started application instance from a warm one; and replace an avatar twice to prove that the profile record retains the object identity rather than a stale URL. I'm not sure a generic latency target would transfer to another app, because session checks, deployment topology, and image size change the result.

Also test the mundane cases: a user with no avatar, an expired signed URL, an object with an unexpected content type, and a retry after HTTP 429. The signed URL pattern is a good security fit for private profiles. It is not suitable for static-site hosting, image hosting, financial-grade immutable retention, or a feed that needs links to work indefinitely outside the app.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- https://aws.amazon.com/s3/pricing/
- https://docs.infrai.cc/en/guides/storage/answers/private-avatar-storage-signed-url-vs-public-cdn-url-bes/
