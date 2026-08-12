# Topic 2.3 — Distribute tools appropriately across agents and configure tool choice

**Domain:** 2 — Tool Design & MCP Integration (18% of exam)
**Task Statement:** 2.3
**Topic:** 10 of 30

---

## 1. Framing

2.1 covered **how a single tool is described**. 2.2 covered **what a tool returns when it fails**. 2.3 covers the two levers that sit *above* the individual tool:

1. **Distribution** — which agent gets which tools. A **design-time** decision about the tool *set*.
2. **`tool_choice`** — whether and which tool the model must call on a given request. A **runtime, per-request** decision.

They are taught together because they fix two failure modes candidates routinely confuse:

- *"The model picked the wrong tool"* → usually a **distribution** or **description** problem.
- *"The model didn't call a tool at all / called them out of order"* → a **`tool_choice`** or **enforcement** problem.

---

## 2. Tool distribution across agents

### 2.1 Tool count degrades selection reliability

Every tool definition is text in context, and every tool is a candidate at every decision point. Tool selection is a classification problem whose difficulty scales with the number of plausible candidates. The exam guide states it explicitly: **18 tools instead of 4–5 degrades tool selection reliability by increasing decision complexity.**

Three things go wrong as the set grows:

- **Candidate confusion** — more tools means more overlapping descriptions and more near-neighbor misrouting (the 2.1 failure, amplified).
- **Attention dilution** — the tool block competes with the system prompt, history, and tool results; a long block makes each description less salient.
- **Out-of-specialization misuse** — the guide's named case: *agents with tools outside their specialization tend to misuse them*, e.g. **a synthesis agent attempting web searches**.

> **An unused tool is not free.** A tool the agent *could* call is one it *will eventually* call — usually in exactly the situation where you'd rather it returned control to the coordinator.

**4–5 is a heuristic for "few enough to stay clearly distinguishable," not a hard cap.** Six genuinely orthogonal tools beat four overlapping ones.

### 2.2 Scoped tool access

Each agent gets only the tools its role requires, plus a small number of deliberately chosen **cross-role tools for specific high-frequency needs**.

Scenario 3 (multi-agent research):

| Agent | Tools | Rationale |
|---|---|---|
| Coordinator | `Task` | Decomposes, delegates, routes. `allowedTools` **must include `"Task"`** (1.3) or it cannot spawn. |
| Search agent | `web_search`, `fetch_search_result` | Retrieval only. |
| Document analysis | `load_document`, `extract_data_points` | Document-scoped; no web access. |
| Synthesis agent | `verify_fact` | The one scoped cross-role exception. |
| Report agent | `Write`, `format_citations` | Output only. |

### 2.3 The scoped cross-role tool (the key judgment)

**Default = no cross-role tool.** It must be *earned*. Situation: the synthesis agent hits one claim that looks wrong.

| Design | Outcome | Verdict |
|---|---|---|
| **A. No tool** | Report doubt → coordinator → re-delegate to search → re-invoke synthesis with full finding set | Correct but 3 extra invocations for one date |
| **B. Give `web_search`** | It searches this gap, then the next, expands scope, duplicates the search agent with a prompt not built for retrieval | ❌ The exam's named anti-pattern |
| **C. Give `verify_fact`** | Checks the one claim, gets `{supported, evidence}`, continues | ✅ Correct |

**Why C is not "a broader tool set":**

```
web_search(query)                → arbitrary results/topics, unbounded scope,
                                   chainable into open-ended investigation
verify_fact(claim, source_url)   → {supported, evidence}; closed question,
                                   bounded output, cannot discover new topics
```

`verify_fact` is a **new, narrow capability**, not a slice of the search agent's capability. The tool set grew by one; the **capability surface** grew by almost nothing. That's the trade: buy back a round-trip for a capability increment small enough to be safe.

**Two-part test for granting a cross-role tool (both must hold):**

1. **Frequency** — is the need common enough that coordinator round-trips are a real cost? (Guide: *"limited cross-role tools for specific **high-frequency** needs."*)
2. **Closability** — can it be expressed as a **closed** question with a bounded answer? Open-ended exploration → coordinator, always, regardless of frequency.

| Need | Frequent? | Closable? | Answer |
|---|---|---|---|
| Spot-check one suspicious claim | Yes | Yes | ✅ Scoped tool |
| An entire subtopic is missing | No | No | ❌ Coordinator re-delegates |
| Full text of an excerpted source | Yes | Yes (`fetch_cited_source`) | ✅ Scoped tool |
| "Findings feel thin on region X" | Occasional | No — a *scope* judgment | ❌ Coordinator |

**Reframe that makes it click:** don't think "give a specialist an extra tool." Think **"define the specialist's role slightly wider, then give it exactly that."** The synthesis agent's role isn't "combine text" — it's *"combine findings into an accurate narrative,"* and accuracy is part of the job. `verify_fact` is *within-role*; it crosses a data boundary, not a responsibility boundary. **Guardrail:** if you can't state the need as part of the agent's actual responsibility, it's scope creep and belongs to the coordinator.

**Why the coordinator owns re-delegation:** hub-and-spoke (1.2). The synthesis agent can't see budget, sibling status, or the overall plan, so "should we research more?" is not its decision to make.

### 2.4 Constrained alternatives instead of generic tools

Guide skill: replace generic tools with constrained alternatives — **`fetch_url` → `load_document` that validates document URLs.**

```
fetch_url(url)              → raw HTTP; can hit APIs, search endpoints, internal hosts
load_document(document_url) → validates target is a document; rejects API endpoints
                              and internal hosts; returns parsed content
```

Four reasons:

