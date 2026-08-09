# Topic 1.4 — Multi-Step Workflows: Enforcement & Handoff

**Domain 1 · Agentic Architecture & Orchestration (27%) · Topic 4 of 7**

---

## Core question this topic tests

When a task has multiple ordered steps (e.g., *validate → look up → check policy → issue refund → confirm*), how do you guarantee Claude performs them **in the right order**, **skips none**, **invents none**, and **passes the right data** from one step to the next?

Central judgment call: **Programmatic (structural) enforcement vs. prompt-based (instructional) guidance.** This is the most-tested tradeoff in the domain.

---

## 1. Two ways to "enforce" a sequence

### A. Prompt-based guidance (soft enforcement)
Tell Claude the steps in the system prompt. It's a **strong suggestion**, i.e., a *probabilistic* instruction — not a guarantee. Under distribution shift (weird input, long context, adversarial user, ambiguous tool result) Claude can skip, reorder, act prematurely, or hallucinate a step.
- **Right tool for:** advisory, low-stakes, hard-to-structure ordering. Cheap, flexible, no engineering.

### B. Programmatic enforcement (hard enforcement)
The **harness/application code** — not Claude — controls what is *possible* at each step. A wrong order becomes *impossible*, not merely discouraged. Mechanisms:
- **Gate tool availability by state** — only expose a tool once its preconditions are met (*omit the tool > forbid it in prose*, from 1.3).
- **Validate tool inputs in code before executing** the side effect; return an `is_error` `tool_result` if a precondition fails, forcing course correction.
- **State machine in the harness** — app tracks "current step"; loop only advances when a step's success condition is met. *Claude proposes; the harness disposes.*
- **`tool_choice`** to force a specific action on a given turn.
- **Deterministic code for deterministic steps** — don't ask Claude to compute what a function can.

**Mental model:** Prompt-based = *Claude is trusted to follow the rules.* Programmatic = *the rules are made physically unbreakable by the code around Claude.*

**Rule of thumb:** the higher the stakes and the more **irreversible** the side effect (money moves, data deleted, email sent), the more you must use **programmatic enforcement**. Prompt guidance alone for an irreversible action is a classic wrong answer.

---

## 2. Handoff — passing state between steps

**Failure mode 1 — Transcript as data store.** Letting earlier tool results sit in context and hoping Claude re-reads them correctly is fragile (long-context burial, paraphrase, re-derivation/drift). Better: the **harness captures each step's concrete output into explicit application state** and injects exactly what the next step needs. Data lives in the program, not only in the model's "memory."

**Failure mode 2 — Loose/unstructured handoff.** Free-form prose between steps → fragile parsing. Prefer **structured outputs** (tool_use + schema / shared output contract) so each step emits machine-readable state consumed mechanically.

**Single-agent vs multi-agent handoff:**
- **Single-agent multi-step:** handoff is automatic — state accumulates in one context. Default, cheapest. Prefer when steps are sequential + dependent + fit one context (over-orchestration trap).
- **Multi-agent multi-step:** each subagent is isolated → handoff must be **explicit** message-passing *through the coordinator*. No shared live memory. Use a **shared output contract** so synthesis/threading is a mechanical merge.

---

## 3. Where to put the logic — decision boundary (Claude vs code)

| Step characteristic | Do it in… | Why |
|---|---|---|
| Deterministic / rule-based (sum, fixed tax, policy-table lookup, formatting) | **Code** | Reliable, cheap, testable, no hallucination |
| Requires judgment / NL understanding / ambiguity resolution | **Claude** | That's the model's job |
| Irreversible side effect | **Code gates it** (Claude *proposes*, code *executes* after validation) | Safety |
| Ordering that must never break | **Harness state machine / tool gating** | Structural guarantee |
| Advisory ordering, low stakes | **Prompt guidance** | Cheap, flexible |

**Bait pattern:** "Add a more forceful system prompt to always validate before refunding." For irreversible/high-stakes steps, more prompt text is *not* the fix — the fix is **structural**. Prompt hardening lowers error probability but never removes it.

---

