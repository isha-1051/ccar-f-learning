# Topic 2.5 — Select and apply built-in tools (Read, Write, Edit, Bash, Grep, Glob) effectively

- **Domain:** 2 — Tool Design & MCP Integration (18% of exam)
- **Topic number:** 2.5 (Topic 12 of 30 overall) — final topic of Domain 2
- **Exam task statement:** *Select and apply built-in tools (Read, Write, Edit, Bash, Grep, Glob) effectively*

---

## 1. The frame — context economy at the filesystem layer

Every built-in tool call is a way of **spending context to buy information**. The exam rarely asks "what does Read do?" — it asks *"the agent is doing X inefficiently or failing at Y; what is the most effective fix?"* The answer is almost always one of three diagnoses:

1. **A broad tool was used where a narrow one existed** (read 40 files instead of grepping one symbol).
2. **The wrong axis was searched** (searched *paths* when the question was about *content*, or vice versa).
3. **A whole-file operation was used where a targeted one existed** (rewrote a 2,000-line file to change one import).

Unifying principle: **retrieve the minimum that answers the question, at the narrowest granularity the task allows.** This is the same economy principle as 1.6 (decomposition granularity) and 2.1 (result shaping), applied to the filesystem.

---

## 2. The six tools

### Grep — search *inside* files (content)
Answers *"Where does this string/pattern appear?"* — function names, class names, error message text, imports, config keys, feature flags, route strings.

- Regex content search across the codebase; ripgrep-backed, respects `.gitignore` (no `node_modules` drowning).
- Returns **matches with locations**, not whole files — cheap in context.
- Scopable by path/glob filter and file type.
- It is the **entry-point discovery tool**: start here whenever you don't know where something lives.

### Glob — search file *paths* (names/structure)
Answers *"Which files exist matching this naming pattern?"* — `**/*.test.tsx`, `src/**/*.py`, `**/Dockerfile`, `**/migrations/*.sql`.

Glob knows **nothing about file contents**; it matches path strings only.

| The question you're really asking | Tool |
|---|---|
| "Where is `validateToken` called?" | **Grep** |
| "What are all the React test files?" | **Glob** (`**/*.test.tsx`) |
| "Which tests reference `validateToken`?" | **Grep**, scoped with a glob path filter |
| "Do we have a `.env.example`?" | **Glob** |
| "What error string does the API return on 429?" | **Grep** |

Grep and Glob are not rivals: **Glob narrows the search space, Grep searches within it.** "Find callers of `foo` in test files only" is *one* Grep with a glob filter — not a Glob followed by 200 Reads.

### Read — load a full file (or a slice)
Takes you from *"there's a match at `auth.ts:112`"* to actually understanding the logic.

- Supports **offset/limit** — read lines 800–900 of a huge file rather than all 5,000.
- **Read is a prerequisite for Edit.** You cannot construct a reliable unique anchor (exact whitespace and indentation) without having seen the text. In Claude Code, Edit on an unread file is rejected outright.

### Write — create or fully overwrite a file
Correct uses: creating a **new** file; **wholesale regeneration** of a generated file; the **Read + Write fallback** (§3).

Anti-pattern: small change to a large existing file. You must reproduce the entire file perfectly, which (a) costs output tokens proportional to file size, (b) risks silently mangling untouched regions, (c) produces an unreviewable diff — the whole file reads as changed.

### Edit — targeted modification by unique text match
`old_string` → `new_string`, where `old_string` must appear **exactly once**. This uniqueness requirement is the most exam-tested property in the topic.

Uniqueness is a **feature, not a limitation**: it makes the edit **fail loudly rather than apply in the wrong place**. A non-unique anchor means the intent was ambiguous, and erroring out is safer than guessing. Contrast with line-number-based editing, which silently corrupts the file as soon as anything shifts.

Advantages over Write: minimal tokens, minimal blast radius, clean reviewable diff, provably untouched surroundings.

### Bash — run shell commands
**Legitimate:** `git status` / `git diff` / `git log`, `npm test`, `pytest`, `tsc --noEmit`, builds, linters, `mkdir`, package installs, project CLIs.

**Illegitimate (favorite exam distractor):** `cat`, `head`, `tail`, `sed`, `awk`, `find`, `ls -R`, `grep` when a dedicated tool exists. Why the dedicated tool wins:

