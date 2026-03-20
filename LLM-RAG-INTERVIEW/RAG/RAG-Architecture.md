## RAG-Architecture (Interview Tutor | Industry-Style)

You can think of RAG as a “two-system collaboration”:
1) an **Information System** (ingest, index, retrieve)
2) a **Reasoning System** (LLM generation)

The LLM is powerful, but without retrieval it is guessing. With retrieval, it becomes grounded in your knowledge base.

---

## Big-Picture Diagram (mental model)

```text
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

ASCII mental-model diagram (renders everywhere):

```
                ┌──────────────────────┐
User Question  │ Query Processing      │
──────────────>│  - retrieve top-k    │
                │  - (optional) rerank│
                └──────────┬───────────┘
                           │context chunks
                           v
                ┌──────────────────────┐
                │ Prompt + Grounding   │
                │ (instructions +       │
                │  retrieved citations) │
                └──────────┬───────────┘
                           │
                           v
                ┌──────────────────────┐
                │ LLM Generation        │
                └──────────┬───────────┘
                           │
                           v
                ┌──────────────────────┐
                │ Answer + Final checks│
                └──────────────────────┘

Offline ingestion (one-time / batch):
Sources → Document Loader → Document Splitter → Embedding Model → Vector DB / Index
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

In the industry, we often say **“garbage in, garbage out.”**  
If your ingestion process is flawed, the most capable LLM cannot rescue your RAG pipeline reliably. It will retrieve the wrong evidence, or retrieve the right evidence but in the wrong shape (broken tables, missing structure, lost authorship, missing timestamps, missing tenant permissions). That means your downstream retrieval scores may look okay, but your *groundedness* and user trust will collapse.

## What is a Document Loader?
A **Document Loader** is the very first component in a RAG ingestion pipeline. Its primary job is to:
1. connect to a data source (filesystem, database, web, enterprise app),
2. extract the raw content,
3. standardize it into a uniform **Document** format that downstream components can reason about.

In most RAG frameworks, a `Document` object typically contains:
- `page_content`: the extracted text (or extracted segments)
- `metadata`: a dictionary with contextual information (source URL/path, timestamps, author, document type, section/page info, tenant_id, access tags, etc.)

In real production systems (especially enterprise), your data almost never arrives as clean text. You deal with:
- PDFs where reading order matters,
- HTML pages that mix content with navigation/ads,
- email threads where replies must stay linked to the right message,
- structured records where fields must remain filterable.

The loader is the universal adapter that translates all those formats into a consistent representation the rest of your RAG system can trust.

## Different Document Loading Techniques (how to select)
Industry practitioners categorize loaders based on the *nature of the source data* and the *extraction fidelity required*.

### 1) File-based loaders (Local / Cloud)
Use these when your knowledge base is stored as files (local disk, S3, GCS, Azure Blob).
- Selection rule: if the file is plain text/markdown, simple text loaders are usually enough; if the file is PDF/Word and layout matters, choose a layout-aware loader.
- Example use case: indexing internal markdown engineering notes so the assistant can answer “How do we run X?” with accurate section citations.

### 2) Unstructured text / markdown loaders
Use when sources are already readable text with minimal layout complexity.
- Selection rule: prefer the simplest option that preserves content boundaries, and ensure metadata includes stable identifiers (file path, last modified time, section name).
- Example use case: indexing runbook markdown for incident Q&A.

### 3) Layout-aware loaders (PDF / Word)
Use when tables, multi-column reading order, headers/footers, or complex structure changes meaning.
- Selection rule: if tables or reading order affect facts, pick a layout-aware extraction approach (layout parsing / vision-layout extraction).
- Example use case: extracting requirements from compliance PDFs while preserving section headings as metadata so chunk retrieval stays aligned with “answers.”

### 4) Structured data loaders (CSV / JSON / XML / logs-as-records)
Use when you can parse fields reliably and want filterable attributes.
- Selection rule: convert specific fields to text for embeddings, but keep structured fields as metadata for strict filtering.
- Example use case: indexing support tickets from CSV where `error_code` and `severity` must remain filterable for reliable triage.

### 5) Web page / website loaders (scraping)
Use when content lives on documentation sites or internal/external web pages.
- Selection rule: remove boilerplate/navigation; store canonical URL, page title, and last updated time.
- Example use case: ingesting product documentation pages and answering feature questions with citations to the exact page section.

