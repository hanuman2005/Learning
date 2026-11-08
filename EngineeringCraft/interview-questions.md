# Engineering Craft: Interview Question Bank

This bank prepares you for craft interviews. Craft interviews ask how you work, not only what you know. They ask about reading code, debugging, testing, clean code, refactoring, review, writing, and estimation.

Every answer is a model, not a script. Do not memorize it. Use it to check your thinking, then retell it with a real story from your own work.

## How to use this file

1. Read the Top 10 first. These cover most craft interviews.
2. Pick a topic. Cover the answer, say yours aloud, then compare.
3. If your answer was weak, open the "Revise" file and re-read that section.
4. Practice the behavioural questions out loud with your own stories.
5. Do the code-review exercises on paper before reading the reviewer notes.

Level tags:
- [Fresher]: first job or internship. Expect basics and clear reasoning.
- [Mid]: 2 to 5 years. Expect judgment, trade-offs, and examples.
- [Senior]: Expect team-level impact, risk thinking, and teaching others.

Levels are a guide only. Companies use them differently. Nothing here guarantees any result.

## Contents

1. Top 10 must-know questions
2. Questions by topic (Q1 to Q69)
3. Behavioural questions (B1 to B12)
4. Code-review exercises (R1 to R8)
5. Common mistakes in craft and behavioural interviews

---

# 1. Top 10 must-know questions

These ten appear most often. Full model answers are in section 2. The number in brackets points to the full answer.

1. How do you approach an unfamiliar codebase? [Q1, Q2]
2. Walk me through debugging a bug. Why reproduce first? [Q6, Q7]
3. What makes a good unit test? [Q12, Q13]
4. What is the test pyramid, and why does its shape matter? [Q19]
5. What makes a name or a function clean? [Q25, Q26]
6. What is a code smell? Name three and their fixes. [Q32, Q33]
7. What is refactoring, and when must you not do it? [Q38, Q39]
8. How do you review code, and in what order? [Q44, Q45]
9. When do you write an RFC, and when an ADR? [Q55, Q56]
10. How do you estimate work you do not understand yet? [Q60, Q61, Q62]

A short spoken version of each:

**1. Unfamiliar codebase.** "I read before I change. I find an entry point, trace one path end to end, and use search and git history. I confirm my understanding by running tests."

**2. Debugging.** "I make the bug reproducible. Then I form one hypothesis, change one thing, and test it. I find the root cause, write a failing test, and then fix."

**3. Good unit test.** "It is fast, isolated, repeatable, and self-checking. It tests behavior, not internal steps. Its name tells me what broke."

**4. Test pyramid.** "Many fast unit tests, fewer integration tests at real boundaries, very few end-to-end tests. It balances speed, cost, and confidence."

**5. Clean names and functions.** "Names reveal intent. Functions do one thing at one level of abstraction. Fewer arguments, no hidden side effects."

**6. Code smell.** "A smell is a surface sign of a deeper design problem. It is not a bug. Examples: long method, primitive obsession, feature envy."

**7. Refactoring.** "Changing structure without changing behavior, under passing tests, in small steps. I do not refactor with no tests or in the middle of an urgent fix."

**8. Code review.** "Intent first, then correctness, then design, then tests, then style last. I tag severity so the author knows what blocks."

**9. RFC versus ADR.** "An RFC is a proposal written before deciding, open to change. An ADR records a decision after it is made, and it is not edited later."

**10. Estimation.** "I break work into thin vertical slices. I give a range, not one number. I run a spike for the biggest unknown."

---

# 2. Questions by topic

## Topic A: Reading and navigating code

**Q1. [Fresher] How do you start on a codebase you have never seen?**
- Do not change code first. Read first.
- Find an entry point for the feature you care about (for example a controller, a main method, or a scheduled job).
- Trace one path end to end. Depth on one path beats shallow reading of everything.
- Use IDE search, ripgrep, and git history to locate and explain code.
- Run the tests and the app so you can watch real behavior.

Revise: 01-reading-navigating-code.md

**Q2. [Fresher] What is the difference between go-to-definition and find-usages?**
- Go-to-definition jumps from a name to where it is declared. It answers "what is this?"
- Find-usages lists every place that uses the name. It answers "who depends on this?"
- Find-usages shows the blast radius (the set of things a change could break).
- Always check usages before you change a public method.

Revise: 01-reading-navigating-code.md

**Q3. [Mid] What does "follow the data" mean, and why does it help?**
- Start from the wrong output or the input value, and trace where it is created, changed, and stored.
- Reading classes one by one hides how values really travel.
- For a bug, ask: where did this value come from, and who changed it last?
- Across a message queue the call stack breaks. Follow the message: find the producer, the topic name, and the consumer.

Revise: 01-reading-navigating-code.md

**Q4. [Mid] What can git history tell you that the code cannot?**
- `git blame` shows who last changed a line and in which commit.
- The commit message and linked ticket explain why the line exists.
- `git log` on one file shows how it grew.
- The pickaxe search (`git log -S "text"`) finds when a string was added or removed.
- This stops you from deleting code that looks odd but exists for a good reason.

Revise: 01-reading-navigating-code.md

**Q5. [Senior] How do you make a large codebase easier for the next reader?**
- Add a short architecture overview and a "where to start" map in the repo.
- Keep names honest and modules small, so search works.
- Write tests that show how code is meant to be used. Tests are living examples.
- Onboard others by walking them through one request path.
- Review for readability as a goal, not only for correctness.

Revise: 01-reading-navigating-code.md (then 09-documentation.md)

## Topic B: Debugging

**Q6. [Fresher] Describe your debugging process.**
- Reproduce the bug. Capture the exact input, environment, and steps.
- Hypothesize a cause from evidence, not from a guess.
- Test the hypothesis by changing one thing at a time.
- Find the root cause, not only the symptom.
- Fix it, and add a test that fails without the fix.
- If a test disproves your idea, that is progress. You eliminated one cause.

Revise: 02-debugging.md

**Q7. [Fresher] Why is reproduction "the whole game"?**
- A bug you can trigger on demand is almost solved. One you cannot is guesswork.
- A repro lets you check that your fix works.
- It becomes a regression test (a test that stops the bug returning).
- Capture at least the input data and the environment (version, config, timing).