- **Structured output** (line numbers, match locations) vs raw stdout the model must parse.
- **Safety** — `sed -i` edits blind, no uniqueness check, no read-before-write; `find /` can wander outside the project.
- **Permissioning and hooks** — the harness can reason precisely about `Edit(path)`; it can only see an opaque string for Bash, so PreToolUse hooks and permission rules (1.5, 2.3) stop working.
- **Truncation handling** — Read handles a 50 MB file sanely; `cat` dumps it into context.

> **Bash is for *doing*, not for *looking*.** If a command's purpose is to observe files, a built-in tool almost certainly beats it.

---

## 3. Edit failure → the escalation ladder (named in the exam guide)

**Scenario:** you want to change one occurrence of `const timeout = 30;` but it appears six times. Edit fails with a non-unique match error.

**Correct order of remedies:**

1. **Expand the anchor first.** `old_string` is a multi-line string — it need not be only the line you're changing. Include the enclosing function signature, the preceding comment, or the following line. This is the *preferred* fix and the one weak candidates skip; it keeps the edit targeted.
2. **Replace-all** — but *only* when you genuinely intend every occurrence to change (e.g. a whole-file symbol rename).
3. **Read + Write fallback** — when no reasonably-sized anchor can be made unique: mechanically generated files, pathologically repetitive content, or many interdependent changes at once. **Read the full file, construct the complete corrected content, Write it back.**

Read *must* precede Write in the fallback: writing without reading means reconstructing the file from assumption, and **you will silently destroy content you never saw**.

Exam framing: an option offering `sed -i` to make the substitution is wrong precisely because it reintroduces the ambiguity Edit's failure was warning about — and does it blindly.

---

## 4. Incremental codebase understanding

Exam guide wording: *"starting with Grep to find entry points, then using Read to follow imports and trace flows, rather than reading all files upfront."*

**Anti-pattern:** "Read every file in `src/` to understand the codebase." On a real repo that's 200k+ tokens, mostly irrelevant, and it actively *degrades* performance by burying the 3 relevant files in noise (lost-in-the-middle, Domain 5). It's also unbounded — cost scales with repo size, not with the task.

**Correct loop:**

```
Grep for the symbol / error string / route  →  a handful of locations
   ↓
Read those specific files (or slices of them)
   ↓
See what they import / call  →  Grep or Read those
   ↓
Repeat until the flow is traced
```

Each step is **hypothesis-driven**: you read a file because a specific prior finding pointed at it. Context grows with the *relevant subgraph*, not the repository.

**Worked example** — *"Users report 'Payment declined: code 42' — find where it originates."*
1. `Grep "Payment declined"` → `src/billing/errors.ts:88` defines the template.
2. `Read src/billing/errors.ts` → thrown by `mapGatewayError`.
3. `Grep "mapGatewayError"` → called in `src/billing/stripe-adapter.ts`.
4. `Read` that file → code 42 maps from the gateway's `insufficient_funds`.

Four cheap calls; the read-everything approach costs ~50× more and answers less reliably.

**When a Grep is too noisy:** don't Read 400 hits — tighten. Add a path filter (`src/api/**`), a file-type filter, a more specific regex (`export function \w*Handler`), or search for a rarer co-occurring token.

---

## 5. Tracing usage across wrapper / barrel modules

Exam guide wording: *"first identifying all exported names, then searching for each name across the codebase."*

**The trap:** you want every caller of the `auth` module, so you `Grep "auth"` or `Grep "from './auth'"`, get 12 hits, and conclude there are 12 callers. **Wrong** — re-export/barrel patterns break the name chain:

```ts
// src/auth/index.ts
export { verifyToken, refreshSession } from './tokens';
export { requireRole as guard } from './rbac';
```

```ts
// src/api/orders.ts
import { guard } from '@/auth';        // never mentions "requireRole"
```

Searching `requireRole` misses `orders.ts` because the call site uses the **alias**. Searching `'./auth'` misses anything importing via `@/auth`, a barrel, or a deeper path.

