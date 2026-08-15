# Topic 3.1 — Configure CLAUDE.md files with appropriate hierarchy, scoping, and modular organization

- **Domain:** 3 — Claude Code Configuration & Workflows (20% of exam)
- **Topic number:** 3.1 (Topic 13 of 30 overall)
- **Exam scenarios that draw on this:** Scenario 2 (Code Generation with Claude Code), Scenario 4 (Developer Productivity with Claude)

---

## 1. Mental model — CLAUDE.md is auto-injected prompt, not config

CLAUDE.md is a **memory file whose contents are prepended into Claude Code's context at session start**. It is persistent, project-scoped prompt engineering directed at a model reader.

It is **not**:

| CLAUDE.md is NOT… | The actual mechanism |
|---|---|
| a permissions system | `settings.json` (`allow` / `deny` rules) |
| an enforcement mechanism | **hooks** (PreToolUse / PostToolUse — Topic 1.5) |
| MCP server configuration | `.mcp.json` / `~/.claude.json` (Topic 2.4) |

Two consequences that drive most exam items on this task statement:

1. **Everything loaded costs context on every turn, for every teammate, in every session.** A monolithic CLAUDE.md is a permanent context tax *and* dilutes attention across rules irrelevant to the current task. "Just add it to CLAUDE.md" is not free.
2. **Compliance is probabilistic.** CLAUDE.md raises the likelihood a convention is followed; it does not guarantee it. For "must never happen" requirements the correct answer is a **hook** (or a `settings.json` deny rule), never a stronger sentence.

---

## 2. The hierarchy (highest-yield fact in the topic)

Ordered broad → narrow:

| Level | Location | Shared with team? | Typical content |
|---|---|---|---|
| **Enterprise / managed policy** | system-wide managed path | Yes — org-wide, IT-deployed | org security & compliance mandates |
| **User (personal / global)** | `~/.claude/CLAUDE.md` | **No** — never version-controlled | personal preferences, tone, local machine facts |
| **Project** | repo root `CLAUDE.md` (or `.claude/CLAUDE.md`) | **Yes** — checked into git | team conventions, architecture, build/test commands |
| **Directory / subtree** | `packages/api/CLAUDE.md` | Yes — checked in | rules that apply only to that package |

### How they combine

- **Layered, additive — not replacement.** A subtree file *refines* the inherited set; the root file still governs everything the narrower file doesn't address.
- **Narrower scope wins on conflict**, and only on the conflicting instruction.
- **Discovery = directory walk from cwd upward to the repo root**, collecting every `CLAUDE.md` on the path, plus user-level and enterprise files.
- **Directory-level files are loaded lazily** — a file in `packages/payments/` is not dead weight in a session that never touches payments. This is why directory scoping actually reduces per-session context.
- **Conflict resolution is model judgment, not a hard rule engine.** "Narrower wins" is a strong tendency, not a guarantee — so the exam-correct fix for contradictory rules is to *remove the contradiction*, or make the override explicit ("this package overrides the repo-wide Vitest rule").
- **Enterprise is broadest in scope but highest in authority.** Broad scope ≠ weak.

### The signature diagnostic

> *Works for one engineer but not for teammates who cloned the same repo at the same commit.*

→ **The instructions are in that engineer's `~/.claude/CLAUDE.md` (user-level), which git does not distribute.**
Fix: move team-relevant instructions into the project-level `CLAUDE.md` and commit it; leave only genuinely personal preferences at user level.

The mirror-image error is equally testable: **personal preferences committed to the project file** impose one developer's taste on everyone and waste shared context.

---

## 3. `@import` — modularity without duplication

```markdown
# Payments Service

Build: `pnpm --filter payments build`

## Standards that apply here
@../../standards/typescript.md
@../../standards/pci-handling.md

## Personal notes (not in git)
@~/.claude/my-payments-notes.md
```

Mechanics worth memorizing:

- Syntax `@path`; **relative**, **absolute**, and **`~/`-prefixed** paths all work.
- **Recursive, max depth 5 hops** — guards against cycles and runaway expansion.
- **Not evaluated inside code spans / fenced code blocks**, so the syntax can be documented safely.
- **Imports expand into context in place** — an import costs the same tokens as pasting the text, and inline content in the importing file coexists with imported content.
- **⚠ Imports are NOT a context saving.** The benefit is **selectivity + single source of truth**. This is the most commonly probed misconception on this topic.
- **Imports can cross scopes** — a project CLAUDE.md may `@import` a `~/`-path file. This is the sanctioned way to layer uncommitted personal notes onto shared config (and why `CLAUDE.local.md` is deprecated).

