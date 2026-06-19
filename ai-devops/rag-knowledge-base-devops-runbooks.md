# RAG Knowledge Base for DevOps Runbooks

**Category:** AI DevOps / AIOps / LLMOps  
**Why this matters:** DevOps runbooks age badly — they live in Confluence wikis, Notion pages, and Git repos that nobody reads during an incident. A RAG (Retrieval-Augmented Generation) system puts that institutional knowledge directly in the hands of engineers via natural language queries, cutting mean time to diagnosis dramatically. For principal-level engineers, understanding how to design a production-grade RAG pipeline over live runbook data is a critical skill as AI becomes a first-class citizen in incident management.

---

## 1. Core Concept

RAG (Retrieval-Augmented Generation) is an architecture pattern that grounds LLM responses in your own private data by combining:

1. **Embedding + Vector Search** — Documents are chunked, embedded into high-dimensional vectors, and stored in a vector database. At query time, the user's question is embedded and the closest matching chunks are retrieved.
2. **LLM Generation with Context** — Retrieved chunks are injected into the LLM prompt as context. The model answers using *your* runbooks, not hallucinated general knowledge.

For DevOps runbooks, this means:
- An on-call engineer asks: *"What is the remediation for ECS task OOM kills in the payments service?"*
- The system retrieves the exact runbook section from your Confluence or Git repo
- The LLM synthesizes a concrete answer with the right commands, thresholds, and escalation paths

**Key difference from a search engine:** RAG doesn't just return document links — it reads the retrieved content and generates a synthesized, actionable answer.

**Chunking strategy is critical.** A runbook has structure (headings, code blocks, numbered steps). Poor chunking (fixed 512-token windows) destroys this structure. Semantic chunking — splitting on headings and section boundaries — dramatically improves retrieval quality.

**Metadata filtering reduces noise.** Tag each chunk with service name, team, severity, and environment. At query time, filter vectors by metadata before semantic search. This prevents a `payments` incident query from retrieving `auth` runbooks.

---

## 2. Production Engineering View

In production, a RAG pipeline for runbooks has several operational concerns that tutorials skip:

**Staleness problem:** Runbooks change. An embedding from 6 months ago may describe a deprecated procedure. You need an ingestion pipeline that watches your Confluence/Git source and re-indexes on changes. Use a webhook or a nightly sync job with hash-based change detection.

**Chunking for runbooks specifically:** Runbooks aren't articles. They have:
- Preconditions (when to use this runbook)
- Diagnosis steps (how to confirm the issue)
- Remediation steps (what to do)
- Escalation paths (who to call if it fails)

These sections should be separate chunks with shared metadata (`runbook_id`, `section_type`). Retrieving the "Remediation" section with high specificity beats retrieving half of a full runbook.

**Retrieval quality metrics:**
- **MRR (Mean Reciprocal Rank):** Is the correct runbook chunk in the top-3?
- **Context precision:** Of what's retrieved, what percentage is relevant?
- **Context recall:** Is the *critical* chunk actually being returned?

Run a nightly eval harness against a golden Q&A dataset of your top 20 historical incidents.

**Latency budget:** During an incident, a 10-second RAG response is unacceptable. Target under 2 seconds end-to-end. Use:
- Approximate nearest neighbor search (HNSW index, not flat search)
- Cached embeddings for the query vocabulary you know (service names, error codes)
- Response streaming to show text as it generates

**Access control:** Not all runbooks are public. Ensure your vector store respects the same permissions as your source. Tag chunks with a team/access-level and filter pre-retrieval.

---

## 3. Tools and Technologies

