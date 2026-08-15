# Topic 3.4 — Plan Mode vs Direct Execution

**Domain:** 3 — Claude Code Configuration & Workflows (20%)
**Topic number:** 3.4 (4th of 6 in Domain 3 · Topic 16 of 30 overall)

---

## 1. What plan mode actually is

Plan mode is **not** a prompting technique or "Claude thinking harder first." It is a **permission state enforced by the harness**.

- **Read-only tools stay enabled** — Read, Grep, Glob, read-only investigation. Claude can explore freely.
- **Mutating tools are blocked at the harness level** — Edit, Write, NotebookEdit, state-changing Bash. The harness *refuses* them; Claude cannot write even if it decides it should.
- **The exit is a gated handoff** — Claude leaves plan mode by calling `ExitPlanMode` with a proposed plan. The human approves, edits, or rejects. **Approval is what restores permissions.**

The essential effect: plan mode inserts a **mandatory human decision point between "Claude has decided what to do" and "Claude does it."**

This is the same principle as *structural gating vs prompt-based guidance* (topics 1.4, 1.5). Saying "please show me a plan before editing" is a prompt — it works most of the time and **fails silently** under pressure (long context, ambiguity, urgent-sounding requests). Plan mode is the **enforced** version, like a `PreToolUse` hook denying `Edit`.

> When the cost of skipping the checkpoint is high, don't *ask* for the checkpoint — *enforce* it.

## 2. What direct execution is

Default Claude Code operation: Claude reads, decides, and edits within its permission settings; the human reviews the result (diff, tests, commit). The review point is **after** the work exists.

Direct execution is **not** the "unsafe" mode — it is correct most of the time. The difference is *where the review point sits*:

| Mode | Review point | Cost profile |
|---|---|---|
| Plan mode | **Before** work exists | Cheap to redirect; costs a round-trip *every* time, including on trivial tasks |
| Direct execution | **After** work exists | Zero up-front latency; a wrong approach means paying for a wrong implementation and unwinding it |

The decision reduces to one question:

> **How expensive is being wrong, compared to the cost of the checkpoint?**

## 3. The decision boundary (examinable core)

### Use plan mode when the *approach* is uncertain and the *cost of a wrong approach* is high

| Signal | Why it favours plan mode |
|---|---|
| Change spans many files / a whole subsystem | Wrong approach = large, messy revert |
| Several valid designs exist and the human has a preference | Only the human can settle a design tradeoff |
| Codebase unfamiliar to Claude (or the developer) | The plan surfaces Claude's *understanding* so a misread is caught before it becomes code |
| Change is hard to reverse | Migrations, schema changes, deletes, public API changes |
| You need to review the *reasoning*, not the *diff* | A diff shows what changed; a plan shows *why*, and what was rejected |
| Requirements are ambiguous | Forces Claude to commit to an interpretation you can correct cheaply |

### Use direct execution when

| Signal | Why |
|---|---|
| Task is small and well-specified | "Rename this variable," "add a null check here" |
| Approach is obvious — only mechanics remain | No design decision for the human to make |
| Work is trivially reversible | Uncommitted git changes, scratch files |
| Verification is automatic and fast | A strong test suite catches a wrong implementation faster than plan review would |
| Iterating conversationally | A gate per turn is pure friction |

### The cleanest single formulation

> **Plan mode when a decision the human owns is still open. Direct execution when the only thing left is execution.**

Note this is driven by *open decisions and reversibility*, **not by task size**.

## 4. Anti-patterns

**A — Plan mode as ceremony.** Requiring plan mode for every task, including one-liners. The "more process = more safety" trap. It doubles round-trips on work that was never risky, and — critically — **trains reflexive rubber-stamping**, which destroys the checkpoint's value on the one plan that mattered.
> **A checkpoint you approve reflexively is not a checkpoint.**

