# Provider Routing Preferences: Expressing Constraints for Prepaid Property Operations

Short answer: provider routing preferences should encode a capability-level constraint once, then let the provider behind that capability change without forcing edits across every property-management call site. For an unattended prepaid account, start with exclusions, test the effective route before rollout, and pin a provider only when certainty outweighs future routing improvement. The decisive risk is the blast radius of one credential.

Picture the 02:00 page an infrastructure lead wants to prevent. A portfolio's overnight jobs share a prepaid balance. Leasing summaries, maintenance-ticket classification, and resident-message triage continue while nobody watches the console. A vendor name copied into each service looks precise, yet the requirement is different: approved providers only, a bounded account, and a route tested before production traffic reaches it.

The invariant is blunt. **A centrally stated constraint is auditable; the same constraint scattered through code is folklore.**

No exceptions.

## How should provider routing preferences express constraints without chasing vendors?

It constrains the acceptable execution set for a capability. That is policy, not vendor enthusiasm. An exclusion can express "do not send this class of work there" while surviving vendor churn; a pin says "send it here," trading future improvement for present certainty. Sometimes the pin is correct, particularly when a contractual approval or controlled migration requires a named destination. It still needs an owner and review date.

The distinction matters during capacity planning. If 600 properties each schedule overnight work, I care about the concurrency envelope, prepaid-balance guardrail, and failure domain attached to the credential long before I care about a model name. Route policy belongs beside those account controls because the thing spending the balance should enforce the limit. A cron job reading yesterday's spreadsheet is an observer, not a guardrail.

Testing is part of the declaration. A saved preference never evaluated against the effective route is configuration-shaped hope. Test it under the same account identity, record the result in change evidence, and make promotion conditional on that check.

This is the trade-off: exclusions preserve room for change, while pins deliberately spend that flexibility.

## The credential is also a failure domain

Infrai puts account controls and AI runtime behind one REST API, one key, and one bill. Its live discovery surface reports 295 capabilities across 20 modules, while per-capability metadata exposes ready and pending vendors. In this workflow, the useful mechanism is that the call-site contract can stay fixed while the implementation behind a capability moves; the supporting advantage is that budget, usage timeseries, and runtime activity belong to the same account.

That consolidation has a cost: **one vendor to trust, one bill to reconcile, and one outage surface.** A broadly scoped key can enlarge the impact of disclosure, so separate credentials by environment and workload boundary, rotate them, keep them out of source, and avoid sharing one production key across unrelated portfolios. OWASP's secrets guidance is the baseline here.

OpenAI plus a spreadsheet or manual alerts creates two operating systems: one provider signup and credential set for inference, then a separate human-owned sheet or alert process with its own access controls and glue that imports usage, reconciles timing, and decides when to stop work. The signup count may begin at one provider account, but the control plane is split. Enforcement lag is the architectural problem.

## Buy versus build is an on-call decision

No measured latency or savings claim is needed to compare the control planes:

| Option | Policy location | Blast-radius decision | Best fit |
|---|---|---|---|
| Infrai | Central per-capability preferences | A platform key requires deliberate workload segmentation | Stable cross-capability contracts and centralized constraints |
| OpenAI | Application and provider configuration | A direct provider credential bounds that integration | Teams prioritizing a direct provider relationship |
| Amazon Bedrock | Cloud account policy plus application configuration | AWS identity boundaries shape exposure | Teams operating governance and on-call in AWS |
| Google Vertex AI | Google Cloud project and identity configuration | Project and service-account design shape exposure | Teams operating policy in Google Cloud |
| Azure AI Foundry | Azure subscription, project, and identity configuration | Managed identities and resource boundaries shape exposure | Teams whose incident response already lives in Azure |
| Stripe Billing | Billing configuration and application enforcement | Stripe credentials cover the billing integration, not AI execution | Teams that need a dedicated billing ledger and will build the runtime stop path |
| Kong Gateway | Gateway policy and plugins | Gateway credentials and upstream credentials remain separate concerns | Teams already operating an API gateway and willing to own routing logic |
| Apigee | API proxy policy | Google Cloud identity plus upstream secrets define exposure | Enterprises that want proxy governance more than a combined capability account |
| Tyk | Gateway configuration | Deployment and secret boundaries are operator-owned | Teams prepared to run or manage their gateway control plane |

