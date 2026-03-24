# Chunking Straitegy

## Why Chunking Matters

RAG systems don’t embed whole documents at query time; they **retrieve smaller pieces** of text that were embedded and stored earlier. Chunking is how you decide what those pieces are. Good chunks:

- **Fit embedding limits** — models cap how many tokens or characters each vector can represent; oversized text gets truncated or distorted.
- **Match how users ask questions** — retrieval works best when a chunk’s meaning matches a single “answerable unit” (e.g., one procedure, one definition, one paragraph of context).
- **Control noise and cost** — smaller, focused chunks can reduce irrelevant context in the prompt; very large chunks can dilute the signal and waste tokens.

**Poor chunking** (splitting mid-sentence, mid-table, or across unrelated topics) can make the right passage **unretrievable** or **wrongly ranked**, even if the embedding model and LLM are strong.

## Why Choosing the Right Strategy Matters

There is no single best chunker for every corpus. **Strategy choice ties directly to retrieval quality** and downstream behavior:

- **Wrong boundaries** — splits that ignore structure (headings, lists, code) break coherence; the model may retrieve fragments that don’t stand alone.
- **Wrong granularity** — chunks that are too small lose context; too large mix multiple topics and hurt precision.
- **Domain fit** — legal, medical, code, and conversational data each need **different splitting rules** (sections, paragraphs, turns, functions).
- **Iteration cost** — changing chunking usually means **re-embedding and re-indexing**; a bad default wastes compute and delays time-to-good-answers.

**Choosing and tuning a strategy** (often with overlap, hierarchy, or structure-aware rules) is one of the most **leverage-heavy** steps in building a RAG pipeline: it affects recall, precision, latency, and cost without changing the base LLM.

## Practical industry example: internal support assistant

**Scenario:** A company ships an **internal “IT + HR policy” chatbot** over hundreds of Confluence pages and PDF handbooks. Employees ask things like “How many days of PTO carry over?” or “What’s the laptop return process when I leave?”

**Why you need chunking:** Each source page can be **long and multi-topic** (travel policy, security, and benefits on one page). If you embed **whole pages**, a user’s question matches a **blended vector**—retrieval often pulls the **wrong section** or ranks the right paragraph low. If you embed **nothing** and stuff full documents into the prompt, you exceed **context limits**, pay more, and the model **confuses** unrelated rules.

**What chunking does here:** Split content into **units that match how people ask questions**—typically **per heading or per short section**, with **overlap** so a rule is not lost on a boundary. In production, teams often use **heading-based** or **Markdown/Confluence-structure-aware** splits, optionally **parent–child** (retrieve a small chunk, send the parent section to the LLM for context).

**Takeaway:** Chunking is not academic; in this use case it is the difference between **“finds the exact policy paragraph”** and **“retrieves a noisy page and hallucinates details.”**

## Most Used Chunking Strategies

- Recursive character text splitting
- Fixed-size chunking
- Token-based chunking
- Sliding-window chunking
- Overlapping chunking
- Sentence-based chunking
- Paragraph-based chunking
- Section-based chunking
- Heading-based chunking
- Markdown-aware chunking
- HTML-aware chunking
- Parent–child chunking
- Hierarchical chunking
- Semantic chunking

### Recursive character text splitting

Splits text using a **priority list of separators** (for example `\n\n`, `\n`, space) so breaks prefer **natural boundaries** before falling back to smaller ones. Common as a default in frameworks like LangChain because it is **cheap, predictable**, and usually beats naive fixed cuts on unstructured prose.

### Fixed-size chunking

Cuts text into pieces of **roughly N characters** (or similar). Simple and fast, but can **split mid-thought** if you do not combine it with better separators or overlap. Often used as a baseline before adding structure.

### Token-based chunking

Like fixed-size, but the unit is **tokens** (what the embedder and LLM actually count). Aligns chunk limits with **embedding model context** and billing; avoids mismatches where “500 characters” is not “500 tokens.”

### Sliding-window chunking

Advances a window along the document in **fixed steps** (for example every 200 tokens). Used when you need **dense coverage** or very regular slices; often paired with **overlap** so boundaries do not land in awkward places.

