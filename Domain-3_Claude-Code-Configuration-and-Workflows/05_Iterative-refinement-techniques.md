# Topic 3.5 — Apply Iterative Refinement Techniques for Progressive Improvement

**Domain:** 3 — Claude Code Configuration & Workflows (20% of exam)
**Task Statement:** 3.5
**Topic:** 17 of 30

---

## 1. Core Idea

Topics 3.1–3.4 were about **configuration** (files, scopes, modes). This one is about **the conversation itself**: what you do on turn 2, 3, 4 when the first output isn't right.

The exam frames this as a **diagnosis problem**. A developer gets a wrong or inconsistent result, and four plausible "try again" moves are offered. The correct one depends entirely on **why** the first attempt failed.

### The master mapping (memorize this)

| Failure symptom | Root cause | Correct refinement technique |
|---|---|---|
| Output inconsistent across runs / across items; "interpreted it differently each time" | Ambiguous **specification** | **Concrete input/output examples** |
| Output looks right but is subtly wrong / breaks on edge cases | No **executable definition of correct** | **Test-driven iteration** |
| Output solves the wrong problem; misses considerations you didn't know existed | **Incomplete requirements**, and you don't know it | **Interview pattern** |
| Several defects at once; fixes keep breaking each other | Wrong **batching strategy** | Single message for **interacting** issues; sequential for independent ones |
| Quality **degrading** over many turns; model cites things that no longer exist | **Context pollution** | Fresh session + structured summary (links to 1.7) |

Most 3.5 items are this table rendered as a scenario.

---

## 2. Concrete Input/Output Examples

**Knowledge point:** concrete input/output examples are *the most effective way* to communicate expected transformations when prose descriptions are interpreted inconsistently.

### Why prose fails
Prose is lossy. "Normalize the phone numbers to a standard format" reads clearly to you because you have shared team context. Claude doesn't, so it fills the gaps — and fills them **differently each run, and differently across items in the same run**. That symptom is **inconsistency**, not incompetence.

### Why examples work
An example is a *specification by demonstration*. It pins down what prose leaves floating:

```
Input:  "(555) 123-4567 ext. 89"   → Output: "+15551234567;ext=89"
Input:  "555.123.4567"              → Output: "+15551234567"
Input:  "+44 20 7946 0958"          → Output: "+442079460958"
```

Three lines answered: which E.164 flavor, whether extensions survive, non-US handling, separator treatment. Prose would need a paragraph and still miss one.

### The 2–3 examples rule
The guide specifies **2–3 concrete input/output examples**. Why that number:

- **One example** is ambiguous about *which* feature matters — Claude can't separate the rule from the coincidence.
- **2–3 examples** span the space: canonical case, variant, edge case. The **contrast between them is the rule**.
- **Many examples** stop adding information, burn context, and can overfit if they're all the same shape.

**Design principle:** examples must **differ along the dimension you care about**. If the ambiguity is "what happens to nulls," one example must contain a null. Examples that all look alike teach nothing past the first.

### Anti-patterns
- **Rewriting the prose more emphatically** — adding `IMPORTANT:`, `You MUST`, more adjectives, a longer paragraph. Emphasis adds no *information*; an emphatically under-specified instruction is still under-specified. The flagship distractor because it *looks* like diligence.
- **"Just show it the code and let it infer the format"** — that's another artifact to interpret, not a specification.
- **Lowering temperature to fix inconsistency** — freezes cross-run variance onto an arbitrary interpretation (possibly wrong, now reproducibly wrong) and does nothing about *within-run* inconsistency. Consistency ≠ correctness.

### Where examples should live (cross-links 3.1 / 3.3)
- **One-off transformation** → examples in the prompt.
- **Recurring project convention** → examples embedded in **CLAUDE.md** (universal) or a **path-scoped `.claude/rules/` file** (per file type). Re-pasting the same examples every session treats a systemic problem as a per-conversation one.

---

## 3. Test-Driven Iteration

**Knowledge point:** write the test suite **first**, then iterate by **sharing test failures** to guide progressive improvement.

### Why it's in an architect exam
Tests are the **objective feedback signal** that closes the refinement loop. Without them, refinement is eyeballing diffs and saying "hmm, still not right" — subjective, low-bandwidth, slow. With them, each iteration produces a precise, machine-generated, unambiguous statement of what's wrong.

