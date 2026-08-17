# Topic 4.6 — Design multi-instance and multi-pass review architectures

**Domain:** 4 — Prompt Engineering & Structured Output (20% of exam)
**Task Statement:** 4.6
**Topic number:** 6 of 6 within Domain 4 · 24 of 30 overall

---

## The one-sentence version

Review quality is a **context architecture** problem, not an effort problem: you fix missed bugs by changing **whose context the reviewer has** (independent instance) and **how much surface each pass must hold at once** (multi-pass) — never by asking the same instance to try harder.

---

## 1. Why self-review structurally fails

When Claude generates code, its context window ends up holding three things:

- the requirements,
- **its own reasoning about why this approach is correct**,
- the code that reasoning produced.

Instructing it, in that same session, to *"now review this code carefully"* does not produce a review. The model is re-reading **its own justification**. Every design decision already has an argument sitting in context explaining why it is right, so the model is anchored and unlikely to question it.

Exam guide wording: *"a model retains reasoning context from generation, making it less likely to question its own decisions in the same session."*

Two consequences you must be able to state:

**(a) It is a context problem, not a capability problem.** The same model finds the bug readily when it sees the code cold. It does not find it while holding the generation rationale.

**(b) Therefore more compute over the same context does not fix it.** The guide's explicit distractor: independent review instances beat self-review instructions **or extended thinking**. Extended thinking spends more reasoning budget on a *contaminated* context — deepening the anchor, not breaking it.

> **Exam pattern:** "Claude generates code, self-reviews, reports no issues, bugs reach production." The answer is *always* a **second independent instance with no generation context** — never a stronger self-review instruction, extended thinking, higher temperature, or repeated self-review.
>
> **Diagnostic tell:** if the scenario says the same code was later found buggy in a *fresh* conversation, that proves capability and criteria were never the bottleneck. Only the context changed.

---

## 2. What "independent" actually means

Independence = **a fresh context window containing the artifact but not the reasoning that produced it.**

It does **not** mean:

- a different model vendor (a second Claude instance is exactly what the guide describes),
- a different model tier,
- a new turn in the same conversation (same context — not independent),
- an instruction to "ignore your prior assumptions" (you cannot instruct context away).

| Include in the reviewer's context | Exclude |
|---|---|
| The generated artifact (code / diff) | The generator's chain of thought |
| The original requirements / spec | The generator's design justifications |
| Explicit review criteria (→ 4.1) | The prior conversation transcript |
| Output schema for findings (→ 4.3) | "I already verified X" claims |

**Rule of thumb: withhold the rationale, supply the spec.** The spec is required so the reviewer can catch code that is internally coherent but does the wrong thing. The rationale is the contaminant.

**Implementation shapes** (equivalent at exam altitude): a separate API call with a clean `messages` array; a subagent spawned with only the artifact in its brief (Domain 1.3 — a subagent receives only its brief, which *is* the isolation property); a separate CI job reviewing the diff.

---

## 3. Multi-pass review: splitting by scope

Second failure mode: one review prompt containing 30–50 changed files.

Two named symptoms in the exam guide:

1. **Attention dilution** — with a huge input the model spreads attention thin; middle-of-input findings get omitted and per-file depth collapses. (Same mechanism as "lost in the middle," formalized in 5.1.)
2. **Contradictory findings** — a single sprawling pass flags a pattern as a defect in one file while endorsing the identical pattern in another, because it never held both consistently.

Fix: split into passes with **distinct scopes**.

| Pass type | Input | Catches | Cannot catch |
|---|---|---|---|
| **Per-file local pass** (one per file, parallelizable) | One file + criteria | Logic bugs, null/error handling, off-by-one, local security issues | Anything requiring another file |
| **Integration pass** (one or a few) | Cross-file surface: interfaces, signatures, emitted/consumed data shapes | Contract mismatches, caller/callee drift, wrong data flow between modules, ordering assumptions | Deep local detail inside any one file |

**Both are required.** The symmetry is the exam's favourite trap:

- **Per-file only** → *structurally* blind to integration bugs. A pass on `writer.py` cannot know `reader.py` expects a different field name; the information is absent from every context in the system. No prompt, criterion, or example fixes it.
- **Whole-repo single pass only** → dilution and contradictions; local bugs get missed.

**Critical design detail:** the integration pass must receive **condensed structured outputs** from the per-file passes — not the full text of all files. Pasting every file back in recreates the diluted single pass you just paid to escape. (This is the 5.1 skill — upstream passes return structured facts, not verbose content — applied one topic early.)

**When single-pass is still right:** a 2-file, 30-line diff. Multi-pass responds to *scale and cross-file surface*, not ritual. N passes cost N× tokens and add latency; justify with size and stakes.

---

## 4. Confidence self-reporting for calibrated routing

Guide skill: *"Running verification passes where the model self-reports confidence alongside each finding to enable calibrated review routing."*

Pattern — each finding carries a structured `confidence` field (via tool use, → 4.3), and the **harness** routes:

```
high    → post directly to the developer / PR
medium  → send to a verification pass (independent instance: finding + code → "does this hold?")
low     → drop, or queue for human triage
```

Two properties make this legitimate: the model **emits everything** (it never silently suppresses), and the **harness decides** deterministically.

### Reconciling confidence across three topics — high-value exam distinction

| Topic | Verdict on confidence | Why |
|---|---|---|
| **4.1** | "Only report high-confidence findings" **fails** to improve precision | Vague instruction asking the model to do the criteria work. Precision comes from explicit categorical criteria. |
| **5.2** | Self-reported confidence is an **unreliable** escalation trigger | It does not track actual case complexity; escalation must trigger on concrete events. |
| **4.6** | Per-finding confidence **is** useful | It is a *label on an already-emitted finding*, consumed by deterministic harness routing — not a suppression filter, not a proxy for something the model cannot observe. |

**Hold this line: confidence is legitimate as routing metadata, illegitimate as a criteria substitute.** Emit everything, label it, let the harness decide.

**Verification pass** — an independent instance whose input is *the finding + the code*, whose only job is "is this real?" It is the false-positive control of a review architecture, and it must be independent for the same reason as §1: the pass that produced a finding is anchored to it and is the worst candidate to audit it.

---

## 5. Reference architecture

For a 30–50 file PR review in CI:

```
1. Generation        (instance A, or a human)
2. Per-file passes   (instances B1..Bn — fresh context each: one file + explicit criteria + finding schema)
3. Integration pass  (instance C — fed condensed interface/data-flow summaries from step 2)
4. Verification pass (instance D — per medium-confidence finding: "does this hold?")
5. Harness merge     (dedupe, route by confidence, post comments)
```

Every instance is Claude. Every instance has a **fresh, scoped context**. The **harness** — not the model — merges, dedupes, and routes. (Domain 1 recurring: deterministic work belongs in the harness.)

**Composition with the rest of the syllabus:**

- Nightly full-repo version → **batch API**; pre-merge version → **synchronous** (4.5).
- Findings carry `detected_pattern` so dismissals can be analyzed (4.4).
- Per-file criteria are explicit inclusion/exclusion lists, not "be thorough" (4.1).
- Few-shot examples calibrate severity and output format (4.2).
- Findings emitted via tool use with a JSON schema so routing is machine-readable (4.3).
- Condensed inter-pass handoffs (5.1).

---

## 6. Anti-pattern checklist

| Anti-pattern | Why it fails |
|---|---|
| "Now carefully review your own code" (same session) | Reasoning context intact; anchored |
| Enable extended thinking to catch its own bugs | More compute over a contaminated context |
| Re-ask the same instance at higher temperature | Randomness ≠ independence; also inflates false positives |
| Repeat the self-review N times in-session | N samples from the same anchored distribution |
| One prompt containing all 30–50 files | Attention dilution + contradictory findings |
| Per-file passes only | Structurally cannot see cross-file issues |
| Feeding full file bodies into the integration pass | Recreates the diluted single pass |
| Reviewer instance applies fixes, then re-reviews | Reviewer becomes generator; independence lost |
| "Only report findings you're confident about" | Vague confidence filter; use explicit criteria (4.1) |
| Escalating to a human on low PR-level confidence | Wrong granularity + unreliable trigger (5.2) |
| Batch API for a blocking pre-merge review | Automated ≠ non-blocking; no latency SLA (4.5) |
| Switching model vendors for "independence" | Independence is about context, not model identity |
| Multi-instance / multi-pass on a 2-file diff | Cost and latency with no dilution problem to solve |