**B — Direct execution on irreversible or architectural work.** Reviewing only the final diff on a migration, broad refactor, or security-sensitive change. The cost is already sunk, and diff review is *worse* than plan review for approach errors: a 40-file diff makes a wrong architecture look like many individually reasonable edits.
> **You catch typos in diffs; you catch wrong strategies in plans.**

**C — Treating a prompt as a substitute for the mode.** "CLAUDE.md tells Claude to always propose a plan first." That's guidance; it degrades. If the rationale is *safety* (irreversible action), you need the enforced version. If the rationale is *collaboration quality*, guidance may suffice.

## 5. What a good plan contains

A plan is not a task list. A useful plan exposes:

- **The understanding** — what Claude believes the current code does and where things live. *This is where a misread is caught before it becomes 400 lines of wrong code.*
- **The approach and rejected alternatives** — the design decision made explicit and therefore correctable.
- **The blast radius** — which files/systems get touched.
- **The verification story** — how you'll know it worked.

A plan of "1. Edit auth.py 2. Edit routes.py 3. Run tests" gives the human nothing to approve — they can't judge whether the approach is right, so they approve it, and the checkpoint becomes ceremony.

> **Plan mode's value is proportional to how much of Claude's *reasoning* the plan makes reviewable.**

## 6. The investigation / read-only use case

Plan mode is also **the safe mode for pure investigation** — "explain how the caching layer works," "audit this module for race conditions," "find every call to the payments API." No mutation is intended, so plan mode guarantees a **zero blast radius** while leaving exploration unrestricted. Here it isn't about approving an approach at all — it's about **enforcing read-only**.

Distinguish it from a hook:

| Mechanism | Character |
|---|---|
| **Plan mode** | Session-scoped, human-toggled, **interactive** |
| **`PreToolUse` deny hook / permissions** | Persistent, programmatic, works **unattended** (including CI) |

## 7. The CI trap (links to 3.6)

**Plan mode requires a human.** Its exit condition is human approval. In headless/CI/non-interactive contexts there is nobody to approve, so plan mode is either meaningless or a deadlock.

- "Our pipeline runs Claude Code on every PR — how do we prevent unintended changes?" → **permission configuration and hooks**, never plan mode.
- Plan mode is also not a mechanism for **team-wide policy**: it's a per-invocation choice by whoever drives the session. Policy that must hold for everyone, always, belongs in settings/permissions/hooks.

## 8. Worked scenarios

| Scenario | Choice | Reason |
|---|---|---|
| Migrate user sessions from in-memory to Redis (15 files, unfamiliar code, real design choices: serialization, TTL, Redis-down fallback) | **Plan mode** | Genuine design decisions belong to the human; Claude's understanding of current session flow must be validated before it's encoded; half-finished migration is expensive to unwind |
| "Line 212 says 'recieve' — fix the typo" | **Direct execution** | No approach to approve; a plan saying "fix the typo" gets rubber-stamped — pure friction that erodes the habit of reading plans |
| Add rate limiting to public API endpoints (per-user vs per-IP, sliding window vs token bucket, placement, breach behaviour) | **Plan mode** | Not size, not reversibility — an **unresolved design decision the human owns** |

---

# Practice Questions

## Question 1

A platform team runs Claude Code in a GitHub Actions workflow triggered on every pull request. The job asks Claude to review the PR diff and post findings as a comment. During a postmortem, the team discovers that on two occasions the automated job modified source files — changes never intended and never reviewed.

The team lead proposes: "Let's make the CI job run Claude Code in plan mode so it can't touch anything without approval."

What is the **most accurate** assessment?

- **A.** It will work, because plan mode blocks Edit and Write at the harness level regardless of whether the session is interactive.
- **B.** It will not reliably work, because plan mode's exit condition is human approval, which does not exist in a non-interactive CI run; permission configuration or a `PreToolUse` hook is the appropriate mechanism.
- **C.** It will work, but only if the team also adds a CLAUDE.md instruction telling Claude to never edit files during PR review.
- **D.** It is unnecessary, because Claude Code automatically operates in read-only mode when it detects a CI environment, so the modifications must have come from another workflow step.