1. **Enforcement layer** — with `fetch_url` you must *instruct* ("only for documents") = probabilistic. With `load_document` the restriction is in the implementation; intent is irrelevant. Same logic as hooks-over-prompts, applied at the tool boundary.
2. **Selection clarity (2.1)** — `fetch_url` is a *mechanism* name; `load_document` is *task altitude*. The latter matches intent directly instead of requiring intent → mechanism inference.
3. **Blast radius** — `fetch_url` inside a network is an SSRF primitive; a validating tool is not.
4. **Error quality (2.2)** — `fetch_url` returns whatever the server said (a 200 HTML error page looks like success). `load_document` can return a structured, actionable failure:

```json
{
  "isError": true,
  "errorCategory": "validation",
  "retryable": false,
  "message": "URL resolves to an API endpoint, not a document. Use web_search
              to find a document URL, or ask the coordinator for the source."
}
```

**When a generic tool is still correct:** `Bash` is maximally generic and legitimate for Scenario 4's exploration agent, because "run arbitrary commands" genuinely *is* the task. Constrain only when (a) the role uses a **narrow slice**, and (b) the unused slice is **harmful or misleading**. If the agent needs the full range, a constrained tool just gets worked around — usually via `Bash` + `curl`, which is worse (no validation *and* no visibility).

### 2.5 Granularity — split vs. merge

- **Split** when one tool does genuinely different things with different inputs/outputs (2.1: `analyze_document` → `extract_data_points` / `summarize_content` / `verify_claim_against_source`).
- **Merge** when several tools do one job with different parameters.
- **Don't over-split** into micro-steps that always run in sequence — that just moves the decision-complexity tax.
- **Then distribute** the results across agents; splitting three ways and handing all three to one already-loaded agent nets nothing.

**The merge test:** if the variants share a schema and answer one question → merge. If they'd need a different schema per value → keep separate.

```
✅ Merge:  process_resolution(order_id, resolution_type, amount, reason)
✅ Merge:  get_customer(identifier, identifier_type: "id"|"email"|"phone")
❌ Mega:   support_action(action: "get_customer"|"process_refund"|"escalate")
           — unrelated operations, different required fields per branch,
             schema validates nothing
```

**Target:** each agent sees a small set of clearly distinguishable tools at task altitude — not "the system has few tools overall."

### 2.6 Worked drift example (Scenario 1)

Launch set — four tools, cleanly partitioned, no two are plausible substitutes:

| Tool | Decision it answers |
|---|---|
| `get_customer` | Who am I talking to / are they verified? |
| `lookup_order` | What did they buy, when, what state is it in? |
| `process_refund` | Take the money action. |
| `escalate_to_human` | I can't or shouldn't resolve this. |

Two quarters later it's 18 tools, each addition individually reasonable (`get_customer_by_email`, `lookup_order_history`, `process_partial_refund`, `process_store_credit`, `issue_replacement`, `escalate_to_billing_team`, …). Now *"my order came damaged and I want my money back"* faces four plausible resolution tools and three plausible lookup tools. **The problem isn't the number 18 — it's that 18 tools contain multiple near-neighbor clusters, and each cluster is a coin flip.**

**Fixes in priority order:** (1) collapse near-neighbors into parameterized task-altitude tools; (2) move genuinely different concerns to another agent/route; (3) only then reconsider the remainder.

**Biggest real-world source of bloat:**

> **A tool you add so the model can't forget a rule is usually a hook you should have written.**

In Scenario 1 the hard work is deliberately *not* in the tool set: ambiguity handling lives in the **system prompt** (not an `ambiguity_classifier` tool); ordering enforcement lives in a **hook** (not a `verify_then_refund` combo tool); escalation policy lives in a **hook** (not a `check_refund_policy` tool the model might forget to call).

### 2.7 Distribution in Claude Code / MCP terms (Scenario 4)

Every connected MCP server contributes its **full tool list at connection time** (2.4) — no lazy loading, no per-task filtering. All tools are live on every request.

```
GitHub MCP ~30 + Jira ~20 + Postgres ~10 + Playwright ~25 + built-ins ~8
= ~93 tools on every request
```

Levers, in order of preference:

1. **Scope servers to where they're needed** — project-level `.mcp.json` for shared team tooling, user-level `~/.claude.json` for personal/experimental. A server relevant to one repo belongs in that repo's `.mcp.json`, not in user config where it loads everywhere. *Scoping is distribution, applied to MCP.*
2. **Restrict per-subagent tool sets** via `AgentDefinition.tools`.
3. **Prefer MCP *resources* over exploratory tools** — expose catalogs (issue summaries, doc hierarchies, DB schemas) as resources. Reduces tool count **and** exploratory round trips at once. Highest leverage, most commonly missed.
4. **Prefer existing community servers** over custom ones for standard integrations (Jira, GitHub); reserve custom servers for team-specific workflows.

**Counter-pressure (2.4):** MCP tool descriptions must be detailed enough that the agent doesn't default to a built-in like `Grep` over a more capable MCP tool. So the resolution is **fewer *and* better-described** — not one at the expense of the other.

---

## 3. `tool_choice`

A **per-request API parameter**. It constrains what the model may produce on *this one* API call. It is not a property of the agent or of the conversation.

### 3.1 The options

| Value | Behavior | Use when |
|---|---|---|
| `{"type": "auto"}` | Model decides: zero, one, or more tools, or plain text. **Default when tools are provided.** | Normal agentic loop operation; the model must be able to stop. |
| `{"type": "any"}` | Model **must** call a tool, but chooses which. Cannot return conversational text as final output. | A tool call must be guaranteed, but the choice is legitimately the model's. |
| `{"type": "tool", "name": "X"}` | Model **must** call tool X. | One deterministic known call — a required first step, or guaranteed structured output. |
| `{"type": "none"}` | No tool may be called. | Force a text-only turn while tool definitions stay in context (and cached). |

### 3.2 Interaction with the agentic loop and `stop_reason`

`stop_reason` is `"tool_use"` **iff** the response contains at least one `tool_use` block.

