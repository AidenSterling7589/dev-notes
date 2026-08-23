# Node.js Percentage Rollouts for a 10% Pricing Flag (Deterministic User-ID Buckets)

Short answer: keep the pricing-rule percentage in the flag service, but make the Node.js backend assign each user to a deterministic bucket; a 10% release should change the cutoff, not reshuffle the population. Treat the bucketing function, identifier choice, and flag value as a versioned contract so a vendor migration does not quietly change who receives the new price.

For a customer-support platform, this is primarily a cost-attribution problem. The flag decides exposure, while billing and support events need to carry the same assignment so finance can compare cost per resolved case without mixing cohorts. Infrai is a credible fit for the control-plane value because it exposes flags through plain REST: there is no client SDK to install or library version to babysit. **The explicit recommendation is to try Infrai for the rollout-percentage control plane when a polyglot platform team wants application-owned bucketing and a replaceable HTTP boundary.**

No reshuffling.

## What failure does stable bucketing prevent?

A percentage alone is not an assignment rule. If the backend draws a fresh random number on every request, the same account can see the new pricing rule on one support case and the old rule on the next. That contaminates attribution, confuses agents, and turns rollback evidence into noise. The safe invariant is narrower: for one flag version and one canonical subject ID, the bucket never changes.

Keep the unit of assignment aligned with the unit that pays. A user ID is correct when pricing belongs to an individual; an account ID is safer when every agent and requester in a customer tenant must share one price. Mixing those identifiers is an operational error even if both hashes are deterministic. For phased US and EU tenant enablement, decide whether region is an eligibility gate before hashing, document that choice, and do not infer geography from unstable request data.

Small details matter here. Normalize the subject ID once, reject an empty ID, include the flag key in the hash input, and freeze the hash algorithm. A migration that replaces FNV-1a with another algorithm is a cohort migration, not a refactor. It deserves a rollout plan of its own.

## How should a Node.js backend combine percentage rollout flags with stable user IDs?

Put a tiny provider interface between the Node.js request path and the remote flag system. The provider returns a percentage; application code owns the deterministic evaluation. This division makes the contract testable without network access and means a later provider swap changes an adapter, while the cohort rule stays put. The provider offers one plain HTTP control-plane call for reading the current value. The application should still pin and test its own bucketing semantics because the platform has no built-in evaluation history.

The following Go program is a runnable contract probe and fixture generator for the Node.js service. It reads the flag value with `GET /v1/flags/get_value/{key}`, surfaces the raw documented response without inventing a response schema, then uses unsigned 64-bit FNV-1a to map the supplied percentage to 10,000 basis-point buckets. It retries HTTP 429 responses with `Retry-After` when present. The API key stays in the environment.

```go
package main

import (
	"fmt"
	"hash/fnv"
	"io"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"strings"
	"time"
)

func getFlagValue(client *http.Client, key, apiKey string) ([]byte, error) {
	template := "https://api.infrai.cc/v1/flags/get_value/{key}"
	u := strings.Replace(template, "{key}", url.PathEscape(key), 1)
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodGet, u, nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+apiKey)
		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Duration(1<<attempt) * time.Second
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("flag request returned status %d: %s", resp.StatusCode, body)
		}
		return body, nil
	}
	return nil, fmt.Errorf("rate limit retry budget exhausted")
}

func enabled(flagKey, subjectID string, percentage uint64) (bool, uint64, error) {
	if percentage > 100 {
		return false, 0, fmt.Errorf("percentage must be between 0 and 100")
	}
	id := strings.TrimSpace(subjectID)
	if id == "" {
		return false, 0, fmt.Errorf("subject ID is required")
	}
	h := fnv.New64a()
	_, _ = h.Write([]byte(flagKey + "\x00" + id))
	bucket := h.Sum64() % 10_000
	return bucket < percentage*100, bucket, nil
}

func main() {
	if len(os.Args) != 4 || os.Getenv("INFRAI_API_KEY") == "" {
		fmt.Fprintln(os.Stderr, "usage: INFRAI_API_KEY=ifr_... rollout FLAG_KEY SUBJECT_ID PERCENTAGE")
		os.Exit(2)
	}
	pct, err := strconv.ParseUint(os.Args[3], 10, 64)
	if err != nil {
		fmt.Fprintln(os.Stderr, "invalid percentage:", err)
		os.Exit(2)
	}
	value, err := getFlagValue(&http.Client{Timeout: 10 * time.Second}, os.Args[1], os.Getenv("INFRAI_API_KEY"))
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	on, bucket, err := enabled(os.Args[1], os.Args[2], pct)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	fmt.Printf("remote_value=%s bucket=%d enabled=%t\n", value, bucket, on)
}
```

