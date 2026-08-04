# Topic 1.1 — Agentic Loops & `stop_reason`

**Domain:** 1 — Agentic Architecture & Orchestration (27%)
**Topic:** 1 of 32 · First task statement in Domain 1

---

## 1. Core mental model — who runs the agent

- **The Claude API is stateless and single-turn.** It is a pure function: `response = f(messages, tools, system, ...)`.
  - **Stateless** → the server remembers nothing between calls. "Memory" = *you* resending the entire `messages` array every time. Drop it → amnesia.
  - **Single-turn** → one call produces exactly one assistant message, then stops. The API never continues on its own.
- **Claude does not "run" an agent — your *harness* does.** Claude has no hands; it can only *emit a request* (a `tool_use` block). Your code executes tools, holds/grows state, loops, and enforces guardrails.
- **Agent SDK / Claude Code feel stateful** only because they wrap the stateless API with harness code that stores and resends history. The underlying API is still stateless. (Session state/resumption = Topic 1.7.)

### The agentic loop
```
1. Send messages[] + tools[] to the API
2. Inspect stop_reason on the response
3. If stop_reason == "tool_use":
     - append Claude's assistant turn (with the tool_use block) VERBATIM
     - execute the tool(s) yourself
     - append a USER turn containing one tool_result per tool_use (matched by id)
     - loop back to step 1
4. If stop_reason == "end_turn":
     - Claude is done → surface text, exit loop
```

### Division of labor (memorize)
| Action | Who |
|---|---|
| Decide whether/which tool + arguments | **Claude** |
| **Execute** the tool | **Harness** |
| Capture result (incl. errors) | **Harness** |
| Feed result back as `tool_result` | **Harness** |
| Interpret result, decide next step | **Claude** |
| Produce final answer (`end_turn`) | **Claude** |

**One-liner:** *Claude chooses; the harness acts; Claude interprets.*

---

## 2. `stop_reason` — the loop's control signal

| `stop_reason` | Meaning | Loop action |
|---|---|---|
| `end_turn` | Finished naturally | Stop; return text |
| `tool_use` | Wants to call tool(s) | Execute, append `tool_result`, **loop** |
| `max_tokens` | Hit the output-token cap **mid-generation** | **Truncated** — continue or raise cap; NOT complete |
| `stop_sequence` | Custom stop sequence hit | Handle per design |
| `pause_turn` | Long server-side turn paused | Resend response to **resume** the turn |
| `refusal` | Declined for safety | Stop; handle gracefully; don't blind-retry |

- **`end_turn` vs `tool_use`** is the fundamental fork. `tool_use` = "not done, waiting on you."
- **`stop_reason` is `null` while streaming**, populated only when the message completes.

---

## 3. Tool-use turn mechanics

When `stop_reason == "tool_use"`, the assistant `content` array can interleave `text` blocks and one or more `tool_use` blocks (each with unique `id`, `name`, `input`).

**One tool call adds TWO entries to `messages`:**

