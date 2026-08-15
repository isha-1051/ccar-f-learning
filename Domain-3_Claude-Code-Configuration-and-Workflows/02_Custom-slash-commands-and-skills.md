# Topic 3.2 — Create and configure custom slash commands and skills

- **Domain:** 3 — Claude Code Configuration & Workflows (20% of exam)
- **Topic number:** 3.2 (Topic 14 of 30 overall)
- **Exam scenarios that draw on this:** Scenario 2 (Code Generation with Claude Code), Scenario 4 (Developer Productivity with Claude)

---

## 1. Mental model — four places instructions can live

Domain 3 repeatedly tests one judgment: **given a piece of team knowledge or a repeatable workflow, which mechanism should hold it?** Topic 3.1 supplied CLAUDE.md; 3.2 adds two more. The most common exam distractor is putting the content in the wrong one.

| Mechanism | Who invokes it | When it enters context | Best for |
|---|---|---|---|
| **CLAUDE.md** (3.1) | nobody — auto-loaded | **always**, every session, every turn | universal standards applying to *all* work |
| **Slash command** (`/foo`) | **the human**, explicitly | only when typed | a workflow the developer decides to run |
| **Skill** | **the model**, by matching `description` | only when the task matches | a capability Claude should reach for on its own |
| **Subagent / hook** (Domain 1) | the harness | — | context isolation & deterministic enforcement |

The task statement's own decisive bullet:

> **Choosing between skills (on-demand invocation for task-specific workflows) and CLAUDE.md (always-loaded universal standards).**

Read it as a **cost argument**, not a stylistic one. Anything in CLAUDE.md is a permanent token tax paid on every turn by every teammate, *and* it dilutes attention across rules irrelevant to the current task.

> **If it applies to every task → CLAUDE.md. If it applies to a specific kind of task → a skill (or a command).**

Canonical anti-pattern: a 350-line "how to run a database migration" runbook in CLAUDE.md. Migrations happen twice a month; the instructions load 100% of the time. That is a skill.

---

## 2. Custom slash commands

### Structure

A slash command is **a Markdown file whose body becomes the prompt**. The filename is the command name.

```
.claude/commands/review-pr.md      →  /review-pr        (project-scoped)
~/.claude/commands/scratch.md      →  /scratch          (user-scoped)
.claude/commands/db/migrate.md     →  /db:migrate       (namespaced by subdirectory)
```

### The scoping decision (highest-yield fact)

| Location | Shared with the team? | Use for |
|---|---|---|
| `.claude/commands/` | **Yes — in the repo, ships via version control** | team workflows: `/review-pr`, `/add-endpoint`, `/release-notes` |
| `~/.claude/commands/` | **No — that person's machine only** | personal habits, experiments, scratch workflows |

Same axis as the CLAUDE.md hierarchy in 3.1, tested the same way: *"a new teammate doesn't have the command"* / *"three engineers wrote the same command independently"* → it was user-scoped and belongs in `.claude/commands/`, committed to git.

The diagnostic asymmetry that makes this a good exam item: **the author never sees a problem**, because it works fine for them. Only teammates notice. Identical signature to the 3.1 "new team member isn't getting the instructions" item.

### Frontmatter and arguments

```markdown
---
description: Review a pull request against our review checklist
argument-hint: <pr-number>
allowed-tools: Read, Grep, Glob, Bash(gh pr view:*)
model: claude-opus-5
---

Review PR #$1. Fetch the diff, then check it against @.claude/rules/review-checklist.md.
```

- `$ARGUMENTS` — everything typed after the command
- `$1`, `$2`, … — positional arguments
- `@path` — inlines a file's contents into the prompt
- `!`-prefixed lines — run a bash command and inline its **output** (pre-fetching context, e.g. `!git diff --staged`)

Frontmatter keys worth knowing: `description`, `argument-hint`, `allowed-tools`, `model`.

---

## 3. Skills

### Structure