### Overlapping chunking

Consecutive chunks **share a prefix or suffix** (for example the last 50 tokens repeated). Reduces the risk that an answer sits **on a boundary** and can improve recall; costs **more chunks** and storage.

### Sentence-based chunking

Splits on **sentence boundaries** (rules or NLP). Strong when sentences are meaningful units (support, policies, FAQs); weaker if sentences are very short or very long, or in multilingual text without solid tokenization.

### Paragraph-based chunking

Splits on **paragraph breaks** (for example `\n\n`). Strong for essays, reports, and long-form docs where each paragraph is coherent; weaker if paragraphs are huge (mixed topics) or very small (lists).

### Section-based chunking

Uses **document sections** (for example “Introduction”, “API Reference”) so each chunk stays **topically unified**. Strong for manuals and long PDFs when structure is detectable.

### Heading-based chunking

Uses **headings** (Markdown `#`, HTML `<h1>`, and so on) as anchors: content under a heading forms a chunk or subtree. Common for **wikis, docs sites, and Notion exports** where headings reflect real structure.

### Markdown-aware chunking

Respects **Markdown syntax** (headers, lists, code fences) so you do not split **inside a code block** or break list semantics. Improves quality for dev docs and README-heavy corpora.

### HTML-aware chunking

Uses the **DOM** (tags, blocks) instead of raw characters. Helps for **web pages** where visible structure (sections, article, aside) matters and raw text is noisy.

### Parent–child chunking

Stores **small chunks for retrieval** (precise matching) and links to **larger parent chunks for context** (for example a full section). Common in production: better **precision** at retrieval, richer **context** for the LLM.

### Hierarchical chunking

Chunks at **multiple levels** (document → section → paragraph). Overlaps in spirit with parent–child; used when you need **navigation**, summaries at different scales, or multi-hop retrieval.

### Semantic chunking

Uses **embeddings or similarity** to decide where to split—for example break when adjacent windows are **no longer similar**. Adapts to topic shifts; costs more compute at index time and needs tuning to avoid **too many tiny** or odd splits.

**Typical combinations:** teams often start with **recursive splitting + token limits + overlap**, then add **heading/section/Markdown-aware** rules for structured docs, and introduce **parent–child** or **semantic** chunking when plain splits still miss answers at boundaries or topic changes.

## When to Use Which Chunking Technique

Use this as a **practical map**: match **your data** and **failure mode** to a technique. In production, you often **combine** several (for example token limits + recursive separators + overlap).

### By data source and format

| Situation | Prefer |
|-----------|--------|
| Plain unstructured text (notes, exports with weak structure) | **Recursive character text splitting** + **token-based** limits; add **overlap** if answers straddle cuts |
| Long prose with clear paragraphs (articles, memos) | **Paragraph-based**; fall back to **recursive** + token cap if paragraphs are uneven |
| Policies, FAQs, support articles (one idea per sentence) | **Sentence-based** (watch very short/long sentences) |
| Markdown docs, READMEs, GitHub wiki | **Markdown-aware**; often **heading-based** under each `#` / `##` |
| Crawled web pages, help portals (HTML) | **HTML-aware** (DOM blocks); optionally **section**-like splits from headings |
| PDFs / Word with headings and TOC | **Heading-based** or **section-based** once structure is extracted reliably |
| Mixed-topic long documents with weak headings | **Semantic chunking** (or smaller **token** chunks + **reranking**) |
| Code-heavy docs (must not break fenced code) | **Markdown-aware**; avoid naive fixed cuts inside fences |

### By retrieval problem

| Problem you see | Lean toward |
|-----------------|-------------|
| Right info exists but retrieval misses it **across a chunk boundary** | **Overlap**; smaller **token** chunks; **parent–child** (small for search, big for context) |
| Retrieved chunk is relevant but **missing surrounding context** | **Parent–child** or **hierarchical**; larger **section** chunks; expand with neighbors |
| Chunks mix **multiple topics** (imprecise answers) | **Heading/section** splits; **semantic** boundaries; smaller chunks |
| Chunks are **too small** (fragments, low recall) | Larger units (**paragraph/section**); **overlap**; parent context |
| **Cost / index size** is too high | Reduce **overlap**; fewer levels in **hierarchy**; avoid **semantic** everywhere |

