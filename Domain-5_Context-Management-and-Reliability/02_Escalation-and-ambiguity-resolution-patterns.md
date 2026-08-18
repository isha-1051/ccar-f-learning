# Topic 5.2 — Design effective escalation and ambiguity resolution patterns

**Domain:** 5 — Context Management & Reliability (15% of exam)
**Task Statement:** 5.2
**Topic number:** 2 of 6 within Domain 5 · 26 of 30 overall
**Primary scenario:** Scenario 1 — Customer Support Resolution Agent (`get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target **80%+ first-contact resolution while knowing when to escalate**)

---

## The one-sentence version

Escalation is a **routing decision driven by defined properties of the request** — the customer asked for a human, policy doesn't cover it, or progress has stalled — and **never a reaction to how the customer feels or how the model feels about itself**; identity ambiguity is resolved by **asking one more question**, never by picking a winner.

Two halves, and the exam tests the line between them:

| Half | Question it answers |
|---|---|
| **Escalation design** | When does the agent hand off to a human vs. resolve autonomously? |
| **Ambiguity resolution** | When the agent lacks information to act safely, does it ask, guess, or escalate? |

The three-way discrimination — **resolve / ask / escalate** — is the whole topic. Most wrong answers collapse two of the three.

---

## 1. The three legitimate escalation triggers

The guide's list is effectively **closed**:

1. **The customer explicitly requests a human.**
2. **Policy exceptions / policy gaps** — *"not just complex cases."*
3. **Inability to make meaningful progress.**

### The parenthetical is the exam question

> "policy exceptions/gaps **(not just complex cases)**"

**Case complexity is not an escalation trigger.** A four-order billing dispute with three refunds and a partial reship is *complex* — but if policy covers every piece and the tools work, the agent **resolves it**. That is what an 80% FCR target buys. Conversely, a trivially simple request can be a **mandatory** escalation: "Can you match Competitor X's price?" is one sentence, needs no investigation, and must escalate — because policy is silent.

The axis is **not difficulty**:

| Property of the request | Action |
|---|---|
| Policy is clear **and permits** it, tools can do it | **Resolve** (however complex) |
| Policy is clear **and denies** it | **Resolve** — explain the denial autonomously; a denial grounded in stated policy is a resolution, not an escalation |
| Policy is **ambiguous or silent** on this request | **Escalate** |
| Customer explicitly asks for a human | **Escalate immediately** |
| Agent is looping / cannot advance | **Escalate** |
| Information is missing but the customer has it | **Ask** — neither resolve-by-guessing nor escalate |

Row 2 matters: "escalate everything the customer won't like" is a distractor pattern. A 45-day-old return against a 30-day policy with no discretionary clause gets a stated outcome and reason. Escalating that is **over-escalation**, a graded failure mode on this exam — not a safe default.

---

## 2. Trigger 1 — explicit request for a human: honor it *immediately*

Guide skill verbatim: *"Honoring explicit customer requests for human agents immediately **without first attempting investigation**."*

The tempting wrong design: *"When a customer asks for a human, first attempt to resolve; escalate only if that fails."* It looks like FCR engineering and is the exam's favorite trap here.

Why it's wrong:

- It **overrides an explicit instruction** — the one signal in the system that requires no inference and has a zero false-positive rate.
- It adds latency and tool calls to a case that is going to a human anyway, making the eventual handoff worse.
- It converts a satisfied constraint into a trust failure.
- The FCR "win" is illusory: resolutions extracted from a customer who asked to leave don't hold.

### The narrow boundary: explicit demand vs. frustrated venting

Counterweight skill: *"Acknowledging frustration while offering resolution when the issue is within the agent's capability, escalating only if the customer reiterates their preference."*

| Customer message | Read | Action |
|---|---|---|
| "Transfer me to a person." / "Get me a manager." | **Explicit request** | Escalate now, no investigation |
| "This is ridiculous, I've been waiting a week." | **Frustration, no request** | Acknowledge + offer to resolve now |
| After that offer: "No — I want a human." | **Reiterated** | Escalate now |

Explicit request → immediate escalation. Frustration alone → acknowledge and offer. Reiteration after the offer → escalate. **One offer, not two.**

### What travels with the handoff

Escalating immediately does **not** mean escalating empty. Pass the structured summary already held — customer ID, orders, amounts, commitments, steps taken, reason for escalation (the 5.1 case-facts block doing double duty; same discipline as 5.3's structured error context). The prohibition is on **new investigation**, not on transmitting context you already have. A human restarting from zero is a second failure.

---

## 3. Trigger 2 — policy gaps and exceptions

Guide skill: *"Escalating when policy is ambiguous or silent on the customer's specific request (e.g., **competitor price matching when policy only addresses own-site adjustments**)."* Expect this example near-verbatim.

Policy covers price adjustments **on our own site within 14 days**; the customer wants a **competitor** match. Policy is not restrictive here — it is **silent**.

- ❌ **Reason by analogy and grant it.** "It's in the spirit of the adjustment policy." The agent has now **authored policy** — unbounded financial exposure, run-to-run inconsistency, precedent no human approved. Most attractive distractor, because the reasoning reads as helpful.
- ❌ **Deny it because it isn't covered.** "Our policy doesn't allow competitor price matching" is a **false statement** — policy doesn't *address* it. A denial is as much an unauthorized policy decision as a grant, and may deny what a human would approve. "When in doubt, say no" is the same error with the sign flipped.
- ✅ **Escalate.** Deciding an uncovered case *is* the exception-handling authority, and that authority is human.

**Also report the gap.** A recurring gap is a signal to the policy owners; escalation is the correct runtime behavior, **closing the gap in policy (then in the prompt) is the durable fix**. An option combining "escalate these *and* surface the recurring gap" beats bare escalation.

### Adjacent but different: hard thresholds

*"Refunds above $500 require human approval"* is **not** ambiguity — policy is explicit. That is a deterministic business rule, so it belongs in a **`PreToolUse` hook** that blocks `process_refund` above the threshold and redirects to escalation (Domain 1.5; named in Preparation Exercise 1). Enforcement matched to stakes (Domain 1.4). Don't confuse "programmatic gate on a known rule" with "escalate because policy is unclear."

---

## 4. Trigger 3 — inability to make meaningful progress

The stop condition. Symptoms:

- The same clarifying question asked twice with the same non-answer.
- Repeated tool failures that local retry did not resolve (contrast 5.3: **transient** failures get local recovery; only the unresolvable propagates).
- Several turns with no new information and no state change.
- The required action isn't available through any tool the agent has.

Anti-patterns it prevents:

- **Looping indefinitely** — burning turns and tokens (Domain 1.1: the harness owns loop control and iteration limits).
- **Fabricating a resolution** to close the case — worse, since it produces a confident false answer and a customer who believes the problem is solved.

Distinguish from ambiguity: "I need one identifier the customer can give me" is **not** inability to progress — that's §6, ask.

---

## 5. Why sentiment and self-reported confidence are unreliable proxies

Guide knowledge point: *"Why **sentiment-based escalation** and **self-reported confidence scores** are unreliable proxies for actual case complexity."*

### Sentiment-based escalation

- **Orthogonal to whether a human is needed.** A furious customer with a clear duplicate charge is a one-tool-call resolution; a calm customer asking for a competitor match is a mandatory escalation.
- **Escalates the loud, strands the quiet** — both error directions at once. Over-escalation destroys FCR; under-escalation leaves politely-worded policy gaps to the agent's invention.
- **Answers the wrong question.** Frustration calls for **acknowledgement**, which the agent can supply while resolving. Routing to a human is not the remedy for an emotion.
- **Gameable and drifty** — tone, phrasing, and language move the score without the case changing.

### Self-reported confidence scores

- **Uncalibrated by construction.** Tracks *fluency*, not difficulty or correctness. The cases most needing a human are the ones it doesn't recognize — it doesn't know policy is silent, so it answers by analogy at 0.9.
- **Self-attestation by the failing component** — same defect as 5.1's "confirm you used all sources" and 4.6's self-review.
- **Unstable across runs and prompt versions**, so any tuned threshold decays.
- Threshold tuning trades one error direction for the other with no good operating point, because the signal carries little information about the real triggers.

### The 5.2 vs 5.5 contrast (heavily testable)

| 5.2 — rejected | 5.5 — endorsed |
|---|---|
| Agent's **self-reported confidence about its own case handling**, used live to decide escalation | **Field-level confidence scores** on extractions, **calibrated against labeled validation sets**, used to route review attention |
| No ground truth, no calibration, self-referential | Ground truth exists, calibration is the explicit requirement, routing is offline/batch |

The difference is **calibration against labeled data** and **what the score is about**. Confidence-based escalation *without* calibration is the 5.5 pattern smuggled into a 5.2 setting — always wrong.

---

## 6. Ambiguity resolution — multiple matches require clarification, not heuristics

Guide knowledge: *"How multiple customer matches require clarification (**requesting additional identifiers**) rather than heuristic selection."* Skill: *"Instructing the agent to ask for additional identifiers when tool results return multiple matches, rather than selecting based on heuristics."*

Canonical setup: `get_customer("John Smith")` returns **three** records.

**Correct:** ask for one more identifier — order number, email, ZIP, phone, last 4 of card. One turn, ambiguity gone.

**Wrong — every heuristic, and all appear as distractors:** most recent activity · highest lifetime value · the one with an open order matching the complaint · first result / highest match score.

Why heuristics are categorically wrong, not merely less accurate:

1. **Silent wrong-entity binding.** The tool call succeeded (HTTP 200), downstream calls succeed, nothing in the logs distinguishes a failed session from a good one. Same family as 5.1's facts-bound-to-the-wrong-issue and 5.3's silent error suppression.
2. **Irreversible downstream actions.** A refund, cancellation, address change, or unlock applied to the wrong person can't be undone.
3. **Privacy breach.** Reading a different customer's orders and shipping address back in chat is a disclosure incident.
4. **The customer holds the answer**, one question away. Guessing to preserve a "smooth experience" trades certainty for a coin flip.

### Ambiguity is not escalation

Escalating on multiple matches is an equally-graded error. No policy gap, no explicit request, no stalled progress — nothing requires human authority, and a human agent would resolve it by asking for an email address. **Ask when the missing information is something the customer can supply; escalate when the missing thing is *authority* or *capability*.**

### Tool-side complement

The named skill is the prompt instruction, but a tool that **auto-selects a best match and returns one record** makes the failure invisible to the model — it can't ask about ambiguity it never sees. A well-designed `get_customer` **returns all matches with disambiguating fields** (masked email, ZIP, last order date) so the agent can detect the ambiguity and phrase a useful question (Domain 2.1 result shaping). This composes with the instruction; when the choice is heuristic vs. asking, asking wins outright.

Ask for the **minimum sufficient** identifier — never sensitive data (full card number, SSN) to disambiguate a name.

---

## 7. Implementing it: explicit criteria + few-shot examples

Guide skill: *"Adding **explicit escalation criteria** with **few-shot examples** to the system prompt demonstrating when to escalate versus resolve autonomously."*

- **Explicit criteria (Domain 4.1).** Enumerate the triggers as rules. "Escalate when you're not sure you can help" is the under-specified-boundary problem: vague pressure produces over-triggering — here, over-escalation.
- **Few-shot examples (Domain 4.2).** The hard cases are **judgment boundaries** prose can't pin down: frustration-with-no-request vs. explicit demand; policy-silent vs. policy-denies; complex-but-covered vs. simple-but-uncovered. Examples calibrate; they don't guarantee, which is why hard money rules go in a hook.
- **Enumerate both sides.** Include **resolve-autonomously** examples, especially a complex-but-covered case. One-sided example sets bias toward escalation and wreck the FCR target — the direct analogue of 4.1's inclusion **and** exclusion criteria.

```
ESCALATE when any of these is true:
  1. The customer explicitly asks for a human, manager, or agent
     -> escalate immediately; do not investigate first
  2. Policy is silent or ambiguous about the specific request
     -> e.g. competitor price matching (policy covers own-site adjustments only)
  3. You cannot make meaningful progress
     -> repeated tool failures after retry, or no new information across turns

RESOLVE AUTONOMOUSLY when:
  - Policy clearly covers the request, even if the case is complex
  - Policy clearly denies the request -> explain the outcome and the reason
  - The customer is frustrated but the issue is within your capability
    -> acknowledge, offer to fix it now; escalate only if they reiterate

ASK, don't guess and don't escalate, when:
  - A lookup returns multiple matches -> request one additional identifier
  - A required detail is missing and the customer can provide it

EXAMPLES
  [escalate] "Put me through to someone."
  [resolve]  "I've been waiting a week, this is unacceptable."
             -> acknowledge + process the refund now
  [escalate] "Can you match the $89 price on <competitor>?"
  [resolve]  "Your site listed it at $89 two days ago." -> own-site, in policy
  [resolve]  45-day-old return, no discretionary clause -> explain the denial
  [ask]      get_customer returns 3 "John Smith" records -> request order # or email
```

Two placement notes:

- **`escalate_to_human`'s tool description** is a second place criteria get read — at decision time, next to the action (Domain 2.1). State trigger conditions and required payload fields there.
- **Layer prompt vs. hook by stakes** (Domain 1.4 / 1.5): judgment-laden triggers → prompt criteria + examples; hard rules with a threshold → deterministic gate.

---

## 8. Decision table — signal → correct action

| Signal in the scenario | Correct action | Attractive wrong answer |
|---|---|---|
| "Let me speak to a human" | Escalate **immediately**, no investigation; pass known context | Attempt resolution first, escalate only if it fails |
| "This is unacceptable, I've waited a week" (in policy) | Acknowledge + offer to resolve now | Escalate on negative sentiment |
| After the offer: "No, a human" | Escalate now | Offer again / keep trying |
| Competitor price match; policy covers own-site only | Escalate — policy is silent | Grant by analogy; or deny "per policy" |
| 45-day return, policy says 30 days, no discretion | Resolve — explain the denial | Escalate because the customer will be unhappy |
| Complex 4-order dispute, fully covered by policy | Resolve | Escalate because it's complex |
| Refund exceeds the approval threshold | `PreToolUse` hook blocks and routes to escalation | Prompt instruction "don't refund over $500" |
| Agent escalates 40% of cases | Tighten criteria; add **resolve-autonomously** few-shot examples | Raise the confidence threshold |
| `get_customer` returns 3 matches | Ask for an additional identifier | Most recent / highest LTV / best match; or escalate |
| Repeated tool failures after retries, no progress | Escalate (inability to progress) with what was attempted | Keep retrying; or close with a guess |
| Sentiment-triggered escalation proposed | Reject — uncorrelated with the real triggers | "Combine sentiment with confidence for robustness" |
| Self-reported confidence thresholds proposed | Reject — uncalibrated self-attestation | Tune the threshold on production traffic |

---

## 9. Anti-patterns worth naming

1. **Investigating before honoring an explicit human request.** Overrides the one zero-inference signal, and worsens a handoff it can't prevent.
2. **Escalating on sentiment.** Escalates the loud, strands the quiet; frustration calls for acknowledgement, not routing.
3. **Escalating on self-reported confidence.** The failing judgment grading itself, uncalibrated.
4. **Treating complexity as the trigger.** Complex-but-covered → resolve. Simple-but-uncovered → escalate.
5. **Filling a policy gap by analogy** — the agent authors policy, inconsistently, with real financial exposure.
6. **Denying because policy is silent** — the mirror error; "policy doesn't allow" misstates "policy doesn't address."
7. **Heuristic disambiguation of multiple matches** — silent wrong-entity binding, irreversible actions, privacy exposure.
8. **Escalating what a single question would resolve** — over-escalation dressed as caution.
9. **One-sided criteria.** Listing only escalate cases guarantees over-escalation.
10. **Escalating with no structured context**, forcing the human to restart from zero.
11. **Prompt-requesting a hard money rule** instead of gating it deterministically.
12. **Gating a valid trigger behind an invalid one** (`frustration AND policy-gap`) — the three triggers are independent and each is sufficient alone.

---

## 10. Compact recall list

- **Three triggers only:** explicit human request · policy exception/gap · inability to progress. **"Not just complex cases."**
- Explicit request → **escalate immediately, no investigation first**. Frustration alone → **acknowledge + offer**; escalate **only on reiteration**.
- Policy **silent or ambiguous** → escalate (canonical: **competitor price match** vs. own-site adjustments). Policy **clear**, even if it denies → resolve and explain.
- **Sentiment** and **self-reported confidence** are unreliable proxies — both error directions, both self-referential. Calibrated **field-level** confidence belongs to **5.5**.
- Multiple matches → **ask for an additional identifier**; never heuristics, never escalation.
- Implement with **explicit criteria + few-shot examples covering escalate *and* resolve** (4.1 + 4.2); hard thresholds in a **`PreToolUse` hook** (1.5); trigger conditions also in `escalate_to_human`'s **description** (2.1).
- **Resolve / ask / escalate.** Ask when the *customer* has what's missing; escalate when *authority or capability* is what's missing.
- Escalation carries **structured context** (5.1 case facts, 5.3 error context) — immediate ≠ empty.

---

# Practice Questions

**Score: 5 / 5.**

## Question 1 — Explicit request for a human

**Scenario 1.** Transcript pattern: customer opens with *"I've already been through this twice with your chat bot. Just transfer me to a real person."* The agent replies that it will pull up the account first so the human gets the full picture, calls `get_customer`, then `lookup_order` twice, then asks two clarifying questions. At turn 4 the customer says *"I said I want a person. Why is this so hard?"* and the agent calls `escalate_to_human`. System prompt: *"Before escalating, gather relevant account and order context so the human agent can resolve the issue efficiently. Escalate when the customer's needs exceed your capability."* Satisfaction on these sessions is among the lowest in the dataset, and nearly all escalate anyway.

**What is the most effective change?**

- **A.** Keep context-gathering but cap it at a single tool call and no clarifying questions.
- **B.** Change the system prompt so an explicit customer request for a human triggers immediate escalation with no prior investigation, passing along whatever case context the agent already holds.
- **C.** Add sentiment analysis so that when frustration is detected alongside a transfer request, the agent skips context gathering and escalates right away.
- **D.** Have the agent acknowledge the request, then ask the customer whether they'd prefer transfer now or an attempt at resolution first.

**Correct answer: B** ✅ *(answered correctly)*

**Why B is right.** An explicit request for a human is the one trigger requiring **zero inference** — no sentiment model, no confidence score, no complexity judgment. Guide skill verbatim: *honoring explicit customer requests for human agents immediately without first attempting investigation*. The current prompt fails because context-gathering is unconditional and sits **before** the escalation check, so the agent audibly overrides a direct instruction. The scenario seals it: **nearly all of these escalate anyway** — the investigation buys no resolutions, only delay. B's second clause matters too: passing context the agent *already holds* is not investigation. Immediate ≠ empty.

**Why A is wrong.** Strongest distractor: the delay shrinks, so the symptom improves. But the defect survives — the agent still acts against an explicit instruction before honoring it. The complaint isn't "four turns," it's "I told you what I wanted and you did something else." Any cap is also an arbitrary guess; optimizing the size of a step that shouldn't run.

**Why C is wrong.** Makes an **unreliable proxy the gate on a reliable signal** — exact inversion of the right design. The transfer request alone is sufficient and unambiguous; conditioning correct behavior on a frustration score means a calm "please transfer me" still gets the investigation loop. Being upset becomes the price of being listened to.

**Why D is wrong.** Re-asks a question the customer **already answered**; the transfer request *was* the preference. It also misapplies the acknowledge-and-offer rule, which is for **frustration with no explicit request**. Here the request came first, so there's nothing to offer.

> **Rule:** explicit request → escalate immediately, no investigation, with known context attached. Frustration *without* a request → acknowledge + offer once, escalate on reiteration.

---

## Question 2 — Policy gap (competitor price matching)

**Scenario 1.** Published policy: *"If an item you purchased is listed at a lower price **on our site** within 14 days of purchase, we will refund the difference."* Nothing about competitors. Customers regularly ask for competitor matches. The agent handles them inconsistently: sometimes granting a refund by "spirit of the policy" reasoning, sometimes replying "our policy doesn't allow competitor price matching," sometimes asking for the competitor link first. Finance flags ~$30,000 in unbudgeted refunds last quarter; customer-facing teams report complaints from customers denied after a friend with an identical request was approved.

**What is the most effective change?**

- **A.** Extend the system prompt with an explicit rule: competitor price matching is not covered by policy, so the agent must decline these and explain that adjustments apply only to items listed lower on our own site.
- **B.** Add an explicit escalation criterion, with few-shot examples, directing the agent to escalate when a request falls outside what policy addresses — including competitor price matching — and surface the recurring gap to the policy owners.
- **C.** Add a `PreToolUse` hook that blocks `process_refund` when the refund reason is a price adjustment and the amount exceeds $25, routing those cases to `escalate_to_human`.
- **D.** Have the agent output a confidence score for its policy interpretation and escalate below a calibrated threshold.

**Correct answer: B** ✅ *(answered correctly)*

**Why B is right.** The guide's canonical example almost verbatim. Policy is **silent**, not restrictive, and deciding an uncovered case *is* the exception-handling authority — human by definition. B fixes both harms with one mechanism: the $30k stops (no more self-granted exceptions) and the fairness complaints stop (one consistent human decision instead of a per-session coin flip). Two details make it *most* effective: **few-shot examples** calibrate policy-silent vs. policy-denies vs. policy-permits (otherwise "outside policy" over-triggers on the 45-day return); and **surfacing the gap** is the durable fix — escalation is the stopgap, the policy update is the cure.

**Why A is wrong.** Sharpest distractor: it fixes both *measured* symptoms cheaply. But its justification is itself **a policy decision the prompt author isn't authorized to make** — converting silence into prohibition is an unauthorized ruling with the sign flipped, foreclosing every case a human would have approved. And the customer-facing text is a **factual misstatement**: policy doesn't *address* competitor matching. The system prompt is not where policy is authored; if the business decides to decline, that ruling comes from policy owners and *then* gets encoded — which is the loop B's second clause opens.

**Why C is wrong.** Legitimate mechanism (§3), wrong variable. The defect is **coverage**, not **amount**, so the gate is wrong in both directions: a $12 competitor match passes unauthorized, and a $40 **own-site** adjustment that policy explicitly permits gets blocked and dumped on a human, wrecking FCR on cases the agent should own. It also depends on the model self-labeling "refund reason," reintroducing the judgment the hook was meant to make deterministic. Hooks enforce **known explicit rules**; they can't decide what policy never answered.

**Why D is wrong.** §5's rejected proxy, and "calibrated" can't be supported — there's no labeled ground truth for "was this the right reading of a policy that doesn't address the case." Worse, the failure mode is a **confident** one: analogy reasoning feels fluent, so the cases you most need to catch score highest. (Contrast 5.5, where confidence *is* endorsed because extractions have verifiable answers and a labeled validation set.)

> **Rule:** policy **silent or ambiguous** → escalate, and close the gap upstream. Policy **clear**, even when it denies → resolve and explain. Neither granting by analogy nor denying by default is the agent's call.
>
> *Side note:* asking for the competitor's link isn't ambiguity resolution. Ambiguity-resolution is for information the customer holds that the agent needs to **act correctly**. Here the missing thing is **authority**, which no customer-supplied detail creates.

---

## Question 3 — Multiple matches / heuristic selection

**Scenario 1.** `get_customer` returns **four** records for *"Maria Garcia."* Current instruction: *"Identify the customer from the `get_customer` results and proceed with their request."* The agent selects the record with the most recent account activity. Two incidents: (1) it cancelled a **different** Maria Garcia's order and confirmed the cancellation to the caller, who found out when her own order shipped; (2) it read back recent orders — items, dates, shipping address — belonging to someone else. Logs don't distinguish these sessions: `get_customer` returned HTTP 200 and all downstream tool calls succeeded.

**What is the most effective change?**

- **A.** Improve the selection heuristic to weight several signals together — recent activity, whether an open order matches the stated request, account status — and select the highest-scoring record.
- **B.** Instruct the agent to escalate to a human whenever `get_customer` returns more than one match, since misidentification carries privacy and financial risk.
- **C.** Instruct the agent that when `get_customer` returns multiple matches it must ask the customer for an additional identifier (order number, email, or ZIP) and re-query, rather than selecting a record itself.
- **D.** Add a `PreToolUse` hook blocking refund and cancellation calls when the preceding `get_customer` returned more than one match, requiring human approval.

**Correct answer: C** ✅ *(answered correctly)*

**Why C is right.** Guide skill verbatim. The decisive facts: the **customer holds the missing information** and it costs **one turn**. Asking makes identity certain rather than probable, matches what human agents do, and returns a single record so nothing downstream is guessing. The logs paragraph is the tell — HTTP 200, all calls successful — a **silent** wrong-entity binding, invisible to monitoring (same family as 5.1's wrong-issue binding and 5.3's suppressed errors), so it must be prevented at the decision point.

**Why A is wrong.** The topic's core error, seductive because a multi-signal score really is more accurate. But accuracy is the wrong frame: the failure isn't a poor guess, it's **guessing at all**. Both incidents recur at any non-zero error rate with unbounded consequences — an irreversible cancellation on a stranger's order, and a **privacy disclosure**. A better heuristic makes errors rarer and *harder to detect*. It's also circular: "an open order matching the stated request" uses the caller's claim to decide whose record to trust — the very thing needing verification.

**Why B is wrong.** Over-escalation, failing the three-way test: escalate when **authority or capability** is missing, ask when the **customer has the information**. A human would resolve this by asking for an email. B queues the most common ambiguity in support work, burning the FCR target and delivering a worse experience. The risk framing argues for *certainty*, not for a *human* — and C delivers certainty.

**Why D is wrong.** Strongest distractor, and genuine defense-in-depth for irreversible actions — but a backstop, not the fix. **Incomplete:** incident 2 involved no gated tool call; the disclosure happened when the agent read back already-retrieved details, which no `PreToolUse` hook on refunds/cancellations sees. **Too late:** it intercepts at the action boundary, after the agent has bound to the wrong customer and started speaking to them as that person. **Needlessly expensive:** every multi-match session becomes a human approval when one question dissolves it. D and C compose; only C addresses the root cause.

> **Rule:** multiple matches → **ask for one more identifier**. Never a heuristic, never an escalation. Guessing returns HTTP 200 all the way down.
>
> *Tool-side complement:* a `get_customer` that silently returns its single best match hides the ambiguity from the agent too. Return **all candidates with disambiguating fields** so the model can detect it and ask (Domain 2.1).

---

## Question 4 — Over-escalation and criteria design

**Scenario 1.** After two months, FCR is **52%** against an 80% target; roughly one session in three escalates. Audit of 200 escalated sessions: ~15% correct escalations (explicit request, or policy didn't address the request); **~85% fully covered by policy and executable with existing tools**, clustering on sessions with **multiple issues**, **long detailed messages**, or **three or more tool calls**. Human agents resolved that 85% using the same policy documents, granting no exceptions. System prompt line: *"Escalate to a human when you are not confident you can fully resolve the customer's issue, or when the case is complex."*

**What is the most effective change?**

- **A.** Replace the vague criterion with a measurable one: escalate when a session exceeds five tool calls or the customer raises more than two distinct issues.
- **B.** Have the agent emit a self-reported confidence score per session and escalate below a threshold tuned against the audited 200-session sample.
- **C.** Replace the criterion with explicit escalation triggers — explicit request for a human, policy silent or ambiguous, inability to make meaningful progress — plus few-shot examples demonstrating both escalation cases **and** complex-but-covered cases the agent should resolve autonomously.
- **D.** Add a second Claude instance that reviews each proposed escalation before dispatch and returns sessions it judges resolvable.

**Correct answer: C** ✅ *(answered correctly)*

**Why C is right.** The audit hands you the diagnosis: escalations cluster on **multiple issues, long messages, 3+ tool calls** — all measures of *effort*, not *authority* — and humans closed 85% with the same documents and **no exceptions granted**, proving no human was required. The prompt line contains both rejected axes in nine words: *"not confident"* (self-assessment proxy) and *"complex"* (explicitly negated: **"not just complex cases"**). C replaces them with properties of the **request** (4.1's explicit-criteria discipline applied to routing). The decisive clause is the **resolve-side examples**: one-sided boundaries over-trigger, and the shape that needs demonstrating is precisely a **complex-but-covered** case worked end to end.

**Why A is wrong.** Most instructive distractor: it fixes the *stated* flaw (vagueness) and still gets it wrong, because "measurable" isn't the goal — measuring the **right property** is. A hardcodes complexity, the ruled-out trigger. Wrong in **both** directions: it mandates escalating the 3-tool-call sessions the audit says to resolve, and lets a one-sentence competitor price-match request through untouched (under-escalating the correct 15%). It also punishes multi-issue messages, which the right design **decomposes and works through** (1.4, with per-issue state from 5.1).

**Why B is wrong.** §5's rejected proxy, quantified. The score tracks **fluency**, so long multi-issue messages depress it exactly where the agent is already wrong. "Tuned against the audited sample" isn't 5.5 calibration: the label would be "could this have been resolved autonomously," a judgment about the model's own capability with no ground truth at decision time. Even a perfect threshold sits on a signal carrying almost no information about the three triggers — a case can be authority-blocked and *feel* easy, or fully covered and *feel* hard.

**Why D is wrong.** A **repair stage where a criteria fix is available** — same anti-pattern as 5.1's fact-checking subagent. The reviewer **inherits the defect**: nothing gives it better criteria than the primary agent has (fresh context helps with *contamination* per 4.6, not with a wrong rule both instances read). It's also one-directional — catches over-escalation, never the missed policy gap — and taxes every escalation with an extra call plus a bounce-back, so the correct 15% wait longer.

> **Rule:** triggers are properties of the **request** (authority, capability, progress), never of the **effort** (tool calls, issue count, message length) or of the **model's feelings about itself**. Enumerate **both** sides.

---

## Question 5 — Sentiment as a router

**Scenario 1.** A sentiment classifier escalates any session where detected frustration crosses a threshold. Three months of data: escalation volume is dominated by **frustrated customers with straightforward, fully-covered issues** (duplicate charges, delayed shipments) that humans resolve in under two minutes with the same tools; **calmly-worded requests policy doesn't address** (competitor price matching, warranty transfers to a second owner) are almost never escalated and are answered inconsistently by the agent; a handful of customers who **asked plainly for a human in a neutral tone** stayed in the automated flow for several turns. Post-escalation surveys from the first group frequently say *"I didn't need a person, I just needed someone to fix it."*

**What is the most effective change?**

- **A.** Raise the frustration threshold so only severe frustration escalates.
- **B.** Keep sentiment escalation but require both conditions: escalate when frustration is high **and** the request is not covered by policy.
- **C.** Remove sentiment as an escalation trigger. Drive escalation solely from the request-based triggers, and instruct the agent to acknowledge frustration while offering resolution when the issue is within its capability, escalating only if the customer reiterates a preference for a human.
- **D.** Route frustrated sessions to an empathy-tuned prompt variant that continues attempting resolution, escalating only if that variant fails to resolve the issue.

**Correct answer: C** ✅ *(answered correctly)*

**Why C is right.** All four observations trace to one error: **sentiment was installed as the router**, and it is uncorrelated with the properties that require a human. C removes it and restores the division of labor — frustration gets a **conversational** response, escalation gets a **request-based** trigger. The survey quote names the mechanism outright: *"I didn't need a person, I just needed someone to fix it"* — the remedy for frustration is **resolution plus acknowledgement**, not routing. The final clause is the guide's skill verbatim and also repairs observation 3, since a plainly-worded request for a human now triggers on its own, tone irrelevant.

**Why A is wrong.** Tunes a proxy carrying no information about the real triggers, so it only moves error between directions. Group 1 shrinks; group 3 gets **worse** (a calm request is even further from firing); group 2 — the politely-worded policy gaps, the highest-risk failures here — is untouched. "Safety net for the angriest customers" is the wrong frame: anger isn't the hazard, unauthorized decisions and stalled progress are.

**Why B is wrong.** Strongest distractor, since it introduces the correct signal (policy coverage) and does fix group 1. The defect is the **`AND`**: conjoining a valid trigger with an invalid one makes the valid trigger *conditional on the invalid one*, so a calm competitor price-match request still never escalates — group 2 untouched, and group 3 isn't in the gate at all. B narrows an over-firing rule instead of replacing it, handing sentiment a veto over a criterion that should stand alone. Three independent `OR` triggers, none gated on mood.

**Why D is wrong.** Half-right, worth naming: empathy *is* the correct treatment for frustration, and D reads group 1's need partly correctly. But it changes **tone** while leaving **routing** broken — groups 2 and 3 untouched. Worse, "escalating only if that variant fails to resolve" reintroduces Q1's error as policy: a customer saying "just transfer me" gets more empathetic language and continued attempts, because escalation is gated on the *agent's* assessment of failure rather than the customer's explicit request. It also spends a whole prompt variant on what C gets from one instruction.

> **Rule:** frustration is handled **in the conversation**; escalation is decided by the **request**. Never gate a valid trigger behind an invalid one.

---

## Clarifications & cross-links

- **Escalate vs. ask vs. resolve.** Escalate when **authority or capability** is missing; ask when the **customer holds** the missing information; resolve whenever policy covers it — however complex. Most distractors in this topic collapse two of the three.
- **"Immediate" escalation is not "empty" escalation.** Honoring a human request immediately forbids **new investigation**, not the transmission of context already held. The 5.1 case-facts block is the natural payload; a human restarting from zero is a second failure.
- **Policy-silent vs. policy-denies.** Silent → escalate. Denies → resolve by explaining. Stating "policy doesn't allow it" when policy is merely silent is a factual misstatement *and* an unauthorized decision.
- **5.2 vs. 5.5 on confidence.** Rejected here (self-assessed, uncalibrated, live routing of a whole case); endorsed there (field-level, calibrated on labeled validation sets, routing review attention). An option offering confidence-based escalation without calibration is the 5.5 pattern misplaced.
- **Hooks vs. prompt criteria.** Known explicit rules with thresholds (refund > $X) → `PreToolUse` hook (1.5), enforcement matched to stakes (1.4). Judgment-laden triggers → explicit criteria + few-shot (4.1 + 4.2). Hooks can't decide questions policy never answered.
- **The upstream-fix pattern, again.** Q4's option D (a reviewer instance) is the same anti-pattern as 5.1's fact-checking subagent and 4.4's retry limit: **never add a repair stage where a criteria or contract fix is available** — the repairer inherits the defect. This axis recurs in 5.3 (error propagation) and 5.6 (provenance).
- **Silent failures.** Heuristic disambiguation returns HTTP 200 with all downstream calls succeeding, so monitoring can't see it — the same invisibility that makes 5.3's suppressed errors and 5.1's wrong-entity bindings dangerous.
- **Forward links.** 5.3 covers what a *subagent* propagates upward when it can't proceed (structured error context) — the multi-agent analogue of this topic's escalation payload; 5.5 covers the human-review side (calibrated confidence, stratified sampling) that this topic deliberately excludes.
