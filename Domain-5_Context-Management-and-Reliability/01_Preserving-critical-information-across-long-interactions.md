# Topic 5.1 — Manage conversation context to preserve critical information across long interactions

**Domain:** 5 — Context Management & Reliability (15% of exam)
**Task Statement:** 5.1
**Topic number:** 1 of 6 within Domain 5 · 25 of 30 overall

---

## The one-sentence version

The context window is a **budget you architect, not a buffer you fill**: critical facts must live in a *stable, structurally privileged* location (a persistent facts block, near the top, in structured form), while everything lossy — summaries, verbose tool payloads, reasoning chains — is *compressed or trimmed before it accumulates*, never after it has already destroyed the facts.

The exam tests **diagnosis before fix**: three distinct failure mechanisms live in this task statement and their fixes are not interchangeable.

---

## 1. The three failure mechanisms

| # | Mechanism | Symptom in the scenario | Fix family |
|---|---|---|---|
| 1 | **Progressive summarization loss** | Agent "forgets" a specific number/date/promise it demonstrably knew earlier in the same conversation | Persistent structured **case facts** layer, exempt from summarization |
| 2 | **Lost in the middle** (position effect) | Information is *present* in context, but middle-of-input findings are omitted from the output | **Reposition + structure**: key findings first, explicit section headers |
| 3 | **Context bloat from tool results** | Context fills fast, cost/latency climb, agent degrades over a long session | **Trim at ingestion** — filter tool output fields before they enter context |

The most common trap is applying the fix for mechanism 2 (or a bigger model, or a bigger window) to mechanism 1.

---

## 2. Mechanism 1 — Progressive summarization destroys transactional facts

### What happens

Long conversations exceed the practical budget, so the harness compresses old turns into a running summary. Summarization is **lossy in a predictable direction**: it preserves *gist* and discards *precision*.

Exam guide wording: the risk is condensing **numerical values, percentages, dates, and customer-stated expectations** into vague summaries.

| Turn 3 actual content | What the summary becomes |
|---|---|
| "Order #A-4471, placed March 3, $247.80, refund promised within 5 business days" | "Customer discussed a recent order and a refund" |
| "Agent offered a 15% goodwill credit" | "Agent offered a discount" |
| "Customer needs it before their trip on the 22nd" | "Customer is in a hurry" |

By turn 20 the agent contradicts its own commitment or re-asks for the order ID. **What failed is not reasoning and not retrieval — the token that held the fact no longer exists in context.** No prompt can recover a deleted value.

Secondary tell: when the agent produces a *wrong but plausible* number that no tool ever returned (e.g. the current subtotal instead of the approved refund), it reconstructed from what remained visible. That is diagnostic of summarization loss, not of a tool bug.

### The fix: a persistent "case facts" block outside summarized history

Split context into two layers with **different lifecycles**:

```
┌─ System prompt (stable) ──────────────────────────┐
├─ CASE FACTS (rebuilt & re-injected every request) ┤   ← never summarized
│  customer_id: C-88213                             │
│  orders: [{id: A-4471, date: 2026-03-03,          │
│            total: 247.80, status: delivered}]     │
│  commitments: [{type: refund, amount: 247.80,     │
│            promised_by: 2026-03-10, turn: 3},     │
│           {type: goodwill_credit, pct: 15}]       │
│  customer_expectations: ["needs resolution before  │
│            trip on 2026-03-22"]                   │
├─ Summarized older turns (lossy — acceptable) ─────┤
└─ Recent verbatim turns ───────────────────────────┘
```

Four properties the exam rewards:

1. **Extraction is continuous — at the moment the fact appears.** Extracting at summarization time extracts nothing; the precision is already gone.
2. **Re-injected in full on every request.** A tool the agent *might* call is probabilistic; a fact in the prompt is deterministic.
3. **Structured, not prose.** `refund_amount: 247.80` cannot soften into "a refund." Prose facts get re-summarized.
4. **Exempt from compression by construction** — the summarizer's input is the transcript, and the facts block is not part of it.

### Multi-issue sessions need their own layer

One message may contain a billing dispute *and* a shipping problem *and* an account lockout. Across a long session these **cross-contaminate**: a remedy gets applied to the wrong issue, or the agent declares "all set" with issues still open.

