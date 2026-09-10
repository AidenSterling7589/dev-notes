# User Documents: Browser Uploads to Object Storage, Presigned or Server

Short answer: for most user documents, issue a short-lived presigned upload URL and send the bytes from the browser to object storage; keep a server relay for files that must be inspected before persistence, very small files where another round trip is wasteful, or environments that cannot set bucket CORS. The choice is an SLO and failure-domain decision, not a React-versus-Node preference.

I learned this during a month-end intake run. Our Node service relayed scanned PDFs, so each byte crossed a pod on the way in and again on the way out. At the peak, upload traffic consumed the same connection pool used by metadata requests. The document endpoint stayed within its own latency target, but the unrelated API missed its p99 SLO. Nothing was technically down; capacity planning had simply treated file volume as zero.

The invariant was uncomfortable: the upload path is part of the platform budget even when the application team calls it “just a form.”

## What failed, and what the incident actually measured

We switched one cohort to direct uploads. The browser reported a generic CORS failure, with no useful response body. A request made outside the browser showed an authentication mismatch caused by a signing configuration that did not match the storage endpoint. Since the error response lacked the browser's cross-origin header, the browser hid the real message. I spent a day changing bucket rules that were already correct. The misleading symptom also distorted our incident timeline: the frontend team kept shipping policy changes, the storage team kept confirming policy matches, and the signing service looked healthy because it had returned a URL. Only a verbose, non-browser request exposed the canonical error body, after which the fix was a single configuration value and a redeploy. It failed quietly.

That experience changed the runbook. A signed-upload incident starts with a reproducible request using the exact URL and signed headers, then checks endpoint, region, clock, and object key. Only after that do we inspect CORS. Your mileage may vary when a provider has a different signature dialect, so record the canonical request and the storage service's documented limits alongside the alert.

The lesson is operational, not vendor-specific: CORS is a visibility boundary, while signing is an authorization boundary. Treating one as the other makes diagnosis slow.

Keep that distinction visible.

## Should browser uploads for user documents use presigned URLs or a server relay?

Compare the failure domains before comparing implementation effort. A relay has one application request to trace and no bucket CORS policy, but it also makes your workers, ingress, memory, and deploy behavior part of every upload. A presigned flow keeps large bodies away from the API and gives a cleaner request-duration SLO; in exchange, you operate signing, CORS, expiry, and reconciliation.

| Decision axis | Browser to object storage | Server relay |
| --- | --- | --- |
| Application bandwidth | No document bytes | Every byte traverses the service |
| First-day setup | Signing endpoint and CORS | One upload handler |
| Burst capacity | Storage service absorbs body traffic | Pods and ingress need headroom |
| Before-write inspection | Requires quarantine or later scan | Inline inspection is straightforward |
| Browser failure visibility | CORS can hide response bodies | Application controls the response |
| Restart behavior | Storage owns the transfer | Deploys can terminate in-flight requests |

For capacity planning, size the relay for concurrent bytes, not only request count. If a 10 MB median file arrives from 200 browsers at once, the service is carrying roughly 2 GB of payload before replication or buffering; streaming reduces memory but does not remove network and connection pressure. A direct path moves that pressure to the storage boundary and leaves the API to handle small control-plane calls.

The catch is that presigning does not create resumability. A failed plain PUT can restart from byte zero. For unreliable mobile links or multi-gigabyte objects, use a multipart or resumable protocol and make its part lifecycle observable; otherwise a direct upload can improve your API SLO while quietly worsening user completion time.

## How should CORS, limits, and object ownership be designed?

CORS should be narrow and explicit: list the application origins, allow only the methods and headers the browser sends, expose only headers the client needs, and keep preflight caching finite enough that policy changes take effect. Do not use `*` with credentialed requests. Test an OPTIONS preflight and an actual PUT from a deployed origin, because localhost success proves very little.

