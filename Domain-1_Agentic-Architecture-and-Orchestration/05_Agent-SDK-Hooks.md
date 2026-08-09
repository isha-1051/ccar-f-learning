# Topic 1.5 — Agent SDK Hooks

**Domain 1 · Agentic Architecture & Orchestration · Topic 5 of 7**

---

## Core idea

**Hooks are deterministic, user-defined callbacks that fire at fixed points in the agent's lifecycle, letting you programmatically observe, modify, or block what the agent does — without relying on the model to choose to comply.**

This is the deterministic end of the enforcement spectrum from Topic 1.4 (prompt guidance = probabilistic vs structural gating = deterministic). Hooks are the SDK's primary mechanism for guarantees. Exam trigger phrase: *"how do you guarantee X always/never happens"* → hooks.

Two defining properties:
- **Deterministic** — runs every time the event fires, regardless of what Claude decided or how it was prompted.
- **Out-of-model** — it's *your* code running in *your* environment with *your* privileges. Claude can't run it and can't skip it. (Remember from 1.1: the *harness* owns the loop; hooks are how you inject logic into that harness loop.)

---

## Lifecycle events (know cold)

| Event | Fires when… | Can block? | Classic use |
|---|---|---|---|
| **PreToolUse** | *Before* a tool call executes | **Yes** — deny/allow/ask | Guardrails: block `rm -rf`, block writes to `prod/`, deny `curl` to external hosts |
| **PostToolUse** | *After* a tool returns | Yes (feedback/block) | Auto-format after Edit, run tests/lint, validate output |
| **UserPromptSubmit** | User submits a prompt, before Claude sees it | Yes | Inject context, redact secrets, block disallowed requests |
| **Stop** | *Main* agent about to finish its turn | Yes — force continue | Enforce "not done until tests pass" |
| **SubagentStop** | A *subagent* finishes (before result returns to coordinator) | Yes | Validate a subagent's result against its output contract before it's accepted |
| **PreCompact** | Before context compaction | No | Preserve key info before it's summarized (ties to Domain 5) |
| **SessionStart** | Session starts/resumes | No (adds context) | Load project state, inject standards |
| **SessionEnd / Notification** | Session ends / notification sent | No | Logging, alerting, cleanup |

Highest-yield events: **PreToolUse** (prevention) and **PostToolUse** (reaction/feedback).

---

## How a hook communicates its decision

### A) Exit codes (simple shell hooks)
- `exit 0` → success / allow / proceed.
- `exit 2` → **blocking error**; `stderr` is fed back to Claude so it can self-correct.
- any other non-zero → non-blocking error (logged, execution continues).

### B) Structured JSON on stdout (advanced control)

Input arrives on **stdin** as JSON (`session_id`, `transcript_path`, `cwd`, `hook_event_name`, plus event-specific fields like `tool_name`, `tool_input`, `prompt`).

**Common output fields (any hook):**
```json
{
  "continue": true,            // false = HARD STOP the whole agent (global kill switch)
  "stopReason": "string",      // shown when continue=false
  "suppressOutput": true,
  "systemMessage": "string"
}
```

