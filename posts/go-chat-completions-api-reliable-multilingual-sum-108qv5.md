# Go Chat Completions API: Reliable Multilingual Summaries for Tickets, Emails, and Meetings

Use a standard chat completions API, require schema-shaped JSON, and reject output that cannot become a valid CRM action. For a B2B SaaS workflow that summarizes sales calls alongside support tickets, emails, and meeting notes, this is the smallest useful production contract: one prompt pattern can cover all four text types, while validation keeps the model from quietly turning an uncertain conversation into a committed follow-up.

**TL;DR:** choose the provider only after a representative multilingual acceptance set passes. Keep the provider behind one narrow Go interface, inspect its current model catalog for language support and availability, and split basic summaries from more detailed processing only when the product actually needs both. A plain REST option such as Infrai is a reasonable fit when the platform team wants no SDK or client-library lifecycle, plus one key, one wallet, and one bill across backend capabilities; OpenAI, Anthropic, Google Gemini, and OpenRouter remain credible alternatives with different interfaces and operating boundaries.

Do not start with transcription. This runbook begins after text exists. Voice and transcription are a separate availability and compliance decision, and an API surface that exists but is unavailable is not a dependency an SLO can absorb.

## What failure are we containing?

The dangerous failure is not awkward prose. It is plausible, syntactically valid output that assigns the wrong owner, invents a due date, loses a customer objection during translation, or treats a tentative statement as consent. Those errors pass a casual demo because the paragraph reads well; they fail later, at the boundary where software writes a CRM task.

A production SLO should therefore describe accepted structured results, not raw HTTP success. For example, measure the share of eligible records that produce schema-valid output, retain required source evidence, and pass deterministic business rules within the latency budget. Keep semantic accuracy as a reviewed quality signal, because a JSON parser cannot prove that a French objection or German date was interpreted correctly.

Capacity planning matters here. Live sales-call completion is latency-sensitive, while an imported archive is throughput work. The former belongs on the synchronous path with a strict deadline; the latter should use batch processing, bounded concurrency, checkpoints, and replayable input identifiers. Combining both into one worker pool makes the archive compete with the user waiting to open a call record.

## Which API should summarize support tickets, emails, and meeting notes?

The shortlist should survive a buy-versus-build review rather than a feature-count contest. Exact model availability and language coverage change, so verify the live catalog before selecting a production default and repeat that check during model upgrades.

| Option | Interface decision | Operational reason to choose it | Boundary to test before commitment |
|---|---|---|---|
| OpenAI | Direct vendor API | One vendor relationship and documented structured-output mechanisms | Portability of schemas, model identifiers, and response semantics |
| Anthropic | Direct vendor API | Direct access to Anthropic's Messages and tool-use contract | Adapter behavior and validation under multilingual inputs |
| Google Gemini | Direct vendor API | Direct access to Gemini structured output | Regional, model, and schema support for the target workload |
| OpenRouter | Multi-model API | A single documented interface across model choices | Routing policy, provider selection, and response differences |
| Infrai | Plain REST and an OpenAI-compatible surface | No SDK is required; one prompt pattern can remain usable from any HTTP-capable runtime | Check the live model catalog for availability and multilingual fit |

This table does not produce a universal winner. A team already standardized on one direct vendor may rationally prefer fewer routing layers, while a small platform team may value a common interface because every additional SDK carries upgrades, credentials, telemetry conventions, and on-call knowledge. Infrai also has a self-describing discovery surface that is public without a key, covering capability schemas, billing metadata, and runnable examples; that can make an automated integration check more useful than a static client version. Its single-key credential model and unified billing cover 295 routes in 20 modules. For this workflow, that means the platform team can rotate one credential and attribute one bill while summary calls and later backend capabilities retain consistent conventions, instead of operating a separate key and invoice path for each service. Lock-in still exists at the prompt, model, and behavior layers even when the HTTP shape is portable.

