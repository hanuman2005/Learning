# React for Backend-Leaning Full-Stack Engineers

Accuracy status: Reviewed against official documentation on 2026-11-06; items still unverified are marked.

A working guide to React, Vite and Tailwind for people who already build APIs and now need to build, review and debug the client that consumes them. It teaches the mental model first, so you can reason about why a component renders, why an effect runs and why a request fires twice, instead of memorising recipes.

**Audience:** backend-leaning developers who know HTTP, JSON APIs, authentication and databases, and want production-level React skill.
**Prerequisites:** solid JavaScript (closures, `async`/`await`, modules, array methods, destructuring) and basic HTML and CSS. TypeScript helps, and the snippets use it lightly.
**Scope:** client-side React with function components and hooks, built with Vite, styled with Tailwind, fetching data from a REST or JSON API. Server rendering and frameworks are covered conceptually only. Class components are legacy and mentioned only so you can read old code. React Native is out of scope.
**Note on accuracy:** the core model (rendering, state, hooks, effects) is stable. React 19 added Actions, `useActionState`, `useOptimistic`, `use` and `ref` as a prop. The React Compiler is a separate, optional tool. Library APIs change between major versions (TanStack Query, React Router, Tailwind, Vite). For version-dependent features, confirm exact signatures in the official docs listed in chapter 21.

## How to use this guide

Each chapter has four parts:

1. **Understand**: the concepts and the reasoning behind them.
2. **Tools to master**: what to learn to use fluently, and the level to reach.
3. **Common mistakes**: what goes wrong in practice.
4. **Self-check**: questions you should be able to answer without notes.

Chapter 19 lists hands-on labs. Chapter 20 collects interview questions, each with a model answer. Chapter 21 lists primary references.

**What "good enough" means for a backend-leaning full-stack role:** you can build a typed, accessible UI against a real API, handle loading, error and empty states, fetch and cache data correctly, implement login without introducing a security hole, find why something re-renders, and explain your choices. Chapters 1, 3, 5, 7 and 10 matter most.

## Contents

1. The rendering model
2. JSX, components and props
3. State and immutability
4. The core hooks
5. Effects: synchronisation, not lifecycle
6. State management
7. Data fetching and TanStack Query
8. Forms and validation
9. Routing
10. Authentication in single-page apps
11. Performance
12. Error boundaries and Suspense
13. Accessibility basics
14. Testing
15. Vite
16. Tailwind CSS
17. Server rendering and frameworks
18. Common pitfalls
19. Practice labs
20. Interview questions
21. References
22. Glossary

---

## 1. The rendering model

**In plain words:** React draws the screen from your data. When the data changes, React runs your component again and updates only what changed. Knowing these two steps explains most bugs.

### Understand

**UI as a function of state.** React asks you to describe what the screen should look like for the current data, not how to mutate the screen from one state to the next. A component is a function from props and state to a description of UI. If you give it the same inputs, it returns the same description. This is the same idea as a pure function in a backend service: no hidden writes, so the result is predictable and testable.

**Render versus commit.** These are two distinct phases and confusing them causes most early bugs.

- **Render phase:** React calls your component functions to compute the new UI description (a tree of "elements", plain objects). This must be pure. It does no DOM work. React may call a component more than once, pause it or discard the result.
- **Commit phase:** React compares the new tree to the previous one, applies the minimum DOM changes, then runs effects. Side effects belong here (in effects and event handlers), not in the render body.

"Rendering" in React does not mean "painting pixels". A component can render and produce an identical result, in which case the commit changes nothing.

**Reconciliation (finding what changed between two renders) and keys.** React keeps a tree of elements from the previous render and compares it to the new one. Two rules drive the diff:

1. Elements of different types at the same position are replaced. The old subtree is unmounted and its state is lost.
2. Elements of the same type are updated in place and keep their state.

For lists, position alone is not enough when items move or are inserted. A **key** gives each item a stable identity so React matches old and new items by identity instead of by index. Keys must be stable and unique among siblings, normally a database id. An array index is only acceptable for static lists that never reorder, insert or delete. Changing a component's key deliberately is also the supported way to reset its state.

**State updates are asynchronous and batched.** Calling a state setter does not change the variable you already hold. It schedules a new render. In that next render, the component function runs again and `useState` returns the new value. React batches several setter calls made in the same event into one render, including calls inside promises and timeouts in React 18 and later. The practical consequence: within one render, state is a **snapshot**. Every closure created during that render, including event handlers and effects, sees that render's values.

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  function addThree() {
    setCount(count + 1); // count is 0 in this snapshot
    setCount(count + 1); // still 0
    setCount(count + 1); // result: 1, not 3
  }
  function addThreeCorrectly() {
    setCount((c) => c + 1); // updater functions queue and chain
    setCount((c) => c + 1);
    setCount((c) => c + 1); // result: 3
  }
  return <button onClick={addThreeCorrectly}>{count}</button>;
}
```

Use the updater form whenever the next state depends on the previous state.

**StrictMode.** In development, `<StrictMode>` renders components twice and runs each effect's setup, cleanup, setup sequence once on mount. Production does not do this. The point is to expose impure render functions and effects with missing cleanup. If your component breaks when rendered twice, it has a bug that would surface later under concurrent rendering or fast navigation. Treat a double network request in development as a signal to check cleanup, not as something to disable.

**Concurrent rendering.** React can interrupt, resume and prioritise renders so urgent updates (typing) are not blocked by slow ones (filtering a large list). You rarely call this directly. It is why render must be pure and why `useTransition` and `useDeferredValue` exist (chapter 4).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Seeing what rendered and why | React DevTools Components tab (highlight updates, "why did this render" in Profiler) | Explain each re-render of a component |
| Verifying purity | `<StrictMode>` in development | Understand every double invocation |
| Keys | the `key` prop, ESLint `react/jsx-key` rule | Choose a correct key for any list |
| Snapshot semantics | updater functions `setX(prev => ...)` | Predict state after several calls in one handler |

### Common mistakes

- Reading state right after calling the setter and expecting the new value.
- Using array index as key in a list that can reorder, which corrupts input state and animations.
- Doing side effects (fetching, writing to `localStorage`, mutating a variable outside the component) in the render body.
- Generating keys with `Math.random()` on every render, which remounts the whole list each time.
- Defining a component inside another component, which creates a new component type every render and destroys its state.

### Self-check

- What is the difference between rendering and committing? Which one may run twice?
- Why does `setCount(count + 1)` three times add only one?
- Why does a list with index keys break when you delete the first row?
- What does StrictMode do in development, and why does it not run in production?

---

## 2. JSX, components and props

**In plain words:** A component is a function that returns a piece of screen. JSX is the HTML-like text you write inside it. Parents give data to children through props.

### Understand

**JSX is syntax for function calls.** `<Button size="lg">Save</Button>` compiles to a call that creates an element object `{ type: Button, props: { size: "lg", children: "Save" } }`. It is not HTML and not a template string. Consequences: attribute names follow the DOM property style (`className`, `htmlFor`, `onClick`), expressions go in braces, and there must be a single root (use a fragment `<>...</>` to avoid an extra DOM node).

**Expressions, not statements.** Inside braces you write expressions. Use ternaries, `&&` and `map` for conditional and list rendering. Beware `{count && <X />}`: when `count` is `0`, React renders the number `0`. Write `{count > 0 && <X />}`.

**Escaping.** JSX escapes interpolated strings, which is the main defence against cross-site scripting (XSS, an attacker's script running in your page). The escape hatch `dangerouslySetInnerHTML` bypasses it. Only use it with content you sanitised (for example with DOMPurify), never with raw API data.

**Components are functions with capitalised names.** Lowercase tags (`div`) are DOM elements, capitalised tags are components. A component receives one argument, `props`, and returns JSX, `null`, a string or an array.

**Props flow down, events flow up.** Data moves from parent to child through props. A child cannot modify its props. To change parent state, the parent passes down a callback, and the child calls it. This one-way flow is what makes data movement traceable.

```tsx
type UserCardProps = {
  user: { id: string; name: string };
  onSelect: (id: string) => void;
};

function UserCard({ user, onSelect }: UserCardProps) {
  return (
    <button type="button" onClick={() => onSelect(user.id)}>
      {user.name}
    </button>
  );
}
```

**`children` and composition.** `children` is an ordinary prop containing nested JSX. Prefer composition (a `Card` that accepts `children`) over configuration flags and inheritance. Composition also fixes much prop drilling (chapter 6): pass the already-built element down instead of the data it needs.

**Refs as props.** In React 19, function components can receive `ref` as a normal prop, so `forwardRef` is no longer needed for new code. The React 19 release post says `forwardRef` will be deprecated in a future version. Class components do not receive `ref` as a prop.

**Controlled design.** A component is **controlled** when its parent owns the value and passes `value` and `onChange`, and **uncontrolled** when it keeps its own state. Choose deliberately (chapter 8).

**TypeScript.** Type props explicitly. For API data, derive types from a schema (chapter 8 and 7) so the compiler and the runtime agree. Types are erased at runtime, so they do not validate what the server actually sends.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Typing components | TypeScript, `React.ReactNode`, `ComponentProps<'button'>` | Type props and extend native element props |
| Linting | ESLint with `eslint-plugin-react-hooks`, `jsx-a11y` | Zero warnings in CI |
| Inspecting props | React DevTools Components tab | Read live props and state of any component |
| Formatting | Prettier | Consistent JSX formatting |

### Common mistakes

- `{items.length && ...}` rendering a stray `0`.
- Spreading unknown props onto DOM elements and producing invalid attributes.
- Passing a new object or function literal as a prop to a memoised child and defeating the memoisation (chapter 11).
- Trusting TypeScript types as if they validated an API response.
- Using `dangerouslySetInnerHTML` with unsanitised data.

### Self-check

- What does `<Foo a={1}>hi</Foo>` become after compilation?
- Why can a child not change its props, and how does it request a change?
- Why does `{list.length && <List />}` sometimes print `0`?

---

## 3. State and immutability

**In plain words:** State is data that a component remembers between renders. You must never change state in place. You make a new copy and give it to React.

### Understand

**What counts as state.** State is data that changes over time and that the UI must reflect. If a value can be computed from existing props or state, it is not state, it is derived data. Keep state **minimal** and compute the rest during render.

**Treat state as immutable.** React decides whether state changed by comparing the old and new value with `Object.is`. If you mutate an object or array in place and pass the same reference to the setter, React sees "no change" and may skip the render. Always create a new object or array.

```tsx
// Wrong: mutates, same reference
todos.push(newTodo);
setTodos(todos);

// Right: new array
setTodos([...todos, newTodo]);
setTodos(todos.filter((t) => t.id !== id));
setTodos(todos.map((t) => (t.id === id ? { ...t, done: true } : t)));

