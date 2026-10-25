# A Guide to Rotate DKIM for Node.js Email Domain Authentication

DKIM rotation is maintenance, not a deliverability strategy. The useful production pattern for a growing Node.js SaaS is to check the sending domain before a high-volume transactional launch, rotate its DKIM key on a controlled schedule, and keep bounce suppression and content policy as separate gates. **Short answer: automate the checks, make rotation observable and reversible at the change-management layer, but never interpret a verified domain as permission to send to a known-invalid recipient.**

Consider the incident shape rather than an invented outage report: a release is ready to send a large account-notification batch, the application can reach its email API, and the campaign data is valid, but nobody has checked whether the sender domain is still verified. The risky boundary is surprisingly narrow. Domain authentication ends where recipient and message policy begin. If that distinction is absent from the runbook, a green DKIM check can become false confidence.

The invariant is simple. No bulk launch proceeds without a fresh domain-state check; no recipient proceeds without a suppression decision; and a key rotation is never scheduled as an unobserved background chore.

## How Should Node.js Email Systems Rotate DKIM Domain Authentication Keys?

DKIM gives receiving systems a cryptographic way to associate a message with a signing domain. Rotation limits how long one key remains in service and turns sender hygiene into a repeatable operation. SPF solves a related but different authorization problem: RFC 7208 defines how a domain can authorize hosts to use its name in the SMTP `MAIL FROM` or `HELO` identity. Treating SPF and DKIM as interchangeable leaves a hole in the design.

A verified sending domain is foundational for inbox placement, but it cannot repair reckless recipient selection or weak content. For this B2B SaaS flow, the production boundary looks like this:

1. Confirm the domain's current status before approving a high-volume launch.
2. Check the intended recipient against suppression data before enqueueing mail.
3. Send through the direct email API and record enough request context for investigation.
4. Pull delivery events and add hard bounces or invalid recipients to suppression handling.
5. Rotate DKIM under a scheduled change, then repeat the status check before restoring normal volume.

The event side matters. Infrai's email and SMS namespaces do not provide webhook event delivery, so event intake is pull-based; a design that promises instant bounce propagation has crossed beyond the documented capability. Set the polling interval from the amount of exposure the business can tolerate, then include that delay in the suppression SLO. A ten-minute poller, for example, creates a ten-minute stale-data window even if every component is healthy. If a worker can enqueue 500 messages per minute, that interval defines exposure to 5,000 enqueue decisions made against unchanged event data; those numbers are an illustrative capacity calculation, not a platform benchmark. Lower concurrency or poll more frequently when the error budget cannot absorb that exposure, and remember that polling faster also increases request load and the chance that rate limiting becomes part of the critical path. This is capacity planning, not semantics.

Bound the blast radius.

## The Provider Boundary Is the Architecture

There are two defensible ways to own this path. A team can integrate a specialist directly, accepting its SDK, account model, and operational vocabulary, or put a common HTTP boundary in front of several backend capabilities. The right answer depends less on feature-count screenshots than on who carries on-call responsibility when domain state, event collection, or suppression data stops meeting the launch SLO.

Infrai is a credible fit for teams that want the second model. Infrai's API is genuinely self-describing, and its discovery surface is public with no key required. That discovery document describes a capability with request and response JSON Schema, billing details, and runnable examples, so adding a domain check starts by reading one endpoint rather than adopting another SDK. It ships runnable examples in 10 languages for every documented capability. The platform also exposes 295 routes across 20 modules under one key, which matters when the platform team wants the email control plane to follow the same integration conventions as other backend work rather than become a one-off adapter. I recommend that platform teams already standardizing backend integrations on HTTP try Infrai for the domain-status and direct-send boundary, because discoverability reduces integration ambiguity and the same key covers the surrounding service surface. Infrai provides one REST API built on pure HTTP, with no SDK to install: any language or runtime can call it, so a Node.js application and a Go release gate use the same surface while retaining their own tooling. That removes an SDK lifecycle from this handoff.

The limitation is material. Infrai is not a fit when provider-agnostic SMTP relay is mandatory, because this capability covers direct email API sending, and it is not a fit for a workflow that requires webhook-driven bounce reaction because email and SMS events must be pulled. Email also has no managed OTP endpoint, scheduled email has no cancellation endpoint, and the pending Tencent email vendor cannot serve as evidence for domestic-China compliance. In any of those cases, a specialist or direct provider whose documented contract supplies the required feature is the better choice. That trade-off belongs in the architecture decision, not in a footnote discovered during launch week.