**Intended exam skill:** *"Using `@import` to selectively include relevant standards files in each package's CLAUDE.md based on maintainer domain knowledge."* A platform team owns `standards/`; each package's maintainers import only what applies (`ml-pipeline` → `python.md` + `data-privacy.md`; `web` → `typescript.md` + `accessibility.md`).

Rejected alternatives:
- **Copy-paste standards into each package** → drift within a quarter.
- **Put every standard in the root file so nothing is missed** → every developer carries every other team's rules on every turn.

---

## 4. `.claude/rules/` — splitting a monolith by topic

Named in the exam guide as *"an alternative to a monolithic CLAUDE.md"*: topic-specific files such as `testing.md`, `api-conventions.md`, `deployment.md`.

Use when the symptom is *"our CLAUDE.md is hundreds of lines, nobody reviews changes to it, and no one owns any part of it."*

### Choosing between the modular mechanisms

| Split by… | Mechanism | Solves |
|---|---|---|
| **Topic** | `.claude/rules/*.md` | ownership, reviewability (still project-wide, still all loaded) |
| **Location** | directory-level `CLAUDE.md` | context cost + scoping (lazy loading, no cross-package bleed) |
| **Shared canonical text** | `@import` | single source of truth, no drift |

They compose freely — a package-level `CLAUDE.md` that `@import`s shared standards is often the best answer.

---

## 5. Content judgment — what belongs in CLAUDE.md

**Good** (things Claude cannot infer and would otherwise get wrong):
- exact build/test/lint commands (`pnpm test:unit`, not "run the tests")
- repo-specific architecture facts ("all DB access goes through `repositories/`")
- conventions with a concrete rule ("use `zod` for request validation")
- gotchas ("`legacy/` is generated — never edit by hand")

**Bad:**
- generic advice the model already applies ("write clean code", "meaningful variable names") — no information, unactionable → **delete, don't relocate**
- vague aspirations ("be careful with performance")
- anything CI/linters already enforce deterministically
- **secrets, tokens, credentials** — committed file *and* it flows into context
- long prose, changelogs, full API docs — link or import

**Specificity beats emphasis.** A rule being ignored is more often too vague than too quiet. "Always test your code" → "before finishing any change under `src/api/`, add a test in `tests/api/` and run `pnpm test:api`".

Two-question placement test:
1. *"True for this person, or true for this repo?"* → chooses the level.
2. *"Does this tell Claude something it couldn't infer, specifically enough to act on?"* → decides whether it belongs at all.

---

## 6. Anti-patterns checklist

1. Team conventions in `~/.claude/CLAUDE.md` → invisible to teammates.
2. Personal preferences / local machine facts committed to the project file → imposed on everyone, actively misleading.
3. Monolithic root CLAUDE.md holding every package's rules → context tax + attention dilution + cross-package rule bleed.
4. Duplicated standards text across packages → drift.
5. CLAUDE.md used as enforcement for a must-never-happen rule → use a **hook**.
6. Escalating emphasis (CAPS, "CRITICAL!!!", repetition) instead of specificity or scoping.
7. Restating repo-wide rules inside subtree files "so nothing is lost" → misunderstands additive layering.
8. Documenting what CI already enforces.
9. Secrets in CLAUDE.md.
10. Confusing CLAUDE.md with `settings.json` (permissions) or `.mcp.json` (servers).

---

## 7. Decision table

| Symptom / requirement | Correct mechanism |
|---|---|
| Works for me, not for teammates on the same repo | Move user-level → project-level `CLAUDE.md`, commit it |
| Rules relevant to only one package in a monorepo | Directory-level `CLAUDE.md` in that package |
| Same standard needed by several packages | Shared standards file + `@import` in each |
| Root CLAUDE.md too long, mixes unrelated topics, unowned | Split into `.claude/rules/*.md` |
| Irrelevant rules loaded every session / applied to wrong package | Directory-level scoping (lazy load) |
| Rule must be obeyed 100% of the time | **PreToolUse hook** (or `settings.json` deny), not CLAUDE.md |
| Only *I* want this behavior; local machine facts | `~/.claude/CLAUDE.md` |
| Org-wide compliance mandate for all repos | Enterprise / managed policy level |
| Restrict which commands Claude may run | `settings.json` permissions |

---

