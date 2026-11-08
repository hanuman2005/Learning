# AI Engineer Interview Question Bank

A practice bank for production AI backend interviews. It follows the 30 day files in this folder. Each answer points back to the day file where you can revise.

## How to use this file

- Read the question. Say your answer out loud before you read the model answer.
- Model answers are short on purpose. In a real interview, add one example from your own project.
- Tags show the level the question usually targets: [Fresher], [Mid], [Senior].
- Use "Revise:" to jump to the day file when an answer feels weak.
- This file makes no promise about any job result. It is a study aid.

## Important note: this field changes fast

- Model names, prices, context limits, rate limits and API shapes change often.
- Never quote a specific model name, price or limit in an interview as a fixed fact.
- Say "check the current provider docs" and explain how you would compute the cost. This shows good judgement.
- The ideas in this file (tokens, retrieval, evaluation, guardrails) change much more slowly than product details.
- Numbers in the answers below are examples to show the method. They are not current prices.

## Contents

1. Top 15 must-know questions
2. Question bank by topic (100 questions)
3. Eight AI system design prompts
4. Common mistakes in AI engineering interviews

---

# Part 1. Top 15 must-know questions

If you only have one evening, learn these. Each has a short answer. The full bank below goes deeper.

**T1. What is a token, and why does it matter?**
A token is a small piece of text the model reads and writes. Cost, speed and context limits are all counted in tokens. Always estimate tokens before you design anything.
Revise: day-01-how-llms-work.md, day-04-cost-token-management.md

**T2. Why do LLMs hallucinate?**
The model predicts the next likely token. It does not look up facts. When it lacks knowledge, it still produces fluent text. Lower temperature does not fix this. Grounding with retrieved sources helps.
Revise: day-01-how-llms-work.md

**T3. What is RAG and when do you use it?**
RAG (Retrieval-Augmented Generation) means: find relevant text first, then give it to the model with the question. Use it for private, fresh or large knowledge. It is the default way to add knowledge.
Revise: day-08-rag.md

**T4. RAG vs fine-tuning?**
RAG adds facts. Fine-tuning changes behaviour or format. Do not fine-tune to inject knowledge. It is unreliable and hard to update.
Revise: day-08-rag.md, day-30-ai-system-design-interview.md

**T5. How do you debug a bad RAG answer?**
Check retrieval first: were the right chunks returned? If not, fix chunking, embeddings, hybrid search or reranking. If yes, fix the prompt, context packing or model. Measure both separately.
Revise: day-08-rag.md, day-09-hybrid-search-retrieval-quality.md

**T6. Why hybrid search?**
Dense (vector) search finds meaning. Sparse (keyword, BM25) search finds exact terms like error codes. Combine them with a fusion method such as RRF, then rerank.
Revise: day-09-hybrid-search-retrieval-quality.md

**T7. How do you cut LLM cost without hurting quality?**
Measure first. Then use caching, model routing, shorter prompts, output limits and batching. Check quality with an eval set after every change.
Revise: day-04-cost-token-management.md, day-16-caching-cost-optimization.md

**T8. How does tool calling work?**
The model does not run tools. It returns a request with a tool name and arguments. Your code validates the arguments, runs the tool, and sends the result back. The loop repeats until the model answers.
Revise: day-12-tool-calling.md

**T9. What is prompt injection and how do you defend?**
Attacker text in the input or in retrieved data tries to override your instructions. There is no perfect prompt-only fix. Use layers: least privilege, treat data as untrusted, validate outputs, human approval for risky actions.
Revise: day-19-prompt-injection-jailbreaks.md

**T10. How do you evaluate an LLM app?**
Build a golden set from real traces. Run it offline in CI. Use deterministic checks first, then an LLM judge with known biases controlled. Sample live traffic online.
Revise: day-21-llm-evaluation-fundamentals.md

**T11. How do you make an LLM app reliable?**
Timeouts, retries with backoff and jitter, fallback models, circuit breakers, rate limits and queues for long jobs. Plan for provider failure.
Revise: day-17-resilience-rate-limiting-queues.md

**T12. Why stream responses?**
Streaming lowers time to first token, so the app feels fast. It makes cancellation, error handling and output guardrails harder. SSE is the usual choice.
Revise: day-15-streaming-realtime.md

**T13. How do you observe an LLM app?**
Trace every step (retrieval, tool calls, generation). Track tokens, cost, latency and quality signals per request. Log carefully, because logs can leak PII.
Revise: day-18-observability-llm-apps.md

**T14. How do you handle PII?**
Detect and redact before the model and before logs. Use tenant isolation in retrieval. Plan deletion across vector stores, caches and memory.
Revise: day-20-data-protection-pii-guardrails.md

**T15. How do you answer an AI system design question?**
Clarify, estimate, design data and retrieval, choose model and serving, cover AI-specific concerns (eval, guardrails, observability, cost), then scale and failure modes.
Revise: day-30-ai-system-design-interview.md

---

# Part 2. Question bank by topic

## A. How LLMs work (Day 1)

**Q1. [Fresher] What is a token? How is it different from a word?**
A token is a chunk of text, often a word piece. One word can be one or several tokens. Different models use different tokenizers (programs that split text), so token counts differ by model.
Token count controls cost and the context limit. Non-English text and code often use more tokens.
Revise: day-01-how-llms-work.md

**Q2. [Fresher] Explain next-token prediction in simple words.**
The model reads all tokens so far. It gives a probability for each possible next token. One token is picked, added to the text, and the loop repeats.
This is why output is generated one piece at a time and why long outputs take longer.
Revise: day-01-how-llms-work.md

**Q3. [Fresher] What is the context window?**
It is the maximum number of tokens the model can handle in one request. Input and output both count.
When you go over it, you must cut, summarise or retrieve less. Check the current limit for your model in provider docs.
Revise: day-01-how-llms-work.md

