# Topic 3.3 — Apply path-specific rules for conditional convention loading

- **Domain:** 3 — Claude Code Configuration & Workflows (20% of exam)
- **Topic number:** 3.3 (Topic 15 of 30 overall)
- **Exam scenarios that draw on this:** Scenario 2 (Code Generation with Claude Code), Scenario 4 (Developer Productivity with Claude)

---

## 1. The problem this mechanism solves

Topic 3.1 gave us a hierarchy that scopes by **location**: root `CLAUDE.md` is universal; a subdirectory `CLAUDE.md` applies when working in that subtree.

Location is the wrong axis for a large class of real conventions:

- Test conventions for `**/*.test.tsx` — test files sit *next to* the source they cover, scattered across dozens of directories.
- DB migration rules — `db/migrations/` in one service, `packages/*/migrations/` in another.
- `.proto` files, `*.stories.tsx`, `*.tf`, generated files.

Two bad ways to cope, both of which the exam uses as distractors:

| Bad option | Failure |
|---|---|
| Copy the convention into every relevant subdirectory `CLAUDE.md` | **Duplication → drift.** Some copies get updated, some don't; a newly added directory silently has none. |
| Put it in root `CLAUDE.md` | **Always-loaded.** Token cost paid on every request in every session, plus **instruction dilution** — a wall of irrelevant rules lowers adherence to the rules that *do* apply now. |

**Path-specific rules scope by file pattern instead of by location.** Defined once, activated only when a matching file is in play.

---

## 2. The mechanism

A file in `.claude/rules/` with a YAML frontmatter `paths` field holding glob patterns:

```markdown
---
paths: ["**/*.test.tsx", "**/*.test.ts"]
---

# Test conventions

- Use React Testing Library; never `enzyme`.
- One `describe` per exported symbol.
- No snapshot tests for anything with conditional rendering.
- Mock at the network boundary with MSW, never by stubbing hooks.
```

```markdown
---
paths: ["terraform/**/*"]
---

# Terraform conventions

- Every resource carries the standard tag block (`owner`, `cost-center`, `env`).
- Never inline credentials; read from the `aws_secretsmanager_secret` data source.
- State changes go through the PR plan check — no local `terraform apply`.
```

### Semantics

- **Conditional activation.** Content is injected only when Claude reads or edits a file matching one of the globs. Otherwise it costs nothing.
- **Just-in-time delivery.** The convention arrives *with* the relevant file, at the moment it matters — adherence tends to be better than for the same text buried in a 400-line root `CLAUDE.md`.
- **Additive, not replacing.** Path rules layer on top of `CLAUDE.md` and on top of other matching rules. Multiple rules can match one file simultaneously (`**/*.tsx` and `**/*.test.tsx` both fire for a test component).
- **Project-level and version-controlled.** `.claude/rules/` lives in the repo → shared with the team. Same sharing axis as 3.1's `~/.claude/` vs project distinction.

### The `paths`-less variant (easy to confuse — know both)

A `.claude/rules/` file **without** a `paths` field is simply **always-loaded**. That is the 3.1 use case: splitting a monolithic `CLAUDE.md` into `testing.md`, `api-conventions.md`, `deployment.md`.

> **The `paths` frontmatter is what turns a modular rule file into a *conditional* one.** Same directory, two different jobs.

---

## 3. Glob mechanics

| Pattern | Matches |
|---|---|
| `**/*.test.tsx` | test files at **any** depth, anywhere |
| `*.test.tsx` | only at the project root — almost always a bug |
| `terraform/**/*` | everything under `terraform/`, any depth |
| `src/*.ts` | `.ts` files directly in `src/`, **not** `src/lib/` |
| `["**/*.test.ts", "**/*.spec.ts"]` | OR across the array |

Two high-yield facts:

1. **`**` crosses directory boundaries; `*` does not.**
2. **Patterns are anchored at the project root.** Omitting the leading `**/` is the #1 cause of "I wrote the rule and Claude ignores it."

**A non-matching glob fails silently** — no error, no warning, the model just doesn't follow the convention. Diagnostic path (from 3.1): run **`/memory`** to see which memory/rule files are actually loaded, then check anchoring.

---

## 4. The decision boundary — the core of what is tested

Ask: **does this convention correlate with a location, or with a kind of file?**

