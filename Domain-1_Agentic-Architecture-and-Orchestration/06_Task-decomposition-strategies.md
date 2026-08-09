# Topic 1.6 — Task Decomposition Strategies

**Domain 1: Agentic Architecture & Orchestration · Topic 6 of 7**

---

## Summary of Key Learnings

### What decomposition is
Converting one large/ambiguous task into a structured set of smaller subtasks with defined inputs, outputs, and ordering. It answers four questions, **in order**:
1. **Should I split at all?** (single agentic loop vs. multiple units) — the most-tested judgment.
2. **Along what axis do I split?** (stage / data / role)
3. **What's the dependency shape?** (independent → parallel; chained → sequential; mixed → DAG)
4. **How coarse/fine are the pieces?** (granularity)

The exam's favorite trap: answering #2–#4 when the right answer was #1 — *don't decompose*.

### The three decomposition axes
- **Sequential / pipeline (split by *stage*)** — a chain of transformations where each step needs the previous step's output (A→B→C). Can't parallelize the chain; value = clarity, checkpointing, smaller context per step. Orchestrate via structured handoff (topic 1.4).
- **Parallel / data (split by *independent unit*)** — same operation over many independent items, or independent sub-questions. Shape = fan-out → fan-in. Wins **latency** and **context isolation**. Cost goes *up* (~15×), not down. Orchestrate via coordinator spawning parallel subagents then synthesizing.
- **Functional / role (split by *expertise/responsibility*)** — subtasks need different skills, tools, instructions, or an independent perspective (planner/implementer/reviewer). Win = quality via separation of concerns + independent review (fresh-context reviewer isn't anchored). Each unit gets a specialized brief + scoped tools.

Real tasks **mix** these → a DAG.

### DAG = Directed Acyclic Graph
- **Graph**: nodes (subtasks) + edges (dependencies).
- **Directed**: "A → B" means B needs A's output; arrows show data/order flow (not symmetric).
- **Acyclic**: no cycles — following arrows can never loop back. A cycle = deadlock = impossible/invalid plan. Discovering a cycle usually means two "steps" are really one coupled step that shouldn't be split.
- Pure sequential chain and pure parallel fan-out are just **special shapes** of DAG; the DAG is the general form capturing "these run in parallel, those must wait."
- Execution falls out of the DAG mechanically: nodes with no unmet dependencies run in parallel *now*; a node runs only after all its input arrows arrive; graph "layers" = execution waves.
- **Review/critique loops** naturally *want* to be cycles — keep the graph acyclic by (a) bounding the retry in the harness loop (topic 1.1) with a hard cap, or (b) modeling a revision as a **new node** (U6′) rather than an arrow back.

### Granularity — the core tradeoff
- **Too coarse**: context bloat, weaker instruction-following, hard to verify, all-or-nothing failure, no meaningful checkpoints.
- **Too fine**: coordination/handoff overhead, token cost, latency, fragmented context, over-engineering.
- **Heuristic**: decompose at **natural verification / decision boundaries** (checkable artifact, a failure that should stop downstream work, or a genuinely different capability). Each unit should be **minimal-sufficient**: as small as possible while still independently meaningful and verifiable.
- **Transferable rule**: split one level finer *only when a new test starts firing* — a real verification boundary, genuine independent parallelism on the critical path, distinct tooling, or context overflow. If splitting adds coordination/synthesis cost without tripping a new test, you've over-decomposed.

### Should you decompose at all? (first + most-tested decision)
Default to the simplest thing that works.
- **Don't decompose** when: fits one context window; steps tightly coupled sharing lots of state; inherently sequential *and* cheap; no latency pressure and no independent parallel work.
- **Do decompose** when at least one fires: **context overflow** (needs isolation); **independent parallel work** (latency); **distinct expertise/tooling** or **independent perspective** (e.g., review); **reliability via checkpointing** (gate/verify high-stakes steps individually).

### Dependency shape → orchestration
| Shape | Pattern |
|---|---|
| Independent units | Parallel fan-out → synthesize |
| Strict chain | Sequential pipeline w/ handoff |
| Mixed | DAG: parallelize independent layers, sequence the rest |
| "Split" that shares tons of state | Don't split — single loop |

### Anti-patterns (high-yield)
- **Over-decomposition** — many tiny agents; coordination + cost dominate.
- **Under-decomposition** — one giant step that overflows / can't be verified.
- **False parallelism** — parallelizing steps that actually depend on each other.
- **Decomposing for cost savings** — multi-agent is ~15× tokens; it's a context/latency/quality tool, never thrift.
- **Ignoring synthesis cost** — fan-out is easy; coherent fan-in (reconcile, dedupe, prioritize) is the hard, underestimated part.
- **Fragmented context** — chopped so fine no unit has enough context to do its part.
- **No verification boundaries** — steps with no checkable artifact; failures surface late.

### Exam one-liner
> Decompose along the axis that matches the task's structure (stage / data / role), only when isolation, parallelism, distinct expertise, or checkpointing justifies it — at the coarsest granularity that still gives independent, verifiable units. Splitting is a quality/latency/context tool, never a cost-saver, and dependent steps must be sequenced, not parallelized.

---

## Worked example: messy task → DAG

**Task:** "Considering entering the meal-kit market. Briefing: look at 3 competitors, check whether the market is growing, pull internal sales for adjacent products, give a go/no-go recommendation with a risk section — make it defensible, no fluff."

**Units:** U1–U3 research each competitor (data/parallel); U4 market growth, U5 internal sales (independent sub-questions); U6 synthesize go/no-go (functional + depends on U1–U5); U7 critic review for defensibility (functional + depends on U6).

**DAG:**
```
        ┌── U1 (Competitor A) ──┐
        ├── U2 (Competitor B) ──┤
  START ┼── U3 (Competitor C) ──┼──▶ U6 (Synthesize ──▶ U7 (Critic ──▶ DONE
        ├── U4 (Market growth)──┤        go/no-go)         review)
        └── U5 (Internal sales)─┘
```

**Execution waves:**
- Wave 1 (parallel): U1–U5, each isolated context + scoped tools (sales-DB tool only for U5; web tools for researchers).
- Wave 2 (sequential fan-in): U6 after all of Wave 1; needs a **shared output contract** + real reconciliation (the expensive part).
- Wave 3 (sequential): U7 — independent critic, **fresh context** so "no fluff" gets a genuine second opinion.

**The defensibility loop** would make it cyclic → keep acyclic by bounding the redo in the harness (max N revisions) or modeling a revision as a new node.

All four decompose-triggers fire (parallel work, distinct tooling, context isolation, checkpoint) → textbook decompose case. Contrast: "research Competitor A only" fires none → single loop.

## Borderline granularity drill

Should **U1 "Research Competitor A"** be split into pricing / product / sentiment / funding?
- Test 1 verification boundary: only if stakeholder consumes sub-parts separately (weak here).
- Test 2 independence/parallel payoff: A is already one parallel branch; splitting it further adds 4× overhead for marginal latency → no.
- Test 3 distinct tooling: all use same web-search brief → no.
- Test 4 context budget: one profile fits → no.
→ **Keep U1 whole** (minimal-sufficient = one agent, one competitor).

**Flip the facts:** if pricing needs a live-site scraper AND sentiment needs analyzing 5,000 reviews (review-DB tool + heavy context), Test 3 (distinct tooling) and Test 4 (overflow) now fire → splitting A into pricing-agent + sentiment-agent **is** justified. Same task, different facts, opposite call — granularity is not a fixed number.

---

## Practice Questions

### Q1 — False parallelism on a dependency chain
**Scenario:** Agent for logistics: "extract line items from a manifest → validate each against the warehouse inventory DB → generate a discrepancy report." Junior engineer proposes 3 subagents in parallel (extract, validate, report) to minimize latency. Manifests always fit one context window.

Which critique is most accurate?
- A. Design is sound; parallelizing minimizes latency.
- B. The three steps have strict data dependencies (validate needs extracted items; report needs validation), so they can't run in parallel — false parallelism; and given small manifests a single agentic loop is likely better.
- C. Flawed only because it uses three subagents instead of five; finer decomposition improves reliability.
- D. Correct in shape but should add specialized reviewer agents at each stage for quality.

**Correct: B.**
- **B right:** strict sequential chain (A→B→C); validate needs extracted items, report needs validation results — no independent work to parallelize. Firing all three at once = false parallelism (downstream agents launch with no upstream data). Small manifests fit one window → no overflow, no distinct tooling → a single bounded loop is simplest. (The chain can't be parallelized; you'd only fan out if there were *many manifests* — data decomposition — which wasn't proposed.)
- **A wrong:** asserts parallelizability but the steps are dependent; parallelizing dependent steps breaks correctness, doesn't cut latency.
- **C wrong:** wrong axis — problem isn't "too few subagents"; more pieces still can't run concurrently and add handoff cost; over-decomposition of a one-loop task.
- **D wrong:** per-stage reviewers add cost/complexity with no justifying signal (low-stakes, small, mechanical); also silently accepts the false-parallel shape.

### Q2 — Under-designed synthesis (fan-in)
**Scenario:** Daily task: gather from web search + internal wiki + academic DB → single synthesized literature brief. Implemented as 3 parallel source subagents → synthesis merge. Source agents work well, but briefs are incoherent — contradictions side by side, duplication, "three reports stapled together."

Most likely root cause + best corrective focus?
- A. Parallel fan-out is the problem; gather sequentially so each agent sees prior findings.
- B. Subagents lack tools; give each access to all three sources.
- C. Decomposition shape is fine, but the synthesis (fan-in) is under-designed — merging independent branches coherently is the hard part; needs a shared output contract from branches + a real reconciliation step, not a staple-together merge.
- D. Shouldn't have decomposed; one agent should gather all three and write the brief.

**Correct: C.**
- **C right:** fan-out works ("source agents work well"); failure is at fan-in. Coherent synthesis = reconcile contradictions, dedupe, weigh sources — a first-class task. Fix: shared output contract (uniform, mergeable, source-tagged results) + genuine reconciliation, not concatenation. "Stapled together" is the literal symptom of merge-not-reconcile.
- **A wrong:** sources are independent → parallel is correct; serializing leaks partial context, bloats windows, loses latency, and doesn't fix a merge-time problem.
- **B wrong:** giving all agents all sources destroys context isolation, triples work, produces overlapping redundant reports → *more* to reconcile; violates scope-by-omission.
- **D wrong:** collapsing to one agent discards valid benefits (latency, isolation, source-specific tooling) to fix a problem that lives only in synthesis; fix the fan-in.

### Q3 — Irreversible step needs structural gating
**Scenario:** Refund agent: read request → look up order → check eligibility → **if eligible, issue refund (irreversible payment)** → send confirmation email. Refund must never run on an ineligible order or run twice. Choose the single best approach.
- A. Parallel subagents for eligibility, refund, email to minimize latency.
- B. Sequential pipeline relying on a carefully worded system prompt to only refund when eligible and never repeat.
- C. Sequential pipeline, but enforce the critical ordering *structurally* — harness makes the refund tool available only after eligibility passes — rather than prompt guidance alone.
- D. Merge eligibility-check and refund into a single step so there's no handoff.

**Correct: C.**
- **C right:** high-stakes irreversible step → match enforcement to stakes → structural gating: harness controls which tools exist per workflow state, so the refund tool isn't selectable until the eligibility gate passes → illegal action is *impossible*, not merely discouraged. Sequential shape + structural gate at the critical boundary. Orthogonal to `tool_choice` (which only selects among existing tools; can't hide the refund tool by state).
- **A wrong:** false parallelism (refund needs eligibility, email needs refund outcome) + optimizing an irreversible money step for latency while removing ordering guarantees is backwards.
- **B wrong:** prompt guidance lowers but never eliminates error probability; nonzero chance of a wrong/duplicate refund is unacceptable for irreversible money movement. The planted anti-pattern.
- **D wrong:** merging removes the decide/act verification boundary where the gate belongs and still gives no structural guarantee against ineligible/duplicate execution; coarsening to dodge a handoff ≠ an ordering guarantee.

### Q4 — Data decomposition with bounded concurrency
**Scenario:** Process 2,000 independent legal contracts: for each, extract key clauses then flag deviations from the standard template. One contract's extract-then-flag fits one context window. Want fastest reliable design.
- A. One sequential pipeline looping all 2,000 in a single agent context.
- B. Data (parallel) decomposition across contracts — process concurrently in batches of independent subagents — each subagent running extract→flag sequentially for its contract.
- C. Functional decomposition into "extractor" and "flagger" agents, each processing all 2,000, run in parallel.
- D. Spawn 2,000 subagents at once, one per contract.

**Correct: B.**
- **B right:** same operation over many independent items = data decomposition (fan-out across contracts, sequential extract→flag within each). Batches of concurrent subagents = latency win within concurrency limits; each subagent gets an isolated context for its one contract. "Map over independent items with bounded concurrency."
- **A wrong:** "one contract fits" ≠ "2,000 fit" — accumulating all 2,000 in one window overflows and degrades (under-decomposition), and a serial loop is the slowest design when independence offers a free parallel win.
- **C wrong:** flagger depends on extractor → false parallelism; and each agent handling all 2,000 re-creates the overflow; wrong axis (split by item, not by function).
- **D wrong:** right axis, wrong granularity — 2,000 at once ignores concurrency limits and explodes coordination/cost; the fix is batching (bounded concurrency), i.e., B. "More parallel" ≠ "better."

### Q5 — Functional decomposition for independent review
**Scenario:** Single "writer" agent produces marketing copy (quality is good). New requirement: every piece must be checked for legal/regulatory compliance before going out, and the compliance judgment must be *genuinely independent* of the writing. Engineer proposes adding a compliance-self-check rule to the writer's own system prompt.

Strongest critique + better alternative?
- A. Self-check is fine; prompt instructions are simplest and avoid ~15× cost of another agent.
- B. A self-check by the same agent isn't genuinely independent (writer anchored on its own reasoning) → functional decomposition: a separate compliance-reviewer agent with a fresh context and its own guideline-focused brief, as a sequential review node after the writer.
- C. Switch to parallel decomposition, run writer and compliance agent concurrently to save latency.
- D. Split the writer into finer sub-writers (headline/body/CTA), each self-checking compliance.

**Correct: B.**
- **B right:** "genuinely independent" is the key constraint; same-agent self-check is anchored on its own reasoning and rationalizes. Trigger for functional/role decomposition: separate compliance-reviewer with a **fresh context** (no exposure to writer's reasoning) + guideline-focused brief, as a **sequential downstream review node**. Real second opinion + separation of concerns + verification boundary.
- **A wrong:** fails the actual requirement — self-check can't provide independence (anchoring); cost-thrift doesn't override a stated independence requirement, which is exactly what justifies the extra agent.
- **C wrong:** compliance review is a check on finished copy → needs the writer's output → can't run concurrently (false parallelism); it's a downstream node. Latency wasn't even the concern.
- **D wrong:** wrong axis — splitting the writer doesn't add independent review; each sub-writer still self-checks → multiplies anchoring across three agents; over-decomposition of the wrong unit.

---

## Clarifications from follow-ups
- **What a DAG is** and *why acyclic matters*: a cycle is a deadlock / invalid plan; review loops that "want" to be cycles are kept acyclic by bounding the retry in the harness or modeling a revision as a new node.
- **Reading execution off a DAG**: no-unmet-dependency nodes run in parallel now; a node waits for all input arrows; graph layers = waves.
- **Granularity is not a fixed number**: split one level finer only when a *new* test fires (verification boundary / real parallelism on the critical path / distinct tooling / context overflow). Demonstrated with the "Research Competitor A" flip — same task, different facts, opposite call.
