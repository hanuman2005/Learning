# TypeScript for Backend Engineers

A working guide to TypeScript for people who build Node.js services. It teaches how the type system thinks, so you can model real data safely, not just copy syntax.

**Audience:** developers who know JavaScript and want production-level TypeScript for backend work.
**Prerequisites:** basic JavaScript (functions, objects, promises) and Node.js.
**Scope:** TypeScript 5.x on Node.js. Examples use a neutral order and event-processing domain. Front-end frameworks are out of scope.
**Accuracy status:** Reviewed against official documentation on 2026-11-06; items still unverified are marked.

## How to use this guide

Each chapter has five parts:

1. **In plain words**: a short, simple summary.
2. **Understand**: the concepts and the reasoning behind them.
3. **Tools to master**: what to learn, and the level to reach.
4. **Common mistakes**: what goes wrong in practice.
5. **Self-check**: questions you should be able to answer without notes.

Chapter 17 collects common gotchas. Chapter 18 lists hands-on labs. Chapter 19 collects interview questions with model answers. Chapter 20 is a glossary of hard terms. Chapter 21 lists references.

**What "good enough" means for a backend role:** you can model data with unions, validate untrusted input at the edge, keep `any` out of your code, and configure a strict project that runs in Node. Chapters 2, 4, 5, 6, 10 and 11 matter most.

When a hard word appears for the first time, its meaning is in brackets. Chapter 19 repeats them.

## Contents

1. The mental model
2. tsconfig and strict options
3. Types versus interfaces
4. Unions and narrowing
5. Discriminated unions and exhaustive checks
6. unknown versus any
7. Generics
8. Utility types
9. Async typing
10. Runtime validation with zod
11. The Result pattern
12. Branded types
13. The satisfies operator
14. Declaration files and @types
15. Modules: ESM and NodeNext
16. Tooling and running TypeScript in Node
17. Common gotchas
18. Practice labs
19. Interview questions
20. Glossary
21. References

---

## 1. The mental model

**In plain words.** TypeScript is JavaScript plus a checker. The checker reads your code before it runs and finds mistakes. When the program runs, the types are gone.

### Understand

**Types exist only at compile time.** The compiler (`tsc`) checks types, then removes them. The output is plain JavaScript. Nothing in the type system exists at runtime. This is called **type erasure** [removing type information when code is compiled].

Consequence: a type says what you *believe* about a value. It does not prove it. Data from a network, a file or a database is only as safe as your check at the point where it enters.

**Structural typing.** TypeScript compares the *shape* of values, not their names. If two types have the same members, they are compatible.

```ts
type OrderId = { id: string };
type CustomerId = { id: string };

const a: OrderId = { id: "o-1" };
const b: CustomerId = a; // allowed: same shape
```

This is why two IDs can be mixed by mistake. Chapter 12 shows how to prevent it.

**Inference.** You do not need to write every type. The compiler infers types from values. Annotate function parameters, exported function return types and public data shapes. Let it infer local variables.

**Two worlds: values and types.** An `interface` or `type` exists only in the type world. A `class` or `enum` exists in both. `typeof x` in a value position gives a string; in a type position it gives the type of `x`.

**The compiler is not the runtime.** `tsc` checks and compiles. Node (or another runtime) runs the output. Tools such as `tsx` run TypeScript without checking it (chapter 16).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Checking | `tsc --noEmit` | Run a full type check without writing files |
| Seeing inferred types | Editor hover, `tsc --declaration` | Read inferred types and know why they appear |
| Playground | typescriptlang.org/play | Reproduce a type problem in a small snippet |
| Type position vs value position | `typeof`, `keyof`, `ReturnType<typeof f>` | Derive types from values |

### Common mistakes

- Thinking a type annotation validates data at runtime. It does not.
- Annotating everything. Over-annotation hides useful inference and adds noise.
- Using `as` to silence an error. A cast [telling the compiler to trust you] can hide a real bug.

### Self-check

- What exists at runtime after `tsc` runs: the types, or only JavaScript?
- Why can an `OrderId` object be assigned to a `CustomerId` variable?
- When should you write a type annotation, and when should you let inference work?

---

## 2. tsconfig and strict options

**In plain words.** `tsconfig.json` is the settings file for the compiler. The `strict` setting turns on the checks that catch most bugs. Start every project with it on.

### Understand

**`strict` is a bundle.** Setting `"strict": true` turns on these options (per the official TSConfig reference): `alwaysStrict`, `strictNullChecks`, `strictBindCallApply`, `strictBuiltinIteratorReturn`, `strictFunctionTypes`, `strictPropertyInitialization`, `noImplicitAny`, `noImplicitThis` and `useUnknownInCatchVariables`. New strict checks can be added in later TypeScript versions.

The most important ones:

- **`strictNullChecks`**: `null` and `undefined` are separate types. A `string` cannot be `null`. Without it, null errors hide everywhere.
- **`noImplicitAny`**: a value with no type and no inference is an error, not a silent `any`.
- **`useUnknownInCatchVariables`**: `catch (e)` gives `e` the type `unknown`, so you must check it.

**Extra checks that are NOT part of `strict`.** Turn these on yourself:

| Option | Effect |
|---|---|
| `noUncheckedIndexedAccess` | Reading `arr[i]` or `obj[key]` from an index signature gives `T \| undefined` |
| `exactOptionalPropertyTypes` | `?: T` means "may be missing", not "may be set to `undefined`" |
| `noImplicitOverride` | A class method that overrides a parent method must use the `override` keyword |
| `noFallthroughCasesInSwitch` | A `case` that falls into the next one is an error |

**A solid backend `tsconfig.json`** (a starting point; adjust to your project):

```json
{
  "compilerOptions": {
    "target": "es2022",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "rootDir": "src",
    "sourceMap": true
  },
  "include": ["src"]
}
```

Notes:

- `module: "nodenext"` makes TypeScript follow Node's rules for ESM and CommonJS. It implies a modern `target` (chapter 15).
- `verbatimModuleSyntax` forces `import type` for type-only imports, so the emitted code matches what you wrote.
- `isolatedModules` makes sure each file can be compiled alone. Fast tools depend on this.
- `skipLibCheck` skips type checking of `.d.ts` files in `node_modules`. It speeds up builds and hides some third-party type errors. Most teams accept this trade-off.
- `rootDir`, `outDir`, `sourceMap` and `include` shape the build output.

`noUncheckedIndexedAccess` example:

```ts
const sizes: Record<string, number> = { small: 1 };
const n = sizes["large"]; // number | undefined with the flag
console.log(n.toFixed()); // error: n may be undefined
```

`exactOptionalPropertyTypes` example:

```ts
interface Filter { status?: "open" | "closed" }
const f: Filter = {};
f.status = undefined; // error with the flag on
```

**Project references and `extends`.** A large repo can share a base file with `"extends": "./tsconfig.base.json"`. Project references (`composite`, `references`) split a monorepo into separately built parts. Learn these when you need them.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Show final config | `tsc --showConfig` | See merged settings after `extends` |
| Type check only | `tsc --noEmit` | Use in CI |
| Explain why a file is included | `tsc --explainFiles` | Debug wrong `include` patterns |
| Watch mode | `tsc --watch` | Fast feedback during development |
| Starter bases | `@tsconfig/node22` style packages | Reuse a maintained base (check the package name and version) |

### Common mistakes

- Starting with `strict: false` and "turning it on later". The later cost is high. Start strict.
- Assuming `strict` includes `noUncheckedIndexedAccess`. It does not.
- Putting test files in the build `include` so they ship in `dist`.
- Setting `module: "commonjs"` with `moduleResolution: "node"` and then being confused by ESM-only packages.

### Self-check

- Name three options that `strict` turns on.
- What does `noUncheckedIndexedAccess` change about `arr[0]`?
- Why use `verbatimModuleSyntax`?
- What is the downside of `skipLibCheck`?

---

## 3. Types versus interfaces

**In plain words.** Both `type` and `interface` describe the shape of an object. They are very close. Pick one rule for your team and stay consistent.

### Understand

Both can describe objects, and both can be extended.

```ts
interface Order {
  id: string;
  total: number;
}

interface PaidOrder extends Order {
  paidAt: Date;
}

type Shipment = {
  id: string;
  carrier: string;
};

type TrackedShipment = Shipment & { trackingCode: string }; // intersection
```

Real differences (from the official Handbook):

| Feature | `interface` | `type` alias |
|---|---|---|
| Reopen and add members (declaration merging) | Yes | No (duplicate identifier error) |
| Extend | `extends` | Intersection `&` |
| Name a union, tuple or primitive | No | Yes |
| Mapped and conditional types | No | Yes |
| Implemented by a class | Yes | Yes, if it is an object shape |

**Declaration merging** [two declarations with one name join into one type] is how libraries let you add fields. Example: adding a field to Express's `Request` type.

```ts
type OrderStatus = "open" | "paid" | "cancelled"; // only a type can do this
type OrderRow = [id: string, total: number];       // tuple
```

**A practical rule.** Handbook advice: use `interface` until you need a feature only `type` gives. Many teams choose the opposite: `type` everywhere, `interface` only for merging. Both are valid. The rule is consistency.

**Why some teams prefer interfaces for objects.** Interface relationships are cached by the compiler, which can help in very large type graphs. Treat this as a minor point. Verify with your own measurements before acting on it (not verified here).

**`readonly` and optional members.**

```ts
interface Event {
  readonly id: string;   // cannot be reassigned (type level only)
  type: string;
  note?: string;         // may be missing
}
```