# Practice Questions

## Question 1 — the per-user split diagnostic

A staff engineer has used Claude Code on a TypeScript monorepo for three months with excellent results: Claude uses the repository pattern, runs package-specific test commands, and never edits the generated `legacy/` directory. Two new engineers clone the same repository at the same commit and report that Claude writes raw SQL in route handlers, suggests `npm test` instead of `pnpm --filter <pkg> test`, and has modified files under `legacy/`. Nothing in the repository changed; all three are on the same Claude Code version and model. **Most likely root cause and correct fix?**

- **A.** Context-window pressure is evicting the project `CLAUDE.md`; shorten it and move detail into `.claude/rules/`.
- **B.** The conventions live in the staff engineer's `~/.claude/CLAUDE.md`, which is not version-controlled; move the team-relevant instructions into the project-level `CLAUDE.md` and commit it.
- **C.** The new engineers have not run `/init`, so no memory file has been generated for them; have each run `/init` after cloning.
- **D.** The conventions are in the project `CLAUDE.md` but stated too weakly; rewrite with emphasis markers (**IMPORTANT**, **NEVER**).

**Correct answer: B**

**Why B is right:** The fingerprint is *identical repo, identical commit, identical tooling — different behavior per person*. Anything stored in the repo is byte-for-byte the same for all three, so a repo-resident cause cannot explain a per-user split. The only memory layer that varies by person on the same repo is **user-level `~/.claude/CLAUDE.md`**, which lives on the individual's machine and is never distributed by git.

**Why the others are wrong:**
- **A** misdescribes the mechanism (memory is injected persistent context, not silently evicted early in a fresh session) and, decisively, would affect the staff engineer identically — it cannot produce a user-specific failure. `.claude/rules/` is a real mechanism aimed at a different problem (monolithic organization). Classic *correct-mechanism-wrong-problem* distractor.
- **C** misunderstands `/init`: it is a bootstrapping convenience that generates a starter CLAUDE.md by inspecting the codebase, not a per-user activation step. If a committed CLAUDE.md existed, cloning would deliver it. Worse, three engineers running `/init` produces three divergent memory files instead of one reviewed source of truth.
- **D** presumes the instructions are reaching the new engineers, contradicting the scenario — the same text yields perfect compliance for the staff engineer, so wording is not the variable. Emphasis is the escalation anti-pattern; even where a rule is present-but-inconsistently-followed the exam prefers greater specificity, or a hook when compliance must be guaranteed.

---

## Question 2 — choosing the modular-organization axis

A monorepo with 14 packages has an ~800-line root `CLAUDE.md` accumulated by appending every package's conventions (React accessibility, Python data-pipeline privacy, Go deployment, PCI handling). Engineers report (1) every session loads a large block of mostly irrelevant instructions, and (2) Claude applies rules from the wrong package — e.g. suggesting a React accessibility check while working on the Go payments service. A platform team owns a `standards/` directory of canonical documents. **Which approach most effectively addresses both problems?**

- **A.** Keep the single root `CLAUDE.md` but replace each inlined section with an `@import` to the corresponding `standards/` file.
- **B.** Move each package's conventions into a `CLAUDE.md` inside that package's directory, have each `@import` only the `standards/` documents its maintainers know apply, and keep genuinely repo-wide conventions in the root file.
- **C.** Split the root `CLAUDE.md` into topic files under `.claude/rules/` so each concern is separately owned and reviewable.
- **D.** Keep everything in the root `CLAUDE.md` but add a routing preamble instructing Claude to apply only the section matching the current package.

**Correct answer: B**

**Why B is right:** Two symptoms, and only B lands both. *Context bloat* → directory-level `CLAUDE.md` files load lazily when Claude works in that subtree, so a Go payments session never pays for React rules. *Cross-package bleed* → the same scoping fixes it structurally: the React rules aren't ignored by judgment, they're simply absent from context. *Shared standards* → `@import` gives one canonical source with no drift. Repo-wide conventions correctly stay in the root file.

**Why the others are wrong:**
- **A** is the misconception trap: **imports expand into context, costing the same tokens as inlined text**. The root file looks short in the editor while the session still loads all the material — context is unchanged and every package's rules still reach every session, so the bleed continues. It fixes maintainability only.
- **C** improves ownership and reviewability, but `.claude/rules/` files are still **project-wide**. Reorganizing by topic scopes nothing to a location, so the Go session still carries accessibility rules. Split by *topic* when the complaint is "unreviewable and unowned"; split by *location* when the complaint is "irrelevant rules loaded and misapplied."
- **D** pays the full context cost of all 800 lines *and* asks the model to self-filter — probabilistic, and exactly the failure already occurring. When structure can eliminate a problem, the exam never rewards asking the model to compensate for it in prose.

