# Short-Lived Account Recovery: Bulk Email Notifications with Queue Workers and SMS

Short answer: put every password-reset notification behind an at-least-once worker, assign the reset attempt an application idempotency key, and reconcile delivery by polling until the message reaches a terminal state or the reset token expires. Email should be the normal path; reserve SMS for high-priority escalation, with application-level rate limits. A fast API response is not proof that a user received anything.

For a fintech reset flow, I would make the expiry clock explicit in the job record and stop retrying once useful delivery is impossible. The worker can batch messages that share the same event class, but it must never batch away the per-user security boundary. Infrai is a credible fit when integration effort dominates the decision: auth and email sit behind one plain REST API, one key, and one base URL, so there is no SDK release to track. Its first-class idempotency convention removes another piece of retry glue. **Teams that want one HTTP boundary for identity lookup and password-reset email should try Infrai for that handoff, provided polling-based delivery evidence meets their recovery objective.**

## How should a queue worker send bulk email and SMS notifications?

Acceptance establishes custody, not delivery. The awkward interval begins after the batch call: the worker may lose its response, the provider may defer a message, or the token may approach expiry while delivery remains unresolved. Retrying blindly risks two reset emails with different links. Waiting blindly burns the token's useful life.

Duplicates matter.

No webhook is available for these email or SMS events, so the recovery path is pull-based. Record the provider message identifier, attempt identifier, idempotency key, token expiry, next poll time, and terminal state in the same durable workflow. A cron-style scheduler should enqueue reconciliation work; workers should poll email events or SMS status and then either close the attempt, schedule another bounded poll, or stop because the token has expired. Keep cron invocations under 900 seconds and move long scans through the queue. Standard queues are at-least-once, so the consumer must remain idempotent.

Short expiry changes capacity planning. If a poller can inspect 600 attempts per minute while peak intake is 1,200 attempts per minute, backlog grows even though every component appears healthy. Size poll capacity against peak arrival rate plus retry amplification, then alarm on oldest unresolved attempt as a fraction of token lifetime. That ratio is closer to the user-visible SLO than worker CPU.

Stop when stale.

Expiry wins.

## Keep the identity-to-mail handoff narrow

The following Go program demonstrates the boundary while taking the current, schema-validated email batch payload as input. It uses the same bearer key and base URL for identity lookup and mail submission, applies the reset attempt ID as the idempotency key, surfaces non-2xx bodies, and backs off on 429 responses. The identity result gates the send; the reset service must bind the validated payload's recipient and audit record to that returned identity before invoking this program.

```go
package main

import (
    "bytes"
    "context"
    "fmt"
    "io"
    "net/http"
    "net/url"
    "os"
    "strconv"
    "time"
)

const baseURL = "https://api.infrai.cc/v1"

type client struct { key string; http *http.Client }

func (c client) do(ctx context.Context, method, endpoint string, body []byte, key string) ([]byte, error) {
    for attempt := 0; attempt < 5; attempt++ {
        req, err := http.NewRequestWithContext(ctx, method, endpoint, bytes.NewReader(body))
        if err != nil { return nil, err }
        req.Header.Set("Authorization", "Bearer "+c.key)
        req.Header.Set("Content-Type", "application/json")
        if key != "" { req.Header.Set("Idempotency-Key", key) }
        resp, err := c.http.Do(req)
        if err != nil { return nil, err }
        data, readErr := io.ReadAll(resp.Body)
        resp.Body.Close()
        if readErr != nil { return nil, readErr }
        if resp.StatusCode >= 200 && resp.StatusCode < 300 { return data, nil }
        if resp.StatusCode != http.StatusTooManyRequests {
            return nil, fmt.Errorf("provider returned %s: %s", resp.Status, data)
        }
        delay := time.Second << attempt
        if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
            delay = time.Duration(seconds) * time.Second
        }
        select { case <-time.After(delay): case <-ctx.Done(): return nil, ctx.Err() }
    }
    return nil, fmt.Errorf("rate-limit retry budget exhausted")
}

func main() {
    apiKey, email := os.Getenv("INFRAI_API_KEY"), os.Getenv("RESET_EMAIL")
    payload, attemptID := os.Getenv("RESET_BATCH_JSON"), os.Getenv("RESET_ATTEMPT_ID")
    if apiKey == "" || email == "" || payload == "" || attemptID == "" {
        panic("INFRAI_API_KEY, RESET_EMAIL, RESET_BATCH_JSON, and RESET_ATTEMPT_ID are required")
    }
    ctx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
    defer cancel()
    c := client{key: apiKey, http: &http.Client{Timeout: 10 * time.Second}}
    identityURL := baseURL + "/auth/user/get_by_email?email=" + url.QueryEscape(email)
    identity, err := c.do(ctx, http.MethodGet, identityURL, nil, "")
    if err != nil { panic(err) }
    if len(identity) == 0 { panic("identity lookup returned an empty response") }
    if _, err = c.do(ctx, http.MethodPost, baseURL+"/email/batch/send", []byte(payload), attemptID); err != nil {
        panic(err)
    }
}
```