Revise: 02-debugging.md

**Q8. [Fresher] How do you read a Java stack trace?**
- Read the top line for the exception type and message.
- Find the first frame in your own code, not library code.
- `Caused by:` shows a wrapped exception. The bottom-most cause is usually the real origin.
- Then open that line and ask what value was unexpected.

Revise: 02-debugging.md

**Q9. [Mid] When do you use a debugger, and when do you use logging?**
- Debugger: local, reproducible bugs where you need to inspect state and step through logic. A conditional breakpoint stops only when a condition is true.
- Logging and tracing: production, concurrency, timing issues, and distributed calls where pausing is not possible or changes behavior.
- Choose by access: can you attach and reproduce? If not, use logs.

Revise: 02-debugging.md

**Q10. [Mid] How do you find which commit introduced a bug?**
- Use `git bisect`. Mark one good commit and one bad commit.
- Git checks out the middle. You test and mark good or bad. It halves the range each time.
- For 1,000 commits that is about 10 steps, because log2(1000) is about 10.
- Automate with a script that exits 0 for good and non-zero for bad.
- It breaks when commits do not build or the bug is flaky.

Revise: 02-debugging.md

**Q11. [Senior] How do you debug a production issue when you cannot attach a debugger?**
- Check metrics first (is something wrong, and when did it start?).
- Use traces to see where in the chain it fails.
- Use logs to see details of the failing requests.
- Form a hypothesis, then test it safely: a feature flag, a canary, or extra targeted logging.
- Stabilize users first (rollback if needed), then find the root cause.
- Write down what you learned in a blameless review.

Revise: 02-debugging.md

## Topic C: Unit testing and TDD

**Q12. [Fresher] What is a unit test, and what makes one good?**
- A unit test checks a small piece of behavior in isolation.
- Good tests are fast, isolated, repeatable, and self-checking.
- Isolation means no real database or network. Doubles replace them.
- The test fails for one clear reason.

Revise: 03-testing-unit-tdd.md

**Q13. [Fresher] What is the AAA pattern?**
- Arrange: set up the data and objects.
- Act: call the one thing under test.
- Assert: check the result.
- Blank lines between the three parts make a test readable at a glance.

Revise: 03-testing-unit-tdd.md

**Q14. [Fresher] How should you name a test?**
- The name should say the scenario and the expected result.
- Bad: `test1`. Good: `rejectsOrderWhenStockIsZero`.
- At 2am a failing test name should tell you what broke without opening the code.

Revise: 03-testing-unit-tdd.md

**Q15. [Mid] Explain stub, mock, fake, and spy. What is the rule for stubs and mocks?**
- Stub: returns canned answers.
- Mock: records calls so you can verify them.
- Fake: a simple working version, such as an in-memory repository.
- Spy: a real object that also records calls.
- Rule: stub queries, mock commands. Do not verify calls to things that only return data.

Revise: 03-testing-unit-tdd.md

**Q16. [Mid] What does "test behavior, not implementation" mean?**
- Assert on outputs and visible effects, not on private steps.
- Litmus test: if you refactor and keep behavior, the test should still pass. If it fails, it is brittle.
- Verifying that one confirmation email was sent is behavior, because the email is the required effect.
- Verifying that a private helper was called is implementation.

Revise: 03-testing-unit-tdd.md

**Q17. [Mid] Walk through red-green-refactor.**
- Red: write a failing test for the next small behavior.
- Green: write the simplest code that passes. Hardcoding is allowed here.
- Refactor: clean the code while the tests stay green.
- Watch the test fail first. Otherwise you do not know the test can fail.

Revise: 03-testing-unit-tdd.md

**Q18. [Senior] When is strict TDD the wrong choice?**
- Exploratory spikes where you do not yet know the design.
- Throwaway scripts and UI layout experiments.
- Instead: explore first, then write tests around what you keep, or rewrite it test-first.
- Say clearly that TDD is a tool, not a rule.

Revise: 03-testing-unit-tdd.md

## Topic D: Testing strategy

**Q19. [Fresher] What is the test pyramid?**
- Many unit tests at the base: fast and cheap.
- Fewer integration tests in the middle: real boundaries like a database.
- Very few end-to-end tests at the top: slow but realistic.
- It spends three budgets wisely: time, money, and maintenance effort.
- An upside-down pyramid gives slow, flaky pipelines.

Revise: 04-testing-strategy.md

**Q20. [Mid] Why use Testcontainers instead of an in-memory database for repository tests?**
- Testcontainers runs the real database (for example PostgreSQL) in a container.
- An in-memory database can differ in SQL dialect, types, locking, and constraints.
- A query can pass on the in-memory database and fail in production.
- Trade-off: containers are slower and need Docker. Use them for real boundary tests only.

Revise: 04-testing-strategy.md

**Q21. [Mid] What is a contract test and when is it worth it?**
- It checks that a provider and a consumer agree on an API shape and meaning.
- A unit test with a mock cannot catch a changed field in the real provider.
- Worth it when separate teams own the two sides and they deploy independently.
- Not worth it for a small service with one team and one deploy.

Revise: 04-testing-strategy.md

**Q22. [Mid] Is 100% code coverage a good goal?**
- Coverage shows which lines ran. It does not show that anything was checked.
- A test with no assertions can reach 100% coverage and verify nothing.
- Use coverage to find untested areas, not as a target to chase.
- Better signals: tests tied to risk, and bugs that tests actually catch.

Revise: 04-testing-strategy.md

**Q23. [Mid] What causes flaky tests and how do you fix them?**
- Shared state between tests: isolate data and reset it.
- Time and randomness: inject a clock and fixed seeds.
- Async waits and sleeps: wait on a condition, not a fixed delay.
- Blanket auto-retry is dangerous. It hides real bugs and teaches the team to ignore red.
- Quarantine a flaky test quickly, then fix it.

Revise: 04-testing-strategy.md

**Q24. [Senior] Sketch a test strategy for a payment service.**
- Unit: fee rules, rounding, state transitions, and edge cases.
- Integration: database writes, idempotency keys, and transactions with a real database.
- Contract: the API shared with callers and with the payment provider.
- End-to-end: one or two critical journeys, such as pay and refund.
- Put the most tests where the risk is highest, and say what you will not test.

