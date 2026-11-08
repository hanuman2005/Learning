# Python for Backend Engineers

Accuracy status: Reviewed against official documentation on 2026-11-06; items still unverified are marked.

A working guide to Python for people who build and operate backend services. It teaches the language's execution and object model, so you can reason about performance, concurrency and bugs, not only recall syntax.

**Audience:** developers with 1 to 3 years of experience who want production-level Python and interview readiness for strong product-company backend roles.
**Prerequisites:** basic programming skill and some Python syntax. Chapter 3 of this series (Linux) helps for processes and signals.
**Scope:** CPython 3.10 and later, server-side work (APIs, workers, data pipelines). Web frameworks, databases and Docker have their own chapters.
**Note on accuracy:** language semantics described here are stable. Version-specific features are marked with the version that introduced them. Tool options and defaults change between releases, so confirm exact flags and settings in each tool's official documentation.

## How to use this guide

Each chapter has four parts:

1. **Understand**: the concepts and the reasoning behind them.
2. **Tools to master**: what to learn to use fluently, and the level to reach.
3. **Common mistakes**: what goes wrong in practice.
4. **Self-check**: questions you should be able to answer without notes.

Chapter 18 lists hands-on labs. Chapter 19 collects interview questions, each with a concise model answer.

**What "good enough" means for a backend role:** you can explain what happens when a line of Python runs. You can choose between threads, processes and asyncio for a workload. You can write typed and tested code. You can find why a service is slow or leaking memory, and ship a reproducible build. Chapters 1, 3, 11, 12, 13 and 15 matter most.

## Contents

1. The object model
2. Core types and their costs
3. Iteration, comprehensions and generators
4. Functions, closures and decorators
5. Object-oriented Python
6. Dataclasses
7. Typing and Protocols
8. Exceptions and context managers
9. Modules, packages, environments and dependencies
10. Static type checking
11. Concurrency: the GIL, threads, processes
12. asyncio in depth
13. Testing with pytest
14. Logging
15. Profiling and performance
16. Standard library essentials
17. Packaging and production habits
18. Practice labs
19. Interview questions
20. References
21. Glossary

---

## 1. The object model

**In plain words:** A Python variable is a name that points to an object. Two names can point to the same object, so a change made through one name shows through the other. This chapter teaches you to predict that behaviour.

### Understand

**Everything is an object.** Integers, functions, classes and modules are all objects with an identity (`id()`), a type and a value. A **variable is a name bound to an object**, not a box holding a value. Assignment never copies the object; it makes another name point at it.

```python
a = [1, 2]
b = a          # same object, two names
b.append(3)
print(a)       # [1, 2, 3]
```

**Identity versus equality.** `is` compares identity (same object). `==` calls `__eq__` (same value). Use `is` only for singletons: `None`, `True`, `False`. Small integers and some strings are cached by CPython as an implementation detail, so `a is b` on numbers or strings gives results you must never rely on.

**Mutability.** Immutable: `int`, `float`, `str`, `bytes`, `tuple`, `frozenset`, `bool`, `None`. Mutable: `list`, `dict`, `set`, `bytearray` and most user-defined objects. Rebinding a name never affects other names, but mutating a shared object affects everyone holding it. A tuple is immutable in its own slots, yet a list inside a tuple can still change.

**Argument passing.** Python passes object references by value ("call by assignment"). A function that mutates a list argument changes the caller's list. A function that rebinds its parameter name does not.

**The mutable default argument trap.** Default values are evaluated once, at function definition.

```python
def add(item, bucket=[]):      # wrong: one shared list for all calls
    bucket.append(item)
    return bucket

def add(item, bucket=None):    # correct
    if bucket is None:
        bucket = []
    bucket.append(item)
    return bucket
```

**Copying.** `copy.copy` (shallow) copies the container but shares the elements. `copy.deepcopy` copies recursively and handles cycles, but is slow. `list(x)`, `x[:]`, `dict(x)` and `x.copy()` are shallow. `[[0] * 3] * 3` creates three references to one inner list, a classic bug.

**Garbage collection.** CPython uses **reference counting**: an object is freed when its count reaches zero. A separate cyclic garbage collector (`gc` module) finds unreachable reference cycles. Consequences: deterministic cleanup for most objects, but `__del__` timing is not guaranteed, so never use it for resource management (use context managers, chapter 8). Weak references (`weakref`) avoid keeping objects alive, for example in caches.

**Hashing.** A hashable object (one with a fixed hash value, so it can be a dict key) has a stable `__hash__` and an `__eq__` consistent with it. Dict keys and set members must be hashable. Mutable built-ins are not. Objects that compare equal must have equal hashes.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Inspecting objects | `id()`, `type()`, `is`, `sys.getrefcount`, `dir()`, `vars()` | Predict aliasing and mutation effects |
| Copying | `copy.copy`, `copy.deepcopy` | Know exactly what each shares |
| Interactive exploration | REPL, `python -i`, `help()`, IPython | Test a hypothesis in seconds |
| Lifetimes | `weakref`, `gc.collect`, `gc.get_objects` | Reason about reference cycles and leaks |
| Static checks for traps | `ruff` rule set `B` (bugbear) | Catch mutable defaults automatically |

### Common mistakes

- Using `is` to compare strings or numbers.
- Mutable default arguments and shared class-level mutable attributes.
- Assuming a copy where only a new name was created (`b = a`).
- Relying on `__del__` to release connections or locks.

### Self-check

- What does `b = a` do for a list, and what does `b = a + []` do?
- Why does `[[0] * 3] * 3` behave unexpectedly?
- Why can a tuple be a dict key but a list cannot?
- When is an object's memory freed in CPython, and what does the cyclic collector add?

---

## 2. Core types and their costs

**In plain words:** Each built-in type is fast at some jobs and slow at others. Choose the type that fits how you will use the data. A small change of type can make a program much faster.

### Understand

Choosing a data structure by its complexity is the most common source of Python performance wins in backend code.

| Type | Backing structure | Key costs |
|---|---|---|
| `list` | dynamic array of pointers | index O(1), append amortised O(1), `x in lst` O(n), insert/pop at front O(n) |
| `tuple` | fixed array | immutable, hashable if elements are, slightly smaller than list |
| `dict` | hash table, insertion-ordered (guaranteed since 3.7) | get/set/delete average O(1), key must be hashable |
| `set` | hash table | membership average O(1), set algebra (`&`, `|`, `-`) |
| `str` | immutable Unicode text | concatenation in a loop is O(n^2) in the worst case, use `"".join(parts)` |
| `bytes` | immutable bytes | what sockets, files and hashes deal in |
| `collections.deque` | doubly linked blocks | O(1) append and pop at both ends, O(n) indexing in the middle |

**Practical rules.**
- Membership tests against a collection you query repeatedly belong in a `set` or `dict`, not a `list`.
- Queues need `collections.deque`, not `list.pop(0)`.
- `heapq` gives a priority queue over a list. `bisect` keeps a sorted list searchable in O(log n).
- `sorted()` returns a new list, `list.sort()` sorts in place. Both are stable, so multi-key sorting can be done in passes. Use `key=` functions, not comparison functions.
- Dict views (`d.keys()`, `d.items()`) are live and set-like. Mutating a dict while iterating it raises `RuntimeError`.

**Text versus bytes.** `str` is Unicode text, `bytes` is raw data. Convert at the boundary with explicit encoding (`s.encode("utf-8")`, `b.decode("utf-8")`) and never rely on the platform default. Most production encoding bugs come from mixing the two or from opening files without `encoding=`.

**Numbers.** `int` has arbitrary precision. `float` is IEEE 754 double, so `0.1 + 0.2 != 0.3`. Money belongs in `decimal.Decimal` (built from strings, not floats) or in integer minor units such as cents. `bool` is a subclass of `int`.

**Memory.** Python objects carry overhead: a small `int` or a short `str` takes tens of bytes, and a list of a million Python ints is far larger than a C array. For large homogeneous numeric data use `array`, NumPy or a columnar format. For many small objects with fixed fields, `__slots__` (or `@dataclass(slots=True)`, 3.10+) removes the per-instance `__dict__` and reduces memory.

**Truthiness.** Empty containers, `0`, `0.0`, `""` and `None` are falsy. Test `is None` explicitly when `0` or `""` are valid values.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Containers | `collections` (`deque`, `Counter`, `defaultdict`, `OrderedDict`, `ChainMap`), `heapq`, `bisect` | Pick the right structure from the access pattern |
| Timing | `timeit`, `python -m timeit` | Measure a micro-choice, do not guess |
| Size | `sys.getsizeof`, `tracemalloc`, `pympler` (third party) | Estimate and measure memory of a structure |
| Money and precision | `decimal`, `fractions`, integer minor units | Never use float for currency |
| Text | `str` methods, `re`, `unicodedata`, `str.encode`/`bytes.decode` | Handle encoding explicitly |

### Common mistakes

- `if x in big_list` inside a loop (accidental O(n^2)).
- Building strings with `+=` in a loop over large inputs.
- Using floats for money.
- Opening files in text mode without specifying `encoding="utf-8"`.
- Modifying a dict or list while iterating it.

### Self-check

- Why is `x in some_set` fast and `x in some_list` slow?
- When would you use `deque` over `list`?
- Why does `sys.getsizeof([1, 2, 3])` understate the memory of the list?
- How would you represent prices to avoid rounding errors?

---

## 3. Iteration, comprehensions and generators

**In plain words:** A loop takes items one at a time from an iterable (an object you can loop over). A generator makes each item only when it is asked for, so it does not hold all the data in memory. This helps with very large files.

### Understand

**The iteration protocol.** An **iterable** has `__iter__`, which returns an **iterator**. An iterator has `__next__`, which returns the next value or raises `StopIteration`. A `for` loop calls `iter()` once, then `next()` until `StopIteration`. Iterators are single-use. Lists are iterables, not iterators.

**Comprehensions** build collections concisely: `[f(x) for x in xs if p(x)]`, `{k: v for ...}`, `{x for ...}`. They have their own scope, so the loop variable does not leak. Prefer them over `map`/`filter` with lambdas. If a comprehension needs nested conditions across several lines, a normal loop reads better.

**Generators** are functions that use `yield`. Calling one returns a generator object (an iterator) and runs lazily, pausing at each `yield` and keeping local state. **Why it matters for backends:** a generator processes an unbounded or very large stream (rows from a database cursor, lines of a file, pages of an API) with constant memory.

```python
def read_events(path):
    with open(path, encoding="utf-8") as f:
        for line in f:                  # file objects iterate lazily
            if line.strip():
                yield json.loads(line)

total = sum(e["amount"] for e in read_events("events.jsonl"))
```

A generator expression `(x for x in xs)` is the lazy form of a list comprehension. Pass it directly to `sum`, `any`, `all`, `max`, `"".join`.

**Generator features.** `yield from sub` delegates to a sub-iterator. `send()` and `throw()` exist but are rarely needed in application code. Closing a generator raises `GeneratorExit` inside it, so `try/finally` cleanup works. Generators are the conceptual ancestor of coroutines (chapter 12).

**itertools** supplies composable lazy tools: `chain`, `islice`, `groupby` (input must be sorted by key), `batched` (3.12+, otherwise write a small chunking helper), `tee`, `accumulate`, `product`, `combinations`, `zip_longest`.

**Unpacking and helpers.** `enumerate(xs, start=1)`, `zip(a, b)` (use `strict=True` in 3.10+ when lengths must match), starred unpacking `first, *rest = xs`, `dict(zip(keys, values))`.

**Pagination as a generator.** Wrapping a paged API in a generator hides pagination from callers and keeps memory flat.

```python
def iter_orders(client):
    cursor = None
    while True:
        page = client.list_orders(cursor=cursor)
        yield from page.items
        cursor = page.next_cursor
        if cursor is None:
            return
```

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Lazy pipelines | generators, generator expressions, `itertools` | Process a multi-GB file in constant memory |
| Chunking | `itertools.islice`, `itertools.batched` (3.12+) | Batch database writes |
| Grouping | `itertools.groupby`, `collections.defaultdict`, `Counter` | Aggregate without pandas |
| Protocol inspection | `iter()`, `next()`, `collections.abc.Iterator` | Implement a custom iterator class |

