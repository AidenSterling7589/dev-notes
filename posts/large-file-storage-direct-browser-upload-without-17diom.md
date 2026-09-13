# Large-File Storage: Direct Browser Upload Without Proxying and Signed Downloads in Europe

Short answer: for a fintech backup system, use private storage for a direct browser upload without proxying the archive through the application, then let the application issue a short-lived presigned upload and a separate signed download. Choose among backends only after testing large-file throughput, restore concurrency, regional placement, policy ownership, and egress under the same workload.

The scenario I design for is per-tenant backups with an operator selecting one snapshot to restore. The browser should move the archive directly to storage, while the application remains responsible for identity, tenant ownership, and the state transition from “uploaded” to “restorable.” That split avoids making the application proxy a bandwidth bottleneck without pretending that a storage URL is an authorization system.

The hard part is recovery correctness.

## What does a restore race reveal about tenant ownership?

The first failure to design around is not a slow upload. It is a valid operator selecting an older snapshot after a newer one has already passed verification. A browser retry can arrive late, a second tab can hold an old page, and a storage service can quite reasonably accept both writes. The application must therefore own the state transition, reserve an opaque key for each attempt, and make the database decide which verified snapshot is current. Storage correctness and business correctness are different contracts.

It is a 401, not a retry policy.

If a restore request arrives with an expired authorization, reread the tenant and snapshot state before issuing another grant. If two requests target one logical snapshot, make the reservation idempotent and record the winning state transition. I would put the audit event beside that transition, because an on-call engineer needs to answer who selected the archive, which version was verified, and when the download authority was issued without reconstructing the story from browser logs.

## How should storage handle browser upload, CORS, presigned URLs, and signed download?

Reserve an opaque object key in a database before issuing a presigned upload. Include the tenant and snapshot identity in the application record, but do not make a reusable business identifier the object name that two tabs can overwrite. The browser uploads directly, then the application verifies the expected size, checksum, media type, and ownership before exposing the snapshot to restore. A successful grant response is not a successful backup.

CORS belongs in the same release review as the upload method and request headers. Test the deployed origin, preflight, actual upload, and the headers the browser is allowed to read. CORS controls browser visibility; it does not authenticate a tenant. Keep objects private and issue a separate, expiring signed download only after the operator passes the restore authorization check. When a restore should download as an attachment, use `Content-Disposition` deliberately; MDN documents how that response header communicates the suggested filename and disposition, while the user agent retains final handling rules.

One short rule: one logical attempt, one opaque key. Retries can reuse an attempt record, but two competing snapshot selections must be resolved by the application database, not by an accidental last-writer-wins object update.

## What should a Europe throughput test measure before storage pricing?

Run the same test from the European locations that matter to the tenants, using representative archive sizes rather than a convenient small fixture. Record upload rate, time to finalization, restore download rate, p95 completion time, failed bytes, and the time between transfer completion and verified availability. Test a 401 or expired grant, a browser refresh, an interrupted multipart transfer, a checksum mismatch, and two concurrent attempts for one snapshot.

Capacity planning starts with the slow path. If a tenant can restore a large archive while another tenant is uploading, the SLO must include concurrent network use, checksum work, authorization latency, retries, and cleanup. In one test window, I would run a large restore beside several smaller uploads, interrupt the restore near completion, let the grant expire, and then repeat it after a browser refresh; the useful result is not the fastest run but the p95 completion time, the bytes retransmitted, the time spent cleaning abandoned parts, and whether the application briefly exposed an object before verification. A headline throughput number is not a restore SLO. Measure it.

The cost worksheet should contain stored GB, upload requests, download requests, outbound GB, retention days, retry bytes, incomplete multipart data, and the engineering time required to own lifecycle and access policy. The AWS S3 pricing page is a primary reference for the categories that may appear on a bill, but it cannot select a backend without the workload shape. I'm not sure which candidate wins your region and object-size mix; your mileage may vary. Run the probes before treating a price comparison as a decision.

## Compare storage by ownership and failure boundary

The candidate set may include AWS S3, Cloudflare R2, Backblaze B2, and Bunny Storage, but a fair comparison keeps the application flow constant. For each candidate, document who owns CORS changes, signing, lifecycle cleanup, replication, restore verification, and incident response. The platform team should also record which features become assumptions in the tenant metadata model.

| Question | Managed abstraction | Native backend integration |
|---|---|---|
| Large-file path | Fewer provider-specific adapters | More direct access to storage-specific transfer controls |
| Browser policy | One application contract for authorization | Provider-native CORS and signing are explicit |
| Recovery | Application owns selection and verification | Storage-native retention or versions may be available, depending on the service |
| Operations | Less integration code, but a policy ceiling to document | More settings and SDK behavior for the team to own |
| Lock-in | A common interface can reduce code changes | Native features can become data-model dependencies |

This is a buy-versus-build decision, not a scorecard. A managed boundary is reasonable when the team values a consistent HTTP contract and can live with its policy limits. A native integration is better when replication, retention, region controls, or storage-specific transfer behavior are mandatory. Price is one input after those constraints are known; storage, operations, and egress are different quantities, and a lower capacity rate does not explain restore traffic or migration work.

## How can the application reserve a snapshot before granting a browser upload?

The reservation must be idempotent and durable. This Go example shows the invariant without tying the design to a provider SDK; production code should enforce the same uniqueness rule in a database transaction.

```go
package main

import (
	"crypto/rand"
	"encoding/hex"
	"errors"
	"fmt"
	"sync"
)

var errReserved = errors.New("snapshot attempt already reserved")

type store struct {
	mu   sync.Mutex
	keys map[string]string
}

func (s *store) reserve(tenant, snapshot string) (string, error) {
	s.mu.Lock()
	defer s.mu.Unlock()

	claim := tenant + ":" + snapshot
	if _, exists := s.keys[claim]; exists {
		return "", errReserved
	}

	b := make([]byte, 16)
	if _, err := rand.Read(b); err != nil {
		return "", err
	}
	key := "private/" + tenant + "/" + hex.EncodeToString(b)
	s.keys[claim] = key
	return key, nil
}

func main() {
	s := store{keys: make(map[string]string)}
	key, err := s.reserve("tenant-42", "snapshot-17")
	if err != nil {
		panic(err)
	}
	fmt.Println(key)
}
```

The handler should authenticate the operator, validate tenant and snapshot state, reserve the key, and only then request the bounded upload grant. On completion, it should verify metadata, emit an audit event, and mark the snapshot available in one controlled state transition. An expired grant should create a new authorization for the same application attempt; a checksum mismatch should be a failed backup, not a partial success. Do not blindly retry a restore selection: reread state so an older operator action cannot replace a newer verified snapshot.

## When is direct browser upload the wrong fit?

The catch is the policy boundary. This design is not suitable when every byte must pass through inspection, when public delivery is a product requirement, or when the compliance model requires retention and replication controls the selected storage path cannot provide. Keep a proxy or choose a backend with the required native controls when that condition applies.

It is also a poor fit for a team that will not maintain an application index for tenant ownership, cleanup, and snapshot selection. Object listing cannot answer those questions safely. A signed link limits duration, but it does not decide whether the requested archive belongs to the operator's tenant.

My decision rule is narrow: approve direct upload when the application owns authorization and verification, the browser-to-storage path meets the large-file SLO in target European locations, and retries, egress, and cleanup are included in the operating model. Otherwise, select the architecture that makes the missing control visible.

## References

- [MDN: Content-Disposition response header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition)
- [AWS S3 pricing](https://aws.amazon.com/s3/pricing/)