Revise: 04-testing-strategy.md

## Topic E: Clean code, naming and functions

**Q25. [Fresher] What makes a good name?**
- It reveals intent: `elapsedDays`, not `d`.
- It avoids noise words and misleading names.
- It is pronounceable and searchable.
- It uses one word per concept across the codebase.
- Example: `getThem()` becomes `getFlaggedCells()`.

Revise: 05-clean-code-naming-functions.md

**Q26. [Fresher] What does "a function should do one thing" mean?**
- All steps in the function sit at the same level of abstraction (the same level of detail).
- You can describe it in one sentence without the word "and".
- The step-down rule: code reads top to bottom, high level first, details below.
- Do not over-extract. Stop when a new function would only rename one line.

Revise: 05-clean-code-naming-functions.md

**Q27. [Fresher] Why are boolean flag arguments a smell?**
- `render(true)` is unclear at the call site.
- The flag means the function does two things.
- Fix: split into two functions, such as `renderForPrint()` and `renderForScreen()`.
- Fewer arguments are better. Zero to two is a good aim.

Revise: 05-clean-code-naming-functions.md

**Q28. [Mid] What is command-query separation?**
- A command changes state and returns nothing. A query returns data and changes nothing.
- A query with a hidden side effect surprises callers. Calling it twice gives different results.
- Example: `getNextId()` that also increments is a trap.
- Some APIs (like stack pop) break the rule on purpose. Name them clearly.

Revise: 05-clean-code-naming-functions.md

**Q29. [Mid] When is a comment useful and when is it a smell?**
- Useful: why a decision was made, a warning, a legal note, a link to a spec.
- Smell: restating the code, commented-out code, or a comment that excuses a bad name.
- Comments are not checked by the compiler, so they can become false over time.
- First try to improve the code so the comment is not needed.

Revise: 05-clean-code-naming-functions.md (then 09-documentation.md)

**Q30. [Mid] Exceptions or error codes? What is fail fast?**
- In Java, exceptions keep the happy path clear and cannot be silently ignored like a returned code.
- Go and Rust use result values that the type system forces you to handle.
- Fail fast: check inputs at the boundary and stop early with a clear message.
- An empty `catch` block hides failures and makes bugs very hard to find.

Revise: 05-clean-code-naming-functions.md

**Q31. [Senior] How do you justify clean code to a manager?**
- Code is read far more often than written, so unclear code costs time on every change.
- Show real cost: slow onboarding, repeated bugs, slow reviews.
- Work in small steps tied to features you already deliver. Do not ask for a big pause.
- Pass tests are not enough. Working code that nobody can change safely is debt.

Revise: 05-clean-code-naming-functions.md

## Topic F: Code smells and SOLID

**Q32. [Fresher] What is a code smell, and how is it different from a bug?**
- A smell is a sign of a deeper design problem. The code still works.
- A bug is wrong behavior now. A smell raises the cost and risk of future change.
- Smells compound like interest: small ones attract more mess.
- A smell is a prompt to look, not a command to rewrite.

Revise: 06-clean-code-smells-solid.md

**Q33. [Fresher] Name five smells and the fix for each.**
- Long method: Extract Method.
- Primitive obsession: replace with a value object.
- Feature envy: Move Method to the class that owns the data.
- Duplicated code: extract and reuse.
- Large class: split by responsibility.

Revise: 06-clean-code-smells-solid.md

**Q34. [Mid] What is primitive obsession and how does a value object help?**
- It means using plain `String` or `double` for concepts like money or email.
- A `Money` or `EmailAddress` class holds validation and rules in one place.
- The compiler stops you from passing a price where a quantity is expected.
- Do not wrap everything. Wrap values that have rules or are easy to mix up.

Revise: 06-clean-code-smells-solid.md

**Q35. [Mid] Explain the five SOLID principles briefly.**
- S: a class has one reason to change.
- O: extend behavior without editing tested code.
- L: a subtype must work wherever its parent works.
- I: many small interfaces beat one large one.
- D: depend on abstractions, not concrete classes.
- Smell link: a large class breaks S, and a type-switch often breaks O.

Revise: 06-clean-code-smells-solid.md

**Q36. [Mid] Can SOLID go too far?**
- Yes. Applying it everywhere early creates speculative generality (abstractions for futures that never come).
- Interfaces with one implementation and layers that only pass data add cost.
- Mature approach: react to real change. Add the abstraction when the second case appears.
- A type-switch is fine if it is small, stable, and in one place.

Revise: 06-clean-code-smells-solid.md

**Q37. [Senior] What are guard clauses and boundaries/adapters?**
- Guard clauses return early for invalid cases, so the main logic is not deeply nested. They match fail fast.
- An adapter wraps an outside library or service behind your own interface.
- Benefits: easy to test, easy to swap, and outside changes stay in one place.
- Do not wrap simple, stable things such as standard library collections.

Revise: 06-clean-code-smells-solid.md

## Topic G: Refactoring

**Q38. [Fresher] What is refactoring?**
- Changing the internal structure of code without changing its behavior.
- If behavior changes, it is not refactoring. It is a feature or a fix.
- Typical moves: rename, Extract Method, Move Method, inline.
- The goal is code that is cheaper to understand and change.

Revise: 07-refactoring.md

**Q39. [Fresher] What is the golden rule of refactoring?**
- Refactor only under green tests.
- A red test mid-refactor means you changed behavior. Undo the last step and try a smaller one.
- Do not mix refactoring and behavior change in one commit.

Revise: 07-refactoring.md

**Q40. [Mid] How do you refactor legacy code that has no tests?**
- Write characterization tests first. They record what the code does now.
- They do not claim the behavior is correct. They only protect you from accidental change.
- Use refactorings the IDE performs automatically (like rename) as safe first steps.
- Then make small steps and run the tests after each.

Revise: 07-refactoring.md

**Q41. [Mid] Why separate refactoring commits from feature commits?**
- A reviewer can check "no behavior change" quickly in one, and focus on logic in the other.
- If a bug appears, you can revert the right commit.
- It keeps the history readable and makes bisect more useful.