The signer, not the browser, chooses the object key. Derive a tenant or user prefix from the authenticated session, reject path separators in the display name, and bind the intended content type when the storage API supports it. A URL is a capability; anyone who obtains it can use it until expiry within the scope you signed. Five minutes is a reasonable starting point for ordinary documents, then adjust from observed upload duration and clock-skew data.

Presigned PUT generally cannot enforce a maximum byte count by itself. If the product needs a hard pre-write size condition, use a policy-based form upload where the storage service supports a content-length range, or accept the bandwidth risk and reject at a commit check. A single-object PUT has a finite service limit, and larger objects require multipart upload; consult the target service's current quota page rather than copying a number into an assumption that will age.

## The control plane must close the consistency gap

The browser needs three explicit states: URL issued, object transferred, and document committed. The last state is an application transaction, not a hopeful `200` from the storage service. On commit, the server should check that the key exists, verify expected size and type, attach the authenticated owner, and write the database record idempotently.

Here is the relevant shape in Go. The storage client is deliberately an interface so tests can model expiry, duplicate commits, and a missing object without a network dependency.

```go
package upload

import (
	"context"
	"errors"
	"fmt"
	"net/http"
	"strings"
	"time"
)

type ObjectStore interface {
	PresignPut(ctx context.Context, key, contentType string, ttl time.Duration) (string, error)
	Head(ctx context.Context, key string) (size int64, contentType string, err error)
}

type Service struct {
	Store ObjectStore
	TTL   time.Duration
}

func (s Service) Sign(w http.ResponseWriter, r *http.Request) {
	user := authenticatedUser(r)
	name := r.URL.Query().Get("filename")
	if user == "" || name == "" || strings.ContainsAny(name, `/\\`) {
		http.Error(w, "bad upload request", http.StatusBadRequest)
		return
	}
	key := fmt.Sprintf("documents/%s/%d-%s", user, time.Now().UTC().UnixNano(), name)
	url, err := s.Store.PresignPut(r.Context(), key, "application/pdf", s.TTL)
	if err != nil {
		http.Error(w, "unable to create upload", http.StatusInternalServerError)
		return
	}
	writeJSON(w, map[string]string{"url": url, "key": key})
}

func (s Service) Commit(ctx context.Context, user, key string, expected int64) error {
	if !strings.HasPrefix(key, "documents/"+user+"/") {
		return errors.New("key is outside the user prefix")
	}
	size, typ, err := s.Store.Head(ctx, key)
	if err != nil {
		return err
	}
	if size != expected || typ != "application/pdf" {
		return errors.New("object does not match the upload contract")
	}
	return saveDocumentIdempotently(ctx, user, key, size)
}
```

The relay version uses the same commit contract; it merely performs the storage write inside the request. Stream the body instead of buffering it, cap the request size at the edge, and make shutdown drain active uploads. Those controls are easy to omit because the happy path is a single handler.

## When is the simpler relay the better engineering choice?

Use a relay when policy requires malware or DLP inspection before an object becomes durable, when the bucket is managed by a team that cannot grant browser CORS, or when files are tiny enough that a signing round trip dominates the interaction. A quarantine bucket plus asynchronous scanning is a valid middle ground only when compliance accepts temporary storage before approval.

Stick with direct storage when the API's latency SLO must remain independent of document size and the organization can operate signed URLs, bucket policy, and a reconciler. The recommendation changes when you cannot observe orphaned objects, enforce tenant prefixes, or explain who may read the resulting key. In those cases, the apparent simplicity is deferred operational debt.

No design removes failure; it chooses where failure is visible and which team pays for it. Keep a metric for issued-versus-committed uploads, an age-bounded orphan sweep, and a synthetic preflight plus upload test from the real web origin. That is enough evidence to revisit the choice with data instead of another architecture debate.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS
- https://docs.aws.amazon.com/AmazonS3/latest/API/sigv4-query-string-auth.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- https://developers.cloudflare.com/r2/
- https://cloud.google.com/storage/docs