**Q4. [Mid] What do temperature and top_p do? What values do you use for extraction?**
Temperature changes how sharp the probability distribution is. Low values are focused. High values are more varied. Top_p keeps only the most likely tokens up to a probability total.
For extraction and classification, use low temperature. Usually tune temperature or top_p, not both.
Low temperature reduces randomness. It does not remove wrong answers.
Revise: day-01-how-llms-work.md

**Q5. [Mid] Why do models hallucinate? Does temperature 0 fix it?**
The model optimises for likely text, not true text. It has no built-in "I do not know" check. Gaps in training data lead to confident guesses.
Temperature 0 makes the output repeatable, not correct. A repeatable wrong answer is still wrong.
Fixes: grounding in retrieved sources, citations, refusal rules, evaluation.
Revise: day-01-how-llms-work.md

**Q6. [Mid] What is the KV cache and why does it matter to a backend engineer?**
The KV cache (key-value cache) stores attention results for earlier tokens, so the model does not recompute them for each new token. It speeds up generation but uses GPU memory.
Long contexts and many parallel users use more memory. This affects self-hosting capacity and also explains why long prompts are slower and costlier.
Revise: day-01-how-llms-work.md

## B. Prompt engineering (Day 2)

**Q7. [Fresher] What are message roles?**
Roles separate who is speaking: system (rules and behaviour), user (the request), assistant (model replies). Some APIs add tool roles.
Put stable rules in the system message. Put user input in the user message. This also helps with caching and safety.
Revise: day-02-prompt-engineering.md

**Q8. [Fresher] Zero-shot vs few-shot prompting?**
Zero-shot gives only instructions. Few-shot adds a few examples of input and expected output.
Few-shot helps with format and edge cases. Choose examples that are varied and correct. Examples cost tokens on every call.
Revise: day-02-prompt-engineering.md

**Q9. [Mid] How do you get reliable structured output?**
Ask for a schema, use the provider's structured output or tool feature if available, then validate the result in code (for example with a schema library). On failure, retry with the error message or fall back.
Never trust that the output is valid JSON. Always parse and validate.
Revise: day-02-prompt-engineering.md

**Q10. [Mid] What makes a prompt robust?**
Clear task, clear format, delimiters around user data, examples for edge cases, and a rule for "not enough information". Test it on a varied eval set, not on three happy examples.
Small wording changes can change results, so version and test every change.
Revise: day-02-prompt-engineering.md, day-22-prompt-versioning-ab-testing.md

**Q11. [Mid] A prompt works on one model but fails on another. Why, and what do you do?**
Models differ in training, instruction following and format habits. A prompt tuned for one model may rely on its quirks.
Keep prompts simple and explicit, keep a per-model prompt variant if needed, and run the same eval set on each model before switching.
Revise: day-02-prompt-engineering.md, day-03-ai-apis-provider-abstraction.md

**Q12. [Senior] Should you use chain-of-thought (step-by-step reasoning) in production?**
It can improve hard reasoning tasks, but it adds tokens, cost and latency. Some models reason internally already. Do not expose raw reasoning to users without thought, because it may contain wrong or sensitive text.
Test on your eval set. Use it only where it measurably helps. Check current provider guidance, as reasoning features change.
Revise: day-02-prompt-engineering.md

## C. AI APIs and provider abstraction (Day 3)

**Q13. [Fresher] What is a provider abstraction layer?**
It is your own thin interface (for example `generate(messages, options)`) that hides each provider's API details. Your app code calls it, and adapters talk to each provider.
Benefits: you can switch or add providers, add fallbacks, and track cost in one place.
Revise: day-03-ai-apis-provider-abstraction.md

**Q14. [Mid] What goes into a good abstraction layer, and what should it not hide?**
Include: unified message format, streaming, tool calls, error mapping, retries, usage and cost data, timeouts. Do not hide important differences, such as token limits or features only one provider has.
Avoid a "lowest common denominator" that blocks useful features.
Revise: day-03-ai-apis-provider-abstraction.md

**Q15. [Mid] Build your own abstraction or use a gateway library?**
A library gives speed and many providers. Your own layer gives control and fewer dependencies. A common path: start with a small own interface, and use a library behind it if it fits.
Judge by maintenance cost, feature gaps and how well it surfaces errors.
Revise: day-03-ai-apis-provider-abstraction.md

**Q16. [Mid] When would you use a local model (Ollama, vLLM)?**
For data that cannot leave your network, for very high steady volume, for offline use, or for development and tests without API cost.
Costs: hardware, operations, and often lower quality than top hosted models. Compute break-even before you decide.
Revise: day-03-ai-apis-provider-abstraction.md

**Q17. [Senior] How do you move to a new model version safely?**
Pin versions. Run your golden set on the new model. Compare quality, cost and latency. Roll out with a small traffic share (canary) and watch online metrics. Keep a rollback.
Providers retire models, so plan for this on a schedule. Check deprecation notices in provider docs.
Revise: day-03-ai-apis-provider-abstraction.md, day-22-prompt-versioning-ab-testing.md

## D. Cost and token management (Day 4)

**Q18. [Fresher] How is LLM cost calculated?**
Usually: input tokens times input price plus output tokens times output price. Output tokens are often priced higher than input tokens. Check current prices in provider docs.
Count tokens before you send, and record usage returned by the API.
Revise: day-04-cost-token-management.md

**Q19. [Mid] How do you estimate the monthly cost of a feature?**
Estimate requests per day, average input tokens (system prompt, history, retrieved text, question) and output tokens. Multiply by prices. Add retries, embeddings and a safety margin.
Show the formula in the interview. The method matters more than exact numbers.
Revise: day-04-cost-token-management.md

