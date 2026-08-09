# Topic 1.7 — Session State, Resumption & Forking

**Domain 1 · Agentic Architecture & Orchestration · Topic 7 of 7**

---

## Core summary

### 1. Foundational fact (from 1.1): the API is stateless
Each `messages.create` call is a pure function of the `messages` array you send. The model holds **no** memory between calls; there is **no** server-side conversation object on Anthropic's side. Therefore "session state," "resumption," and "forking" are all **harness/client-side** concepts operating on a stored `messages` array — never on anything the model retains. If a question implies Claude "recalls" a prior session on its own, that's wrong.

### 2. What is actually in "session state" (layers)
| Layer | What | Persisted where |
|---|---|---|
| **Message history** | Full `messages` array: user/assistant turns, `tool_use`, `tool_result` blocks | Harness store (file/DB/transcript) |
| **System prompt / config** | System prompt, model, temperature, tool definitions | Harness config |
| **External side effects** | Files written, DB rows, git commits, emails, API calls | The **real world** — NOT in the array |
| **In-flight harness state** | Loop counters, workflow step, subagent handles | Harness memory (ephemeral unless serialized) |

**Critical distinction:** message history is *replayable text*; external side effects are **not**. Restoring a conversation does not restore or re-sync the world.

### 3. Resumption — continue the same timeline later
Reload a persisted session and keep appending to **the same** array (one head).
- Harness loads stored array + system prompt + tools → appends new user turn → calls API with full reconstructed context → loop continues.
- Free from the model's perspective (it can't tell a resumed session from a continuous one). Works **only** because the harness faithfully persisted and replayed the array.
- **Claude Code:** `--continue` / `--resume` reload the on-disk transcript and extend the same session ID.
- Resumption is **NOT** re-execution: prior tool calls replay as *text history*; tools are not re-run and the world state they assumed may be stale.
- Durability is the **harness's** job; a crash without persistence loses the session — nothing on Anthropic's side to recover.

### 4. Forking — branch a session into an independent timeline
Copy the `messages` array up to a fork point into a **new, independent** session that evolves separately (two heads sharing a prefix). Appending to a fork does not affect the parent, and vice versa.

**Why fork:**
- Explore alternatives without contaminating the mainline.
- Preserve a known-good checkpoint (fork *before* a risky step, fall back to the clean parent).
- Run parallel independent variations from a shared, expensive-to-build context.

**Fork vs. resume:** resume = same timeline continued later (one head); fork = new timeline branched from a snapshot (two heads sharing a prefix).

**Fork vs. subagent (key boundary):**
- **Subagent (1.2/1.3):** fresh **isolated** context, inherits only the *brief*, returns **only a string**; intermediate steps never pollute the coordinator.
- **Fork:** copies the **entire conversation history**; branch continues **as the same agent** with full shared context.
- Rule: need the accumulated conversation to *continue* → **fork**; need an isolated worker that reports back a compact string → **subagent**.

### 5. Session lineage is a DAG
Linear session = chain; forks = branch points. Branches **do not auto-merge** — combining insights from two branches requires a coordinator/synthesizer to read both and reconcile (same fan-in / shared-output-contract discipline as 1.6). "Branches auto-merge back" is wrong.

### 6. Anti-patterns & decision boundaries (judgment the exam rewards)
- **A. Conversation state ≠ world state (resumption staleness).** Resuming restores the transcript, not the current world. Files/DB may have drifted. Well-built agents **re-observe** (re-read files, re-check state) on resume rather than trusting stale history.
- **B. Forking copies conversation, not the external world.** Two branches that both `git commit`/send email produce **two real commits / two real emails**. Discarding a branch's transcript does **not** undo its side effects. For irreversible external actions, back forks with **real isolation** (git worktrees/branches, sandboxes, dry-run/deferred actions) — a forked conversation is not an undo button.
- **C. Don't use one long linear session where you needed a fork.** Trying A then B then C in one timeline bloats context, confuses the model, inflates cost. Correct: fork from a clean checkpoint per attempt.
- **D. Don't confuse fork and subagent.** Fork = keep full shared history, continue as same agent. Subagent = isolated fresh context returning a string. Wrong choice wastes context or loses needed context.
- **E. Durability is the harness's responsibility.** Anthropic stores nothing between calls; persist the transcript yourself.
- **F. Resumption is not re-execution.** Reloading replays text; it does not re-run tools.

---

## Claude Code specifics

**Where state lives:** each session is a **transcript file on disk (JSONL)**, namespaced **per project directory** (`~/.claude/projects/<hashed-cwd>/`), each with a **session ID**. That file *is* the persisted `messages` array — concrete proof that durability is the harness's job.