### Common mistakes

- Iterating a generator twice and getting nothing the second time.
- Materialising with `list(...)` a stream that should stay lazy.
- `groupby` on unsorted data, producing repeated groups.
- Late-binding in comprehension lambdas: `[lambda: i for i in range(3)]` all see the final `i`.

### Self-check

- What methods make an object an iterator versus an iterable?
- Why does a generator use constant memory, and what is the cost?
- What does `yield from` do?
- Why does `sum(x for x in big)` beat `sum([x for x in big])`?

---

## 4. Functions, closures and decorators

**In plain words:** A function is an object, so you can pass it to another function. A decorator is a function that wraps another function to add behaviour, such as retries. You will also learn how Python finds names and how an inner function remembers values from the outer one.

### Understand

**Functions are objects.** They can be passed, stored and returned. This enables callbacks, strategy dispatch (`{"card": pay_card, "bank": pay_bank}[kind]`) and decorators.

**Parameters.** Positional, keyword, defaults, `*args` (tuple of extra positionals), `**kwargs` (dict of extra keywords). Syntax markers: parameters after a bare `*` are keyword-only, parameters before `/` are positional-only (3.8+).

```python
def charge(amount, /, currency="USD", *, idempotency_key):
    ...
```

Keyword-only parameters make call sites self-documenting and let you add parameters without breaking callers. Use them for flags and options.

**Scope: LEGB.** Name lookup goes Local, Enclosing, Global, Built-in. Assigning to a name inside a function makes it local for the whole function. `nonlocal` and `global` rebind outer names. Avoid both in normal code.

**Closures.** An inner function that references variables of the enclosing function keeps those variables alive ("closes over" them). The variable is captured, not its value at definition time, which explains the late-binding surprise in loops. Fix with a default argument (`lambda i=i: i`) or `functools.partial`.

**Decorators.** `@dec` above `def f` means `f = dec(f)`. A decorator takes a function and returns a function that usually adds behaviour: logging, retries, caching, timing, authorisation, metrics.

```python
import functools, time

def retry(times=3, delay=0.5, exceptions=(ConnectionError,)):
    def decorator(fn):
        @functools.wraps(fn)                  # keep name, docstring, signature info
        def wrapper(*args, **kwargs):
            for attempt in range(1, times + 1):
                try:
                    return fn(*args, **kwargs)
                except exceptions:
                    if attempt == times:
                        raise
                    time.sleep(delay * attempt)
        return wrapper
    return decorator

@retry(times=4)
def fetch(url): ...
```

Always apply `functools.wraps`. A decorator factory (a function returning a decorator) is how you pass arguments. A decorator for `async def` functions must itself be async and `await` the call. Retries without backoff and jitter, and on non-idempotent operations (ones that are not safe to repeat), cause incidents.

**Useful `functools`.** `lru_cache` and `cache` memoise pure functions with hashable arguments (beware unbounded growth with `cache`, and on methods the cache holds `self` alive). `partial` pre-binds arguments. `singledispatch` dispatches on argument type. `cached_property` computes an attribute once per instance.

**Lambdas** are single-expression anonymous functions. Use them for short `key=` arguments, not for anything you would want a name or a docstring for.

**Pure functions** (no hidden state, same output for same input) are the easiest to test and to cache. Push side effects to the edges of a service.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Decorators | `functools.wraps`, decorator factories, class-based decorators | Write a retry, timing and cache decorator from memory |
| Caching | `functools.lru_cache`, `cache`, `cached_property` | Know size limits, invalidation and the method pitfall |
| Partial application | `functools.partial`, `operator.itemgetter`, `attrgetter` | Replace trivial lambdas |
| Signatures | `inspect.signature`, `typing.ParamSpec` | Type decorators without losing the wrapped signature |
| Callable design | keyword-only args, `*args`/`**kwargs` forwarding | Design stable function APIs |

### Common mistakes

- Missing `functools.wraps`, which breaks introspection, tracebacks and framework routing.
- `lru_cache` on a function with mutable or unhashable arguments, or on methods of long-lived objects.
- Retrying non-idempotent calls blindly.
- Catching `Exception` inside a decorator and swallowing the error.
- Closures capturing a loop variable.

### Self-check

- What does `@dec` desugar to? What about `@dec(arg)`?
- Why is the mutable default trap a closure-time versus call-time issue?
- What happens to `__name__` of a decorated function without `wraps`?
- What are positional-only and keyword-only parameters for?

---

## 5. Object-oriented Python

**In plain words:** A class describes how to build objects that hold data and methods. Python looks for a method on the object first, then on its parent classes. Prefer small classes that use other objects over deep inheritance.

### Understand

**Classes are objects too** and instances hold state in `__dict__` (unless `__slots__` is used). Attribute lookup on an instance checks the instance, then the class and its bases in **MRO** (method resolution order, the list of classes searched in turn) order (C3 linearisation, see `Class.__mro__`).

**Methods.** Instance methods receive `self`. `@classmethod` receives the class (alternative constructors such as `from_row`). `@staticmethod` receives nothing (a namespaced function, use sparingly). `@property` exposes computed or validated attributes with attribute syntax.

**Dunder methods** let objects integrate with the language: `__init__`, `__repr__` (unambiguous, for developers), `__str__` (readable), `__eq__`, `__hash__`, `__lt__`, `__len__`, `__iter__`, `__contains__`, `__getitem__`, `__enter__`/`__exit__`, `__call__`, `__bool__`. Implement `__repr__` on every domain class, because it is what appears in logs and tracebacks. If you define `__eq__` without `__hash__`, the class becomes unhashable.

**Inheritance versus composition.** Inheritance models "is a", composition models "has a". Prefer composition and small interfaces. Deep hierarchies and mixins that depend on each other are hard to change. `super()` follows the MRO, so cooperative multiple inheritance requires every class to call `super().__init__(...)` consistently.

**Duck typing.** Python cares whether an object supports the operations, not what class it is. Protocols (chapter 7) make this checkable statically.

**Abstract base classes.** `abc.ABC` with `@abstractmethod` prevents instantiating a class that has not implemented the required methods. Use them where you own the hierarchy and want an enforced contract. `collections.abc` supplies ABCs such as `Mapping`, `Sequence`, `Iterable` for type hints and custom containers.

**Class attributes versus instance attributes.** A mutable class attribute is shared by all instances, which is almost never intended. Initialise mutable state in `__init__`.

**Descriptors and metaclasses** underlie `property`, ORMs and validation libraries. You should know the idea (an object with `__get__`/`__set__` placed on a class controls attribute access) and that metaclasses are rarely needed in application code. Prefer class decorators, `__init_subclass__` or plain functions.

**Design for backends.** Keep business logic in plain classes and functions that do not import the web framework or the database driver. Pass dependencies in (constructor or function arguments) so tests can substitute fakes.

```python
class OrderService:
    def __init__(self, repo, payments, clock):
        self._repo, self._payments, self._clock = repo, payments, clock

    def place(self, order):
        order.placed_at = self._clock.now()
        self._payments.authorise(order)
        self._repo.save(order)
```

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Introspection | `__mro__`, `isinstance`, `issubclass`, `vars`, `inspect` | Explain method resolution for a diamond |
| Special methods | dunder methods, `functools.total_ordering` | Make a domain type behave like a built-in |
| Contracts | `abc`, `collections.abc`, `typing.Protocol` | Define and enforce an interface |
| Memory layout | `__slots__` | Know when it helps and what it forbids |
| Properties | `@property`, `cached_property` | Validate and compute without breaking call sites |

### Common mistakes

- Shared mutable class attributes.
- Defining `__eq__` and forgetting hashing consequences.
- Inheriting to reuse code when composition is clearer.
- Catching or checking concrete types where duck typing or a Protocol is correct.
- Business logic coupled to framework request objects, making it untestable.

### Self-check

- How does Python find `obj.method()`? What is the MRO of a diamond hierarchy?
- When is `@classmethod` the right tool?
- What does `super()` return in multiple inheritance?
- Why should every domain class define `__repr__`?

---

## 6. Dataclasses

**In plain words:** A dataclass writes the repetitive methods of a data-holding class for you. It does not check types when the program runs. Use it for simple records, and use a validation library for input from outside.

### Understand

`dataclasses` generates `__init__`, `__repr__` and `__eq__` from annotated fields, removing boilerplate for data-carrying classes such as events, DTOs and configuration.

```python
from dataclasses import dataclass, field
from datetime import datetime
from decimal import Decimal

@dataclass(frozen=True, slots=True)
class Payment:
    id: str
    amount: Decimal
    currency: str = "USD"
    tags: tuple[str, ...] = ()
    created_at: datetime = field(default_factory=datetime.now)
```

**Key options.**
- `frozen=True`: instances are immutable and hashable, safer for values shared across threads or used as dict keys.
- `slots=True` (3.10+): lower memory, faster attribute access.
- `order=True`: generates comparison methods from field order.
- `kw_only=True` (3.10+): all fields keyword-only, which prevents positional mix-ups.
- `field(default_factory=list)`: required for mutable defaults. A bare `= []` raises an error for exactly the reason in chapter 1.
- `__post_init__`: runs after `__init__`, for validation or derived fields. `field(init=False)` excludes a field from the constructor.

**Tools in the standard library:** `dataclasses.asdict`, `astuple`, `replace` (copy with changes, the idiom for frozen objects), `fields`.

**Dataclass versus other options.**
- `NamedTuple` (`typing.NamedTuple`): immutable tuple with named fields, unpackable, minimal overhead.
- `TypedDict`: typed shape of a plain dict, no runtime validation. Good for JSON payloads you pass through.
- **Pydantic** (third party): runtime validation and parsing of untrusted input, widely used in API layers. Dataclasses perform no validation or type coercion, annotations are only hints. Use validation libraries at the system boundary (request bodies, config) and plain dataclasses inside.
- `attrs` (third party): a richer predecessor with more options.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Value objects | `@dataclass(frozen=True, slots=True)`, `replace` | Model events and entities immutably |
| Defaults | `field(default_factory=...)`, `__post_init__` | Avoid shared defaults, validate invariants |
| Lightweight records | `NamedTuple`, `TypedDict` | Choose by mutability and validation needs |
| Boundary validation | `pydantic` (see its docs) | Parse untrusted input into typed objects |
| Serialisation | `asdict`, `json`, custom encoders | Convert `Decimal` and `datetime` explicitly |

### Common mistakes

- Expecting dataclasses to validate or coerce types.
- Mutable defaults (the framework rejects the plain list form, but a shared object via a function default is still possible).
- `frozen=True` containing mutable fields (a list inside is still mutable).
- `json.dumps` on a dataclass directly. It fails, convert with `asdict` and handle `Decimal`/`datetime`.

### Self-check

- What does `@dataclass` generate, and what does `frozen=True` change?
- Why does `field(default_factory=list)` exist?
- When would you choose Pydantic over a dataclass?
- How do you "modify" a frozen instance?

---

## 7. Typing and Protocols

**In plain words:** A type hint is a note that says what kind of value a name should hold. Python does not enforce hints while the program runs. Separate tools read them and report mistakes. A Protocol says which methods an object must have.

### Understand

**Type hints** are annotations that tools check before runtime. CPython does not enforce them. They document intent, power editor completion, and catch a class of bugs (None handling, wrong argument types) in CI. Backend teams that adopt them consistently refactor more safely.

**Syntax in modern Python** (3.10 and later): built-in generics (`list[int]`, `dict[str, float]`, `tuple[int, ...]`), union with `|` (`int | None`), `X | Y` in `isinstance`. `typing.Optional[X]` is the older spelling of `X | None`. Python 3.12 added the `type` statement and generic syntax (`def first[T](xs: list[T]) -> T`). Confirm which features your minimum version supports.