**PreToolUse** — nested `hookSpecificOutput`:
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",              // "allow" | "deny" | "ask"
    "permissionDecisionReason": "Writes to prod/ are forbidden"
  }
}
```
- `allow` → bypass the normal permission prompt, tool runs.
- `deny` → block the call; `reason` fed back to Claude.
- `ask` → defer to the human for confirmation.

**PostToolUse:**
```json
{ "decision": "block", "reason": "Lint failed: unused import on line 42" }
```
`block` doesn't undo the tool (it already ran) — it injects the reason as feedback so Claude self-corrects.

**UserPromptSubmit:** can inject `hookSpecificOutput.additionalContext` (its plain stdout is also injected as context); can `decision: "block"` to reject a prompt.

**Stop / SubagentStop:** `decision: "block"` **forces the agent to keep working** instead of ending.

---

## ⚠️ Vocabulary/timing split (biggest exam trip-wire)

- **`PreToolUse` → `permissionDecision`: `allow` | `deny` | `ask`** — governs whether a call runs (before it runs).
- **`PostToolUse` / `Stop` / `SubagentStop` → `decision`: `block`** — feeds reason back / forces continuation (after the fact).
- **`continue: false`** — top-level GLOBAL kill switch, independent of both.

Mixing `deny` (PreToolUse) with `block` (Post/Stop), or `Stop` (main) with `SubagentStop` (child), is a deliberate distractor.

---

## Matchers in `settings.json` (Claude Code shell hooks)

```json
{
  "hooks": {
    "PreToolUse": [
      { "matcher": "Write|Edit",
        "hooks": [ { "type": "command", "command": "$CLAUDE_PROJECT_DIR/.claude/guard.sh" } ] }
    ],
    "PostToolUse": [
      { "matcher": "Edit",
        "hooks": [ { "type": "command", "command": "prettier --write \"$file\"" } ] }
    ]
  }
}
```
- **`matcher` filters by tool name** (regex/string): `"Write"`, `"Write|Edit"`, `"Bash"`, `"mcp__.*"`, `"*"`/empty = every tool.
- Events without a tool concept (`UserPromptSubmit`, `SessionStart`, `Stop`, `PreCompact`) **don't use a matcher**.
- Multiple matchers/hooks can fire per call; **any one returning block/deny blocks the call** (most-restrictive wins).
- **Hierarchy (additive, not override):** enterprise → user (`~/.claude/settings.json`) → project (`.claude/settings.json`) → local (`.claude/settings.local.json`). Ties to Domain 3's CLAUDE.md hierarchy.
- Command reads event JSON from stdin; parse `tool_input` (e.g., with `jq`) for argument-aware decisions.

---

## Two authoring surfaces: shell command vs SDK callback

Same lifecycle, same decision schema — **the difference is authoring surface and access to state, not capability.**

- **Shell-command hook (`settings.json`)** — harness spawns a process, pipes event JSON to stdin, reads stdout/exit code. Language-agnostic, config-driven, team-shareable via checked-in config. Best for stateless ops policy, formatters, guardrails. No access to app state.
- **SDK callback (Python/TS)** — in-process function passed at agent construction (e.g., `HookMatcher(matcher="Write", hooks=[fn])`). Best when the decision needs **application state/services** (live user role, DB, in-memory config) or you're *building* an agent product.

**Decision rule:** need live app/service state → SDK callback. Stateless, portable, shareable → shell hook. Don't over-engineer a stateless formatter into an in-process callback; don't cripple a state-dependent check by forcing it into a shell process.

---

## How hooks relate to other Domain-1 concepts

- **vs prompt instructions** — prose is probabilistic and decays over long sessions; a hook is a guarantee. Hard requirement → hook.
- **vs tool omission (1.3)** — omission is coarse (all-or-nothing). A `PreToolUse` hook inspects **arguments**, so it can allow `Bash` in general but deny `git push --force`. Fine-grained least privilege.
- **vs `tool_choice` (1.4)** — orthogonal. `tool_choice` = model-side selection among available tools; a hook = harness-side gating of whether an attempted call runs.
- **Self-correction loop** — `PostToolUse` + `exit 2`/`decision:block` feeds errors back → automatic verify → feedback → fix (previews Domain 4 validation/retry).
- **Subagents** — hooks apply *inside* subagent loops too (govern subagents' tool calls, argument-level least privilege). `SubagentStop` enforces the output contract that 1.2/1.3 could only *describe*. Context isolation still holds — coordinator still receives only the subagent's final string.

---

## Anti-patterns

- **Prompting for a guarantee** — using CLAUDE.md prose for something that must always/never happen.
- **Putting judgment into a hook** — hooks encode deterministic *rules*, not reasoning/opinions.
- **Over-broad matchers** — too broad stalls legitimate work; too narrow leaks. Match precisely.
- **Feedback-less blocks** — blocking with no useful `reason`/`stderr` leaves Claude unable to self-correct → blind loop.
- **Security blind spot** — hooks run your shell with your credentials automatically; interpolating untrusted `tool_input` into a shell command is an injection risk. Treat hook code as privileged.

---

## Practice questions

### Q1 — Guarantee no writes to `/infra/production/` (incl. subagents), no model dependence
A. Strengthen the CLAUDE.md instruction (bold/caps/repeat).
B. **`PreToolUse` hook on `Write|Edit` that inspects `tool_input.file_path` and returns `permissionDecision: "deny"` for `/infra/production/`.** ✅
C. Remove Write/Edit tools entirely.
D. `PostToolUse` hook that reverts the change after the edit.

**Correct: B.** PreToolUse deny fires *before* the write → the write never happens (hard guarantee); argument-aware so only the scoped path is blocked; applies to subagents; no model cooperation needed.
- **A** — the exact anti-pattern shown failing; prompt guidance is probabilistic and decays.
- **C** — deterministic but far too coarse; breaks all legitimate editing to protect one directory (tool omission = all-or-nothing).
- **D** — the file is already modified on disk by the time PostToolUse fires; relies on Claude to revert (reintroduces model dependence) and leaves a bad-state window. "Impossible" means *prevent*, not *detect-and-undo*.

### Q2 — Run `ruff` on every `.py` write and auto-feed errors back to fix (no human, no app state)
A. `PreToolUse` deny if proposed contents wouldn't pass lint.
B. **`PostToolUse` hook on `Write|Edit` running `ruff`; on failure return `decision: "block"` with lint output as `reason`.** ✅
C. System-prompt instruction to run ruff via Bash.
D. `Stop` hook that lints the whole project at the end.

**Correct: B.** PostToolUse fires after each write (satisfies "every time"), file is on disk, and `block`+reason feeds errors straight back → canonical verify→feedback→fix loop.
- **A** — PreToolUse blocks the write, so code never lands on disk and can't be iteratively fixed; you *want* the write then validate.
- **C** — fails the "every time" guarantee; probabilistic.
- **D** — runs only at finish, bulk-lints the whole project, loses the tight per-file loop; Stop hooks suit a final gate (all tests pass), not per-edit linting.

### Q3 — Map three `PreToolUse` outcomes to `permissionDecision` values
Safe read-only → run without prompt; destructive → block+explain; else → ask human.
A. **`allow`, `deny`, `ask`** ✅
B. `ask`, `deny`, `allow`
C. `allow`, `block`, `ask`
D. `continue`, `deny`, `ask`

**Correct: A.** PreToolUse permission vocabulary is exactly `allow`/`deny`/`ask`.
- **C** *(common miss)* — `block` is **not** a PreToolUse `permissionDecision`; it's `PostToolUse`/`Stop`'s `decision` value. The PreToolUse block-the-call keyword is `deny`.
- **B** — right values, inverted mapping (would prompt for safe commands, silently allow the unclassified bucket).
- **D** — `continue` isn't a permissionDecision; it's a top-level boolean global kill switch.

### Q4 — Deterministically reject a non-conforming subagent result before the coordinator receives it
A. `Stop` hook validating the JSON.
B. **`SubagentStop` hook validating against the contract; `decision: "block"` with schema errors on failure.** ✅
C. `PostToolUse` on the coordinator's synthesis tool.
D. Strengthen each subagent's prompt with a few-shot JSON example.

**Correct: B.** SubagentStop fires when the subagent finishes, *before* its string returns to the coordinator — exactly the interception point; `block` forces the subagent to self-correct/retry; deterministic.
- **A** *(common miss)* — `Stop` = *main/coordinator* agent's turn ending, not a subagent; wrong event/timing.
- **C** — too late; by synthesis time the malformed result was already returned up and absorbed; also forces the *coordinator* to redo, not the subagent to retry.
- **D** — improves conformance but stays probabilistic; the problem is "compliance isn't reliable." Best real answer = few-shot *plus* SubagentStop.

### Q5 (bonus) — Shell hook vs SDK callback pairing
Scenario 1: PreToolUse guardrail that calls an internal auth microservice using live in-memory user role + DB. Scenario 2: run `gofmt` on every `.go` file after edit (stateless).
A. Shell + shell.
B. **SDK callback (Scenario 1) + shell hook (Scenario 2).** ✅
C. Shell + SDK callback.
D. SDK + SDK.

**Correct: B.** State-dependent decision → in-process SDK callback (direct access to live app state/services). Stateless formatter → portable, team-shareable `settings.json` shell hook.
- **A** — reaching live in-memory state from a spawned shell needs awkward/insecure out-of-band channels.
- **C** — inverted; couples trivial formatting to app code and handles the hard state case with the weaker tool.
- **D** — over-engineers the stateless formatter; loses the shareable-config benefit for no gain.

**Meta-takeaway:** same lifecycle event + same decision schema — shell vs SDK is about **access to application state**, not capability.

---

## Score & focus areas
- **Session score: 5/6.** Both misses (Q3, Q4) were **vocabulary/timing recall**, not concepts.
- **Pre-exam drill:** memorize the `deny` (PreToolUse) vs `block` (Post/Stop) split and the `Stop` (main) vs `SubagentStop` (child) distinction.
