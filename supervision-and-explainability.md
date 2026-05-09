---
title: Supervision and explainability
nav_order: 8
---

# Supervision and explainability

Two ideas converge in this chapter, and they belong together.

The first: *humans in the loop*. The phrase is in every AI policy document, every vendor pitch, every responsible-AI white paper. It's also frequently performative — humans clicking through workflows they're not actually reviewing. Designing supervision that *actually catches errors*, rather than producing the appearance of supervision, takes architectural care.

The second: *explainability*. Almost every operational system needs to be able to answer "why?" — why was this PO approved, why was this transaction flagged, why was this customer routed to the high-risk queue. When the decision was made by deterministic code, the answer comes from the code's structured logic. When the decision was made by an LLM, the answer comes from the LLM — and there's a fundamental issue with that answer that almost everyone underestimates.

These two failure modes interact. A supervisor reviewing an LLM's output, with an LLM-generated explanation alongside, can produce *more* mistakes than the LLM alone — because the explanation makes wrong outputs feel more justified. The combination needs to be designed deliberately.

---

## The supervision spectrum

"Human in the loop" isn't a single design; it's a spectrum.

- **Pre-action review.** The human reviews each output before any action is taken. The Drafter pattern. High supervision; high cost-per-action.
- **Post-action review.** The action happens automatically; humans sample and audit later. Lower supervision; lower cost; works only when actions are reversible enough that delayed correction is acceptable.
- **Threshold-based gating.** Outputs with high confidence (or low risk by some measure) auto-execute; others go to human review. Lower supervision on the high-volume cases, higher on the unusual ones.
- **Out-of-band sampling.** Occasional human review of randomly selected outputs to detect drift. Very low supervision per action; depends on systematic detection of patterns.
- **No supervision.** The model runs unattended, consequences land, errors are caught only when something else makes them visible.

Most production systems mix these. A well-designed system has *layered* supervision — fast checks per action, slower audits per batch, periodic deep-dive reviews. Each layer catches different failure modes; together they're more robust than any single layer.

The architectural question is *which layer catches what*, and whether any failure mode falls through all of them.

---

## Why "humans in the loop" frequently fails

The most common failure mode of nominal human supervision: the human is in the loop *structurally* but not *functionally*. The form has a "Review and approve" button. People press the button. They don't actually review.

The reasons this happens are predictable:

- **Volume mismatch.** A buyer is asked to review 200 LLM-generated draft replies per day. Cognitive throughput says they cannot meaningfully review more than ~30. The other 170 get clicked through.
- **Approval bias.** The system shows the LLM's draft prominently and the alternative (write your own) as a small link. The default is to accept; rejecting is friction. Humans take the easier path.
- **Trust calibration.** The first 100 LLM drafts the buyer reviewed were good. The buyer learned the LLM is reliable. Now the buyer reviews superficially. The 101st draft is subtly wrong; the buyer misses it.
- **Time pressure.** End of day, queue full, manager wants the work cleared. Reviews get faster. Reviews become rubber stamps.
- **Confirmation bias.** The buyer expects the LLM to be approximately right. The buyer reads the draft looking for confirmation that it's approximately right. Outputs that confirm the expectation get approved; outputs that should fail the review pass anyway.

Recognizing these patterns is half the work. The other half is designing supervision that resists them.

---

## Designing supervision that actually catches errors

Tactics that move structural supervision toward functional supervision:

**Make the wrong output more obvious.** If the most common failure mode is hallucinated specifics (a wrong supplier name, a wrong part number), surface those fields prominently and visually distinct from the rest of the draft. The reviewer's eye lands on them first.

**Sample for adversarial review.** Periodically inject *known-wrong* outputs into the review queue and track whether reviewers catch them. If they don't, the supervision is decorative. The numbers will tell you when supervision degrades.

**Dual-layer for high-stakes.** For outputs with low reversibility or high blast radius, require *two* independent reviewers, neither knowing the other's verdict. Disagreements escalate. Doubles the cost; reduces single-point human failure dramatically.

**Reduce volume by raising the bar.** If 200 drafts per day produces rubber-stamping, the answer might be that not all 200 should be drafts. Raise the criteria for which inputs the LLM responds to; let the rest fall to a default (which itself might be deterministic or human-handled).

**Surface the LLM's uncertainty.** If the LLM is unsure (low log-prob, refused-to-answer, hedged in language), make that visible. Reviewers should spend more time on uncertain outputs and less on confident ones — provided the confidence is calibrated.

**Track accept rates.** If 99% of outputs are accepted unchanged, two interpretations exist: the LLM is excellent, or the supervision is decorative. Distinguish by adversarial sampling.

**Periodic recalibration.** Trust changes. The LLM gets updated; the world changes; the prompt drifts; the inputs evolve. Supervision should be reviewed and re-tuned on a cadence, not set once and forgotten.

---

## The deepest trap: explanations are not causes

Here's the property of LLMs that almost every team underestimates:

> An LLM can produce a fluent, plausible explanation for any output, regardless of whether the explanation has any bearing on the actual reason for the output.

This is a structural property, not a model-quality issue. An LLM trained to produce plausible-sounding text will produce plausible-sounding explanations on demand. The explanation's job, from the model's perspective, is to be plausible-sounding — not to be causally connected to the output it's explaining.

The implications are deep, and they affect how you design supervision.