| Convention correlates with… | Use | Example |
|---|---|---|
| Everything, always | root `CLAUDE.md` | build command, PR conventions, "never commit secrets" |
| A coherent subtree / domain | subdirectory `CLAUDE.md` | `services/payments/CLAUDE.md` — package with its own stack, owners, idioms |
| A file **type** spread across the tree | `.claude/rules/*.md` **with `paths`** | `**/*.test.tsx`, `**/*.tf`, `**/migrations/*.sql` |
| A modular chunk that must always apply | `.claude/rules/*.md` **without `paths`** | split-up universal standards |
| A multi-step **procedure** invoked on demand | a **skill** (3.2) | "cut a release", "add a new API endpoint" |
| A hard, non-negotiable constraint | **hook / permission** | block writes outside `src/` |

### Three boundaries to hold precisely

**Path rules vs. subdirectory `CLAUDE.md`.** Both are conditional. Path rules when the files are *scattered*, or when the convention is about a file's **kind** rather than its neighbourhood. Subdirectory `CLAUDE.md` when the subtree itself is the unit of meaning. When a convention is genuinely confined to one directory (`terraform/`), either works — path rules are still often preferred because all conventions stay discoverable in one `.claude/rules/` directory instead of scattering `CLAUDE.md` files around the repo.

**Rules vs. skills.** Rules are *declarative constraints* that apply passively while editing matching files — you don't invoke them and can't forget them. Skills are *procedures* the model invokes when a task calls for them. "Test files use MSW" is a rule. "Scaffold a new test suite" is a skill.

> Path scoping matches **file kinds**. It can never approximate **task intent** — that is what skill invocation is for.

**Rules vs. enforcement.** A path rule is prompt-based guidance: it raises the probability of compliance, it does not guarantee it. Anything that must be *guaranteed* belongs in a **PreToolUse hook** or a permission rule. Same "match the fix to its layer" principle as 1.5 (hooks) and 2.3 (`tool_choice`), and the exam reuses it constantly.

---

## 5. Anti-patterns

1. **Path-scoping something universal.** Gating "never log PII" behind `paths: ["**/*.ts"]` means it's absent the moment Claude edits a `.py` file or a config.
2. **Over-broad globs.** `paths: ["**/*"]` is an always-loaded rule with extra ceremony — complexity paid, savings zero, and it misleads the next reader.
3. **Wrong anchoring.** `*.test.tsx` instead of `**/*.test.tsx`. Fails silently.
4. **Duplicating one convention across many subdirectory `CLAUDE.md` files** instead of one path rule. Guarantees drift.
5. **Putting workflows in rules.** A 900-line release runbook loaded whenever a `.ts` file opens = slow session starts + instruction dilution. Make it a skill.
6. **Treating rules as a security control.** Guidance ≠ enforcement.
7. **Contradictory overlapping rules.** Matching rules are additive, so two rules both matching `**/*.test.tsx` and asserting opposites produce ambiguous behaviour. Design rule scopes to be non-overlapping in what they *assert*, not merely in what they match.

---

## 6. Worked scenario

Monorepo, ~800 files:

- Universal build/test commands, commit format → **root `CLAUDE.md`**.
- `packages/payments/` has its own domain model and compliance constraints → **`packages/payments/CLAUDE.md`**, using `@import ./standards/pci.md` (3.1) to stay modular.
- Test conventions for ~180 test files spread across every package → **`.claude/rules/testing.md`**, `paths: ["**/*.test.ts", "**/*.test.tsx"]`.
- Terraform in `infra/` → **`.claude/rules/terraform.md`**, `paths: ["infra/**/*.tf"]`.
- "Never edit files under `generated/`" must be *guaranteed* → **PreToolUse hook**, not a rule.

Result: editing a React component loads root `CLAUDE.md` only. Editing that component's test additionally loads `testing.md`. Nobody pays for Terraform conventions unless they're in the Terraform.

---

# Practice Questions

## Question 1 — duplication across a monorepo

A fintech company has a TypeScript monorepo with 14 packages. Test files live alongside the source they cover (`packages/*/src/**/*.test.ts`), so ~200 test files are spread across every package. Agreed conventions: mock only at the network boundary with MSW, no snapshot tests for conditionally-rendered output, one `describe` block per exported symbol.