---

## Question 3 — the CLAUDE.md-vs-hooks boundary

Security requires that in the `infra/` repository Claude Code must **never** execute `terraform apply` (humans only, after review), while `terraform plan` stays freely available. An engineer adds an emphatic `## CRITICAL — DO NOT VIOLATE / You must NEVER run terraform apply` block to the root `CLAUDE.md`. Over a month, compliance is high but not perfect — Claude ran `terraform apply` in two sessions. **Most effective way to meet the requirement?**

- **A.** Move the instruction into `infra/CLAUDE.md` for higher precedence and restate it at the top and bottom of the file.
- **B.** Implement a `PreToolUse` hook that inspects Bash tool calls, blocks any command matching `terraform apply`, and returns a message directing Claude to request human approval.
- **C.** Deploy the instruction at the enterprise/managed policy level so it takes organizational precedence over project- and user-level memory.
- **D.** Add the rule to `.claude/rules/security.md` and `@import` it into the root `CLAUDE.md` so the security team owns a separately reviewable file.

**Correct answer: B**

**Why B is right:** The requirement is *"must never"* — deterministic compliance. CLAUDE.md is prompt text with a non-zero failure rate, demonstrated empirically here: the rule was already maximally emphatic and still failed twice. Once a violation is observed, no rewrite of the same *category* of mechanism fixes it, because the category is the problem. A `PreToolUse` hook is harness-level code that runs before execution and can deny outright; it is deterministic, granular enough to block `apply` while leaving `plan` untouched, and its returned message lets Claude recover gracefully by routing to human approval. **Match the mechanism to the stakes.**

**Why the others are wrong:**
- **A** combines two anti-patterns. The whole repo *is* the infra repo, so moving to `infra/CLAUDE.md` narrows nothing; and "restate at top and bottom" is escalating emphasis — the move that already failed. Precedence resolves *conflicts between instructions*; there is no conflicting instruction here.
- **C** is attractive because enterprise-managed policy is the highest-authority memory layer — but it is still **memory, i.e. instruction text**. Raising the authority of a probabilistic mechanism does not make it deterministic; it changes who may override the rule, not whether the model always obeys.
- **D** is good hygiene aimed at the wrong problem. Ownership and reviewability improve, but the content reaching context is identical to what already failed.

**Also worth knowing:** `settings.json` permission **deny** rules are the other legitimate programmatic layer — simpler for a flat block. Hooks win when conditional logic (thresholds, argument inspection) or a redirect to an alternative workflow is needed. Both beat any CLAUDE.md answer for a "must never" requirement.

---

## Question 4 — content and level judgment (multiple response: select TWO)

A PR adds this block to a project's root `CLAUDE.md`, committed and shared by 30 engineers:

1. Always explain your reasoning conversationally before making any change, and address me as "boss".
2. Database access must go through `repositories/`; never write raw SQL in route handlers.
3. Write clean, well-structured code with meaningful variable names and good separation of concerns.
4. Before finishing any change under `src/api/`, add a test in `tests/api/` and run `pnpm test:api`.
5. My local Postgres runs on port 5433, not 5432 — use that when generating connection strings for scripts.

**Which TWO items should be removed from the project-level `CLAUDE.md` and relocated to the reviewer's `~/.claude/CLAUDE.md`?**

- **A.** Item 1 — conversational-explanation and "boss" address preference
- **B.** Item 2 — repository-pattern rule for database access
- **C.** Item 3 — general "clean code, meaningful names" guidance
- **D.** Item 4 — API test-and-run requirement
- **E.** Item 5 — local Postgres port 5433 detail

**Correct answers: A and E**

**Why A and E:** The test is *"true for this person, or true for this repository?"* **Item 1** is pure personal interaction preference — committing it imposes one developer's tone on 29 colleagues and burns shared context every session. **Item 5** is a fact about **one developer's machine**; committed, it actively misleads, since the other 29 run Postgres on 5432 and Claude will generate wrong connection strings for all of them.

**Why the others stay:**
- **B** is a genuine repo-specific architectural constraint Claude cannot infer from code alone, and must apply uniformly. Exactly what project-level memory is for.
- **D** is the model of a well-written rule: names the trigger (`src/api/`), the artifact (`tests/api/`), and the command (`pnpm test:api`).

