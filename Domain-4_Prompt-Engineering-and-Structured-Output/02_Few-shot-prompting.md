# Topic 4.2 — Few-shot prompting

**Domain:** 4 — Prompt Engineering & Structured Output (20% of exam)
**Topic number:** 2 of 6 within Domain 4 · Topic 20 of 30 overall

---

## 1. Core mental model

A few-shot prompt is **not** "teaching Claude the task." Claude already knows how to
summarize, classify, and extract. What examples actually do is **collapse the space of
acceptable outputs**.

Three distinct things you can specify about a task, each with its own right tool:

| What you're specifying | Best tool |
|---|---|
| **What** to do (the task) | Instructions |
| **Which** things count (the boundary) | Explicit criteria — *Topic 4.1* |
| **What the output should look like** (form, tone, depth, shape) | **Few-shot examples** |

**The dividing line the exam tests most:**

> **Examples are right when the requirement is easier to *show* than to *state*.**
> **Instructions are right when the requirement is easier to *state* than to *show*.**

- "Never include the customer's account number" — trivial to state, nearly impossible to
  teach by example (absence carries no signal). → **Instruction.**
- "Match our house voice" / "this is the right level of detail" — nearly impossible to
  state precisely, captured instantly by two good examples. → **Few-shot.**

Recurring anti-pattern: **using examples to encode a rule, or using prose to encode a
style.** Both are tool/problem mismatches.

---

## 2. What examples actually control

**Examples are strong at:**

