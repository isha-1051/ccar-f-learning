# Topic 2.1 — Tool Interface Design

**Domain:** 2 — Tool Design & MCP Integration (18%)
**Topic number:** 2.1 (first topic in Domain 2)

---

## Core mental model

A tool is a **contract exposed to Claude**, made of four parts:

| Part | Field | Who reads it | Purpose |
|------|-------|-------------|---------|
| Name | `name` | Claude | Identifies the capability; part of the prompt |
| Description | `description` | Claude | What it does, *when* to use it, when NOT to |
| Input schema | `input_schema` (JSON Schema) | Claude + your code | Params, types, required fields; constrains what Claude can emit |
| Result | `tool_result` content | Claude | The observation Claude reasons over next |

**Key insight:** the tool definition IS prompt engineering. Name, description, and schema are injected into Claude's context; Claude decides whether/how to call the tool based entirely on these strings. You're writing for a *model reader*, not just a compiler. A great implementation behind a vague description will be misused.

---

## Design principles the exam tests

### 1. Descriptions must be explicit and behavioral
State: **what** it does · **when to use / not use** · **what it returns** · **side effects** (writes? sends? charges money?) · **preconditions** (e.g., "call search_customer first for an ID").

- ❌ `"Gets user data"`
- ✅ `"Retrieves a customer's profile and subscription status by customer ID. Use for billing/account questions. Does not return payment card data. Requires a valid customer_id from search_customers."`

### 2. Design at the right granularity / altitude (heavily tested tradeoff)
- **Too fine-grained** (`open_file`, `seek`, `read_bytes`, `close_file`) → Claude orchestrates plumbing; more calls, more errors, more latency.
- **Too coarse** (one mega-tool with a `mode` flag) → ambiguous, poor error locality.
- **Right altitude** → each tool = a **meaningful unit of work** a human recognizes as one action (`search_flights`, `book_flight`).
- **Don't wrap your REST API 1:1.** Internal APIs are for programmers who read docs; tools are consumed by a model mid-reasoning. Consolidate, rename, reshape around the agent's *workflow/tasks*, not your endpoints or DB tables.

### 3. Make inputs hard to get wrong (constrain the schema)
Structural constraints beat pleading in prose (echoes Domain 1: enforcement > guidance).
- `enum` for fixed choices (not free-text + "please only use these").
- Mark truly-required fields `required`; keep genuinely-optional ones optional (forcing values causes hallucination).
- Precise types (`integer`, `number`, `boolean`); `description` on *each* property.
- Prefer well-named params over a generic `params: object` blob.
- Encode formats (`"format": "date-time"`, patterns) so Claude emits well-formed values.

### 4. Return useful, model-legible results
The result *is Claude's next observation*.
- Return **relevant** data, not raw dumps (token bloat degrades reasoning + costs money).
- Self-describing (labeled keys), not positional mystery values.
- On failure, return a **structured, actionable error** (root of Topic 2.2).
- Consistent shape across calls.
- **Security angle:** don't return sensitive internal fields at all — if the model never sees them, it can't leak them.

### 5. Naming
- Verb-based, action-oriented, unambiguous (`create_ticket`, `get_order_status`).
- Distinct from siblings — near-synonym names (`fetch_user` vs `get_user`) get confused. Disambiguation *between* tools matters as much as clarity *within* one.

---

## Anti-patterns (exam traps)

| Anti-pattern | Why it fails | Right move |
|---|---|---|
| Wrapping each REST endpoint 1:1 | Wrong altitude; Claude orchestrates plumbing | Task-level tools around the agent's job |
| Free-text param for fixed choices | Invalid/variant values | `enum` in schema |
| Vague description | Misuse or ignore | Behavioral: what/when/returns/side-effects |
| Returning huge raw payloads | Bloat, worse reasoning, cost, leakage | Filtered, relevant, labeled fields |
| Overlapping/ambiguous names | Wrong tool picked | Distinct names + when-not-to-use |
| One mega-tool with `mode` switch | Ambiguous, poor error locality | Split into distinct tools |
| Prose to enforce constraints ("never pass negative") | Prose ≠ enforcement | Schema min/max, enum, required |
| Forcing values for optional fields (`required` on everything) | Claude fabricates plausible-but-fake values | Mark genuinely-optional fields optional |
| Exposing internal IDs with no way to obtain them | Claude hallucinates them | Provide a lookup tool that *returns* the ID |

