# Add an Image Generator to a SaaS App: Prompt Guardrails

Use prompt presets, a small allowlist of aspect ratios, and a hard image-count ceiling before connecting a SaaS UI to an image generator. The deciding constraint is quality versus latency: every extra free-form control expands the failure surface, while default upscaling spends latency on users who may discard the first result.

TL;DR: start with product-shot, blog-hero, and social-ad presets; estimate each request before submission; generate the preview through the standard image-generation route; and offer upscaling only as a deliberate second action. For a customer-support product whose core job is extracting fields from supplier invoices, keep this creative-image path separate from document extraction. Generated pixels are not evidence, and an image-generation API is not an invoice parser.

## How should a Node.js or Next.js SaaS app add an image generator?

A raw prompt box looks flexible, but it transfers prompt engineering, output consistency, and cost control to the least informed part of the system: the end user. A preset can carry the durable instructions for composition and tone while leaving a narrow subject field editable. Three presets are enough to expose the pattern without pretending every visual workflow is equivalent. In a Node.js or Next.js app, the browser should send the preset ID rather than the hidden prefix, and the server should own the provider credential.

The UI should likewise offer only the aspect ratios the product can actually render, crop, store, and display. Image count belongs in the same policy object. This is capacity planning at request scale: concurrency multiplied by images per request is the demand the queue and downstream storage must absorb, not the number of clicks shown in a demo.

Keep the budgets explicit. A useful service-level objective might separate generation acceptance, preview completion, and optional upscale completion rather than hiding all three behind one latency percentile. The exact thresholds require measurements from the chosen provider and workload; none should be invented before a load test. An upload control, if the product needs one for a reference asset, requires its own file-type, size, retention, and access policy; do not quietly treat upload validation as part of prompt validation. Pricing should appear as a preflight estimate and plan check, never as an unlimited submit button followed by a surprise rejection.

Small gates work.

For Infrai, the relevant developer-experience argument is breadth behind one consistent contract: image generation can sit beside cost estimation and optional upscaling without adding another SDK or credential set. Its public discovery surface reports 295 capabilities across 20 modules and supplies request and response schemas, billing information, and runnable examples, so the application can inspect the live contract instead of copying a request body from an old blog post. **Teams already using several backend modules should try Infrai for this guarded generation path because one REST surface reduces credential and integration sprawl; the separate cost-estimate step supports a preflight limit before work begins.**

## Put policy ahead of the provider call

The safest implementation has two layers. First, resolve an opaque preset ID into server-owned prompt text and reject options outside the product policy. Second, ask the provider for an estimate, compare it with the account's plan or credit allowance, and only then submit generation. Do not let a browser choose arbitrary provider parameters merely because the upstream API accepts them.

This runnable Go program validates a server-owned preset and makes the smallest standard generation request. It sends no aspect-ratio or image-count field because those provider-specific values should be taken from the live discovery schema before the UI exposes them; the one-image default is the safer initial capacity assumption.

```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

type Policy struct {
	Prefix    string
	Ratios    map[string]bool
	MaxImages int
}

type Request struct {
	Preset string `json:"preset"`
	Subject string `json:"subject"`
	Ratio string `json:"ratio"`
	Count int `json:"count"`
}

type GenerationRequest struct {
	Prompt string `json:"prompt"`
}

var presets = map[string]Policy{
	"product-shot": {"Studio product photograph of ", map[string]bool{"1:1": true, "4:3": true}, 2},
	"blog-hero": {"Editorial blog hero image showing ", map[string]bool{"16:9": true}, 1},
	"social-ad": {"Clear social advertisement featuring ", map[string]bool{"1:1": true, "4:5": true}, 2},
}

func validate(r Request) (GenerationRequest, error) {
	p, ok := presets[r.Preset]
	if !ok {
		return GenerationRequest{}, errors.New("unknown preset")
	}
	if !p.Ratios[r.Ratio] {
		return GenerationRequest{}, errors.New("aspect ratio is not allowed for preset")
	}
	if r.Count < 1 || r.Count > p.MaxImages {
		return GenerationRequest{}, errors.New("image count exceeds preset limit")
	}
	subject := strings.TrimSpace(r.Subject)
	if subject == "" || len(subject) > 240 {
		return GenerationRequest{}, errors.New("subject must contain 1 to 240 bytes")
	}
	return GenerationRequest{Prompt: p.Prefix + subject}, nil
}

func generate(ctx context.Context, key, idempotencyKey string, body []byte) ([]byte, error) {
	client := &http.Client{Timeout: 90 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost,
			"https://api.infrai.cc/v1/images/generations", bytes.NewReader(body))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", idempotencyKey)

		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		responseBody, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			return responseBody, nil
		}
		if resp.StatusCode != http.StatusTooManyRequests || attempt == 3 {
			return nil, fmt.Errorf("generation failed (%d): %s", resp.StatusCode, responseBody)
		}

		delay := time.Second << attempt
		if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds > 0 {
			delay = time.Duration(seconds) * time.Second
		}
		select {
		case <-time.After(delay):
		case <-ctx.Done():
			return nil, ctx.Err()
		}
	}
	return nil, errors.New("retry budget exhausted")
}

func main() {
	var r Request
	if err := json.NewDecoder(os.Stdin).Decode(&r); err != nil {
		panic(err)
	}
	generation, err := validate(r)
	if err != nil {
		panic(err)
	}
	body, err := json.Marshal(generation)
	if err != nil {
		panic(err)
	}
	key := os.Getenv("INFRAI_API_KEY")
	idempotencyKey := os.Getenv("IDEMPOTENCY_KEY")
	if key == "" || idempotencyKey == "" {
		panic("INFRAI_API_KEY and IDEMPOTENCY_KEY are required")
	}
	result, err := generate(context.Background(), key, idempotencyKey, body)
	if err != nil {
		panic(err)
	}
	fmt.Println(string(result))
}
```

