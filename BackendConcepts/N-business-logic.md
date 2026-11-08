# Section N — Repo-Specific Business Logic

> Reference outline — the domain layer of the buyer/supplier/warehouse platform.

## N1 — Supplier onboarding state machine

**Diagram**
```
 registered -> kyc_pending -> verified -> active -> suspended
                    |                        ^         |
                    +--> rejected            +---------+ (reactivate)
 only drawn arrows are allowed; each move is audited
```
**In plain words.** A board game with fixed squares. You cannot jump from "registered" straight
to "active".

- Model onboarding as explicit **states + transitions** (registered → KYC pending → verified → active →
  suspended). Guard illegal transitions; audit each (D15). Drives resilient KYC (G3).
- One line: model onboarding as a guarded state machine with audited transitions.

## N2 — KYC pipeline

**Diagram**
```
 submit -> job (idempotent per applicant)
        -> a KYC provider (timeout, retry, breaker)
        -> save request+response (PII masked)
        -> PENDING | VERIFIED | REJECTED
```
**In plain words.** One careful, repeatable checking line that leaves a paper trail.

- Orchestrate a KYC provider calls with retries/circuit breaker (G1), persist request/response
  (masked PII, D14), idempotent per applicant (F2), and a clear pending/verified/rejected outcome.
- One line: a resilient, idempotent, audited KYC orchestration with masked PII.

## N3 — Banking details encryption

**Diagram**
```
 bank a/c -> AES-GCM encrypt (key in KMS) -> stored ciphertext
 read: decrypt only when needed -> mask in UI/logs (XXXX1234) -> audit who looked
```
**In plain words.** Bank details sit in a locked box. The key is kept in a different building.

- **Field-level AES-GCM** for bank account/IFSC, key in KMS (D12), masked in logs/UI (D14), access
  audited (D15). Decrypt only when strictly needed.
- One line: field-encrypt bank data (AES-GCM + KMS), mask everywhere, audit access.

## N4 — Catalog moderation

**Diagram**
```
 supplier adds product -> PENDING -> moderation queue -> admin reviews
   -> APPROVED | REJECTED (reason)   audited, descriptions output-encoded
```
**In plain words.** Nothing goes on the shelf until a checker approves it.

- Review/approve supplier catalog entries; state machine + audit; guard against injection in
  descriptions (D2/D4); moderation queue (F1).
- One line: moderate catalog via a queued, audited approval workflow with output-encoding.

## N5 — Loyalty / points system

**Diagram**
```
 ledger (append-only):  +100 order7 | -30 redeem | +50 order9
 balance = SUM(entries) = 120
 accrual idempotent by key (order id); inserted in a transaction
```
**In plain words.** A bank passbook: you only add lines, never erase. The balance is the total of
the lines.

- Company-level point accumulation, **ledger schema** (append-only entries, derive balance), failure-mode
  analysis, idempotent accrual (F2). Money-adjacent → audit (D15) + transactions (E5).
- One line: model points as an append-only ledger with idempotent, audited, transactional accrual.

## N6 — Pricing / quotes

**Diagram**
```
 price rule v3 (versioned) -> quote uses a specific version
 validate inputs (no negative, no overflow) -> save in a transaction -> audit change
```
**In plain words.** Prices follow written rules with version numbers, so any past quote can be
explained.

- Deterministic pricing rules, versioned; audit changes; transactional writes; guard against negative/
  overflow inputs (D1).
- One line: version pricing rules, validate inputs hard, and audit changes.

## N7 — Admin RBAC + audit

**Diagram**
```
 admin action -> role ok? -> tenant ok? -> ownership ok? -> do it -> audit row
```
**In plain words.** Every admin move passes the same three gates and leaves a footprint.

- Admin actions gated by role+ownership (C9), tenant-scoped (C10), every sensitive action in an
  append-only tamper-evident audit log (D15).
- One line: gate admin actions by RBAC+ownership+tenant and audit every one immutably.

## N8 — Notifications fan-out

**Diagram**
```
 event -> fan-out -> email job | SMS job | in-app job
 dedupe key = (event, channel, recipient)
```
**In plain words.** One event, many messengers, and a stamp that stops duplicate letters.

- On events, fan out to email/SMS/in-app via queues (F1); DLT templates for SMS (G4); idempotent per
  (event, channel, recipient) so no duplicates.
- One line: fan out notifications via queues, idempotent per channel, DLT-compliant SMS.

## N9 — Document upload & handling

**Diagram**
```
 client asks -> API checks type/size -> issues SAS URL -> client uploads to Blob
 stored under a generated id (never the user filename)
 scan/validate -> audit access to sensitive docs
```
**In plain words.** The visitor drops the parcel at the loading dock, and you give it your own
label.

- Signed SAS URLs to Azure Blob (G5), validate type/size (D1), map to generated IDs (D7), virus/content
  checks, audit access to sensitive docs.
- One line: upload via signed URLs, validate + ID-map files, audit sensitive-doc access.