**Common constructs.**
- `Optional`/`| None`: forces callers to handle absence.
- `Callable[[int, str], bool]`, `Awaitable`, `Iterator`, `Iterable`, `Sequence`, `Mapping` (prefer accepting abstract types such as `Iterable` and `Mapping`, and returning concrete ones).
- `TypeVar` and generics for reusable containers and functions.
- `Literal["pending", "paid"]` for closed sets of strings, and `enum.Enum`/`StrEnum` (3.11+) when you need runtime behaviour.
- `TypedDict`, `NamedTuple`, `dataclass` for structured data.
- `Final`, `ClassVar`, `Self` (3.11+), `ParamSpec` for typed decorators, `Annotated` for metadata used by frameworks.
- `Any` disables checking. `object` is the safe "anything". Prefer `object` or a Protocol over `Any`.

**Protocols (structural subtyping).** A `Protocol` describes the methods and attributes a type must have. Any class with those members satisfies it, without inheriting. This is static duck typing, ideal for dependency injection.

```python
from typing import Protocol

class PaymentGateway(Protocol):
    def charge(self, amount_cents: int, token: str) -> str: ...

class FakeGateway:
    def charge(self, amount_cents: int, token: str) -> str:
        return "ch_test"

def checkout(gw: PaymentGateway, amount_cents: int) -> str:
    return gw.charge(amount_cents, "tok")      # FakeGateway type-checks
```

Contrast with ABCs, where a class must declare inheritance (nominal typing). Use `@runtime_checkable` only if you need `isinstance` checks, which verify method presence but not signatures.

**Gradual typing.** You can adopt hints module by module. Annotate public function signatures first, since return types and parameter types give most of the benefit. Do not annotate by writing `Any` everywhere.

**Runtime access.** `typing.get_type_hints` resolves annotations. Frameworks such as FastAPI and Pydantic read annotations at runtime to build validation and documentation, so types are sometimes functional, not only advisory. `from __future__ import annotations` defers annotation evaluation, which can interact with such libraries, so check their documentation.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Annotations | `typing`, `collections.abc`, `typing_extensions` | Annotate a public API fully |
| Structural typing | `typing.Protocol`, `runtime_checkable` | Replace a base class with a Protocol for DI |
| Generics | `TypeVar`, `Generic`, 3.12 generic syntax | Type a reusable repository or cache |
| Closed sets | `Literal`, `Enum`, `StrEnum`, `TypedDict` | Prevent invalid string states |
| Typed decorators | `ParamSpec`, `Callable`, `Concatenate` | Keep signatures through a decorator |

### Common mistakes

- Treating hints as runtime guarantees.
- Overusing `Any` or `# type: ignore` without a reason comment.
- Returning `Optional` and never handling `None` at call sites.
- Annotating mutable parameters with concrete `list` when `Sequence` or `Iterable` is enough.
- Confusing `Protocol` (structural) with `ABC` (nominal).

### Self-check

- What is the difference between `Optional[int]` and a default of `None`?
- When would you choose a Protocol over an ABC?
- Why prefer `Iterable[str]` over `list[str]` for a parameter?
- Does `def f(x: int)` stop `f("a")` from running?

---

## 8. Exceptions and context managers

**In plain words:** An exception is an error signal that stops normal flow until some code handles it. A context manager (the `with` statement) makes sure cleanup always happens. Together they stop programs from leaking files and connections.

### Understand

**Exceptions are the error channel.** Python favours EAFP ("easier to ask forgiveness than permission"): try the operation and handle the failure, rather than checking conditions first. EAFP also avoids race conditions between a check and a use (for example checking that a file exists, then opening it).

**Structure.**

```python
try:
    row = repo.get(order_id)
except NotFoundError:
    raise OrderMissing(order_id) from None
except (TimeoutError, ConnectionError) as exc:
    log.warning("db unavailable", exc_info=exc)
    raise
else:
    process(row)          # runs only if no exception
finally:
    metrics.incr("lookups")   # always runs
```

- Catch the narrowest exception you can handle. Bare `except:` and `except Exception: pass` hide bugs. A bare `except:` also catches `KeyboardInterrupt` and `SystemExit`.
- `raise` (no argument) re-raises preserving the traceback. `raise New(...) from exc` sets an explicit cause, and `from None` suppresses chaining.
- Hierarchy: `BaseException` (system-exiting: `SystemExit`, `KeyboardInterrupt`, `asyncio.CancelledError` in 3.8+) and `Exception` (normal errors). Catching `Exception` does not catch cancellation, which is intentional.
- **Custom exceptions:** define a small hierarchy per domain (`class PaymentError(Exception)`, `class CardDeclined(PaymentError)`). Callers can then catch by category and map to HTTP status codes in one place.
- **Exception groups** (3.11+): `ExceptionGroup` and `except*` handle several concurrent errors, which is how `asyncio.TaskGroup` reports failures.
- `warnings` module for deprecations, `assert` for internal invariants only (assertions are removed under `python -O` and must not validate user input).

**Fail loudly at the boundary, translate at the layer.** Low-level exceptions (driver errors) should be translated to domain errors at the layer boundary, and one top-level handler should convert domain errors to responses and log unexpected ones with a stack trace.

**Context managers** guarantee setup and teardown, which is how you avoid leaked files, connections, locks and transactions. A `with` statement calls `__enter__` and always calls `__exit__`, even when an exception occurs.

```python
from contextlib import contextmanager

@contextmanager
def transaction(conn):
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
```

`contextlib` helpers: `contextmanager`, `closing`, `suppress` (use sparingly), `ExitStack` (a dynamic number of contexts), `nullcontext`, `redirect_stdout`. Async code uses `async with` and `contextlib.asynccontextmanager`. Open resources with `with` every time (`open`, locks, DB sessions, HTTP clients, temporary files).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Handling | `try/except/else/finally`, `raise from`, `ExceptionGroup` | Write precise, minimal handlers |
| Domain errors | custom exception hierarchy | Map exceptions to API responses in one place |
| Resource safety | `with`, `contextlib` (`contextmanager`, `ExitStack`, `closing`) | Never leak a connection or file |
| Diagnostics | `traceback`, `logging.exception`, `sys.excepthook` | Capture full tracebacks in logs |
| Retry logic | `tenacity` (third party) or hand-written backoff | Retry only transient, idempotent operations |

### Common mistakes

- `except Exception: pass`, or logging without the traceback.
- Catching too broadly around large blocks of code.
- Using exceptions for normal control flow in hot loops (raising is comparatively costly).
- Returning `None` on error and forgetting to check it.
- Losing the original cause by raising a new exception without `from`.
- Doing cleanup in `__del__` rather than a context manager.

### Self-check

- What is the difference between `else` and `finally` in `try`?
- Why does `asyncio.CancelledError` derive from `BaseException`?
- What does `raise` without arguments do, and where is it legal?
- Write a context manager that times a block, using both a class and `@contextmanager`.

---

## 9. Modules, packages, environments and dependencies

**In plain words:** A module is one Python file. A package is a folder of modules. A virtual environment gives each project its own set of installed libraries. A lockfile records exact versions, so every build installs the same code.

### Understand

**Modules and packages.** A module is a `.py` file. A package is a directory of modules (an `__init__.py` file is conventional, and namespace packages without it exist). `import x` executes `x` once and caches it in `sys.modules`, so module-level code runs a single time per process. Resolution uses `sys.path`. Use **absolute imports** inside a project and relative imports (`from . import x`) sparingly within a package.

**Circular imports** occur when two modules import each other at load time. Fix by restructuring (extract the shared piece), importing inside a function, or moving type-only imports under `if TYPE_CHECKING:`.

**`if __name__ == "__main__":`** runs code only when the file is executed directly. Run modules with `python -m package.module` so imports resolve from the right root.

**Why virtual environments.** Different projects need different dependency versions. A **virtual environment** is an isolated directory with its own interpreter link and `site-packages`. Never install project dependencies into the system Python. On modern Linux distributions, system Python may refuse global `pip install` (PEP 668), which is intended.

**Tooling landscape.**
- `python -m venv .venv` creates an environment using the standard library. `pip` installs packages from PyPI. `pip freeze` lists exact installed versions.
- **uv** (from Astral) is a fast package and project manager that handles Python versions, environments, dependency resolution and lockfiles. It is a drop-in accelerator for many `pip` workflows and also offers a project workflow (`uv init`, `uv add`, `uv sync`, `uv run`). `uv init`, `uv run`, `uv lock`, `uv sync` and the `uv_build` backend are confirmed in the uv docs. Check the docs for current options.
- Other established tools: `pip-tools` (`pip-compile` for pinned requirements), `Poetry`, `PDM`, `Hatch`, `pyenv` (interpreter versions).
- **`pyproject.toml`** is the standard project file (PEP 518, PEP 621). It declares build system, project metadata, dependencies, optional dependency groups and tool configuration (`[tool.ruff]`, `[tool.pytest.ini_options]`, `[tool.mypy]`).

```toml
[project]
name = "orders-service"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
  "httpx>=0.27,<1",
  "pydantic>=2,<3",
]

[project.optional-dependencies]
dev = ["pytest", "ruff", "mypy"]

[tool.pytest.ini_options]
testpaths = ["tests"]
```

(Version bounds above are illustrative. Choose bounds that match your own testing.)

**Reproducibility.** There are two kinds of dependency specification. **Abstract** ranges in `pyproject.toml` say what the code is compatible with. A **lockfile** or pinned requirements file records the exact resolved versions, including transitive dependencies, ideally with hashes. Applications (services) should deploy from a lockfile. Libraries declare ranges and are tested against several versions.

**Supply chain hygiene.** Pin and review dependencies, scan them (`pip-audit`), prefer well-maintained packages, and be wary of typosquatted names. Use hash-checked installs in production builds where possible (`pip install --require-hashes`, consult pip documentation).

**Project layout.** The `src/` layout (`src/package/`) prevents tests from accidentally importing the working directory instead of the installed package. Install the project in editable mode during development (`pip install -e .`).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Environments | `python -m venv`, `uv venv`, `pyenv` | Create and recreate an environment from scratch |
| Installing | `pip install`, `pip list`, `pip show`, `pip freeze`, `uv pip`, `uv add`, `uv sync` | Resolve and install reproducibly |
| Locking | `uv lock`, `pip-compile` (pip-tools), `poetry lock` | Produce and use a lockfile in CI and builds |
| Metadata | `pyproject.toml`, `requires-python`, optional dependencies | Write one from memory |
| Diagnosing imports | `python -m site`, `sys.path`, `pip check`, `python -X importtime` | Explain "module not found" and wrong-version bugs |
| Security | `pip-audit`, Dependabot or Renovate | Detect vulnerable dependencies |

### Common mistakes

- Installing into system Python or using `sudo pip`.
- Committing `.venv` to version control or not committing a lockfile for an application.
- Unpinned dependencies leading to a different build every day.
- Putting dev tools and runtime dependencies in the same list for production images.
- Shadowing standard library names (a local file called `json.py` or `random.py`).
- Running scripts with a different interpreter than the one the environment installed packages into.

### Self-check

- What does a virtual environment contain, and why is it needed?
- Difference between `pyproject.toml` ranges and a lockfile? Which does CI deploy from?
- How do you debug `ModuleNotFoundError` when the package is installed?
- Why does the `src/` layout exist?

---

## 10. Static type checking

**In plain words:** A type checker reads your code without running it and reports type mistakes. It finds bugs early, in the editor or in CI (the automatic checks that run on each change). It cannot check data that arrives from outside at run time.

### Understand

A type checker reads annotations without running the code and reports inconsistencies. It runs in CI and in the editor. The two mainstream tools are **mypy** (the reference implementation, configurable and widely used) and **pyright** (fast, strong inference, powers Pylance in VS Code). Newer checkers such as `pyrefly` and `ty` are also listed on typing.python.org. Their maturity and rule coverage are unverified here, so evaluate them on your code before adopting.

