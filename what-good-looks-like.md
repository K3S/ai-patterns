---
title: What good looks like
nav_order: 10
---

# What good looks like

The principles in the previous chapters are abstract. This chapter is a single worked example showing them all together: a production AI integration in a long-running operational system, designed deliberately around capability-fit, hybrid architecture, bounded risk, real supervision, and respect for invisible complexity.

The example is the **AI-assisted pricing exception review** in K3S's replenishment software. Pricing exceptions are flagged purchase-order line items where the price differs from the expected price — by tolerance, against contract, against history, or against current market signal — and require buyer review before the PO is finalized. K3S has had a deterministic exception-detection engine for years. The AI integration helps buyers process the resulting exception queue.

The pattern is described at a level useful for teaching. The specific thresholds, prompts, and tuning constants are K3S internal. The architectural shape is what matters and travels.

---

## What the system does

Each morning, before buyers arrive, the deterministic replenishment engine scans the day's pending POs. For each line, it compares the proposed price against:

- The contracted price for that supplier and item.
- The historical price band over the last 12 months.
- Current market-signal data where available.
- Tolerance bands tuned per category, per supplier, per customer arrangement.

Lines that fall outside acceptable tolerance get flagged as exceptions and routed to the appropriate buyer's review queue. The deterministic logic does the *flagging*. It does not, on its own, do anything more — the buyer reviews each exception and decides: approve as-is, ask the supplier to reprice, escalate to a senior buyer, or reject.

The pre-AI version of this workflow gave the buyer a list of flagged lines and a few structured fields: SKU, supplier, expected price, actual price, deviation amount. The buyer manually pieced together the rest of what they needed: prior PO history, recent communications with the supplier, any notes from previous reviews, knowledge about market conditions. This took 5–15 minutes per exception. A buyer might process 30–60 exceptions in a morning. Most of the buyer's time was spent assembling context, not deciding.

The AI integration changes that. For each exception, the system now also produces a concise **buyer briefing**: a short paragraph that summarizes the relevant context the buyer would otherwise have to assemble manually. The briefing might say:

*"This SKU's contracted price with Supplier X is $4.12, last validated 6 months ago. The proposed price of $4.78 is 16% above contract, exceeding the 10% tolerance. Historical pattern: prices from this supplier on this category have been creeping up over the last quarter, with a similar 12–15% deviation seen on three other SKUs. Recent communication thread (Aug 22) referenced an upcoming steel surcharge. Prior buyer note from Jan 2024: 'always verify Q1 pricing on items from Supplier X — they pad it.' Recommended action: ask supplier to itemize the increase before approving."*

The buyer reads the briefing, decides, acts. Exception handling time drops from 5–15 minutes to roughly 1–3 minutes. The decisions are demonstrably better, because the relevant context is more reliably surfaced — the buyer doesn't have to remember to check the comms history or the prior notes; the system does it.

That's the user-visible feature. The architecture underneath is what makes it work.

---

## The architecture

A simplified view of the data flow:

1. **The deterministic exception engine** (RPG) scans pending POs, flags lines that exceed tolerance, writes to an exceptions table.
2. **A worker pool** (PHP, with parallel HTTP via Guzzle) picks up new exceptions in batches.
3. **For each exception**, the worker assembles a *structured input package*: the exception details, prior PO history for this SKU and supplier, the contracted-price record, recent communications referencing the supplier, prior buyer notes, the deterministic engine's flagging reason.
4. **The structured package is sent to an LLM** with a tightly-scoped prompt: produce a brief buyer briefing using the provided context, in a specific structured format, without inventing details.
5. **The LLM's response goes through validation**: it must conform to the expected structure, reference at least one piece of provided context, contain no values not present in the input. Failed validation triggers retry; persistent failure logs and routes to a human reviewer with the raw exception (no AI briefing).
6. **The validated briefing is stored** in a results table alongside the exception ID.
7. **The buyer's review screen** shows the exception, the deterministic flagging reason, and the AI-generated briefing — clearly labeled as such.
8. **The buyer decides and acts**. Every action is logged: timestamp, buyer identity, decision, any free-text note. The audit trail captures the prompt sent, the model output received, the validation outcome, the final decision.