A skill is a **directory** containing `SKILL.md`:

```
.claude/skills/
  security-audit/
    SKILL.md
    references/owasp-checklist.md
```

```markdown
---
name: security-audit
description: Use when auditing code for security vulnerabilities, reviewing
  auth flows, or checking for injection risks before a release.
allowed-tools: Read, Grep, Glob
argument-hint: <path-to-audit>
context: fork
---

# Security audit procedure
1. ...
```

### The defining difference from a slash command

> **A slash command is invoked by the human. A skill is invoked by the model.**

Claude sees only skill **names and descriptions** until one is needed; when the task matches a description, the body loads. This is **progressive disclosure** — why skills cost no context until relevant.

Consequence the exam leans on: **the `description` is the trigger mechanism**, and it is the part most often wrong. A skill that never fires is almost always a description problem — it says *what the skill is* ("Security stuff") instead of *when to use it* ("Use when auditing code for security vulnerabilities, reviewing auth flows…").

**This is the same principle as Topic 2.1 tool descriptions:** the description is a prompt written for a model reader, and vague or overlapping descriptions produce unreliable selection. Recognizing that 3.2 and 2.1 share a failure mode is worth real points.

Skills can also be invoked explicitly by name when the user wants to force them, but the *design intent* is model invocation.

---

## 4. The three frontmatter options named in the objectives

### `context: fork`

Runs the skill **in an isolated sub-agent context**. The skill works in its own context window; **only its final result returns** to the main conversation.

```
NO FORK:   main context += skill body + 40 file reads + grep output + reasoning + answer
FORK:      main context += answer
```

And the mirror-image cost:

```
NO FORK:   everything the skill saw stays queryable for the rest of the session
FORK:      everything the skill saw is GONE — only the return value survives
```

**You buy context headroom by paying in detail loss.** Same contract as a subagent in Domain 1.2/1.3: isolated context in, **a string out**.

**Two distinct justifications for forking** (both named in the objectives):

1. **Volume** — verbose process, compact result (**codebase analysis**: reads 40 files, returns a 20-line architecture map).
2. **Contamination** — exploratory reasoning you don't want conditioning later turns (**brainstorming alternatives**: explores five approaches, returns a comparison; the four rejected designs genuinely disappear instead of lingering and pulling the session back toward them).

**Decision boundary — when NOT to fork:** when the *intermediate* content is the working material for the rest of the session. Forking discards everything except the summary.

> **The test:** after the skill returns, will the conversation need to interrogate what the skill *saw*, or only what it *concluded*?
> Only the conclusion → **fork**. The material itself → **don't fork**.

**Design rule for any forked skill: return findings and pointers, not payloads.** A forked codebase-analysis skill should name the 3–5 files most relevant to the change, so the main session can read those directly. The fork destroys the *bulk* while preserving *access*.

**`fork` is not "on = safer."** It costs a sub-agent round trip and destroys detail. Fork only when there is genuinely something to isolate.

### `allowed-tools`

Restricts which tools Claude may use **during that skill's execution** — a narrowing scoped to the skill.

The objective's own example: **limiting to file write operations to prevent destructive actions.** General form: a scaffolding skill that should never run `Bash`; an audit skill that should be read-only (`Read, Grep, Glob`).

**The tradeoff the exam tests:** `allowed-tools` is real configuration, so it genuinely constrains that skill — but it is **not** a **PreToolUse hook** (Topic 1.5).

- "This *skill* shouldn't be able to write files" → `allowed-tools`
- "Nothing in this repo may ever run `terraform apply`" → **hook** or `settings.json` deny rule

`allowed-tools` bounds one skill. It is not an org-wide or repo-wide guarantee, and it does not cover work outside the skill.

### `argument-hint`

Displays expected parameters when the user invokes without arguments (e.g. `argument-hint: <service-name> <environment>`). A **UX affordance for the developer**. It does not validate anything and does not constrain the model.

---

## 5. Personal customization without breaking the team

