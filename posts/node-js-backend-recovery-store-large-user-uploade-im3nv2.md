# Node.js Backend Recovery: Store Large User-Uploaded Documents with Database Metadata

Short answer: for selectable per-tenant backups, keep authorization, snapshot manifests, and the active document set in Postgres, while storing immutable document bytes in S3-compatible object storage. Keep blobs in the database only when the corpus is bounded and a single transactional backup is more valuable than separating file delivery from relational recovery.

The important mechanism is publication, not upload. A restore should build a complete candidate set, verify its tenant and bytes, then switch one database pointer from the old set to the candidate. Until that commit, students and instructors continue to see the old course documents. This makes the access-control boundary explicit while keeping the delivery design changeable.

Ship that protocol before tuning the bill.

## Implement an invisible candidate set

Treat each backup as an immutable manifest. The manifest belongs to one tenant and names a set of document IDs, object keys, and content digests. The live course record points to one published set. Restoring snapshot `snap_fall_01` creates a new candidate set; it does not edit the current rows or reuse mutable object keys.

That distinction fixes an awkward failure mode. If a restore copies 19 of 20 documents and then loses access to the final source object, replacing live rows as each copy finishes exposes a mixed course. Staging keeps the incomplete candidate invisible. The old set remains readable, and the restore can be retried with the same operation ID. No drama.

The runnable example below models the protocol without binding it to a vendor SDK. `ByteStore` can be implemented with a database byte column or an object store. `Catalog` is the authorization and publication boundary; a production adapter would place `publish` in a Postgres transaction.

```ts
import { createHash } from "node:crypto";

type SnapshotDocument = {
  documentId: string;
  sourceKey: string;
  sha256: string;
};

type Snapshot = {
  snapshotId: string;
  tenantId: string;
  documents: SnapshotDocument[];
};

type CandidateDocument = {
  documentId: string;
  objectKey: string;
  sha256: string;
};

interface ByteStore {
  read(key: string): Promise<Uint8Array>;
  write(key: string, bytes: Uint8Array): Promise<void>;
}

interface Catalog {
  findPublishedOperation(tenantId: string, operationId: string): Promise<string | undefined>;
  publish(
    tenantId: string,
    operationId: string,
    candidateSetId: string,
    documents: CandidateDocument[],
  ): Promise<void>;
}

const digest = (bytes: Uint8Array): string =>
  createHash("sha256").update(bytes).digest("hex");

const keyPart = (value: string): string => {
  if (!/^[a-zA-Z0-9_-]+$/.test(value)) throw new Error("INVALID_KEY_PART");
  return value;
};

async function restoreSnapshot(
  authenticatedTenantId: string,
  operationId: string,
  snapshot: Snapshot,
  bytes: ByteStore,
  catalog: Catalog,
): Promise<string> {
  if (snapshot.tenantId !== authenticatedTenantId) {
    throw new Error("SNAPSHOT_NOT_AVAILABLE");
  }

  const existing = await catalog.findPublishedOperation(authenticatedTenantId, operationId);
  if (existing) return existing;

  const tenant = keyPart(authenticatedTenantId);
  const operation = keyPart(operationId);
  const candidateSetId = `restore_${operation}`;
  const candidate: CandidateDocument[] = [];

  for (const document of snapshot.documents) {
    const source = await bytes.read(document.sourceKey);
    if (digest(source) !== document.sha256) {
      throw new Error("CHECKSUM_MISMATCH");
    }

    const documentId = keyPart(document.documentId);
    const objectKey = `tenants/${tenant}/sets/${candidateSetId}/${documentId}`;
    await bytes.write(objectKey, source);
    candidate.push({ documentId, objectKey, sha256: document.sha256 });
  }

  await catalog.publish(
    authenticatedTenantId,
    operationId,
    candidateSetId,
    candidate,
  );
  return candidateSetId;
}
```

`publish` has four jobs. It verifies that the candidate still belongs to the authenticated tenant, inserts the complete candidate manifest, changes the tenant's active-set pointer, and records the operation ID in one transaction. A unique constraint on `(tenant_id, operation_id)` makes a retry converge on the published result. The request body should never be allowed to choose the authenticated tenant ID; derive that value from the verified server-side session and constrain the snapshot lookup by tenant as well as snapshot ID.

Bytes written before a failed publication are unreferenced, not live. A reconciliation task can compare candidate prefixes with catalog records and delete old unreferenced candidates according to a documented retention rule. Keep that cleanup away from the request's correctness path — publication determines visibility, while reconciliation controls waste.

## How should Node.js store user uploaded documents for tenant recovery?

Ask which boundary must be simple. Database blobs give the application one transactional system for rows and bytes. They can be the sensible choice for a small, predictably bounded corpus when the team already tests database backup and recovery as one unit. The catch is coupling: document growth becomes database backup growth, and every delivery request competes with relational work unless another layer absorbs it.

