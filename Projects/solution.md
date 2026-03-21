# Solution: Azure Log RAG — Data Landing in ADLS

**Context:** Application on Azure; monitoring via Azure Monitor / Log Analytics / Application Insights; target landing zone **Azure Data Lake Storage Gen2 (ADLS)** for downstream RAG ingestion.

**Design posture:** Treat ADLS as the **curated, auditable lakehouse landing** for analytics and RAG preprocessing—not as a blind dump of every raw log line forever. Decisions below follow common enterprise patterns: **partitioned data, schema contracts, PII boundaries, and pipeline observability**.

---

## 1. List of all data required

Data is grouped by **purpose** so retrieval and grounding stay explainable (aligned with a three-layer RAG view: evidence + metadata + correlation).

### 1.1 Application & workload logs (primary evidence)

| Data | Source (typical Azure) | Why required |
|------|-------------------------|--------------|
| Request / dependency / exception / trace tables | Application Insights, Log Analytics (App-dependent tables) | Core “what the software did” narrative for incidents |
| Container / stdout logs | AKS, Container Apps, App Service | Gaps where APM is thin |
| Function / background job logs | Azure Functions, Logic Apps, Durable Functions | Async failures and retries |

**Key attributes to retain:** `TimeGenerated`, severity, message, `cloud_RoleName` / service id, `trace_id` / `span_id` when present, environment, region, subscription/resource id, selective `customDimensions`.

### 1.2 Platform & resource diagnostic logs

| Data | Source | Why required |
|------|--------|--------------|
| Data plane diagnostics | Diagnostic settings → LA (SQL, Cosmos, Storage, Key Vault, APIM, Redis, Event Hubs, etc.) | Explains throttling, auth, quota, dependency failures not visible in app logs alone |

### 1.3 Control plane & change history

| Data | Source | Why required |
|------|--------|--------------|
| Activity Log (write/delete/action) | Subscription / management group | Answers “what changed?”—critical for incident correlation |
| Deployment / release records | Azure DevOps, GitHub Actions, ARM deployment history | Links errors to releases and rollbacks |
| Config change signals | App Configuration, Key Vault **metadata** (not secret values), policy assignments | Config drift and misconfiguration hypotheses |

### 1.4 Security & network (where in scope)

| Data | Source | Why required |
|------|--------|--------------|
| Firewall / WAF / NSG-related signals (as approved) | Azure Firewall, App Gateway WAF, etc. | Connectivity and abuse patterns |

### 1.5 Inventory & topology (for scoped retrieval)

| Data | Source | Why required |
|------|--------|--------------|
| Resource inventory snapshots | Azure Resource Graph export | Filters RAG by env, team, region, service; avoids cross-tenant leakage in queries |

### 1.6 Operational artifacts (optional but strong)

| Data | Source | Why required |
|------|--------|--------------|
| Alert rules + fired alert instances | Azure Monitor | Aligns user questions with how incidents were declared |
| Incident / ticket metadata | ITSM integration | Bridges chat questions to official incident IDs |

### 1.7 Reference knowledge (separate corpus, same lake)

| Data | Source | Why required |
|------|--------|--------------|
| Runbooks, SLOs, on-call playbooks, KQL snippets | Git/wiki exports | Grounding in **procedures**, not only raw logs |

### 1.8 Governance pack (non-negotiable for enterprise)

| Data | Purpose |
|------|---------|
| **Schema / field dictionary** (JSON) | Stable parsing, embedding templates, versioning |
| **PII & redaction policy** (manifest) | What fields are dropped, hashed, or tokenized before RAG indexing |

---

## 2. Approach: pipeline to bring data into ADLS

**Industry baseline:** Use **multiple paths** by freshness, volume, and source—one ETL pattern rarely fits all. Prefer **idempotent writes**, **partition keys**, and **monitoring** on each path.

### 2.1 Path A — Continuous / near-real-time (Log Analytics workspace)

**Pattern:** **Log Analytics workspace data export** (where supported) to **Storage Account** used as **ADLS Gen2** (hierarchical namespace enabled), **or** export to **Event Hubs** → **Azure Stream Analytics** / **Azure Functions** / **Databricks** → writes **JSON Lines** or **Parquet** into ADLS with your **canonical folder layout**.

**Why this path:** Operational logs already centralize in LA; export or stream avoids bespoke per-service scrapers.

**Validation step:** Confirm current Microsoft support matrix for **destination type** (Blob vs Gen2-specific constraints) in [Log Analytics data export](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/logs-data-export) and adjust (e.g., Blob account behind Gen2, or hub-and-spoke writer).

### 2.2 Path B — Resource diagnostic tables already in LA

**Pattern:** Same as Path A for selected tables (e.g., `AzureDiagnostics`, resource-specific tables). Use **table-level export rules** or **scheduled queries** that materialize **narrow** datasets (reduce cost and noise).

**Why:** Platform logs are high cardinality; **filter at export or in a downstream Silver job** so ADLS does not become an unbounded copy of everything.

### 2.3 Path C — Activity Log & administrative events

**Pattern:** Diagnostic settings on Activity Log to **Storage** or **Event Hubs**, or **scheduled ADF/LA query** → ADLS **daily/hourly** partitions.

**Why:** Lower volume than app logs; **hourly or daily** files are acceptable and cheap to govern.

### 2.4 Path D — Deployments & CI/CD

**Pattern:** **Azure DevOps / GitHub** webhooks or scheduled API pull → **Azure Function** or **ADF pipeline** → append JSON events into `changes/deployments/`.

**Why:** Not in LA by default; must be **first-class pipeline** or RAG loses “what shipped when.”

### 2.5 Path E — Inventory snapshots