### What the suite must cover (all three, before implementation)
1. **Expected behavior** — the happy path
2. **Edge cases** — nulls, empties, boundaries, malformed input
3. **Performance requirements** — the non-functional constraint prose usually drops

Point 3 matters: "make it fast" is prose; `assert elapsed < 900s for 40M rows` is a specification. Encoding throughput up front is what prevents a correct-but-quadratic implementation being discovered in the maintenance window.

### The loop
```
write tests (behavior + edges + perf)
   → Claude implements
   → run tests
   → paste the FAILURES back verbatim
   → Claude fixes
   → repeat until green
```

**Share the failures, not a paraphrase.** Test name, expected, actual, stack trace is high-density unambiguous context. "The migration test is failing on some rows" throws away exactly the precision that makes the technique converge.

### Sub-technique: edge cases via specific test cases
The guide calls this out separately — *providing specific test cases with example input and expected output to fix edge case handling (e.g., null values in migration scripts)*. This is techniques 1 and 2 **fused**: a failing test case *is* an input/output example with an executable assertion attached. When told "the script crashes on null `middle_name`," the answer isn't "handle nulls properly" — it's a test case with the null input and the expected output row.

### The ordering trap (major exam distractor)
**"Implement first, then generate a comprehensive test suite to verify the implementation"** is WRONG.

Tests written *after* an implementation encode **what the code does**, not **what the requirement was**. They will be comprehensive, they will be green, and they will be **green on the bug** — a confident regression harness built around a defect. TDD works because the suite is an **independent** statement of correctness the implementation must be bent to satisfy. Letting the implementation shape the tests inverts that authority and destroys the feedback signal.

Watch the phrasing: *"generate tests to verify the implementation"* ≠ *"write tests, then implement."*

### When TDD is the right tool
- **Good fit:** correctness is verifiable and objective — data transformations, migrations, parsers, API contracts, algorithms.
- **Poor fit:** "correct" is a judgment call — visual design, prose tone, architectural taste. You can't write the assertion, so use examples or the interview pattern.
- **Prerequisite:** you must be able to write a *correct definition of correct*. In an unfamiliar domain you can't, so interview first, then TDD.

---

## 4. The Interview Pattern

**Knowledge point:** have **Claude ask *you* questions** to surface considerations the developer may not have anticipated — **before implementing**.

### The inversion
Normally you specify and Claude executes. Here: *"Before you write anything, ask me the questions you need answered to build this well."* The value is in the questions you **couldn't have asked yourself**, because you didn't know the domain well enough to know they existed.

The guide's own examples of what gets surfaced: **cache invalidation strategies, failure modes**.

### The trigger condition (the exam discriminator)
Use the interview pattern in **unfamiliar domains** — the guide says it explicitly: *"before implementing solutions in unfamiliar domains."*

| Situation | Correct technique |
|---|---|
| Know the domain, requirements clear | Direct execution |
| Know the domain, spec ambiguous | Examples or tests — you *can* write the spec, you just wrote it badly |
| **Do NOT know the domain** | **Interview pattern** |

**Key insight:** interview is for **unknown unknowns**; examples and tests are for **known-but-badly-expressed** requirements.

> **You cannot exemplify or assert your way out of a requirement you don't know exists.**

### The second half of the pattern
Half the value is asking; the other half is that **your answers become the specification**. They should be captured/consolidated (into the prompt, plan, or design doc) and confirmed *before* implementation — not left scattered across chat turns. An item may contrast:
- "ask clarifying questions, then start coding immediately" (weaker)
- "ask clarifying questions, answer them, confirm consolidated requirements, then implement" (stronger)

### Anti-patterns
- Interviewing on a trivial, well-understood single-file change — real overhead, no risk reduction (conversational cousin of plan mode for a typo fix).
- **Confusing high stakes with unfamiliarity.** High risk raises the bar on **verification** (→ TDD), not on requirements discovery. A team that has done five migrations doesn't gain from being interviewed just because this one hits production.

### Interview pattern vs. plan mode (3.4)
| | Reduces uncertainty about | Mechanism |
|---|---|---|
| **Plan mode** | **The code** | Enforced read-only state; Claude explores and proposes an approach for human approval |
| **Interview pattern** | **The requirements** | Prompting technique; Claude extracts what's in your head and not in the codebase |

They compose: interview first to fix requirements, then plan mode to fix the approach.

---

## 5. Batching: One Message vs. Sequential Iteration