The tech lead maintains this by copying an identical "Testing" section into each package's `CLAUDE.md`. Three packages have drifted out of sync; a newly added package has no testing section at all.

Which approach best resolves this?

- **A.** Move the testing conventions into the root `CLAUDE.md` so all 14 packages inherit them from a single definition.
- **B.** Create `.claude/rules/testing.md` with frontmatter `paths: ["**/*.test.ts", "**/*.test.tsx"]` and delete the duplicated sections from the package `CLAUDE.md` files.
- **C.** Create `.claude/rules/testing.md` with no `paths` frontmatter, and add `@import ../../.claude/rules/testing.md` to each package's `CLAUDE.md`.
- **D.** Create a `testing` skill containing the conventions, and add a line to the root `CLAUDE.md` instructing Claude to invoke it before editing any test file.

**Correct answer: B.** *(answered correctly)*

- **Why B:** The convention correlates with a *file kind*, not a location — tests are scattered and sit next to their source. A glob-scoped rule is defined once (kills drift and the missing-package problem) and loads only when a matching file is in the working set (kills token cost for the majority of sessions). This is the exam guide's own bullet: glob-pattern rules beat directory-level `CLAUDE.md` for conventions spanning multiple directories.
- **Why not A:** Fixes duplication, not cost. Root `CLAUDE.md` is always loaded, so every request in every package pays for test conventions, and the always-loaded instruction set gets diluted. Right on dedup, wrong on conditional loading.
- **Why not C:** Two faults. A `.claude/rules/` file with no `paths` is *already* always-loaded, so the imports are redundant — and they reintroduce the per-package maintenance burden (14 import lines; the new package will forget one) that caused the drift.
- **Why not D:** Confuses rules with skills. These are passive constraints that must apply whenever a test file is edited, with no chance to forget. Routing them through skill invocation replaces a deterministic path match with a probabilistic decision, for no benefit.

---

## Question 2 — the silent-failure diagnostic

A platform team adds `.claude/rules/terraform.md`:

```markdown
---
paths: ["*.tf"]
---
# Terraform conventions
- Every resource carries the standard tag block.
- Never inline credentials; read from Secrets Manager.
```

Terraform lives at `infra/environments/prod/main.tf`, `infra/environments/staging/main.tf`, `infra/modules/vpc/main.tf`. After merge, engineers report Claude still generates untagged resources when editing those files. No errors appear; `/memory` shows the project `CLAUDE.md` loading normally.

Most likely root cause?

- **A.** Path-specific rules only activate on file *reads*, not writes, so generated resources bypass the rule.
- **B.** The `paths` glob is anchored at the project root and `*` does not cross directory boundaries, so no Terraform file matches it.
- **C.** `.claude/rules/` files require a corresponding `@import` in the root `CLAUDE.md` before they are eligible to load.
- **D.** Rules in `.claude/rules/` are overridden by the nearest directory-level `CLAUDE.md`, and `infra/` has one.

**Correct answer: B.** *(answered correctly)*

- **Why B:** Globs are anchored at the project root, and a single `*` matches within one path segment. `*.tf` matches `main.tf` *at the repo root* and nothing else; every real file is three-plus segments deep. The rule loads for zero files. Fix: `["**/*.tf"]` or `["infra/**/*.tf"]`. The tell in the question is **no error, no complaint, just non-compliance** — the signature of a non-matching glob.
- **Why not A:** Rules activate when a matching file enters the working set, read *or* edit — edits are precisely the case the conventions exist to govern.
- **Why not C:** Inverts the mechanisms. `@import` pulls external files into a `CLAUDE.md`; `.claude/rules/` files are discovered on their own. (Plausible trap for someone half-remembering 3.1.)
- **Why not D:** Memory and rule files merge **additively**. Narrower scope wins only where instructions directly conflict; it never prevents a rule from loading. The scenario also never establishes that `infra/CLAUDE.md` exists.

---

## Question 3 — assigning four conventions to four layers

Four conventions need a home:

1. "Never commit files containing API keys" — every file, every session.
2. "`services/billing/` uses hexagonal architecture; adapters in `adapters/`, never `domain/`" — that package's whole subtree, own stack and owners.
3. "Files under `generated/` must never be edited" — hard requirement; last incident cost a day.
4. "GraphQL resolvers (`**/*.resolver.ts`, spread across 9 services) must not perform N+1 queries; use the DataLoader helper."