Revise: 07-refactoring.md (then 08-code-review.md)

**Q42. [Mid] When should you refactor, and when not?**
- Do: before adding a feature in messy code ("make the change easy first"), and when you see a smell while working.
- Do not: with no tests and no time, right before a release, or in code that will be deleted.
- A big "refactoring sprint" each quarter can be a sign that small cleanup is being skipped.

Revise: 07-refactoring.md

**Q43. [Senior] What is the strangler fig pattern and why prefer it to a rewrite?**
- Build new code beside the old system. Route one feature at a time to the new code.
- The old parts shrink until you can remove them.
- A big-bang rewrite delays all value, hides risk, and often misses behavior the old code had.
- Each step is small, shippable, and reversible.

Revise: 07-refactoring.md

## Topic H: Code review

**Q44. [Fresher] In what order do you review a pull request?**
- Intent: what is this change trying to do?
- Correctness: edge cases, errors, security, concurrency.
- Design: is it the right shape? Does it fit the codebase?
- Tests: do they check real behavior?
- Style and naming last. Tools should handle formatting.

Revise: 08-code-review.md

**Q45. [Fresher] Why do teams do code review?**
- Find defects early, when they are cheap.
- Share knowledge so no one person owns a part of the code.
- Keep consistent standards and design.
- Mentor in both directions.
- Linters cannot judge design, intent, or missing cases.

Revise: 08-code-review.md

**Q46. [Mid] What is severity tagging and why use it?**
- Label each comment: blocker, should, nit, question, or praise.
- The author knows what must change and what is optional.
- If everything is marked a blocker, the labels lose meaning and people stop trusting them.

Revise: 08-code-review.md

**Q47. [Mid] How do you make your own pull request easy to review?**
- Keep it small. Large PRs get shallow reviews.
- Write a description: what, why, how to test, risks, and what you want checked.
- Review your own diff first and remove noise.
- Keep refactoring and behavior changes apart.

Revise: 08-code-review.md

**Q48. [Mid] Should a review comment be a command or a question?**
- A question invites discussion: "What happens if the list is empty?"
- A directive is fine for clear rules or blockers: "Please add a null check here."
- Be direct but kind. Comment on the code, not the person.
- Contempt is the real failure, not directness.

Revise: 08-code-review.md

**Q49. [Senior] When do you request changes and when approve with comments?**
- Request changes for correctness, security, data loss, or serious design problems.
- Approve with comments when only nits or optional ideas remain, and trust the author.
- Rubber-stamping and over-blocking both damage the team.
- Switch to a call when comments go back and forth more than two rounds.

Revise: 08-code-review.md

## Topic I: Documentation

**Q50. [Fresher] Why do we still need docs if the code is self-documenting?**
- Clean code cannot express why a choice was made, how to run the system, or the big picture.
- It also cannot show rejected options or business context.
- Code explains how. Docs explain why and where.

Revise: 09-documentation.md

**Q51. [Fresher] What is the most important job of a README?**
- Help a new person run the project quickly.
- The Quickstart section serves that job.
- Test it: ask someone new to follow it on a clean machine.

Revise: 09-documentation.md

**Q52. [Mid] What is the golden rule of inline comments?**
- Explain why, not what.
- Bad: `// add one to i`. Good: `// Provider limits requests to 5 per second, so we wait.`
- Too many comments hide the important ones and go stale.

Revise: 09-documentation.md

**Q53. [Mid] Why do some docs rot, and how do you keep them true?**
- Docs far from the code are not updated when code changes.
- Keep docs in the repo, near the code they describe.
- Review docs in the same pull request as code changes.
- Generate what can be generated, such as API docs from annotations.
- Delete outdated pages rather than leaving wrong ones.

Revise: 09-documentation.md

**Q54. [Senior] Javadoc versus OpenAPI, and what is a changelog for?**
- Javadoc describes code for developers who use or maintain the library.
- OpenAPI describes an HTTP API for clients, often in generated tools.
- A changelog tells users what changed and what they must do, unlike `git log`, which lists every commit.
- Generation saves effort, but a human must still write the why and the migration advice.

Revise: 09-documentation.md

## Topic J: Technical writing: RFCs and ADRs

**Q55. [Mid] What is an RFC (design doc) and why write one before coding?**
- An RFC is a written proposal with a problem, goals, options, and a recommendation.
- Fixing a wrong idea in a document is far cheaper than fixing it in code.
- It gathers feedback early and aligns the team.
- Load-bearing parts: the problem statement and the options with trade-offs.
- An RFC with only one option is not a real comparison.

Revise: 10-technical-writing-rfcs-adrs.md

**Q56. [Mid] What is an ADR and how does it differ from an RFC?**
- An ADR (Architecture Decision Record) records a decision that was made: context, decision, consequences.
- RFC: before the decision, open to change, written to get input.
- ADR: after the decision, written in past or present tense, not edited later.
- If a decision changes, write a new ADR that supersedes the old one.

Revise: 10-technical-writing-rfcs-adrs.md

**Q57. [Mid] What are non-goals and why include them?**
- Non-goals are things the design will deliberately not do.
- They stop the review from drifting into every related topic.
- They set clear scope for estimates and promises.

Revise: 10-technical-writing-rfcs-adrs.md

**Q58. [Senior] How do you get agreement on a design asynchronously?**
- Share a short TL;DR first, then the details.
- Name reviewers and a deadline for comments.
- Answer comments in the doc, and record open questions.
- Buy-in means people understand and can support the decision. It does not need everyone to agree.
- Decide who has the final call before you start.

Revise: 10-technical-writing-rfcs-adrs.md

**Q59. [Senior] When do you write an RFC, an ADR, or nothing?**
- RFC: a large, costly, or cross-team change with real options.
- ADR: a decision that future people will ask "why?" about.
- Nothing: a small, easy-to-reverse change. A pull request description is enough.
- Writing a doc for a trivial change wastes time.

Revise: 10-technical-writing-rfcs-adrs.md

## Topic K: Estimation and breaking down work

**Q60. [Fresher] Why estimate if estimates are often wrong?**
- Estimates help people plan, set priorities, and decide whether to start.
- A good estimate states its uncertainty.
- Refusing to estimate does not remove the question. Others will guess instead.