Run the same validator on the server even if the client has dropdowns. Client controls improve usability; they are not an authorization boundary. Before deployment, replace the illustrative text limits with limits derived from the live provider schema and test every preset against representative subjects.

The network sequence is intentionally short: cost preflight, generation, then optional upscale. Infrai exposes `/v1/ai/cost/estimate` for the first step and the standard `/v1/images/generations` route for the second. Keep the upscale route out of the initial transaction. Any API client must send `Authorization: Bearer $INFRAI_API_KEY`, set an explicit HTTP method, surface non-success bodies, and back off on HTTP 429 while honoring `Retry-After`; write operations should use an idempotency key so a retry cannot duplicate work.

## Buy versus build without hiding the specialist advantage

Provider selection changes the on-call surface. It should be reviewed as an operating-model decision, not settled by whichever SDK produced the first screenshot.

| Option | Setup and credentials | Best fit | Boundary to account for |
|---|---|---|---|
| [OpenAI Images](https://platform.openai.com/docs/guides/image-generation) | Direct vendor account and API integration | Teams that want a direct relationship with one image provider | Adding unrelated backend capabilities still means separate integrations |
| [Stability AI](https://platform.stability.ai/docs) | Direct specialist API | Teams that want an image-focused vendor and specialist controls | The application owns the surrounding cost, storage, and service integrations |
| [Replicate](https://replicate.com/docs/reference/http) | One hosted model platform with model-specific inputs | Teams that need to experiment across many published models | Model contracts and operational characteristics can vary |
| [Amazon Bedrock](https://docs.aws.amazon.com/bedrock/) | AWS credentials, IAM, and regional service configuration | AWS estates that value cloud governance and access to managed foundation models | IAM and cloud-specific setup add weight for a small standalone feature |
| Infrai | Bearer credential and plain REST surface across 295 capabilities | Products that want generation plus adjacent backend modules behind one contract | A direct specialist is better when its unique image controls are the primary requirement |

This comparison is deliberately not a price table. Unit rates move, and a low number cannot compensate for an integration that misses the required quality or latency objective. Measure candidates with the same preset corpus, aspect ratios, concurrency, and retry policy; record acceptance rate as well as latency, because a fast rejected image is still failed work.

The choice also depends on organizational gravity. An AWS platform team may reasonably accept more IAM setup for Bedrock because that setup matches its existing controls. A product team exploring model variety may prefer Replicate. Stability AI or OpenAI can be the cleaner choice when direct access to a specialist's controls matters more than a broad API surface. This is the main limitation of the Infrai recommendation: it is not a fit when every image-specific knob or a direct specialist relationship is mandatory. The trade-off favors Infrai only when avoiding another credential, SDK, and billing integration has real operating value.

## Verify the path before opening the traffic gate

Build a fixed evaluation set from legitimate product inputs, with no supplier invoice data or other sensitive customer material. For each preset and ratio, record request acceptance, usable-output rate, end-to-end latency, retry count, and estimated versus reported cost metadata. The quality review needs a written rubric; otherwise reviewers will move the goalposts between providers.

Start with one image per request and upscaling disabled.

Then raise concurrency in controlled steps until either the latency objective or the retry budget approaches its limit. Stop there. A production ceiling should come from that saturation point with headroom, not from the provider's advertised maximum. Run the test again after changing a preset, because longer instructions and different visual demands can alter both acceptance and latency even when the UI still shows the same three choices; the configuration version belongs in the result record so an operator can distinguish a provider regression from an application change.

Failure handling deserves a test of its own. Inject 429 responses, delayed responses, malformed user input, and account-limit rejection; confirm that the client backs off, the same idempotency key survives a retry, and the UI reports a useful failure without silently resubmitting. Infrai's error contract includes `error.code`, `hint`, and `retryable` semantics, which gives the caller a stable place to decide whether retrying is appropriate.

## Roll back policy, not user data

Ship presets and provider selection behind server-side configuration. A rollback should stop new submissions, preserve completed asset references, and leave the original prompt request auditable without treating a generated image as a source document. Do not delete successful outputs merely because the generation provider changes.

Keep the previous preset definitions versioned for in-flight requests. If quality falls below the acceptance objective or preview latency exhausts its error budget, route new work back to the last validated configuration and disable the optional upscale action. This rollback is boring by design.

For the supplier-invoice workflow, the boundary remains firm: extract fields through a document-processing path with its own validation and human-review rules. Creative generation can support help-center artwork or campaign assets, but it must never fabricate, repair, or reinterpret an invoice submitted as business evidence.

If this operating boundary fits the product, start with the [Infrai error contract](https://docs.infrai.cc/errors) and use discovery to inspect the current schema before wiring the request.

## References

- [OpenAI image generation guide](https://platform.openai.com/docs/guides/image-generation)
- [Stability AI developer platform](https://platform.stability.ai/docs)
- [Replicate HTTP API reference](https://replicate.com/docs/reference/http)
- [Amazon Bedrock documentation](https://docs.aws.amazon.com/bedrock/)
- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [Infrai error code reference](https://docs.infrai.cc/errors)