**Meta-principle:** a good tool interface is *self-documenting and hard to misuse*. If correct use requires Claude to "just know" things not in the description/schema, the interface is under-specified.

---

## Clarification captured this session — "filtered result" vs "returns only a string"

- **"Returns only a string"** = a constraint on the *transport/type*. What flows back is a `tool_result` text payload — an opaque string, not the subagent's messages array or a live object.
- **"Filtered result"** = a discipline about *what content* goes into that string: relevance selection + compression. Return the findings that matter, drop the noise. Done *before* handoff, because the coordinator can't thin it later.
- These don't conflict: it's *always* a string; "filtered" describes *which* info earns a place in it.
- **JSON ≠ filtered.** JSON is one *format* the string can hold. You can have filtered prose or unfiltered JSON — independent axes. Use JSON / a **shared output contract** when outputs must be *parsed or merged programmatically* (e.g., synthesizing multiple subagents). Even then it's *text that looks like JSON*; the harness parses it, the model doesn't return a native object.
- Same discipline applies to tools: return filtered fields, not raw payloads.

---

## Practice questions

### Q1 — Endpoint-mapped tools, hallucinated IDs, fragile chains
Support agent wraps REST endpoints 1:1. Claude guesses/hallucinates `customer_id` (users only give email) and chains 4–5 calls, sometimes forgetting a step.

- **A.** Add "always call lookup first" to each description.
- **B.** Keep 1:1 mapping, set `tool_choice: any`.
- **C.** ✅ Redesign around tasks: `find_customer(email)` returns the customer + ID; `get_billing_summary(customer_id)` consolidates subscription + invoices; descriptions state how IDs are obtained.
- **D.** Add `"format": "uuid"` to reject malformed IDs.

**Correct: C.** Attacks both failures structurally — Claude *obtains* the ID from an observation instead of inventing it, and the consolidated tool collapses the fragile chain into one meaningful unit of work (right altitude).
- **A** — prose plea; lowers but never removes error, and ignores chaining.
- **B** — `tool_choice: any` forces *some* tool call, not the *right* tool or valid args; a forced call with a hallucinated ID is still wrong. `tool_choice` controls *whether* a tool is used, not which/with what args.
- **D** — `format: uuid` only checks *shape*; a well-formed but fabricated UUID passes. Fix is to *return* the real ID, not validate harder.

**Takeaway:** Hallucinated identifier → give Claude a tool that *returns* it. Forgotten chain steps → raise altitude so the chain collapses.

### Q2 — Invalid enum-like values rejected downstream
`log_ticket_priority` has `priority: string` with "must be one of low/medium/high/urgent" in prose. Claude emits `"High"`, `"critical"`, `"P1"`, `"medium-high"`.

- **A.** Strengthen the description ("MUST use only these four lowercase words…").
- **B.** ✅ Change to `enum: ["low","medium","high","urgent"]`; remove the prose.
- **C.** Keep schema; add a PostToolUse hook to normalize.
- **D.** Add a `map_priority` tool Claude must call first.

**Correct: B.** `enum` makes invalid values structurally impossible (constrained decoding). Removing redundant prose avoids drift. Fixes the root cause.
- **A** — stronger prose is still a plea; never reaches zero.
- **C** — masks it; can't sensibly map `critical`/`P1` without guessing; reserve normalization for genuinely unconstrainable inputs.
- **D** — extra round-trip + new failure point + still relies on Claude picking a valid target; over-engineered for 4 fixed values.

**Takeaway:** Fixed known value set → `enum` is the canonical enforcement fix.

### Q3 — Over-broad result: slow, wrong order, leaks internal fields
`query_orders` returns entire order records (line items, addresses, warehouse codes, raw payment blobs, fraud scores). Answers are slow, cite the wrong order, and leak internal fields.