**Adoption strategy for an existing codebase.**
1. Start with the checker in a lenient mode on new or critical modules.
2. Annotate function signatures at module boundaries.
3. Raise strictness gradually (`--strict` in mypy, `strict` in pyright), module by module, using per-module overrides.
4. Fail CI on new errors. Track `# type: ignore` counts and require a specific error code and a reason.

**What it catches in backend code:** attribute access on `None`, wrong argument types, missing return paths, calling a function with a removed parameter after a refactor, inconsistent dictionary shapes (`TypedDict`), unhandled enum or `Literal` cases (with `assert_never` for exhaustiveness).

**Limits.** It cannot validate data coming from outside (JSON, database rows, environment variables). Combine with runtime validation at the boundary. Untyped third-party libraries need stubs (`types-*` packages) or `ignore_missing_imports` scoped narrowly.

**Narrowing.** `if x is not None`, `isinstance`, `Literal` comparisons and `TypeGuard`/`TypeIs` refine types inside branches. Understanding narrowing explains most checker complaints.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Checking | `mypy`, `pyright` (CLI or Pylance) | Configure in `pyproject.toml`, run in CI |
| Strict mode | `mypy --strict`, per-module overrides | Migrate a module to strict |
| Third-party types | `types-*` stub packages, `py.typed` marker | Resolve missing-stub errors correctly |
| Exhaustiveness | `typing.assert_never`, `Literal`, enums | Make unhandled cases a type error |
| Editor integration | Pylance, mypy daemon (`dmypy`) | Get feedback in seconds |

### Common mistakes

- Blanket `# type: ignore` or `ignore_errors` for whole packages.
- Believing a passing check means validated runtime data.
- Running the checker with a different Python version or environment than production.
- Chasing "100% coverage" instead of typing the interfaces that matter.

### Self-check

- What kinds of bugs does a static checker catch that unit tests often miss?
- How would you introduce mypy to a 50,000-line untyped service?
- Why can't type hints validate an incoming JSON body?

---

## 11. Concurrency: the GIL, threads, processes

**In plain words:** Concurrency means making progress on many tasks in the same period. In standard CPython, the GIL lets only one thread run Python code at a time. Choose threads, processes or asyncio by asking whether the work mostly waits or mostly computes.

### Understand

**The problem classes.**
- **I/O-bound**: time is spent waiting (network, database, disk). Most API backends are here.
- **CPU-bound**: time is spent computing (parsing, hashing, image or numeric work).

**The GIL.** CPython has a **Global Interpreter Lock**: only one thread executes Python bytecode at a time within one process. It exists to make reference counting and interpreter internals safe. Consequences:
- Threads do **not** speed up CPU-bound pure-Python code.
- Threads **do** help I/O-bound work, because the lock is released while a thread waits on blocking I/O, and by many C extensions during long computations (NumPy, hashing, compression, many database drivers).
- The GIL does not make your code thread-safe. `counter += 1` is several bytecode steps and can interleave. Compound operations need locks.
- Python 3.13 introduced an experimental **free-threaded build** (PEP 703) with no GIL. Python 3.14 lists it as officially supported (PEP 779), but it is still a separate, optional build. Sub-interpreters have had a per-interpreter GIL since 3.12 (PEP 684, C API only), and 3.14 exposes them to Python code through `concurrent.interpreters` (PEP 734). Many third-party extension modules are not yet compatible with either. Check the release notes before relying on them in production.

**The four models.**

| Model | Parallelism | Overhead | Best for | Watch out for |
|---|---|---|---|---|
| Threads (`threading`, `ThreadPoolExecutor`) | I/O overlap, not CPU | low | blocking libraries, moderate concurrency | races, locks, GIL on CPU work |
| Processes (`multiprocessing`, `ProcessPoolExecutor`) | true CPU parallelism | high (memory, pickling) | CPU-bound batch work | serialisation cost, shared state, start method |
| `asyncio` | I/O overlap in one thread | very low per task | thousands of concurrent connections | any blocking call stalls everything |
| Multiple worker processes (gunicorn, uvicorn workers) | parallel requests | per process | scaling a web service across cores | per-process state, connection pools per worker |

**Decision guide.**
1. Is it I/O-bound with libraries that are async-capable and a high concurrency need? Use `asyncio`.
2. I/O-bound but using blocking libraries, or modest concurrency? Use a thread pool.
3. CPU-bound? Use a process pool, a task queue with worker processes, or move the hot loop to a C-backed library (NumPy, a compiled extension) so the GIL is released.
4. Scaling a service across cores: run several worker processes behind a server and scale out horizontally.

**concurrent.futures** gives a uniform interface: `ThreadPoolExecutor`, `ProcessPoolExecutor`, `submit`, `map`, `as_completed`, with futures that carry results and exceptions.

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

with ThreadPoolExecutor(max_workers=16) as pool:
    futures = {pool.submit(fetch, u): u for u in urls}
    for fut in as_completed(futures):
        try:
            handle(fut.result())
        except Exception as exc:
            log.error("failed %s: %s", futures[fut], exc)
```

**Threading primitives:** `Lock`, `RLock`, `Semaphore`, `Event`, `Condition`, `queue.Queue` (thread-safe, the preferred way to pass work between threads), `threading.local`. Prefer message passing through queues over shared mutable state.

**Multiprocessing notes.** Arguments and results are **pickled** (converted to bytes to cross the process boundary), so they must be picklable and large payloads are costly. Start methods (`fork`, `spawn`, `forkserver`) differ by platform and version, and `fork` with threads or open connections is hazardous. Code that starts processes needs the `if __name__ == "__main__":` guard on platforms using `spawn`. Never share a database connection or socket across a fork.

**Graceful shutdown.** Handle `SIGTERM` (see the Linux chapter) to stop accepting work, finish in-flight tasks and close pools. In containers, make sure the Python process is the one receiving the signal.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Thread pools | `concurrent.futures.ThreadPoolExecutor`, `threading`, `queue` | Parallelise blocking I/O safely |
| Process pools | `ProcessPoolExecutor`, `multiprocessing`, `joblib` | Parallelise CPU work and size the pool |
| Synchronisation | `Lock`, `Semaphore`, `Event`, `queue.Queue` | Identify and fix a race condition |
| Servers | `gunicorn`, `uvicorn` (workers), process managers | Choose worker model and count for a service |
| Task queues | Celery, RQ, Dramatiq, or cloud queues | Move slow or CPU work off the request path |
| Signals | `signal` module | Implement graceful shutdown |

### Common mistakes

- Using threads for CPU-bound Python and expecting a speed-up.
- Assuming the GIL removes the need for locks.
- Unbounded thread or process creation instead of fixed pools.
- Sharing connections, clients or random-number state across forked processes.
- Pickling errors from lambdas, open files or locks sent to workers.
- Running CPU work inside a request handler and starving other requests.

### Self-check

- What does the GIL protect, and what does it not protect?
- Why do threads still help for HTTP calls?
- How do you scale CPU-bound work across 8 cores?
- What is the difference between running 4 worker processes and 4 threads for a web service?

---

## 12. asyncio in depth

**In plain words:** asyncio runs many tasks in one thread. A task hands control back to the event loop when it waits with `await`. If a task never waits, or calls a blocking function, all other tasks stop.

### Understand

**The mental model.** `asyncio` runs many tasks on **one thread** with a **cooperative event loop**. A coroutine runs until it reaches an `await` on something not ready, then yields control, and the loop runs another ready task. The loop uses OS readiness notification (epoll, kqueue) to know when sockets are ready. Nothing preempts a running coroutine. **If a coroutine does not `await`, nothing else runs.** This is both the efficiency and the hazard.

**Vocabulary.**
- **Coroutine function**: `async def`. Calling it returns a coroutine object and runs nothing yet.
- **`await`**: run an awaitable and suspend until it completes.
- **Task**: a coroutine scheduled on the loop (`asyncio.create_task`). It starts running concurrently without waiting for `await` on it.
- **Future**: a low-level placeholder for a result.
- **Event loop**: started with `asyncio.run(main())`, which creates the loop, runs `main`, and closes it.

```python
import asyncio, httpx

async def fetch(client, url):
    r = await client.get(url, timeout=5)
    r.raise_for_status()
    return r.json()

async def main(urls):
    async with httpx.AsyncClient() as client:
        results = await asyncio.gather(*(fetch(client, u) for u in urls))
    return results

asyncio.run(main(["https://example.test/a", "https://example.test/b"]))
```

Awaiting coroutines one after another is **sequential**. Concurrency comes from creating tasks or using `gather`/`TaskGroup`.

**Structured concurrency (3.11+).** `asyncio.TaskGroup` runs tasks together, waits for all, and if one fails cancels the rest and raises an `ExceptionGroup`. It prevents orphaned tasks.

```python
async with asyncio.TaskGroup() as tg:
    t1 = tg.create_task(load_user(uid))
    t2 = tg.create_task(load_orders(uid))
user, orders = t1.result(), t2.result()
```

**gather versus TaskGroup.** `gather(*coros)` collects results in order. By default the first exception propagates to the caller, but the other awaitables are not cancelled and keep running. With `return_exceptions=True` exceptions are returned as results. `TaskGroup` gives safer failure semantics and is preferred on 3.11 and later.

**Cancellation.** `task.cancel()` raises `asyncio.CancelledError` inside the task at its next `await`. Rules:
- Do not swallow `CancelledError`. If you catch it to clean up, re-raise it.
- Use `try/finally` or `async with` for cleanup, which runs on cancellation.
- Timeouts: `async with asyncio.timeout(5):` (3.11+), or `asyncio.wait_for(coro, timeout)`. A timeout cancels the inner work. Every external call in a service needs a timeout, because the default for most network clients is none or very long.
- Shield critical sections only with care (`asyncio.shield`).
- Keep a **reference** to tasks you create. The loop holds only weak references, so a fire-and-forget task with no stored reference can be garbage collected mid-run. Store it in a set, or use a `TaskGroup`.

**Synchronisation and limiting.** `asyncio.Semaphore` bounds concurrency (for example at most 20 concurrent outbound calls), `asyncio.Lock` for async-aware mutual exclusion, `asyncio.Queue` for producer/consumer pipelines with back-pressure (producers slow down when the queue is full; set `maxsize`), `asyncio.Event`. Unbounded `gather` over 100,000 items opens 100,000 sockets, so bound it.

**The blocking-call pitfall.** Any synchronous call inside a coroutine blocks the entire loop, including all other requests: `time.sleep`, `requests.get`, a synchronous database driver, heavy JSON parsing, CPU loops, `subprocess.run`, reading large files. Symptoms: sudden latency spikes across all endpoints, health checks timing out. Remedies:
- Use async-native libraries (`httpx`, `aiohttp`, `asyncpg`, async SQLAlchemy, `aiofiles`).
- Offload blocking functions: `await asyncio.to_thread(fn, *args)` (3.9+) or `loop.run_in_executor` with a thread pool. Use a process pool for CPU-bound work.
- Replace `time.sleep` with `await asyncio.sleep`.
- Enable asyncio debug mode (`PYTHONASYNCIODEBUG=1`, `python -X dev`, or `asyncio.run(..., debug=True)`). It logs callbacks slower than `loop.slow_callback_duration` (default 0.1 seconds), raises errors for thread-unsafe calls from the wrong thread, and adds creation tracebacks to "never awaited" warnings. Use `logging.basicConfig(level=logging.DEBUG)` to see the messages.

**Async iteration and context managers.** `async for` over async iterators and generators (`async def` with `yield`), `async with` for connection pools, sessions and transactions.

**When asyncio is the right choice.** Many concurrent, mostly idle connections (API gateways, websockets, fan-out calls to several services). It adds little for a service with modest traffic and CPU-heavy work, and it forces async libraries throughout the stack. A mixed codebase (sync and async) needs deliberate boundaries.

**Framework link.** ASGI servers (servers for async web applications: uvicorn, hypercorn) run async frameworks (FastAPI, Starlette). In such frameworks, a plain `def` endpoint is typically executed in a thread pool, while an `async def` endpoint runs on the loop and must not block. Confirm the exact behaviour in your framework's documentation.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Running | `asyncio.run`, `create_task`, `TaskGroup`, `gather`, `wait` | Fan out calls and handle partial failure |
| Time limits | `asyncio.timeout`, `wait_for`, `asyncio.sleep` | Put a deadline on every external call |
| Limiting | `Semaphore`, `Queue(maxsize=...)` | Bound concurrency and apply back-pressure |
| Offloading | `asyncio.to_thread`, `run_in_executor` | Isolate blocking and CPU code from the loop |
| Libraries | `httpx`, `aiohttp`, `asyncpg`, `anyio` | Use an async-native client correctly |
| Debugging | asyncio debug mode, `asyncio.all_tasks()`, `py-spy dump` | Find a blocked loop or a leaked task |
| Testing | `pytest-asyncio` or `anyio` pytest plugin | Test coroutines and cancellation |

### Common mistakes

- Calling a coroutine without `await` (it never runs, and Python warns "coroutine was never awaited").
- Blocking calls inside `async def`.
- Using `gather` over an unbounded list.
- Swallowing `CancelledError`, which makes shutdown hang.
- Fire-and-forget tasks with no reference or error handling (exceptions are only reported when the task is collected).
- No timeouts on network calls.
- Creating a new HTTP client per request instead of reusing one with a connection pool.
- Mixing `asyncio.run` inside an already running loop.

### Self-check

- What is the difference between a coroutine and a task?
- Why does one `time.sleep(1)` in a handler hurt every request?
- What does `gather` do when one task raises? How does `TaskGroup` differ?
- How would you limit a fan-out to 10 concurrent calls and cancel everything after 30 seconds?
- What is the safe way to call a blocking function from async code?

---

## 13. Testing with pytest

**In plain words:** A test is code that checks that other code works. pytest finds functions named `test_*` and runs them. Good tests are fast, independent of each other, and check behaviour rather than internal details.

### Understand

**Why tests, and what kind.** Tests turn requirements into executable checks and make refactoring safe. A balanced backend suite: many fast **unit tests** (pure logic, no I/O), fewer **integration tests** (real database, real queue, often in containers), and a small number of end-to-end checks. Fast feedback matters more than coverage numbers.

**pytest basics.** Plain functions named `test_*` in files named `test_*.py`, plain `assert` statements with rich failure output (assertion rewriting), no base class needed.

```python
import pytest

