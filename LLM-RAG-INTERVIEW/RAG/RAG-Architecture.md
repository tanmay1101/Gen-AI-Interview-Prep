## RAG-Architecture (Interview Tutor | Industry-Style)

You can think of RAG as a “two-system collaboration”:
1) an **Information System** (ingest, index, retrieve)
2) a **Reasoning System** (LLM generation)

The LLM is powerful, but without retrieval it is guessing. With retrieval, it becomes grounded in your knowledge base.

---

## Big-Picture Diagram (mental model)

```mermaid
flowchart LR
  U[User Question] --> QP[Query Processing Layer]
  QP --> R[Retrieve top-k chunks]
  R --> PR[Prompt Assembly + Grounding Controls]
  PR --> G[LLM Generation]
  G --> A[Answer (with citations) + Final checks]

  subgraph ING[Ingestion Layer (Offline/Batch)]
    S[Sources] --> L[Document Loader]
    L --> C[Document Splitter]
    C --> E[Embedding Model]
    E --> V[Vector DB / Index]
  end
```

---

## 1) Three Layers in RAG

### 1.1 Ingestion Layer
In the industry, we say: **“Garbage in, garbage out.”**  
If ingestion loses structure, drops important metadata, or chunks in the wrong way, retrieval will consistently return the wrong evidence—no prompt can fully fix that.

The ingestion layer runs offline/batch:
**Load → Clean → Split → Embed → Index (with metadata).**

---

#### 1.1.1 Document Loader

**What is a Document Loader?**  
A Document Loader is the first component in a RAG ingestion pipeline. Its job is to:
1. connect to a data source,
2. extract raw content,
3. standardize it into a **uniform Document format**, usually:
   - `page_content`: extracted text (or extracted segments)
   - `metadata`: context like source URL, timestamp, author, tenant, page number, section id, access tags, etc.

**Why it matters:** In production RAG, loaders determine whether you preserve meaning:
- PDFs with tables can become garbage text without layout-aware extraction.
- HTML pages can become navigation + ads without boilerplate removal.
- Emails without thread structure can lose who said what.

##### Different Document Loader Types (with selection guidance + one example use case each)

1. **Local / Cloud File Loaders (File-based sources)**
   - **Use when:** your content is stored as files (local disk, S3, GCS, Azure Blob).
   - **How to select:** if it’s plain text/markdown, use simple text loaders; if it’s PDF/Word and layout matters, use layout-aware loaders.
   - **Example use case:** load `*.md` engineering notes from a repo and index them for “How do we run X?” retrieval.

2. **Unstructured Text / Markdown Loaders**
   - **Use when:** sources are already “mostly readable text”.
   - **How to select:** simplest option; ensure metadata adds file path + last modified time.
   - **Example use case:** index internal “runbook” markdown files to answer incident questions.

3. **PDF / Word Document Loaders (Layout-aware)**
   - **Use when:** tables, multi-column layouts, headers/footers exist.
   - **How to select:** prefer **layout-aware** extraction (layout parsing) when tables and reading order affect meaning.
   - **Example use case:** extract compliance requirements from a PDF and keep each section heading as metadata.

4. **Structured Data Loaders (CSV / JSON / XML / Logs-as-records)**
   - **Use when:** data is structured and you can parse fields reliably.
   - **How to select:** extract “searchable text fields” and keep the rest as metadata (e.g., `error_code`, `severity`, `customer_id`).
   - **Example use case:** index support tickets stored in CSV where `error_code` must remain filterable.

5. **Web Page / Website Loaders (Scraping)**
   - **Use when:** documentation and public/private websites are the source.
   - **How to select:** use boilerplate removal and keep canonical URL + title; handle dynamic rendering when needed.
   - **Example use case:** ingest product docs pages and answer “What does feature X do?” with citations.

6. **API / Application Connectors (Enterprise loaders)**
   - **Use when:** your documents live in systems like Jira, Confluence, GitHub, Slack, Notion, Salesforce.
   - **How to select:** prefer official APIs, incremental sync, and robust pagination; map entities to metadata (project key, labels).
   - **Example use case:** ingest Jira issues + resolution comments to help triage new incidents.

7. **Email / Ticket Thread Loaders**
   - **Use when:** message threads contain context and turn-by-turn updates.
   - **How to select:** parse message boundaries (sender, timestamp, quoted text) so you don’t index entire quoted history repeatedly.
   - **Example use case:** build a “customer email responder” assistant grounded in prior thread decisions.

8. **Database / Data Warehouse Loaders (SQL)**
   - **Use when:** content is in tables (Postgres, MySQL, BigQuery, Snowflake).
   - **How to select:** use SQL projection to generate document text from specific columns; keep primary keys for traceability.
   - **Example use case:** retrieve policy versions from a DB and index by `effective_date` for time-aware answers.