**Knowledge point:** provide all issues in a **single message when the problems interact**; fix them **sequentially when they're independent**.

### Interacting issues → one detailed message
When fixes touch the same code paths, shared state, or each other's assumptions, one-at-a-time is actively harmful:
- Fix #1 is designed without knowledge of constraint #2, so it's structurally wrong.
- Fix #2 partially undoes fix #1.
- You churn, and may land in a **local optimum** neither fix would have chosen with full information.

Given all at once, Claude designs **one coherent solution** satisfying the whole constraint set — often simpler than the sum of the individual patches.

### Independent issues → sequential
Unrelated defects in unrelated modules. Batching costs:
- **Attention dilution** — many unrelated concerns in one prompt degrades quality on each (same principle as per-file review passes in 1.6).
- **Attribution loss** — a regression can't be traced to one of five changes.
- **Reviewability** — one commit doing five unrelated things is hard to review or revert.

### The decision test
> **Would knowing about issue B change how I'd fix issue A?**
> Yes → same message. No → sequential.

**Batch by dependency, never by count or surface similarity.** "Two balanced batches of two" is tidiness, not coupling analysis.

### Both extremes are distractors
- Dumping 20 unrelated bugs into one giant message ("comprehensive!") → shallow fixes.
- Feeding 3 tightly-coupled bugs one at a time ("methodical!") → fix thrash and rework.

Neither "always batch" nor "always sequential" is correct. **Coupling decides.**

---

## 6. Knowing When to Stop Iterating (links to 1.7)

Refinement is a loop, but the exam also tests knowing when the loop has gone bad.

### Symptoms of context pollution
- Many turns invested; quality **degrading rather than converging**
- Each fix introduces a new problem
- Multiple abandoned approaches still sitting in history
- Superseded versions of the same file in context
- **Smoking gun:** the model references code that was removed several turns ago

### The correct move
Not a sixth iteration. **Start a fresh session with a structured summary** carrying:
1. The confirmed requirements
2. **The approaches that failed and why** (this is what stops the fresh session re-proposing a rejected design)
3. The true current state of the code

Guide (1.7): *"why starting a new session with a structured summary is more reliable than resuming with stale tool results"* and *"choosing between session resumption (when prior context is mostly valid) and starting fresh with injected summaries (when prior tool results are stale)."*

A clean context with distilled knowledge beats a long context with accumulated noise. This is **not** starting over — you keep the knowledge, discard the noise.

**Partial-fix trap:** "re-read the files from disk and continue" repairs only the stale-file symptom. The rejected architectures, superseded instructions, and dead-end reasoning all remain in context. Valid only when prior context is *otherwise mostly valid*.

---

## 7. The Unifying Principle

Every technique replaces **describing** with **demonstrating or extracting**:

| Vague | Concrete |
|---|---|
| Prose description | Input/output examples |
| "It doesn't work" | Test failure output |
| Your incomplete mental model | Questions Claude asks you |
| A pile of complaints | Correctly-batched constraint set |

**Meta-skill:** when an output is wrong, **diagnose the failure before choosing the fix**. Inconsistent → examples. Unverified → tests. Unknown domain → interview. Multiple defects → check coupling. Degrading over many turns → fresh session.

**Re-prompting harder is never the answer.**

---

# Practice Questions

## Question 1

A developer is using Claude Code to write a script that normalizes messy customer address records from an acquired company into the parent company's canonical format. They wrote a detailed paragraph describing the rules: how to expand abbreviations, how to handle apartment/suite designators, how to standardize state codes, and how to treat missing postal codes.

The script runs, but the results are inconsistent. Some records get `"Apt 4B"`, others get `"#4B"`, others get `"Unit 4B"`. Some missing postal codes become empty strings, others become `null`, and a few get a best-guess value inferred from the city. When the developer re-runs the same prompt on the same data, the choices shift again.

**Which approach is most effective?**

- **A.** Rewrite the description with stronger, more emphatic language — mark the formatting rules as `IMPORTANT:` and `You MUST`, and expand the paragraph to cover each rule in more detail.
- **B.** Provide 2–3 concrete input/output example pairs that span the ambiguous cases, including one record with an apartment designator and one with a missing postal code.
- **C.** Ask Claude to interview the developer about the address normalization requirements before revising the script.
- **D.** Lower the temperature setting and re-run the existing prompt so the model's output becomes deterministic.