// Nested object: copy every level you change
setUser({ ...user, address: { ...user.address, city: "Pune" } });
```

Immutability also gives you cheap change detection, safe memoisation and undo or time-travel debugging. For deeply nested state, libraries such as Immer let you write mutating-looking code that produces immutable updates. Better still, flatten state.

**Lazy initial state.** `useState(expensive())` calls `expensive()` on every render. Pass a function instead: `useState(() => expensive())`.

**Where to put state.** Place state in the lowest component that needs it, and lift it to the nearest common parent when two siblings need it (chapter 6).

**Derive, do not duplicate.** Storing `fullName` in state when you already store `firstName` and `lastName` creates two sources of truth that drift apart. The same applies to a filtered list: store the list and the filter, compute the filtered result.

**Reset state with a key.** To reset a form when the selected record changes, render it as `<Editor key={recordId} />`. A new key creates a fresh instance with fresh state, which is cleaner than an effect that copies props into state.

**Reducers for related state.** When several values change together or transitions have names, a reducer centralises the logic (chapter 4).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Immutable updates | spread, `map`, `filter`, `toSpliced`/`toSorted`, `structuredClone` | Update nested state without mutation |
| Nested updates | Immer (`use-immer`, or Zustand/Redux Toolkit built-in use) | Know when it helps |
| Inspecting state | React DevTools, state shown on component | Verify state shape live |

### Common mistakes

- `arr.sort()` or `arr.reverse()` on state (both mutate). Copy first, or use `toSorted`.
- Copying props into state (`useState(props.value)`), so later prop changes are ignored.
- Storing derived values in state and syncing them with effects.
- Shallow-copying only the top level of a nested object.

### Self-check

- Why does mutating an array in state and calling the setter sometimes not re-render?
- How do you update one field of an object inside an array in state?
- When should two pieces of data be one state variable, and when two?

---

## 4. The core hooks

**In plain words:** Hooks are special functions that give a component memory and other React features. They follow a few strict rules. This chapter shows the main hooks and when to use each.

### Understand

**What a hook is.** A hook is a function that lets a component use React features (state, context, refs, effects) across renders. React tracks hooks by **call order**, which is why the **rules of hooks** exist: call hooks only at the top level of a component or custom hook, never inside conditions, loops or nested functions, and only from React functions. The ESLint plugin enforces this.

**`useState`.** Returns `[value, setValue]`. The setter accepts a value or an updater function. State persists between renders and is private to the component instance.

**`useReducer`.** `const [state, dispatch] = useReducer(reducer, initial)`. A reducer is a pure function `(state, action) => newState`. Prefer it when state has several fields that change together, when the next state depends on named events, or when you want the transition logic testable in isolation. It maps well to a backend mindset of commands and state machines.

```tsx
type Action = { type: "add"; item: string } | { type: "clear" };

function reducer(state: string[], action: Action): string[] {
  switch (action.type) {
    case "add":
      return [...state, action.item];
    case "clear":
      return [];
  }
}
```

**`useRef`.** Returns a mutable object `{ current }` that persists across renders and does **not** trigger a render when changed. Two uses: holding a DOM node (`ref={inputRef}`) and holding a mutable value that is not part of the UI (timer ids, the latest callback, an `AbortController`). Do not read or write `ref.current` during render, except for lazy initialisation.

**`useMemo` and `useCallback`.** `useMemo(fn, deps)` caches a computed value. `useCallback(fn, deps)` caches a function identity. They do not make code faster by existing: they cost memory and comparison work. They help in three cases:

1. The computation is genuinely expensive (measure first).
2. The value is passed to a child wrapped in `React.memo`, and a stable reference prevents its re-render.
3. The value is a dependency of another hook, and a stable identity prevents an effect from re-running.

Otherwise skip them. The React Compiler (stable per react.dev, works with React 17, 18 and 19) can insert much of this memoisation automatically. It is optional, so keep writing code that does not depend on it.

**`useContext`.** Reads the value of the nearest matching Provider above in the tree. It avoids passing props through every level. Limits are covered in chapter 6.

**`useId`.** Generates a stable unique id that matches between server and client, used to connect `<label htmlFor>` to an input or `aria-describedby` to a message. Do not use it for list keys.

**`useTransition` and `useDeferredValue` (conceptual).** Both mark work as lower priority so urgent input stays responsive.
- `useTransition` returns `[isPending, startTransition]`. State updates inside `startTransition` are non-urgent: React may keep showing the old UI while preparing the new one, and `isPending` lets you show a subtle indicator.
- `useDeferredValue(value)` returns a copy of a value that lags behind during urgent updates. Use it when you cannot wrap the setter, for example a search input driving an expensive list.

Neither makes slow code fast. They keep the interface responsive while slow code runs. Fix the slow work itself first.

**React 19 additions (verified on react.dev).**
- **Actions:** async functions used in transitions, especially with forms (`<form action={fn}>`), which manage pending state and errors for you.
- **`useActionState(reducerAction, initialState)`:** returns `[state, dispatchAction, isPending]`. The action receives the previous state and a payload (the `FormData` when used as a form action).
- **`useOptimistic(value, reducer?)`:** returns `[optimisticState, setOptimistic]`. Call the setter inside an Action. While the Action runs you see the optimistic value. When it finishes, or throws, the UI shows the real `value`.
- **`use`:** reads a promise or context during render and may be called conditionally. The promise must be cached (not created on each render). Do not wrap `use` in try/catch. Use an error boundary.

These are most valuable when your framework or router integrates them. In a client-only SPA with TanStack Query, you may use Query's own mutation APIs instead.

**Custom hooks.** A custom hook is a function whose name starts with `use` that calls other hooks. It shares **logic**, not state: each call gets its own state. Extract one when the same stateful logic appears twice or when a component mixes concerns.

```tsx
function useDebouncedValue<T>(value: T, delayMs: number): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delayMs);
    return () => clearTimeout(id);
  }, [value, delayMs]);
  return debounced;
}
```

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Hook rules | `eslint-plugin-react-hooks` (`rules-of-hooks`, `exhaustive-deps`) | Fix every warning by understanding it, not suppressing it |
| Reducers | `useReducer`, discriminated unions in TypeScript | Model a multi-step flow as a reducer |
| Refs | `useRef`, `useImperativeHandle` (rarely) | Hold DOM nodes and non-UI mutable values |
| Memoisation | `useMemo`, `useCallback`, `React.memo`, Profiler | Prove with a measurement that it helped |
| Priority | `useTransition`, `useDeferredValue` | Keep a text input responsive over a heavy list |
| Reuse | custom hooks, tests via `renderHook` | Extract and test a hook |

### Common mistakes

- Calling a hook conditionally or after an early `return`.
- Wrapping everything in `useMemo` and `useCallback` "for performance" without measuring.
- Mutating `ref.current` during render and creating unpredictable output.
- Expecting a custom hook to share state between components. It shares code only. Share state with context or a store.
- Writing a reducer that mutates state or has side effects.

### Self-check

- Why can hooks not be called inside an `if`?
- When does `useCallback` actually help, and when is it pure overhead?
- What is the difference between `useRef` and `useState` for holding a value?
- What problem does `useTransition` solve, and what does it not solve?

---

## 5. Effects: synchronisation, not lifecycle

**In plain words:** An effect connects your component to something outside React, such as a server or a timer. It runs again when the values it uses change. Many tasks do not need an effect at all.

### Understand

**The mental model.** An effect is not "run on mount" or "run on update". An effect describes how to **synchronise** your component with something outside React (a network connection, a browser API, a subscription, a non-React widget) for a given set of values. React runs the effect after commit, and re-runs it when the values it reads have changed, first running the previous cleanup.

```tsx
useEffect(() => {
  const conn = createConnection(roomId); // setup: start synchronising
  conn.connect();
  return () => conn.disconnect();        // cleanup: stop synchronising
}, [roomId]);                            // re-synchronise when roomId changes
```

Ask "what outside thing am I keeping in sync, and with which values?" not "when should this run?".

**Dependency array semantics.**
- No array: runs after every render.
- `[]`: runs once after the first commit (and again in StrictMode development), with cleanup on unmount.
- `[a, b]`: runs after the first commit and after any commit where `a` or `b` changed by `Object.is`.

The array is not a choice of when to run. It is a **declaration of what the effect reads**. Every reactive value (props, state, anything derived from them, functions defined in the component) used inside the effect must be listed. The linter rule `exhaustive-deps` checks this. Suppressing it hides a stale-closure bug (chapter 18). Fix it by moving the function inside the effect, moving constants outside the component, using an updater function, or restructuring.

**Cleanup.** The returned function runs before the next run of the effect and on unmount. Use it for unsubscribing, clearing timers, closing sockets and aborting requests. An effect without necessary cleanup leaks and, under StrictMode, reveals itself as duplicated work.

**You might not need an effect.** Effects are an escape hatch for external systems. Most uses by beginners are unnecessary:

| Situation | Instead of an effect |
|---|---|
| Derive a value from props or state | Compute it during render (use `useMemo` only if costly) |
| Reset state when a prop changes | Give the component a `key` |
| Respond to a user action (submit, click) | Do the work in the event handler |
| Adjust state based on a prop change | Compute during render, or restructure state |
| Chain effects that set state to trigger other effects | Compute in one place or one event handler |
| Fetch data | A data-fetching library (chapter 7) |
| Subscribe to an external store | `useSyncExternalStore` or the store library's hook |

Rule of thumb: if the cause is a user event, handle it in the event handler. If the cause is "the component is on screen", an effect may be right.

**Fetching in effects: the race condition.** A hand-written fetch effect has real failure modes. If `id` changes from 1 to 2 quickly, the response for 1 may arrive after the response for 2, and the UI shows stale data. The fix is cleanup that ignores or cancels the outdated request.

```tsx
useEffect(() => {
  const controller = new AbortController();
  fetch(`/api/users/${id}`, { signal: controller.signal })
    .then((r) => {
      if (!r.ok) throw new Error(`HTTP ${r.status}`);
      return r.json();
    })
    .then(setUser)
    .catch((err) => {
      if (err.name !== "AbortError") setError(err);
    });
  return () => controller.abort();
}, [id]);
```

Even this version omits loading state, caching, deduplication, retries, refetch on focus and shared data between components. That is why chapter 7 recommends a library. `fetch` does not reject on HTTP error statuses, so you must check `response.ok`.

**Layout effects.** `useLayoutEffect` runs synchronously after DOM mutation and before paint. Use it only to measure the DOM and adjust before the user sees a frame. It is not for data work.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Dependency correctness | `eslint-plugin-react-hooks` `exhaustive-deps` | Never suppress it without a written reason |
| Cancellation | `AbortController`, `AbortSignal`, effect cleanup | Write a race-free fetch effect from memory |
| External subscriptions | `useSyncExternalStore` | Know it exists for stores and browser APIs |
| Observing effects | StrictMode, React DevTools Profiler, console logging in setup and cleanup | Trace which effect ran and why |

### Common mistakes

- Treating the dependency array as a performance knob and omitting values.
- Infinite loops: an effect sets state that is in its own dependencies, or depends on an object or function recreated every render.
- Using an effect to derive state or to respond to a click.
- Fetching with no cleanup, producing out-of-order responses and updates after unmount.
- Ignoring `response.ok` so a 500 is treated as data.
- Disabling StrictMode to hide a double effect.

### Self-check

- What is the correct way to describe what `useEffect(..., [])` does?
- A request fires twice in development. Is this a bug? How would you tell?
- Give three cases where an effect is the wrong tool.
- Show how a fetch effect avoids showing an outdated response.

---

## 6. State management

**In plain words:** Decide where each piece of data lives. Data from the server and data owned by the screen need different tools. Start with the simplest option and add more only when needed.

### Understand

**Two kinds of state.** The most useful distinction:

- **Client state** is owned by the UI: form drafts, open modals, selected tab, theme. It is synchronous and local.
- **Server state** is owned by your backend and merely **cached** in the browser: lists, profiles, orders. It is asynchronous, can become stale, can be changed by other users and needs invalidation. Treating server state as client state (copying API data into `useState` or a global store) is the root of most synchronisation bugs. Use a server-state library (chapter 7).

After moving server state out, the remaining client state is usually small and needs no global store.

**Escalation ladder.** Use the simplest option that works:

1. Local `useState` or `useReducer`.
2. **Lift state up** to the nearest common parent and pass props.
3. **Composition**: pass elements as `children` or props so intermediate components do not need the data.
4. **Context** for values that many distant components read and that change rarely.
5. An **external store** for client state that is shared, frequently updated, or needed outside React.

**Prop drilling** is passing props through components that do not use them. A little is fine and keeps data flow explicit. It becomes a problem when many layers forward unrelated props. Try composition before context.

**Context limits.** When a Provider's value changes, **every** consumer re-renders, even if it reads only part of the value. Context has no built-in selector. Consequences:
- Good for low-frequency values: theme, locale, current user, a dependency such as an API client.
- Poor for high-frequency or large state (for example a form with many fields or a list being edited).
- Mitigations: split contexts by change frequency, memoise the value object, or use a store with selectors.

```tsx
const ThemeContext = createContext<"light" | "dark">("light");
// Consumer
const theme = useContext(ThemeContext);
```

**External stores (conceptual).**
- **Zustand:** a small store with a hook. Components subscribe to a selected slice, so only components whose selected value changed re-render. Minimal boilerplate, good for UI state.
- **Redux Toolkit:** the standard Redux toolset. A single store, actions, reducers (with Immer), middleware and strong DevTools with time travel. More ceremony, valuable on large teams that want strict structure and traceability. RTK Query, part of the toolkit, covers server state.
- **Others:** Jotai (atoms), MobX (observables), Context plus reducer for small cases.

Choose by problem, not fashion. Many applications need only server-state caching plus local state plus a tiny store or context.

**State in the URL.** Filters, pagination, selected tab and search text that users should be able to bookmark or share belong in the URL (search params), not in component state. The URL is state that survives reload and works with the back button.

**Persistence.** `localStorage` suits non-sensitive preferences. Never store secrets there (chapter 10).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Local and lifted state | `useState`, `useReducer` | Default choice, no library |
| Shared low-frequency values | `createContext`, `useContext` | Use for theme, locale, current user |
| Client store | Zustand, Redux Toolkit | Explain selectors and when each fits |
| Server state | TanStack Query, RTK Query | Keep API data out of client stores |
| URL state | router search params | Make filters and pages shareable |
| Observing | Redux DevTools, React DevTools | Trace an update to its source |

### Common mistakes

- Copying fetched data into global state, then writing sync code to keep it fresh.
- Putting frequently changing values in a single large context.
- Reaching for a global store for state that only two nearby components share.
- Keeping shareable filters only in memory so links and refresh lose them.
- Storing tokens or personal data in a persisted client store.

### Self-check

- What is the difference between server state and client state, and why does it matter?
- Why can a context with a large changing object cause many re-renders?
- When would you choose Zustand over context? Redux Toolkit over Zustand?
- Where should a table's sort and page live?

---

## 7. Data fetching and TanStack Query

**In plain words:** Loading data from an API has many hard details, such as caching and retries. TanStack Query handles them for you. You describe what data you need, and it manages the rest.

### Understand

**Why not hand-roll it.** To do fetching properly in components you would need loading and error state, cancellation, race protection, caching, deduplication of identical requests, retries with backoff, background refresh, pagination, invalidation after writes and cache sharing across components. That is a client-side data layer. TanStack Query (formerly React Query) is that layer. Your backend instincts apply: it is a read-through cache with TTL-like staleness, explicit invalidation and request coalescing. The API below is for TanStack Query v5. Confirm against the docs for your installed version.

**Queries.** A query is identified by a **query key** (an array) and loads data with a **query function** that returns a promise and throws on failure.

```tsx
const { data, isPending, isError, error } = useQuery({
  queryKey: ["users", userId],
  queryFn: () => api.getUser(userId), // must throw on non-2xx
});
```

The key is like a cache key: it should include every input the function depends on. When the key changes, Query treats it as a different cache entry and fetches it. Components that use the same key share one entry and one request.

**Staleness versus garbage collection.** Two separate timers:
- `staleTime`: how long fetched data is considered fresh. Default is 0, so data is stale immediately and refetched on triggers (new mount, window focus, reconnect). Raise it for data that changes slowly, to cut requests.
- `gcTime`: how long an unused entry stays in cache before removal (default 5 minutes).

Stale does not mean absent. Stale data is shown immediately while a background refetch updates it (stale-while-revalidate). Distinguish `isPending` (no data yet) from `isFetching` (any request in flight, including background).

**Mutations and invalidation.** Writes use `useMutation`. After a successful write, mark affected queries stale so they refetch.

```tsx
const queryClient = useQueryClient();
const mutation = useMutation({
  mutationFn: (input: NewTodo) => api.createTodo(input),
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ["todos"] });
  },
});
// mutation.mutate({ title: "Write guide" });
```

Invalidating by key prefix is the cache-invalidation story: the hard problem becomes "which keys does this write affect?" Design key hierarchies (`["todos"]`, `["todos", id]`, `["todos", { status }]`) with that in mind.

**Optimistic updates.** Update the UI before the server confirms, then roll back if it fails. TanStack Query v5 documents two approaches:
- **Via the UI:** render the pending item from the mutation's `variables` and `isPending`. No cache writes and no rollback code. Best when only one place shows the result. Use `onSettled` to call `invalidateQueries`.
- **Via the cache:** `onMutate` (call `cancelQueries`, snapshot with `getQueryData`, write with `setQueryData`, return the snapshot), `onError` (restore the snapshot) and `onSettled` (invalidate to reconcile with the server). Best when several components show the same data.

React 19's `useOptimistic` is a third route, outside Query. Optimism is only appropriate when failure is rare and rollback is understandable.

**Other behaviour to know.**
- **Retries:** failed queries retry a few times with backoff by default. Disable for non-retryable errors such as 401, 403 and 404, otherwise you slow down error display.
- **Dependent queries:** the `enabled` option delays a query until its inputs exist.
- **Pagination and infinite lists:** `placeholderData` to keep the previous page visible, `useInfiniteQuery` for cursor-based feeds. Your cursor or offset contract comes from the API.
- **Prefetching:** `queryClient.prefetchQuery` on hover or in a route loader.
- **Error handling:** an error from `queryFn` lands in `error`. Centralise 401 handling in the API client, not in every query.
- **Types:** validate responses at the boundary with a schema (zod) inside the API client, so a changed backend fails loudly at one point instead of corrupting the UI.

**Thin API client.** Keep one module that attaches credentials, handles base URL, parses JSON, checks `response.ok`, maps errors to a typed error class and handles refresh (chapter 10). Query functions call it. Generating the client from an OpenAPI spec is a good fit for a backend-owned contract.

**Query and server rendering.** With frameworks, Query data can be prefetched on the server and hydrated on the client. That is framework-specific. See the docs.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Reads | `useQuery`, query keys, `staleTime`, `gcTime`, `enabled` | Design keys and freshness for a given endpoint |
| Writes | `useMutation`, `invalidateQueries`, `setQueryData` | Keep cache consistent after create, update and delete |
| Optimistic UI | `onMutate`/`onError`/`onSettled` | Implement it with rollback, and know its risks |
| Lists | `useInfiniteQuery`, `placeholderData` | Paginate smoothly |
| Debugging | TanStack Query Devtools | Read cache entries, staleness and fetch status |
| Boundary validation | zod, OpenAPI client generators | Fail fast on contract drift |

### Common mistakes

- Omitting a variable from the query key, so different inputs share a cache entry.
- A `queryFn` that does not throw on HTTP errors, so errors look like data.
- Copying query data into `useState` and then editing the copy.
- Leaving `staleTime` at 0 for data that rarely changes and flooding the API.
- Not invalidating after a mutation, so lists show old data.
- Retrying 4xx errors three times before showing a message.
- Using Query for purely client state such as a modal toggle.

### Self-check

- Explain `staleTime` versus `gcTime` in terms of a server-side cache.
- What goes in a query key and why?
- After creating an item, how does the list update? What are two ways?
- Why is a fetch inside `useEffect` usually inferior to `useQuery`?
- What happens on rollback in an optimistic update when the server returns 409?

---

## 8. Forms and validation

**In plain words:** Forms collect input from users. Check the input in the browser for fast feedback, and check it again on the server for safety. Libraries such as react-hook-form and zod reduce the work.

### Understand

**Controlled versus uncontrolled.**
- **Controlled:** React state is the source of truth. The input has `value={state}` and `onChange` updates it. Every keystroke renders the component. Easy to validate live and derive UI, but costly in big forms.
- **Uncontrolled:** the DOM holds the value. You read it on submit with a ref or `FormData`. Cheap, with fewer renders, and good for simple forms.

```tsx
function onSubmit(e: React.FormEvent<HTMLFormElement>) {
  e.preventDefault();
  const data = new FormData(e.currentTarget);
  const email = String(data.get("email") ?? "");
  // validate, then call the API
}
```

**Native first.** Use real `<form>`, `<label>`, `<input type="email" required>` and a `type="submit"` button. You get Enter-to-submit, browser validation, autofill and accessibility for free. React 19 form Actions build on the native form model. Confirm details in react.dev.

**react-hook-form.** A library that registers inputs as uncontrolled by default and tracks state in a form object, so typing in one field does not re-render the whole form. It gives `register`, `handleSubmit`, `formState.errors`, field arrays, dirty and touched tracking, and resolvers for schema validators. Use `Controller` to wrap controlled third-party components.

**zod.** A TypeScript-first schema library. One schema serves three purposes: runtime validation, a static type via inference, and an error message source.

```tsx
const schema = z.object({
  email: z.string().email(),
  age: z.coerce.number().int().min(18),
});
type FormValues = z.infer<typeof schema>;