`readonly` has no runtime effect. Use `readonly T[]` or `ReadonlyArray<T>` for arrays you do not want changed.

**Enums versus string unions.** `enum` creates real runtime code. Under Node's type stripping, enums fail (chapter 16). A string union is often simpler:

```ts
type Role = "admin" | "agent" | "viewer";
```

Or an object with `as const`:

```ts
const ROLES = ["admin", "agent", "viewer"] as const;
type Role = (typeof ROLES)[number];
```

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Extending | `extends`, `&` | Compose shapes without copy-paste |
| Deriving types | `typeof`, `keyof`, indexed access `T["id"]` | Build types from one source of truth |
| `as const` | const assertions | Make literal arrays and objects precise |
| Lint rule | `@typescript-eslint/consistent-type-definitions` | Enforce one style |

### Common mistakes

- Debating `type` versus `interface` for hours. Choose, write it in the lint config, move on.
- Using `enum` and then failing under Node's type stripping.
- Believing `readonly` protects data at runtime. It does not.
- Creating two interfaces with the same name by accident and merging them without noticing.

### Self-check

- Which one supports declaration merging?
- How do you name a union type: with `type` or `interface`?
- How do you get a union of strings from an `as const` array?

---

## 4. Unions and narrowing

**In plain words.** A union means a value can be one of several types. Narrowing means you check the value, and TypeScript then knows the exact type inside that check.

### Understand

```ts
type Id = string | number;

function format(id: Id): string {
  if (typeof id === "string") {
    return id.toUpperCase(); // id is string here
  }
  return id.toFixed(0);      // id is number here
}
```

**Narrowing tools (the compiler understands these):**

| Check | Narrows by |
|---|---|
| `typeof x === "string"` | Primitive type |
| `x instanceof Date` | Class |
| `"swim" in x` | Property exists |
| `x === null`, `x == null` | Null and undefined |
| `if (x)` | Truthiness [any value that is not false, 0, "", null, undefined, NaN] |
| `x.kind === "a"` | A literal tag (chapter 5) |
| `Array.isArray(x)` | Arrays |
| Type predicate | Custom function |

**Type predicates.** A function with return type `x is T` tells the compiler what a true result means:

```ts
interface Fish { swim(): void }
interface Bird { fly(): void }

function isFish(p: Fish | Bird): p is Fish {
  return "swim" in p;
}
```

If your predicate lies, the compiler trusts it. Keep predicates simple and test them.

**Assertion functions.** `function assertString(x: unknown): asserts x is string` throws on failure and narrows after the call.

**The `in` operator and optional members.** With optional properties, both branches can still include the same type. Prefer a tag field (chapter 5) over `in` for your own data.

**Control flow.** Narrowing follows `if`, `switch`, early `return` and `throw`:

```ts
function total(price: number | undefined): number {
  if (price === undefined) throw new Error("missing price");
  return price * 2; // number here
}
```

**Optional chaining and nullish coalescing.**

```ts
const city = order.customer?.address?.city ?? "unknown";
```

`?.` stops at `null` or `undefined`. `??` replaces only `null` or `undefined`. `||` replaces any falsy value, including `0` and `""`. Use `??` for defaults.

**Non-null assertion `!`.** `x!` tells the compiler "this is not null". It adds no runtime check. Prefer a real check.

**Narrowing is lost in callbacks and after mutation.** If you narrow a `let` variable and then call a function, TypeScript may reset the narrowing. Copy the value to a `const` first.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Hover on a variable inside a branch | Editor | Confirm the narrowed type |
| Type predicates | `x is T`, `asserts x is T` | Write a safe custom guard |
| Lint | `@typescript-eslint/no-unnecessary-condition`, `no-non-null-assertion` | Catch useless checks and unsafe `!` |

### Common mistakes

- Using `||` for defaults when `0` or `""` are valid values.
- Writing a type predicate that returns `true` too easily.
- Sprinkling `!` to remove errors.
- Expecting narrowing to survive across an `await` or callback for a mutable variable.

### Self-check

- What is the difference between `??` and `||`?
- What does a type predicate tell the compiler?
- Why can narrowing disappear in a callback?

---

## 5. Discriminated unions and exhaustive checks

**In plain words.** A discriminated union is a group of object types that share one field with a fixed value, like `type`. You check that field, and TypeScript knows which object you have. You can also make the compiler warn when you forget a case.

### Understand

Each member has a **discriminant** [a shared field whose value is a literal, unique to that member]:

```ts
type OrderEvent =
  | { type: "created"; orderId: string; total: number }
  | { type: "paid"; orderId: string; paidAt: Date }
  | { type: "shipped"; orderId: string; carrier: string }
  | { type: "cancelled"; orderId: string; reason: string };

function describe(e: OrderEvent): string {
  switch (e.type) {
    case "created":
      return `Order ${e.orderId} created: ${e.total}`; // e.total exists
    case "paid":
      return `Order ${e.orderId} paid at ${e.paidAt.toISOString()}`;
    case "shipped":
      return `Order ${e.orderId} shipped by ${e.carrier}`;
    case "cancelled":
      return `Order ${e.orderId} cancelled: ${e.reason}`;
  }
}
```

**Exhaustiveness check.** Assign the leftover value to `never` in the default branch. If a case is missing, the code does not compile. The Handbook uses this pattern.

```ts
function assertNever(x: never): never {
  throw new Error(`Unexpected value: ${JSON.stringify(x)}`);
}

function handle(e: OrderEvent): void {
  switch (e.type) {
    case "created":
    case "paid":
    case "shipped":
    case "cancelled":
      return;
    default:
      return assertNever(e); // error if a new event type is added and not handled
  }
}
```

If you add `{ type: "refunded" }` to the union, every `switch` that ends with `assertNever` fails to compile. This is the main value of the pattern: the compiler lists the places you must update.

**Runtime safety.** `assertNever` also throws at runtime if bad data slips past the type system. Events from a queue can contain a type your code does not know. Throwing, or sending the event to a dead-letter queue [a place for messages that could not be processed], is better than ignoring it.

**Use `satisfies never` or a lookup map as alternatives.**

```ts
const handlers: { [K in OrderEvent["type"]]: (e: Extract<OrderEvent, { type: K }>) => void } = {
  created: (e) => console.log(e.total),
  paid: (e) => console.log(e.paidAt),
  shipped: (e) => console.log(e.carrier),
  cancelled: (e) => console.log(e.reason),
};
```

A missing key is a compile error. `Extract` picks the union member with that tag (chapter 8).

**Modelling states.** Discriminated unions make illegal states impossible:

```ts
type Payment =
  | { status: "pending" }
  | { status: "succeeded"; receiptId: string }
  | { status: "failed"; error: string };
```

With one object that has `receiptId?` and `error?` both optional, the compiler cannot stop `{ status: "pending", receiptId: "x" }`. The union can.

**Lint help.** The rule `@typescript-eslint/switch-exhaustiveness-check` reports a `switch` that does not cover every member (it needs type-aware linting).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Tag fields | `type`, `kind`, `status` literals | Model every state as a tagged member |
| `never` check | `assertNever` helper | Add to every `switch` on a union |
| Extract a member | `Extract<U, { type: "paid" }>` | Get one member's type |
| Lint | `switch-exhaustiveness-check` | Enforce coverage in CI |

### Common mistakes

- Using a `default` branch that returns a fallback. It hides new members.
- Using a `string` tag instead of literal types.
- Modelling states as one object with many optional fields.
- Trusting an event's `type` without validating it at the boundary (chapter 10).

### Self-check

- What does assigning to `never` check?
- What happens to existing `switch` statements when you add a new union member?
- Why is a union better than one object with optional fields?

---

## 6. unknown versus any

**In plain words.** `any` turns the checker off for a value. `unknown` means "I do not know the type yet", and the checker forces you to check before use. Prefer `unknown`.

### Understand

```ts
let a: any = "hello";
a.nothing.here();      // no error, crashes at runtime

let u: unknown = "hello";
// u.toUpperCase();    // error: u is of type unknown
if (typeof u === "string") {
  u.toUpperCase();     // fine after narrowing
}
```

| | `any` | `unknown` |
|---|---|---|
| Assign any value to it | Yes | Yes |
| Use it without a check | Yes (unsafe) | No |
| Assign it to other types | Yes (unsafe) | Only after narrowing |
| Spreads through code | Yes, it infects results | No |

**`any` is contagious.** A value from an `any` source gives `any` results, so one `any` can disable checking across a whole call chain.

**Where `unknown` belongs:**

- `JSON.parse` returns `any` by default. Immediately validate it (chapter 10).
- `catch (e)` is `unknown` under `strict` (via `useUnknownInCatchVariables`).
- Function parameters that accept anything, such as a logger.

```ts
try {
  await processOrder(id);
} catch (e) {
  const message = e instanceof Error ? e.message : String(e);
  logger.error({ message });
}
```

**`never`** is the empty type: no value can have it. A function that always throws returns `never`. It is the end of narrowing (chapter 5).

**`object`, `{}` and `Object`.** `{}` means any non-null value. `object` means any non-primitive. Avoid `Object`. Most code needs a real shape or `Record<string, unknown>`.

**Casts.** `x as Order` tells the compiler to trust you. Casting `unknown` to a type skips checking. Prefer validation. Use a cast only when you can prove the shape in another way.

**Double cast.** `x as unknown as T` bypasses almost every check. Treat it as a last resort and comment why.