Diagnostic: **the values it cites are individually correct, but bound to the wrong entity, or completeness across items fails.** What was lost is the *binding* and the *per-issue state*, not the facts.

Fix — persist **structured issue data as a separate context layer**, one record per issue:

```
ISSUES:
  1. duplicate_charge | order A-4471 | $247.80 | action: refund_approved | state: refund_pending
  2. shipping_delay   | order A-4502 | —       | action: reship_arranged | state: open
  3. account_lockout  | —            | —       | action: unlock_applied  | state: resolved
```

The guide lists this **separately** from the case-facts skill precisely because a flat facts block is insufficient: facts need an owner and a state.

### The counterpart — don't under-send either

The API is stateless (Domain 1.1), so you must **pass the complete conversation history in each subsequent request** to maintain coherence. Sending only the latest message plus a summary produces an agent that re-asks answered questions. The correct posture is not "send less"; it is: **send everything that matters, in a form that survives compression, and stop sending what never mattered.**

---

## 3. Mechanism 2 — "Lost in the middle"

### What happens

Models process the **beginning and end** of long inputs reliably and may omit findings from the **middle**. It is a *position* effect, not a capacity effect.

Canonical exam setting: **aggregated inputs.** A coordinator concatenates six subagent reports and asks for synthesis; the final report covers reports 1 and 6 and silently drops report 3. Likewise a review over 40 concatenated files that flags issues only near the ends.

**The tell: nothing was truncated, no error occurred, input quality is uniform — output coverage varies with input position.** Whenever a scenario states the full input fit in the window, every capacity-based answer is eliminated.

### The fix — two moves, both required

1. **Put a compact key-findings summary at the very beginning** of the aggregated input, relocating must-not-be-lost material into the reliably-processed region.
2. **Organize detailed results under explicit section headers** (`## Subagent 3 — Regulatory landscape (findings 7–9)`), giving the middle addressable structure and letting the model walk from each lead-summary line to its supporting detail.

Distractors that do **not** work:

- ❌ "Instruct the model to read all sections carefully / attend equally." Prompt pressure against a position effect.
- ❌ "Use a larger context window." Longer inputs make position effects **worse**; the mechanism is relative position within the input, not window occupancy.
- ❌ "Enable extended thinking." More reasoning over the same badly-ordered input.
- ❌ "Require the agent to confirm it incorporated all sources." Self-attested by the agent whose attention *is* the defect — converts a silent omission into a silent omission plus a false assurance. It also sits at the end of the prompt, doing nothing for the stranded middle.
- ✅ Adjacent but a **different** topic: splitting into scoped passes (Domain 1.6 / 4.6). Choose by emphasis — *findings present but omitted from a synthesis* → reposition + headers; *surface too large for one pass to hold* → multi-pass.

---

## 4. Mechanism 3 — Tool results consume context disproportionately to relevance

### What happens

Tool results are appended to conversation history (Domain 1.1) and **remain there for the rest of the session**, re-sent on every subsequent request. Backend payloads are shaped for systems: the guide's example is a `lookup_order` returning **40+ fields when only 5 are relevant**.

Compounding: 12 lookups × ~47 fields costs as much as dozens of conversational turns, spent on bin locations and warehouse cost codes. Consequences:

- Budget consumed by noise → summarization triggers **earlier** → mechanism 1 fires.
- The noise enlarges the middle where mechanism 2 lives.
- Cost and latency climb steadily through the session.

### The fix — trim at ingestion, "before they accumulate in context"

That phrase is the whole point: trim **on the way in**, not as cleanup later. Two implementation shapes, both in scope, and the choice between them is about **ownership of the tool**, not effectiveness:

- **Narrow the tool definition** (Domain 2.1) so it returns only decision-relevant fields. First choice when the tool is yours. The noise never enters the system at all.
- **A `PostToolUse` hook** (Domain 1.5) that projects the payload down before the model processes it. Correct when the backend is shared/third-party, or when normalization is also needed (Unix vs ISO 8601 timestamps, numeric status codes) — one place for both.

Both are **deterministic**, which is why they beat the distractors:

- ❌ **"Instruct the agent to ignore irrelevant fields."** Saves **zero tokens** — the payload is already in the `messages` array. Wrong on mechanism, not merely weaker. Every symptom (early summarization, rising cost/latency) is caused by token occupancy.
- ❌ **"Summarize tool results more aggressively."** Spends the one lossy operation on the noise, and aggressive compression is exactly what destroys order numbers, totals, and dates — accelerating mechanism 1. **Trim first, compress only what's left.**
- ❌ **"Drop tool results older than the last N."** Deletes records *whole*, taking the relevant fields with the irrelevant ones, and only caps the bloat rather than removing it. Trim on the **field** axis (relevance), not the **time** axis (recency).

⚠️ Boundary: **trim by relevance to the task, not by volume.** Truncating to the first N characters converts a bloat problem into an information-loss problem (cf. Domain 2.5 — retrieve the minimum that answers the question).

---

## 5. Cross-agent context budgets: the upstream output contract

Subagents return only a string into the coordinator's context (Domain 1.2 / 1.3), so **an upstream agent's output contract is a context-management decision for everyone downstream.** Two skills, and the exam tests failures in *both directions*.

### (a) Require metadata in structured outputs to support downstream synthesis

Subagents must include **dates, source locations, and methodological context** (sample size, study design) as explicit fields. Otherwise:

- No dates → a 2019 finding is merged with a 2026 finding as equally current.
- No source locations → the report cannot cite; attribution is unrecoverable, because the searcher's context is gone.
- No methodology → a 40-respondent survey is weighted equally with a 20,000-participant longitudinal study.

Prose returns are the culprit: *"one large study showing substantial gains"* has already lost the sample size, the design, and the URL.

**The synthesizer cannot reconstruct any of this.** It can only relay what it was handed. Therefore:

- ❌ "Instruct the synthesizer to include dates and citations" — **fixing an upstream contract with a downstream prompt.** It also pressures a model that has no dates, inviting fabricated or vague attribution.
- ❌ "Insert a fact-checking agent to annotate the summaries" — a repair stage that cannot recover what was discarded ("the details **it can determine**" is the giveaway); the only path is re-searching, duplicating work and possibly landing on different sources. **Never add a repair stage where a contract fix is available.**
- ❌ "Pass the full raw source material through as well" — guarantees availability but destroys the reason subagents exist (isolating verbose exploration), overflows the synthesizer, and buries the decisive numbers in the middle where mechanism 2 eats them.

### (b) Return structured data, not verbose content and reasoning chains

When downstream agents have limited context budgets, upstream agents should return **key facts, citations, and relevance scores** — not full document text plus their own chain of thought. Two independent reasons, both examinable:

1. **Volume** — four subagents × 12–18k tokens overflows the synthesizer before it starts.
2. **Signal / anti-anchoring** — importing another instance's reasoning biases the reader (Domain 4.6 contamination, applied to a handoff). The classic symptom: *the synthesizer's judgments mirror whichever subagent argued at greatest length, even when its sources were weakest* — argument length acting as a proxy for evidential strength. A **relevance score** is the compressed, comparable form of that judgment: it transmits the conclusion without the persuasive narrative, and puts all subagents on one scale.

Distractors here:

- ❌ Summarize each subagent's output downstream to a fixed budget — compresses after the fact, loses citations and figures, while *preserving* the argumentative gist that causes the bias.
- ❌ Split synthesis into sequential per-report refinement passes — relieves volume but leaves the reasoning chains intact, trading "longest argument wins" for "last argument wins," and adds an ordering effect plus four sequential calls.
- ❌ Tell subagents to "be concise / return only what's needed" — a soft instruction where a **contract** is required; unmeasurable, varies per run, no guarantee citations survive.

**General handoff rule: transmit conclusions and the evidence needed to verify them; drop the deliberation.**

---

## 6. Decision table — symptom → fix

