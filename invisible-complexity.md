---
title: Invisible complexity
nav_order: 9
---

# Invisible complexity

There's a Chesterton parable about reform: a man comes upon a fence in the middle of a field, finds it inconvenient, and proposes to tear it down. The reformer's correct response is *"go away and learn why the fence is there. Then come back and tell me, and we'll talk about whether to remove it."* If you don't know why the fence is there, you don't know what removing it costs.

This chapter is about why almost every AI demo of an operational system is the man at the fence. The demo shows an LLM producing a recommendation, a forecast, a schedule, a routing decision — and the recommendation looks reasonable. The implicit claim is *"see, the LLM can do what your system does."* The unspoken assumption is that what your system does is what the demo shows.

It isn't. What your system does — the part that took fifteen years to build and is the actual reason the system is valuable — is invisible in any demo. The demo shows the *output*. The value is in the *machinery that produced it*.

This is most visible in forecasting and replenishment, because those are domains where K3S and the readers of this guide have the most credibility. But the pattern travels: it applies to scheduling, pricing, routing, fraud detection, credit decisions, healthcare-coding workflows, and dozens of other places where a long-running operational system has accumulated the actual structure of the work.

---

## A demo that looks impressive

Picture a vendor pitch. The slide deck shows a screenshot of an AI replenishment recommendation. "Order 1,200 units of SKU 12345 from Supplier ACME, expected arrival in 14 days." The pitch makes the case: *the AI saw your historical demand, your current inventory, your supplier lead times, and your seasonality. It picked a quantity and a supplier. It's at least as good as your existing system and it's much cheaper to run.*

The recommendation looks reasonable. It probably *is* reasonable, on this SKU, on this day, in the demo. The buyer in the audience looks at the screen and thinks *that's what my system does. Maybe I could replace my system with this.*

What the buyer's existing system actually does — that the demo doesn't show:

- It knows that this SKU has a packaging-conversion factor: the supplier sells in cases of 24, but the warehouse stocks in eaches. Order quantities have to be in case multiples.
- It knows that ACME's listed lead time is 14 days, but on this SKU, on Tuesdays, in Q1, the actual lead time is 18 because of how ACME schedules their production runs.
- It knows that ACME has a price break at 1,000 units and another at 5,000 units, and the system has tuned itself over years to land orders just past the relevant break.
- It knows that this SKU has a seasonal pattern that intersects with the company's customer base differently from the manufacturer's listed seasonality, because the company's customers run a Q4 promotion that the manufacturer doesn't see.
- It knows that the Atlanta warehouse has a stockout-cost penalty 3× higher than the Dallas warehouse for this category, because the customer base in Atlanta is more service-sensitive.
- It knows that 18 months ago this SKU had a quality issue with one ACME lot, and the buyer who handled it added a note that gets attached to every PO recommendation: *"Q3-2024: ACME lot 4421 had bad packaging. Watch for similar."*
- It knows that the *secondary* supplier, BETA, is currently in the middle of a labor dispute, and orders to them should be quietly avoided this quarter.
- It knows that this customer's largest end-customer changed their PO frequency from weekly to bi-weekly six months ago, and the demand pattern needs to be re-segmented to reflect that.
- It knows that the company's CEO has a standing rule: orders above $50K need an extra approval step. The system handles this transparently.
- It knows that the IRS has a section-263A capitalization implication for inventory held more than 90 days, and the order-quantity logic accounts for it.
- It knows about the time the company tried to switch this SKU's supplier and it took eight months to fully unwind. The system remembers that supplier and quietly deprioritizes it.

Maybe a few hundred more pieces of accumulated knowledge — heuristics, exceptions, edge-case handling, quiet preferences, regulatory awareness, customer-specific tuning, seasonal overlays, supplier-specific quirks. None of it is in the demo. None of it is in the LLM's training data. Most of it isn't documented anywhere — it's encoded in the procedural code, the data, the parameter tables, the buyer notes, the years of small adjustments.

The LLM, given the same SKU on the same day, can produce *a* recommendation that looks similar. What it can't do is produce a recommendation that's right *given everything the system actually knows*.

---

## The visible 5%

A useful framing: a long-running operational system's output is the visible 5% of a 95% iceberg of invisible logic.

The demo shows the 5%. The vendor's pitch is implicitly that the 5% is the value. The buyer's evaluation, if they're not being careful, gets done at the 5% level — does the demo's recommendation look reasonable? It does. Sale closes. New system goes in. Six months later, on average, the new system is 7% worse on operational metrics, and nobody can quite explain why, because the 95% iceberg of accumulated logic was never visible to anyone evaluating either system.

The 95% lives in:

- **Procedural code** with branching for cases that came up over the years. The original developer had a normal flow and a special case; six developers later there are 47 special cases, each handling a real situation that the system used to get wrong.
- **Configuration tables** with per-SKU, per-supplier, per-customer overrides. The base algorithm is reasonable; the overrides are where the actual operational tuning lives.
- **Calibration constants** that have been adjusted over time. The lead-time padding is 2 days; the safety-stock multiplier is 1.15; the seasonal smoothing is 0.7. These numbers got their current values through dozens of adjustments and validations. They look like nothing in the code; they're load-bearing.
- **Exception-handling logic** that intercepts edge cases. *"If supplier X has more than three open exceptions, route to senior buyer." "If quantity exceeds N, require additional approval."* The base algorithm handles 90% of cases; this code handles the 10% that would otherwise produce bad outcomes.
- **Buyer notes and tribal knowledge** captured in fields no demo touches. *"Always check Q1 pricing on parts from this supplier — they pad it."* This is the literal text that prevents bad orders.
- **Negative space** — things the system *doesn't* do because someone learned the hard way. *"Don't auto-order this SKU; always require buyer review."* Why? Some painful incident from years ago that doesn't appear in any document but that the system's behavior reflects.