**`@ts-expect-error` versus `@ts-ignore`.** `@ts-expect-error` fails when the error disappears, so stale suppressions get found. Prefer it.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Ban `any` | `@typescript-eslint/no-explicit-any`, `no-unsafe-*` rules | Make `any` an error in CI |
| Safe errors | `catch (e: unknown)` with `instanceof Error` | Never read `e.message` unchecked |
| Validation | zod, `safeParse` | Turn `unknown` into a typed value |

### Common mistakes

- Typing `JSON.parse` output with a cast and trusting it.
- Writing `catch (e: any)` to avoid the check.
- Using `any` to "finish later". Use `unknown` and narrow.
- Accepting `Function` or `Object` as parameter types.

### Self-check

- Why is `unknown` safer than `any`?
- What type does `e` have in `catch (e)` under `strict`?
- What is the difference between `@ts-ignore` and `@ts-expect-error`?

---

## 7. Generics

**In plain words.** A generic is a type with a blank that you fill later. It lets one function or type work for many types and still keep them safe.

### Understand

```ts
function first<T>(items: readonly T[]): T | undefined {
  return items[0];
}

const n = first([1, 2, 3]);       // number | undefined
const s = first(["a", "b"]);      // string | undefined
```

`T` is a **type parameter** [a placeholder for a type]. The compiler infers it from the call.

**Constraints with `extends`.** Limit what `T` can be so you can use its members:

```ts
interface HasId { id: string }

function indexById<T extends HasId>(items: T[]): Map<string, T> {
  return new Map(items.map((i) => [i.id, i]));
}
```

**`keyof` and indexed access.** Tie a key to an object:

```ts
function getField<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
getField({ id: "o1", total: 5 }, "total"); // number
getField({ id: "o1", total: 5 }, "nope");  // error
```

**Default type parameters.** Make a parameter optional with a default:

```ts
interface Page<T, Cursor = string> {
  items: T[];
  next?: Cursor;
}
```

**Generic types and interfaces.**

```ts
type ApiResponse<T> = { data: T; requestId: string };
type OrderResponse = ApiResponse<{ id: string; total: number }>;
```

**Generic classes.** A repository can be written once:

```ts
class Repository<T extends HasId> {
  private items = new Map<string, T>();
  save(item: T): void { this.items.set(item.id, item); }
  get(id: string): T | undefined { return this.items.get(id); }
}
```

**Conditional types.** A type that picks by a condition:

```ts
type ElementOf<T> = T extends readonly (infer U)[] ? U : never;
type A = ElementOf<string[]>; // string
```

`infer` [capture a type found inside another type] works only inside `extends` of a conditional type.

**Mapped types.** Build a new type by looping over keys:

```ts
type Flags<T> = { [K in keyof T]: boolean };
type Handlers = { [K in OrderEvent["type"]]: (e: Extract<OrderEvent, { type: K }>) => void };
```

**Template literal types.**

```ts
type EventName = `order.${"created" | "paid" | "shipped"}`;
```

**`NoInfer`.** Added in TypeScript 5.4. It stops a position from being used for inference. Useful when a default value should not widen the type parameter.

**Rules of thumb.**

- Use a generic only when the output type depends on the input type. If a type parameter appears once, you may not need it.
- Name parameters clearly (`TEvent`, `TResult`) for anything non-trivial.
- Keep constraints as loose as the body allows.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Constraints | `T extends X` | Write a generic with a useful bound |
| Key-safe access | `K extends keyof T`, `T[K]` | Build typed getters |
| Conditional and mapped types | `extends ? :`, `infer`, `[K in keyof T]` | Read and write simple ones |
| Debug | Editor hover, `type X = ...` in a scratch file | Inspect what a complex type becomes |

### Common mistakes

- Making everything generic. Complex types slow the compiler and confuse readers.
- A type parameter used only once in a signature.
- Returning `T` from a function that really returns a cast value.
- Writing deeply nested conditional types that nobody can read.

### Self-check

- What does `T extends HasId` allow you to do inside the function?
- How does `K extends keyof T` make `getField` safe?
- When is a generic not needed?

---

## 8. Utility types

**In plain words.** Utility types are ready-made helpers that change other types. They save you from rewriting similar shapes.

### Understand

From the official Handbook:

| Utility | Effect | Example |
|---|---|---|
| `Partial<T>` | All properties optional | patch payloads |
| `Required<T>` | All properties required | validated config |
| `Readonly<T>` | All properties `readonly` | frozen settings |
| `Record<K, V>` | Object with keys `K`, values `V` | lookup tables |
| `Pick<T, K>` | Keep some properties | public view of a row |
| `Omit<T, K>` | Remove some properties | create input without `id` |
| `Exclude<U, M>` | Remove members from a union | drop a state |
| `Extract<U, M>` | Keep members of a union | pick one event type |
| `NonNullable<T>` | Remove `null` and `undefined` | after a check |
| `ReturnType<F>` | Return type of a function | derive result type |
| `Parameters<F>` | Tuple of parameter types | wrap a function |
| `Awaited<T>` | Unwrap a Promise recursively | async results |
| `NoInfer<T>` | Block inference at a position | typed defaults |
| `ConstructorParameters<C>` | Constructor argument types | factories |
| `InstanceType<C>` | Instance type of a class | factories |

```ts
interface Order {
  id: string;
  customerId: string;
  total: number;
  note?: string;
}

type CreateOrder = Omit<Order, "id">;                 // client sends no id
type OrderPatch = Partial<Omit<Order, "id">>;         // update some fields
type OrderSummary = Pick<Order, "id" | "total">;      // list view
type StatusLabels = Record<"open" | "paid", string>;  // both keys required

type PaidEvent = Extract<OrderEvent, { type: "paid" }>;
type NonCancelled = Exclude<OrderEvent, { type: "cancelled" }>;

async function loadOrder(id: string) {
  return { id, total: 10 };
}
type Loaded = Awaited<ReturnType<typeof loadOrder>>; // { id: string; total: number }
```

**Keep one source of truth.** Define the base type (or zod schema) once. Derive others with utilities. Do not copy fields by hand.

**`Omit` on unions.** `Omit<Union, K>` does not distribute over union members. It collapses them. For unions, write a distributive version:

```ts
type DistributiveOmit<T, K extends PropertyKey> = T extends unknown ? Omit<T, K> : never;
```

**`Partial` is shallow.** It does not make nested objects optional.

**`Record<string, T>` and `noUncheckedIndexedAccess`.** Reading a key gives `T | undefined` when the flag is on (chapter 2). Good: missing keys are a real case.

**`as const` and `readonly`.** `as const` makes literal values narrow and readonly, which pairs with `typeof` and `keyof` to build unions.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Picking and omitting | `Pick`, `Omit`, `Partial` | Write DTO types from one base |
| Union filtering | `Extract`, `Exclude` | Select event types |
| Function types | `ReturnType`, `Parameters`, `Awaited` | Derive types from functions |
| Custom helpers | Mapped types | Write `DistributiveOmit` and similar |

### Common mistakes

- Copying a type by hand, then forgetting to update one copy.
- Using `Omit` on a union and losing the members.
- Expecting `Partial` to apply to nested objects.
- Using `Record<string, any>` instead of `Record<string, unknown>`.

### Self-check

- How do you type "create input" with no `id` from `Order`?
- What does `Awaited<ReturnType<typeof f>>` give for an async function?
- Why does `Omit` misbehave on a union?

---

## 9. Async typing

**In plain words.** An `async` function always returns a Promise. TypeScript tracks what the Promise will give you. The main risk is forgotten `await` and unhandled errors.

### Understand

```ts
async function getOrder(id: string): Promise<Order> {
  const row = await db.findOrder(id);
  if (!row) throw new Error("not found");
  return row;
}
```

An `async` function's return type is `Promise<T>`. Returning `T` is enough; the Promise wraps it.

**`Awaited<T>`** unwraps nested Promises. `await` on a non-Promise returns the value.

**Parallel work.**

```ts
const [order, customer] = await Promise.all([
  getOrder(id),
  getCustomer(customerId),
]); // typed as a tuple: [Order, Customer]
```

`Promise.all` keeps the tuple types. If one Promise rejects, `Promise.all` rejects at once; the others keep running.

`Promise.allSettled` returns a result per Promise, each with `status: "fulfilled" | "rejected"`. This is a discriminated union (chapter 5).

```ts
const results = await Promise.allSettled(ids.map(getOrder));
for (const r of results) {
  if (r.status === "fulfilled") console.log(r.value.id);
  else console.error(r.reason); // reason is any
}
```

**Floating promises.** A Promise that is not awaited or handled can fail silently or crash the process. The lint rule `@typescript-eslint/no-floating-promises` (type-aware) reports it. Also `no-misused-promises` catches an async function passed where a plain callback is expected, such as in an `if` condition or an event handler.

**Errors in async code.** `catch` gives `unknown`. Check with `instanceof Error`. Do not assume the thrown value is an `Error`.

**Typing callbacks and events.** Use function types:

```ts
type Handler<E> = (event: E) => Promise<void>;

const handlers: { [K in OrderEvent["type"]]?: Handler<Extract<OrderEvent, { type: K }>> } = {};
```

**Async iterators and streams.**

```ts
async function* readEvents(): AsyncGenerator<OrderEvent, void, void> {
  // yield events one by one
}
for await (const e of readEvents()) {
  // e: OrderEvent
}
```

**Timeouts and cancellation.** Use `AbortSignal` and pass it to APIs that accept it. Types alone do not cancel anything; the code must handle the signal.

**Return `Promise<Result>`, not exceptions, for expected failures.** See chapter 11.

**Avoid mixing callbacks and Promises.** Wrap old callback APIs with `util.promisify` or a small Promise wrapper, and type the result.