- **Format and shape.** Field ordering, delimiters, bullet vs. prose, presence of headings.
  *(Caveat — see §6: if output is machine-parsed, few-shot is the weaker tool. Schemas are
  the strong tool. Examples cover what a schema can't express.)*
- **Tone and register.** Formal vs. casual, hedged vs. decisive. "Be empathetic but
  concise" produces wild variance; two exemplar replies pin it down.
- **Depth and length calibration.** "Be brief" is relative. Two examples of the *right*
  brevity beat a word count, which is a blunt constraint that fights the content.
- **Edge cases and decision boundaries.** Highest-value use. An example showing an
  *ambiguous* input and its *correct* handling teaches a tie-break rule prose struggles with.
- **Implicit reasoning depth.** Examples showing a one-line justification before the verdict
  propagate that process. Examples transmit process, not just answers.

**Examples are bad at:**

- **Negative constraints / prohibitions.** "Never do X" cannot be shown. Absence is not a
  signal — Claude can't tell a required omission from a coincidental one.
- **Coverage rules.** "Consider all five dimensions" is an instruction; examples that happen
  to cover five may be read as coincidence.
- **Hard guarantees.** Examples bias, they don't enforce. Anything that must *always* hold
  belongs in a schema, validator, or hook. *(Cross-links Domain 1.4, 2.3: match enforcement
  to stakes.)*

---

## 3. Quantity — how many examples

Expected reasoning, in order:

- **Zero-shot** — correct when the task is common, output is free-form prose, and there's no
  house-specific convention. Adding examples here costs tokens and *narrows* output
  diversity for no gain.
- **1 example** — anchors format. Often enough when only the shape is under-specified.
- **3–5 examples** — the workhorse range and the right answer to most "how many" questions.
  Enough to show variation, cheap enough to keep in every request.
- **Beyond ~5–8** — sharp diminishing returns, *unless* the extra examples buy **coverage of
  distinct categories or edge cases** rather than repetition.

**Key nuance the exam rewards: example diversity matters more than example count.** Ten
near-identical examples are worse than four spanning the real distribution.

**Overfitting / over-narrowing** is the high-count failure mode: Claude treats examples as a
template to pattern-match rather than a demonstration of a principle. Symptoms — novel
inputs force-fit into the closest example's mold; incidental surface features reproduced
(an example that happened to start with "Summary:" makes every output start with "Summary:").

---

## 4. Selection — which examples

Where most real-world few-shot prompts fail; the exam probes this via failure-diagnosis
scenarios.

**Cover the real distribution, including classes the model gets wrong.** Example
distribution becomes a **prior over the outputs**. Five categories with only three
represented → the two missing categories get under-predicted. Four of five examples
`high` severity → over-prediction of `high`.

**Exam-critical diagnostic — distinguishing 4.2 bugs from 4.1 bugs:**

> **Skewed examples → biased distribution. Vague criteria → inconsistent boundary.**
>
> - Errors **systematically directional** (all leaning one way) → example-set problem.
> - Errors **inconsistent** (same input classified differently run to run, no lean) →
>   criteria problem.
> - Balanced examples + vague criteria → fix criteria (4.1).
> - Crisp criteria + skewed examples → fix examples (4.2).

**Prefer hard/boundary cases over easy ones.** An example Claude would have gotten right
anyway teaches nothing. The examples that earn their tokens are near-misses: the input that
*looks* like category A but is B, showing the discriminating feature.

**Use real data, not invented data.** Synthetic examples are cleaner than production input —
shorter, better punctuated, unambiguous. Claude calibrates to a distribution that doesn't
exist and degrades on messy real input. Sourcing from actual logged inputs (especially
failures) is the strongest fix.

**Examples must be correct.** An erroneous example is worse than no example — it's an
authoritative-looking instruction to do the wrong thing, and it *will* be reproduced.
**When a few-shot prompt produces a consistent, specific wrong behavior, audit the examples
first.**

---

## 5. Construction and placement

### Structural separation

Examples must be unmistakably delimited from the live input and from each other. XML tags
are the canonical Claude idiom:

```xml
<examples>
  <example>
    <input>Payment failed 3x, card expired last week</input>
    <output>category: billing
    severity: low
    reason: self-serve card update resolves this</output>
  </example>
  <example>
    <input>Checkout 500s on all Safari users since deploy</input>
    <output>category: bug
    severity: high
    reason: total failure for a browser segment</output>
  </example>
</examples>

Now classify this ticket:
<ticket>{{TICKET}}</ticket>
```

Why it matters beyond tidiness — without clear boundaries Claude can (a) treat example
content as part of the live input, or (b) leak example content into the answer.

> **Diagnostic: if output contains content from an example, the root cause is a delimiting
> failure, and the fix is structural separation — not an instruction telling it to ignore
> the examples.**

### Other construction rules

- **Input–output pairing must be explicit.** Every example shows both sides. A list of "good
  outputs" with no inputs teaches style but not the mapping.
- **Consistency across examples is load-bearing.** Same field order, casing, delimiters,
  level of detail. **Inconsistency across examples is read as permitted variation** — if one
  example writes `severity: high` and another `Severity: HIGH`, both appear in production.

### Placement

**Order: system/instructions → examples → variable input.**

- *Quality reason:* the live input is what Claude should attend to at generation time.
- *Cost reason:* a stable example block early forms a perfect **cache prefix** — identical
  on every request, so a cache breakpoint after it means those tokens are written once and
  read cheaply thereafter. Putting examples *after* the variable input destroys the static
  prefix and eliminates caching entirely.

> Favorite exam framing: "the few-shot examples make each call expensive" → the answer is
> usually **cache the static example block**, not **delete examples**. The
> accuracy-vs-cost tradeoff is often a false one.

---

## 6. The central decision boundary: few-shot vs. structured output

Domain 4's internal fault line; 4.3 depends on it.

Two mechanisms for getting reliable JSON:

| Mechanism | What it does |
|---|---|
| **Few-shot examples of JSON** | Shows the shape. Biases toward it. **Guarantees nothing.** Claude may add a preamble, wrap in a code fence, add/omit a field, or produce a subtly wrong type. |
| **Tool use with `input_schema` + `tool_choice`** | The API constrains generation to the schema. Field names, types, required fields, and enums enforced structurally. |

> **Rule: if the output is consumed by a program, use structured output via tool_use. If the
> output is consumed by a human, use few-shot to shape it.**

Scenario pattern: "our pipeline breaks ~5% of the time because Claude occasionally adds text
before the JSON — we added more JSON examples and it still happens." → **Stop fixing a
structural problem with a probabilistic tool.** Move to tool_use. Adding a sixth example is
the attractive distractor because it's the obvious next step, and it's wrong for the reason
that recurs across this whole exam: **prompting guides, mechanisms guarantee.**

**Where examples still help alongside schemas:** schemas constrain **shape**, not **content
quality**. A schema can require a `summary` string; it can't make the summary the right
length, granularity, or register. The mature pattern is **schema for structure + few-shot
for judgment within the fields.** Deleting examples because "the schema handles it" degrades
output *silently* — nothing errors, everything validates, quality drops unnoticed.

---

## 7. Anti-patterns checklist

| Anti-pattern | Why it fails | Fix |
|---|---|---|
| Examples used to encode a prohibition | Absence isn't a learnable signal | State the rule explicitly + enforce programmatically |
| Examples used where a schema is needed | Bias ≠ guarantee | tool_use + JSON schema |
| All examples from one class | Becomes a prior; skews predictions | Balance across the real distribution |
| Only easy/clean examples | Teaches nothing; miscalibrates to tidy input | Use real, hard, boundary cases |
| Inconsistent formatting between examples | Reads as permitted variation | Normalize every example |
| Undelimited examples | Content leaks into the answer | XML tags, one per example, tag the live input too |
| An incorrect example | Reproduced faithfully | Audit examples when a *specific consistent* error appears |
| 20 examples "to be safe" | Overfitting + token cost | 3–5 diverse; add only for new coverage |
| Examples after the variable input | Wrong attention placement; kills cache prefix | instructions → examples → input |
| Adding examples to fix a *criteria* problem | Wrong root cause | Enumerate inclusions/exclusions (4.1) |
| Deleting examples to cut cost | False tradeoff | Prompt caching on the static prefix |

---

## 8. Worked scenarios

**A — Support agent, inconsistent tone.** Replies range from a curt sentence to five
paragraphs; prompt says "be helpful and professional." → **Few-shot is exactly right.** Tone
and length calibration are show-not-tell properties. Add 3 real exemplar replies spanning a
simple question, an angry customer, and an unresolvable request.

**B — Research subagent returns findings the coordinator can't merge.** Sometimes prose,
sometimes bullets, sometimes missing the source URL. → **Few-shot is the weak fix.** The
coordinator is a program consuming this; the answer is a defined output contract via
structured output, with examples only to calibrate the depth of each field. *(Cross-links
Domain 1.3: engineer the brief and the output contract.)*

---

# Practice Questions

---

## Question 1 — JSON pipeline failing 4% of the time

A fintech company runs a Claude-based pipeline that reads incoming merchant support emails
and outputs a triage record consumed directly by their ticketing system's API. The prompt
includes eight few-shot examples, each showing an email and the corresponding JSON triage
record.

The pipeline fails roughly 4% of the time. Failures show three distinct patterns: Claude
sometimes prefixes the JSON with a sentence like *"Based on the email, here is the triage
record:"*, sometimes wraps the JSON in a markdown code fence, and once emitted
`"priority": "urgent"` when the schema's enum only permits `low`, `medium`, `high`.

The team's proposed fix is to add four more JSON examples, including one that explicitly
shows the `priority` field using only permitted values.

What is the most effective action?

- **A.** Add the four examples as proposed, and additionally add an instruction stating
  "Output only raw JSON with no preamble and no code fences."
- **B.** Convert the extraction to a tool definition whose `input_schema` declares the fields
  and the `priority` enum, and set `tool_choice` to force that tool.
- **C.** Keep the few-shot examples but add a post-processing step that strips leading prose
  and markdown fences before parsing, and coerces unexpected enum values to the nearest
  valid one.
- **D.** Reduce the example count from eight to three, selecting only the cleanest examples,
  since eight examples cause the model to overfit to incidental surface features like code
  fences.

### ✅ Correct answer: **B**

*(My answer: D — incorrect.)*

**Why B is correct.** All three failure patterns are *structural* violations: extraneous
preamble, wrapper syntax, out-of-enum value. Those are exactly the guarantees
`tools` + `input_schema` provides. With a forced tool call, output arrives as a `tool_use`
block whose `input` is constrained to the schema — there is no channel for a preamble or
code fence to exist in, and the enum is enforced at generation rather than suggested. The
output is consumed by a program, so it needs a mechanism, not a bias. A 4% failure rate is
the signature of a probabilistic tool doing a deterministic job; no number of examples drives
that to zero, because each example only shifts probability mass.

**Why A is wrong.** The strongest distractor because it's what teams actually do. It stacks
two probabilistic fixes — more examples plus an explicit negative instruction — on a problem
needing a structural one. It's also the §2 anti-pattern stated outright: "never output a
preamble" is a *prohibition*, and prohibitions are precisely what examples cannot teach.
The added instruction helps somewhat, and "helps somewhat" is the whole problem — you land
at 1–2% instead of 4%, and the pipeline still has an unhandled failure mode.

**Why C is wrong.** Treats the symptom and introduces a silent corruption path. Stripping
fences is defensible defensive coding, but **coercing an unexpected enum to "the nearest
valid one"** means a record Claude believed was `urgent` is silently written as `high` — with
no signal to anyone, in a fintech triage system. Validation that repairs data by guessing is
worse than validation that rejects. *(Previews 4.4: a validation loop should detect and retry
or escalate, not silently patch.)*

**Why D is wrong (my error).** Misdiagnoses the root cause. Overfitting shows up as inputs
being *force-fit into an example's mold* — novel input classified as whatever the nearest
example was, incidental surface features reproduced. Here the model does the opposite: it
*departs* from the examples' format. Code fences and preambles are Claude's general-purpose
helpfulness leaking through, not artifacts copied from eight tidy examples. And even if
count were a factor, three examples still leaves a probabilistic guarantee on machine-parsed
output — maybe 4% → 3%, pipeline still breaks. "Selecting only the cleanest examples" is
independently a bad instinct: clean examples miscalibrate to a tidier input distribution than
production has.

> **Generalization:** when a scenario describes a **machine consumer** plus a
> **small-but-persistent failure rate**, the answer is almost never "improve the prompt."
> Ask what layer can *guarantee* the property.

---

## Question 2 — News classifier over-predicting one category

A media monitoring company classifies news articles into `politics`, `business`,
`technology`, `health`, `sports`. The prompt contains clear written definitions of all five
categories with explicit inclusion and exclusion rules, plus six few-shot examples.

In production, `technology` is heavily over-predicted: articles about pharmaceutical R&D,
sports analytics platforms, and campaign data operations are labeled `technology` rather than
`health`, `sports`, `politics`. Articles unambiguously about a single domain are classified
correctly.

The six examples are: two technology, one business, one politics, one health, one sports —
all clear-cut, single-domain articles.

Most likely root cause and best fix?

- **A.** The category definitions are too vague at the boundaries; rewrite them to enumerate
  explicitly which cross-domain articles belong to which category.
- **B.** The example set is skewed toward `technology` and contains no boundary cases;
  rebalance to one example per category and replace the clear-cut examples with cross-domain
  articles showing the discriminating feature.
- **C.** Six examples is too few for a five-category task; expand to at least fifteen so each
  category has three representatives.
- **D.** The task should use `tool_choice` with an enum-constrained schema so Claude cannot
  drift toward `technology`.

### ✅ Correct answer: **B**

*(My answer: B — correct.)*

**Why B is right.** Two independent defects, both signposted in the scenario.

*The skew:* technology has twice the representation of any other category. Example
distribution acts as a **prior over outputs** — when uncertain, Claude drifts toward whatever
the example set made salient. The over-predicted category being exactly the over-represented
category is the fingerprint of this bug.

*The missing boundary cases:* all six examples are clear-cut, and the scenario says clear-cut
articles are *already* classified correctly. Every example teaches a case that needed no
teaching. Meanwhile all failures are cross-domain articles, and there is not one cross-domain
example. Pharma R&D, sports analytics, and campaign data ops all have a technology *surface*
and a non-technology *subject*, and nothing demonstrates which wins.

**Why A is wrong.** The sharpest distractor — it's the right fix for a *different* root
cause, the one from 4.1. The tell: definitions already include explicit inclusion/exclusion
rules, and unambiguous articles classify correctly. Crisp criteria producing correct behavior
on clear cases means the boundary is *stated*, just not *demonstrated*. Apply the triage
rule: errors here are **systematically directional** (all lean technology) → distribution
problem, not criteria problem. Practically, enumerating every cross-domain combination in
prose for 5 categories means describing 20 pairwise boundaries — exactly where showing beats
stating.

**Why C is wrong.** Fixes count without fixing composition. Fifteen clear-cut examples would
remove the skew (partial credit) but still contain zero boundary cases, so cross-domain
failures continue. Inverts §3: **diversity matters more than count.** Three well-chosen
cross-domain examples beat fifteen clear-cut ones at a fifth of the token cost. "More
examples" as the reflex response to any few-shot problem is the habit being tested against.

**Why D is wrong.** An enum schema guarantees output is one of five permitted strings; it has
no opinion on *which*. Every misclassification here is a well-formed, in-enum, semantically
wrong answer. Inverse of Q1: there the failure was structural so the structural tool was
right; here the failure is judgment, and structure can't adjudicate judgment.

---

## Question 3 — Draft assistant: tone variance + internal detail leakage

An enterprise SaaS company uses Claude to draft replies to customer emails for human agents
to review and send. The prompt says: *"Be empathetic but concise. Acknowledge the customer's
frustration. Do not make commitments about timelines. Match a professional but warm tone.
Never disclose internal ticket IDs or engineer names."*

Two recurring complaints: (1) draft length varies wildly — one terse sentence to six
paragraphs for the same class of issue; (2) roughly once a week a draft includes an internal
engineer's name pulled from ticket history passed in as context.

Budget for one substantial change. Which approach best addresses both?

- **A.** Add four few-shot examples of ideal replies covering a range of issue types, and rely
  on the examples to demonstrate both the target length and the omission of internal details.
- **B.** Add four few-shot examples of ideal replies to calibrate length and tone, and
  separately move the internal-details prohibition to a programmatic check that scans the
  draft against the known engineer names and ticket IDs before it reaches the agent.
- **C.** Replace the prose guidance entirely with a strict word-count range (120–180 words)
  and an instruction repeated at both the start and end of the prompt stating that internal
  details must never appear.
- **D.** Redact engineer names and internal ticket IDs from the ticket history before it is
  passed into the prompt, and add a word-count range to control length.

### ✅ Correct answer: **B**

*(My answer: B — correct.)*

**Why B is right.** The two complaints are *different kinds of problem* and must not be
solved with one tool.

*Length/tone variance* is a calibration problem. "Empathetic but concise," "professional but
warm" are relative terms with no fixed referent, and the evidence (wild variance on the same
class of issue) shows prose isn't pinning anything down. Canonical show-don't-tell case —
four exemplar replies set the target instantly.

