# Embedding Models, Part 2 — Choosing and Operating One

> **Where we are:** [Part 1](./embedding-models-part-1-concepts.md) covered what an embedding actually is, how the vector space is trained, dimensionality and Matryoshka truncation, distance metrics, the dense/sparse/multi-vector/multimodal taxonomy, the symmetric-vs-asymmetric search distinction, and where Bedrock's model catalog sits in all of that. Part 2 is the operational half: how to actually pick a model, evaluate it against your own data, deploy it, and keep it healthy once real traffic hits it.

---

## 1. Choosing a model — start with the benchmark, finish with your own data

**MTEB (Massive Text Embedding Benchmark)** is the standard public leaderboard, hosted on Hugging Face, scoring models across dozens of tasks: retrieval, classification, clustering, reranking, semantic similarity, across many languages. It's a legitimate starting point for narrowing the field — but treat it as a screening tool, not a purchase decision.

Why the gap between leaderboard rank and your results matters:

- **Domain mismatch.** MTEB's retrieval tasks lean on Wikipedia, news, and general Q&A datasets. A model that tops that leaderboard has no guarantee of understanding Terraform state files, Datadog alert payloads, or your company's internal acronyms. Domain-specific evaluation isn't a nice-to-have; it's the only evaluation that actually predicts your outcome.
- **Task mismatch.** A model tuned for symmetric semantic similarity can underperform on the asymmetric short-query-long-document retrieval your RAG system actually does (Part 1, §6). Look specifically at MTEB's *Retrieval* subscore, not the aggregate average.
- **Language and format mismatch.** If a meaningful slice of your corpus is code, YAML, or non-English text, check the model's reported performance on those splits specifically — an aggregate English-prose score tells you little about either.

### The eval set you should actually build

Before evaluating any model, assemble 30–100 real (query, correct-chunk) pairs from your own domain — pull them from support tickets, past incident questions, or have a few engineers write realistic questions against your existing docs. This is the same eval set the Knowledge Bases guide recommended keeping for regression testing; it does double duty here.

For each candidate model, compute standard retrieval metrics against that eval set:

- **Recall@K** — of the top K results returned, what fraction of queries had their correct chunk somewhere in there? The most intuitive metric to start with.
- **MRR (Mean Reciprocal Rank)** — rewards the correct answer appearing *early*, not just present. A correct chunk at rank 1 scores far better than the same chunk at rank 8.
- **NDCG (Normalized Discounted Cumulative Gain)** — like MRR but handles graded relevance (some chunks are "the answer," others are "somewhat relevant"), useful once your eval set has more nuance than binary correct/incorrect.

```python
def recall_at_k(results: list[list[str]], correct: list[str], k: int = 5) -> float:
    """results: retrieved chunk IDs per query, ranked. correct: the known-good chunk ID per query."""
    hits = sum(1 for r, c in zip(results, correct) if c in r[:k])
    return hits / len(correct)
```

Run this same harness against every model you're evaluating, on the same eval set, with the same chunking. The winner is whichever model actually retrieves *your* runbooks correctly — which, more often than people expect, is not the model sitting at #1 on the public leaderboard.

## 2. Fine-tuning an embedding model — when it's worth it

Most teams never need this; managed models cover the vast majority of use cases well. Fine-tuning earns its complexity when:

- Your domain vocabulary is dense with jargon or identifiers general models weren't trained on (internal service names, product codenames, error taxonomies).
- Your eval numbers plateau below what the business needs, and better chunking/hybrid search haven't closed the gap.
- You have — or can generate — a meaningful volume of labeled (query, relevant-document) pairs, ideally in the thousands.

The mechanism is the same contrastive learning from Part 1, §2, just continuing training from a pretrained checkpoint on your labeled pairs instead of training from scratch. In practice this means using `sentence-transformers` (Python) with a base open-source model:

```python
from sentence_transformers import SentenceTransformer, InputExample, losses
from torch.utils.data import DataLoader

model = SentenceTransformer("BAAI/bge-base-en-v1.5")

train_examples = [
    InputExample(texts=["rollback procedure for payments service", "To roll back payments-svc, run..."]),
    InputExample(texts=["CrashLoopBackOff on pod", "Check container logs with kubectl logs --previous..."]),
    # ... hundreds to thousands more real pairs
]

train_loader = DataLoader(train_examples, shuffle=True, batch_size=16)
loss = losses.MultipleNegativesRankingLoss(model)

model.fit(
    train_objectives=[(train_loader, loss)],
    epochs=3,
    warmup_steps=100,
    output_path="./fine-tuned-devops-embeddings",
)
```