| `tool_choice` | `"tool_use"` possible? | `"end_turn"` possible? |
|---|---|---|
| `auto` | ✅ | ✅ |
| `any` | ✅ (guaranteed) | ❌ **structurally impossible** |
| `{"type":"tool","name":X}` | ✅ (guaranteed, and it's X) | ❌ **structurally impossible** |
| `none` | ❌ | ✅ (guaranteed) |

**The bug (most commonly tested `tool_choice` mistake):** hold `any` or a forced tool across the loop and `stop_reason` is never `"end_turn"`, so the `break` is **dead code**. With `{"type":"tool"}` it calls the *same tool* every iteration until the iteration cap, budget, or token limit stops it. **The model is not malfunctioning — it finished the task and would like to reply; you have forbidden it.**

If your loop terminates on an iteration cap, this bug is *invisible* — it looks like "hit its budget" rather than "finished on turn 2 and spun 18 more times."

**Correct pattern — force once, then relax:**

```python
tool_choice = {"type": "tool", "name": "extract_metadata"}

while iterations < MAX:
    response = client.messages.create(..., tools=TOOLS, tool_choice=tool_choice)
    messages.append({"role": "assistant", "content": response.content})

    tool_choice = {"type": "auto"}        # <-- relax after request 1

    if response.stop_reason != "tool_use":
        break

    results = [execute(b) for b in response.content if b.type == "tool_use"]
    messages.append({"role": "user", "content": results})   # tool_result MUST be role:user
    iterations += 1
```

The forced constraint is a property of **request 1**, not of the conversation. This is the guide's stated skill: *force a specific tool first (e.g. `extract_metadata` before enrichment tools), then process subsequent steps in follow-up turns.* The iteration cap stays as a **safety backstop** — it's an anti-pattern only when it's the *primary* stopping mechanism.

**Edge behaviors:**

- **`max_tokens` + forced tool use** — truncating mid-arguments yields `stop_reason: "max_tokens"` with an incomplete `tool_use` block. The loop exits correctly, but the partial input must not be treated as valid.
- **Single-turn extraction needs no loop.** For Scenario 6-style extraction: one request, forced tool, read `response.content[0].input`, done. **Forced `tool_choice` is safest precisely where there is no loop** — recognize that shape and the infinite-loop distractor doesn't apply.
- **Tool results are always `"role": "user"`** with `tool_result` blocks matching `tool_use_id`. That's the Messages API contract, not a workaround.

### 3.3 The other named skill

**`tool_choice: "any"` to guarantee a tool call rather than conversational text.** Fixes the failure where the model replies with prose ("I'll look that up for you…") and returns `end_turn` with no `tool_use` block for the harness to execute. `any` makes that structurally impossible.

`tool_choice` constrains *whether* a tool is called, not *how many* — multiple tool calls in one response (e.g. parallel `Task` spawning, 1.3) are still permitted unless `disable_parallel_tool_use` is set.

### 3.4 `tool_choice` as the structured-output mechanism (forward link to 4.3)

Define exactly one tool whose `input_schema` is the target JSON schema, then force it:

```python
response = client.messages.create(
    tools=[EXTRACTION_TOOL],
    tool_choice={"type": "tool", "name": "record_extraction"},
)
data = response.content[0].input     # guaranteed schema-conforming
```

Far more reliable than "respond with JSON only" in the prompt, because conformance is enforced at the **sampling layer**, not requested in prose.

### 3.5 Secondary behaviors

- **Extended thinking is incompatible with forced tool use** — it works with `auto` or `none`; `any`/`tool` disables it. If a scenario needs careful reasoning about *whether* to act, forcing a tool removes that reasoning.
- **Changing `tool_choice` between requests invalidates cached message blocks** for prompt caching — another reason "force once, then auto" beats "toggle constantly."
- **`disable_parallel_tool_use: true`** restricts the model to at most one tool call per turn; useful when side effects must be strictly serialized. Contrast 1.3, where emitting *multiple* `Task` calls in one response is the correct parallel-spawn pattern.

---

## 4. The four layers — the central decision framework

| Layer | Mechanism | Question it answers | When it acts | Guarantee | Cost of overuse |
|---|---|---|---|---|---|
| **Distribution scopes** | `AgentDefinition.tools`, `allowedTools`, MCP scoping | *Can this agent ever do X?* | Design time | Absolute — tool isn't in context | Coordinator round-trips for legitimate needs |
| **Descriptions guide** | Tool `description`, schema docs, subagent `description` | *Which available tool fits?* | Model's decision, every turn | Probabilistic (high if well written) | Context bloat; non-zero misroute rate |
| **`tool_choice` forces** | Per-request API param | *Must a tool be called now, and which?* | Sampling time, this request only | Absolute for **this one request** | Infinite loops; disables thinking; cache invalidation |
| **Hooks block** | `PreToolUse` / `PostToolUse` | *Given state and these arguments, is this call permitted?* | Between decision and execution | Absolute and **conditional** | Brittle rules; agent hits walls with no path forward |

**Distinguishing axes:** hooks are the only layer that inspects **arguments** and **prior state**. `tool_choice` is the only layer that can **compel**. Distribution is the only layer that removes the option **permanently**. Descriptions are the only layer operating on **judgment** rather than mechanism.

**Routing questions, in order:**

1. Should this agent **ever** do this? No → **distribution**.
2. Must something happen **unconditionally** on this request? Yes → **`tool_choice`**.
3. Does the rule depend on **arguments or prior state**? Yes → **hook**.
4. Is it a **judgment call** among legitimate options? Yes → **descriptions**.

### Worked examples

**1. "Refunds must never exceed $500 without a human."**
❌ Description ("do not use above $500") — probabilistic, unacceptable for money. ❌ Distribution — kills the 95% of refunds that are fine. ❌ `tool_choice` — cannot inspect `amount`, cannot express "unless."
✅ **PreToolUse hook** — reads `input.amount`, denies above 500, redirects to `escalate_to_human`. *The rule is conditional on an argument value; only hooks see arguments.* (Guide 1.5 names this case.)

**2. "`get_customer` must succeed before `process_refund`."**
❌ Forced `tool_choice` on turn 1 — forces `get_customer` even for "where's my package?", and if left forced you can never refund. ❌ Description — non-zero failure on an irreversible financial action.
✅ **PreToolUse prerequisite gate** — checks session state for a verified customer ID, denies with corrective guidance. *The rule is conditional on prior state.* (Guide 1.4 names it verbatim.)
> **Unconditional ordering → `tool_choice`. Conditional ordering → hook.** (`extract_metadata` first, always = unconditional.)

**3. "Model calls `analyze_content` when it should call `extract_web_results`."**
❌ Hook — leaves the agent stuck with no signal; the tool is legitimate elsewhere. ❌ Forced `tool_choice` — requires *you* to already know the right tool, defeating the model-driven loop. ❌ Distribution — may break the other use case.
✅ **Descriptions (2.1)** — rename, rewrite web-specifically, add boundaries and when-to-use-instead; then check the **system prompt for keyword-sensitive phrasing** creating unintended associations.

**4. "The synthesis agent keeps running web searches."**
❌ Prompt "do not search" — non-zero failure rate, and the agent is *motivated* to violate it. ❌ Hook — works, but pays interception cost forever to police a capability with no legitimate use. ❌ `tool_choice` — nothing is being compelled.
✅ **Distribution** — remove `web_search` from its `tools`. *When the answer is "never, for this agent," don't hand over the tool.*
> **Hooks are for conditional denial; distribution is for categorical absence.**

**5. "Extraction sometimes returns prose instead of JSON."**
❌ "Always respond with valid JSON" — classic unreliable pattern. ❌ Hook — nothing to intercept; no tool call was made. ❌ Distribution — the tool is already the only one.
✅ **`tool_choice: {"type":"tool","name":"record_extraction"}`** — *only `tool_choice` compels.*

---

## 5. Distribution × `AgentDefinition` (link to 1.3)

Distribution is the *decision*; `AgentDefinition` is where you *encode* it. Three nesting layers:

```
options.allowedTools          ← what the top-level/coordinator agent may use
   └─ AgentDefinition.tools   ← what each subagent type may use
        └─ hooks              ← runtime conditions on any of the above
```

```typescript
const options = {
  allowedTools: ["Task"],            // coordinator delegates; it does not research
  agents: {
    "search": {
      description: "Searches the web for sources on an assigned subtopic.",
      prompt: "You are a search specialist. Given a subtopic, find high-quality
               primary sources. Return findings as structured JSON with
               {claim, source_url, snippet} per item. Do not synthesize.",
      tools: ["web_search", "fetch_search_result"],
    },
    "doc-analysis": {
      description: "Extracts data points from a provided document.",
      prompt: "...",
      tools: ["load_document", "extract_data_points"],
    },
    "synthesis": {
      description: "Combines findings from search and analysis into a coherent
                    narrative with preserved attribution.",
      prompt: "You are given all findings inline. Synthesize them. You have
               verify_fact for spot-checking a single suspicious claim. If an
               entire subtopic is missing, do NOT research it — report the gap
               so the coordinator can re-delegate.",
      tools: ["verify_fact"],        // the scoped cross-role tool
    },
    "report": {
      description: "Produces the final cited report.",
      prompt: "...",
      tools: ["Write", "format_citations"],
    },
  },
};
```

**Four testable points:**

1. **`allowedTools` must contain `"Task"`** or the coordinator cannot spawn (stated verbatim in 1.3). A "coordinator returns text instead of delegating" scenario often has this root cause — and it is *not* a `tool_choice` problem, even though the symptom looks identical. **Discriminator: missing `Task` predicts 0% success, not partial success.**
2. **Omitting `tools` on a subagent is the bloat default** — it inherits the parent's full tool set, silently erasing every specialization argument. *The restriction is opt-in; the misuse is opt-out.*
3. **`description` vs `prompt` do different jobs.** `description` is read by the **coordinator** to decide *which subagent to invoke* — it's a tool description in the 2.1 sense, and overlapping descriptions across subagents cause the same misrouting. `prompt` is the subagent's own system prompt, defining behavior *once selected*, and is where tool-use boundaries are stated.
4. **The prompt reinforces the tool restriction; it does not substitute for it.** The synthesis agent has `verify_fact` (structural scope) *and* an instruction to report gaps rather than investigate (behavioral guidance). Structure prevents the capability; the prompt shapes behavior inside what remains. A prompt saying "don't search" while `web_search` sits in `tools` is the enforcement mismatch 1.4 warns about.

---

## 6. Anti-patterns checklist (high-yield)

1. **"Give every agent every tool; the descriptions will sort it out."** Descriptions get *worse* at their job as candidates multiply. Distribution is the primary control.
2. **Giving a specialist a generic tool "just in case."** `web_search` on the synthesis agent guarantees cross-specialization misuse.
3. **Leaving `tool_choice: "any"` / forced set across the whole loop.** The model can never `end_turn`; infinite loop.
4. **Using `tool_choice` to paper over overlapping descriptions.** Wrong layer — fix the descriptions (2.1).
5. **Using `tool_choice` for ordering constraints that depend on prior state.** Wrong layer — prerequisite gate / `PreToolUse` hook (1.4, 1.5).
6. **Widening a tool set to solve a small cross-role need.** Give a *narrower* scoped tool (`verify_fact`), or route through the coordinator.
7. **Keeping a broad generic tool when a validating one exists.** `fetch_url` → `load_document`.
8. **Over-splitting into micro-tools** so one agent holds twelve sequential steps — the decision-complexity tax moved, not paid down.
9. **Adding MCP servers without pruning** — every server's tools go live at connection time.
10. **Adding a tool so the model can't forget a rule** — that's usually a hook.
11. **Classifying intent up front and dictating the tool** — replaces the agentic loop with a pre-configured decision tree (1.1).

---

## 7. One-paragraph summary

Tool distribution and `tool_choice` are the two levers above individual tool design. Distribution is the **design-time** control: give each agent the smallest set of clearly distinguishable tools its role needs (4–5, not 18), because extra tools degrade selection reliability and invite cross-specialization misuse; handle small cross-role needs with a *narrower* scoped tool and complex ones by routing through the coordinator; and prefer constrained tools (`load_document`) over generic ones (`fetch_url`) so restrictions live in the schema, not in instructions. `tool_choice` is the **runtime, per-request** control: `auto` for normal loop operation, `any` to guarantee a tool call instead of prose, and `{"type":"tool","name":...}` to force one specific known step — applied on the first request and then relaxed to `auto`, because leaving it forced makes `end_turn` impossible. **`tool_choice` forces, hooks block, descriptions guide, distribution scopes** — matching the failure to the right layer is what the exam actually scores.

---

# Practice Questions

## Question 1 — Synthesis agent producing contradictory reports

**Scenario 3 — Multi-Agent Research System**

The system uses a coordinator delegating to four subagents: search (`web_search`, `fetch_search_result`), document analysis (`load_document`, `extract_data_points`), synthesis (`verify_fact`), and report (`Write`, `format_citations`).

In ~40% of research runs the synthesis agent produces reports containing at least one claim that contradicts a source cited elsewhere in the same report. In each case the contradicting information **was present in the findings the synthesis agent received** — the agent had the correct data but combined it inconsistently.

An engineer proposes giving the synthesis agent `web_search` so it can independently re-check facts it is unsure about. **Which action is the most effective response?**

- **A.** Add `web_search` as proposed, and instruct it in the system prompt to search only when it detects a contradiction between findings.
- **B.** Reject the proposal; instead have the coordinator run a second synthesis pass with `tool_choice: {"type": "tool", "name": "verify_fact"}` so every synthesis turn is forced to verify at least one claim.
- **C.** Reject the proposal, because it does not address the root cause; the failure is in how the synthesis agent reasons over findings it already has, so the fix belongs in the synthesis agent's prompt and the structure of the findings passed to it.
- **D.** Reject the proposal, and remove `verify_fact` as well, routing all fact-checking through the coordinator so verification is centrally observable.

**Correct answer: C** *(answered B — incorrect)*

**Why C is right.** The scenario states the agent *had the correct data but combined it inconsistently*. Both the claim and its contradiction were already in the findings. The failing step is **reasoning over context already possessed** — no tool call is missing, so no tool-layer change can fix it. Root cause lives in the two things shaping that reasoning: (1) the synthesis agent's **system prompt** (identify claims appearing in multiple findings, reconcile conflicts explicitly, flag irreconcilable ones rather than silently picking one), and (2) the **structure of the findings passed in** — 1.3's skill of *using structured data formats that separate content from metadata to preserve attribution*. Flat prose makes cross-referencing hard; structured records keyed by claim make contradictions visible.

**Why B is wrong.** Three independent failures. (1) **It doesn't touch the root cause** — `verify_fact(claim, source_url)` checks claim-vs-*source*, and both contradicting claims are individually supported by their own sources, so it returns `supported: true` for both. The failure is claim-vs-*claim*. (2) **"Forced to verify at least one claim" is the wrong guarantee** — forcing a call guarantees an *action*, not a *correct* action; the model chooses which claim, and it doesn't reliably notice the contradiction, which is the bug. (3) **It's the `tool_choice` loop anti-pattern** — "every synthesis turn is forced" pins `tool_choice`, so `stop_reason` can never be `end_turn` and the agent can never produce its final synthesis. Note what B gets *partly* right: a coordinator-run second pass **is** real (1.2's iterative refinement loop). A second pass targeting **internal consistency** would complement C. The forced-`tool_choice` mechanism is what makes B wrong.

