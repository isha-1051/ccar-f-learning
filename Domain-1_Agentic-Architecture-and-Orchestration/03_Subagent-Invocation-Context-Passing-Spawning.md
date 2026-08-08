# Topic 1.3 — Subagent Invocation, Context Passing & Spawning

**Domain:** 1 — Agentic Architecture & Orchestration (27%)
**Topic number:** 3 of 7 (Domain 1)
**Status:** ✅ Complete

---

## 1. Core mental model

A subagent is **an opaque tool call that takes a string (the brief) and returns a string (the result).** Everything testable flows from three hard facts:

1. **Own context window.** A subagent inherits *nothing* — not the coordinator's message history, system prompt, loaded files, or sibling results. It starts fresh with only its brief (+ its own system prompt/tools).
2. **One-shot, text-only in each direction.** Down = the brief (a prompt string). Up = the result (a string, or a structured payload if a contract is imposed). No shared memory, no pointer passing, no live variable handoff.
3. **The coordinator is the only router.** Subagents can't call each other, can't see each other, can't ask a follow-up mid-run. Whatever they need must be in the brief **at spawn time**.

> One-line takeaway: *A subagent inherits nothing but its brief and returns nothing but a string, so context passing is a deliberate, minimal-sufficient, contract-shaped serialization in both directions — and spawning is only justified by real parallelism or context overflow.*

### "Stateful-internally-but-isolated-externally"
- **Internally stateful:** while running, a subagent has its own full conversation and remembers everything *it* did within its run (a complete 1.1-style agentic loop).
- **Externally isolated + ephemeral:** none of that internal state is visible to anyone else, and it is discarded when the run ends. Only the **final returned string** survives.
- Analogy: a contractor who works in a locked room for a week, hands you a one-page report, and burns all their notes.

### "Spawn time"
The instant the coordinator constructs the brief and launches the subagent — your **only write window** to that subagent's context. There is no live channel to a running subagent; if you realize mid-run it needed a fact, you can't inject it — you must let it finish (likely wrong) and re-spawn.

---

## 2. Invocation — two surfaces

### (a) API level — subagent-as-a-tool (you build it)
The coordinator is a normal agentic loop with a tool like `run_research_agent` whose input schema is `{ "brief": string }`. When Claude emits a `tool_use` for it, **your harness code** — not Claude — starts a *separate* `messages.create` conversation seeded only with the brief, runs that inner loop to completion, and returns the final string to the coordinator as a `tool_result`. Full control over inner system prompt, model, tools, loop bounds. You write the plumbing.

### (b) Claude Code / Agent SDK level — Task tool + subagent types (framework gives it)
Claude Code ships a **Task tool** and a registry of **subagent types** (e.g. `Explore` = read-only search, `Plan`, `general-purpose`, `code-reviewer`). Each type has a name, a `description`, a system prompt, an allowed-tools list, and optionally a model. You call Task, pass a `description` + `prompt`, and pick a type — the framework runs the isolated loop for you.
- **"Choosing a subagent type"** = selecting which preconfigured specialist (with its own system prompt + tool permissions + model) runs this delegation.

### Which to use?
| Raw API-level tool | Predefined subagent type |
|---|---|
| Building a custom product/agent on the API; need full control of inner loop/model/tools/bounds | Working inside Claude Code and a suitable specialist already exists |
| Bespoke behavior/schema; enforce org-specific output contract programmatically | Want least-privilege + curated system prompt "for free"; faster, less code |

Exam framing: not "one is better." Predefined types give preconfigured isolation, tool-scoping, and a system prompt out of the box; raw tool calls give full programmatic control at the cost of writing the harness. Reach for a type when one fits; reach for a raw tool when building custom orchestration.

---

## 3. Terminology: prompt vs brief vs description
- **`prompt`** — the full self-contained instruction set the subagent receives. This IS "the brief."
- **`brief`** — a *teaching word* (NOT a Claude keyword/API field) for a well-engineered `prompt`. On the exam, look for `prompt`, not `brief`.
- **`description`** — a short label identifying the invocation; in Claude Code it's also the **routing signal** for auto-selecting a subagent type ("the job title").