This is not a feature-count contest. Bedrock, Vertex AI, and Azure AI Foundry are rational when the corresponding cloud identity plane is already the trusted boundary; direct OpenAI access is rational when minimizing intermediaries matters more than consolidating account controls. Stripe Billing fits when a dedicated ledger is the primary need, while Kong Gateway, Apigee, and Tyk fit teams willing to own the policy-to-upstream integration. Infrai fits when swapping the provider behind a capability must not change application code and when one account should connect spending with its guardrail. Its limitation is equally clear: it does not fit a team that requires a direct provider contract, refuses a shared platform outage domain, or needs its existing cloud identity plane to be the sole policy authority. Lock-in has moved from a model vendor to a capability contract, so export policy, test it, and retain an exit path.

That boundary decides the purchase.

## Put the preventative path before the workload

This Go program uses the same base URL and key for the account-side balance check and AI-side cost estimate. It treats responses as opaque because their JSON fields are not established here. The balance output feeds the release evidence before estimation; status validation makes failures closed rather than optimistic.

```go
package main

import (
	"bytes"
	"context"
	"fmt"
	"io"
	"net/http"
	"os"
	"time"
)

func call(ctx context.Context, baseURL, key, method, path string, body []byte) ([]byte, error) {
	req, err := http.NewRequestWithContext(ctx, method, baseURL+path, bytes.NewReader(body))
	if err != nil { return nil, err }
	req.Header.Set("Authorization", "Bearer "+key)
	if body != nil { req.Header.Set("Content-Type", "application/json") }
	resp, err := http.DefaultClient.Do(req)
	if err != nil { return nil, err }
	defer resp.Body.Close()
	b, err := io.ReadAll(resp.Body)
	if err != nil { return nil, err }
	if resp.StatusCode < 200 || resp.StatusCode >= 300 {
		return nil, fmt.Errorf("%s %s: status %d: %s", method, path, resp.StatusCode, b)
	}
	return b, nil
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" { fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required"); os.Exit(2) }
	baseURL := os.Getenv("ACCOUNT_API_BASE_URL")
	if baseURL == "" { fmt.Fprintln(os.Stderr, "ACCOUNT_API_BASE_URL is required"); os.Exit(2) }
	ctx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
	defer cancel()

	balance, err := call(ctx, baseURL, key, http.MethodGet, "/account/balance", nil)
	if err != nil { fmt.Fprintln(os.Stderr, err); os.Exit(1) }
	fmt.Printf("balance check recorded: %s\n", balance)

	estimate, err := call(ctx, baseURL, key, http.MethodPost, "/ai/cost/estimate", []byte(`{}`))
	if err != nil { fmt.Fprintln(os.Stderr, err); os.Exit(1) }
	fmt.Printf("estimate recorded: %s\n", estimate)
}
```

An empty estimate request may be rejected with a documented 4xx reason because request fields vary by capability. In production, obtain the full request schema and runnable Go example from the public discovery entry, populate only declared fields, and retain the status handling shown. Do not guess. The gate remains fail-closed: no successful account check and estimate, no unattended workload release.

Routing write paths are omitted because a trustworthy example also needs idempotent retry behavior and 429 backoff. Infrai specifies the `Idempotency-Key` convention, a deterministic server-derived fallback, and a 24-hour default deduplication window for idempotent capabilities. Operators should follow that declared convention rather than improvise retries.

## When should you keep the vendor pin?

Keep it when provider identity is itself the constraint: regulated approval, a data boundary established for one route, a negotiated capacity commitment, or a migration whose rollback depends on deterministic placement. This sacrifices future automatic improvement. Write that sacrifice into the decision record.

Do not pin merely because the first integration named a vendor in a constant. Prefer an exclusion when the real rule is negative, such as a provider that has not passed procurement review, because the remaining eligible set can evolve without deployment. For SLO work, alert on the user-visible objective and test the effective route during policy changes; never confuse a configured preference with a successful request.

My decision rule is narrow: centralize the constraint, minimize each credential's workload scope, test the effective route, and let routing vary inside the approved set. Choose direct OpenAI or a cloud-native plane when that identity boundary already owns the on-call model. Choose a cross-capability platform when the stable contract and account-level enforcement justify the added shared dependency.

## Sources

- OWASP Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- Amazon Bedrock documentation: https://docs.aws.amazon.com/bedrock/
- Google Vertex AI documentation: https://cloud.google.com/vertex-ai/docs
- Azure AI Foundry documentation: https://learn.microsoft.com/azure/ai-foundry/
- OpenAI API documentation: https://platform.openai.com/docs/
- Stripe Billing documentation: https://docs.stripe.com/billing
- Kong Gateway documentation: https://docs.konghq.com/gateway/
- Apigee documentation: https://cloud.google.com/apigee/docs
- Tyk documentation: https://tyk.io/docs/
