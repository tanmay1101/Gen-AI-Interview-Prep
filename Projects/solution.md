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

### 1.9 Illustrative samples (5 per category)

*Synthetic examples for design reviews; field names align with common Application Insights / Azure Monitor shapes. PII values are fake.*

#### 1.9.1 Application & workload logs — 5 samples

```json
{"TimeGenerated": "2026-03-19T14:02:11.203Z", "type": "requests", "cloud_RoleName": "checkout-api", "env": "prod", "region": "eastus", "resultCode": "500", "durationMs": 8421, "name": "POST /api/v1/orders", "operation_Id": "a1b2c3d4e5f67890", "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736", "subscriptionId": "sub-1111", "resourceId": "/subscriptions/sub-1111/resourceGroups/rg-prod/providers/Microsoft.Web/sites/checkout-api", "customDimensions": {"tenantId": "tnt_***", "release": "2026.03.18.3"}}
```

```json
{"TimeGenerated": "2026-03-19T14:02:11.881Z", "type": "dependencies", "cloud_RoleName": "checkout-api", "env": "prod", "region": "eastus", "target": "payments-sql", "type_dependency": "SQL", "resultCode": "Timeout", "durationMs": 30012, "success": false, "operation_Id": "a1b2c3d4e5f67890", "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736"}
```

```json
{"TimeGenerated": "2026-03-19T14:02:12.010Z", "type": "exceptions", "cloud_RoleName": "checkout-api", "env": "prod", "region": "eastus", "problemId": "System.TimeoutException at PaymentClient.Charge", "outerMessage": "Charge call exceeded 30s budget", "severityLevel": "Error", "operation_Id": "a1b2c3d4e5f67890", "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736"}
```

```json
{"TimeGenerated": "2026-03-19T14:01:58.440Z", "type": "containerlog", "kubernetes_namespace": "prod", "pod_name": "checkout-api-7d9c8f4b5-xk2zq", "container_name": "checkout-api", "log": "WARN retry=2 upstream=payments timeout=30s trace_id=4bf92f3577b34da6a3ce929d0e0e4736", "env": "prod", "region": "eastus"}
```

```json
{"TimeGenerated": "2026-03-19T14:00:05.000Z", "type": "function_execution", "functionName": "OrderReconciliationTimer", "env": "prod", "region": "eastus", "success": false, "durationMs": 120500, "message": "Durable orchestration failed: activity InventoryReserve timed out", "invocationId": "inv-998877", "trace_id": "00f067aa0ba902b7"}
```

#### 1.9.2 Platform & resource diagnostic logs — 5 samples

```json
{"TimeGenerated": "2026-03-19T14:02:05.100Z", "ResourceProvider": "MICROSOFT.SQL", "category": "Deadlocks", "resourceId": "/subscriptions/sub-1111/resourceGroups/rg-prod/providers/Microsoft.Sql/servers/sql-prod/databases/payments", "env": "prod", "region": "eastus", "deadlock_xml_snippet": "<deadlock>...</deadlock>", "victimProcess": "process2"}
```

```json
{"TimeGenerated": "2026-03-19T14:01:22.555Z", "ResourceProvider": "MICROSOFT.DOCUMENTDB", "category": "DataPlaneRequests", "resourceId": "/subscriptions/sub-1111/resourceGroups/rg-prod/providers/Microsoft.DocumentDB/databaseAccounts/cosmos-prod", "statusCode": 429, "requestResourceType": "Document", "operationName": "Query", "env": "prod", "region": "eastus"}
```

```json
{"TimeGenerated": "2026-03-19T14:00:10.001Z", "ResourceProvider": "MICROSOFT.STORAGE", "category": "StorageRead", "resourceId": "/subscriptions/sub-1111/resourceGroups/rg-prod/providers/Microsoft.Storage/storageAccounts/stcheckoutprod", "statusCode": 503, "callerIpAddress": "10.20.30.40", "uri": "/orders/container/blob", "env": "prod", "region": "eastus"}
```

