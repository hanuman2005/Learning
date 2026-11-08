# Engineering Craft 05 — Clean Code I: Naming, Functions, Errors

> **Series:** Engineering Craft
> **Goal:** Write code the next human reads without a decoder ring — intention-revealing names, small single-purpose functions, honest error handling, and comments that earn their place.

---

## Table of Contents

1. [Why Clean Code Is an Economic Argument](#1-why-clean-code-is-an-economic-argument)
2. [Intention-Revealing Names](#2-intention-revealing-names)
3. [Small Functions & One Level of Abstraction](#3-small-functions--one-level-of-abstraction)
4. [Function Arguments](#4-function-arguments)
5. [Side Effects & Command-Query Separation](#5-side-effects--command-query-separation)
6. [Comments: When They Help vs When They Smell](#6-comments-when-they-help-vs-when-they-smell)
7. [Error Handling: Exceptions vs Codes, Fail Fast](#7-error-handling-exceptions-vs-codes-fail-fast)
8. [Practice: Clean a Messy Method](#8-practice-clean-a-messy-method)
9. [Self-Check](#9-self-check)
10. [Next](#10-next)

---

## 1. Why Clean Code Is an Economic Argument

**In plain words:** Clean code isn't about being tidy for its own sake — it's about money and time. Code is read far more than written (**Craft 01**), and every hour a teammate spends decoding your cleverness is an hour not shipping features. Clean code is you paying a small cost now so the team pays a much smaller cost, forever.

### Why

The cost of software is dominated by *maintenance*, not initial writing. Code that's hard to read is hard to change safely, so changes get slower and buggier over time — the "you touch it, you break it" tax. Clean code keeps the *cost of change* low as the system ages. It's not aesthetics; it's the difference between a codebase that stays fast to work in and one that grinds to a halt.

### What

Code whose *intent* is obvious to the next reader: names that say what they mean, functions that do one thing, errors handled honestly, comments that explain *why* not *what*.

### When & Where

Continuously, in the small — every name you type, every function you extract. And especially in review (**Craft 08**), where clean code is a shared standard. This builds directly on the **SOLID** and OOP boundaries from **System Design Day 2**; clean code is those principles at the line-and-function level.

> **🤔 Wait — the code works and passes tests. Isn't "clean" just senior engineers imposing taste?**
> "Works" is necessary, not sufficient. Working-but-unreadable code is a *liability that hasn't been billed yet*: it works until someone must change it — and then they misread the tangle, introduce a bug, and the "working" code causes an outage. Readability is a *functional* requirement of any code that will be maintained, because maintainability is a requirement. There's genuine taste at the margins (tabs, line length), but "can the next person understand and safely change this?" is objective and measurable — it shows up as bug rates and delivery speed. Clean code is engineering economics, not decoration.

**✅ Good scenario (where it saves you):** A new hire needs to add a fee to checkout. The pricing code reads like prose — `applyPlatformFee(order)`, `isEligibleForFreeShipping(cart)` — so they find the right spot in minutes and ship correctly on day three. The code taught them the domain.

**❌ Bad scenario (the pain without it):** The same feature lands in a 300-line `process()` method full of `d`, `tmp`, `flag2`, and nested conditionals. The new hire guesses wrong, adds the fee in the wrong branch, and ships a bug that overcharges a segment of customers. The code hid the domain and cost a hotfix.

**When NOT to gold-plate:** A genuine throwaway spike or a 20-line script you'll delete next week doesn't need extraction ceremony. But be honest — most "throwaway" code survives. Clean is the default; scruffy is the justified exception.

---

## 2. Intention-Revealing Names

**In plain words:** A name is a tiny piece of documentation that can never go out of date (unlike a comment). A good name answers "what is this and why does it exist?" so the reader never has to reverse-engineer it from the code below.

```java
// ❌ BEFORE — names hide meaning; you must read the body to understand the head.
public List<int[]> getThem() {
    List<int[]> list1 = new ArrayList<>();
    for (int[] x : theList)
        if (x[0] == 4)              // what is 4? what is index 0?
            list1.add(x);
    return list1;
}

// ✅ AFTER — names ARE the documentation; the loop reads like a sentence.
public List<Cell> getFlaggedCells() {
    List<Cell> flaggedCells = new ArrayList<>();
    for (Cell cell : gameBoard)
        if (cell.isFlagged())        // no magic number; the domain concept is named
            flaggedCells.add(cell);
    return flaggedCells;
}
```

Rules that pay off:
- **Reveal intent.** `elapsedTimeInDays` beats `d`. `isEligibleForRefund` beats `check`.
- **Avoid disinformation.** Don't call it `accountList` if it's a `Set`; don't name a thing `xData` when it holds names.
- **Make distinctions meaningful.** `a1`, `a2` tell the reader nothing; `source`/`destination` do.
- **Use pronounceable, searchable names.** You can grep `MAX_RETRIES` (from **Craft 01**); you can't reliably grep `7` or `mr`.
- **Class = noun, method = verb.** `Order`, `PaymentGateway`; `place()`, `charge()`, `isExpired()`.
- **Booleans read as questions:** `isActive`, `hasCoupon`, `canRefund`.
- **No encodings.** Drop Hungarian prefixes (`strName`, `m_count`) — the type system already knows the type.

> **🤔 Wait — long descriptive names make lines huge and ugly. Isn't `numberOfRegisteredUsersWhoConfirmedEmail` worse than a comment plus `n`?**
> A name that's long *because it's precise* is a feature, not a bug — it's read far more than typed, and your IDE autocompletes it anyway. But if a name needs to be *that* long, it's often a hint the concept deserves its own **type** or method: `confirmedUsers.size()` where `confirmedUsers` is a well-named collection reads better than any mega-name. So: prefer precise over short, but let an over-long name push you toward extracting a concept. The comparison isn't "long name vs `n`+comment" — it's "long name vs a named abstraction," and one of those wins.

**When NOT to:** Loop counters in a three-line loop can stay `i`; a lambda's single obvious parameter can be `x`. Convention-bound short names in tiny scopes are fine — the rule is "name proportional to scope and ambiguity."

---

## 3. Small Functions & One Level of Abstraction

**In plain words:** A function should do *one thing* and read at *one altitude*. Mixing high-level policy ("charge the customer") with low-level mechanics ("open a socket, parse bytes") in the same function is like a recipe that says "bake at 180°C" and then, mid-sentence, explains the chemistry of gluten. Pick an altitude and stay there; push the details down into named helpers.

```java
// ❌ BEFORE — one giant function, many altitudes, does five things.
public void checkout(Cart cart, User user) {
    // validation altitude
    if (cart == null || cart.items().isEmpty()) throw new IllegalStateException("empty");
    // pricing altitude — low-level arithmetic mixed with policy
    BigDecimal total = BigDecimal.ZERO;
    for (Item i : cart.items()) total = total.add(i.price().multiply(BigDecimal.valueOf(i.qty())));
    if (user.tier() == Tier.GOLD) total = total.multiply(new BigDecimal("0.9"));
    total = total.multiply(new BigDecimal("1.08"));   // tax — magic number
    // persistence altitude
    Order order = new Order(user.id(), cart.items(), total);
    em.persist(order);
    // notification altitude
    emailClient.send(user.email(), "Order confirmed", "Total: " + total);
}

// ✅ AFTER — checkout() reads as a table of contents; each step is one named thing.
public Order checkout(Cart cart, User user) {
    validateNotEmpty(cart);                       // one altitude: policy steps
    Money total = priceCart(cart, user);
    Order order = persistOrder(cart, user, total);
    sendConfirmation(user, order);
    return order;
}

private Money priceCart(Cart cart, User user) {   // details live one level DOWN
    Money subtotal = cart.subtotal();
    Money discounted = applyTierDiscount(subtotal, user.tier());
    return applyTax(discounted);
}
```

- **One thing:** if you can extract a meaningfully-named sub-function, the original was doing more than one thing. (Ask: does its name fully describe it without "and"?)
- **One level of abstraction:** the top function names *steps*; each step's *how* lives one level down. The reader zooms in only where they need to.
- **Small:** "small" is a symptom of "one thing," not a line-count law — but if a function scrolls off the screen, be suspicious.

> **🤔 Wait — won't breaking one method into ten tiny ones just scatter the logic and make me jump around to follow it?**
> It's the opposite when done right, because of the **step-down rule**: a well-decomposed function reads top-to-bottom like an outline — you read `checkout()` and understand the *whole flow* without opening a single helper. You descend into `priceCart()` *only if* you care about pricing details. Fragmentation hurts when the pieces are badly named or arbitrarily cut ("extract for extraction's sake"); it *helps* when each piece is a named concept at a consistent altitude. The cure for "too much jumping" is good names, not giant functions. A 300-line method forces you to hold *everything* in your head at once — that's the real scatter.

**When NOT to:** Don't extract a one-line helper used once with a name no clearer than the line itself (`private int addOne(int x){return x+1;}`) — that's ceremony. Extraction earns its keep when it *names a concept*, is *reused*, or *lowers the altitude* of its caller.

---

## 4. Function Arguments

**In plain words:** Every argument is a thing the caller must understand and get right. Fewer is friendlier. And a boolean flag argument is a quiet confession that your function does two things.

```java
// ❌ BEFORE — too many args; order is easy to swap; a boolean flag hides two behaviors.
public void createUser(String name, String email, boolean isAdmin,
                       boolean sendWelcome, boolean isTrial, String region) { ... }
// call site is unreadable: createUser("A", "a@x.com", false, true, true, "EU")  // which flag is which??

// ✅ AFTER — a parameter object names the fields; flags become intent-revealing methods.
public User register(RegistrationRequest request) { ... }   // one cohesive argument

// Two behaviors become two honest methods instead of a boolean flag:
public User registerAdmin(RegistrationRequest request) { ... }
public User registerStandardUser(RegistrationRequest request) { ... }
```

- **0–1 args ideal, 2 fine, 3 a smell, 4+ almost always wrong.** Many args → bundle related ones into a **parameter object** (a `record`), which also names the group (see *data clumps* in **Craft 06**).
- **No boolean flag arguments.** `render(true)` — true *what*? A flag means the function does two things; split it into two named functions.
- **No output arguments** (a param the function mutates to "return" a value). Return a value instead; it's what readers expect.
- **Argument order shouldn't require memory.** `copy(source, destination)` is guessable; `copy(a, b)` is a coin flip — a parameter object or clear names fix it.

**When NOT to:** A well-known builder or a `record` DTO with many fields is fine — the fields are *named at the call site*. The rule targets *positional* argument overload, not richly-named data.

---

## 5. Side Effects & Command-Query Separation

**In plain words:** A function should either *answer a question* (a query, returns data, changes nothing) or *do something* (a command, changes state, returns nothing) — not both secretly. A function named like a question that also mutates state is a landmine: the caller reads it as harmless and gets surprised.

```java
// ❌ BEFORE — looks like a harmless getter, but SECRETLY mutates and even logs in.
public boolean isValidPassword(String user, String pw) {
    User u = repo.find(user);
    if (u.password().equals(hash(pw))) {
        session.initialize(u);      // HIDDEN side effect! caller has no idea a session started
        return true;
    }
    return false;
}
// A caller writes `if (isValidPassword(...))` to CHECK, and accidentally logs the user in.

// ✅ AFTER — the query answers; the command acts. Names match reality.
public boolean isPasswordCorrect(String user, String pw) {   // query: no side effects
    return repo.find(user).matches(hash(pw));
}
public Session login(User user) {                            // command: does the thing
    return session.initialize(user);
}
```

- **Command–Query Separation (CQS):** queries return and don't mutate; commands mutate and (usually) return void or a result object. The *name* must match: `getX`/`isX`/`hasX` promise no side effects.
- **Make side effects visible.** If a method sends email, saves, or mutates global state, its name should say so (`sendConfirmation`, `save`) — no hidden surprises.
- **Prefer pure functions where you can.** A function whose output depends only on its inputs is trivially testable (no doubles needed — **Craft 03**) and can't surprise anyone.

**When NOT to:** Some idioms intentionally blend (an iterator's `next()` returns *and* advances; an atomic `getAndIncrement()`). These are fine *because they're conventional and named to signal it*. CQS is the default; break it only with a name that shouts the exception.

---

## 6. Comments: When They Help vs When They Smell

**In plain words:** The best comment is the one you didn't need because the code said it itself. Comments explain *why* — the reasoning a reader can't recover from the code. A comment that explains *what* the code does is usually a sign the code should have been clearer.

| Comment type | Verdict | Why |
|---|---|---|
| **Why / rationale** ("200ms backoff: vendor rate-limits bursts, INCIDENT-4412") | ✅ Gold | Reasoning the code can't express; saves a future incident (**Craft 01** git archaeology) |
| **Warning of consequences** ("not thread-safe — external sync required") | ✅ Good | Prevents misuse the type system can't |
| **Public API docs (Javadoc)** | ✅ Good | Contract for callers who won't read the body (**Craft 09**) |
| **TODO/FIXME with a ticket** | ✅ OK | Tracked intent; without a ticket it rots |
| **Restating the code** (`i++; // increment i`) | ❌ Noise | Adds nothing; lies when code changes |
| **Commented-out code** | ❌ Delete it | Git remembers (**Craft 01**); dead code confuses |
| **Compensating for a bad name** (`// this is actually the retry count`) | ❌ Smell | Fix the name instead |
| **Journal / changelog comments** (`// 2019-changed by Bob`) | ❌ Delete | That's what `git log` is for |

```java
// ❌ BEFORE — comment restates code, and a comment props up a bad name.
int d;   // elapsed time in days
d = today - start;
// loop over all the users
for (User u : users) { ... }

// ✅ AFTER — the name replaces the comment; a real comment explains a non-obvious WHY.
int elapsedDays = today - start;
for (User user : users) { ... }

// Retry only on 429/503: the vendor returns 500 for permanent errors we must NOT retry.
// (see INCIDENT-4412) — this is the WHY the code alone can't tell you.
if (isTransient(status)) retry(request);
```

> **🤔 Wait — I was taught "always comment your code." Now you're saying most comments are bad. Which is it?**
> The old advice was a well-meaning overcorrection from an era of terse, cryptic code. The modern refinement: **comments don't age with the code, so every comment is a future lie waiting to happen.** A comment that restates *what* the code does will drift out of sync the moment someone edits the code but not the comment — now it actively misleads. So the rule isn't "never comment"; it's "prefer expressing intent in *code* (names, types, small functions) because code can't lie, and reserve comments for the *why* that code genuinely can't hold." Fewer, higher-value comments beats a comment on every line.

**When NOT to strip comments:** Public library APIs, non-obvious algorithms, regulatory/security constraints, and hard-won "why" all deserve comments. Don't delete a comment that captures irreplaceable context just to hit a "self-documenting" ideal.

---

## 7. Error Handling: Exceptions vs Codes, Fail Fast

**In plain words:** When something goes wrong, you can either *return a special value* saying "it failed" (an error code) or *throw* and interrupt the flow (an exception). In Java, exceptions are the idiom — they keep the happy path readable and make it *impossible* to silently ignore a failure. And when input is bad, **fail fast**: blow up loudly at the boundary, not five layers deep where the cause is unrecoverable.

```java
// ❌ BEFORE — error codes tangle happy path with failure checks; easy to ignore a return.
public int transfer(Account from, Account to, int amount) {
    if (from == null) return -1;              // caller must remember -1 means "no source"
    if (amount <= 0)  return -2;              // ...and -2 means "bad amount"...
    if (from.balance() < amount) return -3;   // ...and -3 means "insufficient". Good luck.
    from.debit(amount); to.credit(amount);
    return 0;                                 // 0 means success. A caller who forgets to check
}                                             // proceeds as if it worked. Silent data loss.

// ✅ AFTER — exceptions: happy path is clean, failures are impossible to ignore, causes are named.
public void transfer(Account from, Account to, Money amount) {
    Objects.requireNonNull(from, "from");     // FAIL FAST: reject bad input at the boundary
    if (amount.isNotPositive())               // guard clauses up top (see Craft 06)
        throw new IllegalArgumentException("amount must be positive: " + amount);
    if (from.balance().isLessThan(amount))
        throw new InsufficientFundsException(from.id(), amount);   // domain-specific, informative
    from.debit(amount);
    to.credit(amount);                        // happy path reads with zero error-check noise
}
```

Principles:
- **Prefer exceptions to error codes** in Java. Codes clutter the happy path and are silently ignorable; exceptions can't be ignored and carry context.
- **Fail fast.** Validate inputs at the boundary and throw immediately with a clear message. A bug should surface *close to its cause*, not corrupt data and surface three layers away.
- **Throw specific, informative exceptions.** `InsufficientFundsException(accountId, amount)` beats `RuntimeException("error")` — the message is the first thing a debugger reads (**Craft 02**).
- **Don't swallow exceptions.** An empty `catch {}` is how bugs go silent (**Craft 02** bad scenario). Catch only what you can *handle*; otherwise let it propagate.
- **Checked vs unchecked:** modern Java leans on **unchecked** (`RuntimeException`) for programmer/unrecoverable errors to avoid `throws` clutter; reserve checked exceptions for genuinely recoverable, caller-actionable conditions. Never catch-and-ignore just to satisfy a `throws`.
- **Don't use exceptions for normal control flow.** "User not found" on a lookup is often better modeled with `Optional<User>` than a thrown exception — exceptions are for the *exceptional*.

> **🤔 Wait — languages like Go and Rust return errors as values and it's considered clean. So why are exceptions "better" in Java?**
> It's idiom, not universal truth. Go/Rust make error-values ergonomic with language support (`if err != nil`, `?` operator, exhaustive `match`) so you *can't* accidentally ignore them — that's a fine model. Java's error *codes* (magic ints) have none of that safety: nothing forces the caller to check, so failures go silent. Java's *exceptions* give you the "can't ignore it" property that Go/Rust get from their type systems. The deeper principle is shared across all of them: **failures must be impossible to silently ignore, and must carry enough context to diagnose.** In Java, exceptions achieve that; ad-hoc int codes don't. (For *expected* absence, Java's `Optional` plays the role of a lightweight result type.)

**✅ Good scenario:** A malformed request hits the API. Fail-fast validation throws `IllegalArgumentException("amount must be positive")` at the controller boundary; the client gets a clear 400, and no bad data ever reaches the pricing engine. The error message *is* the fix.

**❌ Bad scenario:** A service ignores a `-3` return code from `transfer()` (someone forgot to check it), proceeds to mark the order paid, and ships goods for money that never moved. Because the failure was a return value nobody was forced to inspect, it silently corrupted business state.

**When NOT to:** Don't wrap-and-rethrow at every layer (log-and-rethrow storms bury the real trace). Catch at the boundary where you can *do* something (return an HTTP status, retry, fall back) and let it propagate cleanly in between.

---

## 8. Practice: Clean a Messy Method

Take a real 60–150 line "god method" (from your codebase, or write a deliberately bad one — a `processOrder` that validates, prices, taxes, persists, emails, and logs, full of `tmp`, `flag`, magic numbers, a boolean flag argument, an empty `catch`, and comments restating code). Then, **keeping behavior identical** (this is the golden rule you'll formalize in **Craft 07**):

1. **Pin behavior with tests first** (**Craft 03**). You cannot safely clean untested code — write characterization tests that capture what it does *now*.
2. **Rename** every unclear identifier to reveal intent (§2). Replace magic numbers with named constants.
3. **Extract** each altitude into a well-named private method so the top method reads as an outline (§3). Aim for one level of abstraction per function.
4. **Fix arguments:** replace the boolean flag with two methods; bundle any data clump into a `record` parameter object (§4).
5. **Separate command from query** where they're tangled (§5).
6. **Fix error handling:** remove the empty `catch`, add fail-fast guard clauses, throw a specific exception with context (§7).
7. **Strip noise comments; keep/add one real "why" comment** (§6).
8. **Run the tests after every step.** Green throughout = behavior preserved. If a test goes red, you changed behavior — undo and try smaller.

**Deliverable:** a before/after diff plus the test suite that stayed green. Success = a reviewer understands the "after" in under a minute and the diff changed *only* readability, not behavior.

---

## 9. Self-Check

1. Frame clean code as an economic argument in one sentence. Why is "it works and passes tests" insufficient?
2. Give the six naming rules. Rewrite `int d;` and `getThem()` to reveal intent.
3. What do "one thing" and "one level of abstraction" mean? What is the step-down rule, and how does it answer "too much jumping around"?
4. Why is a boolean flag argument a smell? What do you do instead? What's the ideal argument count?
5. What is Command–Query Separation, and what's the danger of a query with a hidden side effect?
6. Give three comment types that add value and three that are smells. Why is "every comment is a future lie"?
7. Why prefer exceptions over error codes in Java? What property do Go/Rust get from their type systems instead?
8. What does "fail fast" mean and where do you do it? Why is an empty `catch` block dangerous?
9. When is a comment genuinely worth keeping, and when should extraction stop (not over-extract)?
10. Why must you pin behavior with tests *before* cleaning a messy method?

---

## 10. Next

**Clean Code II — Smells & SOLID in Practice.** Now that you can write clean lines and functions, learn to *smell* the bad ones at a glance: a catalog of code smells (long method, feature envy, primitive obsession, data clumps, shotgun surgery) each with its fix, and SOLID (from **System Design Day 2**) applied at the code level with guard clauses and clean boundaries.