const { register, handleSubmit, formState } = useForm<FormValues>({
  resolver: zodResolver(schema),
});
```

Zod 4 documents `z.email()` as the top-level email validator. `z.string().email()` is the older form, so check which one your installed version prefers. The React Hook Form docs say `zodResolver` (from `@hookform/resolvers/zod`) needs `@hookform/resolvers` 3.10 or newer with Zod 3.25 or Zod 4, and `moduleResolution` set to `bundler`, `node16` or `nodenext`. With `z.coerce`, the input type is `unknown`, which can cause type mismatches with `useForm<FormValues>` in some versions. I did not verify that point.

**Validation is layered.** Client validation is a **usability** feature: it gives fast feedback. It is not security. The server must validate everything again, because anyone can send requests without your UI. Share the schema or generate one from the API contract to avoid drift. Map server validation errors (for example a 422 with field errors) back onto fields.

**Submission states.** Disable or guard the submit button while pending to prevent double submit, show server errors in a summary and per field, and keep the user's input on failure. Pair submit with `useMutation` (chapter 7).

**File upload.** Use `FormData` and let the browser set the multipart boundary. Do not set `Content-Type` manually for `FormData`.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Native forms | `<form>`, `FormData`, constraint validation | Build a working form with no library |
| Form state | react-hook-form (`register`, `Controller`, `handleSubmit`) | Build a multi-field form with minimal re-renders |
| Schemas | zod, `z.infer`, `zodResolver` | One schema for types and validation |
| Server errors | mapping 4xx field errors to `setError` | Show API errors on the right field |
| Accessibility | `<label>`, `aria-invalid`, `aria-describedby`, `useId` | Announce errors to assistive technology |

### Common mistakes

- Treating client validation as sufficient and trusting the request on the server.
- Controlled inputs for every field in a very large form, with sluggish typing.
- Forgetting `e.preventDefault()` so the page reloads.
- Switching an input between controlled and uncontrolled (value `undefined` then defined).
- Placeholder text instead of a label.
- Allowing double submits on slow networks.

### Self-check

- When is an uncontrolled form better than a controlled one?
- Why must the server validate when the client already did?
- What does `z.infer` give you and why is it useful against an API?
- How do you show a server-side field error returned as JSON?

---

## 9. Routing

**In plain words:** A router shows a different screen for each web address, without reloading the page. It also lets users bookmark and share links. Real security still happens on the server.

### Understand

**What a client router does.** A single-page application serves one HTML document, then the router maps the URL to a component tree and updates the address bar through the History API without a full page load. Navigation changes which components render, and the data they need is fetched through your API.

**React Router modes.** The docs describe three. **Declarative** (`<BrowserRouter>`, `<Routes>`, no loaders or actions), **Data** (`createBrowserRouter` with `<RouterProvider>`, loaders, actions, `useFetcher`) and **Framework** (Data mode plus a Vite plugin, `routes.ts`, route modules, SSR or static rendering). Check which one you use.

**React Router concepts:**
- **Routes and nesting:** a route tree where a parent route renders an `<Outlet />` for its child. Layouts (header, sidebar) are parent routes.
- **Dynamic segments:** `/users/:id`, read with `useParams`.
- **Navigation:** `<Link>` and `<NavLink>` for navigation, `useNavigate` for programmatic redirects after actions. Use real links so middle-click, copy-link and accessibility work.
- **Search params:** `useSearchParams` for filters and pagination (chapter 6).
- **Data mode:** `createBrowserRouter` lets routes declare **loaders** (fetch before render) and **actions** (handle form submissions), with automatic revalidation after actions. You can use these or keep fetching inside components with TanStack Query. If using Query with loaders, a common approach is a loader that prefetches into the Query cache. Decide on one data strategy per app.
- **Not-found and error routes:** a catch-all route for 404 and an `errorElement` for errors thrown in a route (chapter 12).

**Server must cooperate.** Deep links like `/users/42` hit your web server directly on refresh. It must serve `index.html` for unknown paths (history fallback), while API paths and static assets are routed separately. This is a frequent deployment bug.

**Code splitting by route.** Routes are the natural boundary for lazy loading (chapter 11).

**Route guards are UX, not security.** A "protected route" that redirects unauthenticated users prevents confusion. It does not protect data, because all JavaScript in the bundle is public and a user can edit state in DevTools. Authorisation must be enforced by the API on every request (chapter 10).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Route tree | `createBrowserRouter`, `<Outlet>`, nested layouts | Model an app with layouts and a 404 |
| Navigation | `<Link>`, `<NavLink>`, `useNavigate`, `<Navigate>` | Navigate with correct semantics |
| URL state | `useParams`, `useSearchParams` | Keep filters in the URL |
| Data | loaders, actions, or Query prefetching | Pick one pattern and apply it consistently |
| Deployment | SPA fallback in nginx, CDN or hosting rules | Make refresh on a deep link work |

### Common mistakes

- Using `<a href>` and causing full page reloads, or `onClick` navigation on a `<div>`.
- No history fallback on the server, so refresh on a deep link returns 404.
- Treating a route guard as authorisation.
- Putting everything in one giant route component.
- Keeping filters only in state and losing them on refresh and back navigation.

### Self-check

- Why does refreshing `/users/42` return 404 in production but work in development?
- What does a nested route with `<Outlet>` give you?
- Why is a protected route not a security control?

---

## 10. Authentication in single-page apps

**In plain words:** Users can read and change everything that runs in their browser. So the browser can show who is logged in, but only the server can enforce what they may do. This chapter explains how to store login credentials with fewer risks.

### Understand

**The central fact.** Everything delivered to the browser is visible and modifiable by the user. The client can **display** identity but cannot **enforce** it. Enforcement lives on the API. Design the SPA so that compromise of one piece (a stolen token, an XSS bug) limits damage.

**Where to keep credentials: the trade-offs.**

| Storage | Readable by JavaScript | Main risk | Notes |
|---|---|---|---|
| `localStorage` / `sessionStorage` | Yes | XSS steals the token and uses it anywhere, until expiry | Simple, but avoid for long-lived tokens |
| In-memory variable (access token) | Yes, but only the running page | XSS can still use it while the page lives. Lost on reload | Short-lived token, re-obtained via refresh |
| `httpOnly` cookie | No | CSRF (browser attaches it automatically) | Not readable by script, so XSS cannot exfiltrate it, though XSS can still make requests as the user |

Key points:
- **XSS** is script injection into your page. With XSS, an attacker running in your origin can call your API as the user regardless of storage. `httpOnly` stops token theft and reuse elsewhere, not in-page abuse. Prevent XSS: React escaping by default, no `dangerouslySetInnerHTML` with raw data, a strict Content-Security-Policy, careful third-party scripts, dependency hygiene.
- **CSRF** (cross-site request forgery) is a malicious site causing the browser to send an authenticated request using your cookie. Defences: `SameSite=Lax` or `Strict` cookies, CSRF tokens (synchroniser or double-submit) for state-changing requests, checking `Origin`/`Referer`, and requiring a custom header on JSON APIs. Bearer tokens in an `Authorization` header are not sent automatically, so they are not CSRF-prone.
- **Cookie flags:** `HttpOnly`, `Secure`, `SameSite`, a scoped `Path`, and a sensible `Max-Age`. Same-site deployment (the app and API under one site) makes cookies simpler and safer. Cross-site needs `SameSite=None; Secure` and CORS with credentials, which is harder to secure.

**A common robust pattern.**
1. Login sends credentials to the API over TLS.
2. The API returns a short-lived **access token** (kept in memory) and sets a long-lived **refresh token** in an `httpOnly`, `Secure`, `SameSite` cookie scoped to the refresh endpoint.
3. The client sends the access token in `Authorization: Bearer ...`.
4. On 401, the API client calls the refresh endpoint (cookie sent automatically), gets a new access token, and retries the original request once.
5. Refresh token **rotation** (each use issues a new one and invalidates the old) lets the server detect theft by reuse.
6. On reload, the app calls refresh to restore the session.

The alternative is a server-managed **session cookie** (opaque id, `httpOnly`) with CSRF protection. It is simpler, easy to revoke and avoids tokens in JavaScript. For first-party apps it is often the better choice. Pick by deployment: same-site first-party apps suit cookie sessions, and third-party API consumers suit bearer tokens.

**Handle concurrent 401s.** If five requests fail at once, trigger one refresh and queue the others behind it. Otherwise you fire five refreshes, and with rotation, four fail and log the user out. Implement this single-flight refresh in the API client.

**OAuth 2.0 and OpenID Connect.** For third-party login, use the **authorization code flow with PKCE** for public clients like SPAs. RFC 9700 says clients should not use the implicit grant, so do not use it. Prefer a vetted library or identity provider over writing token handling yourself. See RFC 9700 in the references.

**Authorisation in the UI.** Hide buttons the user cannot use because it improves UX. Never rely on that. Every API endpoint checks permissions server-side.

**Other hygiene.**
- Never put secrets in the client bundle (chapter 15).
- Do not log tokens. Clear Query cache and client stores on logout, otherwise the next user on the device may see cached data.
- Handle expiry gracefully: redirect to login and return to the original page after success. Validate the return URL to avoid open redirects.
- Use HTTPS everywhere and set security headers on the app server.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Inspect cookies and storage | browser DevTools Application tab | Verify flags (`HttpOnly`, `Secure`, `SameSite`) |
| Inspect traffic | DevTools Network tab, `curl -v` | Trace the login and refresh flow |
| API client | a single fetch wrapper with refresh and retry | Single-flight refresh, one retry, typed errors |
| XSS defence | Content-Security-Policy, DOMPurify, ESLint security rules | Explain and apply layered mitigations |
| CSRF defence | `SameSite`, CSRF tokens, `Origin` checks | Choose a defence for a given cookie setup |
| Identity | OIDC provider, PKCE libraries | Use a standard flow rather than a custom one |

### Common mistakes

- Storing a long-lived token in `localStorage` for convenience.
- Believing `httpOnly` removes the need to fix XSS.
- Using cookies with no CSRF defence for state-changing endpoints.
- Treating route guards or hidden buttons as access control.
- Several concurrent 401s each triggering a refresh.
- Leaving the previous user's cached data after logout.
- Shipping API secrets or privileged keys in the bundle.
- An unvalidated `redirect` parameter after login.

### Self-check

- Compare memory, `localStorage` and `httpOnly` cookie storage for an access token. What attack does each face?
- Why does `httpOnly` not make XSS harmless?
- Why are bearer tokens in headers not vulnerable to CSRF, and cookies are?
- Describe a refresh flow and how you avoid a refresh stampede.
- Why is a protected route not security?

---

## 11. Performance

**In plain words:** Measure before you fix anything. Most slow pages come from large downloads, slow requests or very long lists, not from React itself. Fix the biggest cost first.

### Understand

**Measure first.** Optimise a measured problem. React is fast for typical UIs, and most "slowness" is a network waterfall, an oversized bundle, a huge list or one expensive computation, not generic re-rendering.

**Why components re-render.** A component re-renders when:
1. Its own state changes.
2. Its parent re-renders (by default, **all children re-render**, regardless of whether their props changed).
3. A context it consumes changes.

Props changing does not itself trigger anything. Props are only a reason a memoised component would decide to skip. A re-render is cheap if render is cheap and produces no DOM changes. It is a problem when render work is heavy or runs too often.

**Cheap fixes first.**
- **Move state down.** Isolate frequently changing state (a text input) into a small component so the rest of the tree does not render.
- **Pass children.** `<Layout>{expensiveTree}</Layout>` where the tree is created by a parent that does not re-render keeps the element reference stable.
- **Lift content up**, or split a context by update frequency.

**`React.memo`.** Wraps a component so it skips re-rendering when props are shallow-equal. It helps only when the component is expensive and props are stable. If you pass a new object, array or function each render, the comparison fails every time. Stabilise with `useMemo` and `useCallback`, or restructure. The React Compiler may automate this (chapter 4).

**Lists.** Rendering thousands of DOM nodes is slow no matter how well memoised. **Virtualisation** (windowing) renders only visible rows. Use a library such as TanStack Virtual or react-window. Also consider server-side pagination, which is usually the better answer for data from your API.

**Expensive computation.** Use `useMemo` if profiling shows a calculation dominates, or move it to the server, a Web Worker, or an indexed structure.

**Responsiveness tools.** `useTransition` and `useDeferredValue` (chapter 4) keep typing smooth while heavy work follows.

**Bundle size and loading.**
- **Code splitting:** `React.lazy(() => import("./Page"))` with `<Suspense fallback=...>` loads a component's code when first needed. Split by route first, then by heavy widgets (charts, editors, maps).
- **Analyse the bundle:** a visualiser shows which dependencies dominate. Replace heavy libraries (date and utility libraries often), import specific modules, and use tree-shakeable packages.
- **Network:** compression (gzip or Brotli), long cache lifetimes on hashed asset filenames, HTTP/2 or 3, a CDN, and `preload` for critical resources.
- **Images:** correct dimensions, modern formats, `loading="lazy"`, `width` and `height` to avoid layout shift.
- **Waterfalls:** a component that fetches, renders a child, which then fetches, creates serial requests. Prefetch, use route loaders or fetch in parallel.

**Core Web Vitals.** Largest Contentful Paint (loading), Interaction to Next Paint (responsiveness) and Cumulative Layout Shift (visual stability) are the field metrics Lighthouse and Chrome report. Confirm current metric definitions on web.dev.

**Profiling workflow.**
1. Reproduce in a production build (development mode is slower and double renders).
2. Open the React DevTools Profiler, record the interaction, and read which components rendered and how long they took. Enable "why did this render".
3. Fix the largest cost. Re-measure.
4. Use the browser Performance panel for long tasks, layout and network.
5. Run Lighthouse for load performance and Core Web Vitals.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Render profiling | React DevTools Profiler, "highlight updates" | Find the component and cause of a slow interaction |
| Browser profiling | Chrome DevTools Performance and Network panels | Identify long tasks and waterfalls |
| Page quality | Lighthouse, web.dev Core Web Vitals | Read a report and prioritise fixes |
| Bundle | `vite build` output, `rollup-plugin-visualizer` | Find the largest dependencies |
| Splitting | `React.lazy`, `Suspense`, dynamic `import()` | Split by route and heavy widgets |
| Large lists | TanStack Virtual, react-window, pagination | Render 100k rows smoothly |
| Memoisation | `React.memo`, `useMemo`, `useCallback` | Apply only with a measured gain |

### Common mistakes

- Memoising everything without profiling, adding complexity and no gain.
- `React.memo` on a component whose props include inline objects or functions.
- Measuring in development mode.
- Rendering an unpaginated 10,000-row table.
- Shipping a charting or editor library on the first page.
- Fetching in a child after the parent rendered, creating a waterfall.
- Storing fast-changing values (mouse position) in a high-level state or context.

### Self-check

- Why does a child re-render when its parent re-renders even if its props are identical?
- When does `React.memo` fail to help?
- A page loads slowly. List five things you check, in order.
- How would you render a very large list?

---

## 12. Error boundaries and Suspense

**In plain words:** An error boundary catches a crash in part of the screen and shows a message instead. Suspense shows a loading message while part of the screen is not ready. Together they stop one problem from breaking the whole page.

### Understand

**Error boundaries.** An error thrown while rendering would unmount the whole tree. An **error boundary** catches errors in its subtree during render, in lifecycle methods and in constructors, and shows a fallback. It does **not** catch errors in event handlers, asynchronous code (promises, timeouts), server rendering or errors in the boundary itself. Handle those with try/catch and error state.

Historically, an error boundary had to be a class component (this is the one remaining case where class components appear). In practice, use a maintained library such as `react-error-boundary`, or the error element of your router, so that you write no class yourself. Place boundaries deliberately: one around the whole app for a last resort page, and finer ones around independent regions (a widget, a route) so that one failure does not blank the screen. Provide a reset mechanism (retry button, reset on navigation) and report errors to a monitoring service.

TanStack Query can throw errors to the nearest boundary (an option on queries), which fits this model.

**Suspense.** `<Suspense fallback={<Spinner />}>` shows a fallback while a child is not ready to render. Ready means a lazily loaded component's code has arrived, or a Suspense-enabled data source (a library or framework integration, or React's `use` with a promise) is waiting. Suspense is a declarative loading boundary: loading UI is decided by where you place the boundary, not by each component's own `isLoading` branch.

Key ideas:
- Place boundaries to control the granularity of loading states and to avoid a page where every widget flickers independently.
- Combine with **transitions**: a transition keeps the old UI visible while new content suspends, instead of replacing it with the fallback.
- A Suspense boundary catches "not ready", and an error boundary catches "failed". Typically nest them together around async regions.
- Suspense for data requires a supported source. A bare `fetch` inside a component does not suspend. Confirm which data-fetching integrations support it in your library's docs and in react.dev.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Error containment | `react-error-boundary`, router `errorElement` | Add boundaries at app, route and widget level |
| Loading states | `<Suspense>`, `React.lazy`, Query Suspense options | Design loading granularity |
| Reporting | a client-side error monitoring service, source maps | Get actionable stack traces from production |
| Debugging | React DevTools (inspect suspended tree), source maps | Reproduce and trace a boundary-caught error |

### Common mistakes

- Expecting an error boundary to catch errors in `onClick` or in a `fetch` promise.
- A single top-level boundary only, so any small failure blanks the app.
- No retry or reset path in the fallback.
- Wrapping every component in its own Suspense and producing a spinner storm.
- Swallowing errors without reporting them.

### Self-check

- Name three kinds of errors an error boundary does not catch.
- What is the difference between what Suspense handles and what an error boundary handles?
- Where would you put boundaries in a dashboard with five independent widgets?

---

## 13. Accessibility basics

**In plain words:** Accessible apps work for people who use a keyboard, a screen reader or zoom. Most of this comes from using the right HTML elements. Test with the keyboard, not only with the mouse.

### Understand

**Why it matters.** Accessibility (a11y) is correctness for users of keyboards, screen readers, voice control and zoom, and in many jurisdictions a legal requirement. Most of it comes from using correct HTML. Aim for conformance with WCAG 2.x at level AA and confirm the current version on w3.org.

**Principles in practice.**
- **Semantic HTML first.** Use `<button>` for actions, `<a href>` for navigation, `<label>`, `<nav>`, `<main>`, `<h1>` to `<h6>` in order, `<ul>` for lists, `<table>` for tabular data. A `<div onClick>` has no role, no keyboard support and no focus.
- **Keyboard.** Everything operable by mouse must work with Tab, Enter, Space, Escape and arrow keys where appropriate. Focus order should follow visual order. Never remove the focus outline without a visible replacement.
- **Labels and names.** Every input has an associated label (`htmlFor` and `id`, `useId`). Icon-only buttons need an `aria-label`. Images need meaningful `alt`, or `alt=""` if decorative.
- **ARIA is a supplement.** First rule: do not use ARIA if a native element does the job. Use it for things HTML cannot express: `aria-expanded`, `aria-live` regions for status messages, `role="dialog"` with `aria-modal` (or the native `<dialog>`), `aria-describedby` for error text.
- **Focus management in SPAs.** Route changes do not reload the page, so screen readers get no signal. Move focus to the new page heading or announce the change. Modals must trap focus and restore it on close. Prefer a tested headless component library (Radix UI, React Aria, Headless UI) for dialogs, menus and comboboxes rather than building them from scratch.
- **Colour and contrast.** Meet contrast ratios (4.5:1 for normal text at AA). Do not convey meaning with colour alone. Respect `prefers-reduced-motion`.
- **Dynamic content.** Announce async results (form errors, toasts, loaded results) through `aria-live` regions.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Static checks | `eslint-plugin-jsx-a11y` | Fix all reported issues |
| Automated audit | Lighthouse accessibility, axe DevTools, `jest-axe`/`vitest-axe` | Catch the machine-detectable subset |
| Manual testing | keyboard-only run, a screen reader (NVDA, VoiceOver) | Complete a core flow with no mouse |
| Test queries | React Testing Library `getByRole`, `getByLabelText` | Write tests that mirror accessible use |
| Headless components | Radix UI, React Aria | Use accessible primitives for complex widgets |

### Common mistakes

- Clickable `div`s and `span`s.
- Placeholder as the only label.
- Missing focus management on route change and modal open or close.
- Colour-only error indication.
- Overusing ARIA, producing conflicting semantics.
- Treating a green Lighthouse score as proof of accessibility. Automated tools find only part of the problems.

### Self-check

- Why is `<div onClick>` worse than `<button>`?
- What is the first rule of ARIA?
- What must you do for screen reader users when an SPA route changes?

---

## 14. Testing

**In plain words:** Tests check that your app works the way a user expects. Test what the user can see and do, not the internal code. Keep the tests useful and few enough to maintain.

### Understand

**What to test.** Test **behaviour a user can observe**, not implementation. A good test would still pass after you refactor internals. Avoid asserting on state variables, hook calls or component structure.

**The test pyramid for a front end.**
- **Unit tests** for pure logic: reducers, formatters, validators, custom hooks.
- **Component and integration tests** (the sweet spot): render a component or a screen, interact with it as a user, assert on what is visible. The API is mocked at the network boundary.
- **End-to-end tests** (Playwright or Cypress) for a few critical journeys (login, checkout) against a real or staging backend. Slower and more fragile, so keep them few.
- **Contract tests or a shared schema** to keep the client aligned with your API.

**Vitest or Jest.** Both are test runners with assertions, mocking, watch mode and coverage. Vitest integrates with Vite config, uses ESM natively and has a Jest-compatible API, so it is the natural choice in a Vite project. Jest remains common in older codebases and in Create React App era projects. Component tests need a DOM environment (`jsdom` or `happy-dom`).

**React Testing Library.** It renders components and provides queries that reflect how users find things. Priority order of queries: `getByRole` (with `name`), `getByLabelText`, `getByText`, then `getByTestId` as a last resort. Use `userEvent` over `fireEvent` for realistic interaction. Use `findBy*` (async) for content that appears later and `queryBy*` to assert absence.

```tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { expect, test } from "vitest";