**`return await` inside `try`.** To catch an error from a Promise you return inside `try/catch`, you must `await` it. `return promise;` without `await` skips the `catch`.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Concurrency | `Promise.all`, `allSettled`, `race` | Pick the right one and type results |
| Lint | `no-floating-promises`, `no-misused-promises`, `require-await` | Catch missing awaits in CI |
| Cancellation | `AbortController`, `AbortSignal` | Time-limit calls |
| Streams | `AsyncGenerator`, `for await` | Process events in order |

### Common mistakes

- Forgetting `await`. The code then holds a Promise, not the value. Under strict checks this is often a type error, but not always (for example, in an `if (promise)`).
- Using `forEach` with `async` callbacks. `forEach` does not wait. Use `for...of` or `Promise.all(map(...))`.
- Running thousands of Promises at once with no limit.
- `catch (e)` then reading `e.message` with no check.

### Self-check

- What is the return type of `async function f(): Promise<number>` when you write `return 1`?
- Why does `forEach(async ...)` not wait?
- When does `return promise` inside `try` skip the `catch`?

---

## 10. Runtime validation with zod

**In plain words.** TypeScript types disappear at runtime. Data from outside (HTTP, queues, files) can be anything. A library like zod checks the real data and gives you a typed value only if it is correct.

### Understand

**The boundary rule.** Validate at every place where untrusted data enters: HTTP bodies, queue messages, environment variables, file contents, third-party API responses. Inside the system, trust the types.

**Zod 4 basics.** This section uses Zod 4 syntax, checked against the official docs at zod.dev. Import: `import { z } from "zod";`.

```ts
import { z } from "zod";

const CreateOrder = z.object({
  customerId: z.uuid(),
  email: z.email(),
  quantity: z.int().min(1),
  note: z.string().max(200).optional(),
});

type CreateOrder = z.infer<typeof CreateOrder>; // type derived from the schema
```

Key points from the docs:

- In Zod 4, string formats are top-level functions: `z.email()`, `z.uuid()`, `z.url()`, `z.ipv4()`, `z.iso.datetime()`, `z.iso.date()`. The older method style `z.string().email()` may still work in some versions, but the docs show the top-level form. Use the top-level form.
- `z.int()` accepts integers in the safe integer range.
- `z.object()` removes unknown keys from the output by default. `z.strictObject()` throws on unknown keys. `z.looseObject()` keeps them.
- Everything is required by default. `.optional()` allows `undefined`. `.nullable()` allows `null`. `.nullish()` allows both. `.default(value)` fills in a missing value.

**Parse versus safeParse.**

```ts
const result = CreateOrder.safeParse(req.body);
if (!result.success) {
  // result.error is a ZodError with an `issues` list (code, path, message)
  return reply.status(400).send({ errors: result.error.issues });
}
const input = result.data; // fully typed CreateOrder
```

- `.parse(data)` returns the typed value or throws a `ZodError`.
- `.safeParse(data)` returns a discriminated union: `{ success: true, data }` or `{ success: false, error }`. Use it at boundaries, where bad input is normal.
- `.parseAsync` and `.safeParseAsync` are needed if the schema has async refinements or transforms.
- The docs mention `z.prettifyError` and `z.treeifyError` to format errors. The exact signatures were not verified here; check zod.dev before use.

**Infer types, do not duplicate them.** `z.infer<typeof Schema>` gives the output type. `z.input<typeof Schema>` and `z.output<typeof Schema>` differ when a schema uses `.transform()` or `.default()`.

**Discriminated union in zod.**

```ts
const OrderEventSchema = z.discriminatedUnion("type", [
  z.object({ type: z.literal("created"), orderId: z.string(), total: z.number() }),
  z.object({ type: z.literal("paid"), orderId: z.string(), paidAt: z.iso.datetime() }),
  z.object({ type: z.literal("cancelled"), orderId: z.string(), reason: z.string() }),
]);

type OrderEvent = z.infer<typeof OrderEventSchema>;
```

`z.discriminatedUnion(key, options)` uses the tag to pick the right schema directly, so errors are clear and parsing is fast. The tag must be a literal, enum, `null` or `undefined` schema.

**Enums and literals.** `z.enum(["open", "paid"])` gives a string union. `z.literal("paid")` matches one value. In Zod 4, `z.enum()` also accepts TypeScript enums; `z.nativeEnum()` is deprecated.

**Coercion.** Query strings and env vars are strings. `z.coerce.number()` converts with `Number(input)`:

```ts
const Env = z.object({
  PORT: z.coerce.number().int().default(3000),
  NODE_ENV: z.enum(["development", "test", "production"]),
});
const env = Env.parse(process.env); // fail fast at startup
```

Coercion can surprise: `z.coerce.boolean()` uses `Boolean(input)`, so the string `"false"` becomes `true`. Parse booleans explicitly.

**Transforms and refinements.** `.transform()` changes the value after validation. `.refine()` adds a custom check. Keep them small; complex logic belongs in your code, not in the schema.

**Dates.** JSON has no date type. Validate an ISO string with `z.iso.datetime()`, then convert to `Date` in a transform if you need it.

**Error customisation.** Zod 4 supports a custom message through an `error` option or a `message` string. The exact option shape was not verified here; check the docs for your version.

**Other validators.** Valibot, ArkType and TypeBox solve the same problem. The boundary rule applies to all.

**Version note.** Zod 4 changed APIs from Zod 3 (top-level formats, error handling, `z.record` needs a key schema, among others). When reading old tutorials, check the version.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Schemas | `z.object`, `z.enum`, `z.discriminatedUnion` | Describe any payload |
| Parsing | `.safeParse`, `.parse` | Choose by context |
| Type derivation | `z.infer`, `z.input`, `z.output` | One source of truth |
| Env config | `z.coerce`, `.default` | Fail fast on bad configuration |

### Common mistakes

- Casting `req.body as CreateOrder` and skipping validation.
- Defining a TypeScript type and a zod schema separately so they drift apart.
- Using `.parse` at a boundary and letting a `ZodError` become a 500 response.
- Using `z.coerce.boolean()` on strings.
- Following Zod 3 examples with Zod 4.
- Returning raw validation errors that reveal internal field names to clients without thought.

### Self-check

- Why does a TypeScript type not protect against a bad HTTP body?
- What is the difference between `.parse` and `.safeParse`?
- How do you get a TypeScript type from a zod schema?
- Why does `z.discriminatedUnion` need a literal tag?

---

## 11. The Result pattern

**In plain words.** Some failures are normal, such as "customer not found". A Result type makes success and failure part of the return type, so the caller must handle both.

### Understand

Exceptions are invisible in types. A function signature does not tell you what it can throw. A **Result** is a discriminated union (chapter 5):

```ts
type Ok<T> = { ok: true; value: T };
type Err<E> = { ok: false; error: E };
type Result<T, E> = Ok<T> | Err<E>;

const ok = <T>(value: T): Ok<T> => ({ ok: true, value });
const err = <E>(error: E): Err<E> => ({ ok: false, error });
```

Use:

```ts
type PlaceOrderError =
  | { code: "out_of_stock"; sku: string }
  | { code: "invalid_quantity" }
  | { code: "payment_declined"; reason: string };

function placeOrder(input: CreateOrder): Result<Order, PlaceOrderError> {
  if (input.quantity > 100) return err({ code: "invalid_quantity" });
  return ok({ id: "o-1", customerId: input.customerId, total: 10 });
}

const r = placeOrder(input);
if (!r.ok) {
  switch (r.error.code) {
    case "out_of_stock":
      return reply.status(409).send();
    case "invalid_quantity":
      return reply.status(422).send();
    case "payment_declined":
      return reply.status(402).send();
    default:
      return assertNever(r.error);
  }
}
console.log(r.value.id); // narrowed to Order
```

Zod's `safeParse` already follows this idea.

**When to use which.**

| Situation | Use |
|---|---|
| Expected, recoverable failure the caller should handle | Result |
| Programmer error, broken invariant, impossible state | throw |
| Failure that the framework will turn into a 500 anyway | throw |

**Mixing styles.** Wrap code that throws at the edge:

```ts
async function tryAsync<T>(fn: () => Promise<T>): Promise<Result<T, Error>> {
  try {
    return ok(await fn());
  } catch (e) {
    return err(e instanceof Error ? e : new Error(String(e)));
  }
}
```

**Libraries.** `neverthrow`, `effect` and `ts-results` offer richer APIs. A 10-line own type is enough to start. Library names and APIs are not verified here.

**Cost.** Result adds code at every call site. Do not use it for every function. Use it at module boundaries where the failure is part of the contract.

**Error types.** Prefer a tagged union of error objects over `string` or `Error` subclasses when callers need to branch.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Tagged results | `ok: true / false` | Write and use a Result type |
| Error unions | Tagged error objects | Exhaustive handling with `assertNever` |
| Edge wrapping | `tryAsync` helper | Convert thrown errors into Results |

### Common mistakes

- Using Result for everything, creating noise.
- Using `string` as the error type so callers cannot branch.
- Throwing inside a function that returns a Result.
- Ignoring the result: `placeOrder(x);` with no check. A lint rule or a team habit must prevent this.

### Self-check

- Why is a Result a discriminated union?
- Give one case where you throw, and one where you return a Result.
- How does exhaustiveness help with error handling?

---

## 12. Branded types

**In plain words.** Two string IDs have the same type, so you can mix them by mistake. A brand adds a hidden label to the type, so the compiler treats them as different.

### Understand