| Layer | Tool Options | Notes |
|-------|-------------|-------|
| **Embedding model** | `text-embedding-3-small` (OpenAI), `amazon.titan-embed-text-v2` (Bedrock), `cohere.embed-english-v3` | Bedrock embeddings stay in your VPC — preferred for enterprise |
| **Vector store** | Pinecone, pgvector (PostgreSQL), OpenSearch k-NN, Weaviate, ChromaDB | pgvector is excellent for teams already on RDS — no new infra |
| **LLM** | Claude 3.5 Sonnet (via Bedrock), GPT-4o, Llama 3 on SageMaker | Claude's 200K context window is ideal for multi-runbook synthesis |
| **Orchestration** | LangChain, LlamaIndex, custom Python | LlamaIndex has better out-of-box runbook/document parsing |
| **Runbook sources** | Confluence REST API, Git repo (Markdown), Notion API, PagerDuty runbooks | Confluence is most common in enterprise DevOps |
| **Eval framework** | RAGAS, custom harness with golden Q&A | RAGAS scores context recall/precision automatically |
| **Ingestion pipeline** | AWS Lambda + S3, Airflow DAG, GitHub Actions | Choose based on what triggers re-indexing |

---

## 4. Architecture Pattern

```
┌────────────────────────────────────────────────────────────────────────┐
│                         INGESTION PIPELINE                             │
│                                                                        │
│  Confluence / Git ──► Chunker ──► Embedder ──► Vector Store (pgvector) │
│       (webhook)      (by heading)  (Bedrock)    + Metadata index       │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│                           QUERY PIPELINE                               │
│                                                                        │
│  User Query (Slack/CLI)                                                │
│       │                                                                │
│       ▼                                                                │
│  Query Embedder ──► Metadata Filter ──► ANN Search (HNSW)             │
│                      (service=X,                                       │
│                       section=remediation)                             │
│       │                                                                │
│       ▼                                                                │
│  Top-K Chunks ──► Context Assembly ──► LLM (Claude via Bedrock)       │
│                   (dedup, rerank)       with system prompt             │
│       │                                                                │
│       ▼                                                                │
│  Answer + Source Citations ──► Slack thread / PagerDuty note          │
└────────────────────────────────────────────────────────────────────────┘
```

**Critical design decisions:**

- **Reranker:** A cross-encoder reranker (e.g., Cohere Rerank) re-scores retrieved chunks against the query before sending to the LLM. Reduces irrelevant context by ~40%.
- **Hybrid search:** Combine dense vector search with BM25 keyword search. Pure semantic search misses exact error codes like `Exit code 137`. Hybrid handles both.
- **Source citations in response:** Always include the runbook name, section, and last-modified date in the LLM output. On-call engineers must be able to trust and verify.

---

## 5. Common Mistakes

**Chunking on token count, not structure.** A 512-token chunk that splits a numbered remediation step across two chunks means the LLM gets half a procedure. Use heading-based or structural splitting.

**No metadata filtering.** Without filtering, a query about `payments-service CPU spike` retrieves runbooks from 10 unrelated services. Metadata filters by service/team/environment are not optional in enterprise environments.

**Embedding stale runbooks.** A vector store with 18-month-old runbook content can confidently return deprecated procedures. Build a freshness pipeline and surface the last-modified date in every response.

**Not evaluating retrieval separately from generation.** Engineers test by asking questions and reading answers. But a bad answer might be a retrieval problem (wrong chunks returned) OR a generation problem (right chunks, bad synthesis). Measure retrieval quality (MRR, recall) independently.

**Overly long context windows as a crutch.** Dumping 20 runbook chunks into a 200K context window and hoping the LLM figures it out is lazy and expensive. Retrieve 5-7 *high-precision* chunks. Quality of retrieval > quantity of context.

**No fallback for low-confidence responses.** If the retrieval score is below a threshold (e.g., cosine similarity < 0.75), the system should say "I don't have a confident match — here are the closest runbooks" rather than hallucinating.

---

## 6. Hands-on Lab

Build a minimal RAG system over local Markdown runbooks using pgvector and Claude via the Anthropic SDK.

### Prerequisites
```bash
pip install anthropic psycopg2-binary pgvector langchain-text-splitters
# Requires PostgreSQL with pgvector extension
# CREATE EXTENSION vector;
```

### Step 1: Ingest runbooks

