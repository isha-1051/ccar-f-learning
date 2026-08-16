# Topic 4.1 — Explicit Criteria & False Positives

**Domain 4: Prompt Engineering & Structured Output (20%) · Topic 1 of 6 · Overall Topic 19 of 30**

---

## 1. The core idea

A false-positive problem in an LLM judgment task is almost never a model-quality problem. It is an
**underspecified-criteria problem**.

**Why over-flagging is the default failure mode:** when a category boundary is vague, the model has
nothing task-specific to decide with, so it falls back on general priors about the word you used.
Ask it to flag "sensitive information" and it applies the internet's broadest, most cautious notion
of sensitive — far wider than yours. Flagging also feels like the helpful, cautious action, so the
boundary drifts outward. **Vagueness in a detection prompt systematically resolves toward
over-inclusion.**

The fix is not "make the model more careful." It is: **move the decision boundary out of the model's
head and into the prompt, in writing.**

---

## 2. The five mechanisms that work

### (a) Define the category positively AND negatively
Most prompts only say what counts. The high-leverage half is what does *not* count — an explicit
exclusion list directly cancels the model's over-broad prior.

> ❌ "Flag any ticket containing personally identifiable information."
>
> ✅ "Flag a ticket only if it contains one of: full name + street address together, government ID
> number, full payment card number, or personal email address.
> **The following do NOT count and must not be flagged:** first name alone, a company email at the
> customer's employer domain, an order number, a partial card number (last 4), a city or country
> without a street address, or a support agent's own name."

Shape: an **enumerated closed list** of what counts, plus a **"does not count" list** built from the
mistakes actually observed.

### (b) Replace adjectives with decision rules
Adjectives are where false positives live: *sensitive, urgent, critical, toxic, relevant,
significant*. Convert each into an operational test.

> ❌ "Escalate urgent tickets."
> ✅ "Escalate only if the ticket states an active service outage, a failed payment blocking a
> transaction, or a stated legal/regulatory threat. Customer frustration, tone, capital letters, or
> the word 'urgent' in the customer's own text are **not** sufficient."

That final clause matters — it explicitly breaks the model's tendency to mirror the input's tone.

### (c) Require evidence before the verdict
Force the model to quote the triggering text, and require it to appear **before** the boolean:

```json
{ "evidence_quote": "...", "criterion_matched": "...", "is_pii": true }
```

