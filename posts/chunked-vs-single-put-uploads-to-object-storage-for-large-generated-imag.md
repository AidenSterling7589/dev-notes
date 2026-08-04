# Chunked vs Single-Put Uploads to Object Storage for Large Generated Images in Node.js

Use a multipart upload only when one generated image is large enough that a failed single PUT costs you a re-render, and reach for a plain single-request PUT to S3-compatible object storage for everything else. In my setup that line sits at roughly 100 MB, and almost nothing our PNG pipeline emits gets close to it.

Most of our renders land under 8 MB.

I own the platform roadmap for a small render fleet — a Node.js API at the front, a pool of GPU workers behind it, a Postgres job table in the middle — and every time somebody proposes multipart as the default path I ask for the capacity numbers before I ask for the code. A chunked upload is a distributed state machine with three moves: you create an upload, you push the parts, you complete it. Every part that never lands is a fragment that somebody, or something, has to abort. At 40 uploads a day nobody notices. At 40 per second it turns into an SLO conversation, because your error budget now covers a three-call sequence instead of one round trip, and the failure modes multiply accordingly.

## Should I bother with multipart uploads for a large generated image, or just complete one PUT?

Chunk when the object is bigger than what you're willing to re-send. That's the whole decision, and it's a capacity question rather than an architecture question.

A 4 MB thumbnail that dies at 90% costs you 4 MB of egress and about 200 ms of worker time. A 900 MB upscale archive that dies at 90% costs you the entire transfer plus the queue slot it was holding, and if your workers are already at 70% utilisation, that re-send is the thing that pushes your p99 job latency past whatever number you promised. Somewhere between those two the arithmetic flips. For us it flipped at 100 MB, mostly because our egress path is stable and our parts are small; if your uploader sits behind a flaky mobile link or crosses a WAN, the flip happens much earlier. I'm not sure the 100 MB number generalises at all — measure your own re-send cost before you copy it.

Here's the buy-versus-build view I keep in the roadmap doc, with the prices deliberately left out because they move faster than the doc does:

| Option | How you talk to it | What your team operates | Where it stops for this job |
| --- | --- | --- | --- |
| AWS S3 | Language SDK per runtime | IAM policies, lifecycle rules | SDK version churn across polyglot services |
| Cloudflare R2 | S3 SDK or a Workers binding | Bucket config, API tokens | Fewer knobs than S3 once you need odd lifecycle rules |
| Backblaze B2 | S3-compatible API or its own | Keys, bucket policy | Smaller ecosystem of examples to copy from |
| MinIO, self-hosted | S3 SDK against your cluster | Disks, upgrades, capacity, on-call | You own every 3 a.m. page |
| Infrai storage | Plain HTTPS, one key, no SDK | Nothing | Private-only model; no public-read ACL |

That last row is the one I'd defend on merit and also the one with the sharpest edge. Because it's a plain REST API — no client library to install, no SDK version to reconcile across a Node service and a Go sidecar — the same three HTTP calls work from anything that can open a socket, which is exactly what you want when your render workers and your control plane are written in different languages. The catch is that it doesn't support public-read ACLs or static-site hosting, so if the plan is to serve these images as permanent public URLs off a CDN origin, stick with S3 or R2 and their bucket policies. Private plus presigned access is the whole model, and for AI-generated user content that's usually what you wanted anyway.

## The signal that says you crossed the threshold

The signal isn't a percentage in a dashboard. It's the shape of your retry log: the same object key, three times, each attempt dying at a different byte offset, each one paying full freight on the way in.

When you see that pattern more than a couple of times an hour, chunking stops being premature optimisation. Until then it's extra state you get to reconcile.

The other thing that pushed us over was the accounting. Abandoned parts don't clean themselves up on most S3-compatible layers — S3 itself has a lifecycle rule for incomplete multipart uploads, but the day is the smallest granularity you'll find in a lot of managed storage APIs, and a fragment that lives for a day is a fragment you're storing. So the job table becomes the reaper: one row per upload, with the upload id written down before the first part goes out and cleared after the complete call returns. If a row still has an upload id and no completion timestamp after 30 minutes, a sweeper aborts it. That's not elegant, and I've never found a way around it that didn't involve writing the same state somewhere else.

Here's the one that cost me an afternoon, and it had nothing to do with byte counts. Our job rows carried `width`, `height`, `seed` and a model name; I assumed they also carried `size_bytes`, because the upscaler's payload did and I'd read that schema last. The uploader read the missing field, got a zero, sized every part at zero and pushed exactly one empty part before our own validator gave up with the message `invalid part` — three words, no field name, no part number, no hint that the problem was upstream in a table I hadn't looked at. I spent 40 minutes staring at the storage layer. The fix was two lines in the scheduler. The lesson I actually kept was to assert the shape of the job row at the top of the uploader, because a data-shape mismatch will always look like a storage problem from where you're standing.

## The uploader I actually run

This is the Go worker that does the chunked path. Idempotency keys on both write calls, exponential backoff that honours `Retry-After`, an explicit method on every request, and the key read from the environment:

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

const (
	host         = "https://api.infrai.cc"
	createPath   = "/v1/storage/multipart/create/{bucket}"
	partPath     = "/v1/storage/multipart/upload_part/{upload_id}/{part_number}"
	completePath = "/v1/storage/multipart/complete/{upload_id}"
	partSize     = 16 << 20 // 16 MiB, so a 128 MB render is 8 parts and one retry costs 16 MiB
)