Object storage plus Postgres metadata splits the responsibilities. Postgres decides who may restore which snapshot and which document set is live. The object store holds immutable payloads. This fits independently selectable tenant recovery and lets byte delivery evolve separately, but it introduces cross-system publication and reconciliation. If the team cannot operate those two workflows reliably, the simpler blob design is better even when its raw storage line looks less attractive.

| Decision pressure | Postgres blob | Object storage with Postgres metadata |
| --- | --- | --- |
| Restore publication | Rows and bytes can share one transaction | Stage bytes, then publish a manifest transaction |
| Tenant authorization | Enforced with relational queries | Enforced in metadata; keys are not credentials |
| Backup scope | Database and documents move together | Manifests and payloads have separate recovery paths |
| Delivery simplicity | Node.js reads from the database | Node.js proxies bytes or delegates limited access |
| Operational burden | Database growth and recovery load | Reconciliation, retention, and two-system testing |

There is no universal cheapest backend option. Compare the whole workload: retained bytes, write and read operations, data transfer, database backup size, restore exercises, monitoring, and engineering time. I'm not sure a generic break-even file size is useful; actual document sizes, download frequency, retention, and recovery objectives would be needed to calculate one. A spreadsheet built from a representative tenant and the current providers' published billing dimensions is more defensible than a copied threshold.

Stick with Postgres blobs when one recovery unit is the requirement, volume is capped, and adding a second persistence system would create more risk than it removes. Choose the split design when payload growth should not enlarge relational recovery, tenants need individually selectable snapshots, or document delivery needs an independent path. Neither model excuses an untested restore.

## Make object location useless without authorization

An object key is a locator. It isn't proof that a caller may read the object. For the least complex delivery path, authorize the tenant and document in Node.js, then stream the bytes through the application. This concentrates policy in one place and is often a reasonable starting point for modest traffic.

Delegated, time-limited object access can remove application bandwidth from the hot path, but it moves part of the security boundary into the delegation. The grant must be scoped to the intended object and operation, expire deliberately, and be issued only after the same tenant-constrained metadata lookup. Don't put permanent public access in front of restored school documents merely to simplify delivery.

Caching deserves an explicit response policy too. MDN documents that `private` permits storage by a private cache and that `no-store` asks caches not to store the response. Select between them from the sensitivity and reuse requirements of the document. After publication, a new immutable object identity prevents an old cached representation from masquerading as the newly restored live version; an application mapping can resolve the current logical document ID to that identity.

Access control wins here.

## Run the recovery contract as a deployment gate

A green upload test says little about snapshot recovery. Build a deterministic fixture with two tenants, `school_17` and `school_42`, and give both a logical document called `lesson_7`. Authenticate as `school_17` and request the other tenant's snapshot. The application should return the same non-revealing `SNAPSHOT_NOT_AVAILABLE` result used when no accessible snapshot exists, and it must create neither candidate metadata nor payloads for the caller.

Then test the publication boundary, not just the adapters. Restore a snapshot with 20 manifest entries while making the twentieth fixture unavailable; the active-set pointer must remain unchanged. Change one fixture byte and require `CHECKSUM_MISMATCH` before `publish`. Retry a successful operation with the same operation ID and verify that it returns the original candidate set rather than creating another one. Finally, place an unreferenced candidate object in the fixture, run reconciliation under the chosen retention rule, and verify that live and retained snapshot objects remain. These cases cover tenant isolation, completeness, integrity, idempotency, and cleanup without pretending that a successful `write` call proves recoverability.

Partial restore is failure.

Observe the same stages in production with operation IDs, tenant-safe identifiers, document counts, byte counts, durations, and final states. Logs must avoid document contents and access credentials. Alert on candidates that never reach publication, repeated integrity failures, and reconciliation growth, but keep visibility distinct from authorization: an operator's dashboard must not become a side door to student files.

Deletion also has a wider scope than the live pointer. An erasure workflow may need to address live manifests, retained snapshots, payload objects, derived representations, logs, and backups according to the system's legal basis and obligations. GDPR Article 17 defines a right to erasure and enumerates exceptions. Product and legal owners must turn that into a retention schedule; a storage adapter cannot choose the policy. Test the resulting inventory by tenant and document ID so a deletion claim has evidence behind it.

The operational checklist is short enough to state in prose. Before release, restore a real fixture into an isolated tenant, verify every digest and count, attempt a cross-tenant restore, repeat the same operation ID, confirm the active pointer changes only after the full candidate is present, and exercise reconciliation and erasure against retained data. Record the recovery duration and the workload used. Your mileage may vary, so rerun the exercise after changing document size limits, retention, delivery mode, or storage adapter.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control
- https://gdpr-info.eu/art-17-gdpr/

## Further reading

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control
- https://gdpr-info.eu/art-17-gdpr/
