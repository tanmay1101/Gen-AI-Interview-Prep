# Eval set: IT Incident Root Cause Analyzer

`queries.jsonl` contains:

- `query_id`
- `query` (user question)
- `expected_root_cause_id`
- `expected_runbook_id`
- `relevant_evidence` (paths to gold evidence files you should retrieve in your prototype)
- `expected_signature_tokens` (lexical anchors for BM25/hybrid retrieval)

Use this eval set to measure:

- retrieval: Recall@k (did we retrieve the expected runbook/log evidence?)
- reranking quality: MRR/nDCG if you score by rank
- grounding: whether generation uses the retrieved evidence and cites it correctly