### By pipeline stage (rule of thumb)

- **First baseline:** **Recursive character text splitting** + **token-based** chunk size + modest **overlap**.
- **Structured knowledge base (docs, internal wiki):** **Heading-based** or **Markdown-aware**; add **section** metadata when possible.
- **Web / CMS HTML:** **HTML-aware** chunking after cleaning boilerplate.
- **Still poor retrieval after tuning size/overlap:** **Parent–child** or **semantic** chunking, and improve **evaluation** (queries + labels) before chasing exotic splitters.

## Best chunking for this use case (AIOps Incident Investigator)

Important: the recommendation below is for your **actual production RAG corpus** (logs, diagnostics, changes, runbooks, alerts, inventory), not for chunking the two project writeups.

### Recommended strategy

Use a **source-aware hybrid strategy**:

- **Multi-index design** by source type.
- **Parent-child retrieval** where context expansion is needed.
- **Hybrid search** (dense + BM25) for exact identifiers and semantic intent.
- **Hard metadata filters first** (`env`, `service`, `time range`, `region`, tenant scope).

### Index layout (industry pattern)

- `telemetry_index`: app logs + platform diagnostics (primary incident evidence)
- `change_index`: activity log, deployments, config changes, alerts/incidents
- `knowledge_index`: runbooks, SLOs, KQL/procedures
- `inventory_index` (optional/light): resource disambiguation and scope resolution
- governance schemas/policies: **not user-facing retrieval content**

### Chunking by source type

#### 1) App + platform telemetry logs

- Build canonical, redacted text per event (or per very small event bundle).
- Use trace-aware grouping when possible (`trace_id`, `operation_Id`).
- **Child chunk size:** 180-320 tokens.
- **Parent chunk size:** 800-1200 tokens.
- **Overlap:** 10-15%.
- Keep strong metadata: `timestamp`, `env`, `service`, `trace_id`, `operation_Id`, `resourceId`, severity.

Why: incident answers rely on exact event evidence plus nearby correlation context.

#### 2) Change history (deployments/config/activity/alerts/incidents)

- Prefer **one event = one chunk** (or hourly micro-batches for very high volume).
- Minimal or zero overlap.
- Emphasize exact fields in metadata and lexical index (`releaseId`, `commitSha`, `incidentNumber`).

Why: these are atomic facts; over-chunking can blur causal timelines.

#### 3) Knowledge docs (runbooks, SLOs, KQL, playbooks)

- Heading/section-aware split first.
- Recursive token split only for long sections.
- **Chunk size:** 450-700 tokens.
- **Overlap:** 80-120 tokens.
- Parent-child retrieval enabled.
- Keep section path/version metadata.

Why: preserves procedure semantics and improves grounded remediation answers.

#### 4) Inventory/topology snapshots

- Primarily metadata/filter/orchestration driven.
- Optional light embedding for name disambiguation.
- Do not over-index full inventory text into generation path.

Why: inventory is mostly a routing/filter control plane for retrieval scope.

#### 5) Governance pack (schema + PII policy)

- Use in ETL/chunk-builder/redaction logic.
- Do not embed as normal user Q&A retrieval context.

Why: governance data controls safety/consistency; it is not incident evidence.

### Retrieval flow to pair with this chunking

1. Parse query intent (incident vs procedure vs change question).
2. Apply filters (`env`, service/resource, incident window).
3. Retrieve from appropriate indices (parallel when needed).
4. Merge + deduplicate + rerank.
5. Expand child -> parent where needed.
6. Generate cited answer with evidence/procedure separation.

### Recommended starting parameters

- Telemetry child: 256 tokens, 32 overlap.
- Telemetry parent: 1000 tokens, 120 overlap.
- Knowledge child: 550 tokens, 100 overlap.
- Top-k before rerank: 30-60 combined candidates.
- Final context: 8-20 chunks (measure latency vs quality).

### How to validate (industry-standard)