The problem (structural typing, chapter 1):

```ts
type OrderId = string;
type CustomerId = string;

function loadOrder(id: OrderId) {}
const customerId: CustomerId = "c-1";
loadOrder(customerId); // compiles, but wrong
```

The fix is a **brand** [a fake extra property that exists only in the type]:

```ts
declare const brand: unique symbol;
type Brand<T, B extends string> = T & { readonly [brand]: B };

type OrderId = Brand<string, "OrderId">;
type CustomerId = Brand<string, "CustomerId">;

function loadOrder(id: OrderId) {}

const c = "c-1" as CustomerId;
loadOrder(c); // error: CustomerId is not assignable to OrderId
```

The brand has no runtime cost. The value is still a normal string.

**Create values in one place.** The only unsafe step is the cast. Hide it in a function that validates:

```ts
function toOrderId(s: string): OrderId {
  if (!/^o-\d+$/.test(s)) throw new Error("invalid order id");
  return s as OrderId;
}
```

With zod 4, brand a schema (verify the `.brand()` method in the docs for your version; not verified here):

```ts
const OrderIdSchema = z.string().regex(/^o-\d+$/).brand<"OrderId">();
type OrderId = z.infer<typeof OrderIdSchema>;
```

**Good uses.** IDs of different entities, money amounts in different currencies, validated values such as `Email`, units (`Milliseconds` versus `Seconds`), and "already sanitised" strings.

**Limits.** A brand is not a runtime guarantee. If you cast without checking, the brand lies. Branded values also need care when parsing JSON, since the brand must be applied again after validation.

**Alternative.** A class with a private field gives nominal typing [types compared by name] but has runtime cost and a different shape.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Brand helper | `Brand<T, B>` with `unique symbol` | Define once in a shared file |
| Smart constructors | `toOrderId` functions | One place for validation and cast |
| Schema brands | `.brand()` in zod | Apply a brand during parsing |

### Common mistakes

- Casting to a brand everywhere with `as`.
- Branding too many things. Use it where mixing is costly.
- Forgetting that JSON loses the brand: re-validate after reading.

### Self-check

- Why does TypeScript allow passing a `CustomerId` string as an `OrderId`?
- Does a brand exist at runtime?
- Where should the cast to a brand live?

---

## 13. The satisfies operator

**In plain words.** `satisfies` checks that a value matches a type, but keeps the value's own more exact type. A type annotation does the check but loses the detail.

### Understand

Introduced in TypeScript 4.9 (confirmed in the official release notes).

```ts
type Route = { path: string; method: "GET" | "POST" };

// Annotation: checks, but widens the type
const routesA: Record<string, Route> = {
  list: { path: "/orders", method: "GET" },
};
routesA.lst; // allowed by the type, because any string key exists

// satisfies: checks, and keeps exact keys
const routesB = {
  list: { path: "/orders", method: "GET" },
  create: { path: "/orders", method: "POST" },
} satisfies Record<string, Route>;

routesB.list.method; // "GET"
routesB.lst;         // error: no such key
```

The official example catches a typo in a key and still lets `palette.green.toUpperCase()` work, because `green` stays a `string`.

**Common uses.**

- Config objects: check shape, keep literal keys and values.
- Lookup tables whose keys come from a union:

```ts
const labels = {
  created: "Created",
  paid: "Paid",
  cancelled: "Cancelled",
} satisfies Record<OrderEvent["type"], string>; // missing key is an error
```

- With `as const`: `const x = { ... } as const satisfies Config;` gives readonly literal types and a shape check.

**Annotation versus `satisfies` versus `as`.**

| Tool | Checks the shape | Keeps exact type | Can hide errors |
|---|---|---|---|
| `const x: T = v` | Yes | No (becomes `T`) | No |
| `v satisfies T` | Yes | Yes | No |
| `v as T` | Weakly | No | Yes |

**Excess property checks.** `satisfies` reports unknown keys in an object literal, like an annotation does.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Config tables | `satisfies Record<K, V>` | Catch missing and extra keys |
| Literal types | `as const satisfies T` | Exact and checked |
| Choosing | annotation vs `satisfies` | Explain the difference |

### Common mistakes

- Using `as` where `satisfies` would check the value.
- Using a wide annotation and losing key information.
- Expecting `satisfies` to change the variable's type. It does not.

### Self-check

- What does `satisfies` keep that an annotation loses?
- How can `satisfies Record<Union, string>` find a missing key?
- When is `as` the wrong tool?

---

## 14. Declaration files and @types

**In plain words.** A `.d.ts` file holds only types, no code. It tells TypeScript what a JavaScript library looks like. Many libraries ship one, and the rest use `@types` packages.

### Understand

**Declaration file (`.d.ts`).** It describes the shape of JavaScript code. It has no runtime output.

```ts
// legacy-billing.d.ts
declare module "legacy-billing" {
  export function charge(customerId: string, cents: number): Promise<{ ok: boolean }>;
}
```

**Where types come from, in order of preference:**

1. The package ships its own types. Its `package.json` has a `types` field (or `typings`) or an `exports` entry with types.
2. A community package from DefinitelyTyped. Install it as a dev dependency: `npm install --save-dev @types/express`. DefinitelyTyped publishes to the `@types` scope on npm.
3. Your own `.d.ts` with `declare module`.

TypeScript finds `node_modules/@types` packages automatically, unless the `types` compiler option limits it.

**Shorthand.** `declare module "untyped-lib";` makes everything from that module `any`. Use it only as a temporary step.

**Publishing a library.** Set `"declaration": true` so `tsc` writes `.d.ts` files. Point to them with `types` in `package.json`, and use the `exports` map so that ESM and CommonJS consumers each get matching types.

**Global augmentation.** To add a field to a global or library type, use declaration merging:

```ts
// types/express.d.ts
declare global {
  namespace Express {
    interface Request {
      requestId?: string;
    }
  }
}
export {};
```

The `export {}` makes the file a module, which `declare global` needs. Include the file in `tsconfig.json`.

**`@types` version match.** Keep the `@types/node` major version equal to your Node major version. The types describe a specific runtime.

**`skipLibCheck`.** It skips checking `.d.ts` files. Conflicts between two `@types` versions then do not break your build. Real problems may stay hidden.

**Import types without runtime code.** `import type { Order } from "./order.js"` is erased, so it adds no runtime import.

**The `"types"` compiler option.** `"types": ["node"]` restricts which `@types` packages load globally. In a project with many `@types`, an explicit list avoids surprises such as clashing global names between test frameworks.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Finding types | `npm view @types/<pkg>`, package `types` field | Know if a package needs `@types` |
| Writing declarations | `declare module`, `declare global` | Type a small untyped module |
| Generating | `declaration: true` | Publish types with a library |
| Checking packages | `npx @arethetypeswrong/cli` (verify it for your case) | Detect wrong `exports` and types |

### Common mistakes

- Installing `@types/x` into `dependencies` of an application with no need. Use `devDependencies`.
- Mismatched `@types/node` and Node versions.
- Forgetting `export {}` in a global augmentation file.
- Leaving `declare module "x";` forever, which turns everything into `any`.

### Self-check

- Where does TypeScript look for types of a package?
- What does `@types/express` contain?
- Why does a global augmentation file need `export {}`?

---

## 15. Modules: ESM and NodeNext

**In plain words.** Node has two module systems: ESM (`import`) and CommonJS (`require`). `NodeNext` tells TypeScript to follow Node's real rules. The rules about file endings in imports cause most confusion.

### Understand

**Two systems.**

| | CommonJS (CJS) | ES Modules (ESM) |
|---|---|---|
| Syntax | `require`, `module.exports` | `import`, `export` |
| Loading | Synchronous | Asynchronous |
| Top-level `await` | No | Yes |
| `__dirname` | Yes | No (use `import.meta.dirname`) |

`import.meta.dirname` and `import.meta.filename` exist in modern Node versions (Node 20.11 and later; verify for your version).

**How Node picks the system.** Per the TypeScript docs for `node16` and `nodenext`:

- `.mts` and `.mjs` files are always ESM. `.cts` and `.cjs` files are always CommonJS.
- `.ts` and `.js` files are ESM if the nearest `package.json` has `"type": "module"`. Otherwise they are CommonJS.

So with `module: "nodenext"` and no `"type"` field, your `.ts` files compile to CommonJS. For ESM, add:

```json
{ "type": "module" }
```

**File extensions in imports.** In ESM, relative imports need a file extension. Write the **runtime** extension, `.js`, even in a `.ts` file:

```ts
import { priceOf } from "./pricing.js"; // file on disk is pricing.ts
```

TypeScript looks for `./pricing.ts`, then `./pricing.d.ts`, then `./pricing.js`. This looks odd but matches what Node will run after compilation. Directory imports (`./dir`) are also not allowed in ESM; write `./dir/index.js`.

Newer TypeScript (5.7 and later) has `rewriteRelativeImportExtensions`, and Node's own TypeScript support requires real `.ts` extensions in imports (chapter 16). Use one approach per project.

**`import type` and `verbatimModuleSyntax`.** With `verbatimModuleSyntax`, type-only imports must use `import type` or `type` modifiers:

```ts
import type { Order } from "./order.js";
import { priceOf, type Currency } from "./pricing.js";
```

This keeps the output predictable and avoids a runtime error when Node tries to load a type as a value.

**Default imports and interop.** `esModuleInterop` (implied by `nodenext`) lets you `import x from "cjs-lib"`. Some CommonJS libraries still need care with default exports. When a default import is `undefined` at runtime, check how the library exports.