test("shows an error for an invalid email", async () => {
  const user = userEvent.setup();
  render(<SignupForm />);

  await user.type(screen.getByLabelText(/email/i), "not-an-email");
  await user.click(screen.getByRole("button", { name: /sign up/i }));

  expect(await screen.findByRole("alert")).toHaveTextContent(/valid email/i);
});
```

(`toHaveTextContent` comes from `@testing-library/jest-dom`, which you register in a setup file.)

**Mocking the network.** Mock at the HTTP layer with a tool such as **Mock Service Worker (MSW)** so the real client code, headers and error handling run, rather than mocking `fetch` or your API module in every test. Define realistic success and failure handlers, including 401, 422 and 500 responses.

**Testing TanStack Query code.** Wrap the component in a fresh `QueryClientProvider` per test, with retries disabled so failures surface quickly. Share nothing in the cache between tests.

**Testing hooks.** Use `renderHook` for custom hooks. If a hook is only used by one component, test it through the component.

**Async and `act`.** React Testing Library wraps interactions in `act` for you. Warnings about updates not wrapped in `act` usually mean you asserted before an async update completed. Use `findBy*` or `waitFor` rather than adding delays.

**Do not over-test.** Skip snapshot tests of large trees, tests of third-party libraries and tests that duplicate type checks. Coverage percentage is a poor goal. Test the behaviours that would hurt if broken.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Runner | Vitest (or Jest), `jsdom`/`happy-dom` | Configure and run tests in watch mode and CI |
| Component tests | React Testing Library, `user-event`, `jest-dom` | Write role-based queries and user-driven tests |
| Network mocks | MSW | Simulate success, validation and server errors |
| E2E | Playwright or Cypress | Cover login and one core flow |
| Accessibility | `vitest-axe`/`jest-axe` | Add an automated a11y assertion to key screens |
| Type safety | `tsc --noEmit` in CI | Fail builds on type errors |

### Common mistakes

- Testing implementation details (internal state, method calls).
- Querying by test id or CSS class when a role or label would do.
- Mocking your own API module everywhere, so network errors and headers are never exercised.
- Sharing one `QueryClient` across tests and getting cross-test cache leaks.
- Fixed `setTimeout` waits instead of `findBy` or `waitFor`.
- Chasing 100 percent coverage with low-value tests.

### Self-check

- What makes a component test resilient to refactoring?
- Why mock at the network boundary rather than the API module?
- Why query by role and label instead of test id?
- What should an end-to-end suite cover versus component tests?

---

## 15. Vite

**In plain words:** Vite is the tool that runs your app while you develop and builds it for release. It starts fast and updates the page as you edit. Its environment variables are public, so never put secrets in them.

### Understand

**What Vite is.** A build tool and dev server. In development it serves your source files as native **ES modules** to the browser, which requests them directly. Dependencies from `node_modules` are pre-bundled once (with a fast bundler) so that hundreds of internal modules become a few requests. Because the browser loads only what a page imports and only transforms files on demand, start-up stays fast regardless of project size, and **Hot Module Replacement (HMR)** updates changed modules in place, preserving component state in most edits.

**Production build.** `vite build` bundles, tree-shakes, minifies, splits code at dynamic `import()` boundaries and emits hashed filenames into `dist/`. Hashed names allow immutable long-term caching: a changed file gets a new URL. The bundler used internally has changed across Vite major versions, so confirm in the Vite docs. The output is static files you can serve from any web server or CDN.

**Project shape.**
- `index.html` is the entry point at the project root, and it references `/src/main.tsx` through a script tag of type module.
- `vite.config.ts` configures plugins (`@vitejs/plugin-react`), aliases, the dev server and build options.
- `public/` holds files copied as-is. Everything else imported from `src/` is processed and hashed.

**Dev proxy.** The browser blocks cross-origin requests without CORS headers. In development, configure `server.proxy` so requests to `/api` go through the Vite server to your backend. The browser sees one origin and no CORS setup is needed locally. In production, the same effect comes from a reverse proxy routing `/api` to the backend. This also makes cookie-based auth simpler (same site).

**Environment variables.** Vite loads `.env`, `.env.local`, `.env.[mode]` and `.env.[mode].local` files. **Only variables prefixed with `VITE_` are exposed** to client code (the prefix can be changed with `envPrefix`), through `import.meta.env.VITE_API_URL`. Others stay out of the bundle. The prefix exists so you do not accidentally leak server secrets, but it is a guard, not protection.

**Why they are public.** Values are **inlined into the JavaScript at build time** and shipped to every visitor. Anyone can read them from the bundle or DevTools. Therefore:
- Never put secrets (API keys with write access, signing keys, database credentials, client secrets) in `VITE_` variables or anywhere in the client.
- Public identifiers (an API base URL, a public analytics id, an OAuth client id for a public client) are fine.
- Because they are inlined at build time, changing a value needs a **rebuild**, so one image cannot be promoted across environments unless you load runtime config (for example a `/config.json` fetched at startup) instead.
- Other built-ins: `import.meta.env.MODE`, `DEV`, `PROD`, `BASE_URL`, `SSR`.

**Modes.** `vite` runs in `development` mode, `vite build` in `production`. A custom `--mode staging` loads `.env.staging`.

**Types.** Declare your variables in `vite-env.d.ts` (`interface ImportMetaEnv { readonly VITE_API_URL: string }`) so access is typed.

**Preview.** `vite preview` serves the built `dist/` locally to check the production bundle. It is not a production server.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Dev server | `vite`, HMR, `server.proxy` | Run against a local API with no CORS errors |
| Build | `vite build`, `vite preview` | Produce and verify a production bundle |
| Config | `vite.config.ts`, plugins, aliases, `build.rollupOptions` | Add a plugin and an alias |
| Env | `.env*` files, `import.meta.env`, `vite-env.d.ts` | State which variables are public and why |
| Analysis | `rollup-plugin-visualizer`, build size report | Find what bloats the bundle |
| Tests | Vitest (shares Vite config) | One config for build and test |

### Common mistakes

- Putting a secret in a `VITE_` variable.
- Expecting to change an environment variable on a deployed bundle without a rebuild.
- Forgetting the SPA history fallback when serving `dist/`.
- Hard-coding `http://localhost:3000` instead of using the proxy or a configured base URL.
- Serving with `vite preview` in production.
- Importing a Node-only module into client code.

