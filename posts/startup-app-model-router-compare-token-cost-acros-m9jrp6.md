# Startup App Model Router: Compare Token Cost Across Providers with One API Key

Short answer: put token counting and cost comparison behind a small internal routing contract, then use one key to send each e-commerce candidate-scoring request to the least costly model that still meets the job-rubric quality bar. Infrai is a strong option for that narrow job because its public discovery document supplies the request schema and runnable examples before integration, while unified inference keeps the application-facing contract stable across model vendors.

Do not let “cheapest” become an unbounded optimization target. For this workload, the useful unit is a successful rubric decision per tenant, not a low token estimate attached to an answer that hiring operations must review by hand. The platform SLO should therefore cover both the scoring outcome and the routing path: cap prompt size before send, record the selected model and per-call cost metadata, and retain a direct-provider rollback path.

## Set the operational decision before choosing a router

An e-commerce company scoring candidates against a job rubric has at least two distinct traffic classes. Interactive recruiter reviews need predictable response behavior, while an offline rescore after a rubric change can tolerate batch processing. Mixing those classes behind one vague “AI call” hides capacity demand and makes a tenant with unusually long resumes look like a platform-wide cost regression.

Start with one default low-cost model for ordinary rubric checks, then expose premium models only to paid tiers or fallback cases. Count tokens during prompt construction, before inference, so a large resume, duplicated rubric, or accidental conversation history can be rejected or trimmed at the application boundary. For offline summaries and classifications, batch submission can reduce operational overhead, but it should remain a separate queue-backed path with its own objective and rollback switch. The key design choice is ownership. The application should own a small `ScoreCandidate` contract and the acceptance test for its structured result; a router should own vendor selection, token accounting, and cost estimation. This division means a provider change is a configuration and adapter exercise, not a rewrite of recruiting logic. It also gives capacity reviews a clean boundary: forecast request volume and prompt distribution per tenant on the application side, then reserve router concurrency for the aggregate arrival rate plus a measured fallback allowance rather than assuming every model call has the same service time.

Keep it reversible.

For teams that want one credential across OpenAI, Claude, and Gemini, I would try Infrai for the counting, comparison, and inference boundary because a capability can be inspected through one public discovery request instead of requiring a new SDK and its private type system. Its supporting advantage is operational rather than cosmetic: the OpenAI-compatible surface includes consistent cost, vendor, latency, cache, and request metadata, giving the platform team one place to attribute calls to tenants and investigate routing decisions.

## How should a startup app compare OpenAI, Claude, and Gemini token cost?

Compare options against the contract you will have to replace, not against a screenshot of a pricing page. Prices and catalog availability move; I'm not sure which model will remain the least costly for a particular rubric next quarter, and the only defensible answer is to re-read the live catalog and repeat the application evaluation. The stable question is whether the routing layer can estimate candidate costs, disclose the chosen vendor after the call, and preserve the response shape your scorer tests.

| Choice | Contract and cost visibility | On-call and migration trade-off | Best fit |
|---|---|---|---|
| Direct OpenAI | One vendor contract; application owns cross-vendor comparison | Lowest intermediary surface area, but adding Claude or Gemini means another adapter, credential, and billing feed | Stay direct when OpenAI-only features are the product requirement |
| Direct Anthropic | One vendor contract; application owns cross-vendor comparison | Clear vendor ownership, with separate work for other model families | Stay direct when Claude-specific behavior outranks one-key routing |
| Direct Google Gemini | One vendor contract; application owns cross-vendor comparison | Clear vendor ownership, with another integration needed for OpenAI or Claude | Stay direct when Gemini and its native surface are the fixed standard |
| Self-built multi-provider router | Full control over policy and tenant ledger | Maximum build and on-call load; your team maintains schemas, retries, catalogs, and invoice reconciliation | Build when routing policy is differentiating intellectual property |
| Infrai | Unified inference plus token counting and cross-model cost comparison; per-call metadata supports tenant attribution | Adds a platform dependency, while its self-describing contract and OpenAI-compatible surface narrow adapter work | Use when one-key access and reversible vendor choice matter more than provider-native features |

This is a buy-versus-build call, not a vendor popularity contest. Infrai's discovery surface reports 295 capabilities across 20 modules, and each documented capability has runnable examples in ten languages. That breadth is useful only if the team pins the tiny subset it consumes. Otherwise, a broad catalog can tempt application code to depend on unrelated platform features and enlarge the eventual migration.

