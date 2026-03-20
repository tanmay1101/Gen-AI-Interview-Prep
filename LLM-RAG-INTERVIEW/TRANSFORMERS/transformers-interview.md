# Architecture of Transformers - Interview Notes (Senior GenAI Engineer POV)

## 1) How I Explain Transformer Architecture in Interviews

When I explain transformer architecture, I start from the problem it solved: RNN/LSTM pipelines struggled with long-range dependencies and poor parallelism. Transformers replaced recurrence with self-attention, which lets every token directly attend to every other token in the same sequence.

Core blocks I cover:

1. Input Embedding + Positional Encoding  
   - Token embeddings convert discrete tokens into dense vectors.  
   - Positional encoding (learned or sinusoidal) injects order information because attention alone is permutation-invariant.

2. Multi-Head Self-Attention  
   - For each token, model creates Query, Key, Value vectors.  
   - Attention scores are computed using scaled dot-product attention:
     Attention(Q, K, V) = softmax(QK^T / sqrt(d_k))V  
   - Multiple heads learn different relationships in parallel (syntax, semantic dependencies, entity links, etc.).

3. Feed-Forward Network (FFN)  
   - Position-wise MLP transforms each token representation independently after attention mixing.

4. Residual Connections + Layer Normalization  
   - Stabilize training and improve gradient flow in deep stacks.

5. Encoder/Decoder Stacking  
   - Encoders build contextual representations.  
   - Decoders generate outputs autoregressively and use masked self-attention.

### Use case on real data — how each architecture step acts on a ticket (visual mental model)

**Real input sentence (support ticket):**  
`"Customer cannot reset login after billing update."`

**Assume tokenization (illustrative subwords):**  
`[CLS]`, `Customer`, `cannot`, `reset`, `login`, `after`, `billing`, `update`, `[SEP]`  

(Your tokenizer may split differently; the idea is the same: each piece becomes one row in the sequence.)

| Step | What happens on this real data | What to say in interview |
|------|-------------------------------|---------------------------|
| **1. Embeddings** | Each token becomes a vector (e.g. 768-dim). Before the model runs, `"billing"` and `"login"` are just unrelated lookup vectors. | "Discrete IDs → dense vectors the network can mix." |
| **2. Positional encoding** | Without this, shuffling tokens would give the same math; adding position makes `"after billing update"` ordered vs `"billing after update"`. | "Attention does not know order by itself; position encodings fix that." |
| **3. Self-attention (one layer)** | Token `"cannot"` builds its **Q**; every token supplies **K** and **V**. High attention weights might link **`cannot` → `reset`, `login`** (negation + action) and also **`after` → `billing`, `update`** (temporal cause chain). Different **heads** can specialize: one head for negation scope, another for "billing" as domain hint. | "Each token asks: which other tokens should update my meaning? That is the attention map." |
| **4. FFN (per token)** | After mixing, each position gets a small MLP. The vector at `"login"` is refined into "authentication issue" style features using information already blended from neighbors. | "Attention mixes information across positions; FFN processes each position independently." |
| **5. Residual + LayerNorm** | `x_out = LayerNorm(x + sublayer(x))`. Prevents vanishing signal when the stack is deep. | "Residuals let the model pass raw signal forward; norm keeps activations stable." |
| **6. Stack of layers** | Layer 1 might capture local phrases ("cannot reset"); deeper layers combine into **intent**: e.g. "account access after billing change" → label **Access** or **Billing** cross-over. Final **`[CLS]`** vector feeds your classifier head. | "Early layers = local patterns; later layers = global task-level meaning." |

**One-sentence picture for whiteboard / interview:**  
"For this ticket, embeddings turn words into points in space; attention lets `cannot` pull meaning from `reset` and `login` in one hop; FFN sharpens each token's role; stacking builds the `[CLS]` vector we use to predict triage intent."

Interview line I use:  
"A transformer layer alternates between token-to-token communication (attention) and token-wise reasoning (FFN), with residual + normalization keeping optimization stable."

---

## 2) Working of Transformer (Practical, Step-by-Step)

### A) Encoder-only flow (example: BERT-style classification)
1. Tokenize text and create embeddings.
2. Add positional signals.
3. Pass through N encoder blocks:
   - self-attention -> add & norm
   - FFN -> add & norm
4. Use final contextual embeddings for downstream task:
   - CLS token for classification, or token-level outputs for NER.
Use case to explain in interview:  
"In a support-ticket triage system, I used an encoder model to classify tickets into Billing, Access, and Incident classes. Because encoder attention reads both left and right context, it handled phrases like 'not able to login after password reset' better than keyword rules."

### B) Decoder-only flow (example: GPT-style generation)
1. Input prompt tokens.
2. Masked self-attention ensures token t cannot see future tokens.
3. Model predicts next token probability distribution.
4. Decode one token at a time (greedy, beam, top-k, nucleus, etc.).
Use case to explain in interview:  
"For an internal DevOps assistant, I used a decoder-only model to generate step-by-step runbooks from incident context. Autoregressive generation made it ideal for producing coherent multi-step instructions."

### C) Encoder-decoder flow (example: T5/BART translation/summarization)
1. Encoder builds source representation.
2. Decoder uses:
   - masked self-attention on generated target tokens
   - cross-attention over encoder outputs
3. Generate target sequence autoregressively.
Use case to explain in interview:  
"In compliance reporting, I used an encoder-decoder model to convert long audit notes into executive summaries. Cross-attention helped the decoder stay grounded in source content while generating concise output."

---

## 3) How Transformer Architecture Helped in My Projects

