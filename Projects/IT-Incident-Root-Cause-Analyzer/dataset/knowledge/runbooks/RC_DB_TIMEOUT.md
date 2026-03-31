# Runbook: DB Timeout / SQL Connection Timeout

`root_cause_id: RC_DB_TIMEOUT`

## When this root cause applies

Use this runbook when the incident shows patterns like:

- checkout (or payments) requests failing with `TimeoutException`, `i/o timeout`, `Read timed out`
- dependency call durations spiking (e.g., `durationMs > 20000`)
- upstream services reporting generic `502/504` while DB layer shows timeouts or saturation

## Evidence signatures (what to look for)

In logs, search for combinations of:

- `service`: your app service (example: `checkout-api`)
- `dependency`: `payments-sql` / `sql-db` / `postgres`
- `error`: `System.TimeoutException` (or `Timeout`)
- keywords: `Timeout`, `timed out`, `deadlock` (deadlock is sometimes a subtype, still use this runbook)

## Likely contributing factors

- DB CPU throttling / saturation
- connection pool exhaustion
- slow queries or lock contention
- network path degradation between app and DB

## Recommended remediation steps (safe-first)

### 1) Confirm the dependency is the bottleneck

- Compare app-side error timestamps with DB-side timeout timestamps.
- Group by `trace_id` / `operation_Id` and confirm the first failure is the dependency call.

### 2) Check DB saturation indicators

- CPU / DTU or vCore saturation
- lock wait / deadlocks (if available)
- active connections vs connection pool max

### 3) Mitigate quickly

- If a pool is exhausted: reduce concurrency or increase pool/limits (where allowed).
- If queries are slow: identify top slow queries and apply query optimization or indexes.
- If network issue: verify connectivity/NSG/route path (no risky changes unless confirmed).

### 4) Validate

- Confirm reduced timeout frequency for the same `operation_Id` patterns.
- Confirm error rate returns toward baseline within the incident window.

## What NOT to do (avoid)

- Don’t restart the entire app unless evidence strongly suggests app-level stuck threads.
- Don’t roll deployments repeatedly without checking DB saturation and query latency.

## Evidence-to-conclusion rules (for your analyzer)

- If the earliest evidence is a dependency call `TimeoutException` to the DB, prefer `RC_DB_TIMEOUT`.
- If errors start in auth/token validation, do **not** select this root cause.