```json
{"TimeGenerated": "2026-03-19T13:59:44.200Z", "ResourceProvider": "MICROSOFT.KEYVAULT", "category": "AuditEvent", "resourceId": "/subscriptions/sub-1111/resourceGroups/rg-prod/providers/Microsoft.KeyVault/vaults/kv-checkout-prod", "operationName": "SecretGet", "resultSignature": "Forbidden", "callerIdentity": "msi-checkout-api", "env": "prod", "region": "eastus"}
```

```json
{"TimeGenerated": "2026-03-19T13:58:01.900Z", "ResourceProvider": "MICROSOFT.CACHE", "category": "ConnectedClientList", "resourceId": "/subscriptions/sub-1111/resourceGroups/rg-prod/providers/Microsoft.Cache/redis/redis-checkout", "connectedClients": 512, "env": "prod", "region": "eastus", "note": "sample metric-style diagnostic row"}
```

#### 1.9.3 Control plane & change history — 5 samples

```json
{"time": "2026-03-19T13:45:00Z", "type": "ActivityLog", "operationName": "Microsoft.Web/sites/write", "status": "Succeeded", "caller": "pipeline@contoso.com", "subscriptionId": "sub-1111", "resourceGroup": "rg-prod", "resourceId": "/subscriptions/sub-1111/resourceGroups/rg-prod/providers/Microsoft.Web/sites/checkout-api", "properties": {"entity": "/subscriptions/.../sites/checkout-api", "statusCode": "OK"}}
```

```json
{"time": "2026-03-19T12:10:00Z", "type": "ActivityLog", "operationName": "Microsoft.Network/networkSecurityGroups/write", "status": "Succeeded", "caller": "netops@contoso.com", "subscriptionId": "sub-1111", "resourceGroup": "rg-prod", "resourceId": "/subscriptions/sub-1111/resourceGroups/rg-prod/providers/Microsoft.Network/networkSecurityGroups/nsg-aks", "properties": {"message": "Security rule added: AllowAppGwToAks"}}
```

```json
{"time": "2026-03-19T11:05:33Z", "type": "Deployment", "source": "AzureDevOps", "project": "Commerce", "releaseId": "R28491", "environment": "prod", "stage": "Deploy EastUS", "commitSha": "a9f3c21", "services": ["checkout-api", "payments-worker"], "triggeredBy": "release-pipeline", "status": "succeeded"}
```

```json
{"time": "2026-03-19T11:04:10Z", "type": "Deployment", "source": "GitHubActions", "workflow": "deploy-prod.yml", "runId": 88112233, "environment": "prod", "commitSha": "a9f3c21", "imageTag": "checkout-api:2026.03.18.3", "status": "succeeded"}
```

```json
{"time": "2026-03-19T10:00:00Z", "type": "ConfigChange", "source": "AppConfiguration", "store": "appconfig-prod", "key": "Payments:TimeoutSeconds", "oldValue": "[redacted]", "newValue": "[redacted]", "label": "prod", "changedBy": "sp-checkout-config", "etag": "5"}
```

#### 1.9.4 Security & network — 5 samples

```json
{"time": "2026-03-19T14:02:00Z", "type": "AzureFirewall", "category": "AzureFirewallApplicationRule", "action": "Deny", "sourceIp": "203.0.113.50", "targetFqdn": "malware.example.invalid", "policy": "fw-prod", "ruleCollectionGroup": "DefaultDeny", "env": "prod", "region": "eastus"}
```

```json
{"time": "2026-03-19T14:01:12Z", "type": "AzureFirewall", "category": "AzureFirewallNetworkRule", "action": "Allow", "sourceIp": "10.10.1.25", "destinationIp": "10.20.3.40", "port": "443", "protocol": "TCP", "env": "prod", "region": "eastus"}
```

```json
{"time": "2026-03-19T14:00:40Z", "type": "AppGatewayWAF", "instanceId": "appgw-prod_0", "clientIP": "198.51.100.22", "requestUri": "/api/v1/orders", "ruleId": "942100", "action": "Blocked", "message": "SQL Injection Attempt", "env": "prod", "region": "eastus"}
```

