# Java for Backend Engineers

A working guide to Java and the JVM for people who build and operate backend services. It teaches how the platform behaves, so you can reason about a slow, leaking or deadlocked service in production, not just write code that compiles.

**Audience:** developers who know JavaScript/Node (or another language) and want production-level Java skill for backend roles.
**Prerequisites:** basic programming, HTTP and SQL. The Linux chapter of this repository helps for the operations material.
**Scope:** server-side Java on the JVM, the core language and libraries, concurrency, memory, Spring Boot, persistence, testing and diagnostics. Android and desktop UI are out of scope.
**Accuracy status:** Reviewed against official documentation on 2026-11-06; items still unverified are marked.
**Note on accuracy:** the concepts here are long-stable. Language features and JVM options change between releases. Where a feature depends on the Java version, this guide names the release when it was verified. Unverified items are marked "confirm for your JDK". JVM flags and tool output formats also vary by version and vendor, so confirm with the tool's `--help` or the documentation for your JDK.

## How to use this guide

Each chapter has four parts:

1. **Understand**: the concepts and the reasoning behind them.
2. **Tools to master**: what to learn to use fluently, and the level to reach.
3. **Common mistakes**: what goes wrong in practice.
4. **Self-check**: questions you should be able to answer without notes.

Each chapter starts with an "In plain words" summary in simple English. Words marked in brackets are explained where they first appear, and chapter 21 is a glossary of the hardest terms. Chapter 18 lists hands-on labs. Chapter 19 collects interview questions, each with a level tag and a concise model answer. Chapter 20 lists primary references.

**What "good enough" means for a backend role:** you can explain what the JVM does with your code, choose the right collection and concurrency tool, read a thread dump and a GC log, build a Spring Boot service with correct transactions and a sized connection pool, and say when Java is the better or worse choice than Node. Chapters 1, 5, 9, 10, 11, 13, 14 and 16 matter most.

## Contents

1. The JVM mental model
2. Types, boxing and strings
3. Object-oriented design, interfaces and the equals/hashCode contract
4. Generics and type erasure
5. The collections framework
6. Streams and Optional
7. Exceptions and resource management
8. Records, sealed types and pattern matching
9. Concurrency foundations and the Java Memory Model
10. Concurrency toolkit: executors, CompletableFuture, locks, atomics, virtual threads
11. Garbage collection and memory
12. Build tools: Maven and Gradle
13. Spring Boot essentials
14. JDBC, JPA and connection pools
15. Testing with JUnit and Mockito
16. Profiling and troubleshooting
17. Java versus Node for backend services
18. Practice labs
19. Interview questions
20. References
21. Glossary

---

## 1. The JVM mental model

**In plain words:** Java code is first turned into bytecode (a compact, machine-independent instruction format). The JVM (Java Virtual Machine, the program that runs bytecode) then runs it. Knowing how the JVM uses memory and speeds up code helps you explain slow starts and memory limits.

### Understand

**Compile once, run on a virtual machine.** `javac` compiles `.java` source into **bytecode** stored in `.class` files. The **Java Virtual Machine (JVM)** loads and executes that bytecode. Bytecode is portable across operating systems, and the JVM is the platform-specific part. Contrast with Node: V8 receives JavaScript source and compiles it at runtime. In Java the compile step is separate and produces a typed, verifiable artifact (usually packaged into a `.jar`).

**JDK, JRE, JVM.** The JVM runs bytecode. The JRE is the JVM plus the standard library. The JDK adds the compiler and tools (`javac`, `jcmd`, `jstack`, `jfr`). Backend work needs the JDK. "OpenJDK" is the open source reference implementation, and several vendors ship builds of it. Production services normally run on a long-term-support (LTS) release, so check which LTS your platform supports.

**Class loading.** The JVM loads classes lazily, the first time they are needed. Three phases: **loading** (find the bytes, create a `Class` object), **linking** (verify the bytecode, prepare static fields, resolve references) and **initialisation** (run static initialisers). Class loaders form a hierarchy and delegate to their parent first, so core classes cannot be replaced by application classes. Application servers and frameworks use extra class loaders for isolation, which is the root of `ClassNotFoundException`, `NoClassDefFoundError` and "same class, different loader" bugs.

**Execution: interpreter and JIT.** The JVM starts by interpreting bytecode. It profiles which methods are hot and compiles them to native machine code with the **just-in-time (JIT) compiler**. HotSpot, the common JVM, has tiered compilation: a fast, lightly optimised compiler first, then a heavily optimising one for the hottest code. The optimiser uses runtime profile data to inline methods, remove dead branches and eliminate allocations (escape analysis). Consequences for backend work:

- **Warm-up.** A fresh JVM is slower than a warmed one. The first requests after a deploy can be slow, and benchmarks must include warm-up. Use JMH for microbenchmarks, because naive timing loops mislead.
- **Peak throughput is excellent.** Long-running services settle into highly optimised native code, which is why Java suits steady, high-throughput servers.
- **Deoptimisation.** If an assumption made by the JIT fails, the JVM falls back to less optimised code and recompiles. Rare, but explains some latency blips.
- Ahead-of-time options exist (GraalVM native image, class data sharing) that trade peak performance for faster startup. They are an operational choice, not a language change.

**Memory areas.** Know this map.

| Area | Holds | Notes |
|---|---|---|
| Heap | all objects and arrays | shared by all threads, managed by the garbage collector, sized with `-Xms` and `-Xmx` |
| Thread stacks | one stack per thread: frames with local variables, parameters and return addresses | primitives and references live here, the objects they point to live on the heap; size set with `-Xss` or per thread; deep recursion causes `StackOverflowError` |
| Metaspace | class metadata, method data | native memory (not heap), replaced the older permanent generation; grows by default, can be capped with `-XX:MaxMetaspaceSize` |
| Code cache | JIT-compiled native code | fixed size, can fill up in rare cases |
| Direct and native memory | NIO direct buffers, thread stacks, GC data structures, JNI | not counted in `-Xmx`; a container's memory limit covers it |

The key operational lesson: the process's resident memory is larger than the heap. Setting `-Xmx` equal to a container's memory limit invites an OOM kill by the operating system, because metaspace, thread stacks and native memory sit on top. Leave headroom, or size the heap as a percentage of container memory (`-XX:MaxRAMPercentage`; confirm for your JDK version that the JVM is container-aware, which modern JDKs are).

**References, not pointers.** Java has no pointer arithmetic. A variable of an object type holds a **reference**. Java is **pass by value**: for objects, the value copied is the reference. A method can mutate the object a reference points to, but cannot reassign the caller's variable. Contrast with JavaScript, where the same rule applies to objects, but Java also has distinct primitive types.

**Everything is a class, and types are checked at compile time.** Java is statically typed. Type errors appear at compile time, and the IDE can navigate and refactor with confidence. This is a major reason large Java codebases stay maintainable.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Compile and run | `javac`, `java`, `jar`, `java -jar app.jar` | Build and run a multi-class program from the terminal |
| Inspect bytecode | `javap -c -p ClassName` | Read bytecode for a simple method and see what the compiler generated |
| JVM flags | `java -XX:+PrintFlagsFinal -version`, `java -XshowSettings:vm -version` | Find effective heap and GC settings |
| Class loading | `-verbose:class`, `-Xlog:class+load` (unified logging) | Diagnose which jar a class came from |
| JIT | `-XX:+PrintCompilation`, JMH | Explain warm-up and write a correct microbenchmark |
| Runtime facts | `jcmd <pid> VM.flags`, `jcmd <pid> VM.version`, `jcmd <pid> VM.system_properties` | Read live configuration of a running JVM |
| IDE | IntelliJ IDEA, Eclipse, VS Code with Java extensions | Navigate, refactor and debug fluently |

### Common mistakes

- Setting `-Xmx` to the full container memory and getting OOM-killed by the kernel.
- Benchmarking with a single cold run, or with a loop the JIT eliminates.
- Confusing "pass by reference" with "references passed by value".
- Having two versions of a library on the classpath and not knowing which one loads.

### Self-check

- Trace what happens from `java -jar app.jar` until `main` runs.
- Why is a JVM slow for the first seconds and then fast?
- Name four memory regions of a JVM process and which are inside `-Xmx`.
- A container is killed with exit code 137 while the heap graph looked healthy. Why?

---

## 2. Types, boxing and strings

**In plain words:** Java has simple types such as `int` and full object types such as `Integer`. Java converts between them for you (boxing), and this can cause surprises. Strings never change after they are created, which makes them safe to share.

### Understand

**Primitives and objects.** Java has eight primitives: `byte`, `short`, `int`, `long`, `float`, `double`, `boolean`, `char`. They hold raw values, cannot be `null` and are stored inline (in a local frame or inside an object). Everything else is an object reference. Primitives have fixed sizes (`int` is 32 bits, `long` 64 bits) on every platform. Contrast with JavaScript, which has a single `number` type (a double), so Java integer overflow is something you must reason about: `Integer.MAX_VALUE + 1` wraps silently to a negative value. Use `Math.addExact` and friends to fail loudly.

**Money and floating point.** `double` cannot represent decimal fractions like 0.1 exactly. For money use `BigDecimal` (construct from a `String` or use `BigDecimal.valueOf`, not from a `double` literal) or store integer minor units (cents) in a `long`.

**Boxing.** Each primitive has a wrapper class (`Integer`, `Long`, `Boolean`...). Generics and collections work only with objects, so the compiler converts between primitive and wrapper automatically (**autoboxing** and **unboxing**). Traps:

- **Allocation cost.** `List<Integer>` stores references to `Integer` objects, not raw ints. Large numeric collections use far more memory and cache-unfriendly layout than `int[]`. Hot paths and large data sets may justify primitive arrays or primitive-specialised streams (`IntStream`).
- **NullPointerException on unboxing.** Assigning a `null` `Integer` to an `int` throws. A database column mapped to `Integer` may be null.
- **`==` on wrappers compares references.** Small values are cached by the specification, so `Integer a = 100, b = 100; a == b` may be true while the same test with 1000 is false. Always use `equals` for wrapper and object comparison.
- **Unintended boxing in loops** (`Long sum = 0L; sum += x;`) creates garbage on every iteration.

**Strings are immutable.** A `String` never changes after creation. Operations like `concat` or `replace` return new strings. Why this matters:

- Safe to share between threads and to use as `HashMap` keys (the hash code can be cached).
- The JVM can **intern** string literals (store one copy in a pool), so `"a" == "a"` may be true but `new String("a") == "a"` is false. Compare strings with `equals`, never `==`.
- Concatenating in a loop (`s += x`) creates a new string per iteration, quadratic in total. Use `StringBuilder` (single thread). The compiler optimises simple one-line concatenations, not loops.
- Strings are UTF-16 internally, and the JVM may use a compact representation for Latin-1 text. `length()` counts UTF-16 code units, not user-visible characters, so emoji and some scripts need code point methods. Always specify a charset explicitly (`StandardCharsets.UTF_8`) when converting between bytes and strings. Do not rely on the platform default.
- **Secrets.** Because strings are immutable and may sit in memory until garbage collected, sensitive values are sometimes held in `char[]` and cleared. In practice, the more relevant rule is never logging them.

**`var`, text blocks.** Modern Java supports local variable type inference with `var` (the type is still static, inferred by the compiler) and multi-line text blocks with `"""`. `var` arrived in Java 10 and text blocks in Java 15 (from memory, not re-checked).

**Immutability as a design tool.** Immutable objects (`final` fields, no setters, defensive copies of mutable inputs) need no synchronisation and cannot be corrupted by another thread. Prefer them for value types, DTOs and anything shared.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Numeric safety | `Math.addExact`, `Math.multiplyExact`, `BigDecimal`, `BigInteger` | Handle overflow and money correctly |
| Strings | `String`, `StringBuilder`, `String.format`, `String.join`, `String.strip`, `Objects.equals` | Build strings efficiently and compare safely |
| Encoding | `StandardCharsets`, `getBytes(charset)`, `new String(bytes, charset)` | Never depend on the default charset |
| Time | `java.time`: `Instant`, `LocalDate`, `ZonedDateTime`, `Duration`, `ZoneId` | Use the modern API, store `Instant` or UTC, avoid `java.util.Date` and `Calendar` |
| Inspecting boxing | `javap -c`, IDE inspections | Spot hidden boxing in hot loops |

### Common mistakes

- Comparing strings or boxed numbers with `==`.
- `new BigDecimal(0.1)`: the exact binary value of the double, not 0.1.
- Using the default charset, so behaviour differs between a laptop and a server.
- Using `LocalDateTime` for instants. It has no time zone, so it cannot identify a moment.
- `s += x` in a loop.

### Self-check

- Why does `Integer a = 1000, b = 1000; a == b` behave differently from the same code with 100?
- Why are strings safe as `HashMap` keys?
- What does `Integer.MAX_VALUE + 1` produce, and how would you detect it?
- Which type do you store for "the moment an order was placed"?

---

## 3. Object-oriented design, interfaces and the equals/hashCode contract

**In plain words:** Good Java code depends on interfaces (lists of promised methods), not on one fixed class. Two objects that are equal must also give the same hash code (a number used to place an object in a hash table). If you break this rule, maps and sets stop working correctly.

### Understand

**Classes and objects.** A class defines fields and methods. Objects are created with `new` and live on the heap. Access modifiers: `private`, package-private (default), `protected`, `public`. Prefer the narrowest. Java has **single inheritance of classes** and **multiple implementation of interfaces**.

**Interfaces are the main abstraction tool.** An interface defines a contract. Code depends on the interface and a concrete implementation is supplied from outside. This is the basis of dependency injection (chapter 13), testing with fakes and mocks (chapter 15), and swapping implementations. Modern interfaces can have `default` and `static` methods, which allow evolving an interface without breaking implementers. Functional interfaces (one abstract method, like `Runnable`, `Comparator`, `Function`) can be implemented with lambdas.

**Abstract class versus interface.** Use an interface to describe capability and the contract. Use an abstract class only when implementations share state or a skeleton algorithm. Interfaces have no instance state.

**Composition over inheritance.** Inheritance couples subclass to the superclass's implementation details and breaks easily when the parent changes. Prefer holding a collaborator and delegating. Mark classes `final` unless designed for extension.

**Polymorphism.** Method calls on a reference are dispatched at runtime to the object's actual class (virtual dispatch). Overloading (same name, different parameter types) is resolved at compile time by the declared types, which surprises people. Use `@Override` always so the compiler checks you really override.

**Encapsulation and immutability.** Keep fields private, expose behaviour, copy mutable inputs and outputs (defensive copies), and return unmodifiable views or immutable copies (`List.copyOf`, `Collections.unmodifiableList`). Note that `unmodifiableList` is a read-only view of the original, while `List.copyOf` is an independent immutable copy.

