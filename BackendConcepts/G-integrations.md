# Section G — Integrations

> Reference outline. Stack: axios; a KYC provider; an email provider; an SMS provider (DLT); Azure Blob.

## G1 — HTTP client resilience (axios + circuit breaker)

**Diagram**
```
 call -> timeout? -> retry (backoff + jitter, idempotent only)
                       |
        too many failures -> CIRCUIT OPEN: fail fast, no calls
        wait -> HALF-OPEN: try one -> works? CLOSED again
```
**In plain words.** The fuse box in your house. When a device keeps failing, the fuse trips so it
stops hurting everything else.

- Set **timeouts** (never hang forever), **retries with backoff + jitter** (only on safe/idempotent
  calls), and a **circuit breaker** (stop hammering a failing dependency; fail fast, recover gracefully).
- One line: every outbound call gets a timeout, careful retries, and a circuit breaker.

## G2 — Webhooks (inbound)

**Diagram**
```
 inbound webhook -> verify HMAC(raw body) + timestamp -> dedupe event id
                 -> enqueue job -> reply 200 fast
 heavy work happens in the worker, not in the request
```
**In plain words.** Check the seal, write the note, say "got it" quickly, do the job later.

- Verify **HMAC over the raw body** (D16), **timestamp window** (replay), **idempotent** processing
  (F2). Respond fast (2xx) and do heavy work async.
- One line: verify signature + timestamp, process idempotently, ack fast and work async.

## G3 — Third-party KYC provider

**Diagram**
```
 supplier submits PAN/GSTIN
   v
 KYC job: client(timeout+retry+breaker) -> a KYC provider
   v
 PENDING -> VERIFIED | REJECTED   (state machine, audited, PII masked)
```
**In plain words.** Treat the KYC provider like an unreliable courier: set deadlines, retry
carefully, and track the parcel status.

- Treat as **untrusted, flaky** dependencies: timeouts, retries, circuit breaker, and a **state machine**
  for the verification lifecycle. Store request/response for audit (mask PII). Fixed URLs (no SSRF).
- One line: wrap KYC providers in resilient clients + a verification state machine, with masked audit.

## G4 — Email / SMS providers (+ DLT)

**Diagram**
```
 event (OTP needed) -> queue -> worker -> SMS provider/email provider
 SMS (India): must use DLT-approved template + sender (RAFTTR)
 idempotent: same (event, channel, user) never sent twice
```
**In plain words.** Messages go out through a queue. India's SMS rules require pre-approved
templates, so the text must match exactly.

- Send async via a queue (F1). India SMS needs **DLT-registered templates/sender IDs**. Handle provider
  failures, bounces, rate limits; make sends idempotent (don't double-send OTPs).
- One line: send comms async, use DLT-approved templates for India SMS, and make sends idempotent.

## G5 — Object storage (Azure Blob) & signed URLs

**Diagram**
```
 client -> API: "I want to upload"
 API -> returns short-lived SAS URL (limited path, limited time)
 client -> uploads straight to Azure Blob (your server not involved)
 API validates type/size after upload
```
**In plain words.** A temporary visitor pass for one room, instead of handing out the building
master key.

- Store files in blob storage (sidesteps path traversal, D7). Serve/upload via **short-lived SAS/signed
  URLs** so clients talk to storage directly without exposing keys. Validate content-type/size on upload.
- One line: keep files in blob storage and grant access via short-lived signed (SAS) URLs, not keys.
