# Tenant-Isolated Avatar Thumbnails with Node.js Sharp and Private Storage

Short answer: keep the uploaded original private, create a fixed set of Sharp derivatives in a worker, and publish them only through a tenant-scoped immutable release record. The important design choice is the release boundary, not the resize call: one tenant must never be able to discover, read, or activate another tenant's objects.

This is a runbook for a developer-tools product that stores per-tenant user avatars. I would use it for the same reason I use release records for other tenant data: a partial write must remain invisible, and a rollback must be a database decision rather than a race between five object deletions.

## Tenant governance is the first avatar-storage requirement

Keep it boring.

Save the original and a small, predetermined set of square sizes, for example 32, 64, 128, and 256 pixels. Preserve the original under the same release prefix as its derivatives, but never serve that key directly to a browser. The tenant ID belongs in the authorization decision and in the object key; it is not merely a folder label for an operator's convenience.

Use a key shape such as `tenants/t-17/avatars/a-42/r-0008/original` and `tenants/t-17/avatars/a-42/r-0008/128.webp`. The application allocates `r-0008`, records the expected objects in a release manifest, and keeps the current release pointer on the tenant-owned avatar row. A new upload gets a new release. It does not overwrite `current.webp`.

The worker reads the accepted bytes once, retains those bytes as the original, and asks Sharp to produce each configured output. Crop policy, maximum input dimensions, format, and quality are product decisions; there is no honest universal focal point for a portrait and a logo. I'm not sure a single crop rule serves both, so I would make the asset class explicit or collect a focal point before pretending the image operation is deterministic.

The boundary has to survive ordinary maintenance work. A support export, a retry queue, a lifecycle rule, and a deletion job all need the same tenant predicate as the read path; otherwise the application can be perfectly correct during upload and still leak data during reconciliation. I would make `tenant_id` mandatory in every manifest query, include it in the worker message, and reject a job whose object prefix does not match the manifest rather than trying to repair the mismatch in place. This is governance, but it is also capacity hygiene: a runaway cleanup scan or a cross-tenant retry is expensive precisely because it bypasses the assumptions used in the normal path.

Here is the language-neutral part of the contract expressed in Go. The worker can be written in Node.js, while other services can test the same naming invariant without importing an image library.

```go
package avatar

import "fmt"

var Sizes = []int{32, 64, 128, 256}

type Release struct {
	Original string
	Images   map[int]string
}

func Keys(tenantID, avatarID string, revision uint64) Release {
	root := fmt.Sprintf("tenants/%s/avatars/%s/r-%04d", tenantID, avatarID, revision)
	images := make(map[int]string, len(Sizes))
	for _, size := range Sizes {
		images[size] = fmt.Sprintf("%s/%d.webp", root, size)
	}
	return Release{
		Original: root + "/original",
		Images:   images,
	}
}
```

The invariant is simple: every read resolves the avatar row, checks that the caller belongs to the tenant, loads the active manifest, and then authorizes the requested derivative against that manifest. A guessed object key must not be a read API.

## A Go contract for the Node.js Sharp thumbnail worker

Treat an avatar update as a small transaction with explicit phases: validate the upload, allocate a revision, generate derivatives, write the original and derivatives, verify each stored object's content type and byte size, then atomically change the active pointer. The pointer changes last. If generation or verification stops midway, the old release remains active and the abandoned prefix is cleanup work.

The database manifest should contain the tenant ID, avatar ID, revision, original key, derivative keys, expected media types, byte sizes, creation state, and activation time. A manifest in application storage gives reconciliation a finite set to inspect. Listing a bucket and guessing which objects belong together is not an isolation policy.

The awkward cases deserve more attention than the happy path. Two uploads for the same avatar need serialization through a database transaction or a queue key; otherwise both workers can finish correctly and still publish an unexpected winner. A retry must be idempotent for one revision, while a new upload must receive a different revision. On an activation conflict, I would record the release as superseded and return the row's current revision rather than silently switching ownership. That is the kind of small, explicit state machine that keeps a 409 from becoming a customer-visible split-brain image.