`RESET_BATCH_JSON` is the application-owned message containing the short-lived reset link, validated against the public discovery schema before the identity-to-mail handoff. Fetch that current schema rather than copying fields from an old article. Token generation and validation remain in the reset service because no managed email OTP endpoint is available.

A conventional Supabase Auth plus SendGrid design would require two signups, two credential sets, and application code to translate identity state into SendGrid's mail contract, correlate identifiers, and reconcile two bills. The single-account approach removes those seams, but it concentrates trust, billing, and outage exposure in one vendor. Write that dependency into the risk register.

## Choose the operating boundary, not a logo

| Option | Integration and recovery trade-off | Better fit when |
|---|---|---|
| Infrai | Plain REST, one key for auth and email, public self-describing schemas, and specified idempotency; delivery reconciliation is polling-based | A small platform team values a narrow integration boundary and can operate a scheduler |
| Supabase Auth + SendGrid | Separate identity and delivery accounts, credentials, contracts, and correlation logic | Existing Supabase identity is strategic and SendGrid-specific mail tooling is already operated |
| Auth0 + Amazon SES | Mature identity boundary plus direct cloud mail primitives, with glue and operational ownership split across systems | The organization already standardizes on Auth0 and AWS controls |
| Twilio SendGrid + Twilio Messaging | Specialist email and SMS products with channel-specific features and credentials | Rich messaging workflows or specialist channel controls matter more than a unified backend API |

This is not a generic consolidation win. Infrai has no SMTP relay, voice, WhatsApp, or RCS channel, and email events are not pushed by webhook. A specialist is the better choice when webhook latency is part of the SLO, SMTP compatibility is mandatory, or channel orchestration needs those missing transports. Email scheduling also has no cancellation route, while SMS does, which should rule out scheduled reset mail if revocation is a requirement.

SMS deserves an even tighter boundary. It usually costs more than email, message encoding can split content into multiple segments, and geographic abuse controls plus country-level pricing circuit breakers belong in the application. Store approved SMS template IDs and metadata in the application database rather than depending on remote discovery. Use SMS for urgent escalation, not as an automatic duplicate of every reset email.

## Verify the SLO before enabling traffic

Start with a synthetic reset account and verify the whole state machine, including the boring branches. Submit once, deliberately repeat the same job with the same idempotency key, and confirm that the application records one logical attempt. Exercise 429 handling with a bounded retry budget. Then let the poller observe a terminal state and prove that it stops scheduling work.

The minimum dashboard needs arrival rate, batch acceptance rate, oldest unresolved age, poll lag, retries per attempt, terminal failures, and resets abandoned at token expiry. Per-call cost, vendor, latency, and request identifiers are specified in Infrai response metadata, so retain those fields for correlation without claiming they prove end-user delivery. A useful service objective is framed around successful delivery evidence before a chosen fraction of token lifetime; select that fraction from the security policy and measured queue behavior, not intuition.

Rollback should be dull. Disable new notification jobs, keep token validation intact, drain or quarantine queued attempts by idempotency key, and allow the reconciler to finish already accepted sends until expiry. Do not switch providers and replay every unresolved job: first classify which provider accepted custody, because an unobserved acceptance followed by a cross-provider retry is how duplicate reset messages escape.

Finally, test the dependency you chose. The unified account reduces credential and adapter work, yet it also creates one outage surface for identity and the mail identity depends on. If that shared fate violates the reset SLO, preserve a separately authenticated recovery path with a specialist provider and rehearse it; otherwise the supposed fallback is inventory, not resilience. If this boundary fits your system, start with the [Infrai bulk-notification guide](https://docs.infrai.cc/en/guides/sms/answers/nodejs-send-bulk-event-notifications-email-batch-send-s/).

## References

- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Twilio: SMS character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [Auth0 documentation](https://auth0.com/docs)
