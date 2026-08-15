# Topic 3.6 — Integrate Claude Code into CI/CD pipelines

**Domain:** 3 — Claude Code Configuration & Workflows (20%)
**Task Statement:** 3.6 — Integrate Claude Code into CI/CD pipelines
**Topic number:** 18 of 30 (6th and final topic of Domain 3)
**Primary exam scenario:** Scenario 5 — *Claude Code for Continuous Integration* (automated code reviews, test generation, PR feedback, minimize false positives). Pairs Domain 3 with Domain 4.

---

## 1. The core mental shift

Interactive Claude Code assumes a **human at a terminal**. CI removes three affordances at once:

| Interactive | CI |
|---|---|
| A human answers prompts | Nobody is there — a prompt = a hung job |
| A human reads prose and judges it | A pipeline step must consume output programmatically |
| A human carries context across turns | Each run starts cold; context must be *supplied* |

The whole task statement answers one question: **how do you make a conversational tool behave like a deterministic pipeline step?**

Three mechanisms:
1. `-p` → removes interactivity
2. `--output-format json` + `--json-schema` → makes output machine-consumable
3. `CLAUDE.md` → supplies project knowledge without a human

Plus one architectural rule: **the reviewer must be a fresh instance.**

---

## 2. `-p` / `--print` — non-interactive mode

```bash
claude -p "Review the changes in this PR for correctness bugs."
```

`-p` (long form `--print`) runs a **single non-interactive turn**: Claude reads the prompt, runs its agentic loop internally (tools, file reads), prints a final result, and **exits**. No REPL, no waiting on stdin.

**The classic CI failure:** invoking `claude "prompt"` *without* `-p`. The CLI starts an interactive session, waits on terminal input that never arrives, and the job hangs until the runner's timeout kills it.