**Pattern:** **Daily** Azure Resource Graph query via **Automation runbook**, **Function**, or **ADF** → single snapshot per day (or per subscription) under `inventory/`.

**Why:** Slow-changing; **batch** is correct and cheap.

### 2.6 Path F — Silver / curated layer (recommended)

**Pattern:** **Raw** landing (`bronze/`) from exports → **Azure Synapse / Databricks / Fabric** job deduplicates, **redacts PII**, normalizes schema → **Silver** Parquet/JSONL under `silver/` for **RAG ingestion**.

**Why:** **Industry standard** “medallion” thinking: RAG reads **curated** evidence with consistent fields—improves retrieval quality and security reviews.

### 2.7 Cross-cutting controls

| Control | Rationale |
|---------|-----------|
| **IAM:** Managed identity per pipeline; least privilege on ADLS | Security baseline |
| **Encryption:** CMK if required | Compliance |
| **Lineage:** Pipeline run id, source table, export version in file metadata or sidecar manifest | Audit and debugging |
| **DLQ / dead letter** for failed batches | Operability |

---

## 3. How to store this data (ADLS layout & formats)

### 3.1 Format choice

| Layer | Recommended format | Justification |
|-------|-------------------|---------------|
| **Bronze (raw)** | **JSON Lines** (`.jsonl`) or Avro/CSV as emitted by export | Matches many export tools; easy to debug |
| **Silver (RAG-ready)** | **Parquet** (primary) + optional **JSONL** for small teams | Columnar compression, schema evolution, fast scans for chunk building jobs |
| **Knowledge** | **Markdown** or **JSON** (chunked later) | Human-authored content stays readable |

### 3.2 Directory layout (example)

```text
abfss://<container>/
  bronze/
    la-export/<table>/ingest_date=YYYY-MM-DD/hour=HH/*.jsonl
    activitylog/ingest_date=YYYY-MM-DD/*.jsonl
  silver/
    logs/app/
      env=<env>/service=<service>/date=YYYY-MM-DD/part-*.parquet
    logs/platform/
      resource_type=<type>/date=YYYY-MM-DD/part-*.parquet
    changes/
      deployments/date=YYYY-MM-DD/part-*.parquet
    inventory/
      snapshot_date=YYYY-MM-DD/inventory.parquet
  knowledge/
    runbooks/.../*.md
  governance/
    schemas/field_dictionary.json
    policies/pii_manifest.json
```

### 3.3 Partitioning keys (minimum)

- **`date`** (always) — time-travel queries and retention policies  
- **`env`** (dev/stage/prod) — **mandatory** for safety and RAG filtering  
- **`service` or `cloud_RoleName`** — scoped retrieval  
- **`region`** (optional) — large global estates  

### 3.4 Retention & tiers

- **Hot:** recent windows (e.g., 7–30 days) heavily used for incident RAG  
- **Cool/archive:** older partitions per compliance; RAG may **not** index full archive  

### 3.5 Security

- **Private endpoints**, **ACLs/ACL-style** or **RBAC** on paths per team where needed  
- **No secrets** in lake; redact before `silver/`  
- **Separate containers** for **prod vs non-prod** if policy requires hard isolation  

---

## 4. How many files will be there

**Honest industry answer:** File count is **not fixed**—it is driven by **export granularity, partition strategy, event rate, and file rollover policy**. You **design** it via **target file size** and **schedule**, not a single magic number.

### 4.1 Rules of thumb (design targets)

| Goal | Typical practice |
|------|------------------|
| **Queryable / parallel** | Many **moderate-sized** files beat one giant file (Spark/parallel ingest) |
| **Avoid small-file problem** | Avoid millions of **tiny** files (<~64–128 MB for heavy analytics; relax for JSONL if volume is modest) |
| **Rollover** | **Hourly** or **by size** (e.g., 256 MB–1 GB) for high-volume streams |

### 4.2 Order-of-magnitude examples (illustrative, not a promise)

Assume **continuous JSONL** export for **one** high-volume app log stream:

- **Hourly partitions:** ~**24 files/day/stream** (1 per hour if single writer per partition)  
- **Per service × per env × per day:** multiply by **number of services** you split out in Silver  

Example: **10 services**, **hourly** Silver Parquet for **prod only**:

- Roughly **10 × 24 = 240 Parquet files/day** in that slice (before compactions)  

**Activity / deployments / inventory:**

- **Activity Log:** often **1–24 files/day** depending on export batching  
- **Deployments:** **tens to hundreds** of small JSON events/day → often **1 file/day** after aggregation or **append-style** micro-batches  
- **Inventory:** **1 snapshot file/day** per scope (subscription or tenant)  

### 4.3 What you should document for stakeholders

State explicitly:

1. **Partitioning** (date, env, service)  
2. **Rollover policy** (time vs size)  
3. **Compaction job** (optional daily job merging small files → fewer, larger Parquet files)  
4. **Expected order of magnitude** after a **pilot** (measure 24h volume, then extrapolate)  

---

## Summary table

| # | Topic | Industry baseline takeaway |
|---|--------|----------------------------|
| 1 | **Data** | Logs + platform + activity + deploy/config changes + inventory + (optional) alerts/knowledge + governance artifacts |
| 2 | **Pipeline** | LA export and/or Event Hubs → stream/batch writers; CI/CD and ARG as separate batch paths; **Bronze → Silver** for RAG |
| 3 | **Storage** | Partitioned ADLS Gen2; **Parquet in Silver**; strict **env/service** separation; PII redaction before RAG-facing layer |
| 4 | **File count** | **Policy-driven** (hourly/size rollover); use **pilot metrics** to quote numbers; optional **compaction** to control file count |

---

*This document is a solution design baseline; tune partitions and export mechanisms to your org’s compliance review and Azure subscription capabilities.*