*Leaking engineer names* is a prohibition with real consequences — the one thing examples
structurally cannot teach, since an example that omits a name carries no signal the omission
was mandatory. "Roughly once a week" is the same tell as Q1: a low-but-persistent rate means
a probabilistic mechanism guards something needing a deterministic one. Match enforcement to
stakes (Domain 1.4) → programmatic check.

**Why A is wrong.** B minus the enforcement layer; fails on the half examples can't reach.
Fixes length, leaves disclosure roughly where it is. "Rely on the examples to demonstrate the
omission" is the anti-pattern stated out loud — you cannot demonstrate an absence.

**Why C is wrong.** A hard word-count range is the blunt-constraint failure from §2 — it
clamps length but fights the content (padding simple issues, truncating complex ones) and
does nothing for *tone*, half of complaint 1. "Repeat the instruction at start and end" is
prompt-stacking: doing more of what already isn't working. Repetition raises compliance
somewhat; it doesn't convert a bias into a guarantee, and a weekly PII-adjacent leak isn't
solved by asking more emphatically.

**Why D is wrong — with a real caveat.** Its first half is genuinely strong: **not putting
sensitive data in context at all is a better security posture than scanning on the way out** —
if the name never enters the prompt there is no leak path, whereas an output scanner is a net
evadable by paraphrase. In a production design review, input redaction would be the
recommendation, and the strongest real answer is *both* layers. D fails as the exam answer on
its second half: it solves length with the same word-count blunt instrument as C, ignores
tone entirely, and abandons few-shot calibration — the actual subject of this topic.
**Tiebreak when two options both contain a good idea: pick the one that leaves nothing
unfixed.**