Example (Claude-Code-style Task call):
```
Task(
  subagent_type: "Explore",
  description:  "Find auth middleware",              # short label / routing hint
  prompt:       "You are searching a Node.js/Express repo. Find every place
                 request authentication is enforced (middleware like requireAuth,
                 authGuard, JWT verification). Return {file, line, function}.
                 Read-only; do not modify files. Report 'none found' if empty."  # the BRIEF
)
```
Returns a plain **string**; any structure exists only because the brief asked for it (→ output contract).

---

## 4. Giving tools to a subagent (spawn-time security boundary)
- **Raw API level:** whatever `tools` array you pass to the inner `messages.create` call is all the subagent can use. Read-only researcher = only search/read tools; omit write/edit/bash.
- **Predefined type:** the allowed-tools list is baked into the type (`Explore` *cannot* mutate files regardless of the brief). You choose permissions by choosing the type.

**Key principle:** tool scoping is a **spawn-time decision and a hard boundary**, not a prompt suggestion. Omitting a tool (programmatic enforcement) > forbidding it in prose (prompt-based guidance). A capability the subagent was never given cannot be invoked.

---

## 5. API-level tool-call mechanics (the round-trip)
It's the 1.1 loop with a tool whose implementation calls the model again:
1. **Coordinator decides:** returns `stop_reason: tool_use` with a `tool_use` block (id `toolu_01A`, name `run_research_agent`, input `{brief:...}`). Claude only *requests*; it runs nothing.
2. **Harness intercepts:** sees the tool name, starts a **fresh** `messages.create` (new `messages` array, own system prompt, own `tools`). Coordinator history is NOT passed in — only `input.brief` becomes the inner first user message. *(Isolation is a property of your harness choosing a fresh array.)*
3. **Inner loop runs to completion** (its own tool rounds) until it hits `end_turn`; harness captures the final assistant text.
4. **Harness returns up:** appends to the *coordinator's* conversation a `tool_result` with `tool_use_id: "toolu_01A"` (id-matching, same discipline as 1.1). On failure set `is_error: true` (→ 5.3).
5. **Coordinator continues** holding only the distilled string.

Four exam-critical notes:
- The **harness, not Claude**, spawns the subagent.
- **Isolation is your harness code's choice** (fresh array). Reusing the coordinator's array = no isolation, no savings.
- The coordinator sees only its own request + the returned string; the subagent's entire inner loop/tokens never enter its window — **that** is the context saving that justifies ~15× token cost.
- No special "subagent API" exists at this level; a subagent is just a tool.

---