**Correct two-phase procedure:**
1. **Enumerate the surface.** Read (or Grep) the module's export statements to collect the *complete* set of exported names, **including aliases created at re-export time**. Follow the barrel — the internal name and the exposed name can differ, and you need both.
2. **Search each name.** Grep every name in that set across the codebase, then reconcile (internal callers use the original name, external consumers the alias).

Generalizable lesson: **a single search over one spelling is not a complete usage trace when the language allows renaming.** Same applies to Python `from x import y as z`, and to dynamic access such as `handlers["create"]`, which no name search catches — a reason to also grep the *string* form.

---

## 6. Built-in tools vs MCP tools (crossover with 2.3 / 2.4)

Topic 2.4 named an anti-pattern: *"preferring built-in tools (like Grep) over more capable MCP tools."* Reconcile it carefully — the exam tests the boundary in both directions.

- **Built-in tools own the local filesystem.** Finding, reading, editing code → Grep/Glob/Read/Edit, always. Using an MCP server to read a local file is pure overhead.
- **MCP tools own external systems and semantics the filesystem lacks.** A Sentry MCP tool returning "the 5 production stack traces for this error, grouped by frequency" cannot be substituted by Grep, which can only report which line contains a string. A Jira ticket is not in the repo.

**Decision rule:** *Does the answer live in this repo's files? → built-in. Does it live in an external system, or require semantics beyond text matching? → MCP.*

The named failure mode is a model with a rich MCP tool available that still tries to reconstruct the answer by grepping local files: semantically wrong data, more calls, worse result. The correction is usually a **sequence, not a replacement** — MCP retrieves the external evidence, then built-in tools investigate the specific code it implicates.

---

## 7. Anti-pattern checklist

| Anti-pattern | Why it's wrong | Do this instead |
|---|---|---|
| Read every file to "understand the codebase" | Unbounded cost; buries signal in noise | Grep entry point → Read → follow imports |
| `Bash: cat file.py` | Unstructured output, no line numbers, no truncation handling | **Read** |
| `Bash: find . -name "*.test.ts"` | Slower, noisier, ignores gitignore semantics | **Glob** |
| `Bash: grep -r "foo" .` | Traverses `node_modules`, unstructured results | **Grep** |
| `Bash: sed -i 's/a/b/' file` | Blind, no uniqueness check, no read-before-write | **Edit** (or Read + Write) |
| Write to change 1 line of a 2,000-line file | Token cost, corruption risk, unreviewable diff | **Edit** |
| Edit a file you haven't Read | Anchor is guessed; will fail or hit the wrong spot | **Read first**, then Edit |
| Retrying the same failed Edit verbatim | Non-uniqueness doesn't fix itself | Widen the anchor → then Read + Write |
| Replace-all to silence a uniqueness error | Succeeds silently while changing unintended sites | Widen the anchor |
| Glob to find code by content | Glob can't see inside files | **Grep** |
| Grep one name to trace a barrel module's usage | Aliased re-exports are invisible | Enumerate exports → Grep each name |
| Guessing location from directory names | Substitutes convention for evidence | Grep the literal you already have |
| "Increase the context window" for a sweep | Treats the symptom; strategy still scales with repo size | Directed, hypothesis-driven retrieval |
| Re-reading a file you just edited to "verify" | Edit errors on failure; success is already proof | Trust the tool result |

---

## 8. How exam items are phrased

- *"An engineer needs to find all callers of a function across a large monorepo. Which approach is most efficient?"*
- *"Claude's Edit call fails with 'string appears multiple times.' What is the best next step?"*
- *"An agent consumes its entire context window before completing the task. Which change most effectively addresses the root cause?"*
- *"Which tool should be used to locate all files matching `**/*.integration.spec.ts`?"*

Distractors are **plausible but broader, blinder, or on the wrong axis**: Bash equivalents of built-in tools; Write where Edit fits; Glob where Grep fits; "read the whole directory first"; "retry the edit"; "give it a bigger context window."

---

# Practice Questions

## Question 1 — Tracing usage across a barrel module

**Scenario:** In a large TypeScript monorepo, `formatCurrency` is defined in `packages/utils/src/money.ts`. The package barrel file `packages/utils/src/index.ts` contains:

```ts
export { formatCurrency as fmtMoney, parseCurrency } from './money';
```

You must find every place in the repo that depends on the helper before changing its implementation. Which approach most reliably produces a complete list of dependent call sites?