**The equals/hashCode contract.** This underpins every hash-based collection.

- `equals` must be **reflexive** (`x.equals(x)`), **symmetric**, **transitive**, **consistent** (same result while the objects are unchanged) and `x.equals(null)` is false.
- If two objects are `equals`, they **must** return the same `hashCode`. Unequal objects may share a hash code, but collisions cost performance.
- Override both or neither. Overriding only `equals` breaks `HashMap` and `HashSet`: the object is placed in a bucket by hash code and then not found, because the default `hashCode` is identity-based.
- Do not use mutable fields in `hashCode` for objects used as map keys. If the field changes after insertion, the entry is in the wrong bucket and becomes unreachable (a silent leak).
- Using a subclass with `instanceof` in `equals` can break symmetry. Records and IDE or library generated implementations get this right more often than hand-written code.
- Entities with database-generated IDs need care: an ID that is `null` before persistence changes the hash code after save. Use a stable business key or a documented strategy.

A correct, idiomatic pair:

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Money other)) return false;
    return amountMinor == other.amountMinor && currency.equals(other.currency);
}

@Override
public int hashCode() {
    return Objects.hash(amountMinor, currency);
}
```

`toString` should be informative and must never include secrets. `Comparable` (natural order) and `Comparator` (external order) should be consistent with `equals` when objects are used in sorted sets and maps, because `TreeMap` uses comparison, not `equals`, to decide equality.

**Packages and modules.** Packages namespace classes. The Java module system exists, but most backend services use the classpath and packages, with build tools managing dependencies. Know that it exists and read about it only if you ship libraries.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Contracts | `Objects.equals`, `Objects.hash`, `Comparable`, `Comparator.comparing().thenComparing()` | Write a correct equals, hashCode and comparator |
| Immutability | `final`, `List.of`, `Map.of`, `List.copyOf`, records (chapter 8) | Design immutable value types |
| Design | interfaces, abstract classes, `@Override`, `@FunctionalInterface` | Choose interface versus abstract class and justify it |
| Static analysis | IDE inspections, SpotBugs, Error Prone, Checkstyle | Run them in the build and fix the findings |
| Generated boilerplate | IDE generators, Lombok (a third-party library; know its trade-offs) | Understand what generated code does before relying on it |

### Common mistakes

- Overriding `equals` without `hashCode`.
- Mutable keys in a `HashMap`.
- Deep inheritance hierarchies for code reuse.
- Exposing internal mutable collections through getters.
- Overload ambiguity, for example passing `null` or boxed types to overloaded methods.

### Self-check

- State the equals/hashCode contract and what breaks when it is violated.
- What is the difference between `Collections.unmodifiableList(x)` and `List.copyOf(x)`?
- Why can a `TreeSet` treat two objects as equal even though `equals` says otherwise?
- When would you choose an interface, and when an abstract class?

---

## 4. Generics and type erasure

**In plain words:** Generics let a list or method work with many types while the compiler still checks them. After compiling, Java removes the type arguments (type erasure), so the running program does not know them. This explains several limits of generic code.

### Understand

**Why generics.** Generics give compile-time type safety to containers and algorithms: `List<String>` cannot accidentally receive an `Integer`, and reading from it needs no cast. They move errors from runtime to compile time.

**Type erasure.** The compiler checks generics, then **erases** type parameters in the bytecode. A `List<String>` and a `List<Integer>` are the same class at runtime. Consequences:

- You cannot do `new T()`, `new T[n]`, `instanceof List<String>` or overload methods that differ only by type argument.
- Primitives cannot be type arguments (`List<int>` is illegal), so boxing is required (chapter 2).
- Runtime type information is lost. Libraries that need it (JSON deserialisation, for example) use a **type token** such as Jackson's `TypeReference` or Spring's `ParameterizedTypeReference`.
- **Heap pollution** and unchecked warnings arise from raw types and unsafe casts. Treat `unchecked` warnings as real. Never use raw types (`List` without arguments) in new code.

**Generic methods and bounds.**

```java
static <T extends Comparable<T>> T max(List<T> items) { /* ... */ }
```

`T extends X` is an upper bound. A type parameter can have multiple bounds.

**Wildcards and PECS.** Generics are **invariant**: `List<Integer>` is not a `List<Number>`, because otherwise you could insert a `Double` through the `Number` view. Wildcards express flexibility:

- `List<? extends Number>`: a producer. You can read `Number`s out, but cannot add (except `null`).
- `List<? super Integer>`: a consumer. You can add `Integer`s, but reads give only `Object`.
- Mnemonic **PECS**: Producer Extends, Consumer Super. `Collections.copy(dest, src)` has `dest` as `? super T` and `src` as `? extends T`.

Contrast with TypeScript, whose structural, erased-at-compile-time types feel similar, but Java's nominal typing and invariance rules are stricter.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Writing generic code | generic classes and methods, bounded types, wildcards | Write a type-safe utility method with a bound |
| Runtime type info | `TypeReference` (Jackson), `ParameterizedTypeReference` (Spring), `Class<T>` tokens | Deserialise `List<Dto>` correctly |
| Diagnostics | `javac -Xlint:unchecked`, IDE warnings | Treat unchecked warnings as defects |

### Common mistakes

- Using raw types to silence compiler errors.
- Expecting `List<Dog>` to be assignable to `List<Animal>`.
- Deserialising JSON into `List<Dto>` without a type token and getting a list of maps, then a `ClassCastException` far from the cause.
- Overusing wildcards in return types, which burdens callers.

### Self-check

- What is type erasure and what three things can you not do because of it?
- Explain PECS with an example.
- Why is `List<Integer>` not a `List<Number>`?

---

## 5. The collections framework

**In plain words:** Collections are the ready-made containers in Java, such as lists, sets and maps. Each one is fast at some jobs and slow at others. Choose by what you do most often, and use the concurrent versions when many threads share a container.

### Understand

The framework has interfaces (`Collection`, `List`, `Set`, `Queue`/`Deque`, `Map`) and implementations. Choose by the operations you need and their complexity. Program to the interface (`List<Order> orders = new ArrayList<>();`).

**ArrayList.** A resizable array. Index access is O(1). Appending is amortised O(1): when the backing array is full, it allocates a larger one (about 1.5 times larger) and copies. Insert or remove in the middle is O(n) because of shifting. Cache-friendly. This is the default `List`. If you know the size, pass an initial capacity to avoid repeated resizing.

**LinkedList.** A doubly linked list. Indexed access is O(n), and each element costs an extra node object. Insertion and removal are O(1) only if you already hold the position (an iterator). In practice `ArrayList` beats it in almost every real workload because of memory locality, so `LinkedList` is rarely the right choice. For queues and stacks, prefer `ArrayDeque`.

**HashMap internals.** Understand this well, it is the classic interview topic and underlies production performance.

- Backed by an array of **buckets**. The key's `hashCode()` is spread (higher bits mixed into lower bits) and the bucket index is derived from it and the array length (a power of two).
- Collisions: entries in the same bucket form a linked list. In modern implementations, when a bucket gets long (a threshold of 8) and the table is large enough, the bucket is converted to a balanced tree, bounding the worst case at O(log n). It reverts to a list when it shrinks.
- Lookup: compute hash, find the bucket, then compare keys with `equals`. That is why the equals/hashCode contract matters (chapter 3).
- **Resizing.** Default initial capacity 16 and load factor 0.75. When the number of entries exceeds capacity times load factor, the table doubles and entries are redistributed. Resizing is O(n) and causes a latency spike on a large map. If you know the expected size, size the map up front (capacity of roughly `expected / 0.75` plus one, or use `HashMap.newHashMap(n)`, available since Java 19 per the API docs).
- Allows one `null` key and `null` values. **Not thread-safe.** Concurrent modification can corrupt it, and historically could cause infinite loops during resize.
- Iteration order is unspecified. Do not depend on it.
- `LinkedHashMap` keeps insertion order (or access order, which makes an LRU cache with `removeEldestEntry`).

**TreeMap and TreeSet.** Red-black tree. Keys are kept sorted by `Comparable` or a `Comparator`. Operations are O(log n). Use when you need ordered iteration, range queries (`subMap`, `headMap`, `tailMap`) or `floorKey`/`ceilingKey`. Equality is decided by the comparator, not `equals`.

**HashSet / TreeSet / LinkedHashSet.** Sets backed by the corresponding maps.

**PriorityQueue.** Binary heap. `peek` O(1), `offer`/`poll` O(log n). Iteration order is not sorted.

**ArrayDeque.** Resizable circular array, the preferred stack and queue. Faster than `LinkedList` and `Stack`. Do not use `Stack` or `Vector` (legacy, synchronised).

**Immutable collections.** `List.of`, `Set.of`, `Map.of` and `copyOf` variants create unmodifiable collections that reject `null`. `Set.of` and `Map.of` have unspecified iteration order and reject duplicates at creation.

**Fail-fast iterators.** Modifying a standard collection while iterating over it (other than through the iterator) throws `ConcurrentModificationException` on a best-effort basis. Use `Iterator.remove`, `removeIf` or collect then remove.

**Concurrent collections.** Pick these over wrapping with `Collections.synchronizedX`, which locks the whole collection and still needs external locking for compound actions and iteration.

| Class | Use |
|---|---|
| `ConcurrentHashMap` | General concurrent map. Reads mostly lock-free, updates lock at fine granularity (modern implementation uses compare-and-set and per-bin locking). No `null` keys or values. Provides atomic compound methods: `putIfAbsent`, `computeIfAbsent`, `compute`, `merge`. Size is an estimate under concurrent change. |
| `CopyOnWriteArrayList` | Read-heavy, write-rare lists (listeners, configuration). Every write copies the array. Iterators see a snapshot. |
| `ConcurrentLinkedQueue`, `ConcurrentLinkedDeque` | Non-blocking unbounded queues. |
| `BlockingQueue` implementations (`ArrayBlockingQueue`, `LinkedBlockingQueue`) | Producer-consumer hand-off with back-pressure through bounded capacity. `put` blocks when full, `take` blocks when empty. |
| `ConcurrentSkipListMap` | Concurrent sorted map. |

Compound check-then-act on a concurrent map is a race unless you use an atomic method:

```java
// Racy: two threads can both see null and both create the value.
if (cache.get(key) == null) cache.put(key, load(key));

// Atomic per key:
cache.computeIfAbsent(key, k -> load(k));
```

Keep the function passed to `computeIfAbsent` short and free of calls that update the same map. Long or blocking work inside it holds a lock on that bin.

**Complexity cheat sheet (typical).** ArrayList: get O(1), add-at-end amortised O(1), insert-middle O(n), contains O(n). HashMap/HashSet: get/put/contains O(1) average. TreeMap: O(log n). ArrayDeque: O(1) at both ends. PriorityQueue: O(log n) insert and remove, O(1) peek.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Choosing structures | `ArrayList`, `ArrayDeque`, `HashMap`, `LinkedHashMap`, `TreeMap`, `PriorityQueue`, `EnumMap`, `EnumSet` | Pick by operation and complexity, justify it |
| Utilities | `Collections`, `Arrays`, `List.of`, `Map.of`, `Map.entry` | Sort, search, copy, create immutable collections |
| Concurrent | `ConcurrentHashMap`, `CopyOnWriteArrayList`, `BlockingQueue`, `ConcurrentSkipListMap` | Use atomic map methods and bounded queues |
| Large or specialised data | Guava collections, Eclipse Collections, fastutil (third-party primitive collections) | Know they exist for memory-heavy numeric data |
| Caching | Caffeine (third-party) | Prefer it to a hand-built `LinkedHashMap` cache for production |
| Memory cost | JOL (Java Object Layout), heap histogram via `jcmd GC.class_histogram` | Estimate per-entry overhead of a map |

### Common mistakes

- Using a plain `HashMap` from several threads.
- Check-then-act (`containsKey` then `put`) on a shared map.
- Mutating a key after inserting it.
- Using `LinkedList` for performance, or `Vector`/`Stack` by habit.
- Unbounded queues and caches, which turn load spikes into out-of-memory errors.
- Modifying a list while looping over it.
- Using `Set.of`/`Map.of` and being surprised by a `NullPointerException` or duplicate error.

### Self-check

- Describe what happens inside `HashMap.put` from hashing to a possible resize.
- Why is the default `HashMap` capacity a power of two, and what is the load factor for?
- When does a `TreeMap` beat a `HashMap`?
- Why is `computeIfAbsent` safer than `containsKey` plus `put` on a `ConcurrentHashMap`?
- Why is `CopyOnWriteArrayList` a poor choice for a frequently updated list?

---

## 6. Streams and Optional

**In plain words:** A stream describes a chain of steps to process data, and it runs only when you ask for a final result. Optional is a box that may hold a value or be empty. It is a clear way to say that a result may be missing.

### Understand

**Streams** express data processing as a pipeline: a **source**, zero or more lazy **intermediate** operations (`filter`, `map`, `flatMap`, `sorted`, `distinct`, `limit`) and one **terminal** operation (`collect`, `forEach`, `reduce`, `count`, `findFirst`, `anyMatch`) that triggers execution. Nothing runs until the terminal operation. A stream is single-use and does not store data or modify its source.

```java
Map<String, Long> ordersPerCustomer = orders.stream()
    .filter(o -> o.status() == Status.PAID)
    .collect(Collectors.groupingBy(Order::customerId, Collectors.counting()));
