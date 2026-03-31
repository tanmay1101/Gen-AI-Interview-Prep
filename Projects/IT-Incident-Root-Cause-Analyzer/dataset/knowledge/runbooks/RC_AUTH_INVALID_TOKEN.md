# Runbook: Invalid / Expired Auth Token (401 / Auth Failures)

`root_cause_id: RC_AUTH_INVALID_TOKEN`

## When this root cause applies

Use this runbook when requests fail due to authentication/authorization problems.

Typical symptoms:

- HTTP `401 Unauthorized` / `403 Forbidden`
- errors containing `invalid token`, `expired token`, `signature`, `JWT`, `unauthorized`

## Evidence signatures

- `statusCode` = `401` or `403`
- `error` includes `invalid token`, `token expired`, `signature invalid`
- auth gateway logs show denial for the same incident window

## Likely contributing factors

- bad token generation or clock skew
- key rotation mismatch (public/private key mismatch)
- misconfigured audience/issuer values
- stale secrets/config cached incorrectly

## Recommended remediation steps

### 1) Confirm auth failure patterns

- Identify affected auth provider and endpoint.
- Verify earliest evidence in the incident window is auth rejection.

### 2) Mitigate

- Restore correct signing key material (via approved runbook steps).
- Validate issuer/audience configuration.
- Roll back misconfigured auth settings if evidence supports it.

### 3) Validate

- Confirm 401/403 rates drop.
- Confirm successful authentication for new requests.

## Evidence-to-conclusion rules

- If earliest evidence includes `invalid token` or `token expired`, pick `RC_AUTH_INVALID_TOKEN`.