Two effects: the model must locate something concrete (can't flag on a vibe), and you get an
auditable trail showing *which* criterion is too loose. Ordering matters because the model conditions
on its own prior tokens — reasoning after the verdict is post-hoc rationalisation.

### (d) Use boundary examples, not central examples
Three examples of blatant PII teach nothing; the model already gets those right. Spend examples on
**near-misses** — cases just inside and just outside the line, especially ones the system got wrong.

> "My order #48211 hasn't shipped to my apartment in Denver" → `false` (order number and city are not PII)
> "Ship to 412 Oak St, Denver CO, I'm Maria Chen" → `true` (full name + street address)

This is calibration, not demonstration.

### (e) Give an explicit default for ambiguity
Never leave the ambiguous case undefined — undefined means the model picks, and it picks "flag."

> "If a ticket does not clearly satisfy one of the four listed criteria, return `false`. Do not flag
> on the possibility that information *might* be identifying."

Or route ambiguity to a third value / human review when misses are the expensive error (→ 5.2, 5.5).

---

## 3. The tradeoff: which error is expensive?

There is no "reduce errors" — only **moving the boundary**. Every movement trades one error type for
the other. The exam's judgment question is which direction the deployment demands.

| Context | Expensive error | Correct posture |
|---|---|---|
| Security scan that **blocks merges** | False positive — devs get blocked, then disable the check | Narrow criteria, high bar, require concrete exploit path |
| Security scan posting **advisory comments** | False negative — a missed vuln ships | Broader criteria acceptable; noise is cheap |
| PII redaction before third-party export | False negative — a leak is unrecoverable | Broad criteria, over-redact, flip ambiguity default to flag |
| Auto-refunding / auto-freezing accounts | False positive — direct customer harm / money loss | Strict criteria + human approval gate |

**Key insight:** a false-positive-heavy system doesn't just produce noise — it produces
**abandonment**. A check that cries wolf gets ignored or disabled, at which point effective recall is
zero. "The tool became untrusted and was disabled" is a root cause the exam tests.

---

## 4. Anti-patterns (each is a common distractor)

| Anti-pattern | Why it fails |
|---|---|
| "Be more careful / more conservative / avoid false positives" | Pressure without information. The model doesn't know which cases you consider wrong, so it shifts the whole boundary inward by an unknown amount. **Vague pressure moves the threshold; criteria move the boundary.** |
| Lowering temperature | Affects *consistency*, not *correctness*. A miscalibrated boundary at temp 0 is reliably wrong. |
| Switching to a larger model | A more capable model reads your ambiguous instruction more competently — it still doesn't know *your* definition. Capability doesn't supply missing specification. |
| Adding a post-filter as the primary fix | If you can write the filter rule you could have written it as a criterion, where it also improves the model's reasoning. Post-filters also can't recover false *negatives*, and they miss mixed cases where the bad signal is stacked with a second weak one. Legitimate only as a safety net on top of fixed criteria. |
| Piling on more positive examples | Reinforces the class already detected. Over-flagging is a boundary problem — you need edge cases, not volume. |
| Rewriting criteria from imagination | The correct loop is: collect actual false positives → cluster → encode the clusters as exclusions. Rewriting from scratch discards your only real signal (cf. 3.5). |

---

## 5. The workflow

1. **Measure** — labeled sample; know actual FP/FN rates. "It flags too much" is not a starting
   point; "it flags 38%, ground truth 6%, and 90% of FPs are order numbers" is.
2. **Cluster the failures** — false positives fall into a handful of recurring patterns.
3. **Encode each cluster as an explicit exclusion.**
4. **Add boundary examples** for clusters prose can't pin down.
5. **Constrain the output structurally** — evidence quote + matched criterion, before the verdict.
6. **Re-measure** and check you haven't overshot into false negatives.

Theme for all of Domain 4: **the prompt is a specification.** If a human reviewer would need to ask a
clarifying question to apply your criteria consistently, the model has the same ambiguity — and it
resolves it silently rather than asking.

---

## 6. Worked examples — FP / FN with numbers

### Confusion matrix
Task: flag support tickets containing PII. 1,000 tickets; **60** truly contain PII.

| | Model says **flag** | Model says **don't flag** |
|---|---|---|
| **Actually has PII** | True Positive | **False Negative** — the miss (leak ships) |
| **No PII** | **False Positive** — the noise | True Negative |

**Run 1 — vague prompt** ("flag tickets containing sensitive information"): 380 flagged, 57 correct.
- TP 57, FP 323, FN 3, TN 617 → **Precision 15%**, **Recall 95%**
- Classic shape: near-perfect recall, terrible precision. Looks safe, operationally dead — a reviewer
  facing 380 tickets at a 15% hit rate stops reading carefully, so *human* review starts missing too.
  Effective recall is worse than the number implies.

**False-positive clusters found in the audit:**

| Cluster | Count | Example |
|---|---|---|
| Order / invoice numbers | 140 | "order #48211 never arrived" |
| First name only | 95 | "Hi, this is Maria, my app crashed" |
| City / country, no street | 50 | "I'm in Denver, is there an outage?" |
| Last-4 of card | 38 | "the card ending 4429 was declined" |

**Run 2 — explicit criteria** (enumerated inclusion + "does not count" list built from those four
clusters + required evidence quote): 58 flagged, 54 correct.
- TP 54, FP 4, FN 6, TN 936 → **Precision 93%**, **Recall 90%**
- Precision 15% → 93%; recall 95% → 90%. **The 5-point recall loss is a real cost**, not a rounding
  error — 6 PII tickets now pass unflagged. Name the trade honestly; criteria are not free.

### Same task, opposite direction
Change the deployment to **redaction before export to a third-party vendor** — a miss is an
unrecoverable leak, a false positive costs one redacted word. Run 2's config is now *wrong*:
- Move first-name-alone and last-4 back into the flag set.
- Flip the ambiguity default from "return false" to "return true."
- Result: recall → 99%, precision → 55%. **45% noise accepted on purpose**, because the consumer is
  an automated redactor, not a human.

**The point:** criteria don't make the model "better" — they give you a **knob you can aim**. Before,
the boundary sat wherever the model's prior put it and you couldn't move it deliberately.

### CI vulnerability scanning (abandonment)
- "Report any security vulnerabilities in this diff." → 200 PRs, 12 real vulns, 90 findings, 11
  correct. **Precision 12%.**
- Consequence chain: 8 findings per PR, 7 noise → devs stop reading → they add a bypass →
  **effective recall becomes 0%.**
- Fix: *"Report a finding only if you can name (a) the untrusted input source, (b) the line where it
  reaches a sink, and (c) the sink type. If you cannot identify all three, do not report."* → 14
  findings, 10 correct. **Precision 71%, recall 83%.**
- Lost 1 real vuln, gained a check developers keep enabled. **A precise-but-imperfect detector that
  stays on beats a thorough one that gets disabled.**

---

## 7. Temperature — what it is, and why it isn't the fix

**Mechanically:** at each step the model produces a logit per candidate token; softmax converts these
to probabilities. Temperature divides the logits before that conversion.

- **→ 0**: distribution sharpened to a spike on the top token; model always takes its highest-probability
  continuation. Near-deterministic (not *guaranteed* identical — floating-point / batching cause rare drift).
- **= 1** (Claude's default): the distribution as produced.
- **higher**: distribution flattens, lower-ranked tokens get a realistic chance. More varied, more error-prone.

**Claude API specifics:** `temperature` ranges **0.0–1.0**, default **1.0**. Anthropic recommends
tuning *either* `temperature` or `top_p`, not both. With **extended thinking enabled, `temperature`
must be 1** — it cannot be lowered.

**What it changes:** *variance* — how much two runs on the same input differ.
**What it does not change:** *where the decision boundary sits.* It reweights sampling from the
model's beliefs; it does not alter the beliefs.

So if the model believes "order #48211" is 80% likely PII, at temp 1 it flags ~80% of the time; at
temp 0 it flags it **100% of the time**. Lowering temperature on an over-flagging classifier makes it
*more reliably wrong*. That is why "lower the temperature" is a distractor on false-positive items.

**When temperature is the right lever:**
- **Low (0–0.2)**: classification, extraction, structured output, tool arguments — anything parsed,
  or where two runs disagreeing is itself a bug.
- **Higher (0.7–1.0)**: brainstorming, drafting, and deliberately **generating diverse candidates**
  (the mechanism behind multi-pass review in 4.6).

> **Rule of thumb:** temperature is a *consistency* dial, not a *correctness* dial. If the complaint
> is "wrong," fix the prompt. If the complaint is "different every time," fix the temperature.

---

## 8. Clarifications from follow-up questions

### 8a. How explicit criteria interact with structured output schemas (previews 4.3)

Criteria live in the prompt; the schema is what makes the model *use* them.

1. **Schema fields are enforcement.** "Cite the triggering text" as a criterion is a *request*; a
   required `evidence_quote` field is a *constraint* — no valid response without it. Same rule, two
   enforcement strengths (same programmatic-vs-prompt axis as 1.4 and 2.3, applied to output).
2. **Field ordering encodes reasoning order.** The model generates left to right, conditioning on what
   it already wrote. `{evidence, criterion_matched, verdict}` → the verdict is computed *from* the
   citation. `{verdict, evidence}` → evidence is invented to justify a committed verdict. Same fields,
   opposite reliability.
3. **Enums collapse the ambiguity criteria were fighting.**
   `"criterion_matched": enum[...]` forces the model to name *which* rule fired. It can't flag on a
   general feeling of sensitivity — there's no enum value for that — and a mismatched criterion is
   visible in review. Free-text reason fields don't give you this.

**Framing:** criteria define *what correct means*; the schema makes the criteria *unavoidable and
auditable*. Distractor to watch for: relying on prose alone where a schema field would have made the
same rule structurally binding.

### 8b. When a third `"uncertain"` value beats forcing a binary

Forcing a binary on a genuinely ambiguous case doesn't remove uncertainty — it **hides** it. The model
guesses and reports the guess with the same confidence as a clear-cut case.

**Add `"uncertain"` only when all three hold:**
1. A genuinely ambiguous middle exists.
2. **Both** error types are expensive — otherwise just set the default in the cheaper direction.
3. A real router exists: human queue, second pass, fallback tool. `"uncertain"` with nowhere to send
   it is a `false` with extra steps.

**Keep the binary when:** one error is clearly cheaper, volume makes human review impossible, or the
consumer is automated and must act either way.

**Failure mode:** `"uncertain"` becomes an escape hatch — the model routes 60% of cases there because
it's the low-effort answer. Prevention: define it as narrowly as the other two values (*"return
`uncertain` only when the text matches a listed criterion but a required element is missing"*), not as
"when you're not sure." An undefined third value is just new vagueness.

Bridges to **5.2 (escalation)** and **5.5 (confidence calibration)** — the third value is the
prompt-layer half of an escalation design.

### 8c. How this differs from the "evaluate then decide" ordering trick

| | Explicit criteria | Evaluate-then-decide |
|---|---|---|
| Defect fixed | Model doesn't know **where the line is** | Model knows the line but **commits before checking** |
| Layer | *Content* of the prompt — the definition | *Sequencing* of the output — order of generation |
| Failure without it | Consistent, systematic over-flagging of a whole category | Scattered, inconsistent errors; reasoning that rationalises the answer |
| Concretely | Adding "order numbers do not count as PII" | Emitting `evidence` and `criterion` before `verdict` |

**The diagnostic — read the model's reasoning on a false positive:**
- Reasoning **coherent but applies a definition you disagree with** ("order numbers are account-linked
  identifiers, so this is PII") → **criteria problem.** Ordering won't help; it reasoned correctly from
  the wrong definition.
- Reasoning **incoherent or contradicting the verdict** ("no identifying details are present… flagging
  as PII") → **ordering problem.** The verdict was fixed first and text was backfilled.

**They compose; neither substitutes for the other.** Criteria without ordering → good rules, skipped.
Ordering without criteria → careful reasoning toward the wrong boundary, producing *well-argued* false
positives that are harder to catch in review than sloppy ones. The schema (8a) is what makes the
ordering binding rather than requested.

---

## 9. Practice questions

### Q1 — Fintech auto-freeze (fraud flagging)

Claude reads transaction metadata and flags fraud; **flagged transactions are automatically frozen**
and the customer is emailed. Prompt: *"Review this transaction and flag it if it appears fraudulent or
suspicious."* After two weeks: flags **11%** vs a **0.4%** baseline fraud rate; complaints up sharply;
the model's stated reasoning on incorrect flags is **internally consistent and well-argued** (it
explains that a first-time merchant, an unusual hour, or a new city is "suspicious"); ~70% of incorrect
flags fall into those three patterns.

**Which change most effectively reduces false positives?**

- **A.** Set `temperature` to 0 so the model stops producing variable judgments about what counts as suspicious.
- **B.** Restructure the schema so `reasoning` is emitted before `is_fraud`, ensuring the verdict follows from analysis.
- **C.** Replace "fraudulent or suspicious" with an enumerated list of qualifying fraud indicators, plus an explicit list stating that first-time merchants, off-hours purchases, and new-city transactions do **not** by themselves qualify.
- **D.** Keep the prompt and add a downstream rule-based filter discarding any flag whose only cited signals are merchant novelty, purchase hour, or location change.

**Correct answer: C** ✅ *(answered correctly)*

**Why C is right:** the scenario states the model's reasoning is **internally consistent and
well-argued** — that rules out an entire class of fixes. The model isn't confused or rationalising; it
reasons *competently* from a definition of "suspicious" broader than the fraud team's. That is the
signature of a **criteria defect**. And 70% of errors cluster into three named patterns — the failure
clusters are already identified for you, so the exclusions write themselves. C fixes the problem at the
layer where the boundary lives, so the model stops treating those signals as sufficient *and* stops
treating them as corroborating evidence in mixed cases.

**Why A is wrong:** temperature controls *variance*, not boundary placement. The evidence points the
other way anyway — the model is repeatedly flagging the *same three patterns*, i.e. already consistent.
Temp 0 makes it flag them 100% of the time instead of most of the time: strictly worse.

**Why B is wrong:** the trap for someone who learned the mechanism but not the diagnostic.
Evaluate-then-decide fixes a model that **commits then backfills** — its tell is reasoning that is
incoherent or contradicts the verdict. Here the reasoning is coherent and well-argued, so the model
*is already* reasoning before deciding. Reordering changes nothing.

**Why D is wrong:** (i) if you can write the filter rule you could have written it as a criterion,
where it also improves the model's reasoning; (ii) the filter only catches flags whose *only* signal is
one of the three — a transaction flagged on "new city **plus** a slightly high amount" sails past,
because the model still treats new-city as evidence and now stacks a second weak signal. Suppresses the
pure cases, leaves the mixed ones. Valid as a safety net, not as the primary fix.

**Deployment note:** auto-freeze with no human in the loop is a false-positive-expensive design →
tighten aggressively and default ambiguity to "don't flag." If it were "flag for analyst review," the
correct posture would be looser. Same task, opposite tuning, driven by what sits downstream.

---

### Q2 — Healthcare PHI export

Claude reviews clinician notes before nightly **automated export** to an external research partner;
anything marked clean goes out the door. The prompt already has a tight inclusion list (patient name,
MRN, DOB, address, phone) and an exclusion list (clinician names, department names, generic ages like
"elderly", relative dates like "three weeks ago"). On 2,000 notes (110 contain PHI): **96 flags, 94 TP,
2 FP, 16 FN.** Compliance says this is unacceptable. Budget exists for a nightly human review queue
handling ~300 notes.

**Which change should you make?**

- **A.** Keep criteria; add a `confidence` score and export only notes the model is highly confident are clean.
- **B.** Broaden the inclusion criteria to cover borderline identifiers, move ambiguous cases from the exclusion list into the flag set, and change the ambiguity default from "do not flag" to "flag."
- **C.** Add an `"uncertain"` value for partial matches, route only those to the human queue, and continue auto-exporting everything marked clean.
- **D.** Add few-shot examples of the 16 missed notes so the model learns the patterns it failed to catch.

**Correct answer: B** ✅ *(answered correctly)*

**Why B is right:** precision is **98%** (94/96), recall **85%** (94/110) — a system tuned for a
*human-reviewer* deployment dropped into one where **nothing catches what it misses**. Every FN is an
unrecoverable disclosure. The criteria aren't broken in the usual sense; they're **aimed the wrong
way**. The exclusion list is the culprit: relative dates and generic ages are safe exclusions under
human review but can re-identify a patient when combined with other quasi-identifiers. B widens
inclusion, pulls borderline items out of the exclusion list, and — the piece people forget — **flips
the ambiguity default from `false` to `true`**, which is where a large share of the 16 misses live. The
300-note queue capacity is the feasibility check: flags rise into the low hundreds, precision falls to
perhaps 40–50%, and **that noise is accepted on purpose**.

**Why C is wrong:** the strong distractor. The three conditions for a third value require that **both**
error types be expensive — here FNs are catastrophically worse, so the correct move is to *set the
default in that direction*, not add a third value. Worse, C explicitly keeps auto-exporting the clean
path and leaves the criteria untouched. The 16 misses weren't partial matches — the model marked them
**confidently clean** under criteria that told it to.

**Why A is wrong:** self-reported confidence is not calibrated probability — it's another generated
token from the same model that just confidently cleared 16 PHI notes. It also inverts the task into
proving a *negative* ("I'm confident nothing identifying is here"), a much weaker signal than citing a
positive match, and leaves the actual defect untouched.

**Why D is wrong:** the misses stem from **stated exclusion rules the model followed correctly**.
Examples that contradict the written rules give the model two conflicting instructions → erratic
behaviour, not a moved boundary. Fix the rule first, then use boundary examples for what prose can't
pin down. It also only teaches the 16 sampled patterns.

**Transferable lesson:** the same criteria set can be correct or dangerous depending on what sits
downstream. Before tuning, ask *what catches the error I don't catch?* Here the answer was "nothing" —
which dictates the direction before you touch a word of the prompt.

---

### Q3 — Accessibility PR reviewer (self-contradicting explanations)

A PR-review agent has carefully enumerated criteria (missing `alt` on informative images, inputs
without labels, interactive elements with no accessible name, contrast below 4.5:1) and an explicit
"does not count" list (decorative images with `alt=""`, `aria-hidden="true"` elements, disabled-control
contrast). Output shape: `{ "violation": true, "rule": "missing_label", "explanation": "..." }`.
An audit of 50 incorrect findings shows explanations that **describe the correct analysis and then
contradict it** — *"This input is wrapped in a `<label>` element, so it has an associated label.
Flagging as missing_label."* Different runs on the same PR produce different findings.

**Most effective fix?**

- **A.** Add the 50 audited cases as few-shot examples showing the correct verdict for each.
- **B.** Reorder the schema to `{ "rule", "explanation", "violation" }` so analysis is generated before the boolean verdict.
- **C.** Tighten the "does not count" list further, stating that `<label>`-wrapped inputs and `alt=""` images must never be reported.
- **D.** Set `temperature` to 0 to eliminate run-to-run variation, then re-audit.

**Correct answer: B** ✅ *(answered correctly)*

**Why B is right:** the model **recites the rule accurately, including the exclusion, then contradicts
it**. It doesn't misunderstand the rule — it committed to `"violation": true` first, because the
boolean is the **first field generated**, and everything after is conditioned on a verdict already on
the page. The explanation isn't the reasoning; it's commentary on a decision already made. The
run-to-run variation *confirms* this rather than pointing at temperature: an ungrounded verdict token is
sampled from a genuinely uncertain distribution because the model has nothing but the raw diff to
condition on at that moment. Force analysis out first and the boolean becomes near-deterministic
**because it's conditioned on the model's own written reasoning**.

**Refinement:** `{rule, explanation, violation}` is a large improvement but not ideal — `rule` is still
a commitment made before analysis. The strongest ordering is `{explanation/evidence, rule, violation}`,
so the criterion is *selected by* the analysis rather than anchoring it.

**Why A is wrong:** examples teach *where the boundary is*, and the model already knows — it says so,
correctly, in every failing explanation. You'd re-teach knowledge it demonstrably has while leaving the
structural cause untouched.

**Why C is wrong:** the exclusion list already covers these cases and the model *cites it by name*
before violating it. When the model can quote your rule back to you and still get it wrong, restating
the rule is the one thing guaranteed not to help.

**Why D is wrong:** most defensible wrong answer, since variation *is* what temperature controls. But
it addresses the symptom and locks in the disease: the verdict token is still generated first with
nothing to condition on, so temp 0 makes the model pick its single most likely ungrounded guess **every
time**. You trade inconsistent self-contradiction for consistent self-contradiction and lose the
variation signal. Reorder first; low temperature is a reasonable *addition* afterwards.

**Contrast with Q1 (deliberate):** reasoning coherent and well-argued toward a wrong boundary →
criteria defect. Reasoning coherent and correct but contradicting the verdict → ordering defect.
**Read the reasoning on the failures; it tells you which layer is broken.**

---

### Q4 — Secret scanner abandonment (multiple response, select TWO)

A Claude-based scanner posts a **blocking** review flagging secrets in PRs; the PR can't merge until a
human dismisses each finding. Prompt: *"Identify any secrets, credentials, or sensitive values in this
diff."* Six weeks in: **9 findings per PR**, roughly **1 in 12** real. Repeat findings: test-fixture
example values, documentation placeholders, env-var *names* with no value (`STRIPE_SECRET_KEY`), and
lockfile hashes. Three teams added a bypass label. **A real leaked production token merged last month on
a PR from one of the bypassing teams.**

**Which TWO changes together most effectively address the root cause?**

- **A.** Rewrite the criteria to enumerate what qualifies (a value with the structural shape of a live credential, in a non-test path, not a bare variable name) and explicitly exclude test fixtures, documentation placeholders, env-var names without values, and lockfile hashes.
- **B.** Instruct the model to err on the side of reporting anything that could conceivably be a credential, since a leaked token is far costlier than a dismissed comment.
- **C.** Change the integration from a blocking review to a non-blocking advisory comment.
- **D.** Require the finding schema to include the matched literal value and the criterion it satisfies, and drop any finding where the model cannot supply a concrete credential-shaped value.

**Correct answers: A and D** — *(answered A only; half credit)*

**Root cause:** **precision collapse caused abandonment, and abandonment set effective recall to
zero.** Nine findings per PR at a 1-in-12 hit rate means eight pieces of noise per real one.
Predictably, teams built a bypass — and a real token then merged *through* one. The scanner didn't miss
it for lack of capability; **it was no longer running.** The fix must make the check trustworthy enough
that teams keep it on.

**Why A is right:** classic criteria repair driven by the observed clusters (test fixtures, doc
placeholders, bare env-var names, lockfile hashes). Enumerate what qualifies — credential-shaped value,
non-test path, not a bare name — and cancel each cluster explicitly. Alone this should take 9
findings/PR down to roughly 1.

**Why D is right:** A defines the boundary in prose; **D makes it structurally unavoidable and
auditable**. Requiring the matched **literal value** plus the **criterion satisfied**, and dropping
findings that can't produce one: (i) kills the env-var-name class *by construction* — `STRIPE_SECRET_KEY`
has no value to quote, so the finding can't be formed, no rule-following required; (ii) is the
evidence-before-verdict pattern, forcing the model to locate something concrete; (iii) gives the
iteration signal — a surviving FP names exactly which rule is too loose. **A and D are the two halves:
criteria say what correct means; the schema makes the criteria binding.**

**Why B is wrong:** vague pressure aimed in the direction that caused the outage. The per-instance cost
logic is seductive but wrong at the system level: more noise → more bypass → **zero** recall. It
optimises per-finding cost while destroying whether any finding gets produced at all.

**Why C is wrong:** the most tempting distractor, since it relieves the pain teams complained about.
But (i) it treats the symptom — eight noise comments get ignored whether they block or not, trading
*explicit* bypass labels for *silent* scroll-past, which is worse because you lose the visible signal
that the check is failing; and (ii) secrets are exactly the category where blocking is justified, since
a committed production credential is unrecoverable. **Make the gate trustworthy; don't remove the
gate.** After A and D land at ~1 finding/PR with 70%+ precision, blocking is reasonable and the bypass
labels come out on their own.

---

## 10. Exam-day checklist for this topic

- Over-flagging + **coherent, well-argued** reasoning → **criteria defect** (enumerate + exclude).
- Over-flagging + reasoning that **contradicts the verdict** → **ordering defect** (evidence before verdict).
- Complaint is "different every time" → temperature. Complaint is "wrong" → prompt. Never the reverse.
- Before tuning direction, ask **what catches the error I don't catch?** Nothing downstream → widen and
  flip the ambiguity default. A human queue downstream → tighten and let the queue absorb the rest.
- Precision collapse causing bypass/disable → **raise precision** (criteria + schema enforcement).
  Never *lower the stakes*, never *flag harder*.
- Criteria + schema compose. Prose alone gets followed inconsistently; a schema constraint gets
  followed always.
- A third `"uncertain"` value needs **both** error types expensive **and** a real router — otherwise
  set the default and keep the binary.

**Practice score: 3.5 / 4.**