```

Contrast with JavaScript arrays: `Array.map` and `filter` are eager and produce a new array at each step, while Java streams are lazy and fuse the stages into a single pass, and can short-circuit (`findFirst`, `limit`).

**Rules for good stream code.**
- Keep lambdas pure: no shared mutable state, no side effects except in `forEach` at the end. Mutating an external list inside `map` is a bug waiting for parallelism.
- Use `Collectors.toList()` or the `toList()` terminal method (since Java 16, returns an unmodifiable list) to materialise. Know `groupingBy`, `partitioningBy`, `toMap` (which throws on duplicate keys unless you give a merge function), `joining` and `summarizingInt`.
- Primitive streams (`IntStream`, `LongStream`, `DoubleStream`) avoid boxing.
- A plain loop is sometimes clearer. Streams are a tool for transformations, not a mandate.

**Parallel streams.** `.parallel()` splits work across the shared `ForkJoinPool.commonPool()`. It helps only for CPU-bound work on large data with cheap-to-split sources and no shared state. It is dangerous in a server: blocking I/O inside a parallel stream stalls the common pool that other code also uses, and results are hard to reason about. Default to sequential streams in request-handling code and measure before parallelising.

**Optional** represents "a value that may be absent" as a return type, so callers cannot forget the absent case. Use `map`, `flatMap`, `filter`, `orElse`, `orElseGet`, `orElseThrow`, `ifPresent`. Guidelines:

- Use it as a **return type** for finder methods (`Optional<User> findByEmail(...)`).
- Do **not** use it for fields, method parameters or collections (return an empty collection instead of `Optional<List<T>>`).
- `orElse(expr)` evaluates `expr` always. Use `orElseGet(() -> ...)` when the fallback is expensive.
- Never call `get()` without checking. Prefer `orElseThrow`.
- `Optional` is not serialisable by default in many frameworks. Use it at the API of your code, not in your DTOs.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Pipelines | `Stream`, `IntStream`, `Collectors`, `Stream.iterate`, `Stream.generate` | Write group, partition and aggregate pipelines |
| Absent values | `Optional`, `Objects.requireNonNull`, `@Nullable` annotations | Model absence without null leaks |
| Functional interfaces | `Function`, `Predicate`, `Supplier`, `Consumer`, `BiFunction`, `UnaryOperator` | Pass behaviour as arguments |
| Performance | JMH, JFR (chapter 16) | Measure before and after a rewrite |

### Common mistakes

- Reusing a stream (`IllegalStateException`).
- `Collectors.toMap` with duplicate keys.
- Side effects in intermediate operations.
- `parallel()` with blocking calls or in a request path.
- `Optional.get()` without a check, and `Optional` as a parameter.
- Long, unreadable pipelines where a loop is clearer.

### Self-check

- Why does a stream do nothing until a terminal operation?
- When is `parallelStream()` a bad idea in a web service?
- Difference between `orElse` and `orElseGet`?
- Group a list of orders by customer and sum the amounts.

---

## 7. Exceptions and resource management

**In plain words:** An exception is an object that reports an error and stops the normal flow of the program. Handle errors in a clear place, and keep the original cause. Always close files and connections, and try-with-resources does this for you.

### Understand

**Hierarchy.** `Throwable` has two branches: `Error` (serious JVM problems such as `OutOfMemoryError` and `StackOverflowError`, generally not caught) and `Exception`. Under `Exception`, `RuntimeException` and its subclasses are **unchecked**. All other exceptions are **checked**: the compiler requires you to catch them or declare them with `throws`.

**Checked versus unchecked.** The intent: checked exceptions for recoverable conditions the caller can reasonably handle (an I/O failure), unchecked for programming errors (`NullPointerException`, `IllegalArgumentException`, `IllegalStateException`). In practice the community has moved toward unchecked exceptions for most application code, because checked exceptions leak through layers, do not work well with lambdas and streams, and are often caught and ignored. Frameworks like Spring wrap checked exceptions (`SQLException`) into an unchecked hierarchy (`DataAccessException`). Be consistent: define a small set of domain exceptions and translate at layer boundaries.

**Rules.**
- Catch specific exceptions. Do not catch `Exception` or `Throwable` broadly except at a top-level boundary (an HTTP error handler, a worker loop) where you log and respond.
- **Never swallow** an exception with an empty catch. At minimum log it with context.
- **Preserve the cause**: `throw new OrderException("failed to place " + id, e);`. Losing the cause loses the stack trace you need.
- Log **or** rethrow, not both at every layer. Logging and rethrowing at each level produces duplicate noise. Log once where it is handled.
- Do not use exceptions for ordinary control flow. Creating an exception captures a stack trace, which is comparatively expensive.
- Do not return from or throw in `finally`, which can mask the original exception.
- `InterruptedException` means a thread was asked to stop. Do not swallow it. Either rethrow it or restore the flag with `Thread.currentThread().interrupt()`.

**try-with-resources.** Any `AutoCloseable` declared in the `try (...)` header is closed automatically, in reverse order, even when an exception occurs. If both the body and `close()` throw, the body's exception is primary and the close exception is attached as **suppressed**. Use it for every connection, statement, stream and file.

```java
try (Connection c = dataSource.getConnection();
     PreparedStatement ps = c.prepareStatement(SQL)) {
    ps.setLong(1, id);
    try (ResultSet rs = ps.executeQuery()) {
        // read rows
    }
}
```

Forgetting to close a pooled connection is a leak that eventually exhausts the pool (chapter 14).

**Error handling in web services.** Map exceptions to HTTP status codes in one place (in Spring, `@RestControllerAdvice` with `@ExceptionHandler`). Return a consistent error body, never leak stack traces or internal messages to clients, and include a correlation ID.

**Stack traces.** Read from the top: the exception and message, then the frames, then `Caused by:` chains. The root cause is usually the last `Caused by`. Learn to ignore framework frames and find your own code.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Resource safety | try-with-resources, `AutoCloseable`, `Closeable` | Never close resources manually in `finally` |
| Defensive checks | `Objects.requireNonNull`, `IllegalArgumentException`, `IllegalStateException` | Validate arguments at boundaries |
| Boundary handling | Spring `@RestControllerAdvice`, `@ExceptionHandler`, Problem Details (RFC 9457 style) responses | Centralised, consistent API errors |
| Logging | SLF4J API with Logback or Log4j 2, MDC for correlation IDs | Log with context, once |
| Reading failures | stack traces, `Throwable.getCause`, `getSuppressed` | Find the root cause quickly |

### Common mistakes

- Empty `catch` blocks and `e.printStackTrace()` in services.
- Catching `Exception` and losing the distinction between failures.
- Dropping the cause when wrapping.
- Swallowing `InterruptedException`, which makes shutdown hang.
- Logging passwords, tokens or whole request bodies in exception messages.
- Returning `null` for failure instead of throwing or returning `Optional`.

### Self-check

- Checked versus unchecked: what are the arguments for each and what does Spring do?
- What does try-with-resources guarantee, and what are suppressed exceptions?
- Why must you not swallow `InterruptedException`?
- Where should an exception be logged in a layered service?

---

## 8. Records, sealed types and pattern matching

**In plain words:** A record is a short way to write a class that only holds data. A sealed type allows only a fixed list of subtypes. Together with pattern matching (testing a type and reading its parts in one step), the compiler can check that you handled every case.

Release status, verified in the JEP index (openjdk.org/jeps/0): records are final in Java 16 (JEP 395), sealed classes in Java 17 (JEP 409), and pattern matching for `switch` in Java 21 (JEP 441). Pattern matching for `instanceof` is final in Java 16. Record patterns are final in Java 21 (from memory, not re-checked). Features outside these versions may still be preview, so check your JDK.

### Understand

**Records** are concise immutable data carriers. Declaring `record Point(int x, int y) {}` generates the constructor, accessors (`x()`, `y()`), `equals`, `hashCode` and `toString` based on all components. Fields are final. Records suit DTOs, API responses, value objects, map keys and tuple-like results. They can have a compact constructor for validation. Limits: a record cannot extend another class, and the shallow immutability still means a component that is a mutable list can change unless you copy it. They are not a good fit for JPA entities, which need a no-argument constructor and mutable state in most setups.

**Sealed types** restrict which classes may implement an interface or extend a class (`sealed interface Shape permits Circle, Square`). The compiler then knows the full set of subtypes. This models a closed set of alternatives, an algebraic data type: for example a `PaymentResult` that is exactly `Success`, `Declined` or `Failure`.

**Pattern matching** lets you test a type and extract data in one step. Two parts to know conceptually:

- `instanceof` with a binding variable removes the cast: `if (o instanceof Money m) { m.amountMinor(); }`.
- `switch` can match on type and deconstruct records. Combined with a sealed hierarchy, the compiler checks the switch is **exhaustive**: add a new subtype and every non-exhaustive switch fails to compile, which turns a runtime bug into a compile error.

```java
String describe(PaymentResult r) {
    return switch (r) {
        case Success s  -> "paid " + s.amount();
        case Declined d -> "declined: " + d.reason();
        case Failure f  -> "error: " + f.message();
    };
}
```

**Why this matters for backend work.** Closed result types make domain logic explicit and checkable, replace brittle `instanceof` chains and avoid the visitor pattern for simple cases. Records reduce boilerplate in API models, though JSON libraries and validation frameworks need versions that support them, so confirm support in your Jackson and framework versions.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Data carriers | `record`, compact constructors, nested records | Replace plain DTO boilerplate |
| Closed hierarchies | `sealed`, `permits`, `final`, `non-sealed` | Model a closed set of outcomes |
| Matching | `instanceof` patterns, `switch` expressions and patterns | Write exhaustive switches over a sealed type |
| Compatibility | release notes of your JDK, Jackson/Spring documentation | Confirm library support for the features you use |

### Common mistakes

- Using records as JPA entities.
- Assuming a record is deeply immutable.
- Adding a `default` branch to a switch over a sealed type, which defeats exhaustiveness checking.
- Using preview features in production without a plan to track them.

### Self-check

- What does a record generate, and what are its limits?
- How does a sealed hierarchy plus `switch` convert a runtime bug into a compile error?
- Why can a record still be mutated indirectly?

---

## 9. Concurrency foundations and the Java Memory Model

**In plain words:** Many threads (independent lines of execution) can run at the same time and share the same objects. This can cause wrong results if they change data together without rules. The Java Memory Model says when one thread is guaranteed to see another thread's changes.

### Understand

**Threads.** A Java thread maps to an operating system thread (platform thread) with its own stack. Threads share the heap, so shared mutable state is the source of all concurrency bugs. Node avoids this with a single-threaded event loop for JavaScript code. Java serves many requests in parallel across a thread pool, so every shared object (a singleton service, a static field, a cache) must be safe for concurrent use.

**Three hazards.**
1. **Race conditions**: the result depends on timing (check-then-act, read-modify-write such as `count++`, which is three steps).
2. **Visibility**: one thread's write may not be seen by another.
3. **Ordering**: the compiler, the JIT and the CPU may reorder operations as long as a single thread sees no difference.

**The Java Memory Model (JMM).** The JMM defines when a write by one thread is guaranteed to be visible to a read by another. It is expressed as **happens-before** relationships. If action A happens-before action B, then A's effects are visible to B and ordered before it. Key happens-before edges:

- Unlocking a monitor happens-before every later lock of the same monitor.
- A write to a `volatile` field happens-before every later read of that field.
- `Thread.start()` happens-before the started thread's actions. A thread's actions happen-before another thread's successful `join()` on it.
- Writes to `final` fields are safely published when the constructor completes (if `this` does not escape during construction).
- Actions before submitting a task to an executor happen-before the task runs, and the task's actions happen-before the result is retrieved through the `Future`.

Without a happens-before edge, a thread may see a stale value forever. Data without synchronisation can also appear "torn" for `long` and `double` on some platforms unless declared `volatile`.

**`synchronized`.** Acquires the object's **monitor** (an intrinsic lock). Only one thread holds a given monitor at a time, and entering and exiting give the visibility edges above. It is **reentrant** (a thread can re-enter a monitor it holds). Forms: a synchronised instance method (locks `this`), a synchronised static method (locks the `Class` object) and a synchronised block on a chosen object. Prefer a private final lock object over `this`, so outside code cannot interfere. Keep critical sections small, and never call unknown (alien) code or do I/O while holding a lock.

**`volatile`.** Guarantees visibility and ordering for a single variable, but **not atomicity** of compound operations. `volatile int count; count++` is still a race. Use it for flags (`volatile boolean running`) and safe publication of an immutable object, and use atomics or locks for read-modify-write.

**Deadlock.** Two or more threads each hold a lock and wait for the other's. Avoid it with a **global lock ordering**, holding fewer locks, using timeouts (`tryLock`) and avoiding calls to unknown code under locks. A thread dump (chapter 16) detects Java-level deadlocks and prints the cycle. Related: **livelock** (threads keep reacting without progress) and **starvation** (a thread never gets a lock or CPU).

**Thread safety strategies, in order of preference.**
1. **Do not share**: keep state local to the method or thread, make services stateless. Most Spring beans are stateless singletons, which is why they are safe.
2. **Share immutable data.**
3. **Use thread-safe building blocks**: concurrent collections, atomics, executors.
4. **Lock** with `synchronized` or `java.util.concurrent.locks`.

**Safe publication.** An object is safely published if it is shared through a happens-before edge: a `final` field, a `volatile` field, a lock, a concurrent collection or a static initialiser. A classic bug is double-checked locking without `volatile`: another thread can see a non-null reference to a partly constructed object. Prefer the holder-class idiom, an enum singleton or letting the framework manage singletons.

**`ThreadLocal`.** Per-thread storage. Used for per-request context (security, correlation IDs). In thread pools the thread is reused, so values must be cleared in `finally` or they leak to the next request and keep objects alive. `InheritableThreadLocal` and thread-local-heavy designs interact badly with very many threads and with virtual threads (chapter 10), where scoped values are the intended modern direction (final in Java 25, JEP 506; preview in earlier releases).

**Interruption.** Java's cooperative cancellation. `thread.interrupt()` sets a flag and wakes blocking calls with `InterruptedException`. Code should check and respond promptly. There is no safe way to forcibly stop a thread (`Thread.stop` is deprecated for removal and, in the Java 21 API docs, always throws `UnsupportedOperationException`).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Locks and visibility | `synchronized`, `volatile`, `final`, `java.util.concurrent.locks` | State exactly what each guarantees |
| Diagnosing | `jstack <pid>`, `jcmd <pid> Thread.print`, VisualVM, JFR | Read a thread dump and find blocked, waiting and deadlocked threads |
| Reasoning aids | happens-before rules (JLS chapter 17), `@GuardedBy` (annotation from Java Concurrency in Practice style libraries) | Document which lock protects which field |
| Stress testing | jcstress (OpenJDK), repeated tests under load | Know a tool exists to test memory-model behaviour |
| Static checks | SpotBugs, Error Prone, IDE concurrency inspections | Catch unsynchronised shared state |

### Common mistakes

- Believing `volatile` makes `count++` safe.
- Sharing a `SimpleDateFormat` or other non-thread-safe object across threads (use `java.time` formatters, which are thread-safe).
- Mutable fields in a singleton Spring bean.
- Synchronising on a string literal, a boxed value or a public object.
- Holding a lock while calling a remote service or the database.
- Not clearing a `ThreadLocal` in a pool.
- Swallowing `InterruptedException`.

### Self-check

- What is a happens-before relationship? Give three ways to create one.
- Why is `volatile int counter; counter++` not thread-safe?
- Why is a typical Spring singleton service safe without synchronisation?
- How do you detect and fix a deadlock?
- What is wrong with double-checked locking that omits `volatile`?

---

## 10. Concurrency toolkit: executors, CompletableFuture, locks, atomics, virtual threads

**In plain words:** Do not start threads by hand. Use an executor (a managed pool of worker threads) with limits on its size and queue. Virtual threads are very light threads that make simple blocking code scale, but they do not remove limits of the database or other services.

### Understand

**Do not create raw threads in services.** Creating a thread per task is expensive and unbounded. Use an **executor** that owns a pool and a queue.

**ExecutorService.** Submit `Runnable` or `Callable` tasks, get a `Future`. Created through `Executors` factories (`newFixedThreadPool`, `newCachedThreadPool`, `newSingleThreadExecutor`, `newScheduledThreadPool`) or, for production control, constructed directly as `ThreadPoolExecutor` with explicit parameters:

- **core and maximum pool size**, **keep-alive**, the **work queue** and the **rejection policy**.
- The behaviour is surprising: the pool grows beyond the core size only when the **queue is full**. With an unbounded queue (the default in `newFixedThreadPool`), the pool never grows past the core size and the queue grows without limit, hiding overload until memory runs out. `newCachedThreadPool` has an effectively unbounded thread count, which can exhaust memory under load.
- Production pools should use a **bounded queue**, a named thread factory (so thread dumps are readable), a deliberate rejection policy (for example caller-runs or abort with an error the caller handles) and an explicit shutdown (`shutdown`, `awaitTermination`, then `shutdownNow`).

**Sizing pools.** For CPU-bound work, roughly the number of cores. For blocking work (database, HTTP), more threads are needed, in proportion to the ratio of wait time to compute time. Better: measure, and bound concurrency to what downstream systems can handle (the database pool size is a hard ceiling, chapter 14). Separate pools for separate kinds of work prevent a slow dependency from starving everything (the **bulkhead** pattern).

**ForkJoinPool.** A work-stealing pool for divide-and-conquer CPU work. The shared **common pool** is used by parallel streams and by `CompletableFuture` async methods when no executor is given. Do not run blocking I/O on it.

**CompletableFuture.** Composable asynchronous results, similar in spirit to JavaScript Promises.

```java
CompletableFuture<Profile> profile =
    CompletableFuture.supplyAsync(() -> userClient.fetch(id), ioExecutor)
        .thenCombine(
            CompletableFuture.supplyAsync(() -> orderClient.recent(id), ioExecutor),
            Profile::new)
        .orTimeout(2, TimeUnit.SECONDS)
        .exceptionally(ex -> Profile.degraded(id));
