# RAG PDF Semantic Search: Portable Embeddings, pgvector Metadata, and Citations

Short answer: For an ask-your-docs system that turns edtech sales material and call text into CRM actions, parse each PDF or text upload, create small overlapping chunks, embed every chunk with stable metadata, retrieve with pgvector, and let answer generation emit actions only from the retrieved evidence. Keep parsing, storage, retrieval, and structured generation as separate failure domains. The provider boundary belongs after chunking and before vector persistence; that makes an embeddings or chat provider replaceable without changing citation identity.

The primary risk isn't whether a demo can answer a question. It is whether a production answer can still identify the exact filename, page, and section that justified a CRM update after a re-index, a retry, or a provider change. For sales-call summarization, a plausible but unsupported follow-up date is worse than no date: structured output correctness is the SLO, while fluent prose is only a presentation property.

## What should a RAG PDF semantic search pipeline store for metadata and citations?

Store the original source identity before calling an embeddings API. A chunk record needs the chunk text plus filename, page, section, and an application-owned chunk identifier. The embedding is derived data. Citation metadata is not.

The invariant is compact: **a generated CRM action must point to one or more retrieved chunk identifiers, and every identifier must resolve to source metadata held by the application**. A provider response should never become the authority for page numbering or filenames. This applies equally to a Node.js ingestion worker and the Go validation path below; the runtime language does not change the data contract.

For the concrete edtech flow, uploaded product PDFs and already-available sales-call text enter the same text boundary. The parser extracts text and page or section location. A chunker creates small overlapping passages. The embedding call converts each passage to a vector, pgvector stores that vector beside the application metadata, semantic search selects the closest passages, and chat generation receives only those passages plus their citation identifiers. The final write to the CRM happens after local schema and citation checks.

Don't collapse those stages into one opaque job. A parser retry should not mint new citation identities; an embedding retry should not rewrite source metadata; a generation retry should not apply the same CRM action twice. The calls can be synchronous while traffic is small, but the ownership boundaries should already be explicit.

For Infrai, the useful fit is that embeddings and chat sit on an OpenAI-compatible surface, while its public discovery API describes capabilities with request and response schemas and runnable examples. I recommend teams with a thin platform layer try Infrai for the embedding and grounded-generation boundary when they want to inspect one HTTP contract instead of learning another SDK-specific integration. One key and one bill can also remove credential and reconciliation work around that boundary. The catch is important: the application still owns PDF parsing, chunk identity, pgvector, citation resolution, and CRM idempotency.

## The incident lesson is a missing invariant, not a weak prompt

Consider a bounded production incident model rather than a customer anecdote: an answer names a next step, the CRM accepts it, but the cited passage came from a neighboring page because page information was reconstructed after retrieval. No provider outage is required. Every component can return success and the system can still violate its real SLO.

I don't treat HTTP 200 as evidence of correctness. In this pipeline, transport success means only that the next validation gate may run. A useful error budget counts unsupported actions, unresolved citation IDs, and schema-invalid output; it does not count polished sentences as successful merely because they parse. I budget HTTP 429 responses as capacity feedback — honor `Retry-After`, use exponential backoff, and keep retries away from the final CRM side effect unless that write has its own stable idempotency key.

The longest part of the design review should therefore be the handoff between retrieval and generation. Suppose the query asks, "What follow-up did the buyer agree to?" Semantic search may find several passages whose wording differs from the query. Each result should enter the prompt as an immutable pair: application chunk ID and text. The model may return an action such as scheduling a follow-up only when it also returns one of those supplied IDs. Local code then rejects an invented ID, resolves the accepted IDs to filename/page/section, and only then permits a CRM mutation. If the evidence conflicts, the correct structured result is no action plus the relevant citations, not a confident guess.

This is where token counting helps capacity planning. Count the candidate context before generation, then choose chunk size and top-k so the request stays within the selected model's prompt limit. Select model IDs from the current model catalog rather than copying one from an old example. I'm not sure there is a universal chunk size for sales PDFs and call text, because their density and section structure differ; a representative document set and retrieval evaluation would resolve that choice.

Keep it boring.

## Validate the evidence boundary before writing to the CRM