```json
{"time": "2026-03-19T13:55:00Z", "type": "NSGFlowAggregated", "note": "often processed from flow logs; sample aggregate row", "nsg": "nsg-aks", "direction": "InBound", "allowed": 125000, "denied": 42, "windowStart": "2026-03-19T13:50:00Z", "env": "prod", "region": "eastus"}
```

```json
{"time": "2026-03-19T13:50:00Z", "type": "DDoSProtection", "publicIp": "203.0.113.10", "attackType": "Volumetric", "mitigationState": "MitigationInProgress", "maxpps": 890000, "env": "prod", "region": "eastus"}
```

#### 1.9.5 Inventory & topology — 5 samples

```json
{"snapshotDate": "2026-03-19", "id": "/subscriptions/sub-1111/resourceGroups/rg-prod/providers/Microsoft.Web/sites/checkout-api", "name": "checkout-api", "type": "microsoft.web/sites", "location": "eastus", "subscriptionId": "sub-1111", "resourceGroup": "rg-prod", "tags": {"env": "prod", "team": "commerce", "costCenter": "CC-4401", "criticality": "tier1"}}
```

```json
{"snapshotDate": "2026-03-19", "id": "/subscriptions/sub-1111/resourceGroups/rg-prod/providers/Microsoft.Sql/servers/sql-prod", "name": "sql-prod", "type": "microsoft.sql/servers", "location": "eastus", "subscriptionId": "sub-1111", "resourceGroup": "rg-prod", "tags": {"env": "prod", "team": "data", "dataClass": "pci"}}
```

```json
{"snapshotDate": "2026-03-19", "id": "/subscriptions/sub-1111/resourceGroups/rg-prod/providers/Microsoft.DocumentDB/databaseAccounts/cosmos-prod", "name": "cosmos-prod", "type": "microsoft.documentdb/databaseaccounts", "location": "eastus", "subscriptionId": "sub-1111", "resourceGroup": "rg-prod", "tags": {"env": "prod", "team": "platform"}}
```

```json
{"snapshotDate": "2026-03-19", "id": "/subscriptions/sub-1111/resourceGroups/rg-prod/providers/Microsoft.ContainerService/managedClusters/aks-prod", "name": "aks-prod", "type": "microsoft.containerservice/managedclusters", "location": "eastus", "subscriptionId": "sub-1111", "resourceGroup": "rg-prod", "tags": {"env": "prod", "team": "platform", "workload": "checkout"}}
```

```json
{"snapshotDate": "2026-03-19", "id": "/subscriptions/sub-1111/resourceGroups/rg-prod/providers/Microsoft.Network/applicationGateways/appgw-prod", "name": "appgw-prod", "type": "microsoft.network/applicationgateways", "location": "eastus", "subscriptionId": "sub-1111", "resourceGroup": "rg-prod", "tags": {"env": "prod", "team": "network", "waf": "enabled"}}
```

#### 1.9.6 Operational artifacts — 5 samples

```json
{"time": "2026-03-19T14:02:30Z", "type": "AlertFired", "alertRuleName": "checkout-5xx-spike", "severity": "Sev2", "targetResource": "/subscriptions/sub-1111/.../sites/checkout-api", "firedCount": 1, "dimensions": {"region": "eastus"}, "linkToPortal": "https://portal.azure.com/#blade/..."}
```

```json
{"time": "2026-03-19T14:02:31Z", "type": "AlertFired", "alertRuleName": "payments-sql-dtu", "severity": "Sev3", "targetResource": "/subscriptions/sub-1111/.../databases/payments", "firedCount": 1, "dimensions": {"metric": "dtu_consumption_percent"}}
```

```json
{"time": "2026-03-19T14:00:00Z", "type": "AlertRule", "name": "checkout-5xx-spike", "enabled": true, "condition": "requests/failed > 50 in 5m", "actionGroup": "ag-commerce-prod", "env": "prod"}
```