### Project 1: Enterprise Support Copilot (RAG Q&A over policy + ticket history)
- Problem: Old keyword/BM25 retrieval missed semantic variants and gave inconsistent answers.
- Transformer impact:
  - Used encoder transformers for semantic embeddings to improve retrieval recall.
  - Used decoder LLM for grounded answer generation with citations.
  - Better context understanding reduced hallucination and improved answer relevance.
- Business outcome:
  - Lower average handling time for support engineers.
  - Improved first-response quality and consistency.

### Project 2: Contract Intelligence Pipeline
- Problem: Rule-based extraction failed on wording variability across vendors.
- Transformer impact:
  - Fine-tuned encoder model for clause classification and risk tagging.
  - Added long-context chunking + cross-encoder re-ranking for precision.
- Business outcome:
  - Faster legal review cycles.
  - Better recall on non-standard clause language.

### Project 3: Multi-lingual Product Search
- Problem: Lexical search performed poorly across mixed-language catalogs.
- Transformer impact:
  - Bi-encoder multilingual embeddings for query-product matching.
  - Cross-encoder re-ranker for high-precision top results.
- Business outcome:
  - Better click-through on first-page results.
  - Improved relevance in multilingual traffic.

---

## 4) Types of Transformers and Why They Are Needed

### 1. Encoder-only Transformers (BERT, RoBERTa, DeBERTa)
- Need: Strong bidirectional context understanding.
- Best for: Classification, NER, semantic search embeddings, reranking.
- Why: They read full context at once; excellent for understanding tasks.

### 2. Decoder-only Transformers (GPT/Llama/Mistral family)
- Need: Natural language generation and instruction following.
- Best for: Chatbots, code generation, drafting, agentic reasoning pipelines.
- Why: Autoregressive objective aligns with text generation.

### 3. Encoder-Decoder Transformers (T5, BART, FLAN-T5)
- Need: Input-to-output transformation tasks.
- Best for: Summarization, translation, rewriting, structured generation.
- Why: Explicit source-target separation with cross-attention.

### 4. Long-context Transformers (Longformer, BigBird, modern long-context LLMs)
- Need: Handle long documents without aggressive truncation.
- Best for: Legal, compliance, research, log analysis.
- Why: Efficient attention patterns or optimized context windows.

### 5. Vision Transformers (ViT) and Multimodal Transformers (CLIP, LLaVA-like stacks)
- Need: Image + text understanding/generation.
- Best for: Document intelligence, visual QA, e-commerce media pipelines.
- Why: Shared representation space across modalities.

---

## 5) How I Select the Best Transformer (Industry Standard Decision Framework)

In interviews, I emphasize this is not "largest model wins", it is "fit-for-purpose under constraints".

### Step 1: Define task shape
- Understanding task (classify/retrieve) -> encoder-first.
- Generation task (answer/write/compose) -> decoder-first.
- Transformation task (source -> target) -> encoder-decoder.

### Step 2: Define constraints
- Latency SLA (P95/P99 targets)
- Throughput and concurrency
- Context length
- Data privacy/compliance (self-hosted vs API)
- Cost per 1K requests / per token

### Step 3: Build candidate shortlist
- Baseline small/medium/large models.
- Include domain-specific variants (biomedical, legal, code).

### Step 4: Offline evaluation
- Task metrics:
  - Classification: F1, AUROC
  - Retrieval: Recall@k, nDCG, MRR
  - Generation: groundedness, factuality, task success, human eval rubrics
- Robustness tests:
  - prompt variation, adversarial phrasing, OOD samples

### Step 5: Online validation
- A/B in shadow or canary mode.
- Track quality + latency + cost + safety incidents.

### Step 6: Operational readiness
- Observability (traces, token usage, retrieval hit quality)
- Guardrails (PII redaction, policy checks, output filters)
- Rollback strategy and versioned prompts/models.

Use case to explain selection in interview:  
"For a customer-support copilot, I benchmarked an encoder for retrieval quality and a decoder for answer generation. We selected the final stack only after it met Recall@5, groundedness, P95 latency, and cost-per-ticket targets in canary traffic."

Interview line I use:  
"I pick models by quality-to-cost ratio under production constraints, not leaderboard scores."

---

## 6) Best-Fit Scenario for Each Transformer Type

### Encoder-only best-fit scenario
Use case: Financial ticket triage and intent classification  
Why best fit: Needs high semantic understanding, low latency, and deterministic labels, not long-form generation.

### Decoder-only best-fit scenario
Use case: Internal engineering assistant that drafts runbooks and answers "how-to" questions  
Why best fit: Requires fluent multi-turn generation, instruction following, and tool-calling compatibility.

### Encoder-decoder best-fit scenario
Use case: Compliance report summarization from long policy documents into structured executive summaries  
Why best fit: Clear source-to-target transformation with controlled output style.

### Long-context transformer best-fit scenario
Use case: Legal due-diligence assistant over large contract bundles and appendices  
Why best fit: Must reason across long dependencies without losing cross-document references.

### Multimodal transformer best-fit scenario
Use case: Insurance claims processing from uploaded images + text claim descriptions  
Why best fit: Needs joint visual-text understanding for damage assessment and claim validation.

---

## 7) Interview-Ready Closing Summary

If asked "Why transformers?", my concise answer:

"Transformers became the industry default because self-attention gives superior context modeling with parallel training. In production, I choose encoder, decoder, or encoder-decoder variants based on task type, latency, cost, and compliance constraints. The best model is the one that meets business KPIs reliably in real traffic, not just benchmark scores."