**Dual packages.** A library can publish ESM and CJS builds and declare them in the `exports` map. Types must match each build. This is hard to get right; check with a tool such as the "Are the types wrong" checker (verify for your case).

**Requiring ESM from CJS.** In `nodenext`, TypeScript matches Node 22.12 and later, where `require` can load an ES module. It does not work if that module uses top-level `await`, and TypeScript does not warn about it. Dynamic `import()` always works in both systems.

**`moduleResolution`.** Use `nodenext` for Node apps. Use `bundler` only if a bundler such as esbuild or Vite resolves your code. Do not mix them without a reason.

**Path aliases (`paths`).** `paths` only changes how TypeScript resolves types. It does not change the emitted code. Node will not understand `@/lib/x` unless a tool or the `imports` field in `package.json` maps it. Prefer the `imports` field (`"#lib/*"`) or relative paths.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Choosing a system | `"type": "module"`, `.mts`, `.cts` | Set up an ESM service from scratch |
| Extensions | `.js` in imports | Explain why |
| Type imports | `import type`, `verbatimModuleSyntax` | Keep type imports erased |
| Resolution | `tsc --traceResolution` | Debug "cannot find module" |

### Common mistakes

- Writing `./pricing` without an extension in ESM and getting `ERR_MODULE_NOT_FOUND`.
- Writing `./pricing.ts` in imports when compiling with `tsc` and no rewrite option.
- Using `__dirname` in ESM.
- Expecting `paths` aliases to work in Node at runtime.
- Choosing `moduleResolution: "node"` (the old CommonJS mode) for a new project.

### Self-check

- How does Node decide if a `.ts` file is ESM or CJS under `nodenext`?
- Why do you write `.js` in an import inside a `.ts` file?
- What does `verbatimModuleSyntax` force you to do?

---

## 16. Tooling and running TypeScript in Node

**In plain words.** Several tools do different jobs. `tsc` checks types and compiles. `tsx` runs TypeScript quickly but does not check types. ESLint finds bug patterns. Vitest runs tests.

### Understand

**Who does what.**

| Tool | Job | Checks types? |
|---|---|---|
| `tsc` | Type check and compile to JavaScript | Yes |
| `tsx` | Run `.ts` files in development, watch mode | No |
| Node (type stripping) | Run `.ts` files by removing types | No |
| `vitest` | Run tests, including TypeScript | No (unless you add a type check step) |
| ESLint with typescript-eslint | Lint rules, some use type information | Partly |

Run type checking as its own step: `tsc --noEmit` in CI and before commit.

**Node running TypeScript directly.** From the Node.js docs: type stripping is enabled by default in Node 23.6.0 and 22.18.0 and later, and is marked stable from 24.12.0 and 25.2.0. Node removes the types and runs the rest. It does no type checking.

```bash
node src/server.ts
```

Rules from the docs:

- Only **erasable syntax** works. `enum`, runtime `namespace`, and parameter properties (constructor shorthand like `constructor(private x: string)`) are not supported and throw `ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX`. Decorators are not supported.
- Imports must use the real `.ts` extension, for example `import "./file.ts"`.
- Type-only imports must use `import type` (or the `type` modifier). The docs recommend `verbatimModuleSyntax`.
- Docs recommend, for TypeScript 5.8 and later: `noEmit`, `target: "esnext"`, `module: "nodenext"`, `rewriteRelativeImportExtensions`, `erasableSyntaxOnly` and `verbatimModuleSyntax`.
- Files under `node_modules` are not stripped. Libraries must ship JavaScript.

```json
{
  "compilerOptions": {
    "noEmit": true,
    "target": "esnext",
    "module": "nodenext",
    "rewriteRelativeImportExtensions": true,
    "erasableSyntaxOnly": true,
    "verbatimModuleSyntax": true
  }
}
```

`erasableSyntaxOnly` makes `tsc` report syntax that Node cannot strip.

For older Node versions, or when you need unsupported syntax, use `tsx` or compile with `tsc` first.

**tsx.** A tool that runs TypeScript through esbuild. Typical use: `npx tsx src/server.ts` and `npx tsx watch src/server.ts`. It works with both ESM and CJS. It does not type check. (The tsx website could not be fetched in this review, so confirm commands on its repository.)

**Production build.** Compile with `tsc` to `dist/`, then run `node dist/server.js`. Ship only `dist/` and production dependencies. Enable source maps and run Node with `--enable-source-maps` for readable stack traces.

**ESLint with typescript-eslint.** Install (from the official getting-started page):

```bash
npm install --save-dev eslint @eslint/js typescript typescript-eslint
```

Flat config, with type-aware rules (from the official typed-linting page):

```js
// eslint.config.mjs
import js from "@eslint/js";
import { defineConfig } from "eslint/config";
import tseslint from "typescript-eslint";

export default defineConfig({
  files: ["**/*.{js,ts}"],
  extends: [
    js.configs.recommended,
    tseslint.configs.strict,
    tseslint.configs.strictTypeChecked,
  ],
  languageOptions: {
    parserOptions: {
      projectService: true,
      tsconfigRootDir: import.meta.dirname,
    },
  },
});
```

`recommended` and `recommendedTypeChecked` are the gentler sets; `strict` and `strictTypeChecked` add more opinionated rules. Typed rules read your `tsconfig.json`, so they are slower but catch floating promises and unsafe `any`. Rule names change between versions; confirm in the rule list.

**Vitest.** Install with `npm install -D vitest`. Test files need `.test.` or `.spec.` in the name. Run `vitest` for watch mode, `vitest run` for one pass. The docs list Node 22.12 or later as the requirement for the current version.

```ts
// pricing.test.ts
import { describe, expect, it } from "vitest";
import { priceOf } from "./pricing.js";

describe("priceOf", () => {
  it("adds tax", () => {
    expect(priceOf(100)).toBe(120);
  });
});
```

Vitest has type-level test helpers such as `expectTypeOf` (not verified here; check the docs).

**Typical scripts.**

```json
{
  "scripts": {
    "dev": "tsx watch src/server.ts",
    "build": "tsc -p tsconfig.build.json",
    "start": "node dist/server.js",
    "typecheck": "tsc --noEmit",
    "lint": "eslint .",
    "test": "vitest run"
  }
}
```

**Other tools.** `ts-node` (older runner), `swc` and `esbuild` (fast compilers with no type check), `tsup` and `tsdown` (library bundlers). Evaluate current status before choosing one.

**CI order.** `typecheck`, then `lint`, then `test`, then `build`.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Type check | `tsc --noEmit` | Own a CI gate |
| Dev run | `tsx watch`, `node file.ts` | Run and reload quickly |
| Lint | ESLint, typescript-eslint | Configure typed rules |
| Test | Vitest | Write unit tests and mocks |
| Build | `tsc`, source maps | Produce and run `dist/` |

### Common mistakes

- Thinking `tsx` or `node file.ts` catches type errors. They do not.
- Using `enum` and then hitting `ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX` in Node.
- Running `ts-node` in production. Compile first.
- Skipping `typecheck` in CI because "tests pass".
- Enabling typed lint rules without `projectService` or a correct `tsconfigRootDir`.

### Self-check

- Which tools in your setup check types, and which only remove them?
- Which syntax fails under Node's type stripping?
- Why compile to `dist/` for production?

---

## 17. Common gotchas

**In plain words.** These are the surprises that cost people hours. Learn them once, and many bugs become obvious.

### Understand

1. **Types are not checked at runtime.** A correct type with bad data still crashes. Validate at the boundary (chapter 10).

2. **`Object.keys` returns `string[]`.** Not `(keyof T)[]`, because objects may have extra keys. Cast carefully or use a typed helper when you know the shape.

3. **Excess property checks only apply to fresh object literals.**

```ts
const extra = { id: "1", total: 5, secret: "x" };
const o: { id: string; total: number } = extra; // allowed
const p: { id: string; total: number } = { id: "1", total: 5, secret: "x" }; // error
```

4. **Arrays are covariant and mutable.** A `Dog[]` can be assigned to `Animal[]`, then someone pushes a `Cat`. Use `readonly Animal[]` for parameters you do not change.

5. **`JSON.stringify` loses types.** `Date` becomes a string, `Map` and `Set` become `{}`, `undefined` is dropped, `BigInt` throws. Your type says `Date`; the data is a string after a round trip.

6. **`number` is a float.** Money in `number` has rounding errors. Store integer cents or use a decimal library. Large database IDs can exceed `Number.MAX_SAFE_INTEGER`; use strings or `bigint`.

7. **`enum` quirks.** Numeric enums accept any number in older versions, and enums break type stripping. Prefer string unions.

8. **`this` in methods.** Passing `obj.method` as a callback loses `this`. Use an arrow function or `bind`. `strictBindCallApply` and `noImplicitThis` help.

9. **Optional versus `undefined`.** `{ a?: string }` and `{ a: string | undefined }` differ. Only `exactOptionalPropertyTypes` makes that precise.

10. **`as` is not a conversion.** `"5" as unknown as number` is still the string `"5"` at runtime.

11. **Widening.** `let x = "a"` is `string`; `const x = "a"` is `"a"`. Object properties widen, so `{ method: "GET" }` has `method: string`. Fix with `as const` or `satisfies`.

12. **Index signatures hide missing keys.** Without `noUncheckedIndexedAccess`, `record[key]` is typed as present when it may not be.

13. **Empty array inference.** `const xs = []` may become `any[]` or `never[]` depending on flow. Annotate it.

14. **Promise in condition.** `if (asyncCheck())` is always truthy, since a Promise is an object. Lint rule `no-misused-promises` catches it.