**Q20. [Mid] What is model routing (cascade)?**
Send easy requests to a cheaper model. Send hard ones, or low-confidence cases, to a stronger model.
You need a way to decide: a classifier, simple rules, or "try cheap first, escalate on low confidence". Validate with evals so quality does not drop.
Revise: day-04-cost-token-management.md, day-16-caching-cost-optimization.md

**Q21. [Mid] How do you reduce context size without losing quality?**
Retrieve fewer but better chunks (rerank), trim chat history, summarise old turns, remove duplicate text, and shorten the system prompt.
Test each cut on the eval set. Cutting blindly can remove the one fact needed.
Revise: day-04-cost-token-management.md

**Q22. [Senior] Scenario: Cut LLM cost by 60% without hurting quality. What do you do?**
1. Measure: cost per feature, per tenant, per step. Find the top spenders.
2. Build or confirm an eval set so you can detect quality loss.
3. Cheap wins: provider prompt caching for the stable prefix, exact and semantic response caching, output length limits.
4. Route easy traffic to a cheaper model.
5. Cut context: better reranking, history trimming.
6. Batch non-urgent work.
7. Re-run evals after each step and ship one change at a time.
Say clearly that 60% is a target to test, not a promise.
Revise: day-04-cost-token-management.md, day-16-caching-cost-optimization.md

**Q23. [Senior] How do you stop one user or bug from burning the budget?**
Per-user and per-tenant token budgets, hard caps on agent loop steps, max output tokens, alerts on spend spikes, and a cost tracker wrapper that records every call.
Fail safely: reject or degrade when a budget is reached.
Revise: day-04-cost-token-management.md, day-17-resilience-rate-limiting-queues.md

## E. Embeddings (Day 5)

**Q24. [Fresher] What is an embedding?**
An embedding is a list of numbers (a vector) that represents the meaning of text. Similar meanings give vectors that are close together.
It lets you search by meaning, not only by exact words.
Revise: day-05-embeddings-deep.md

**Q25. [Fresher] Name common similarity metrics.**
Cosine similarity (angle between vectors), dot product, and Euclidean distance. For normalised vectors, cosine and dot product give the same ranking.
Use the metric the embedding model recommends.
Revise: day-05-embeddings-deep.md

**Q26. [Mid] How do you choose an embedding model?**
Check quality on your own data (a small retrieval test), language coverage, dimensions (storage and speed), max input length, cost, and whether you can self-host. Public leaderboards are a start, not the answer.
Revise: day-05-embeddings-deep.md

**Q27. [Mid] What is asymmetric search?**
Queries are short and documents are long, so they look different. Some models use different prefixes or modes for queries and documents.
Using the wrong mode lowers quality. Read the model's docs.
Revise: day-05-embeddings-deep.md

**Q28. [Mid] You change the embedding model. What happens?**
Vectors from different models are not comparable. You must re-embed all documents and rebuild the index. Plan a new index, test it, then switch traffic.
Store the model name and version with each vector. Cache embeddings by content hash to save cost.
Revise: day-05-embeddings-deep.md

## F. Chunking (Day 6)

**Q29. [Fresher] Why do we chunk documents?**
Models and embeddings have size limits, and one vector for a whole book is too vague. Small focused chunks give precise retrieval and use fewer tokens.
Revise: day-06-chunking-document-processing.md

**Q30. [Mid] Compare chunking strategies.**
Fixed size: simple but may cut ideas. Recursive (split by paragraph, then sentence): good default. Structure-aware (by headings, code blocks, tables): best for structured docs. Semantic: split by meaning change, costs more.
Choose by document type and measure recall.
Revise: day-06-chunking-document-processing.md

**Q31. [Mid] What is chunk overlap and why use it?**
Overlap repeats the end of one chunk at the start of the next. It protects facts that fall on a boundary.
Too much overlap wastes storage and returns duplicates. Sweep a few sizes and overlaps and measure recall@k.
Revise: day-06-chunking-document-processing.md

**Q32. [Mid] What metadata should each chunk have?**
Source document, page or section, title, date, version, tenant or access level, and chunk position. Metadata supports filtering, citations, freshness and permission checks.
Revise: day-06-chunking-document-processing.md

**Q33. [Senior] Real PDFs are messy (tables, scans, columns). What is your plan?**
Use a parser that keeps layout and tables, run OCR (optical character recognition) for scans, clean headers and footers, and keep tables as whole units. Review a sample by eye.
Track parse quality as a metric. Bad parsing cannot be fixed later by a better model.
Revise: day-06-chunking-document-processing.md, day-24-project-pdf-rag.md

## G. Vector databases (Day 7)

**Q34. [Fresher] Why use a vector database instead of a normal loop over all vectors?**
Comparing a query with every vector is too slow at scale. A vector database uses an index for approximate nearest neighbour (ANN) search, which is much faster.
Revise: day-07-vector-databases.md

**Q35. [Mid] Explain the ANN trade-off.**
ANN indexes (such as HNSW, a graph-based index) trade a small loss in recall for big speed gains. Parameters let you tune recall, latency and memory.
Measure recall against exact search on your data to choose settings.
Revise: day-07-vector-databases.md

**Q36. [Mid] When is pgvector enough, and when do you need a dedicated vector database?**
pgvector is often enough for small to medium data, when you want one database, SQL joins and transactions. Dedicated systems help at very large scale, with heavy filtering needs or special features.
Decide by data size, query rate and team skills. Test with your own data.
Revise: day-07-vector-databases.md

**Q37. [Mid] What problems come with metadata filtering?**
Filtering after search can return too few results. Filtering before search can slow or hurt ANN quality, depending on the engine.
Test filtered queries for recall and speed. Permission filters must be correct every time.
Revise: day-07-vector-databases.md