### 6) API / application connectors (enterprise systems)
Use when content is inside Jira/Confluence/GitHub/Slack/Notion/Salesforce and needs incremental sync.
- Selection rule: use official APIs, handle pagination, and maintain stable IDs for updates/deletions.
- Example use case: ingest Jira issues + resolution comments to speed up incident triage recommendations.

### 7) Email / ticket thread loaders
Use when messages are conversational and meaning depends on boundaries.
- Selection rule: parse message boundaries (sender, timestamp, quoted text) so you don’t repeatedly index the entire quoted history.
- Example use case: building a customer email responder grounded in prior resolution patterns from the same thread.

### 8) Database / warehouse loaders (SQL)
Use when the knowledge base is stored in tables.
- Selection rule: project the right columns into text for embeddings; keep primary keys and “effective dates” for traceability and time-aware retrieval.
- Example use case: retrieving policy versions by `effective_date` so the assistant uses the correct policy for a given time window.

### 9) OCR loaders (images / scanned PDFs)
Use when documents are scans and have no embedded text.
- Selection rule: OCR quality matters; keep page/region metadata so retrieved evidence maps back to the right page.
- Example use case: extracting invoice line-items from scanned PDFs for finance QA.

### 10) Audio/video loaders (transcription)
Use when knowledge comes from spoken media (calls, demos, meetings).
- Selection rule: transcribe with timestamps; if possible, keep speaker segments for better attribution.
- Example use case: indexing call transcripts and answering “What did the customer request?” with timestamp references.

### 11) Code repository loaders
Use when the knowledge base is in source code (README, docstrings, configuration, implementation).
- Selection rule: retrieve both file context and relevant code blocks; store language + path metadata for traceability.
- Example use case: answering “How does our RAG pipeline chunk documents?” by retrieving the actual implementation and config.

## Industry-standard approach to loaders
- **Routing:** automatically detect the file/source type and route to the best extraction method (PDF with tables ≠ plain text).
- **Aggressive metadata tagging:** tenant_id, doc type, timestamps, permission tags, and stable IDs.
- **Incremental sync + versioning:** avoid re-indexing everything; support updates and deletions cleanly.

Now, in a typical RAG pipeline, you continue with chunking.

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

Now that we have successfully loaded messy data into a standardized representation, we hit the next major engineering bottleneck: **you cannot simply feed a large document directly into an embedding model or an LLM**.

## What is a Document Splitter?
A **Document Splitter** (also called **chunker**) is the component that takes a big body of text and breaks it down into smaller, discrete, and meaningful segments called **chunks**.

In the industry, we do this for two critical reasons:
1. **Embedding limits**: most embedding models have strict token limits. If the chunk is too large, you either truncate (losing the most relevant part) or fail.
2. **Retrieval precision**: retrieval is not “read the whole book and then answer.” Retrieval is “find the best evidence among many chunks.” If a single chunk contains too much unrelated text, the retriever will bring in noise and the LLM may connect the wrong dots.

So chunking is not a minor preprocessing step; it is the foundation of how accurately the system can locate the “needle” inside the “haystack.”

## Key Chunking Strategies (Industry Patterns)
Choosing the right chunking strategy is an art:
- If chunks are too small, you lose context and important definitions.
- If chunks are too big, you inject noise and reduce ranking precision.

Here are the main strategies used in production, from baseline to advanced:

### 1) Fixed-Size Chunking (Token/Character Splitting)
This splits by a fixed size and uses overlap to protect sentence boundaries.
- Industry selection: use it when your document structure is mostly uniform (e.g., logs, transcripts, repetitive records).
- The “overlap trick”: add overlap (like 10–20% of chunk size) so that important sentences that straddle a boundary still appear in at least one chunk fully.
- Example use case: indexing system logs where each line is similar; fixed chunks are fast and predictable.

### 2) Recursive / Separator-Hierarchy Chunking
Instead of splitting blindly by size, you split by a hierarchy of separators (paragraph → sentence → words).
- Industry selection: use it as a default for general text because it preserves semantic structure.
- Why it works: you avoid cutting in the middle of paragraphs unless forced.
- Example use case: chunking technical documentation where headings and paragraphs drive meaning.

### 3) Structural Chunking (Markdown / HTML / XML / Headings)
If your content has meaningful structure, leverage it.
- Industry selection: use it when your documents have stable structural markers (Markdown headings, HTML sections, XML tags).
- Why it works: chunks align with what humans read as sections.
- Example use case: chunking a knowledge base so “policy section → clause details” remain together.

