# Topic 4.4 — Implement validation, retry, and feedback loops for extraction quality

**Domain:** 4 — Prompt Engineering & Structured Output (20% of exam)
**Task Statement:** 4.4
**Topic:** 22 of 30 overall · 4th of 6 in Domain 4

---

## Core Idea

**Tool use guarantees the *shape* of the output. Nothing guarantees the *content*.**
Validation, retry, and feedback loops are three distinct layers built on top of that gap — and **each layer fixes a different class of failure**. The exam tests whether you can classify the failure *before* choosing the remedy.

Topic 4.3 ended with "schemas guarantee shape, never truth." Topic 4.4 is the answer to "so what do you do about truth?"

---

## 1. The Failure Taxonomy (drives every answer)

| Class | Example | Fixed by |
|---|---|---|
| **Syntax / schema errors** | Malformed JSON, missing required field, string where number expected | **Already eliminated** by `tool_use` + JSON schema. If a question implies retry logic is needed for JSON parse errors, the real answer is "you're not using tool use." |
| **Semantic errors** | `line_items` sum to $4,830 but `total` says $4,530; vendor name in `bill_to`; date parsed MM/DD when doc is DD/MM | **Validation code + retry with error feedback.** The model *had* the information and mis-assembled it. |
| **Absent information** | Invoice genuinely has no PO number because it's on a separate cover sheet not passed in | **Retry is useless and actively harmful.** Fix = nullable schema field, route to human, or fetch the missing source. |

> **The single highest-yield judgment in this topic:** a retry only helps when the model **had** the information and got it wrong. It **never** helps when the information was never there.

---

## 2. Where Validation Lives

Validation is **deterministic code in your harness** — not a prompt instruction, not the schema. Schema validation is free (tool use enforces it); *business* validation is yours to write.

```python
def validate(extraction, doc):
    errors = []
    computed = sum(li["amount"] for li in extraction["line_items"])
    if abs(computed - extraction["total"]) > 0.01:
        errors.append(
            f"line_items sum to {computed} but total is {extraction['total']}. "
            f"Either a line item is missing or an amount was misread."
        )
    if extraction["invoice_date"] > extraction["due_date"]:
        errors.append("invoice_date is later than due_date — check date format (DD/MM vs MM/DD).")
    return errors
```

Three properties the exam cares about:

1. **It's programmatic** → a *guarantee*, not a probability. (Same logic as hooks in 1.5: when you need determinism, don't ask the model.)
2. **Error messages are written for a model reader.** `"Validation failed: E_SUM_MISMATCH"` is useless to Claude. `"line_items sum to 4830 but total is 4530 — a line item may be missing"` is a *corrective prompt*. This is the 2.2 principle (errors are prompts) applied to your own validator.
3. **It runs on every extraction**, not just suspicious ones. Confidence-based sampling is not validation.

---

## 3. Retry-with-Error-Feedback: the Exact Payload

The retry request must contain **three things**:

1. **The original document** (in full — the retry is a fresh, stateless request; the API has no memory of attempt #1),
2. **The failed extraction** (so the model critiques an object rather than re-deriving from scratch),
3. **The specific validation errors** (so the model knows *what* is wrong and *where*).

```python
retry_prompt = f"""Here is the original invoice:
<document>{doc}</document>

Here is your previous extraction:
<extraction>{json.dumps(failed)}</extraction>

Validation found these problems:
<errors>
- line_items sum to 4830.00 but total is 4530.00. A line item may have been
  missed, or an amount misread.
</errors>

Re-extract, correcting these specific problems. If a field genuinely cannot be
determined from the document, set it to null rather than guessing."""
```

**Common distractors:**

- ❌ **Retry with the same prompt** (hoping temperature variance saves you) — burns cost, no directed signal, produces a *different* wrong answer as often as a right one.
- ❌ **"Be careful and double-check your math"** — vague pressure, no specific defect. Same failure mode as "be conservative" in 4.1.
- ❌ **Send only the errors without the document** — the model can't re-read the source to find the missed line item.
- ❌ **Escalate to human on first failure** — right eventually, wasteful immediately; format/semantic errors self-correct at high rates.

**Retry budget:** one, maybe two attempts. If a second retry *with explicit feedback* fails, the cause is almost certainly absent information or a genuinely ambiguous document → escalate. Unbounded retry loops are an anti-pattern: cost grows linearly, success probability does not.

---

## 4. Self-Correction Designed *Into the Schema*

Rather than only validating after the fact, shape the schema so the model **surfaces its own inconsistencies**:

- **`calculated_total` alongside `stated_total`** — the document prints a total; you *also* ask the model to sum the line items itself. Your validator now compares two model-produced fields, and a mismatch becomes a first-class signal.
- **`conflict_detected` boolean (+ `conflict_description`)** — for documents whose own data is inconsistent (invoice says $4,530, its line items sum to $4,830). Without this field the model silently picks one and you never learn there was a conflict.
- **Nullable / optional fields** (from 4.3) — a required field forces fabrication; an optional field lets "absent" be reported honestly, which is what lets you **distinguish class 3 from class 2** and skip a futile retry.

> **Design principle: make disagreement representable.** If the schema has no slot for "these two things don't match," the model resolves the conflict invisibly and the error signal is lost permanently.

---

## 5. Feedback Loops (the third layer — do not conflate with retry)

| | Retry | Feedback loop |
|---|---|---|
| **Scope** | Per-request | Cross-request |
| **Timing** | Online, seconds | Offline, days/weeks |
| **Target** | One bad output | The prompt, criteria, or examples producing bad outputs systematically |

**The canonical mechanism — `detected_pattern`:** a dismissed finding tells you nothing unless you know what triggered it. So label every structured finding with the code construct that produced it:

```json
{
  "file": "auth/session.py",
  "line": 142,
  "severity": "medium",
  "issue": "Possible unhandled null return",
  "detected_pattern": "optional_chaining_on_orm_result",
  "suggested_fix": "..."
}
```

Now dismissals aggregate: *"87% of dismissed findings carry `detected_pattern: optional_chaining_on_orm_result`."* That's a diagnosis. The fix is **upstream** — tighten explicit criteria (4.1) or add few-shot examples distinguishing the acceptable pattern from the genuine bug (4.2) — **not** more retries, and **not** "raise the confidence threshold."

**The chain:** `detected_pattern` makes false positives measurable → measurement identifies the bad category → the fix is an explicit-criteria or few-shot change. Without instrumentation you're guessing at prompt edits.

Also in scope: the 4.1 move of **temporarily disabling a high-false-positive category** while you fix it — the feedback loop gives you the evidence for *which* category to disable.

---

## 6. Decision Table (what actually gets tested)

| Symptom | Right move |
|---|---|
| JSON won't parse / fields missing | Switch to `tool_use` with a schema. Don't build a parser-retry loop. |
| Totals don't reconcile, values in wrong fields | Validate in code → retry **once** with document + failed output + specific errors |
| Field empty because the doc doesn't contain it | Make it nullable; **do not retry**; route to human or supply the missing source |
| Model fabricates plausible values for required fields | Schema fix (optional/nullable + "null rather than guess"), not retry |
| Doc's own numbers contradict each other | Schema fix: `conflict_detected` + both values; then human review |
| High dismissal rate on one finding category | Feedback loop: `detected_pattern` → aggregate → fix criteria/examples upstream |
| Failures cluster by source / field / format | Fix the prompt (few-shot + normalization rules), don't lean on retry |
| Retry #2 with explicit errors still fails | Stop. Escalate with a structured handoff (failed extraction + errors + attempts made) |

---

## 7. Anti-Patterns (recognize these as wrong answers)

1. **Retrying blindly** (same prompt, more attempts) — non-directed; cost scales, quality doesn't.
2. **Unbounded retry loops** — no budget, no escalation path.
3. **Retrying for absent information** — *the* defining "wrong but tempting" answer of this topic; converts an honest null into a hallucination.
4. **Asking the model to validate its own output in the same request** — it retains the reasoning that produced the error (4.6's self-review limitation). Validation belongs in deterministic code; an *independent second pass* is 4.6's answer, not a same-session "please check your work."
5. **Replacing a deterministic validator with an LLM validator** — never trade a guarantee for a probability when the guarantee is cheap (1.5 logic).
6. **Generic validator messages** ("validation failed") — throws away the corrective signal.
7. **Treating low confidence as a substitute for validation** — self-reported confidence is a routing input, not a correctness check.
8. **Fixing false positives by tightening a threshold instead of instrumenting** — you don't know which pattern misfires, so you suppress true positives too.
9. **Retry that omits the original document** — the model can't recover the data it needs.
10. **Reporting recovery rate as success rate** — a retry loop that always saves you is masking a systematic defect, and it has no headroom.

---

## Quick Self-Check Answers

- **Why `tool_use` kills a retry category** — the schema is enforced at generation, so malformed JSON and missing/mistyped required fields can't occur; only *semantic* errors survive.
- **Three things in a retry request** — original document (re-read the source), failed extraction (critique rather than re-derive), specific validation errors (directed, not random, correction).
- **When retry is guaranteed not to help** — when the information is genuinely absent from the input; retry then buys only fabrication and cost.
- **What `conflict_detected` buys** — it makes the document's *own* inconsistency representable, so the model reports the conflict instead of silently resolving it and destroying the signal.
- **Feedback loop vs retry** — retry is per-request and online, targeting one bad output; a feedback loop is cross-request and offline, targeting the prompt/criteria/examples that produce bad outputs systematically.

---

# Practice Questions

## Question 1 — Invoice extraction: two failure groups

**Scenario (Structured Data Extraction).** An invoice pipeline calls Claude with a forced `extract_invoice` tool (`vendor`, `invoice_date`, `line_items[]`, `total`). A Python validator checks that line items reconcile with the total. Over 10,000 invoices the validator flags 6%, splitting into:

- **Group A (~4%)** — line items visibly present in the PDF sum correctly, but the extraction omitted one line item sitting below a page break.
- **Group B (~2%)** — the invoice itself is internally inconsistent: the printed total genuinely doesn't equal the sum of its own printed line items.

Current handling is identical for both: re-send the original prompt with the same document up to 3 times, then mark `failed`.

**Which change most improves extraction quality across both groups?**

- **A.** Increase the retry limit from 3 to 6 and lower temperature to 0 on retries.
- **B.** For Group A, retry once with a request containing the original document, the failed extraction, and the specific reconciliation error; for Group B, add `calculated_total` and a `conflict_detected` boolean to the schema so the discrepancy is reported as data rather than treated as a failure.
- **C.** Add a system prompt instruction telling Claude to carefully double-check that line items sum to the total, and to re-read across page breaks.
- **D.** Make `total` optional/nullable so the model can omit it when it doesn't reconcile, and have the validator compute the total from line items in all cases.

### ✅ Correct answer: **B**

**Why B is right.** It's the only option that classifies the two failures differently and applies a different layer to each.
- **Group A is a semantic/extraction error** — the information *was* in the document; the model missed it. This is exactly where retry-with-error-feedback works, and the retry carries all three required pieces (document, failed extraction, specific error) so attention lands on the real defect.
- **Group B is not an extraction failure at all** — the extraction is *correct*; the source is inconsistent. No amount of retrying fixes a vendor's arithmetic. The fix is schema-level self-correction: `calculated_total` + `conflict_detected`, so the discrepancy becomes first-class data and routes to human review instead of being discarded as `failed`.

Second-order win: the 6% flag rate splits cleanly into "we got it wrong" (retryable) and "the document is wrong" (routable).

**Why A is wrong.** Blind retry with more attempts. Group A gets no directed signal — attempt #6 is no better informed than #1. Group B is guaranteed futile: the inconsistency is in the source, so it produces the same correct-but-flagged extraction six times. Temperature 0 makes it *worse* by removing even the accidental variance that might stumble onto the missing line item.

**Why C is wrong.** Vague prompt pressure — the "be conservative" failure mode from 4.1. It also asks the model to validate itself *in the same request*, where it retains the reasoning that produced the omission. For Group B it's actively harmful: instructed to make the numbers reconcile, the model will fabricate or adjust a line item to force agreement, converting a detectable source conflict into silent data corruption.

**Why D is wrong.** It destroys the evidence. Dropping the stated `total` means both numbers now come from the same source, so Group B becomes undetectable — and you'd ship a total the invoice never stated. Nullable fields are the remedy for **absent** information, not **contradictory** information; the remedy there is representing the conflict, not deleting one side of it.

---

## Question 2 — CI code review: 68% dismissal rate

**Scenario (Claude Code for CI).** Automated PR review via `claude -p --output-format json` produces findings (`file`, `line`, `severity`, `issue`, `suggested_fix`) posted as inline comments. After six weeks, **68% of findings are dismissed without a code change.** Engineers call the reviews "mostly noise" but can't articulate a rule — "it flags stuff that's fine in our codebase." You have full history of every finding and whether it was dismissed.

**What is the most effective first step to reduce the false positive rate?**

- **A.** Add a `confidence` field and suppress findings below 0.8 before posting, then monitor the dismissal rate.
- **B.** Add a `detected_pattern` field identifying which code construct triggered each finding, then aggregate dismissals by pattern to identify which categories are misfiring.
- **C.** Implement a validation-and-retry loop: re-submit each finding with surrounding code and ask Claude to confirm it's genuine, discarding retractions.
- **D.** Add a CLAUDE.md instruction that Claude should only report findings it is highly confident are genuine bugs and skip stylistic preferences.

### ✅ Correct answer: **B**

**Why B is right.** You have a *measurement* problem before you have a prompt problem. 68% dismissal says the system is noisy but not *where* the noise comes from, and the developers can't tell you either. Any prompt edit right now is a guess. `detected_pattern` is instrumentation: it labels every finding with its triggering construct, making the existing dismissal history analyzable. The distribution is typically wildly uneven — one or two patterns account for most dismissals while other categories are acted on. That's a diagnosis, and it points at the upstream fix: tighten explicit criteria (4.1), add few-shot examples distinguishing the acceptable pattern from the genuine bug (4.2), and optionally disable that one category temporarily to restore trust — without touching the categories that work. This is a **feedback loop**, the correct layer for a systematic quality problem; retry is the wrong layer entirely since nothing is wrong with any individual run.

**Why A is wrong.** Confidence-based filtering instead of explicit criteria — the exact 4.1 anti-pattern. Confidence is poorly calibrated and *orthogonal to the problem*: the model is confident these findings are real, because by general standards they are — they're only wrong **in your codebase**. The threshold suppresses true positives about as often as false ones, and you still learn nothing about which categories misfire.

**Why C is wrong.** Same-session self-review (4.6): the model retains the reasoning that generated the finding and rarely retracts. You'd pay a second inference per finding and — even where it works — fix only *this run's* output. The next PR reproduces identical false positives, because the prompt is unchanged. It treats a systematic prompt defect as a per-request validation problem.

**Why D is wrong.** Vague pressure again — "only report high-confidence genuine bugs" is precisely the general instruction the guide calls ineffective versus specific categorical criteria. It also misdiagnoses: these aren't nitpicks, they're findings legitimate *in general* but acceptable under your codebase's conventions. Telling Claude to be more confident doesn't teach it your conventions, and you've made a blind edit with no way to measure whether it helped.

---

## Question 3 — Research papers: the required `funding_source` field

**Scenario (Structured Data Extraction).** A forced `extract_paper_metadata` tool marks `title`, `authors`, `publication_year`, `funding_source` as **required**. The validator rejects empty required fields; ~15% rejection rate, all on `funding_source`; pipeline retries 3× with the original prompt then marks `failed`. An analyst reviews 50 rejects:

- **38** — the paper has no funding statement anywhere in the PDF supplied; funding is disclosed only in a supplementary file on the publisher's site.
- **12** — the funding statement exists but is buried in an unlabeled last-page footnote or phrased unusually ("This work was made possible by…").
- Separately: ~30 papers that **passed** contain a plausible-sounding grant agency that appears nowhere in the document.

**Which change best addresses this situation?**

- **A.** Increase retries to five and add "search the entire document including footnotes and acknowledgements" to the retry prompt.
- **B.** Make `funding_source` nullable and instruct the model to return `null` when no funding statement exists; add 2–3 few-shot examples showing funding in unlabeled footnotes and non-standard phrasings; retry only when validation fails for reasons other than a null `funding_source`.
- **C.** Keep `funding_source` required but add `funding_source_confidence`, routing extractions below 0.7 to human review.
- **D.** Remove `funding_source` from the schema entirely and populate it downstream by querying the publisher's API for supplementary files.

### ✅ Correct answer: **B**

**Why B is right.** There are **three** distinct problems and B addresses all three at the right layer:

1. **38/50 = absent information.** The one condition where retry is *guaranteed* not to help. Fix at the schema layer: **nullable**, so "absent" becomes a representable, honest outcome.
2. **12/50 = present but hard to find.** A genuine extraction failure — exactly what **few-shot examples** fix. Prose can't enumerate every phrasing, but examples teach the model to *generalize* to novel ones (4.2), and they fix the first pass rather than paying for a second.
3. **The ~30 fabricated agencies — the most serious problem**, because those records *passed* validation and are silently corrupting the dataset. The cause is structural: a **required** field forces the model to produce *something*, and a plausible agency name is what "something" looks like. Nullable + "return null rather than guess" removes the pressure that manufactures the fabrication (4.3 feeding into 4.4).

The final clause matters: **retry only for reasons other than a null `funding_source`** — you stop spending retries on the unfixable class and reserve them for what the model can actually correct.

**Why A is wrong.** It doubles down on retry for a population that is 76% unfixable-by-retry. Worse, sustained pressure to find a funding source in a document that has none is precisely what *drives* the fabrication — the ~30 hallucinated agencies would likely get worse.

**Why C is wrong.** Confidence as a substitute for fixing the structural cause. The field stays **required**, so the model still fabricates — you've added a filter downstream of the fabrication instead of removing its cause. Confidence is also badly calibrated for this case: a fabricated-but-plausible agency doesn't feel low-confidence to the model, so confident fabrications sail through while honest uncertainty gets routed to humans.

**Why D is wrong** (the tempting one). It does solve the 38 and does eliminate fabrication, but it **throws away the 12 papers where the data is right there in your document** — adding an external API call, latency, rate limits, and a new failure mode for a case a few-shot example solves for free. It's also a larger architectural change than the problem requires and doesn't compose: you *still* need the nullable design to represent "no funding disclosed anywhere." A publisher-API enrichment step is a reasonable **follow-on** for the records B correctly returns as `null` — the second move, not the first.

---

## Question 4 — Purchase orders: improving the retry mechanism (select TWO)

**Scenario (Structured Data Extraction).** Extraction uses `tool_choice: {"type": "tool", "name": "extract_po"}` with a strict JSON schema. A deterministic validator produces failures like:

- `ship_to.postal_code "0X1AA" is not a valid postal code for country "US"`
- `line_items[3].quantity is 0, but line_items[3].amount is 450.00`

The service retries by re-sending **only** the original document and system prompt, up to four times, then writes the record to a `failed` table with no other information.

**Which TWO changes would most improve this retry mechanism?**

- **A.** Include the failed extraction and the specific validator error strings in the retry request alongside the original document.
- **B.** Add a JSON-parsing fallback that catches malformed tool output and re-prompts the model to fix its JSON syntax before semantic validation runs.
- **C.** Reduce the retry budget to one or two attempts and, on exhaustion, escalate with a structured record containing the last extraction, the unresolved validator errors, and the attempts made — rather than a bare `failed` flag.
- **D.** Replace the deterministic validator with an LLM-based validator in the same request, asking Claude to review its own extraction for semantic inconsistencies before returning.

### ✅ Correct answers: **A and C**

**Why A is right.** The current mechanism re-sends only the document, making every retry a *blind re-derivation* rather than a correction — the model has no idea which field was wrong or why. A retry needs all three pieces: the **original document**, the **failed extraction**, and the **specific validator errors**. Those errors are already excellent corrective prompts: `"0X1AA" is not a valid postal code for country "US"` tells the model exactly where to look, and a model given that typically discovers it swapped `ship_to`/`bill_to` or misread the country. That signal is currently thrown away.

**Why C is right.** Two fixes in one. **Four attempts is over-budget**: with proper error feedback, fixable semantic errors resolve on attempt one or two; if attempt two *with explicit errors* fails, the cause is almost certainly unfixable-by-retry, so attempts three and four are pure cost. And a **bare `failed` flag destroys everything you learned** — the escalation record is a structured handoff (last extraction, unresolved errors, attempts made) so whoever picks it up doesn't start from zero. Same principle as structured escalation handoffs in 1.4.

**Why B is wrong (the trap).** With `tool_choice` forcing a tool and a strict JSON schema, **malformed JSON cannot occur** — tool use enforces the schema at generation, eliminating the entire syntax-error class. A JSON-repair fallback is dead code guarding a path that never executes. Recognizing this is the point of the 4.3 → 4.4 boundary: once you're on tool use, *every* remaining failure is semantic.

**Why D is wrong.** Two failures: (1) same-session self-review — the model retains the reasoning that produced the error (4.6); (2) it *replaces a deterministic validator with a probabilistic one*. `quantity == 0 && amount == 450.00` is an arithmetic contradiction code catches 100% of the time; an LLM catches it most of the time. Never trade a guarantee for a probability when the guarantee is cheap (1.5 logic).

---

## Question 5 — Medical records: a retry loop that's working too well

**Scenario (Structured Data Extraction).** ~8,000 documents/day from three hospital systems. Tool use with a strict schema; deterministic validator; failures retry once with document + failed extraction + specific errors.

- First-pass validation success: **91%**
- Of the 9% that fail, **retry succeeds 85%** → end-to-end ~98.6%
- **Hospital C alone accounts for 78% of all first-pass failures**, nearly all on `medication_dosage`, because its forms express dosage as "2 tabs BID" rather than the "500mg twice daily" format used by the other two systems.

The team is satisfied because retry recovers nearly all of these.

**What is the most effective recommendation?**

- **A.** Nothing structural — 98.6% end-to-end is strong and the retry loop is working as designed for a recoverable semantic error class.
- **B.** Add few-shot examples of Hospital C's dosage notation (and normalization rules mapping "2 tabs BID" to the target format) to the extraction prompt so these succeed on the first pass; keep the retry loop for residual failures.
- **C.** Route Hospital C documents to a dedicated path that retries twice instead of once.
- **D.** Make `medication_dosage` nullable so Hospital C's notation returns `null` instead of failing, and handle those records downstream.

### ✅ Correct answer: **B**

**Why B is right.** The retry loop working is exactly what makes this dangerous — a functioning recovery mechanism is masking a diagnosable systematic defect. The data already hands you the diagnosis: 78% of failures trace to one source and one field with a single identifiable cause (a notation convention the prompt never taught). That's a **prompt gap firing predictably on every Hospital C record**, so the right layer is the **feedback loop**, not retry. Few-shot examples are the correct instrument (4.2): prose struggles to specify a notation convention, but examples teach generalization to the *other* shorthand Hospital C uses ("PRN", "q6h", "1 tab TID"). Pairing them with **format normalization rules alongside the strict output schema** is the pattern the exam guide names directly. Payoff: ~7 points of first-pass failure eliminated, ~700 redundant inference calls/day removed, halved latency for those records, and a failure class removed that currently survives only because the recovery path happens to hold.

**Why A is wrong.** "98.6% end-to-end" is the metric that hides the problem — treating recovery as success stops distinguishing "we got it right" from "we got it wrong and paid twice." The loop has no headroom: change a form, double volume, or shift the 15% residual and the failure surfaces at full size. You're paying ~2× inference on 9% of an 8,000/day workload indefinitely to avoid a one-time prompt edit.

**Why C is wrong.** It scales the band-aid — retry budget is the wrong dial. It also misclassifies the failure: these documents fail because the model was never taught the notation, and a second identical retry adds no new information beyond the first. You'd raise cost on the worst source while leaving its first-pass rate untouched.

**Why D is wrong.** Nullable is the remedy for **absent** information; this information is present and perfectly extractable, just in an untaught notation. You'd deliberately discard data you could capture and rebuild it downstream at higher cost. In a medical context `medication_dosage: null` is materially dangerous — systematically emptying a safety-critical field for an entire hospital system. Misclassifying "unfamiliar format" as "missing data" is the exact error this topic trains you to avoid.

---

## Cross-Topic Connections

- **4.3 → 4.4** — tool use eliminates syntax errors; everything left is semantic. Nullable/optional field design from 4.3 is *also* the mechanism that prevents fabrication and lets you separate retryable from futile failures.
- **4.1 → 4.4** — `detected_pattern` produces the evidence that tells you *which* criteria to make explicit and which category to temporarily disable.
- **4.2 → 4.4** — few-shot examples are the standard upstream fix a feedback loop points you toward, and they beat retry because they fix the first pass.
- **4.6 → 4.4** — same-session self-review is unreliable; validation belongs in deterministic code, and a genuine second opinion requires an independent instance.
- **1.4 / 1.5 → 4.4** — escalate with a structured handoff, not a status code; and prefer deterministic enforcement over probabilistic instruction whenever the guarantee is cheap.
- **2.2 → 4.4** — errors are prompts. Your validator's messages are read by a model, so write them as corrective guidance, not error codes.