`MultipleNegativesRankingLoss` is the standard choice for retrieval fine-tuning: within each batch, every other example's "document" side acts as a negative for your "query" side, so you get a lot of useful contrastive signal without hand-labeling explicit negatives.

**The operational cost of this path**: a fine-tuned model is no longer Bedrock's managed embedding endpoint. You now own hosting (SageMaker endpoint, or self-hosted behind your own service), versioning, and retraining cadence as data drifts — the trade Part 1 flagged as the "escape hatch" territory. Don't reach for it before proving the managed models genuinely fall short on your own eval set.

## 3. Deployment options, compared

| Option | You manage | Best for |
|---|---|---|
| **Bedrock (Titan/Cohere)** | Nothing — pure API call | Default choice; fastest to production, scales automatically, pay-per-token |
| **SageMaker endpoint (open-source model)** | Instance sizing, scaling, patching | Fine-tuned or specialized open-source models (BGE, E5, GTE) not on Bedrock |
| **Self-hosted (ECS/EKS + sentence-transformers)** | Everything | Full control, air-gapped/regulatory requirements, or embedding as a side-car process |

For the overwhelming majority of teams building a Knowledge Base, staying on Bedrock's managed embedding models is the right default — it's the same reasoning as the Knowledge Bases guide's overall thesis: don't build infrastructure AWS will operate for you unless you have a specific, proven reason not to.

## 4. Calling embedding models directly with Python

Sometimes you need raw vectors outside the Knowledge Base flow — for clustering, deduplication, or a custom retrieval pipeline you're building by hand.

**Bedrock Titan Text Embeddings V2:**

```python
import boto3, json

bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")

def embed_titan(text: str, dimensions: int = 1024) -> list[float]:
    response = bedrock.invoke_model(
        modelId="amazon.titan-embed-text-v2:0",
        body=json.dumps({
            "inputText": text,
            "dimensions": dimensions,      # 256, 512, or 1024 — Matryoshka truncation
            "normalize": True,             # unit-normalize, so cosine == dot product
        }),
    )
    return json.loads(response["body"].read())["embedding"]

vec = embed_titan("The pod is stuck in CrashLoopBackOff")
print(len(vec), vec[:5])
```

**Cohere Embed, with explicit asymmetric input types (Part 1, §5.5/§6):**

```python
def embed_cohere(texts: list[str], input_type: str) -> list[list[float]]:
    """input_type: 'search_query' | 'search_document' | 'classification' | 'clustering'"""
    response = bedrock.invoke_model(
        modelId="cohere.embed-english-v3",
        body=json.dumps({
            "texts": texts,
            "input_type": input_type,
        }),
    )
    return json.loads(response["body"].read())["embeddings"]

query_vec = embed_cohere(["rollback procedure for payments"], input_type="search_query")
doc_vecs  = embed_cohere(["To roll back payments-svc, run helm rollback..."], input_type="search_document")
```

Getting `input_type` right is not cosmetic — using `search_document` for a short user query (or vice versa) measurably degrades ranking quality with Cohere's models, precisely because the model was trained to expect the asymmetry to be labeled.

**Computing similarity by hand (useful for debugging retrieval issues outside the vector store):**

```python
import numpy as np

def cosine_similarity(a: list[float], b: list[float]) -> float:
    a, b = np.array(a), np.array(b)
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))

score = cosine_similarity(query_vec[0], doc_vecs[0])
print(round(score, 4))
```

When a Knowledge Base returns a surprising result, dropping into a notebook and computing this by hand against a few candidate chunks is often faster than staring at OpenSearch relevance scores trying to reason about them abstractly.

## 5. Batch embedding a corpus — throughput and cost mechanics

Embedding an entire document corpus at ingestion time (whether by hand or via a Knowledge Base sync) is the expensive event Part 1 flagged. A few concrete practices:

- **Batch your requests.** Both Titan and Cohere accept multiple texts per API call (Cohere natively; Titan via multiple invocations, though check current batch API availability). Batching cuts round-trip overhead substantially versus one document per call.
- **Deduplicate before embedding.** Near-identical chunks (boilerplate headers, repeated legal disclaimers across every PDF) waste embedding spend for zero retrieval benefit. A cheap MinHash or simple exact-match dedup pass before embedding pays for itself immediately at any real corpus size.
- **Cache aggressively.** If your ingestion pipeline re-runs on unchanged documents (a common CI misconfiguration), you're paying to re-embed identical text. Hash each chunk's content and skip re-embedding on a cache hit — this is exactly the kind of quiet, compounding cost that shows up as a monthly AWS bill line item nobody investigates until it triples.
- **Watch token limits per input.** Titan v2 caps around 8K tokens per input; Cohere similarly bounds input length. This is yet another reason chunking strategy (from the Knowledge Bases guide) and embedding model choice aren't independent decisions — oversized chunks silently truncate rather than erroring in some configurations, quietly losing the back half of every long section.

## 6. Versioning and migration — the operational trap

Part 1 flagged this as embedding models' one genuinely irreversible decision, and it deserves the operational detail here: **vectors from different models, or even different dimension settings of the same model, are not comparable.** A 1024-dim Titan v2 vector and a 1536-dim OpenAI vector don't share a coordinate system — comparing them is meaningless, not just imprecise.

This has real consequences for how you run a Knowledge Base or any embedding-backed system over time:

- **Switching models requires re-embedding the entire corpus.** There's no incremental migration path — every existing vector in the store is now stale and must be recomputed and re-indexed from source. This is precisely why the source documents living durably in S3 (rather than only existing as vectors) is a design principle worth protecting, not an incidental detail.
- **Never mix vectors from different models/settings in one index.** If you truncate to 512 dimensions for half your corpus and use 1024 for the other half, similarity comparisons across that boundary are simply wrong — the vector store won't warn you; it'll return confidently incorrect rankings.
- **Plan model upgrades as a blue/green operation:** build a new index with the new model, validate it against your eval set (§1) side-by-side with the old index, then cut over — rather than upgrading in place. Bedrock Knowledge Bases doesn't currently support in-place embedding model swaps for exactly this reason; you create a new data source or new Knowledge Base.
- **Record which model and dimension embedded each index**, in Terraform state, tags, or a runbook — not just in someone's memory. "Which embedding model is this OpenSearch collection actually using" is a genuinely common incident-time question with an embarrassingly common answer of "nobody's sure."

## 7. Monitoring and drift — what actually goes wrong in production

Embedding-backed retrieval fails quietly. Nothing throws an exception when results are subtly worse; they're just worse. A few concrete things worth watching:

- **Re-run your eval set (§1) on a schedule**, not just at initial model selection. If retrieval metrics drift downward over months, it's often not the model — it's the corpus drifting away from what the model (or your chunking) was tuned against, e.g., a new product line introducing vocabulary the model handles worse.
- **Track retrieval score distributions, not just top-K success.** A sudden drop in average top-1 cosine similarity across queries is an early warning sign worth alerting on, well before users start complaining that answers feel off.
- **Log queries with no chunk above a reasonable similarity threshold.** These are your corpus gaps — questions the Knowledge Base structurally cannot answer well, which is a documentation problem, not a model problem, and conflating the two sends people chasing the wrong fix.
- **Sanity-check with known-similar and known-dissimilar pairs periodically**, especially after any infrastructure change (region migration, model version bump within the "same" model, vector store upgrade) — a quick regression test that the geometry still behaves as expected, since silent misconfiguration (wrong distance metric, mismatched normalization) produces plausible-looking but degraded results rather than clean failures.

## 8. Quick decision guide

```
Do you need retrieval for a Knowledge Base / RAG system?
│
├── Yes, and general text (docs, wikis, tickets)
│     → Titan Text Embeddings V2. Default choice, Matryoshka flexibility, AWS-native.
│
├── Yes, and query/document asymmetry matters a lot (short Qs, long docs)
│     → Cohere Embed with explicit input_type. Built for exactly this.
│
├── Yes, and exact identifiers matter (error codes, part numbers, CVEs)
│     → Add hybrid (dense + keyword/sparse) search. Don't rely on dense alone.
│
├── Yes, and it's images + text together
│     → Titan Multimodal Embeddings.
│
├── Domain vocabulary general models handle poorly, eval numbers won't budge
│     → Consider fine-tuning an open-source base model (BGE/E5) via SageMaker.
│
└── No RAG — clustering, dedup, anomaly detection, classification
      → Any solid general embedding model; task fit matters more than the leaderboard rank.
```

---

*Suggested next step: take the eval set you (hopefully) already built for your Knowledge Base and run it through both Titan v2 and Cohere Embed side-by-side using the harness in §1 — before assuming the default is the right one for your corpus.*