### ✅ Correct Answer: **B**

**Why B is right.** The diagnostic signal is **inconsistency**, both *within* a run (different records handled differently) and *across* runs. That is the exact fingerprint of an **ambiguous specification**, not an unwilling model. The prose left real degrees of freedom — it never said which apartment designator wins, or whether a missing postal code should be empty, null, or inferred — so Claude filled those gaps differently every time. Concrete pairs close the gaps by demonstration:

```
Input: "123 main st apt 4b, springfield, IL"  → "123 Main Street, Apt 4B, Springfield, IL 62701"
Input: "45 Oak Ave #12, Portland, oregon, —"  → "45 Oak Avenue, Apt 12, Portland, OR, null"
```

Two lines settle the designator format, abbreviation expansion, state-code rule, and the missing-postal-code policy — including the decision **not** to infer. Note the construction requirement embedded in the option: the examples must **span the ambiguous cases**. An example set with no apartment and no missing postal code teaches nothing about the two things actually going wrong.

**Why A is wrong.** The flagship anti-pattern. `IMPORTANT:` and `You MUST` add *emphasis*, not *information*. Claude never ignored the rules — the rules never specified which output form to pick. Emphasizing an under-specified instruction yields an emphatically under-specified instruction. Expanding the paragraph is worse still: more prose = more surface area for interpretation, in exactly the medium that already failed. Dangerous because it *looks* like diligence.

**Why C is wrong.** Wrong technique for this failure mode. The interview pattern is for **unfamiliar domains** where unknown unknowns exist. This developer knows address normalization — they already enumerated abbreviations, designators, state codes, and missing postal codes. There's no **knowledge** gap, only an **expression** gap. Interviewing costs a round trip to retrieve information they could have demonstrated directly.

**Why D is wrong.** Two failures. First, determinism would only freeze the *cross-run* variance — onto whichever arbitrary interpretation the model happened to land on, possibly wrong and now reproducibly wrong. Second and decisively, it does nothing about the **within-run** inconsistency: different records in the *same* execution got different treatments, which comes from an ambiguous spec being resolved case-by-case, not from sampling noise. Consistency is not correctness.

**Transferable pattern:** inconsistent output → ambiguous specification → demonstrate with examples. Never re-prompt harder.

---

## Question 2

A backend team is adding a distributed rate-limiting layer to their API gateway. None of the engineers on the team has built distributed rate limiting before — their experience is with single-instance in-memory throttling. They know they need it to work across their twelve gateway instances, and they know roughly what limits they want to enforce per API key.

The tech lead opens Claude Code and is about to write a prompt describing the requirement so Claude can implement it.

**Which approach is the most effective first step?**

- **A.** Write a test suite covering the expected rate-limit behavior, edge cases, and performance requirements, then have Claude implement against it and iterate on the failures.
- **B.** Provide 2–3 concrete examples showing request sequences and the expected allow/deny decision for each, then have Claude implement the logic.
- **C.** Ask Claude to interview the team — surfacing the design considerations they need to decide on — and answer those questions before any implementation begins.
- **D.** Have Claude implement a first version directly, then review the resulting code together and iterate on whatever looks wrong.

### ✅ Correct Answer: **C**

**Why C is right.** The decisive phrase is *"none of the engineers on the team has built distributed rate limiting before"* — the exam's explicit trigger for the interview pattern: **unfamiliar domain**. The gap is not expression but **unknown unknowns**. The team can state what they want ("limits per API key, across twelve instances") but not what it *implies*. A competent interview surfaces decisions nobody thought to make:

- Which algorithm — fixed window, sliding window, token bucket, GCRA — and what burst behavior each implies?
- Where does the shared counter live, and is the read-modify-write **atomic** across twelve instances?
- **Failure mode:** if the shared store is unreachable, fail **open** (availability, limits vanish) or fail **closed** (safety, outage takes down the API)?
- Clock skew between instances; hot-key contention on a popular API key; global limits vs. per-instance quotas.

None of these is in their prompt because they didn't know the questions existed. Note the option's wording — *"answer those questions before any implementation begins"* — the second half of the pattern: the answers **become the specification**.