**Q38. [Senior] How do you scale a vector store to hundreds of millions of vectors?**
Estimate memory first: vectors times dimensions times bytes, plus index overhead. Options: smaller dimensions, quantisation (storing numbers with fewer bits), sharding, disk-based indexes, tenant partitioning.
Plan re-indexing, backups and cost. Accept a small recall loss only if evals show it is fine.
Revise: day-07-vector-databases.md, day-30-ai-system-design-interview.md

## H. RAG (Day 8)

**Q39. [Fresher] Describe the basic RAG pipeline.**
Ingest: parse, chunk, embed, store. Query: embed the question, retrieve top chunks, put them in the prompt, generate an answer with citations.
Revise: day-08-rag.md

**Q40. [Mid] Why does naive RAG fail?**
Wrong chunks retrieved (bad chunking, vague queries), right chunk ranked too low, too much noise in context, model ignores context, or the answer is not in the data at all.
Each failure has a different fix, so measure each stage.
Revise: day-08-rag.md

**Q41. [Mid] What does a reranker do?**
Retrieval returns candidates quickly. A reranker (often a cross-encoder, which reads query and chunk together) scores them more precisely and reorders them.
Typical flow: retrieve a larger set, rerank, keep the best few. It adds latency and cost.
Revise: day-08-rag.md, day-09-hybrid-search-retrieval-quality.md

**Q42. [Mid] How do you make RAG answers trustworthy?**
Ask for answers only from the context, require citations that point to chunk IDs, check that citations exist, and allow "I do not know" when context is weak.
Consider an automatic faithfulness check on samples.
Revise: day-08-rag.md

**Q43. [Mid] How do you pack context?**
Order chunks by relevance, remove duplicates, add source labels, and stay within a token budget. Very long contexts can hide facts in the middle, so keep it focused.
Revise: day-08-rag.md

**Q44. [Senior] Scenario: RAG answers are wrong. How do you debug retrieval vs generation?**
1. Take failing cases and log the question, retrieved chunks and final answer.
2. Ask: does the correct information exist in the corpus? If not, it is a data gap.
3. If yes, was it in the top-k retrieved? If not, it is a retrieval problem: check chunking, embeddings, hybrid search, reranking, filters, query rewriting.
4. If it was retrieved, was it in the prompt? Check context packing and truncation.
5. If it was in the prompt and the answer is still wrong, it is a generation problem: improve the prompt, lower noise, test another model.
6. Measure retrieval (recall@k) and generation (faithfulness) separately, with a labelled set.
Revise: day-08-rag.md, day-09-hybrid-search-retrieval-quality.md, day-21-llm-evaluation-fundamentals.md

**Q45. [Senior] Name advanced RAG techniques and when each helps.**
Query rewriting (vague or follow-up questions), multi-query (recall), hypothetical document embedding (short queries), parent-child chunks (precision plus context), metadata routing (many sources).
Each adds cost and complexity. Add one only when evals show a specific failure it fixes.
Revise: day-08-rag.md

## I. Hybrid search and retrieval quality (Day 9)

**Q46. [Fresher] Dense vs sparse retrieval?**
Dense uses embeddings and finds similar meaning. Sparse (BM25, a keyword scoring method) matches exact words.
Dense can miss codes and names. Sparse can miss paraphrases. Hybrid uses both.
Revise: day-09-hybrid-search-retrieval-quality.md

**Q47. [Mid] What is Reciprocal Rank Fusion (RRF)?**
RRF combines ranked lists by rank, not by score. Each document gets a score from its position in each list, and scores are added.
It is simple and works because dense and sparse scores are on different scales.
Revise: day-09-hybrid-search-retrieval-quality.md

**Q48. [Mid] Which metrics measure retrieval?**
Recall@k (was the right chunk in the top k), precision@k, MRR (mean reciprocal rank, how high the first right result is), and nDCG (quality of the whole ranking).
Recall@k matters most for the retrieval stage. Precision matters after reranking.
Revise: day-09-hybrid-search-retrieval-quality.md

**Q49. [Mid] How do you build a retrieval eval set?**
Collect real questions, mark which chunks or documents answer each, and keep it versioned. You can also generate questions from chunks with an LLM, but review them by hand.
Start small (dozens of cases), then grow from production failures.
Revise: day-09-hybrid-search-retrieval-quality.md

**Q50. [Senior] Users search for error codes and exact IDs, and results are poor. Fix?**
Dense search blurs exact tokens. Add keyword search (BM25), fuse with RRF, and consider exact-match boosts for ID patterns. Then rerank.
Prove it with recall@k on a set of exact-token queries before and after.
Revise: day-09-hybrid-search-retrieval-quality.md

## J. LangChain and LangGraph (Days 10-11)

**Q51. [Fresher] What is LangChain used for?**
It is a library with building blocks: model wrappers, prompts, retrievers, output parsers, and a pipe syntax (LCEL) to join them.
It saves time for common patterns but adds a layer you must understand.
Revise: day-10-langchain.md

**Q52. [Mid] When should you use plain Python instead of LangChain?**
When the flow is simple, when you need full control and clear debugging, or when you want fewer dependencies. A simple RAG is short in plain code.
Use the library when its parts save real work, such as many integrations.
Revise: day-10-langchain.md

**Q53. [Mid] Why use a graph (LangGraph) instead of a linear chain?**
Agents need loops, branches and retries. A graph has state, nodes and edges, with conditional edges for decisions.
It also supports checkpointing (saving state) and pausing for human approval.
Revise: day-11-langgraph-orchestration.md

**Q54. [Mid] Why is state management the top source of agent bugs?**
State is shared by many nodes. Bad updates overwrite data, lists grow without limit, and loops never end. Typed state and reducer functions (rules for merging updates) make changes predictable.
Log state at each step.
Revise: day-11-langgraph-orchestration.md