```json
{"time": "2026-03-19T14:05:00Z", "type": "ITSMIncident", "system": "ServiceNow", "incidentNumber": "INC00188421", "title": "Checkout API elevated 5xx - East US", "priority": "P2", "cmdb_ci": "checkout-api", "assignedGroup": "Commerce-SRE", "correlationId": "a1b2c3d4e5f67890"}
```

```json
{"time": "2026-03-19T14:06:12Z", "type": "ITSMIncident", "system": "ServiceNow", "incidentNumber": "INC00188421", "workNote": "Pager triggered; investigating SQL dependency timeouts per App Insights operation_Id a1b2c3d4e5f67890", "author": "oncall@contoso.com"}
```

#### 1.9.7 Reference knowledge — 5 samples (Markdown snippets as stored in ADLS)

**Sample K1 — Runbook excerpt**

```markdown
# Runbook: Checkout API 5xx spike
## Preconditions
- Confirm region and env (prod vs stage).
- Pull last successful deployment time from release dashboard.
## Steps
1. Open App Insights Failures blade; filter `cloud_RoleName == checkout-api`.
2. Correlate `operation_Id` with SQL dependency failures.
3. If DTU saturation: follow `RB-SQL-Scale` (read-only checks first).
```

**Sample K2 — SLO snippet**

```markdown
## SLO: Checkout API (prod)
- Availability target: 99.95% monthly
- Latency p95: < 800ms for POST /api/v1/orders
- Error budget owner: Commerce SRE
```

**Sample K3 — KQL snippet (documentation)**

```kusto
requests
| where cloud_RoleName == "checkout-api"
| where resultCode startswith "5"
| summarize count() by bin(timestamp, 5m), resultCode
| render timechart
```

**Sample K4 — Escalation**

```markdown
# Escalation: Payment dependency
- If `dependencies` to `payments-sql` show Timeout > 30s for > 10% of requests in 15m:
  - Page DBA on-call + Commerce TL
  - Do not restart SQL without DBA approval
```

**Sample K5 — Post-incident checklist**

```markdown
# After checkout incident
- [ ] Link App Insights operation_Ids in incident ticket
- [ ] Attach top 3 exception stacks
- [ ] Record deploy version and config keys touched in last 24h
```

#### 1.9.8 Governance pack — 5 samples

```json
{"schemaVersion": "1.0", "table": "requests", "field": "url", "type": "string", "description": "Request path without query string", "pii": false, "embedInText": true}
```

```json
{"schemaVersion": "1.0", "table": "requests", "field": "client_IP", "type": "string", "description": "Caller IP", "pii": true, "embedInText": false, "transform": "hash_sha256"}
```

```json
{"schemaVersion": "1.0", "table": "traces", "field": "message", "type": "string", "description": "Log line body", "pii": "possible", "embedInText": true, "notes": "Run redaction regex set v3 before silver"}
```

```json
{"policyVersion": "2026.03.1", "ruleId": "PII-EMAIL", "pattern": "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}", "action": "mask", "replacement": "<EMAIL>"}
```

```json
{"policyVersion": "2026.03.1", "ruleId": "SECRET-JWT", "pattern": "eyJ[a-zA-Z0-9_-]+\\.eyJ[a-zA-Z0-9_-]+\\.[a-zA-Z0-9_-]+", "action": "drop_line", "replacement": null}
```

---

### 1.10 How each data category is used in RAG

This section maps **ADLS-landed data** to the RAG pipeline (aligned with **Ingestion → Query processing → Generation**). Not every row is embedded: some data is **metadata for filters**, some is **primary evidence** in the prompt, and some is **orchestration-only** (never shown to the LLM raw).

#### RAG roles (quick glossary)