- **A.** Add "don't mention internal fields" to the system prompt.
- **B.** ✅ Shape the result: return only relevant user-facing fields (id, date, status, total, item summary), support filtering/limiting (recent-N / date range), exclude internal metadata.
- **C.** Enlarge context window / bigger model.
- **D.** Add a `fields` param so Claude picks fields per call.

**Correct: B.** Fixes all three at the common root (result returns too much of the wrong stuff): restores signal-to-noise *and* structurally prevents leakage (data the model never sees can't leak).
- **A** — prose plea that still puts sensitive data in context; unreliable guardrail; ignores slowness/wrong-order.
- **C** — pays more to make the root cause bigger; irrelevant tokens still degrade reasoning; doesn't stop leakage.
- **D** — shifts burden to Claude to enumerate fields correctly every call; internal fields still exposable; doesn't limit order count. Good as a *refinement* on top of safe defaults, not the primary fix.

**Takeaway:** Slowness + confusion + leakage together → suspect over-broad results. Return relevant, filtered, safe fields *by default*. Never rely on prose to hide data the model can already see.

### Q4 — Two near-synonym tools, wrong/expensive choices
`get_user` (cheap: id/name/email) vs `fetch_user_details` (expensive 3-service call: full profile). Claude calls the expensive one when it only needed a name, and sometimes calls cheap then expensive (double call).

- **A.** Merge into one `get_user` that always returns the full profile.
- **B.** ✅ Rewrite names + descriptions to be distinct/behavioral: `get_user_basic` ("id, name, email only; quick lookups; cheap") and `get_user_full_profile` ("complete profile incl roles/prefs/history; slower/expensive; use only when basic insufficient").
- **C.** Force `tool_choice: get_user` first every time.
- **D.** Delete `get_user`, keep only `fetch_user_details`.

**Correct: B.** Root cause = no basis to choose (synonym names, no field/cost info). Distinct behavioral names + descriptions encoding *what each returns*, *when*, and *cost* let Claude choose correctly, fixing both problems.
- **A** — every lookup now pays the expensive call; makes problem 1 permanent + result bloat.
- **C** — forces a useless cheap call before every legitimately-full-profile need — institutionalizes the double-call; `tool_choice` is blunt, not a per-need router.
- **D** — removes the valuable cheap path; same cost/latency regression as A.

**Takeaway:** Wrong-tool-picked → better names + behavioral descriptions encoding the distinction and tradeoff (cost/latency/completeness). Don't delete, merge, or hard-force order.

### Q5 — Required fields force fabrication; unparseable times
`schedule_meeting` marks `title, start_time, attendees, location, agenda_notes` all required. Meetings are often virtual (no location) and often have no agenda, so Claude invents "Conference Room B" and fake agenda notes. `start_time` comes back in inconsistent formats.

- **A.** Keep all required; add "leave blank if none" + "output ISO 8601" to the description.
- **B.** ✅ Make `location` and `agenda_notes` optional; add `"format": "date-time"` + ISO-8601 description to `start_time`; keep `title, start_time, attendees` required.
- **C.** Make every field optional; let the API fill defaults.
- **D.** Add an `enum` of conference rooms to `location`; add a `has_agenda` boolean.

**Correct: B.** `required` forces a value → removing genuinely-optional fields lets Claude omit them (no fabrication). `format: date-time` + description is the best *interface-level* lever for an open-ended timestamp (weaker than `enum`; still validate downstream). Keeping the truly-essential fields required is correct judgment.
- **A** — prose can't override a `required` constraint; empty string still forced/meaningless; ISO instruction weaker without `format`.
- **C** — over-corrects; Claude could omit `start_time`/`attendees` — can't schedule at all; punts quality to the API.
- **D** — `enum` of rooms breaks for virtual meetings (recreates the trap); `has_agenda` is needless cleverness when "optional" expresses absence natively; ignores the time-format problem.

**Takeaway:** Mark `required` only for genuinely-always-needed fields — forcing optional fields causes hallucination. For open-ended values, `format` + precise description is the best interface constraint (still validate downstream). Match the schema to the real shape of the data.

**Score:** 5/5.
