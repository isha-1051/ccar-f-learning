# Topic 4.3 — Enforce structured output using tool use and JSON schemas

**Domain:** 4 — Prompt Engineering & Structured Output (20% of exam)
**Topic number:** 3 of 6 within Domain 4 · Topic 21 of 30 overall

---

## 1. The problem this solves

You need machine-consumable output. The naive approach — *"Respond with a JSON object
containing `invoice_number`, `total`, and `line_items`. Return only JSON, no other text."* —
fails in production at a low but fatal rate:

- Preamble text ("Here's the extracted data:") before the `{`
- Markdown ```` ```json ```` fences
- Trailing commas, unescaped quotes inside string values
- Truncation mid-object at `max_tokens`
- The model deciding a caveat is more helpful than compliance

At 1,000 documents and a 2% malformed rate that's 20 pipeline failures — and they are
*intermittent*, the worst failure mode to debug. Prompting your way to 100% is not
achievable, because you are asking the *model* to be reliable at something the *API* can
simply guarantee.

> **Core insight:** JSON formatting is not a prompting problem. It is a constraint-enforcement
> problem, and it belongs at the API layer.

---

## 2. Tool use as a schema carrier

Define a tool whose `input_schema` **is** your output schema.

```json
{
  "name": "extract_invoice",
  "description": "Record the structured fields extracted from an invoice document.",
  "input_schema": {
    "type": "object",
    "properties": {
      "invoice_number": { "type": "string" },
      "invoice_date":   { "type": "string", "description": "ISO 8601, YYYY-MM-DD" },
      "total":          { "type": "number" }
    },
    "required": ["invoice_number"]
  }
}
```

Claude responds with a content block, and `stop_reason: "tool_use"`:

```json
{ "type": "tool_use", "id": "toolu_01A...", "name": "extract_invoice",
  "input": { "invoice_number": "INV-4471", "invoice_date": "2026-03-04", "total": 1840.50 } }