| Role | Meaning |
|------|--------|
| **Embed** | Text (or a derived “embedding text” field) is chunked, embedded, and stored in the vector index with metadata. |
| **Filter** | Fields used in **pre-retrieval** filters (env, service, time window, tenant) — may not need their own embedding. |
| **Keyword / BM25** | Exact tokens (error codes, `operation_Id`, resource IDs) — hybrid search over the same or a parallel text index. |
| **Context** | Top-k chunks passed into the LLM prompt as **grounding** with **citations** (source path, timestamp, ids). |
| **Orchestration** | Used by the **application** to build queries or UI (e.g., resolve “checkout-api” → resource scope); optional small embeddings for “org knowledge graph” patterns. |

---

#### 1.10.1 Application & workload logs

| RAG stage | How it is used |
|-----------|----------------|
| **Ingestion** | Normalize to a **canonical log document**: human-readable line(s) for `page_content` (or a templated summary string), plus rich **metadata** (`TimeGenerated`, `env`, `service`, `trace_id`, `operation_Id`, `severity`, `resourceId`). Apply **governance** redaction before embed. |
| **Indexing** | **Embed** windowed or single-event chunks (or “trace-scoped” bundles: same `operation_Id` / `trace_id`). **Hybrid**: vector for paraphrase (“timeouts on checkout”) + **keyword** on `resultCode`, exception type, `operation_Id`. |
| **Query processing** | User/incident time range + `env` + `service` **filters** narrow the index; optional **expand** with adjacent chunks (same trace) for parent-child retrieval. |
| **Generation** | Primary **evidence** in the prompt: “Here are log excerpts…” with **citations** (timestamp, service, trace/operation id, ADLS/Silver path). Model answers *what happened* and *what correlated* within retrieved evidence. |

**Why this split:** Logs are the **main factual grounding**; correlation IDs tie multiple chunks into one incident story for the LLM.

---

#### 1.10.2 Platform & resource diagnostic logs

| RAG stage | How it is used |
|-----------|----------------|
| **Ingestion** | Same pattern as app logs: `page_content` = summarized diagnostic row (category, resource, status, short detail); **metadata** includes `ResourceProvider`, `resourceId`, `category`. |
| **Indexing** | **Embed** for semantic retrieval (“Cosmos throttling”, “SQL deadlock”); **keyword** on `statusCode`, `category`, resource name. Often a **separate collection** or **metadata tag** `source_type=platform` so users can filter “platform vs app” evidence. |
| **Query processing** | When the user question implies dependency (`SQL`, `Cosmos`, `KeyVault`), boost or **filter** `source_type=platform` and dependency resource family. |
| **Generation** | Used as **supporting evidence** next to app logs — e.g., “App shows SQL timeout; platform row shows 429/503/deadlock at same window.” Citations must name **resource** and **time**. |

**Why:** Prevents the model from blaming only application code when the **data plane** shows quota, auth, or storage errors.

---

#### 1.10.3 Control plane & change history

| RAG stage | How it is used |
|-----------|----------------|
| **Ingestion** | Short, structured **documents**: one Activity operation, one deployment event, one config change per chunk (or batched by hour). `page_content` = natural-language summary line + key fields. |
| **Indexing** | **Embed** for “what changed before the incident?” **Keyword** on `commitSha`, `releaseId`, `operationName`, resource id. |
| **Query processing** | **Time-slice retrieval**: “changes in [-6h, 0] from incident start” is a common pattern — either **metadata filter** on `time` or a dedicated “changes” index sorted by time. |
| **Generation** | Grounds **narrative** (“Deploy `2026.03.18.3` completed 45m before spike”) and **hypothesis** language — still with citations to change records, not speculation. |

**Why:** Change data answers the most common stakeholder question — **what changed** — without guessing from logs alone.

---

#### 1.10.4 Security & network

| RAG stage | How it is used |
|-----------|----------------|
| **Ingestion** | Strict **redaction**; `page_content` = policy-safe summary (action, rule, high-level target without sensitive detail if policy requires). |
| **Indexing** | **Embed** + **keyword** on rule id, action Allow/Deny, service name. Often **restricted index** or **row-level security** so only security-approved roles retrieve these chunks. |
| **Query processing** | If question is connectivity / attack / block, **route** to this corpus or apply **metadata** `domain=security`. |
| **Generation** | **Supporting evidence** for “was traffic blocked?” “was WAF triggered?” Always cite rule/category; avoid inventing PCAP-level detail not in chunks. |