## 4. Handling failure mid-workflow (ties to 1.1 / 5.3)

- Return step failures to Claude as a `tool_result` with **`is_error: true`** so it can adapt (retry / alternate path / stop) instead of proceeding on bad data.
- **Retry vs escalate vs abort**, per step: transient error → **bounded retry** (scoped to the failed step, aware of what already succeeded — don't blindly re-run side-effecting steps); precondition permanently unmet → don't retry, return outcome; unresolvable ambiguity → **escalate** to a human (5.2).
- **Bound the workflow** (max steps/iterations) so a stuck loop can't run forever (1.1).
- **Don't let one step's failure silently corrupt the handoff** — a failed step must halt progression to dependent steps (error propagation, 5.3).

---

## 5. Anti-patterns checklist (high-yield)

1. Prompt-only enforcement of an irreversible step → gate the tool / validate in code.
2. Exposing all tools all the time and trusting ordering → gate by workflow state.
3. Transcript as source of truth for critical values → capture state in the harness.
4. Free-form handoff → structured output / shared contract.
5. Asking Claude to do deterministic computation → move it to code.
6. No failure branch / unbounded loop → `is_error` + retry/escalate/abort + step cap.
7. Over-orchestrating a simple sequential task into many agents → single agent, one context.

---

## Deep-dive: Tool-gating vs. `tool_choice` (two different axes)

**Tool-gating = controlling *which tools exist* this turn** (the `tools` array).
- If a tool isn't in the array, Claude *cannot* call it — the action is **impossible**, not discouraged.
- Stateful: harness adds/removes tools as the workflow advances.
- **Strongest** structural guarantee for ordering and safety. → *shapes the menu.*

**`tool_choice` = controlling *how* Claude picks among tools that exist** (API parameter):

| `tool_choice` | Behavior | Use when |
|---|---|---|
| `{"type":"auto"}` | Claude decides whether to call a tool or answer (default) | Normal agentic turns |
| `{"type":"any"}` | Must call *some* tool (no plain-text answer), Claude picks which | Want an action, any tool |
| `{"type":"tool","name":"X"}` | Must call *exactly* tool X | Force a specific step now |
| `{"type":"none"}` | May not call any tool | Force a text-only turn (e.g., final summary) |

→ *shapes the selection.*

**Critical distinction:** Gating controls the set of possibilities; `tool_choice` controls the obligation to pick from that set. `tool_choice` does **not** make ordering safe — forcing/among-selecting can even *violate* intended order if an unsafe tool is present prematurely. Only **gating** makes an unsafe action impossible. They **compose**: force the first step via array=`[step1]` + `tool_choice:{tool:step1}`; guarantee later safety by simply *omitting* the dangerous tool until preconditions pass (then `auto` is fine).

- Prevent an action / enforce ordering / protect irreversible step → **gating**.
- Compel an action on a specific turn (force entry step, force structured extraction) → **`tool_choice`**. (Forcing structured output is the canonical single-call use — see 4.3.)

**Trap:** "Use `tool_choice` to ensure Claude never refunds before checking eligibility." Wrong lever — `tool_choice` compels/among-selects, it doesn't prevent premature actions across turns. Correct enforcement = **gating** + code validation.

---

## Deep-dive: Handoff in the multi-agent case

Why it's hard: a subagent inherits **nothing but its brief** and returns **nothing but a string**; siblings are totally isolated with no shared live memory; **spawn time is the only write window**. So handoff happens only *through the coordinator*: the **brief going down**, the **distilled result coming up**.

**Sequential handoff (A → B, data dependency):** coordinator spawns A, waits, **distills** A's result, **threads it into B's brief** at spawn. No live A→B channel. Quality depends on the coordinator extracting minimal-sufficient state (avoid starvation and context-dumping).

**Parallel handoff (independent siblings → coordinator):** siblings run concurrently (multiple `tool_use` blocks in one turn), cannot hand off to each other; all results flow **up** to the coordinator, which **synthesizes**. Risk = fragile synthesis if outputs are free-form.

**Key technique — shared output contract:** push an identical schema (e.g., `{finding, confidence, provenance, status}`) into **every** sibling brief at spawn. Then synthesis is a **mechanical merge** of uniform structures. The contract also **carries the metadata** the coordinator needs: **confidence** (weight/resolve conflicts), **provenance** (which source, 5.6), **status** (success/failure/partial).

**Error propagation across the handoff (5.3 preview):** a subagent's internal failure is invisible unless reported in its contract (`status:"error"`, low `confidence`). Blind merge → **silent corruption** of the final synthesis. So status/confidence aren't niceties — they're the mechanism that makes failure legible and stops silent propagation.

**Two one-liners to memorize:**
1. **Gating shapes the menu (makes wrong actions impossible); `tool_choice` shapes the selection (makes a chosen action mandatory this turn). They compose; only gating enforces safe ordering.**
2. **Multi-agent handoff is explicit message-passing through the coordinator — down via the brief, up via the returned string — reliable only when every sibling obeys a shared output contract carrying confidence/provenance/status.**

---

## Practice Questions

### Q1 — Irreversible refund ordering
Support agent must: verify order → confirm eligibility → issue refund (irreversible) → email confirmation. With a cleverly worded message Claude sometimes calls `issue_refund` before eligibility. Current design exposes all four tools every turn + a system-prompt line "Always confirm eligibility before issuing any refund." Most effective way to **guarantee** a refund can never precede eligibility?

- **A.** Rewrite the system prompt to be far more forceful (caps, repetition, worked example).
- **B.** Only include `issue_refund` in the `tools` array *after* the harness has run the eligibility check and recorded a pass; until then, don't expose it. ✅
- **C.** Set `tool_choice:{"type":"tool","name":"check_eligibility"}` on every turn.
- **D.** Keep all tools exposed; add a second Claude call that reviews the transcript after each turn and flags violations.

**Correct: B.** Irreversible + "must never" → **structural** enforcement. If the tool isn't in the array, the action is *impossible* regardless of prompt cleverness — a true guarantee.
- **A** wrong: prompt hardening lowers probability, never removes it; "can never" can't be met by prose.
- **C** wrong: confuses axes — `tool_choice` compels/among-selects, doesn't *prevent* a premature action across turns; `issue_refund` is still in the array, and forcing the check every turn breaks the later refund. Gating, not `tool_choice`, prevents.
- **D** wrong: post-hoc detection is too late — the refund already executed. Detection ≠ prevention.

### Q2 — Fragile multi-agent synthesis + silent failure
Coordinator spawns 3 parallel subagents (one per vendor). Synthesis is fragile (each returns different free-form prose → misparse/drop). Also, a subagent that fails partway returns a plausible-but-incomplete answer, merged as valid → silent corruption. Best **single** change to fix **both**?

- **A.** Have the coordinator re-read full raw outputs and use a more powerful model for synthesis.
- **B.** Switch subagents from parallel to sequential so each sees the previous one's format.
- **C.** Push a shared output contract (fixed schema incl. `status` and `confidence`) into every brief at spawn; coordinator merges on that structure. ✅
- **D.** Spawn a 4th "validator" subagent to reformat and completeness-check the three outputs.

**Correct: C.** Shared contract fixes both at the root: uniform structure → mechanical merge (kills fragile parsing); `status`/`confidence` make failure legible → coordinator drops/flags instead of merging blind. Spawn time is the only write window.
- **A** wrong: more inference on a structural problem — still probabilistic parsing; no reliable completeness signal.
- **B** wrong: wastes genuine parallelism; a later sibling only sees what the coordinator threads in; specifying a contract > copying a format by example; ignores silent failure.
- **D** wrong: extra isolated agent to retrofit structure you could require up front; validator still parses varied prose and guesses completeness without a ground-truth `status`.

### Q3 — Wrong invoice totals
Workflow: fetch line items → sum + apply fixed 8.5% tax → present total → charge card. Claude occasionally miscalculates. All steps done by Claude. Best fix for incorrect totals?

- **A.** Add a few-shot worked example and instruct Claude to double-check arithmetic.
- **B.** Perform the sum + tax in deterministic application code; hand Claude the computed total to present. ✅
- **C.** Force the step with `tool_choice` so Claude reliably calls a `calculate_total` tool.
- **D.** Add a second Claude call that recomputes and compares; retry on disagreement.

**Correct: B.** Deterministic arithmetic belongs in **code** — reliable, cheap, testable, zero drift. Claude presents (language task), doesn't compute.
- **A** wrong: leaves math in the probabilistic model; "usually correct" is a defect for money.
- **C** wrong: `tool_choice` guarantees the *call*, not correct *computation*; the win is the deterministic implementation (= B), not forcing selection. If the "tool" still routes math through Claude, nothing is fixed.
- **D** wrong: two probabilistic calculators can share errors; agreement ≠ correctness; doubles cost/latency to approximate one line of code.

### Q4 — Right-sizing the design
Expense agent: extract receipt → categorize → submit via `submit_expense`. Submission is **reversible** (pending queue, human approves later, recallable). Steps are **sequential + dependent**, all data fits one context. Want reliable ordering but simple. Most appropriate design?

- **A.** Coordinator + three subagents, one per step, threading results into next brief.
- **B.** Single agent in one context; step outputs accumulate naturally; light prompt guidance on step order. ✅
- **C.** Gate every tool behind harness state and force each step with `tool_choice` as hard checkpoints.
- **D.** Run the three steps as parallel tool calls in one turn, then reconcile.

**Correct: B.** Sequential + dependent + fits one context → single agent (handoff automatic). Reversible + simple → light prompt guidance is proportionate. Calibrate enforcement to stakes.
- **A** wrong: over-orchestration — adds cost/latency/manual handoff for isolation you don't need.
- **C** wrong: over-enforcement — hard checkpoints are for irreversible/high-stakes; forcing each step removes useful flexibility.
- **D** wrong: steps are dependent — can't parallelize a dependency chain; later steps get missing inputs.

### Q5 — Failure not surfaced, corrupt handoff
Travel agent: search flights → `book_flight` (side effect) → `book_hotel` (side effect) → email itinerary. `book_flight` fails (fare sold out); the harness forwards only the *successful search* data, so Claude — with no failure signal — books the hotel and emails an itinerary referencing an unbooked flight. Best combination to fix?

- **A.** Return the `book_flight` failure as a `tool_result` with `is_error:true`, and design the workflow so a failed booking halts progression (retry the fare / escalate / abort) rather than continuing to dependent steps. ✅
- **B.** Add a system-prompt instruction to always check whether the previous booking succeeded.
- **C.** Wrap the whole workflow in a bounded retry loop that re-runs all four steps from the beginning on any failure.
- **D.** Move the itinerary email to the very start so the customer is notified before any booking.

**Correct: A.** Two-part root cause: (1) surface the failure with `is_error:true` so it's in Claude's context; (2) halt progression to dependent steps (retry scoped / escalate / abort). Converts silent corruption into a controlled failure branch.
- **B** wrong: the failure data isn't even in context (harness forwarded only success); can't instruct Claude to notice what it wasn't shown; soft enforcement for a data-integrity guarantee.
- **C** wrong: steps 2–3 reserve/charge — blind full-restart risks double-booking/double-charging; retry must be scoped and aware of prior successes; also retries non-transient failures pointlessly.
- **D** wrong: reorders to hide the symptom; flight still unbooked, hotel booked against a nonexistent flight, and now the customer is emailed before anything is confirmed.

---

## Clarifications raised

- **Gating vs `tool_choice` are orthogonal.** Gating = which tools exist (the menu, makes actions impossible); `tool_choice` = obligation to pick among existing tools (the selection). Only gating enforces safe ordering; they compose.
- **`tool_choice` values:** `auto` (default, may answer), `any` (must call some tool), `tool` (must call named tool), `none` (no tool this turn).
- **Multi-agent handoff** is explicit message-passing through the coordinator (down via brief, up via string) — reliable only under a shared output contract carrying `confidence`/`provenance`/`status`; those fields are what stop silent error propagation (5.3).