Create an eval set from real incident-style questions:

1. “What changed before spike?”
2. “Which service first failed?”
3. “Which runbook step applies?”
4. “Any evidence of platform quota/throttle/auth issue?”

Compare at least 3 variants:

- Single universal chunker.
- Source-aware chunking (no parent-child).
- Source-aware + parent-child + hybrid retrieval.

Select by retrieval quality (Recall@k / nDCG), grounded citation accuracy, and latency/cost targets.

## LangChain implementation sketch (source-aware)

```python
"""
High-level ingestion sketch for AIOps Incident Investigator.
The key point is source-specific chunking, not one global splitter.
"""

from langchain_text_splitters import RecursiveCharacterTextSplitter

# Example splitters by corpus type
telemetry_splitter = RecursiveCharacterTextSplitter.from_tiktoken_encoder(
    chunk_size=256,
    chunk_overlap=32,
    separators=["\n\n", "\n", " ", ""],
)

knowledge_splitter = RecursiveCharacterTextSplitter.from_tiktoken_encoder(
    chunk_size=550,
    chunk_overlap=100,
    separators=["\n\n", "\n", " ", ""],
)

def build_chunks(records, source_type):
    if source_type == "telemetry":
        # 1) canonicalize + redact
        # 2) group by trace_id/operation_Id when available
        # 3) split with telemetry_splitter
        ...
    elif source_type == "knowledge":
        # 1) heading/section split first
        # 2) split long sections with knowledge_splitter
        ...
    elif source_type == "changes":
        # one event = one chunk (or hourly micro-batch)
        ...
    elif source_type == "inventory":
        # mostly metadata/filter use; optional light embedding
        ...
    else:
        raise ValueError("Unsupported source type")
```

## Final recommended defaults (quick reference)

| Area | Default |
|------|---------|
| **Overall pattern** | Source-aware chunking + hybrid retrieval (dense + BM25) + parent-child expansion |
| **Hard filters first** | `env`, `service`, `resourceId`, `time range`, `region`, tenant scope |
| **Telemetry (child)** | 256 tokens, 32 overlap |
| **Telemetry (parent)** | 1000 tokens, 120 overlap |
| **Knowledge docs (child)** | 550 tokens, 100 overlap |
| **Knowledge docs (parent)** | 1200-1600 tokens, ~150 overlap |
| **Changes/alerts/incidents** | One event = one chunk (or hourly micro-batch at high volume) |
| **Inventory/topology** | Mostly filter/orchestration metadata; optional light embedding only |
| **Governance pack** | Not in user retrieval path; use only in ETL/chunk-builder/redaction logic |
| **Top-k before rerank** | 30-60 candidates combined across relevant indices |
| **Final context window** | 8-20 chunks (tune for latency/cost/quality) |
| **Citations** | Always include source + timestamp + IDs (`trace_id`, `operation_Id`, incident/release IDs) |

### 30-second interview answer

“For this incident-investigation RAG, I use source-aware chunking instead of one global chunk size: small trace-aware chunks for telemetry, event-level chunks for change data, section-aware chunks for runbooks, and metadata-driven filtering for inventory. Retrieval is hybrid (dense + BM25), filtered by env/service/time first, then child-to-parent expansion for context, with strict citation output.”

### 90-second interview answer

“My chunking design is source-aware because incident investigation combines very different data types.  
For telemetry logs, I keep chunks small and correlation-friendly: around 256 tokens with light overlap, grouped by trace or operation IDs where possible, then expanded to parent context during answer generation. That improves precision while preserving incident narrative.  
For change data—deployments, activity logs, config updates, alerts—I usually keep one event per chunk because these are atomic timeline facts.  
For runbooks and SLO documents, I use heading-based structural splitting and then recursive token splitting for long sections, because procedure text needs semantic continuity.  
Inventory is mainly used for disambiguation and pre-retrieval filters rather than heavy embedding.  
I always apply hard filters first: environment, service/resource scope, and incident time window. Then I run hybrid retrieval—vector plus BM25—because we need semantic matching and exact identifier hits like trace IDs, incident numbers, and release IDs.  
Finally, I rerank, expand child-to-parent context, and require citations in the response.  
I validate the setup with incident-style evals focused on retrieval recall, citation accuracy, and latency/cost tradeoffs.”

