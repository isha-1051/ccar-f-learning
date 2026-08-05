# Topic 1.2 — Coordinator–Subagent Orchestration

**Domain:** 1 — Agentic Architecture & Orchestration (27%)
**Topic:** 2 of 32

---

## Core mental model

Scale the single agentic loop (Topic 1.1) up: a **coordinator** (orchestrator / lead / primary) decomposes a task and delegates pieces to **subagents** (workers), each running its **own independent agentic loop in its own isolated context window**.

> **THE key fact:** Each subagent has its own isolated context. The coordinator does NOT see the subagent's internal reasoning, tool calls, or intermediate tokens — it receives **only the subagent's final returned result** (a string). This single fact is the source of every tradeoff the exam tests.

---

## Why use coordinator–subagent at all? (three real drivers)

1. **Context isolation / token economy** — a subagent can burn 50k tokens and return a 200-token summary; the coordinator's context only grows by those 200 tokens. Lets the system do more total work than one context window could hold.
2. **Parallelism** — independent subtasks run concurrently; total time ≈ slowest branch, not the sum. A single agent is inherently sequential.
3. **Separation of concerns / focus** — a narrow prompt + narrow toolset + single objective beats a generalist with 30 tools. Improves per-subtask reliability.

## Canonical shape: orchestrator–worker

Coordinator's job: **Decompose → Delegate (self-contained brief) → Collect → Synthesize.**
The coordinator is itself running an agentic loop; **spawning a subagent is just a tool call** inside it.

---

## Mechanics: how the coordinator "sees" a subagent as a tool call

- There is **no API primitive called "subagent."** It's a **tool** whose *implementation happens to be another agentic loop.*
- Turn-by-turn:
  1. Coordinator emits `stop_reason: tool_use` for e.g. `run_subagent(brief)`. Its loop **pauses** — it thinks it called an ordinary tool.
  2. The harness's implementation of that tool **spins up a fresh, separate agentic loop** (new empty context, own system prompt, own tools), seeded with `brief`.
  3. The nested loop runs to completion (its own `tool_use → tool_result → … → end_turn`), possibly tens of thousands of tokens — **all inside the single tool-call implementation, invisible to the coordinator.**
  4. The harness captures **only the subagent's final `end_turn` text** and packages it as the **`tool_result`**.
  5. Coordinator resumes; the `tool_result` looks identical to any tool returning a string. It has no idea a whole agent lived and died.
- Exam phrasing: *the coordinator experiences a subagent as a single, expensive, opaque tool call that returns a string.*