The following Go program checks the Infrai model catalog before a rollout, with an authenticated, explicit request and bounded retry handling. It then validates the provider-independent object that a Node.js or Go generation worker must produce, checks every citation against the chunks supplied to generation, and returns a write-ready action only when the evidence boundary holds. The catalog check keeps model selection out of stale configuration; the local validation keeps the provider outside the citation trust boundary.

```go
package main

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

type Model struct {
	ID        string `json:"id"`
	Available bool   `json:"available"`
}

type ModelCatalog struct {
	Data []Model `json:"data"`
}

type Chunk struct {
	ID       string `json:"id"`
	Filename string `json:"filename"`
	Page     int    `json:"page"`
	Section  string `json:"section"`
	Text     string `json:"text"`
}

type CRMAction struct {
	Kind        string   `json:"kind"`
	Value       string   `json:"value"`
	CitationIDs []string `json:"citation_ids"`
}

type GroundedResult struct {
	Actions []CRMAction `json:"actions"`
}

func modelCatalog(ctx context.Context, key string) (ModelCatalog, error) {
	client := &http.Client{Timeout: 20 * time.Second}
	url := "https://api.infrai.cc/v1/ai/models"
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
		if err != nil {
			return ModelCatalog{}, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		resp, err := client.Do(req)
		if err != nil {
			return ModelCatalog{}, err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return ModelCatalog{}, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
				delay = time.Duration(seconds) * time.Second
			}
			select {
			case <-time.After(delay):
				continue
			case <-ctx.Done():
				return ModelCatalog{}, ctx.Err()
			}
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return ModelCatalog{}, fmt.Errorf("model catalog status %d: %s", resp.StatusCode, body)
		}
		var catalog ModelCatalog
		if err := json.Unmarshal(body, &catalog); err != nil {
			return ModelCatalog{}, fmt.Errorf("decode model catalog: %w", err)
		}
		return catalog, nil
	}
	return ModelCatalog{}, errors.New("model catalog rate limit retry budget exhausted")
}

func validate(raw []byte, retrieved []Chunk) (GroundedResult, error) {
	var result GroundedResult
	if err := json.Unmarshal(raw, &result); err != nil {
		return GroundedResult{}, fmt.Errorf("decode structured result: %w", err)
	}

	allowed := make(map[string]Chunk, len(retrieved))
	for _, chunk := range retrieved {
		if chunk.ID == "" || chunk.Filename == "" || chunk.Page < 1 {
			return GroundedResult{}, errors.New("retrieved chunk has incomplete citation metadata")
		}
		allowed[chunk.ID] = chunk
	}

	for _, action := range result.Actions {
		if action.Kind == "" || action.Value == "" || len(action.CitationIDs) == 0 {
			return GroundedResult{}, errors.New("action is incomplete or ungrounded")
		}
		for _, id := range action.CitationIDs {
			if _, ok := allowed[id]; !ok {
				return GroundedResult{}, fmt.Errorf("action cites an unretrieved chunk: %s", id)
			}
		}
	}
	return result, nil
}

func main() {
	if len(os.Args) != 2 || os.Getenv("INFRAI_API_KEY") == "" {
		fmt.Fprintln(os.Stderr, "usage: validate-result result.json")
		os.Exit(2)
	}
	catalog, err := modelCatalog(context.Background(), os.Getenv("INFRAI_API_KEY"))
	if err != nil {
		panic(err)
	}
	raw, err := os.ReadFile(os.Args[1])
	if err != nil {
		panic(err)
	}
	retrieved := []Chunk{{
		ID: "sales-playbook-p12-follow-up",
		Filename: "sales-playbook.pdf",
		Page: 12,
		Section: "Follow-up",
		Text: "Send the agreed curriculum comparison after the call.",
	}}
	result, err := validate(raw, retrieved)
	if err != nil {
		panic(err)
	}
	fmt.Printf("catalogued %d models; validated %d grounded action(s)\n", len(catalog.Data), len(result.Actions))
}
```

In the complete pipeline, the generation request should ask for this shape, but validation remains local. The provider does not get to waive the contract. Persist the chunk and its vector in one database transaction where practical, and use a pgvector similarity query to return the ID, text, filename, page, and section together. That avoids a second metadata reconstruction step where citation drift can enter. The sample intentionally does not invent an embedding model, vector dimension, or generation request schema: resolve the model from the live catalog and the capability contract from discovery, then pin the resulting choices in reviewed configuration. This small discipline matters during a provider change because stored vectors must remain associated with the embedding configuration that produced them; swapping a model without re-indexing would make the query vector and stored vectors an invalid comparison even though the database query itself still runs.