**Q55. [Senior] How do you add human-in-the-loop to an agent?**
Pause the graph before a risky step using an interrupt, save state with a checkpoint, show the reviewer the proposed action, and resume with approve or reject.
Make the pause durable (stored, not in memory) so it survives restarts.
Revise: day-11-langgraph-orchestration.md, day-27-project-ai-customer-support.md

## K. Tool calling (Day 12)

**Q56. [Fresher] How does tool calling work?**
You describe tools with names, descriptions and a schema. The model may reply with a tool request. Your code runs the tool and sends the result back. The model then continues.
The model never runs code itself.
Revise: day-12-tool-calling.md

**Q57. [Mid] What makes a good tool schema?**
Clear name, a description that says when to use it, few and well-typed parameters, enums where possible, and examples in descriptions.
Vague tools lead to wrong calls. Fewer tools usually means better choices.
Revise: day-12-tool-calling.md

**Q58. [Mid] Why validate model-supplied arguments?**
The model can produce wrong types, missing fields or malicious values. Validate with a schema, check permissions in code, and never build SQL or shell commands from raw arguments.
Treat arguments like untrusted user input.
Revise: day-12-tool-calling.md

**Q59. [Mid] How should a tool report errors?**
Return a clear, short error message as the tool result, so the model can fix its call or tell the user. Do not return stack traces or secrets.
Revise: day-12-tool-calling.md

**Q60. [Senior] What are idempotency and loop guards in agents?**
Idempotent means repeating an action has the same effect as doing it once. Use idempotency keys for actions like refunds or emails, because retries and model repeats happen.
Loop guards limit steps, total tokens and repeated identical calls.
Revise: day-12-tool-calling.md, day-25-project-tool-calling-agent.md

**Q61. [Senior] Scenario: Defend against prompt injection in a tool-using agent.**
1. Assume the model can be tricked. Limit what damage a trick can do.
2. Least privilege: give each tool the smallest scope, read-only where possible, per-user credentials.
3. Mark retrieved and tool content as untrusted data, in clear delimiters.
4. Validate tool arguments in code and check authorisation outside the model.
5. Human approval for high-impact actions (payments, deletes, sending data out).
6. Block data exfiltration paths: restrict outbound URLs, links and images.
7. Filter outputs, log all tool calls, and red-team with known attack cases in CI.
No single step is enough. Use layers.
Revise: day-19-prompt-injection-jailbreaks.md, day-12-tool-calling.md, day-25-project-tool-calling-agent.md

## L. MCP (Day 13)

**Q62. [Fresher] What is MCP?**
MCP (Model Context Protocol) is an open standard for connecting AI apps to tools and data sources. A host app uses a client to talk to MCP servers that expose capabilities.
Revise: day-13-mcp.md

**Q63. [Mid] Name the MCP primitives.**
Tools (actions the model can call), resources (data to read), prompts (reusable templates), and, in newer versions, tasks for long operations. Check the current spec for details, as it evolves.
Revise: day-13-mcp.md

**Q64. [Mid] MCP vs raw function calling?**
Function calling is the model feature for requesting tools in one app. MCP is a shared protocol so many apps can reuse the same tool servers.
MCP helps with reuse and ecosystem. It adds a moving part and a security surface.
Revise: day-13-mcp.md

**Q65. [Senior] What are the security risks of MCP servers?**
Untrusted servers can return malicious instructions (injection), tool descriptions can be misleading, and servers may have too much access. Use trusted sources, least privilege, authentication, user approval for risky tools, and logging.
Revise: day-13-mcp.md, day-19-prompt-injection-jailbreaks.md

## M. Memory (Day 14)

**Q66. [Fresher] Short-term vs long-term memory in chat apps?**
Short-term is the current conversation sent in the prompt. Long-term is stored facts or history, retrieved when needed.
Revise: day-14-ai-memory.md

**Q67. [Mid] How do you manage long conversation history?**
Keep recent turns, summarise older turns, and retrieve older facts only when relevant. Always set a token budget.
Summaries can lose details, so keep key facts in a separate structured store.
Revise: day-14-ai-memory.md

**Q68. [Mid] How do you isolate memory per user?**
Key every memory record by user and tenant, filter in every query, and test for cross-user leaks. Never rely on the prompt to separate users.
Revise: day-14-ai-memory.md, day-20-data-protection-pii-guardrails.md

**Q69. [Senior] What risks come with long-term memory?**
Wrong or outdated facts stored forever, privacy and deletion duties, memory poisoning (an attacker plants false facts), and growing cost.
Add expiry, user control to view and delete, and checks before writing memory.
Revise: day-14-ai-memory.md

## N. Streaming (Day 15)

**Q70. [Fresher] Why stream LLM responses?**
Users see text start sooner (lower time to first token), so the app feels faster even if total time is the same.
Revise: day-15-streaming-realtime.md

**Q71. [Mid] SSE or WebSockets?**
SSE (server-sent events) is one-way server to client over HTTP and is simple and proxy-friendly. WebSockets are two-way and fit voice or interactive cases.
For text generation, SSE is usually enough.
Revise: day-15-streaming-realtime.md

**Q72. [Mid] How do you handle cancellation?**
When the client disconnects or presses stop, cancel the upstream provider call so you stop paying for tokens. Clean up resources and record partial usage.
Revise: day-15-streaming-realtime.md

**Q73. [Senior] What is hard about streaming plus guardrails or tool calls?**
Output checks need complete text, but you already sent partial text. Options: buffer small windows, check input strictly, stream only low-risk content, or retract on violation.
Tool calls in streams arrive in pieces and must be assembled before running. Proxy buffering can also break streaming, so test the full path.
Revise: day-15-streaming-realtime.md

## O. Caching (Day 16)

**Q74. [Fresher] What types of caching exist for LLM apps?**
Exact-match response cache, semantic cache (similar questions), provider prompt caching (repeated prefix), and embedding cache.
Revise: day-16-caching-cost-optimization.md

