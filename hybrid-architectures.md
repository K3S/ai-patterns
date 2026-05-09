---
title: Hybrid architectures
nav_order: 6
---

# Hybrid architectures

This is the chapter that turns the principles into building blocks. The pattern catalog in the [Good fits](good-fits.html) chapter described capabilities — what the LLM does well in isolation. The chapter on [the variance spectrum](the-spectrum.html) described why deterministic-vs-probabilistic is the wrong framing. This chapter combines both: the named architectural patterns for using an LLM *as a node inside a deterministic system*, with the seam between them designed deliberately.

Five patterns cover most operational uses.

---

## Pattern 1: LLM-Front, Deterministic-Back

**The shape:** the user interface accepts natural language; an LLM translates it into structured input for a deterministic engine; the deterministic engine does the actual work; the result is returned, optionally with an LLM-generated narration.

**When it fits:** any time the existing system has a powerful deterministic capability that's hard to discover or operate through traditional UIs (forms, menus, query builders), and the users would benefit from a more flexible front door.

**Concrete example.** A buyer wants to know which items are below safety stock at a specific warehouse. The traditional UI requires opening a screen, picking the warehouse from a dropdown, picking "stock status" from a menu, sorting by some field. The natural-language front door lets the buyer type "show me items below safety stock in Atlanta." The LLM converts that to:

```json
{
  "intent": "list_items_below_safety_stock",
  "filters": {"warehouse": "ATL"}
}
```

A deterministic handler validates the intent, runs the actual SQL or RPG procedure, gets back a list, and returns it. Optionally, an LLM call produces a narration: "12 items are below safety stock at Atlanta. The largest gap is on SKU 12345 (current 24, safety 80, lead time 21 days)."

**Where the seam lives.** The seam is the structured intent object. On the user-facing side, the LLM does whatever it needs to in order to produce a valid intent. On the system side, the deterministic code only sees structured intents from a known schema and executes them by traditional means. The LLM never touches the database directly.

**Why it works.** The deterministic engine is exactly what it always was — auditable, reproducible, fast. The LLM is doing one of its strongest patterns (translation) on a constrained schema. If the LLM produces a malformed intent, validation catches it. If it produces a valid intent that doesn't match what the user meant, the user sees the wrong list and can correct ("not Atlanta, Atlas").

**Things to be careful about.** The LLM's interpretation of the user's question should be visible to the user before any consequential action. *"Interpreted as: items below safety stock at warehouse=ATL"* is a one-line addition that turns silent misinterpretation into a correctable mistake. Don't hide the seam.

---

## Pattern 2: Deterministic-First, LLM-Last

**The shape:** the deterministic engine produces the actual decision (recommendation, score, plan, output). The LLM, given the structured inputs and the deterministic output, produces a human-readable explanation or summary.

**When it fits:** any time the decision logic is non-negotiable (regulatory, financial, operational) but the *communication* of the decision benefits from natural-language fluency.

**Concrete example.** A replenishment engine produces a purchase-order recommendation: SKU, quantity, supplier, expected arrival. The buyer needs to understand *why* — what factors drove the quantity, why this supplier was chosen, how it compares to last week's pattern. Producing the explanation manually requires either a complex generated report (rigid, hard to maintain) or expensive analyst time. The LLM is given:

- The PO details.
- The historical baseline ("usually we order ~1,200/week of this SKU").
- The current contributing factors (forecast multiplier, current inventory level, recent stockout, lead-time signal).
- The decision rule that fired.

…and produces a paragraph: *"This PO is for 1,800 units, 50% above the recent four-week average of 1,200. The increase is driven by a seasonality multiplier (currently 1.4) and an elevated forecast following the August stockout. Supplier ACME was selected because their lead time is currently 8 days, vs. 21 days for the alternate."*

**Where the seam lives.** The LLM never alters the recommendation. It only narrates. Critical: the LLM gets *structured reasoning trace data*, not just inputs and output. If the deterministic system says "I picked supplier ACME because alternate supplier BETA is currently on hold," the LLM should be told that explicitly, not asked to infer it. Otherwise the LLM will invent a plausible-but-fake reason.

**Why it works.** The actual decision is reproducible, auditable, and made by a system you can prove correct. The LLM is doing the Explainer pattern, which it's strong at. Users get a result they can understand without sacrificing the operational integrity of the underlying decision.

**Things to be careful about.** This is the pattern most prone to the *post-hoc rationalization* failure described in [Supervision and explainability](supervision-and-explainability.html). If the LLM is fed only the inputs and output and asked "explain this decision," it will produce a plausible explanation that has nothing to do with the actual reason. Always feed it the *reasoning trace*, and the trace must come from the deterministic system, not from the LLM's own analysis.

---

## Pattern 3: LLM-Triage, Deterministic-Decide

**The shape:** incoming work arrives at the system; the LLM classifies and routes; deterministic handlers do the actual work for each category.

**When it fits:** the inbound flow is heterogeneous (lots of different types of work mixed together), the categorization can be done from text content alone, and each category has a well-understood deterministic handler.

**Concrete example.** Inbound supplier email arrives in a shared inbox. Some are invoices. Some are advance ship notices. Some are price-change announcements. Some are exception messages requiring buyer attention. The LLM classifies each one: `invoice | asn | price_change | exception | other`. The deterministic system routes each to the right handler — invoices to AP processing, ASNs to the receiving system, price changes to the catalog update queue, exceptions to the buyer's worklist.

**Where the seam lives.** The seam is the classification label and a confidence score. Below a threshold, the email goes to a human review queue, not to an automated handler. Misclassifications are caught when humans review the destination handler's output and can re-tag.

