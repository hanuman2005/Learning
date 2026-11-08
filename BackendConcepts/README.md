# BackendGrind — Backend Engineering Revision Pack

A grep-friendly revision reference built from the visual tutoring sessions, modelled on a typical
multi-service stack: **service-A / service-B / service-C** (each owns its DB, behind a gateway),
Node.js + Express (CommonJS), Drizzle ORM + PostgreSQL/Neon, MongoDB, Bull + Redis, Azure Blob,
KYC, SMS and email via third-party providers.

## How to use this
- One file per curriculum section (A–N). Each topic has: **core idea → mechanism / why → real-world
  variations → stack note → one-line summary**.
- Read the one-liners first for a fast pass; drop into a topic when you need the "why".
- Search across everything: `grep -ri "idempoten" BackendGrind/` etc.

## The one spine that runs through everything
> Everything from outside is **untrusted**. Nearly every security bug is **data being treated as
> code**, or **client-supplied identity/privilege being trusted**. The fix is always to **separate
> data from code** and to **derive identity/authority from a source the client cannot control**
> (the signed token, the stored row, the authenticated session).

## Status
| Section | Topic | Depth |
|---|---|---|
| A | Runtime & Node internals | ✅ full |
| B | Express layer | ✅ full |
| C | Authentication & authorization | ✅ full |
| D | Security (OWASP-mapped) | ✅ full |
| E | Data layer | ◻ reference (deepen as taught) |
| F | Async work & jobs | ◻ reference |
| G | Integrations | ◻ reference |
| H | Observability | ◻ reference |
| I | Performance | ◻ reference |
| J | Architecture | ◻ partial (J1–J2 full) |
| K | Testing | ◻ reference |
| L | DevOps / delivery | ◻ reference |
| M | API contract & docs | ◻ reference |
| N | Repo-specific business logic | ◻ reference |
| O | Gaps supplement (comms, concurrency, correctness, jobs, errors, capstone map) | ✅ full |

## Files
- `A-runtime.md`
- `B-express.md`
- `C-auth.md`
- `D-security.md`
- `E-data-layer.md`
- `F-async-work.md`
- `G-integrations.md`
- `H-observability.md`
- `I-performance.md`
- `J-architecture.md`
- `K-testing.md`
- `L-devops.md`
- `M-api-contract.md`
- `N-business-logic.md`
- `O1-communication-styles.md` — WebSockets/SSE, GraphQL, gRPC, outbound webhooks, search, file streaming
- `O2-concurrency-correctness.md` — optimistic/pessimistic locking, races, money, time, state machines
- `O3-config-jobs-errors-capstone.md` — config/flags, graceful shutdown, error taxonomy, queue depth, backpressure, capstone→curriculum map