**Q75. [Mid] How does semantic caching work, and what is the risk?**
You embed the question and return a stored answer if a similar one exists above a threshold. The risk is a false hit: a similar question with a different correct answer.
Tune the threshold on real data, avoid it for personal or time-sensitive answers, and include tenant in the key.
Revise: day-16-caching-cost-optimization.md

**Q76. [Mid] How do you use provider prompt caching well?**
Put stable content (system prompt, tools, long documents) first and changing content last, so the shared prefix can be reused. Rules and discounts differ by provider, so check current docs.
Revise: day-16-caching-cost-optimization.md, day-04-cost-token-management.md

**Q77. [Senior] What are cache invalidation risks for AI answers?**
Source documents change, prompts change, and models change. A cached answer may become wrong or unsafe. Include prompt and model version in the key, set time limits, and clear cache when sources update. Never cache across tenants.
Revise: day-16-caching-cost-optimization.md

## P. Resilience and rate limiting (Day 17)

**Q78. [Fresher] What should you do on a 429 or timeout from a provider?**
Retry with exponential backoff and jitter (random delay), respect any retry-after header, and set a limit on attempts. Do not retry errors that cannot succeed, such as invalid requests.
Revise: day-17-resilience-rate-limiting-queues.md

**Q79. [Mid] What is a circuit breaker?**
After many failures, it stops sending calls for a short time, so you fail fast and let the provider recover. Then it tests with a few calls before closing again.
Pair it with a fallback model or a clear degraded response.
Revise: day-17-resilience-rate-limiting-queues.md

**Q80. [Mid] How do you rate limit users when your own provider limit is shared?**
Limit per user and per tenant (requests and tokens), and use a queue with priorities. Reserve capacity for important traffic. Return clear 429 responses with retry hints.
Revise: day-17-resilience-rate-limiting-queues.md

**Q81. [Mid] When do you use an async job queue?**
For long generations, batch work or heavy agent runs. Return a job ID, process in workers, and let the client poll or receive a callback. This avoids HTTP timeouts and lets you control concurrency.
Revise: day-17-resilience-rate-limiting-queues.md

**Q82. [Senior] Design fallback chains. What can go wrong?**
Fallback to another provider or a smaller model on failure. Risks: different output format, lower quality, different safety behaviour, and retry storms that multiply load.
Test fallback paths in evals, cap total retries across layers, and use a total timeout budget for the request.
Revise: day-17-resilience-rate-limiting-queues.md

## Q. Observability (Day 18)

**Q83. [Fresher] What should you log for each LLM call?**
Request ID, user or tenant ID, model and prompt version, token counts, latency, cost, status or error, and (with care) inputs and outputs.
Revise: day-18-observability-llm-apps.md

**Q84. [Mid] What is tracing for an LLM app?**
A trace shows one request as connected steps (spans): retrieval, rerank, tool calls, generation. It helps find where time, cost or errors come from, and why an answer was wrong.
Revise: day-18-observability-llm-apps.md

**Q85. [Mid] How do you measure quality in production?**
Use user feedback (thumbs, edits, retries), automatic checks, and an LLM judge on sampled traffic. Add human review for samples. Watch trends over time.
Revise: day-18-observability-llm-apps.md, day-21-llm-evaluation-fundamentals.md

**Q86. [Senior] How do you detect drift?**
Drift is a slow change in quality or input patterns. Track sampled quality scores, topic mix, refusal rates, token use and retrieval hit quality. Alert on changes. Causes include new user behaviour, changed documents, and silent provider model updates.
Revise: day-18-observability-llm-apps.md

## R. Prompt injection, jailbreaks, PII and guardrails (Days 19-20)

**Q87. [Fresher] What is prompt injection?**
An attacker puts instructions in text the model reads, trying to change its behaviour. Direct injection comes from the user. Indirect injection hides in documents, web pages or tool results.
Revise: day-19-prompt-injection-jailbreaks.md

**Q88. [Mid] Prompt injection vs jailbreak?**
Jailbreak tries to make the model break its safety rules. Injection tries to hijack your application's instructions, often through data. Defence differs: jailbreak is mostly model behaviour, injection is mostly architecture.
Revise: day-19-prompt-injection-jailbreaks.md

**Q89. [Mid] Why is prompt injection harder than SQL injection?**
SQL has a clear boundary between code and data, and parameterised queries enforce it. An LLM reads instructions and data as one stream of text, so there is no reliable boundary.
So you cannot fully fix it with escaping. You limit impact instead.
Revise: day-19-prompt-injection-jailbreaks.md

**Q90. [Senior] Describe defence in depth for an LLM app.**
Input checks, clear delimiters and system prompt rules, least-privilege tools, output validation, human approval for risky actions, network egress limits, monitoring, and regular red-team tests.
Say that each layer can fail, which is why you need several.
Revise: day-19-prompt-injection-jailbreaks.md

**Q91. [Mid] How do you handle PII (personal data) in an LLM pipeline?**
Map where data flows (prompts, logs, vector store, caches, memory). Detect and redact before sending to the model and before logging. Use a "redact, process, rehydrate" pattern when the final answer needs real values.
Revise: day-20-data-protection-pii-guardrails.md

**Q92. [Senior] How do you enforce tenant isolation in RAG?**
Store tenant or access-control data on each chunk, filter on it inside the retrieval query, and enforce it in code, not in the prompt. Test with cross-tenant probes. Consider separate indexes for high-risk tenants.
Revise: day-20-data-protection-pii-guardrails.md

**Q93. [Senior] A user asks for data deletion. What is hard about it?**
The data may exist in source stores, chunks, embeddings, caches, memory, logs, backups and evaluation sets. You need a map of all copies and a delete process that covers each. Check legal advice for your region.
Revise: day-20-data-protection-pii-guardrails.md