```

Three things to internalize:

**(a) `input` arrives as a parsed object, not a string.**
There is no `JSON.parse()` in your code path. There is nothing to fail. That is the entire
guarantee.

**(b) The tool is never executed.**
In an agentic loop, `tool_use` means "run this function and return a `tool_result`." In the
*extraction* pattern you read `input` and **stop** — no `tool_result`, no second API call, no
loop. The tool is a **contract, not an action**; you are borrowing the tool-calling machinery
purely for its schema enforcement.

**(c) Validity is enforced by the serving layer, not the model's good intentions.**
Types, `required` presence, and `enum` membership are structurally guaranteed.

> **Exam phrasing to recognize:** "guaranteed schema-compliant structured output,"
> "eliminates JSON syntax errors." When one option is *tool use with a JSON schema* and the
> others are *prompt for JSON / return only JSON / prefill with `{` / add a few-shot JSON
> example* — tool use wins. The others reduce the error rate; only tool use eliminates the
> failure class.

---

## 3. `tool_choice` — the most-tested decision

Understand each value by **what it forbids**, not what it allows.

| Value | Model must… | Can return plain text? | Picks which tool? |
|---|---|---|---|
| `{"type": "auto"}` *(default when `tools` present)* | nothing | **Yes** | model |
| `{"type": "any"}` | call **some** tool | **No** | model |
| `{"type": "tool", "name": "X"}` | call **tool X** | **No** | you |
| `{"type": "none"}` | not call tools | Yes | n/a |

### When each is correct

**`auto` — the model may legitimately have no structured output to give.**
Conversational agents that sometimes answer from knowledge, sometimes call a tool, sometimes
ask a clarifying question.

> **Anti-pattern:** `auto` in a batch extraction pipeline. It *usually* calls the tool — which
> is what makes it dangerous. 96% structured, 4% prose, non-deterministic parser failures. If
> structured output is required, `auto` is wrong by definition.

**`any` — output must be structured, but which schema depends on the input.**
The canonical multi-schema case: invoices, receipts, and purchase orders arrive interleaved and
type can't be determined without reading content. Supply all three tools, set `any`. Two
benefits:

1. **Tool selection becomes classification** — the returned `name` tells you the document type,
   with no separate classifier.
2. **Each schema keeps its own integrity** — `required` stays meaningful per type, because each
   schema only ever applies to its own document type.

**`{"type": "tool", "name": "..."}` — you already know which extraction must run.**
Two triggers:
1. **Single known document type** — remove the model's discretion entirely.
2. **Pipeline sequencing** — a later stage (enrichment, routing, scoring) consumes this
   extraction, so forcing the named tool makes the step **structurally guaranteed** rather than
   hoped-for.

*(Same principle as Domain 1.4: when correctness depends on a step definitely happening,
enforce it structurally rather than asking nicely. `tool_choice` is the API-layer version.)*

**`none`** — rarely tested; suppresses tools mid-conversation without rebuilding the request.

### Two mechanics worth knowing

- **`disable_parallel_tool_use: true`** (a field on `tool_choice`). With `any` or a forced tool
  this guarantees *exactly one* call. For "one object from one document," set it — otherwise you
  may get competing extractions and have to decide which is real.
- **Forcing a tool suppresses preamble reasoning.** Under `auto` Claude often thinks in text
  before calling; under `any`/forced it goes straight to the call. Recover deliberation by making
  a **`reasoning` string the first property in the schema** — properties are filled in order, so
  the model reasons before it commits. Also: **extended thinking is only compatible with
  `tool_choice: auto`** — it cannot be combined with `any` or a forced tool.

---

## 4. The boundary that matters most: syntax vs. semantics

**Tool use eliminates syntax errors. It does nothing about semantic errors.**

| Guaranteed by the schema | Not guaranteed |
|---|---|
| `total` is a number | `total` equals the sum of `line_items` |
| `invoice_number` is present | it isn't actually the PO number |
| `status` ∈ `{paid, pending, overdue}` | it's the *correct* status |
| `invoice_date` is a string | it's real, not fabricated |
| all `required` fields present | `end_date` isn't before `start_date` |

Stock exam examples: **line items that don't sum to the stated total**, and **values in the
wrong field** (invoice number vs. PO number — both plausible strings, both schema-valid).

When a scenario shows output that is *well-formed but wrong*, "tighten the schema" and "add
`strict: true`" are **distractors** — the schema is already doing its whole job. The fix lives in
validation and retry (Topic 4.4), which is why these task statements sit adjacent.

> **One-line version:** the schema guarantees the *shape* of the answer, never its *truth*.

---

## 5. Schema design — where the real judgment is

### (a) `required` creates fabrication pressure

The highest-yield principle in this task statement.

`required` is a hard constraint. If the source document lacks the information, the model cannot
return nothing — so it returns *something*: a plausible date, a nearby number, a guessed vendor.
**You have converted "missing data" into "silently wrong data,"** which is strictly worse, because
missing data is detectable and wrong data is not.

**Rule:** a field is `required` only if it is *guaranteed present in every source document*.
Otherwise make it optional or nullable.

```json
"purchase_order_number": {
  "type": ["string", "null"],
  "description": "PO number if present on the document; null if the document does not state one."
}
```

The description carries real weight: it tells the model `null` is a *correct answer*, not a failure
to try harder. Without it, models still lean toward guessing on optional fields.

> **Exam phrasing:** *"the model fabricates values to satisfy required fields"* → make fields
> optional/nullable. **Not** "add 'do not guess' to the prompt" — that is a soft instruction against
> a hard constraint, and the constraint wins.

### (b) Enums need escape hatches — two different ones

A closed enum gives consistency and downstream safety. It also means an out-of-set input gets
**silently coerced into the nearest listed value** — schema-valid, well-typed, wrong.

| Value | Means | Use when |
|---|---|---|
| `"other"` + `other_detail` string | "A real category you didn't list" | Category space is **open/extensible** |
| `"unclear"` / `"not_specified"` | "The document doesn't let me determine this" | Source is **ambiguous or silent** |

```json
"document_type": {
  "type": "string",
  "enum": ["invoice", "receipt", "purchase_order", "credit_note", "other", "unclear"]
},
"document_type_detail": {
  "type": ["string", "null"],
  "description": "Required when document_type is 'other'. Name the actual document type."
}
```

The detail field is also a **discovery mechanism**: aggregate `other` details over a month and you
learn which categories to promote into the enum. A closed enum with no escape hatch hides that
signal permanently — you cannot distinguish "no such documents exist" from "we miscategorize every
one of them."

### (c) Descriptions are prompt real estate

The model reads `description` on every field. Field-level rules belong there, adjacent to the
decision point, not buried in a system-prompt paragraph.

```json
"severity": {
  "type": "string",
  "enum": ["critical", "high", "medium", "low"],
  "description": "critical = exploitable security flaw or data loss; high = incorrect behavior in a common path; medium = incorrect behavior in an edge case; low = style or maintainability."
}
```

This composes with Topic 4.1: the *explicit criteria* live in the description, the *enum* makes the
classification structurally clean.

### (d) Normalization rules go in the prompt/description, not the type system

JSON Schema can say `"type": "string"`. It cannot reliably say "dates as `YYYY-MM-DD` even when the
source writes `3/4/26` or `4 March 2026`," or "currency as a bare decimal, no symbols or separators."
Type checking doesn't reach format conventions.

**Strict schema for structure + explicit normalization rules in prompts/descriptions for formatting.**
The exam names this directly. Neither alone is sufficient.

### (e) Keep schemas flat and shallow

Deep nesting raises the semantic error rate — fields land at the wrong nesting level, schema-valid at
both, so nothing catches it. Flatten where reasonable. To extract N repeated things use **one array
field inside one tool call**, not N calls: one coherent object, one place to validate.

---

## 6. Anti-pattern summary

1. **Treating schema compliance as data correctness.** Valid ≠ true.
2. **Marking everything `required` "so downstream code is simpler."** Complexity moves into
   hallucinations you can't see.
3. **Closed enums with no `other`/`unclear`.** Confident miscategorization, and the signal that
   would have told you is destroyed.
4. **`auto` when structured output is mandatory.** Intermittent prose responses.
5. **Forcing one named tool across heterogeneous inputs.** Guarantees the *wrong* schema is applied.
6. **Prompting harder for clean JSON.** Wrong layer — that's `tool_choice`'s job.
7. **Treating an extraction tool like a real tool** — returning a `tool_result` and continuing the
   loop, when you should read `input` and stop.
8. **Forcing a tool on a judgment-heavy task without a `reasoning` field**, then wondering why
   quality dropped.
9. **Splitting one extraction across multiple calls** for "smaller schemas" — the model can no longer
   see related fields together, so cross-field inconsistency becomes undetectable in-model.

---

## 7. Decision cheat-sheet

| Situation | Setting |
|---|---|
| Output must be structured, one known schema | `tool_choice: {"type":"tool","name":"X"}` |
| Must be structured, schema depends on input | `tool_choice: "any"` |
| A specific extraction must run before a later stage | forced named tool |
| Model may legitimately answer in prose / ask a question | `auto` |
| Exactly one object expected | + `disable_parallel_tool_use: true` |
| Judgment-heavy task under a forced tool | leading `reasoning` field in the schema |
| Field may be absent from source | optional / `["string","null"]` |
| Category set may grow | enum + `"other"` + detail string |
| Source may be ambiguous | enum + `"unclear"` |
| Output well-formed but factually wrong | **not a schema problem** → Topic 4.4 |

---

# Practice Questions

## Q1 — Mixed document queue, text-only responses

**Scenario.** A logistics company processes ~8,000 shipping documents per night: commercial
invoices, packing lists, bills of lading, and customs declarations, interleaved, with no reliable
filename convention — type cannot be determined without reading content. Four separate JSON schemas
exist, one per type. A downstream router dispatches the extracted object to the correct billing or
compliance service. All four extraction tools are defined and `tool_choice` is left at its default.
Nightly, ~300 documents fail the router with a parse error; in those cases Claude returned a text
response — usually a note that the document appeared to be a scanned form, asking how to handle it —
instead of calling any tool.

**Which change most effectively guarantees the router always receives structured output?**

- **A.** Add a system prompt instruction: "You must always call one of the provided extraction tools.
  Never respond with plain text under any circumstances."
- **B.** Set `tool_choice` to `{"type": "any"}`.
- **C.** Set `tool_choice` to `{"type": "tool", "name": "extract_commercial_invoice"}`, since invoices
  are most common, and handle the other three types in a separate pass.
- **D.** Merge the four schemas into a single tool with all fields optional, and force
  `{"type": "tool", "name": "extract_document"}`.

### ✅ Correct answer: B

`any` forbids a text-only response — the model must call one of the tools while retaining the choice
of *which*. That maps exactly onto the problem: the requirement is "always structured," the
uncertainty is "which schema applies."

Two consequences that matter:

1. **Tool selection becomes classification.** The returned `name` tells the router the document type
   — free type detection from the same call, no separate classifier.
2. **Each schema keeps its integrity.** `required` stays meaningful per type (a bill of lading can
   require carrier and vessel; a customs declaration can require an HS code), because each schema only
   applies to its own type.

The failure mode being fixed is *behavioral discretion*, not capability. Claude was choosing to ask a
question — reasonable for a scanned form, and correctly removed by `any`. If fields are genuinely
unreadable, that now surfaces as nulls in the data (a detectable condition) rather than a router parse
error.

### ✗ Why the others are wrong

- **A — right intent, wrong layer.** A prompt instruction is a soft constraint against a problem with a
  hard-constraint solution available. It will reduce the 300 failures, possibly a lot, but cannot
  eliminate them: compliance remains the model's decision. Same category of error as prompting for
  clean JSON instead of using a schema.
- **C — forces the wrong schema.** Named-tool forcing is for a *known* type. Here type is unknown by
  premise, so C applies the invoice schema to packing lists and customs declarations. The model cannot
  refuse; it must populate invoice fields from non-invoices. Output is schema-valid and largely
  fabricated — **worse** than the current text responses, which at least fail loudly. The "separate
  pass" doesn't rescue it, since nothing identifies which documents were mishandled.
- **D — strongest distractor; guarantees structure but at high cost.** It *does* guarantee a `tool_use`
  response. But (i) it destroys the classification signal — `name` is always `extract_document`, so the
  router must infer type from which fields happen to be populated, reintroducing the ambiguity in
  brittle heuristic code; (ii) it forfeits all `required` enforcement, since no field is universal
  across four types; (iii) it invites cross-type field bleed — a bill-of-lading number written into
  `invoice_number`, both strings, both optional, fully schema-valid and silently wrong. D raises the
  *semantic* error rate to fix a *syntax* problem. (D is defensible when types are nearly identical
  and differ by a field or two — not with four substantially different schemas.)

**Takeaway:** structured output required **+** schema depends on content the model must read → `any`.
Forced named tool is for when *you* already know which extraction must run.

---

## Q2 — Fabricated NPI numbers in healthcare referrals

**Scenario.** A claims processor extracts referral-letter data with a forced named tool. All six
schema fields — `patient_id`, `referring_provider`, `npi_number`, `diagnosis_code`, `referral_date`,
`urgency` (enum: `routine`/`urgent`/`emergent`) — are `required`. Zero parse failures in six months;
every response schema-valid. A compliance audit of 200 extractions finds **31 records** with a
correctly formatted 10-digit `npi_number` that appears nowhere in the source letter and matches no
registered provider — in every case the letter omitted the NPI entirely. It also finds **12 records**
marked `urgency: "routine"` where the letter states no urgency at all.

**What is the most effective fix?**

- **A.** Add a validation step checking each `npi_number` against the national provider registry, and
  retry the extraction with the validation error appended when the lookup fails.
- **B.** Change `npi_number` to `{"type": ["string", "null"]}`, remove it from `required`, add
  `"not_specified"` to the `urgency` enum, and describe in each field when the null / `not_specified`
  value is correct.
- **C.** Add `"pattern": "^[0-9]{10}$"` to `npi_number` and a system prompt instruction: "Do not guess
  or infer values. Only extract information explicitly stated in the source document."
- **D.** Switch to `tool_choice: {"type":"any"}` and add a second tool `report_incomplete_referral`, so
  Claude can signal a letter is missing required fields instead of completing the extraction.

### ✅ Correct answer: B

Root cause is a **hard constraint colliding with absent data**. `required` is enforced structurally —
the model cannot return the object without `npi_number`. When the letter omits it, the model's only
options are violating the schema (impossible) or producing something. It produces something
well-formed. The `urgency` finding is the same bug: the enum offers no value meaning "the letter
doesn't say," so `routine` becomes the landing spot for silence.

Both are one defect: **the schema provides no way to express "this information is not in the source,"
so the model expresses it as data instead.** B fixes it at the layer that caused it. Nullable +
non-required makes absence *representable*; `"not_specified"` does the same for urgency. The field
descriptions matter as much as the type change — they establish that `null` is a *correct answer*.

The real win is downstream: 31 records change from *silently wrong* to *explicitly incomplete*. No
information is recovered (it was never in the letters) — but its absence becomes visible and
routable to human review, instead of propagating into claims adjudication.

### ✗ Why the others are wrong

- **A — retry against the wrong error category.** Registry validation belongs in this pipeline, which
  makes it tempting, but as *the* fix it retries an extraction that cannot succeed. The NPI is not in
  the document; no error feedback grants access to information that isn't there. The model will
  fabricate a different well-formatted number, fail again, and loop with no reachable exit. Per Topic
  4.4: retry corrects **format and structural errors**, never **absent source information**.
- **C — makes it worse, plus wrong layer.** All 31 fabricated values *already* match
  `^[0-9]{10}$`. The regex narrows what the model may invent while leaving `required` intact — more
  constraint, no alternative to inventing. And "do not guess" is a soft instruction against a hard
  constraint: when the prompt says "don't produce a value" and the schema says "you must," the schema
  wins.
- **D — right idea, wrong granularity.** An escape-hatch tool is a legitimate pattern, but this one is
  all-or-nothing at the *document* level: a letter missing only the NPI now routes to
  `report_incomplete_referral`, discarding the patient ID, diagnosis code, referral date, and provider
  name that were present and correct — across ~15% of letters. It also adds discretion (how incomplete
  is "incomplete"?), an unstated judgment boundary of exactly the kind Topic 4.1 warns against. B keeps
  the extraction whole and pushes absence down to the field level, where it occurred.

**Takeaway:** *schema-valid values that don't exist in the source* → look for a `required` field the
source may legitimately lack. *A category assigned when the source is silent* → look for a missing
`"unclear"`/`"not_specified"` enum value. Both are schema-design fixes — not prompt fixes, not retries.

---

## Q3 — Perfectly valid, arithmetically wrong invoices

**Scenario.** An expense platform extracts invoices with a forced named tool. The schema is
well-designed: optional fields nullable, enums include `"other"` + detail, dates normalized to ISO 8601
via description rules. Across 50,000 invoices there has never been a malformed response. Auditing 500
extractions against source PDFs, finance finds: 23 invoices where `line_items` sums to a different value
than `total`; 14 where the vendor's internal reference number was extracted into `invoice_number` and
the actual invoice number into `purchase_order_number`; 9 where `subtotal + tax ≠ total`. All 46 passed
schema validation and posted to the general ledger unflagged.

**Which change most directly addresses the root cause?**

- **A.** Add `"additionalProperties": false` and `"strict": true`, and mark `line_items`, `total`,
  `subtotal`, `tax` as required so the model cannot omit values needed for consistency.
- **B.** Add 3–4 few-shot examples showing correctly extracted invoices where line items sum to the
  total and the invoice number is distinguished from the vendor reference number.
- **C.** Add a `calculated_total` field alongside `stated_total`, add a `conflict_detected` boolean, and
  implement a post-extraction validation step comparing the arithmetic and routing mismatches to human
  review.
- **D.** Split into two forced tool calls — `extract_invoice_header` and `extract_line_items` — so each
  handles a smaller schema and the model has less opportunity to misplace values.

### ✅ Correct answer: C

All 46 failures are **semantically wrong and syntactically perfect**. The schema is doing its entire
job; none of its constraints can express "the numbers must be internally consistent" or "this string
must be the invoice number rather than a different number that also looks like one." JSON Schema
constrains shape and has no notion of arithmetic or real-world referents. So the fix cannot live in
the schema — it must live in a layer that can *check* the output.

- **`calculated_total` alongside `stated_total`** makes the model do the arithmetic explicitly and
  surface it as data. You compare two numbers in code instead of reconstructing intent, converting an
  invisible error into a **detectable** one.
- **`conflict_detected`** gives the model a channel to flag inconsistency it noticed while reading.
  Documents genuinely do contradict themselves; without this the model silently picks one value and you
  never learn a conflict existed.
- **Post-extraction validation + human routing** closes the loop. Note the terminal: arithmetic
  mismatches usually mean the *source* is inconsistent or line items were misread — neither is fixable
  by asking again, so human review is correct, not retry.

The honest goal: you will not reach zero semantic errors through schema design; you reach zero
*undetected* semantic errors through validation.

### ✗ Why the others are wrong

- **A — the reflex answer this question is built to catch.** `strict`,
  `additionalProperties: false`, and more `required` fields all tighten **structural** conformance, a
  dimension already at 100% across 50,000 invoices. You cannot improve a perfect metric, and no JSON
  Schema keyword expresses a cross-field arithmetic relationship. Worse, forcing `subtotal` and `tax`
  to be required means invoices that don't state them separately now get invented values — adding a
  fabrication vector (Q2's bug) to fix a problem it doesn't touch.
- **B — fairest distractor, genuinely partially useful.** Examples contrasting an invoice number with a
  vendor reference number would likely reduce the 14 placement errors, since disambiguation is exactly
  what few-shot is good at (Topic 4.2). It fails as *the* answer on two counts: it does nothing for the
  32 arithmetic errors (examples can't make a model reliably correct at addition over arbitrary line
  counts), and more importantly **examples bias, they don't detect**. Halve the error rate and 23 wrong
  records still post unflagged. The defining failure here is not that errors occur — it's that they
  reach the ledger unnoticed.
- **D — mitigation that makes the primary failure worse.** Smaller schemas do lower misplacement rates,
  but with header and line items in **separate calls** the model never sees the total and the line items
  together, so it cannot notice they disagree — removing the only context where the arithmetic check was
  possible in-model. Also doubles cost and latency and adds a join/partial-failure problem.

**Takeaway:** when a scenario stresses that output is *always schema-valid* and then describes wrong
values, every schema-tightening option is a distractor. Look for the option that **detects** — extract
stated *and* derived values, add an explicit conflict flag, validate after the call. Then decide
deliberately whether mismatches route to retry (format/structure) or to a human (source disagreement).

---

## Q4 — Severity calibration degrades after forcing a tool

**Scenario.** A security code-review agent returns findings with a `severity` enum, `location`, and
`suggested_fix`. Originally `tool_choice: {"type":"auto"}`: Claude wrote two or three sentences of
analysis — weighing whether an unsanitized input was reachable from user-controlled data, whether a
missing check was compensated elsewhere — *then* emitted the `report_findings` call. Severity was
well-calibrated and trusted. Because ~5% of responses came back as text with no tool call, the team
switched to `tool_choice: {"type":"tool","name":"report_findings"}`. The 5% gap closed immediately.
But severity calibration visibly degraded: findings requiring reachability tracing are now frequently
`critical` when the path is unreachable, and genuine issues are sometimes `low`. Schema, field
descriptions, and severity criteria in the system prompt are all unchanged.

**What is the most effective fix?**

- **A.** Revert to `auto` and handle the 5% text-only responses by retrying those requests with the same
  parameters until a tool call is returned.
- **B.** Enable extended thinking so Claude can reason at length before producing the forced tool call.
- **C.** Add a `reasoning` string field as the first property in the `report_findings` schema, described
  as an analysis of reachability and compensating controls to be completed before assigning severity.
- **D.** Add 3–4 few-shot examples showing correctly classified findings, including one where an
  unsanitized input is unreachable and is therefore marked `low`.

### ✅ Correct answer: C

Forcing a tool constrains not just the *format* of the response but *when the model commits*. Under
`auto`, text came first and the tool call second, so reachability analysis genuinely happened before
`severity` was chosen. Under a forced tool the response opens directly with the call — no preamble, so
the deliberation carrying classification quality has nowhere to occur. Nothing about the criteria
degraded (the team correctly noted prompt and schema are unchanged); what was lost is **reasoning
space**, as a side effect of an unrelated fix.

C restores it *inside* the constraint. Models fill properties in order, so a `reasoning` field declared
**first** is generated before `severity` — the analysis happens, conditions the classification, and
becomes durable structured data instead of transient prose (reviewable, auditable).

Two design details: the field must be **first** (placed after `severity` it is post-hoc
rationalization and buys nothing), and its **description should name the analysis to perform** —
reachability, compensating controls — rather than "explain your thinking," which reproduces the
vagueness problem of Topic 4.1.

### ✗ Why the others are wrong

- **A — trades the guarantee back for the calibration.** Retrying with identical parameters re-rolls the
  same distribution: usually a tool call eventually, but unbounded latency and cost on every 20th
  request and still no guarantee. Fundamentally, `auto` was the wrong setting for a pipeline that
  *requires* structured output; reverting re-introduces a defect the team had correctly fixed. The right
  answer keeps both properties.
- **B — hard trap; the configuration is invalid.** It sounds right (missing reasoning, and extended
  thinking is the reasoning feature), but **extended thinking is not compatible with forced tool
  choice** — it requires `tool_choice: auto` (or `none`). Making B valid means abandoning the forcing
  that closed the 5% gap, which collapses B into A.
- **D — right tool, wrong diagnosis.** Few-shot is the correct instrument for calibrating judgment on
  ambiguous cases, and an example marking an unreachable input `low` is exactly the "show the tie-break
  rule" pattern from Topic 4.2. But calibration was good and broke at a known moment from a known
  change — the scenario states the cause. Examples demonstrate *conclusions* without restoring the
  *process*, and reachability analysis is per-file work no fixed example can pre-compute. D addresses
  *what good output looks like*; C addresses *whether the model gets to think before producing it*. Only
  the second one broke.

**Takeaway:** quality dropping right after a switch to `any` or a forced named tool on a judgment-heavy
task → diagnosis is lost preamble reasoning; fix is a leading `reasoning` field. Not reverting
`tool_choice`, and not extended thinking (which the forcing rules out).

---

## Q5 — Support ticket enum after a product expansion

**Scenario.** Tickets are routed via a forced extraction tool with a closed `category` enum:
`["billing", "authentication", "data_export", "api_errors", "performance"]`. After launching a mobile
app and a partner integrations program, an audit of 400 auto-routed tickets finds two distinct problems:
**61 tickets** concern topics with no listed category (push notification failures, partner OAuth
handshake issues, mobile offline sync), each assigned an existing value (mostly `authentication` or
`api_errors`) and routed to a team that could not resolve it; **28 tickets** are genuinely too vague to
categorize ("it's broken again, please fix"), and were assigned `performance`, which is acting as a
catch-all. Every response has been schema-valid throughout. The team wants routing accuracy restored
**and** wants to avoid re-auditing manually every time the product surface expands.

**Which change best addresses both findings?**

- **A.** Change `category` from an enum to a free-form string with a description listing the five known
  categories as guidance, allowing Claude to name new categories as they arise.
- **B.** Expand the enum to include `push_notifications`, `partner_oauth`, `mobile_sync`, and
  `vague_request`, covering the categories surfaced by the audit.
- **C.** Add `"other"` and `"unclear"` to the enum, plus a nullable `category_detail` string described as
  required when `category` is `"other"`, with descriptions distinguishing "a real category not in the
  list" from "the ticket does not state enough to determine one."
- **D.** Keep the enum closed and add a `confidence` number field, routing any ticket below a 0.7
  confidence threshold to a human triage queue.

### ✅ Correct answer: C

The audit describes **two different causes** that both surface as "wrong enum value," and each needs its
own mechanism:

- The 61 tickets are a **coverage** failure — the category exists in the world, it just wasn't in the
  list. → `"other"` + `category_detail`.
- The 28 tickets are an **ambiguity** failure — no category is determinable from the source. → `"unclear"`.

Collapsing them would be wrong: a partner-OAuth ticket is fully categorizable (you hadn't defined the
bucket); "it's broken again" is not categorizable by anyone. They need different handling — new-category
review vs. clarification request back to the customer — so they need different values. The descriptions
doing the disambiguation matter as much as the values; without them the two blur into one
undifferentiated bucket.

What satisfies the *second* requirement is `category_detail`: it converts the enum from a static list
into an **instrumented** one. Aggregate the detail strings weekly, see `push_notifications` appear 40
times, promote it into the enum deliberately. The failure becomes self-reporting. Note what a closed enum
costs: it cannot distinguish "no such tickets exist" from "we miscategorize every one of them" — both
look identical in the data. That is the deeper reason closed enums without escape hatches are an
anti-pattern: they don't just misclassify, they destroy the signal that would have told you.

### ✗ Why the others are wrong

- **B — most tempting; fixes the past, not the future.** It does fix the 400 audited tickets, but
  retrospectively, and it fails the requirement the team stated explicitly. The enum stays closed, so
  the next launch reproduces the identical failure — silent miscategorization discovered only by another
  manual audit months later. That treats the audit as the detection mechanism, which is what needs
  replacing. `vague_request` is half credit: right idea for the 28 tickets, but without `"other"` it
  misses the extensibility case entirely.
- **A — solves coverage by discarding what the enum provided.** You'll get `partner_oauth`,
  `Partner OAuth`, `oauth-partner`, `OAuth (partner integrations)` — semantically identical,
  structurally distinct, every one a routing-table miss. It also breaks the ambiguity case: with no
  constrained vocabulary a vague ticket gets some invented label rather than a recognizable "can't tell"
  signal, so those 28 remain unroutable *and* become unidentifiable. C keeps the closed set for the known
  95% and adds *controlled* escape hatches — the whole point of the pattern.
- **D — confidence-based filtering, which Topic 4.1 addresses directly.** Self-reported confidence is
  poorly calibrated and doesn't map to the actual decision boundary. Concretely: a partner-OAuth ticket
  *is* clearly about authentication, so the model likely reports high confidence in `authentication` and
  sails past the 0.7 threshold. Confidence measures how well the ticket matches the *available* options,
  not whether the right option is present. It also yields a number where you needed a category, so even
  the vague tickets it catches tell you nothing about *what* was missing — the manual re-auditing
  requirement is untouched.

**Takeaway:** two escape hatches, two jobs. `"other"` + detail = *extensible category space*, and the
detail field is what makes it self-instrumenting. `"unclear"`/`"not_specified"` = *ambiguous or silent
source*. If a scenario shows both symptoms, the answer must include both — expanding the enum, going
free-form, and adding a confidence score are all distractors.

---

# Cross-topic connections

- **Topic 4.1 (explicit criteria):** severity/category criteria belong in schema field `description`s,
  adjacent to the decision point. Confidence-score filtering is a distractor in both topics.
- **Topic 4.2 (few-shot):** examples calibrate *judgment and form*; schemas *guarantee structure*. When a
  question needs a guarantee, few-shot is the wrong tool; when it needs disambiguation of a genuinely
  ambiguous case, few-shot is the right one.
- **Topic 4.4 (validation & retry):** picks up exactly where this topic stops — semantic errors. Retry
  corrects format/structural errors; it cannot correct absent source information.
- **Domain 1.4 (enforcement vs. guidance):** `tool_choice` is the API-layer instance of "enforce
  structurally when correctness depends on it, rather than asking nicely."
- **Domain 2.1 (tool interface design):** `description` fields are prompt engineering for a model reader
  — true whether the tool is executed or used purely as a schema carrier.

---

# Score

**5 / 5 correct**, including the two hardest items — the lost-preamble diagnosis (Q4) and separating
coverage from ambiguity (Q5).