### Self-check

- Why is Vite's dev server start-up fast compared to a bundle-first approach?
- Why can `VITE_` variables never hold secrets?
- How do you avoid CORS problems in development, and in production?
- Why does a changed env var require a rebuild?

---

## 16. Tailwind CSS

**In plain words:** Tailwind gives you many small CSS classes that you put directly on elements. You build a design by combining them. Write complete class names so Tailwind can find them.

### Understand

**Utility-first model.** Instead of writing named CSS classes and rules, you compose small single-purpose classes in markup: `className="flex items-center gap-2 rounded-md bg-blue-600 px-3 py-2 text-white hover:bg-blue-700"`. Tailwind scans your source files for class names and generates only the CSS you use, so the output stays small. Benefits: no naming problem, local reasoning (styles live next to the element), consistent scale from a design token set, no dead CSS growth. Costs: long class lists and a learning curve.

**Design tokens, not arbitrary values.** Spacing, colour, type size, radius and breakpoints come from a theme scale. Using the scale gives visual consistency. Arbitrary values (`w-[137px]`) exist as an escape hatch.

**Configuration.** Configuration differs by major version:
- Tailwind v3 used a `tailwind.config.js` with a `content` array (the files to scan) and a `theme.extend` object for custom tokens.
- Tailwind v4 is CSS-first: install `tailwindcss` and `@tailwindcss/vite`, add the plugin to `vite.config.ts`, and write `@import "tailwindcss";` in your CSS. No `tailwind.config.js` or `content` paths are needed, because sources are detected automatically. Theme tokens go in CSS with `@theme`.
Either way, you extend the default theme with your brand colours, fonts and spacing rather than replacing it.