Which assignment is correct?

- **A.** 1 → root `CLAUDE.md`; 2 → `services/billing/CLAUDE.md`; 3 → PreToolUse hook; 4 → `.claude/rules/` with `paths: ["**/*.resolver.ts"]`
- **B.** 1 → root `CLAUDE.md`; 2 → `.claude/rules/` with `paths: ["services/billing/**/*"]`; 3 → `.claude/rules/` with `paths: ["generated/**/*"]`; 4 → root `CLAUDE.md`
- **C.** 1 → `.claude/rules/` with `paths: ["**/*"]`; 2 → `services/billing/CLAUDE.md`; 3 → PreToolUse hook; 4 → a `resolvers` skill invoked when editing resolvers
- **D.** 1 → root `CLAUDE.md`; 2 → `services/billing/CLAUDE.md`; 3 → `.claude/rules/` with `paths: ["generated/**/*"]`; 4 → `.claude/rules/` with `paths: ["**/*.resolver.ts"]`

**Correct answer: A.** *(answered correctly)*

- **Why A:** Each item lands at the layer matching its scope *and* its required strength. (1) universal invariant → always-loaded; path-scoping it would leave it absent exactly when a secret slips through. (2) location-coherent subtree with own stack/ownership → subdirectory `CLAUDE.md`. (3) "hard requirement" + prior incident → **enforcement**, so PreToolUse hook. (4) a file *kind* scattered across 9 services → glob-scoped rule.
- **Why not B:** Puts the `generated/` prohibition in a rule (guidance for something stated as a hard requirement), and puts resolver conventions in root `CLAUDE.md` (always-loaded for a narrow slice of files). Its handling of (2) is defensible alone; the other two errors sink it.
- **Why not C:** `paths: ["**/*"]` is an always-loaded rule with extra ceremony, and it makes a universal invariant *look* conditional. It also routes (4) through a skill — resolver conventions are passive constraints, not a procedure. Gets (2) and (3) right, which makes it the strongest distractor.
- **Why not D:** Identical to A except (3), where a path rule replaces the hook. That is the single most-tested distinction here: **guidance vs. enforcement.** "Must never" + a real incident means the constraint has to hold even when the model is confused or working from a stale plan; a rule can't promise that.

---

## Question 4 — mis-housed content (missed on first attempt)

A staff engineer audits `.claude/rules/` and finds six files. Three have `paths`; three do not. The unscoped three are `commit-conventions.md`, `build-commands.md`, and `release-runbook.md` — the last a **900-line, 14-step** procedure for cutting a production release, performed roughly twice a month.

Sessions are consistently slow to start, and engineers report Claude sometimes ignores conventions it clearly has access to.

Which change most directly addresses the reported problems?

- **A.** Add `paths: ["**/*"]` to the three unscoped rule files so all six load through the same mechanism.
- **B.** Convert `release-runbook.md` into a skill, leaving `commit-conventions.md` and `build-commands.md` unscoped.
- **C.** Add `paths` frontmatter to all three unscoped files, scoping `release-runbook.md` to `["**/CHANGELOG.md", "**/package.json"]`.
- **D.** Merge all six rule files back into the root `CLAUDE.md` so the configuration has a single source of truth.

**Correct answer: B.** *(answered C — see below)*

- **Why B:** Two symptoms, one cause. A 900-line always-loaded runbook costs tokens at every session start (slow starts) and carries release-procedure text into every turn (instruction dilution → ignored conventions). The fix follows from *what kind of content it is*: a 14-step procedure run twice a month is a **workflow**, not a convention → a skill. `commit-conventions.md` and `build-commands.md` are genuinely universal and small; they correctly stay unscoped.
- **Why not C (the answer I chose):** It correctly identifies the runbook as the problem but applies the **wrong remedy**. Scoping a release procedure to `**/CHANGELOG.md` / `**/package.json` makes activation depend on which files happen to be touched, which barely correlates with "the engineer is cutting a release." Both failure directions occur: it loads during an unrelated dependency bump, and it's absent when a release starts elsewhere. **Path scoping matches file kinds; it cannot approximate task intent.** C also needlessly scopes the two small universal files, risking their absence when they should apply.
- **Why not A:** `**/*` matches everything, so all three stay always-loaded. Nothing changes except the config now *looks* conditional while behaving universally — arguably worse for the next reader.
- **Why not D:** Moves 900 lines into the always-loaded file and deletes the modularity — both symptoms get worse. "Single source of truth" is real, but `.claude/rules/` already gives one definition per convention; these aren't duplicates.