The negative space is especially important and especially invisible. A new system, trained on data, has no record of "things the old system carefully didn't do." It will do them, find out they were bad ideas, and rediscover the lesson — sometimes painfully, sometimes after irreversible consequences.

---

## Why this is especially acute for AI

Conventional system replacements have this problem too. Replacing a 20-year-old system with a 2-year-old system has always been hard, for the same reasons. The 20-year-old system encodes 20 years of accumulated learning; the new system encodes 2.

But AI replacements have a property that makes the problem worse: *the LLM is fluent enough that it can pass the demo without having any of the invisible knowledge*. A traditionally-built replacement system would visibly have empty configuration tables, no calibration history, no override rules, no buyer notes. The buyer evaluating it would see the gaps and ask "where does our SKU-specific tuning go?" — and the conversation would naturally surface the invisible 95%.

An LLM-based replacement *looks complete from the surface*. There's nothing visibly missing. The output is a confidently-formatted recommendation. The demo runs through the visible 5% smoothly. The 95% gap doesn't show up until production, until the wrong PO goes out for the SKU with the packaging conversion, until the order to BETA gets placed during the labor dispute, until the section-263A consequence emerges at year-end.

This is the crux of the AI-replacement-of-operational-systems failure mode. It's not that LLMs aren't smart enough. It's that they're *fluent without being competent at this specific work*, and the fluency hides the absence of the competence.

---

## What this implies architecturally

The implication is not "don't use AI in operational systems." That would throw out a lot of value. The implication is: *AI works well alongside the existing system, not instead of it.*

The Hybrid patterns from earlier in this guide are exactly the architectural answer. The existing system, with all its invisible knowledge, continues to make the actual decisions. The LLM contributes specific capabilities at specific points — typically translation, classification, augmentation, narration — without trying to replicate the operational logic.

For replenishment specifically:

- **The existing engine** continues to compute order quantities, supplier selection, timing. All the calibration, exception handling, override tables, and buyer notes stay where they are.
- **An LLM-Augmentation pattern** can help the buyer review exceptions: pull together relevant context, recent communications with the supplier, prior buyer notes, the actual reason for the flag.
- **A Deterministic-First LLM-Last pattern** can produce human-readable explanations of why each PO recommendation looks the way it does.
- **An LLM-Front pattern** can let the buyer ask questions about the system's recommendations in natural language ("why was BETA not chosen this week?") rather than navigating screens.
- **A Triage pattern** can route inbound supplier communications to the right buyer faster than the manual sort.

Each of these adds value. None of them replaces the engine. The 95% iceberg stays where it is, doing what it does. The LLM does the things LLMs are good at, without pretending to know things it doesn't.

---

## How to argue this with someone who disagrees

You will sometimes find yourself in a conversation with a stakeholder who's been told (by a vendor, by a board member, by a consultant) that AI can replace the engine. The conversation is hard. The stakeholder isn't an idiot; they've seen the demo, the demo looked good, the savings projection is real if it works.

Useful moves:

**Ask for a backtest.** *"Pick a representative week from last year. Run our existing system's recommendations side-by-side with the AI's recommendations. Compare to actual outcomes."* Vendors will resist; backtests aren't favorable terrain for AI replacements. If the vendor refuses, that's the answer.

**Ask about the small print.** *"What does the AI do when our supplier has a price break at 5,000 units and we'd otherwise order 4,800?"* If the answer is "it's smart, it figures it out," the vendor doesn't know about the price break. If the answer is "we'd configure it," ask how that configuration gets every other piece of accumulated logic transferred — and how long that takes.

**Ask about the negative space.** *"What does the AI do for SKUs we've decided shouldn't be auto-ordered? How will we tell it not to auto-order them, and how do we make sure that knowledge stays current?"*

**Ask about month-end.** *"What does the AI do during inventory close? When the section-263A capitalization rules apply? When we're building a customer-specific contract for next year?"* These are operational moments that don't show up in demos but do show up in production.

**Reframe the conversation.** *"What I'm hearing is the AI is impressive at producing recommendations that look reasonable. I want to be confident the recommendations are right* given everything our system has learned over the years*. Can we test the second claim, not just the first?"*

The conversation usually ends with either a path to a hybrid integration (good) or with the vendor walking away (also good — they wouldn't have been able to deliver). Occasionally the vendor produces a serious answer that does engage the invisible-95% question; that's the case where a deeper evaluation is worth doing.

---

## The corollary: don't believe your own demos

If you build AI features for operational systems, this principle applies in reverse. Your own demos look impressive. Your own demos hide the same invisible 95%, only now you're the one fooled.

The discipline to apply: every AI feature you ship should be evaluated on *what it does on the cases that already work*, not just on the cases you chose to demo. The questions are the same as for an external vendor:

- Does the AI feature respect the existing calibrations, overrides, and exception rules, or does it ignore them?
- Does it preserve the negative space — the things the system carefully doesn't do?
- Does it know about the per-SKU, per-supplier, per-customer specifics, or does it produce generic outputs that are good on average and bad on the cases that matter most?
- When operational metrics look worse three months in, will the team be able to point to *what changed* — or will the cause be diffused across a system whose invisible logic now flows through an LLM that doesn't account for it?

The teams that get this right end up shipping less impressive-sounding AI features that work better than the impressive-sounding ones. That's a feature, not a bug.

---

## Where next

The principles, patterns, and risk frameworks are now in place. The final chapter, **[What good looks like](what-good-looks-like.html)**, walks through a worked example showing all of these together in a real production system — one that respects the invisible 95%, uses the LLM where it actually fits, and builds in the supervision and reversibility the framework calls for.