Revise: 11-estimation-breaking-down-work.md

**Q61. [Mid] How do you turn a vague ticket into scoped work?**
- Ask clarifying questions about the goal, users, and what "done" means.
- Write assumptions and non-goals down.
- Split the work into small pieces.
- Good clarifying questions show care, not confusion.

Revise: 11-estimation-breaking-down-work.md

**Q62. [Mid] What is a vertical slice and why prefer it to horizontal layers?**
- A vertical slice delivers a thin piece of working behavior through all layers: UI, logic, data.
- Horizontal layers (all database first, then all API) delay integration and risk to the end.
- A slice can be shipped and tested within about a day or two.
- Example: "user can save one address" instead of "build the address table."

Revise: 11-estimation-breaking-down-work.md

**Q63. [Mid] Explain a three-point (PERT) estimate with O=2, M=5, P=14 days.**
- Formula: (O + 4M + P) / 6 = (2 + 20 + 14) / 6 = 6 days.
- O is optimistic, M is most likely, P is pessimistic.
- The wide gap between 2 and 14 shows high uncertainty.
- Report the range and the main risks, not only the 6.

Revise: 11-estimation-breaking-down-work.md

**Q64. [Mid] What is the cone of uncertainty?**
- Early in a project the true cost can be far above or below the estimate.
- The range narrows as you learn more and build.
- So an early wide range is honest and useful. A confident single number is not.
- Re-estimate as facts arrive.

Revise: 11-estimation-breaking-down-work.md

**Q65. [Senior] Hidden padding versus an explicit buffer, and what is a spike?**
- Hidden padding inflates every task quietly. It erodes trust and the work expands to fill it.
- An explicit buffer is a visible, named amount for known risk.
- A spike is a short, time-boxed experiment to answer one unknown. It moves you down the cone.
- Good spikes have a clear question, a time limit, and a written answer.

Revise: 11-estimation-breaking-down-work.md

## Topic L: Capstone, interviews and growth

**Q66. [Fresher] What is the craft loop?**
- Read, debug, test, clean, refactor, review, document, write, estimate: the skills of files 01 to 11 used together.
- It is a menu, not a ceremony. Pick what the task needs.
- A one-line typo fix does not need an RFC.

Revise: 12-capstone-interview-promotion.md

**Q67. [Mid] How do you make an interview answer more than a recited script?**
- Anchor the method in one real story with specific details.
- Say what you did, what you learned, and what you would change.
- Answer the question asked. Do not list every skill you know.

Revise: 12-capstone-interview-promotion.md

**Q68. [Senior] How does a craft skill look different at each level?**
- Junior: does it well on their own work.
- Mid: does it consistently and independently.
- Senior: raises the team's practice by teaching, reviewing, and setting standards.
- Staff: builds systems and culture so the whole organization does it by default.
- The pattern is the scope of impact growing from self to team to organization. Levels vary by company.

Revise: 12-capstone-interview-promotion.md

**Q69. [Mid] How do you use a gap analysis?**
- Rate your honest confidence in each skill.
- Pick the weakest one and write one concrete action for this week.
- The actions matter more than the scores.
- Repeat it every few months.

Revise: 12-capstone-interview-promotion.md

---

# 3. Behavioural questions

## The STAR method

STAR gives your story a clear shape. Keep the whole answer to about two minutes.

- **S: Situation.** One or two sentences of context. Where, when, what was at stake.
- **T: Task.** Your specific responsibility. Not the team's, yours.
- **A: Action.** What you did, step by step. This is the longest part. Use "I", not "we".
- **R: Result.** The outcome, with a number if possible, and what you learned.

Tips:
- Choose real stories. Interviewers ask follow-up questions, and invented detail falls apart.
- Do not blame people. Describe actions and facts.
- End with a lesson. Showing growth matters more than a perfect story.
- Below, each model example is invented and generic. Replace it with your own.

## B1. Tell me about a conflict in a code review.

**What they look for:** respect, focus on the code, and ability to reach agreement.

**STAR structure:**
- S: A review where you and the author disagreed.
- T: You needed to protect quality without damaging the working relationship.
- A: You asked questions, explained your reasoning, and used evidence or a call.
- R: A decision, and a better relationship or process.

**Model example (generic):** "A teammate's pull request added a cache with no expiry (Situation). I was the reviewer and worried about stale data (Task). I did not just write 'wrong'. I asked what happens when a price changes, and I added a small test showing stale output. We then had a ten-minute call. They agreed to add an expiry, and I accepted their simpler design for the rest (Action). The change merged the next day, and we added a note in our review guide about cache expiry (Result). I learned that a question plus an example works better than a verdict."

Revise: 08-code-review.md

## B2. Tell me about a time you missed a deadline.

**What they look for:** honesty, early warning, and learning.

**STAR structure:**
- S: A commitment that slipped.
- T: Your ownership of the date.
- A: When you noticed, who you told, and what options you gave.
- R: The outcome and what you changed in your estimating.

**Model example (generic):** "I had promised a reporting feature in three weeks (Situation). I owned the estimate and the delivery (Task). In week two I found that the data format was messier than I assumed. I told my lead that day, showed what was done, and offered two options: ship a smaller version on time, or the full one a week later. We chose the smaller version first (Action). The users got a working report on time, and the rest followed a week later. Afterwards I began giving ranges and running a short spike on unknowns before committing (Result)."

Revise: 11-estimation-breaking-down-work.md

## B3. Tell me about a production bug you caused.

**What they look for:** ownership, calm response, and root-cause thinking without blame.

**STAR structure:**
- S: What broke and who was affected.
- T: Your role in the fix.
- A: Detection, stop the harm, find the root cause, fix, prevent.
- R: Impact, and the lasting improvement.

**Model example (generic):** "A change of mine removed a null check, and an order page failed for some users (Situation). I was the author, so I took the lead on the fix (Task). I rolled the change back first to stop the errors. Then I reproduced the failure with the odd input, wrote a failing test, and shipped a proper fix. I wrote a short blameless review: the cause was a missing test for empty data, and our review did not look at that path (Action). We added the test and a checklist item. Errors stopped within minutes of the rollback, and the same bug class did not return (Result)."