```

Key operations: `supplyAsync`/`runAsync`, `thenApply` (map), `thenCompose` (flatMap), `thenCombine` (join two), `allOf`/`anyOf`, `handle`/`exceptionally`/`whenComplete` for errors. Rules:

- **Always pass your own executor** to the async methods for blocking work, so you do not starve the common pool.
- Without `orTimeout` or `completeOnTimeout` a future can hang forever. Always set timeouts on remote calls.
- Exceptions are wrapped in `CompletionException` when propagated through dependent stages. Unwrap when handling.
- Calling `join()` or `get()` blocks the thread. Compose rather than block, or block only at the boundary.
- Contrast with JavaScript: Promise callbacks run on the single event loop, while `CompletableFuture` callbacks run on whatever thread completed the stage or on the executor, so shared state is again your concern.

**Locks (`java.util.concurrent.locks`).** `ReentrantLock` offers what `synchronized` lacks: `tryLock` with timeout, interruptible acquisition, fairness option and multiple `Condition`s. Always unlock in `finally`:

```java
lock.lock();
try {
    // critical section
} finally {
    lock.unlock();
}
```

`ReadWriteLock` allows many readers or one writer, useful when reads dominate and are non-trivial. `StampedLock` offers optimistic reads for advanced cases. Start with `synchronized`, move to `ReentrantLock` when you need its features.

**Atomic classes.** `AtomicInteger`, `AtomicLong`, `AtomicReference`, and others use hardware **compare-and-swap (CAS)** for lock-free atomic updates: `incrementAndGet`, `compareAndSet`, `updateAndGet`. `LongAdder` scales better than `AtomicLong` under heavy write contention for counters and metrics. CAS retries under contention and suffers the ABA problem in some designs (`AtomicStampedReference` addresses it).

**Coordination utilities.** `CountDownLatch` (wait for N events, one use), `CyclicBarrier`, `Semaphore` (limit concurrent access, a simple way to cap concurrency to a downstream), `Phaser`, `Exchanger`. `BlockingQueue` for producer-consumer.

**Virtual threads (conceptual).** Modern Java adds **virtual threads**: lightweight threads managed by the JVM and mapped onto a small pool of OS carrier threads. They are a final feature in Java 21 (JEP 444, verified). `Executors.newVirtualThreadPerTaskExecutor()` and `Thread.ofVirtual()` exist since Java 21 (verified in the API docs). The idea:

- The classic model needs one costly OS thread per concurrent blocking request. A thread pool of a few hundred caps concurrency, so blocking I/O wastes threads.
- A virtual thread is cheap (small, heap-allocated stack). When it blocks on supported I/O or locks, the JVM **unmounts** it from the carrier, and the carrier runs another virtual thread. You can have very large numbers of them.
- The result: you write simple, blocking, thread-per-request code, with scalability approaching that of asynchronous or event-loop designs. This is the Java answer to Node's non-blocking model without callback or reactive complexity.
- Guidance: **do not pool virtual threads**. Create one per task (for example with an executor that starts a new virtual thread per task). Pooling is for scarce resources, and threads are no longer scarce.
- They do not make CPU-bound work faster. They help with many concurrent blocking operations.
- Limits are still real: a database pool of 20 connections still allows 20 concurrent queries. Use semaphores or pool limits to bound access to scarce downstream resources.
- Caveat: some operations **pin** a virtual thread to its carrier (the OS thread that runs it), which reduces scalability. In Java 21, blocking inside a `synchronized` block or in native code pins it (JEP 444). In Java 24, JEP 491 removed pinning for `synchronized`. Native calls that call back into Java and block, and blocking in class initialisers, can still pin. Heavy use of `ThreadLocal` is also costly.
- Frameworks may offer a configuration switch to run request handling on virtual threads. Check your framework version's documentation.

**Reactive programming** (Project Reactor, RxJava, Spring WebFlux) is the other approach to scalable I/O: non-blocking pipelines with back-pressure. It is powerful for streaming and high fan-out, but costs readability and debugging effort, and every layer including the database driver must be non-blocking to benefit. Know it exists and when it applies, and treat virtual threads as the simpler default for most new blocking-style services.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Pools | `ThreadPoolExecutor`, `Executors`, `ScheduledExecutorService`, `ForkJoinPool` | Configure a bounded, named, monitored pool with a rejection policy |
| Async composition | `CompletableFuture` | Fan out, join, time out and handle errors without blocking |
| Locks | `ReentrantLock`, `ReadWriteLock`, `StampedLock`, `Condition` | Use `tryLock` with timeout and unlock in `finally` |
| Atomics | `AtomicInteger`, `AtomicReference`, `LongAdder`, `ConcurrentHashMap.merge` | Choose atomics over locks for single-variable updates |
| Coordination | `Semaphore`, `CountDownLatch`, `BlockingQueue`, `CyclicBarrier` | Bound concurrency and hand off work safely |
| Virtual threads | `Thread.ofVirtual()`, `Executors.newVirtualThreadPerTaskExecutor()` (both Java 21 and later) | Explain when they help and what they do not fix |
| Reactive (awareness) | Project Reactor, Spring WebFlux, RxJava | Describe the trade-off versus virtual threads |
| Observation | `jstack`, JFR thread events, executor metrics via Micrometer | See queue depth and active threads in production |

### Common mistakes

- `newFixedThreadPool` with its unbounded queue as the production default.
- Blocking calls on the common `ForkJoinPool` through default `CompletableFuture` async methods or parallel streams.
- No timeouts on remote calls or futures.
- Not shutting down executors, leaving non-daemon threads that stop the JVM from exiting.
- Forgetting `unlock` in `finally`.
- Pooling virtual threads, or assuming they remove downstream capacity limits.
- Unnamed threads, which make thread dumps unreadable.
- Losing exceptions thrown in tasks submitted with `submit` (they are stored in the `Future`, not logged).

### Self-check

- Explain how `ThreadPoolExecutor` decides between queueing a task and creating a new thread.
- Why is `newFixedThreadPool` risky under overload?
- How does `CompletableFuture` differ from a JavaScript Promise in where callbacks run?
- When would you choose `LongAdder` over `AtomicLong`?
- What problem do virtual threads solve, and what do they not solve?

---

## 11. Garbage collection and memory

**In plain words:** The garbage collector (GC) frees objects that your program can no longer reach. It uses some CPU time and can pause your program. A memory leak in Java means objects are still reachable but no longer needed, so the GC cannot free them.

### Understand

**What GC does.** Java frees objects automatically. The collector finds objects that are no longer **reachable** from **GC roots** (thread stacks, static fields, JNI references) and reclaims their memory. You do not call `free`. This removes classes of bugs (dangling pointers, double free) but not memory leaks, and GC has costs: CPU time and **pauses**.

**The generational hypothesis.** Most objects die young (request-scoped objects, temporary strings). So the heap is divided:

- **Young generation**: new objects are allocated in **Eden**. A **minor GC** copies survivors to survivor spaces and eventually promotes long-lived objects. Minor collections are frequent and fast, because they scan only live objects in a small region.
- **Old generation (tenured)**: long-lived objects. Collected less often, and collection is more expensive. A **full GC** collects the whole heap and is the pause to avoid.
- Metaspace is separate (chapter 1).

Allocation is cheap (bump a pointer in a thread-local buffer), so short-lived garbage is not the problem. **Promotion of objects that should have died** (for example big per-request buffers that survive a young collection because the young generation is too small) and **leaks** are.

**Common collectors.** Names and defaults vary by JDK version, so confirm for yours.

| Collector | Character | Typical use |
|---|---|---|
| Serial | single-threaded, stop-the-world | tiny heaps, small containers, single CPU |
| Parallel | multi-threaded, throughput-oriented, longer pauses | batch jobs where total throughput matters more than pause time |
| G1 | region-based, mostly concurrent marking, aims for a pause-time goal (`-XX:MaxGCPauseMillis`) | the default on modern JDKs for server-class machines, good general-purpose choice |
| ZGC | concurrent, very low pause times, scales to large heaps | latency-sensitive services, very large heaps; generational mode became the default (JEP 474) and the non-generational mode was removed in Java 24 (JEP 490, verified) |
| Shenandoah | concurrent compaction for low pauses | similar goals to ZGC, availability depends on the JDK build |

Selection flags look like `-XX:+UseG1GC`, `-XX:+UseZGC`. Confirm exact flags for your JDK.

**Throughput versus latency versus footprint.** The central trade-off. A larger heap reduces GC frequency but raises memory cost and, for some collectors, pause length. Concurrent collectors cut pauses at the cost of CPU overhead and sometimes throughput.

**Tuning basics.** Most services need little tuning. A sane order of work:
1. Pick the JDK LTS and its default collector. Measure first.
2. Set heap size deliberately: `-Xms` equal to `-Xmx` avoids resize pauses and gives predictable memory. In containers, size relative to the limit and leave room for non-heap memory.
3. If pauses exceed the objective, check allocation rate and live set before touching flags. Reducing allocation (avoiding needless objects, boxing, large temporary buffers) is often the best tuning.
4. Adjust one thing at a time, and compare under realistic load.
5. Avoid copying flag lists from old blog posts. Many flags are obsolete or default already.
6. Do not call `System.gc()`. If an old library does, check `-XX:+DisableExplicitGC` or its alternatives for your setup.

**Reading GC logs.** Enable unified logging, for example `-Xlog:gc*:file=gc.log:time,uptime,level,tags` (confirm syntax for your JDK). A typical line reports the collection type and cause, heap before and after (`512M->128M(1024M)`) and the pause duration. Read for:

- **Pause times** versus your latency objective.
- **Frequency** of collections and the allocation rate implied.
- **Heap after full or mixed GC**: if the post-GC occupancy climbs steadily over hours, you have a leak or a growing live set. If it is flat but collections are frequent, you have high allocation churn.
- **Causes**: "Allocation Failure", "Humongous Allocation" (G1, objects larger than half a region, which are expensive: avoid giant arrays and buffers), "Metadata GC Threshold", "System.gc()".
- **Back-to-back full GCs** with little reclaimed: the heap is effectively full, and the service will soon throw `OutOfMemoryError` or spend all its time collecting (a GC "death spiral", reported as `GC overhead limit exceeded` on some collectors).

Visual tools such as GCViewer, GCeasy (an online service, avoid uploading sensitive logs) and JDK Mission Control help, but learn to read raw lines first.

**Memory leaks in a garbage-collected language.** A leak means objects that are **reachable but no longer needed**. Typical sources:

- **Unbounded caches and static collections** (a static `Map` that only grows).
- **Listeners and callbacks not deregistered.**
- **`ThreadLocal` values in pooled threads** (chapter 9).
- **Mutable keys** lost in a `HashMap` (chapter 3).
- **Unclosed resources** (connections, streams) and classloader leaks on redeploy in application servers.
- **Unbounded queues** between producers and slow consumers.
- Large **session** or request data kept alive.

Diagnosis: watch post-GC heap trend, take a **heap dump** (`jcmd <pid> GC.heap_dump file.hprof`) and analyse the **dominator tree** and **path to GC roots** in Eclipse MAT or VisualVM, or compare two histograms over time (`jcmd <pid> GC.class_histogram`). Set `-XX:+HeapDumpOnOutOfMemoryError` and `-XX:HeapDumpPath=...` so you capture evidence at the failure. Heap dumps contain all in-memory data, including secrets, so handle them as sensitive.

**`OutOfMemoryError` flavours.** "Java heap space" (heap exhausted: leak or undersized), "Metaspace" (class metadata: classloader leak or runaway dynamic class generation), "unable to create native thread" (OS thread or memory limit reached, often too many threads), "Direct buffer memory", and "GC overhead limit exceeded". The message tells you which area to investigate. And remember the operating-system OOM kill (chapter 1, exit code 137) leaves no Java exception at all.

**References.** `WeakReference`, `SoftReference` and `PhantomReference` allow the GC to reclaim an object when only they point to it. `WeakHashMap` uses weak keys. They are specialised tools and easy to misuse for caches. Prefer a real cache library with size and time limits. `finalize` is deprecated, do not use it. Use try-with-resources or `Cleaner` where needed.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| GC logging | `-Xlog:gc*` (unified logging), GCViewer | Read pause, frequency and post-GC occupancy |
| Live stats | `jstat -gcutil <pid> 1s`, `jcmd <pid> GC.heap_info`, `jcmd <pid> VM.native_memory` (needs NMT enabled) | Watch generations and native memory live |
| Heap dumps | `jcmd <pid> GC.heap_dump`, `jmap -dump:live,format=b,file=...`, `-XX:+HeapDumpOnOutOfMemoryError` | Capture a dump safely and know it pauses the JVM |
| Dump analysis | Eclipse MAT, VisualVM, JDK Mission Control | Find the dominator and the GC-root path of a leak |
| Allocation | JFR allocation profiling, async-profiler (third-party) | Find the code that allocates most |
| Flags | `-Xms`, `-Xmx`, `-XX:MaxRAMPercentage`, `-XX:MaxGCPauseMillis`, `-XX:+UseG1GC`/`-XX:+UseZGC` | Set heap and pick a collector deliberately |
| Container awareness | `java -XshowSettings:system -version` (confirm for your JDK) | Verify the JVM sees the container limits |

### Common mistakes

- Tuning flags before measuring, or copying flags from old articles.
- `-Xmx` equal to the container limit.
- Treating `OutOfMemoryError` as "add RAM" when the post-GC heap grows without bound.
- Giant per-request allocations (reading whole files or result sets into memory), causing humongous allocations and promotion.
- Caching without a size or time bound.
- Taking a heap dump from a production node under load with no plan (it pauses the JVM and the file is as large as the heap).
- Relying on `finalize` or `System.gc()`.

### Self-check

- Explain the generational hypothesis and why minor GCs are cheap.
- How do you tell a leak from high allocation churn using a GC log?
- Name three typical leak sources in a service and how a heap dump reveals them.
- Why might a container be OOM-killed while `-Xmx` was below the limit?
- When would you choose Parallel over G1, and ZGC over G1?

---

## 12. Build tools: Maven and Gradle

**In plain words:** A build tool compiles your code, runs tests and builds the final package. It also downloads the libraries (dependencies) your project needs. Maven and Gradle are the two common choices.

### Understand

A build tool compiles, tests, packages and manages **dependencies**. Backend Java uses **Maven** or **Gradle**. Contrast with `npm`: Maven and Gradle also run the compiler and test lifecycle, resolve a dependency graph from repositories (Maven Central is the default) and produce reproducible artifacts. Versions are normally pinned, not floating.

**Maven.** Declarative XML in `pom.xml`. Concepts:

- **Coordinates**: `groupId`, `artifactId`, `version` (GAV) identify an artifact.
- **Lifecycle phases**: `validate`, `compile`, `test`, `package`, `verify`, `install`, `deploy`. Running a phase runs all earlier ones. `mvn package` compiles, tests and builds the jar.
- **Plugins** do the work in each phase (compiler, surefire for unit tests, failsafe for integration tests).
- **Dependency scopes**: `compile` (default), `provided` (supplied by the runtime), `runtime`, `test`.
- **Transitive dependencies** and **conflict mediation**: the "nearest definition wins" rule can pick a different version than you expect. Inspect with the dependency tree.
- **Dependency management and BOMs**: a `dependencyManagement` section or a imported Bill of Materials fixes versions across a set of libraries. Spring Boot provides one, so you declare dependencies without versions.
- **Multi-module builds**: a parent POM with modules, shared versions and plugins.
- **Wrapper**: `mvnw` pins the Maven version for the project.

**Gradle.** Build scripts in Groovy or Kotlin DSL (`build.gradle`, `build.gradle.kts`). A task graph rather than fixed phases, with incremental builds, a build cache and a daemon, which make large builds faster. Configurations such as `implementation` (not exposed to consumers), `api` (exposed, for libraries via the `java-library` plugin), `runtimeOnly`, `testImplementation`. The wrapper (`gradlew`) pins the version. More flexible and faster on big builds, but scripts are code and can become complicated. Maven is stricter and more uniform.

**Practices for both.**
- Commit the wrapper and use it in CI.
- Pin versions, use a BOM, and avoid dynamic version ranges.
- Keep a single source of truth for the Java version (toolchain or release setting).
- Separate unit tests (fast, no I/O) from integration tests.
- Produce one runnable artifact: Spring Boot's plugin builds an executable "fat" jar, or build a layered container image.
- Scan dependencies for known vulnerabilities in CI (for example OWASP Dependency-Check, or your platform's equivalent) and update regularly.
- Know the dependency tree, since many production problems are version conflicts or an unexpected transitive library.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Maven | `mvn clean verify`, `mvn dependency:tree`, `mvn -pl <module> -am`, `mvn versions:display-dependency-updates`, `./mvnw` | Build, test, inspect and debug a dependency conflict |
| Gradle | `./gradlew build`, `./gradlew dependencies`, `./gradlew test --tests`, `--scan` (optional) | Same outcomes with Gradle tasks |
| Dependency hygiene | BOMs, `dependencyManagement`, OWASP Dependency-Check, Dependabot or Renovate | Keep versions aligned and updated |
| Packaging | Spring Boot Maven/Gradle plugin, Jib, Cloud Native Buildpacks | Produce a reproducible jar or image |
| Quality gates | JaCoCo (coverage), Checkstyle, SpotBugs, Spotless | Fail the build on agreed rules |

### Common mistakes

- Not knowing which version of a transitive dependency actually loads.
- Unpinned or snapshot dependencies in release builds.
- Building with a different JDK locally than in CI.
- Copying jars into the repository.
- Skipping tests in the default build (`-DskipTests` habit).
- Overcomplicated Gradle scripts nobody understands.

### Self-check

- What do the Maven lifecycle phases do, and what does `mvn package` run?
- How do you find out why a particular library version ended up on the classpath?
- What is a BOM and why does Spring Boot use one?
- `implementation` versus `api` in Gradle?

---

## 13. Spring Boot essentials

**In plain words:** Spring Boot helps you build a Java web service quickly, with sensible default settings. It creates and connects your objects for you (dependency injection). Most problems come from transactions, proxies and slow outgoing calls.

### Understand

**Spring** is the dominant Java application framework. **Spring Boot** adds opinionated defaults: auto-configuration, an embedded web server and a runnable jar, so a service starts with `java -jar`. Equivalent roles in Node: Express or NestJS plus a DI container, a config loader and an ORM, packaged together. Annotations and class names may change between major versions, so check the reference documentation for the version you use (for example, the move from `javax.*` to `jakarta.*` packages in recent major versions).

**Inversion of Control and dependency injection.** Instead of a class creating its collaborators with `new`, the container creates objects (**beans**) and **injects** their dependencies. Benefits: classes depend on interfaces, wiring is centralised, and tests can substitute fakes.

- **Beans** are objects managed by the container. Created by component scanning (`@Component`, `@Service`, `@Repository`, `@RestController`) or explicitly in a `@Configuration` class with `@Bean` methods.
- **Constructor injection** is the preferred style: dependencies are final and required, the class is easy to test without Spring, and a missing bean fails at startup. Avoid field injection (`@Autowired` on fields).
- **Scope**: beans are **singletons** by default (one instance, shared across all requests), so they must be stateless or thread-safe (chapter 9). `prototype` and request scopes exist for special cases.
- **Lifecycle**: instantiate, inject, initialise (`@PostConstruct`), use, destroy (`@PreDestroy`). Graceful shutdown lets in-flight requests finish.
- **Circular dependencies** between beans are a design smell. Break them with a restructure.

**How auto-configuration works (conceptually).** Boot examines the classpath and your properties and registers default beans conditionally (`@ConditionalOnClass`, `@ConditionalOnMissingBean`). Define your own bean and the default backs off. When something is surprising, inspect what was configured (the conditions evaluation report through debug logging, or the Actuator `beans` and `conditions` endpoints if exposed securely).

**REST controllers.**

```java
@RestController
@RequestMapping("/api/orders")
class OrderController {
    private final OrderService service;