| Scenario symptom | Mechanism | Correct fix | Attractive wrong answer |
|---|---|---|---|
| Agent contradicts a refund amount it promised at turn 3 | Summarization loss | Persistent structured case-facts block, re-injected each request | Retain more verbatim turns; "summarize more carefully"; bigger model |
| Agent produces a plausible number no tool ever returned | Summarization loss | Case facts | Debug the tool |
| Agent re-asks for an order number it was given | Summarization loss / under-sent history | Case facts + pass complete history | Ask the customer to repeat it |
| Remedy applied to the wrong issue; premature "all set" | Unbound facts / no per-issue state | Separate structured issue layer, one record per issue | One combined prose summary; re-read-and-confirm instruction |
| Synthesis omits findings from middle reports | Lost in the middle | Key findings first + explicit section headers | Larger context window; "read all sections carefully"; extended thinking |
| Context fills fast; cost/latency climb across a session | Tool-result bloat | Trim to relevant fields at ingestion (narrow tool or `PostToolUse` hook) | Tell the agent to ignore irrelevant fields; summarize harder; drop old results |
| Synthesizer can't date or cite claims | Missing upstream metadata | Require dates/sources/methodology as fields in subagent output | Instruct the synthesizer to add citations; add a fact-checking stage |
| Synthesizer overflows and echoes the longest argument | Verbose upstream returns | Key facts + citations + relevance scores | Summarize subagent outputs after the fact; sequential refinement passes |

---

## 7. Anti-patterns worth naming explicitly

1. **Compressing before trimming.** Summarizing a transcript that is 70% raw tool payloads spends the one lossy operation on noise while damaging the facts.
2. **Treating "bigger window" as context management.** It delays overflow and worsens position effects; it fixes neither mechanism 1 nor 2.
3. **Storing critical facts only in a tool the agent may call.** Retrieval is probabilistic; injection is deterministic. The agent that lost a fact doesn't know it's missing, so it never fires the tool.
4. **Prose where structure is needed.** Prose re-summarizes into vagueness and loses the binding to the right entity.
5. **Extracting facts only when summarization triggers.** Too late by construction.
6. **Fixing an upstream contract with a downstream prompt (or a repair stage).** If information never arrived, no instruction to the receiver creates it — same hard limit as Domain 4.4 (retry cannot recover what the source never contained).
7. **Soft verbosity requests instead of field-level contracts.** "Be concise" is unenforceable; a named-field schema is inspectable.

---

## 8. Compact recall list

- Summarization loses **numbers, percentages, dates, customer-stated expectations** → persistent, structured, re-injected **case-facts** layer, outside summarized history, extracted continuously.
- Multi-issue → **separate structured issue layer**, one record each, with identifiers, actions, and state.
- **Lost in the middle**: reliable at start and end → **key findings summary first + explicit section headers**. "No truncation occurred" eliminates all capacity answers.
- Tool results **accumulate** and are **disproportionate to relevance** (40+ fields, 5 useful) → trim **at ingestion** via narrower tool or `PostToolUse` hook. Instructing the model to ignore fields saves zero tokens.
- API is stateless → **pass complete history**; the answer is never "send less of what matters."
- Upstream agents must emit **metadata** (dates, source locations, methodology) and **structured data** (key facts, citations, relevance scores) — never verbose content plus reasoning chains.

---

# Practice Questions

## Question 1 — Summarization loss in a long support conversation

**Scenario 1: Customer Support Resolution Agent.** Billing-dispute conversations run 30+ turns; the harness summarizes turns older than the most recent six. At turn 4 the agent says "I've approved a refund of $247.80 for order #A-4471, which will post within 5 business days." At turn 22 it says the refund "is being processed" and, when pressed, states **$198.00** — the current order subtotal. Logs confirm the turn-4 message was correct and that no tool ever returned $198.00 as a refund figure.

**What is the most effective fix?**

- **A.** Increase retained verbatim turns from six to twenty so the approval stays unsummarized longer.
- **B.** Continuously extract transactional facts (order number, approved refund amount, promise dates, commitments) into a structured case-facts block re-injected in full on every request and excluded from summarization.
- **C.** Instruct the agent never to state a monetary amount unless it can point to the earlier turn where that amount was approved, and to say "let me confirm that for you" otherwise.
- **D.** Store all commitments in the case management system and add a `get_case_commitments` MCP tool the agent can call when a customer asks about a previously discussed amount.

**Correct answer: B** ✅ *(answered correctly)*