Revise: 02-debugging.md

## B4. Tell me about a time you disagreed with a design.

**What they look for:** reasoned disagreement, evidence, and the ability to commit afterwards.

**STAR structure:**
- S: A design you doubted.
- T: Your wish to raise the concern properly.
- A: You wrote down your concern, offered an alternative with trade-offs, and listened.
- R: The decision, and how you supported it.

**Model example (generic):** "Our team planned to split a small service into three services (Situation). I thought it was too much for our team size (Task). I wrote a short RFC-style note comparing a single module with clear boundaries to three services, with the costs of each. I shared it before the meeting and asked for objections (Action). The team chose a middle path: one deployable with strict module boundaries, and a rule to split later if load required it. I wrote an ADR to record the reason (Result). Even when I lose such an argument, I commit to the decision once it is made."

Revise: 10-technical-writing-rfcs-adrs.md

## B5. Tell me about a time you mentored someone.

**What they look for:** patience, teaching method, and results that belong to the other person.

**STAR structure:**
- S: Who they were and what they struggled with.
- T: Your role as helper.
- A: How you taught: examples, pairing, questions, feedback.
- R: Their growth, and what you learned about teaching.

**Model example (generic):** "A new teammate struggled to find where bugs came from (Situation). I volunteered to help them in their first month (Task). Instead of fixing the bugs myself, we paired on one. I asked them to reproduce it, state a hypothesis, and test it. Then I gave them a small bug alone and reviewed their notes (Action). After six weeks they solved their own bugs and began to explain their steps in standups. I learned to ask questions before giving answers (Result)."

Revise: 02-debugging.md

## B6. Tell me about a time you worked with unclear requirements.

**What they look for:** clarifying questions, small safe steps, and communication.

**STAR structure:**
- S: A vague task.
- T: Your need to move forward without wasting effort.
- A: Questions, written assumptions, a thin first slice, early feedback.
- R: The result and how ambiguity was reduced.

**Model example (generic):** "I was asked to 'improve search' with no details (Situation). I needed to start without building the wrong thing (Task). I asked who used search and what failed for them, and looked at the top failed queries. I wrote my assumptions in a short doc and shared it. Then I built one thin slice, handling spelling mistakes, and showed it after two days (Action). The product owner said that was the right direction, and we added scope in steps. The assumption doc prevented two arguments later (Result)."

Revise: 11-estimation-breaking-down-work.md

## B7. Tell me about a time you had to learn a large codebase quickly.

**What they look for:** a method, not panic.

**STAR structure:**
- S: A new or inherited system.
- T: Become useful by a set date.
- A: Entry points, one traced path, tests, git history, asking questions.
- R: A first contribution, and docs that help others.

**Model example (generic):** "I joined a team with a five-year-old billing system (Situation). I had to fix a rounding bug in my second week (Task). I found the entry point for the invoice request, traced one invoice from request to database, and read the tests for the rounding class. `git blame` showed why a strange rule existed. I asked the original author one precise question (Action). I fixed the bug with a test and wrote a one-page map of the invoice path for the next new person (Result)."

Revise: 01-reading-navigating-code.md

## B8. Tell me about a time you improved code quality without being asked.

**What they look for:** judgment, safety, and small steps.

**STAR structure:**
- S: A painful area of the code.
- T: Your choice to improve it.
- A: Tests first, small steps, separate commits, clear review.
- R: Measured benefit.

**Model example (generic):** "A 300-line method handled discounts and caused many bugs (Situation). I was about to add a new discount type and decided to make the change easy first (Task). I wrote characterization tests, then extracted small methods in separate commits, each with green tests (Action). The new discount took an hour instead of a day, and the reviewer could confirm each refactoring commit changed no behavior. I did not clean unrelated code, so the pull request stayed focused (Result)."

Revise: 07-refactoring.md

## B9. Tell me about feedback that was hard to hear.

**What they look for:** openness and change in behavior.

**STAR structure:**
- S: The feedback and who gave it.
- T: Your need to respond well.
- A: You listened, asked for examples, and made a plan.
- R: A visible change.

**Model example (generic):** "A reviewer told me my pull requests were too large to review well (Situation). I felt defensive at first, but I asked for examples (Task). I saw that my last three changes mixed refactoring with new features. I began to split work into small vertical slices and write clear descriptions (Action). Reviews got faster, from two days to a few hours, and comments became more useful. I thanked the reviewer, and I now share this habit when I mentor others (Result)."

Revise: 08-code-review.md

## B10. Tell me about a time you pushed back on a stakeholder's request or date.

**What they look for:** honesty and the ability to offer options.

**STAR structure:**
- S: A request or date that was not realistic.
- T: Protect quality and trust.
- A: Explain with facts, show options, and ask for a decision.
- R: A decision everyone understood.

**Model example (generic):** "A manager asked for a new payment option in one week (Situation). I knew that safe testing alone needed more time (Task). I did not say no. I showed a range, explained the main risks, and offered three options: a smaller version in one week, the full version in three weeks, or one week with extra risk and manual checks. I asked them to choose (Action). They chose the smaller version first. The release was calm, and the full version came later. They also began to ask for ranges (Result)."

Revise: 11-estimation-breaking-down-work.md

## B11. Tell me about a time a test or build was unreliable and what you did.

**What they look for:** systematic thinking and care for the team's trust in tests.

**STAR structure:**
- S: A flaky test or pipeline.
- T: Restore trust.
- A: Isolate the cause, fix, and prevent.
- R: Fewer failures.

**Model example (generic):** "One integration test failed about one run in ten, and people kept pressing 'retry' (Situation). I decided to find the cause instead of adding retries (Task). I ran it in a loop and found two tests shared one database table and ran in different orders. I gave each test its own data and removed a fixed sleep by waiting on a condition (Action). The failure rate dropped to zero over the next month, and the team stopped ignoring red builds (Result)."

Revise: 04-testing-strategy.md

## B12. Tell me about a time you wrote something that influenced a decision.

**What they look for:** clear writing and the ability to guide a team.