**Variants.** State and condition prefixes apply a utility conditionally: `hover:`, `focus-visible:`, `disabled:`, `group-hover:`, `peer-checked:`, `aria-expanded:`, `first:`, `md:`, `dark:`. They compose (`dark:md:hover:bg-gray-800`).

**Responsive design.** Mobile-first. Unprefixed classes apply at all sizes. A breakpoint prefix (`sm:`, `md:`, `lg:`, `xl:`) applies at that width and **up**. Write the small-screen layout first, then add overrides: `grid grid-cols-1 md:grid-cols-3`.

**Dark mode.** The `dark:` variant applies when dark mode is active. By default it follows the operating system preference (`prefers-color-scheme`). You can configure it to follow a class or data attribute on the root element so users can toggle and persist a choice. In v4, add `@custom-variant dark (&:where(.dark, .dark *));` to your CSS and toggle a `dark` class on `<html>`. In v3, set `darkMode: 'class'` in the config. Define colours as semantic tokens (surface, text) so you do not duplicate `dark:` on every element.

**Dynamic class names.** Tailwind finds classes by **scanning source text**. A class built with string interpolation, such as `` `bg-${color}-500` ``, never appears literally and is not generated. Use complete class names in the source, and map state to full strings:

```tsx
const variants = {
  primary: "bg-blue-600 text-white hover:bg-blue-700",
  danger: "bg-red-600 text-white hover:bg-red-700",
} as const;

function Button({ variant = "primary", className = "", ...props }: ButtonProps) {
  return <button className={`${variants[variant]} rounded-md px-3 py-2 ${className}`} {...props} />;
}
```

Helpers: `clsx` for conditional classes, `tailwind-merge` to resolve conflicting utilities when a caller overrides classes, and `class-variance-authority` for variant maps.

**Component extraction.** Repeating a long class list is solved in React by extracting a **component**, not by CSS. Create `Button`, `Card`, `Input`. The component is the abstraction, and the classes live in one place. Use `@apply` sparingly, and mostly for styling content you do not control (rendered markdown, third-party widgets). Over-using `@apply` recreates the naming and indirection problems utility-first avoids.

**Accessibility and states.** Style `focus-visible` rings, keep contrast in both themes and use `motion-reduce:` to respect reduced motion preference.

**Component libraries.** Headless libraries (Radix UI, Headless UI, React Aria) provide behaviour and accessibility with no styles, and pair well with Tailwind. Copy-in component collections built on them are common. Treat the code as yours, and review it.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Utilities | Tailwind class reference (docs search), IntelliSense extension | Style a screen without leaving markup |
| Theme | config or CSS theme tokens | Add brand colours and fonts |
| Responsive and dark | breakpoint and `dark:` prefixes | Build a mobile-first layout with dark theme |
| Class composition | `clsx`, `tailwind-merge`, `class-variance-authority` | Build a variant-driven Button |
| Formatting | `prettier-plugin-tailwindcss` | Keep class order consistent |
| Debugging | browser DevTools computed styles | Trace which utility won |

### Common mistakes

- Building class names with string concatenation so Tailwind never generates them.
- Overusing arbitrary values instead of extending the theme.
- Copy-pasting long class strings instead of extracting a component.
- Desktop-first thinking with `max-` overrides everywhere.
- Using `@apply` to recreate traditional CSS.
- Removing focus outlines and not replacing them.
- Passing conflicting utilities through `className` without `tailwind-merge`.

### Self-check

- Why does `bg-${color}-500` fail, and how do you fix it?
- What does `md:grid-cols-3` mean in a mobile-first system?
- How do you avoid repeating a 15-class button?
- Where do brand colours belong?

---

## 17. Server rendering and frameworks

**In plain words:** Normally the browser builds the page from JavaScript. With server rendering, the server sends ready HTML first. This helps search engines and first load, but it adds a server to run.

### Understand

**Client-side rendering (CSR).** The server sends a near-empty HTML shell and a JavaScript bundle. The browser downloads, runs, then fetches data and renders. Simple to host (static files) and fine for authenticated dashboards behind a login. Weaknesses: slower first meaningful paint, since content waits for JavaScript and then data, poor default for crawlers and link previews, and a data waterfall.

**Server-side rendering (SSR).** The server runs React and sends **HTML with content** for the first paint. The user sees content sooner, and crawlers and previews get real markup.

**Hydration (making server-made HTML interactive).** After HTML arrives, the browser downloads the JavaScript, and React **hydrates**: it attaches event handlers to the existing DOM and makes it interactive, expecting its render output to match the server's. Until hydration completes the page looks ready but may not respond. **Hydration mismatch** occurs when server and client output differ (for example rendering `Date.now()`, `Math.random()` or reading `window` or `localStorage` during render). Avoid non-deterministic or browser-only values in render, or defer them to an effect. `useId` exists partly to give matching ids on both sides.

**Related strategies.**
- **Static site generation (SSG):** pages rendered at build time.
- **Incremental regeneration and caching:** re-render stale pages in the background.
- **Streaming SSR:** send HTML in chunks, with Suspense boundaries filling in as data arrives.
- **React Server Components (RSC):** components that run only on the server and send a serialised result, never shipping their code to the browser, and may read data directly. They separate server and client components with a directive boundary. This is framework-integrated, so read your framework's documentation and react.dev.

**Frameworks.** React itself is a UI library and does not include routing, data conventions or a build pipeline for SSR. **Next.js** is the most widely used full-stack React framework, offering file-based routing, SSR, SSG, RSC and API routes. Other options include React Router in its framework mode and TanStack Start. Features and defaults change quickly, so confirm current behaviour in each framework's docs.

**When to choose what.**
- Internal tool or app behind login, with a separate backend API: a Vite SPA is often enough and easier to operate.
- Public, SEO-sensitive, content-heavy or performance-critical first load: a framework with SSR or SSG.
- Both: consider whether a backend-for-frontend layer is warranted.

**Backend implications.** SSR moves rendering load to servers and adds a runtime to operate, monitor and scale. Authentication with cookies is easier with SSR since the server can read the cookie on the first request. Caching strategies (CDN, per-user versus public pages) matter. Keep secrets in server-only code and be careful about what a server component or server function passes to the client.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Rendering modes | CSR, SSR, SSG, streaming (concepts) | Choose and justify one for a given product |
| Framework | Next.js (or another) docs, a small sample app | Build a page that fetches on the server |
| Diagnosing hydration | browser console warnings, DevTools | Find the source of a mismatch |
| Metrics | Lighthouse, web.dev Core Web Vitals | Compare CSR and SSR load metrics |

### Common mistakes