**Correct answer: B** ✅ *(answered correctly)*

**Why B is right.** Plan mode's full lifecycle is *enter → explore read-only → propose via `ExitPlanMode` → **human approves** → permissions restore*. In headless CI there is no human at the gate, so the job either doesn't meaningfully enter the mode or stalls at approval instead of doing its work. What's needed is a **persistent, programmatic, unattended** restriction: permission configuration disallowing mutating tools, or a `PreToolUse` hook denying `Edit`/`Write`. Same layered lesson as 2.3 and 1.5 — match the mechanism to the layer: plan mode guards a *human's* session; hooks and permissions guard *the harness*.

**Why A is wrong.** The first half is true (and makes it the strongest distractor): plan mode really does block mutating tools structurally, not by prompting. But the conclusion doesn't follow — blocking is only half the mechanism; the other half is the human handoff. "Regardless of whether the session is interactive" is exactly the false claim. A safety mechanism whose release valve is a human is not an automation safety mechanism.

**Why C is wrong.** Layers a *prompt* under a mechanism and calls it reliable. CLAUDE.md guidance degrades under long context and ambiguity, and fails silently. When the goal is preventing unreviewed writes to a shared repo, guidance is never the answer — enforcement is.

**Why D is wrong.** There is no automatic CI-detection read-only mode. CI jobs typically run with broad permissions and no interactive prompts *precisely because* nobody is there to approve them — which is what allowed the writes. It also deflects from evidence the team already has.

---

## Question 2

A senior engineer sets a policy: every Claude Code session must start in plan mode, and no plan may be approved without a written review comment. After three weeks velocity has dropped and, in a retro, several engineers admit they approve plans without reading them. A junior engineer proposes dropping plan mode entirely and reviewing diffs instead.

Which response best addresses the **root cause**?

- **A.** Keep mandatory plan mode but add a required checklist to the approval step so engineers must confirm they read the plan.
- **B.** Adopt the junior engineer's proposal — diff review happens after the work exists and is therefore always more informative.
- **C.** Scope plan mode to work where a design decision is still open or the change is hard to reverse, and use direct execution for small, well-specified, reversible tasks.
- **D.** Keep mandatory plan mode but require plans to contain only a numbered list of files to be edited, so reviews are faster and engineers are more likely to read them.

**Correct answer: C** ✅ *(answered correctly)*

**Why C is right.** The root cause is indiscriminate application: when most plans under review are trivial and obviously correct, skimming and approving is the *rational* response — and that habit then carries into the one plan that needed scrutiny. A gate passed reflexively provides no safety, only latency. Restoring signal means making the gate selective, so a plan appearing in the queue *means something*. General principle: **match the strength of the control to the cost of being wrong**, not to a uniform policy.

**Why A is wrong.** Treats a symptom as a compliance problem and stacks process on process. A checklist on a low-value review becomes another click-through — two rubber stamps instead of one, plus more latency. Never asks why the plans weren't worth reading.

**Why B is wrong.** Over-corrects and asserts something false. Diffs are excellent for *implementation* errors (typos, missed edge cases, wrong conditionals) but weak for *approach* errors, since a wrong architecture presents as many individually plausible edits — and the implementation cost is already sunk. Dropping plan mode entirely re-exposes the team to irreversible/architectural work reviewed only after the fact.

**Why D is wrong.** Makes the checkpoint faster by stripping the only content that gave it value. A bare file list reveals nothing about Claude's understanding, chosen approach, or rejected alternatives — so misreads and wrong strategies are uncatchable. Shorter plans get approved even more reflexively; this deepens the ceremony problem.

---

## Question 3

A developer working in an unfamiliar service asks: *"Something's causing duplicate charges in the billing flow. Find out what's going on."* They want certainty that the investigation cannot modify code or run side-effecting commands, while giving Claude full freedom to search.