This is the kind of architecture that fits in a single diagram and fits in a single paragraph. The simplicity is intentional. It's not a multi-step agent loop. It's a deterministic workflow that calls an LLM at one well-defined point.

---

## How each principle maps

**Capability frame.** The LLM is asked to do exactly one thing: read structured context and write a brief, factually-grounded summary. That's the Search-and-Summarize pattern combined with the Explainer pattern. Both are within the category of work LLMs are reliably good at. The LLM is not deciding, executing, classifying, or scoring — those are jobs other components do.

**Good-fit pattern.** Specifically, this is an **LLM-Augmentation** integration (Pattern 4 from [Hybrid architectures](hybrid-architectures.html)). The buyer is the decider; the LLM provides context. There are also elements of **Deterministic-First LLM-Last** (Pattern 2): the deterministic engine made the flagging decision; the LLM narrates the surrounding context.

**Bad-fits avoided.** The system is deliberately *not* the Sole Authority (buyer always decides), *not* the Optimizer (the deterministic engine flags exceptions; the LLM never picks prices), *not* the Reconciler (no LLM arithmetic), *not* the Reproducer (we don't need bit-exact reproducibility because the buyer reviews each), *not* the Source of Truth (the LLM never answers a question without grounding in retrieved context).

**Variance bounding.** The LLM call has bounded code-level variance: temperature is low, the output schema is constrained, the prompt requires the model to refuse rather than fabricate when context is insufficient. Input variance is bounded by the deterministic system that prepares the structured input package — the LLM doesn't see free-form data, it sees a curated package. Assumption variance lives in the deterministic engine's tolerance tables, where it's parameterized and reviewable.

**Reversibility.** Every action the buyer takes after reviewing the briefing is itself reversible: an "ask supplier to reprice" is a message that can be followed up; an "escalate" puts the exception in a different queue; an "approve" creates a PO that can still be canceled (depending on supplier). The LLM never commits anything; it produces text the buyer reads.

**Blast radius.** One exception at a time. One PO line at a time. A wrong briefing on one exception affects, at most, the buyer's review of that exception — which is then either correctly resolved (the buyer noticed the briefing was wrong) or incorrectly resolved (one PO line affected). Blast radius is structurally one row.

**Verifiability.** Fast and human-mediated. The buyer reads the briefing and is in a position to notice if it's wrong — the buyer knows the supplier, knows the part, can check the cited facts against their own knowledge or the system's records. The verification feedback loop closes within seconds of the briefing's production.

**Supervision.** Functional, not just structural. The buyer cannot proceed without an explicit decision; the workflow doesn't auto-advance. A weekly metric tracks the rate at which buyers' decisions diverge from what the briefing seemed to recommend — that rate, monitored over time, surfaces any AI drift toward bad recommendations. Outputs are sampled by senior buyers periodically as a calibration check.

**Explainability.** The briefing the LLM produces is *communicative* explanation grounded in structured causal data from the deterministic engine. The deterministic engine knows why it flagged the line ("price exceeds 10% tolerance against contract"). The LLM translates the deterministic reason plus the surrounding context into prose. The audit trail captures what the LLM was given and what it produced; the *causal* explanation of any binding decision still lives in the buyer's logged action.

**Invisible complexity.** None of the pre-existing replenishment logic was replaced. The exception-detection engine, with its years of tolerance tuning, supplier-specific overrides, category-specific rules, and calibration constants, runs unchanged. The LLM was added *around* the existing system, not *in place of* it. When the exception engine produces a flag, the engine's reasons are exactly as auditable as they were before. The LLM contributes a new piece of value — buyer briefing — without touching the operational logic that determines what gets flagged in the first place.

---

## What the system deliberately does not do

A list of things this system was *asked* to do during scoping and explicitly *did not*:

- **Auto-approve any exception**, no matter how confident the LLM. The buyer always decides. The cost-benefit analysis showed that the marginal value of auto-approving low-risk exceptions wasn't worth the compounding risk of trust-drift in the human review.
- **Decide which lines should be flagged.** The exception detection is deterministic, runs the same way it always has, and uses the same tuning the buyers and analysts have refined over years. The LLM was never given the job of "find anomalies"; that's not what it would do well, and replacing the engine wasn't an option.
- **Negotiate with suppliers.** The system doesn't draft messages back to suppliers automatically. It can produce a draft if the buyer asks (Drafter pattern), but transmission requires the buyer's explicit click.
- **Train on the buyers' decisions to "learn" what to recommend.** The integration doesn't do this. Adding it would create a feedback loop where the LLM optimizes against the buyers' recent patterns, including their mistakes; would erode the deterministic engine's role as source of truth on the flagging logic; and would create a model whose behavior drifts unpredictably with operational consequences. The buyers' decisions are logged for analytics; they don't fine-tune the model.
- **Make the briefing fancier.** Early drafts had richer formatting, more sub-sections, prediction confidence scores, and graphical indicators. The buyers asked for less, not more. The current briefing is a paragraph or two, plain text. It's read in 30 seconds. That's the right product.
- **Replace the existing UI.** The buyer's review screen looks essentially the same as it did before. The briefing is a new field. The rest of the workflow is unchanged. Buyers' muscle memory is preserved.

These are all good-faith feature requests that got declined because they conflicted with the principles in this guide. Saying no — and being able to articulate *why* — is part of what good architecture looks like.

---

## What was learned in production

Some operational findings from running this in production:

**The structured input package is more important than the prompt.** When briefings were wrong, the most common cause was that the deterministic system hadn't surfaced the right context — a missing supplier note, a stale contract record, a missed comm thread. Improving briefings was almost always a matter of improving the data packaged into the LLM call, not improving the prompt or the model. The LLM is, in effect, a stage of a pipeline; the upstream stages do most of the work.

**Buyers calibrate quickly.** Within a few weeks of the rollout, buyers had developed a sense of when to trust the briefing more or less, even though no one had told them how to. They picked up cues from briefing structure, length, the way historical context was integrated. This calibration is human, distributed, and durable — it doesn't require retraining or system updates. But it also depends on the briefings being reliably honest about uncertainty; a system that confidently asserts wrong things degrades the calibration.

**The metric to watch is buyer-outcome quality, not briefing accuracy.** "Was the briefing accurate?" is the natural metric, but it's not the right one. The right metric is: are the exception decisions getting better — fewer downstream issues, fewer reversals, fewer supplier disputes? An accurate briefing that doesn't change the buyer's behavior contributes nothing. An imperfect briefing that consistently helps the buyer make better decisions is the goal.

**Drift detection is mostly mechanical.** Routine sampling of LLM outputs, comparison against the structured input, validation against the buyer's logged decision: these catch drift when it happens. Manual review of the briefing language is much less useful than checking that the briefing's claims are present in the input package.

**The hybrid architecture survived an LLM provider change with negligible impact.** When a different model was evaluated and adopted for the production deployment, the rest of the system didn't change. The structured input format, the validation, the buyer UI, the deterministic engine — all unchanged. The "AI part" was contained in a single replaceable component. This is a property of the architecture, and it's the difference between an integration that costs you a quarter to migrate and one that costs you a weekend.

---

## What this example demonstrates

If you want a single sentence: *the LLM is one node in a deterministic graph, doing exactly one thing it's good at, in a workflow where the consequences of error are bounded by design.*

That's what good looks like. It's not glamorous. It doesn't make for an impressive demo, because the demo would have to show the existing system operating exactly as it always has, with a small new field on the buyer's review screen. The *integration* is small; the *value* is large; the *risk* is bounded; the *machinery underneath* is mostly the same machinery that was there before.

This is the shape of AI integration that lives, scales, survives audits, and is still in production three years later. It's also the shape that, when described to senior business stakeholders or to engineers familiar with the domain, generates the response: *"oh, that's just sensible — why aren't more people doing it that way?"*

The answer to that question is most of what this guide has been about. People aren't doing it that way because the dominant frames in AI discourse — "AI is a thing," "agents are the future," "the next model is the answer" — push toward different architectures. The architectures pushed by those frames mostly don't survive contact with operational reality. The architecture in this chapter does, and the principles that produced it are the durable thing.

---

## Where next

This is the last conceptual chapter. The final chapter, **[Reference](reference.html)**, collects the further reading, links to companion guides, and acknowledgments worth coming back to.
