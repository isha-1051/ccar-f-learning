# Topic 2.2 — Structured MCP Error Responses

**Domain 2 – Tool Design & MCP Integration (18%) · Topic 2 of 5 in this domain · Topic 9 of 30 overall**

---

## 1. Core Mental Model

**An error response is prompt engineering for a model reader** (the mirror image of topic 2.1's rule for tool definitions). When a tool call fails, the error text becomes part of Claude's context on the next turn. The design question behind everything in this topic:

> *"If Claude reads this error, does it know what to do next?"*

A good error turns a failure into one corrective turn. A bad error produces blind retry loops, hallucinated workarounds, or silently wrong answers.

## 2. The Two Error Channels (most-tested distinction)

| | Channel A — Protocol-level (JSON-RPC error) | Channel B — Tool-execution error (`isError: true`) |
|---|---|---|
| What failed | The *machinery*: unknown method, malformed request, server crash, transport broken | The *task*: not found, invalid argument value, permission, state conflict |
| Wire shape | JSON-RPC `error` object (`code`, `message`) — **no `result` key** | A **successful** JSON-RPC response; `isError: true` sits inside `result` next to the normal `content` array |
| Who consumes it | Client/harness (developer-facing) | The model — content lands in Claude's context |
| Standard codes | `-32602` invalid params, `-32601` method not found, `-32603` internal error | No error schema — structure is a convention you impose on the text |

**Decision boundary:** *Can the model plausibly recover by changing its next action?* Yes → in-band `isError: true` result. No → protocol/harness level (looping the model on unrecoverable errors burns turns and context).

**Classic anti-pattern:** throwing a protocol-level exception for a business-logic failure ("customer not found"). An unhandled exception in a tool handler typically becomes a `-32603` internal error — the business-logic failure silently downgrades itself into the invisible channel. "Not found" is *information*, not a malfunction; it belongs in the result.

### `isError` mechanics
- `isError` defaults to false/absent; a success just omits it. Returning error text without the flag loses the explicit failure signal harnesses and logging rely on — set the flag.
- `content` is the same content array as success (text blocks etc.). The what/why/what-next structure is convention, not protocol.
- Handler rule of thumb: **catch everything task-related and convert to `isError: true`**; let only genuine infrastructure/protocol faults propagate as JSON-RPC errors.

### How harnesses (e.g., Claude Code) surface the two channels
- **Channel B (smooth path):** harness builds a `tool_result` block with `is_error: true` and your content **verbatim**, attached to the failing call via `tool_use_id`. You control every word the model reads; the loop continues and the next tokens can be a corrected retry.
- **Channel A (lossy path):** no `result.content` to forward, so the harness synthesizes a generic error into the `tool_result` slot (the API contract requires some `tool_result` for every `tool_use`), logs to developer-facing diagnostics (`/mcp`, debug logs), and for connection-level failures the tools may simply **disappear from the tool list** — the model gets *silence*, not an error, and may hallucinate around the absence.
- Parallel: harness-native tools (e.g., Bash) already work in-band — stderr + exit code come back as the tool result and Claude self-corrects. MCP's `isError` gives custom tools the same recovery loop.

## 3. Anatomy of a Good Error

1. **What failed** — specific: `"Invalid value for 'date_range': '2026-13-01'"` not `"Bad request"`.
2. **Why** — the violated constraint: `"Month must be 01–12. Expected format: YYYY-MM-DD."`
3. **What to do next** — corrective action with valid values or a working example.

High-value ingredients: enumerate valid options when the input space is small; distinguish "0 results" (not an error — suggest broadening) from real failure; actionable transient signals (`"Retry after 30 seconds"`); consistent shape across all of a server's tools.

## 4. Anti-Patterns (each is a plausible exam distractor)

- **Raw stack traces** — huge context cost, leak internals (paths, SQL, credentials), bury the actionable line.
- **Opaque codes alone** (`"Error 422"`) — no mapping to a corrective action. Codes are fine alongside explanation, not instead.
- **Silent failure / empty success** — returning `""` or `{"results": []}` when the call broke; the model treats garbage as ground truth. Worst case.
- **Human-support messages** (`"Contact your administrator"`) — useless to a model needing a next action.
- **Letting the model loop on unrecoverable errors** — say explicitly "not retryable" or escalate out of band.
- **Leaking sensitive internals** — error text goes into model context and transcripts/logs.

## 5. Recoverable vs. Unrecoverable Classification

Test: fixable **by the model, from inside the loop, this session** — not "fixable in principle."

**Recoverable → in-band with corrective guidance:** malformed argument (give format + example); unknown enum value (enumerate valid); query too broad (suggest filters/pagination); missing required field; resource not found (suggest search tool by name); precondition not met (state required state and how to reach it); ambiguous match (list candidates — recovery may be asking the user, which still counts).

**Unrecoverable → say "stop" explicitly or surface to harness:** expired credentials ("not retryable; user must re-authenticate"); missing permission (state plainly so the agent reports, doesn't route around); backend down (harness retry/backoff or clear "do not retry now"); server misconfiguration (ops problem); hard quota exhausted (state reset time).

**Subtlety:** classification can depend on what the error *says*, not what the failure *is*. "Permission denied" alone is a dead end; "Permission denied: your role can read tickets but not billing — use get_billing_summary" converts it into a recoverable pivot. Architect's job: move failures from the right column to the left.

## 6. The Middle Band — Recoverable, But Not by the Model

Unified rule: **who owns the missing ingredient?** Time/infrastructure → harness/server handles silently (retry, backoff). Knowledge or authority → user; the error's job is to package the escalation. The model only gets involved when there's a genuine decision that is *its* to make — and then the error must carry the facts that decision needs.

- **Rate limit with retry-after:** best = server/harness retries transparently with backoff (short waits). If model-visible (long waits), include the wait time plus a completed/pending inventory so the model has a real decision (proceed partial vs. wait vs. tell user). Exam angle: fix parallel fan-out rate limiting with throttling/backoff in infrastructure, never a prompt instruction to "space out calls."
- **Transient network blip:** server wraps upstream in 2–3 jittered retries; the model never sees it — correctly, since there's no decision for it to make. Intermittent errors are noise and models fit narratives to noise ("the deploy system is down"). If retries exhaust, the failure has *become* terminal — reclassify and surface explicitly ("unreachable after 3 attempts; report, don't retry").
- **Ambiguity only the user can resolve:** the error *is* the clarifying question, pre-drafted — list candidates with distinguishing fields and name the disambiguating follow-up call. Never silently pick the first match (wrong-account/privacy risk).
- **Destructive confirmation:** the tool doesn't trust the model's authority — first call returns `isError: true` with a ground-truth consequence summary and requires `confirm=true` after user approval. Structural human-in-the-loop, far stronger than a prompt rule.

## 7. Interaction with Input-Schema Validation — the Three Gates

1. **Gate 1 — the schema (prevention, zero turns):** enums, formats, patterns, required fields, examples in descriptions — read by the model *before* it calls. The cheapest error is the one that never happens.
2. **Gate 2 — pre-execution validation (cheap):** schema validation plus semantic checks the schema can't express (date ranges, cross-field constraints). No backend call, no side effects, maximally precise errors. If it surfaces as a bare "Invalid params", catch validation yourself and return `isError: true` with specifics.
3. **Gate 3 — execution errors (expensive):** only failures that genuinely require running the tool — not-found, permission, state conflicts, backend faults.

**Feedback rule (exam favorite):** *a recurring execution error is telling you a constraint belongs at an earlier gate.* Repeated invalid enum values → `enum` in the schema. Repeated format mistakes → `pattern` + example. Repeated unbounded queries → make the range required. Division of labor: schema for statically expressible constraints; runtime errors for what only execution can discover (you can't enumerate valid ticket IDs).

## 8. Tradeoffs & Judgment

- **Detail vs. context cost:** errors are tokens; include what changes the next action, summarize the rest ("…and 12 more errors of the same type").
- **Recoverable → guide; unrecoverable → stop.** Best-first-step questions hinge on this split.
- **Chronic failures fix upstream:** better description/schema (2.1), not ever-fancier error prose.
- **Retry placement:** transient faults → harness/server with backoff, invisible when possible; semantic faults → model, guided by the error. Don't make the model babysit flakiness.
- **Multi-agent amplifier:** a subagent returns only its string, so its misreading of a blip becomes an unchallengeable "fact" in the coordinator's synthesis — error-handling flaws propagate upward (foreshadows 5.3).

---

## Practice Questions

### Q1 — Unhandled exception for a business-logic failure *(answered correctly)*

Support agent calls `get_ticket` with mistyped IDs; the server raises an unhandled `TicketNotFoundError` → JSON-RPC `-32603`. The agent blindly retries or tells customers "the system is down." Most effective change?

- **A.** System-prompt instruction: don't assume the system is down; ask the customer to re-check the ID.
- **B.** ✅ Catch the exception and return `isError: true` with: no ticket matched, the expected ID format, and `search_tickets(email=...)` as an alternative.
- **C.** Harness auto-retry on `-32603` with exponential backoff.
- **D.** Add a regex `pattern` for `ticket_id` to the `inputSchema`.

**Why B:** the root cause is a channel error — "not found" is task-level information stuck in the invisible protocol channel. Moving it in-band gives the model what failed / why / what next, fixing both blind retries and the false outage report.
**Why not A:** the model still can't distinguish "bad ID" from a real outage — prompt patches can't compensate for missing signal. **Why not C:** "not found" is deterministic; backoff adds latency and ends in the same dead end — misclassifies a semantic failure as infrastructure. **Why not D:** tempting Gate-1 answer, but a *well-formed nonexistent* ID sails through the schema into the same unhandled exception; schemas can't enumerate which IDs exist. Complementary hardening, not the fix.

### Q2 — Intermittent upstream 503s in parallel fan-out *(answered correctly)*

Research subagents calling `search_journals` hit transient 503s (resolve in seconds). Server passes them through as `isError: true, "Upstream error: 503"`. Subagents waste turns retrying; some report "the journal database is unavailable," poisoning synthesis. Best design change?

- **A.** Better error text: "Temporary; wait a moment and retry the identical call."
- **B.** ✅ Automatic jittered exponential backoff **inside the MCP server** for 503s so short transients are invisible; if retries exhaust, return `isError: true` saying the service is unreachable and should be reported, not retried.
- **C.** Prompt instruction in each subagent: retry up to 3 times before concluding it's down.
- **D.** Return the 503 as a JSON-RPC protocol-level error so the harness handles it.

**Why B:** a short 503 presents the model with no genuine decision — recovery is owned by *time*, so it belongs in infrastructure, silently. B also reclassifies correctly when retries exhaust (transient → terminal, explicit "report don't retry"). Multi-agent amplifier makes fixing at the source critical.
**Why not A:** well-written coaching for a job the model shouldn't have; every blip still costs a turn, and "usually resolves" still yields occasional false outage reports. **Why not C:** everything wrong with A, plus prompt-based retry counting is unreliable and must be duplicated/maintained across subagent prompts. **Why not D:** moves the failure into the *invisible* channel; the harness can't fix upstream semantics and the model gets a dead end — recovery becomes less likely.

### Q3 — Recurring invalid enum values *(answered correctly)*

`query_orders` has `status` typed as plain `string`; 15% of calls fail with near-miss values (`"complete"`, `"shipped_out"`). The `isError` message lists the four valid values and the model always self-corrects next turn. Team calls it handled. Best recommendation?

- **A.** Agree — 100% recovery shows the feedback loop works as designed.
- **B.** ✅ Change `status` to a JSON Schema `enum` of the four valid values; keep runtime errors for conditions the schema cannot express.
- **C.** Put the four valid values in the agent's system prompt, keep the error as fallback.
- **D.** Accept near-miss values and fuzzy-map them server-side to the closest valid status.

**Why B:** three-gates feedback rule — a recurring execution error means the constraint belongs at an earlier gate. A small static set is exactly what `enum` is for: the model reads the legal values before generating the call and the failure class disappears. Recovery isn't free: 15% of calls double their turn count (latency, tokens, log noise).
**Why not A:** confuses *recoverable* with *well-designed* — the feedback loop is the safety net, not the mechanism of first resort. **Why not C:** tool-contract knowledge in the wrong place: duplicated, drift-prone, consumes system-prompt space on every request, and softer than schema enforcement (the tool definition is the canonical home — 2.1). **Why not D:** silent intent-guessing; a wrong mapping produces confidently wrong query results — the silent-failure anti-pattern, strictly worse than a visible error, and the model never learns.

**Discrimination note:** in Q1 the schema answer was the trap (failure not schema-expressible: well-formed but nonexistent ID); in Q3 it's correct (fixed enum is schema-expressible).

### Q4 — Enforcing human approval for destructive actions *(multiple response, select TWO — answered correctly)*

Ops agent deleted a shared staging environment on its own (incorrect) inference that it was orphaned; the call succeeded first try with no human involved. Which TWO changes together best enforce informed human approval?

- **A.** System prompt: "Never call `delete_environment` without asking the user and receiving a yes."
- **B.** ✅ Structural two-step: a call without `confirm=true` never deletes — returns `isError: true` with a ground-truth consequence summary (age, resource count, teams with recent activity) and requires user approval before calling with `confirm=true`.
- **C.** ✅ `PreToolUse` hook / harness permission rule that intercepts `delete_environment` and requires interactive human approval before execution.
- **D.** Return `isError: true` with a warning on the first attempt and execute automatically if the same call is repeated, treating the retry as confirmation.

**Why B + C:** both close the authority gap structurally, at different layers — defense in depth. B makes single-decision destruction mechanically impossible and supplies the *informed* half of informed consent from ground truth (the incident was an inference error; B replaces inference with facts). C enforces approval deterministically at the harness layer (hooks always run — topic 1.5) regardless of prompt, model intent, or server bugs; the layers fail independently.
**Why not A:** prompt-based enforcement for irreversible actions is the canonical anti-pattern — probabilistic, degrades in long sessions, and a model convinced it's right may reason that asking is unnecessary. **Why not D:** best-disguised trap — looks like B, but the confirmation signal (retrying) is one **the model can produce by itself**; retrying after errors is exactly what agents do, so the gate is a one-turn speed bump to deletion with zero humans involved. A confirmation only works if the confirming token *cannot originate from the agent*.

---

## Clarifications from Follow-ups

1. **`isError` is a flag inside a successful JSON-RPC `result`** — the RPC worked, the task failed. A protocol error has no `result` key at all. There is no protocol-level error schema; what/why/what-next is a convention you impose on the content text.
2. **Unhandled handler exceptions typically become `-32603`** — i.e., business-logic failures silently downgrade into the invisible channel. Catch task-related exceptions and convert to `isError: true`.
3. **Channel A degrades from terse to invisible in harnesses:** generic synthesized `tool_result` text at best; at worst (connection failures) the tools vanish from the tool list and the model gets silence.
4. **Middle-band litmus test:** does the failure present the model with a *genuine decision*? No decision (blip, short rate limit) → handle in infrastructure, invisibly. User-owned decision (ambiguity, destructive action) → the error packages the escalation. Model-owned decision (proceed-partial vs. wait) → give the facts the decision needs (wait time, completed/pending inventory).
5. **Errors can change a failure's class:** adding a pivot ("use get_billing_summary instead") turns an unrecoverable dead end into a recoverable redirect.
6. **Retry-as-confirmation is not confirmation** — any gate the agent can satisfy alone closes no authority gap.

**Score: 4/4**
