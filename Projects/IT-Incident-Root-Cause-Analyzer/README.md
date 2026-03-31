# IT Incident Root Cause Analyzer (Prototype Dataset)

This folder contains a small but realistic, resume-ready dataset for an **“IT Incident Root Cause Analyzer”** RAG prototype.

The goal is to showcase capability on **messy real-world formats** and **rigorous retrieval evaluation**, not to be a perfect production log platform.

## What’s included

1. **Tickets (text)**: user/service request narratives in plain Markdown.
2. **Logs (messy text / JSONL + stack traces)**: mixed structure, noisy multi-service events.
3. **Runbooks (knowledge base)**: structured Markdown procedures keyed by `root_cause_id`.
4. **Images (lightweight placeholders)**: simple SVG “screenshots/diagrams” for a multi-modal vibe (text-only retrieval can still use OCR/caption text later).
5. **Eval set**: natural language queries + gold `root_cause_id` so you can measure retrieval quality (Recall@k / MRR / nDCG) and grounding accuracy.

## File layout

- `dataset/tickets/*.md`
- `dataset/logs/*.(txt|jsonl)`
- `dataset/knowledge/runbooks/*.md`
- `dataset/images/*.svg`
- `eval/queries.jsonl`

## How you should use this dataset in your RAG prototype

- **Chunking**: treat runbooks as heading/section-aware; treat logs as event/chunk-per-block.
- **Indexing**: store each evidence type in its own “source type” collection/index:
  - `tickets`, `logs`, `runbooks`
- **Retrieval**:
  - hybrid retrieval (dense + BM25)
  - rerank top candidates
  - answer with evidence and citations
- **Evaluation**:
  - build a gold mapping from queries to root-cause runbooks
  - measure ranking quality and grounding (no-evidence fallback).

## Root-cause IDs

Runbooks use stable root-cause identifiers like:

- `RC_DB_TIMEOUT`
- `RC_DNS_FAILURE`
- `RC_TLS_CERT_EXPIRED`
- `RC_RATE_LIMIT`
- `RC_DISK_FULL`
- `RC_AUTH_INVALID_TOKEN`