---

## Question 5 — constraints vs. procedure in one system

A design-systems team maintains a shared React component library consumed by four product teams. Two needs:

- **(i)** Every component file (`packages/ui/src/**/*.tsx`, ~120 files) must use design tokens from `@acme/tokens` rather than raw hex, must forward `ref`, and must not import from `packages/ui/src/internal/`.
- **(ii)** Adding a *new* component is an eight-step process (scaffold directory, register export in `index.ts`, add Storybook story, add visual-regression baseline, add changelog entry, …). Happens ~once a fortnight; developers frequently forget steps 4 and 5.

Which configuration handles both correctly?

- **A.** Put (i) and (ii) together in `.claude/rules/design-system.md` with `paths: ["packages/ui/src/**/*.tsx"]`.
- **B.** Put (i) in `.claude/rules/design-system.md` with `paths: ["packages/ui/src/**/*.tsx"]`; make (ii) a skill with an `argument-hint` for the component name.
- **C.** Put (i) in `packages/ui/CLAUDE.md`; make (ii) a skill with an `argument-hint` for the component name.
- **D.** Make (i) a skill that Claude invokes when it detects a component edit; put (ii) in `.claude/rules/design-system.md` with `paths: ["packages/ui/src/index.ts"]`.

**Correct answer: B.** *(answered correctly)*

- **Why B:** (i) is a set of **passive constraints** — properties that must hold every time a component file is touched, for any task, with no invocation moment and no chance to forget. Glob scoping on the component `.tsx` files matches the exact file kind and costs nothing in sessions that never open the library. (ii) is a **procedure** with a start and an end, run twice a month; loading eight steps into every padding fix is waste, and the stated failure ("forget steps 4 and 5") is exactly what a skill fixes — the whole checklist arrives at once, when the task starts. `argument-hint` (3.2) prompts for the component name when invoked without arguments.
- **Why not C (second-best):** Gets (ii) right, and its instinct on (i) — the library is a coherent owned subtree — isn't unreasonable. But `packages/ui/CLAUDE.md` loads for the *entire package*: build config, test setup, and critically `packages/ui/src/internal/` itself, where "never import from `internal/`" is incoherent. Glob scoping expresses "component files" precisely; directory scoping only approximates it. When both are available, pick the one matching what the convention is actually about.
- **Why not A:** Correctly scopes (i) but drags the eight-step runbook with it, so the procedure loads on every component edit — the Q4 waste pattern, just gated behind a glob. It also weakens (ii): a checklist sitting in ambient context during an unrelated edit ≠ a skill invoked for the task.
- **Why not D:** Inverts both. Constraints become a skill (applied only if the model *decides* to invoke it — probabilistic where it must be unconditional), and the procedure is path-scoped to `index.ts`, betting the developer happens to open that file when starting a new component. Q4's error again: **path scoping matches file kinds, never task intent.**

---

# Clarifications & retained takeaways

- **`.claude/rules/` has two modes.** With `paths` → conditional. Without `paths` → always-loaded (the 3.1 modularization use case). Distinguishing them is a frequent distractor axis.
- **The one-line decision rule:** *does this convention correlate with a location, or with a kind of file?* Location → subdirectory `CLAUDE.md`. Kind → glob-scoped rule.
- **The one-line rules-vs-skills rule:** conventions are passive and unconditional; procedures are invoked. Path scoping can express *which files*, never *which task*.
- **The one-line guidance-vs-enforcement rule:** "must never" + a real incident → hook/permission, not a rule.
- **Silent failure is the signature.** A mis-anchored glob produces no error — just quiet non-compliance. Check for the leading `**/`, then run `/memory` to see what actually loaded.
- **Score on this topic: 4 / 5.** The miss was Q4 (rules vs. skills for a mis-housed runbook); the same boundary was then applied correctly in Q5.