**Why A is wrong.** The exam's explicitly named anti-pattern (*synthesis agent attempting web searches*). It is also **prompt-based enforcement of a capability restriction** (non-zero failure rate, and the agent is motivated to violate it); it fails the **closability test** ("re-check facts I'm unsure about" is open-ended, so it belongs to the coordinator); and it wouldn't work anyway — an agent that fails to *notice* the contradiction won't trigger the conditional search either.

**Why D is wrong.** Over-corrects by removing a correctly scoped tool. `verify_fact` passes both tests — high-frequency, closed question, bounded output, no capability overlap with the search agent. Removing it forces a full coordinator round-trip per spot-check, exactly the latency the scoped tool exists to avoid. The observability argument is weak: subagent results already flow through the coordinator (hub-and-spoke, 1.2). And like A and B, D changes tool distribution when the defect isn't in tool distribution.

> **Takeaway:** when a scenario says the agent *had the right data*, every answer that adds, removes, or forces a tool is a distractor. Tool-layer options are correct only when information or capability is genuinely missing.

---

## Question 2 — Extraction pipeline that never terminates

**Scenario 6 — Structured Data Extraction**

Insurance-claim extraction must begin by extracting document metadata before any enrichment tools run:

```python
tool_choice = {"type": "tool", "name": "extract_metadata"}

while iterations < 20:
    response = client.messages.create(
        model=MODEL, messages=messages, tools=TOOLS, tool_choice=tool_choice
    )
    messages.append({"role": "assistant", "content": response.content})

    if response.stop_reason != "tool_use":
        break

    results = [execute(b) for b in response.content if b.type == "tool_use"]
    messages.append({"role": "user", "content": results})
    iterations += 1
```

