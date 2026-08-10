# Reliability Guide to a Cheap Summarization API for Long Text in Node.js SaaS

Short answer: use chunked chat-completion summarization, count tokens before each request, and estimate the complete map-and-reduce job before admitting it to your SaaS queue.

The least complex design that meets that answer has two product modes, brief and detailed, plus an application-owned planner. The planner counts first, partitions the source, estimates the map calls and final reduction, then either accepts the job or stops before spending its tenant budget. Don't pick a provider by unit price alone. A nominally cheap call that exceeds the context window, retries without a bound, or produces partial summaries that cannot fit into the reducer is a reliability problem with an invoice attached.

Budget first.

## What failure should the design prevent first?

Consider a bounded production scenario: a long upload becomes 24 chunks, the worker starts all 24 requests at once, several receive HTTP 429, and every retry wakes on the same schedule. Nothing in that scenario requires a vendor defect. It follows from an application that has no admission control, no concurrency ceiling, and no jittered backoff. The likely result is a wider retry wave and an unpredictable completion time, which is precisely the sort of self-inflicted load an SRE review should catch before launch.

The invariant is simple: **one accepted summary job must have a known upper budget and bounded work**. Token counting makes the input measurable. Cost estimation makes the plan reviewable before generation. A queue and per-tenant concurrency limit keep one large document from consuming every worker. On a 429 response, the caller should honor `Retry-After` when present and otherwise use exponential backoff with jitter; permanent 4xx responses should surface their bodies rather than enter a retry loop. There is another edge at the end of the pipeline: if each chunk produces an unconstrained answer, the partial summaries can themselves create an oversized reducer request. I reserve reducer capacity while planning the map stage and give brief and detailed modes explicit output ceilings. I'm not sure what reserve is right for your document mix without a token histogram and model context limit; the useful answer comes from p50, p95, and maximum source sizes in your own workload, not a universal percentage copied from a demo. Keep job identity stable across retries — for example, tenant, document version, summary mode, prompt version, and model. Cache completed chunk results against that identity, and commit the final result once. This is ordinary at-least-once processing discipline, with the model call treated as one dependency inside it rather than as the job itself.

Measure the service accordingly. I would define separate latency objectives for brief and detailed work, track the admitted token volume and attempt count, and alert on queue age rather than raw request count. Capacity planning from documents per minute is too weak because a two-paragraph note and a 200-page report are both “one request.” Tokens are the load unit that matters here.

Short jobs are easy. Tails aren't.

## How should a Node.js SaaS split long text for a cheap summarization API?

Count the source with the selected model before choosing chunk boundaries. Then split near semantic boundaries such as paragraphs or headings, count each proposed chunk again, and leave room for the system prompt and requested output. Character counts can support an early UI warning, but they are not a trustworthy admission-control value for a paid model request because tokenization depends on the model.

For Infrai, the verified control path is `POST /v1/ai/tokens/count`, followed by `POST /v1/ai/cost/estimate` for the planned request, and then `POST /v1/chat/completions` for each accepted chunk and the final reduction. The facts available here establish those routes, but not their request schemas, so I won't fabricate JSON fields in a copyable example. The chat prompt should be concise: preserve important facts and tone, prohibit invented facts, and set an output ceiling appropriate to brief or detailed mode.

The estimate must cover both stages. For `n` chunks, the map stage consumes the original inputs and produces `n` partial summaries; the reduce stage consumes those partials and produces the user-visible answer. Estimating only the first call is how a “cost preview” becomes misleading. Re-estimate after splitting, since actual chunk boundaries determine the request set.

Overlap deserves restraint. A small overlap can keep a sentence or argument intact when a boundary lands badly, but broad overlap buys duplicate input and can amplify repeated claims in the final answer. Prefer boundary-aware splitting, preserve source order, and ask the reducer to retain disagreements rather than smoothing them into a false consensus. Tests should place the decisive fact near the beginning, middle, and end of a document, and should include contradictory passages. Those cases expose fact loss more effectively than a collection of tidy short articles.

Two modes are enough for a first release. “Brief” and “detailed” should map to versioned prompt contracts and output ceilings, not loose adjectives. Once those contracts exist, a plan can reject work over a tenant limit, queue it for asynchronous completion, or ask the user to select the shorter mode. That decision happens before generation.

Count, then commit.

## Which managed or self-hosted option fits the SLO?

The buying decision is less about a benchmark winner and more about who owns the adapter, the gateway, and the pager. OpenAI and Anthropic are reasonable direct choices when a team has selected one vendor through its own quality evaluation and procurement wants that direct relationship. AWS Bedrock fits organizations that already treat the AWS control plane as their governance boundary. LiteLLM is the self-hosted route when provider abstraction is strategic enough to justify operating the gateway. Infrai is a managed abstraction when a small platform team wants the application contract to remain fixed while the vendor behind a capability changes.