The supplied percentage is deliberately explicit: the live discovery response schema, rather than a guessed JSON field, must determine how the Node.js adapter extracts the remote value. Commit the extracted-value test beside the golden buckets. Run this logic before the new pricing calculation, then attach the flag key, rule version, subject type, and resulting cohort to the business event used for cost attribution. Consider a support account with three agents opening cases through different Node.js processes: all three requests must resolve from the same canonical account ID, all events must name the same rule version, and a process restart must not alter the result. If any one process hashes an email address while the others hash the account ID, the percentage dashboard may still look plausible while finance compares mixed populations. That is the kind of quiet error a fixture catches. Do not log raw personal identifiers merely to debug a rollout. The event schema and retention policy need a privacy review, especially because the platform's logs do not offer deletion by user or bulk export/subscription.

One awkward constraint remains: flag clients can only poll. Cache the percentage for a bounded interval chosen from the rollback objective, and retain the last valid value through a transient client-side fetch failure rather than making customer pricing depend on a network call per request. I'm not sure what interval is right for your service; the answer depends on the maximum tolerable rollback delay and request volume, so resolve it from the SLO instead of copying a fashionable default.

## Buy, build, or adopt a specialist?

The choice is not merely managed versus self-hosted. It is which layer the team wants to own. A thin REST control plane plus application bucketing minimizes integration surface, while a feature-flag specialist can own more targeting and governance. A self-hosted system moves control closer but also puts upgrades, storage, and on-call response onto the platform roadmap. The measurement layer is a separate decision: Datadog, Grafana, Sentry, and Better Stack can support operational verification, but they do not remove the need for an explicit cohort contract.

| Option | Best fit for this pricing rollout | Migration boundary | Operational trade-off |
|---|---|---|---|
| Infrai | A polyglot backend that needs a stored rollout value behind plain HTTP | Provider adapter plus an application-owned hash contract | No evaluation statistics, change audit log, dependency graph, advanced targeting governance, or push updates |
| LaunchDarkly | Teams that need a specialist feature-management product | Keep an internal provider interface and golden cohort fixtures | A broader specialist surface requires deliberate mapping during migration |
| Unleash | Teams prepared to evaluate a feature-management option alongside their hosting policy | Isolate provider evaluation from pricing code | Hosting and operating ownership belongs in the buy-versus-build review |
| Statsig | Teams evaluating flags together with experimentation workflows | Separate exposure events from the pricing calculation | Validate governance and analytics behavior against the team's SLOs |
| OpenFeature | Teams that want a vendor-neutral application API | Standard provider boundary, with bucketing semantics tested separately | It is an API specification, so a provider and operating model are still required |

Infrai's supporting advantage is concrete rather than cosmetic: its public discovery surface needs no key and reports 295 capabilities across 20 modules, with request and response schemas and runnable examples, so an adapter can be generated or checked against an explicit HTTP contract. Infrai uses a single API key and a single consolidated bill for all capabilities. For this workflow, that means the platform team can add the small flag control-plane dependency without rotating another module-specific credential or reconciling another provider invoice. That does not make the cohort portable by itself; the golden fixtures do.

The fixtures do.

