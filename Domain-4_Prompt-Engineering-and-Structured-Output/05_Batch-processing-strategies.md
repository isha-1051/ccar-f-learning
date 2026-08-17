# Topic 4.5 — Design efficient batch processing strategies

**Domain:** 4 — Prompt Engineering & Structured Output (20% of exam)
**Task Statement:** 4.5
**Topic number:** 5 of 6 within Domain 4 · 23 of 30 overall

---

## The one-sentence version

Batch is a **different delivery contract, not a different model**: you trade *when* you get the answer for **50% off**, and that trade is only legal when nothing downstream is blocked while waiting.

---

## 1. What the Message Batches API is

You submit an **array of complete, independent Messages API requests** in one call. Anthropic processes them asynchronously; you retrieve results later.

| Property | Value | Why it matters |
|---|---|---|
| Cost | **50% off** input *and* output tokens | The headline benefit — and the only benefit |
| Processing window | Up to **24 hours** | 24h is the *ceiling*, and the number you plan with |
| Latency SLA | **None guaranteed** | You may not assume "usually 1 hour" |
| Correlation | `custom_id` per request | Results return in **arbitrary order** |
| Result statuses | `succeeded` / `errored` / `canceled` / `expired` | Drives failure-handling strategy |

Everything else is a normal Messages API request: system prompts, vision, prompt caching, structured output via tool use. **A batch is a scheduling wrapper, not a reduced-capability endpoint.**

Mental model: **batch is the async job queue of the Claude API.** Same worker, same quality, deferred delivery, discounted price.