def test_total_includes_tax():
    assert total(100, tax_rate=0.2) == 120

def test_negative_amount_rejected():
    with pytest.raises(ValueError, match="amount"):
        total(-1)
```

**Fixtures.** A fixture is a function decorated with `@pytest.fixture` that provides setup (and teardown through `yield`) and is injected by parameter name. Scope controls reuse: `function` (default), `class`, `module`, `session`. Put shared fixtures in `conftest.py`. Fixtures can depend on other fixtures, which builds composable test environments such as app, client and database.

```python
@pytest.fixture
def db():
    conn = make_test_connection()
    yield conn
    conn.rollback()
    conn.close()
```

Built-in fixtures: `tmp_path`, `monkeypatch`, `capsys`, `caplog`, `request`.

**Parametrization.** Run one test over many inputs, each reported separately.

```python
@pytest.mark.parametrize(
    "amount, expected",
    [(0, "free"), (100, "standard"), (10_000, "premium")],
)
def test_tier(amount, expected):
    assert tier(amount) == expected
```

**Mocking and test doubles.**
- A **stub** returns canned data, a **fake** is a working lightweight implementation (in-memory repository), a **mock** records calls so you can assert on them.
- `unittest.mock` (`Mock`, `MagicMock`, `patch`, `AsyncMock`) and `pytest`'s `monkeypatch` replace dependencies. **Patch where the name is looked up**, not where it is defined (`patch("app.service.requests.get")`, not `patch("requests.get")`, if `app.service` imported `requests`).
- Use `autospec=True` or `create_autospec` so mocks fail if called with a wrong signature.
- Prefer fakes and dependency injection (chapters 5 and 7) over heavy patching. Heavy mocking couples tests to implementation and gives false confidence. Mock at system boundaries (HTTP, clock, payment gateway), not your own pure logic.
- Freeze or inject time (`freezegun`, `time-machine`, or a `clock` dependency). Do not depend on the real clock or random values.

**Markers and selection.** `@pytest.mark.skip`, `skipif`, `xfail`, custom markers (`slow`, `integration`) registered in config, selected with `-m`. Run a subset with `-k expression`, a single test with `path::test_name`.

**Useful options.** `-x` stop at first failure, `--lf` rerun last failures, `-q`, `-vv`, `--durations=10` show slowest tests, `--pdb` drop into the debugger on failure, `-p no:randomly` if using random ordering.

**Plugins.** `pytest-cov` (coverage), `pytest-xdist` (`-n auto` parallel), `pytest-asyncio` or `anyio` (async tests), `pytest-mock` (`mocker` fixture), `hypothesis` (property-based testing), `pytest-randomly` (order randomisation, exposes hidden test coupling).

**Integration testing with real dependencies.** Use a real database (often through containers, for example `testcontainers`) for tests that touch SQL, because mocking a database misses query, transaction and migration bugs. Isolate tests with transaction rollback or per-test schemas.

**Test design.** One behaviour per test, descriptive names, Arrange-Act-Assert structure, no dependence on execution order, no shared mutable global state, deterministic data. A flaky test is a bug in the test or in the code, so fix or quarantine it, never ignore it.

**Coverage.** Measures which lines run, not whether behaviour is verified. Use it to find untested areas, not as a target to game. Branch coverage is more informative than line coverage.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Runner | `pytest`, `conftest.py`, markers, `-k`, `-x`, `--lf` | Run and select tests fluently |
| Fixtures | `@pytest.fixture`, scopes, `yield`, `tmp_path`, `monkeypatch` | Design layered, reusable fixtures |
| Data-driven | `@pytest.mark.parametrize`, `pytest.param`, `ids` | Cover edge cases compactly |
| Mocking | `unittest.mock.patch`, `AsyncMock`, `autospec`, `pytest-mock` | Patch the right target, assert calls |
| Async | `pytest-asyncio`, `anyio` | Test coroutines and timeouts |
| Quality | `pytest-cov`, `pytest-xdist`, `hypothesis` | Measure coverage, parallelise, find edge cases |
| Integration | `testcontainers`, Docker Compose, transaction rollback | Test against a real database |
| HTTP testing | framework test clients, `respx`/`responses` | Test endpoints without a network |

### Common mistakes

- Patching the wrong import path.
- Over-mocking so tests pass while production breaks.
- Tests that depend on order, on the clock, or on shared state.
- Large session-scoped fixtures that leak state between tests.
- Asserting on implementation details (private calls) rather than behaviour.
- Chasing a coverage percentage with assertion-free tests.
- No test for the failure path (timeouts, retries, partial failures).

### Self-check

- Where do you patch `requests.get` if `service.py` does `import requests`? What if it does `from requests import get`?
- What is a fixture scope, and what goes wrong with a wide scope?
- When would you use a fake instead of a mock?
- How do you test code that depends on the current time?

---

## 14. Logging

**In plain words:** Logging records what a program does while it runs, with a level of importance for each message. Services write structured logs to standard output so that tools can search them. Never write secrets into logs.

### Understand

**Why not `print`.** `logging` provides severity levels, structured context, routing to multiple destinations, runtime configuration and per-module control. It is the standard way services emit diagnostics.

**Model.** **Loggers** form a dot-separated hierarchy (`app.payments` is a child of `app`). A logger creates **records**, **handlers** send them somewhere (stream, file, syslog, HTTP), **formatters** shape them, and **filters** drop or enrich them. A record propagates up to ancestor handlers. Libraries create loggers but never configure handlers, and **only the application's entry point configures logging**.

```python
import logging

log = logging.getLogger(__name__)      # one logger per module

def charge(order_id):
    log.info("charging order", extra={"order_id": order_id})
    try:
        ...
    except GatewayError:
        log.exception("charge failed", extra={"order_id": order_id})
        raise
```

**Levels.** `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL`. Use `INFO` for business-significant events, `WARNING` for recoverable surprises, `ERROR` for failed operations. `log.exception(...)` inside an `except` block logs at ERROR with the traceback.

**Lazy formatting.** Write `log.info("order %s", order_id)`, not an f-string. The arguments are formatted only if the record is emitted, and log aggregators can group by message template.

**Production practices.**
- Log to **stdout/stderr** and let the platform collect (twelve-factor, see the Linux chapter).
- Use **structured logging** (JSON) with stable field names: timestamp, level, logger, message, request or trace ID, user or tenant ID where allowed. Options include `python-json-logger`, `structlog`, or a custom formatter.
- Propagate a **correlation ID** through requests with `contextvars` (which work correctly across asyncio tasks, unlike thread-locals) and attach it with a filter or processor.
- Configure with `logging.config.dictConfig` or the framework's mechanism, from environment-driven settings (level per environment).
- **Never log secrets**, tokens, full card numbers or sensitive personal data. Redact at the formatter or filter level.
- Control volume: do not log inside tight loops, de-duplicate repetitive errors, and sample noisy debug logs. A single error loop can fill a disk and hide the real failure (see the Linux chapter's log hygiene).
- Pair logs with **metrics** (counters, histograms) and **traces** (OpenTelemetry). Logs explain individual events, metrics show trends, traces show request paths.
- Asynchronous and non-blocking handlers: file and network handlers can block the event loop, so use `QueueHandler` and `QueueListener` to move I/O to a background thread when needed.

**Warnings and `exc_info`.** Pass `exc_info=True` to log a traceback outside an except block. `stack_info=True` logs the current stack.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Core | `logging`, `getLogger(__name__)`, levels, handlers, formatters | Configure logging for a service entry point |
| Configuration | `logging.config.dictConfig`, environment-driven level | Change levels without code edits |
| Structured logs | `structlog`, `python-json-logger`, `contextvars` | Emit JSON with a request ID on every line |
| Non-blocking | `QueueHandler`, `QueueListener` | Keep logging off the hot path |
| Observability | OpenTelemetry Python SDK, Prometheus client, Sentry | Connect logs, metrics and traces |
| Test | `caplog` | Assert that a failure is logged |

### Common mistakes

- Calling `logging.basicConfig` in library code, or configuring handlers in many places (duplicate lines).
- f-strings in log calls, or logging full request bodies containing secrets.
- `log.error(str(exc))` which drops the traceback, instead of `log.exception`.
- Logging and re-raising at every layer, producing the same error many times.
- Using thread-locals for request context in async code.

### Self-check

- Why should libraries not configure handlers?
- What does `propagate` do in the logger hierarchy?
- Why use `%s` arguments instead of f-strings in log calls?
- How do you attach a request ID to every log line in an async service?

---

## 15. Profiling and performance

**In plain words:** Profiling measures where a program spends time or memory. Measure first, change one thing, then measure again. Most slow web services wait on databases or networks, not on Python itself.

### Understand

**Method first.** Measure, do not guess. Establish a baseline, find the real bottleneck, change one thing, measure again. Most slow backend endpoints are slow because of I/O (queries, N+1 patterns (one query, then one more query for each row), remote calls, missing indexes, missing connection pooling), not because of Python's interpretive overhead. Look at the database query and external call count before micro-optimising loops.

**What to measure.**
- **Latency percentiles** (p50, p95, p99), throughput, error rate, saturation. Averages hide tail problems.
- **CPU time versus wall time**: wall time includes waiting. A function with high wall time and low CPU time is waiting on I/O.

**CPU profiling.**
- **cProfile** (standard library, deterministic, adds overhead): `python -m cProfile -o out.prof script.py`, inspect with `pstats` or a viewer such as `snakeviz`. Read **tottime** (time in the function itself) versus **cumtime** (including callees).
- **py-spy** (third party, sampling): attaches to a running process without code changes or restart, with low overhead. `py-spy top --pid <pid>` for a live view, `py-spy dump --pid <pid>` for the current stack of every thread (excellent for hangs and blocked event loops), `py-spy record -o profile.svg --pid <pid>` for a flame graph. On Linux, attaching to a process you did not start usually needs root (`sudo`), and macOS always needs root. In Docker, add `--cap-add SYS_PTRACE` to the container, because the default profile blocks the `process_vm_readv` system call (per the py-spy README).
- Others: `scalene`, `pyinstrument` (statistical, readable call tree), `line_profiler` (per-line timing of chosen functions), `perf` with Python 3.12 perf-map support (check the docs).

**Memory profiling.**
- **tracemalloc** (standard library): take snapshots, compare them, and see which lines allocated the growth.

```python
import tracemalloc
tracemalloc.start()
# ... run workload ...
snap = tracemalloc.take_snapshot()
for stat in snap.statistics("lineno")[:10]:
    print(stat)