Which approach is **most appropriate**?

- **A.** Run in direct execution and rely on git to revert anything Claude changes unintentionally.
- **B.** Run in plan mode, which leaves read-only tools available while structurally preventing edits, and requires approval before any change is made.
- **C.** Add an instruction at the top of the prompt: "Do not modify any files during this investigation — investigation only."
- **D.** Ask Claude to first list the files it intends to read, approve that list, then run in direct execution.

**Correct answer: B** ✅ *(answered correctly)*

**Why B is right.** The investigation use case — both properties of plan mode work together. Read-only tools stay fully available, so exploration is unrestricted (critical in an unfamiliar service where scope can't be pre-declared), while mutating tools are blocked by the harness, making "cannot modify code" a **structural fact rather than a hope**. If a fix emerges, the exit path is already correct: Claude proposes, the human approves. Here plan mode isn't approving an *approach* — it's **enforcing read-only**.

**Why A is wrong.** Confuses *recoverable* with *prevented*. Git revert is cleanup after an unwanted change; the developer asked for certainty it cannot happen. It also only covers tracked file edits — nothing about a side-effecting Bash command outside the working tree.

**Why C is wrong.** Prompt-as-enforcement. Guidance degrades under exactly these conditions (long investigative session, wandering context, a moment where fixing looks obviously helpful) and fails silently the once it matters. When the requirement is *certainty*, guidance is disqualified.

**Why D is wrong.** Gates the wrong thing. Reading is harmless, so pre-approving a read list buys no safety while forcing Claude to fix a search scope before it knows anything — and root-cause investigation is inherently iterative. Then it hands the actual work to direct execution, leaving mutations entirely ungated: maximum friction on the safe part, zero protection on the risky part.

---

## Question 4

Four tasks are queued in a mature repository with a fast, comprehensive test suite:

1. Rename a misspelled internal helper function and update its call sites.
2. Replace the custom-built job queue with a third-party library across the service.
3. Add a missing null check that a stack trace has already pinpointed to one line.
4. Change a user-facing API response shape that three downstream teams consume.

Which selection is **best**? (Select ONE.)

- **A.** Plan mode for 2 and 4; direct execution for 1 and 3.
- **B.** Plan mode for 2, 3, and 4; direct execution for 1.
- **C.** Plan mode for all four, since a mature repository justifies consistent review.
- **D.** Plan mode for 4 only; the test suite covers the risk in 1, 2, and 3.

**Correct answer: A** ✅ *(answered correctly)*

**Why A is right.** Run each task through *is a decision the human owns still open, and how expensive is being wrong?*
- **Task 2:** wide-open design space (library choice, retry/dead-letter semantics, atomic vs phased migration); touches the whole service; half-finished swap is painful. The reason is not "it's big" — Claude's understanding of the current queue's semantics must be validated **before** it's encoded into fifteen files.
- **Task 4:** the code change may be tiny, but consequences leave the repo — versioning, deprecation window, consumer coordination. You can revert a commit; you can't un-break someone's integration.
- **Task 1:** mechanical and internal; compiler + tests catch a missed call site immediately. Nothing to approve.
- **Task 3:** the stack trace already did the diagnosis. No approach left, only execution.

**Why B is wrong.** Gates task 3, whose plan would read "add a null check at line 212" — approved without thought. Ceremony in miniature, and corrosive: it trains reflexive approval that carries over to task 2's plan, which genuinely needed reading.

**Why C is wrong.** "Consistent review" is the failing policy from Q2. Uniform gating destroys the gate's signal value; a mature repo with strong tests argues for *less* up-front gating on mechanical work, since verification is already fast and automatic.

**Why D is wrong.** Over-trusts the test suite. Tests verify an implementation satisfies specified behaviour — they don't tell you the *approach* was right. Green tests after a wholesale queue replacement say the tests still pass, not that the new library's delivery guarantees match the old queue's or that the migration strategy was sound. **Tests catch implementation errors; plans catch strategy errors** — different failure classes.

---

## Question 5

A developer enters plan mode for a substantial refactor. Claude explores and presents:

> 1. Modify `src/auth/session.py`
> 2. Modify `src/auth/middleware.py`
> 3. Modify `src/api/routes.py`
> 4. Update `tests/test_auth.py`
> 5. Run the test suite

The developer sees nothing objectionable and approves. Claude implements it. Two hours later the developer discovers Claude had misunderstood how session invalidation currently works, and the entire implementation rests on that misunderstanding.

What is the **primary** lesson?

- **A.** Plan mode is unsuitable for refactors; direct execution with diff review would have exposed the misunderstanding.
- **B.** The plan enumerated actions but not reasoning, so it gave the developer nothing reviewable — a plan's value comes from exposing Claude's understanding of the current system, its chosen approach, and the alternatives it rejected.
- **C.** The developer should have required Claude to run the test suite before presenting the plan, which would have surfaced the misunderstanding.
- **D.** The failure was unavoidable; plan mode only guarantees changes are approved before being made, and cannot protect against a model misunderstanding existing code.

**Correct answer: B** ❌ *(answered D — see the correction below)*

**Why B is right.** The checkpoint fired exactly as designed — Claude stopped, proposed, waited. What failed was the **content** submitted for review. A list of five files to modify contains no claim the developer can evaluate as true or false. Catching this required the plan to surface Claude's model of the current system: *"session invalidation is currently handled by X, which means Y; I'll therefore restructure it as Z, rather than W because…"* That sentence is **falsifiable** — the developer, who knows how invalidation actually works, rejects it in ten seconds, before a line is written. Hence "saw nothing objectionable": there was nothing there *to* object to.

**Why D is wrong (the miss worth remembering).** It states a real limitation (plan mode is a gate, not a comprehension guarantee) and then draws a **fatalistic conclusion that doesn't follow**. Plan mode exists precisely to make Claude's reasoning — including its reading of existing code — visible while correction is still cheap. Misunderstanding the current system isn't a residual risk plan mode can't reach; it is **the single failure class plan mode is best at catching**, provided the plan articulates the understanding. Calling it unavoidable mistakes a badly-formed plan for a ceiling of the mechanism.
> **Exam pattern to watch for:** an option that names a genuine limitation accurately, then uses it to excuse a *preventable* failure.

**Why A is wrong.** The opposite over-correction. A diff is *worse* here — a wrong mental model manifests as forty individually reasonable edits, and diff reviewers naturally ask "is this code correct?" rather than "is the premise behind all of it correct?" Plus two hours of implementation are already sunk. Approach errors are exactly where plan review beats diff review.

**Why C is wrong.** The suite reflects the code as it exists today; running it pre-plan passes and reveals nothing. Tests confirm current behaviour is intact — they cannot show that Claude's *interpretation* of that behaviour is wrong. The misunderstanding lived in Claude's head, not in the repo's health.

---

# Key Takeaways

1. **Plan mode is an enforced permission state, not a prompt.** Read-only tools on, mutating tools blocked by the harness, exit gated by human approval.
2. **Its exit condition is a human** — so it is an *interactive* control only. CI, headless runs, and team-wide policy need permissions and hooks instead.
3. **Choose by cost of a wrong approach, not by task size.** Plan mode when a decision the human owns is still open; direct execution when only execution remains.
4. **Two anti-patterns:** ceremony (plan mode everywhere → reflexive approval → the checkpoint stops working) and unguarded irreversible/architectural work (diff review can't catch strategy errors).
5. **A plan's value = how much reasoning it makes reviewable.** Actions without understanding, approach, and rejected alternatives = a rubber stamp.
6. **Tests catch implementation errors; plans catch strategy errors.** A strong suite does not substitute for approach review.
7. Plan mode doubles as **enforced read-only** for pure investigation — zero blast radius with unrestricted exploration.
