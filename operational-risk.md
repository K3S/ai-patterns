---
title: Operational risk
nav_order: 7
---

# Operational risk

The most useful question to ask about an AI integration is not *"how accurate is the model?"* It's *"what happens when the model is wrong?"*

The first question gets answered with a benchmark number: 92% accurate, 96% on the test set, 88.7% on internal eval. The number is comforting. It's also nearly useless, because it tells you nothing about the *cost* of the 8% or the 12% or whatever fraction is wrong. A model that's 99% accurate but causes catastrophic damage on the wrong 1% is much worse than a model that's 80% accurate but produces only correctable mistakes.

The framework this chapter develops has three axes: **reversibility**, **blast radius**, and **verifiability**. The combination of all three — not any one in isolation — determines whether a workflow is appropriate for AI automation, and at what level of supervision.

---

## Reversibility

How hard is it to undo the consequences of a wrong action?

This is the axis everyone has intuitive feel for, but few teams formalize. Some examples ranked from most to least reversible:

- **Display-only outputs** — a draft email shown to a human. Wrong output costs the time to read and discard. Trivially reversible: just don't send it.
- **Internal-system updates with human review before propagation** — a PO recommendation enters the buyer's queue. Wrong recommendation costs the buyer the time to reject it. Cleanly reversible.
- **Internal-system writes that are themselves modifiable** — an inventory adjustment is recorded in the system. A wrong adjustment can be reversed by another adjustment, with audit trail. Reversible with effort.
- **External communications already sent** — an email went out to a supplier. Reversible only by sending another email; the original is in someone else's inbox forever. Practically irreversible.
- **External system actions** — a PO transmitted via EDI to a supplier. Cancellation requires a follow-up message and depends on the supplier's processing speed. Reversible only sometimes, only with effort.
- **Financial commitments** — a wire transfer has cleared. Reversible only by another transfer in the opposite direction, often with fees, sometimes not at all if the receiving party is uncooperative or unreachable.
- **Permanent record changes** — a customer record was deleted. The system has a soft-delete, so the record can be restored. Reversible but messy.
- **Audit-relevant events** — a regulatory filing was submitted. Cannot be unsubmitted. Reversible only by amendment, which is itself a regulatory event.
- **Real-world physical actions** — an automated shipment was released to the dock. Now the truck is loaded; reversal means unloading.

The pattern: as you go down the list, the *cost* of a wrong action goes up by orders of magnitude, and the *speed* at which reversal becomes possible goes from milliseconds to days to never.

A useful rule: **if reversal takes longer than detection**, the action is effectively irreversible for most purposes. By the time you know the AI was wrong, the consequences have already propagated.

---

## Blast radius

How many things does a single wrong action affect?

Distinct from reversibility. A wrong PO that affects one supplier on one part is small blast. A wrong policy change that affects every customer is large blast. A wrong inventory adjustment that affects one SKU is small; a wrong inventory rebalancing that affects the entire DC is large.

Some examples ranked from smallest to largest blast:

- **One record, one entity** — a single PO, a single customer email, a single ticket categorization.
- **One batch, one entity** — today's POs for one supplier; this morning's emails for one buyer.
- **Many records, one entity type** — all open POs for review; all customer-service tickets for the day.
- **Cross-entity** — affects orders, inventory, AR, and supplier records simultaneously (e.g., a price change cascade).
- **System-wide** — a configuration change that affects every workflow.
- **Industry-wide / public** — a published statement, a regulatory filing, a press release.

The pattern: as blast radius grows, the cost of any single wrong action grows, and the difficulty of containing the damage grows. A small-blast wrong action gets caught and corrected before lunch. A large-blast wrong action makes the news.

The interaction between reversibility and blast radius is what produces the worst outcomes. *Irreversible + large-blast = the headline failures.* "AI bot sent every customer a duplicate confirmation," "AI replenishment ordered three months of supply in a single day," "AI pricing engine zero-priced an entire category for two hours." These aren't model accuracy failures; they're architectural failures of irreversibility-times-blast.

---

## Verifiability

How quickly and cheaply can you tell whether the action was right?

Some examples ranked from most to least verifiable:

- **Immediate, automatic, free** — the model classified an email as spam; you can A/B test against ground-truth spam labels and get hourly accuracy. Or: the model extracted line items from an invoice; deterministic checks (totals, type validation) flag inconsistencies in milliseconds.
- **Immediate, human-in-loop** — the model drafted a reply; the human reads it and either accepts, edits, or rewrites. Verification cost = the human's reading time.
- **Delayed, automatic** — the model produced a forecast; ground truth becomes available next month when actual demand lands. Verification is free but lags.
- **Delayed, manual** — the model proposed a triage routing; verification requires sampling and reviewing what happened to the routed work.
- **Eventual, manual, expensive** — the model produced a strategic recommendation; verification requires multi-month tracking and judgment about whether outcomes are attributable.
- **Practically unverifiable** — the model produced a "creative" output that the user accepted; the only signal is whether the user complains.

The pattern: more verifiable workflows are safer to automate, *because the feedback loop closes fast enough to catch and correct errors before they accumulate*. Less verifiable workflows can drift indefinitely without anyone noticing.

A subtle point: verifiability is not just about whether the answer is *checkable*; it's about whether anyone *will* check. A workflow with a human in the loop "for verification" but where the human is overloaded and rubber-stamping has the verifiability of an unsupervised workflow. The structural verifiability matters; the operational verifiability is what actually applies.

---

## The decision matrix

Combining the three axes:

| Reversibility | Blast Radius | Verifiability | Verdict |
| --- | --- | --- | --- |
| High | Small | Fast | **Safe to automate.** Drafts, classifications, summaries, extractions. |
| High | Small | Slow | **Automate with metrics.** Classification or routing where you can sample and audit. |
| High | Large | Fast | **Automate with care.** Mass communications, with cheap verification per item. |
| High | Large | Slow | **Don't automate alone.** Human gating per batch, periodic audits. |
| Low | Small | Fast | **Automate with check.** Single-record commits, with deterministic gating. |
| Low | Small | Slow | **Don't automate alone.** Human in the loop, irreversible-per-record. |
| Low | Large | Fast | **Don't automate.** The verification is fast but the consequences land too far. |
| Low | Large | Slow | **Never automate.** Catastrophic failure mode. |

The matrix is rough. The actual decisions need judgment. But the matrix is a useful counterweight to the model-benchmark mindset, because it forces you to think about *what you're putting at risk*, not just *how often the model's right*.

---

## A worked decision

A team proposes the following AI feature: *"the system will automatically respond to inbound supplier emails using an LLM."*

The wrong question: *how accurate is the LLM at email-style writing?* The right questions:

- **Reversibility.** Can a sent email be unsent? No. Reversibility = practically irreversible.
- **Blast radius.** Each wrong response affects one supplier relationship. If the system replies to many suppliers per day, the daily blast across all of them is medium. A particularly bad response (offending a major supplier) could be large.
- **Verifiability.** The wrongness is visible only when the supplier complains, or when subsequent business consequences emerge weeks later. Verifiability = slow.

Verdict from the matrix: *low reversibility + medium-to-large blast + slow verifiability = don't automate alone.*

Reshape the proposal. Instead of "LLM auto-responds," redesign as "LLM drafts response; buyer clicks Send." This changes:

- Reversibility: still low (once sent, sent), but now there's a confirmation step.
- Blast radius: same.
- Verifiability: now fast — the buyer reads the draft. Low-effort verification.

New verdict: *low reversibility + medium blast + fast verifiability = automate with check.* This is the Drafter pattern. Same LLM, same prompt, same retrieval. Different architecture. Different operational profile.

The reshape took five minutes. The team's instinct to add the human review wasn't extra friction; it was the architecture decision that made the feature viable.

---

## The verification-loop test

When designing any workflow that uses an LLM, ask:

1. *What does it look like when this output is wrong?*
2. *Who notices first, and how quickly?*
3. *How many wrong outputs can accumulate before the noticer catches one?*
4. *Once noticed, how is the wrongness corrected?*

If you can't answer all four, the workflow needs more design before it ships. If you can answer them but the answers are alarming ("nobody notices, ever" or "by the time we notice, the customer has lost the deal"), the workflow needs supervision redesign.

A good answer: *"if the output is wrong, the buyer reading the draft will edit it before sending; the buyer notices within seconds; a wrong output costs the buyer one minute of edit time; the wrongness is corrected by the edit before any external impact."* That's an answer where the consequences are bounded by design.

A bad answer: *"if the output is wrong, the supplier will receive an incorrect email; the supplier might or might not respond; if they respond negatively, AR or the buyer might find out within days or weeks; correction would require an apology and possibly a follow-up call."* That's an answer where wrongness is unbounded and the system is not safe to automate as proposed.

---

## Beyond accuracy: the failure-class question

A useful follow-on: *what kinds of wrong are possible?*

LLMs don't fail randomly. They fail in characteristic ways:

- **Hallucinated specifics.** Plausible-sounding but invented details. Specific dollar amounts, dates, supplier names, part numbers.
- **Confident misinterpretation.** The model interpreted the request differently than the user meant; the output is internally consistent, just answering a different question.
- **Reasonable-seeming cardinality errors.** Lists with the right kind of items but wrong counts. "Returns 5 items" when there are actually 12.
- **Context-window leaks.** Information from earlier in the conversation surfacing in later outputs where it doesn't belong.
- **Style-shift surprises.** The output is right but the tone is jarring, or vice-versa.
- **Confabulated explanations.** Plausible reasoning that doesn't match the actual cause (when there even is one).

For each workflow, the design question is *which of these failure classes can my supervision actually catch?* A buyer reading a draft email catches hallucinated specifics easily — they know the supplier, they know the part number, they spot the wrongness. The same buyer might miss confabulated explanations completely, because the explanation sounds reasonable and the buyer has no independent source of truth for the *actual* reason.

Different supervision designs catch different failure classes. Match the supervision to the failures the workflow can produce, not just to the average accuracy of the model.

---

## The "what could go wrong" exercise

A practice that pays for itself: before deploying any new AI integration, sit with three or four people for an hour and run a structured premortem.

- "Imagine it's six months from now and this system has caused a problem we're embarrassed about. What happened?"
- "Imagine the worst-case wrong output. Walk through what happens after it's produced. Who sees it? When? Can they catch it?"
- "Imagine an adversarial user trying to make the system produce something we'd be ashamed of. What would they do?"

These conversations surface design issues that benchmark numbers don't. The model could be 99.9% accurate and the 0.1% could still ruin a customer relationship if the architecture lets a wrong output reach the customer unfiltered.

---

## Where next

You have a framework for deciding what to automate. The next chapter, **[Supervision and explainability](supervision-and-explainability.html)**, covers the supervision designs that the framework relies on — and the deepest trap in this whole space: the difference between an LLM-generated explanation and an actual cause.
