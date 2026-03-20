# Transformer Interview Questions - 30-Second Answers

Use this sheet for final revision before interviews. Each answer is intentionally short, practical, and production-oriented.

## 1) What problem did transformers solve?
Transformers solved sequential bottlenecks of RNN/LSTM by enabling parallel training and better long-range dependency handling via self-attention. In production, this improves understanding of distributed context across long tickets or documents.

## 2) Explain self-attention quickly.
Each token creates Query, Key, Value vectors. A token attends to other tokens using `softmax(QK^T / sqrt(d_k))` and aggregates values. This lets the model learn context dynamically instead of fixed windows.

## 3) Why multi-head attention?
Different heads learn different relationships (syntax, entities, negation, temporal links). This improves representation richness and helps on real noisy enterprise text.

## 4) Why positional encoding?
Attention alone does not encode order. Positional encoding injects sequence position so sentence meaning changes with order as expected.

## 5) Encoder-only vs decoder-only vs encoder-decoder?
Encoder-only for understanding tasks, decoder-only for generation/chat, encoder-decoder for source-to-target transformation (translation/summarization). I usually combine encoder retrieval + decoder generation in RAG systems.

## 6) What is masked self-attention?
In decoders, each token can only attend to previous tokens, not future ones. This enforces autoregressive next-token prediction.

## 7) Why are transformers expensive?
Standard attention scales quadratically with sequence length (`O(n^2)`). Cost and latency rise fast for long inputs, so architecture and retrieval choices matter.

## 8) How do you handle long context?
I use chunking, retrieval, reranking, and context compression before generation. I reserve very long context models for use cases that truly need them.

## 9) Why are embeddings critical in RAG?
Embeddings map semantic meaning into vector space, enabling similarity search beyond keyword matching. This improves retrieval recall and answer grounding.

## 10) How do you choose an embedding model?
I benchmark on domain data using Recall@k, nDCG, latency, and cost. I pick the best quality-to-cost model, not the biggest one.

## 11) How do you pick the right transformer model?
Task type + constraints first (latency, cost, compliance), then shortlist, offline eval, canary rollout, and monitoring. Model choice is an engineering decision, not a popularity decision.

## 12) How do you reduce hallucinations?
Ground with retrieval, force citations, add answerability checks, and use post-generation validation for critical fields. Hallucination control is a pipeline problem, not only a prompt problem.

## 13) How do you evaluate production LLM quality?
I track model metrics (relevance/factuality), retrieval metrics (Recall@k/MRR), system metrics (latency/cost/errors), and business KPIs (deflection, handling time, conversion).

## 14) Fine-tuning vs prompt engineering vs RAG?
Start with prompt + RAG for speed and freshness. Fine-tune only when domain behavior still underperforms after retrieval/prompt optimization.

## 15) Common transformer failure modes?
Lost-in-the-middle, retrieval drift, hallucinated joins, and prompt injection. Mitigate with reranking, chunk strategy, guardrails, and robust evaluation sets.

## 16) What is KV cache?
KV cache stores past key/value tensors in decoder inference so previous context is not recomputed each step, reducing generation latency significantly.

## 17) Top backend optimizations for transformer serving?
Dynamic batching, quantization (quality-checked), optimized runtimes, caching, and model routing. Backend optimization often gives larger wins than model swapping.

## 18) Describe a production RAG architecture briefly.
Ingest -> clean -> chunk -> embed -> index -> retrieve -> rerank -> prompt compose -> generate -> guardrails -> observe. Every stage needs monitoring.

## 19) How do you address security/compliance?
PII redaction, tenant isolation, strict logging policy, data retention control, and policy filters. For regulated workloads, deployment boundaries matter as much as model quality.

## 20) What transformer decision are you most proud of?
Replacing a single-model approach with encoder retrieval+rereanking plus decoder generation with citations. It improved quality, reduced hallucinations, and lowered serving cost.

---

## 30-Second Answer Formula
For each question, answer in this pattern:
1. Concept in one line  
2. What you implemented in a project  
3. One measurable impact (quality, latency, or cost)