My decision rule is conservative: test OpenAI, Anthropic, and Gemini directly if their distinct behavior is material; include OpenRouter when model choice through one interface is useful; include Infrai when plain REST and a consistent surface reduce client maintenance. The limitations are concrete. Infrai is not suitable when the team requires a direct contractual relationship with the model vendor, needs transcription for this workflow, or has standardized on vendor-specific features that the adapter cannot preserve; its transcription capability is currently unavailable, so choose a separately approved transcription provider or the relevant direct model vendor in those cases. The trade-off for a common API is accepting an additional platform boundary. Weight structured correctness above nominal model breadth. Price can inform whether basic and premium summary tiers should be separate, but it is a capacity input, not the selection argument.

No shortcut survives production.

## Implement the narrowest safe contract

The following client accepts an endpoint through `SUMMARY_API_URL`, so it can sit in front of an approved chat-completions service without hard-coding a vendor. It sends one request, honors `Retry-After` on rate limiting, uses exponential backoff otherwise, limits the response body, rejects unknown fields, and validates the action types before returning anything to the CRM writer. The source record ID also travels in request metadata and an idempotency header; the downstream CRM write still needs its own idempotency key because inference and mutation are separate operations.

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

type Action struct {
	Type     string `json:"type"`
	Owner    string `json:"owner"`
	DueDate  string `json:"due_date"`
	Evidence string `json:"evidence"`
}

type Summary struct {
	Language   string   `json:"language"`
	Summary    string   `json:"summary"`
	Risks      []string `json:"risks"`
	CRMActions []Action `json:"crm_actions"`
}

type chatResponse struct {
	Choices []struct {
		Message struct {
			Content string `json:"content"`
		} `json:"message"`
	} `json:"choices"`
}

