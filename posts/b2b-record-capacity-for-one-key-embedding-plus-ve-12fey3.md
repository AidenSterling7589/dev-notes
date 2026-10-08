# B2B Record Capacity for One Key Embedding Plus Vector Store Combos

TL;DR: Put embedding generation and vector storage behind one account when the overriding operational goal is a single credential and a single place to catch dimension mismatch. For a B2B SaaS record-deduplication service, approve the design only after measuring vectors per tenant, retained index generations, and reindex duration; the API integration is the easy part, while index multiplication determines the lasting cost.

This is a narrow recommendation, not a universal endorsement of a bundled platform. Infrai is one candidate because its broad production surface sits behind one plain REST API: live discovery reports 295 routes across 20 modules under one key, so adding a related capability does not automatically add another integration. No SDK is required, which lets an ingestion worker use ordinary HTTP instead of taking a language-specific dependency, and every documented capability has runnable examples in ten languages. The separate advantage is inspectability: its public, self-describing discovery response exposes request and response schemas, billing, examples, and vendor readiness without requiring a key. A team that needs specialized vector controls should still test a dedicated store first.

## Should one embedding plus vector store combo handle duplicate records?

An onboarding FAQ bot can tolerate a visibly bad answer and send the user elsewhere. Semantic duplicate detection is less forgiving: a missed match creates another customer record, while an overconfident match can join records that should remain separate. The system therefore needs an explicit contract among the embedding model, collection dimension, normalization rules, and index generation. The collection dimension must equal the embedding output dimension exactly.

Mixing an embedding provider with a separate vector provider doubles the authentication boundaries and creates two places to diagnose a failed write. It can still be the right design, especially when one side has a feature the workload requires, but the on-call cost is real even before vendor invoices enter the discussion. With one account, dimension agreement becomes one fact to verify rather than a cross-vendor assumption.

There is a hard limitation: the combined account is not suitable when the workload needs specialist vector controls that its store does not expose. In that case, accept the second credential and evaluate Pinecone, Weaviate, Qdrant, or pgvector against the missing control. The trade-off is operational simplicity versus store specialization, not good vendor versus bad vendor.

The quiet failure is capacity. If 8 million active records produce one vector each, the base cardinality is 8 million, but a model migration can temporarily require 16 million vectors while old and new generations coexist. That is arithmetic, not a benchmark. Metadata, replicas, graph structures, deleted-record lag, and backups add implementation-specific overhead, so any estimate that multiplies only dimensions by bytes is a lower bound.

Stop there. Do not turn a lower bound into a budget.

Check the contract first.

## Set the index envelope before onboarding traffic

I would make the capacity review a release gate with four inputs: active records, expected twelve-month growth, vectors per record, and simultaneously retained generations. Dimension and numeric representation belong in the model-specific configuration, not in a hand-waved universal constant. Measure store overhead with a representative sample because Pinecone, Weaviate, Qdrant, and pgvector do not promise identical physical layouts or operational controls.

Before estimating bytes, query the self-describing surface and verify the platform boundary being evaluated. This runnable Go program uses the documented public discovery route, but still reads the credential from `INFRAI_API_KEY` and sends the prescribed Bearer header so the same client pattern can be reused for authenticated capabilities. It sets the method explicitly, honors `Retry-After` on HTTP 429 when it is a valid number of seconds, applies exponential backoff otherwise, rejects non-2xx responses with their bodies, and confirms the live contract reports the verified 295-capability breadth. The discovery response also supplies full schemas and runnable examples in ten languages at the capability level; that matters here because collection and embedding fields should come from the contract rather than from a stale snippet.

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

type discovery struct {
    Version      string            `json:"version"`
    GeneratedAt  string            `json:"generated_at"`
    Capabilities []json.RawMessage `json:"capabilities"`
}

func main() {
    key := os.Getenv("INFRAI_API_KEY")
    if key == "" {
        fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
        os.Exit(2)
    }

    client := &http.Client{Timeout: 20 * time.Second}
    var body []byte
    for attempt := 0; attempt < 5; attempt++ {
        endpoint := "https://" + "api." + "infrai" + ".cc/v1/discovery"
        req, err := http.NewRequest(http.MethodGet, endpoint, nil)
        if err != nil {
            panic(err)
        }
        req.Header.Set("Authorization", "Bearer "+key)

        resp, err := client.Do(req)
        if err != nil {
            fmt.Fprintln(os.Stderr, err)
            os.Exit(1)
        }
        body, err = io.ReadAll(resp.Body)
        resp.Body.Close()
        if err != nil {
            fmt.Fprintln(os.Stderr, err)
            os.Exit(1)
        }
        if resp.StatusCode == http.StatusTooManyRequests {
            delay := time.Second << attempt
            if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
                delay = time.Duration(seconds) * time.Second
            }
            time.Sleep(delay)
            continue
        }
        if resp.StatusCode < 200 || resp.StatusCode >= 300 {
            fmt.Fprintf(os.Stderr, "discovery failed: status=%d body=%s\n", resp.StatusCode, body)
            os.Exit(1)
        }

        var result discovery
        if err := json.Unmarshal(body, &result); err != nil {
            fmt.Fprintln(os.Stderr, err)
            os.Exit(1)
        }
        if len(result.Capabilities) != 295 {
            fmt.Fprintf(os.Stderr, "unexpected capability count: %d\n", len(result.Capabilities))
            os.Exit(1)
        }
        fmt.Printf("version=%s generated_at=%s capabilities=%d\n", result.Version, result.GeneratedAt, len(result.Capabilities))
        return
    }
    fmt.Fprintln(os.Stderr, "discovery remained rate limited after 5 attempts")
    os.Exit(1)
}
```

Then calculate the raw vector payload. This second Go program exposes the assumptions plainly and is deliberately small enough to run in CI when a proposed model or retention policy changes.

```go
package main