---

## Question 4 — Nightly extraction job, growing token spend

A data platform team runs a nightly Claude job extracting structured findings from ~4,000
compliance documents. The prompt is a fixed system instruction, a block of twelve few-shot
examples showing document excerpts and their extracted findings, then the document under
analysis. Output goes through a forced tool call with a strict schema.

Extraction accuracy is good. Finance flags that token spend has grown uncomfortably; the
example block accounts for a large majority of input tokens on every one of the 4,000 calls.
An engineer proposes cutting twelve examples to four.

What should the team do?

- **A.** Cut to four examples as proposed, selecting the four covering the most common
  document types, and accept a small accuracy reduction as the cost of the savings.
- **B.** Keep all twelve examples but move the example block to the end of the prompt,
  immediately before the schema definition, so the variable document content appears earlier
  and compresses better.
- **C.** Keep the example block unchanged and in its current position, and apply prompt
  caching with a cache breakpoint after the example block so the static prefix is written
  once and read cheaply on subsequent calls.
- **D.** Remove the example block entirely, since the forced tool call with a strict schema
  already guarantees the output structure, making the examples redundant.

### ✅ Correct answer: **C**

*(My answer: C — correct.)*

**Why C is right.** Every precondition for prompt caching is present:

- The prefix is **static** — same system instruction and twelve examples on all 4,000 calls.
- The variable content sits **after** it — the document is last.
- Volume is **high** — 4,000 nightly calls amortize the one-time cache-write cost.
- The prefix is **large** — a majority of input tokens, so savings apply to the bulk of spend.