9. **Multimodal Loaders: OCR (Images / Scanned PDFs)**
   - **Use when:** documents are scans (no embedded text).
   - **How to select:** OCR quality and layout extraction matter; store page/region metadata for accurate grounding.
   - **Example use case:** ingest scanned invoices and extract line-items for finance QA.

10. **Multimodal Loaders: Audio / Video (Transcription)**
   - **Use when:** meetings, support calls, or demos are spoken.
   - **How to select:** use transcription with timestamps; optionally segment by speaker if supported.
   - **Example use case:** index call transcripts and answer “What did the customer request?” with timestamps.

11. **Code Repository Loaders (Repos + files)**
   - **Use when:** docs are in source code (README, docstrings) or code is the knowledge base.
   - **How to select:** extract both: file descriptions + relevant code blocks; add language + path metadata.
   - **Example use case:** answer “How does our RAG pipeline chunk documents?” by retrieving the actual implementation.

##### Industry-standard approach to loaders
- **Routing:** detect file type / source type and route to the best loader (PDF with tables != plain text loader).
- **Metadata tagging aggressively:** tenant, doc type, timestamps, permission tags, and stable IDs.
- **Incremental sync + versioning:** avoid re-indexing everything; support doc updates and deletions cleanly.

---

#### 1.1.2 Document Splitter (Chunking)

You cannot feed a 50-page manual or massive logs directly into embeddings/LLMs.  
The Document Splitter breaks large text into smaller segments (**chunks**) optimized for retrieval.

**In production, chunking is foundational** for two reasons:
1) **Embedding limits:** embedding models have max tokens.
2) **Retrieval precision:** smaller, meaningful chunks reduce “needle-in-haystack” failure.

##### Chunking Strategy Types (with what to use + where it fits)

1. **Fixed-size chunking (token/character)**
   - **How it works:** cut by a fixed limit (e.g., 500 tokens) with overlap.
   - **Pros:** fast, predictable compute/memory.
   - **Cons:** can cut sentences/meaning; can hurt table/text structure.
   - **Example use case:** indexing uniform “one-record-per-line” system logs.

2. **Recursive chunking (separator hierarchy)**
   - **How it works:** try separators in order (e.g., `\n\n`, then `\n`, then space) to preserve paragraphs/sentences.
   - **Pros:** respects natural boundaries better than fixed cuts.
   - **Cons:** still may split long tables or multi-column docs if loader output is messy.
   - **Example use case:** chunking long technical docs where paragraphs matter.

3. **Token-based chunking (model-aware limits)**
   - **How it works:** split to token counts using the tokenizer that matches your embedding model.
   - **Pros:** reduces mismatch between “chunk size” and “embedding input size”.
   - **Cons:** slightly more compute than pure char splitting.
   - **Example use case:** strict context budgets for cost-sensitive production.

4. **Structural chunking (Markdown / HTML / XML / headings)**
   - **How it works:** split along structure (headers, sections, tags).
   - **Pros:** keeps coherent sections intact; often improves retrieval interpretability.
   - **Cons:** depends on loader preserving structure correctly.
   - **Example use case:** chunking a knowledge base where questions map to headings.

5. **Code/AST chunking**
   - **How it works:** parse code and chunk at function/class boundaries (AST-aware).
   - **Pros:** retrieval returns executable units instead of broken fragments.
   - **Cons:** more complex pipeline and language-specific parsing.
   - **Example use case:** indexing a Python service for “Where is retry logic implemented?”

6. **Table-aware chunking**
   - **How it works:** keep rows/columns or split by table sections; preserve header associations.
   - **Pros:** critical for compliance and numeric QA.
   - **Cons:** requires a good PDF/table extraction loader.
   - **Example use case:** extracting product pricing tables for “What is the cost for plan X?”

7. **Hierarchical chunking (parent-child)**
   - **How it works:** create multiple granularities:
     - small chunks for precision
     - larger parent chunks for context rescue
   - **Pros:** solves “too small loses context” and “too big loses precision”.
   - **Cons:** more storage + more retrieval complexity.
   - **Example use case:** long policy docs (short clause answer + parent section context).

8. **Semantic chunking**
   - **How it works:** compute embeddings/similarity between adjacent sentences; cut where meaning shifts.
   - **Pros:** very high-quality topic boundaries.
   - **Cons:** expensive ingestion: you embed to decide the split.
   - **Example use case:** chunking long unstructured narratives like incident postmortems.