### Tie to Topic 1.1 ("state lives in the harness, not the model")
State lives in the harness → each agent is its **own** harness-managed message list → isolation between agents is **automatic and total** (they're just different arrays that never intersect). The subagent isn't "forgetting" the coordinator's context — it **never had it**. The coordinator didn't "lose" the subagent's reasoning — that lived in a different array the harness discarded.

---

## What crosses the boundary

| Crosses (explicit) | Does NOT cross (isolated) |
|---|---|
| The delegation brief (down) | Subagent's internal reasoning tokens |
| The subagent's **final result** (up) | Subagent's tool calls & tool results |
| Any context you deliberately inject into the brief | The coordinator's full history/plan |

**Downward channel (brief) must serialize everything the subagent will ever know:** objective · necessary context (only what it can't derive) · scope boundaries (so siblings don't overlap) · **output contract** · which tools it gets.
Design tension: **completeness vs. economy** — too little → drift/failure; too much → lose the context savings. Pass the *right minimal* context.

**Upward channel (result):** return **conclusions + evidence, not raw dumps** (a research subagent returns "X = $90/seat [source]", not 6k tokens of HTML).

**Cannot cross either way:**
- **Siblings can't see each other.** If B needs A's output, the **coordinator** runs A, extracts its result, and injects it into B's brief. No lateral channel.
- Coordinator can't peek inside a running/finished subagent.
- A subagent can't call back up mid-task; it can only **return** (e.g., return "I need clarification"), and the coordinator decides (ties to escalation, Topic 5.2).

---

## The delegation brief (make-or-break skill)

No shared context → a vague delegation is failure mode #1.
- ❌ "Research competitor pricing." (which competitors? tier? format? depth?)
- ✅ "Find the current monthly per-seat list price for the **Enterprise** tier of Competitors X, Y, Z from public pricing pages. Return **only** a markdown table: Competitor, Price, Source URL, Date. If not public, write 'not disclosed.' Do not research other tiers."

Objective + scope boundaries + explicit output contract must all travel **with** the delegation.

---

## Key tradeoffs & decision boundaries

**Single agent vs. multi-agent — when to escalate.** Multi-agent is NOT free: ~15× tokens vs. a single chat, coordination complexity, spawn latency.
- **Single agent when:** task is essentially sequential/dependent · fits in one context · subtasks share heavy state that's expensive to pass.
- **Escalate to orchestration when:** subtasks are genuinely parallel & independent · total work exceeds one context window · isolating noisy/token-heavy work protects the main reasoning thread.

**Parallel vs. sequential subagents.** Fan-out when independent; chain when B needs A (coordinator threads A's result into B's brief). All-sequential-and-dependent → probably don't need subagents at all.

**Depth of hierarchy.** Coordinator → subagent (shallow) is the robust default. Deep nesting multiplies cost, latency, lost-context, and error-propagation risk (Topic 5.3). Favor shallow unless depth is clearly justified.

---

## Over-orchestration — identify & prevent

Reach for multi-agent ONLY if you can check most of these:
1. **Genuine independence** — subtasks don't wait on each other.
2. **Parallelism payoff** — enough independent branches that concurrency meaningfully cuts wall-clock time.
3. **Context pressure** — total work exceeds one context, or isolating grunt work protects the main thread.
4. **Low coordination cost** — state to pass is small and cleanly serializable.

**Symptoms of over-orchestration (root-cause questions):**
- Cost blew up ~10–15× with no quality gain.
- Latency got **worse** (subtasks were actually sequential/dependent → spawn overhead, no concurrency).
- Coordinator spends most turns re-passing context (work was really one cohesive task).
- Subagents duplicate/contradict (task wasn't cleanly decomposable).

**Exam judgment:** start with the simplest architecture that works (single agent); escalate only when a specific driver demands it. Sequential + dependent + fits-in-one-context → **single agent**; a multi-agent option offered there is the trap.

---

## Anti-patterns (recognize these)

1. **Expecting shared memory** — assuming a subagent knows what the coordinator knows, or siblings see each other. They can't.
2. **Under-specified delegation** — vague briefs → drift, duplication, wrong shape.
3. **Returning raw dumps upward** — defeats context economy; return distilled results.
4. **Over-orchestration** — multi-agent on a simple sequential task.
5. **Coordinator doing the work itself** — collapses back to a single agent with overhead. Coordinator should delegate + synthesize, not execute.
6. **No result contract** — each subagent invents its own output format → fragile parsing/reconciliation at synthesis (see below).

### The output-contract problem (anti-pattern #6, expanded)
Briefs that don't specify a return shape yield mismatched results (prose vs. table vs. different units vs. raw dump). The coordinator must then do brittle NL parsing to reconcile — easy to get wrong (e.g., comparing per-seat to total, or SOC 2 Type I to Type II).
**Fix:** impose a **shared output contract** (fixed schema, explicit comparable fields, units) in *every* brief, e.g. `{competitor, price_usd_per_seat_month, is_public, source_url, confidence}`. Synthesis becomes a trivial, reliable merge. A `confidence`/`source` field also enables **principled conflict resolution** later. This is the orchestration-level version of structured output (Topic 4.3).

---

## Spotting "shared-memory" distractors (exam tells)
Apply this test to any option: *"Was this info explicitly placed into that agent's message list by the harness? If not, the agent cannot have it."* Wrong-answer tells:
- Access to "the conversation / history / user's previous messages."
- Lateral sibling awareness ("B builds on what A found" without explicit passing).
- Coordinator inspecting subagent internals/reasoning steps.
- Assuming work in one context populates another ("the subagent loaded it, so the coordinator has it").
- Invented primitives (`get_subagent_memory`, persistent replayable subagent context, mid-task call-back-up).

---

## Connections to rest of Domain 1 / 5
- **1.1** — subagent spawning is a tool call; state lives in the harness.
- **1.3** — mechanics of invocation, context passing, spawning (next topic).
- **1.4** — enforcing multi-step workflows & clean handoffs.
- **1.6** — task decomposition (splitting so work is independent/parallelizable).
- **5.3 / 5.5 / 5.6** — error propagation, human review/confidence, provenance & multi-source **conflict-resolution strategies** (voting, confidence-weighting, provenance ranking, escalation) live here, not 1.2.

---

# Practice Questions

### Q1 — Shared-memory isolation
Support system: coordinator delegates a billing question to a billing subagent, which reads 18k tokens of invoice history and returns *"double-charged $40 on invoice #4471; refund warranted."* Later the coordinator wants the specific line items. What can it do?

- **A.** Coordinator already has the invoice history (subagent loaded it same session) — quote directly.
- **B.** Coordinator can inspect the subagent's tool calls/intermediate results since it ran within the coordinator's loop.
- **C.** ✅ Coordinator only received the final string; the 18k tokens were in the subagent's isolated context and are gone. To get line items it must **delegate again** with a brief that explicitly requests them in the return.
- **D.** Call `get_subagent_memory` to replay the subagent's context; subagent contexts persist for the session.

**Correct: C.** Only the final returned string crosses the boundary; re-delegate with an explicit return contract.
- **A wrong** — "same session" ≠ shared context; the subagent's load populated *its* array, not the coordinator's (shared-memory trap).
- **B wrong** — coordinator sees an opaque tool call returning a string; internals were discarded ("ran within the loop" is a misconception — it ran inside a tool-call implementation, isolated).
- **D wrong** — invented primitive; subagent contexts are ephemeral to the tool call, not persistent/replayable.

### Q2 — Over-orchestration diagnosis
Agent must: read issue → locate file → apply fix → run tests → open PR. Each step strictly depends on the previous; whole task fits one context. Architect used 5 subagents (one per step). Result: token cost ~12×, latency **worse**, no quality gain. Best root-cause + fix?

- **A.** Subagents contend for the same tools; give each a dedicated tool copy.
- **B.** ✅ Over-orchestration: strict sequential chain, no parallelism, fits one context → multi-agent adds spawn overhead + 12× tokens for no benefit. Collapse to a single agent running the steps in one loop.
- **C.** Synthesis is the bottleneck; add a 6th synthesizer subagent.
- **D.** Context window is limiting; increase per-subagent context budgets.

**Correct: B.** Sequential + dependent + fits-one-context → single agent.
- **A wrong** — no tool contention; tools are just function defs each loop calls independently; doesn't explain the 12× tokens.
- **C wrong** — adding an agent worsens it; nothing to synthesize in a linear chain ("add more agents" reflex trap).
- **D wrong** — contradicts the given fact that it fits one context; bigger budgets increase cost.

### Q3 — Output contract prevents fragile synthesis
Coordinator fans out 4 subagents on vendor security posture with brief "Research Vendor N's security certifications and summarize." Results come back in 4 different shapes (prose, table, run-on sentence mixing pricing, raw copied text); coordinator mismatches SOC 2 Type I vs Type II. Most effective fix?

- **A.** Add a 5th subagent to normalize the four results before synthesis.
- **B.** Tell the coordinator to parse free-form output more carefully with NL reasoning.
- **C.** ✅ Rewrite each brief to impose a shared output contract (fixed schema, explicit comparable fields like `{vendor, soc2_type, iso_27001, last_audit_date, source_url}`) + scope boundaries (certifications only).
- **D.** Run the 4 subagents sequentially so the coordinator can adjust each brief.

**Correct: C.** Push structure down into briefs → uniform, machine-mergeable results; field-level `soc2_type` catches the Type I/II error.
- **A wrong** — treats symptom; the normalizer still parses the same ambiguous outputs and can make the same mistake (add-an-agent reflex).
- **B wrong** — "parse more carefully" IS the fragile-reconciliation anti-pattern that caused the failure.
- **D wrong** — sequential doesn't fix format inconsistency and throws away parallelism for independent lookups.

### Q4 — Dependent handoff routing
Stage 1 Subagent A: find 3 most-cited papers. Stage 2 Subagent B: extract each paper's stated limitations. B needs A's output. Most reliable design?

- **A.** Spawn A and B in parallel; B automatically has A's findings because they're siblings in the same session.
- **B.** ✅ Run A first; coordinator extracts A's returned list and spawns B with a brief that **explicitly includes those 3 papers** + output contract. B can't see A's context otherwise.
- **C.** Have A directly invoke B when done, passing papers laterally, bypassing the coordinator.
- **D.** Spawn B first with an empty brief; let it call back up to the coordinator mid-task to ask which papers A found.

**Correct: B.** Dependent → sequential; coordinator is the sole conduit (run A → extract → inject into B's brief).
- **A wrong** — siblings have zero awareness; "same session" ≠ shared context; parallel also starts B before A's output exists.
- **C wrong** — no lateral channel; a subagent can't invoke/hand data to a peer; that would be nesting a sub-subagent, not a peer handoff.
- **D wrong** — no mid-task call-back-up; a subagent only returns. (Nearest real pattern: B *returns* "I need the list," coordinator re-delegates — return-then-reinject, not a live callback.)

---

## One-line takeaway
A subagent = an opaque, expensive tool call that returns a string; each has an isolated harness-managed context, so **all coordination is explicit** (seed down / return up, coordinator as sole router) — and you match orchestration complexity to the task's actual structure, pushing shared output contracts down into every brief.