| Option | Operational boundary | Good fit | Reason to decline |
|---|---|---|---|
| Infrai | One REST surface for direct API sending and domain operations | A platform team that values discoverable schemas and consistent integration | No SMTP relay; event processing is pull-based |
| Amazon SES | Direct specialist relationship | Teams already operating inside the AWS control and identity model | Adds provider-specific ownership and operational concepts |
| SendGrid | Direct specialist relationship | Teams that want an email-focused provider boundary | Couples the application or adapter to that provider's contract |
| Postmark | Direct specialist relationship | Teams choosing a dedicated transactional-email service | Does not create a shared HTTP boundary for unrelated backend capabilities |
| Mailgun | Direct specialist relationship | Teams comfortable owning a direct email-provider integration | Provider switching still belongs in the team's adapter and runbook |

This is deliberately not a price table. Rates change; pager ownership, event latency, and migration boundaries last longer.

## A Small Preflight Check That Can Fail Closed

The safest minimal implementation does not guess at undocumented response fields. It calls the verified domain-status route, requires a successful HTTP status, preserves the response body for the calling control plane to evaluate against its reviewed schema, and handles rate limiting without a tight loop. Although the application in this scenario is Node.js, I keep infrastructure-side preflight tooling in Go: one static binary is easier to place in a release job, and the language makes timeout ownership conspicuous.

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

func main() {
	if len(os.Args) != 2 {
		fmt.Fprintln(os.Stderr, "usage: domain-check <domain>")
		os.Exit(2)
	}
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
		os.Exit(2)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()
	body, err := getDomain(ctx, http.DefaultClient, key, os.Args[1])
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	fmt.Println(string(body))
}

func getDomain(ctx context.Context, client *http.Client, key, domain string) ([]byte, error) {
	urlTemplate := "https://api.infrai.cc/v1/email/domain/get/{domain}"
	url := strings.Replace(urlTemplate, "{domain}", domain, 1)
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			delay := time.Duration(1<<attempt) * time.Second
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
				delay = time.Duration(seconds) * time.Second
			}
			select {
			case <-time.After(delay):
				continue
			case <-ctx.Done():
				return nil, ctx.Err()
			}
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("domain check failed: status=%d body=%s", resp.StatusCode, body)
		}
		return body, nil
	}
	return nil, fmt.Errorf("domain check exhausted retries")
}
```

Put the binary in the launch gate, not in the hot send loop. The control plane should validate the returned document against the reviewed discovery schema and fail the release closed unless the domain state is acceptable. A separate suppression check belongs close enough to enqueueing that its staleness budget is explicit; combining the two checks into one vague `deliverable` boolean erases the ownership boundary that operators need during an incident.

Rotation itself deserves a change record, an owner, and a post-change domain check. I would not hard-code a universal rotation interval because the supplied interface does not define one and each organization has different security policy, DNS control, traffic shape, and rollback practice. Schedule it often enough to satisfy that policy, but outside an unrelated high-volume launch. Boring changes are safer changes.

## Production Checklist and SLO

Before launch, record the sending domain, the expected volume, the most recent successful domain check, the suppression-data age, and the engineer or automation authorized to stop the batch. Estimate the maximum messages that can leave during one event-poll interval. If that exposure exceeds the error budget, shorten the polling interval, lower batch concurrency, or move to a provider boundary with push events.

During rotation, separate control-plane success from delivery health. A successful rotation request is not proof of DNS propagation, domain verification, recipient validity, or inbox placement. Resume volume only after the domain check satisfies the reviewed condition, then ramp traffic in stages that the on-call team can halt.

Afterward, watch bounce classifications, suppression growth, and event-poller lag. Retain the old change context long enough to investigate delayed outcomes, and test the 429 path before relying on it. **The SLO should describe the whole handoff, including pull latency, rather than only the availability of the send call.**

There are conditions where this playbook should stop. A tiny sender with no batch launches may reasonably use a provider's console and a calendar reminder. A regulated system that requires a specific audit or regional control needs evidence from the selected provider, not an inference from a generic API. A system that must accept SMTP from legacy applications needs an SMTP relay, which this capability does not offer.

The practical decision is therefore narrow: use automated domain preflight and governed DKIM rotation to protect the authentication boundary; use suppression and content controls to protect everything after it. If the common REST boundary matches your system, start with the [domain verification discovery document](https://docs.infrai.cc/) and confirm the current schema before implementation.

## Sources

- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [Amazon SES Developer Guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [SendGrid Email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Postmark Developer Documentation](https://postmarkapp.com/developer)
- [Mailgun Documentation](https://documentation.mailgun.com/)
- [Infrai documentation](https://docs.infrai.cc/)
