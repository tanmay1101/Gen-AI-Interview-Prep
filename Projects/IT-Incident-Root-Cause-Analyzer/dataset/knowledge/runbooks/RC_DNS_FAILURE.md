# Runbook: DNS Resolution Failure (Host/Name Not Resolved)

`root_cause_id: RC_DNS_FAILURE`

## When this root cause applies

Use this runbook when the incident shows inability to resolve a dependency hostname.

Typical patterns:

- Errors like `Name or service not known`, `getaddrinfo ENOTFOUND`, `Temporary failure in name resolution`
- Affected dependency host appears unreachable only at name resolution layer
- Symptoms: immediate connection failures, sometimes rapid retries then cascading timeouts

## Evidence signatures

Look for:

- `error` values containing `ENOTFOUND`, `DNS`, `getaddrinfo`
- `dependency` targets with expected hostnames (example: `payments-sql.internal`)
- logs showing failures at the dependency call boundary (before long durations)

## Likely contributing factors

- DNS resolver outage or misconfiguration
- blocked UDP/TCP to DNS servers
- expired DNS records / routing changes
- container DNS settings drift

## Recommended remediation steps (safe-first)

### 1) Confirm it’s a resolution issue

- Verify failures cluster by dependency hostname.
- Compare to incidents where direct IP connectivity works (if you track it).

### 2) Check network path to DNS

- Validate outbound DNS traffic is allowed (firewall/NSG) without risky changes.
- Confirm no recent networking/security policy changes coinciding with the incident window.

### 3) Mitigate

- If there’s a known safe override: use a temporary static host mapping only per policy.
- Consider fallback to alternate resolver (only if established and permitted).

### 4) Validate

- After mitigation, confirm dependency resolution errors drop for the same hostnames.

## What NOT to do (avoid)

- Don’t assume DB timeout logic is correct if resolution errors appear earlier.
- Don’t restart apps as the first step unless you have evidence of app-level DNS caching issues.

## Evidence-to-conclusion rules

- If earliest evidence includes `ENOTFOUND` / `getaddrinfo` for the dependency hostname, pick `RC_DNS_FAILURE`.