A cache breakpoint after the example block means the prefix is written once and read at a
fraction of normal input cost thereafter. You keep the accuracy twelve examples bought and
remove nearly all recurring cost — the proposed accuracy-for-cost trade isn't one you have to
make. This is §5's ordering rule paying off on the cost side; correct placement is *what made
caching available*.

**Why A is wrong.** Accepts a real accuracy loss for a problem with a lossless solution. It
sounds like responsible engineering, which is what makes it the primary distractor — the exam
rewards noticing when an apparent tradeoff is false. Its selection criterion is independently
weak: "the four covering the most common document types" optimizes for cases the model
already handles and discards the boundary cases that earn their tokens (§4). If you truly had
to cut, you'd cut redundant clear-cut examples and *keep* the hard ones.

**Why B is wrong.** Actively harmful. Moving examples after the variable document destroys
the static prefix — every request's prefix now begins with a different document, so there's
no reusable cached segment and cost goes *up*. The stated rationale is incoherent: input
tokens aren't compressed by position and the count doesn't change. **Watch for distractors
supplying a mechanism-flavored justification for an action with no mechanism behind it.**

**Why D is wrong.** Takes a correct principle and over-applies it. The schema does guarantee
structure — that's §6, and why accuracy is already good. But **schemas constrain shape, not
content quality.** A schema can require a `finding` string and `severity` enum; it cannot
make the finding the right granularity, teach which passages constitute a reportable finding
versus background, or show how a borderline clause should be characterized. The twelve
examples do the *judgment* work inside the fields. Arguably the worst option because it
degrades **silently**: nothing errors, the schema always validates, and quality drop surfaces
weeks later in a compliance review.