---

# Practice Questions

## Question 1

A team uses Claude to generate implementation code for backend services. Their workflow is a single session: Claude reads the ticket, writes the code, and then — in the same conversation — is instructed *"Now review the code you just wrote carefully and report any bugs, security issues, or requirement mismatches before we commit."*

Claude consistently reports "no issues found." However, over the last quarter, 14 defects reached production, and when the team pasted the same code into a **new** Claude conversation afterward, 11 of the 14 were identified immediately.

Which change most effectively addresses the root cause?

**A.** Enable extended thinking on the self-review step so Claude reasons more deeply about its own output before reporting.

**B.** Send the generated code to a separate Claude instance whose context contains only the code and the original ticket requirements, with no prior generation conversation.

**C.** Strengthen the self-review prompt with explicit categorical criteria and few-shot examples of the defect classes that reached production, and instruct Claude to assume its code contains at least one bug.

**D.** Run the self-review step three times in the same session at a higher temperature and treat any issue raised in any run as a finding.

### Correct answer: **B**

**Why B is correct.** The scenario hands you the decisive evidence: the same model found 11 of 14 defects in a *fresh* conversation. That proves the failure is not capability, criteria, or effort — the only variable that changed was **context**. During generation Claude's context accumulates its own reasoning about why each decision is correct; self-review in that session is re-reading its own justification while anchored to it. A separate instance receives only the artifact and the requirements. Note B correctly *includes the ticket requirements* (needed to catch "coherent but does the wrong thing") while excluding the reasoning trace — withhold the rationale, supply the spec.

**Why A is wrong.** Extended thinking allocates more reasoning budget over the **same contaminated context**. The problem is not that Claude thought too little; everything it thinks is downstream of an argument in context saying the code is right. The guide explicitly ranks independent instances above self-review instructions *or extended thinking*.

**Why C is wrong.** The most seductive distractor: explicit criteria and few-shot examples are correct techniques — for 4.1 and 4.2. Here they target the wrong failure. Criteria improve *what a reviewer looks for*, not *who is looking*. The fresh-conversation result proves criteria were not the bottleneck; that instance found the bugs with no enhanced criteria. "Assume your code contains a bug" is an instruction attempting to undo context, which is impossible; at best it yields performative findings. (Criteria are still worth layering onto the independent reviewer in B — as an addition, not a substitute.)

**Why D is wrong.** Repetition and temperature buy variance, not independence. All three runs sit in the same session with the same rationale in context, sampling three times from an already-anchored distribution. Higher temperature also degrades precision and inflates false positives, which per 4.1 erodes trust in the accurate findings too.

---

## Question 2

An engineering org runs an automated review over large refactoring PRs — typically 35–50 changed files. The current implementation sends the entire diff to a single Claude call with a detailed review prompt.

Two complaints from developers:
1. Issues in files near the middle of the diff are frequently missed, while the first and last few files get detailed feedback.
2. The review sometimes flags a pattern as a defect in one file while explicitly endorsing the identical pattern in another file in the same report.

An engineer proposes: *"Split it — run one review call per changed file, in parallel, then concatenate all the findings into a single report."*

What is the most significant weakness of this proposal?

**A.** Running 35–50 parallel calls per PR will exceed rate limits and increase cost proportionally, making the approach economically unviable compared to tuning the single-pass prompt.

**B.** Per-file passes still contain the generation context for files Claude itself wrote earlier, so the reviews will remain anchored and miss the same defects.

**C.** Per-file passes cannot detect defects that only manifest across file boundaries — such as a caller and callee disagreeing on a data contract — because no single pass ever holds both sides.