Keep URLs short-lived and mint them only after the authorization check, or stream through the application when every request needs policy evaluation. The former keeps image bytes away from application workers; the latter gives tighter control and consumes more read capacity. Private storage is not the same thing as an unguessable URL.

## How can Node.js Sharp keep avatar thumbnails private across tenant releases?

Capacity planning starts with the accepted release, not with an average thumbnail request. The fixed set above means one original, four derivatives, and five writes per accepted upload, plus verification and eventual cleanup. Estimate the arrival rate, retention window, peak input size, worker memory, and queue age before setting concurrency. A thumbnail worker that decodes a very large image can exhaust memory long before its output count looks impressive.

Define the SLO around what the user sees: time from accepted upload to a verified active release, successful reads of the active derivative, and the rate of releases rejected for policy or validation. Queue depth, decode time, object-write latency, and orphan count are useful signals, but they are not the user SLO. Backpressure is mandatory.

Bound queue depth and worker concurrency. Make the pending state visible to the product instead of publishing one derivative while the remaining sizes are still missing. For cleanup, use the manifest to find releases that never activated, apply a retention window longer than the maximum URL lifetime, and measure deletion lag. A cleanup job must be tenant-scoped too; a broad prefix mistake is an isolation incident, not housekeeping.

## Compare ownership before choosing an image service

There are three sensible operating shapes, and none removes the need for an authorization model:

| Approach | Your team owns | Useful when | Poor fit when |
|---|---|---|---|
| Application worker plus object storage | Sharp capacity, manifests, access checks, cleanup | Sizes are stable and tenant policy is custom | Product needs arbitrary transforms at read time |
| Managed image transformation | Input policy, tenant authorization, release semantics | Transformation operations are not a platform differentiator | Tenant-specific isolation or retention rules exceed its model |
| Self-hosted image worker and storage | Worker fleet, storage durability, upgrades, on-call | Portability and control justify the operational load | The platform team cannot staff image-processing incidents |

The catch is operational ownership. A managed service can reduce worker maintenance while adding a dependency and a provider-specific image contract; self-hosting can improve control while moving patching, capacity, and durability onto the platform roadmap. I would choose the smallest contract that preserves tenant isolation and the SLO, then document what happens when the queue is full, a revision is superseded, or a tenant is deleted.

Stick with runtime resizing only when arbitrary dimensions are a real feature and the cache key, authorization, and cost envelope are understood. It is not suitable when a predictable set of UI sizes is enough: every request then becomes another path through decode, authorization, cache invalidation, and tail-latency analysis.

## Evaluate verification, rollback, and deletion together

Verification should exercise both data and policy. Upload a release for tenant `t-17`, confirm that all five expected objects exist beneath its revision, and assert that a request authenticated only for `t-18` cannot obtain a URL or bytes for any of them. Then interrupt the worker between derivative writes and confirm that the active pointer is unchanged. Test a repeated job for the same revision, a concurrent upload, an expired URL, and a tenant deletion with pending cleanup.

Rollback should update the avatar row to the previous verified revision and leave the newer objects untouched until their retention timer permits deletion. That makes recovery reversible and prevents an old signed URL from pointing at a key that disappeared during the rollback transaction. Your mileage may vary on the exact retention duration; the evidence needed is the maximum URL lifetime, support workflow, and regulatory deletion requirement.

A useful release checklist is short:

- verify tenant membership before resolving any object key;
- publish only a complete manifest;
- alert separately on queue age, worker memory, verification lag, and active-read failures;
- reconcile abandoned revisions by manifest, not by an unbounded bucket scan;
- rehearse pointer rollback without deleting the candidate release.

This is the decision rule I would put in the platform roadmap: immutable tenant-scoped releases, bounded derivative sizes, and a database pointer that changes last. The image library is replaceable. The isolation boundary and the recovery procedure are not.

## References

- MDN: Content-Disposition response header: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
- Firebase Cloud Storage documentation: https://firebase.google.com/docs/storage