---

## Question 5 — Incident summaries: leaked example content + inconsistent dates

**(Multiple response — select TWO.)**

A logistics company converts unstructured driver incident reports into standardized
summaries. The prompt includes three few-shot examples, each formatted as a block of text
containing the raw report followed by the expected summary, separated by a blank line, with
no tags or explicit labels. The document under analysis is appended at the end after another
blank line.

Two intermittent defects: (1) occasionally the output summarizes an incident that appears in
one of the *examples* rather than the actual report; (2) summaries are inconsistently
formatted — the three examples themselves render the incident date differently
(`2026-03-14`, `March 14, 2026`, `14/03/2026`), and production output uses all three formats
unpredictably.

Which two actions together resolve the observed problems?

- **A.** Wrap each example in explicit delimiters — e.g.
  `<example><input>…</input><output>…</output></example>` — and wrap the report under analysis
  in its own distinct tag.
- **B.** Increase the number of examples from three to eight so the model has more signal
  about which content is illustrative and which is the live input.
- **C.** Normalize all three examples to a single date format, and state the required date
  format in the instructions.
- **D.** Add an instruction at the end of the prompt reading "Summarize only the final report,
  not the examples, and use ISO-8601 dates."

### ✅ Correct answers: **A and C**

*(My answer: C and D — partially correct; C right, D wrong.)*