**D.** Concatenating findings will produce duplicate reports of the same issue when a defect appears in multiple files, requiring a deduplication pass that the proposal omits.

### Correct answer: **C**

**Why C is correct.** The proposal genuinely fixes both stated complaints — per-file scoping removes attention dilution and gives each pass a consistent bounded judgment scope. That is what makes the item hard: the flaw is in what it *silently discards*. This is **structural** blindness, not a prompting weakness. If `writer.py` emits `{"user_id": ...}` and `reader.py` expects `{"userId": ...}`, neither pass can see the other side — the information required is absent from every context in the system, so no prompt or example can recover it. A refactoring PR is exactly where cross-file contracts change, making this defect class the most likely one present. Fix: keep the per-file passes *and* add an **integration pass** fed condensed interface/data-shape summaries (not the full 50 files, which would rebuild the diluted pass).

**Why A is wrong.** Cost and concurrency are ordinary tuning problems (concurrency caps, reviewing changed hunks only). More importantly, the remedy embedded in the option — "tune the single-pass prompt" — is itself wrong: attention dilution is a property of input size, not prompt quality. The exam ranks cost behind architectural correctness.

**Why B is wrong.** This misapplies §1. Each per-file call is a **fresh instance with a clean context** containing only that file — precisely the independence property you want. Generation context does not leak across separate API calls. B would only apply if the review were another turn inside the generating session.

**Why D is wrong.** Deduplication is a real gap and a complete architecture needs a harness-side merge — but it is a **cosmetic** report defect, not a **detection** failure. Duplicates are noise; undetectable integration bugs reach production. When asked for the *most significant* weakness, rank what escapes the system above what clutters the output.

---

## Question 3

A code-review system posts findings as inline PR comments. Developers report that roughly 40% of comments are not real issues, and several teams have started ignoring the bot entirely.

The team is evaluating four changes. Each is proposed as the *single* next step.

**A.** Append to the review prompt: *"Only report findings you are highly confident are genuine defects. When uncertain, stay silent."*

**B.** Have the review pass emit every finding with a structured `confidence` field via tool use, then route in the harness: high-confidence findings post directly, medium-confidence findings go to a separate verification instance that receives the finding plus the relevant code and judges whether it holds, and low-confidence findings are dropped.

**C.** Escalate to a human reviewer whenever the model's self-reported confidence for a PR falls below a threshold, so a person adjudicates the uncertain reviews.

**D.** Enable extended thinking on the review pass so the model reasons more carefully before committing to each finding, reducing hasty conclusions.

### Correct answer: **B**

**Why B is correct.** Two properties, both load-bearing. First, the model **emits everything** — it is not asked to suppress based on a feeling; it attaches a label. Second, the **harness routes deterministically**, and the medium band goes to a *verification pass*: an independent instance that sees the finding plus the code and judges whether it holds. That verifier is independent for the same reason as Q1 — the pass that produced a finding is anchored to it and is the worst candidate to audit it. B therefore combines both mechanisms of the topic: confidence as routing metadata plus an independent instance as the actual false-positive control.

**Why A is wrong.** This is exactly the instruction 4.1 identifies as ineffective. "Only report high-confidence findings" is a vague pressure valve, not a criterion — it asks the model to do the boundary-definition work the prompt failed to do, and it suppresses unevenly, losing real findings alongside false ones. Precision comes from explicit categorical criteria with concrete examples. Contrast with B: A asks the model to *silently withhold*; B asks it to *label and emit*.

**Why C is wrong.** Two independent flaws. It aggregates confidence to the **PR level**, destroying the per-finding granularity that makes routing useful (nine solid findings plus one shaky one get routed as a unit). And per 5.2, self-reported confidence is an unreliable trigger for **escalation to a human** because it does not track complexity or stakes. It also reduces no false positives — it just moves the 40% onto a human queue, the cost the bot was meant to remove. Distinction: confidence routing *inside an automated pipeline* is fine; confidence as the trigger for *handing work to a human* is not.