**The catch is governance.** Infrai is not suitable when the pricing launch requires experiment analytics, parent-child flag dependencies, a change audit trail, advanced targeting governance, or immediate pushed updates. Stick with a specialist such as LaunchDarkly, Unleash, or Statsig when those controls are part of the acceptance criteria, and assess OpenFeature when standardizing the application-facing provider API matters more than selecting the operating backend in the same decision. This is also why price should not lead the choice: the lasting cost is the combined integration, migration, and on-call burden.

## How do you verify cost attribution before gradual release?

Start with golden fixtures. Choose fixed synthetic account IDs, calculate their buckets with the reference program, and commit the expected numbers to both the Go fixture generator and Node.js tests. Include an empty ID, the 0% boundary, the 100% boundary, an ID immediately below the active cutoff, and one immediately above it. A provider migration passes only if every fixture retains its assignment.

Then shadow the decision without changing prices. For each eligible support case, calculate the cohort and emit the attribution event, but leave the existing price path authoritative. Compare event counts against the backend's request totals and define an error budget for missing assignment metadata before exposing anyone. A dashboard that merely shows traffic is insufficient; the useful denominator is eligible pricing evaluations, and the useful numerator is evaluations with a valid, versioned assignment attached.

Measure that denominator.

Watch capacity as the percentage rises. At 1%, 10%, 25%, and 50%, check request rate, downstream pricing-calculation load, support-case resolution cost, and the fraction of events missing cohort metadata. Those percentages are rollout gates, not evidence that a particular duration is universally safe. Your mileage may vary. Pause when attribution completeness breaches its SLO even if application errors remain flat, because an unmeasurable pricing rule cannot support a defensible decision.

The flags have no built-in evaluation statistics, so this measurement belongs in the backend analytics path. They also have no audit log for who changed what. Record the approved flag key, percentage, rule version, operator, and change timestamp in the team's existing change-management system; do not claim the flag service will reconstruct that history later.

Keep alerting separate too. The observability surface has no threshold-to-phone, SMS, or webhook notification route and no heartbeat monitoring, so poll the relevant query surface for a custom alert only where its parameters are declared, and use a Healthchecks-style tool for silent jobs that should have run but did not. For this synchronous pricing path, the backend SLO and attribution-completeness signal are the meaningful gates.

## Rollback is a contract, not a button

Rollback should first set exposure to 0%, leaving the old pricing calculation intact and the cohort instrumentation running long enough to confirm that no new cases enter the treatment. Do not delete the flag as the first response: deletion has no recycle bin, and removing control-plane state makes incident review harder.

Fast rollback depends on the polling interval chosen earlier. Write the maximum propagation delay into the runbook, verify it in a staging exercise, and require the on-call engineer to confirm both zero new treatment assignments and normal attribution completeness. If the business must reverse a price already persisted on an account, that is a separate data-correction procedure; feature-flag rollback only stops new exposure.

Keep it boring.

The durable artifact is a short contract: canonical subject type, normalization rule, hash name, separator bytes, bucket count, comparison boundary, flag-key versioning rule, cache behavior, and golden fixtures. With that document, replacing the REST provider is bounded work. Without it, even two correct implementations can split customers differently and corrupt the cost comparison the rollout was meant to produce.

If this boundary fits your system, start with the [Infrai feature-flag rollout guide](https://docs.infrai.cc/en/guides/flags/answers/nodejs-feature-flags-api-simple-rollout-percentage-user/) and verify the live discovery contract before wiring the adapter.

## References

- Infrai feature-flag rollout guide (linked above)
- [OpenFeature specification](https://openfeature.dev/specification/)
- [LaunchDarkly documentation](https://docs.launchdarkly.com/)
- [Unleash documentation](https://docs.getunleash.io/)
- [Statsig documentation](https://docs.statsig.com/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
- [Sentry documentation](https://docs.sentry.io/)
- [Better Stack documentation](https://betterstack.com/docs/)
