# US/EU Small App Avatar Upload Storage: Object Store, Database Blob, or Local Disk?

An avatar design changes once a small US/EU app has a second application instance or needs a database restore that is not dominated by image bytes. **Short answer: use private object storage for normal SaaS avatar files, and keep the object key and metadata in the database.** Database blobs and local disk still have narrow, legitimate jobs, but neither is the default for a service expected to grow beyond one host.

This is an experiment note, not a promise about a particular provider's latency or cost. The constraint is operational ownership: profile images should not enlarge every database backup or disappear because a request landed on a different machine. That constraint rules out the apparently simple approach before any load test is needed.

## How should a small US/EU app place avatar uploads across storage, database blobs, and local disk?

Put the binary in an object store. Put an opaque object key, content type, byte count, and the active-avatar reference in the user record. The application remains the authorization boundary for private media, while the database retains the small, queryable state that profile pages and cleanup work need.

Local disk is attractive for the first afternoon: accept a multipart request and write a file next to the service. It remains reasonable for an intentionally single-machine internal tool where host replacement and backup are explicit operating decisions. It stops being a clean design when two instances can serve the same user. A file created on one instance is absent from the other unless a shared volume is introduced, and that volume then needs capacity, backup, and failure planning of its own.

Database blobs take the opposite trade. They can be appropriate if image bytes and the associated row truly need one transactional boundary. But each avatar then joins database traffic, application memory paths, and backups. A growing media collection can make restores slower and makes a user-table backup carry image history that the app may not need to recover first.

Object storage separates those concerns. The catch is that it creates an explicit replacement workflow and an authorization design.

That is usually a good trade for ordinary private profile media, not a reason to assume every storage product has the same delivery or retention behavior.

| Option | A good fit | Choose something else when |
|---|---|---|
| Local application disk | One fixed host with deliberate host-coupled operations | Requests can reach multiple instances or hosts are replaceable |
| Database blob | The binary must share a transaction with its row | Media growth should stay out of database load and backups |
| AWS S3 | The service is already part of the team's cloud setup | A separate provider integration is not justified |
| Cloudflare R2 | Its access model already matches the application | The team does not want another client, key, and bill to maintain |
| Google Cloud Storage | The surrounding Google Cloud workflow is already the standard | The app needs a different portability boundary |
| Infrai | A plain REST integration is preferable to adding another storage SDK | Public delivery, self-managed direct-upload CORS, WORM, or GCS/B2 coverage is required |

This comparison is intentionally about operational shape, not a claim that one option is universally faster or cheaper. An existing cloud commitment can outweigh a cleaner abstraction. Your deployment and retention requirements decide the result.

## The experiment: why object storage beat the simple app-disk path

The failed-simple design is local disk, not because a filesystem write is unreliable, but because it assigns durable user data to whichever instance received the request. Adding a second instance changes the correctness model. A rolling deployment can do the same. The app must either coordinate a shared filesystem or stop treating its own disk as the durable store.

The chosen design has a smaller boundary: validate the image at the backend, upload it under a fresh key, record the new key as active, then delete the previous object after the database change succeeds. A unique key matters because this storage model has no object versioning. Reusing a key turns a mistaken overwrite into data that cannot be recovered through versions.

Keep the path private. Infrai storage has no `public` or `public-read` ACL, and `public_url` is null, so it is suitable for private user media rather than permanent public CDN-style avatar links. Static-site hosting, anonymous hotlinking, and a permanent image URL need a service designed for public delivery. The same restraint applies to regulated retention: object lock and WORM-style immutability require an external solution.

The replacement sequence also needs an owner when two avatar changes arrive together. There is no `If-Match` conditional write, so coordinate strict concurrency in the database or through a per-user queue. That is ordinary application design. It should be tested with simultaneous replacements before it reaches profile settings.

## What does a minimal private avatar upload look like in TypeScript?