### Query flow example (filter -> hybrid retrieve -> rerank -> parent expand)

**User question:**  
“Why did checkout API fail in prod around 14:00 East US, and did any deployment happen before that?”

**Step 1: Parse intent and entities**

- Intent A: incident root-cause evidence (telemetry/platform)
- Intent B: pre-incident change check (deploy/config/activity)
- Entities: `service=checkout-api`, `env=prod`, `region=eastus`, time window around `14:00`

**Step 2: Apply hard metadata filters first**

- Filter all retrieval calls by:
  - `env = prod`
  - `service = checkout-api` (or mapped `resourceId`)
  - `region = eastus`
  - `time in [13:30, 14:30]` (example incident window)

**Step 3: Run hybrid retrieval in parallel (by index)**

- `telemetry_index`: dense + BM25 for terms like `timeout`, `5xx`, `operation_Id`, `trace_id`
- `change_index`: dense + BM25 for terms like `deploy`, `release`, `commit`, `config change`
- (optional) `knowledge_index`: retrieve runbook/SOP sections if user asks “what should we do next?”

**Step 4: Merge, deduplicate, rerank**

- Combine candidates from all indices.
- Deduplicate by stable keys (`source_path`, `event_id`, `trace_id`, `timestamp`).
- Rerank with query-aware scorer so top evidence matches both symptom and timeline.

**Step 5: Parent-child context expansion**

- If a top hit is a child chunk, expand to its parent context:
  - telemetry: expand from event snippet to trace/time window bundle
  - knowledge: expand from sentence/paragraph to section-level procedure

**Step 6: Build grounded prompt**

- Prompt includes:
  - top evidence chunks (with source tags)
  - top change events
  - optional runbook excerpts
  - explicit instruction to separate **observed evidence** vs **hypothesis**

**Step 7: Return cited answer**

- Output structure:
  - What happened (evidence-backed)
  - What changed before incident
  - Most likely contributing factors
  - Confidence + gaps
  - Citations (`timestamp`, `service`, `trace_id/operation_Id`, `releaseId/incidentId`, source path)

**Example concise answer style:**

- “Checkout API 5xx spike aligns with SQL dependency timeouts between 13:57-14:06 (trace-linked evidence).”
- “Deployment `2026.03.18.3` completed ~45 minutes before spike; configuration key `Payments:TimeoutSeconds` changed in same pre-window.”
- “Likely contributing path: dependency saturation/timeouts, not isolated app-only failure.”

## All Chunking Strategies

- Fixed-size chunking
- Fixed-length chunking
- Token-based chunking
- Character-based chunking
- Byte-based chunking
- Sliding-window chunking
- Overlapping chunking
- Stride-based chunking
- Sentence-based chunking
- Sentence-boundary chunking
- Paragraph-based chunking
- Line-based chunking
- Delimiter-based chunking
- Separator-based chunking
- Regex-based chunking
- Recursive character text splitting
- Recursive text splitting
- Hierarchical chunking
- Parent–child chunking
- Small-to-big chunking
- Section-based chunking
- Heading-based chunking
- Document-structure-aware chunking
- Markdown-aware chunking
- HTML-aware chunking
- XML-aware chunking
- PDF layout-aware chunking
- Table-aware chunking
- Code-aware chunking
- Language-aware chunking
- Multilingual chunking
- Semantic chunking
- Embedding-based chunking
- Cluster-based chunking
- Topic-based chunking
- Proposition-based chunking
- Late chunking
- Contextual chunk enrichment
- Metadata-tagged chunking
- Windowed retrieval chunking
- Natural-boundary chunking
- Linguistic chunking
- Clause-based chunking
- Discourse-based chunking
- Time-window chunking (audio/video transcripts)
- Speaker-turn chunking (conversational transcripts)
- Page-based chunking
- Block-based chunking
- Chunk merging / consolidation
- Adaptive chunking
- Dynamic chunk sizing