**An LLM explaining its own output is post-hoc rationalization.** When you ask the LLM "why did you produce this?", the LLM produces a fluent answer. The fluent answer has no privileged access to the actual reasons (which are token probabilities and attention patterns, not the kind of thing the model can introspect). The answer is a *guess* at what reasons would make sense, dressed up as a confident explanation. People reading it cannot distinguish a genuine causal explanation from a confident-sounding fabrication, because both have the same surface form.

**This is different from an LLM explaining a deterministic system's output.** The Explainer pattern is the safe version: the deterministic system has *actual* reasons, the LLM is given those reasons in structured form, the LLM narrates them. The narration is a translation of real causes into prose. That's fine.

**The unsafe version is asking an LLM to explain its own classification, decision, or output.** "Why did you flag this PO as high-risk?" gets back a fluent paragraph about supplier history and quantity patterns. The paragraph might be reasonably correct (because the LLM does take those things into account in its pattern matching), or it might be entirely confabulated (because the actual reason was a stylistic similarity to training-data text the LLM has no way to surface).

In practice, you cannot tell which is happening. The fluent paragraph looks identical in both cases.

---

## What this changes about explainability requirements

Most regulatory and audit-relevant explainability requirements predate LLMs and assume a deterministic system. *"Explain how this decision was made"* meant *"show me the structured rules and the data they were applied to."* That's a tractable answer for an SQL query, a rule engine, a scoring formula.

For an LLM, that question doesn't have a good answer in the same shape. The model didn't apply rules; it produced output. There are interpretation tools (attention visualizations, SHAP values for some classifier setups, log-prob analyses) but none of them produce the kind of "here's the rule that fired" answer regulators expect.

The implications for system design:

**For regulated decisions, don't put the binding decision in an LLM.** This isn't because LLMs are bad — it's because the regulatory framework assumes a deterministic decision can be reconstructed, and an LLM decision can't be. Use the LLM in a hybrid pattern (Pattern 4: Augmentation, Pattern 2: Deterministic-First LLM-Last, Pattern 5: Side-by-Side Verification) where the actual binding decision is deterministic and the LLM contributes around it.

**For unregulated decisions where explanation is for users, not regulators, the bar is lower.** A user wants a plausible-feeling explanation. The post-hoc rationalization issue still exists — the explanation might not match the actual causal reason — but if the decision was right, the rationalization being "approximately reasonable" is acceptable for user trust.

**The audit trail needs to capture inputs and outputs, not LLM-generated explanations.** When a regulator or an auditor asks for evidence later, you can produce: the prompt, the structured input, the model output, the version of the model used. Those are reproducible (with some caveats around model version drift). You cannot produce a reliable record of "why" the model produced what it produced; that question doesn't have a reproducible answer for an LLM.

---

## Three modes of explanation

A taxonomy that helps clarity:

- **Causal explanation** — the actual reason a decision was made. For deterministic systems, the structured rules and data. For LLMs, doesn't really exist in any reliably-extractable form.
- **Communicative explanation** — a translation of a causal explanation into language a human can absorb. The Explainer pattern produces this from structured causal trace data.
- **Post-hoc rationalization** — a plausible-sounding narrative produced by asking the LLM to justify an output. May or may not match the causal reasons; cannot be verified.

The conflation of these three is the source of most explainability confusion. A team builds an LLM that produces "explanations" alongside its outputs and believes the system is now "explainable." It's not, in the regulatory sense; the explanations are post-hoc rationalizations. The system would be genuinely explainable only if a deterministic component could account for the decision and the LLM was translating the deterministic account into language.

If your design depends on the output being explainable, design for *causal explanation by deterministic means, communicated by an LLM* — not for *LLM-generated rationalization*.

---

## Designing the audit trail

For any operational system using an LLM, the audit trail should capture, per output:

- The exact prompt sent to the model (including any retrieved context).
- The exact inputs and structured data the prompt referenced.
- The model name and version used.
- The temperature and any other generation parameters.
- The exact output received from the model.
- Any deterministic post-processing applied.
- The final action taken (the output as committed to downstream systems).
- The supervision step applied (auto-accepted, human-reviewed, deterministic-validated).
- If human-reviewed: the reviewer's identity, the timestamp, the verdict.

Keeping this record makes incidents recoverable: "show me why this PO was sent" produces an answer that's reconstructable. The answer can't claim "the LLM thought X" — it can only claim "given these inputs and this model, this output emerged" — but that's the right shape of answer, and it's the shape that keeps you out of trouble.

The trail is also how you debug systematic issues. If outputs got worse last week, the trail tells you what changed: the prompt, the model version, the input distribution, the supervision step, the reviewer behavior. Without the trail, "the AI got worse" is an unsolvable mystery.

---

## A short list of supervision principles

Distilled:

- *Make supervision functional, not structural.* If reviewers aren't catching errors, the supervision is decorative.
- *Match supervision design to failure-class.* Different layers catch different failure types; combine them.
- *Distinguish causal from communicative explanation.* LLM-generated rationalization is the third thing, and it's not the same.
- *Audit by capturing inputs, outputs, and decisions.* Don't try to capture LLM "reasoning"; that record isn't reliably extractable.
- *Plan for trust drift.* Reviewers' attention degrades over time; design recalibration into the operations.
- *Use the LLM to produce communicative explanations of deterministic causes, not its own rationalizations.*

---

## Where next

The next chapter, **[Invisible complexity](invisible-complexity.html)**, applies all of this to a specific class of operational systems — forecasting and replenishment — where AI demos most visibly fail because of what the demo doesn't see.