**A.** Run a single Grep for `formatCurrency` across the repo and treat the resulting matches as the complete set of call sites.
**B.** Use Glob to list every `**/*.ts` and `**/*.tsx` file, then Read each one and note which files use the helper.
**C.** Read the barrel file to enumerate the exported names and aliases (`formatCurrency`, `fmtMoney`), then Grep for each name across the repo and reconcile the results.
**D.** Run `grep -r "from './money'" .` via Bash to find every module that imports from the defining file.

**Correct answer: C** ✅ *(answered correctly)*

**Why C is right:** This is the two-phase trace the exam guide names — enumerate exported names first, then search each. The barrel renames `formatCurrency` to `fmtMoney` at the package boundary, so external consumers write `fmtMoney` and never mention the original name. Knowing the full set of spellings *before* fanning out is what makes the result trustworthy. Reconciliation matters too: internal callers inside `packages/utils` use the original name, external consumers the alias, and both are dependents.

**Why A is wrong:** The trap. One Grep covers one spelling; every consumer importing the alias is invisible, and you'd report a "complete" list that silently omits them. The tool choice is right — the *coverage assumption* is the defect. A single search over one spelling is not a usage trace when the language permits renaming.

**Why B is wrong:** Finds everything but is the read-all-files-upfront anti-pattern in pure form — cost scales with repo size instead of the relevant subgraph, and real call sites drown in irrelevant source. It also misuses Glob, which matches *paths* and can only hand you a file list, leaving content inspection to brute-force Reads.

**Why D is wrong:** Two failures. It's a Bash `grep -r` where the built-in Grep belongs (traverses `node_modules`, unstructured output, ignores gitignore semantics), and it searches the **wrong axis** — modules importing from the *defining path*, when nearly every external consumer imports from the package root. It would catch the barrel file and little else.

---

## Question 2 — Edit fails on a non-unique anchor

**Scenario:** In `services/ingest.py`, three functions each contain `max_retries = 3`:

```python
def fetch_batch(client, cursor):
    max_retries = 3
    ...
def fetch_manifest(client):
    max_retries = 3
    ...
def fetch_schema(client):
    max_retries = 3
    ...
```

Only `fetch_batch` should become `max_retries = 8`. An Edit with `old_string="    max_retries = 3"` fails: the string is not unique. What is the most effective next step?

**A.** Re-issue the same Edit with a replace-all flag so the uniqueness error no longer blocks it.
**B.** Expand `old_string` to include the enclosing `def fetch_batch(client, cursor):` line, making the anchor unique, and re-issue the Edit.
**C.** Read the entire file, reconstruct its full contents with the single value changed, and Write it back.
**D.** Use Bash `sed -i` with a line-number address to substitute the value on the correct line.

**Correct answer: B** ✅ *(answered correctly)*

**Why B is right:** First rung of the escalation ladder. `old_string` is a multi-line string, so pulling in the enclosing `def` line makes the anchor unique in one step while **preserving the targeted nature of the edit** — minimal tokens, two-line diff, provable non-interference with the other two functions. The uniqueness error wasn't an obstacle to route around; it was the tool correctly reporting an ambiguous anchor, and the direct remedy for an ambiguous anchor is a more specific one.

**Why A is wrong:** Replace-all is correct only when you intend *every* occurrence to change (a whole-file rename). Here the requirement is explicit that only `fetch_batch` changes, so it would silently alter `fetch_manifest` and `fetch_schema`. The most dangerous distractor because it "works" — the call succeeds, no error surfaces, unintended changes ship. Suppressing a safety check is not fixing the cause.

**Why C is wrong:** Read + Write is the correct fallback but the *last* rung, for when no reasonable anchor can be unique. Not the case here — one enclosing line resolves it. Jumping straight to it means reproducing a whole file to change one character, paying tokens proportional to file size, risking corruption in re-emitted regions, and turning a two-line diff into a whole-file diff. Not wrong in *outcome*, but disproportionate — and "most effective" is asking for proportionality.

**Why D is wrong:** Abandons the very safety property the failure was protecting. `sed -i` edits blind — no read-before-write, no uniqueness verification. Line-number addressing is especially brittle: the number is valid only against the file state you *assume* is current. It also sidesteps the harness layer, where permission rules and hooks can reason about `Edit(path)` but not about an opaque shell string.