**Why A is wrong.** The strongest distractor, and a genuinely excellent technique — *later*. TDD presumes you can write a **correct definition of correct**. This team can't yet. What assertion covers behavior during a Redis partition when fail-open vs. fail-closed hasn't been decided? What performance requirement, when they don't know hot-key contention is the thing that will bite? They'd write a confident, comprehensive, green suite encoding the **wrong invariants**. You cannot assert your way out of a requirement you don't know exists. Correct sequence: interview *then* TDD.

**Why B is wrong.** Same structural flaw, one level worse. Request-sequence → allow/deny examples don't merely express a spec, they **silently pick an algorithm**: whether a burst of 10 requests at t=0 is allowed encodes token-bucket-vs-fixed-window without anyone realizing a choice was made. The team would demonstrate behavior they never deliberately chose, and Claude would faithfully implement it. Examples fix a spec you *can* write but wrote ambiguously (Q1); they don't create a spec you don't possess.

**Why D is wrong.** Weakest option, for an important reason: **the quality of a review is bounded by the reviewer's knowledge of the domain.** A team that doesn't know fail-open/fail-closed is a decision won't notice its absence in review — the code will look clean and read plausibly. This converts unknown unknowns into a production incident rather than a design question: precisely the "costly rework" up-front techniques exist to prevent.

**Transferable pattern:** unfamiliar domain → interview *first*. Examples and tests are downstream instruments; they sharpen a spec you have, they don't create one you lack.

---

## Question 3

A developer is using Claude Code on a checkout service and reports four defects from QA:

1. Discount codes are applied **after** tax is calculated instead of before, producing wrong totals.
2. The order-total field is stored as a float, causing rounding drift on multi-item carts.
3. The confirmation email renders the customer's name unescaped, so `O'Brien` breaks the template.
4. When a discount reduces a cart to zero, the payment processor is still called with a `$0.00` charge and returns an error.

The developer wants Claude to fix all four.

**Which approach is most effective?**

- **A.** Send all four defects in a single detailed message so Claude can design one coherent fix across the whole set.
- **B.** Fix each defect in its own separate turn, in the order QA listed them, verifying after each.
- **C.** Send defects 1, 2, and 4 together in one message, and handle defect 3 separately.
- **D.** Send defects 1 and 2 together (both totals-related), then 3 and 4 together in a second message, splitting the work into two balanced batches.

### ✅ Correct Answer: **C**

**Why C is right.** Apply the test — *would knowing about defect B change how I'd fix defect A?*

Defects **1, 2, and 4 are one problem wearing three hats** — all in the order-total pipeline, each constraining the others:
- Fixing the **discount-before-tax ordering** (1) restructures the calculation sequence, which is exactly the sequence determining when a cart can hit zero (4).
- Defect **4 is a downstream consequence of discount application** (1). You can't design the discount path without knowing it must produce a zero-total guard.
- Switching float → decimal (2) changes what "zero" *means*: under floats a discounted cart might land at `0.0000001`; under `Decimal` it's exactly `0`. So the zero-check in (4) and the rounding fix in (2) are the **same decision** — a guard written before the numeric type is settled is written against the wrong representation.

Handed all three at once, Claude designs **one coherent totals pipeline** (Decimal-typed, discount-then-tax, zero-total short-circuit) — often simpler than the sum of three independent patches.

Defect **3 is genuinely independent**: HTML/template escaping in the email renderer, sharing no code path, state, or design constraint with the totals math. Knowing about `O'Brien` wouldn't change one line of the tax-ordering fix. So it gets its own turn — clean attention, clean attribution, clean commit.

**Why A is wrong.** Correct instinct, over-applied. Folding the escaping bug into the totals message buys nothing and costs three things: **attention dilution** (an unrelated concern competing with a subtle three-way numeric interaction), **attribution loss** (a total regression — was it Decimal or the template?), and an unreviewable commit doing two unrelated jobs. The "comprehensive!" distractor.

**Why B is wrong.** The mirror-image anti-pattern that *looks* like discipline. Fixing 1 alone means designing discount ordering with no knowledge that zero-carts are live — a structurally incomplete fix. Fix 2 then rewrites the same arithmetic to Decimal, partially undoing it. Fix 4 bolts on a guard depending on both prior changes. That's **fix thrash**: rework, wasted turns, and real risk of a local optimum no single fix would have chosen with full information.

**Why D is wrong.** The most seductive wrong answer because it *does* batch — but it draws boundaries by **symmetry and surface similarity** rather than dependency. It separates 4 from 1 even though the zero-total error is directly *caused* by discount application, so the discount fix again loses its key constraint. And it pairs 3 with 4, which share nothing — email escaping and payment-processor guards have no common code path. Two batches of two is tidy; it is not a coupling analysis.