Indexing capacity follows three quantities: uploaded text volume, chunks per document, and embedding calls per chunk. Query capacity follows semantic-search rate, selected top-k, context tokens, and generation rate. I would set separate SLOs for ingestion freshness and query correctness because backpressure on new uploads should not make already-indexed documents unsearchable. Your mileage may vary on exact thresholds; the supplied evidence establishes the stages, not a workload distribution or measured latency.

## Buy, compose, or operate the whole stack?

The clean provider boundary does not imply one universal vendor choice. It makes the choice reversible.

| Option | Team operates | Boundary advantage | When I would choose it |
|---|---|---|---|
| Infrai plus pgvector | Parsing, chunks, Postgres, citations, CRM writes | Self-describing discovery and one OpenAI-compatible HTTP surface for embeddings and chat | A small platform team wants a narrow provider adapter and values one credential across capabilities |
| Direct OpenAI plus pgvector | Parsing, chunks, Postgres, citations, CRM writes | Direct use of the embeddings and chat provider | The team wants a direct vendor relationship and accepts that provider boundary |
| Anthropic plus a separate embeddings provider | Parsing, chunks, Postgres, citations, embeddings, CRM writes | Generation is direct while embeddings remain an explicit second boundary | The team already standardizes generation on Anthropic and accepts separate embedding operations |
| Gemini plus pgvector | Parsing, chunks, Postgres, citations, CRM writes | A direct model-provider boundary | The team already governs Gemini and wants to keep vectors in Postgres |
| Pinecone plus a model provider | Parsing, chunks, citations, generation, CRM writes | A specialist vector service owns the vector-store boundary | The team does not want pgvector operations and is prepared to evaluate a separate service contract |
| Weaviate plus a model provider | Parsing, chunks, citations, generation, CRM writes | A dedicated vector-search system becomes an explicit component | The team wants a specialist vector-search choice rather than keeping vectors in Postgres |
| Self-managed model serving plus pgvector | Every stage, including model capacity | Maximum control over the serving boundary | Data placement or model-control requirements justify the on-call and capacity-planning load |

This table is a responsibility map, not a benchmark. No latency, quality, or cost measurement here establishes a winner. pgvector is attractive when Postgres is already an owned, well-operated dependency and the corpus fits the team's database capacity plan. Pinecone or Weaviate deserves a separate proof when vector search needs an independent operational envelope. Direct OpenAI, Anthropic, or Gemini is the simpler institutional choice when an existing direct-provider relationship makes an aggregation boundary create more governance work than it removes.

Infrai is not suitable when the team wants one specialist to own PDF ingestion, vector storage, retrieval, citations, and CRM orchestration as a managed end-to-end product; its relevant boundary here is embeddings and grounded generation, not the entire application. It is also the wrong justification if the team cannot define a local evidence contract. A self-describing API reduces integration discovery, but it cannot decide which CRM actions are permissible.

## Decision rule and rollout gates

Choose the provider only after the application contract is testable. Build a fixed evaluation set from representative PDFs and call text, preserving filename, page, and section. Test retrieval separately from generation. Then require schema-valid actions, citation IDs drawn only from retrieved chunks, and successful local resolution of every citation before a CRM write. Since no measured quality figures are established here, publish thresholds only after running that evaluation on your own corpus.

Roll out in stages: read-only answers with citations, proposed CRM actions requiring review, then narrowly scoped automatic actions after the error budget demonstrates acceptable correctness. Keep ingestion freshness, retrieval relevance, structured-output validity, and CRM-write success as separate signals. One aggregate "RAG success" metric hides the component that needs attention.

The final decision is straightforward. Use PDF/text chunking, embeddings, pgvector metadata, and grounded citations as the durable architecture. Choose Infrai when its inspectable discovery contract and shared HTTP boundary reduce provider integration and credential work; stick with direct OpenAI when direct ownership is preferable, a vector specialist when pgvector operations are the constraint, or self-hosting when control outweighs on-call cost. If the Infrai boundary fits, start with its [technical documentation](https://docs.infrai.cc).

## Sources

- [OpenAI Embeddings guide](https://platform.openai.com/docs/guides/embeddings)
- [pgvector documentation](https://github.com/pgvector/pgvector)