**Why B is right.** Mechanism 1. Two clues fix the diagnosis: the turn-4 message *was* correct (reasoning and tool data were never the problem), and no tool returned $198.00 (so the agent reconstructed a plausible figure from what remained visible — the subtotal). That is what a model does when "approved a refund of $247.80" has been compressed to "approved a refund." B fixes the mechanism: structured (can't soften into "a refund"), extracted **continuously**, **re-injected in full**, and **excluded from the summarizer's input** by construction. Commitments and promise dates are exactly the guide's called-out class.

**Why A is wrong.** Buys time without fixing anything, and fails at exactly the length where the problem occurs — turn 4 is still summarized by turn 25. It also spends a large budget re-sending 20 low-density turns to preserve four values a facts block holds in a few dozen tokens. Any threshold is a guess about conversation length.

**Why C is wrong.** Prompt pressure against information loss. The instruction asks the model to check a source that **no longer exists in its context**, so the honest branch fires on every long conversation and the agent can't stand behind its own commitments — a new failure — while the original failure still survives at a non-zero rate.

**Why D is wrong.** The data engineering is fine; the *delivery mechanism* fails. A tool the agent may call is probabilistic; a fact in the prompt is deterministic. The agent doesn't know it's missing information — it feels confident and produces $198.00 — so it never fires the tool. Correcting that would require it to first recognize its own gap, the very thing it failed to do.

> **Rule:** critical facts get **injected**, not **retrieved**. Extraction must happen when the fact appears — extracting from a summary extracts nothing.

---

## Question 2 — Position effects in an aggregated synthesis

**Scenario 3: Multi-Agent Research System.** Six search subagents' reports (~60,000 tokens total) are concatenated for the synthesis subagent. Final reports thoroughly cover reports 1–2 and 6, but findings from reports 3 and 4 are frequently absent. Logs confirm no truncation, and reports 3 and 4 match the others in quality and length.

**Which change most directly addresses the root cause?**

- **A.** Enable extended thinking on the synthesis subagent.
- **B.** Restructure the aggregated input so a compact list of all key findings appears at the very beginning, with detailed reports following under explicit section headers identifying each subagent and subtopic.
- **C.** Move synthesis to a model with a substantially larger context window so 60,000 tokens is a smaller fraction of available context.
- **D.** Add an instruction at the end of the prompt requiring the agent to confirm, report by report, that it incorporated all six sources before producing output.

**Correct answer: B** ✅ *(answered correctly)*

**Why B is right.** No truncation, no error, uniform input quality, coverage varying purely with position → lost in the middle. Because the cause is *where* material sits, the fix must change *where* it sits. B applies both required moves: front-loading relocates every finding (including 3 and 4's) into the reliably-processed opening region, and section headers give the middle addressable structure and a path from each lead-summary line to its detail. This is the guide's stated skill verbatim.

**Why A is wrong.** Extended thinking spends more budget over the **same badly-ordered input**; the middle stays under-weighted. Same trap as Domain 4.6, where extended thinking fails to fix self-review.

**Why C is wrong.** Backwards: longer inputs make position effects **worse**, and "fraction of available context" is a red herring — the mechanism is relative position within the input, not window occupancy. The logs already ruled out a capacity problem.

**Why D is wrong.** Prompt pressure against an attention effect, and the confirmation is **self-attested by the agent whose attention is the defect** — it will assert full coverage because from its perspective it had it. Converts a silent omission into a silent omission plus a false assurance, and sits at the end of the prompt, doing nothing for the stranded middle. A genuine verification layer would be a separate independent pass (4.6), not a self-check in the same call.

> **Rule:** *findings present in context but missing from output* → reposition + structure. *Surface too large for one pass* → split into scoped passes. "No truncation occurred" eliminates every capacity-based answer.

---

## Question 3 — Tool-result bloat (multiple response: select TWO)

**Scenario 1: Customer Support Resolution Agent.** `lookup_order` proxies the WMS and returns the record verbatim — 47 fields (internal SKUs, bin locations, carrier scan events, pick-batch IDs, warehouse cost codes) — when only five matter (order ID, order date, order total, fulfillment status, line items). Sessions involve 8–14 lookups. The team observes: the summarization threshold is hit far earlier than conversation length suggests, per-turn latency and cost climb steadily, and long-session agents increasingly give vague answers about details discussed earlier.

**Which two changes most effectively address the root cause?**

- **A.** Add to the system prompt: "`lookup_order` returns many fields. Attend only to order ID, order date, order total, fulfillment status, and line items; ignore all warehouse-internal fields."
- **B.** Implement a `PostToolUse` hook that projects each result down to the five needed fields before it is appended to the conversation.
- **C.** Increase summarization aggressiveness so accumulated tool results are compressed sooner and more heavily than conversational turns.
- **D.** Redefine the MCP tool to return only the five decision-relevant fields rather than proxying the raw WMS record.
- **E.** Configure the harness to drop all `lookup_order` results older than the three most recent ones.

**Correct answers: B and D** ✅ *(answered correctly)*

**Why D is right.** The cleanest fix: 42 of 47 fields inform no decision. Fixing the **tool definition** means the noise never enters the system — nothing to trim, no per-call overhead. Domain 2.1 (task altitude) + 2.5 (retrieve the minimum) applied to context management. First choice when you own the tool.

**Why B is right.** The answer when the tool is *not* yours — shared backend, third-party MCP server, or a contract other consumers depend on. `PostToolUse` transforms the result **before the model processes it** (Domain 1.5), deterministically, and is also where format normalization belongs (Unix vs ISO 8601, numeric status codes).

**The teaching point:** both are correct because the choice between them is about **ownership of the tool**, not effectiveness.

**Why A is wrong.** The single most important distractor in this topic: instructing the model to ignore fields **saves zero tokens** — the payload is already in the `messages` array and is re-sent every request. Every observed symptom is caused by token occupancy, which an attention instruction cannot touch. Wrong on mechanism, not merely weaker.

**Why C is wrong.** Treats the symptom and causes harm: it spends the one lossy operation on noise, and aggressive compression is precisely what destroys order numbers, totals, and dates — already the third listed symptom. Correct ordering is always **trim first, compress only what's left.**

**Why E is wrong.** Sharpest distractor, since dropping stale results is a real technique. But it deletes records **whole**, taking the five relevant fields with the 42 irrelevant ones — an order raised again at turn 25 is simply gone, worsening the vagueness symptom. And it only caps the bloat: three 47-field records still dominate, and every call still pays full price on the way in.

> **Rule:** trim by **relevance to the task**, at the **ingestion boundary**. Trimming on volume or recency converts bloat into information loss; leaving tokens in the array (A) or compressing afterward (C) has already lost.

---

## Question 4 — Upstream metadata contract *(missed)*

**Scenario 3: Multi-Agent Research System.** Search subagents return narrative prose: *"I searched several databases and found strong evidence that adoption rates are increasing significantly, with one large study showing substantial gains and a few smaller surveys agreeing."* Reviewers report: (1) final reports cannot cite anything and figures can't be traced; (2) reports treat clearly dated findings as current and weight a 40-respondent survey equally with a 20,000-participant longitudinal study.

**Which change most directly resolves both complaints?**

- **A.** Instruct the synthesis subagent: "For every claim, include the publication date, source location, and study methodology, and weight evidence by sample size and recency."
- **B.** Have the coordinator pass each search subagent's full raw source material — complete article text and search result pages — through to synthesis alongside the narrative summaries.
- **C.** Change each search subagent's output contract to a structured format in which every finding carries its source location, publication date, and methodological context (sample size, study design) as explicit fields.
- **D.** Insert a fact-checking subagent between search and synthesis that annotates each narrative summary with the dates, citations, and methodology details it can determine.

**Correct answer: C** — *(answered B; see below)*

**Why C is right.** Both complaints share one root cause: **the information was never transmitted.** Source locations, dates, and sample sizes existed in the searchers' contexts and were discarded when each returned prose — "one large study" has already lost the sample size. Fixing the **output contract** makes each finding carry its own metadata as fields, so it survives the handoff: complaint 1 is solved (traceable source per claim) and complaint 2 is solved (the synthesizer can *see* 20,000 vs 40 and 2019 vs 2026 and weight accordingly). Guide skill verbatim: *requiring subagents to include metadata (dates, source locations, methodological context) in structured outputs to support accurate downstream synthesis*; plus Domain 1.3's *separate content from metadata to preserve attribution*.

**Why B is wrong.** The instinct is right — metadata must actually be present rather than conjured downstream — but B fixes it by removing the compression instead of correcting the contract. It discards the whole purpose of search subagents (isolating verbose exploration so downstream agents needn't hold it) and overflows the synthesizer; the guide names the opposite move as correct (*return key facts, citations, and relevance scores instead of verbose content when downstream agents have limited context budgets*). It also doesn't reliably fix complaint 2: the synthesizer must now extract sample sizes and dates from raw text *while* synthesizing, with the decisive numbers buried mid-input where position effects strike.

**Why A is wrong.** The classic error: **fixing an upstream contract with a downstream prompt.** The synthesizer cannot include a date it was never given; the searcher's context is gone. Demanding citations from a model that has none invites fabricated or vague attribution ("recent studies suggest"). Same hard limit as Domain 4.4.

**Why D is wrong.** Adds an agent and a stage to recover information discarded one step earlier — and it can't. The giveaway is the option's own wording: "the details **it can determine**." From "one large study showing substantial gains," no reviewer can determine sample size, design, or URL; the only path is re-searching, duplicating work and possibly landing on different sources than were actually used. **Never add a repair stage where a contract fix is available.**

> **Rule:** transmit **conclusions + the evidence needed to verify them**; drop the **deliberation**. B keeps the raw content; A and D try to recreate what was dropped; only C changes what gets sent.

---

## Question 5 — Multi-issue state binding

**Scenario 1: Customer Support Resolution Agent.** Customers open with multiple problems: a duplicate charge on #A-4471, a package from #A-4502 that never arrived, and a locked account. Older turns are summarized past the recent-turn window; a persistent case-facts block already carries the customer ID plus amounts, dates, and order numbers. Late in sessions the agent says "your refund has been processed and your account is unlocked, so you're all set" while the #A-4502 shipping issue is unresolved; in other transcripts it applies the #A-4502 reship to #A-4471. **The amounts and order numbers it cites are always individually correct.**

**What is the most effective fix?**

- **A.** Instruct the agent to re-read the full conversation before any closing statement and confirm each problem the customer raised has been resolved.
- **B.** Reduce summarization aggressiveness for these sessions so more of the multi-issue investigation stays verbatim.
- **C.** Extract and persist structured issue data as a separate context layer — one record per issue with its own identifiers, associated orders, actions taken, and current state — maintained alongside the case-facts block.
- **D.** Decompose the multi-concern request up front and investigate each issue in a separate subagent so each investigation stays in an isolated context.

**Correct answer: C** ✅ *(answered correctly)*

**Why C is right.** The decisive clue is that the cited values are **individually correct** — the case-facts block is working. What was lost is the **binding** between each fact and its issue, plus each issue's **state**. Summarized prose keeps "there were several problems, mostly resolved," so identity and state relationships dissolve even where values persist — producing exactly the two symptoms (premature all-clear; cross-applied remedy). A per-issue structured layer restores both: "all set" becomes checkable against state, and the reship is bound to A-4502 by construction. The guide lists this **separately** from case facts precisely because a flat facts block is insufficient — facts need an owner.

**Why A is wrong.** Prompt pressure pointed at a context that no longer holds the answer: the conversation has been summarized, so "re-read and confirm" verifies against the very prose that lost the state. Self-attested by the agent whose tracking already failed; non-deterministic where the requirement is deterministic (Domain 1.4).

**Why B is wrong.** Same shape as Q1/A — moves the cliff without removing it, at a large token premium to keep low-density turns verbatim in order to preserve three state values a structured layer holds in a few dozen tokens. It also misdiagnoses: the real gap is that nothing durable ever tracked per-issue state.

**Why D is wrong.** Strongest distractor, because the pattern is correct and in scope (Domain 1.4 endorses decomposing multi-concern requests and investigating in parallel). But it addresses the **investigation** phase, while the defect occurs **afterward**, during the long conversation where state must be tracked and reported. Subagent isolation prevents contamination *inside* each investigation and does nothing about the coordinator's later summarization loss — in fact it sharpens the need, since each subagent returns a string and its context is discarded. D and C compose; only C fixes this failure.

> **Rule:** when values are individually correct but attached to the wrong entity, or completeness across items fails, the missing thing is **structure that binds facts to owners and carries state** — not more verbatim history, not a re-read instruction.

---

## Question 6 (bonus) — Verbose upstream returns and imported reasoning

**Scenario 3: Multi-Agent Research System.** Four document-analysis subagents each return their complete working output: full text of every document examined plus step-by-step reasoning about how they evaluated and ranked sources — 12,000–18,000 tokens each. The synthesis subagent now (1) exceeds its context budget on large topics, and (2) when it completes, its final judgments often mirror whichever analysis subagent argued its case at greatest length, including where that subagent's sources were weakest.

**What is the most effective fix?**

- **A.** Have the coordinator summarize each analysis subagent's output down to a fixed token budget before concatenating.
- **B.** Change the analysis subagents' output contract to return structured data — key facts with citations and an explicit relevance score per source — rather than full document text and reasoning chains.
- **C.** Split synthesis into four sequential passes, one per report, each refining a running draft, so no single call holds all four outputs.
- **D.** Instruct the analysis subagents to be concise and return only what the synthesis agent needs.

**Correct answer: B** ✅ *(answered correctly)*

**Why B is right.** Two symptoms, one root cause; B is the only option that changes **what gets sent**. *Overflow* is volume: four × up to 18k tokens is an input built almost entirely from material the synthesizer doesn't need, and structured returns carry the decision-relevant content at a fraction of the cost — the guide's skill verbatim. *Bias* is the more interesting half: "mirrors whichever subagent argued longest" is **anchoring on imported reasoning chains** (Domain 4.6 contamination applied to a handoff), with argument length acting as a proxy for evidential strength. A **relevance score** is the compressed, comparable form of that judgment — it transmits the conclusion without the persuasive narrative and puts all four subagents on one scale, so the synthesizer compares instead of being convinced.

**Why A is wrong.** "Compress after the fact": spends a lossy operation on content that should never have been transmitted, loses citations and precise figures, and *preserves* the argumentative gist that causes the bias — a summary of a persuasive case is still a persuasive case. A fixed token budget also truncates blindly.

**Why C is wrong.** Strongest distractor, since sequential refinement is legitimate (Domain 1.6) and does relieve context pressure. But the contaminating reasoning chains remain fully intact, so the bias persists and gains an **ordering** effect: the last pass to touch the draft dominates and early findings get progressively overwritten. Trades "longest argument wins" for "last argument wins," and spends four sequential calls, surrendering the parallel fan-out's latency benefit.

**Why D is wrong.** A soft instruction where a **contract** is required. "Be concise" and "only what's needed" are unmeasurable, interpreted differently by each subagent, vary run to run, and give no guarantee that citations survive the subagent's own trimming. B specifies *which fields*, which is what makes it deterministic.

> **Rule, now from both sides:** upstream contracts transmit **conclusions + verifiable evidence** (facts, citations, dates, methodology, relevance scores) and withhold **deliberation** (raw content, chains of thought). Q4 showed the failure of transmitting too little metadata; Q6 shows the failure of transmitting too much reasoning. Same lever, opposite errors.

---

## Score

**5 / 6.** The single miss (Q4) was the *upstream contract vs downstream repair* axis — a heavily tested Domain 5 pattern that recurs in 5.3 (error propagation) and 5.6 (provenance). Q6 re-tested the same lever from the opposite direction and was answered correctly.

---

## Clarifications & cross-links raised this session

- **5.1 "reposition + headers" vs 4.6 / 1.6 "multi-pass splitting."** Both are legitimate responses to long aggregated inputs. Pick by symptom emphasis: *findings present in context but omitted from a synthesis* → reposition + section headers (this topic). *A surface too large for any single pass to hold without attention dilution* → scoped passes plus an integration pass (1.6 / 4.6).
- **Injection vs retrieval.** The recurring deterministic-vs-probabilistic axis from Domain 1.4 / 1.5 shows up here as: critical facts belong in the prompt, not behind a tool the agent must decide to call. An agent that lost a fact does not know it is missing.
- **The hard limit shared with 4.4.** No downstream prompt, retry, or repair stage can recover information the input never contained. In 4.4 the missing information was absent from the source document; in 5.1 it was discarded at a summarization or agent boundary. The fix is always upstream.
- **Forward links.** Structured error context and access-failure-vs-empty-result (5.3) and provenance/multi-source synthesis (5.6) both build directly on §5's output-contract discipline; scratchpad files and `/compact` for extended sessions (5.4) are the large-codebase counterpart to the case-facts layer.
