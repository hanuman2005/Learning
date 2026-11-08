# 🤖 AI Backend Engineering — 30-Day Curriculum Tracker

> **Prerequisite:** Database + Backend + System Design chapters done. This course *reuses* all of it — pgvector needs Postgres, AI serving needs your caching/queues/rate-limiting knowledge, provider-swapping is the Adapter pattern + DIP you already learned.
> **Goal:** Not ML research — **building production AI applications**. Intermediate+ level, portfolio-ready.
> **Rule:** 3 hrs/day = ~2h learn + ~1h build. Language: Python. **Evaluation starts on Day 1 of your first project, not at the end** (the one fix to the roadmap — you can't tell if changes help without it).
> **Landscape note (July 2026):** frameworks churn fast. Learn *concepts* deeply, *specific APIs* loosely. Content reflects current tools but verify versions when you build.

**Progress: Stage 1 ⬜ · Stage 2 ⬜ · Stage 3 ⬜ · Stage 4 ⬜ · Stage 5 ⬜ · Stage 6 ⬜ · Stage 7 ⬜**

---

## Stage 1 — LLM Foundations (Days 1–4)

### Day 1 — How LLMs Actually Work
- [ ] Tokens & tokenization (BPE; why "strawberry" counts r's wrong; token ≠ word)
- [ ] Embeddings (intro) — text as vectors, why similarity = distance
- [ ] Transformers conceptually — attention, next-token prediction (no heavy math)
- [ ] Context window — what it is, KV cache, why long context costs quadratically
- [ ] Why LLMs hallucinate — they predict plausible, not true
- [ ] Temperature, top-p, top-k — sampling controls
- [ ] **Apply:** tokenize text with tiktoken; watch how token count → cost

### Day 2 — Prompt Engineering
- [ ] System vs user vs assistant roles; prompt anatomy
- [ ] Zero-shot, few-shot, chain-of-thought, ReAct pattern
- [ ] Structured output — JSON mode, schema-constrained generation, Pydantic
- [ ] Prompt patterns: delimiters, role-setting, output format specs
- [ ] Anti-patterns & the Dark Side (over-prompting, prompt as a crutch for bad architecture)
- [ ] **Apply:** build a prompt that reliably returns valid JSON for a classification task

### Day 3 — AI APIs & Provider Abstraction
- [ ] OpenAI, Anthropic, Google Gemini API shapes — chat completions, messages
- [ ] Local models — Ollama, vLLM (when & why self-host)
- [ ] Streaming responses (token-by-token), tool/function-calling API surface
- [ ] **Provider-swap architecture** — one interface, N backends (Adapter + DIP from SysDesign Day 2/5)
- [ ] LiteLLM / unified gateways
- [ ] **Apply:** write a `LLMProvider` protocol; implement 2 backends; swap with one line

### Day 4 — Cost & Token Management
- [ ] Pricing models (input vs output tokens; cached-input discounts)
- [ ] Token budgeting, context compression, truncation strategies
- [ ] Model tiering (cheap model first, escalate to strong — cascade)
- [ ] Batching, prompt caching (provider-side)
- [ ] **Apply:** build a token+cost tracker wrapper around your provider interface

---

## Stage 2 — Retrieval & Knowledge Systems (Days 5–9)
*Concepts BEFORE frameworks — the key roadmap decision.*

### Day 5 — Embeddings In Depth
- [ ] Embedding models (choosing; dimensions; multilingual; domain-specific)
- [ ] Similarity metrics — cosine vs dot vs euclidean, when each
- [ ] Embedding pitfalls (asymmetric search, query vs doc embeddings)
- [ ] Cost & caching of embeddings
- [ ] **Apply:** embed 100 sentences; build a brute-force nearest-neighbor search by hand

### Day 6 — Chunking & Document Processing
- [ ] Why chunking makes or breaks RAG
- [ ] Strategies — fixed, recursive, semantic, sentence-window, document-structure-aware
- [ ] Chunk size vs overlap trade-offs; metadata attachment
- [ ] Parsing real docs (PDF, HTML, tables, code) — the messy reality
- [ ] **Apply:** chunk a real PDF 3 ways; compare what retrieval returns

### Day 7 — Vector Databases
- [ ] What they do; HNSW & IVF indexing (concept level)
- [ ] Landscape: **pgvector** (reuse your Postgres!), Qdrant, Pinecone, Weaviate, Milvus — honest trade-offs
- [ ] Metadata filtering (why Qdrant shines); hybrid indexes
- [ ] Scaling reality (the 100M-vector HNSW-tuning wall)
- [ ] **Apply:** load your chunks into pgvector; query with filters

### Day 8 — RAG (The #1 Pattern)
- [ ] Naive RAG pipeline end-to-end (ingest → embed → store → retrieve → augment → generate)
- [ ] Why naive RAG fails (bad chunks, lost context, no citations, wrong-doc confidence)
- [ ] Reranking (cross-encoder; why 2-stage retrieval wins)
- [ ] Citations & grounding; context-window packing
- [ ] Advanced: query rewriting, HyDE, parent-document, contextual retrieval
- [ ] **Apply:** build a working RAG over your SystemDesign notes folder

### Day 9 — Hybrid Search & Retrieval Quality
- [ ] Dense (vector) + sparse (BM25/keyword) — why you need both
- [ ] Fusion (Reciprocal Rank Fusion), rerank the merged set
- [ ] Retrieval evaluation (recall@k, MRR, NDCG) — measure before tuning
- [ ] The Dark Side — when RAG is the wrong tool (fine-tune? long-context? just SQL?)
- [ ] **Apply:** add BM25 + RRF + rerank to Day 8's RAG; measure the lift

---

## Stage 3 — AI Frameworks (Days 10–14)
*Now the abstractions make sense because you know what they wrap.*

### Day 10 — LangChain (and when plain Python wins)
- [ ] Core abstractions — models, prompts, output parsers, LCEL, chains
- [ ] Document loaders, retrievers, the RAG you built — now in LangChain
- [ ] The Dark Side — framework lock-in, debugging opacity, when a for-loop beats a chain
- [ ] **Apply:** re-implement Day 8 RAG in LangChain; compare LOC and debuggability

### Day 11 — LangGraph & Agent Orchestration
- [ ] Why graphs beat linear chains (loops, branches, state) — 70%+ of prod agents use graphs
- [ ] State, nodes, edges, conditional edges; checkpointing & durable state
- [ ] State management = #1 source of agent bugs — handle it deliberately
- [ ] Human-in-the-loop, interrupts, resumption
- [ ] **Apply:** build a branching agent graph with persisted state

### Day 12 — Tool Calling & Function Calling
- [ ] The tool-use loop (model requests → you execute → return → model continues)
- [ ] Tool schema design, validation, error handling back to the model
- [ ] Parallel tool calls; forcing/constraining tool choice
- [ ] Reliability (the model calls wrong/hallucinated tools — guarding against it)
- [ ] **Apply:** give an agent 3 real tools (calculator, web search, your DB query)

### Day 13 — MCP (Model Context Protocol)
- [ ] What & why — the "USB-C for AI tools"; client/server/host model
- [ ] Primitives: tools, resources, prompts (+ async Tasks)
- [ ] Building an MCP server (Python SDK); consuming existing servers
- [ ] Where it fits vs raw function calling; ecosystem reality (10k+ servers)
- [ ] Learn concept deeply, spec loosely (it's still evolving)
- [ ] **Apply:** build a minimal MCP server exposing one tool; connect a client

### Day 14 — AI Memory
- [ ] Short-term (conversation buffer) vs long-term (persisted) memory
- [ ] Conversation history management (windowing, summarization, token limits)
- [ ] Retrieval memory (vector-backed recall of past sessions)
- [ ] Session management, user isolation, memory as a design concern (not a framework feature)
- [ ] **Apply:** add summarizing + retrieval memory to your Day 11 agent

---

## Stage 4 — AI System Engineering (Days 15–18)
*Your System Design chapter's payoff.*

### Day 15 — Streaming & Real-Time Delivery
- [ ] Why streaming (perceived latency, TTFT vs total); SSE vs WebSockets for LLMs
- [ ] Backend streaming architecture (async generators, FastAPI StreamingResponse)
- [ ] Streaming + tool calls + structured output (the hard combination)
- [ ] Cancellation, timeouts, partial-response handling
- [ ] **Apply:** build a streaming chat endpoint with cancellation

### Day 16 — Caching & Cost Optimization for AI
- [ ] Exact-match caching; **semantic caching** (embedding-similarity cache hits)
- [ ] Prompt caching (provider-side, cached-input pricing)
- [ ] Embedding caches; model cascade / routing to cheaper models
- [ ] Reuses SysDesign Day 11 (caching) + Day 21 (cost math)
- [ ] The Dark Side — semantic cache serving subtly-wrong hits
- [ ] **Apply:** add a semantic cache; measure hit rate + cost saved

### Day 17 — Resilience, Rate Limiting & Queues for AI
- [ ] Provider outages & fallback chains (primary → secondary model)
- [ ] Rate limits (theirs and yours) — token bucket from SysDesign Day 21
- [ ] Retries with backoff+jitter (SysDesign Day 18); timeout budgets
- [ ] Async job queues for long generations (SysDesign Day 15)
- [ ] **Apply:** wrap your provider with retries, fallback, and a rate limiter

### Day 18 — Observability for LLM Apps
- [ ] Why: bug-free code can still be "broken" (hallucination, bad retrieval, cost triple)
- [ ] Tracing prompts/completions/tool-calls/retrieval (LangSmith, Langfuse — open source)
- [ ] Metrics: latency (TTFT, p99), token usage, cost per request, quality signals
- [ ] Feedback loops (thumbs up/down → dataset), drift detection
- [ ] **Apply:** instrument your RAG app with Langfuse tracing

---

## Stage 5 — AI Security (Days 19–20)
*Core engineering skill now — prompt injection is OWASP LLM #1.*

### Day 19 — Prompt Injection & Jailbreaks
- [ ] The core problem: models can't separate instructions from data
- [ ] Direct vs indirect injection (poisoned docs/webpages/tool outputs)
- [ ] Why it's an ARCHITECTURE problem, not a prompt problem
- [ ] Defenses: least-privilege tools, human approval for high-risk actions, input/output filtering, dual-LLM patterns
- [ ] Jailbreaks & why "add guardrail text" alone fails
- [ ] **Apply:** attack your own Day 12 tool agent; then harden it

### Day 20 — Data Protection, PII & Guardrails
- [ ] Data leakage paths (logs, prompts, training, tool outputs)
- [ ] PII detection & redaction before it hits the model/logs (never log secrets — SysDesign Day 20)
- [ ] Output validation & content filtering (Llama Guard, LLM-Guard, schema validation)
- [ ] Least-privilege data access; tenant isolation in RAG (don't retrieve other users' docs!)
- [ ] The Dark Side — guardrail latency/UX cost, security theater vs real wins
- [ ] **Apply:** add PII redaction + output validation to your pipeline

---

## Stage 6 — AI Evaluation (Days 21–22)
*The skill most roadmaps skip. Start evals with your FIRST project, deepen here.*

### Day 21 — LLM Evaluation Fundamentals
- [ ] Why "it looks good" fails; offline vs online eval
- [ ] Building an eval dataset (golden set); regression tests as unit tests
- [ ] Metrics: exact match, semantic similarity, RAG metrics (RAGAS: faithfulness, relevance)
- [ ] LLM-as-a-judge (and its biases — position, verbosity, self-preference)
- [ ] **Apply:** build a 20-case eval suite for your RAG; run it in CI mentally

### Day 22 — Prompt Versioning, A/B Testing & Human Eval
- [ ] Prompt versioning & management (treat prompts like code)
- [ ] A/B testing prompts/models in production; online metrics
- [ ] Human evaluation (annotation queues, when humans are non-negotiable)
- [ ] Closing the loop: production feedback → eval set → improvement
- [ ] **Apply:** version two prompt variants; evaluate both against your golden set

---

## Stage 7 — Production Projects (Days 23–30)
*Progressive — each reuses the last. This is where you become hireable.*

### Day 23 — Project 1: AI Chatbot
- [ ] Streaming chat, conversation memory, provider abstraction, cost tracking
- [ ] **+ Set up your eval harness now (the fix)** — you'll reuse it in every project

### Day 24 — Project 2: PDF RAG System
- [ ] Upload → chunk → embed → pgvector → hybrid retrieve → rerank → cited answer
- [ ] Eval retrieval quality + answer faithfulness

### Day 25 — Project 3: Tool-Calling Agent
- [ ] Agent with real tools, error handling, guardrails, injection hardening

### Day 26 — Project 4: Multi-Agent Research Assistant
- [ ] LangGraph orchestration, planner + worker agents, shared memory, observability

### Day 27 — Project 5: AI Customer Support System
- [ ] RAG over docs + escalation tools + PII redaction + evals + fallback models (integrates everything)

### Day 28 — Project 6: AI Supplier Onboarding Assistant
- [ ] Domain project (inspired by your internship) — structured extraction, validation, human-in-the-loop

### Day 29 — Project 7: AI Code Review Assistant
- [ ] Diff analysis, tool use, structured findings, guardrails against injected code comments

### Day 30 — AI System Design Interview Day
- [ ] Framework for AI design questions (differs from classic SysDesign)
- [ ] Worked: "Design ChatGPT", "Design a document-Q&A system at scale"
- [ ] Estimation for AI (tokens/sec, GPU cost, vector store sizing)
- [ ] Self-review + gap analysis + "beyond" roadmap (fine-tuning, multimodal, RLHF concepts)

---

## 📁 File Index — all 30 days created ✅

| Day | File | Studied? |
|---|---|---|
| 1 | `day-01-how-llms-work.md` | ⬜ |
| 2 | `day-02-prompt-engineering.md` | ⬜ |
| 3 | `day-03-ai-apis-provider-abstraction.md` | ⬜ |
| 4 | `day-04-cost-token-management.md` | ⬜ |
| 5 | `day-05-embeddings-deep.md` | ⬜ |
| 6 | `day-06-chunking-document-processing.md` | ⬜ |
| 7 | `day-07-vector-databases.md` | ⬜ |
| 8 | `day-08-rag.md` | ⬜ |
| 9 | `day-09-hybrid-search-retrieval-quality.md` | ⬜ |
| 10 | `day-10-langchain.md` | ⬜ |
| 11 | `day-11-langgraph-orchestration.md` | ⬜ |
| 12 | `day-12-tool-calling.md` | ⬜ |
| 13 | `day-13-mcp.md` | ⬜ |
| 14 | `day-14-ai-memory.md` | ⬜ |
| 15 | `day-15-streaming-realtime.md` | ⬜ |
| 16 | `day-16-caching-cost-optimization.md` | ⬜ |
| 17 | `day-17-resilience-rate-limiting-queues.md` | ⬜ |
| 18 | `day-18-observability-llm-apps.md` | ⬜ |
| 19 | `day-19-prompt-injection-jailbreaks.md` | ⬜ |
| 20 | `day-20-data-protection-pii-guardrails.md` | ⬜ |
| 21 | `day-21-llm-evaluation-fundamentals.md` | ⬜ |
| 22 | `day-22-prompt-versioning-ab-testing.md` | ⬜ |
| 23 | `day-23-project-ai-chatbot.md` | ⬜ |
| 24 | `day-24-project-pdf-rag.md` | ⬜ |
| 25 | `day-25-project-tool-calling-agent.md` | ⬜ |
| 26 | `day-26-project-multi-agent-research.md` | ⬜ |
| 27 | `day-27-project-ai-customer-support.md` | ⬜ |
| 28 | `day-28-project-supplier-onboarding.md` | ⬜ |
| 29 | `day-29-project-ai-code-review.md` | ⬜ |
| 30 | `day-30-ai-system-design-interview.md` | ⬜ |

---

## Topics deliberately kept LIGHT (know they exist, don't rabbit-hole)
- **Fine-tuning / LoRA** — covered as a *decision* (fine-tune vs RAG vs prompt) on Day 30; most teams need it far less than they think.
- **Training / pre-training internals** — not a backend engineer's job; conceptual only.
- **Multimodal (vision/audio)** — introduced Day 30; deepen only if a job needs it.
- **Cloud/GPU ops** — your (optional) Cloud chapter's territory; touched in Stage 4.

## Daily Ritual
- [ ] End each session: explain today's concept out loud in 2 min (Feynman test)
- [ ] Every project day: run your eval suite before calling it "done"
- [ ] Commit to git: `git add . && git commit -m "ai-day XX: <topic>"`