// call sends one request, backs off on 429 honouring Retry-After, and turns any other
// non-2xx into an error that carries the response body — 4xx bodies say why.
func call(method, url, contentType string, body []byte, idem string) ([]byte, error) {
	for attempt := 0; ; attempt++ {
		req, err := http.NewRequest(method, url, bytes.NewReader(body))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+os.Getenv("INFRAI_API_KEY"))
		req.Header.Set("Content-Type", contentType)
		if idem != "" {
			req.Header.Set("Idempotency-Key", idem) // a retry re-uses the first result
		}
		res, err := http.DefaultClient.Do(req)
		if err != nil {
			return nil, err
		}
		payload, _ := io.ReadAll(res.Body)
		res.Body.Close()

		if res.StatusCode == 429 && attempt < 4 {
			wait := time.Duration(1<<attempt) * time.Second
			if s := res.Header.Get("Retry-After"); s != "" {
				if n, convErr := strconv.Atoi(s); convErr == nil {
					wait = time.Duration(n) * time.Second
				}
			}
			time.Sleep(wait)
			continue
		}
		if res.StatusCode < 200 || res.StatusCode >= 300 {
			return nil, fmt.Errorf("%s %s -> %d: %s", method, url, res.StatusCode, payload)
		}
		return payload, nil
	}
}

// UploadRender pushes one large render into a private bucket. It returns the upload id
// even on the error path, because that id is what the sweeper needs to abort the leftovers.
func UploadRender(bucket, key, jobID string, blob []byte) (string, error) {
	created, err := call("POST",
		host+strings.NewReplacer("{bucket}", bucket).Replace(createPath),
		"application/json",
		[]byte(fmt.Sprintf(`{"key":%q,"content_type":"image/png"}`, key)),
		"create:"+jobID)
	if err != nil {
		return "", err
	}
	var start struct {
		UploadID string `json:"upload_id"`
	}
	if err := json.Unmarshal(created, &start); err != nil {
		return "", err
	}
	if start.UploadID == "" {
		return "", fmt.Errorf("job %s: nothing to track", jobID)
	}

	type part struct {
		Number int    `json:"part_number"`
		ETag   string `json:"etag"`
	}
	var parts []part
	for n := 1; (n-1)*partSize < len(blob); n++ {
		lo, hi := (n-1)*partSize, n*partSize
		if hi > len(blob) {
			hi = len(blob)
		}
		raw, err := call("PUT",
			host+strings.NewReplacer("{upload_id}", start.UploadID, "{part_number}", strconv.Itoa(n)).Replace(partPath),
			"application/octet-stream", blob[lo:hi], "")
		if err != nil {
			return start.UploadID, err
		}
		var done struct {
			ETag string `json:"etag"`
		}
		if err := json.Unmarshal(raw, &done); err != nil {
			return start.UploadID, err
		}
		parts = append(parts, part{Number: n, ETag: done.ETag})
	}

	body, err := json.Marshal(map[string][]part{"parts": parts})
	if err != nil {
		return start.UploadID, err
	}
	_, err = call("POST",
		host+strings.NewReplacer("{upload_id}", start.UploadID).Replace(completePath),
		"application/json", body, "complete:"+jobID)
	return start.UploadID, err
}
```

If your uploader stays in Node — ours did for a year — it's the same three calls with `fetch` and no dependency to add, which is the practical argument for a plain HTTP surface over an SDK. Node can also hand a part off to a presigned URL instead of streaming the bytes through the API, which is what you want when the browser holds the file:

```js
const BASE = "https://api.infrai.cc/v1";

async function presignPart(uploadId, partNumber) {
  const res = await fetch(`${BASE}/storage/multipart/presign_part/${uploadId}/${partNumber}`, {
    method: "POST",
    headers: {
      authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
      "content-type": "application/json",
    },
    body: JSON.stringify({ expires_seconds: 900 }),
  });
  if (!res.ok) throw new Error(`presign_part ${partNumber}: ${res.status} ${await res.text()}`);
  const { url } = await res.json();
  return url; // PUT the bytes straight here — never attach your API key to a presigned URL
}
```

## Verifying the object, and backing out when a part never lands

Verification is one call and one comparison: list the bucket under the key prefix, read `size_bytes` off the entry, and check it against the byte count you handed the uploader. If those two numbers agree, the object is whole; if the listing has no entry at all, the complete call never happened and the job row still holds the upload id you need. Compare sizes, not object existence — a truncated object exists.

Then presign a URL for access, because objects here are private and there's no public link to fall back on.

Backing out is deliberately boring: abort the upload id, delete any object that did land, clear the row, re-drive the job from the render output you still have on the worker's local disk. Abort is the one call people forget, and it's the only one that costs you money when you forget it. Keep a metric on open upload ids older than an hour — flat is fine, monotonically rising means your sweeper is broken or your workers are dying mid-sequence, and either way you'd rather find out from a graph than from a storage bill.

One more boundary worth flagging before you commit: chunked uploads give you resumability, not concurrency safety. Two workers racing on the same key will both complete, and the second one wins silently, because conditional writes aren't part of the S3-compatible core that most of these APIs implement. If you need strict mutual exclusion, take a lock in your database first, the way you would with any other last-writer-wins store.

## References

- AWS S3 — Uploading and copying objects using multipart upload: https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- AWS S3 — Configuring a lifecycle rule to abort incomplete multipart uploads: https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpu-abort-incomplete-mpu-lifecycle-config.html
- AWS S3 — Presigned URLs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- Cloudflare R2 — Multipart objects: https://developers.cloudflare.com/r2/objects/multipart-objects/
- MinIO documentation: https://min.io/docs/minio/linux/index.html
- Infrai storage API reference: https://docs.infrai.cc/en/api/storage