**STAR structure:**
- S: A decision that needed input.
- T: Write a proposal.
- A: Problem first, options, trade-offs, recommendation, feedback loop.
- R: A decision, and a record of it.

**Model example (generic):** "Our team argued about which queue technology to use (Situation). I volunteered to write the proposal (Task). I opened with a short summary, listed goals and non-goals, compared three options with costs and risks, and gave one recommendation. I shared it, set a deadline for comments, and answered each point in the doc (Action). The team chose an option in one week without another long meeting, and I recorded it as an ADR so new people could see why (Result)."

Revise: 10-technical-writing-rfcs-adrs.md

---

# 4. Code-review exercises

For each exercise, read the code first and write your own list of comments. Then compare with the reviewer notes. Remember the order from file 08: intent, correctness, design, tests, then style.

## R1. JavaScript: loose equality and silent failure

```javascript
function getDiscount(user, total) {
  if (user.type == "premium") {
    return total * 0.1;
  }
  try {
    return total * user.discount;
  } catch (e) {
  }
}
```

**A strong reviewer would say:**
- Blocker: the empty `catch` hides errors. Also, multiplication by `undefined` does not throw. It returns `NaN`, so the `try` does nothing useful.
- Blocker: the function returns `undefined` on the catch path. Callers get an unclear result. Decide on a clear value, such as `0`.
- Should: use `===`, not `==`, to avoid type surprises.
- Should: replace magic numbers like `0.1` with a named constant such as `PREMIUM_RATE`.
- Question: what is `user.discount` when missing? Add a guard clause with a default.
- Should: no tests are shown. Ask for cases: premium, regular with discount, regular without discount, negative total.
- Nit: the name `getDiscount` returns an amount. `calculateDiscount` is clearer.

Revise: 05-clean-code-naming-functions.md, 08-code-review.md

## R2. Java: SQL injection and resource leak

```java
public User findUser(String name) throws Exception {
    Connection c = DriverManager.getConnection(URL, USER, PASS);
    Statement s = c.createStatement();
    ResultSet r = s.executeQuery(
        "SELECT * FROM users WHERE name = '" + name + "'");
    if (r.next()) {
        return new User(r.getString("name"), r.getString("email"));
    }
    return null;
}
```

**A strong reviewer would say:**
- Blocker, security: string concatenation allows SQL injection. Use a `PreparedStatement` with a parameter.
- Blocker: the connection, statement, and result set are never closed. Use try-with-resources.
- Should: a new connection per call is slow. Use a connection pool injected into the class.
- Should: returning `null` forces callers to check. Return `Optional<User>`.
- Should: `throws Exception` is too broad. Use a specific exception or wrap it with context.
- Nit: avoid `SELECT *`. List the columns you need.
- Question: is there an index on `name`, and can names repeat? `r.next()` takes the first match only.
- Should: a test with a real database (see Testcontainers) would catch mapping errors.

Revise: 08-code-review.md, 04-testing-strategy.md

## R3. JavaScript: async bug and unclear flow

```javascript
async function saveAll(items) {
  items.forEach(async (item) => {
    await db.save(item);
  });
  console.log("saved");
  return true;
}
```

**A strong reviewer would say:**
- Blocker, correctness: `forEach` does not wait for async callbacks. The function logs "saved" and returns `true` before saves finish. Errors become unhandled rejections.
- Fix options: `await Promise.all(items.map(i => db.save(i)))` for parallel work, or a `for...of` loop with `await` for sequential work.
- Question: does order matter, and can the database take many parallel writes? Large arrays may need a limit on concurrency.
- Should: decide what happens when one save fails. Is a partial save acceptable?
- Should: returning `true` carries no information. Either return the saved results or return nothing and throw on failure.
- Nit: use the project's logger, not `console.log`.
- Should: a test with a failing save would have shown the bug.

Revise: 02-debugging.md, 08-code-review.md

## R4. Java: long method with mixed levels and a flag argument

```java
public void process(Order o, boolean sendEmail) {
    if (o != null) {
        if (o.getItems().size() > 0) {
            double total = 0;
            for (Item i : o.getItems()) {
                total += i.getPrice() * i.getQty();
            }
            if (total > 100) { total = total * 0.9; }
            o.setTotal(total);
            repo.save(o);
            if (sendEmail) {
                mailer.send(o.getCustomer().getEmail(), "Thanks " + total);
            }
        }
    }
}
```

**A strong reviewer would say:**
- Should: the method does many things: validate, calculate, discount, save, and notify. Split it by level of abstraction.
- Should: the boolean `sendEmail` is a flag argument. Split into two methods or move notification to the caller.
- Should: use guard clauses to remove the nesting: `if (o == null || o.getItems().isEmpty()) return;`.
- Blocker: `double` for money causes rounding errors. Use `BigDecimal` or a `Money` value object (primitive obsession).
- Should: `100` and `0.9` are magic numbers. Name them.
- Smell: feature envy. The total calculation belongs on `Order`, for example `order.calculateTotal()`.
- Question: should the save and the email be in one transaction? What if the email fails after the save?
- Should: tests for each rule: empty order, exactly 100, over 100.

Revise: 05-clean-code-naming-functions.md, 06-clean-code-smells-solid.md

## R5. JavaScript: mutation and a vague name

```javascript
function fix(list) {
  for (let i = 0; i < list.length; i++) {
    if (list[i].active === false) {
      list.splice(i, 1);
    }
  }
  return list;
}
```

**A strong reviewer would say:**
- Blocker, correctness: removing items while looping skips the next element. Two inactive items in a row leave one behind.
- Blocker: the function changes its input and also returns it. This is a hidden side effect, and callers may not expect it.
- Fix: `return list.filter(item => item.active);` which is clear and does not mutate.
- Should: the name `fix` says nothing. Use `removeInactive` or `getActiveItems`.
- Question: is `active` ever `undefined`? `=== false` and `!item.active` behave differently.
- Should: add tests with consecutive inactive items and an empty list.
- Praise: the `===` use is correct. It is useful to say what is good.

Revise: 05-clean-code-naming-functions.md

## R6. Java: tests that check implementation

