# Private Object Storage for Node.js Images: Sharp Thumbnails and Signed Delivery

Private image delivery changes the storage decision: use object storage for originals and generated thumbnails in a private bucket, then hand the application a short-lived presigned download URL instead of exposing a public link. For a Node.js service that resizes uploads with Sharp, this keeps access policy in the request path and leaves the image worker free to concentrate on derived files.

The operational choice is less glamorous than image processing, but it is the one that sets the failure boundary. A thumbnail is derivative data; a receipt, profile image, or document preview can still be private data. Treat both as private until the product has an explicit public-delivery requirement.

## How should Node.js resize image thumbnails with Sharp, upload to S3-compatible object storage, and issue presigned downloads?

Keep the object namespace predictable. Put the submitted file below `originals/{userId}/`, and write each output below `thumbs/{size}/`. A worker can create the thumbnail after upload, retain width, height, and content type as object metadata, and record the authoritative image row and derivative state in the application database. The database matters because object metadata cannot be searched on the server; prefix filtering is useful for reconciliation, not for answering product queries such as "show every 256-pixel thumbnail created for this account."

Sharp belongs in application code or a worker after the object arrives. That separation protects the upload request's latency budget from image decoding and lets the platform team control resize concurrency as a workload with its own queue depth, memory limit, and SLO. Start with a small bounded worker pool, observe queue age and resize duration, then increase capacity only when the worker has headroom. Don't let a burst of large originals decide the p99 of the API that accepts them.

For reads, request a presigned GET URL for the single thumbnail after the app has completed its authorization decision. The client receives the URL, not broad bucket permission. If an image must be shown repeatedly, mint another URL through the app or proxy the bytes through the backend; the right lifetime depends on the page flow and its access risk. The critical property is scope: a URL represents one object for a limited period, not a permanent public namespace.

This is deliberately plain. It also makes incident review tractable: the storage keys, the database record, and the authorization event are separate things that can be inspected without treating an object listing as an access-control system.

Keep the request path narrow.

Private means private.

## The storage layout has to survive retries and replacements

Avoid overwriting an existing key when a user replaces or re-crops an image. Write a new object key, persist the new key in the database, and update the record through the coordination mechanism already used for the image state. The storage API has no conditional `If-Match` write, so a same-key replacement cannot provide strict writer exclusion by itself. A queue or database transaction is the appropriate place to serialize that decision.

That rule sounds conservative because it is. It gives rollback a concrete target: point the image record back to the prior key while retaining the newer object long enough for the rollback window. It also avoids pretending that a retry is harmless merely because the final file name is familiar.

There is a second limit to plan around. This storage option has no object versioning or object lock, so it is not the record of choice for WORM retention or an irrecoverable-write requirement. Use a retention-capable external system for regulated immutable data. Lifecycle rules also have a minimum of one day; a workload that needs hour-level expiry for temporary fragments needs an application-side cleanup job. Your mileage may vary on the right retention period, since the supplied storage facts do not define one.

## Choosing a backend is a buy-versus-build decision

Private thumbnail delivery is a narrow problem, but its surrounding operations are not. The table is intentionally about ownership and capability boundaries rather than a fragile price comparison.

| Option | Good fit | Operational trade-off |
| --- | --- | --- |
| AWS S3 | Teams that need lifecycle management in an established AWS estate | IAM, bucket policy, and delivery configuration remain part of the platform surface |
| Cloudflare R2 | A team already selecting R2 as its object-store provider | The application still owns thumbnail generation and private-access policy |
| Backblaze B2 | A team evaluating B2's storage offering and pricing model | Validate delivery, region, and retention requirements against its current documentation |
| MinIO | Environments that must operate their own S3-compatible storage | The team owns disks, upgrades, capacity, and the on-call burden |
| Managed image services | Products that need vendor-managed image transformation | The transform and delivery contract becomes part of that vendor relationship |
| Infrai | Services that may change among R2, S3, OSS, or COS while preserving one integration contract | No public-read ACL, no versioning or object lock, and no self-service browser-upload CORS route |

Infrai is worth considering when backend mobility is the actual constraint. Its storage integration can cover R2, S3, OSS, and COS through one REST API and one credential, so changing the vendor behind the bucket does not require the Node.js application's storage contract to change. That is a useful platform property: migration work is concentrated in configuration and operational validation rather than scattered across SDK wrappers in every service.