    OrderController(OrderService service) { this.service = service; }

    @GetMapping("/{id}")
    OrderResponse get(@PathVariable long id) {
        return service.find(id);
    }

    @PostMapping
    ResponseEntity<OrderResponse> create(@Valid @RequestBody CreateOrderRequest req) {
        OrderResponse created = service.create(req);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }
}
```

Practices: thin controllers that translate HTTP to service calls, **DTOs** separate from persistence entities (do not expose entities directly: it leaks schema, causes lazy-loading surprises and couples your API to your tables), validation with Bean Validation annotations and `@Valid`, correct status codes, a central exception handler (chapter 7), and pagination for list endpoints. Request body serialisation is done by Jackson, so learn its annotations and configuration (unknown properties, date formats).

**Layering.** Controller (HTTP) calls service (business logic and transactions) calls repository (persistence). Keep transaction boundaries in the service layer.

**Configuration.** Externalised configuration in `application.properties` or `application.yml`, overridden by profiles (`application-prod.yml`), environment variables and command-line arguments, with a defined precedence (see the reference documentation). Bind groups of properties to typed classes with `@ConfigurationProperties` and validate them. Never commit secrets: inject them from the environment or a secrets manager. This matches the twelve-factor approach, and containers pass settings as environment variables. Use profiles for environment differences, not for logic branching.

**Transactions.** `@Transactional` on a service method wraps it in a database transaction: begin before, commit on success, roll back on failure. Understand the mechanics, because most pitfalls come from them:

- Implemented through **proxies**. A call from another bean goes through the proxy and is transactional. A **self-invocation** (a method calling another method of the same class) bypasses the proxy, so the annotation on the called method is ignored.
- By default, rollback happens on **unchecked exceptions and errors only**. A checked exception commits unless you configure `rollbackFor` (confirmed in the Spring Framework reference).
- The annotation works on public methods of proxied beans. Check the documentation for your proxy mode.
- **Propagation** (`REQUIRED` default, `REQUIRES_NEW` for an independent transaction, and others) and **isolation** levels control joining and visibility. Know `REQUIRED` and `REQUIRES_NEW`.
- `readOnly = true` is a hint that can enable optimisations and prevent accidental writes.
- A transaction holds a **database connection** for its whole duration. Doing slow remote calls inside a transaction ties up the pool (chapter 14). Keep transactions short and do external I/O outside them.
- Transactions do not span threads. Work handed to another thread runs outside the caller's transaction.
- Remember side effects that cannot roll back (sending an email, publishing a message) happen even if the transaction later fails. Patterns such as the **transactional outbox** exist for this, and you should know the problem.

**Other essentials.**
- **Actuator**: health, metrics and info endpoints for operations. Expose only what you need, and protect them. Wire liveness and readiness probes for your orchestrator.
- **Micrometer**: metrics facade, exported to Prometheus and others.
- **Spring Security**: authentication and authorisation filters. Learn the filter chain concept, stateless JWT or session options and method security, and never roll your own crypto.
- **Spring Data JPA**: repository interfaces with derived queries (chapter 14).
- **Async and scheduling**: `@Async`, `@Scheduled` use executors (chapter 10) with defaults you should review. A `@Scheduled` task runs on every instance unless you add coordination.
- **HTTP clients**: use a configured client (`RestClient`, `WebClient` or `RestTemplate`, depending on version) with **connect and read timeouts**, never the defaults.
- **Startup time and memory** are higher than Node. Know the options: lazy initialisation, class data sharing, native images, trimming auto-configuration.

**Common pitfalls summary.**
- **N+1 queries** (chapter 14).
- **Open Session in View**: Boot enables a mode (check the property for your version) that keeps the persistence context open through view rendering, letting lazy loading happen in controllers and hiding N+1 queries and long connection holds. Many teams disable it deliberately.
- Self-invocation breaking `@Transactional`, `@Async` and `@Cacheable`.
- Stateful singleton beans.
- Slow startup from heavy component scanning, and surprise beans from auto-configuration.
- Exposing entities in the API.
- Missing HTTP client timeouts.
- Logging sensitive request data.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Project start | Spring Initializr (start.spring.io), Spring Boot Maven/Gradle plugin | Generate and run a service in minutes |
| DI and config | `@Component`, `@Configuration`, `@Bean`, `@ConfigurationProperties`, profiles | Explain bean creation and wire configuration without field injection |
| Web | `@RestController`, `@RequestMapping`, `@Valid`, `@RestControllerAdvice`, Jackson | Build a validated, consistently erroring REST API |
| Transactions | `@Transactional` (propagation, rollbackFor, readOnly) | Predict commit and rollback in nested and self-invocation cases |
| Operations | Spring Boot Actuator, Micrometer, Prometheus, structured logging | Expose health and metrics safely |
| Security | Spring Security, OAuth2 resource server support | Secure endpoints with a standard mechanism |
| API docs | OpenAPI (springdoc-openapi, third-party) | Publish accurate contracts |
| Debugging wiring | debug logging of the conditions report, Actuator `beans` and `conditions` | Find why a bean exists or is missing |

### Common mistakes

- Field injection and `new`-ing services by hand.
- Business logic in controllers.
- Returning entities directly.
- Long transactions with remote calls inside.
- Assuming `@Transactional` works on self-invocation or on a checked exception.
- Putting secrets in `application.yml` in version control.
- No timeouts and no pool limits on outgoing calls.

### Self-check

- What is dependency injection, and why is constructor injection preferred?
- Why must a Spring singleton bean be stateless or thread-safe?
- Why does `@Transactional` sometimes silently do nothing?
- What is the default rollback rule?
- What is Open Session in View, and what problem can it hide?

---

## 14. JDBC, JPA and connection pools

**In plain words:** JDBC is the basic Java way to talk to a database. A connection pool keeps database connections open and lends them to requests. JPA maps objects to tables, but it can hide many extra queries (the N+1 problem), so always check which SQL runs.

### Understand

**JDBC** is the low-level standard API for relational databases: `DataSource`, `Connection`, `PreparedStatement`, `ResultSet`. Drivers implement it per database. Everything above (JdbcTemplate, JPA/Hibernate, jOOQ, MyBatis) is built on it.

**Always use `PreparedStatement` with bound parameters.** It prevents **SQL injection** (data is never concatenated into SQL) and lets the database reuse query plans. Never build SQL by string concatenation with user input.

**Why connection pools.** Opening a database connection is expensive (network handshake, authentication, server-side session). A **pool** keeps a set of open connections and lends them out. A request borrows a connection, uses it and returns it. **HikariCP** is the default pool in Spring Boot and is widely used. Node analogues: `pg.Pool` or knex pools.

**Pool sizing, the core ideas.**
- The pool size is a hard cap on **concurrent database work from this instance**. Total connections across all instances must stay below the database's own limit, with headroom for admin and migrations. With 10 app instances and a pool of 20, the database may see 200.
- Bigger is not faster. The database has limited CPU and disk, and more concurrent queries than it can run in parallel just add contention. A modest pool is often better than a large one. Size from measurements and the database's capacity, and see the HikariCP project's sizing guidance in the references.
- Threads waiting for a connection show up as latency, then timeouts. The **connection timeout** setting controls how long a caller waits before failing. Keep it short enough to shed load.

**Important HikariCP settings** (names from its documentation, confirm defaults for your version): `maximumPoolSize`, `minimumIdle`, `connectionTimeout`, `idleTimeout`, `maxLifetime` (set it shorter than any database or network idle limit so the pool retires connections before something kills them silently), `leakDetectionThreshold` (logs a stack trace for connections held too long, which is invaluable for finding leaks).

**Pool exhaustion.** Symptoms: timeouts getting a connection, requests piling up, the database mostly idle. Causes: **leaked connections** (not closed: use try-with-resources or let the framework manage them), **long transactions** (slow remote calls inside `@Transactional`), **slow queries**, a pool too small for the load, or too many threads for the pool. A thread dump shows many threads blocked in the pool's `getConnection`, and Hikari metrics (active, idle, pending threads) confirm it.

**JPA and Hibernate.** JPA is the specification for object-relational mapping, and Hibernate is the dominant implementation. Entities map to tables, and the **persistence context** (first-level cache, tied to a transaction) tracks managed entities and **flushes** changes to the database on commit (**dirty checking**). Convenient, but it hides SQL, so you must still know what queries run.

**The N+1 problem.** Loading a list of N parent entities (one query) and then touching a lazily loaded association on each one triggers N further queries.

```java
List<Order> orders = orderRepository.findAll();      // 1 query
for (Order o : orders) {
    o.getItems().size();                              // N more queries
}
```

Impact: latency grows with the data size, and the pool is held for the duration. It is invisible in small test data. Fixes, with their trade-offs:

- **Fetch join** in the query (`select o from Order o join fetch o.items`). One query, but fetching several collections multiplies rows, and combining collection fetch with pagination causes in-memory pagination (Hibernate warns about it).
- **Entity graphs** (`@EntityGraph`) to declare what to load for a given repository method.
- **Batch fetching** (`@BatchSize` or the Hibernate batch fetch size property) loads associations for many parents with an `IN` query, a good general mitigation.
- **Projections or DTO queries** that select only the columns the endpoint needs, often the best option for read APIs.
- **Do not just switch everything to eager.** Eager fetching loads data you may not need on every query and can create its own N+1.
- Detect it: enable SQL logging in development, use Hibernate statistics, a datasource proxy that counts queries per request, or tests that assert query counts.

**Other persistence pitfalls.**
- **LazyInitializationException**: touching a lazy association after the transaction and session closed. Fix by loading what you need inside the service, not by widening the session.
- **Mapping bidirectional relations and `equals`/`hashCode`/`toString`** that traverse the graph and recurse or trigger loads.
- **Huge result sets** loaded into memory: use pagination or streaming with care.
- **Schema management**: do not let Hibernate auto-create or update schema in production. Use migration tools (Flyway or Liquibase) with versioned scripts.
- **Optimistic locking** with a `@Version` column detects concurrent modification. Know it, because lost updates are a real concurrency bug.
- **Batch inserts** need configuration and are defeated by some ID generation strategies (identity columns).
- Knowing SQL well matters more than any ORM feature: indexes, `EXPLAIN`, query plans (see the database chapters of this repository).

**Alternatives.** `JdbcTemplate` and Spring's `JdbcClient` for explicit SQL, jOOQ for type-safe SQL, MyBatis for SQL mapping. Many teams use JPA for simple writes and explicit SQL for reporting and complex reads.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| JDBC | `DataSource`, `PreparedStatement`, `ResultSet`, batch updates, try-with-resources | Write safe JDBC without a framework |
| Pools | HikariCP configuration, Hikari metrics through Micrometer, `leakDetectionThreshold` | Size, monitor and diagnose a pool |
| ORM | Hibernate, Spring Data JPA, `@EntityGraph`, `@BatchSize`, fetch joins, DTO projections | Remove an N+1 and justify the chosen fix |
| Observing SQL | Hibernate SQL logging, Hibernate statistics, datasource-proxy or p6spy (third-party), database slow query log | Count queries per request |
| Migrations | Flyway, Liquibase | Version the schema and apply it in CI |
| Query tuning | `EXPLAIN (ANALYZE)` in your database, `pg_stat_statements` (PostgreSQL) | Tie a slow endpoint to a slow query |
| Alternatives | JdbcTemplate/JdbcClient, jOOQ, MyBatis | Know when plain SQL is better |

### Common mistakes

- Not noticing N+1 until production data volume.
- Leaking connections or holding them during remote calls.
- A very large pool "to be safe", which overloads the database.
- Maximum lifetime longer than the database or proxy's connection timeout, causing random connection resets.
- Returning entities from controllers and hitting lazy loading outside a transaction.
- Letting Hibernate generate the production schema.
- Building SQL strings with concatenation.

### Self-check

- What is N+1, how do you detect it and give three fixes with trade-offs?
- Why does a bigger pool not necessarily improve throughput?
- Describe pool exhaustion symptoms and how you would prove the cause.
- Why is `maxLifetime` set below the database's connection time limit?
- How does a prepared statement protect against SQL injection?

---

## 15. Testing with JUnit and Mockito

**In plain words:** Write many small, fast tests and fewer slow tests that use real parts. JUnit runs the tests. Mockito creates fake objects (mocks) so you can test one class alone, but do not fake everything.

### Understand

**Test pyramid.** Many fast **unit tests**, fewer **integration tests** (real database, real HTTP layer), few end-to-end tests. Fast feedback and reliability come from the base.

**JUnit.** The standard test framework. Modern version is JUnit 5 (Jupiter). Core concepts:

- `@Test` methods, lifecycle callbacks (`@BeforeEach`, `@AfterEach`, `@BeforeAll`, `@AfterAll`), `@Nested` classes for grouping and `@DisplayName`.
- **Parameterised tests** (`@ParameterizedTest` with `@ValueSource`, `@CsvSource`, `@MethodSource`) cover many cases without repetition.
- **Assertions**: JUnit's own `assertEquals`, `assertThrows`, `assertAll`, or the fluent AssertJ library (`assertThat(list).containsExactly(...)`) for readable failures.
- Extensions (`@ExtendWith`) integrate Mockito, Spring and others.
- Structure each test as arrange, act, assert, and test one behaviour. Name tests by behaviour.

**Mockito.** A mocking library. A mock replaces a collaborator so the unit is tested in isolation.

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock PaymentClient payments;
    @InjectMocks OrderService service;

    @Test
    void failsWhenPaymentDeclined() {
        when(payments.charge(any())).thenReturn(PaymentResult.declined());

        assertThrows(PaymentDeclinedException.class, () -> service.place(request()));

        verify(payments).charge(any());
    }
}
```