**Transferable pattern:** batch by dependency, never by count.

---

## Question 4

A data engineer needs Claude Code to write a migration script that moves records from a legacy `users` table into a new normalized schema. The transformation rules are well understood by the team — they've done several of these migrations before, and the business logic is documented internally. Correctness matters a great deal: the migration will run once against production, and there are strict requirements that it complete within a 15-minute maintenance window on a 40-million-row table.

**Which approach is the most effective way to get a reliable migration script?**

- **A.** Have Claude write the migration script, then generate a comprehensive test suite against the finished implementation to verify its behavior before the production run.
- **B.** Write a test suite first covering expected transformations, edge cases such as null and malformed legacy values, and the throughput requirement — then have Claude implement against it and iterate by sharing the failing test output.
- **C.** Ask Claude to interview the team about the migration requirements, since a one-time production migration carries high risk and unrecoverable consequences.
- **D.** Have Claude write the script, then run it against a copy of production data and iterate by describing the discrepancies observed in the output tables.

### ✅ Correct Answer: **B**

**Why B is right.** Three signals converge on test-driven iteration:

1. **The domain is familiar** — *"the team has done several of these migrations before, and the business logic is documented."* No unknown unknowns to surface; they know what correct looks like and need to enforce it.
2. **Correctness is objective and verifiable** — a row goes in, a row comes out, you can assert on it. Ideal TDD territory, unlike architectural taste or prose tone.
3. **There is an explicit performance requirement** — 15 minutes, 40M rows. The strongest tell. The guide names the three things a pre-written suite must cover: **expected behavior, edge cases, performance requirements**. "Complete within the window" is prose; `assert elapsed < 900s for 40M rows` is a specification — and encoding it up front prevents a perfectly correct, perfectly row-by-row implementation being discovered to be six hours long at 2 a.m.

B also names the right **edge cases** (nulls, malformed legacy values) — the guide's own worked example — and specifies iterating by **sharing failing test output** rather than paraphrasing it. The stakes reinforce the choice: a one-shot production migration needs a **repeatable, regression-protected** definition of correct, so fixing null handling can't silently break throughput or the happy path.

**Why A is wrong.** The ordering trap, and the most important distractor on this topic. Tests written *against a finished implementation* encode **what the code does**, not **what the requirement was**. They'll be comprehensive, green, and green **on the bug** — a confident regression harness around a defect, shipped to production. TDD works because the suite is an **independent** statement of correctness the implementation must satisfy; letting the implementation shape the tests inverts that authority. Watch the phrasing: "generate tests to verify the implementation" ≠ "write tests, then implement."

**Why C is wrong.** Confuses **risk** with **unfamiliarity**. The interview pattern's trigger is an unfamiliar domain where you can't write a good spec because you don't know what considerations exist. This team has run several migrations with documented logic — no requirements to discover. High stakes don't create a knowledge gap; they raise the bar on **verification**, and the verification instrument is B.

**Why D is wrong.** Two independent failures. First, *"describing the discrepancies observed"* is the paraphrase anti-pattern — "some rows look off in the address column" discards the precision that makes iteration converge. Second and subtler: a production sample only exercises **whatever data happens to be in it**. The rare malformed legacy row — 40 occurrences in 40 million, the one that aborts the migration mid-window — may simply not be in the copy. A test suite **deliberately constructs** those cases rather than hoping to encounter them. D also asserts nothing about throughput and provides no regression protection across iterations.

**Transferable pattern:** familiar domain + objectively verifiable correctness + a non-functional constraint → write the suite first (behavior, edges, performance), then iterate on failure output.

---

## Question 5

A developer has spent eleven turns in a single Claude Code session trying to get a WebSocket reconnection handler working. The session history now contains three abandoned approaches (a retry-decorator design, an event-emitter design, and a state-machine design), several versions of the same file that have since been rewritten, and a long chain of "that's not quite right, try again" corrections.

The last four iterations have each fixed the stated problem while introducing a new one, and the developer notices the code is drifting further from anything they'd want to ship. Claude has recently started referencing a helper function that was removed six turns ago.

**Which is the most effective next step?**