## S. Evaluation, prompt versioning and A/B testing (Days 21-22)

**Q94. [Fresher] Why is "it looks good" not evaluation?**
Small manual checks miss rare failures and change from person to person. You need a fixed test set, clear metrics and repeatable runs, so you can compare versions.
Revise: day-21-llm-evaluation-fundamentals.md

**Q95. [Mid] Offline vs online evaluation?**
Offline runs a fixed golden set before release, for example in CI. Online checks real traffic after release, using feedback, sampling and A/B tests. You need both.
Revise: day-21-llm-evaluation-fundamentals.md

**Q96. [Mid] What are RAG-specific metrics?**
Context recall and precision (retrieval quality), faithfulness (answer supported by context), and answer relevance. Tools such as RAGAS offer these. Verify the metrics against human judgement on a sample.
Revise: day-21-llm-evaluation-fundamentals.md

**Q97. [Senior] What are the biases of LLM-as-a-judge, and how do you control them?**
Judges can prefer longer answers, prefer the first option shown, prefer their own style, and be inconsistent.
Controls: clear rubrics, swap answer order, use pairwise or scored criteria, use a different model as judge, and calibrate against human labels.
Revise: day-21-llm-evaluation-fundamentals.md

**Q98. [Senior] Scenario: Design evaluation for a support chatbot.**
1. Define success: correct and grounded answers, safe tone, right escalation, low cost.
2. Build a golden set from real tickets (with PII removed): common, rare and adversarial cases.
3. Metrics: answer correctness, faithfulness to the knowledge base, retrieval recall@k, escalation precision and recall, refusal on out-of-scope, latency, cost.
4. Layers: deterministic checks, LLM judge with rubric, human review on a sample.
5. Run in CI to block regressions. Include injection and PII test cases.
6. Online: thumbs, resolution rate, handoff rate, sampled judging, drift alerts.
7. Feed failures back into the golden set.
Revise: day-21-llm-evaluation-fundamentals.md, day-27-project-ai-customer-support.md

**Q99. [Mid] How do you version and test prompts?**
Keep prompts in version control or a registry, give each version an ID, record the version in every trace, and run the eval set on each change. Separating prompts from code deploys allows fast rollback.
Revise: day-22-prompt-versioning-ab-testing.md

**Q100. [Senior] How do you A/B test a prompt or model in production?**
Split users randomly and consistently, define the main metric and guardrail metrics (cost, latency, safety), run long enough for enough samples, and check results with care because LLM outputs are noisy.
Offline evals first; online tests confirm real-world effect. Use human review for subjective quality.
Revise: day-22-prompt-versioning-ab-testing.md

## T. Projects and experience (Days 23-29)

**Q101. [Mid] Walk me through your PDF RAG project.**
Cover: parsing, chunking choice and why, embedding model, hybrid retrieval with rerank, citations, evaluation numbers before and after changes, failure cases you found.
Give one honest trade-off and one thing you would do next.
Revise: day-24-project-pdf-rag.md

**Q102. [Senior] When is a multi-agent design a bad idea?**
When one agent with tools can do the job. More agents mean more cost, more latency, harder debugging and more failure points. Use multiple agents only for clear parallel or specialised work.
Revise: day-26-project-multi-agent-research.md

**Q103. [Senior] In customer support AI, when should the bot escalate to a human?**
Low retrieval confidence, sensitive topics, repeated user frustration, high-impact actions (refunds, account changes) and policy limits. Use an approval gate for actions.
Escalating correctly is a feature, not a failure.
Revise: day-27-project-ai-customer-support.md

**Q104. [Senior] In document extraction (supplier onboarding), where do you use rules and where the LLM?**
Use the LLM to read messy text into fields with confidence. Use deterministic code for formats, checksums and required fields. Send low-confidence or conflicting results to human review.
Do not use a model for checks that a regex can do.
Revise: day-28-project-supplier-onboarding.md

**Q105. [Senior] In an AI code review tool, how do you control false positives?**
Ask for structured findings with severity and evidence, filter low-confidence items, give the model repository context with tools, measure precision on labelled reviews, and let users mute rule types.
Too many wrong comments make users ignore the tool.
Revise: day-29-project-ai-code-review.md

---

# Part 3. Eight AI system design prompts

For each prompt, follow the framework from Day 30: clarify, estimate, data and retrieval, model and serving, AI-specific concerns, scale and failure modes. Revise: day-30-ai-system-design-interview.md for all of them.

## Design 1. Document Q&A over company files (multi-tenant)

- Clarify: number of tenants and documents, quality bar, latency target, privacy and region rules.
- Estimate: chunks per tenant, vector storage (vectors times dimensions times bytes), queries per second, tokens per request, monthly cost.
- Ingestion: parse, chunk with metadata, embed, store; handle updates and deletes.
- Retrieval: hybrid search, tenant and permission filter in the query, rerank.
- Generation: grounded answer, citations, "I do not know" path, streaming.
- Eval: golden set per tenant, retrieval recall@k, faithfulness sampling online.
- Safety: tenant isolation tests, injection in documents, PII redaction.
- Cost and scale: caching, model routing, per-tenant budgets, fallback provider.

## Design 2. Customer support assistant with refunds

- Clarify: channels, languages, which actions are allowed, risk of a wrong action.
- Knowledge: RAG over help articles with freshness updates.
- Actions: tools with least privilege, idempotency keys, approval gate for refunds.
- Escalation: confidence and sentiment rules, handoff with a conversation summary.
- Memory: per-user history, trimmed and summarised, with privacy limits.
- Safety: injection from user text and ticket content, PII handling, output filters.
- Eval: golden tickets, resolution and escalation metrics, online sampling.
- Operations: tracing, cost per ticket, rate limits, queue for peaks.