Every run terminates only when `iterations` reaches 20. Token spend is ~8× estimate. Logs show `extract_metadata` executed many times per document with identical inputs. **The extracted metadata itself is correct**, and enrichment tools (`lookup_policy`, `validate_claim_amount`) are never called. **What is the root cause and the most effective fix?**

- **A.** The iteration cap is acting as the primary stopping mechanism (a known anti-pattern). Replace it with a check that inspects the assistant's text for a completion signal such as "extraction complete," and remove the cap.
- **B.** `tool_choice` remains pinned to the forced tool for every request, so the model can never return `stop_reason: "end_turn"` and can never select a different tool. Set `tool_choice` to `{"type": "auto"}` after the first request.
- **C.** Tool results are appended with `"role": "user"`, so the model doesn't recognize them as tool output and re-requests the same tool. Append with `"role": "assistant"` instead.
- **D.** `extract_metadata` and the enrichment tools have overlapping descriptions, so the model defaults to the first tool in the list. Rewrite descriptions with explicit boundaries.

**Correct answer: B** *(answered B — correct)*

**Why B is right.** With the forced `tool_choice` in effect the model **must** emit a `tool_use` block, so `stop_reason` is always `"tool_use"` and the `break` is **dead code**. Because the forced tool is `extract_metadata` *specifically*, the model cannot select the enrichment tools even after metadata is in hand. Every symptom follows:

| Symptom | Explained? |
|---|---|
| Always terminates at exactly 20 iterations | ✅ The cap is the only live exit |
| `extract_metadata` repeated with identical inputs | ✅ Same tool forced each request, same document in context |
| Metadata itself is correct | ✅ Turn 1 worked; the model isn't malfunctioning |
| Enrichment tools never called | ✅ Structurally forbidden by the `name` constraint |
| ~8× token spend | ✅ ~20 requests where 2–3 were needed, each with a growing message array |

Fix = move `tool_choice = {"type": "auto"}` inside the loop after the first request. The cap stays as a safety backstop.

**Why A is wrong.** It observes the *surface* fact (the cap is stopping the loop) but misdiagnoses it as the cause, then proposes a fix that is itself a named 1.1 anti-pattern: **parsing natural language signals to determine loop termination / checking assistant text as a completion indicator**. `stop_reason` is the structured, contract-guaranteed signal; scanning for "extraction complete" is brittle. Worse, with `tool_choice` still forced the model can never produce final text at all, so the string never appears — and A also removes the cap. **A converts a costly bug into a runaway one.**

**Why C is wrong.** It inverts the Messages API contract. Tool results are **required** to be sent in a `user`-role message as `tool_result` blocks matching the `tool_use_id`. Sending them as `assistant` would be malformed. This is the one line in the snippet that needs no change. It's a good distractor because "the model re-requests the same tool" *sounds* like the model failing to see the result — but the model isn't requesting anything; it's being compelled.

**Why D is wrong.** Descriptions influence selection only under `auto` or `any`. Under `{"type":"tool","name":...}` the name is dictated by the API parameter, so descriptions are irrelevant — rewrite them perfectly and behavior is byte-identical. D also invents a mechanism that doesn't exist ("defaults to the first tool in the list"); tool ordering is not a selection tiebreaker.

> **Takeaway:** a loop that only ever stops at its cap and repeats one tool → check `tool_choice` first. And reason from the *full* symptom set: "the extracted metadata is correct" eliminates three of four options at once.

---

## Question 3 — Support agent regression after tool-set growth

**Scenario 1 — Customer Support Resolution Agent**

Launched with four MCP tools; product requests grew the set to sixteen (`get_customer_by_email`, `get_customer_by_phone`, `lookup_order`, `lookup_order_history`, `lookup_shipment_status`, `process_refund`, `process_partial_refund`, `process_store_credit`, `issue_replacement`, …). First-contact resolution fell from **81% to 64%**. Two dominant failure modes: refund requests sometimes routed to `process_store_credit` or `issue_replacement`; and `lookup_order_history` is called where `lookup_order` would suffice, adding latency and filling context with irrelevant orders. **Each of the sixteen tools has an accurate, well-written description.** **Most effective first step?**

- **A.** Consolidate near-neighbor tools into parameterized tools at task altitude — e.g. `get_customer(identifier, identifier_type)` and `process_resolution(order_id, resolution_type, amount, reason)` — reducing the set to a small number of clearly distinguishable tools.
- **B.** Add a `PreToolUse` hook that blocks `process_store_credit` and `issue_replacement` whenever the customer's message requested a refund, redirecting to `process_refund`.
- **C.** Introduce a lightweight classification step that inspects each incoming message and, based on detected intent, issues the API request with `tool_choice` set to forced selection of the appropriate tool.
- **D.** Split the ambiguous tools further into narrower single-purpose tools with even more explicit descriptions — e.g. `lookup_order_by_id`, `lookup_recent_orders`, `lookup_orders_in_date_range`.

**Correct answer: A** *(answered A — correct)*

**Why A is right.** The root cause is **decision complexity** — the guide's stated principle. The FCR drop tracks 4 → 16 tools, and the failure modes are exactly what near-neighbor clusters produce: four candidates for "make me whole," three for "what's the state of my order?" Consolidation converts a **selection problem into a parameter-filling problem** — the model picks a field value inside one tool, where the distinctions sit side by side in a single schema. It passes the **merge test** (shared inputs, shared outputs, one question) and so is not the mega-tool anti-pattern. It also fixes **both** reported failure modes; B and C leave the `lookup_order_history` latency problem untouched. The line *"each of the sixteen tools has an accurate, well-written description"* is deliberately placed to eliminate every description-layer fix.

**Why B is wrong.** (1) It asks a hook to do something hooks cannot do — hooks are deterministic code inspecting **arguments and prior state**; *"whenever the customer's message requested a refund"* is a natural-language intent judgment, requiring an embedded classifier that will misfire on phrasings like "I don't want store credit, I want my money back." (2) The premise is wrong anyway — store credit or replacement is sometimes the *correct* resolution even when the customer opened by asking for a refund; that's a policy judgment the agent should make. Contrast the correct hook use in this same scenario: *"block `process_refund` when `amount > 500`"* reads a **structured argument** against a **fixed threshold**.