**Why:** Closes the loop on **network path** incidents while respecting **least privilege** on sensitive telemetry.

---

#### 1.10.5 Inventory & topology

| RAG stage | How it is used |
|-----------|----------------|
| **Ingestion** | Snapshot rows as documents: `page_content` = “Resource X is type Y in region Z with tags …” |
| **Indexing** | Optional **light embedding** for “which resource is checkout in prod?” queries; often **keyword + filter** is enough. |
| **Query processing** | **Orchestration-heavy**: resolve ambiguous names (“checkout API”) → **resourceId**, **region**, **env** → then **filter** log retrieval. May **not** pass full inventory into the LLM — only **relevant** snapshot rows. |
| **Generation** | **Context** snippets when disambiguation matters (“You have two similarly named resources; incident scope matches …”). |

**Why:** Reduces **wrong-environment** retrieval and **cross-service** confusion — a common failure mode in naive log RAG.

---

#### 1.10.6 Operational artifacts (alerts, incidents)

| RAG stage | How it is used |
|-----------|----------------|
| **Ingestion** | One chunk per fired alert / incident update; `page_content` includes title, severity, linked resource, correlation id if present. |
| **Indexing** | **Embed** so questions in **incident language** (“Sev2 checkout-5xx-spike”) match; **keyword** on `incidentNumber`, `alertRuleName`. |
| **Query processing** | If user provides **INC number** or **alert name**, **filter** first, then **expand** to logs via shared `correlationId` / time window. |
| **Generation** | Aligns answer with **how the org declared** the incident; citations link **ticket id** + **log evidence**. |

**Why:** Bridges chatbot queries to **official incident records** and improves user trust.

---

#### 1.10.7 Reference knowledge (runbooks, SLOs, KQL)

| RAG stage | How it is used |
|-----------|----------------|
| **Ingestion** | Chunk by **heading/section** (structural chunking); keep file path and version in metadata. |
| **Indexing** | **Embed**; usually a **separate collection** `knowledge_base` vs `telemetry`. |
| **Query processing** | **Parallel retrieval**: top-k from **logs** + top-k from **runbooks** (or sequential: logs first, then “what procedure applies?”). |
| **Generation** | **Procedural grounding** — “recommended steps” must cite runbook sections; **must not** contradict retrieved log evidence. If logs don’t support a step, answerability rules apply (“insufficient evidence in logs for step 3”). |

**Why:** RAG is not only “search logs”; it is **logs + how we operate**, with clear **separation of evidence vs procedure**.

---

#### 1.10.8 Governance pack (schema + PII policy)

| RAG stage | How it is used |
|-----------|----------------|
| **Ingestion** | **Not embedded** for end-user Q&A in the typical path. Consumed by **ETL / chunker** to: build `page_content` templates, **drop/mask** fields, validate schema version. |
| **Query processing** | Optionally **version** in audit logs only. |
| **Generation** | **Never** pasted wholesale into prompts. Ensures **consistent, safe** `page_content` for all other categories. |

**Why:** This is **control plane for the RAG factory** — it defines **what the model is allowed to see** and **how text is rendered** before embedding.

---

#### Summary diagram (logical flow)

```text
ADLS (Bronze/Silver)
       │
       ├─► Governance ───────────► chunk/embed pipeline (rules, no LLM)
       │
       ├─► Telemetry chunks ─────► Vector index + (optional) BM25
       │         ▲
       ├─► Changes / alerts ─────► same or parallel index (tagged)
       │
       ├─► Knowledge chunks ─────► separate index (tagged)
       │
       └─► Inventory snapshots ──► filter / disambiguation + small retrieval

User question
       │
       ▼
Query processing (filters, time scope, hybrid search, rerank)
       │
       ▼
Prompt = system + retrieved chunks (citations) + question
       │
       ▼
LLM answer (grounded, cited; runbook + log evidence distinguished)
```

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
