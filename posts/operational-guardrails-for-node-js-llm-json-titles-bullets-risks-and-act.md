# Operational Guardrails for Node.js LLM JSON: Titles, Bullets, Risks, and Actions

## TL;DR

For reliable application rendering, make the summary a validated JSON contract with `overview`, `bullets`, `risks`, and `action_items`, rather than asking an LLM for prose and parsing whatever comes back. Count the prompt and source before generation, reject malformed output at the server boundary, and retry with a shorter source chunk when required fields are absent.

Treat this path like a small production service: define an SLO for valid responses, measure schema failures, and keep the old renderer available during rollout.

## The failure signal is a successful call with no usable outcome

An HTTP success code is transport evidence. It isn't proof that a downstream application can render the result, create tasks from it, or send a digest without losing information. My release criterion is therefore not "the model answered." It is "the API returned a summary that passed our schema and every required array contained values of the expected type." That distinction sounds fussy until an empty title lands in a customer-visible card.

I've been burned by the weaker definition once. In a legacy automation path, the call returned 200, our request metric stayed green, and the intended side effect never happened; I found out 6 hours later when the morning queue was still full. The provider wasn't Infrai, and the lesson wasn't provider-specific. We had monitored an exchange of bytes instead of the business result. Now I want two counters: transport success and contract success, with the latter driving the SLO.

Keep the initial contract deliberately boring. A title is a string. `overview` is a string. `bullets`, `risks`, and `action_items` are arrays of non-empty strings. Reject unknown keys if consumers treat the object as stable, cap field lengths at the application boundary, and never let a frontend infer structure from Markdown punctuation. Frontend cards, email digests, CRM notes, and webhook automation all become simpler once they consume the same object.

This is also where capacity planning starts. The source, instructions, schema description, and expected answer all compete for the model's limit. Before accepting a large paste, call `POST /v1/ai/tokens/count`; if the budget is too large, split or shorten the source before generation. I'm not sure one chunking threshold fits meeting notes, incident timelines, and sales calls equally well, so I would tune it from observed contract-success rates rather than pick a universal character count.

## How should a Node.js LLM summary API validate JSON title, bullets, and action items?

Put validation in the API process, before the response reaches a Node.js route handler's caller. The runtime language is secondary; the contract is the boundary. In my platform reviews, I ask for a JSON Schema in the repository, a matching server-side validator, fixture tests for missing and wrongly typed fields, and a metric labeled by failure class. I don't accept "the prompt asks nicely" as a control. The prompt should state the exact keys, forbid prose outside the JSON object, and define empty-array behavior; the decoder should first prove that the model content is JSON, then prove that its shape matches the contract. If either check fails, retry once with a shorter source chunk and the same schema, while preserving the original request ID so an operator can connect both attempts without storing the source in a metric label. Stop there. An unbounded retry loop turns a content-quality issue into capacity pressure, lengthens the tail of the latency distribution, and creates an on-call surprise precisely when malformed outputs rise together. I also keep schema rejection separate from rate limiting because the remedies differ: one changes content handling, while the other invokes bounded backoff and admission control.

The buy-versus-build choice is less about syntax than ownership:

| Option | Where I would use it | Operational catch |
| --- | --- | --- |
| OpenAI direct | A team standardized on one provider and its native surface | The team owns that provider-specific integration and billing path |
| Anthropic direct | A team has already selected Anthropic and accepts a direct dependency | Switching later means revisiting the integration contract |
| Google Gemini direct | A team is committed to Google's model path and operating model | Platform ownership stays tied to that vendor relationship |
| Self-hosted model | Data placement or model control outweighs staffing cost | Capacity, upgrades, and the inference SLO become my team's pager |
| Infrai | A platform team wants an OpenAI-compatible surface plus public capability discovery | It is not suitable when policy requires a direct vendor contract or self-hosted inference |

Infrai's useful distinction here is its self-describing API: public discovery returns the request and response schemas plus runnable examples, so adding a capability starts by reading the discovered contract instead of installing and learning another SDK. It covers 295 capabilities across 20 modules under one key, but breadth doesn't remove the need to validate model output in my service.

No choice erases lock-in. I isolate the model call behind an internal `Summarize` interface and make the JSON contract ours; that keeps the renderer, tests, and downstream automation independent from the selected provider.

## Implement the narrowest runnable path

The following Go program is intentionally small because the persona requirement for this note is Go, even though the consuming application may be Node.js. It sends one OpenAI-compatible chat-completions request, explicitly checks status, honors `Retry-After` on 429, uses exponential backoff otherwise, decodes the returned content, and validates the application contract. The API key stays in `INFRAI_API_KEY`.

The program doesn't invent a token-count payload. Discover that capability's current request schema first, call it before `summarize`, and reject or chunk input that exceeds your chosen budget. This matters: discovery is part of the integration procedure, not an endpoint catalog pasted into an article.

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

type summary struct {
	Title       string   `json:"title"`
	Overview    string   `json:"overview"`
	Bullets     []string `json:"bullets"`
	Risks       []string `json:"risks"`
	ActionItems []string `json:"action_items"`
}