```python
import os
import hashlib
import anthropic
import psycopg2
from pathlib import Path
from langchain_text_splitters import MarkdownHeaderTextSplitter

client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

def get_embedding(text: str) -> list[float]:
    """Use Anthropic's embedding endpoint (or swap for OpenAI/Bedrock)."""
    # Anthropic doesn't have a native embedding endpoint — use OpenAI or Bedrock
    # This example shows the pattern; swap the client for your embedding provider
    import openai
    response = openai.embeddings.create(
        model="text-embedding-3-small",
        input=text
    )
    return response.data[0].embedding

def chunk_runbook(file_path: str) -> list[dict]:
    """Split Markdown runbook by headings, preserving structure."""
    text = Path(file_path).read_text()
    splitter = MarkdownHeaderTextSplitter(
        headers_to_split_on=[("##", "section"), ("###", "subsection")]
    )
    chunks = splitter.split_text(text)
    
    # Extract service name from filename convention: payments-service-oom.md
    service = Path(file_path).stem.split("-")[0]
    
    return [
        {
            "content": chunk.page_content,
            "section": chunk.metadata.get("section", "general"),
            "service": service,
            "runbook": Path(file_path).name,
            "chunk_hash": hashlib.md5(chunk.page_content.encode()).hexdigest()
        }
        for chunk in chunks
        if len(chunk.page_content.strip()) > 100  # skip tiny chunks
    ]

def ingest_to_pgvector(runbook_dir: str, conn_string: str):
    conn = psycopg2.connect(conn_string)
    cur = conn.cursor()
    
    # Create table (run once)
    cur.execute("""
        CREATE TABLE IF NOT EXISTS runbook_chunks (
            id SERIAL PRIMARY KEY,
            content TEXT,
            section TEXT,
            service TEXT,
            runbook TEXT,
            chunk_hash TEXT UNIQUE,
            embedding vector(1536)
        );
        CREATE INDEX IF NOT EXISTS runbook_embedding_idx 
            ON runbook_chunks USING hnsw (embedding vector_cosine_ops);
    """)
    
    for md_file in Path(runbook_dir).glob("*.md"):
        for chunk in chunk_runbook(str(md_file)):
            # Skip if unchanged
            cur.execute("SELECT 1 FROM runbook_chunks WHERE chunk_hash = %s", 
                       (chunk["chunk_hash"],))
            if cur.fetchone():
                continue
            
            embedding = get_embedding(chunk["content"])
            cur.execute("""
                INSERT INTO runbook_chunks 
                    (content, section, service, runbook, chunk_hash, embedding)
                VALUES (%s, %s, %s, %s, %s, %s)
                ON CONFLICT (chunk_hash) DO NOTHING
            """, (chunk["content"], chunk["section"], chunk["service"],
                  chunk["runbook"], chunk["chunk_hash"], embedding))
    
    conn.commit()
    cur.close()
    conn.close()
    print("Ingestion complete.")
```

### Step 2: Query with Claude

```python
def query_runbooks(
    question: str,
    service_filter: str | None,
    conn_string: str,
    top_k: int = 5
) -> str:
    """Retrieve relevant runbook chunks and synthesize an answer with Claude."""
    
    query_embedding = get_embedding(question)
    
    conn = psycopg2.connect(conn_string)
    cur = conn.cursor()
    
    # Build query with optional metadata filter
    filter_clause = "WHERE service = %s" if service_filter else ""
    params = [query_embedding]
    if service_filter:
        params.insert(0, service_filter)
    
    cur.execute(f"""
        SELECT content, section, runbook, 
               1 - (embedding <=> %s::vector) AS similarity
        FROM runbook_chunks
        {filter_clause}
        ORDER BY embedding <=> %s::vector
        LIMIT {top_k}
    """, [*([service_filter] if service_filter else []), query_embedding, query_embedding])
    
    results = cur.fetchall()
    conn.close()
    
    if not results or results[0][3] < 0.75:
        return "No high-confidence runbook match found. Check runbooks manually."
    
    # Assemble context with source citations
    context_parts = []
    for content, section, runbook, similarity in results:
        context_parts.append(
            f"[Source: {runbook} | Section: {section} | Confidence: {similarity:.2f}]\n{content}"
        )
    context = "\n\n---\n\n".join(context_parts)
    
    # Generate answer with Claude
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        system="""You are a senior SRE assistant. Answer the engineer's question using ONLY 
the provided runbook context. Always cite the specific runbook and section. 
If the context doesn't contain enough information, say so clearly — do not guess.
Format responses with numbered steps when describing procedures.""",
        messages=[
            {
                "role": "user",
                "content": f"""Context from runbooks:
{context}

Engineer's question: {question}"""
            }
        ]
    )
    
    return response.content[0].text

# Example usage
if __name__ == "__main__":
    CONN = "postgresql://user:pass@localhost:5432/devops_kb"
    
    # Ingest
    ingest_to_pgvector("./runbooks/", CONN)
    
    # Query
    answer = query_runbooks(
        question="How do I handle OOM kills in ECS tasks for the payments service?",
        service_filter="payments",
        conn_string=CONN
    )
    print(answer)
```