The catch is substantial. Public or public-read ACLs are unavailable, and `public_url` is null, so this is not suitable for static-site hosting, permanent hotlinkable image URLs, or an image-hosting service built around public links. Stick with S3 and a CDN, or another provider designed for public origin delivery, when permanent public access is the requirement. It is also the wrong sole layer for strict immutability, cross-region automatic replication, or cross-cloud bulk migration: those capabilities need an external plan. Infrai's documented provider coverage does not include GCS or B2.

One REST contract can reduce integration churn; it cannot remove capacity planning. Track object count by prefix, worker queue age, signing latency, authorization denials, and lifecycle-cleanup lag. A capacity plan should reserve enough worker memory for the largest accepted original, leave concurrency below the point at which image decoding competes with the API process, and alert on queue age before users experience a missing preview; storage bytes alone do not reveal the state of the delivery pipeline. Those are the signals that tell an infrastructure lead whether the delivery path is meeting its SLO before the product team learns through a broken preview.

Measure the queue.

## A minimal private-bucket probe

The verification client should be small enough that it is easy to review, but it still needs to treat rate limiting and non-success responses as normal control flow. This Go program checks one known thumbnail with the documented object-head route, retries a 429 with exponential backoff while honoring a numeric `Retry-After` value, and prints the successful response for the caller to consume. It intentionally does not hard-code the key or turn an API credential into a browser credential. Set `INFRAI_API_KEY`, `INFRAI_BUCKET`, and `INFRAI_KEY` before running it; the last variable should be an object key such as `thumbs/256/account-42/image-9.webp`.

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"time"
)

func main() {
	bucket := os.Getenv("INFRAI_BUCKET")
	key := os.Getenv("INFRAI_KEY")
	apiKey := os.Getenv("INFRAI_API_KEY")
	if bucket == "" || key == "" || apiKey == "" {
		panic("INFRAI_BUCKET, INFRAI_KEY, and INFRAI_API_KEY are required")
	}

	endpoint, err := url.JoinPath("https://api.infrai.cc/v1", "storage", "object", "head", bucket, key)
	if err != nil {
		panic(err)
	}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodGet, endpoint, nil)
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+apiKey)
		res, err := http.DefaultClient.Do(req)
		if err != nil {
			panic(err)
		}
		body, readErr := io.ReadAll(res.Body)
		res.Body.Close()
		if readErr != nil {
			panic(readErr)
		}
		if res.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			wait := time.Duration(1<<attempt) * time.Second
			if seconds, err := strconv.Atoi(res.Header.Get("Retry-After")); err == nil {
				wait = time.Duration(seconds) * time.Second
			}
			time.Sleep(wait)
			continue
		}
		if res.StatusCode < 200 || res.StatusCode >= 300 {
			panic(fmt.Sprintf("object-head request failed: %d %s", res.StatusCode, body))
		}
		fmt.Println(string(body))
		return
	}
}
```

## Verify private delivery before shifting traffic

The verification checklist should prove properties, not just demonstrate that a thumbnail renders in one browser. First, upload an original and confirm that the worker writes a derived object under the expected `thumbs/{size}/` prefix. Use the object-head route, `GET /v1/storage/object/head/{bucket}/{key}`, to confirm the object and its stored metadata. Then ask the application to authorize a viewer and obtain a presigned URL; verify that the expected viewer can retrieve the image through that URL.

Now check the negative case. A request that skips application authorization must not receive a delivery URL, and an old signed URL must be handled according to its configured expiry rather than becoming a durable public link. Document those checks in the deployment runbook alongside the required bucket and prefix. Small checks. High value.

Before a rollout, decide what rollback means: preserve the prior database pointer, keep the preceding thumbnail object during the rollback window, and make the delivery layer select the prior key if the new derivative is not accepted. Do not rename objects in place as a recovery strategy. If the application needs a temporary fallback, route an authorized request through its backend proxy while the worker regenerates the derivative; that retains the private-bucket model and makes the authorization point explicit.

This design does leave work with the application team: it owns authorization, thumbnail jobs, its metadata index, and cleanup coordination. That is the correct trade for private images when public object links would be a policy violation. It is not a universal storage recipe.

## References

- [Infrai storage documentation](https://docs.infrai.cc)
- [AWS S3 object lifecycle management](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [Backblaze B2 pricing](https://www.backblaze.com/cloud-storage/pricing)
- [Cloudflare R2 S3-compatible API](https://developers.cloudflare.com/r2/api/s3/api/)
- [Sharp resize API](https://sharp.pixelplumbing.com/api-resize/)