**The three operations:**
- `claude --continue` (`-c`) → **resumption, automatic:** reloads the **most recent** session *for the current directory*, no picker; same session ID, one linear timeline extended. Directory-scoped (won't find sessions from another folder).
- `claude --resume` (`-r`) → **resumption, chosen:** interactive picker of past sessions (or pass an explicit ID); still extends the chosen timeline.
- `--fork-session` (with `--resume`/`--continue`) → **forking:** loads the old transcript as a starting prefix but **writes to a NEW session ID**, leaving the original intact as a clean checkpoint and starting an independent branch.

> Exam note: the cert tests the **behavioral distinction**, not flag spelling. `--continue`/`--resume` extend one timeline; forking branches to a new timeline preserving the parent. "Keep the original conversation intact while trying a variation" = fork.

**Staleness in CC terms:** `--resume` a session from yesterday → transcript "remembers" old file contents even if a teammate rewrote the file overnight. Correct move on resume: **re-read the file**, don't trust the remembered version.

---

## Forking / resumption × prompt caching & cost

**Three facts about prompt caching:**
1. **What's cached:** a **prefix** of the request (system prompt + tool defs + leading messages up to a cache breakpoint). Exact-match prefix → **cache hit**.
2. **Economics:** cache **write** ≈ 1.25× normal input tokens; cache **read** ≈ **0.1× (≈90% off)** and faster (less recompute).
3. **Content-addressed with a TTL:** cache key is the **prefix content** (a hash), NOT the session ID. Default TTL ≈ **5 min** (rolling, refreshed on hit); optional **1-hour** extended. Same org/account.

**Resumption & cost:**
- Stateless API re-sends the **entire** growing array every turn; cost scales with history length.
- **Within TTL:** stable prefix is byte-identical to the last call → cache hit → pay mostly for new tokens only.
- **After a long gap:** cache **expired** → first resumed call must **re-write** the whole prefix (≈1.25× premium) and recompute → that turn is **slower and pricier**; subsequent turns re-amortize.

**Forking & cost (key insight):**
- Two branches share a common prefix. Because the cache is **keyed on prefix content, not session ID**, both forks **read from the same cached prefix** (≈0.1×). The expensive shared context is paid for **once**; only each branch's *divergent* continuation is billed at full rate (then cached separately per branch).
- This is why "fork per alternative from a clean checkpoint" beats "A then B then C in one timeline": cleaner context **and** cheap shared-prefix reuse.
- **Caveats:** (1) Forking does **not** reduce the token *count* sent — each branch still transmits full copied history each turn; caching reduces the *price per token* of the shared prefix and the latency. (2) TTL still applies — cross-branch savings materialize only if forks run **close together in time**; fork now / run a branch next week and the shared prefix has aged out, so that branch pays to re-write it. Practical guidance: fork and explore branches **promptly**.

---

## Practice questions

### Q1 — Resumption staleness (conversation state ≠ world state)
A dev ran a long Claude Code session yesterday ending with the model having read `payments/gateway.py` and proposing a refactor. Overnight a teammate merged changes that rewrote large parts of that file. This morning the dev runs `claude --continue` and says "apply the refactor we discussed." Best understanding of what happens / the risk?

- **A.** API preserves session state on Anthropic's servers, so Claude has the up-to-date file and applies safely.
- **B.** `--continue` reloads the transcript, so Claude's context reflects **yesterday's** file contents; unless it re-reads the file now, it may apply the refactor against a stale mental model of code that no longer exists as remembered. ✅
- **C.** `--continue` auto-re-executes every prior tool call, so the file is re-read and no risk.
- **D.** Resuming starts a brand-new isolated context that forgot the file, so Claude refuses until re-pasted.

**Correct: B.** Resumption reloads the frozen transcript (yesterday's `tool_result` file contents); it does not re-sync with the world, so the memory and the actual file have diverged. Correct move is to **re-observe** (re-read) before acting.
- **A** wrong: API is stateless; Anthropic stores nothing, and the transcript holds *yesterday's* file, not today's.
- **C** wrong: resumption replays tool calls as *text history*, it does not re-run tools (§6F).
- **D** wrong: `--continue` restores the **full** prior context (including stale file), it doesn't start a forgetful fresh context. The problem is over-confident memory, not amnesia.

### Q2 — Fork-from-checkpoint (isolation + preserved parent + cost)
An agent has ~30 turns of expensive base context and must try **three independent fix strategies** from that identical start: (1) no cross-pollution, (2) keep the 30-turn context as a clean fallback, (3) minimize cost given the large prefix. Most effective approach?

- **A.** One linear session: try strategy 1, then 2, then 3 in the same timeline so the model learns from each failure.
- **B.** Three subagents, each handed the full 30-turn transcript as its brief, each returns a completed fix.
- **C.** **Fork** at the clean 30-turn checkpoint into three branches, run them close together in time, synthesize/compare, relying on the shared cached prefix for cost. ✅
- **D.** Three completely fresh sessions, re-loading repo context independently in each, to guarantee isolation.

**Correct: C.** Fork copies the array into three independent timelines (isolation, req 1), preserves the parent as fallback (req 2), and — because caching is content-addressed — all three read the shared prefix at ≈0.1× if run within TTL (req 3). Branches don't auto-merge, so compare/reconcile is real work.
- **A** wrong: the §6C anti-pattern — later strategies pile on earlier ones' context (pollution + re-billing) and destroy the clean checkpoint.
- **B** wrong: subagents are isolated, brief-only, string-returning workers; they don't "continue as the same agent with full shared history." Wrong mechanism for a *branch* of the conversation.
- **D** wrong: rebuilds the 30 turns three times — most expensive, no shared-prefix reuse, no preserved parent.

### Q3 — Forking copies conversation, not the world
A dev forks a session into Branch A and Branch B; both run `git commit` + push, and one sends a notification email. Mental model: "forking is a sandbox, I can throw away whichever branch I don't like and it's as if it never happened." Most important correction?

- **A.** Forking is a full sandbox; discarding a branch's conversation auto-rolls-back its commits and un-sends emails.
- **B.** Forking copies the **conversation array**, not the external world; side effects (commits, pushes, email) actually happened and are **not** undone by discarding the transcript — both branches produced real commits and one a real email. ✅
- **C.** Forking is unsafe because both branches secretly write the same session ID, so B's commits overwrite A's transcript.
- **D.** The email is fine to discard because tool calls are simulated in forks until merged back.

**Correct: B.** A fork copies only the `messages` array; side effects escape into the real world and persist regardless of the transcript. For irreversible external actions, back forks with real isolation (worktrees/branches, sandboxes, dry-run, deferred push/email until the winner is chosen).
- **A** wrong: no mechanism reaches back into git/mail to reverse committed/sent actions.
- **C** wrong: forks are independent timelines written to a **new** session ID; the real danger is real-world side-effect collision, not transcript overwrite.
- **D** wrong: forked tool calls are **real**, executed against the real world; there is no "dry-run until merge," and branches don't auto-merge.

### Q4 — Cold-cache-on-resume cost
A long session builds a large context in the morning, sits idle over a 3-hour lunch/meetings block, then resumes with `--continue`. The first turn after resuming is noticeably slower and pricier than the turns before leaving, though the history is identical. Best explanation?

- **A.** `--continue` re-runs every prior tool call to rebuild state.
- **B.** The stateless API charges a one-time "reconnection fee" for resuming, independent of caching.
- **C.** The prompt cache entry for the large stable prefix **expired** during the idle gap (past TTL), so the first resumed call must **re-write** the whole prefix at full (slightly premium) price instead of a ≈90%-off cache read; later turns re-amortize. ✅
- **D.** Resuming forces the model to re-summarize the entire history, billed as extra output tokens on the first turn.

**Correct: C.** Caching is TTL-bound (~5 min default / 1-hour optional); a 3-hour gap evicts the entry. First resumed turn = cache miss → re-write prefix (~1.25×) + full recompute → slower and pricier; then warm again. Same tokens, changed cache state.
- **A** wrong: resumption is not re-execution.
- **B** wrong: stateless API has no suspended session to reconnect to, and no such fee — it's cache write-vs-read economics, not "independent of caching."
- **D** wrong: plain `--continue` re-sends full history; it doesn't auto-summarize (compaction is a separate, explicit mechanism). The cost is input-side (cache re-write), not output.

### Q5 — Fork vs. subagent boundary
From a shared expensive base context, produce two deliverables: (1) a *continuation of the same conversation* — keep reasoning as itself with full history, explore an alternative while leaving the original line intact as fallback; (2) an *isolated self-contained sub-task* — analyze a module and return **just a summary string**, no need to carry the conversation, no risk of noisy intermediate steps bloating the main context. Correct pairing?

- **A.** D1 → subagent; D2 → fork.
- **B.** D1 → **fork**; D2 → **subagent**. ✅
- **C.** Both → fork (same base context).
- **D.** Both → subagent (both independent units).

**Correct: B.** D1's tells (continue the conversation as itself, full history, parent preserved as fallback) = fork. D2's tells (isolated, string-only return, no context bloat) = subagent. Rule: continue-with-history → fork; isolated-worker-returns-string → subagent.
- **A** wrong: exactly inverted — subagents can't continue as themselves with full history; forks can't return a clean string without dragging the whole array + intermediate steps.
- **C** wrong: shared origin doesn't dictate mechanism; D2 needs isolation + string + no bloat, which a fork defeats.
- **D** wrong: "both independent" is insufficient; D1 needs conversation continuity + preserved fallback, which a subagent can't provide.

---

## Clarifications raised during the session

- **`--continue` vs `--resume`:** `--continue` auto-reloads the *most recent* session for the current directory (no picker); `--resume` opens an interactive picker (or takes an explicit session ID). Both are resumption of one timeline. `--fork-session` (with either) branches to a **new** session ID, preserving the original as a checkpoint.
- **Prompt caching is content-addressed:** cache key = prefix content hash, not session ID — which is *why* two forks sharing a prefix can both hit the same cached entry. Savings only within the TTL (≈5 min default / 1-hour extended), so run forks promptly.
- **Forking reduces price-per-token of the shared prefix, not the token count** — each branch still re-sends its full copied history every turn.