```

- **memray** (third party), **memory_profiler**, `objgraph` (find what holds references), `gc` introspection.
- **Common Python leaks:** unbounded caches or module-level dicts, `lru_cache` without `maxsize`, lists that only grow, reference cycles holding large objects, listeners never removed, closures capturing big objects, loaded-everything-into-memory (use generators and streaming). Also remember that memory returned to the allocator is not always returned to the OS, so RSS (resident set size, the memory a process holds in RAM) may not fall after objects are freed.
- Container limits: a process killed with signal 9 and no traceback is often an OOM kill (Linux chapter).

**Typical optimisations, in order of payoff.**
1. Fix algorithmic complexity and data structure choice (set lookups, avoid O(n^2)).
2. Reduce I/O: batch queries, fix N+1, add indexes, cache with explicit invalidation, reuse connections (pools, `Session`/client reuse), enable compression where useful.
3. Stream instead of loading (generators, chunked reads).
4. Use the right concurrency model (chapters 11 and 12).
5. Push hot numeric loops to C-backed libraries (NumPy, pandas, compiled extensions) or consider Cython, Rust extensions or PyPy for specific workloads.
6. Micro-optimisations (local variable binding, comprehensions over loops, `str.join`) matter only on proven hot paths.

**Startup and import time.** `python -X importtime` shows slow imports, which matters for serverless and CLI tools.

**Benchmarks.** `timeit` for micro-benchmarks, `pytest-benchmark` for tracked benchmarks, and load tools (`locust`, `k6`, `wrk`, `hey`) for services. Benchmark with realistic data and warm caches, and on hardware comparable to production.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Deterministic CPU profile | `cProfile`, `pstats`, `snakeviz` | Find the top cumulative-time functions in a script |
| Production sampling | `py-spy top`, `dump`, `record` | Diagnose a live process without restart |
| Line-level | `line_profiler`, `pyinstrument`, `scalene` | Zoom into one slow function |
| Memory | `tracemalloc`, `memray`, `objgraph`, `gc` | Locate the allocation site of a leak |
| Micro-benchmarks | `timeit`, `pytest-benchmark` | Compare two implementations fairly |
| Load testing | `locust`, `k6`, `wrk`, `hey` | Find a service's saturation point |
| Database side | `EXPLAIN ANALYZE`, ORM query logging, APM tracing | Count and time queries per request |

### Common mistakes

- Optimising without a profile, or profiling a tiny dataset that hides the real cost.
- Profiling under cProfile and trusting absolute numbers (overhead distorts them).
- Ignoring I/O and database time because the profile shows Python code.
- Caches without size limits or invalidation.
- Reading average latency only.
- Benchmarking once, on a cold start, on a laptop.

### Self-check

- How do you diagnose an API that is slow only under load?
- What is the difference between `tottime` and `cumtime`?
- A process's memory grows steadily. What are your first three steps?
- How can `py-spy dump` reveal a blocked asyncio loop?
- Why do N+1 queries dominate so many Python backend slowdowns?

---

## 16. Standard library essentials

**In plain words:** The standard library is the set of modules that ship with Python. It covers paths, JSON, dates, processes and security helpers. Learn it well before you add outside packages.

### Understand

The standard library covers most backend glue work without third-party packages. Know these well.

**pathlib.** Object-oriented paths, replacing string manipulation with `os.path`.

```python
from pathlib import Path

root = Path(__file__).resolve().parent
cfg = root / "config" / "app.toml"
text = cfg.read_text(encoding="utf-8")
for p in (root / "logs").glob("*.log"):
    print(p.name, p.stat().st_size)
```

Use `/` to join, `.exists()`, `.mkdir(parents=True, exist_ok=True)`, `.rglob()`, `.suffix`, `.stem`. For writes that must not leave partial files, write to a temporary file in the same directory and `os.replace` it into place (atomic on the same filesystem).

**collections.** `defaultdict(list)` for grouping, `Counter` for frequencies (`most_common`), `deque(maxlen=n)` for a sliding window, `namedtuple`, `ChainMap` for layered configuration.

**itertools and functools.** Covered in chapters 3 and 4.

**json.** `json.loads`/`dumps`, `load`/`dump` for files. It does not serialise `datetime`, `Decimal`, `set` or dataclasses by default, so supply `default=` or convert beforehand. Use `separators=(",", ":")` for compact output and `ensure_ascii=False` for readable Unicode. Never use `pickle` on untrusted data, since unpickling can execute arbitrary code. For high-volume JSON, faster third-party libraries exist (`orjson`).

**datetime, zoneinfo, time.**
- **Naive** datetimes have no timezone. **Aware** datetimes do. Mixing them raises errors, and naive values cause silent bugs.
- Store and compute in **UTC**, convert to a user's zone only for display. Use `datetime.now(timezone.utc)`. (`datetime.utcnow()` returns a naive value and is deprecated in newer versions.)
- `zoneinfo.ZoneInfo("Europe/Berlin")` (3.9+) uses the IANA time zone database. It handles daylight saving transitions. On Windows you may need the `tzdata` package.
- Use `timedelta` for durations, `time.monotonic()` or `time.perf_counter()` for measuring elapsed time (wall-clock can jump), and ISO 8601 strings (`isoformat()`, `fromisoformat`) for interchange.
- Pitfalls: adding `timedelta(days=1)` across a DST change on an aware local time, `strftime` locale dependence, and comparing naive with aware values.

**subprocess.** Run external programs.

```python
import subprocess

result = subprocess.run(
    ["git", "rev-parse", "HEAD"],     # list, not a shell string
    capture_output=True, text=True, check=True, timeout=10,
)
print(result.stdout.strip())
```

Pass arguments as a **list** and avoid `shell=True` with any user-influenced input, which creates command injection. Use `check=True` to raise on non-zero exit, always set a `timeout`, and read output with `capture_output`. For streaming output use `Popen`. In asyncio use `asyncio.create_subprocess_exec`, never blocking `subprocess.run`.

**Others to know.**
- `os`, `sys`, `shutil`, `tempfile` (`TemporaryDirectory`, `NamedTemporaryFile`).
- `argparse` for CLIs (or `click`/`typer`), `tomllib` (3.11+, read TOML), `configparser`.
- `re`: compile patterns used repeatedly, prefer simple string methods when they suffice, beware catastrophic backtracking on untrusted input.
- `hashlib`, `hmac` (use `hmac.compare_digest` for constant-time comparison), `secrets` for tokens (never `random` for security), `uuid`, `base64`.
- `sqlite3` for tests and tooling, `csv`, `gzip`, `zipfile`.
- `urllib.request` exists, but `httpx` or `requests` are the practical choices for HTTP clients.
- `enum`, `abc`, `typing`, `contextlib`, `weakref`, `operator`.
- `random` is deterministic and not secure. `secrets` is.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Files and paths | `pathlib`, `shutil`, `tempfile`, `os.replace` | Do file work without string paths, write atomically |
| Data structures | `collections`, `heapq`, `bisect`, `itertools` | Replace ad hoc loops with the right primitive |
| Serialisation | `json`, `csv`, `tomllib`, never `pickle` for untrusted data | Round-trip data with custom types handled |
| Time | `datetime`, `zoneinfo`, `time.monotonic` | Store UTC, convert at the edge, avoid naive values |
| Processes | `subprocess.run` (list args, `check`, `timeout`) | Call external tools safely |
| Security primitives | `secrets`, `hmac`, `hashlib` | Generate tokens and compare signatures correctly |
| CLI | `argparse`, `typer`, `click` | Ship an admin script with proper arguments |

### Common mistakes

- Naive datetimes, local-time storage, or `utcnow()` in new code.
- `shell=True` with interpolated strings.
- `json.dumps` of `Decimal` and `datetime` without a plan.
- Using `random` for tokens or `==` for signature comparison.
- Reading a whole large file with `read()` instead of iterating.
- Measuring durations with `time.time()` instead of `time.monotonic()`.

### Self-check

- Why store timestamps in UTC, and what is the difference between naive and aware datetimes?
- What is wrong with `subprocess.run(f"convert {name}", shell=True)`?
- How do you write a file so that readers never see a partial version?
- Why is `pickle.loads` on request data dangerous?

---

## 17. Packaging and production habits

**In plain words:** Production habits are practices that keep a service safe to run and safe to change. They include automatic checks, settings from the environment, timeouts and clean shutdown. This chapter lists the habits and the tools.

### Understand

**Code quality automation.** Run the same checks locally, in a pre-commit hook and in CI.
- **ruff** (Astral) is a fast linter and formatter that replaces many older tools (flake8 and its plugins, isort, pyupgrade, and a Black-compatible formatter through `ruff format`). Configure rule families in `pyproject.toml` (for example errors, import sorting, bugbear, upgrade rules). Choose the rule set deliberately and confirm names in the Ruff documentation.
- **Black** is the established opinionated formatter, **isort** sorts imports (both are now often replaced by `ruff format` and Ruff's import rules).
- **mypy or pyright** for types (chapter 10).
- **pre-commit** runs hooks before each commit. **CI** (GitHub Actions or similar) runs lint, types, tests and a dependency audit on every pull request, and blocks merging on failure.
- **bandit** or Ruff's security rules flag risky patterns.

**Configuration (twelve-factor).** The twelve-factor app methodology says: config lives in the **environment**, not in code. Same build artifact, different environment variables per deployment (dev, staging, prod).
- Read settings once at startup into a typed settings object (`pydantic-settings`, or a dataclass populated from `os.environ`) and **fail fast** with a clear error if a required value is missing or invalid.
- Never commit secrets. Inject them from a secret manager or the platform, and keep `.env` files for local development only and out of version control.
- Do not scatter `os.environ[...]` reads across the codebase. Centralise them.
- Other relevant factors: explicit dependencies (lockfile), one codebase, stateless processes (state in a database or cache, not in process memory), port binding, disposability (fast start, graceful shutdown), dev/prod parity, logs as event streams (chapter 14), admin tasks as one-off processes, backing services as attached resources.

**Production habits.**
- **Health checks**: separate liveness (process is up) from readiness (dependencies reachable).
- **Timeouts everywhere** (HTTP, database, queue), **connection pooling**, **bounded retries with backoff and jitter**, **idempotency keys** for operations that may repeat.
- **Graceful shutdown** on `SIGTERM` (finish in-flight requests, close pools).
- **Do not run as root** in containers. Use a slim base image, install from the lockfile, and set `PYTHONUNBUFFERED=1` so logs appear immediately. `PYTHONDONTWRITEBYTECODE` is optional. Multi-stage builds keep build tools out of the final image.
- **Server choice:** a production WSGI or ASGI server (gunicorn, uvicorn, granian, hypercorn), never a framework's development server. Choose worker count by cores and workload (chapter 11) and measure.
- **Migrations** for database changes (Alembic, Django migrations) applied as a deliberate step with a rollback plan.
- **Observability:** structured logs, metrics, traces, error reporting. Alert on symptoms (error rate, latency), not only causes.
- **Security:** keep dependencies updated, validate all input at the boundary, parameterised SQL always (never string-built queries), safe deserialisation, secrets rotation, least-privilege credentials.
- **Versioning and releases:** semantic versioning for libraries, tagged builds for services, a changelog.

**Building and distributing.** A buildable project defines a build backend in `pyproject.toml` (`setuptools`, `hatchling`, `flit-core`, `pdm-backend`, `uv_build`). `python -m build` produces a **wheel** and an **sdist**. Wheels are prebuilt and install fast. Publishing to PyPI uses `twine` or trusted publishing from CI. Internal services usually ship as container images rather than wheels, but packaging internal shared libraries as wheels in a private index is common.

**Project skeleton to aim for.**

```
orders-service/
  pyproject.toml        # metadata, deps, tool config
  uv.lock               # or requirements.lock (pinned)
  src/orders/           # application code
  tests/
  .pre-commit-config.yaml
  Dockerfile
  README.md