- Choosing SSR by default for a logged-in dashboard with no SEO needs.
- Using `window`, `localStorage`, `Date.now()` or random values during render in SSR code.
- Leaking secrets by using them in code that runs on the client.
- Assuming SSR removes the need for loading and error states.
- Ignoring that SSR adds a server runtime to operate.

### Self-check

- What is hydration, and what causes a hydration mismatch?
- Compare CSR and SSR for a public blog and for an internal admin page.
- What does a framework like Next.js add that React and Vite do not?

---

## 18. Common pitfalls

**In plain words:** This chapter lists frequent mistakes, with the symptom and the fix for each. Read it as a review checklist.

A consolidated list. Each shows the symptom, the cause and the fix.

**Stale closures.** A handler or effect uses an old value because it closed over a previous render's snapshot.
```tsx
// Bug: interval always sees count = 0
useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000);
  return () => clearInterval(id);
}, []);

// Fix: updater form, no dependency on count
useEffect(() => {
  const id = setInterval(() => setCount((c) => c + 1), 1000);
  return () => clearInterval(id);
}, []);
```
Other fixes: list the dependency, or store the latest value in a ref when a long-lived callback needs it.

**Missing or unstable keys, mutating state, derived-state duplication, effects used for events, server data copied into `useState`, and components defined inside components.** These are covered in chapters 1, 3, 5, 6 and 7. In short: use stable ids, copy on write, compute during render, do work in handlers, use a query cache, and define components at module level.

**Infinite effect loops.** Cause: an effect sets state that it also depends on, or depends on an object or function created each render. Fix: correct the dependencies, move values outside the component or inside the effect, use updater functions, or remove the effect.

**Prop drilling overuse (and the opposite).** Passing through five layers: try composition first, then context. Do not wrap everything in context to avoid two levels of props.

**Uncontrolled to controlled switch.** `value={undefined}` initially. Initialise with `""`.

**Async setState after unmount or out of order.** Use cleanup and cancellation (chapter 5) or a query library.

**Premature optimisation and ignoring the production build.** Memoising without measuring adds complexity. Behaviour, performance and env differ from development, so test the built app.

**Security slips.** Secrets in `VITE_` variables, tokens in `localStorage`, unsanitised HTML, auth by route guard only.

### Self-check

- Explain the stale closure in the first example and give two fixes.
- Why does defining a component inside another component cause state loss?
- How do you recognise duplicated derived state in code review?

---

## 19. Practice labs

**In plain words:** These exercises turn the ideas into skill. Build each one yourself before you look for answers. Each lab says what "done" looks like.

Do these in a fresh Vite project (`npm create vite@latest` and pick the React and TypeScript template, confirming the current prompts in the Vite docs). You can use a public mock API or a small API of your own. Each lab states the goal and what "done" looks like. Attempt before looking anything up.

1. **Render model experiments.** Add `console.log` to a parent and child. Trigger parent state changes and observe which components render. Add `React.memo` to the child with a stable prop, then with an inline object prop, and explain the difference. Wrap the app in StrictMode and explain each double log.
2. **Immutable list.** Build a todo list with add, toggle and delete using only immutable updates. Then reproduce a "UI did not update" bug through mutation and fix it. Add stable keys, then break them with index keys and a reorder to see the symptom.
3. **Snapshot and updater.** Write a button that calls `setCount(count + 1)` three times and another using updaters. Explain the results. Then reproduce a stale closure in an interval and fix it two ways.
4. **Race-free fetch effect.** Build a user viewer driven by an id input with a hand-written fetch effect. Throttle the network in DevTools, reproduce an out-of-order response, then fix it with `AbortController`. Check `response.ok`. List what you would still be missing.
5. **Migrate to TanStack Query.** Replace lab 4 with `useQuery`. Add devtools and observe cache, stale and fetching states. Tune `staleTime`. Add a list and detail view that share data through keys.
6. **Mutations and optimism.** Add a create and a toggle mutation with invalidation. Then make the toggle optimistic with rollback, and force the API to fail (for example with MSW) to verify rollback.
7. **Form with validation.** Build a signup form with react-hook-form and a zod schema. Show field errors accessibly (`aria-invalid`, `aria-describedby`). Map a 422 server response to field errors and prevent double submit.
8. **Routing and deep links.** Build a layout route with nested pages, a dynamic route, a 404 and search-param filters. Build, serve `dist/` with a static server, and fix the deep-link refresh using a history fallback.
9. **Auth flow.** Against an API you control (or a mock), implement login with an in-memory access token and an `httpOnly` refresh cookie. Add a single-flight refresh on 401 with one retry. Verify with five simultaneous requests that only one refresh occurs. Clear the cache on logout. Write down the XSS and CSRF analysis for your setup.
10. **Route guard honesty.** Add a protected route, then bypass it by editing client state in DevTools. Show the API still returns 401 or 403. Write one paragraph on why the guard is UX only.
11. **Performance pass.** Render 20,000 rows. Measure with the Profiler, then fix with pagination and with virtualisation. Add `useDeferredValue` to a search box over a heavy list and compare input responsiveness.
12. **Bundle diet.** Run a bundle visualiser on a production build. Add a heavy library, lazy-load it by route, and show the initial bundle shrinking. Run Lighthouse before and after.
13. **Error and loading boundaries.** Add an error boundary and a Suspense boundary around a widget. Make the widget throw in render, and separately throw in an event handler, and document which boundary caught which.
14. **Accessibility audit.** Run Lighthouse and axe on your app, fix findings, then complete the main flow using only the keyboard and a screen reader. Add focus management on route change.
15. **Tests that matter.** Write RTL tests for the form (validation), a Query-backed list (loading, success, error through MSW) and a reducer. Use role-based queries. Break the code and confirm the tests fail for the right reason.
16. **Tailwind design system slice.** Build `Button`, `Input` and `Card` with variants, dark mode and responsive layout using `clsx` and `tailwind-merge`. Demonstrate the dynamic class name pitfall and its fix.
17. **Env and config.** Add `VITE_API_URL`, build, and find the value in the emitted JavaScript. Write a short note on why secrets cannot live there. Implement runtime config loaded at startup to deploy one build to two environments.
18. **SSR comparison (optional).** Build one page in a framework with SSR, compare the initial HTML against your SPA with `curl`, and measure with Lighthouse. Note what moved to the server.

---

## 20. Interview questions

**In plain words:** This chapter lists practice questions with short model answers. Each question is tagged by level: [Fresher], [Mid] or [Senior]. Answer aloud first, then compare with the model answer. Practice improves your skill, but no guide can promise any hiring result.

### Top 10 must-know

Learn these first. Each points to its full model answer below.

1. [Fresher] Props versus state (Q1): props are read-only inputs from a parent. State is data the component owns, and setting it schedules a render.
2. [Fresher] Keys (Q2): a stable, unique key per item lets React match items by identity. Do not use the array index for lists that change.
3. [Fresher] Why not mutate state (Q7): React compares references, so a mutated object looks unchanged. Always create a new object or array.
4. [Mid] Render versus commit (Q13): render is a pure calculation of the next UI. Commit changes the DOM and runs effects.
5. [Mid] Effects as synchronisation (Q16): an effect keeps an outside system in step with current values. The dependency array lists what it reads.
6. [Mid] Stale closure (Q18): a callback kept an old render's value. Fix it with an updater function, correct dependencies, or a ref.
7. [Mid] Server state versus client state (Q22): server state is remote and can go stale, so use a cache such as TanStack Query.
8. [Mid] Race condition in fetch effects (Q19): abort or ignore the old request in cleanup.
9. [Senior] Authentication design (Q31): short-lived in-memory access token, `httpOnly` refresh cookie, XSS and CSRF defences, authorisation on the API.
10. [Senior] Protected route is not security (Q32): the user controls client code, so the API must check every request.

### Fresher

1. [Fresher] What is the difference between props and state?
2. [Fresher] Why do list items need a `key`, and what makes a good key?
3. [Fresher] What does `useState` return, and why does calling the setter not change the variable immediately?
4. [Fresher] What is the dependency array of `useEffect`?
5. [Fresher] What is JSX, and what does it compile to?
6. [Fresher] What is the difference between controlled and uncontrolled inputs?
7. [Fresher] Why must you not mutate state?
8. [Fresher] What are the rules of hooks?
9. [Fresher] How do you conditionally render content, and what is a pitfall with `&&`?
10. [Fresher] What does `useRef` do, and how does it differ from `useState`?
11. [Fresher] What is lifting state up?
12. [Fresher] Why does `fetch` not throw on a 404 or 500?

### Mid

13. [Mid] Explain render versus commit.
14. [Mid] When does a component re-render?
15. [Mid] When do `useMemo`, `useCallback` and `React.memo` actually help?
16. [Mid] Why is "synchronisation, not lifecycle" the right model for effects?
17. [Mid] Give examples of when you do not need an effect.
18. [Mid] What is a stale closure? Show one and fix it.
19. [Mid] How do you prevent a race condition in a fetching effect?
20. [Mid] Why use TanStack Query instead of `useEffect` and `fetch`?
21. [Mid] Explain `staleTime` versus `gcTime`, and how you invalidate after a mutation.
22. [Mid] What is the difference between server state and client state?
23. [Mid] When would you choose context, Zustand or Redux Toolkit?
24. [Mid] How do you validate a form with react-hook-form and zod, and why validate again on the server?
25. [Mid] How do you structure error handling and loading states in a data-heavy page?
26. [Mid] What does an error boundary catch and not catch?
27. [Mid] How do you test a component that calls an API?
28. [Mid] What do `VITE_` variables expose, and what must never go in them?
29. [Mid] Why does refreshing a deep link return 404 on a static host?
30. [Mid] How does Tailwind generate CSS, and why does dynamic class building fail?

### Senior

31. [Senior] Design client-side authentication for an SPA with a separate API. Discuss token storage, refresh, CSRF and XSS.
32. [Senior] Why is a protected route not an authorisation mechanism?
33. [Senior] A page with a large table is sluggish. How do you diagnose and fix it?
34. [Senior] How do you reduce initial bundle size and improve Core Web Vitals?
35. [Senior] Compare CSR, SSR, SSG and streaming. How do you choose for a product?
36. [Senior] What is hydration, and what causes a mismatch?
37. [Senior] How would you design the data layer of a large React app: API client, cache, errors, types, optimism?
38. [Senior] How do you make the client resilient to API contract changes?
39. [Senior] How does Suspense relate to transitions and error boundaries?
40. [Senior] What does the React Compiler change about memoisation, and what should you still understand?
41. [Senior] How would you handle concurrent 401 responses and token refresh?
42. [Senior] How do you structure a front-end test strategy and what do you not test?
43. [Senior] How would you roll out a breaking API change without breaking deployed clients?
44. [Senior] What accessibility problems commonly appear in SPAs, and how do you prevent them?

### Model answers (condensed)

**1.** Props are inputs passed from a parent and are read-only. State is data owned by the component that changes over time and triggers renders when set.

**2.** Keys let React match items across renders by identity so it preserves state and moves DOM nodes correctly. A good key is stable and unique among siblings, such as a database id. Index keys are only safe for static lists.

**3.** The current value and a setter. The setter schedules a re-render. The value in the current render is a snapshot and does not change. Use an updater function when the next value depends on the previous one.

**4.** A declaration of the reactive values the effect reads. The effect re-runs when any changed (compared with `Object.is`). `[]` means it reads none, so it runs after mount only (plus a StrictMode development re-run).

**5.** Syntax that compiles to function calls creating element objects. It looks like HTML but is JavaScript, with `className`, camelCase events and expressions in braces.