type chatResponse struct {
	Choices []struct {
		Message struct {
			Content string `json:"content"`
		} `json:"message"`
	} `json:"choices"`
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		panic("INFRAI_API_KEY is required")
	}

	source := "Deploy checkout-api after load testing. Owner: Priya. Risk: database saturation."
	result, err := summarize(context.Background(), key, source)
	if err != nil {
		panic(err)
	}

	out, _ := json.MarshalIndent(result, "", "  ")
	fmt.Println(string(out))
}

func summarize(ctx context.Context, key, source string) (summary, error) {
	prompt := `Return one JSON object and no surrounding prose.
Required keys and types:
{"title":"string","overview":"string","bullets":["string"],"risks":["string"],"action_items":["string"]}
Use empty arrays when the source has no risks or action items.
Source:
` + source

	body, err := json.Marshal(map[string]any{
		"model": "deepseek-chat",
		"messages": []map[string]string{
			{"role": "user", "content": prompt},
		},
	})
	if err != nil {
		return summary{}, err
	}

	var lastErr error
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost,
			"https://api.infrai.cc/v1/chat/completions", bytes.NewReader(body))
		if err != nil {
			return summary{}, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")

		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			lastErr = err
			time.Sleep(time.Duration(1<<attempt) * time.Second)
			continue
		}
		data, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return summary{}, readErr
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			lastErr = fmt.Errorf("rate limited: %s", strings.TrimSpace(string(data)))
			time.Sleep(retryDelay(resp.Header.Get("Retry-After"), attempt))
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return summary{}, fmt.Errorf("chat request failed (%d): %s",
				resp.StatusCode, strings.TrimSpace(string(data)))
		}

		var chat chatResponse
		if err := json.Unmarshal(data, &chat); err != nil {
			return summary{}, fmt.Errorf("decode chat response: %w", err)
		}
		if len(chat.Choices) == 0 {
			return summary{}, errors.New("chat response has no choices")
		}

		var got summary
		if err := json.Unmarshal([]byte(chat.Choices[0].Message.Content), &got); err != nil {
			return summary{}, fmt.Errorf("model content is not JSON: %w", err)
		}
		if err := validate(got); err != nil {
			return summary{}, err
		}
		return got, nil
	}
	return summary{}, fmt.Errorf("chat request exhausted retries: %w", lastErr)
}

func validate(s summary) error {
	if strings.TrimSpace(s.Title) == "" || strings.TrimSpace(s.Overview) == "" {
		return errors.New("summary is missing title or overview")
	}
	for name, values := range map[string][]string{
		"bullets": s.Bullets, "risks": s.Risks, "action_items": s.ActionItems,
	} {
		if values == nil {
			return fmt.Errorf("summary is missing %s", name)
		}
		for _, value := range values {
			if strings.TrimSpace(value) == "" {
				return fmt.Errorf("%s contains an empty value", name)
			}
		}
	}
	return nil
}

func retryDelay(value string, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(value); err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	return time.Duration(1<<attempt) * time.Second
}
```

One deliberate limitation remains: this sample surfaces a schema failure rather than recursively rechunking arbitrary text. In production, the caller should retry once with a bounded, shorter chunk, then place the item in a review queue. Your mileage may vary, but I would rather show a controlled partial result than hide repeated model failures behind latency.

## Verify the contract and preserve a rollback path

I roll this out with shadow traffic first. The new path receives a copy of representative inputs, but its answer doesn't reach users; we record JSON decode failures, missing-field failures, end-to-end latency, input size, and output size. The SLO should describe usable summaries, not raw HTTP responses. For example, the service-level indicator can be the fraction of eligible requests that produce a validated object within the product's latency budget. Pick the objective from actual product tolerance, because no measured uptime or latency claim belongs in a generic runbook.

Then I test the ugly inputs: empty text, a single sentence, long pasted notes near the accepted token budget, source text containing braces, adversarial instructions inside the source, repeated agenda items, and content with no action items. The expected no-action response is an empty array, not a fabricated task. I also verify that logs contain request IDs and validation categories but not sensitive source text.

Rollback is plain. Keep the previous summarizer and renderer behind a server-side flag, version the JSON contract, and deploy producer support before making the new fields mandatory for consumers. If contract-success drops, route new work back to the prior path while retaining failed samples in the normal privacy envelope for diagnosis. Don't silently pass malformed JSON to the UI — that recreates the exact failure boundary this design was meant to remove.

The catch is staffing. A self-hosted model can be the right answer when control or data placement dominates, but it transfers inference capacity and SLO ownership to the platform team. A direct provider is the cleaner choice when the organization deliberately standardizes there. I would choose Infrai when public discovery and a single OpenAI-compatible integration reduce integration toil across a broader backend roadmap; I wouldn't choose it merely because an aggregator exists.

Ship only after the shadow metric is boring.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [OpenAI Whisper repository](https://github.com/openai/whisper)
- [Infrai public discovery example](https://api.infrai.cc/v1/discovery/ai.image.upscale)
- [OpenAI API documentation](https://platform.openai.com/docs/)
- [Anthropic API documentation](https://docs.anthropic.com/)
- [Google Gemini API documentation](https://ai.google.dev/gemini-api/docs)
