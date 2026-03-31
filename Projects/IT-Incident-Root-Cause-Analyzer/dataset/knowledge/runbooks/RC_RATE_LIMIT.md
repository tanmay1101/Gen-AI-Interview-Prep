# Runbook: Rate Limit / Throttling (429 / Quota Exhausted)

`root_cause_id: RC_RATE_LIMIT`

## When this root cause applies

Use this runbook when failures correlate with throttling, quota exhaustion, or aggressive retries.

Typical symptoms:

- HTTP `429 Too Many Requests`
- dependency returns `rate exceeded`, `throttled`, `quota`, `TooManyRequests`
- latency spikes due to retry storms

## Evidence signatures

- `statusCode` or `response_code` contains `429`
- error messages containing `throttled`, `rate limit`, `quota`
- high retry counts (if present) plus `TooManyRequests`

## Likely contributing factors

- traffic burst exceeding quotas
- missing backoff/jitter in retry logic
- upstream dependency returning throttling
- configuration changes to timeouts/retry policy

## Recommended remediation steps

### 1) Confirm throttling and scope

- Confirm `429` or throttle messages are the earliest evidence.
- Identify affected dependency and resource.

### 2) Mitigate

- Reduce concurrency / enable safe backoff.
- If there is an approved scaling path, scale the throttled component (follow policy).
- If retries are misconfigured, correct retry backoff parameters.

### 3) Validate

- Check that 429 rate drops and error rate normalizes.

## Evidence-to-conclusion rules

- If earliest evidence includes `429` / `rate limit` tokens, pick `RC_RATE_LIMIT`.