**Why A is correct (the half I missed).** Defect 1 is a **delimiting failure** with a
structural cause. With examples separated only by blank lines and no labels, nothing marks
where illustrative content ends and live input begins — Claude infers the boundary from
position and whitespace alone, and intermittently infers it wrong. That is not a judgment
error you can coach; it's missing structure you must supply. Explicit tags fix it at the
source, and the *second half of A matters as much as the first*: tagging examples while
leaving the report as trailing untagged text still relies on position.

> **Memorize: output containing example content → root cause is delimiting → fix is
> structural separation.**

**Why C is correct.** The three examples are the *source* of the variance — production uses
all three formats because the prompt showed all three and thereby licensed them
(inconsistency across examples reads as permitted variation, §5). Normalizing removes the
license; stating the format in the instructions makes the requirement explicit rather than
merely demonstrated, so novel date phrasings in the raw report are handled too. Format is one
of the few properties where you legitimately want both mechanisms.

**Why D is wrong (my error).** A strong distractor — it addresses both defects in one line
and its second clause is real progress. But it fails on both halves for the same reason.

*On leakage:* "summarize only the final report, not the examples" asks Claude to correctly
resolve an ambiguity the prompt itself created, without giving it any reliable way to tell
the content apart — "final report" is still positional. On an intermittent failure that turns
a frequent misread into a less frequent one. A removes the ambiguity so there's nothing left
to resolve.

*On dates:* more instructive. D adds an ISO-8601 instruction while **leaving three examples
that visibly contradict it**. Demonstrated behavior competes with stated behavior. That
conflict is exactly why C specifies *both* normalizing the examples and stating the rule —
the instruction sets the target, and removing the contradictory demonstrations is what lets
it hold.

**Why B is wrong.** Makes defect 1 *worse*: more undelimited example text means more content
mistakable for the live input, and the live report is pushed further from where it was
already being confused. The stated rationale inverts the real relationship — **signal comes
from structure, not quantity.** B also does nothing for date inconsistency and would likely
amplify it if new examples inherit the same sloppy formatting.

---

# Practice recap — 4/5

| Q | My answer | Correct | Judgment being tested |
|---|---|---|---|
| 1 | D | **B** | Machine consumer + persistent low failure rate → structural mechanism, not more examples |
| 2 | B ✓ | B | Directional errors + skewed examples → distribution bug, not criteria bug |
| 3 | B ✓ | B | Split the problem: examples calibrate, enforcement prohibits |
| 4 | C ✓ | C | Correct ordering makes caching available — the accuracy/cost tradeoff was false |
| 5 | C+D | **A+C** | Example content in output → delimiting failure, fixed structurally not by instruction |

**Pattern in both misses:** Q1 and Q5 each had a *structural* defect, and both times the pull
was toward a prompt-level fix. Q1's D and Q5's D are the same instinct — adjust what you
*say* to the model — applied where the fix is to change the model's *inputs or output
channel*. Carry this into 4.3 and 4.4, where that boundary is the whole subject.

---

# Cross-references

- **4.1 (Explicit criteria / false positives)** — the sibling diagnostic. Vague criteria →
  inconsistent boundary; skewed examples → biased distribution. Directionality of the errors
  tells you which one you have.
- **4.3 (Structured output via tool_use & JSON schemas)** — §6 is the bridge; few-shot for
  human consumers, schemas for program consumers, both together for structure + judgment.
- **4.4 (Validation, retry, feedback loops)** — Q1's option C previews the anti-pattern:
  validation that silently repairs by guessing is worse than validation that rejects.
- **Domain 1.3 (Subagent invocation & context passing)** — subagent output shaping is an
  output-contract problem, not a few-shot problem.
- **Domain 1.4 / 2.3 (Enforcement layers)** — "prompting guides, mechanisms guarantee";
  prohibitions belong in hooks/validators, never in examples.
- **Domain 5 (Context management)** — prompt caching on the static example prefix; ordering
  is a cost decision as well as a quality one.