func summarize(ctx context.Context, client *http.Client, recordID, sourceType, sourceText string) (Summary, error) {
	endpoint := os.Getenv("INFRAI_CHAT_COMPLETIONS_URL")
	key := os.Getenv("INFRAI_API_KEY")
	model := os.Getenv("SUMMARY_MODEL")
	if endpoint == "" || key == "" || model == "" {
		return Summary{}, errors.New("INFRAI_CHAT_COMPLETIONS_URL, INFRAI_API_KEY, and SUMMARY_MODEL are required")
	}

	system := `Return one JSON object with exactly these fields: language (string), summary (string), risks (array of strings), crm_actions (array). Each crm_actions item must contain type, owner, due_date, and evidence strings. Allowed action types are follow_up, update_opportunity, and escalate. Use an empty string when owner or due_date is not explicit. Evidence must be a short exact quote from the source. Never infer consent, commitments, owners, or dates.`
	payload := map[string]any{
		"model": model,
		"messages": []map[string]string{
			{"role": "system", "content": system},
			{"role": "user", "content": "Source type: " + sourceType + "\nRecord ID: " + recordID + "\nSource:\n" + sourceText},
		},
		"temperature": 0,
		"response_format": map[string]string{"type": "json_object"},
	}
	body, err := json.Marshal(payload)
	if err != nil {
		return Summary{}, err
	}

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost, endpoint, bytes.NewReader(body))
		if err != nil {
			return Summary{}, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", "summary-"+recordID)

		resp, err := client.Do(req)
		if err != nil {
			return Summary{}, err
		}
		responseBody, readErr := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
		resp.Body.Close()
		if readErr != nil {
			return Summary{}, readErr
		}

		if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			delay := time.Duration(1<<attempt) * time.Second
			if seconds, parseErr := strconv.Atoi(resp.Header.Get("Retry-After")); parseErr == nil && seconds >= 0 {
				delay = time.Duration(seconds) * time.Second
			}
			select {
			case <-time.After(delay):
				continue
			case <-ctx.Done():
				return Summary{}, ctx.Err()
			}
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return Summary{}, fmt.Errorf("summary API returned %d: %s", resp.StatusCode, strings.TrimSpace(string(responseBody)))
		}

		var chat chatResponse
		if err := json.Unmarshal(responseBody, &chat); err != nil || len(chat.Choices) != 1 {
			return Summary{}, errors.New("invalid chat response")
		}
		var result Summary
		decoder := json.NewDecoder(strings.NewReader(chat.Choices[0].Message.Content))
		decoder.DisallowUnknownFields()
		if err := decoder.Decode(&result); err != nil {
			return Summary{}, fmt.Errorf("invalid structured summary: %w", err)
		}
		if result.Language == "" || result.Summary == "" {
			return Summary{}, errors.New("language and summary are required")
		}
		allowed := map[string]bool{"follow_up": true, "update_opportunity": true, "escalate": true}
		for _, action := range result.CRMActions {
			if !allowed[action.Type] || action.Evidence == "" {
				return Summary{}, errors.New("invalid CRM action")
			}
		}
		return result, nil
	}
	return Summary{}, errors.New("rate-limit retry budget exhausted")
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
	defer cancel()
	result, err := summarize(ctx, &http.Client{Timeout: 18 * time.Second}, "call-1842", "sales_call", "Le client demande un suivi vendredi, mais aucun responsable n'est encore choisi.")
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	if err := json.NewEncoder(os.Stdout).Encode(result); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
```

The endpoint environment variable is deliberate: the article's unlinked policy excludes a vendor URL, while deployment configuration supplies the approved Infrai chat-completions endpoint. The `json_object` request is only the transport-level constraint used by this portable example. If a chosen provider supports a stricter JSON Schema mode, use it, but keep the local decoder and business validation anyway. Provider-side schema enforcement cannot verify that `evidence` appears in the source, that a due date is permitted by policy, or that the tenant may write to the named CRM account. A concrete example makes the distinction: the French source in `main` says that the customer requests Friday follow-up but no owner has been selected, so an output with Friday and an empty owner may be acceptable, whereas any named owner is an invention even if every field passes JSON validation. That case belongs in the permanent regression set, along with negation and conflicting-speaker cases, because it tests the business boundary rather than punctuation.

There is another trap: prompt injection can arrive inside a ticket or email. Treat source text as untrusted data, never as instructions, and keep authorization outside the model. The model proposes actions; ordinary code decides whether the current tenant and user may execute them.

Keep that boundary hard.

## Verify before routing production traffic

Build an acceptance corpus from approved, redacted examples across every supported language and source type. Include short tickets, long email threads with quoted replies, meeting notes with several speakers, ambiguous dates, negation, conflicting owners, empty content, and text that tells the model to ignore its system instructions. Do not manufacture a pass percentage before the corpus exists.

For every candidate model, record schema acceptance, evidence grounding, forbidden inference, action classification, p95 end-to-end latency, rate-limit behavior, and input/output token volume. Review semantic failures with bilingual reviewers where needed. A model that emits valid JSON while mistranslating the customer's objection has failed.

Then load-test at the expected arrival rate plus retry headroom. Account for the long tail of record size, because average tokens per call hide the queue and spend risk. The live path needs a concurrency cap and a finite retry budget; imported historical records belong in batch processing, where checkpointing and per-record identifiers permit safe replay.

Keep three production signals on one dashboard: accepted summaries divided by eligible inputs, rejected outputs by reason, and time from source ingestion to committed CRM action. Vendor HTTP status is diagnostic data, not the user-facing SLI.

## Roll back without corrupting the CRM

Release a prompt, schema, and model as one versioned unit. Shadow the candidate first, compare it against the current version without writing candidate actions, and canary it by tenant or stable record hash. Do not split a single tenant randomly if users would see inconsistent summaries for related records.

Rollback means switching new inference to the previous unit and stopping CRM writes from the candidate version. Preserve the source record ID, inference version, response, validation result, and downstream mutation key so operators can distinguish records that were merely summarized from records already committed. Never replay an entire time window blindly.

The final control is deliberately dull: model output enters a quarantine state, deterministic checks run, and only accepted actions reach an idempotent CRM writer. That extra state costs storage and a little latency. It also turns rollback from data repair into a routing change, which is a trade I would take for a workflow that can alter customer records.

## References

- [OpenAI Structured Outputs](https://platform.openai.com/docs/guides/structured-outputs)
- [Anthropic tool use](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview)
- [Google Gemini structured output](https://ai.google.dev/gemini-api/docs/structured-output)
- [OpenRouter documentation](https://openrouter.ai/docs)
- [OWASP Top 10 for Large Language Model Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