- `when(...).thenReturn/thenThrow` stubs behaviour. `verify` checks interactions. `ArgumentCaptor` captures arguments for assertions. `@Spy` wraps a real object (use sparingly).
- Constructor injection makes this trivial: pass mocks to the constructor, no Spring needed.
- **Do not over-mock.** Mocking everything yields tests that restate the implementation and break on every refactor. Prefer real objects for simple collaborators and value types, mock only boundaries (network, clock, slow or non-deterministic things), and prefer fakes (an in-memory repository) when behaviour matters. Do not mock types you do not own without a wrapper. Verify outcomes (state) more than calls.
- Strict stubs (the default with the Mockito extension) fail on unused stubbing, which keeps tests honest.

**Spring Boot testing.** Slice tests start part of the context: `@WebMvcTest` for controllers with `MockMvc`, `@DataJpaTest` for repositories, and `@SpringBootTest` for the full context, which is slow, so use it sparingly. `@MockitoBean` (from Spring Framework 6.2 and later; the Spring Boot 4.1 testing docs use it) replaces a bean in the context. The older Spring Boot `@MockBean` was deprecated in Boot 3.4 and removed in Boot 4.0 (from memory, not re-checked in release notes).

**Integration tests with real dependencies.** Use **Testcontainers** to start a real PostgreSQL or Kafka in a container for tests. This avoids the behavioural differences of in-memory substitutes such as H2, which differ in SQL dialect, locking and types. It needs a container runtime in CI.

**Other practices.**
- Make time and randomness injectable (`java.time.Clock`) so tests are deterministic.
- Avoid shared state between tests and order dependence. Flaky tests erode trust, so fix or delete them.
- **Contract testing** (for example Pact) and API-level tests (REST Assured, `WebTestClient`) verify service boundaries.
- **Coverage** (JaCoCo) is a signal, not a goal. A line executed is not a behaviour verified.
- **Mutation testing** (PIT) measures whether tests would catch bugs.
- For concurrency, tests are weak at proving correctness. Use design (immutability, confinement), stress tests and tools such as jcstress where it matters. Avoid `Thread.sleep` in tests and prefer Awaitility to wait for asynchronous conditions.
- Benchmarks belong in JMH, not unit tests.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Unit testing | JUnit 5, AssertJ, parameterised tests | Write clear, isolated, behaviour-named tests |
| Mocking | Mockito (`@Mock`, `@InjectMocks`, `ArgumentCaptor`, `verify`) | Isolate a service and know when not to mock |
| Spring tests | `@WebMvcTest`, `MockMvc`, `@DataJpaTest`, `@SpringBootTest` | Choose the smallest slice that tests the behaviour |
| Real dependencies | Testcontainers | Run repository tests against the real database engine |
| Async | Awaitility | Assert eventual outcomes without sleeping |
| Quality | JaCoCo, PIT, WireMock (stub HTTP dependencies) | Interpret coverage critically |
| Contracts | Pact, Spring Cloud Contract | Protect inter-service APIs |

### Common mistakes

- Mocking value objects and the class under test.
- Tests coupled to implementation details through excessive `verify`.
- Using H2 as a stand-in and discovering production SQL differences late.
- `@SpringBootTest` for everything, making the suite slow.
- Sleep-based synchronisation, shared test data and test-order dependence.
- Chasing a coverage number.

### Self-check

- What should you mock and what not?
- Why prefer Testcontainers over an in-memory database for repository tests?
- How do you test code that depends on the current time?
- What is the difference between `@WebMvcTest` and `@SpringBootTest`?

---

## 16. Profiling and troubleshooting

**In plain words:** When a service is slow or stuck, collect evidence before you restart it. The JDK has tools to show what every thread is doing, how memory is used and where time is spent. Compare several samples, not one.

### Understand

Diagnose with evidence, not guesses (the method in the Linux chapter applies: observe, hypothesise, test, change one thing, verify). The JDK ships capable tools that attach to a running JVM, usually with the same user as the target process. Most can be used in production, but some have a cost, so know which.

**The standard tools.**

- **`jcmd`** is the unified entry point. `jcmd` alone lists JVMs. `jcmd <pid> help` lists commands for that process. Common ones: `Thread.print` (thread dump), `GC.heap_info`, `GC.class_histogram`, `GC.heap_dump <file>`, `VM.flags`, `VM.system_properties`, `JFR.start`, `JFR.dump`, `JFR.stop`.
- **`jstack <pid>`** prints a thread dump: every thread's state and stack, plus detected **Java-level deadlocks**. Take **three dumps a few seconds apart**: a thread in the same place in all three is stuck or hot, not just passing through.
- **`jmap`**: heap histogram and heap dumps (`jcmd` is the preferred modern route).
- **`jstat -gcutil <pid> 1000`**: live GC statistics (generation occupancy and GC counts and times).
- **`jinfo`**: view and sometimes change flags of a running JVM.
- **Java Flight Recorder (JFR)** is a low-overhead event recorder built into the JVM, safe for production. It records method samples, allocations, GC, locks, I/O and thread events. Start it with `jcmd <pid> JFR.start` or a startup flag and open the `.jfr` file in **JDK Mission Control** or read it with the `jfr` command-line tool. JFR is the first-choice profiler for production questions.
- **VisualVM** is a free GUI for live monitoring, sampling CPU and memory, thread views and heap dump analysis. Convenient in development and staging.
- **Eclipse MAT** for heap dump analysis (leak suspects, dominator tree).
- **async-profiler** (third-party) produces accurate CPU, allocation and lock flame graphs with low overhead and avoids the safepoint bias of some profilers.
- **Application metrics and traces**: Micrometer, Prometheus and Grafana, OpenTelemetry. JVM metrics to chart: heap used after GC, GC pause time and count, thread counts, connection pool active and pending, executor queue depth, request latency percentiles, error rates. Dashboards tell you something is wrong, the tools above tell you why.

**Reading a thread dump.** Each thread has a name, a state and a stack.

| State | Meaning | Typical interpretation |
|---|---|---|
| `RUNNABLE` | running or ready, or in native code such as socket reads | CPU-bound work, or blocked in native I/O (a socket read shows as runnable) |
| `BLOCKED` | waiting to enter a `synchronized` monitor | lock contention. The dump names the owner thread |
| `WAITING` | waiting indefinitely (`Object.wait`, `park`, `join`) | idle pool threads, or waiting for a result |
| `TIMED_WAITING` | waiting with a timeout (`sleep`, timed `wait`) | polling, scheduled work, timeouts |

Patterns: many threads `BLOCKED` on one monitor is lock contention. Many request threads parked in the pool's `getConnection` is pool exhaustion. Many threads in the same remote call is a slow dependency. A "Found one Java-level deadlock" section names the cycle. Named threads make all of this readable (chapter 10).

**Investigation playbooks.**

*High CPU.* Find the process (`top`, `pidstat`). Find the hot thread (`top -H -p <pid>` and convert the thread ID to hexadecimal to match `nid` in a thread dump), or record a JFR or async-profiler CPU profile. Common causes: a hot loop, a regular expression with catastrophic backtracking, serialisation, **GC** (check `jstat` first, since GC threads can be the CPU consumer), or excessive logging.

*High latency.* Look at percentiles and correlate with GC pauses (GC log), pool saturation (Hikari and executor metrics), lock contention (JFR lock events or thread dumps), slow queries and downstream timings (traces). Distinguish waiting from working.

*Memory growth or OOM.* GC log and post-GC heap trend, class histogram diffs over time, a heap dump analysed in MAT, `HeapDumpOnOutOfMemoryError` for the next occurrence. Check native memory and thread counts if the heap looks fine (chapter 11).

*Hang or deadlock.* Thread dumps (three, spaced). Look for `BLOCKED` threads and the deadlock section, and for all pool threads waiting on a shared lock or downstream call.

*Cannot reproduce.* Add instrumentation, enable continuous JFR with a bounded circular recording so the last minutes are available after an incident, and compare with a baseline.