## Design 3. Coding assistant or code review bot

- Clarify: languages, repository size, latency, where results appear (pull request, editor).
- Context: index the repository, retrieve relevant files and symbols, respect file permissions.
- Model use: tool access to read files and run safe checks; structured findings with severity.
- Quality: precision over volume, confidence filter, user muting, feedback learning.
- Safety: untrusted code and comments as injection, secret scanning, no outbound data leak.
- Eval: labelled historical reviews, precision and false-positive rate.
- Cost: review only changed code, cache by commit, route small changes to a cheaper model.
- Failure modes: huge diffs, timeouts, partial results.

## Design 4. Multi-agent research assistant

- Clarify: depth, time limit, source types, cost ceiling per task.
- Decide first whether one agent is enough, and say why or why not.
- Orchestration: planner, parallel researcher agents, aggregator, shared typed state.
- Controls: step limits, token budgets, deduplication, timeouts per agent.
- Grounding: citations to sources, verification step, conflict handling.
- Safety: web content is untrusted, restricted tools, egress limits.
- Eval: sampled human review, citation accuracy, cost per report.
- Delivery: async job queue, progress streaming, resumable checkpoints.

## Design 5. Chat product at large scale (millions of users)

- Clarify: concurrent users, latency target, free vs paid tiers, safety needs.
- Estimate: requests per second, tokens per second, provider rate limits, cost per user.
- Serving: streaming over SSE, gateway with provider abstraction, model routing by tier.
- Reliability: backoff, circuit breakers, fallback models, load shedding.
- State: conversation store, history trimming, per-user memory with isolation.
- Cost: prompt caching, semantic cache for common questions, per-user quotas.
- Safety: input and output moderation, abuse detection.
- Observability: traces, cost metrics by tier, quality sampling, drift alerts.

## Design 6. Semantic search and recommendation for a product catalogue

- Clarify: catalogue size, update rate, language, ranking goals (relevance, business rules).
- Embeddings: model choice tested on real queries, text plus attributes, re-embedding plan.
- Index: vector database with filters (price, category, stock), ANN tuning, sharding.
- Retrieval: hybrid with exact terms (SKUs), reranker, business rules after ranking.
- Freshness: incremental updates, deletion of removed items.
- Eval: labelled queries, recall@k, nDCG, online click and conversion tests.
- Cost and latency: cache popular queries, cache embeddings by content hash.
- Failure modes: cold start, typos, empty results fallback.

## Design 7. Document extraction and onboarding pipeline

- Clarify: document types, volume, accuracy target, cost of an error, compliance.
- Pipeline: ingest, OCR or parse, LLM extraction into a schema with confidence per field.
- Validation: deterministic checks first (formats, checksums, cross-field rules).
- Review: low-confidence and conflicts go to a human queue with evidence highlights.
- State machine: clear states, retries, idempotent steps, audit log.
- Security: PII redaction in logs, access control, retention rules.
- Eval: field-level precision and recall on a labelled set, drift monitoring.
- Scale: async queue, batching for cost, per-document timeouts.

## Design 8. Evaluation and prompt release platform

- Clarify: who edits prompts, how many apps, release speed, compliance needs.
- Registry: versioned prompts and configs, decoupled from code deploys, easy rollback.
- Offline gate: golden sets run in CI, pass and fail thresholds, cost of the eval run.
- Judges: rubrics, bias controls, calibration against human labels.
- Online: canary and A/B traffic split, guardrail metrics (cost, latency, safety).
- Feedback loop: production failures become new test cases.
- Observability: every trace records prompt and model version.
- Governance: approvals for high-risk prompts, audit trail, access control.

---

# Part 4. Common mistakes in AI engineering interviews

Avoid these. Each has a better habit.

1. **Skipping clarification.** Ask about users, quality bar, latency, cost and privacy first. Then state your assumptions.
2. **Quoting model names, prices or limits as facts.** These change. Say you would check current provider docs, and show the formula instead.
3. **Proposing fine-tuning to add knowledge.** Use RAG for facts. Fine-tuning is for style, format or behaviour.
4. **No evaluation story.** "We will check outputs by hand" is a weak answer. Describe a golden set, offline gates and online sampling.
5. **Blaming the model first.** In RAG, most failures are in retrieval, chunking or parsing. Debug stage by stage.
6. **Believing temperature 0 removes hallucination.** It only reduces randomness.
7. **Trusting the prompt for security.** Prompts cannot enforce access control. Enforce it in code and limit tool power.
8. **Forgetting tokens and cost.** Say input and output tokens per request, then compute cost and latency.
9. **Using an LLM where code is enough.** Regex, checksums and rules are cheaper and exact.
10. **Ignoring failure paths.** Describe timeouts, retries, fallbacks, queues and what the user sees when things fail.
11. **Over-engineering with agents.** Start with the simplest design. Add agents, graphs or more tools only when a measured problem needs them.
12. **Ignoring privacy.** Mention PII in prompts, logs, caches and vector stores, and how deletion works.
13. **Claiming perfect numbers.** Do not promise "99% accuracy" or "60% savings" without data. Say what you would measure.
14. **Talking only about tools and frameworks.** Explain the idea and the trade-off. Frameworks change; reasoning lasts.
15. **No project depth.** Prepare one story per project about a hard trade-off, a failure you found and how you measured the fix.
16. **Rambling.** Use the framework out loud and keep answers short. Offer to go deeper.

## Final checklist before an interview

- Can you explain tokens, context, hallucination and sampling in plain words?
- Can you draw RAG, and debug it stage by stage?
- Can you estimate cost and storage on a whiteboard?
- Can you list three layers of defence against prompt injection?
- Can you describe an eval plan with offline and online parts?
- Did you check current provider docs for any names, prices and limits you plan to mention?