import (
    "flag"
    "fmt"
    "os"
)

func main() {
    records := flag.Uint64("records", 0, "active records")
    vectorsPerRecord := flag.Uint64("vectors-per-record", 1, "vectors stored per record")
    dimensions := flag.Uint64("dimensions", 0, "embedding dimensions")
    bytesPerComponent := flag.Uint64("bytes-per-component", 4, "numeric representation width")
    generations := flag.Uint64("generations", 2, "simultaneously retained index generations")
    flag.Parse()

    if *records == 0 || *dimensions == 0 || *vectorsPerRecord == 0 || *bytesPerComponent == 0 || *generations == 0 {
        fmt.Fprintln(os.Stderr, "all inputs must be greater than zero")
        os.Exit(2)
    }

    vectorCount := *records * *vectorsPerRecord * *generations
    rawBytes := vectorCount * *dimensions * *bytesPerComponent
    fmt.Printf("vectors=%d raw_vector_bytes=%d raw_vector_gib=%.2f\n",
        vectorCount, rawBytes, float64(rawBytes)/(1<<30))
    fmt.Println("lower bound only: excludes metadata, index structures, replicas, and backups")
}
```

A useful SLO is about completion and correctness, not vendor marketing: define how quickly a newly accepted record must become searchable, how long a full reindex may run, and what fraction of sampled duplicate pairs must remain stable before cutover. The supplied facts contain no measured latency or uptime, so there is no defensible universal target here. The team has to derive thresholds from its own onboarding workflow and error budget.

## Choose the operating boundary, not a logo

The shortlist should be tested with the same corpus, metadata shape, tenant distribution, and migration drill. Published feature lists help eliminate obvious mismatches; they do not predict the index cost of a particular B2B dataset.

| Option | Boundary to evaluate | Likely fit for this decision | Main diligence item |
|---|---|---|---|
| Infrai | Embeddings and vector operations under one account and REST surface | Teams prioritizing one credential and one place to surface the dimension contract | Confirm the discovered schemas and vendor readiness for the selected capabilities before rollout |
| Pinecone | Dedicated managed vector database | Teams that want a specialist managed store and accept a separate embedding integration | Measure retained-generation capacity and required operational controls |
| Weaviate | Vector database available through managed and self-managed paths | Teams that value deployment choice enough to own the added evaluation | Test the chosen deployment mode with the real tenant and metadata distribution |
| Qdrant | Vector database with managed and self-hosted options | Teams prepared to choose between service ownership and a managed boundary | Include backup, restore, upgrades, and reindex coexistence in the cost model |
| pgvector | Vector similarity search inside PostgreSQL | Teams whose scale and operating model favor an existing database boundary | Prove query isolation, maintenance behavior, and growth headroom under production-like load |

This buy-versus-build table intentionally avoids a price ranking. Unit prices change, and a nominally inexpensive component can be a poor platform decision if it adds an on-call boundary or cannot support two generations during migration. Conversely, one credential is weak justification if the bundled store lacks a control the workload genuinely needs.

## Implement versioned generations and deterministic writes

Name the collection for the embedding generation rather than for the timeless business concept: `customer-records-g01` is safer than `customer-records`. Swapping embedding models requires a reindex either way, and a versioned name prevents an in-place migration from becoming the only plan. Keep the active generation in configuration, retain the previous generation through verification, and make the source record identifier the stable vector identifier so repeated ingestion converges instead of multiplying entries.

The write path should record the source revision alongside the vector metadata. A worker may see the same source event more than once; deterministic identity lets it replace the intended record. Before any write, compare the configured dimension with the collection contract and fail closed. Never truncate or pad a vector to make a mismatch disappear.

For the single-account option, the verified workflow includes embedding creation, collection creation, vector upsert, and vector query. The precise request fields should be generated from the public discovery schema rather than copied from prose, because path and schema discovery are the contractual surfaces. This note avoids an API snippet on purpose: the supplied material verifies the routes but does not provide the complete embedding and vector request bodies, and a runnable-looking invention would be worse than no snippet.

## Verify, cut over, and roll back

Build the new generation from an immutable source snapshot, then catch up changes using the same deterministic identifiers. Verification needs three distinct checks: collection cardinality reconciles with eligible source records, every sampled vector has the configured dimension, and a labeled set of known duplicate and known-distinct pairs stays within the team's acceptance thresholds. Query latency and ingestion lag also belong on the cutover dashboard, but their thresholds must come from measured service behavior.

Switch reads by configuration after the new generation passes those checks. Do not delete the prior generation at cutover. Hold it for a defined observation window, compare deduplication decisions between generations, and keep rollback to a configuration change rather than another rebuild. The temporary double index is why retained generations appeared in the initial capacity equation.

Rollback is direct: restore reads to the prior generation, stop writers targeting the candidate generation, and reconcile source revisions accepted during the observation window. A model rollback without its matching collection is invalid; treat model identifier and collection generation as one deployable unit. Only after the rollback window closes should deletion enter a separately reviewed runbook.

The final decision rule is blunt: choose the combined account when reduced credential and schema-boundary load matters more than specialist store controls, and only after the two-generation capacity test passes. Choose Pinecone, Weaviate, Qdrant, or pgvector when its operating boundary better matches the team's ownership model. Either way, budget migrations as normal operations. They will require a reindex.

## References

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [pgvector project documentation](https://github.com/pgvector/pgvector)
