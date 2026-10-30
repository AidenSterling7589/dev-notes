# LLM Text Classification API Compare: Structured JSON for Healthtech Node.js (One Boundary)

A healthtech LLM text classification API compare has one constraint that changes the usual choice: a fast label is useless if a reviewer cannot trust why a moderation report entered the queue. I would put the quality boundary before the model call, require a schema-valid decision, and send uncertain reports to humans; latency is then measured against the review queue's SLO rather than a vendor's headline response time.

Short answer: do not choose the cheapest text endpoint first. Define the label taxonomy, confidence policy, retry budget, and evidence retention policy, then run the same redacted corpus through OpenAI, Anthropic Claude, Google Gemini, Mistral, and Groq behind one interface. Keep the winner only if its quality at the agreed latency survives malformed output, timeouts, duplicate delivery, and model replacement.

This is a healthtech moderation workflow, not a generic sentiment demo. A report can contain a symptom, a threat, an allegation, or protected health information. The classifier is a sorting aid before human review; it is not the decision-maker.

The queue is the product.

## The incident pattern I design against

I start with a bounded production scenario: a Node.js worker redacts fields that are not needed for triage, asks for one label and a short rationale, validates the response, and writes an immutable classification event. The worker has 2 seconds for the first attempt and 30 seconds of queue age before the report is marked `needs_human_review`. Those numbers are policy inputs, not universal defaults.

The failure I expect is a valid HTTP response containing an invalid decision. A model can return two labels, put a rationale where an enum belongs, or omit the field that downstream routing assumes exists. A retry can make this worse by creating two review events. The invariant is narrow: only schema-valid, policy-allowed output may change routing, and every other result becomes a human-review event with a reason. In a real queue, the worker must persist the input hash and decision state before acknowledging delivery, distinguish a timeout from a policy rejection, and make a replay produce the same event key rather than a second assignment; otherwise a dashboard can report healthy API latency while reviewers receive duplicates and the safety queue silently grows.

My first instinct is usually to tune the prompt. The correction is more useful: the boundary must reject bad output even when the prompt is excellent. The model is an uncertain dependency; the queue, schema validator, idempotency key, and audit record are our controls.

```go
package triage

import (
    "context"
    "encoding/json"
    "errors"
    "fmt"
)

type Decision struct {
    Label      string  `json:"label"`
    Rationale  string  `json:"rationale"`
    Confidence float64 `json:"confidence"`
}

var allowed = map[string]bool{
    "urgent_safety":      true,
    "privacy":             true,
    "routine":             true,
    "needs_human_review":  true,
}

func Accept(ctx context.Context, raw []byte) (Decision, error) {
    var d Decision
    if err := json.Unmarshal(raw, &d); err != nil {
        return Decision{}, fmt.Errorf("invalid JSON: %w", err)
    }
    if !allowed[d.Label] || d.Rationale == "" || d.Confidence < 0 || d.Confidence > 1 {
        return Decision{}, errors.New("policy validation failed")
    }
    select {
    case <-ctx.Done():
        return Decision{}, ctx.Err()
    default:
        return d, nil
    }
}
```

The example deliberately does not call a provider. A real adapter should attach a request id, a corpus version, a prompt version, and an idempotency key to the event. It should also record refusal, timeout, parse failure, and policy failure as different outcomes. That makes a later quality regression explainable.

This advice does not apply unchanged to a low-risk batch where a human will inspect every row before use. There, a permissive intermediate format and a larger deadline may be reasonable. It does apply when an automated label changes urgency, access, or exposure of a health-related report.

## How should we compare an LLM text classification API with structured JSON labels?

The useful comparison is a matrix of failure behavior, not a league table of token prices. For each candidate, I would replay a fixed, access-controlled set containing ordinary reports, ambiguous reports, adversarial instructions, long reports, empty text, duplicated reports, and examples whose correct answer is `needs_human_review`. The reference labels should be agreed by reviewers before the test starts.