```

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Lint and format | `ruff check`, `ruff format`, `black`, `isort` | Zero warnings in CI, auto-fix locally |
| Hooks and CI | `pre-commit`, GitHub Actions or equivalent | A pipeline that blocks bad merges |
| Settings | `pydantic-settings`, `os.environ`, `python-dotenv` (local only) | Typed, fail-fast configuration |
| Dependency hygiene | `uv`, `pip-tools`, `pip-audit`, Dependabot/Renovate | Locked, scanned, regularly updated |
| Serving | `gunicorn`, `uvicorn`, `granian`, reverse proxy | Pick workers, timeouts, graceful shutdown |
| Building | `python -m build`, `twine`, multi-stage Dockerfile | Produce a reproducible artifact |
| Migrations | `alembic`, framework migration tools | Evolve schemas safely |
| Observability | OpenTelemetry, Sentry, Prometheus client | Detect and trace failures in production |

### Common mistakes

- Configuration baked into code or committed secrets.
- Different dependency versions in dev, CI and production.
- Reading environment variables deep inside business logic.
- Running the framework's development server in production.
- Treating lint and type errors as optional.
- Stateful processes (in-memory sessions or caches that assume a single instance) behind several workers.
- No timeouts and no graceful shutdown, leading to dropped requests on every deploy.

### Self-check

- What does "config in the environment" mean, and why is a fail-fast settings object useful?
- What would a minimal CI pipeline for a Python service run?
- Why must a twelve-factor service be stateless between requests?
- How do liveness and readiness probes differ?

---

## 18. Practice labs

**In plain words:** These are exercises you do yourself. Each one trains a skill from the earlier chapters. Try each one before you look up answers.

Do these on a machine with Python 3.11 or later. Each lab states the goal. Attempt before looking anything up. Use a neutral domain such as orders, payments and events.

1. **Object model drills.** Write snippets that demonstrate aliasing, shallow versus deep copy, the mutable default trap and `is` versus `==`. Predict each output before running it, then explain any difference.
2. **Streaming pipeline.** Generate a multi-hundred-megabyte JSON Lines file of order events. Compute totals per customer using generators and `defaultdict`, and prove with `tracemalloc` or `/usr/bin/time -v` that memory stays flat. Compare with a version that loads everything.
3. **Decorator kit.** Implement `retry` (with backoff and jitter), `timed` and a TTL cache as decorators that preserve signatures with `functools.wraps`. Make each work on both sync and async functions. Test them with pytest.
4. **Typed domain model.** Model orders and payments with frozen dataclasses, a `Protocol` for a payment gateway, and a fake implementation. Make `mypy --strict` pass with no `Any`.
5. **Environment and lock.** Create a project with `pyproject.toml`, a dev dependency group, and a lockfile (with `uv` or `pip-compile`). Rebuild the environment from scratch on a clean machine or container and verify identical versions.
6. **Concurrency benchmark.** Run the same task three ways: a CPU-bound function (hashing in a loop) with threads, processes and a single thread, and an I/O-bound function (many `sleep`-based or local HTTP calls) with threads, a thread pool and asyncio. Chart the timings and explain each result with the GIL.
7. **Blocked loop.** Build a small async service, add a `time.sleep` in one handler, measure the latency of another endpoint under load, then fix it with `asyncio.to_thread`. Find the culprit using `py-spy dump` or asyncio debug mode.
8. **Fan-out with limits.** Write an async function that calls 1,000 URLs with at most 20 concurrent requests, a per-request timeout, one retry, and a global deadline. Handle cancellation so shutdown is clean.
9. **Graceful shutdown.** Make a worker process handle `SIGTERM` by finishing the current job and exiting, in both a threaded and an asyncio variant. Verify with `kill -TERM` and `kill -KILL`.
10. **Test suite.** For a small service module, write unit tests with parametrization, fixtures, a fake repository, a mocked HTTP client with `autospec`, and a frozen clock. Add an integration test against a real database in a container. Measure branch coverage and read the uncovered lines.
11. **Logging setup.** Configure JSON logging via `dictConfig` with a request ID carried through `contextvars` across asyncio tasks. Prove that log lines from concurrent requests carry the right IDs, and that no secrets are emitted.
12. **Find the leak.** Write a service with a deliberate leak (unbounded dict cache). Reproduce growth, find the allocation site with `tracemalloc` or `memray`, fix it with a bounded cache, and confirm in a graph.
13. **Profile a slow endpoint.** Take an endpoint with an N+1 query pattern and a slow serialisation step. Use `cProfile` or `py-spy record` and query logging to find both, fix them, and report before and after percentiles.
14. **Safe subprocess and files.** Write a CLI that runs an external tool with a timeout and captures output, writes results atomically with `os.replace`, and handles timezone-aware timestamps in UTC. Add `argparse` or `typer` and tests.
15. **Production-ready skeleton.** Assemble a service skeleton: `src/` layout, `ruff` and `mypy` configured, `pre-commit`, CI running lint, types and tests, a typed settings object that fails fast, a multi-stage Dockerfile running as non-root, and a readiness endpoint.

---

## 19. Interview questions

**In plain words:** This chapter has questions that interviewers often ask, with short model answers. Each question is tagged [Fresher], [Mid] or [Senior]. Learn the top 10 first, then practise with examples from your own projects.

### Top 10 must-know

1. [Fresher] `is` versus `==`? `is` compares identity (same object), `==` compares value through `__eq__`. Use `is` for `None` only. (Q1)
2. [Fresher] Why is a mutable default argument a bug? The default is created once, when the function is defined, so all calls share it. Use `None` and create the object inside. (Q3)
3. [Fresher] What is a generator? A function with `yield` that makes values one at a time, so memory stays small for large streams. (Q6)
4. [Fresher] How does Python manage memory? CPython counts references to each object and frees it at zero. A cycle collector frees objects that only refer to each other. Use `with`, not `__del__`, to release resources. (Q38)
5. [Fresher] Why is `x in some_set` faster than `x in some_list`? A set uses a hash table, so lookup is O(1) on average. A list checks items one by one, which is O(n). (Q39)
6. [Mid] What is the GIL? A lock that lets one thread run Python bytecode at a time in a process. Threads help I/O-bound work, not CPU-bound work. Processes give CPU parallelism. (Q13)
7. [Mid] Threads, processes or asyncio? I/O with async libraries and many connections: asyncio. Blocking I/O: threads. CPU work: processes or C-backed libraries. (Q14)
8. [Mid] What happens if you block inside `async def`? The single event loop stops, so every request waits. Use async libraries or `asyncio.to_thread`. (Q16)
9. [Mid] How do you find why a service is slow? Measure percentiles, check query count and external calls, then profile with `cProfile` or `py-spy`, fix the top cost, and measure again. (Q26)
10. [Senior] A service's memory keeps growing. What do you do? Confirm the OOM kill, compare `tracemalloc` snapshots, and look for unbounded caches and growing collections. (Q28)

Grouped by level. Every question has a concise model answer in the final subsection, using the same numbering. Answers are starting points. In an interview, add an example from your own work.

### Junior

1. [Fresher] What is the difference between `is` and `==`?
2. [Fresher] What is the difference between a list and a tuple? When would you use each?
3. [Fresher] Why is a mutable default argument a bug? Show the fix.
4. [Fresher] How does a dict work, and what is its lookup complexity? What can be a key?
5. [Fresher] What is the difference between shallow and deep copy?
6. [Fresher] What is a generator, and why use one?
7. [Fresher] What does a decorator do? What does `functools.wraps` add?
8. [Fresher] What does `with` guarantee, and how do you write your own context manager?
9. [Fresher] Explain `*args` and `**kwargs`.
10. [Fresher] What is the difference between `@staticmethod`, `@classmethod` and an instance method?
11. [Fresher] How do you create and use a virtual environment, and why?
12. [Fresher] What is the difference between `except Exception` and a bare `except`?
38. [Fresher] How does CPython manage memory?
39. [Fresher] Why is a membership test faster on a set than on a list?

### Mid-level

13. [Mid] What is the GIL, and how does it affect threads versus processes?
14. [Mid] How do you choose between threads, processes and asyncio?
15. [Mid] What is the difference between a coroutine and a task in asyncio?
16. [Mid] A blocking call is inside an `async def` handler. What happens, and how do you fix it?
17. [Mid] What happens when one task in `asyncio.gather` raises? How does `TaskGroup` differ?
18. [Mid] How does cancellation work in asyncio, and what should your code do with `CancelledError`?
19. [Mid] What is a dataclass, and when would you choose Pydantic or a `NamedTuple`?
20. [Mid] What is a `Protocol`, and how does it differ from an ABC?
21. [Mid] Explain pytest fixtures and scopes. How is `parametrize` used?
22. [Mid] Where must you patch when mocking, and why prefer fakes over mocks?
23. [Mid] Why use `logging` instead of `print`, and how do you add a request ID to logs?
24. [Mid] How do you store and compute timestamps correctly?
25. [Mid] What is the difference between `pyproject.toml` dependencies and a lockfile?
26. [Mid] How do you find why a Python service is slow?
27. [Mid] What are the risks of `subprocess.run(cmd, shell=True)` and of `pickle`?

### Senior-level and design

28. [Senior] A service's memory grows until it is OOM-killed. Walk through diagnosis and likely Python-specific causes.
29. [Senior] Design the Python-level strategy for a service that does both I/O-bound API work and CPU-heavy report generation.
30. [Senior] How would you introduce type checking to a large legacy codebase?
31. [Senior] Explain the memory and performance trade-offs of generators, `__slots__` and `lru_cache`.
32. [Senior] An async service shows periodic latency spikes across all endpoints with low CPU. What are your hypotheses and tools?
33. [Senior] How would you make a worker fleet shut down gracefully during a deploy?
34. [Senior] How do you design retries, timeouts and idempotency for calls to a flaky payment provider?
35. [Senior] What do the free-threaded build and sub-interpreters change, and how would you evaluate adopting them?
36. [Senior] How do you structure a Python service so that business logic is testable and not coupled to frameworks?
37. [Senior] What would your CI pipeline and production readiness checklist contain for a new Python service?

### Model answers (condensed)

**1.** `is` tests object identity, `==` tests equality through `__eq__`. Use `is` for `None` and other singletons only. Equal values can be different objects.

**2.** Lists are mutable and for homogeneous, changing sequences. Tuples are immutable, hashable when their contents are, and suit fixed records and dict keys.

**3.** The default is evaluated once at definition, so all calls share one object. Use `None` as the default and create the object inside the function (or `field(default_factory=...)` in dataclasses).

**4.** A hash table with insertion order preserved (3.7+). Get, set and delete are O(1) on average. Keys must be hashable, meaning immutable-style with a consistent `__hash__` and `__eq__`.

**5.** Shallow copy duplicates the outer container but shares the inner objects. Deep copy recursively duplicates everything, handling cycles, at a higher cost.

**6.** A function with `yield` that produces values lazily and keeps its state between yields. It processes large or unbounded streams in constant memory. It is single-use.

**7.** A decorator wraps a function to add behaviour without editing it (`f = dec(f)`). `functools.wraps` copies metadata such as `__name__` and `__doc__` so introspection and tracebacks stay correct.

**8.** `with` calls `__enter__`, runs the block, and always calls `__exit__`, even on exceptions, so resources are released. Write a class with those methods, or use `@contextlib.contextmanager` with a `yield` inside `try/finally`.

**9.** `*args` collects extra positional arguments into a tuple, `**kwargs` collects extra keyword arguments into a dict. At call sites the same symbols unpack sequences and mappings. They are used for forwarding, such as in decorators.

**10.** An instance method receives the instance, a classmethod receives the class (useful for alternative constructors), a staticmethod receives neither and is just a namespaced function.

**11.** `python -m venv .venv`, activate it (or call its interpreter directly), then `pip install`. It isolates a project's dependencies and interpreter from the system and from other projects, which prevents version conflicts and enables reproducible installs.

**12.** `except Exception` catches normal errors but lets `KeyboardInterrupt`, `SystemExit` and `CancelledError` through. A bare `except` catches everything, including those, which breaks shutdown and cancellation. Catch the narrowest type you can handle.

**38.** CPython keeps a reference count on every object and frees the object when the count reaches zero. A cyclic garbage collector (`gc`) finds groups of objects that only refer to each other. Freed memory often stays with the allocator, so process memory may not shrink. Release resources with `with`, because `__del__` timing is not guaranteed.

**39.** A set is a hash table. Python hashes the item and jumps to its slot, so lookup is O(1) on average. A list has no index by value, so Python compares item by item, which is O(n). Items must be hashable to go in a set.

**13.** The Global Interpreter Lock lets one thread run Python bytecode at a time per process. Threads therefore do not speed up CPU-bound Python, but they overlap I/O waits because the lock is released during blocking calls. Processes each have their own interpreter and GIL, so they give true CPU parallelism at the cost of memory and pickling.

**14.** High-concurrency I/O with async-capable libraries: asyncio. Blocking libraries or moderate I/O concurrency: thread pool. CPU-bound: process pool, task queue workers or C-backed libraries. Scale web services across cores with several worker processes.

**15.** A coroutine is what an `async def` call returns, and it does nothing until awaited or scheduled. A task wraps a coroutine and schedules it on the loop to run concurrently, with a result and cancellation handle.

**16.** The blocking call freezes the single-threaded event loop, so every other request stalls and latency spikes. Use an async-native library, or offload with `asyncio.to_thread` or an executor (process pool for CPU work), and add timeouts.

**17.** With `gather`, the first exception propagates to the awaiter, but the sibling awaitables are not cancelled automatically (unless `return_exceptions=True`, where errors are returned as results). `TaskGroup` (3.11+) cancels the remaining tasks when one fails, waits for all, and raises an `ExceptionGroup`, giving structured concurrency.

**18.** `task.cancel()` raises `CancelledError` at the task's next `await`. Code should clean up with `try/finally` or `async with` and re-raise `CancelledError`, never swallow it. Timeouts (`asyncio.timeout`, `wait_for`) are built on cancellation.

**19.** A dataclass generates `__init__`, `__repr__` and `__eq__` from annotated fields, with options such as `frozen` and `slots`. It does no validation. Use Pydantic to parse and validate untrusted input at boundaries, and `NamedTuple` for small immutable, unpackable records.

**20.** A Protocol defines required members, and any class that has them satisfies it without inheriting (structural typing, checked statically). An ABC requires explicit inheritance (nominal typing) and can enforce abstract methods at instantiation. Protocols suit dependency injection and third-party types.

**21.** A fixture provides setup and teardown (through `yield`) and is injected by parameter name. Scope (`function`, `class`, `module`, `session`) controls reuse. `@pytest.mark.parametrize` runs one test body over a table of inputs, each reported as its own case.

**22.** Patch the name where it is looked up (the importing module), not where it is defined. Prefer fakes and injected dependencies because mocks couple tests to implementation. Mock only at boundaries, and use `autospec` so signature mismatches fail.

**23.** `logging` gives levels, hierarchy, handlers, formatters and runtime configuration. Emit structured JSON to stdout, and store a request ID in a `contextvars.ContextVar` that a filter or processor adds to every record. `contextvars` works across asyncio tasks, unlike thread-locals.

**24.** Use timezone-aware datetimes, store and compute in UTC, convert to local zones with `zoneinfo` only for display, and exchange ISO 8601 strings. Avoid naive datetimes and `utcnow()`. Use `time.monotonic()` for durations.

**25.** `pyproject.toml` declares compatible version ranges (what the code supports). A lockfile records the exact resolved versions of all packages, including transitive ones, so builds are reproducible. Applications deploy from the lockfile.

**26.** Measure first: latency percentiles and where time goes (APM or traces). Check query count and time (N+1, missing indexes) and external calls before CPU. Then profile with `cProfile` or `py-spy` (on the live process), fix the top item, and re-measure under realistic load.

**27.** `shell=True` with interpolated input allows command injection. Pass a list of arguments, set `check` and `timeout`. `pickle.loads` on untrusted data can execute arbitrary code, so use JSON or another data-only format.

**28.** Confirm the OOM kill (kernel log, exit by signal 9, container limit). Check whether growth is steady (leak) or load-related. Take `tracemalloc` snapshots or use `memray` and compare, and use `objgraph` to see what holds references. Typical causes: unbounded caches or `lru_cache`, ever-growing module-level collections, reference cycles, loading whole datasets instead of streaming, and per-request objects stored globally. Fix the cause, set a memory limit and add alerting on RSS trend.

**29.** Keep request handling I/O-bound and light, either async or sync with thread pools. Send report generation to a separate worker pool through a queue (Celery, RQ or a cloud queue), using processes (or C-backed libraries) for CPU work, with results stored and polled or delivered by callback. Scale each tier independently, bound queue sizes, and make jobs idempotent.

**30.** Add the checker to CI in a lenient mode, annotate public interfaces and new code first, type the boundaries (models, repositories, settings), enable strictness per module with overrides, ban unexplained `# type: ignore`, add stubs for third-party libraries, and ratchet so the error count never rises. Pair it with runtime validation at inputs.