---

## Question 3 — Context exhaustion during codebase exploration

**Scenario:** An agent is diagnosing a bug in an unfamiliar 3,000-file Django repo. The only clue is a user-facing banner reading `"Sync interrupted — retry in 30s"`. The agent Globs for `**/*.py`, then Reads the returned files in directory order to build understanding. It exhausts its context after ~10% of the repo, never reaching the relevant code. Which change most effectively addresses the **root cause**?

**A.** Increase the agent's context window and re-run the same strategy.
**B.** Grep for the error string `"Sync interrupted"` to locate the originating module, Read that file, and follow its imports and callers outward.
**C.** Use Read with offset/limit to load only the first 50 lines of each Python file, surveying all 3,000 files within budget.
**D.** Narrow the Glob to `**/sync/**/*.py` and Read every file under directories whose names suggest relevance.

**Correct answer: B** ✅ *(answered correctly)*

**Why B is right:** The root cause isn't budget size — it's **undirected retrieval**. The agent held a high-signal, near-unique literal and never used it. One Grep collapses 3,000 files to a handful of exact locations, and every subsequent Read is hypothesis-driven. Context then grows with the *relevant subgraph* rather than the repo, which is why the approach scales identically at 30,000 files. Exactly the guide's prescription: Grep for entry points, then Read to follow imports and trace flows.

**Why A is wrong:** Treats a symptom as the cause. Reading in directory order is unbounded work unrelated to the question, so a bigger window just fails later and more expensively — and stuffing hundreds of irrelevant files into context actively degrades reasoning (lost-in-the-middle). "Give it a bigger budget" is nearly always a distractor when the real defect is undirected retrieval.

**Why C is wrong:** Economizes on the wrong dimension. The first 50 lines are mostly imports and headers, while the error is raised in a function body — so it is nearly guaranteed to miss the target while still burning a large budget across 3,000 files. Offset/limit is excellent for reading a known region of a *known* file, not for skimming a repository.

**Why D is wrong:** The most tempting distractor, since narrowing *is* the right instinct — but it substitutes a **guess about naming conventions** for **evidence**. Django projects routinely place such logic in `tasks.py`, `celery.py`, `views.py`, or an app named `integrations`; nothing guarantees a `sync/` directory exists. You hold a literal that identifies the exact origin — inferring location from folder names when you can search content is choosing the weaker signal.

---

## Question 4 — Built-in tools vs MCP tools

**Scenario:** A Sentry MCP server is connected, exposing `get_issue_details(issue_id)` (production stack trace, occurrence count, affected releases, offending commit link). Ticket: *"Sentry issue PROJ-4471 is spiking since yesterday's deploy — figure out what's breaking and propose a fix."* The agent's first three actions: Grep the repo for `"4471"`, Glob for `**/*.log`, Read `CHANGELOG.md`. It reports it cannot locate the issue. Which statement best characterizes the error and its correction?

**A.** It should have run `git log --since=yesterday` via Bash first, since commit history is the authoritative record of what broke.
**B.** It correctly preferred built-in tools over MCP tools, which is the recommended default; the real fix is broadening the Grep to `PROJ-4471`.
**C.** It attempted to reconstruct external system state from local files; it should have called the Sentry MCP tool to retrieve the issue, then used Grep and Read to investigate the code the stack trace implicates.
**D.** It should Read every file changed in the most recent deploy, since the spike began after that deploy.

**Correct answer: C** ✅ *(answered correctly)*

**Why C is right:** Names the actual defect — answering a question about **external system state** with **local filesystem tools**. A Sentry issue ID is a key into Sentry's database; it appears nowhere in the repo, so no amount of grepping can surface it. Stack traces, occurrence counts, and affected releases are semantics the filesystem doesn't have, making the MCP tool the only path to the data. Note the correction is a **sequence, not a replacement**: MCP retrieves the trace, then built-in tools investigate the specific files and frames it implicates.

**Why A is wrong:** `git log` is legitimate Bash — the "doing/toolchain" category — so the tool choice isn't the flaw; the ordering and reasoning are. Without the stack trace you'd scan a deploy's worth of commits for an unknown defect. Retrieve the evidence first, then let it point at the commit. Git history is also only circumstantial: a post-deploy spike can come from a config change, a dependency bump, or traffic finally hitting dormant code.

