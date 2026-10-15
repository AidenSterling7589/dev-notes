# Go Startup App SMS Alert Service Alternatives — Sender Registration Evidence

Treat password-reset SMS as a short-lived security workflow with an evidence trail, not as a cheap message API. The deciding constraint is compliance evidence: a startup serving US and EU shoppers must be able to show which approved sender identity was used, when the reset capability expired, what the carrier-facing system reported, and how operators handled ambiguous delivery without retaining the secret itself.

Short answer: shortlist at least three services, then run the same synthetic workload through each using registered US and EU sender paths. Keep the choice only if its immutable event export lets an auditor join an internal attempt ID to sender identity, submission time, normalized delivery state, region, and retention policy. Prefer webhooks for timely changes, but retain bounded polling for reconciliation; neither mechanism may extend token validity or trigger an automatic resend.

## How should a startup app compare an SMS alert service alternative?

The reset capability and delivery record need different lifetimes. The capability should be short-lived, single-use, and stored so logs or databases do not disclose the token. The operational record must satisfy the organization's documented evidence-retention policy. Deleting everything at expiry destroys auditability, while retaining bodies and raw tokens enlarges the security and privacy exposure.

Create one internal `attempt_id` before calling any provider. Record an opaque subject reference, policy version, destination region, sender-registration reference, template revision, token expiry, provider-neutral state, and timestamps. Never put the phone number, reset URL, token, or body in the receipt ledger.

Preserve uncertainty. `accepted` means the service accepted a request; it does not prove that a handset received it. Keep `delivered`, `failed`, `expired`, and `unknown` distinct, and retain the original external status beside the normalized state. A delayed receipt may update evidence, but it cannot revive an expired reset.

Unknown is honest.

Measure reset submission availability and receipt-reconciliation lag as separate SLOs. One is user-facing. The other measures the evidence pipeline.

## Capacity-plan the evidence path before comparing services

Start with peak attempts, not monthly volume. If a promotion or credential-stuffing event creates 40 attempts per second for 15 minutes, the system sees 36,000 attempts before retries; that is a planning scenario, not a claimed benchmark. Apply abuse limits first, then size queues, receipt ingestion, and reconciliation for the approved peak plus a documented margin.

SMS encoding affects segments. GSM-7 permits 160 characters in one segment, while UCS-2 permits 70; concatenation lowers the limits because headers consume space. Test exact localized templates with substituted values. A link, expiry statement, and warning about an unrequested reset are enough.

Run each candidate through the same matrix: US and EU destinations, approved sender types, GSM-7 and Unicode templates, rejections, delayed receipts, duplicate callbacks, reversed states, and a receipt outage. Twilio, AWS End User Messaging SMS, and Vonage can be candidates, but names are not conclusions. Verify current registration, geographic, receipt, export, and contract boundaries from official documentation during procurement.

| Decision area | Evidence to collect | Reject when |
|---|---|---|
| Sender registration | Approval tied to sender, purpose, and region | Production identity cannot be mapped to approval |
| Delivery receipts | Raw status, event time, attempt ID, documented meaning | `accepted` is called handset delivery |
| Reconciliation | Callback replay and bounded lookup | Missing callbacks remain silently final |
| Data handling | Region, retention, deletion, and access evidence | Required behavior cannot be documented |
| Operations | Rate controls, queue behavior, and incident export | An outage causes unbounded resend |
| Commercial model | Segment-aware invoice sample | Charges cannot be reconciled to controlled traffic |

The cheapest candidate is the one whose total operating burden fits the error budget and evidence obligations after engineering time, registration work, escalation, and on-call load are included. This is a buy-versus-build decision, not a price sort.

This approach has limitations. SMS is unsuitable when the application cannot establish a defensible phone-number lifecycle, when the destination is outside the approved sender footprint, or when the reset threat model requires phishing-resistant authentication; choose an already-enrolled recovery method instead. Polling is also a trade-off, not a default: it closes callback gaps but adds status-query load and extends the period in which an attempt remains unknown. A webhook-only design is simpler until a callback is dropped, while polling-only delays evidence and consumes capacity, so the bounded combination earns its complexity only when the audit requirement demands convergence. Self-hosting the ledger gives direct retention control but adds migrations, access reviews, backups, and on-call ownership. A managed export reduces that work yet can constrain event detail or retention, which procurement must verify rather than assume.

No transport fixes a weak recovery policy.