### Step 3: Add a Slack command interface

```python
# Minimal Slack slash command handler (Flask)
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route("/runbook-query", methods=["POST"])
def handle_slack_command():
    text = request.form.get("text", "")
    
    # Parse optional service filter: "/runbook payments OOM kill remediation"
    parts = text.split(maxsplit=1)
    if len(parts) == 2 and parts[0].isidentifier():
        service, question = parts
    else:
        service, question = None, text
    
    answer = query_runbooks(
        question=question,
        service_filter=service,
        conn_string=os.environ["PG_CONN"]
    )
    
    return jsonify({
        "response_type": "in_channel",
        "text": f"*Runbook Assistant*\n{answer}"
    })
```

---

## 7. Interview Talking Points

**"Describe how you'd design a RAG system for internal DevOps knowledge."**

The key architectural decision in a production RAG system is that retrieval quality determines answer quality — the LLM can only work with what it's given. I'd start by auditing our runbook corpus: how many documents, what format (Markdown, Confluence, PDFs), how frequently they change, and what access controls apply. This drives the ingestion pipeline design. For chunking, I'd use structure-aware splitting on Markdown headers rather than fixed token windows, because runbooks have distinct sections (preconditions, diagnosis, remediation, escalation) that should stay together as semantic units. Each chunk gets metadata tags: service name, environment, severity, section type, and last-modified timestamp.

For the vector store, pgvector on our existing RDS cluster is my first choice in enterprise environments — it avoids a new managed service, supports HNSW indexing for sub-millisecond ANN search, and integrates with our existing IAM. The retrieval step uses hybrid search: dense vector similarity plus BM25 keyword matching, because exact error codes like "Exit code 137" or specific CloudWatch alarm names won't surface well in pure semantic search. A cross-encoder reranker then re-scores the top-20 candidates to return the top-5 to the LLM.

The LLM prompt is constrained with an explicit instruction: "Answer using only the provided context; if insufficient, say so." Source citations — runbook name, section, and last-modified date — are mandatory in every response. During an incident, an on-call engineer needs to trust the answer and be able to verify it instantly. I'd measure retrieval quality weekly using MRR against a golden dataset of our top 20 historical incident queries, and alert if MRR drops below 0.8.

**"How do you handle runbook drift in a RAG system?"**

Staleness is the silent killer of RAG systems in DevOps. I'd build a change-detection pipeline: a webhook from Confluence or a Git hook that fires on runbook updates, computes a content hash, and re-ingests only changed chunks. The last-modified date surfaces in every LLM response so engineers know if they're reading a 2-year-old procedure. I'd also run a quarterly audit that cross-references embedded chunks against the live source documents and flags any chunks whose source has been deleted or significantly changed.

---

## 8. AI DevOps / AIOps / LLMOps Angle

**Production pattern: Retrieval-grounded incident assistant with confidence gating**

The production-grade addition beyond the basic lab is *confidence gating* — refusing to answer when retrieval confidence is low rather than hallucinating. Here's a pattern for a PagerDuty-triggered runbook assistant using Claude:

```python
import anthropic
import os

client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

def runbook_assistant_with_confidence_gate(
    incident_title: str,
    service: str,
    retrieved_chunks: list[dict],  # [{"content": ..., "similarity": ..., "source": ...}]
) -> dict:
    """
    Confidence-gated runbook assistant.
    Returns answer with structured metadata for downstream systems.
    """
    
    # Confidence gate: require at least one chunk above threshold
    max_similarity = max((c["similarity"] for c in retrieved_chunks), default=0)
    
    if max_similarity < 0.72:
        return {
            "answer": None,
            "confidence": "low",
            "recommendation": "Escalate to on-call lead — no high-confidence runbook match.",
            "top_sources": [c["source"] for c in retrieved_chunks[:3]]
        }
    
    context = "\n\n".join([
        f"[{c['source']} | sim={c['similarity']:.2f}]\n{c['content']}"
        for c in retrieved_chunks
    ])
    
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=800,
        system="""You are an SRE incident assistant. You have runbook excerpts to help 
diagnose and remediate the current incident.

Rules:
1. Cite every claim with [source] notation.
2. If a step requires elevated privileges, say so explicitly.
3. If multiple runbooks conflict, surface the conflict — don't silently pick one.
4. Close with: "Escalate if: [conditions]"

Format: numbered steps, under 600 words.""",
        messages=[{
            "role": "user",
            "content": f"Incident: {incident_title}\nService: {service}\n\nRunbook context:\n{context}"
        }]
    )
    
    return {
        "answer": response.content[0].text,
        "confidence": "high" if max_similarity > 0.85 else "medium",
        "top_sources": [c["source"] for c in retrieved_chunks[:3]],
        "tokens_used": response.usage.input_tokens + response.usage.output_tokens
    }

# Simulate a PagerDuty webhook trigger
if __name__ == "__main__":
    # These would come from your vector search
    mock_chunks = [
        {
            "content": "## ECS OOM Remediation\n1. Check task memory metrics in CloudWatch\n2. Identify the container exceeding limits\n3. Update task definition memory reservation +25%\n4. Force new deployment\nEscalate if: task continues OOM after 3 cycles",
            "similarity": 0.91,
            "source": "payments-ecs-runbook.md | Section: Remediation"
        }
    ]
    
    result = runbook_assistant_with_confidence_gate(
        incident_title="PAYMENTS-SVC: ECS tasks restarting repeatedly — Exit code 137",
        service="payments",
        retrieved_chunks=mock_chunks
    )
    
    print(f"Confidence: {result['confidence']}")
    print(f"Sources: {result['top_sources']}")
    print(f"\n{result['answer']}")
```

**LLMOps consideration:** Track `tokens_used` per query and per incident type. RAG queries for complex multi-step incidents can be 3-5x more expensive than simple ones. Build cost dashboards so you can optimize retrieval chunk count and LLM model tier by incident severity.

---

## 10. What I Should Practice Next

**Immediate (this week):**
- Set up pgvector locally with Docker (`docker run -e POSTGRES_PASSWORD=pass ankane/pgvector`) and run the ingestion lab against 5 real runbooks
- Measure retrieval quality with RAGAS — run `pip install ragas` and score your top 10 incident queries

**Next topic to build on this:**
- **Guardrails for AI DevOps Agents** — once you have a RAG runbook assistant, the next engineering challenge is preventing it from taking destructive actions. How do you constrain an agent that can both query runbooks *and* execute kubectl/AWS CLI commands?

**Deeper skills:**
- Learn HNSW index tuning in pgvector — `m`, `ef_construction`, and `ef_search` parameters have a major impact on recall vs. latency tradeoff
- Study cross-encoder reranking (Cohere Rerank API or `cross-encoder/ms-marco-MiniLM-L-6-v2`) — adds 200ms but cuts irrelevant chunks by ~40%
- Build the nightly freshness pipeline using Confluence's `?modified-after` API parameter

**For interviews:**
- Be able to explain the difference between sparse retrieval (BM25), dense retrieval (vector search), and hybrid — and when each fails
- Know how to evaluate RAG: context recall, context precision, answer faithfulness (RAGAS metrics)
- Describe a production incident where stale runbook data would have caused a wrong automated remediation — this is the story that lands with principal-level interviewers

---

