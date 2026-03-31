# Runbook: Disk Full / Storage Exhaustion

`root_cause_id: RC_DISK_FULL`

## When this root cause applies

Use this runbook when logs indicate storage saturation.

Typical signs:

- `No space left on device`
- `disk quota exceeded`
- failures to write temp files / logs / blobs
- cascading timeouts after I/O stalls

## Evidence signatures

- error messages containing `No space left on device`
- `disk` / `storage` / `quota` keywords
- writes failing followed by downstream failures

## Likely contributing factors

- runaway log growth
- temp files not cleaned
- quota misconfiguration
- unexpected bulk ingestion consuming disk

## Recommended remediation steps

### 1) Confirm storage exhaustion

- Identify affected mount/volume.
- Confirm disk usage metrics correlate with incident window.

### 2) Mitigate quickly

- Free space using safe cleanup steps (approved locations).
- If logs are the cause: rotate/compress and cap retention if policy allows.

### 3) Validate

- Confirm write failures stop for the same incident window.

## Evidence-to-conclusion rules

- If earliest evidence includes `No space left on device` or `quota exceeded`, pick `RC_DISK_FULL`.

