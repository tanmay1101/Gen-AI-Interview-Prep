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