## 6. Brief engineering
The brief is a **compression problem**: pass the *minimal sufficient* slice. Five parts:
1. **Role/objective** — one crisp outcome.
2. **Seeded context** — distilled relevant facts. *Rule:* include a fact only if the output would be wrong without it.
3. **Inputs** — the actual payload, inlined (subagent can't fetch from coordinator memory).
4. **Output contract** — exact return shape (§7).
5. **Boundaries** — scope limits, stop condition, and how to signal failure/uncertainty (it can't ask — escalate via the result → 5.2).

**Two opposing failure modes:**
- **Under-contextualization (starvation):** too little → hallucinated constraints, redone discovery, off-scope output. Symptom: plausible but subtly wrong.
- **Over-contextualization (context dumping):** paste entire history/all files → defeats isolation, balloons cost, buries the task, degrades focus. Symptom: expensive, slow, loses the thread.

Target = **minimal sufficient context** — a deliberate distillation. You are the compression function.

Worked example (✅ engineered brief):
```
Role: You extract liability terms from one vendor contract.
Context: Assessing vendor risk; only liability & indemnity matter. Ignore pricing/SLA.
Input: <<< [full text of contract #3 only] >>>
Output contract: Return JSON {liability_cap, indemnity, mutual: bool, notable_carveouts: []}.
Boundaries: If a field is absent, use null — do not infer. Do not read other contracts.
```

---

## 7. The output-contract pattern (most testable technique)
**Definition:** an identical, machine-parseable return schema pushed into *every* sibling's brief, so the coordinator merges uniform objects mechanically instead of parsing prose.

Why load-bearing:
- Siblings can't see each other → without a contract each invents its own format → synthesis becomes fragile NL parsing that drops/misattributes.
- With a contract, synthesis = a loop over uniform objects (reliable, cheap, testable).
- It's the carrier for metadata that **can't be reconstructed later**: `confidence`, `source`/provenance (→ 5.6), `status` (→ 5.2/5.3). If the subagent doesn't emit it, it's gone.

Exam trap: "coordinator struggles to reconcile inconsistent subagent outputs." Wrong fixes = smarter synthesis prompt, bigger synthesis model, more subagents, a reconciliation agent. **Right fix = impose a shared output contract at spawn time.** Fix the *input* wiring, not the *output* symptom. (Can be hardened by having subagents return via a JSON-schema tool call → Domain 4.3.)

---

## 8. Parallel vs sequential spawning
Deciding question: **is there a data dependency between the sub-tasks?**
- **No dependency → parallel (fan-out).** Wall-clock ≈ slowest single subagent (not the sum). Merge via shared contract.
- **B needs A's output → sequential.** Wait for A, embed A's result into B's brief, then spawn B. There is **no live wire** between running siblings — the coordinator threads A's data into B's spawn-time brief.

**What parallel actually looks like (API):** the coordinator emits **multiple `tool_use` blocks in one assistant turn**; the harness runs them concurrently and returns **all `tool_result` blocks in a single user turn**, each matched by `tool_use_id`. In Claude Code: issue several Task calls in one message ("send them in a single message so they run concurrently").

**Sequential example:** "Find the slowest query, then recommend an index."
- A (read-only Explore) → returns `{sql, table, latency}`.
- Coordinator embeds A's query into B's brief.
- B (DB-optimizer) → returns `{ddl, rationale}`.
A and B *cannot* run in parallel — B has nothing to optimize until A finishes.

**Over-orchestration guardrail (← 1.2):** even when splittable, if tasks fit one context and run naturally in sequence, **don't spawn** — one agent avoids ~15× cost + lossy serialize/deserialize at each boundary. Parallelism wins only for genuine independence (wall-clock savings) or context overflow.

---

## 9. Explicit invocation vs auto-routing
- **Auto-routing (description-as-router):** with a registry of subagent types, Claude picks the type by matching each type's **`description`** ("use this agent when…"). The description is the *selection signal*, not decoration. Convenient; less control.
- **Explicit / programmatic spawning (orchestrator decides):** *you* decide the decomposition and construct each brief yourself, naming which subagent to run with exactly which brief. Preferred when you need: deterministic decomposition, a guaranteed output contract, least-privilege tool-scoping, and reproducibility. This is the mode inside a coordinator pattern when you want control over decomposition rather than menu-matching — the coordinator *computes* the brief string (often stitching in prior results) and fires the subagent tool directly.

Exam framing: auto-routing is fine when good specialist types exist and the task is a clean match; **explicit programmatic spawning is preferred for most production coordinator patterns** (control, contracts, tool-scoping, reproducibility). Enforce > hope.

---

## 10. Connections
- **1.1:** spawning is a `tool_use`/`tool_result` round-trip; the harness runs the inner agent.
- **1.2:** 1.2 = the org chart; 1.3 = the wiring between boxes.
- **4.3:** returning via a JSON-schema tool call hardens the output contract.
- **5.2 / 5.3 / 5.6:** because a subagent can't ask mid-run, ambiguity/errors/provenance must travel *up through the result* — the contract must carry `status`/`confidence`/`source` fields.

---

## Anti-pattern cheat sheet
| Anti-pattern | Correct move |
|---|---|
| Assuming subagents share/inherit context | Re-seed every brief; isolation is total |
| Context dumping (whole history into each brief) | Minimal sufficient distillation |
| Under-contextualized (starved) brief | Include every fact the output depends on |
| Free-form returns → fragile synthesis | Shared output contract at spawn time |
| Spawning sequential+dependent+single-context work | Do it in one agent (avoid ~15× cost) |
| Parallelizing dependent tasks | Sequence; thread A's result into B's brief |
| Forbidding writes in prose | Omit write tools (programmatic enforcement) |
| Over-provisioning all tools "to be safe" | Least-privilege at spawn time |
| Categorical "always one agent / always multi-agent" | Decide by dependency + context-fit + parallelism |

---

# Practice Questions

### Q1 — API-level invocation mechanics
A coordinator on the Claude API exposes an isolated research subagent as a tool `research_subagent`. During a run it emits a `tool_use` block for it with a brief. What happens next, and why?

- **A.** Claude automatically opens a new context window, runs the subagent internally, and places the result back — the harness only needs to display the output.
- **B.** The subagent inherits the coordinator's full message history by default, so the brief only needs the new sub-question.
- **C.** The harness (developer code), not Claude, intercepts the `tool_use`, starts a *separate* `messages.create` seeded only with the brief, runs that inner loop to completion, and appends the returned string to the coordinator as a `tool_result` matched by `tool_use_id`.
- **D.** The coordinator pauses with `stop_reason: "pause_turn"` while Anthropic's servers execute the subagent, then resumes with output merged in.

**Correct: C.**
- **C right:** A subagent is a tool whose implementation calls the model again. Claude only *requests* delegation; the harness intercepts, spins up a fresh `messages.create` seeded only with the brief (creating isolation + context savings), runs the inner loop, and wires the string back with matching `tool_use_id`. Every clause is correct.
- **A wrong:** Attributes spawning to Claude. Claude is stateless/single-turn and runs no tools; the harness does the spawning and heavy lifting, not just display. "Claude does it magically" trap.
- **B wrong:** The context-leakage misconception. Subagents inherit **nothing**; isolation is the point. The brief must be self-contained.
- **D wrong:** Misuses `pause_turn` (a single-turn server-side pause/resume, not subagent delegation). Anthropic's servers don't auto-spawn your subagent or merge output — that's your harness via `tool_use`/`tool_result`.

### Q2 — Root cause of inconsistent synthesis
Five independent research subagents (one per jurisdiction) each return **free-form prose**. Synthesis frequently drops a jurisdiction or misattributes findings. Team's instinct: upgrade the synthesis model + rewrite the synthesis prompt. Most effective root-cause fix?

- **A.** Add a sixth "reconciliation" subagent to normalize the five prose outputs before synthesis.
- **B.** Push an identical structured **output contract** into all five briefs at spawn time (JSON with `jurisdiction`, `finding`, `source`, `confidence`) so the coordinator merges uniform objects mechanically.
- **C.** Run the five sequentially, feeding each output into the next brief.
- **D.** Enlarge the coordinator's window and pass all five full internal transcripts into synthesis.

**Correct: B.**
- **B right:** The symptom is downstream of a missing upstream contract. Identical schema at spawn time turns synthesis from "parse five essays" into "merge five uniform objects"; a required `jurisdiction` key structurally prevents misattribution, uniform shape prevents drops, and `source`/`confidence` carry non-reconstructable metadata. Fix the input wiring, not the output symptom.
- **A wrong:** Treats the symptom; adds another fragile, expensive parsing stage (over-orchestration) instead of preventing inconsistency at the source.
- **C wrong:** Serializing genuinely independent work throws away wall-clock benefit for no reason, and five sequential prose outputs are still five inconsistent formats — doesn't fix the cause.
- **D wrong:** Dumping raw transcripts defeats the purpose of isolation, explodes cost, and buries findings (more drops/misattribution). Subagents should return distilled, contracted conclusions.

### Q3 — Dependency-driven sequencing
Task: "Identify the single slowest SQL query, then generate an optimized index for *that* query." Engineer spawns A (find slowest query) and B (recommend index) **concurrently**. B returns vague/fabricated recommendations. Best explanation + correction?

- **A.** Under-provisioned B; give it a stronger model + web search so it can infer the likely slowest query itself.
- **B.** They share an isolated context pool by default; it's a race condition — add a delay before B reads A's output.
- **C.** B depends on A's output, but concurrent siblings have no channel to each other; run A first, then embed A's returned query into B's brief at spawn time (sequential, dependency-driven spawning).
- **D.** Merge into one brief because any two-step task should *always* be one agent.

**Correct: C.**
- **C right:** B's task is undefined until A produces the query; concurrent siblings are mutually invisible, so starved B fabricates. Correct wiring = sequential + re-seed A's result into B's brief (coordinator is sole router).
- **A wrong:** Fixes a missing-input problem with more guessing power → more convincing fabrication, not correctness.
- **B wrong:** Invents a nonexistent mechanism; there's no shared pool and no race — B never had A's output at all. A delay creates no channel. Concurrent-programming intuitions don't transfer.
- **D wrong:** The single-agent instinct is actually reasonable here, but the **absolute** "always" is false — genuine parallelism/context-overflow justify multi-agent. It also abandons rather than diagnoses the failure. Exam punishes always/never rules.

### Q4 — Tool-scoping vs prompt instruction
Recon subagents must **only read/search**, never modify files. Approach 1: strong read-only instruction in the brief. Approach 2: give only read/search tools and omit all write/edit/execute tools (e.g. a preconfigured read-only type). Which, and why?

- **A.** Approach 1 — prompt instructions are re-read every turn, so they bind more reliably than a one-time tool config.
- **B.** Approach 2 — tool scoping at spawn time is a hard programmatic boundary; a subagent can't invoke a capability it was never given, whereas a prompt is guidance the model can still violate.
- **C.** Neither alone; only both combined can prevent writes, because omitting write tools doesn't actually stop writing.
- **D.** Approach 1 — fewer tools will confuse the subagent; safer to grant all tools and constrain via prompt.

**Correct: B.**
- **B right:** Programmatic enforcement > prompt guidance. No write tool in the set → no `tool_use` block can mutate a file; it's not a rule the model chooses to follow. Prompts can be misread/violated. A read-only type gives this boundary for free.
- **A wrong:** Inverts reliability — re-reading a *soft* instruction doesn't make it binding; "set once at spawn time" is what makes tool scoping an immutable capability boundary. Exposure frequency ≠ enforceability.
- **C wrong:** False premise — omitting a tool *does* stop its use (the model can only act through given tools). Approach 2 alone suffices; the prompt is optional defense-in-depth.
- **D wrong:** Backwards least-privilege. Fewer tools focuses the agent and removes blast radius; "grant all, constrain via prompt" maximizes risk and relies on the weakest mechanism.

---

## Clarifications captured from follow-ups
- **"Brief" is not a Claude keyword** — it's a teaching term for the `prompt` string. Real fields: API = whatever you name it (`prompt`/`task`/`brief`); Claude Code Task = `description` + `prompt`.
- **Two delegation surfaces:** raw API subagent-as-a-tool (you build the harness, full control) vs Claude Code/SDK predefined subagent types (framework provides isolation + tool-scoping + system prompt).
- **"Choosing a subagent type"** = picking a preconfigured specialist, which simultaneously chooses its system prompt, tool permissions, and model.
- **Tool access** = raw: the `tools` array on the inner call; type: baked into the type definition. Enforcement is structural, at spawn time.
- **Parallel = multiple `tool_use` blocks in one turn** (harness runs concurrently, returns all `tool_result`s in one user turn). Sequential = coordinator threads A's result into B's brief.
- **Explicit programmatic spawn** (orchestrator computes the brief and fires) is preferred over description-based auto-routing when you need control over decomposition, guaranteed contracts, tool-scoping, or reproducibility.