**The deliberate trap — item 3 (C):** it should also be removed, but **not by relocating it**. Generic advice a competent model already applies adds no information anywhere, and it is unactionable (no trigger, no artifact, no verification), so moving it to user level merely relocates the waste. **Delete it.** The question asked what should be *relocated*, and C fails that test. Key point: **user-level is not a dumping ground** — personal-but-real → user level; generic and unactionable → delete.

---

## Question 5 — merging semantics vs. replacement

A monorepo's root `CLAUDE.md` holds repo-wide conventions: Vitest for all test suites (`pnpm test`), Conventional Commits, never commit directly to `main`. The Python-based `packages/ml-features/` is the sole testing exception, so its maintainer adds `packages/ml-features/CLAUDE.md` containing a pytest rule (`uv run pytest`) plus `@../../standards/python.md`. A reviewer objects: *"Now that this package has its own CLAUDE.md, Claude will lose the repo-wide rules while working in it — copy the commit-message and main-branch rules into the package file, and re-declare Vitest everywhere else."* **Which response is correct?**

- **A.** The reviewer is right — a directory-level `CLAUDE.md` replaces the root file for work inside that directory, so repo-wide rules must be restated.
- **B.** The reviewer is wrong — memory files are merged, not replaced; the root conventions remain in context inside the package, and the pytest line takes precedence only over the conflicting Vitest rule. No duplication needed.
- **C.** Partially right — the root file remains, but because the package file uses `@import`, only the imported `python.md` content loads and the file's own inline rules are ignored, so the pytest line must move into `standards/python.md`.
- **D.** The reviewer is wrong, but precedence is determined by file size and load order rather than directory depth, so the pytest rule should move into the root `CLAUDE.md` as a documented exception.

**Correct answer: B**

**Why B is right:** Memory files **layer additively**. Claude Code walks from the working directory up to the repo root and collects every `CLAUDE.md` on the path, so inside `packages/ml-features/` both files are in context. Precedence applies only where instructions genuinely conflict — pytest supersedes Vitest, and nothing else is touched. The reviewer's proposal would create the exact drift the layering model prevents.

**Why the others are wrong:**
- **A** is the core misconception: subtree files *refine* the inherited set, they don't shadow it. Under replacement semantics every package file would have to be a full copy of the root, making directory scoping useless.
- **C** invents a mechanic. `@import` expands the referenced content **in place**, alongside the importing file's inline rules; both the pytest line and `standards/python.md` are in context. (Note the file is otherwise well designed: the package-specific rule stays local, the shared standard is imported rather than copied.)
- **D** invents a precedence rule — precedence follows **scope specificity**, not file length or ordering — and its fix is backwards, hoisting one package's exception into everyone's context. Distractors stating plausible-sounding but *invented* mechanisms are common; if you can't name the mechanism from the docs, treat it as suspect.

**Caveat to carry:** because this is all prompt text, "narrower wins" is a strong tendency, not a hard guarantee. Make critical overrides explicit ("this package overrides the repo-wide Vitest rule"), and if it must hold 100% of the time, that's hook territory.

---

## Clarifications raised during the session

**On precedence/merging order:**
- All applicable memory files are merged into context together; none deletes another. Narrower scope wins **only on the conflicting instruction**.
- Discovery is a directory walk from cwd upward to the repo root, plus user-level and enterprise files.
- Conflict resolution is model judgment, not a rule engine — the exam-correct fix for contradictory rules is to remove the contradiction, not rely on precedence.
- Enterprise is broadest in scope but highest in authority; broad ≠ weak.
- Recognition fingerprints: "root says X, package says not-X, inside the package?" → package rule. "Must I repeat root conventions in a package file?" → no, merged. "Two teammates behave differently in the same repo" → a user-level file is contributing and it isn't in git.

**On `@import` mechanics:**
- Path forms: relative, absolute, `~/`-relative.
- Recursive with **max depth 5**.
- Not evaluated inside code spans/blocks.
- **Expanded into context — costs the same tokens as pasting.** The benefit is selectivity + single source of truth, never token savings.
- Can cross scopes (project file importing a `~/` path) — the sanctioned replacement for the deprecated `CLAUDE.local.md`.
- If the question's stated goal is "reduce context loaded per session," imports are the wrong answer — use directory-level scoping (lazy loading) or trim.