**Why B is wrong:** Misapplies a real principle. Preferring Grep over MCP is right when the MCP tool duplicates what the filesystem already does well; but the guide equally flags the inverse — preferring built-ins over **more capable MCP tools** that supply data the built-ins cannot. It also fails on its own terms: widening the pattern searches harder for a string that was never in the repo.

**Why D is wrong:** The "read everything, but scoped" trap. Its premise is unsound (a spike correlating with a deploy doesn't establish the defect is in the diff), and even granting it, a deploy can touch hundreds of files with no criterion for recognizing the bug. Contrast with the correct path: the stack trace names file, function, and line, turning a mass sweep into one Read at a known location.

---

## Question 5 — Multiple response: identify incorrect tool selection

**Scenario:** Task — *"Add an `X-RateLimit-Remaining` header to all API responses, then run the test suite."* Agent transcript on a mid-sized Node/TypeScript service:

1. `Bash: cat src/middleware/headers.ts` — inspect existing middleware
2. `Grep: "res.setHeader"` — find where response headers are currently set
3. `Glob: **/*.test.ts` — locate test files needing updates
4. `Write: src/server.ts` — re-emit all 1,800 lines to add one `import` statement
5. `Bash: npm test` — execute the suite

**Select TWO** actions that represent incorrect tool selection.

**A.** Action 1 — `Bash: cat src/middleware/headers.ts`
**B.** Action 2 — `Grep: "res.setHeader"`
**C.** Action 3 — `Glob: **/*.test.ts`
**D.** Action 4 — `Write: src/server.ts`
**E.** Action 5 — `Bash: npm test`

**Correct answers: A and D** ✅ *(answered correctly)*

**Why A is a defect:** The canonical "Bash is for doing, not for looking" violation. Read returns line-numbered, structured output the model can cite and anchor against, handles large files with truncation and offset/limit, and — decisively — **establishes the read-before-edit precondition**. `cat` yields unnumbered raw stdout, so the agent must then construct an Edit anchor from text it can't reliably locate. It also bypasses the harness layer, where permission rules and PreToolUse hooks can reason about `Read(path)` but only about an opaque shell string for Bash.

**Why D is a defect:** Disproportionate on three counts. *Cost*: output tokens scale with file size — 1,800 lines to add one. *Risk*: every untouched line must be reproduced perfectly, and drift silently corrupts code the agent was never asked to touch, invisibly, because the call succeeds. *Reviewability*: the diff shows the entire file changed. Edit with a unique anchor near the import block yields a two-line diff with provable non-interference.

**Why B is correct usage:** Right axis, right altitude. "Where are response headers set?" is a **content** question, and `res.setHeader` is a high-signal literal that collapses the search space in one cheap call. It's also the correct *first* move — entry-point discovery before reading anything.

**Why C is correct usage:** A **path** question answered with the path tool. `**/*.test.ts` is a naming-convention query with nothing to match inside the files. B and C divide cleanly along the content/path axis — the distinction the exam tests most often.

**Why E is correct usage:** The legitimate Bash category — executing a real toolchain with side effects. No built-in tool runs a test suite, so there's no narrower alternative to prefer. The contrast with Action 1 is the point: Bash used to *observe files* duplicates a better tool; Bash used to *do work* is exactly what it's for.

---

## Clarifications & cross-topic links

- **Uniqueness as a safety property.** Edit's non-unique failure is a *guard*, not a defect. Any option that removes the guard (replace-all, `sed -i`, line numbers) to make the error disappear is wrong unless the semantics genuinely call for it.
- **Content vs path axis.** The single most frequent Grep/Glob discriminator. Ask which axis the question lives on before choosing.
- **Bash boundary.** *Doing* (tests, git, builds, installs) = yes. *Looking* (cat/find/grep/sed) = no.
- **Sequence over replacement.** MCP and built-in tools compose: external retrieval first, local investigation second (links to 2.4).
- **Directed retrieval** is the same idea as coarse-but-independent decomposition (1.6) and result shaping (2.1) — spend context only where it buys answers.