## Implement a narrow Go boundary

Keep provider vocabulary outside the reset service. Create a redacted attempt before submission, accept receipts idempotently, and reconcile nonterminal attempts. Adapters translate external statuses into a small internal state machine.

```go
package smsreset

import (
	"context"
	"errors"
	"time"
)

type State string
const ( Accepted State = "accepted"; Delivered State = "delivered"; Failed State = "failed"; Expired State = "expired"; Unknown State = "unknown" )

type Submission struct {
	AttemptID, Destination, Template, SenderPolicyID string
	TokenExpiresAt time.Time
}
type Receipt struct { AttemptID, ProviderRef, ExternalState string; State State; ObservedAt time.Time }
type Gateway interface { Send(context.Context, Submission) (Receipt, error); Lookup(context.Context, string) (Receipt, error) }
type Ledger interface { Begin(context.Context, Submission) error; ApplyReceipt(context.Context, Receipt) error }
type Service struct { Gateway Gateway; Ledger Ledger; Now func() time.Time }

func (s Service) Submit(ctx context.Context, req Submission) error {
	if req.AttemptID == "" || !req.TokenExpiresAt.After(s.Now()) { return errors.New("invalid reset attempt") }
	if err := s.Ledger.Begin(ctx, req); err != nil { return err }
	receipt, err := s.Gateway.Send(ctx, req)
	if err != nil { return err } // The caller owns retry policy; never resend implicitly.
	return s.Ledger.ApplyReceipt(ctx, receipt)
}
```

The ledger must redact `Destination`; production types should separate transport and audit records. A timeout after submission is ambiguous because acceptance may have occurred before the response was lost. Reconcile first. An automatic second SMS can create two live reset paths and double traffic during the failure.

## Verification, deployment, and rollback

Deploy adapters behind per-region configuration. Before customer traffic, test success, rejection, token expiry, duplicate and reversed receipts, malformed signatures, callback outage, and lookup throttling. Assert that logs contain no phone number, token, or reset URL. Verify that a receipt arriving after expiry updates only evidence.

The acceptance test is reconstruction: from an `attempt_id`, an authorized reviewer can recover policy version, approved sender reference, template revision, submission result, receipt transitions, expiry, and reconciliation actions without recovering the secret or destination.

Observe queue age, submission errors, ambiguous submissions, receipt lag, nonterminal age, backlog, signature failures, and invalid transitions. Alert thresholds must follow the two SLOs and a load-tested capacity model.

Rollback stops new submissions through the adapter while its outstanding receipts continue to ingest and reconcile. Keep callback verification active until attempts are terminal or the documented window closes. Fast rollback matters. Blind failover does not: changing services during an ambiguous window requires an operator decision, rate limits, and a new attempt linked to the original.

## Buy or build the surrounding controls?

Buying transport avoids operating carrier connectivity, but it does not outsource token policy, abuse controls, evidence design, privacy decisions, or incident ownership. A thin adapter and ledger form a stable boundary; building a transport network is a separate undertaking whose staffing and regulatory scope belongs on the roadmap.

| Capability | Buy | Build | Decision test |
|---|---|---|---|
| Carrier transport and onboarding | External service | Direct integrations and compliance operations | Will the team own carrier changes and delivery escalation? |
| Reset-token lifecycle | No | Application boundary | Can a transport event extend or reveal the token? It must not. |
| Receipt normalization | Raw events | Mapping that retains originals | Can every normalized state be traced? |
| Evidence controls | Export may help | Retention and access policy | Can deletion, holds, and regional rules be enforced? |
| Reconciliation | Lookup may exist | Bounded worker and runbook | Does callback loss converge to terminal or visible unknown? |

Choose only after the evidence drill, segmentation test, and outage exercise pass. Per-message cost belongs in the spreadsheet, but a password reset is complete only when the token expires correctly, abuse is bounded, and the organization can explain what happened without exposing the credential.

## References

- Twilio, “What is the SMS character limit?” https://www.twilio.com/docs/glossary/what-sms-character-limit
- NIST SP 800-63B, “Authentication and Lifecycle Management.” https://pages.nist.gov/800-63-4/sp800-63b.html
- OWASP, “Forgot Password Cheat Sheet.” https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- European Union, “General Data Protection Regulation, Article 5.” https://eur-lex.europa.eu/eli/reg/2016/679/oj
- CloudEvents specification. https://github.com/cloudevents/spec