**Operational safety.** Taking a heap dump pauses the JVM and writes a file as large as the live heap, so check disk space and impact first. Dumps contain sensitive data. Enabling remote JMX or debug ports without authentication and network restrictions is a serious security hole. Prefer attaching locally (`jcmd` over SSH or in the container) and exposing metrics instead. In a minimal container image the JDK tools may be absent, so plan for that (an ephemeral debug container, or a JDK-based image in non-production), and confirm the tool can see the process when running as the correct user.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Entry point | `jcmd` (`Thread.print`, `GC.heap_info`, `GC.class_histogram`, `GC.heap_dump`, `JFR.*`, `VM.*`) | Run any diagnostic through one tool |
| Thread analysis | `jstack`, `top -H`, fastThread or similar (avoid uploading sensitive dumps), VisualVM | Diagnose contention, deadlock and pool exhaustion |
| Memory | `jmap`, `jstat -gcutil`, Eclipse MAT, VisualVM | Find a leak from a dump or histogram diff |
| Continuous profiling | JFR and JDK Mission Control, `jfr print`/`jfr summary` | Record safely in production and read the result |
| Flame graphs | async-profiler | Locate CPU and allocation hot spots |
| Metrics and traces | Micrometer, Prometheus, Grafana, OpenTelemetry | Chart the JVM and pool signals above |
| Benchmarks | JMH | Measure a change correctly |
| OS side | `top`, `pidstat`, `ss`, `strace`, `lsof` (see the Linux chapter) | Connect JVM behaviour to the host |

### Common mistakes

- A single thread dump, or none, before restarting.
- Restarting first and losing the evidence.
- Profiling in a development environment with no load and trusting the result.
- Reading `RUNNABLE` as "using CPU" when the thread is in a socket read.
- Taking a heap dump on a saturated node without planning.
- Open JMX or debug ports on production hosts.
- Assuming CPU spikes are application code when GC threads are the cause.

### Self-check

- A service hangs. What do you capture, and how do you read it?
- How do you find which Java thread is burning CPU?
- What does JFR give you that a thread dump does not, and why is it safe in production?
- What do many `BLOCKED` threads versus many threads waiting in `getConnection` each indicate?
- What risks come with taking a heap dump of a live production process?

---

## 17. Java versus Node for backend services

**In plain words:** Java and Node handle many users in different ways. Java uses many threads that share memory, and Node uses one event loop (a single thread that handles events one by one). Choose by the workload and the team, not by habit.

### Understand

Neither is universally better. Pick by workload, team and operational context.

**Concurrency model.**
- **Node**: one JavaScript thread runs an event loop. I/O is non-blocking and handled by the runtime, so a single process handles many concurrent connections cheaply. A CPU-heavy task blocks the loop and stalls every request, unless moved to worker threads or another process. State is not shared across requests by accident of threading, but a single process uses one core, so you run several processes (cluster, containers) to use the machine.
- **Java**: many threads run truly in parallel on all cores and share memory. CPU-heavy work parallelises naturally within one process, and a large heap is shared (in-process caches). The cost is shared-state hazards (chapters 9 and 10). Virtual threads allow a simple blocking style to scale for I/O-heavy work.

**Performance profile.** Java has slower startup and warm-up and a larger baseline memory footprint, then excellent sustained throughput and predictable behaviour from the JIT. Node starts quickly, uses less memory at small scale and is very efficient for I/O-bound, light-compute services. For CPU-bound or very high-throughput, long-lived services, Java usually has the edge. For many small, short-lived functions, Node's startup is an advantage (though native-image and snapshot techniques narrow the gap for Java). Measure your own workload rather than trusting generalities.

**Type system and refactoring.** Java is statically typed with a mature compiler and tooling, so large-scale refactoring and API contracts are safer. TypeScript gives Node similar benefits at compile time, but its types are erased and not enforced at runtime, so input validation is still required at boundaries. Java also checks types at runtime, though you still must validate external input.

**Ecosystem and maturity.** Java has a long history in enterprise systems, a rich ecosystem for data, messaging, transactions, security and observability, and strong backward compatibility. Node has a fast-moving ecosystem, very quick iteration, and the advantage of one language across frontend and backend. Dependency hygiene matters in both (npm and Maven Central each have supply-chain risk).

**Operational concerns.**
- Memory: tune heap and understand GC (chapter 11). Node has a V8 heap limit and a different GC, with its own tuning flags.
- Observability: both have strong support. The JVM offers unusually deep built-in introspection (JFR, `jcmd`).
- Deployment: a Java service is a jar on a JVM, with container sizing and startup time to plan. Node deploys source plus `node_modules`.
- Graceful shutdown and health checks are required in both. Spring Boot provides them through configuration.

**Developer experience.** Node: less boilerplate, faster feedback, easy scripting. Java: more ceremony, though modern features (records, `var`, pattern matching) and frameworks reduce it, plus excellent IDE support and debuggers.

**Typical guidance.**

| Situation | Leans toward |
|---|---|
| I/O-bound API gateway, BFF, real-time features, small team using TypeScript end to end | Node |
| Complex domain logic, transactional systems, heavy CPU or data processing, long-lived high-throughput services, large teams | Java |
| Serverless with strict cold-start budgets | Node (or Java with native image if the team accepts the trade-offs) |
| Existing ecosystem, hiring pool and operational skills | whichever the organisation already runs well |

In interviews, avoid tribal claims. Show that you understand the concurrency models, the GC and warm-up trade-offs, and that you decide using workload and team constraints.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Load comparison | k6, wrk, Gatling (Java-based), JMeter | Load test both stacks with realistic scenarios |
| Observability parity | OpenTelemetry for Java and Node | Instrument either stack consistently |
| Startup and size | GraalVM native image, class data sharing, container layering | Describe options for fast startup |
| Interop | REST, gRPC, message brokers (Kafka, RabbitMQ) | Mix both stacks behind clear service contracts |

### Common mistakes

- Choosing by preference or benchmark folklore rather than workload.
- Comparing a cold JVM against a warm Node process, or the reverse.
- Running a CPU-heavy task on the Node event loop, or a blocking call inside a Java pool sized for CPU work.
- Ignoring operational skills when choosing.

### Self-check

- Explain how each runtime handles 10,000 concurrent slow HTTP calls.
- What happens to a Node service when one request does 2 seconds of CPU work? What happens in Java?
- Why can warm-up distort a comparison?

---

## 18. Practice labs

**In plain words:** These exercises turn the ideas above into skills. Each one has a clear goal, so you know when you are done.

Do these on a machine with a JDK installed and a build tool. Each lab states the goal and what "done" looks like. Attempt before looking anything up.

1. **Inspect the JVM.** Write a small program, compile it with `javac`, run it, and use `javap -c` to read its bytecode. List the effective heap size and GC with `-XX:+PrintFlagsFinal`. Done when you can explain each part of the output you looked at.
2. **Boxing and equality.** Demonstrate `Integer` caching with `==`, an overflow with `Integer.MAX_VALUE + 1`, and a `BigDecimal` constructed from a double versus from a string. Write down why each result occurs.
3. **Break and fix equals/hashCode.** Create a key class that overrides only `equals`, show it fails in a `HashMap`, fix it, then mutate a key after insertion and show the entry becomes unreachable.
4. **HashMap in practice.** Insert keys with a deliberately bad `hashCode` (a constant), measure lookup time against a good one, and explain the result in terms of buckets and tree conversion.
5. **Collections choice.** Benchmark (with JMH) `ArrayList` versus `LinkedList` for random access and middle insertion, and `ArrayDeque` as a queue. State your conclusions with numbers.
6. **Streams.** Group, partition and aggregate a list of orders. Rewrite one pipeline as a loop and decide which is clearer. Show the effect of `toMap` with duplicate keys.
7. **Exceptions.** Write a try-with-resources example where both the body and `close()` fail, and print the suppressed exception. Add a global `@RestControllerAdvice` returning a consistent error body.
8. **Race and fix.** Increment a shared counter from many threads using a plain `int`, then `volatile`, then `synchronized`, `AtomicInteger` and `LongAdder`. Record results and explain each.
9. **Deadlock on purpose.** Create a two-lock deadlock, capture it with `jstack`, read the deadlock report, then fix it with lock ordering.
10. **Bounded executor.** Build a `ThreadPoolExecutor` with a bounded queue, named threads and a rejection policy. Flood it, observe rejection, and compare with `newFixedThreadPool` under the same load using memory growth as the measure.
11. **CompletableFuture.** Call two simulated slow services in parallel, combine results, add a timeout and a fallback, and run blocking work on your own executor. Show the problem when you use the common pool.
12. **Virtual threads.** Run a thousand tasks that each sleep or block on I/O using a fixed pool of platform threads, then with a virtual-thread-per-task executor. Compare total time, and note what does not improve for CPU-bound work.
13. **GC logging.** Run an allocation-heavy program with GC logging enabled under G1, then Parallel, then ZGC (if your JDK supports it). Compare pause times and throughput from the logs.
14. **Create and find a leak.** Write a service with a static map that grows. Watch post-GC heap rise in `jstat`, capture a heap dump with `jcmd`, and find the leak in Eclipse MAT through the dominator tree.
15. **Spring Boot service.** Build a small REST service with validation, DTOs, a service layer, configuration via `@ConfigurationProperties`, a Flyway migration and Actuator health. Done when it runs from the jar and the errors have a consistent format.
16. **Transaction behaviour.** Demonstrate a self-invocation that skips `@Transactional`, a checked exception that does not roll back, and `REQUIRES_NEW` committing independently of a failing outer transaction.
17. **N+1.** Create parent and child entities, enable SQL logging, reproduce N+1, then fix it with each approach (fetch join, entity graph, batch size, DTO projection). Record the query counts before and after.
18. **Pool exhaustion.** Configure a tiny HikariCP pool, hold connections in a slow transaction, observe timeouts, use `leakDetectionThreshold` and a thread dump to find the cause, and fix it.
19. **Testing.** Unit test a service with Mockito, test a repository against PostgreSQL with Testcontainers, test a controller with `@WebMvcTest`, and make a time-dependent class testable with an injected `Clock`.
20. **Production-style incident.** Using JFR, record a service under load that has one slow query, one lock contention point and a GC-heavy endpoint. Identify each from the recording, and write a one-page timeline: symptom, evidence, cause, fix, prevention.

---

## 19. Interview questions

**In plain words:** These are common questions with short model answers. Each question has a level tag: [Fresher] for early career, [Mid] for some years of experience, [Senior] for design and operations. The answers are study aids, so practise saying them in your own words.

### Top 10 must-know

Learn these first. Numbers refer to the model answers below.

1. [Fresher] **4.** `==` versus `equals`.
2. [Fresher] **2.** Pass by value for objects.
3. [Mid] **13.** The equals/hashCode contract.
4. [Mid] **14.** How `HashMap` works and resizes.
5. [Mid] **16.** What `volatile` does and does not do.
6. [Mid] **18.** How `ThreadPoolExecutor` queues and grows.
7. [Mid] **20.** The N+1 problem and its fixes.
8. [Mid] **21.** How `@Transactional` works and when it fails.
9. [Senior] **30.** Investigating growing latency and OOM.
10. [Senior] **32.** Virtual threads and their limits.

Questions are grouped by level. Every question has a model answer in the subsection that follows.

### Fresher

1. [Fresher] What is the difference between the JDK, the JRE and the JVM?
2. [Fresher] Explain pass by value in Java.
3. [Fresher] Why are strings immutable, and why does it matter?
4. [Fresher] What is the difference between `==` and `equals`?
5. [Fresher] Explain autoboxing and one pitfall of it.
6. [Fresher] What is the difference between an interface and an abstract class?
7. [Fresher] What is the difference between `ArrayList` and `LinkedList`?
8. [Fresher] What is the difference between checked and unchecked exceptions?
9. [Fresher] What does try-with-resources do?
10. [Fresher] What is the difference between `HashMap` and `Hashtable`/`ConcurrentHashMap`?
11. [Fresher] What is a functional interface and a lambda?
12. [Fresher] What does `final` mean on a variable, a method and a class?

### Mid

13. [Mid] State the equals/hashCode contract. What breaks if you override only `equals`?
14. [Mid] How does `HashMap` work internally, and when does it resize?
15. [Mid] What is type erasure?
16. [Mid] What does `volatile` guarantee, and what does it not?
17. [Mid] Explain `synchronized` and what visibility guarantee it provides.
18. [Mid] How does `ThreadPoolExecutor` decide when to create threads and when to queue?
19. [Mid] What is `CompletableFuture`, and why pass your own executor?
20. [Mid] What is the N+1 problem in JPA and how do you fix it?
21. [Mid] How does `@Transactional` work, and when does it not?
22. [Mid] How does HikariCP help and how do you size it?
23. [Mid] How does constructor injection differ from field injection, and why prefer it?
24. [Mid] When do you use `Optional`, and when not?
25. [Mid] What are the generational GC regions and why is minor GC cheap?
26. [Mid] What do you mock in unit tests and what not?
27. [Mid] How do you detect a deadlock?
28. [Mid] What is the difference between `orElse` and `orElseGet`?

### Senior and design

29. [Senior] Explain the Java Memory Model and happens-before.
30. [Senior] A service has increasing latency over hours and then restarts with OOM. How do you investigate?
31. [Senior] How would you choose a garbage collector and heap size for a latency-sensitive service in a container?
32. [Senior] What are virtual threads, what problem do they solve and what limits remain?
33. [Senior] Pool exhaustion: a service stalls while the database is idle. Diagnose it.
34. [Senior] Design a bounded, resilient way to call a slow downstream service from a request handler.
35. [Senior] How do you make a Spring service safe for concurrent requests?
36. [Senior] What are the pitfalls of long transactions and external calls inside them?
37. [Senior] Java versus Node: how do you decide for a new backend service?
38. [Senior] How do you roll out a JVM upgrade or collector change safely?
39. [Senior] How would you find and fix a CPU spike in production?
40. [Senior] How do you design for graceful shutdown of a Java service under an orchestrator?

### Model answers (condensed)

**1.** The JVM executes bytecode. The JRE is the JVM plus the standard libraries. The JDK adds development tools (`javac`, `jcmd`, `jstack`). You need the JDK to build and diagnose.

**2.** Arguments are copied. For objects the copied value is the reference, so a method can mutate the referenced object, but cannot rebind the caller's variable.

**3.** An immutable string is safe to share across threads, safe as a map key (hash cached), enables interning and prevents tampering after validation. The cost is that concatenation in loops creates many objects, so use `StringBuilder`.

**4.** `==` compares primitive values or object references (identity). `equals` compares logical equality as defined by the class. Use `equals` for strings and boxed types.

**5.** The compiler converts between primitives and wrappers automatically. Pitfalls: `NullPointerException` when unboxing `null`, reference comparison with `==` on wrappers, and hidden allocation in loops and large collections.