9. **Agentic / Propositional chunking**
   - **How it works:** use a smaller LLM to transform text into standalone factual propositions or key facts before indexing.
   - **Pros:** best retrieval “facts”, minimal ambiguity.
   - **Cons:** cost/latency during ingestion; best for high-value corpora.
   - **Example use case:** building a “facts-only” compliance retrieval set.

##### Production standard (what I do)
Most teams use **hybrid chunking**:
- structural chunking (cheap, keeps coherence)
- hierarchical chunking (context rescue)
- plus reranking at retrieval time to choose the best evidence.

---

#### 1.1.3 Embedding Model

**What is an Embedding Model in RAG?**  
An embedding model converts text into numeric vectors where “meaning is close”.  
Your Vector DB uses these vectors to find relevant chunks at query time.

##### Why embedding choice is critical (industry view)
Even perfect chunking fails if embeddings don’t match your task.
Common reasons embeddings fail:
- query language style differs from document style
- domain jargon / acronyms create semantic gaps
- multilingual corpora need multilingual embeddings
- “query intent” differs from “document wording”

##### Embedding model types (and how to choose)

1. **General-purpose sentence embeddings**
   - **Use when:** your domain is broad or data is well-formed.
   - **How to choose:** test with your own evaluation set.
   - **Example use case:** internal wiki retrieval across mixed topics.

2. **Instruction / retrieval-tuned embeddings**
   - **Use when:** queries are phrased like questions/instructions and you need strong retrieval alignment.
   - **How to choose:** prefer embeddings that support “query/document” roles (or use prompting/prefix strategies).
   - **Example use case:** “Find steps to reset password” returns better when embeddings are query-aware.

3. **Multilingual embeddings**
   - **Use when:** users ask in multiple languages or docs are multilingual.
   - **How to choose:** verify cross-lingual retrieval (query in Language A returns docs in Language B).
   - **Example use case:** multi-lingual product support search.

4. **Domain-adapted / specialized embeddings**
   - **Use when:** legal/medical/code corpora need sharper semantics.
   - **How to choose:** evaluate with domain benchmarks or internal labeled sets.
   - **Example use case:** contract clause retrieval where exact clause semantics matter.

##### Embedding selection checklist (industry standard)
- **Measure retrieval quality**: Recall@k / nDCG on your tasks.
- **Check cost & throughput**: batch size, embedding latency, caching strategy.
- **Confirm embedding dimensionality & index compatibility** with your Vector DB.
- **Validate failure cases**: negation, acronyms, very short queries, and “lost in the middle” contexts.

---

#### 1.1.4 Vector DB

**What is a Vector DB?**  
A Vector DB stores embeddings and supports fast similarity search (nearest neighbors).  
It also stores metadata so you can filter by tenant, document type, permissions, or timestamps.

##### Vector DB selection types

1. **Managed cloud vector databases**
   - **Pros:** easier ops, scaling, backups.
   - **Cons:** recurring cost, data residency concerns.
   - **Example use case:** a startup shipping quickly and needing reliable scaling.

2. **Self-hosted vector databases**
   - **Pros:** control, potentially lower long-term cost.
   - **Cons:** requires DevOps ownership and tuning.
   - **Example use case:** enterprise environments with strict security.

3. **Local / embedded vector indexes (FAISS-like)**
   - **Pros:** great for small scale, prototypes, on-prem single-node apps.
   - **Cons:** scaling complexity when data grows.
   - **Example use case:** personal RAG assistant over a small doc set.

4. **Hybrid DB approach (Postgres + pgvector)**
   - **Pros:** combine relational metadata filtering + vector search.
   - **Cons:** performance depends on configuration/index.
   - **Example use case:** teams already invested in Postgres and need strong filtering.

##### Indexing and retrieval mechanics (what interviewers ask)
- Most vector DBs use **approximate nearest neighbor (ANN)** indexes like:
  - HNSW (often great quality/latency)
  - IVF (cluster-based indexing)
  - PQ (compression for memory)
- Similarity metric: **cosine similarity** or **dot product** (depends on embedding normalization).

##### Metadata filtering (the hidden production requirement)
In real products, you rarely search globally. You filter:
- tenant_id / org_id
- permission level
- document type (policy vs tickets)
- time constraints (“use policy version effective at date X”)

This filtering prevents both poor quality and security leaks.

---

### 1.2 Query Processing Layer
This layer runs at request-time. It converts a user question into retrieved evidence and a context package the LLM can use safely.

Think of it as: **“turn user intent into the right chunks, in the right order, with the right permissions.”**

##### 1.2.1 Query understanding & normalization
Common steps:
1. **Query rewriting** (LLM or rule-based)
   - Converts short/ambiguous queries into retrieval-friendly queries.
   - Handles synonyms and domain language.