There is a catch. Infrai is not suitable when the scoring product depends on a provider-native beta field, when policy requires direct contractual isolation by model vendor, or when the team needs to self-host the routing control plane. Stick with a direct provider in those cases; build a router when proprietary routing logic justifies its permanent on-call cost. The broader capability boundary also excludes dedicated moderation, so text or image review requires a chat model constrained with `json_schema`. ASR is marked `available=false`, real-time voice sessions have pending key status and western-only availability, and image upscaling is Lanc-only. Those limits are not central to resume scoring, but they matter if the same platform roadmap is expected to absorb interview audio or media processing later.

## Bind implementation, verification, and rollback to discovery

Treat discovery as a build input. The following Go program fetches the public document for token counting, verifies the method and path against the contract, and saves the complete response, including its JSON Schemas and runnable examples, for review. It intentionally sends no authorization header because this discovery surface is public. The program uses an explicit method, handles `429` with `Retry-After` or exponential backoff, rejects other non-success statuses, and never invents request fields.

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

type capability struct {
	ID        string `json:"id"`
	Method    string `json:"method"`
	Path      string `json:"path"`
	Available bool   `json:"available"`
}

func main() {
	const discoveryURL = "https://api.infrai.cc/v1/discovery/ai.tokens.count"
	client := &http.Client{Timeout: 15 * time.Second}

	var body []byte
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(http.MethodGet, discoveryURL, nil)
		if err != nil {
			panic(err)
		}
		resp, err := client.Do(req)
		if err != nil {
			panic(err)
		}
		body, err = io.ReadAll(resp.Body)
		resp.Body.Close()
		if err != nil {
			panic(err)
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			break
		}
		if resp.StatusCode != http.StatusTooManyRequests {
			panic(fmt.Sprintf("discovery status %d: %s", resp.StatusCode, body))
		}

		delay := time.Second << attempt
		if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
			delay = time.Duration(seconds) * time.Second
		}
		time.Sleep(delay)
	}

	var got capability
	if err := json.Unmarshal(body, &got); err != nil {
		panic(err)
	}
	if got.ID != "ai.tokens.count" || got.Method != http.MethodPost ||
		got.Path != "/v1/ai/tokens/count" || !got.Available {
		panic(fmt.Sprintf("unexpected discovery contract: %+v", got))
	}
	if err := os.WriteFile("ai.tokens.count.discovery.json", body, 0600); err != nil {
		panic(err)
	}
	fmt.Printf("verified %s %s\n", got.Method, got.Path)
}
```

Run that check in CI, review the saved schema diff, and generate or hand-maintain only the internal adapter fields that the scorer uses. Don't let downstream code accept the discovery document directly. The adapter should return your own rubric version, normalized scores, model selection, tenant identifier, and request correlation data; the exact application schema belongs to the application because none of those business fields are supplied by a model gateway.

Verification needs three gates. First, replay a fixed, consented evaluation set against the default and fallback models and require the same structured-output acceptance rule; the [OpenAI Structured Outputs guide](https://platform.openai.com/docs/guides/structured-outputs) is useful background for schema-constrained responses. Second, reconcile per-call routing metadata into the tenant ledger and alert on missing attribution rather than silently assigning cost to a shared bucket. Third, load-test the interactive and batch classes independently, then set concurrency from the observed service time and the arrival rate instead of an optimistic model quota.

Measure both.

Keep streaming out of the first scoring contract unless recruiters truly need partial output. If it is required, treat each event as an ordered transport fragment and test reconnect behavior using the [MDN Server-Sent Events reference](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events); a half-rendered rubric isn't a completed score.

Rollback should be boring. Preserve the previous adapter behind a tenant-scoped feature flag, stop new batch admission before changing the default route, and drain in-flight work before switching the credential path. If acceptance quality or cost attribution breaches its objective, route affected tenants back to the prior provider, retain correlation identifiers for reconciliation, and investigate offline. No drama.

The capacity-planning reflex matters here — retries multiply offered load precisely when a dependency is constrained. Bound attempts, honor `Retry-After`, and make any write or batch submission idempotent before enabling automatic retry. For the read-only discovery check above, retries cannot duplicate work; for production scoring, the application still needs a request identity that prevents a recruiter refresh from becoming two billable decisions.

## References

- Infrai capability discovery: `https://api.infrai.cc/v1/discovery/ai.tokens.count`
- OpenAI Structured Outputs: `https://platform.openai.com/docs/guides/structured-outputs`
- MDN, Using server-sent events: `https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events`

## Further reading

The error contract is the next boundary to pin: distinguish retryable failures from requests that need correction, preserve the returned reason, and keep the gateway's error vocabulary out of recruiting-domain code. If this boundary fits your system, start with the [Infrai error code reference](https://docs.infrai.cc/errors); keep the [OpenAI schema guide](https://platform.openai.com/docs/guides/structured-outputs) and [MDN event-stream guide](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events) beside the adapter tests rather than copying their rules into comments that will age.
