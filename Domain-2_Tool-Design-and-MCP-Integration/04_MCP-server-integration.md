# Topic 2.4 — Integrate MCP servers into Claude Code and agent workflows

- **Domain:** 2 — Tool Design & MCP Integration (18% of exam)
- **Topic number:** 2.4 (Topic 11 of 30 overall)
- **Exam task statement:** *Integrate MCP servers into Claude Code and agent workflows*

---

## 1. The frame

Topics 2.1–2.3 covered *one server's tools*: how to design them (2.1), how they fail (2.2), and how they get scoped and forced (2.3). **2.4 is the operations layer** — where server config lives, who it's for, how credentials get in without leaking, what the agent sees when servers connect, and when you should not write a server at all.

Four decision boundaries, all tested as **decisions**, not facts:

1. **Where does this server go?** — project scope vs user scope
2. **How do secrets get in?** — env var expansion, not literals
3. **Why isn't the agent using my MCP tool?** — description quality, not force
4. **Should I build this server?** — community server vs custom

---

## 2. Configuration scope: project vs user

The config **location encodes intent**.

### Project scope — `.mcp.json` at the repo root

```json
{
  "mcpServers": {
    "jira": {
      "command": "npx",
      "args": ["-y", "@company/jira-mcp-server"],
      "env": { "JIRA_API_TOKEN": "${JIRA_API_TOKEN}" }
    }
  }
}
```

- **Checked into version control** — travels with the repo.
- Everyone who clones gets the same tooling, same version, same names.
- Answer whenever the stem says: *"the whole team should…"*, *"standardize across engineers"*, *"new hires get it automatically"*, *"consistent tooling in the repo."*
- **Security consequence:** a `.mcp.json` arrives from a repo you may not have written, so Claude Code **prompts for approval before trusting project-scoped servers**. Committed config is executable content; the approval prompt is the mitigation. Expected behavior, not a bug.

### User scope — `~/.claude.json`

- Lives in your **home directory**, never committed.
- Applies to **you, across all your projects**.
- Answer for: *"I'm experimenting with…"*, *"my personal server"*, *"I want this in every project I work on"*, *"the rest of the team doesn't need this."*

### Decision rule

> **Shared and repo-specific → project `.mcp.json`. Personal or experimental → user `~/.claude.json`.**

Both directions of error are tested:

- **Personal/experimental server committed to `.mcp.json`** → imposes an unvetted dependency on the whole team; everyone gets an approval prompt for a server they never asked for.
- **Team-critical server left in `~/.claude.json`** → works on one machine only. Onboarding becomes "ask Priya for her config." Shows up in stems as *"inconsistent results across the team."*

**Mental model:** `.mcp.json` is to MCP servers what `package.json` is to dependencies — a committed, shared declaration. `~/.claude.json` is your dotfiles.

### Scope precedence (clarification raised in session)