**6.** An interface defines a contract, has no instance state and a class can implement many. An abstract class can hold state and partial implementation, and a class extends only one. Prefer interfaces for contracts, an abstract class for shared state or a template skeleton.

**7.** `ArrayList` is an array: O(1) index access, amortised O(1) append, O(n) middle insert. `LinkedList` is a doubly linked list with O(n) index access and per-node overhead. `ArrayList` wins in nearly all practical cases because of memory locality, and `ArrayDeque` is the better queue.

**8.** Checked exceptions must be caught or declared, and are intended for recoverable conditions. Unchecked exceptions (`RuntimeException` and subclasses) are not enforced by the compiler and signal programming errors. Modern frameworks mostly use unchecked exceptions.

**9.** It closes every `AutoCloseable` declared in the header automatically, in reverse order, even on exceptions. If both the body and `close` throw, the close exception is attached as suppressed.

**10.** `HashMap` is not thread-safe and allows one null key. `Hashtable` is a legacy fully synchronised map with coarse locking. `ConcurrentHashMap` is the scalable concurrent map with atomic compound methods, and does not allow nulls.

**11.** A functional interface has exactly one abstract method (`Comparator`, `Runnable`, `Function`). A lambda is a concise implementation of one, for example `(a, b) -> a.age() - b.age()`.

**12.** `final` variable: assigned once (the object it refers to can still change). `final` method: cannot be overridden. `final` class: cannot be subclassed.

**13.** Equal objects must have equal hash codes, and `equals` must be reflexive, symmetric, transitive, consistent and false for `null`. Overriding only `equals` means equal objects land in different buckets by the identity hash, so `HashMap` and `HashSet` fail to find them and allow duplicates.

**14.** An array of buckets. The key's hash selects a bucket, collisions are chained and converted to trees when long, and keys are compared with `equals`. It doubles when size exceeds capacity times load factor (default 16 and 0.75), redistributing entries, which is O(n). Presize when you know the size.

**15.** Generics are checked at compile time then erased, so `List<String>` and `List<Integer>` are the same class at runtime. You cannot instantiate `T`, create generic arrays or test `instanceof` against a parameterised type, and runtime libraries need type tokens.

**16.** `volatile` gives visibility and ordering for reads and writes of that variable (a happens-before edge between a write and later reads). It does not make compound actions such as `count++` atomic. Use atomics or locks for those.

**17.** `synchronized` provides mutual exclusion on an object's monitor and a happens-before edge from an unlock to the next lock of the same monitor, so changes made inside are visible to the next thread that enters. It is reentrant.

**18.** It starts threads up to the core size, then queues tasks. Only when the queue is full does it add threads up to the maximum. If both are saturated it applies the rejection policy. An unbounded queue means it never grows past core size and hides overload until memory runs out.

**19.** `CompletableFuture` composes asynchronous steps (`thenApply`, `thenCompose`, `thenCombine`, `allOf`) with error handling and timeouts. Without an explicit executor it uses the shared common pool, so blocking calls can starve other work. Provide a dedicated, bounded executor.

**20.** Loading N parents then lazily loading a child association for each produces 1+N queries. Fix with fetch joins, entity graphs, batch fetching or DTO projections, detect it with SQL logging or query counting, and do not blanket-switch to eager.

**21.** Spring wraps the bean in a proxy that begins a transaction before the method and commits or rolls back after. It does not apply to self-invocation (bypasses the proxy), rolls back only on unchecked exceptions by default, and ties up a connection for its duration.

**22.** It keeps open connections for reuse, avoiding the cost of creating one per request, and caps concurrent database use. Size it from the database's capacity and measured workload, not as large as possible, keep the total across instances under the database limit, set a connection timeout and a `maxLifetime` below network and database limits, and monitor active, idle and pending counts.

**23.** Constructor injection makes dependencies required and final, enables testing without the container and exposes circular dependencies and too many collaborators. Field injection hides dependencies and requires reflection or the container to test.

**24.** Use `Optional` as a return type when absence is a normal outcome. Do not use it for fields, parameters or collections, and avoid `get()` without a check.

**25.** Young generation (Eden and survivors) holds new objects, and the old generation holds long-lived ones. Most objects die young, so a minor GC only copies the few survivors in a small region, which is quick. Full collections scan much more and cause long pauses.

**26.** Mock boundaries: network clients, clocks, slow or non-deterministic dependencies. Do not mock value objects, simple collaborators or the class under test. Prefer fakes when behaviour matters and verify outcomes more than interactions.

**27.** Take a thread dump with `jstack` or `jcmd Thread.print`. It reports "Found one Java-level deadlock" with the lock cycle. Fix with a consistent lock ordering, fewer locks, timeouts with `tryLock`, or by not calling foreign code under locks.

**28.** `orElse(x)` evaluates `x` eagerly every time. `orElseGet(supplier)` calls the supplier only if the optional is empty, so use it for costly or side-effecting fallbacks.

**29.** The JMM specifies when one thread's writes are visible to another through happens-before edges: monitor unlock to lock, volatile write to read, thread start and join, safe publication through final fields and concurrent collections. Without an edge there is no visibility or ordering guarantee, because compilers, the JIT and CPUs reorder.

**30.** Check the GC log and metrics: does post-GC heap climb steadily? Compare class histograms over time, capture a heap dump (or enable dump on OOM) and use MAT to find the dominator and GC-root path. Typical culprits: unbounded caches, static collections, ThreadLocals, listeners, unbounded queues. Also check thread count and native and metaspace memory. Fix the retention, bound the structure, add monitoring on post-GC heap.

**31.** Start with the JDK's default (G1) and measure. For tight pause targets and larger heaps, evaluate ZGC. Set heap relative to the container limit, leaving room for metaspace, thread stacks and native memory, set `-Xms` equal to `-Xmx`, enable GC logging, and verify under realistic load that pauses meet the objective. Reduce allocation before tuning flags.

**32.** Virtual threads are cheap JVM-scheduled threads that unmount from carrier threads when blocked, allowing thread-per-request code to scale with many concurrent blocking operations. They do not speed CPU-bound work, do not raise downstream limits (database pool, rate limits), should not be pooled, and certain operations can pin the carrier. Pinning on `synchronized` was removed in Java 24 (JEP 491), but native calls can still pin.

**33.** Look for threads parked in the pool's `getConnection` in a thread dump, and Hikari metrics showing active equal to maximum with pending threads. Then find who holds connections: leaked connections (use leak detection), long transactions with remote calls inside, or slow queries. Fix the holder first, resize only with evidence, and keep timeouts short to shed load.

**34.** A dedicated bounded executor or semaphore (bulkhead), connect and read timeouts, an overall deadline, limited retries with backoff and jitter only for idempotent calls, a circuit breaker (for example Resilience4j), a fallback or degraded response, and metrics on queue depth, latency and rejections.

**35.** Keep beans stateless, make shared state immutable or use concurrent structures, avoid mutable fields in singletons, clear `ThreadLocal`s, keep transactions short, bound pools and queues, and set timeouts. Verify with load tests and review any static or shared caches.

**36.** A transaction holds a connection and locks for its duration. Remote calls inside it extend that time, exhaust the pool, increase lock contention and deadlocks, and their side effects cannot be rolled back. Keep transactions short, do external I/O outside, and use patterns such as the transactional outbox for reliable messaging.

**37.** Compare workload (CPU versus I/O bound), latency and throughput targets, startup constraints, domain complexity, team skills, ecosystem needs and operational maturity. Java for complex, CPU-heavy or long-lived high-throughput systems and strong typing at scale. Node for I/O-bound, fast-iteration services and TypeScript-centred teams. Validate with a load test of the real workload.

**38.** Read release notes for removed or changed flags and behaviour, run the full test suite and load tests on the new JDK in staging, compare GC logs and latency percentiles with the old baseline, roll out gradually (canary) with rollback ready, and watch heap, pause, error and latency metrics.

**39.** Identify the process and hot thread (`top -H`, convert thread ID to hex, match `nid` in a thread dump), or record a JFR or async-profiler CPU profile. Check whether GC threads are the cause. Typical causes are hot loops, regex backtracking, serialisation, excessive logging and allocation churn. Fix, then verify with a profile and compare.

**40.** Handle `SIGTERM` (the JVM runs shutdown hooks), stop accepting new requests while readiness fails so the load balancer drains traffic, let in-flight requests finish within a deadline, stop consumers and schedulers, shut down executors and close pools, and set the orchestrator's termination grace period longer than the drain time. Spring Boot enables graceful shutdown by default for embedded Tomcat, Jetty and Reactor Netty (`server.shutdown`, with the grace period set by `spring.lifecycle.timeout-per-shutdown-phase`, per the Boot 4.1 reference).

---

## 20. References

Primary sources, preferred over blog posts. Links marked "checked" were opened on 2026-11-06. Others were not re-checked.

- Java Language Specification and Java Virtual Machine Specification: https://docs.oracle.com/javase/specs/
- JLS chapter 17, Threads and Locks (Java Memory Model, happens-before in 17.4.5), checked: https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html
- OpenJDK JEP index (each feature and its release), checked: https://openjdk.org/jeps/0
- JEP 395 Records, JEP 409 Sealed Classes, JEP 441 Pattern Matching for switch: listed in the JEP index above
- Java SE 21 API documentation (checked for `HashMap`, `Stream`, `Executors`, `Thread`): https://docs.oracle.com/en/java/javase/21/docs/api/index.html
- `jcmd` reference, JDK 21, checked: https://docs.oracle.com/en/java/javase/21/docs/specs/man/jcmd.html
- `java` command reference (flags, `-Xlog`), JDK 21, checked: https://docs.oracle.com/en/java/javase/21/docs/specs/man/java.html
- Java SE 21 Troubleshooting Guide, checked: https://docs.oracle.com/en/java/javase/21/troubleshoot/
- Java Flight Recorder API Programmer's Guide, Release 21, checked: https://docs.oracle.com/en/java/javase/21/jfapi/
- JDK Mission Control documentation (not re-checked): https://docs.oracle.com/en/java/java-components/
- HotSpot Garbage Collection Tuning Guide, JDK 21, checked: https://docs.oracle.com/en/java/javase/21/gctuning/introduction-garbage-collection-tuning.html
- JEP 444 Virtual Threads (final in 21), checked: https://openjdk.org/jeps/444
- JEP 491 Synchronize Virtual Threads without Pinning (24), checked: https://openjdk.org/jeps/491
- JEP 506 Scoped Values (final in 25), checked: https://openjdk.org/jeps/506
- JEP 490 ZGC: Remove the Non-Generational Mode (24), checked: https://openjdk.org/jeps/490
- Spring Boot reference documentation, checked: https://docs.spring.io/spring-boot/reference/
- Spring Boot graceful shutdown, checked: https://docs.spring.io/spring-boot/reference/web/graceful-shutdown.html
- Spring Boot testing (`@MockitoBean`), checked: https://docs.spring.io/spring-boot/reference/testing/spring-boot-applications.html
- Spring Framework `@Transactional` reference, checked: https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html
- Spring Framework `@MockitoBean`, checked: https://docs.spring.io/spring-framework/reference/testing/annotations/integration-spring/annotation-mockitobean.html
- Spring Framework reference (DI, transactions, web), not re-checked: https://docs.spring.io/spring-framework/reference/
- Spring Data JPA reference (not re-checked): https://docs.spring.io/spring-data/jpa/reference/
- Hibernate ORM documentation (not re-checked): https://hibernate.org/orm/documentation/
- HikariCP "About Pool Sizing", checked: https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing
- HikariCP README: https://github.com/brettwooldridge/HikariCP
- Apache Maven documentation: https://maven.apache.org/guides/
- Gradle user manual: https://docs.gradle.org/current/userguide/userguide.html
- JUnit 5 user guide: https://junit.org/junit5/docs/current/user-guide/
- Mockito documentation: https://site.mockito.org/
- Testcontainers documentation: https://testcontainers.com/
- JMH (Java Microbenchmark Harness), OpenJDK code tools: https://github.com/openjdk/jmh
- Eclipse Memory Analyzer (MAT): https://eclipse.dev/mat/
- Joshua Bloch, *Effective Java* (book)
- Brian Goetz et al., *Java Concurrency in Practice* (book)
- Brendan Gregg, *Systems Performance* (book), for the OS-side method

Verify any URL before depending on it, since pages move.

---

## 21. Glossary

**In plain words:** This table explains the hardest terms in the guide with everyday words.

| Term | Plain meaning |
|---|---|
| Bytecode | Compact instructions that the JVM runs. `javac` produces them from source code. |
| JIT compiler | Part of the JVM that turns often-used bytecode into fast machine code while the program runs. |
| Warm-up | The short time after start when the JVM is still optimising, so code runs slower. |
| Heap | Memory area where all objects live. The garbage collector manages it. |
| Metaspace | Native memory where the JVM keeps information about loaded classes. |
| Garbage collector (GC) | JVM part that finds and frees objects the program can no longer reach. |
| Memory leak | Objects that are still reachable but no longer needed, so they are never freed. |
| Type erasure | The compiler removes generic type arguments, so they do not exist at run time. |
| Boxing | Automatic conversion between a primitive such as `int` and its object type `Integer`. |
| Hash code | A number computed from an object, used to choose its place in a hash table. |
| Immutable | Cannot change after it is created. |
| Race condition | A bug where the result depends on the timing of threads. |
| Happens-before | A rule in the Java Memory Model that guarantees one action's effects are visible to another. |
| Monitor | The lock that `synchronized` uses on an object. |
| `volatile` | A keyword that makes a variable's writes visible to other threads, without making `count++` atomic. |
| Deadlock | Threads wait for each other forever because each holds a lock the other needs. |
| CAS (compare-and-swap) | A CPU step that changes a value only if it still has the expected old value. |
| Executor | An object that runs tasks on a managed pool of threads. |
| Virtual thread | A very light thread scheduled by the JVM instead of the operating system. |
| Pinning | A virtual thread stuck to its carrier thread, so the carrier cannot run other work. |
| Dependency injection | A framework creates the objects a class needs and passes them in. |
| Bean | An object created and managed by the Spring container. |
| Proxy | A stand-in object that adds behaviour, such as starting a transaction, around a method call. |
| Transaction | A group of database changes that all succeed or all roll back. |
| Connection pool | A set of open database connections that requests borrow and return. |
| N+1 problem | One query for a list plus one extra query per item, which makes a request slow. |
| Mock | A fake object used in a test to replace a real dependency. |
| Thread dump | A snapshot of what every thread is doing, used to find stuck or blocked code. |
| JFR (Java Flight Recorder) | A low-overhead recorder built into the JVM for production diagnosis. |