**31.** Generators trade repeatability and random access for constant memory. `__slots__` removes the per-instance dict (lower memory, faster access) but forbids dynamic attributes and complicates inheritance. `lru_cache` trades memory for speed and needs a `maxsize`, hashable arguments, and care on methods (it retains `self`). Measure each.

**32.** Suspect blocking calls on the loop (synchronous drivers, `time.sleep`, large JSON parsing, CPU work), garbage collection pauses, a saturated thread pool used by `to_thread`, exhausted connection pools, and slow DNS. Use `py-spy dump` to see what the loop thread is doing, asyncio debug mode for slow callbacks, loop-lag metrics, and pool and queue metrics.

**33.** Handle `SIGTERM`: stop taking new work, finish or hand back in-flight jobs (acknowledge or requeue messages), close pools, then exit within the orchestrator's grace period. Make jobs idempotent so a hard kill is survivable, ensure the Python process is PID 1 or an init forwards signals, and test it with `kill -TERM`.

**34.** Set connect and read timeouts, retry only transient errors with exponential backoff and jitter and a retry budget, and respect rate limits. Attach an idempotency key so repeated requests do not double-charge, record request state durably, add a circuit breaker for sustained failure, and emit metrics and alerts on error and retry rates.

**35.** The free-threaded build (PEP 703, experimental in 3.13, officially supported but optional from 3.14 under PEP 779) removes the GIL so threads can run Python code in parallel. It needs compatible C extensions and adds some single-thread overhead. Sub-interpreters have had their own GIL since 3.12 (PEP 684, C API only). Python 3.14 adds the `concurrent.interpreters` module and `InterpreterPoolExecutor` (PEP 734). They offer isolation with parallelism, but object sharing is limited and many extension modules are not yet compatible. Evaluate with benchmarks on your workload, check dependency compatibility, and read the current release notes before production use.

**36.** Put domain logic in plain functions and classes that depend on small interfaces (Protocols), inject repositories, clients and the clock, keep framework code thin (routing, parsing, serialising), validate at the boundary, and test the core with fakes and a few integration tests against real infrastructure.

**37.** CI: `ruff check`, `ruff format --check`, type check, tests with coverage and parallelism, a dependency audit, a build of the container image from the lockfile. Readiness: typed fail-fast settings from the environment, health and readiness endpoints, structured logs with correlation IDs, metrics and tracing, timeouts, pooled and bounded connections, graceful shutdown, a non-root container, migration and rollback plan, and an on-call runbook.

---

## 20. References

**In plain words:** These are the official sources for the facts in this guide. Use them to check details that change between Python versions.

Primary sources, preferred over blog posts:

- Python documentation (language reference, tutorial, standard library): https://docs.python.org/3/
- Python Data Model: https://docs.python.org/3/reference/datamodel.html
- `asyncio` documentation: https://docs.python.org/3/library/asyncio.html
- `concurrent.futures`, `threading`, `multiprocessing`: https://docs.python.org/3/library/concurrency.html
- `logging` HOWTO and Cookbook: https://docs.python.org/3/howto/logging.html
- What's New in Python (per version release notes): https://docs.python.org/3/whatsnew/
- Python Enhancement Proposals: https://peps.python.org/ (notably PEP 8 style, PEP 484 type hints, PEP 544 Protocols, PEP 518 and PEP 621 project metadata, PEP 668 externally managed environments, PEP 684 per-interpreter GIL, PEP 703 and PEP 779 free-threaded CPython, PEP 734 multiple interpreters)
- Python Packaging User Guide: https://packaging.python.org/
- typing documentation and the typing specification: https://typing.python.org/
- mypy documentation: https://mypy.readthedocs.io/
- pyright documentation: https://microsoft.github.io/pyright/
- pytest documentation: https://docs.pytest.org/
- Ruff documentation: https://docs.astral.sh/ruff/
- uv documentation: https://docs.astral.sh/uv/
- py-spy repository and documentation: https://github.com/benfred/py-spy
- The Twelve-Factor App: https://12factor.net/
- Luciano Ramalho, *Fluent Python* (book)
- Brett Slatkin, *Effective Python* (book)

Verify any URL before depending on it, since pages move. On 2026-11-06 the docs.python.org, typing.python.org, mypy, pyright, pytest, Ruff, uv and py-spy URLs were fetched and loaded. Not fetched (unverified): peps.python.org, packaging.python.org, 12factor.net.

---

## 21. Glossary

**In plain words:** This table explains the hardest terms in plain English. Each term is used in the chapters above.

| Term | Plain meaning |
|---|---|
| Hashable | Has a fixed hash value, so it can be a dict key or set member |
| Reference counting | Python counts the names that point to an object and frees it at zero |
| Generator | A function that gives values one at a time with `yield` |
| Iterator | An object that returns the next item each time you ask |
| Decorator | A function that wraps another function to add behaviour |
| Closure | An inner function that remembers variables from its outer function |
| MRO | Method resolution order: the order of classes Python searches for a method |
| Protocol | A description of required methods; any matching class fits |
| Dataclass | A class whose boilerplate methods are generated from its fields |
| Context manager | An object used with `with` that sets up and always cleans up |
| EAFP | Try the action first and handle the error if it fails |
| Idempotent | Safe to repeat: running it again gives the same result |
| Lockfile | A file that records the exact version of every dependency |
| Virtual environment | A private folder of installed packages for one project |
| GIL | Global Interpreter Lock: one thread runs Python code at a time |
| Free-threaded build | A CPython build without the GIL |
| Coroutine | The object made by calling an `async def` function |
| Event loop | The scheduler that runs asyncio tasks one at a time |
| Task | A coroutine that the event loop has been asked to run |
| Cancellation | A request to stop a task by raising `CancelledError` inside it |
| Back-pressure | Slowing a fast producer when the consumer is full |
| Pickle | Python's format for turning objects into bytes |
| Fixture | A pytest function that prepares what a test needs |
| Mock | A fake object that records how it was called |
| Profiler | A tool that measures where time or memory goes |
| N+1 queries | One query for a list, then one more query for each item |
| RSS | Resident set size: memory a process currently holds in RAM |
| OOM kill | The operating system stops a process that uses too much memory |
| Twelve-factor | A set of rules for building services that run well in the cloud |
| Wheel | A prebuilt Python package file that installs quickly |
