# Database Metadata Before Bucket Prefixes for PDF, DOCX, and Invoice Storage

The constraint that changes this design is authorization: an object name must never be enough to read a user file. **Short answer: keep document bytes behind a private object boundary, make the application authorize every document ID against tenant-scoped database metadata, and use prefixes only for organization.** This works for PDFs, DOCX files, and invoices without pretending that a folder-like key is a security control.

A single table full of binary data can be a sensible early choice, especially for a small internal tool with tiny attachments and one transactional backup boundary. It becomes harder to live with as files, replicas, restores, and download traffic grow. The common shortcut in object storage is worse: accept a client-provided path such as `tenant-a/invoice.pdf`, then rebuild that path during download. It looks tidy until a missing tenant predicate turns a naming convention into an access rule.

The useful split is mundane. The object system holds bytes; the database owns identity, state, retention, and authorization context. A key may include an internal tenant partition to make operations easier, but it is not a capability.

No key is authority.

## How should a multi-tenant SaaS handle private PDF, DOCX, invoice documents?

Create the database record under the authenticated tenant before accepting bytes. The server creates the final object key, ideally from opaque identifiers, and the browser receives only a narrowly scoped upload instruction. Store the original filename as display metadata rather than treating it as the key. After transfer, validate the size, allowed media type, and any integrity value your delivery path can verify before changing the row from `pending` to `ready`.

On a read, accept a document ID, not a bucket, prefix, or raw object key. Query using both the document ID and the tenant ID established by authentication. A missing row and a row owned by another tenant should take the same public path, so document IDs do not become an ownership probe. The following repository boundary is deliberately small; the important part is the query shape.

```ts
type DocumentRow = {
  id: string;
  tenantId: string;
  objectKey: string;
  state: "pending" | "ready" | "deleted";
};

type Documents = {
  findReadyForTenant(
    documentId: string,
    tenantId: string,
  ): Promise<DocumentRow | null>;
};

export async function resolveDownload(
  documents: Documents,
  authenticatedTenantId: string,
  documentId: string,
): Promise<{ objectKey: string }> {
  const document = await documents.findReadyForTenant(
    documentId,
    authenticatedTenantId,
  );

  if (!document) {
    throw new Error("Document not found");
  }

  return { objectKey: document.objectKey };
}
```

The function has no `objectKey` argument from the caller. That absence matters more than the prefix format. A service layer can then issue the short-lived access mechanism supported by its storage system, or stream bytes after the same check. Neither option removes the need for a tenant-scoped lookup.

## The failed simple model puts too much meaning in a prefix

Prefixes are helpful for inventory, lifecycle reports, and cleanup jobs. They are not directories with inherited authorization. Different object systems apply access policies differently, and an application that relies on string parsing such as `key.startsWith(tenantId)` creates an easy place for normalization mistakes, alternate encodings, or a later query without the required predicate.

An application-owned row gives the file a lifecycle that an object listing cannot provide on its own. It can hold `tenant_id`, a stable document ID, object key, uploader, media type, expected byte count, checksum where applicable, display filename, created time, deletion state, and retention policy. Those fields support list views and audits without exposing storage internals to the client.

For tenant removal, first block new reads by changing the application state, then delete or expire the bytes through an idempotent cleanup process, and retain an audit record according to the policy. This has a two-system cost: database and object store do not share a transaction. Design for rows without bytes and bytes without ready rows, then reconcile them on a schedule. A reconciler needs a bounded policy rather than a vague retry loop: find pending records older than the permitted transfer window; ask the object layer whether the expected key exists; mark a verified upload ready only through the same validation path as a normal completion; and arrange cleanup for rows that never obtained bytes as well as objects with no surviving row. Deletion needs the mirror image. The application state closes read access first, repeated cleanup attempts remain harmless, and the audit record explains why an object is no longer available. That sequence gives support and operations a way to distinguish an unfinished transfer from a deliberately removed file. Don't hide this work behind an optimistic "upload complete" response.

There are legitimate alternatives. Keep binary columns in the database when the files are small, volume is modest, and atomic changes with related records outweigh the operational cost of larger backups and replicas. A single-host filesystem can fit a tightly controlled deployment. Neither pattern automatically survives horizontal scaling, direct browser transfer, or a future need to prove tenant isolation. The catch is operational ownership, not fashion.

## Upload progress, caching, and delivery need separate choices

Upload authorization and byte transfer are adjacent concerns, not the same concern. A browser may need progress feedback even when the application never proxies the full payload. MDN documents upload progress events on `XMLHttpRequest.upload`; test the browser behavior your product actually supports before making completion UI depend on it. The API should still create the pending record and accept the completion signal only after it can verify the expected state.

Downloads need an explicit cache policy. For sensitive document responses, `Cache-Control: no-store` instructs caches not to store the response, as MDN describes. It is a response directive, not a substitute for authorization. Decide separately whether each class of document renders inline or downloads as an attachment, and sanitize display filenames before using them in response headers.

This is where a small state machine earns its keep:

| State | Reader behavior | Operator action |
|---|---|---|
| `pending` | Not readable | Reconcile old records and confirm completion |
| `ready` | Readable after tenant authorization | Apply retention and observe transfer behavior |
| `deleted` | Not readable | Complete idempotent byte cleanup |

Interrupted transfers, duplicate completion events, and deletion retries are ordinary cases. Treating them as states makes testing much clearer than trying to infer intent from a bucket listing.

## What should be measured before adopting this document storage pattern?

Start with isolation tests. Create two tenants with the same display filename and prove that list, read, update, and delete operations remain tenant-scoped. Attempt access with a known document ID from the other tenant. Verify that it produces the same external result as an unknown ID. Then test an interrupted upload, a repeated completion request, a stale pending record, and an orphaned object.

Measure the time from upload start to ready, authorization denials by reason, age and count of pending rows, reconciliation discrepancies, restore time for one tenant, and delivery latency at representative file sizes. Cost modeling belongs here too, but stored bytes are only one line: account for requests, transfer, version retention, database growth, backup duration, and the engineering time needed to recover one tenant cleanly. For an AI application, model usage can dominate the bill, yet document restoration is still the incident that disrupts users.

Avoid this arrangement when immediate rollback of file bytes inside a multi-row database transaction is the real requirement; a binary column may be clearer. Avoid it when an offline desktop client is the authoritative file owner; use a local-first synchronization design instead. If regulated controls require a storage model the team cannot verify and operate, choose an approach your auditors can inspect. Your mileage may vary with file sizes and recovery objectives, but the invariant should stay fixed: a document is readable only when an authorized tenant has a ready database record pointing to it.

## Sources

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control
- https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest_API/Using_XMLHttpRequest