2. **Metadata extraction**
   - Extract tenant, product line, doc type, timeframe, severity, etc.
3. **Query expansion**
   - Multi-query (generate multiple reformulations)
   - HyDE-style (generate hypothetical answer to improve recall)

**Example use case:**  
User asks: “login failing after billing change”  
Query processor rewrites to: “Cannot login after billing update - access problem - steps and troubleshooting”  
It may also set metadata filter `doc_type=ticket_resolution`.

##### 1.2.2 Embedding the query
The query embedding must be compatible with the embedding model used during ingestion.
For best results:
- normalize query formatting the same way as docs
- apply query/document role prefixes if your embedding approach uses them

##### 1.2.3 Retrieval (top-k candidates)
Two dominant strategies:
1. **Vector retrieval**
   - similarity search over embeddings
2. **Hybrid retrieval (vector + keyword/BM25)**
   - helps when exact identifiers matter (error codes, SKUs, policy clause numbers)

Industry pattern: start vector-first, then add hybrid when you see:
- keyword miss failures
- short queries failing
- “exact match” constraints.

##### 1.2.4 Reranking (optional but common in production)
Reranking re-orders retrieved candidates using a stronger scoring model, often:
- cross-encoder reranker
- LLM-based judge (slower, higher nuance)

Why it matters: initial retrieval optimizes recall, reranking optimizes precision.

##### 1.2.5 Context window building (context packing)
You must decide:
- which chunks to include
- how many chunks to include (token budget)
- ordering (by relevance, by recency/version, by document hierarchy)
- deduplication (avoid repeating identical chunks)

Production practice:
- prioritize top reranked chunks
- include parent context if hierarchical chunking is used
- keep citations traceable to specific chunks.

---

### 1.3 Generation Layer
This layer runs after you have evidence. It turns the question + evidence into a final response.

##### 1.3.1 Prompt assembly with grounding controls
Prompt components I typically include:
- **system instructions**: answer style + “use only provided context”
- **context block**: retrieved chunks with references/citations
- **user question**
- **output constraints**: bullet points, JSON schema, or “if not found, say I don’t know”

In production, I explicitly enforce answerability:
- If evidence is insufficient, the model should refuse or ask clarification.

##### 1.3.2 Answer generation
LLM generates token-by-token using the context.
For enterprise assistants, you often want:
- structured response (steps, summary, risks)
- citations mapped to each major claim

##### 1.3.3 Post-processing and guardrails
Common production steps:
- citation consistency check
- PII redaction / safety filters
- formatting enforcement (valid JSON, required fields)
- “prompt injection resistance” (treat retrieved content as data, not instructions)

---

## 2) End-to-End Example (Industry Use Case)

Support ticket copilot:
1. **Ingestion**
   - loaders: policy PDFs (layout-aware), ticket history (API), PDFs/emails (OCR when needed)
   - splitters: structural + hierarchical chunking (keep sections and parent clauses)
   - embeddings: retrieval-tuned embeddings for query-document alignment
   - vector DB: store chunk embeddings + metadata (`tenant_id`, `product`, `effective_date`)
2. **Query processing**
   - normalize query + extract metadata filters
   - retrieve top-k chunks
   - rerank with cross-encoder
   - pack best context under token budget
3. **Generation**
   - generate answer with citations
   - abstain when context doesn’t support claims
4. **Evaluation**
   - monitor retrieval hit rate + groundedness + user satisfaction

---

## 3) Evaluation & Observability (what interviewers expect)

### 3.1 Offline evaluation
- Retrieval:
  - `Recall@k`, `MRR`, `nDCG`
- Generation:
  - groundedness / citation correctness
  - answerability (did the answer actually use the provided context?)

Groundedness is commonly defined as how well generated content is supported by the retrieved documents (often treated as faithfulness). For example, see:
- deepset groundedness evaluation guidance: https://deepset.ai/blog/rag-llm-evaluation-groundedness
- and their groundedness observability docs: https://docs.cloud.deepset.ai/docs/use-groundedness-observability

### 3.2 Online evaluation
- latency and cost (P95/P99)
- retrieval hit rate (did retrieved context contain the answer?)
- safety/security events (prompt injection attempts, PII leaks)

---

## 4) Practical Production Checklist (quick answers)
- Use best-fit document loaders (layout-aware when needed).
- Use chunking that preserves structure; add hierarchical chunking for context rescue.
- Use embeddings validated on your retrieval task.
- Use Vector DB with correct index + metadata filters.
- Add query rewrite + hybrid retrieval when short/keyword-heavy queries fail.
- Add reranking for precision when it impacts user KPIs.
- Force grounding and answerability in generation prompts.