1. **Assistant turn (the request), appended verbatim** — must be preserved or the next `tool_result` references a dangling id → API error. (The API is stateless; it didn't store its own output — *you* must.)
   ```json
   { "role": "assistant", "content": [
       { "type": "text", "text": "Let me check that order." },
       { "type": "tool_use", "id": "tu_1", "name": "lookup_order", "input": { "order_id": "12345" } } ] }
   ```
2. **User turn carrying the result** — note `role: "user"` (a tool result is *input to Claude*, so it rides in a user turn, NOT an "assistant"/"tool" role):
   ```json
   { "role": "user", "content": [
       { "type": "tool_result", "tool_use_id": "tu_1", "content": "{ \"status\": \"shipped\", \"eta\": \"Aug 5\" }" } ] }
   ```

**Pairing rule:** every `tool_use` must be answered by a `tool_result` with the same `tool_use_id`, in the **immediately following user turn**. Can't skip, reorder by anything but id, defer, or inject unrelated messages between them.

---

## 4. Error results — `is_error`

- A failed tool is **still answered** — as a `tool_result` with `is_error: true` and a **descriptive** message. Never swallow, drop, or crash.
  ```json
  { "type": "tool_result", "tool_use_id": "tu_1",
    "content": "Error: order 12345 not found in the database.", "is_error": true }
  ```
- Feeding the error back = evidence Claude uses to **re-plan**: retry with corrected inputs, switch tools, ask the user, or degrade honestly. Prevents hallucinated success.
- **Error content is a tool-design decision:** actionable ("amount must be positive cents; got -5") enables smart retry; opaque ("Error: 500") leaves Claude guessing.
- **Bound retries** (max attempts) and distinguish **transient** (timeout → retry) vs **permanent** (record missing → don't retry). Optionally the harness auto-retries transient failures before surfacing `is_error`.
- **Anti-pattern:** mapping a failure onto a valid-looking success value (e.g., returning `"[]"` on a 503) = silent failure → confident wrong answers.

---

## 5. Parallel tool use

- Claude can emit **multiple `tool_use` blocks in one assistant turn** for **independent** calls (e.g., weather in Paris AND Tokyo).
- Harness: run them **concurrently**; return **all** `tool_result`s in **one** user turn, matched by `tool_use_id` (id binds them, not array position).
- **Independent → parallel. Dependent → sequential.** If tool B's arguments come from tool A's output, Claude must do A, see the result, then B on the next iteration. Forcing parallelism on dependent calls → guessed/hallucinated inputs (correctness bug).
- Distinct from multi-agent parallelism (Topic 1.2): one Claude issuing several calls vs. a coordinator spawning several Claude instances.

---

## 6. `pause_turn` & server-side tools

- **Client-side tools:** your harness executes → you return a `tool_result`.
- **Server-side tools** (web search/fetch): **Anthropic's infrastructure executes** inside the turn. If it runs long, the turn is **paused** → `stop_reason: "pause_turn"` with progress so far.
- Handling: **send the paused response back** in the next call's `messages` to **resume and finish**. Your harness executes nothing and returns no `tool_result`.
- Contrast: `tool_use` = you *act*; `pause_turn` = you *resume* the server's work; not a truncation, not a refusal.

---

## 7. Streaming & `stop_reason`

- Streaming = incremental SSE events (`message_start`, `content_block_start/delta/stop`, `message_delta`, `message_stop`); lower perceived latency.
- **`stop_reason` is `null` during the stream**, arrives near the end in `message_delta`; stream closes with `message_stop`.
- **Don't branch the loop until the stream completes** — accumulate deltas, reconstruct the final message (tool args arrive as `input_json_delta` fragments — concatenate and parse only when the block is complete), *then* read `stop_reason`.
- Streaming changes *when you learn* `stop_reason` and *how you receive* output — **not** the loop logic. Truncation still surfaces as `max_tokens` in the final `message_delta`.

---

## 8. Anti-patterns / exam traps

- Exiting the loop on "the tool succeeded" instead of on `end_turn`. **Exit only on `end_turn`.**
- Treating `max_tokens` as a completed answer (it's **truncated**).
- Forgetting to re-append the assistant turn → dangling `tool_use_id`.
- Sending `tool_result` with `role: "assistant"` (must be `user`).
- Swallowing tool errors / returning fake-success data (silent failure).
- Forcing parallel tool use on dependent calls.
- Confusing `pause_turn` with `tool_use`/`refusal`/truncation.
- Unbounded loop → **add a max-iteration guard** (canonical fix for runaway cost).
- Believing "the API maintains conversation state" or "Claude executes the tool" (who-does-what boundary).

---

## Practice Questions

### Q1 — Tool-result mechanics
Harness gets `stop_reason: "tool_use"` (`tu_9`, `refund_order`), runs it successfully. Correct next step?

- **A.** Append only a user `tool_result`; skip re-adding the assistant turn.
- **B.** Append the assistant turn (with `tool_use`), then a user turn with `tool_result` (`tool_use_id: tu_9`), then call the API again. ✅
- **C.** Append the assistant turn, then an **assistant** message with the `tool_result`.
- **D.** Tool succeeded → surface `"refunded"` and exit the loop.

**Correct: B.** One tool call = append assistant turn (request) + append **user** turn with matching `tool_result`, then loop. Exit only on `end_turn`.
- **D wrong:** "tool success = done" trap — the result is *input to Claude*; Claude must still interpret it and produce the user-facing reply. Claude decides completion via `end_turn`, not the harness.
- **A wrong:** skipping the assistant turn → `tool_result` references a `tool_use_id` not in history → malformed request. (API is stateless; it didn't store its own output.)
- **C wrong:** `tool_result` must be `role: "user"`, never `assistant`.

### Q2 — `max_tokens` truncation
`max_tokens: 1024`, ~40-field JSON, longest docs → `JSON.parse` throws, JSON cut off mid-string, `stop_reason: "max_tokens"`. Best explanation + fix?

- **A.** Prompting problem; add few-shot valid-JSON examples; stop_reason incidental.
- **B.** Truncated at the token ceiling — detect `max_tokens`, raise cap and/or continue generation instead of parsing partial as final. ✅
- **C.** `max_tokens` means Claude finished; try/catch and accept the partial object.
- **D.** It's a disguised `refusal`; shorten the input document.

**Correct: B.** `max_tokens` = truncation (budget problem), not completion. Raise the cap and/or continue generation.
- **A wrong:** misdiagnoses as quality/prompting; examples can't fix output severed at a hard ceiling — it'd truncate again.
- **C wrong:** the #1 trap — `max_tokens` ≠ `end_turn`; accepting the partial propagates incomplete data (silent failure).
- **D wrong:** `max_tokens` and `refusal` are distinct; fix belongs on the **output** budget, not the input.

### Q3 — Parallel vs sequential
`get_user_profile` (returns home airport) then `search_flights(from_airport, ...)` across two turns. Teammate: "should've been parallel in one turn." Most accurate?

- **A.** Teammate right — any two tools should always be parallel.
- **B.** Teammate wrong — `search_flights` depends on the profile's output, so calls must be **sequential**; parallel only for **independent** calls. ✅
- **C.** Teammate wrong — parallel tool use only works for server-side tools.
- **D.** Teammate right — return both results across two separate user turns, correlate by position.

**Correct: B.** `from_airport` comes from the profile result → data dependency → sequential. Parallel is only for independent calls.
- **A wrong:** over-generalizes; forcing parallelism on dependent calls → guessed/hallucinated inputs (correctness bug).
- **C wrong:** parallel tool use is fully supported for client-side tools.
- **D wrong:** parallel results go in **one** user turn, matched by `tool_use_id`, never by position across turns.

### Q4 — Silent failure / `is_error`
`check_inventory` hits a `503`; harness returns `"[]"`; Claude says "out of stock." Most effective fix?

- **A.** Keep `"[]"`, add a system-prompt hedge for empty inventory.
- **B.** Return the failure as a descriptive `tool_result` with `is_error: true` ("503 ... does not mean out of stock") so Claude retries / re-routes / tells the truth. ✅
- **C.** Throw and terminate the loop on any non-200.
- **D.** Have Claude call the microservice directly to read the 503 itself.

**Correct: B.** `"[]"` is indistinguishable from a real empty result — a silent failure. Report `is_error: true` with descriptive, disambiguating content so Claude adapts.
- **A wrong:** prompt patch over a lying data contract; Claude still can't distinguish outage from genuine empty, and hedging harms real out-of-stock answers.
- **C wrong:** over-correction — a transient 503 is recoverable per-tool; crashing the whole loop is brittle.
- **D wrong:** violates who-does-what — Claude can't execute tools or read HTTP statuses; it only emits requests.

### Q5 — `pause_turn` + runaway cost
Server-side web search → `stop_reason: "pause_turn"` with partial content; separately, agent loops long and costs a lot. Correct pair of actions?

- **A.** `pause_turn` → harness executes the search + appends `tool_result`; cost → lower `max_tokens`.
- **B.** `pause_turn` → resend the paused response so Claude resumes/finishes (harness executes nothing, server-side); cost → add a **max-iteration guard**. ✅
- **C.** `pause_turn` → a safety refusal, apologize; cost → turn off streaming.
- **D.** `pause_turn` → truncation like `max_tokens`, raise cap + resend original query from scratch; cost → strip `tools` on later iterations.

**Correct: B.** `pause_turn` = long server-side turn paused → resend to resume; nothing to execute, no `tool_result`. Runaway cost → bound the loop with a max-iteration guard.
- **A wrong:** misclassifies as `tool_use` (no client tool to run); lower `max_tokens` truncates each response, doesn't cap iterations.
- **C wrong:** `pause_turn` ≠ refusal; streaming is a delivery mechanism unrelated to iteration count.
- **D wrong:** not truncation; re-sending from scratch discards completed searches; stripping `tools` mid-run can strand Claude — the disciplined control is a max-iteration guard.

---

## Clarifications from follow-ups

- **Stateless/single-turn:** API is a pure function; continuity is entirely manufactured by resending `messages`. Agent SDK/Claude Code only *feel* stateful via wrapping harness code.
- **Harness** = the program around the model (loop + tool execution + state + guardrails). Synonyms: orchestrator, agent loop, application code, runtime. Claude = brain; harness = hands + memory.
- **`is_error` decision-steering:** the *content* (not just the flag) drives recovery — descriptive errors enable smart retries; bound retries and separate transient vs permanent failures.
