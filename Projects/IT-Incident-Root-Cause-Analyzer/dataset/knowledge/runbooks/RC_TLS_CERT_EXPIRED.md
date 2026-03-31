# Runbook: TLS Certificate Expired / Not Valid Yet

`root_cause_id: RC_TLS_CERT_EXPIRED`

## When this root cause applies

Use this runbook when secure connection failures are due to certificate validity problems.

Typical signs:

- `certificate has expired`
- `x509: certificate is not valid`
- TLS handshake errors
- retries followed by connection closure

## Evidence signatures

Search for:

- error messages containing `x509`, `certificate`, `expired`, `not valid`
- dependency types involving HTTPS, mTLS, or TLS-secured DB endpoints

## Likely contributing factors

- certificate renewal job missed / expired in managed store
- wrong cert deployed to endpoint
- clock skew on client/container (less common but possible)

## Recommended remediation steps (safe-first)

### 1) Confirm certificate validity mismatch

- Identify the exact endpoint hostname from logs.
- Determine if the failure began around the certificate expiry time.

### 2) Check rotation/renewal

- Verify cert renewal/rotation pipeline logs.
- Confirm which certificate serial/issuer was deployed.

### 3) Mitigate

- Prefer restoring valid certificate from approved source.
- If possible, roll back to last known-good cert only after evidence.

### 4) Validate

- Verify handshake success rate improves immediately.
- Monitor for repeated TLS failures after mitigation.

## Evidence-to-conclusion rules

- If the earliest evidence includes `x509` + `expired` or `not valid yet`, pick `RC_TLS_CERT_EXPIRED`.