- **A.** Continue iterating, but write a test suite now covering the reconnection behavior and edge cases, and share the failing output each turn.
- **B.** Send one detailed message enumerating every remaining problem at once so Claude can design a single coherent fix across the whole set.
- **C.** Start a fresh session with a structured summary: the confirmed requirements, the three approaches that failed and why, and the current state of the file.
- **D.** Ask Claude to re-read the relevant files from disk to refresh its understanding, then continue refining in the same session.

### ✅ Correct Answer: **C**

**Why C is right.** The scenario is a checklist of **context pollution** — the failure is no longer in the *code*, it's in the *conversation*:

- Three abandoned designs still occupy context, so Claude reasons over already-rejected approaches.
- Multiple superseded versions of the same file sit in history with no marker for which is real.
- Quality is **degrading rather than converging** — four consecutive fix-one-break-one iterations.
- **Smoking gun:** Claude referenced a helper removed six turns ago — direct evidence it's reasoning over stale context rather than reality. Once you see that, no additional instruction in that thread is reliable, because correct instructions now compete with a large volume of confidently-stated wrong ones.

This is the 1.7 judgment applied to a refinement loop: *"why starting a new session with a structured summary is more reliable than resuming with stale tool results"* and *"choosing between session resumption (when prior context is mostly valid) and starting fresh with injected summaries (when prior tool results are stale)."* Prior context here is demonstrably **not** mostly valid.

What makes C strong rather than wasteful: the summary is **structured**, carrying confirmed requirements, **the failed approaches and why they failed**, and the true current file state. That "why" section stops the fresh session cheerfully re-proposing the event-emitter design. You keep eleven turns of *knowledge* and discard eleven turns of *noise*.

**Why A is wrong.** The right instrument, mis-sequenced. Tests are excellent for reconnection logic and *should* be part of the plan — but writing them now appends failure output on top of eleven turns of contradictory history: a better signal into an already-jammed channel. Fix the channel first; carry the test suite in as part of the structured summary. Technique quality doesn't survive a poisoned context.

**Why B is wrong.** Misapplies Q3's lesson. Batching-by-coupling is right for multiple genuinely interacting defects **in a healthy context**. Here the defects are largely *artifacts of the churn* — symptoms of three half-merged designs — not a stable constraint set worth designing against. And the remedy worsens the disease: a large new message added to an already-saturated thread gives Claude more to reconcile against contradictory history.

**Why D is wrong.** The best of the wrong answers. It repairs exactly **one** symptom — stale file contents — and the guide does endorse informing a resumed session about specific file changes for targeted re-analysis. But that technique is scoped to *prior context being otherwise mostly valid*, which it isn't here. Re-reading files leaves the three rejected architectures, superseded instructions, and dead-end reasoning fully in context; Claude would have accurate files and still be pulled toward discarded designs. Partial remediation of one symptom while the cause persists.

**Transferable pattern:** when iteration quality **degrades** instead of converging, and the model cites things that no longer exist, stop refining. "Keep iterating" and "re-explain the requirement" are both wrong answers to context pollution.

---

# Clarifications Raised During the Session

**1. Test-first means *before*, not *after*.** The distractor "have Claude implement it, then write a comprehensive test suite" is common practice but inverts the authority: tests written after encode what the code does, not what the requirement was. They pass, and they pass on the bug.

**2. When to stop iterating and restart.** Symptom: many turns, each fix introducing a new problem, quality degrading. Cause: context pollution (failed approaches, stale file contents, superseded instructions). Fix: fresh session with a structured summary (1.7), **not** a sixth iteration.

**3. Where examples should live.** One-off transformation → examples in the prompt. Recurring project convention → examples embedded in CLAUDE.md or a path-scoped `.claude/rules/` file (3.1 / 3.3). Re-pasting the same examples every session treats a systemic problem as a per-conversation one.

**4. The interview pattern's second half.** The questions are half the value; your **answers become the specification** and must be consolidated and confirmed before implementation, not scattered across chat turns. Ordering is fixed: interview happens **before implementing**, not as a mid-implementation rescue.

**5. Interview vs. plan mode.** Plan mode reduces uncertainty about **the code** (enforced read-only exploration, human approval to exit). The interview pattern reduces uncertainty about **the requirements** (extracting what's in your head, not in the codebase). They compose: interview → plan mode → implement.

**6. High stakes ≠ unfamiliar domain.** Risk raises the bar on verification (TDD), not on requirements discovery (interview). Q4 tests exactly this confusion.