There is a third scope — **local**: a private, per-project config stored in user settings for *that project only* (not committed, not shared, doesn't apply to your other projects). When the same server name appears at multiple scopes:

> **local → project (`.mcp.json`) → user (`~/.claude.json`)**

Most-specific wins; one server runs, the lower-precedence definition is **shadowed, not merged**.

Practical takeaways:
- A developer can override a shared team server (point `jira` at a sandbox) without editing committed `.mcp.json`.
- If a teammate reports the team server "behaving strangely," a shadowing local/user entry with the same name is a prime suspect.
- Prefer distinct names when you don't intend an override.

---

## 3. Environment variable expansion — credentials without committed secrets

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" }
    },
    "internal-api": {
      "type": "http",
      "url": "${API_BASE_URL:-https://api.staging.corp}",
      "headers": { "Authorization": "Bearer ${API_KEY}" }
    }
  }
}
```

- `${VAR}` expands from the environment at server-launch time.
- `${VAR:-default}` supplies a fallback for non-secret values (base URLs, regions).
- Works in `command`, `args`, `env`, `url`, `headers`.

**The point:** the **shape** of the integration (which server, which version, which name, which endpoint) is shared and version-controlled; the **secret** stays per-developer, from the shell profile, `direnv`, or a CI secret store. One config, N credentials. Rotation becomes an environment change, not a commit.

**Anti-patterns:**

- **Literal token in `.mcp.json`** — in git history forever; rotation requires a commit; every contributor holds prod credentials. The single most-tested mistake in this task statement.
- **Moving the server to user scope so each person can hardcode their own token** — trades a leaked secret for a lost standard. Env var expansion is precisely what lets you keep project scope *and* keep secrets out.

**Key insight: scope and secrets are independent axes.** Project scope gives sharing; `${VAR}` gives privacy. Distractors are built by conflating them — offering a scope change as the fix for a secrets problem, or a secrets mechanism as the fix for a sharing problem.

CI/CD extension (links to Domain 3.6): the same committed `.mcp.json` runs in the pipeline, with `${GITHUB_TOKEN}` resolved from the CI secret store instead of a developer's shell.

---

## 4. Discovery: all servers, all tools, all at once

At startup Claude Code connects to **every configured server** (project + user scope together) and performs **tool discovery at connection time**.

- Tools from **all** connected servers are in the tool list **simultaneously**; the agent chooses among them per turn.
- Tools are namespaced by server, conventionally `mcp__<server>__<tool>` (e.g. `mcp__jira__search_issues`). Namespacing prevents collisions when two servers both expose `search`.
- Discovery is a **connection-time** event, not per-turn. New tools mid-session = a server reconnected.
- The agent sees one **flat** tool list — it has no notion of "which server this came from" when choosing.

Three consequences:

**(a) Tool-set bloat is real.** Every discovered tool's name, description, and JSON schema sits in context on **every** request. Ten chatty servers consume a large slice before the user types anything. Links directly to Domain 5 (context management). "Configure fewer, better-scoped servers" is a legitimate remedy.

**(b) Selection ambiguity grows with tool count.** With 60 tools, overlapping ones compete. The *distribution* layer (which servers at which scope) is the first control on what the agent can even consider.

**(c) A configured-but-unused server still costs you** — context and a connection. Remove it.

---

## 5. The "agent ignores my MCP tool" problem

Canonical scenario shape:

> Your team ships an MCP server with `search_codebase` (semantic search with a pre-built symbol index). Claude keeps using built-in **Grep** instead, giving worse results.

**Root cause is almost always the tool description.** The agent picks tools by reading descriptions. Built-ins have clear, well-written descriptions; a fresh MCP tool often says `"Search the codebase."` — strictly less informative, so Grep wins. The agent isn't stubborn; it's given a worse pitch.

**Fix — a description that explains capabilities, outputs, and the preference boundary:**

```
Semantic code search across the entire monorepo using a pre-built symbol
index. Returns ranked results with file path, line range, enclosing symbol
name, and surrounding context. Understands intent, so it finds conceptually
related code that literal pattern matching misses (e.g. "where do we handle
payment retries" finds the relevant code even when it never says "retry").
Prefer this over Grep for any conceptual or exploratory search. Use Grep
only when you need exact literal matches of a known string.
```

Four qualities: *what it does*, *what it returns*, *what it can do that the alternative can't*, *an explicit preference boundary*. This is 2.1's "tool definitions are prompt engineering" at the integration layer.

**Descriptions also fix over-use.** For an over-eager `escalate_to_human`, write the precedence rule into the *over-used* tool: "Use ONLY when (a) the action is beyond this agent, or (b) you have searched the knowledge base and found no answer. Do NOT escalate merely because the customer is frustrated." Half of "wrong tool" problems are solved by stating what must happen *before* a tool.

**Portability argument (why description beats CLAUDE.md):** the description **travels with the server** — every project, every client, every teammate — and is present at the exact moment of decision. CLAUDE.md guidance is repo-local and patches the symptom while leaving the defective artifact in place.

> **CLAUDE.md is for facts about *this repository* (conventions, commands, directory rules). The tool description is for facts about *the tool*.**

**Where descriptions stop working:** they are **advisory** — they shift probability, they don't guarantee. The moment a stem says *"must never," "compliance requires," "a single occurrence is unacceptable"* you have left the territory descriptions can serve.

---

## 6. Build vs. buy

> **Choose existing community MCP servers for standard integrations; reserve custom servers for team-specific workflows.**

**Community server** when the integration is with a *standard, widely used system with a public API* — Jira, GitHub, Slack, Sentry, Google Drive, Postgres. Auth flows, pagination, rate limits, error mapping, and ongoing API-change maintenance are already solved. Rebuilding is undifferentiated work you own forever.

**Custom server** when the capability is genuinely **yours**:

- an internal service with no public MCP server (deployment orchestrator, feature-flag system, proprietary warehouse);
- a **team-specific workflow** composing several systems into one task-level operation — e.g. `prepare_release(version)` = tag repo + update changelog + transition Jira tickets + post to Slack.

That second case is 2.1's **task altitude** at the server level: a custom server earns its existence by exposing operations at the altitude of your team's actual tasks. **The justification is the composition, not the systems it touches.**

**Anti-patterns:**
- Building "our own Jira server" because the community one lacks a field → contribute the field or wrap it; don't sign up to maintain a Jira client.
- **Forking** a maintained community server to add an unrelated capability → inherits all the maintenance burden *plus* a merge conflict on every upstream change, for the purely cosmetic benefit of "one server." Since tools are namespaced and the agent sees a flat list, there is no coherence benefit to buy.
- Composing a multi-step, side-effecting workflow across three community servers **via prompt guidance** → turns an atomic operation into an advisory sequence (see §8).

---

## 7. MCP resources — the underrated half

Servers expose more than tools. **Resources** are addressable pieces of **content**, identified by URI, listed on request, read into context.

| | **Tools** | **Resources** |
|---|---|---|
| Nature | Actions the model invokes | Content that can be read |
| Driven by | The model, autonomously | Application/user |
| Cost | Round trip + result tokens | A read of known content |
| Example | `search_issues(jql)` | `jira://issues/open` — a catalog |

**Why the exam cares: resources reduce exploratory tool calls.**

Without a catalog, an agent asked "which open bug relates to checkout timeouts?" guesses its way to the data — `search_issues("checkout")`, `search_issues("timeout")`, `list_projects()`, retry — four blind round trips before real work. With a **content catalog** resource, it gets visibility into what exists up front and its first tool call is the right one. Discovery-by-trial becomes discovery-by-catalog.

Canonical catalog examples from the exam scope:

- **Issue summaries** — which tickets exist, without blind querying.
- **Documentation hierarchies** — which docs exist, before fetching.
- **Database schemas** — correct query on the first attempt instead of `SELECT *`-ing to learn column names.

**Heuristic:** *if the agent's first N tool calls are consistently spent learning the shape of the data rather than acting on it, that shape belongs in a resource.*

**Anti-pattern:** exposing an enormous resource (the whole ticket database) that floods context. A catalog is an **index** — IDs, summaries, schema metadata — designed to make the next tool call precise. Not a data dump. (Mirrors 2.1's "shape and scope the results.")

### Resources: Claude Code vs. Agent SDK (clarification raised in session)

The MCP spec calls resources **application-controlled** — unlike tools, the model does not autonomously fetch them.

- **Claude Code:** resources surface in the UI; you pull one into context by **`@`-mentioning** it (picker lists resources from connected servers, addressed like `@server:protocol://path`). Human-in-the-loop selection.
- **Agent SDK:** nothing is automatic. Your harness calls list-resources / read-resource and decides what to inject. Resources are a **harness responsibility**, not an agent capability.

**Consequence:** if a scenario wants the *agent itself* to autonomously decide to retrieve something mid-loop, that's a **tool**, not a resource. Resources are for content you or your harness deliberately place in front of it — which is exactly why they fit the "catalog that prevents exploratory tool calls" pattern.

---

## 8. The escalation ladder (deep dive)

Each rung is a different **kind** of intervention, ordered by how much agent judgment it destroys.

| Rung | Mechanism | What it does | What it costs | Scope of effect |
|---|---|---|---|---|
| **1. Description** | Tool definition text | Teaches the agent *when* to choose it | Nothing — judgment preserved | Everywhere that tool goes |
| **2. Distribution** | Which tools exist in this context at all | Removes the wrong option from consideration | Tool is gone even when it'd be right | Per agent/subagent/scope |
| **3. `tool_choice`** | Per-request API parameter | Forces a tool on *this* call | All judgment on that call | One request |
| **4. Hooks** | Deterministic harness callback | Blocks/permits regardless of the model | Rigid; needs code + maintenance | Every call, guaranteed |

**Governing question:** *Is this a knowledge problem, an availability problem, a single-call determinism problem, or a safety guarantee?*

### Rung 1 — Description (knowledge problem)

*Worked example:* the `search_codebase` vs Grep case above. The agent had both tools and **chose wrong** after reading two descriptions. Fix the description, because Grep is still correct for literal matches (rung 2 would break that) and `tool_choice` would force semantic search onto the literal case too.

*Tells:* "prefers the built-in," "doesn't seem to know," "isn't using our tool," "escalates too readily."

### Rung 2 — Distribution (availability problem)

*Worked example:* a research coordinator spawns summarizer subagents that inherit `write_file`, `execute_sql`, and `deploy`. No description work makes a summarizer need `deploy`. Give them a read-only tool set — `deploy` isn't blocked, it's absent.

**Prefer not-granting over granting-then-blocking**: simpler, free, no maintained code, and it reclaims the context those schemas consumed.

*In Claude Code, distribution IS scope management.* Ten accumulated servers producing both context bloat and mis-selection are fixed by removing abandoned experiments and moving one-person servers to `~/.claude.json` — a tool-selection problem solved by a configuration-scope decision, with zero description or hook work.

*Tells:* subagent with over-broad tools, too many servers, "this agent shouldn't be able to," context/latency from tool count.

### Rung 3 — `tool_choice` (single-call determinism)

Values: `auto` (default), `any`, `tool`, `none`. Governs **one API call**.

*Right use:* a batch pipeline extracting invoice fields where 2% of responses come back as prose and break the parser → `tool_choice: {"type":"tool","name":"extract_invoice"}`. Forcing costs nothing because **there was no judgment worth preserving** — you never asked *whether* to extract.

*Wrong use:* forcing `search_codebase` to beat Grep. Wrong granularity (applies to *every* request, breaking valid literal-match turns), wrong layer (defect stays in the server, so other clients hit it), and it doesn't compose with an agent loop whose next action varies per turn.

*Tells:* "sometimes replies in prose," "must always return valid JSON," "downstream parser breaks," fixed-shape extraction/classification.

*Limitation:* `tool_choice` constrains **which tool is called**, never **arguments**, and never side effects.

### Rung 4 — Hooks (safety guarantee)

`PreToolUse` runs deterministically before a tool executes and can block it. Independent of what the model reads, understands, or is told — including prompt-injected instructions.

*Right use:* an agent may `SELECT` against production but must never `DROP`/`DELETE`/`TRUNCATE`. Descriptions are advisory; the tool must remain available; `tool_choice` doesn't see arguments. Only a hook reaches it.

**Distinguishing feature: hooks inspect *arguments*.** When the rule is "this tool is fine but not with those inputs," a hook is typically the only rung that reaches it.

*Wrong use:* blocking Grep to compensate for a bad description — disproportionate, permanently maintained code preventing a legitimate tool from doing a legitimate thing. **Hooks are for consequences that are unacceptable, not preferences that are unmet.**

*Tells:* "must never," "regardless of instructions," "compliance/audit," "irreversible," "even once is unacceptable."

### The four-question triage

1. **Would a better-informed agent have chosen correctly?** → **Description**
2. **Does this tool have any legitimate use in this context?** If no → **Distribution**
3. **Is the correct action fully determined by the workflow for this call?** → **`tool_choice`**
4. **Must this be impossible regardless of the model's decision — especially based on arguments?** → **Hook**

### The tie-breaker

> **When two rungs would both work, the lower one is usually the answer** — cheaper, more portable, preserves judgment you'll want later. **Exception:** when the scenario states an absolute requirement, the *lowest rung that actually guarantees it* wins, and advisory rungs become distractors no matter how elegant.

Refined rung-2-vs-rung-4 boundary:

> **Zero legitimate uses of the tool/connection → distribution (don't grant it). Some legitimate uses but forbidden arguments or conditions → hook (grant it, constrain it).**

---

## 9. Signal glossary (how the exam flips the answer)

**Keep the LOW rung** (advisory suffices; higher rung is over-engineering):
"improve," "reduce," "more often," "encourage," "tends to," "sometimes," "the team finds it annoying," "results are worse," "wastes tokens/time," "without restricting legitimate use," "most maintainable," "simplest effective," "least disruptive," "the tool is useful in other situations."

**Force the HIGH rung** (advisory becomes a distractor however well-written):
"must never," "under no circumstances," "compliance requires," "audit," "regulatory," "PII," "irreversible," "destructive," "production," "a single occurrence is unacceptable," "guarantee," "deterministically," "regardless of the prompt," "prompt injection," "untrusted input," **"the previous fix worked most of the time but failed occasionally."**

> **Load-bearing insight: "most of the time" is a passing grade for a preference and a failing grade for a guarantee.** Nearly every tie-breaker item turns on which of those the scenario describes.

**The consequence test:** ask *"what happens on one more occurrence?"*
- Someone is annoyed / a result is worse / tokens wasted → **low rung**
- Data destroyed / secret leaked / audit failed / payment sent → **high rung**

Intensity words ("constant complaint," "frustrated," "keeps doing it") are **not** absolute words — staying low is correct there. That's the *false absolute* trap.

**The "previous fix partially worked" stem:** *"we prompted it to X but it occasionally still does Y"* means advisory control was applied to a problem requiring a guarantee. A "write a better description" option will always be present and is always wrong. The stem is telling you the **class** of control was wrong, not the **quality** of its instance.

**Select-TWO items** exploit that rungs are complementary, not competing — usually one advisory problem and one absolute problem in the same stem. The trap is picking two options that address the *same* problem at two different rungs. First ask how many distinct problems the stem describes.

---

## 10. Anti-pattern summary

| Anti-pattern | Why it fails | Correct move |
|---|---|---|
| Literal API token in committed `.mcp.json` | Secret in git history, shared team-wide, painful rotation | `${TOKEN}` expansion; secret from env/CI store |
| Gitignoring `.mcp.json` to protect a literal token | Config no longer arrives on clone; 40 hand-written files drift | Commit it, use `${VAR}` |
| Team-critical server in `~/.claude.json` | Works on one machine; inconsistent team results | Project-scoped `.mcp.json`, committed |
| Experimental server in project `.mcp.json` | Imposes unvetted dependency + approval prompt on everyone | User scope until proven |
| Forcing the MCP tool with `tool_choice`/hooks | Removes judgment for a description problem | Rewrite the description: capabilities, outputs, preference boundary |
| CLAUDE.md rule instead of fixing the description | Patches the symptom; repo-local; doesn't travel with the server | Fix the description |
| Custom server for Jira/GitHub/Slack | Undifferentiated maintenance burden | Community server; custom only for internal/team-specific workflows |
| Forking a community server to add a capability | Full maintenance burden + perpetual merge conflicts, for cosmetic "one server" | Wrap alongside, or contribute upstream |
| Composing a side-effecting multi-step workflow via CLAUDE.md prompt | Advisory ordering → partial-completion states | Custom tool making the sequence structural |
| Many overlapping servers configured | Tool-set bloat: context cost + selection ambiguity | Scope deliberately; remove unused servers |
| Agent burns calls discovering data shape | No catalog to orient from | Expose schema/index as an MCP resource |
| Resource that dumps all the data | Floods context | Catalog = index (IDs, summaries, schema), not contents |
| Post-hoc validation for irreversible actions | Damage is done at tool execution; nothing to catch after | Pre-execution `PreToolUse` check |

---

## 11. One-line compression

> **Scope encodes intent (`.mcp.json` = team, `~/.claude.json` = you); `${VARS}` keep the config shareable and the secrets private; all servers' tools land in context at once, so configure deliberately; the agent chooses tools by description, so fix descriptions before forcing anything; buy the standard integrations and build only what's yours; and expose catalogs as resources so the agent stops guessing.**

---

# Practice Questions

## Question 1 — Scope + credentials

A platform team at a fintech company maintains a monorepo used by 40 engineers. They have built an internal MCP server, `deploy-tools`, exposing operations for their custom release pipeline (tagging, changelog generation, staging promotion). The server authenticates to their internal deployment API using a bearer token that is unique per engineer and rotates every 30 days.

The team lead wants every engineer to get this tooling automatically when they clone the repo, at a consistent version, without following setup instructions in a wiki page. Security has stated that no credential may appear in version control.

Which configuration approach best satisfies these requirements?

**A.** Add `deploy-tools` to the project-scoped `.mcp.json` at the repo root, with the token value written directly into the `env` block, and add `.mcp.json` to `.gitignore` so the credential is never committed.

**B.** Add `deploy-tools` to the project-scoped `.mcp.json` at the repo root, using `"DEPLOY_TOKEN": "${DEPLOY_TOKEN}"` in the `env` block, and commit the file. Each engineer exports `DEPLOY_TOKEN` in their shell environment.

**C.** Have each engineer add `deploy-tools` to their user-scoped `~/.claude.json` with their personal token, and document the required configuration block in the team wiki.

**D.** Add `deploy-tools` to the project-scoped `.mcp.json` with a placeholder token, commit it, and have each engineer override the entry in their local-scoped configuration with their real token.

### ✅ Correct answer: **B**

**Why B is right.** It satisfies all three constraints, each with the mechanism designed for it:

- *Automatically on clone, consistent version, no wiki setup* → **project scope, committed `.mcp.json`**. Config travels with the repo; a new hire gets it by cloning.
- *No credential in version control* → **`${DEPLOY_TOKEN}` expansion**. What's committed is the *shape* of the integration; the *value* resolves at server-launch time from the engineer's environment.
- *Unique per engineer, rotates every 30 days* → rotation is a shell/secret-store change, not a commit. Nobody touches the repo when a token rolls.

Central lesson: **scope and secrets are independent axes.** Project scope gives sharing; env var expansion gives privacy. No trade-off required.

**Why A is wrong.** Solves the secret problem by destroying the sharing requirement. A gitignored `.mcp.json` isn't in the repo, so it does *not* arrive on clone and every engineer must hand-author it — failing "consistent version" (40 files drift) and adding leak risk from a single `git add -f`. Gitignoring the shared config is the opposite of what project scope is for: it's `package.json` in `.gitignore`.

**Why C is wrong.** The "works on my machine" anti-pattern. User scope means the config exists only for that engineer: new hires must follow the wiki (explicitly ruled out), 40 hand-copied blocks drift in version/args/name, and any change requires 40 people to re-read a page. C *does* keep credentials out of git, which is why it's the strongest distractor — a legitimate mechanism applied to the wrong requirement. **User scope is for personal or experimental servers, not team-critical tooling.**

**Why D is wrong.** Misuses shadowing: local-scope overrides exist so an individual can point at a *different* server (sandbox, fork), not as routine credential delivery. Every engineer maintains a duplicate definition locally, reintroducing C's drift on top of a committed placeholder. And a placeholder token makes the config broken-by-default, failing at connection time with a confusing auth error instead of a clear unset-variable failure. `${DEPLOY_TOKEN}` expresses the dependency honestly.

**Carry forward:** when a stem pairs *"whole team / on clone / consistent"* with *"no secrets in git,"* the answer is committed project `.mcp.json` + `${VAR}` expansion. Distractors offer scope changes as the fix for a secrets problem.

---

## Question 2 — Tool selection: the description rung

A developer-tools team ships an MCP server exposing `query_metrics`, which runs aggregations over their observability warehouse and returns time-bucketed results with anomaly annotations. The tool description currently reads: `"Query the metrics warehouse."`

Engineers report that when they ask Claude Code questions like *"was the checkout latency spike yesterday related to the deploy?"*, Claude tends to run `Bash` commands against local log files instead of calling `query_metrics` — producing shallow, incomplete answers. The `Bash` tool remains genuinely necessary for many other tasks in this repo.

Which action most effectively addresses the root cause?

**A.** Configure a `PreToolUse` hook that blocks `Bash` invocations containing log-file paths, forcing the agent toward `query_metrics`.

**B.** Set `tool_choice` to `{"type": "tool", "name": "query_metrics"}` for requests in this workflow so the agent reliably uses the observability data.

**C.** Rewrite the `query_metrics` description to detail its capabilities, its return shape, and when it should be preferred over local log inspection.

**D.** Add a rule to the repository's `CLAUDE.md` instructing Claude to always use `query_metrics` for any question about latency, errors, or deploy impact.

### ✅ Correct answer: **C**

**Why C is right.** The agent had both tools available and *chose* between them. Nothing blocked it — it read `"Query the metrics warehouse."` next to `Bash`'s rich description and rationally judged `Bash` the better instrument. **Knowledge problem**, and the defective artifact is the tool definition.

A fixed description:

```
Runs aggregations over the observability warehouse. Returns time-bucketed
series with p50/p95/p99, annotated with detected anomalies and correlated
deploy events. Covers all services and all environments with 90 days of
retention. Prefer this for any question about latency, error rates, traffic,
or whether an incident correlates with a deploy. Local log files are
incomplete — they cover only one service and are rotated after 24 hours.
```

Two elements do the work: **what it can do that the alternative cannot**, and **an explicit preference boundary** naming the competing approach. And the fix **travels with the server** — every team, every client, every repo, with no per-project configuration.

**Why A is wrong.** Disproportionate and damages a legitimate capability. `Bash` is "genuinely necessary for many other tasks," so you'd permanently maintain harness code preventing a good tool from doing a valid thing, to compensate for a description you could fix in five minutes. Nothing here is unsafe, irreversible, or non-compliant — the stated harm is "shallow answers." **Hooks are for consequences that are unacceptable, not preferences that are unmet.**

**Why B is wrong.** Wrong granularity and wrong layer. `tool_choice` governs a single request applied uniformly — it would force `query_metrics` on turns needing `Read`, `Grep`, or `Bash`, which in an interactive loop is most turns. It converts a *sometimes*-wrong heuristic into an *always*-rigid constraint, and the underlying defect stays in the server for every other client.

**Why D is wrong — the sharpest distinction here.** D is also advisory, so it's not over-escalation; it's tempting because it would probably help somewhat. It loses on **locality and portability**: it patches the symptom while leaving the defective artifact in place, it's repo-local (another team installing the server hits the identical problem), and it sits far from the decision point, competing with the thin description the model reads at selection time.

> **CLAUDE.md is for facts about *this repository*; the tool description is for facts about *the tool*.** "When to prefer `query_metrics` over log files" is a fact about the tool.

---

## Question 3 — Escalating to a guarantee

A data-engineering team runs Claude Code in a CI pipeline to investigate failing nightly ETL jobs. The agent is configured with a Postgres MCP server exposing `execute_sql`, connected to a read-replica for analysis and — because the same server config is reused — to the production primary for a small set of remediation runbooks.

The team has already added to the tool description: *"Only use SELECT statements when investigating. Never issue DDL or DML against production."* Over three months this has worked in nearly every run, but last week the agent executed a `TRUNCATE` against a production table while attempting to reset a stuck job's checkpoint. Recovery took six hours.

Leadership requires that this cannot happen again under any circumstances, while the agent must retain the ability to query production for investigation.

What is the most appropriate change?

**A.** Rewrite the tool description with more explicit, emphatic prohibitions and concrete examples of forbidden statements, including the exact `TRUNCATE` case that occurred.

**B.** Remove the production connection from the MCP server configuration so the agent can only reach the read-replica.

**C.** Implement a `PreToolUse` hook on `execute_sql` that parses the statement and denies any non-`SELECT` operation targeting the production connection.

**D.** Set `tool_choice` to `{"type": "tool", "name": "execute_sql"}` and add a validation layer that inspects the model's output before the pipeline commits any changes.

### ✅ Correct answer: **C**

**Why C is right.** Three stacked signals, each independently forcing this answer:

1. **"Cannot happen again under any circumstances"** — a categorical requirement. Only a deterministic control supplies a guarantee.
2. **The low rung is shown failing.** The prohibition was already in the description and worked "in nearly every run." The *class* of control was wrong, not the quality of its instance.
3. **The rule is about arguments, not tool identity.** `execute_sql` must stay available; the constraint is on what's *inside* the call. Only a hook inspects the argument payload pre-execution.

`PreToolUse` runs in the harness, deterministically, independent of what the model reads or is told — including a prompt-injected instruction arriving from an ETL error message. It also satisfies the second requirement: `SELECT` against production passes through untouched.

**Why A is wrong.** The stem was engineered to disqualify it — the team *already did this*, and the failure is inherent to the mechanism, not its wording. Adding the specific `TRUNCATE` example over-fits the last incident and does nothing for `DROP`, `DELETE`, `ALTER`, or `UPDATE`. Whenever a stem says *"we prompted it and it mostly works, but occasionally…"*, "write a better prompt" is present and wrong.

**Why B is wrong — the strongest distractor.** Distribution *is* right when a capability has no legitimate use, and this would genuinely guarantee the outcome. It fails the explicit second requirement: *"the agent must retain the ability to query production for investigation."* B buys safety by destroying a needed capability.

> **Zero legitimate uses → distribution. Some legitimate uses but forbidden arguments → hook.** Production access is legitimate for `SELECT`, so the constraint must live at argument level.

**Why D is wrong.** Misunderstands `tool_choice`: it constrains *which tool is called*, has no view of arguments, and there's only one tool in play, so forcing accomplishes nothing. The "validation before the pipeline commits" is worse than useless — `TRUNCATE` executes the instant the tool runs; there is no later commit gate. **Post-hoc validation cannot protect against irreversible side effects; the check must be pre-execution.**

---

## Question 4 — Build vs. buy

A team is building an internal agent that helps engineers work with Jira. Their requirements:

1. Standard Jira operations — searching issues, reading issue details, adding comments, transitioning status.
2. A company-specific operation called "sprint handoff," which reads the current sprint's unfinished issues, moves them to the next sprint, generates a summary document in Confluence, and posts a digest to a specific Slack channel following their team's format.

A well-maintained community MCP server exists for Jira and covers requirement 1 completely.

Which approach best reflects sound MCP integration architecture?

**A.** Build a single custom MCP server covering both requirements, so the team owns the full surface and can guarantee consistent behavior across all Jira operations.

**B.** Use the community Jira MCP server for requirement 1, and build a small custom MCP server exposing a `sprint_handoff` tool for requirement 2.

**C.** Use the community Jira, Confluence, and Slack MCP servers, and instruct the agent in `CLAUDE.md` to perform the sprint handoff by composing calls to those servers in the correct order.

**D.** Fork the community Jira MCP server and extend the fork with the sprint-handoff capability, so all Jira-related functionality remains in one server.

### ✅ Correct answer: **B**

**Why B is right.** It applies the build-vs-buy boundary exactly where the exam draws it — **community servers for standard integrations, custom servers reserved for team-specific workflows** — per *capability*, not per *system*.

- Requirement 1 is a standard integration with a public API. Auth, pagination, rate limits, JQL handling, error mapping, and ongoing API-change maintenance are already solved.
- Requirement 2 is genuinely yours. No community server knows your sprint conventions, Confluence template, or target Slack channel.

`sprint_handoff` is 2.1's **task altitude** at the server level: one call, one outcome, at the altitude of a real team task. **A custom server is justified by the composition, not by the systems it touches.**

**Why A is wrong.** "Own the full surface for consistency" produces the most-warned-against outcome: maintaining a Jira client forever, inheriting every future API change, for zero differentiated value.

**Why C is wrong — the subtlest option.** Appealing (all three servers exist, no custom code), but it turns a **single atomic operation into a prompt-guided multi-step workflow** — the Domain 1.4 anti-pattern. The handoff has ordered steps with **irreversible side effects**: issues move, a document publishes, a message reaches a channel people read. Advisory enforcement means the agent may move issues then fail to post the digest, post before the doc exists, format differently each sprint, or skip a step on a long context — partial-completion states a human must detect and repair.

A custom tool makes the sequencing **structural**: ordering lives in code that always runs in order, and the agent's only decision is *whether* to run the handoff. Correct match of enforcement to stakes — and cheaper in context (one schema vs. orchestrating three servers' full tool sets).

**Why D is wrong.** Worst of both worlds: all of A's maintenance burden *plus* a merge conflict on every upstream change, forever, for the cosmetic benefit of "one server." "All Jira functionality in one place" is a false constraint — tools are namespaced (`mcp__jira__*`, `mcp__sprint-tools__*`), all servers' tools are discovered at connection time, and the agent sees them **simultaneously in a flat list** with no notion of origin when choosing. No coherence benefit to buy.

> **Contribute upstream or wrap alongside — don't fork a maintained community server to add a capability that isn't its concern.**

---

## Question 5 — Resources as content catalogs

An agent assists analysts with an internal reporting database of 200+ tables. The MCP server exposes `run_query(sql)` and `list_tables()`.

Reviewing traces, the team finds a consistent pattern. For a request like *"what was refund volume by region last quarter?"*, the agent typically:

1. calls `list_tables()`,
2. runs `SELECT * FROM refunds LIMIT 5` to learn the column names,
3. runs another exploratory query after guessing wrong about the region column's location,
4. joins to a `regions` table it discovers by trial and error,
5. finally issues the correct query.

Each session burns four to six exploratory calls before productive work begins, consuming latency and context. The queries themselves are correct once the agent understands the schema.

What is the most effective architectural improvement?

**A.** Expose the database schema — table names, column names and types, and foreign-key relationships — as an MCP resource that provides a catalog of available data.

**B.** Improve the `run_query` tool description to instruct the agent to inspect the schema thoroughly before writing queries, and to avoid `SELECT *`.

**C.** Add a `get_table_schema(table_name)` tool so the agent can retrieve column information for a specific table in one call instead of sampling rows.

**D.** Increase the result limit on `list_tables()` so it returns column metadata alongside table names, giving the agent more information per call.

### ✅ Correct answer: **A**

**Why A is right.** The stem describes exactly the failure MCP resources exist to eliminate — **the agent spending its opening tool calls learning the *shape* of the data rather than acting on it.** The queries are correct once it understands the schema; all waste is in discovery.

A resource is **application-controlled content**, not a model-invoked action: the catalog is placed in front of the agent *before* it starts guessing, so discovery-by-trial becomes discovery-by-catalog and the first `run_query` is the right one. Database schemas are one of the three canonical catalog examples in this task statement (with issue summaries and documentation hierarchies) because schema is *stable, bounded, and needed on nearly every request* — the ideal profile for context you provide rather than context the agent must earn. Including **foreign-key relationships** kills the trial-and-error join in step 4.

> **Heuristic:** if the agent's opening moves are consistently spent orienting rather than acting, that orientation belongs in a resource.

**Why B is wrong.** Advisory guidance for a problem that isn't about judgment. The agent behaves *correctly* given that the schema exists nowhere it can reach except by sampling rows. Telling it to "inspect the schema thoroughly" while giving it no schema to inspect just prescribes more exploratory calls, and "avoid `SELECT *`" removes its only current technique. **You cannot instruct an agent into information it doesn't have access to.**

**Why C is wrong.** A real improvement that treats the symptom, partially. It compresses step 2, but steps 1, 3, and 4 survive: the agent still doesn't know *which* tables are relevant among 200, so it still lists, guesses, and discovers the join by trial. Each blind probe is cheaper; the blindness remains. It's also still **model-invoked** — a round trip it must decide to make, every session, per table.

**Why D is wrong — the closest distractor.** Right direction, three defects:

- Still a **tool call** the agent must decide to make, costing a round trip before any work.
- **Overloads a listing tool into a schema dump**, so its result grows unbounded — 200 tables × every column, paid for inside the conversation, mid-loop. Violates 2.1's "shape and scope the results."
- Carries column metadata but **not relationships**, so step 4's join discovery persists.

> **Tools are for actions the model decides to take; resources are for content the application decides to provide.** "The agent should already know what exists" → resource. "The agent should be able to look something up situationally" → tool.

Caveat the exam also tests in reverse: a catalog is an **index**, not a data dump. Table/column/relationship metadata is right; table *contents* would be context flooding.

---

## Score

**5 / 5** on first attempt.

Discriminators that carried each item: scope-vs-secrets as independent axes (Q1), locality/portability of the fix (Q2), argument-level control requiring a hook (Q3), custom servers earning their place through composition (Q4), application-provided context vs. model-invoked discovery (Q5).