**Why D is wrong.** False positives here stem from **underspecified criteria** — the model does not know which patterns count as defects — and no amount of extra reasoning resolves a boundary never defined. More deliberation over an ambiguous rule yields more elaborately justified false positives, at higher cost and latency.

*Note: B is the best **single** next step; a complete fix layers in 4.1's explicit criteria and 4.4's `detected_pattern` field so dismissal patterns can be analyzed and criteria tightened over time. When the exam asks for one step, pick the architectural one.*

---

## Question 4 — *multiple response: select TWO*

A team is designing a **pre-merge** review architecture for a service repo. Claude generates much of the code; PRs average 30 files. The design so far: per-file review passes in parallel, then a single integration pass for cross-file data flow, then harness-side merge and posting to the PR. The merge is blocked until the review completes.

Which **two** of the following should be part of the design?

**A.** Provide the integration pass with condensed structured outputs from the per-file passes — exported interfaces, function signatures, and emitted/consumed data shapes — rather than the full text of all 30 changed files.

**B.** Submit the review calls through the Message Batches API to halve token cost, since the review is fully automated and no human is typing in real time.

**C.** Give each per-file reviewer instance the original ticket requirements and explicit review criteria, but not the conversation in which Claude generated the code.

**D.** Have the reviewer instance apply fixes for the findings it raises, then re-review the patched files to confirm each issue is resolved before posting results.

### Correct answers: **A and C**

**Why A is correct.** The integration pass exists to catch cross-boundary defects and needs the *contract surface* — exports, signatures, emitted/consumed shapes — not implementation bodies. Handing it all 30 files verbatim reconstructs the diluted single-pass input the split was meant to eliminate: you pay for the split, then discard its benefit at the last step. It also composes with 5.1 — upstream passes return structured facts, not verbose content, when the downstream consumer has a bounded context budget.

**Why C is correct.** The independence rule stated precisely, and both halves matter. **Include the requirements**, or the reviewer can only judge internal coherence and will approve code that flawlessly does the wrong thing. **Exclude the generation transcript**, because that is the anchoring contaminant. Same model, same vendor, fresh context: independence is a property of the context window, not of model identity.

**Why B is wrong.** A pre-merge check is **blocking by definition** — the scenario states the merge is held until review completes. The Batches API gives 50% savings with up to a 24-hour window and **no latency SLA**. "Automated, no human typing" is the trap 4.5 sets: *automated ≠ non-blocking* — a pipeline with no human is still blocking if a merge, deploy, or downstream job stalls behind it. Cost savings never justify converting a blocking workflow to unbounded latency. (A nightly full-repo audit of the same code *would* be a valid batch workload.)

**Why D is wrong.** It destroys the property the architecture was built for. Once the reviewer edits the code it holds the generation rationale for its own patch, so the re-review is a **self-review**, anchored exactly as in Q1, and will predictably confirm its own fixes. It also conflates two roles that must stay in separate contexts: findings should be posted or routed to an independent verification pass, and any fix should be generated in one context and reviewed in another — on *every* iteration, not just the first.

---

# Clarifications worth remembering

- **The diagnostic tell for self-review contamination:** the scenario mentions the same model succeeding in a *fresh* conversation. That single fact eliminates every "try harder" option (extended thinking, stronger instructions, repetition, temperature) because it proves capability and criteria were never the constraint.
- **Independence is about context, not identity.** A second *Claude* instance is the guide's answer. Switching vendors is neither required nor the mechanism.
- **Per-file blindness is structural, not remediable by prompting.** If the information is absent from every context in the system, no criteria, examples, or reasoning budget recover it. Only an architecture change does.
- **Confidence has three verdicts across the syllabus** (4.1 suppression = bad, 5.2 human escalation trigger = bad, 4.6 harness routing metadata = good). The dividing line is whether the model *withholds* based on it or merely *labels* while the harness decides.
- **"Most significant weakness" ranks detection failures above report-quality failures** (Q2: missed integration bugs > duplicate findings).
- **Automated ≠ non-blocking** — the 4.5 rule reappears inside review architecture questions whenever batch is offered as a cost saving on a pre-merge gate.