| Gate | Measurement | Release rule |
| --- | --- | --- |
| Quality | Per-label precision, recall, and false-negative review | No label may hide an urgent safety report beyond the agreed threshold |
| Contract | Valid JSON, enum validity, rationale presence, and refusal handling | Invalid output never updates routing |
| Latency | p50, p95, timeout rate, and queue age under concurrency | p95 stays inside the review-queue SLO |
| Operations | Retry duplicates, provider errors, trace completeness, and replayability | Every outcome is attributable and repeatable |

I would compare OpenAI, Claude, Gemini, Mistral, and Groq only through the same adapter and the same corpus. Their relevant differences are API contracts, model context limits, structured-output behavior, rate-limit semantics, and model lifecycle; those details change over time, so the test artifact must record the exact model identifier and date. A gateway such as LiteLLM can normalize some request and response plumbing, but normalization does not make quality, refusal behavior, or retention policy identical.

The limitation is practical. A shared adapter cannot make a provider suitable for every privacy boundary, and a self-hosted gateway cannot remove the staffing cost of upgrades and on-call. The trade-off is clear: a direct adapter is a poor fit when the team cannot maintain several behavioral contracts, while a gateway is a poor fit when adding another availability dependency violates the service's SLO. Those are selection constraints, not reasons to rank one product.

A buy-versus-build table makes the operating cost visible without pretending that a per-token number is the whole decision:

| Boundary choice | What the team owns | When it is defensible |
| --- | --- | --- |
| Direct provider adapters | Authentication, retries, schemas, quotas, and five sets of behavioral tests | A small platform team can maintain the adapters and needs direct control |
| Self-hosted gateway | Routing, observability, policy configuration, upgrades, and gateway availability | Several workloads share a stable internal contract and the on-call team accepts another critical service |
| Human-first fallback | Queue capacity, reviewer guidance, audit retention, and escalation | The classifier cannot meet the false-negative threshold or the consequence of error is high |

The cheapest request path can produce the most expensive queue if its false negatives create manual investigation or delayed safety review. I would set a maximum retry count of one for a synchronous classification attempt, then make the queue consumer idempotent; retrying indefinitely is not resilience, it is an unbounded latency policy.

This is where the price comparison stops being useful.

## How do we preserve quality when the runtime changes?

Treat the classifier like a production dependency with an SLO. A practical starting contract might be 99.5% of accepted events schema-valid, p95 classification latency below the queue budget, and zero silent drops. The exact values belong to the service owner and clinical-safety review; the important part is that “the API returned 200” is not the success metric.

Use a shadow path for model changes. New output is validated and scored against a held-out set, but it does not route live reports. Then canary by a bounded fraction of traffic while watching false negatives, abstentions, parse failures, p95 latency, and queue age. Keep the previous adapter available until the canary window closes.

Streaming can improve time to first bytes, but a partial JSON object is not a classification event. Server-sent events are a transport mechanism, not a schema guarantee. For this workflow, I would use streaming only if the adapter can buffer and validate the complete final object before publishing the routing event. Otherwise, the apparent latency gain buys a more complicated failure mode.

The data path needs equal discipline. Redact identifiers before inference, restrict logs to the fields needed for diagnosis, encrypt stored audit events, and set a retention period with the healthtech privacy owner. Do not place the raw report in a retry log by accident. A good trace can contain a hash, policy version, and outcome code without becoming a second clinical record.

## The decision rule

Choose a runtime only after it passes the same contract and quality gates on the same redacted corpus. If two candidates meet the quality threshold, select the one with the smaller operational surface for the team that owns the SLO; if neither meets it, route more work to humans and fix the taxonomy or evidence set before shopping for a lower price.

That rule is intentionally conservative. In moderation triage, latency is a service property and quality is a safety property. A model swap, gateway upgrade, or retry change is therefore a production change even when the application code stays still.

## Sources

- https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- https://github.com/BerriAI/litellm
- https://json-schema.org/learn/getting-started-step-by-step
- https://opentelemetry.io/docs/concepts/observability-primer/
- https://platform.openai.com/docs/guides/structured-outputs
- https://docs.anthropic.com/en/docs/build-with-claude/tool-use
- https://ai.google.dev/gemini-api/docs/structured-output