15. **`catch` clause value.** It is `unknown`. Anything can be thrown, including strings.

16. **Declaration merging surprises.** Two `interface Foo` blocks join. Name clashes between modules in global scripts are a silent risk. Make files modules with an `import` or `export`.

17. **Type-only circular imports.** Use `import type` to avoid runtime cycles.

18. **Compile speed.** Huge unions, deep conditional types and big inferred types slow the compiler. Annotate exported functions and split big types. Use `tsc --extendedDiagnostics` to measure.

19. **Library types can be wrong.** A `@types` package may differ from the real behaviour. Test critical calls.

20. **Strict mode in tests.** Test files often cast and mock. Keep them type-checked but allow a few lint exceptions.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Finding wrong inference | Editor hover, `// ^?` in the playground | Check what the compiler thinks |
| Performance | `tsc --extendedDiagnostics`, `--generateTrace` | Find slow types |
| Lint | `no-misused-promises`, `no-unsafe-*`, `no-unnecessary-type-assertion` | Catch many gotchas automatically |

### Common mistakes

- Trusting a type for data that came from outside.
- Fixing a type error with `as` before understanding it.
- Storing money as floating-point numbers.
- Forgetting that JSON round trips change types.

### Self-check

- Why does `Object.keys(obj)` not return `(keyof T)[]`?
- Why does `JSON.parse(JSON.stringify(x))` not return the same types?
- What does `const x = { method: "GET" }` infer for `method`, and how do you fix it?

---

## 18. Practice labs

Do these in order. Each one is small. Work in a fresh folder with `strict: true`.

**Lab 1. Strict project.** Create a project with the `tsconfig.json` from chapter 2. Add `noUncheckedIndexedAccess` and `exactOptionalPropertyTypes`. Write code that breaks each one, then fix it. Run `tsc --noEmit`.

**Lab 2. Event model.** Define `OrderEvent` as a discriminated union with four members. Write `describe(e)` with a `switch` and `assertNever`. Add a fifth member and watch the compile error list the places to fix.

**Lab 3. Narrowing drills.** Write one function for each narrowing tool: `typeof`, `instanceof`, `in`, a type predicate, and an assertion function. Hover on variables to confirm the narrowed types.

**Lab 4. Remove `any`.** Take a 50-line JavaScript file that uses `JSON.parse` and `catch (e)`. Convert it to TypeScript with zero `any`. Turn on `@typescript-eslint/no-explicit-any` and `no-unsafe-*`.

**Lab 5. Generics.** Write `indexById<T extends { id: string }>`, `getField<T, K extends keyof T>`, and a generic `Repository<T>`. Add a test with vitest.

**Lab 6. Utility types.** From one `Order` interface, derive `CreateOrder`, `OrderPatch`, `OrderSummary` and `PaidEvent`. Do not retype any field.

**Lab 7. Async.** Write a function that loads 20 orders with limited concurrency (for example 5 at a time) and returns `Result` values. Use `Promise.allSettled` once, and compare. Enable `no-floating-promises` and fix every warning.

**Lab 8. Zod boundary.** Build a small HTTP handler (any framework). Validate the body with a zod 4 schema. Return `400` with the issues on failure. Validate `process.env` at startup with `z.coerce`. Use `z.infer` for all types.

**Lab 9. Result and errors.** Implement `placeOrder` that returns `Result<Order, PlaceOrderError>`. Map each error code to an HTTP status with an exhaustive `switch`.

**Lab 10. Branded IDs.** Create `OrderId` and `CustomerId` brands with validating constructors. Show that mixing them fails to compile.

**Lab 11. satisfies.** Create a route table and a label table with `satisfies Record<...>`. Remove one key and observe the error.

**Lab 12. Declaration file.** Create a tiny untyped JavaScript module. Write a `.d.ts` with `declare module`. Then add a global augmentation for `Request`.

**Lab 13. ESM service.** Set `"type": "module"`, `module: "nodenext"`, and `.js` extensions in imports. Build with `tsc`, run `node dist/index.js`. Then try `node src/index.ts` on a Node version that supports it, using `erasableSyntaxOnly`. Note what fails when you add an `enum`.

**Lab 14. CI gate.** Write the scripts from chapter 16. Make a failing type error, lint error and test failure, one at a time, and confirm each script catches its own.

---

## 19. Interview questions

Tags: **[Fresher]** entry level, **[Mid]** some years of experience, **[Senior]** design and trade-off depth. The answers are short models, not scripts. Practice saying them in your own words.

### Top 10 must-know

**1. [Fresher] What is TypeScript, and does it check types at runtime?**
TypeScript is JavaScript with a static type checker. It checks types at compile time, then removes them. At runtime only JavaScript runs, so data from outside needs runtime validation.

**2. [Fresher] What does `strict: true` do?**
It turns on a group of checks: `strictNullChecks`, `noImplicitAny`, `strictFunctionTypes`, `strictPropertyInitialization`, `useUnknownInCatchVariables` and others. It does not include `noUncheckedIndexedAccess` or `exactOptionalPropertyTypes`.

**3. [Fresher] What is the difference between `unknown` and `any`?**
`any` disables checking and spreads unsafety. `unknown` accepts any value but you must narrow it before use. Use `unknown` for untrusted data.

**4. [Fresher] When do you use a `type` and when an `interface`?**
Both describe object shapes. Only `type` can name unions, tuples and mapped types. Only `interface` supports declaration merging. Pick one convention and stay consistent.

**5. [Mid] What is a discriminated union, and how do you check it is exhaustive?**
It is a union whose members share a literal field, such as `type`. A `switch` on that field narrows each case. In the default branch, assign the value to `never`; a missing case then fails to compile.

**6. [Mid] How do you validate untrusted input in a TypeScript service?**
Use a runtime schema library such as zod at every boundary. Parse with `safeParse`, return a clear error on failure, and derive the static type with `z.infer`. Types alone do not validate.

**7. [Mid] What does `satisfies` do, and how does it differ from a type annotation or `as`?**
It checks a value against a type but keeps the value's exact inferred type. An annotation widens the value to the annotated type. `as` overrides the checker and can hide errors.

**8. [Mid] How does Node decide if a `.ts` file is ESM or CommonJS under `NodeNext`?**
By file extension (`.mts` is ESM, `.cts` is CJS) and the nearest `package.json` `"type"` field. With `"type": "module"` the `.ts` files are ESM. In ESM, relative imports need the runtime extension, such as `.js`.

**9. [Mid] What is the difference between `tsc`, `tsx` and Node running `.ts` files?**
`tsc` checks types and compiles. `tsx` and Node's type stripping only remove types and run the code, with no type check. So you still need `tsc --noEmit` in CI. Node's stripping also rejects `enum` and similar syntax.

**10. [Senior] How do you model expected failures in a typed service?**
Return a typed Result (a discriminated union of success and tagged errors) for failures the caller must handle. Throw for bugs and broken invariants. Handle error codes with an exhaustive `switch`, so a new error forces updates.

### Fresher

**11. [Fresher] What is type inference?**
The compiler works out types from values, so you need not annotate everything. Annotate function parameters and exported APIs, and let local variables be inferred.

**12. [Fresher] What is a union type?**
A type that can be one of several types, such as `string | number`. You narrow it with checks like `typeof` before using type-specific members.

**13. [Fresher] What is narrowing?**
Using a check (`typeof`, `instanceof`, `in`, equality, a type predicate) so the compiler knows a more exact type inside a branch.

**14. [Fresher] What is the difference between `??` and `||`?**
`??` replaces only `null` and `undefined`. `||` replaces any falsy value, including `0` and an empty string. Use `??` for defaults.

**15. [Fresher] What does `readonly` do?**
It stops reassignment at the type level. It does not freeze the value at runtime.

**16. [Fresher] What is `as const`?**
It makes a literal value readonly and gives it the narrowest type, so `["a", "b"] as const` has type `readonly ["a", "b"]`.

**17. [Fresher] What is a generic?**
A type or function with a type parameter, so it works for many types while keeping them linked, for example `first<T>(items: T[]): T | undefined`.

**18. [Fresher] Name five utility types.**
`Partial`, `Required`, `Pick`, `Omit`, `Record`. Also `Readonly`, `Exclude`, `Extract`, `ReturnType`, `Awaited`.

**19. [Fresher] What type does `e` have in `catch (e)`?**
`unknown` under `strict` (via `useUnknownInCatchVariables`). Check with `instanceof Error` before reading `message`.

**20. [Fresher] What is a `.d.ts` file?**
A declaration file with only type information and no runtime code. It describes a JavaScript library to the compiler.

**21. [Fresher] What is `@types/node`?**
A package of type declarations for Node's built-in modules, from DefinitelyTyped. Install it as a dev dependency.

**22. [Fresher] What is the return type of an async function?**
`Promise<T>`, where `T` is the type of the returned value.

### Mid

**23. [Mid] Why can two different ID types be mixed up, and how do you stop it?**
TypeScript is structurally typed, so two `string` aliases are the same type. A branded type adds a type-only marker, such as `string & { readonly [brand]: "OrderId" }`. Create values through a validating constructor.

**24. [Mid] What does `noUncheckedIndexedAccess` do?**
It adds `undefined` to the result of index access, such as `arr[0]` or `record[key]`. It forces you to handle a missing element.

**25. [Mid] What does `exactOptionalPropertyTypes` do?**
It treats `prop?: T` as "may be absent" only. You cannot assign `undefined` unless the type includes it.