Scenario: the team ships `.claude/skills/code-review/SKILL.md`. One engineer wants a stricter variant.

**Correct answer:** create a **personal variant in `~/.claude/skills/` under a different name** (e.g. `code-review-performance`).

Two independent, separately testable reasons:

1. **`~/.claude/skills/` is not version-controlled** — the change is personal by construction, not by asking others to ignore it.
2. **A different name avoids ambiguity.** Two skills named `code-review` with near-identical descriptions is exactly the 2.1 overlapping-description problem, and it also *replaces* the shared skill for that engineer — losing team updates and losing the ability to run a normal review where the extra checks are just noise.

Wrong alternatives: editing the shared project skill (imposes a preference on everyone, plus merge conflicts), or adding the checks to `~/.claude/CLAUDE.md` (now always-loaded on every task).

---

## 6. Slash command vs skill — the decision in depth

### The deciding question

> **Who knows *when* this should run — the human, or the model?**

- The **human** knows → **slash command**. "Now I want a PR review" → `/review-pr`.
- The **model** should recognize it from the task → **skill**. "This login flow feels sketchy" → Claude reaches for `security-audit` unprompted.

### Why that's the right axis

A slash command is **deterministic invocation**: typed → runs, 100% of the time. A skill is **probabilistic invocation**: it fires when the model matches the task against the description. That is a feature (Claude applies expertise the developer didn't know to ask for), but:

> **If a workflow must run every time a certain human action occurs, do not depend on a skill to notice it. Make it a command.**

Same reasoning shape as Domain 1.4/1.5 — **guaranteed vs probabilistic** — one layer up.

### Worked contrasts

| Need | Answer | Why |
|---|---|---|
| Release notes / pre-release checklist | **command** | no natural-language cue for "the human decided to cut a release"; ceremony triggered by human decision |
| Security audit | **skill** | its value is firing when the engineer *didn't* think to ask; a command only runs when they already suspected a problem |
| Migration runbook | **skill** (fork optional) | Claude can recognize "we're altering a table" from the task; far too long for CLAUDE.md |
| Accessibility expertise for UI work | **skill** | must apply "whether or not the engineer thinks to ask" |

### They compose

```
.claude/commands/review-pr.md   ← human types /review-pr 4821
    body: "Fetch PR $1's diff. Audit it for security and performance issues."
.claude/skills/security-audit/  ← model-invoked; the command's wording triggers it
```

The command guarantees the **entry point**; the skill supplies the **expertise** and stays available when nobody typed a command. Writing command bodies so they naturally trigger the right skills is good design, not redundancy.

### Comparison table

| | Slash command | Skill |
|---|---|---|
| Invocation | human types `/name` | model matches `description` |
| Guarantee | deterministic | probabilistic |
| Unit | one `.md` file | a directory with `SKILL.md` (+ supporting files) |
| Discovery | developer must know it exists | Claude surfaces it |
| `context: fork` | ✗ (it's your prompt) | ✓ |
| Best for | ceremonies, human-triggered workflows | expertise Claude should apply on its own |
| Scoping | `.claude/…` = team · `~/.claude/…` = personal | same |

---

## 7. Choosing the mechanism — the decision the exam scores

| The situation | Answer |
|---|---|
| Applies to **every** task in the repo (style, build commands, architecture) | **CLAUDE.md** |
| Applies only to files matching a pattern | **path-scoped `.claude/rules/`** (Topic 3.3) |
| A repeatable workflow **the developer chooses** to trigger | **slash command**, project-scoped |
| A specialized capability **Claude should recognize and reach for** | **skill** |
| That capability produces noise on the way to a small answer | **skill + `context: fork`** |
| Must be impossible to violate | **hook / `settings.json` deny** (Topic 1.5) |
| One person's preference | user-scoped (`~/.claude/…`), **different name** |

---

## 8. Anti-patterns to recognize

1. **Runbooks in CLAUDE.md** — task-specific procedures loaded always → skill.
2. **User-scoped when the team needs it** — the "works on my machine" configuration bug → commit to `.claude/`.
3. **A skill description that describes rather than triggers** — "Testing helper" never fires; "Use when writing or fixing tests for the API layer…" does.
4. **Forking a skill whose intermediate output you need** — you get the summary and lose the material.
5. **Forking a write-heavy generation skill** — its value is edits you iterate on conversationally; a report *about* files is worse than the files.
6. **Returning full file contents from a forked skill** — pays the round trip and lands in the same place as not forking.
7. **Treating `allowed-tools` as a security guarantee** — it scopes one skill; guarantees are hooks.
8. **Treating `context: fork` as a sandbox** — it isolates *context*, not *side effects*.
9. **Editing a shared skill for a personal preference** instead of a renamed variant in `~/.claude/skills/`.
10. **Duplicating a skill's name** — selection ambiguity, same as overlapping tool descriptions (2.1).
11. **Command where a skill was needed** (nobody runs it) or **skill where a command was needed** (it runs only sometimes).
12. **Command/skill where a hook was needed** — both mechanisms here are **affordances**, not enforcement.

---

## 9. Quick recall

- **Commands = human-invoked; skills = model-invoked (the `description` is the trigger).**
- **`.claude/…` = shared via git; `~/.claude/…` = personal, invisible to teammates.**
- **`context: fork`** = verbose process, compact result → isolate it; returns a string, discards the rest.
- **`allowed-tools`** = scoped restriction for one skill, *not* an enforcement guarantee.
- **`argument-hint`** = prompts the developer for parameters.
- **Skill vs CLAUDE.md** = on-demand task-specific vs always-loaded universal.

---
---

# Practice Questions

## Question 1 — where a task-specific runbook belongs

*Scenario 2: Code Generation with Claude Code.*

A platform team maintains a repository used by twelve engineers. Six months ago a senior engineer wrote a detailed database-migration runbook — roughly 350 lines covering pre-flight checks, backfill batching strategy, rollback procedure, and post-migration verification queries — and added it to the project's root `CLAUDE.md` so the whole team would have it.

Migrations happen about twice a month. Engineers have recently reported two problems: sessions hit context limits faster than they used to, and Claude has started referencing migration terminology during unrelated frontend work.

Which change best addresses both problems?

- **A.** Move the runbook into `.claude/rules/migrations.md` and add `@.claude/rules/migrations.md` to the root `CLAUDE.md`, keeping the main file short.
- **B.** Move the runbook into `.claude/skills/database-migration/SKILL.md` with a description explaining when to use it, and remove it from `CLAUDE.md`.
- **C.** Create `.claude/commands/migrate.md` containing the runbook, and remove it from `CLAUDE.md`, instructing the team to type `/migrate` before starting any migration work.
- **D.** Keep the runbook in `CLAUDE.md` but wrap it in an explicit conditional instruction: "The following applies ONLY when performing database migrations — ignore it otherwise."

### Correct answer: **B**

**Why B is right.** Two symptoms, one root cause: **task-specific content in an always-loaded file.**

- *Context exhaustion* — 350 lines paid on every turn of every session by all twelve engineers, for a procedure used twice a month.
- *Bleed into unrelated work* — content in context is content the model attends to; migration vocabulary present during frontend work is attention dilution, not a wording bug.

A skill fixes both by construction: only **name and description** are visible until the task matches, then the body loads. This is the objective's exact framing — **skills = on-demand task-specific; CLAUDE.md = always-loaded universal**. The description also lets Claude recognize migration work from the *task* ("add a nullable column and backfill it") even when the engineer didn't label it a migration.

**Why A is wrong.** `@import` **does not do conditional loading** — the highest-value trap in the item, hinging on Topic 3.1. `@import` resolves at load time and pulls contents into context. The root `CLAUDE.md` *looks* short, but the 350 lines still load on every turn. `@import` buys **editing modularity**, not **token savings**; both symptoms survive. (Near-miss: `.claude/rules/migrations.md` with a `paths:` glob *would* load conditionally — Topic 3.3 — but this option imports it unconditionally, and migrations aren't cleanly identified by a file glob anyway.)

**Why C is wrong.** Plausible — it does fix the token cost — but invocation is wrong. A slash command is deterministic *only if a human remembers to type it*, and the runbook's value lies largely in pre-flight checks and rollback planning, exactly what an engineer under-weights when skipping the runbook. Engineers who don't classify their change as "a migration" never type `/migrate`. Commands suit **human-triggered ceremonies with an obvious cue**; this needs model recognition.

**Why D is wrong.** Solves neither problem. Tokens still load (zero reduction), so context exhaustion is untouched; and "ignore this unless…" is **prompt-based guidance over content that is physically present** — probabilistic, and useless against attention dilution since the model must read the section to decide it doesn't apply. Classic 3.1 anti-pattern: stronger wording where the fix is structural.

---

## Question 2 — configuring a codebase-analysis skill

*Scenario 4: Developer Productivity with Claude.*

A team builds a skill that helps engineers understand unfamiliar parts of a large monorepo. During execution it globs for entry points, greps for symbol definitions across ~2,000 files, and reads roughly 30–50 source files before producing its output.

The team is deciding how to configure it. Engineers use it at the *start* of a task and then spend the next hour implementing changes in the code it surveyed.

Which configuration best supports that workflow?

- **A.** Set `context: fork` and have the skill return a module map, the data flow between components, and the specific file paths most relevant to the requested change.
- **B.** Set `context: fork` and have the skill return the full contents of every file it read, so the implementation phase has complete information.
- **C.** Omit `context: fork` so the surveyed files remain in the main conversation, and set `allowed-tools: Read, Grep, Glob` to keep the skill read-only.
- **D.** Omit `context: fork`, and instead instruct engineers to invoke the skill in a separate Claude Code session, then paste its output into their working session.

### Correct answer: **A**

**Why A is right.** The canonical `context: fork` case, and the objectives' own example — *"isolate skills that produce verbose output (e.g., codebase analysis)."* The shape fits exactly: **enormous process, compact product.** Fifty file reads plus greps across 2,000 files is tens of thousands of tokens; the useful residue is a module map and a short list of paths. Fork keeps the survey in the sub-agent's window so the engineer's *next hour of implementation* gets the headroom.

What makes A correct rather than merely fork-shaped is the second half: **it returns findings and pointers, not payloads.** Naming the files most relevant to the change lets the main session `Read` those few directly. Fork discarded the *bulk* while preserving *access*.

**Why B is wrong.** Forks, then defeats the fork. Returning every file's contents pushes the whole payload across the boundary into main context — you pay the sub-agent round trip and land where you'd be without forking, minus the ability to ask follow-ups inside the survey. A return value is not a place to smuggle the process back in.

**Why C is wrong.** Right instinct, wrong question. `allowed-tools: Read, Grep, Glob` is good hygiene for an analysis skill, but it bounds *what the skill may do*, not *where its output lands*. Omitting fork leaves 30–50 files plus every grep result permanently in the main window — precisely the exhaustion to be prevented. Read-only and context-isolated are independent settings.

**Why D is wrong.** Solves it by hand and loses the pointers. A second session does isolate noise, but converts a *configuration* problem into a *manual process* problem (remember, switch, copy-paste) when `context: fork` is the built-in mechanism. It's also strictly worse operationally: the survey session knows nothing about the working session's task, so its output can't be targeted at the change being made.

---

## Question 3 — personal customization of a shared skill

*Scenario 2: Code Generation with Claude Code.*

A team ships a project-scoped skill at `.claude/skills/code-review/SKILL.md`, committed to the repository and used by all eight engineers.

One engineer, who owns the performance-critical rendering pipeline, wants their reviews to additionally check for unnecessary re-renders, allocation in hot paths, and unbounded list rendering. The rest of the team does not want these checks — they generate noise on the CRUD services that make up most of the codebase.

Which approach best satisfies the engineer without affecting teammates? **(Select ONE.)**

- **A.** Add the performance checks to the existing `.claude/skills/code-review/SKILL.md`, with an instruction that they apply only to rendering-pipeline files.
- **B.** Create `~/.claude/skills/code-review/SKILL.md` containing the extended version, so the personal copy takes precedence over the project one.
- **C.** Create `~/.claude/skills/code-review-performance/SKILL.md` with a description scoped to rendering and performance-sensitive review work.
- **D.** Add the performance checks to the engineer's `~/.claude/CLAUDE.md` so they are applied automatically during that engineer's sessions.

### Correct answer: **C**

**Why C is right.** It satisfies both halves independently, which is what makes it robust:

1. **`~/.claude/skills/` is not version-controlled** — personal *by construction*, not by asking others to ignore it. Same axis as user- vs project-level CLAUDE.md (3.1) and user- vs project-scoped commands.
2. **A distinct name avoids ambiguity** — stated explicitly in the objectives: *"creating personal variants in `~/.claude/skills/` with different names to avoid affecting teammates."*

Description scoping does the rest: skills are **model-invoked**, so a description bounded to rendering/performance-sensitive review lets Claude pick the strict variant on pipeline work and the shared one elsewhere, with nothing for the engineer to remember.

**Why A is wrong.** Imposes a personal preference on eight people — it's committed, so everyone gets it. The "only applies to rendering files" qualifier is **prose-based conditional loading**: probabilistic and token-costly (same failure as Q1/D). Conventions that attach to a set of files belong in a **path-scoped `.claude/rules/` file** (Topic 3.3), not a sentence inside a shared skill. It also creates a maintenance conflict with future updates to the shared skill.

**Why B is wrong.** Right directory, wrong name — the strongest distractor. The location correctly keeps it off other machines, but the **duplicate name** creates two `code-review` skills with near-identical descriptions: at best resolution depends on precedence semantics the engineer is betting on rather than controlling. Worse, the engineer has **replaced** the shared skill for themselves — losing team updates *and* the ability to run a normal review on CRUD services, where the extra checks are the very noise they agreed to avoid. Underlying issue is Topic 2.1: **overlapping descriptions degrade selection.**

**Why D is wrong.** Correctly scoped to the individual, but `~/.claude/CLAUDE.md` is **always loaded** — performance-review criteria would sit in context during frontend work, CRUD work, debugging, and documentation, for every task. Q1's anti-pattern at user scope. Review criteria are task-specific and belong in an on-demand skill.

---

## Question 4 — preventing a destructive command

*Scenario 4: Developer Productivity with Claude.*

A platform team maintains an infrastructure repository. They build a skill that inspects Terraform state and configuration to explain what a proposed change would affect — which resources are touched, which are replaced, and what the blast radius is.

During testing, an engineer reports that on one occasion the skill ran `terraform apply` while investigating, modifying live infrastructure. The team needs to prevent this.

The lead proposes adding to the skill body: **"You are an analysis skill. NEVER run `terraform apply`, `terraform destroy`, or any state-modifying command. Only inspect."**

Which assessment is most accurate?

- **A.** The instruction is sufficient, because skill bodies are loaded directly into context and instructions in a skill body are followed more reliably than instructions in CLAUDE.md.
- **B.** The instruction should be replaced with `allowed-tools: Read, Grep, Glob` in the skill frontmatter, which fully solves the problem for this repository.
- **C.** `allowed-tools` should be set on the skill to constrain it, but if `terraform apply` must never run in this repository under any circumstances, a PreToolUse hook or a `settings.json` deny rule is required.
- **D.** The skill should be given `context: fork`, so any commands it runs execute in an isolated sub-agent context and cannot affect the main session's environment.

### Correct answer: **C**

**Why C is right.** It gets two things right that the others each get half of.

**First — configuration beats prose.** `allowed-tools` is a real constraint on the skill's execution; the lead's sentence is a request. This is Topic 1.5's distinction one layer up: **deterministic mechanisms vs probabilistic instructions.** When the requirement is "must not happen," a sentence is the wrong instrument regardless of wording.

**Second — `allowed-tools` is scoped to the skill, so it is not a repository guarantee.** The stem asks about `terraform apply` running *at all*. `allowed-tools` bounds one skill and nothing else: a normal session, a different skill, a slash command, or a spawned subagent is unconstrained. Repository-wide invariants live where the harness enforces them — a **PreToolUse hook** or a **`settings.json` deny rule**.

Both belong: `allowed-tools` narrows the skill (right-sized, local, self-documenting); the hook makes the guarantee (global, unconditional). Matching each mechanism to the scope of the requirement it can actually cover is the judgment tested.

**Why A is wrong.** Invents a reliability hierarchy that doesn't exist. Skill bodies and CLAUDE.md are both text in the context window; neither is enforced, and compliance with either is probabilistic. The stem's own evidence refutes it — the skill already ran `apply` once, which is what "non-zero failure rate" looks like in production. **An option whose fix is "say it more forcefully / in a better place" is almost always the distractor.**

**Why B is wrong.** Right mechanism, overclaimed scope — the strongest distractor, since the frontmatter is exactly what the objective prescribes (*"limiting to file write operations to prevent destructive actions"*). The failure is **"fully solves the problem for this repository."** It solves it for one skill; everything outside can still run `apply`. Watch for scope overclaims in options that name the right tool.

**Why D is wrong.** Conflates **context isolation** with **side-effect isolation** — the most important misconception in the item. `context: fork` isolates the **conversation context**; it does **not** sandbox execution. A forked sub-agent runs real `Bash` against the real filesystem and real cloud account, so `terraform apply` inside a fork destroys production just as thoroughly — and you'd see *less* of it, since only the summary returns. Fork is a **context-budget** tool, never a **safety** tool.

---

## Question 5 — command vs skill for two different needs

*Scenario 2: Code Generation with Claude Code.*

A team has a mandatory pre-release checklist: verify the changelog is updated, confirm migrations have a rollback path, check that no feature flags are left hard-coded, and regenerate the API docs. It runs when a release captain decides to cut a release — roughly weekly.

They also have a body of accessibility expertise — ARIA patterns, focus management, contrast requirements — that they want applied whenever anyone touches UI components, whether or not the engineer thinks to ask about accessibility.

Which configuration best fits both needs? **(Select ONE.)**

- **A.** Make the pre-release checklist a project-scoped slash command in `.claude/commands/`, and the accessibility expertise a project-scoped skill in `.claude/skills/` with a description covering UI component work.
- **B.** Make both project-scoped skills in `.claude/skills/`, so Claude can invoke either one when it detects the relevant context.
- **C.** Make both project-scoped slash commands in `.claude/commands/`, so engineers explicitly control when each one runs.
- **D.** Make the pre-release checklist a project-scoped skill with `context: fork` to isolate its verbose verification output, and put the accessibility expertise in the root `CLAUDE.md` so it is always available.

### Correct answer: **A**

**Why A is right.** The two halves have opposite invocation ownership.

**Pre-release checklist → slash command.** The trigger is *"a release captain decides to cut a release"* — a human decision with no reliable signal in the task text. "Mandatory" means invocation must be **deterministic**: typed → runs, every time.

**Accessibility expertise → skill.** The requirement is explicit: applied *"whether or not the engineer thinks to ask."* A command only runs when someone already has accessibility in mind — exactly the case where the reminder isn't needed. A skill is **model-invoked**, so a description covering UI component work applies the expertise while the engineer is thinking only about the feature.

Both project-scoped in `.claude/…` so they ship via version control.

Reusable question: **who knows when this should run — the human, or the model?**

**Why B is wrong.** Makes a mandatory ritual probabilistic. Skill invocation depends on matching a description against the task, and there is no dependable cue for "we are cutting a release," so the checklist fires inconsistently. A checklist that runs *most* of the time is the worst outcome — the team stops verifying manually while still missing steps.

**Why C is wrong.** Checklist is right; accessibility is inverted. `/accessibility-check` only runs when the engineer already knows to run it, dropping exactly the cases the team wants caught. "Whether or not the engineer thinks to ask" is a direct instruction to choose model invocation.

**Why D is wrong.** Both halves fail, for reasons from earlier questions.
- Checklist as a **forked skill** stacks two errors: invocation isn't guaranteed (as in B), and `context: fork` is misapplied — checklist output is short and is the material the release captain reviews line by line. Fork is for **verbose process, compact result**.
- Accessibility in **root `CLAUDE.md`** is Q1's anti-pattern: always-loaded task-specific content sitting in context during backend work, migrations, and CI debugging. "Always available" is the seductive phrase; the correct reading is that a skill *is* always available and only *loads* when relevant.

---
---

# Clarifications from follow-up discussion

### Two distinct reasons to fork (not one)

- **Volume** — the process is huge and the product is small (codebase analysis).
- **Contamination** — provisional reasoning about options you will *not* take (brainstorming). Left in the main window, rejected designs keep conditioning later turns and the model may drift back toward them.

Either signal in a question stem justifies `context: fork`.

### Worked fork examples

**Fork correct — architecture analysis.** Internally reads ~40 files; returns ten lines: module map, data flow, and the 3–5 files most relevant to the requested change. The engineer's next hour of implementation gets the context budget, and the named files can be read directly if needed.

**Fork wrong — spec review.** A skill that reads a 60-page requirements document *looks* like the fork case (big input, small output), but the engineer will ask a dozen follow-ups against the spec ("what does §4.3 say about partial refunds?"). Forking optimized a token count at the cost of the actual workflow. The intermediate content **is** the working material.

**Fork wrong — scaffolding/generation.** A skill that writes routes, handlers, and tests produces value in the *edits*, which the developer reviews and iterates on conversationally ("make the validation stricter"). Forking returns a report *about* files instead of code you can immediately refine. Fork suits **read-heavy investigation**, not **write-heavy generation you'll iterate on**.

**Fork pointless — already-small output.** A lint check or version bump has nothing to isolate; forking adds a round trip for no gain.

### Fork decision table

| Signal | Fork? |
|---|---|
| Reads/searches many files, returns a summary | ✅ |
| Explores options you'll mostly discard | ✅ |
| Runs long noisy commands, you need pass/fail + failures | ✅ |
| Output is consulted repeatedly for the rest of the session | ❌ |
| Skill writes/edits code you'll iterate on conversationally | ❌ |
| Output is already small | ❌ |

### The three command/skill errors to spot in a stem

1. **Skill where a command was needed** — "the pre-deploy checklist runs *sometimes*." Non-deterministic invocation for a mandatory ritual.
2. **Command where a skill was needed** — "we wrote `/security-audit` but nobody runs it." Only fires when someone already suspected the problem.
3. **Command/skill where a hook was needed** — "must run before *every* deploy, no exceptions." Neither mechanism in this topic guarantees anything; that is a **PreToolUse hook** (1.5).

### Cross-topic links worth carrying forward

- **2.1 (tool descriptions) ≡ 3.2 (skill descriptions)** — both are prompts written for a model reader; vague or overlapping descriptions cause unreliable selection.
- **1.2/1.3 (subagents) ≡ `context: fork`** — isolated context in, a string out.
- **1.4/1.5 (enforcement)** — commands and skills are **affordances**; hooks and `settings.json` deny rules are **guarantees**.
- **3.1 (`@import`)** — modularity for editing, *not* conditional loading; contrast with 3.3 path-scoped rules, which genuinely load conditionally.