**Why it works.** Classification is one of the LLM's strongest patterns, especially with a known label space. Wrong classifications are recoverable (a misrouted email gets re-routed; a queue tagged "exception" can be reviewed by a human). The deterministic handlers do their work as they always have.

**Things to be careful about.** Track the distribution of classifications over time. If the share of "other" or "low-confidence" cases is growing, the world has changed and the prompt hasn't. Treat that as an architectural signal, not just an inconvenience.

---

## Pattern 4: LLM-Augmentation

**The shape:** at a decision point, the LLM provides relevant context that helps a human decide. The decision still belongs to the human; the LLM just makes the decision easier.

**When it fits:** the decision is genuinely judgment-dependent (the right answer requires expertise the system doesn't have), the human would benefit from broader context but doesn't have time to assemble it manually, and reducing the human's cognitive load improves quality.

**Concrete example.** A buyer reviews a flagged purchase order. Why is it flagged? The deterministic system says "price is 23% higher than the last six months' average" — a specific signal. The LLM is given the buyer's question implicitly ("should I approve this PO?") and the structured context: PO details, supplier history, recent communications with the supplier (from the email system), notes from prior buyers about this part. The LLM produces a brief situational summary: *"This part has been on a stable price for two years. The recent supplier email mentioned a steel surcharge starting this quarter. The previous buyer's note from 18 months ago says 'always check ACME pricing in Q1, they pad it.' Recommend: ask supplier to itemize the increase."*

The buyer reads, decides, acts. The LLM's contribution was *making the relevant context legible*, not making the decision.

**Where the seam lives.** The LLM has access to context the human couldn't reasonably assemble in real time, but it never executes. It produces a *briefing*, the human acts.

**Why it works.** The buyer's expertise is preserved (the call is theirs), the LLM contributes what it's good at (reading and summarizing across heterogeneous text sources), the system is faster than purely manual review and more reliable than purely automated review.

**Things to be careful about.** The LLM's briefing can subtly bias the human's decision. If the LLM says "I recommend rejecting" with confident framing, the human is more likely to reject — even if the recommendation is wrong. Calibrate the language. Frame as context ("here are the relevant facts"), not recommendation ("you should reject"), unless the human has specifically asked for one.

---

## Pattern 5: Side-by-Side Verification

**The shape:** the LLM proposes an action; a deterministic check evaluates whether the action is acceptable; the action only executes if both agree.

**When it fits:** moderate-stakes automation where the LLM's pattern-matching is useful but the consequences of a wrong action are high enough that pure LLM-driven execution is unacceptable.

**Concrete example.** An exception-handling workflow processes inbound POs that didn't auto-match against expected receipts. The LLM proposes a resolution for each one: *"This PO matches receipt #12345 from yesterday — quantity differs by 2 units, supplier reference matches, item code matches. Recommend matching."* A deterministic check then evaluates the proposal against constraints: same supplier, same item, quantity within tolerance, dates plausible, totals consistent. If all checks pass, the match is recorded. If any check fails, the case escalates to a human reviewer with both the LLM proposal and the failed check.

**Where the seam lives.** The deterministic check is the gate. The LLM is a *proposer*, not a *committer*. The deterministic check has full veto, no override.

**Why it works.** The LLM contributes its pattern-matching strength on heterogeneous, messy inputs (where deterministic alone might miss valid matches because the structure varies); the deterministic check contributes its rigor (catches LLM hallucinations and confident-but-wrong proposals). Together they're more accurate than either alone.

**Things to be careful about.** The deterministic check has to be more conservative than the LLM, otherwise the gate is decorative. If the deterministic check passes 99% of LLM proposals, you're not gated; you're just delayed. The check should rule out the *kinds of failures the LLM is most likely to make* — confidently mismatched IDs, hallucinated values, plausible-but-wrong matches.

---

## Choosing among the patterns

A rough decision tree:

- **The system needs a more flexible front door for an existing engine?** → LLM-Front Deterministic-Back.
- **The system already produces correct answers; users need them explained?** → Deterministic-First LLM-Last.
- **Heterogeneous inbound work needs categorization for downstream handlers?** → LLM-Triage Deterministic-Decide.
- **Humans are the right deciders but need help marshaling context?** → LLM-Augmentation.
- **Pattern-matching at scale, with stakes high enough that errors must be caught?** → Side-by-Side Verification.

Most non-trivial systems use multiple patterns. A modern operational workflow might have an LLM-Front front door (Pattern 1), do triage on inbound work (Pattern 3), augment a buyer's review of exceptions (Pattern 4), and provide LLM-narrated explanations of automated decisions (Pattern 2). All five patterns can coexist; they're not exclusive.

---

## What none of these patterns do

A worth-naming property of all five patterns: in none of them does the LLM execute consequential operations directly. Across all five:

- The LLM never writes to the database without a deterministic check between proposal and write.
- The LLM never sends external communications without human or deterministic gating.
- The LLM never makes the final decision in any high-stakes workflow.
- The LLM is always inside a deterministic envelope.

That's not because LLMs are forbidden from this work. It's because none of these patterns *needs* to delegate execution to the LLM. The LLM is borrowed for what it's good at — translation, classification, narration, summarization, augmentation — and the deterministic system handles what it's good at: state, audit, atomicity, exact arithmetic, well-defined operations.

This is the architectural answer to the question this guide has been building toward: *what does it look like to use AI well in operational software?* It looks like deterministic systems that have absorbed an LLM as one capability among many — not LLMs that are pretending to be operational systems.

---

## Where next

You have a vocabulary of patterns. Now the question of *which patterns are safe to automate* and *which need supervision*. **[Operational risk](operational-risk.html)** introduces the framework — reversibility, blast radius, verifiability — for that decision.