**Why C is wrong.** It **replaces model-driven decision-making with a pre-configured decision tree** (the 1.1 distinction). It also **relocates the problem without solving it** — the classifier faces the same 16-way discrimination; **breaks multi-step resolution** — forcing one tool per turn based on the incoming message can't express "verify, then look up, then refund," and the classifier only sees turn 1; costs an **extra model call per turn**; and loses adaptability to what tool results actually returned. *Rule of thumb: when an option classifies intent up front and dictates the tool, ask whether you still have an agent.*

**Why D is wrong.** It applies 2.1's "split generic tools" advice where it backfires. That advice targets a **single generic tool doing genuinely different things**; here the tools are already distinct and already well described. Splitting takes 16 → 17+ and **adds a new near-neighbor cluster** to a system already failing from cluster density. The stated rationale is refuted by the scenario (descriptions are already good) — the ambiguity is in the *number of plausible candidates*, not the wording. More text per tool × more tools also worsens attention dilution.

> **Takeaway:** split when one tool does several jobs; **merge** when several tools do one job with different parameters. "The descriptions are all good" is the exam's signal that the fix is structural, not editorial.

---

## Question 4 — Coordinator returning a prose plan

**Scenario 3 — Multi-Agent Research System**

The coordinator is configured with `allowedTools: ["Task"]` and a system prompt describing its subagents and decomposition responsibilities. In ~15% of requests it returns a well-formed plan in prose — *"I'll first search for recent publications, then analyze the three key papers, then synthesize"* — with `stop_reason: "end_turn"` and no `tool_use` blocks. The harness exits the loop and returns the prose as the final answer; no subagents run. In the other **85%** it emits its `Task` calls correctly. **Most effective fix?**

- **A.** Add `"Task"` to the coordinator's `allowedTools`, since a coordinator cannot spawn subagents unless `allowedTools` includes `"Task"`.
- **B.** Have the harness detect responses with no `tool_use` blocks, parse the prose plan to identify intended subagents and assignments, and spawn the corresponding `Task` calls programmatically.
- **C.** Issue the coordinator's initial delegation request with `tool_choice: {"type": "any"}`, then use `{"type": "auto"}` for subsequent requests in the loop.
- **D.** Revise the system prompt to state it must always respond by calling `Task` and never reply with a textual plan, and add a few-shot example of a correct multi-`Task` response.

**Correct answer: C** *(answered B — incorrect)*

**Why C is right.** This is exactly what the guide's named skill exists for: *setting `tool_choice: "any"` to guarantee the model calls a tool rather than returning conversational text.* Under `{"type":"any"}` the model **must** emit at least one `tool_use` block, so `end_turn` is structurally impossible on that request — the 15% failure goes to **zero**, not "improves," because it's an API-level constraint rather than a behavioral tendency. Three details: **`"any"` rather than `{"type":"tool","name":"Task"}`** is the better expression of intent and stays correct if the coordinator later gains a second tool; **multiple `Task` calls in one response are still permitted** (`tool_choice` constrains *whether*, not *how many*), preserving 1.3's parallel spawning unless `disable_parallel_tool_use` is set; and **the relaxation to `auto` is load-bearing** — after delegation the coordinator must receive results, evaluate coverage, possibly re-delegate, and eventually `end_turn`.

**Why B is wrong.** It commits 1.1's named anti-pattern: **parsing natural language signals to drive loop control flow**. (1) The harness behavior it "fixes" is already correct — exiting when `stop_reason != "tool_use"` is the correct loop. (2) It requires a **prose-plan parser** — a new, untested, non-deterministic component in the control path; the prose names no subagent types, no `AgentDefinition` keys, no scope partitions, and the next plan will be phrased differently. (3) **Reconstructed briefs will be lossy** — from 1.3, subagents receive *only* what's in their prompt, so a brief inferred from a vague sentence produces a poorly-scoped subagent; even "successful" parsing yields silent quality degradation, which is worse for debugging than a visible failure. (4) It leaves the actual defect in place — the coordinator still fails 15% of the time. Cost comparison: B is a parser + spawning logic + ongoing maintenance, and probabilistic; C is one parameter on one request, and absolute.

**Why A is wrong.** The scenario states `allowedTools: ["Task"]` — it's already there. The **85% success rate is the independent proof**: if `Task` were missing the coordinator could *never* spawn, so success would be 0%, not 85%. **Any hypothesis predicting total failure is refuted by partial success.** A is well built because the `allowedTools` requirement is a real 1.3 fact and the symptom genuinely matches what missing-`Task` produces; the discriminator is reading the given config and checking against the failure *rate*.

**Why D is wrong.** Right direction, wrong layer — and "most effective" does the work. The coordinator already has a prompt describing delegation and complies 85% of the time; strengthening it plus few-shot might reach ~95%, but it remains **prompt-based guidance with a non-zero failure rate** when a deterministic API-level mechanism exists for the exact requirement. It also costs tokens on every request permanently. D would be correct if the problem were *which* subagent to pick or *how* to partition scope — judgment calls no API parameter can enforce. It's wrong for *whether it acts at all*.

---

## Question 5 — Read-only exploration agent modifying the working tree

**Scenario 4 — Developer Productivity with Claude**

A codebase-exploration agent helps engineers understand unfamiliar and legacy systems. It is read-only by design — a separate code-generation agent handles modifications. Its `AgentDefinition` grants `Read`, `Write`, `Edit`, `Bash`, `Grep`, `Glob`. It **legitimately relies on `Bash`** for `git log`, `git blame`, `ls -R`, `wc -l` — commands `Grep` and `Glob` cannot express. Incident reports show it has run `git checkout .` (discarding uncommitted work), redirected output over an existing file with `>`, and invoked `rm` on suspected dead code. Its system prompt **already** states it is read-only. **Which change most effectively guarantees it cannot modify the working tree, while preserving investigative capability?**