### 4) Semantic Chunking (Meaning-Aware)
This uses semantic similarity to detect when topics shift.
- Industry selection: use it only when retrieval precision is a top priority and you can afford more ingestion compute.
- Why it works: boundaries follow topic changes, not just punctuation.
- Example use case: chunking long incident postmortems where the narrative topic changes gradually.

### 5) Propositional / Agentic Chunking
This uses a smaller model to extract standalone facts/propositions before indexing.
- Industry selection: use it when you need “fact-level retrieval” for high-stakes workflows.
- Tradeoff: additional ingestion cost and latency, but strong retrieval accuracy.
- Example use case: building a compliance evidence index that returns standalone factual snippets.

In real production pipelines, we often use a **hybrid approach**:
- structural/recursive chunking for cheap coherence,
- hierarchical chunking (parent-child) to rescue context,
- and a reranker at query time to choose the best evidence.

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

In production RAG, the embedding model is the component that decides **the geometry of meaning**.

It is not just “some model that turns text into vectors.” The embedding space is where similarity is measured, where retrieval quality is born, and where many silent failures happen:
- if your query language is phrased differently than your documents, the vectors may not land near each other,
- if the domain uses jargon/acronyms, generic embeddings can collapse distinctions,
- if you rely on the wrong embedding model for the task (general vs retrieval-tuned), the retriever will confidently return the wrong evidence.

## What is an Embedding Model in RAG?
An embedding model converts text into numeric vectors where “meaning is close”.  
Your Vector DB uses these vectors to find relevant chunks at query time.

## How to think about embedding choice (industry intuition)
If chunking decides “what pieces to store,” embeddings decide “whether the pieces can be found for real user questions.”

In my experience building enterprise RAG, embedding selection usually comes down to this practical loop:
1. take a small labeled set of real user queries (and expected relevant chunks),
2. generate embeddings using candidate embedding models,
3. measure retrieval quality (`Recall@k`, `nDCG`, and answer success),
4. pick the best **quality-to-cost** option that meets latency constraints.

## Embedding model types (and when you pick each)
You usually choose based on the shape of your queries and your corpus:
- instruction/query-like embeddings for question-driven retrieval,
- multilingual embeddings for cross-language search,
- domain-adapted embeddings when subtle semantics matter (legal/medical/code).

Once you pick an embedding model, you must keep ingestion and query-time embeddings consistent (same model family + same preprocessing conventions), or the vector index becomes a “wrong map.”

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

## What is a Vector DB (in real production)?
In a RAG system, the Vector DB is where your “search engine” lives. It stores embeddings (vectors) for each chunk and returns the nearest chunks to a query embedding.

But in real industry deployments, the Vector DB is not only about similarity. It is also about:
- **metadata-aware filtering** (tenant, permissions, document type, time ranges),
- **fast latency** under load (ANN indexing + batching),
- **traceability** (ability to map a retrieved vector back to the original document and chunk boundaries),
- **operational reliability** (backups, migrations, and index rebuild strategies).

This is why Vector DB selection is an architecture decision, not a tooling preference.

## Vector DB: the core idea
You store `embedding(chunk)` + `metadata(chunk)`. At query time, you compute `embedding(query)` and retrieve the closest vectors within the allowed metadata scope.

With that mental model, the rest of the section becomes much easier to evaluate.

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
At request-time, RAG is where the “real world” complexity shows up.

The user question you receive is rarely a clean retrieval query. It might be short, ambiguous, full of slang, missing product names, or it might include metadata implicitly (tenant, environment, time period). If your query processing is weak, you retrieve the wrong chunks—even if your ingestion and embeddings are perfect.

So the Query Processing Layer is responsible for turning user intent into:
1) the best retrieval query,
2) the right retrieval scope (permissions + metadata filters),
3) and a context package that the LLM can safely consume.

In other words, you are engineering the “evidence selection” step, not just the prompt.

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
This layer is where users judge your system.

After retrieval, you finally have evidence—chunks with metadata and (ideally) citations. But the LLM still needs strict instructions to behave like an analyst, not like a storyteller.

In the industry, a common failure pattern is:
- retrieval returns some relevant text,
- generation ignores it too freely,
- and the answer becomes fluent but not fully grounded.

So the Generation Layer has two engineering responsibilities:
1) **Prompt assembly with grounding controls** (tell the model exactly how to use the provided context)
2) **Answerability + safety guardrails** (if the evidence is insufficient, the system must refuse/ask clarification instead of hallucinating)

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