```java
@Test
void test1() {
    OrderService service = new OrderService(repo, mailer, clock);
    when(repo.findById(1)).thenReturn(order);
    service.cancel(1);
    verify(repo).findById(1);
    verify(order).setStatus("CANCELLED");
    verify(mailer, times(1)).send(any(), any());
    verifyNoMoreInteractions(repo);
}
```

**A strong reviewer would say:**
- Should: the name `test1` hides the scenario. Use `cancelledOrderBecomesCancelledAndCustomerIsNotified`.
- Should: `verify(repo).findById(1)` and `verifyNoMoreInteractions(repo)` test the internal steps. A harmless refactor will break them. This is brittle.
- Question: `findById` is a query. The rule is stub queries, mock commands. Do not verify it.
- Good: verifying one email was sent checks a visible effect of the behavior. Keep it, but assert on the recipient and subject, not `any()`.
- Should: use a real `Order` object, or a builder, and assert on its final status. A mock for a plain data object adds noise.
- Should: add no-blank-line structure, with Arrange, Act, Assert clearly separated.
- Missing cases: order not found, order already shipped, mail failure.

Revise: 03-testing-unit-tdd.md

## R7. JavaScript: leaked secret and poor error handling

```javascript
const API_KEY = "sk-live-12345-example";

app.get("/user/:id", async (req, res) => {
  const r = await fetch(`https://api.example.com/users/${req.params.id}`, {
    headers: { Authorization: API_KEY },
  });
  const data = await r.json();
  res.send(data);
});
```

**A strong reviewer would say:**
- Blocker, security: the secret is in source code and will be in git history. Move it to an environment variable or secret store, and rotate this key because it is exposed.
- Blocker: no error handling. If the fetch fails or the status is not 2xx, the user sees a crash or wrong data. Check `r.ok` and handle errors.
- Should: validate `req.params.id` before using it in a URL. Encode it with `encodeURIComponent`.
- Should: forwarding the whole upstream response can leak internal fields. Return only needed fields.
- Should: add a timeout to the outside call so a slow service does not hang requests.
- Question: should the outside call sit behind an adapter, so the route is easy to test?
- Should: tests for 404 from upstream, timeout, and bad id.

Revise: 06-clean-code-smells-solid.md, 08-code-review.md

## R8. Java: comments that lie and a swallowed exception

```java
// Returns the user's age in years
public int getAge(User u) {
    int a = 0;
    try {
        // calculate age
        a = (int) ChronoUnit.DAYS.between(u.getBirthDate(), LocalDate.now()) / 365;
    } catch (Exception e) {
        // ignore
    }
    return a;
}
```

**A strong reviewer would say:**
- Blocker: the empty `catch` hides a null birth date. The method then returns `0`, which looks like a valid baby. Fail fast with a clear exception, or return `OptionalInt`.
- Should: `days / 365` is wrong across leap years and near birthdays. Use `Period.between(birth, LocalDate.now()).getYears()`.
- Note on precedence: the cast `(int)` applies to the result of `between` before the division. It works here, but it is hard to read. The `Period` call removes the problem.
- Should: `// calculate age` repeats the code and adds nothing. Remove it.
- Should: `a` and `u` are poor names. Use `birthDate` and `user`.
- Should: inject a `Clock` so tests can fix "today".
- Should: test a birthday today, a day before, a leap-day birth, and a null date.

Revise: 05-clean-code-naming-functions.md, 03-testing-unit-tdd.md

---

# 5. Common mistakes in craft and behavioural interviews

## Mistakes in craft questions

1. **Reciting a memorized script.** Interviewers notice. Tie each method to a real example from your work or a project.
2. **Jumping to a fix.** In a debugging question, start with reproduction and a hypothesis. Never say "I would just add a try-catch."
3. **Listing every technique.** Depth on two or three relevant points beats a shallow list of ten.
4. **Dogma.** Phrases like "always use TDD" or "never write comments" sound inexperienced. Show trade-offs and when you would not do it.
5. **Treating coverage or the number of tests as the goal.** Talk about risk and what the tests prove.
6. **Reviewing style first.** In a code-review exercise, find the correctness and security problems before naming nits.
7. **Giving a single-number estimate.** Offer a range, key assumptions, and a way to reduce uncertainty.
8. **Ignoring the question behind the question.** "Review this code" may also test how kind and clear you are. Say comments the way you would write them.
9. **Refactoring without tests.** If you say you would restructure legacy code, say how you would protect behavior first.
10. **Forgetting to ask clarifying questions.** Ask about scope, users, and constraints before you design or estimate.
11. **Silence while thinking.** Think aloud so the interviewer can follow your reasoning and help you.
12. **Overclaiming.** If you do not know a tool or pattern, say so and describe how you would learn it. Honesty is better than a wrong answer.

## Mistakes in behavioural questions

1. **Saying "we" for everything.** The interviewer needs to know what you did. Use "I" for your actions and credit the team separately.
2. **No result.** Always finish with the outcome and, if possible, a number or a clear effect.
3. **Blaming others.** Describe facts and your own actions. Do not criticize former colleagues or employers.
4. **A story where you were never wrong.** For failure questions, give a real mistake and a real lesson. A fake weakness sounds dishonest.
5. **Too long or too vague.** Aim for about two minutes. Include one specific detail, such as a date, a size, or a quote.
6. **Inventing details.** Follow-up questions expose made-up stories. Use real, smaller stories instead of impressive false ones.
7. **No learning.** End with what you do differently now.
8. **Story with no conflict or stakes.** Choose an example where something was at risk, so the action matters.
9. **Using the same story for every question.** Prepare five to six varied stories that you can adapt.
10. **Leaking private information.** Do not share confidential data, names, or numbers from past employers. Generalize them.
11. **Ignoring the question's focus.** If asked about disagreement, do not tell a story about teamwork with no disagreement.
12. **Promising outcomes.** Do not assume an interview or a promotion will follow from preparation. Work on skill and honest communication, and accept that results depend on many factors.

## A last checklist before the interview

- Can I explain the craft loop in one minute?
- Do I have five real stories mapped to B1 to B12?
- Can I review a small snippet in the right order out loud?
- Do I know my weakest skill and what I am doing about it?
- Do I have two clarifying questions ready for any design or estimation task?

Good luck, and keep practicing on real code.