*(Supporting API limits — 100,000 requests / 256 MB per batch, 29-day result retention — are **not** in the exam guide. Recognize, don't memorize.)*

---

## 2. The decision boundary: blocking vs. non-blocking

The question is never "is this a lot of documents?" It is:

> **Is a human or a pipeline stuck waiting on this result?**

| Blocking → **synchronous API** | Non-blocking → **batch API** |
|---|---|
| Pre-merge PR checks (the merge waits) | Nightly test generation |
| Interactive chat / support agent replies | Overnight report generation |
| Real-time moderation on submit | Weekly compliance audits |
| A CI gate that fails the build | Monthly analytics over a corpus |
| Anything a user watches a spinner for | Backfilling historical classifications |

**Classic distractor:** "We process 50,000 documents daily — batch it to save cost." **Volume is not the criterion.** If those documents arrive one at a time from live users needing an answer now, batch is wrong at any volume. Conversely, 40 documents in an overnight job is a fine batch use case.

**Inverse distractor:** "Our pre-merge check is expensive, use batch to halve the cost." A pre-merge check is blocking by definition. **Cost savings never justify converting a blocking workflow to an unbounded-latency one.**

**Tiebreaker:** *latency-tolerant* → batch. *latency-sensitive* → sync. The scenario always hands you the tolerance ("reviewed the next morning", "the developer needs feedback before merging").

⚠️ **Automated ≠ non-blocking.** A pipeline with no human typing in real time can still be blocking if a merge, deploy, or downstream job is stalled behind it.

---

## 3. Hard constraint: no multi-turn tool calling within a batch request

**A batch request is one round trip.** It cannot execute a tool mid-request, feed the result back, and continue. There is no harness inside the batch (contrast topic 1.1).

| Pattern | Batch-compatible? |
|---|---|
| Tool use for **structured output** (single `tool_use` block as the final answer — topic 4.3) | ✅ Yes |
| `tool_choice: {"type": "tool", ...}` forcing a schema | ✅ Yes |
| An **agentic loop** — model calls tool, harness executes, result returns, model continues | ❌ No |
| Live retrieval *during* the request | ❌ No |
| Your code acting on the result *after* the response | ✅ Irrelevant — outside the request |
| Prompt caching a shared prefix across batch requests | ✅ Yes (stacks with the 50% discount) |

### The test procedure

> Does the model need a tool **executed** and its result **fed back** *before it can finish this response*?
> **No** → batchable. **Yes** → not batchable as written.

### Two legal conversions

**Design 1 — resolve first, then batch** *(preferred)*
```
your code performs the lookup/query
   → embed results into N prompts
   → submit one batch of N requests
```
One hop, up to 24h. Works whenever your code can determine what the tool call would have been.

**Design 2 — chain batches** *(only at very relaxed SLAs)*
```
batch 1 emits tool calls → your code executes → batch 2 continues with results embedded
```
**Two** stacked 24-hour ceilings. Almost always worse than Design 1.

**Exam framing:** "use batch for this agentic workflow" is wrong. "Resolve the retrieval step first, then batch the analysis" is the right shape.

---

## 4. `custom_id` — correlation and retry scoping

Each request carries a `custom_id` you assign; each result returns carrying the same one.

**(a) Results arrive in arbitrary order.** Never pair results to inputs by array position. Build a map keyed by `custom_id`. Position-based matching is a **silent data-corruption bug** — every record gets some other record's output, all results report `succeeded`, counts match, and nothing errors.

**(b) `custom_id` is the unit of retry.** When 12 of 10,000 fail, the failing IDs identify exactly which source objects to fix and resubmit. Make the ID map to your domain object (`doc-4471`, `invoice-2024-Q3-0088`), not `req-1`.

---

## 5. Failure handling: diagnose → modify → resubmit the subset

| Status | Meaning | Correct action |
|---|---|---|
| `succeeded` | Fine | Store it. Do **not** resubmit. |
| `errored` — validation / invalid request | Request is malformed or oversized | **Fix the request, then resubmit.** Blind retry loops forever. |
| `errored` — transient / server side | Infrastructure hiccup | Safe to resubmit unmodified |
| `expired` | Didn't finish in 24h | Resubmit |
| `canceled` | You canceled the batch | Resubmit if still needed |

The exam guide's own example: **documents that exceeded the context limit get chunked** before resubmission.

**Principle (topic 4.4 at the batch layer):** *retry fixes what the model mis-assembled; it never fixes what the request made impossible.*

**Two questions to answer, in order, on any batch-failure item:**
1. *Which requests do I resend?* → only the failures, by `custom_id`
2. *What must change first?* → nothing if transient; **the request itself** if it's a validation error

Options that get one right and the other wrong are the designed traps.

---

## 6. SLA math: batch submission frequency

A document waits **twice**:

```
document arrives                                    result available
      │                                                     │
      ├──────── WAIT 1 ────────┬────────── WAIT 2 ──────────┤
      │  in YOUR queue until   │   Anthropic processing     │
      │  the next submission   │   (up to 24h)              │
      │                    submission                       │
```

**Wait 1** is under your control (set by cadence). **Wait 2** is not — ceiling is always 24h.
Worst-case Wait 1 = **the full submission interval** (a doc arriving one second after a batch goes out waits the entire next cycle).

### The formula, both directions

```
worst-case latency  =  submission interval  +  24h

Given an SLA → interval ≤ SLA − 24h   (then pick UNDER the ceiling for headroom)
Given a cadence → SLA = interval + 24h
```

### Worked example (the exam guide's own)

30-hour SLA: `interval ≤ 30 − 24 = 6h` (the ceiling, zero margin). Choose **4 hours** → 4 + 24 = **28h** worst case, 2h headroom.

Timeline for a document arriving at 00:01 with 4-hourly submissions (00:00, 04:00, 08:00…):

| Time | Event | Elapsed |
|---|---|---|
| 00:01 | Arrives, queued (just missed 00:00 batch) | 0h |
| 04:00 | Next submission — enters the batch | ~4h |
| 28:00 | Worst-case completion (04:00 + 24h) | ~28h |

### Two rules that never bend

1. **Always budget the full 24 hours.** Typical completion time is not a guarantee. *There is no latency SLA.*
2. **Never land exactly on the ceiling.** The maximum permitted cadence is a bound to stay *under*, not a target to hit.

### The inversion worth knowing

If `SLA − 24h` is **negative**, no cadence works — even submitting continuously. **An SLA under 24 hours is a disguised "don't batch" question.** Answer: synchronous API.

### Second-order tradeoff

Submit **as infrequently as the SLA allows**, not as frequently as you can. Over-frequent submission fragments batches, multiplies jobs to track, reduces shared-prefix cache reuse, and buys latency headroom nobody asked for.

### Bursty arrivals

The "interval" in the formula is really **worst-case queue wait**. With uniform arrivals and fixed-clock submission they're identical. With bursty arrivals you can align the cadence to the burst, collapsing queue wait to roughly the burst width.

---

## 7. Prompt refinement on a sample before the big run

Batch **amplifies prompt defects economically**. A 15% failure rate on 50,000 documents means 7,500 failures, wasted spend on bad output, *and* another up-to-24-hour cycle to redo them — effective turnaround becomes 48 hours.

**The correct sequence:**
1. Take a **representative sample** (~50–200 items, covering known edge cases)
2. Run it **synchronously** — you need fast iteration; at that volume cost is trivial
3. Refine using explicit criteria (4.1), few-shot examples (4.2), schema enforcement (4.3), validation rules (4.4)
4. Only then submit the full volume as a batch

**The deliberate inversion:** use the *expensive* API for the *small* run and the *cheap* API for the *large* run.

| Phase | Volume | What matters | Right API |
|---|---|---|---|
| Refinement | ~150 | **Iteration speed** | Synchronous |
| Production | 80,000 | **Cost per token** | Batch |

**Rule:** *Iterate synchronously on a small sample; commit the volume to batch once.* **Never use the batch API as your feedback loop** — a 24-hour cycle time is fine for delivering results and disastrous for developing a prompt.

---

## 8. Anti-patterns to recognize on sight

| Anti-pattern | Why it's wrong |
|---|---|
| Batching a pre-merge / interactive / gating workflow to save money | Converts a bounded wait into an unbounded one; cost never justifies it |
| Choosing batch because the volume is large | Volume is irrelevant; **latency tolerance** is the criterion |
| Treating "automated" as "non-blocking" | A stalled pipeline is still a blocked workflow |
| Planning cadence from typical completion time | There is no SLA; budget the full 24h |
| Landing exactly on the cadence ceiling | Zero margin; one late job breaches the SLA |
| Matching results to inputs by array position | Arbitrary ordering → silent cross-contamination |
| Resubmitting the whole batch after partial failure | Pay twice for completed work; use `custom_id` |
| Blind-retrying a validation error unmodified | Deterministic failure; must change the request |
| Designing an agentic tool loop inside one batch request | Batch has no harness |
| Abandoning batch entirely because one design won't batch | Restructure the workload (resolve-then-batch) instead |
| Chaining batches when your code could resolve the step | Stacks two 24h ceilings for no reason |
| Firing large volume at an untested prompt | Failures cost a second 24h cycle plus wasted spend |
| Using batch as the prompt-refinement feedback loop | 24h per iteration; use sync for the sample |
| Disabling prompt caching for batch workloads | Caching works in batch and stacks with the discount |

---

## 9. Quick recall card

- **Batch = 50% off, ≤24h, no SLA.**
- **Blocking → sync. Latency-tolerant → batch.** Volume is not the criterion.
- **No multi-turn tool execution** inside a batch request; structured-output tool use is fine.
- **`custom_id`** correlates results (arbitrary order) and scopes retries.
- **`interval ≤ SLA − 24h`.** SLA < 24h → batch impossible.
- **Resubmit only failures, with modifications** for validation errors.
- **Refine on a sample synchronously first**, then batch the volume.

### The three facts everything derives from
1. **50% off, up to 24 hours, no guaranteed SLA.**
2. **The criterion is latency tolerance, never volume.**
3. **One round trip per request — no tool execution loop.**

### Cross-topic seams the exam likes

| Combined with | Question shape |
|---|---|
| **1.1** (agentic loops / harness) | "Why can't this agent workflow be batched?" → the harness lives outside the API |
| **4.3** (structured output) | "Can this extraction pipeline be batched?" → yes, schema tool use is single-turn |
| **4.4** (validation & retry) | "How to handle the failures" → modify then resubmit |
| **3.6** (CI/CD) | The pre-merge-check-is-blocking case sits exactly on this seam |

---

# Practice Questions

## Question 1 — blocking vs. non-blocking

An engineering organization runs two automated code-analysis workloads, both currently on the synchronous Messages API. Leadership has asked the team to reduce API spend.

- **Workload A — pre-merge security analysis.** Every PR triggers an analysis of the diff. The CI pipeline blocks the merge until a pass/fail verdict returns. ~400 PRs/day.
- **Workload B — nightly architectural drift report.** At 01:00, a job analyzes all commits merged in the previous 24 hours and produces a summary. The platform team reads it the next morning. ~2,000 commits/night.

Both use tool definitions with forced `tool_choice` to return findings in a fixed JSON schema.

Which approach most effectively reduces cost?

**A.** Move both workloads to the Message Batches API. Both are fully automated pipelines with no human typing in real time, and batching halves the cost of both.
**B.** Keep Workload A on the synchronous Messages API; move Workload B to the Message Batches API.
**C.** Move Workload A to the Message Batches API and keep Workload B synchronous, since Workload B processes five times the volume and needs faster turnaround at that scale.
**D.** Keep both on the synchronous Messages API, since the Message Batches API does not support the tool use both workloads depend on.

### ✅ Correct answer: **B**

**Why B is right.** The criterion is what is waiting on the result. Workload A is blocking by definition — the CI pipeline holds the merge until the verdict returns; batch's 24-hour ceiling with no SLA could leave a PR unmerged for a day. This is the exam guide's named example of an inappropriate batch workload. Workload B runs at 01:00, is read the next morning, and blocks nothing — the textbook appropriate case.

**Why A is wrong.** "Automated, therefore non-blocking" — true premise, invalid conclusion. *Automated* and *non-blocking* are different properties. No human types in real time for Workload A, but a developer waits for their merge and a pipeline is stalled behind it. The question is never "is a person at a keyboard," it's "is anything downstream stuck."

**Why C is wrong.** Two stacked errors. It moves the blocking workload to batch and keeps the tolerant one synchronous — exactly backwards. And it reasons from volume (2,000 > 400), which is not the criterion. Volume affects how much you *save*; latency tolerance determines whether you're *allowed* to batch.

**Why D is wrong.** Conflates tool *use* with tool *execution*. Batch fully supports forced `tool_choice` producing a single `tool_use` block as the final answer — one round trip. What batch cannot do is multi-turn tool calling. Neither workload does that.

---

## Question 2 — SLA cadence calculation

A legal services company processes contracts submitted by clients throughout the day. Contracts are queued and submitted to the Message Batches API on a fixed schedule. The customer agreement guarantees every contract is analyzed and returned **within 34 hours** of submission.

Over the past four months, batches have completed in an average of 70 minutes and never longer than 5 hours.

What is the most appropriate submission schedule?

**A.** Every 24 hours. Worst case is 24h queue + the observed 5h maximum processing = 29h — within the guarantee.
**B.** Every 10 hours. Worst case is 10h + 24h = 34h — exactly meeting the guarantee.
**C.** Every 6 hours. Worst case is 6h + 24h = 30h — within the guarantee.
**D.** Every 30 minutes. Minimizing queue time provides the largest possible safety margin.

### ✅ Correct answer: **C**

**Why C is right.** `interval ≤ 34 − 24 = 10h` is the ceiling. Six hours sits comfortably under it: 6 + 24 = 30h against a 34h guarantee, leaving 4h of headroom, while keeping batches large (4/day) and operations simple.

**Why A is wrong — the central trap.** The arithmetic is internally consistent and the operational history is real, but **the Batches API has no guaranteed latency SLA**. Observed behavior is not a contract. The only plannable number is the 24-hour window, making the true worst case 24 + 24 = **48h** — a 14-hour breach. The reassuring "four months, never longer than 5 hours" detail exists to make A feel evidence-backed; that kind of history should increase suspicion, not confidence.

**Why B is wrong.** Correct ceiling, zero margin. 10 + 24 = 34 lands precisely on the guarantee. One late submission, clock drift, or a batch genuinely using its full window breaches a contractual commitment. When an exam offers both the exact ceiling and a sensible value beneath it, take the value beneath.

**Why D is wrong.** It doesn't breach (0.5 + 24 = 24.5h), but margin beyond what the guarantee requires buys nothing, while 48 submissions/day fragments batches, multiplies jobs to track, and reduces shared-prefix cache reuse. **Submit as infrequently as the SLA allows, not as frequently as you can.**

---

## Question 3 — partial failure handling

A media monitoring company submits a nightly batch of 12,000 news articles for entity extraction, each with `custom_id` of the form `article-<id>`. Results: 11,540 `succeeded`, 460 `errored`. All 460 are validation errors indicating the request exceeded the context window — long-form investigative pieces. There is time for one more batch cycle before a 09:00 dashboard deadline.

What is the most effective next step?

**A.** Resubmit the entire batch of 12,000, since a uniform re-run guarantees a consistent extraction pass across the full corpus.
**B.** Resubmit only the 460 failed articles by `custom_id`, without modification — context-window errors are transient and typically succeed on a second attempt.
**C.** Resubmit only the 460 failed articles by `custom_id`, after splitting each long article into smaller chunks that fit within the context window.
**D.** Reprocess the 460 through the synchronous Messages API instead, since the synchronous endpoint is not subject to the context-window limits that apply to batch requests.

### ✅ Correct answer: **C**

**Why C is right.** Two decisions, both correct. *Which to resend:* only the failures — `custom_id` maps them straight back to source articles, and 11,540 already have correct, paid-for results. *What to change:* a context-window overflow is **deterministic** — a property of the input, not the attempt. The exam guide names this exact remedy: chunk the oversized documents, then resubmit.

**Why A is wrong.** Pays full price a second time for 11,540 completed extractions, consumes another 24h window on finished work, and still fails the 460 — nothing about them changed. "Consistency across the corpus" is hollow: the successes are already consistent. `custom_id` exists so retries can be scoped.

**Why B is wrong — strongest distractor.** Half correct: right subset, wrong treatment. It treats a validation error as transient. Those categories behave oppositely — transient/server-side errors are safe to retry unmodified; **validation errors are deterministic and fail identically forever**. An article 40,000 tokens too long on Monday is 40,000 tokens too long on Tuesday. This burns the last cycle and misses the deadline. Read the error *category*, not just the error *count*.

**Why D is wrong.** False premise. The context window is a **model** limit, not a batch-endpoint limit. Switching APIs changes scheduling and price, not how much text fits. The same 460 fail the same way, faster and at full price. The *only* capability difference between batch and sync is multi-turn tool execution.

---

## Question 4 — the tool-use boundary

A compliance team wants to audit 30,000 vendor contracts monthly. For each contract the analysis must (1) look up the vendor's current risk rating from an internal risk-scoring service, (2) analyze the contract against policy rules for that risk tier, (3) return findings in a fixed JSON schema. The audit runs on the 1st and is reviewed over the following week. Cost is a significant concern.

An engineer proposes defining a `get_vendor_risk_rating` tool and submitting all 30,000 contracts to the Message Batches API, letting the model call the tool per contract and then produce findings.

Which assessment is most accurate?

**A.** The proposal will work as designed. The Batches API supports tool definitions, and the workload is latency-tolerant, so batching is appropriate.
**B.** The proposal will not work, because the Batches API cannot execute a tool mid-request and return its result to the model. The team should look up all 30,000 risk ratings first, embed each rating in its contract's request, and then submit a single batch.
**C.** The proposal will not work, because the Batches API cannot execute tools. The team should run the entire audit on the synchronous Messages API, accepting the higher cost as the price of tool support.
**D.** The proposal will work, but only if the team removes the fixed JSON schema requirement, since forced `tool_choice` and a client-side tool cannot both be used in a single batch request.

### ✅ Correct answer: **B**

**Why B is right.** *Diagnosis:* the proposal requires the model to call the tool, have a harness execute it, receive the rating, and then analyze — multi-turn tool calling, which batch does not support. *Remedy:* the model never needed to *decide* anything about the lookup; each contract has one vendor with one current rating, so your code can resolve all 30,000 up front and embed them. The workload collapses to single-turn and batches perfectly — while keeping the monthly cadence, the 50% discount at 30,000 requests, and the JSON schema via forced `tool_choice`.

**Why A is wrong.** Conflates supporting tool *definitions* with *executing* them. Both halves of its reasoning are individually true, but the restriction is on the execute-and-feed-back loop. A batch request will return a `tool_use` block asking for the rating and then stop — 30,000 unanswered tool calls instead of 30,000 audits.

**Why C is wrong — correct diagnosis, over-corrected remedy.** Treats "this design can't be batched" as "this *workload* can't be batched," and overstates the restriction: the requirement is that the rating be *present*, not that the model *fetch* it. Abandons the 50% discount on 30,000 requests to preserve a design choice that was never load-bearing. **When a workload is latency-tolerant, restructure it to fit batch rather than abandoning batch.**

**Why D is wrong.** Invented constraint — nothing prevents forced `tool_choice` and other tool definitions coexisting. It also inverts causation: the JSON schema is the one part that was never a problem, since single-turn schema enforcement is what batch handles well. It sacrifices the working component and preserves the broken one.

**Note on the road not taken.** Chaining batches (batch 1 emits tool calls → code executes → batch 2 continues) is legal but stacks two 24h ceilings. **Resolve-then-batch beats chain-batches whenever your own code can determine what the tool call would have been.**

---

## Question 5 — sample-first prompt refinement

A retail analytics company needs structured sentiment and product-attribute data from 80,000 customer reviews. Results feed a quarterly report due in **six days**. The extraction prompt is newly written and untested at scale. Cost matters; the team has correctly identified this as latency-tolerant and suited to batch.

Which approach is most effective?

**A.** Submit all 80,000 as a single batch immediately. Review outputs when they return, then refine the prompt and resubmit any requests whose output was unusable.
**B.** Run a representative sample of ~150 reviews through the synchronous Messages API, refining the prompt until first-pass output is reliably correct, then submit all 80,000 as a single batch.
**C.** Submit a representative sample of ~150 as a small batch, refine based on those results, repeat with further small batches until reliable, then submit the full 80,000.
**D.** Split the 80,000 into eight sequential batches of 10,000, refining the prompt between each batch based on the previous batch's output quality.

### ✅ Correct answer: **B**

> **⚠️ I answered C on this question.** The strategy (sample → refine → scale) was right; the *mechanism* was wrong. **The sample goes through the synchronous API.**

**Why B is right.** It splits the two phases by what each actually needs: refinement needs **iteration speed** (~150 requests, cost trivial) → synchronous; production needs **cost per token** (80,000 requests) → batch. That inversion — expensive API for the small run, cheap API for the large run — is the point. Refine with 4.1–4.4 techniques, then commit once: ~24 hours, five days of margin.

**Why C is wrong — closest distractor.** Right strategy, wrong mechanism: it runs the sample through **batch**, so every refinement cycle costs up to 24 hours.

```
cycle 1 (sample)   up to 24h
cycle 2 (refine)   up to 24h
cycle 3 (refine)   up to 24h
full 80,000 run    up to 24h
                 = up to 96h (4 days) of a 6-day budget
```

Three cycles is optimistic for a new prompt. You spend the deadline waiting on a queue, for a sample whose synchronous cost is a rounding error.

**Why A is wrong.** Refine after the fact, at full scale. A 15% unusable rate means 12,000 bad extractions paid for, plus a second batch cycle — up to 48h plus diagnosis, with the wasted spend already sunk. This is exactly the "iterative resubmission cost" sample-first exists to prevent: batch **amplifies prompt defects economically**, because you discover them 80,000 units too late.

**Why D is wrong.** Inherits C's flaw and adds its own. Eight sequential batches at up to 24h each is up to **eight days** — breaks the six-day deadline on scheduling alone. And the early batches are production runs against an unrefined prompt, so you pay for low-quality output rather than cheap sample output. Slow iteration *and* wasted spend.

---

## Question 6 (bonus) — the `custom_id` ordering trap

A recruiting platform submits 5,000 résumés for skills extraction. Each request has `custom_id` of the form `candidate-<id>`; all 5,000 share a large cached system prompt containing the skills taxonomy. The batch returns with all 5,000 `succeeded` — no errors, no expirations.

QA finds extracted skills attributed to the **wrong candidates**. Spot-checking shows each result object is internally coherent and accurately reflects *some* résumé in the submission — just not the one it was filed under. The ingestion pipeline pairs each result with the input request at the same array index.

What is the most likely root cause?

**A.** Model hallucination compounding at scale. Reduce batch size and add a validation pass cross-checking extracted names against the source résumé.
**B.** The Message Batches API returns results in arbitrary order. The pipeline must correlate each result to its source request by `custom_id` rather than by array position.
**C.** Silent partial expiration. Some requests exceeded the 24-hour window and were dropped from the results array, shifting every subsequent result out of alignment.
**D.** Prompt cache cross-contamination. Because all 5,000 share a cached prefix, context leaked between requests; disable prompt caching for batch workloads.

### ✅ Correct answer: **B**

**Why B is right.** The diagnostic tell: each result is *internally coherent and accurate for some résumé* — the extractions are correct, the **pairing** is wrong. That rules out model quality and points at the join. Batch returns results in **arbitrary order**; `custom_id` exists to solve this. The pipeline zips by index, so every record is filed against whichever request landed at that position.

The dangerous property: this failure is **completely silent**. All requests succeed, no error is raised, counts match, and each record looks plausible in isolation. Position-based correlation doesn't fail loudly — it quietly produces a fully-populated, entirely wrong dataset. Fix: build a map keyed by `custom_id`.

**Why A is wrong.** Hallucination produces fabricated or incoherent output. What's described is the opposite — every extraction is faithful to a real résumé in the batch. That's a misfiling signature, not a generation signature. The remedies don't touch the cause either: a smaller batch reorders just as freely, and a name cross-check would *detect* the mismatch without fixing it.

**Why C is wrong — most sophisticated distractor.** The failure mode it describes (dropped elements shifting indices) is a real bug class, but its premise is false: expired requests are **not silently dropped** — they return with `result.type: "expired"`, so counts stay intact and the condition is visible. The stem also forecloses it (all 5,000 `succeeded`). C would also produce *partial* misalignment, whereas the stem reports the majority affected.

**Why D is wrong.** Invented mechanism. Prompt caching is a deterministic prefix-reuse optimization; it does not transfer content between requests, and requests stay isolated. Acting on D surrenders a real cost saving that stacks with the 50% discount while the actual bug keeps corrupting every run.

**Generalizable lesson:** when batch output is *correct but attributed wrongly*, look at the correlation layer, not the model. Be suspicious of any pipeline reasoning about batch results positionally.

---

# Clarifications from follow-up discussion

## The SLA math, unpacked

The confusion point was why `4 + 24 = 28`. **A document waits twice** — first in *your* queue until the next scheduled submission (Wait 1, under your control), then for Anthropic's processing (Wait 2, ceiling 24h, not under your control). Worst-case Wait 1 is the **full submission interval**, because a document can arrive one second after a batch goes out.

Worked variations:

| SLA | Ceiling (`SLA − 24`) | Sensible choice | Note |
|---|---|---|---|
| 36h | 12h | 8h → 32h worst case | 12h is the bound, not the target |
| 48h | 24h | 12–18h | Daily = exactly 48h, reckless |
| 34h | 10h | 6h → 30h worst case | Q2 scenario |
| 30h | 6h | 4h → 28h worst case | Exam guide's example |
| 26h | 2h | ~1h | Budget almost entirely consumed by processing |
| 20h | −4h | **impossible** | Use synchronous API |

Reverse direction: submitting every 6h → you can promise **30h** (`6 + 24`).

Two-hop chained batch at 4h cadence each: `(4 + 24) + (4 + 24) = 56h`, plus execution time between hops. Chaining roughly doubles the ceiling.

Trap restated: "batches have completed in 90 minutes for three months" is an **observation, not a guarantee**. Any option budgeting less than 24h for processing is wrong regardless of its arithmetic.

## Tool-use boundary — classification drill

| Workload | Verdict | Reason |
|---|---|---|
| Extract fixed fields from 40,000 invoices with forced `tool_choice`, text in prompt | ✅ | Schema enforcement, no execution |
| Support agent that looks up an order via internal API then replies | ❌ | Needs execution mid-request; also blocking |
| Classify 20,000 tickets via a tool with an `enum` field | ✅ | Output shape only |
| Research agent that searches, reads, decides to search again | ❌ | Agentic loop; needs the harness (1.1) |
| Generate a unit test per source file, contents embedded | ✅ | Single turn; guide's "nightly test generation" case |
| Analyze DB records, but the model must run the query itself | ❌ as written | **Converts:** resolve query first, embed results, batch |
| Model returns `tool_use` block; your pipeline writes it to a warehouse afterwards | ✅ | Execution is *after* the response |
| Model reads a file, runs a linter, sees output, rewrites code | ❌ | Multi-turn |
| 50,000 requests sharing a 30,000-token cached system prompt | ✅ | **Caching works in batch and stacks with the discount** |
| Summarize 5,000 documents, text already in the request | ✅ | No tools at all |

**Two common over-applications of the restriction:**
- Thinking tool use *in general* blocks batching — it doesn't; schema-enforcement tool use is fine.
- Thinking your code acting on the result afterwards matters — it doesn't; that's outside the request.

## Exam-priority ranking for this topic

**Tier 1 (near-certain — the four "Skills in" bullets):** blocking vs. non-blocking decision; SLA cadence calculation; partial-failure handling (subset + modification); sample-first prompt refinement.

**Tier 2 (likely standalone):** no multi-turn tool calling; `custom_id` and arbitrary ordering.

**Tier 3 (recognize only):** numeric API limits (100k requests / 256 MB / 29-day retention) — **not in the exam guide**; SDK method names and result-object field shapes. This exam tests architectural judgment, not SDK surface.
