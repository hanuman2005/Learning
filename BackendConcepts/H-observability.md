# Section H — Observability

> Reference outline.

## H1 — Structured logging (winston)

**Diagram**
```
 BAD : "user 5 failed login for a@b.com pwd=abc"
 GOOD: {"level":"warn","msg":"login_failed","userId":5,"reqId":"r-91"}
       redact PII/secrets -> ship to central store -> search
```
**In plain words.** Logs are your black-box flight recorder. Make them machine-searchable and
free of secrets.

- Log **structured JSON** (not string soup) with levels; **redact PII/secrets** (D14/D11). Ship to a
  central log store. Logs answer "what happened".
- One line: structured, level-based, PII-redacted logs shipped centrally.

## H2 — Correlation / request IDs

**Diagram**
```
 client -> central [req-id: r-91] -> buyer [r-91] -> warehouse [r-91]
 search r-91 in logs -> the whole journey in order
```
**In plain words.** One tracking number follows the parcel through every hand.

- Attach a **request id** at the edge and propagate it through logs and **across services** (central →
  buyer → warehouse) so one request's whole path is traceable.
- One line: tag every request with an id and propagate it across services for end-to-end tracing.

## H3 — Metrics (RED / USE)

**Diagram**
```
 RED (per service): Rate (req/s)  Errors (%)  Duration (ms)
 USE (per resource): Utilization  Saturation  Errors
   e.g. DB pool: 9/10 in use (utilization), 20 waiting (saturation)
```
**In plain words.** The dashboard gauges of a car: speed, engine trouble, and fuel level.

- **RED** (per service): Rate, Errors, Duration. **USE** (per resource): Utilization, Saturation,
  Errors. Metrics answer "how much / how healthy".
- One line: track RED for services and USE for resources.

## H4 — Distributed tracing (OpenTelemetry)

**Diagram**
```
 request r-91 total 900 ms
 |-- central       40 ms
 |-- buyer        120 ms
 |-- warehouse    700 ms  <-- the slow hop
```
**In plain words.** A stopwatch on each leg of the relay race, so you see which runner is slow.

- Spans across services show **where time goes** in a multi-service request. OpenTelemetry is the
  vendor-neutral standard.
- One line: use OpenTelemetry spans to see latency across the service-A / service-B / service-C hop.

## H5 — Health checks

**Diagram**
```
 /health/live  : process is up?          no -> restart it
 /health/ready : DB + Redis reachable?   no -> LB stops sending traffic
```
**In plain words.** Live = "is he breathing?". Ready = "is he able to work right now?".

- **Liveness** (is the process up?) vs **readiness** (can it serve — DB/Redis reachable?). Load
  balancers/orchestrators use these to route and restart.
- One line: expose liveness and readiness endpoints for the orchestrator.

## H6 — SLO / alerting

**Diagram**
```
 SLO: 99.9% of reads < 300ms over 30 days
 error budget = the 0.1% you may fail
 alert when budget burns too fast (symptom), not on every blip
```
**In plain words.** Decide what "good enough" is, then wake someone only when you are really
heading below it.

- Define **SLOs** (e.g. 99.9% of reads < 300ms), alert on **symptoms** (error budget burn), not noise.
- One line: set SLOs and alert on error-budget burn, not every blip.