Diagnostic signature of this failure:
- **No output at all** (an agentic loop would still emit tool activity)
- **Fails on every PR regardless of size** (a size-independent failure isn't a workload problem)
- **Process starts successfully but never exits**

Wrong fixes to recognize: raising the job timeout, shrinking the diff scope, piping `/dev/null` in, "the prompt is too vague." All treat symptoms; the cause is running an interactive tool non-interactively.

**Related CI hazard:** in non-interactive mode there is no human to approve tool use either. If the job needs Bash/Write, tool permissions must be **pre-authorized in configuration** before the run. Know the principle; flag spellings are not worth memorizing.

---

## 3. `--output-format json` and `--json-schema`

Prose is fine for a human reading a terminal; it is a liability for a pipeline that must produce **inline PR comments**, which need discrete fields (file, line, severity, message).

```bash
claude -p "Review the diff and report findings." \
  --output-format json \
  --json-schema ./review-schema.json \
  > findings.json
```

- `--output-format json` — emit a structured JSON envelope instead of free text
- `--json-schema` — supply a JSON Schema the result must conform to, so the shape is **enforced**, not hoped for

**Anti-pattern:** asking for nicely formatted markdown and regex-parsing it in the CI script. Works for the first ten PRs, then breaks when phrasing drifts — natural-language output has **no contract**.

Symptoms of the missing-contract failure: intermittent breakage, comments on wrong lines, script crashes on malformed entries, and **re-running the same PR sometimes works**. Same input → different parse results means you're depending on generation variance, and variance is only fixable by constraint, never by instruction.

**Principle (recurs in Domain 4):** when a downstream system consumes the output, **constrain the output at generation time; don't repair it afterward.** Prompting shapes a *tendency*; schemas provide a *guarantee*.

Typical finding schema:

```json
{
  "findings": [{
    "file": "src/auth.ts",
    "line": 42,
    "severity": "high" | "medium" | "low",
    "category": "correctness",
    "message": "...",
    "confidence": "high" | "medium" | "low"
  }]
}
```

---

## 4. `CLAUDE.md` as the CI context mechanism

In CI there's no human to say "we use pytest fixtures, not setup methods" or "don't flag missing docstrings — the linter owns that." That knowledge must already be in the repo.

`CLAUDE.md` is the right answer because it is **committed alongside the code**, giving three properties a CI prompt string lacks:

1. **Versioned** — standards evolve with the branch they apply to
2. **Shared** — the *same* file governs the developer's local session and the CI review, so CI stops surfacing surprises
3. **Single source** — no duplication across workflow YAML files, no drift

Contents relevant to CI: testing standards, fixture conventions, review criteria, **what not to flag**, architectural constraints, naming conventions.

It is loaded in `-p` mode exactly as interactively — no flag required.

**Composes with 3.1 and 3.3:** path-specific rules in `.claude/rules/` with `paths:` globs load only for matching files, so a CI run touching only `terraform/**` doesn't drag in React conventions.

### The scoping trap (ties 3.1 → 3.6)

A CI runner does a **fresh checkout**. It gets **project-level** `CLAUDE.md` and `.claude/rules/` because those are committed. It does **not** get `~/.claude/CLAUDE.md` — that lives on the developer's machine.

So the symptom *"the review respects our conventions locally but ignores them in CI"* has one root cause: **the standards were in user-level memory and were never committed.** This is the 3.1 hierarchy question wearing a CI costume.

**Anti-pattern:** the entire review rubric embedded in the workflow YAML prompt string — invisible to developers, un-reviewable in a PR, diverges from local behavior.

---

## 5. Session context isolation — why a fresh reviewer beats the author

**Claim (explicit in the exam guide):** the same session that generated the code is **less effective at reviewing its own changes** than an independent review instance.

Why:
- The generating session's context is saturated with its own reasoning — assumptions made, tradeoffs already settled. That reasoning is *evidence in context* that the code is correct. **A reviewer that has already been persuaded is not a reviewer.**
- Bugs live in the gap between what the author **intended** and what the code **does**. The author's session holds the intent, so it reads the intent into the code and the gap closes invisibly.
- The generating context is also full of noise (exploration, discarded approaches, tool output) that dilutes attention on the diff.

The independent instance sees only what actually exists: **the diff, the surrounding code, and the standards in `CLAUDE.md`.** It must derive intent from the code — exactly the check you want. This reproduces the condition of the cold human reviewer who spots the bug in minutes.

**Diagnostic tell in questions:** reviews that are *detailed and well-written but conclude the code is correct*. The reviewer wasn't lazy — it was **compromised**.

**CI implication:** "Claude writes the fix, then Claude reviews the fix" must be **two separate `claude -p` invocations**, not one session doing both.

Same isolation argument as coordinator/subagent (1.2) and `context: fork` for skills (3.2), applied to *evaluation*: **an evaluator must not inherit the producer's context.** Recurs in Domain 4.6 (multi-instance review) and Domain 5.

---

## 6. Re-running reviews on new commits — duplicate comments

Problem: a PR is reviewed, the author pushes more commits, CI re-runs, and now the same issue appears three times plus comments about already-fixed problems. Developers filter the bot out. This is a **trust** failure.

**Correct fix: include the prior review findings in the context of the new run, and instruct Claude to report only new or still-unaddressed issues.**

Claude then does the comparison **semantically and against the current code** — it recognizes that "unvalidated input at line 42" and "missing input check at line 47" are the same issue after a refactor moved lines, and it can *verify* whether an earlier issue was actually fixed. This fixes both failure modes (duplicates **and** stale comments), which come from the same missing state.

| Wrong approach | Why it fails |
|---|---|
| Dedupe posted comments by string/path+line matching | Line numbers shift every commit; wording varies; loosening the match suppresses genuinely new findings; **and it can never detect that an issue was *resolved*** |
| Review only the incremental diff (last commit) | Can never confirm prior findings were addressed, so stale comments persist forever; also misses bugs arising from interaction between new and existing code |
| Delete all bot comments and repost each run | Destroys discussion threads and re-notifies everyone — makes notification fatigue worse |
| Only review once, on PR open | Everything pushed afterward goes unreviewed |

---

## 7. Test generation — same principle, different artifact

Identical failure mode: Claude proposes tests that **already exist**.

**Fix: provide the existing test files in context** so generation is aware of current coverage and proposes only genuinely uncovered scenarios. Pair with `CLAUDE.md` carrying fixture conventions and testing standards so generated tests match house style and actually run.

**Symmetry with §6:** the pipeline is stateless across runs, so **prior state must be supplied as context** — prior findings for review, existing tests for generation.

---

## 8. Schema & severity gating design

**The one idea:** severity is a **field the model fills in**, and the **pipeline** reads that field to decide the build outcome. Two separate responsibilities.

```json
"severity": { "enum": ["high", "medium", "low"] }
```

Three exam-relevant points:

**1. Severity must be an enum, not prose.** An enum gives the pipeline a closed set to branch on exhaustively — `if any(f.severity == "high"): exit 1`. Free-text severity ("this seems fairly serious") is unbranchable and puts you back to regex-parsing prose.

**2. Claude classifies; the pipeline enforces.** Claude answers *"how bad is this?"* (a judgment about code). The pipeline answers *"does this block the merge?"* (a policy decision).

Asking Claude to decide the merge outcome directly (e.g. a `should_block_merge` boolean) is the anti-pattern:
- Merge policy becomes probabilistic — same finding class may block Monday, pass Tuesday
- Policy can't change without editing the prompt (tightening before a release, relaxing on a hotfix branch should be a config change)
- The model lacks visibility into release timing, branch, and team norms — it can't answer the question well

Same instinct as 1.4 and 1.5: **when compliance must be guaranteed, enforce it programmatically, not by instruction.**

**3. Comment on everything, block on high only.** This is the false-positive answer. A gate that fails the build on *every* finding means one wrong "low" finding blocks a correct PR — developers respond by demanding the bot be disabled. Two tiers preserve both signal and trust. If offered "block on all findings" vs "block on high-severity only," the tiered option wins, and the reason is **developer trust**, not throughput.

Adding a `confidence` field alongside `severity` is a valid refinement to recognize, but not required.

---

## 9. Minimizing false positives (Scenario 5's stated goal)

Levers in scope here:
- **Explicit criteria in `CLAUDE.md`**, including explicit **non**-criteria ("do not flag formatting; Prettier owns that"). Vague instructions like "review for issues" generate noise by construction.
- **Severity + confidence in the schema** — post low-confidence items as non-blocking notes; gate only on high severity.
- **Independent review instance** — a self-reviewing author produces both false negatives and shallow findings.

Domain 4.1 covers false positives in depth; here, know that CI is where they hurt and that the levers are **project context + structured severity**, not "tell Claude to be more careful."

---

## 10. Anti-pattern summary (exam-ready)

| Anti-pattern | Correct approach |
|---|---|
| `claude "..."` without `-p` in CI | `claude -p "..."` — the job hangs otherwise |
| Regex-parsing prose output into PR comments | `--output-format json` + `--json-schema` |
| Few-shot examples to stabilize a prose format a script parses | Right technique, wrong layer — enforce with a schema |
| Making the parser more tolerant / skipping unparseable entries | Silently drops real findings while the job still reports success |
| Review criteria embedded in workflow YAML | `CLAUDE.md`, committed and shared with local dev |
| Standards in `~/.claude/CLAUDE.md` | Not checked out by CI — must be project-level |
| Same session generates then reviews the code | Separate, independent `claude -p` invocation |
| "Same session, but be more skeptical / review twice" | Instruction can't un-see context; two contaminated passes fail on the same cases (correlated errors) |
| Script-side dedup of repeat comments | Pass prior findings into context; ask for new/unaddressed only |
| Test generation blind to the existing suite | Provide existing test files in context |
| Fail the build on every finding | Severity enum in schema; gate on high severity only |
| Model emits `should_block_merge` | Model classifies severity; pipeline applies policy |

---

## 11. The one-line version

**CI removes the human, so everything the human used to supply — interactivity (`-p`), interpretation (`--output-format json` + `--json-schema`), project knowledge (`CLAUDE.md`), and memory of prior runs (prior findings / existing tests in context) — must be moved into configuration and context; and the reviewer must be a fresh instance, because the author's context is evidence for the author's code.**

---

# Practice Questions

## Question 1

**Scenario.** Your team has added an automated code review step to the GitHub Actions workflow that runs on every pull request:

```yaml
- name: Claude review
  run: claude "Review the changes in this PR and report any correctness bugs."
```

Locally, the same prompt produces a useful review in under a minute. In CI, the step produces no output and the job is eventually killed by the runner's 60-minute timeout. This happens on every pull request, regardless of size. The runner logs show the process started successfully and never exited.

**What is the most likely root cause?**

- **A.** The CI runner lacks a `CLAUDE.md` file, so Claude is waiting for project context to be supplied before it can begin the review.
- **B.** The command omits the `-p` flag, so Claude Code starts an interactive session and blocks waiting for terminal input that never arrives.
- **C.** The prompt is too open-ended, causing Claude to enter an unbounded agentic loop reading files across the repository without reaching a stopping condition.
- **D.** The runner's default job timeout is too short for a full-repository review; the step needs a longer timeout and a smaller diff scope.

**Correct answer: B** ✅ *(answered correctly)*

**Why B is right.** Without `-p` / `--print`, Claude Code launches its interactive REPL. It initializes fine — hence "the process started successfully" — then blocks on stdin waiting for a human to type. In CI there is no terminal and no human, so it waits until the runner kills it. All three tells point here: **no output at all** (an agentic loop would still emit tool activity), **every PR regardless of size** (a size-independent failure isn't a workload problem), and **starts but never exits**. Fix: `claude -p "..."`.

**Why A is wrong.** A missing `CLAUDE.md` degrades review *quality* — generic standards instead of yours — but never blocks execution. Claude Code doesn't wait for context files; it proceeds with whatever memory it finds. The distractor works by pairing a real 3.6 concept with a failure mode it doesn't cause.

**Why C is wrong.** An unbounded loop would produce continuous log output and consume tokens, and it would correlate with repository size — but the failure is identical on every PR with zero output. A vague prompt is a real CI quality problem (it drives false positives) but it produces *bad reviews*, not *hung jobs*.

**Why D is wrong.** Treats the symptom. The job isn't slow, it's stuck — 60 minutes far exceeds any review, and raising the timeout wastes more CI minutes before the same kill. Any option proposing a longer timeout or smaller scope for a process producing *zero* output and never terminating is treating a hang as if it were latency.

**Transferable pattern.** When a tool that works interactively fails in automation, first ask what the human was supplying. Here it was keystrokes.

---

## Question 2

**Scenario.** Your CI pipeline runs `claude -p` on every pull request to review the diff, then a Python script posts each finding as an inline comment. The prompt asks Claude to format the review as a markdown list where each entry begins with the file path and line number, and the script uses a regular expression to extract those fields.

For the first several weeks this works. Over the following months the team reports intermittent failures: some PRs get no comments at all despite the job succeeding, others get comments attached to the wrong lines, and occasionally the script crashes on a malformed entry. Re-running the same PR sometimes produces correct comments.

**Which change most effectively addresses the root cause?**

- **A.** Add few-shot examples to the prompt showing the exact markdown format required, and instruct Claude to follow the format precisely without deviation.
- **B.** Make the parsing script more tolerant — broaden the regular expression, skip entries that fail to parse, and log them for manual follow-up.
- **C.** Invoke Claude with `--output-format json` and a `--json-schema` defining the finding fields, and have the script consume the validated JSON directly.
- **D.** Pin the CI job to a fixed model version and set a lower temperature so that output formatting becomes deterministic across runs.

**Correct answer: C** ✅ *(answered correctly)*

**Why C is right.** The root cause is a **contract that doesn't exist**. Markdown formatting is a *request*; the model can perform the review perfectly while phrasing the wrapper differently, and any drift breaks a consumer that requires discrete fields. `--output-format json` with `--json-schema` moves the contract from "asked for" to "enforced at generation time." The intermittency is the signature: *same input, different parse results* means you're depending on generation variance, which is fixable only by constraint, not instruction.

**Why A is wrong (strongest distractor).** Few-shot examples raise format compliance from ~95% to ~99% but never to 100%, because prose formatting has no enforcement mechanism. For a step running on every PR forever, a residual failure rate is a recurring outage — and you've added prompt complexity while keeping the fragile parser. The trap: few-shot prompting is legitimate (Domain 4.2), so this is **correct advice applied at the wrong layer**.

**Why B is wrong.** Explicit symptom management, and it worsens what matters most: silently skipping unparseable entries means **real bugs never reach the developer**, while the job still reports success. "Some PRs get no comments at all" becomes designed behavior. Broadening the regex also increases wrong-line attachments.

**Why D is wrong.** Neither lever solves it. Pinning the model version removes one drift source but not run-to-run variance within a version, and there is no temperature control on the Claude Code CLI. Even granting the premise, "make free-text output more stable" is the same category error as A with a weaker mechanism.

**Transferable pattern.** When a program consumes model output, constrain the output at the point of generation. Prompting shapes a *tendency*; schemas provide a *guarantee*. Options proposing parsing, repairing, or tolerating malformed prose treat a missing contract as a parsing problem.

---

## Question 3

**Scenario.** Your team runs a two-stage automated workflow on internal maintenance PRs. Stage one invokes Claude Code to implement a small refactor described in the ticket. Stage two reviews the changes for correctness before a human approves the merge. To keep the workflow efficient, both stages run in a **single Claude Code session**: the same session that made the edits is then asked to review its own diff.

Over three months, the review stage has approved several changes that later caused production incidents. In each case, a post-mortem found the bug was visible in the diff, and an engineer reading the diff cold identified it in minutes. The reviews themselves were detailed and well-written — they simply concluded the changes were correct.

**What is the most effective change to the workflow?**

- **A.** Keep the single session, but add an explicit instruction to the review prompt directing Claude to adopt a skeptical, adversarial stance and to assume defects are present.
- **B.** Keep the single session, but clear the conversation history between the implementation and review stages so the review begins from a clean context.
- **C.** Run the review as a separate, independent Claude Code invocation that receives only the diff, the surrounding code, and the project's review standards.
- **D.** Keep the single session and add a second review pass, so the same session reviews its own diff twice with different review criteria each time.

**Correct answer: C** ✅ *(answered correctly)*

**Why C is right.** The diagnostic detail is that reviews were *detailed and well-written but wrong* — the reviewer wasn't lazy, it was **compromised**. The generating session's context contains its own reasoning: assumptions made, tradeoffs settled, the belief the approach is sound. Bugs live in the gap between intended and actual behavior, and a session holding the intent reads that intent into the code, closing the gap invisibly. The cold engineer catches it precisely *because* they only have the artifacts. An independent invocation reproduces that condition by construction — it must derive intent from the code. In CI this is nearly free: a second `claude -p` call.

**Why B is wrong (strongest distractor).** It targets the right variable (contaminating context) with the wrong mechanism. First, it is **isolation by cleanup rather than by construction** — mutating a session mid-workflow and then trusting the removal was complete, with no way to verify. Second, note where it converges: if you truly cleared everything and re-supplied the diff and standards, you have built an independent instance via a fragile stateful path instead of just launching one. When the design goal is isolation, get it **structurally** — mirroring `context: fork` (3.2) and subagent isolation (1.2).

**Why A is wrong.** Attempts to fix a **context** problem with an **instruction**. The session isn't insufficiently critical; it holds evidence that the code is correct. Instructing it to assume defects against that evidence yields either the same conclusion in skeptical language, or manufactured findings — a false-positive generator. Prompting cannot un-see context.

**Why D is wrong.** Two contaminated passes aren't better than one. Both inherit the identical bias, so errors are **correlated** — the second pass fails on the same cases as the first, at double the cost, producing unearned confidence. Multi-pass review is real (Domain 4.6), but its value comes from *independence* between passes; from a shared polluted context, that independence is gone.

**Transferable principle.** *An evaluator must not inherit the producer's context.* "Same session, but ___" options tend to be distractors when the question is about review quality.

---

## Question 4

**Scenario.** Your automated review posts inline comments on every pull request and re-runs on each new push. Developers report the bot has become noise: after a few rounds of pushes, the same issue appears as three or four separate comments on nearly identical lines, and comments persist for problems the author already fixed two commits ago. Several developers now filter the bot's notifications out of their inbox entirely.

The pipeline currently re-reviews the full diff on each push and posts every finding it returns.

**Which change most effectively addresses this?**

- **A.** Have the pipeline delete all previously posted bot comments before each re-run, then post the current findings as a fresh set.
- **B.** Include the findings from previous review runs in the context of the new run, and instruct Claude to report only issues that are new or still unaddressed.
- **C.** Restrict each re-run to reviewing only the commits pushed since the previous review, so previously reviewed code is not examined again.
- **D.** Have the pipeline compare each new finding against previously posted comments by file path and message text, and suppress any that match.

**Correct answer: B** ✅ *(answered correctly)*

**Why B is right.** The pipeline is stateless across runs, so it has no idea what it already said or what got fixed. Supplying prior findings as context restores that state to the component best able to use it: Claude compares new findings against old ones **semantically and against the current code**. It recognizes that "unvalidated input at line 42" and "missing input check at line 47" are the same issue after a refactor shifted the lines, and — critically — it can *verify in the current code* whether an earlier issue was resolved, which is the only way to stop commenting on fixed problems. Both failure modes stem from the same missing state, and this fixes both.

**Why A is wrong.** Suppresses duplicates by destroying the record: replies, resolution threads, and discussion all disappear on every push, and everyone is re-notified each round — making the notification problem *worse*, which is the scenario's actual complaint. It still can't distinguish fixed from unfixed issues.

**Why C is wrong (plausible-efficiency trap).** Reviewing only new commits means the bot can never confirm earlier findings were addressed — it has stopped looking at that code — so stale comments persist permanently. It also introduces false negatives: bugs arising from the *interaction* between a new commit and existing code fall outside the narrowed scope. A noise problem traded for a coverage problem.

**Why D is wrong (strongest distractor — looks like the disciplined engineering answer).** It fails on mechanics: line numbers shift with every commit so path+line matching misses; message text varies between runs so text matching misses; loosening the match starts suppressing *genuinely new* findings that read similarly. Most decisively, string comparison has no notion of "fixed" — it detects repetition, never resolution, so the stale-comment half survives untouched.

**Transferable principle (same shape as Q2).** Don't reconstruct a judgment mechanically downstream when you can give the model the state it needs to make that judgment directly. In Q2 the missing piece was a contract; here it's memory. Test generation works identically — pass in existing test files.

---

## Question 5

**Scenario.** Your CI review step returns structured JSON findings and fails the build whenever the findings array is non-empty. Engineering leadership reports two complaints: PRs are being blocked by findings that turn out to be stylistic preferences or outright false positives, and developers have started asking for the review step to be made optional.

You want the bot to keep surfacing everything it finds while blocking merges only for genuinely serious defects.

**Which approach is most appropriate?**

- **A.** Add a `severity` field with an enumerated set of values to the output schema; post all findings as PR comments, and configure the pipeline to fail the build only when a finding's severity is `high`.
- **B.** Add a `should_block_merge` boolean to the output schema and have Claude set it per finding; configure the pipeline to fail the build when any finding has that flag set to true.
- **C.** Instruct Claude in the prompt to report only serious defects and to omit stylistic or low-confidence observations entirely, leaving the pipeline's failure condition unchanged.
- **D.** Keep the current failure condition, but document a process by which developers can request an override from a reviewer when they believe a finding is a false positive.

**Correct answer: A** ✅ *(answered correctly)*

**Why A is right.** It separates two responsibilities. Claude answers *"how bad is this?"* — a judgment about code, which it's good at. The pipeline answers *"does this block the merge?"* — a policy decision that must be deterministic and belongs in configuration. An **enum** makes the gate implementable: a closed set the pipeline can branch on exhaustively. The two-tier outcome satisfies both stated goals — everything surfaces as a readable/dismissable comment, while only high-severity findings stop the merge. That keeps signal without spending team trust.

**Why B is wrong (strongest distractor).** Looks equivalent but hands the **policy decision** to the model. Three consequences: (1) merge policy becomes probabilistic — the same class of finding may block Monday and pass Tuesday; (2) policy can't change without editing the prompt, though tightening before a release or relaxing on a hotfix branch should be a config change; (3) the model is asked a question it can't answer well, since whether a defect blocks a merge depends on release timing, branch, and team norms it can't see. In A, the same underlying judgment is expressed as *data* that policy consumes. This is the CI instance of the 1.4/1.5 principle: **when compliance must be guaranteed, enforce it programmatically rather than by instruction.**

**Why C is wrong.** Solves blocking by discarding information: real-but-minor findings vanish, so the team loses feedback it explicitly wants ("keep surfacing everything it finds"). It also doesn't fix false positives — an item Claude misjudges as serious still passes the filter and still blocks the build. The funnel narrows without improving classification, and the gate remains all-or-nothing.

**Why D is wrong.** Pure process wrapped around an unfixed defect. Every false positive still blocks a PR and now additionally consumes a second engineer's time. Friction goes *up*. Whenever an option answers a fixable technical problem with a human escalation procedure, check whether the underlying gate was corrected — here it wasn't.

**Transferable principle.** Structured output isn't only about parseability; it's about **placing decisions at the right layer**. Model emits classification → pipeline applies policy. The tiered gate wins because of developer trust: a gate that fires on false positives gets disabled, and a disabled reviewer catches nothing.

---

# Clarifications raised during the session

**Q: What else is worth knowing for the exam beyond the main teaching?**

Ranked by likelihood of appearing:

1. **`CLAUDE.md` scoping in CI** *(explicitly in scope)* — CI checks out project-level memory only; `~/.claude/CLAUDE.md` never reaches the runner. Root cause of "works locally, ignored in CI." See §4.
2. **Who decides pass/fail** — model classifies, pipeline enforces. See §8.
3. **Tool permissions must be pre-authorized** *(adjacent)* — no human means no runtime approvals. Know the principle, not the flag spellings.
4. **`--json-schema` (CLI) vs `tool_use` (API)** *(bridges to 4.3)* — same goal, different layer: the CLI flag constrains a Claude Code run; `tool_use` constrains a Messages API call. Don't conflate them if a question mixes CI and API contexts.
5. **Domain 4 overlap** — Scenario 5 pairs Domain 3 with Domain 4, so several "CI" questions are really prompt-engineering questions (explicit criteria, false positives → 4.1).

Not worth study time: `stream-json` formats, GitHub Action YAML specifics, runner configuration.

**Q: More on schema/severity gating, exam-necessary only.**

Covered in §8. Three points: severity must be an **enum** (branchable, closed set); **Claude classifies, the pipeline enforces** (never let the model emit the merge decision); **comment on everything, block on high only** (the tiered gate wins on developer-trust grounds, not throughput). A `confidence` field alongside `severity` is a recognizable refinement, not a requirement.
