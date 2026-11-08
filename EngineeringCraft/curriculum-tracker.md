# 🛠️ Engineering Craft — The Craft Layer Tracker

> **Goal:** The skills that separate junior from senior — not *what* you build, but *how well*. This is what product companies screen for in code review, interviews, and promotions.
> **Language:** Java (matches your SystemDesign course).
> **Rule:** each file = ~2h read + ~1h practice on real code (your own or an open-source repo).
> **Teaching style (same as your other courses):** In plain words → Why/What/When/How/Where → 🤔 misconception hook → ✅ good / ❌ bad scenario → When NOT to.
> **Sizing note:** this is a *concentrated* course (~12 files), not 30 days — craft skills are deep in practice but not padded. Do the practice, not just the reading.

**Progress: ⬜⬜⬜⬜⬜⬜⬜⬜⬜⬜⬜⬜**

---

## The through-line
You spend more time **reading, changing, and verifying** code than writing it fresh. This course follows that reality: read code → debug it → test it → keep it clean → refactor it → review it → document it → propose changes → estimate the work.

---

## Files

### 1 — Reading & Navigating Code ⭐ *the most underrated skill*
- [ ] Why reading > writing; getting productive in an unfamiliar codebase fast
- [ ] Techniques: entry points, call graphs, "follow the data", bisecting by feature
- [ ] Tools: grep/ripgrep, IDE navigation, git blame/log as archaeology
- [ ] **Practice:** get oriented in an unfamiliar open-source Java repo in 30 min

### 2 — Debugging
- [ ] The scientific method of debugging (reproduce → hypothesize → test → fix)
- [ ] Tools: debuggers/breakpoints, logging, stack traces, `git bisect`
- [ ] Production debugging (can't just add print statements — reuse observability)
- [ ] **Practice:** debug a planted bug using bisect + a debugger

### 3 — Software Testing I (Unit & TDD)
- [ ] Why test; unit tests; JUnit; the AAA (Arrange-Act-Assert) shape
- [ ] TDD (red-green-refactor); mocking/stubbing (Mockito); test doubles
- [ ] What to test vs what not to; testing behavior not implementation
- [ ] **Practice:** TDD a small class end to end

### 4 — Software Testing II (Strategy)
- [ ] The test pyramid; integration & e2e tests; contract tests
- [ ] Coverage myths; flaky tests; test data; fixtures
- [ ] Test strategy for a service (reuse System Design testing instincts)
- [ ] **Practice:** write an integration test with a real DB (Testcontainers)

### 5 — Clean Code I (Naming, Functions, Errors)
- [ ] Naming that reveals intent; small functions; one level of abstraction
- [ ] Comments (when they help vs when they're a smell); error handling
- [ ] **Practice:** clean up a messy method without changing behavior

### 6 — Clean Code II (Smells & SOLID in practice)
- [ ] Code smells catalog (long method, large class, feature envy, primitive obsession…)
- [ ] SOLID applied (ties to SystemDesign Day 2); guard clauses; boundaries
- [ ] **Practice:** spot 5 smells in real code and name each

### 7 — Refactoring
- [ ] What refactoring is (and isn't); refactor only under a green test suite
- [ ] The catalog: extract method/class, rename, inline, replace conditional with polymorphism
- [ ] Refactoring safely in small steps; strangler pattern for big ones
- [ ] **Practice:** refactor a smelly class, tests staying green throughout

### 8 — Code Review
- [ ] How to review (what to look for, in what order); severity vs nitpick
- [ ] How to *get* reviewed (small PRs, good descriptions, self-review first)
- [ ] Giving/receiving feedback without ego; async review etiquette
- [ ] **Practice:** review a real PR/diff with a structured checklist

### 9 — Documentation
- [ ] READMEs that get people running fast; API docs; inline comments done right
- [ ] Docs that stay alive (near the code) vs docs that rot
- [ ] **Practice:** write a README + API doc for one of your projects

### 10 — Technical Writing (RFCs & ADRs)
- [ ] Design docs / RFCs — proposing a change that gets approved
- [ ] ADRs (Architecture Decision Records) — capturing *why*
- [ ] Structure: context → options → trade-offs → decision; writing for skimmers
- [ ] **Practice:** write an RFC for a real change + one ADR

### 11 — Estimation & Breaking Down Work
- [ ] Turning a vague ticket into scoped, shippable pieces
- [ ] Estimation techniques; why estimates are wrong and what to do about it
- [ ] Vertical slices; spikes; avoiding the planning trap
- [ ] **Practice:** decompose a feature into 1-day tasks with estimates

### 12 — Capstone + Interview/Promotion Prep
- [ ] Apply all of it: take a real (messy) codebase → read it → test it → refactor it → document it → propose an improvement via RFC
- [ ] Interview craft: "walk me through how you'd debug X", "review this code", "how do you decide what to test"
- [ ] Promotion signal: how these skills map to senior/staff expectations
- [ ] Gap analysis (skill → confidence 1-5 → action)

---

## 📁 File Index

| # | File | Studied? |
|---|---|---|
| — | `curriculum-tracker.md` | (this file) |
| 1–12 | *created in this batch* | ⬜ |

---

## Why this course matters most
Domain knowledge (system design, AI, DevOps, DB) gets you in the door. **These craft skills get you promoted** — they're visible in every PR you open and every review you give. Practice them on real code, every day, forever.