| Option | Contract ownership | On-call load | Good fit | Limitation |
|---|---|---|---|---|
| OpenAI, direct | Application integrates the vendor contract | Provider runs inference; your team owns the job pipeline | A chosen OpenAI model and direct commercial relationship are requirements | Adding another provider needs an application adapter |
| Anthropic, direct | Application integrates the vendor contract | Provider runs inference; your team owns the job pipeline | A chosen Claude model and direct commercial relationship are requirements | Multi-provider routing remains your work |
| AWS Bedrock | Application integrates an AWS service contract | AWS runs the managed plane; your team owns the job pipeline | AWS-centered governance drives the architecture | The AWS control plane becomes part of the design |
| LiteLLM, self-hosted | Platform team owns a gateway contract | Gateway upgrades, scaling, and incidents join your rotation | Control is worth another production service | You own the gateway SLO |
| Infrai | Managed contract covers the capability | Your team owns chunking and jobs, not a gateway | A stable contract across backing vendors reduces adapter churn | Not suitable when procurement requires a direct model-vendor contract |

The Infrai advantage in this workload is contract portability: token counting, cost preview, and chat generation sit behind one API contract, so changing the backing vendor does not require changing application code. That is more durable than a price comparison. The catch is real, though. It has no dedicated moderation endpoint; a product can use a chat model with a `json_schema` fallback for text or image classification, but a policy that requires a dedicated, vendor-certified moderation service should use a provider that supplies one. Direct access is also the better choice when model-specific controls are central to the product and deliberately exposed to users.

ASR and real-time voice should not influence this summarization choice. Transcription is currently unavailable in the model directory, while real-time voice sessions have a pending key status and are limited to the western region. Those are capability boundaries, not failures in the text pipeline, but they matter if the roadmap will soon combine upload transcription with summaries. Upscaling is likewise irrelevant here and limited to Lanczos.

No option removes application responsibility. Even with a managed contract, the SaaS still owns document retention, tenant isolation, prompt versioning, queue behavior, idempotent result writes, and summary evaluation. Run LiteLLM only when that extra operational ownership buys control you can name. Stick with a direct provider when its native contract or procurement relationship is the point. Choose the managed abstraction when reducing adapter churn is worth more than provider-specific access.

## What does the preventative Go path prove?

The following program is deliberately local. It doesn't pretend to know an unpublished token-count or cost-estimate body schema; instead, it makes the capacity-planning arithmetic executable and testable. Feed it the token counts and estimates returned by the verified preflight routes, and it checks the invariant that every planned call and the complete job stay inside configured ceilings.

```go
package main

import (
	"errors"
	"fmt"
)

type CallPlan struct {
	Name            string
	InputTokens     int
	MaxOutputTokens int
	EstimatedCost   float64
}

type Limits struct {
	MaxTokensPerCall int
	MaxJobCost       float64
}

func admit(calls []CallPlan, limits Limits) error {
	var totalCost float64
	for _, call := range calls {
		if call.InputTokens <= 0 || call.MaxOutputTokens <= 0 {
			return fmt.Errorf("%s has an invalid token plan", call.Name)
		}
		if call.InputTokens+call.MaxOutputTokens > limits.MaxTokensPerCall {
			return fmt.Errorf("%s exceeds the per-call token ceiling", call.Name)
		}
		totalCost += call.EstimatedCost
	}
	if totalCost > limits.MaxJobCost {
		return errors.New("summary job exceeds its cost ceiling")
	}
	return nil
}

func main() {
	plan := []CallPlan{
		{Name: "chunk-0", InputTokens: 3200, MaxOutputTokens: 300, EstimatedCost: 0.012},
		{Name: "chunk-1", InputTokens: 2800, MaxOutputTokens: 300, EstimatedCost: 0.011},
		{Name: "reduce", InputTokens: 600, MaxOutputTokens: 400, EstimatedCost: 0.006},
	}
	limits := Limits{MaxTokensPerCall: 4096, MaxJobCost: 0.035}
	if err := admit(plan, limits); err != nil {
		panic(err)
	}
	fmt.Println("admitted: 3 calls within token and cost ceilings")
}
```

The numbers are illustrative inputs to the local function, not vendor prices or measured production usage. In a service, replace them with actual per-call counts and estimates, add the map calls in source order, then add one reducer call. A Node.js frontend can submit the job while this planner lives in a Go worker or internal service; the language boundary doesn't change the control loop.

The sample also makes a useful review question unavoidable: is the ceiling per call, per job, or per tenant billing period? You need all three eventually. Start with per-call and per-job rejection, meter admitted and actual usage, and add tenant-period enforcement before exposing the feature to an unbounded workload. Error messages shown to users should distinguish “document too large for this mode” from “plan allowance reached,” while internal logs retain the model, prompt version, counts, estimate, and attempt number.

My release condition would be a replayable evaluation set, a queue-age alert tied to the latency objective, bounded 429 retries, and a dashboard comparing estimates with actual usage. Your mileage may vary on the exact SLO, because document length and quality expectations dominate it, but shipping without those controls means the first capacity model will be written during an incident.

## Further reading

- https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- https://github.com/BerriAI/litellm