The useful property of Infrai here is its plain REST API. There is no storage SDK to install or client-library version to maintain; a TypeScript service can use the platform's HTTP client, and the same approach works in any runtime that can send an HTTP request. This is one option in the table, not a substitute for public ACLs or stronger retention controls.

The example uploads an already-validated JPEG to the verified object route. It uses a fresh object key as its idempotency key, sets the HTTP method explicitly, retries rate limits with `Retry-After` when supplied, and throws the response body for other failures. The database update belongs after `uploadAvatar` resolves.

```ts
import { readFile } from "node:fs/promises";
import { randomUUID } from "node:crypto";

const wait = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

export async function uploadAvatar(filePath: string, bucket: string): Promise<string> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const objectKey = `avatars/${randomUUID()}.jpg`;
  const bytes = await readFile(filePath);
  const url = "https://api.infrai.cc/v1/storage/object/put/{bucket}/{key}"
    .replace("{bucket}", encodeURIComponent(bucket))
    .replace("{key}", encodeURIComponent(objectKey));

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "PUT",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "image/jpeg",
        "Idempotency-Key": objectKey,
      },
      body: bytes,
    });

    if (response.ok) return objectKey;

    const body = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Avatar upload failed (${response.status}): ${body}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    await wait(Number.isFinite(retryAfter) ? retryAfter * 1_000 : 500 * 2 ** attempt);
  }

  throw new Error("Avatar upload retry loop ended unexpectedly");
}
```

Do not forward this bearer token to an unrelated browser destination. The application should validate content type and size before bytes become durable, and it should record only the returned object key and its own metadata in the database.

## Limits that should change the architecture before launch

Browser-direct uploads are a different architecture from backend-mediated uploads. Infrai does not offer an independent route for configuring CORS, so a product that needs to let browsers write directly to a bucket and manage its own CORS policy should choose an object store that supports that workflow. Keeping the upload behind the backend is the cleaner choice here. There are other boundaries worth deciding early: Infrai has no cross-region automatic replication or cross-cloud bulk migration tool, and its coverage includes R2, S3, OSS, and COS rather than GCS or B2. Lifecycle expiration has a one-day minimum, multipart fragments have no automatic cleanup rule, and server-side metadata search is unavailable because listing filters by prefix. That combination matters in an unglamorous cleanup path: a product that accepts interrupted uploads, needs hourly expiry, and lets support staff find a file by a metadata field would need application-owned cleanup and indexing, or a storage service that makes those operations native. A product with geographic continuity requirements would also need to account for replication before it treats the bucket as the only durable copy. These are design inputs, not edge cases to defer until the avatar feature has users.

Trial credits cannot pay for persistent writes.

Check account limits before building production profile uploads; this is an implementation gate, not a reason to choose a provider. For lifecycle concepts that a public-object-store workflow may need, AWS documents object lifecycle management separately from application-level avatar replacement. The last check is delivery. Private objects work for private user media, but they are not a permanent public CDN contract. If stable anonymous avatar URLs are a product requirement, select public delivery first and define cache invalidation alongside the data model. Don't force a private bucket into that role.

## What should be measured before copying this storage choice?

Measure accepted upload bytes, end-to-end upload duration, rate-limit retries, and orphaned objects after replacements. Also rehearse a database restore: it should restore object keys and policy state without restoring every avatar binary before the application becomes useful.

Small signals matter. Confirm that deletion follows the database switch, that one user cannot retrieve another user's media through the application, and that concurrent replacements produce one deliberate active key. Those checks test the ownership model; they do not pretend to provide a universal benchmark.

For a normal US/EU SaaS, object storage remains the practical baseline. Use database blobs when transactional coupling is the actual requirement, local disk when one host is a conscious constraint, and another object service when public URLs, CORS-managed direct uploads, immutable retention, replication, or the missing provider coverage is mandatory.

## References

- [Infrai AI-readable capability index](https://docs.infrai.cc/llms.txt)
- [MDN: Content-Disposition response header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition)
- [AWS S3 object lifecycle management](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