**6.** Controlled: React state holds the value (`value` and `onChange`). Uncontrolled: the DOM holds it and you read it by ref or `FormData`. Uncontrolled suits simple forms and fewer renders. Controlled suits live validation and derived UI.

**7.** React detects change by reference comparison. A mutated object keeps the same reference, so updates can be skipped, memoisation breaks and history is lost. Create new objects and arrays.

**8.** Call hooks only at the top level of components or custom hooks, never in conditions, loops or nested functions, because React identifies hooks by call order. Only call them from React functions.

**9.** Ternary, `&&`, early return, or extracting a component. With `&&`, a left side of `0` renders `0`, so use an explicit boolean such as `count > 0`.

**10.** It holds a mutable `current` that persists across renders. Changing it does not re-render. Use it for DOM nodes and non-UI values (timer ids, abort controllers). State changes trigger renders and are for UI data.

**11.** Moving state to the nearest common ancestor of components that need it, passing it down as props with callbacks to change it.

**12.** `fetch` rejects only on network failure. HTTP error statuses resolve normally, so check `response.ok` and throw yourself.

**13.** Render calls components to compute the next UI description and must be pure. Commit applies the diff to the DOM and runs effects. Render can be repeated, paused or discarded, and commit happens once per update.

**14.** When its state changes, when its parent re-renders (regardless of props, unless memoised), or when a consumed context changes.

**15.** When the work is measurably expensive, when a memoised child needs stable props, or when a value is a hook dependency that must not change every render. Otherwise they add cost and complexity. The React Compiler may automate some of it.

**16.** An effect keeps an external system in sync with current values: setup when values apply, cleanup when they change or the component leaves. Thinking "on mount, on update" leads to missing dependencies and bugs. Thinking "what am I synchronising and with which values" yields correct dependency lists and cleanup.

**17.** Derived values (compute in render), resetting state on a prop change (use a key), responding to events (handler), data fetching (use a library), and chains of effects that set state.

**18.** A closure captured an old render's value, such as an interval callback using `count` from the first render. Fix with an updater function, correct dependencies with cleanup, or a ref holding the latest value.

**19.** Abort the in-flight request in cleanup with `AbortController`, or ignore the outdated result with a flag, so only the latest request updates state. Better, use a library that does this.

**20.** It provides caching, deduplication, race handling, retries, background refresh, invalidation, pagination and devtools. A hand-written effect covers none of it reliably and spreads duplicate logic.

**21.** `staleTime` is how long data counts as fresh before refetch triggers apply. `gcTime` is how long an unused entry stays cached. After a mutation, call `invalidateQueries` with the affected key prefix (or write the result with `setQueryData`).

**22.** Server state is remote, asynchronous, shared and can go stale, so it is cached and revalidated. Client state is local UI state that is synchronous and owned by the app. Mixing them causes synchronisation bugs.

**23.** Context for low-frequency values such as theme or the current user. Zustand for shared client state with selectors and little boilerplate. Redux Toolkit for large teams needing structure, middleware and DevTools traceability. Most apps need little beyond a query cache and local state.

**24.** Define a zod schema, pass it through the resolver to `useForm`, and render errors from `formState`. Validate on the server again because clients can be bypassed, and map server field errors back to the form.

**25.** Use the query's pending, error and empty states, put error boundaries and Suspense boundaries at region level, give retry paths, centralise API error mapping, and report errors.

**26.** It catches errors during rendering, lifecycle methods and constructors of its subtree. It does not catch event handler errors, asynchronous errors, server-rendering errors or errors in the boundary itself.

**27.** Render with a fresh `QueryClientProvider`, mock the network with MSW, interact with `user-event`, and assert on visible output using role and label queries, using `findBy*` for async results.

**28.** They are inlined into the public bundle at build time, so anyone can read them. Only non-secret configuration belongs there. Secrets stay on the server.

**29.** The browser requests that path from the server, and a static host has no such file. Configure a fallback to `index.html` for non-asset, non-API paths. The dev server does this automatically.

**30.** It scans source files for complete class-name strings and generates CSS for those. An interpolated name never appears literally and is not generated. Use full class names, mapped from variants.

**31.** Prefer a first-party cookie session or a short-lived in-memory access token with an `httpOnly`, `Secure`, `SameSite` refresh cookie with rotation. Defend XSS with escaping, CSP and no raw HTML. Defend CSRF with `SameSite`, tokens or header checks on cookie-authenticated mutations. Enforce authorisation on the API, use single-flight refresh, clear caches on logout, and consider OIDC with PKCE for third-party identity.

**32.** The client code and state are controlled by the user, so a guard can be bypassed. It improves UX only. The API must authorise every request.

**33.** Profile in a production build. Typical causes: too many DOM nodes, unstable props defeating memo, heavy render work or context fan-out. Fix with server pagination or virtualisation, moving state down, stabilising props, memoising a proven hot spot, and deferring non-urgent updates.

**34.** Analyse the bundle, split by route and heavy widgets, replace large dependencies, compress, use hashed assets with long cache lifetimes, optimise images, remove waterfalls with prefetching and parallel fetches, then re-measure with Lighthouse and field data.

**35.** CSR is simple to host and fine behind a login. SSR improves first paint and SEO at the cost of a server runtime. SSG suits mostly static content. Streaming sends HTML progressively behind Suspense. Choose by SEO needs, first-load sensitivity, personalisation and operational capacity.

**36.** Hydration attaches React to server-rendered HTML to make it interactive. A mismatch arises when server and client render different output, for example time, randomness or browser-only values read during render.

**37.** A typed API client with credential and error handling, a query cache keyed by resource, schema validation at the boundary, centralised 401 handling, consistent loading and error patterns, optimistic updates only where safe, and a small client store for UI state.

**38.** Validate responses with schemas at the client boundary, share or generate types from the OpenAPI contract, version the API, and use contract tests so drift is detected in CI rather than production.

**39.** Suspense shows a fallback while a subtree is not ready. A transition keeps old UI visible instead of showing that fallback for non-urgent updates. An error boundary handles failure. Nest them around async regions.

**40.** The compiler (stable, per react.dev) can automatically memoise components and values, reducing manual `useMemo` and `useCallback`. You still need to understand render purity, dependencies and identity, since the compiler assumes correct code.

**41.** Keep one in-flight refresh promise shared by all callers. Queue failed requests until it resolves, retry each once, and log out if refresh fails. With rotation, concurrent refreshes would invalidate each other.

**42.** Many component and integration tests of user-visible behaviour with a mocked network, unit tests for pure logic, a few end-to-end journeys, type checking and linting in CI. Do not test implementation details, third-party internals or large snapshots, and do not chase a coverage number.

**43.** Version the API or make additive changes first, support old and new shapes during a transition, ship the client tolerant of both, then remove the old path after clients upgrade. Note that SPA users may hold a cached old bundle, so plan for that.

**44.** Non-semantic controls, missing labels, focus not managed on route change or modals, colour-only cues, and unannounced dynamic updates. Prevent with semantic HTML, headless accessible primitives, linting, automated audits, and keyboard and screen reader testing.

---

## 21. References

**In plain words:** These are the official sources for this guide. Use them when a detail may have changed in a newer version.

Primary sources, preferred over blog posts:

- React documentation: https://react.dev
- React reference for hooks, including `useActionState`, `useOptimistic` and `use`: https://react.dev/reference/react
- React Compiler documentation (stable; verified): https://react.dev/learn/react-compiler
- React 19 release post (Actions, `ref` as a prop): https://react.dev/blog/2024/12/05/react-19
- React: "You Might Not Need an Effect": https://react.dev/learn/you-might-not-need-an-effect
- React: Synchronizing with Effects: https://react.dev/learn/synchronizing-with-effects
- Vite documentation (env variables, build, server proxy): https://vite.dev
- Tailwind CSS documentation: https://tailwindcss.com/docs
- TanStack Query documentation: https://tanstack.com/query/latest
- React Router documentation: https://reactrouter.com
- React Hook Form documentation: https://react-hook-form.com
- Zod documentation: https://zod.dev
- Vitest documentation: https://vitest.dev
- Testing Library (React Testing Library and guiding principles): https://testing-library.com/docs/react-testing-library/intro/
- Mock Service Worker documentation: https://mswjs.io
- Zustand documentation: https://zustand.docs.pmnd.rs
- Redux Toolkit documentation: https://redux-toolkit.js.org
- Next.js documentation: https://nextjs.org/docs
- MDN Web Docs (HTML, Fetch, AbortController, History API, cookies, ARIA): https://developer.mozilla.org
- MDN HTTP cookies and `Set-Cookie` attributes: https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies
- OWASP Cheat Sheet Series (XSS Prevention, CSRF Prevention, Session Management): https://cheatsheetseries.owasp.org
- Best Current Practice for OAuth 2.0 Security (RFC 9700): https://www.rfc-editor.org/rfc/rfc9700
- PKCE (RFC 7636): https://www.rfc-editor.org/rfc/rfc7636
- Web Content Accessibility Guidelines (WCAG): https://www.w3.org/WAI/standards-guidelines/wcag/
- WAI-ARIA Authoring Practices Guide: https://www.w3.org/WAI/ARIA/apg/
- web.dev Core Web Vitals and Lighthouse: https://web.dev
- React DevTools and Profiler (documented on react.dev): https://react.dev/learn/react-developer-tools

Verify any URL before depending on it, since pages move. The React, TanStack Query, Tailwind, Vite, React Router, React Hook Form, Zod, Zustand, Next.js and RFC 9700 links above were checked on 2026-11-06.

---

## 22. Glossary

**In plain words:** These are the hardest terms in this guide, explained simply. Each term is also explained in brackets where it first appears.

| Term | Plain meaning |
|---|---|
| Render | React runs your component function to work out what the screen should look like. |
| Commit | React applies the changes to the real page and then runs effects. |
| Reconciliation | React compares the old and new UI descriptions and finds what changed. |
| Key | A stable id on a list item so React can tell items apart. |
| Snapshot | The fixed values a component sees during one render. |
| Immutable | Never changed in place. You make a new copy instead. |
| Hook | A function (`useState`, `useEffect`) that lets a component use React features. |
| Effect | Code that connects a component to something outside React, such as a network or timer. |
| Cleanup | A function that stops or undoes the work of an effect. |
| Stale closure | A function that still uses an old value from an earlier render. |
| Memoisation | Saving a result so it is not calculated again for the same inputs. |
| Context | A way to give a value to many nested components without passing props. |
| Controlled input | A form field whose value is kept in React state. |
| Server state | Data that lives on the server. The browser only keeps a copy, which can become old. |
| Query key | The array that names one cached request in TanStack Query. |
| Stale (data) | Data that may be out of date and can be fetched again. |
| Invalidation | Marking cached data as stale so it is fetched again. |
| Optimistic update | Showing the expected result before the server confirms it. |
| Hydration | React attaching event handlers to HTML that the server already made. |
| Suspense | A React feature that shows a fallback while part of the page is not ready. |
| Error boundary | A component that catches render errors in its children and shows a fallback. |
| XSS | An attacker's script running inside your page. |
| CSRF | A harmful site making your browser send a request with your logged-in cookie. |
| PKCE | An extra check in the OAuth code flow that protects public clients such as SPAs. |
| HMR | Hot Module Replacement: the dev server updates changed code without a full reload. |
| Tree shaking | Removing code that is never used from the final bundle. |
| Code splitting | Cutting the bundle into parts that load only when needed. |
| SSR / SSG | Making HTML on the server for each request (SSR) or at build time (SSG). |
| Virtualisation | Drawing only the rows that are visible in a long list. |
