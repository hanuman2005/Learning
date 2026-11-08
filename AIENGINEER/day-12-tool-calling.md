# Day 12 — Tool Calling & Function Calling

> Series: AI Backend Engineering
>
> **Goal:** Master the tool-use loop precisely — model emits a call, your code executes it, you feed the result back — including schema design, argument validation, error recovery, parallel calls, loop guards, and idempotency, built framework-free with your Day 3 provider first.

---

## Table of Contents

1. [Why Tool Calling Exists](#1-why-tool-calling-exists)
2. [The Tool-Use Loop, Precisely](#2-the-tool-use-loop-precisely)
3. [Tool Schema Design](#3-tool-schema-design)
4. [Validating Model-Supplied Arguments](#4-validating-model-supplied-arguments)
5. [Returning Errors So the Model Recovers](#5-returning-errors-so-the-model-recovers)
6. [Parallel Calls and Forcing Tool Choice](#6-parallel-calls-and-forcing-tool-choice)
7. [Building It Framework-Free (3 Tools)](#7-building-it-framework-free-3-tools)
8. [Reliability: Guards, Loops, Idempotency](#8-reliability-guards-loops-idempotency)
9. [The Prebuilt LangGraph Agent](#9-the-prebuilt-langgraph-agent)
10. [The Dark Side](#10-the-dark-side)
11. [Day 12 Self-Check](#11-day-12-self-check)
12. [Tomorrow](#12-tomorrow)
13. [Apply (1 hour)](#13-apply-1-hour)

---

## 1. Why Tool Calling Exists

**Why.** An LLM is a text predictor trapped in its training data. It can't do arithmetic reliably, can't read your database, can't check today's weather, can't hit an API. **Tool calling** (a.k.a. function calling) gives the model *hands*: a structured way to say "call `get_weather(city='Paris')`" so your code can run the real function and hand back the result.

**What.** The model doesn't execute anything. It emits a *structured request* — a tool name plus JSON arguments — matching a schema you supplied. **Your backend executes the function** and returns the result. The model then continues, incorporating that result. It's a clean separation: the model *decides*, your code *acts*.

**When.** Whenever the model needs live data (DB, API), an action with side effects (send email, create ticket), or reliable computation (math, code execution) it can't do internally.

**Where.** In the same node from Day 11 that used to only think — now it thinks *and* calls tools. This is the mechanism under every "agent" and, tomorrow, under MCP (Day 13).

```mermaid
flowchart LR
    M["LLM: decides"] -->|"tool call (name + JSON args)"| Y["Your backend: executes"]
    Y -->|"tool result"| M
    Y --> DB[("Database (your SQL)")]
    Y --> API["External API"]
    Y --> CALC["Pure function"]
    style M fill:#2c5282,color:#fff
    style Y fill:#22543d,color:#fff
```
*The model never touches your systems — it requests, your code executes, you return. That boundary is the whole security model (teaser: Day 19).*

> **🤔 Wait — why let the model call tools at all? Can't I just parse its text answer and run the function myself?**
> You can, and for *one* fixed function in a fixed order you *should* (that's the §10 lesson). Tool calling earns its keep when **the model must decide, at runtime, which tool and which arguments** — and you don't know in advance. Parsing free-form text for that is a nightmare: the model writes "I'll look up user 42's logins" in prose and you're regexing English into a function call, brittle on every rephrase. Tool calling makes the model emit a *structured* request — validated tool name + JSON args against a schema you control — so you get typed, checkable intent instead of prose to reverse-engineer. Crucially, the model still executes *nothing*; it requests, your code acts. That boundary is exactly why it's safe to give an LLM "hands" at all.

**✅ Good scenario (where it saves you):** A support agent handles "how many purchases did user 42 make last week, and is that above their plan limit?" The model calls `user_metric(user_id=42, metric='purchases', days=7)`, reads the result, then calls `plan_limit(user_id=42)`, then answers — chaining live DB lookups it could never do from training data. One conversational turn resolves a query that used to require a human pulling two internal dashboards.

**❌ Bad scenario (when misused):** A team exposes a `run_sql(query: str)` tool that runs whatever SQL the model writes. The model hallucinates a `DELETE` inside a "read" query, or a prompt-injected document convinces it to `DROP TABLE`, and because the tool passed model text straight to the DB with write permissions, it's a real irreversible incident with the LLM as the attacker's proxy. The fix isn't "don't use tools" — it's narrow, validated, least-privilege tools (§4).

**When NOT to:** if your app always needs the same function in the same order — every request runs `classify → fetch → summarize` — call those functions directly; asking an LLM to "decide" a route that never varies is slower, costlier, non-deterministic, and harder to test than a plain `if`.

---

## 2. The Tool-Use Loop, Precisely

The single most important diagram in this file. The loop is multi-turn: the model may call several tools in sequence before producing a final answer.

```mermaid
sequenceDiagram
    participant U as User
    participant A as Your App (loop)
    participant M as LLM (Day 3 provider)
    participant T as Tool (function)
    U->>A: question
    A->>M: messages + tool schemas
    M-->>A: stop_reason = "tool_use"<br/>tool: db_query, args: {...}
    A->>A: validate args
    A->>T: execute db_query(args)
    T-->>A: result rows
    A->>M: append tool_result, call again
    M-->>A: stop_reason = "tool_use"<br/>tool: calculator, args: {...}
    A->>T: execute calculator(args)
    T-->>A: 42
    A->>M: append tool_result, call again
    M-->>A: stop_reason = "end_turn"<br/>final text answer
    A-->>U: answer
```
*The tool-use loop is turn-based: keep looping while stop_reason is "tool_use", stop when it's "end_turn" — with a hard iteration cap so you never loop forever.*

The loop in pseudocode:

```
messages = [user_question]
while True:
    response = model.call(messages, tools=schemas)
    if response.stop_reason == "tool_use":
        for call in response.tool_calls:
            result = execute(call.name, call.args)   # validate first!
            messages.append(tool_result(call.id, result))
        continue
    else:  # "end_turn"
        return response.text
```

Two non-negotiables you'll build in §8: a **max-iterations guard** on that `while True`, and **validation** before `execute`.

---

## 3. Tool Schema Design

**Why.** The model chooses tools and fills arguments *entirely from the schema and descriptions you write.* A vague description or a fuzzy parameter name and the model guesses wrong. **Schema quality is prompt engineering** — arguably the highest-leverage part of tool calling.

**What.** A tool schema is a name, a natural-language description, and a JSON Schema for the arguments. The cleanest way to author it in Python is Pydantic → JSON Schema.

```python
from pydantic import BaseModel, Field


class DBQueryArgs(BaseModel):
    """Query the read-only analytics database for user metrics."""
    user_id: int = Field(description="Numeric user ID to look up. Required.")
    metric: str = Field(
        description="One of: 'logins', 'purchases', 'sessions'. Exactly these values.",
    )
    days: int = Field(default=30, ge=1, le=365,
                      description="Look-back window in days, 1-365. Default 30.")


# Pydantic gives you the JSON Schema the model needs:
schema = {
    "name": "db_query",
    "description": DBQueryArgs.__doc__,
    "input_schema": DBQueryArgs.model_json_schema(),
}
```

**How to write descriptions that work:**
- State *what the tool does* and *when to use it* in the description.
- Enumerate allowed values ("One of: ...") — the model respects enums.
- Say what's required vs optional and give defaults.
- Add units and ranges ("days, 1-365") — ambiguity produces bad args.

```mermaid
flowchart LR
    P["Pydantic model<br/>+ Field descriptions"] --> S["JSON Schema"]
    S --> M["Model reads schema"]
    M --> C["Emits tool call<br/>(name + args)"]
    C --> V["Your validator (Pydantic again)"]
    V --> E["Execute"]
    E --> R["Return result to model"]
```
*Author the schema once in Pydantic; it does double duty — it tells the model what to send and validates what comes back.*

| Good schema practice | Why it matters |
|---|---|
| Clear, action-oriented description | Model picks the right tool |
| Enumerate allowed values | Prevents invalid arguments |
| Units, ranges, defaults | Removes ambiguity |
| Few, focused parameters | Model fills them correctly |
| One clear job per tool | Avoids overlap/confusion (§10) |

---

## 4. Validating Model-Supplied Arguments

**Why.** The model's arguments are *untrusted input* — exactly like a request body from a client. The model can hallucinate a `user_id` that doesn't exist, pass a `metric` outside the enum, or inject SQL. **Never pass model args straight into a query or shell.** This is the same input-validation discipline you apply to any API.

```python
from pydantic import ValidationError

def safe_execute(name: str, raw_args: dict) -> str:
    try:
        args = DBQueryArgs(**raw_args)          # Pydantic validates & coerces
    except ValidationError as e:
        return f"ERROR: invalid arguments: {e.errors()}"   # see §5
    # Parameterized query — NEVER string-format model input into SQL.
    rows = db.execute(
        "SELECT value FROM metrics WHERE user_id = %s AND metric = %s "
        "AND ts > now() - (%s || ' days')::interval",
        (args.user_id, args.metric, args.days),
    )
    return format_rows(rows)
```

**The Dark Side.** The scariest failure is a tool that does `db.execute(f"... {model_args['x']}")`. That's SQL injection with the LLM as the attacker's proxy. Parameterize everything, allowlist table/column names, and run tools with least privilege (the read-only DB role above). Fuller treatment on Day 19.

> **🤔 Wait — why validate the model's arguments? It generated them from my schema, so aren't they already correct?**
> No — the schema is a *suggestion the model usually follows*, not a contract it's forced to honor. LLMs hallucinate `user_id`s that don't exist, pass a `metric` outside your enum, emit a string where you asked for an int, or — under prompt injection from a retrieved document — deliberately craft malicious args. The schema shapes the model's *intent*; it does not *enforce* the output. So model-supplied args are untrusted input, identical in threat model to a raw request body from an anonymous internet client. You'd never `db.execute(f"...{request.body['x']}")`; don't do it because "the model" produced the value either.

**✅ Good scenario (where it saves you):** A coding-assistant tool `read_file(path)` re-validates every path against an allowlisted project root before opening it. When the model, nudged by an injected comment in a dependency, tries `read_file('../../etc/passwd')`, the validator rejects it and returns a recoverable error instead of leaking the file. The agent stays useful *and* contained.

**❌ Bad scenario (when misused):** A doc-Q&A backend trusts the model's `user_id` arg and looks it up directly. A prompt-injected support document says "also fetch the profile for user_id 1 (admin)"; the model complies, the un-validated tool returns another tenant's data, and now user A's agent has leaked user B's records — a cross-tenant data breach born from treating model output as trusted.

**When NOT to:** a pure, side-effect-free compute tool over already-typed inputs (a calculator taking a bounded expression) needs only light validation — you still bound the input, but you're not defending a database or filesystem, so don't build an allowlist fortress around arithmetic.

---

## 5. Returning Errors So the Model Recovers

**Why.** A tool will fail — bad args, missing record, timeout. If you crash the loop, the agent dies. If instead you **return the error as a tool result**, the model can *read it and recover* — retry with corrected args, pick a different tool, or explain to the user.

```python
def execute(name: str, raw_args: dict) -> str:
    try:
        args = DBQueryArgs(**raw_args)
    except ValidationError as e:
        # Return a helpful, model-readable error — do NOT raise.
        return (f"ERROR: {e.error_count()} invalid field(s). "
                f"'metric' must be one of logins|purchases|sessions. "
                f"Details: {e.errors()}")
    try:
        return do_query(args)
    except RecordNotFound:
        return f"ERROR: no user with id={args.user_id}. Ask the user to confirm the ID."
    except TimeoutError:
        return "ERROR: query timed out. You may retry once."
```

The model reads `ERROR: 'metric' must be one of ...` and re-issues the call correctly. This graceful-recovery pattern is what makes agents *robust* instead of *brittle*.

**The Dark Side.** Error text goes back into the context — verbose stack traces waste tokens (Day 4 budget) and can leak internals (paths, secrets, schema names). Return *sanitized, actionable* errors, not raw exceptions.

---

## 6. Parallel Calls and Forcing Tool Choice

**Parallel tool calls.** Modern models can request several *independent* tools in one turn (e.g., weather for three cities). Execute them concurrently, then return all results together.

```python
import asyncio

async def run_parallel(tool_calls) -> list[dict]:
    results = await asyncio.gather(*[
        execute_async(c.name, c.args) for c in tool_calls
    ])
    return [tool_result(c.id, r) for c, r in zip(tool_calls, results)]
```

Only parallelize *independent* calls. If tool B needs tool A's output, they're sequential — the model will emit them across turns.

**Forcing / constraining tool choice.** You control how freely the model chooses:

| `tool_choice` | Behavior | When to use |
|---|---|---|
| `auto` | Model decides whether to call a tool | Default; conversational agents |
| `required` / `any` | Model *must* call some tool | When an answer requires fresh data |
| specific tool | Model must call *that* tool | Structured extraction; forced first step |
| `none` | No tools this turn | Force a text-only summary/answer |

```python
response = provider.chat(               # your Day 3 client
    messages=messages,
    tools=schemas,
    tool_choice={"type": "tool", "name": "db_query"},  # force this tool
)
```

```mermaid
flowchart TD
    Q["Turn"] --> TC{"tool_choice"}
    TC -->|auto| A["Model may answer or call"]
    TC -->|required| B["Model must call a tool"]
    TC -->|specific| C["Model must call named tool"]
    TC -->|none| D["Model must answer as text"]
```
*Constraining tool choice is how you make a probabilistic model behave deterministically at decision points.*

---

## 7. Building It Framework-Free (3 Tools)

Framework-free first — so you own the loop. Three tools: a **calculator** (pure), a **web search stub** (external API), and a **DB query** (your SQL skills).

```python
import json
from typing import Callable
from pydantic import BaseModel, Field, ValidationError
from my_provider import LLMClient   # your Day 3 abstraction

client = LLMClient(provider="anthropic")

# ---- 1. Tool argument models ----
class CalcArgs(BaseModel):
    """Evaluate a basic arithmetic expression. Use for any math."""
    expression: str = Field(description="e.g. '3 * (4 + 5)'. Digits and + - * / ( ) only.")

class SearchArgs(BaseModel):
    """Search the public web for recent facts the model may not know."""
    query: str = Field(description="Concise search query.")

class UserMetricArgs(BaseModel):
    """Look up a metric for a user in the read-only analytics DB."""
    user_id: int = Field(ge=1)
    metric: str = Field(description="One of: logins, purchases, sessions")

# ---- 2. Implementations (validated, sandboxed) ----
import ast, operator
_OPS = {ast.Add: operator.add, ast.Sub: operator.sub,
        ast.Mult: operator.mul, ast.Div: operator.truediv, ast.USub: operator.neg}

def _eval(node):  # safe arithmetic — NOT eval()
    if isinstance(node, ast.Constant): return node.value
    if isinstance(node, ast.BinOp): return _OPS[type(node.op)](_eval(node.left), _eval(node.right))
    if isinstance(node, ast.UnaryOp): return _OPS[type(node.op)](_eval(node.operand))
    raise ValueError("unsupported expression")

def calculator(a: CalcArgs) -> str:
    try:
        return str(_eval(ast.parse(a.expression, mode="eval").body))
    except Exception:
        return "ERROR: could not evaluate; use only digits and + - * / ( )."

def web_search(a: SearchArgs) -> str:
    # stub — wire to a real search API in production
    return json.dumps([{"title": "stub result", "snippet": f"about {a.query}"}])

def user_metric(a: UserMetricArgs) -> str:
    if a.metric not in {"logins", "purchases", "sessions"}:
        return "ERROR: metric must be logins|purchases|sessions."
    # rows = db.execute("SELECT ... WHERE user_id=%s AND metric=%s", (a.user_id, a.metric))
    return json.dumps({"user_id": a.user_id, "metric": a.metric, "value": 123})

# ---- 3. Registry: schema + validator + fn ----
TOOLS: dict[str, tuple[type[BaseModel], Callable]] = {
    "calculator": (CalcArgs, calculator),
    "web_search": (SearchArgs, web_search),
    "user_metric": (UserMetricArgs, user_metric),
}

def build_schemas() -> list[dict]:
    return [{"name": n, "description": m.__doc__, "input_schema": m.model_json_schema()}
            for n, (m, _) in TOOLS.items()]

def dispatch(name: str, raw_args: dict) -> str:
    if name not in TOOLS:
        return f"ERROR: unknown tool '{name}'. Available: {list(TOOLS)}"  # hallucinated tool
    model_cls, fn = TOOLS[name]
    try:
        return fn(model_cls(**raw_args))
    except ValidationError as e:
        return f"ERROR: invalid args: {e.errors()}"

# ---- 4. The loop, WITH a guard ----
def run_agent(question: str, max_iters: int = 6) -> str:
    messages = [{"role": "user", "content": question}]
    for i in range(max_iters):                      # hard cap — never `while True`
        resp = client.chat(messages=messages, tools=build_schemas(), tool_choice="auto")
        if resp.stop_reason != "tool_use":
            return resp.text
        messages.append(resp.assistant_message)
        for call in resp.tool_calls:
            result = dispatch(call.name, call.args)
            messages.append({"role": "tool", "tool_call_id": call.id, "content": result})
    return "Stopped: exceeded max tool iterations."   # loop guard exit
```

Every failure mode is handled inline: unknown tool, invalid args, and the iteration cap. You can `print(messages)` and see every prompt, arg, and result — the transparency you value from Day 3.

---

## 8. Reliability: Guards, Loops, Idempotency

```mermaid
flowchart TD
    L["Tool-use loop"] --> G1["Max-iterations guard<br/>(never while True)"]
    L --> G2["Validate every arg<br/>(Pydantic)"]
    L --> G3["Unknown-tool check<br/>(reject hallucinated)"]
    L --> G4["Return errors, don't crash"]
    L --> G5["Idempotency for side-effect tools"]
    L --> G6["Token/cost budget (Day 4)"]
```
*Six guardrails wrap the loop — remove any one and you have a production incident waiting.*

**Max-iterations guard.** A model can loop forever — call a tool, get a result it dislikes, call again, forever. Cap iterations (6–10 typical). Combine with the Day 4 token budget to cap spend per run.

**Hallucinated tools/args.** Models invent tool names and pass arguments outside the schema. Your `dispatch` rejects unknown tools and Pydantic rejects bad args — both return recoverable errors (§5).

**Idempotency of tools.** Reuse your **System Design Day 19 idempotency knowledge**. The loop may retry, and the model may re-issue the same call. A read tool (`user_metric`) is naturally idempotent. A *write* tool (`create_ticket`, `send_email`) is not — a retry could create two tickets. Make side-effecting tools idempotent with an **idempotency key**:

```python
class CreateTicketArgs(BaseModel):
    title: str
    idempotency_key: str = Field(description="Stable key; reuse to avoid duplicates.")

def create_ticket(a: CreateTicketArgs) -> str:
    if existing := tickets.find_by_key(a.idempotency_key):
        return f"Ticket already exists: {existing.id}"   # safe on retry
    return f"Created ticket {tickets.create(a.title, key=a.idempotency_key).id}"
```

| Tool type | Idempotent by nature? | Precaution |
|---|---|---|
| Read (query, search) | Yes | None needed |
| Compute (calculator) | Yes | None needed |
| Write (create, send, pay) | No | Idempotency key + dedupe; confirm (§10) |

---

## 9. The Prebuilt LangGraph Agent

You built the loop by hand so you *understand* it. In production, LangGraph (Day 11) ships a prebuilt tool-calling agent — the ReAct-style loop, guards and all — as a compiled graph.

```python
from langgraph.prebuilt import create_react_agent   # verify name/signature at build time
from langchain_anthropic import ChatAnthropic

agent = create_react_agent(
    model=ChatAnthropic(model="claude-sonnet-4-5"),
    tools=[calculator_tool, web_search_tool, user_metric_tool],  # LangChain tool objects
)
result = agent.invoke({"messages": [("user", "How many logins did user 42 have?")]})
```

It gives you the loop, tool dispatch, message threading, checkpointing (Day 11 §6), and streaming for free. **But** it hides the loop internals — the same Day 10 trade-off. Because you built the raw version, you know exactly what it's doing and can debug it when it misbehaves.

---

## 10. The Dark Side

**Too many tools confuse the model.** Past ~10–15 tools (especially with overlapping descriptions), selection accuracy drops — the model picks the wrong tool or the wrong argument. Keep tool sets small and focused; group rarely-used tools behind a router or a sub-agent; make descriptions crisp and non-overlapping.

**Side effects with no confirmation.** A model that can `send_email`, `delete_record`, or `issue_refund` autonomously is a production hazard. A hallucinated or prompt-injected call becomes a real irreversible action. **Gate destructive/side-effecting tools behind human confirmation** (the Day 11 `interrupt()` pattern) and idempotency keys. Full security treatment — including tool-based prompt injection — is **Day 19**.

**Tool calling where a plain function suffices.** If your app always needs the same function in the same order, *call it directly* — don't ask an LLM to decide. Letting the model choose is slower, costlier, non-deterministic, and harder to test than `if intent == "refund": issue_refund()`. A **deterministic router** (intent classification → fixed handler) beats tool calling whenever the routing is knowable. This is the Day 11 "don't over-agentify" lesson at the tool layer.

```mermaid
flowchart TD
    Q{"Is the tool/order<br/>knowable in advance?"} -->|Yes| Det["Call the function directly /<br/>deterministic router"]
    Q -->|No, needs model judgment| Tool["Tool calling"]
    Tool --> Side{"Side effects?"}
    Side -->|Yes| Conf["Idempotency + human confirmation"]
    Side -->|No| Go["Run in loop with guards"]
    style Det fill:#742a2a,color:#fff
    style Tool fill:#22543d,color:#fff
```
*Tool calling is for genuine uncertainty; when the answer is deterministic, a plain function or router wins on every axis.*

---

## 11. Day 12 Self-Check

1. Walk through the tool-use loop turn by turn. What does the model actually emit, and who executes the tool?
2. Why is tool-schema description quality "prompt engineering"? Give three rules for writing good ones.
3. Why must model-supplied arguments be treated as untrusted input, and what's the worst-case failure?
4. Why return a tool error as a result instead of raising it? What must you sanitize before doing so?
5. When can you run tool calls in parallel, and when must they be sequential?
6. Give the four `tool_choice` modes and a use case for each.
7. Why is `while True` a bug in a tool loop, and what two limits do you impose instead?
8. A write tool gets retried by the loop. How does an idempotency key (System Design Day 19) prevent a duplicate action?
9. The prebuilt LangGraph agent gives you the loop for free. What does it hide, and why did you build it by hand first?
10. Give two situations where a plain function or deterministic router beats tool calling.

---

## 12. Tomorrow

**Day 13 — MCP (Model Context Protocol).** Today every tool was bespoke to your app. Tomorrow: the standard — "USB-C for AI tools" — that lets you expose tools, resources, and prompts as reusable, discoverable servers any AI host can consume. You'll build a minimal MCP server in Python and see how it differs from the raw function calling you just mastered.

---

## 13. Apply (1 hour)

1. **(30 min)** Build the §7 three-tool agent framework-free with your Day 3 provider. Ask a question that forces at least two tool calls (e.g., "look up user 42's purchases and multiply by 3"). Print `messages` and trace every hop of the loop.
2. **(10 min)** Feed it a question that makes the model pass an invalid `metric`. Confirm your validator returns a recoverable error and the model self-corrects on the next turn.
3. **(10 min)** Add a fourth, side-effecting tool (`create_ticket`) with an idempotency key. Call it, then force a retry with the same key; confirm no duplicate is created.
4. **(10 min)** Remove the `max_iters` guard and give the model a deliberately unsatisfiable task. Watch it loop (kill it fast). Restore the guard. You've now felt why the cap is non-negotiable.