**26. [Mid] Explain `T extends keyof U` in a generic function.**
It limits `T` to keys of `U`. `getField<U, K extends keyof U>(o: U, k: K): U[K]` returns the exact property type and rejects unknown keys.

**27. [Mid] What does `infer` do?**
Inside a conditional type it captures a type from a matched position, for example `T extends (infer U)[] ? U : never` gets the element type.

**28. [Mid] How does `Omit` behave on a union, and what is the fix?**
It does not distribute, so union members collapse. Write `type DistributiveOmit<T, K extends PropertyKey> = T extends unknown ? Omit<T, K> : never`.

**29. [Mid] `Promise.all` versus `Promise.allSettled`?**
`all` rejects as soon as one rejects, and gives a typed tuple on success. `allSettled` waits for all and returns a status object for each. Choose by whether partial success is useful.

**30. [Mid] Why is `array.forEach(async ...)` a bug?**
`forEach` ignores the returned Promises, so it does not wait and errors become unhandled. Use `for...of` with `await`, or `Promise.all(array.map(...))`.

**31. [Mid] What lint rules protect async code?**
`@typescript-eslint/no-floating-promises` and `no-misused-promises`, both needing type-aware linting with `parserOptions.projectService`.

**32. [Mid] Zod 3 to Zod 4: what changed in string validators?**
The docs show top-level formats such as `z.email()`, `z.uuid()` and `z.iso.datetime()` instead of chaining off `z.string()`. Check the migration guide for the full list.

**33. [Mid] What is the difference between `z.infer`, `z.input` and `z.output`?**
`z.infer` is the output type. `z.input` is what the schema accepts. They differ with transforms and defaults.

**34. [Mid] Why does the `.js` extension appear in `.ts` imports?**
ESM in Node needs the runtime file name. TypeScript maps `./x.js` to `./x.ts` during resolution, and the emitted code is correct for Node.

**35. [Mid] Why is `paths` not enough for aliases in Node?**
`paths` affects only type resolution. Node does not read it. Use the `imports` field in `package.json`, or a bundler or loader that rewrites paths.

**36. [Mid] What fails under Node's type stripping?**
Syntax that needs generated code: `enum`, runtime `namespace`, parameter properties, and decorators. Use `erasableSyntaxOnly` to find them at compile time.

**37. [Mid] When would you use a type predicate versus an assertion function?**
A predicate (`x is T`) returns a boolean for use in `if`. An assertion function (`asserts x is T`) throws on failure and narrows the code after the call.

**38. [Mid] What is declaration merging and a real use?**
Two declarations with the same name combine. Libraries use it so you can add fields to a shared interface, such as adding a field to a request type through `declare global`.

**39. [Mid] How would you type a handler map for every event type?**
A mapped type over the tag union: `{ [K in E["type"]]: (e: Extract<E, { type: K }>) => void }`. A missing handler is a compile error.

**40. [Mid] Why not store money as `number`?**
Binary floating point cannot represent many decimals exactly. Use integer minor units (cents) or a decimal library.

### Senior

**41. [Senior] How do you enforce no `any` across a large codebase?**
Enable `strict`, ban `no-explicit-any`, and turn on the typed `no-unsafe-*` rules. Validate at boundaries so `unknown` becomes typed data. Use `@ts-expect-error` with a reason for rare exceptions, and track them.

**42. [Senior] Design a boundary strategy for a multi-service event system.**
Define schemas once (zod or a shared package), derive types, and validate on both publish and consume. Use a tagged union with a version field. On unknown or invalid events, send to a dead-letter queue instead of crashing. Make handlers exhaustive.

**43. [Senior] How do you evolve an event schema safely?**
Add optional fields first, keep old versions readable, and add a `version` tag in the union. Remove fields only after all consumers have upgraded. Test with recorded real messages.

**44. [Senior] Interface or type for a large domain model, and why?**
Both work. Choose one rule, enforce it with lint, and use `type` for unions and mapped types. Performance claims about interfaces should be measured with `tsc --extendedDiagnostics` before they drive decisions.

**45. [Senior] What are the downsides of heavy type-level programming?**
Slow compile and editor responses, unreadable errors, and hard maintenance. Prefer simple types and runtime schemas. Use advanced types where they remove a whole class of bugs.

**46. [Senior] How do you publish a library with correct types for ESM and CJS?**
Build both formats, set `declaration: true`, and use an `exports` map with matching `types` per condition. Validate with a package-checking tool and test by importing from both module systems. Keep `skipLibCheck` in mind for consumers.

**47. [Senior] How do you speed up a slow TypeScript build?**
Measure with `--extendedDiagnostics` and `--generateTrace`. Use `isolatedModules` with a fast transpiler for dev, `tsc --noEmit` for checks, `incremental` builds, project references for monorepos, and `skipLibCheck`. Simplify huge unions and deep conditional types.

**48. [Senior] How do you decide between exceptions and Result at module boundaries?**
Use Result for failures that are part of the contract and need branching. Throw for bugs. Keep one style per layer, and convert at the edge, for example in the HTTP layer.

**49. [Senior] How would you migrate a JavaScript service to TypeScript?**
Add `tsconfig` with `allowJs` and `checkJs` off, convert file by file starting at leaves, enable `strict` for new files first, then raise strictness globally. Add boundary validation early. Track `any` and `@ts-expect-error` counts.

**50. [Senior] Is `skipLibCheck: true` acceptable?**
It is a common trade-off. It speeds builds and avoids conflicts in third-party `.d.ts` files, but hides problems in them. Acceptable for applications; for a published library, also check your own generated declarations.

**51. [Senior] How do you test types?**
Use `// @ts-expect-error` lines in tests, type-level helpers such as `expectTypeOf` in Vitest (verify in the docs), and `tsc --noEmit` in CI over test files.

**52. [Senior] What is a risk of type predicates and casts in a team?**
They are promises the compiler cannot verify. A wrong one spreads false safety. Keep them small, centralised and tested, and prefer schema validation.

---

## 20. Glossary

| Term | Plain meaning |
|---|---|
| Type erasure | The compiler removes types, so only JavaScript runs |
| Structural typing | Types match by shape, not by name |
| Inference | The compiler works out a type from the code |
| Annotation | A type you write by hand |
| Cast (type assertion) | Telling the compiler a value has a type, with no check |
| Strict mode | A group of compiler options that catch more errors |
| Union | A type that can be one of several types |
| Intersection | A type that has all members of several types |
| Narrowing | Using a check so the compiler knows a more exact type |
| Type guard | A check or function that narrows a type |
| Type predicate | A function return type `x is T` that narrows |
| Discriminant | A shared field with a unique literal value per union member |
| Exhaustive check | A check that every union member is handled |
| `never` | The type with no values; used to prove nothing is left |
| `unknown` | A value of any type that you must check before use |
| Generic | A type or function with a type placeholder |
| Constraint | A limit on what a generic type can be (`extends`) |
| Conditional type | A type that chooses by a condition |
| Mapped type | A type made by looping over the keys of another |
| Utility type | A built-in helper type, such as `Partial` |
| Declaration merging | Two declarations with one name become one |
| Declaration file | A `.d.ts` file with types only |
| Ambient declaration | A type declaration for code that exists elsewhere |
| Branded type | A type with a hidden label to keep similar types apart |
| Nominal typing | Types match by name, not shape |
| Runtime validation | Checking real data while the program runs |
| Schema | A description of what valid data looks like |
| Result type | A return value that is either success or a typed error |
| ESM | ECMAScript modules; the `import` and `export` system |
| CommonJS | The older `require` module system in Node |
| Type stripping | Removing type syntax so Node can run a `.ts` file |
| Dead-letter queue | A place for messages that could not be processed |

---

## 21. References

Official documentation checked on 2026-11-06 (pages loaded unless noted):

- TypeScript Handbook, Everyday Types: https://www.typescriptlang.org/docs/handbook/2/everyday-types.html
- TypeScript Handbook, Narrowing: https://www.typescriptlang.org/docs/handbook/2/narrowing.html
- TypeScript Handbook, Generics: https://www.typescriptlang.org/docs/handbook/2/generics.html
- TypeScript Handbook, Utility Types: https://www.typescriptlang.org/docs/handbook/utility-types.html
- TypeScript Handbook, Modules Reference: https://www.typescriptlang.org/docs/handbook/modules/reference.html
- TypeScript Handbook, Declaration Files Introduction: https://www.typescriptlang.org/docs/handbook/declaration-files/introduction.html
- TypeScript 4.9 release notes (satisfies): https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-9.html
- TSConfig Reference: https://www.typescriptlang.org/tsconfig/
- TypeScript Playground: https://www.typescriptlang.org/play
- Node.js, Running TypeScript natively: https://nodejs.org/api/typescript.html
- Zod documentation, basics: https://zod.dev/basics
- Zod documentation, API: https://zod.dev/api
- typescript-eslint, Getting started: https://typescript-eslint.io/getting-started/
- typescript-eslint, Typed linting: https://typescript-eslint.io/getting-started/typed-linting/
- Vitest, Getting started: https://vitest.dev/guide/
- DefinitelyTyped repository (source of `@types` packages): https://github.com/DefinitelyTyped/DefinitelyTyped

Not verified in this review (confirm before relying on them): the tsx website (https://tsx.is/, fetch failed with a certificate error), Zod `.brand()`, `z.prettifyError` and `z.treeifyError` signatures, Zod 4 error customisation option names, Vitest `expectTypeOf`, the `@arethetypeswrong/cli` tool, `neverthrow`, `import.meta.dirname` version range, and the interface performance claim in chapter 3.