- **A.** Remove `Write` and `Edit` from `AgentDefinition.tools`, and replace `Bash` with a constrained `run_readonly_command` tool that validates each command against an allowlist of read-only commands and returns a structured validation error for anything else.
- **B.** Keep the tool set and add a `PreToolUse` hook on `Bash` denying commands matching a denylist of destructive patterns (`rm`, `git checkout`, `git reset`, `>`, `mv`, `truncate`).
- **C.** Remove `Write`, `Edit`, and `Bash` entirely, leaving `Read`, `Grep`, `Glob`, and route any need for command output through the coordinator.
- **D.** Strengthen the system prompt with an explicit prohibited-command list and few-shot examples, and add a `PostToolUse` hook logging every `Bash` invocation for audit and prompt refinement.

**Correct answer: A** *(answered C — incorrect)*

**Why A is right.** The question imposes **two** constraints — guarantee no modification **and** preserve investigative capability — and only A satisfies both. It applies three principles in the right order: (1) **Distribution for the categorical case** — `Write`/`Edit` are *never* appropriate for a read-only analysis agent; no conditional, no threshold, so don't grant them. (2) **Constrained alternative for the mixed case** — `Bash` is different because the agent needs part of it; this is `fetch_url` → `load_document` exactly, replacing a generic capability with a narrower tool whose restriction lives in the implementation and schema. (3) **Allowlist, not denylist** — an allowlist is **deny-by-default** (unforeseen destructive commands fail closed); a denylist is **allow-by-default**. Plus (4) the **structured validation error (2.2)** lets the agent recover and continue rather than stall.

**Why C is wrong.** It over-restricts, and its remedy makes things worse. **It breaks a stated requirement** — the scenario says `Grep`/`Glob` *cannot express* `git log`, `git blame`, `ls -R`, `wc -l`; `git blame` is not a text search. **It fails the frequency test** — running `git log` is both high-frequency and perfectly closable, so it's precisely the case for a narrow scoped tool, not a coordinator round-trip (the same error as stripping `verify_fact` in Q1's option D). **It relocates the hazard upward** — the coordinator would now need unrestricted `Bash`, moving arbitrary execution to the agent with the broadest responsibility and least specialized prompt; the working tree is no safer. **The contrast to hold onto:** removing `web_search` from the synthesis agent was right because that capability was *categorically* wrong for the role; `Bash` here is *partially* right. **Categorical → remove. Partial → constrain.**

**Why B is wrong.** (1) **The denylist is incomplete and permanently so** — it misses `sed -i`, `tee`, `>>`, `find -delete`, `git clean -fd`, `dd`, `chmod`, `python -c "open(f,'w')"`, chaining with `;`/`&&`, and command substitution. You cannot enumerate every way a shell writes to disk. (2) **Fatal on its own: it leaves `Write` and `Edit` in the tool set** — "keep the tool set unchanged" means two full-strength modification tools sit untouched beside the guarded `Bash`, so the agent can modify the tree without invoking `Bash` at all. B doesn't achieve the guarantee even against its own threat model. This isn't an argument against hooks generally — a hook is exactly right for "block refunds over $500," a conditional threshold on a structured argument. It's wrong here because the rule is categorical.

**Why D is wrong.** Pre-refuted by the scenario: the prompt *already* states read-only, and that control has failed three times in production. The `PostToolUse` hook **logs; it does not prevent** — by the time it observes `git checkout .`, the uncommitted work is gone. **PreToolUse prevents, PostToolUse reacts** (1.5). "Refine the prompt over time" is the wrong response to irreversible data loss, per 1.4's deterministic-compliance rule.

---

## 8. Clarifications raised during follow-ups

- **Why give a specialist *any* extra tool?** You don't, by default. The cross-role tool must be earned by the **frequency × closability** test, and it must be *narrower* than the capability it substitutes for — `verify_fact`, never `web_search`. The tool count grows by one; the capability surface barely moves.
- **Merge vs. split, reconciled.** Both are moves toward the same target (a small set of clearly distinguishable tools at task altitude). Split when one tool does several jobs with different schemas; merge when several tools do one job with different parameter values.
- **The `allowedTools` / `tool_choice` discriminator.** "Coordinator returns text instead of delegating" has two candidate causes. Missing `Task` predicts **0%** success; a behavioral tendency predicts **partial** success. Read the failure rate.
- **Tool results are always `role: "user"`.** Not a workaround — the Messages API contract.
- **Forced `tool_choice` is safest where there is no loop.** Single-turn extraction (force → read `content[0].input` → done) has no termination problem at all.
- **`tool_choice` constrains *whether*, not *how many*.** Parallel `Task` spawning survives `tool_choice: "any"` unless `disable_parallel_tool_use` is set.
- **When a generic tool is still correct.** If the agent legitimately needs the full range (e.g. `Bash` for a genuine shell task), constraining it just means the agent works around you — usually with less validation and less visibility.
- **Layer-selection is the scored skill.** Across these five items, the tempting wrong answers were all elaborate compensating machinery (a forced verification pass, a text-parsing completion check, a prose-plan parser) or the strongest available restriction, in place of the smallest change at the layer where the defect lives.

### Quick layer table for revision

| Defect | Layer |
|---|---|
| Agent reasons badly over data it already has | Prompt / input structure — **not** tools |
| Agent could do something it should never do | **Distribution** — don't grant the tool |
| Agent needs part of a broad capability | **Constrained tool** — allowlist in the schema |
| Rule depends on an argument value or prior state | **PreToolUse hook** |
| An action must happen on this request | **`tool_choice`** — forced once, then `auto` |
| Model picks the wrong tool among valid ones | **Descriptions** |

**Two mechanical facts most likely to appear verbatim:** forced `tool_choice` held across a loop makes `end_turn` impossible; `tool_choice: "any"` is the fix for "returned prose instead of calling a tool."
