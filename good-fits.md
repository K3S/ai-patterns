---
title: Good fits
nav_order: 3
---

# Good fits

Seven patterns describe most of the work where an LLM is genuinely the right tool. Each has a name, a description, the property that makes it work, examples in operational software, and the failure mode to watch for.

The pattern names are deliberately short and concrete. Use them. *"That's a Drafter pattern, not an Extractor pattern"* is a much sharper sentence than *"that's a different kind of AI use case."*

---

## The Drafter

**The pattern:** the LLM produces a draft; a human edits and decides what to do with it.

**Why it works:** drafts are starting points, not commitments. The reviewer will read every word before anything happens, so the cost of an LLM mistake is the cost of editing a sentence — bounded, immediate, and visible.

**Examples:**

- A draft reply to an inbound supplier email, presented to the buyer who clicks Send (or rewrites and clicks Send).
- A draft sales-order acknowledgment in the customer's preferred tone, presented to the rep before transmission.
- A draft commit message based on the diff, presented to the developer before push.
- A draft policy document, FAQ entry, or knowledge-base article, presented to the SME before publication.

**Property that makes it work:** the human edit-and-confirm step is *inherent*, not optional. The LLM cannot skip the human; the workflow doesn't proceed without the click. Drafts are by nature pre-decision artifacts.

**Failure mode:** automation creep. If the system is built so the human can "just click through" and the click happens fast enough, the click becomes ceremonial — the human is a stamp, not a reviewer. The fix is process design (require explicit confirmation, surface the parts most likely to be wrong, throttle the speed of the workflow) and metrics (sample reviewed outputs and check accept rates against ground truth). When 99% of drafts are clicked through unchanged, the supervision is theatre. See [Supervision and explainability](supervision-and-explainability.html).

---

## The Translator

**The pattern:** the LLM converts content from one formal language or register to another.

**Why it works:** there's typically a verifiable answer. The original meaning is preserved or it isn't. The output is checkable, either by a human reading it or by a deterministic post-check.

**Examples:**

- A natural-language question becomes SQL.
- A user's free-text request ("show me items low on stock at our Atlanta warehouse") becomes structured query parameters.
- An EDI message becomes a JSON object an API can consume.
- A code snippet in one language becomes its equivalent in another.
- A formal policy document gets translated into a plain-language explanation for end users.
- A call-center transcript becomes a structured ticket with extracted fields.

**Property that makes it work:** there's a target form with rules, the rules can be checked, and the input has enough information to determine the output. The LLM is doing the kind of work it's actually good at — recognizing patterns of how language maps to language.

**Failure mode:** ambiguity in the source. If the user's question can be reasonably interpreted three ways, the model picks one and runs. The downstream system then takes a confident action on a question the user might not have meant. Mitigations: have the model surface its interpretation alongside the answer ("interpreted as: items below safety stock at warehouse=ATL"), and design the workflow so the user can correct it cheaply.

---

## The Classifier

**The pattern:** the LLM assigns one or a small set of labels to an input from a known label space.

**Why it works:** classification is a closed-world problem with a finite set of correct answers. You can measure accuracy, build confusion matrices, set confidence thresholds, escalate ambiguous cases. It looks structurally like every machine-learning problem the field has been solving for decades — because it is one.

**Examples:**

- Inbound support email gets tagged: *billing*, *technical*, *escalation*, *spam*.
- A purchase order gets tagged: *normal*, *unusual_quantity*, *unusual_supplier*, *unusual_price*.
- A document gets categorized into a content-management taxonomy.
- A customer-service interaction gets sentiment-tagged for trend tracking.
- A code change gets risk-tagged based on diff content.

**Property that makes it work:** known label space, measurable accuracy, room to set confidence thresholds. When the model isn't confident, you don't have to make it commit; you can route to human review.

**Failure mode:** label-space drift. The list of labels you trained the prompt on doesn't match the labels that actually exist in production. The model produces plausible-but-wrong answers because it's been forced to pick from the wrong set. Mitigation: include an explicit "other / I'm not sure" option in the label space and treat its frequency as a signal.

---

## The Extractor

**The pattern:** the LLM pulls structured data from unstructured text.

**Why it works:** the target schema is fixed; the source is messy; checking the output is straightforward (does the JSON parse? do the required fields have plausible values? do the totals add up?).

**Examples:**

- Invoice line items extracted from a PDF (line description, quantity, unit price, total).
- Contact information extracted from an inbound supplier email signature.
- Product specifications extracted from a vendor's catalog page.
- Compliance-relevant clauses extracted from a contract.
- Numerical values extracted from a regulatory filing.

**Property that makes it work:** the schema is fixed, structural validation is cheap (every line item should have a quantity that's a positive integer; every contact should have a non-empty name field), and inconsistencies trigger review.

**Failure mode:** the model fills in plausible-but-fictional values to satisfy the schema. If the source doesn't contain a "vendor PO number" field but the schema requires one, the model invents a plausible-looking one. Mitigation: every required field must have an explicit "not present in source" path, and the system must distinguish that from "the model is confident the value is X." Also: deterministic cross-checks. If the line items extracted from an invoice don't sum to the invoice total, surface that as an exception, don't accept the extraction silently.

---

## The Explainer

**The pattern:** a deterministic system produced a result; the LLM narrates *why* in human-readable form.

**Why it works:** the actual decision was made by the deterministic system, which knows its own reasons. The LLM's job is communication, not analysis. The user gets a readable explanation; the system retains the audit trail of what was actually decided and why.

**Examples:**

- A replenishment engine produced a purchase-order recommendation; the LLM writes the human-readable justification ("this PO is 30% larger than the typical weekly volume because the seasonality multiplier is currently 1.4 and the safety-stock buffer was raised after the August stockout").
- A fraud-detection system flagged a transaction; the LLM produces a buyer-friendly explanation that doesn't reveal the precise rules but makes the flag legible.
- A forecasting model produced a number; the LLM explains what factors moved compared to last period.
- A query optimizer chose a plan; the LLM explains the choice to a non-DBA developer.

**Property that makes it work:** the explanation is *grounded in actual structured inputs* about the deterministic decision. The LLM isn't guessing why the system did what it did; it's reading the inputs and outputs of the actual decision and translating them.

**Failure mode:** the explanation diverges from the actual reason. This is the deepest failure mode in this entire catalog. An LLM will happily produce a fluent, plausible explanation for a decision regardless of whether the explanation has any bearing on the actual cause. If the structured inputs to the LLM are incomplete, the LLM fills in the gap with whatever sounds right — and the user, reading a confident paragraph, has no way to tell. Mitigation: never let the LLM see the raw inputs and the result alone; always feed it the *structured reasoning trace* from the deterministic system, so its narration is grounded in the actual reason. See [Supervision and explainability](supervision-and-explainability.html).

---

## The Search-and-Summarize (RAG)

**The pattern:** retrieve relevant content from a known corpus, then have the LLM synthesize an answer using that content.

**Why it works:** the LLM is constrained to the retrieved content, which means hallucinations are bounded by the quality of the retrieval. You can require citations back to source documents, which gives reviewers a path to verify the answer.

**Examples:**

- Internal knowledge base Q&A: an employee asks a question about company policy, the system retrieves the relevant policy documents, the LLM synthesizes an answer with citations.
- Documentation search: a developer asks "how do I configure X?", the system retrieves the relevant docs, the LLM produces a tailored answer.
- Contract review: a lawyer asks "what's our position on indemnification?", the system retrieves relevant clauses across contracts, the LLM summarizes patterns.
- Technical-support assistance: a customer issue arrives, the system retrieves prior similar tickets and resolutions, the LLM drafts a starting point for the agent.

**Property that makes it work:** retrieval grounds the answer in source material; citations make verification possible; answers stay closer to the corpus than to the model's training data.

**Failure mode:** the retrieval doesn't surface the right documents, the model answers anyway from training data, the answer sounds correct but is uncited or wrongly cited. Mitigation: require citations for every claim, validate that cited documents actually contain the claim, design the system to say "I don't know" rather than answer from training when retrieval is empty. Strict RAG ("answer only from the provided sources, refuse otherwise") works better in practice than soft RAG ("here's some context, answer the question").

---

## The Triager

**The pattern:** the LLM is the front door of a workflow — it categorizes, prioritizes, and routes incoming work to the right handler (human or system).

**Why it works:** triage is fundamentally classification with side effects (routing). It runs at the edge of the system, so wrong decisions are recoverable — a misrouted ticket gets re-routed, a misclassified email gets re-tagged. The LLM saves human time on the easy cases; the harder cases get to the right person faster.

**Examples:**

- Inbound customer support: the LLM tags by topic, severity, and urgency; routes the easy cases to FAQ responses and the hard cases to specialists.
- Procurement exception handling: the LLM categorizes incoming exceptions and routes to the appropriate buyer based on supplier, category, and dollar amount.
- Inbound EDI from suppliers: the LLM classifies and routes — invoices to AP, advance ship notices to the receiving system, exception messages to the buyer.
- Resume screening: the LLM tags applications by role-fit and seniority; routes to recruiter queues. (This one has its own ethics conversation; see also [Operational risk](operational-risk.html).)

**Property that makes it work:** wrong decisions are recoverable, the volume justifies automation, the categories are stable, and the system tracks accuracy over time so drift is detectable.

**Failure mode:** the categories drift but the prompt doesn't. The world adds new types of work, the LLM doesn't have a label for them, it forces them into existing buckets. Mitigation: include "uncategorizable / route to human" as a first-class option, monitor its frequency, treat increases as a signal to update the system.

---

## What these patterns have in common

If you read across all seven, several properties recur:

- **Bounded output space.** Drafts go to a human. Classifications come from a known set. Extractions fit a schema. Explanations narrate already-made decisions. The LLM isn't being asked to produce arbitrary new content; it's being asked to produce content that fits an existing slot.
- **Cheap verification.** Most of these patterns have a fast, low-cost way to check whether the output is right. A human reads the draft. The classification gets compared to actual behavior. The extracted data is structurally validated. The cited document is checked against the source.
- **Linguistic, not numerical.** All of these are about language: drafting language, translating language, classifying language, extracting from language, explaining in language. None of them ask the LLM to do exact arithmetic, optimize a constraint problem, or produce reproducible deterministic answers.
- **Forgiving environments.** A misclassified ticket gets re-classified. A bad draft gets edited. A wrong extraction gets corrected before it becomes truth. The patterns are built to tolerate the LLM's nature.

The next chapter, **[Bad fits](bad-fits.html)**, describes the inverse: patterns where one or more of these properties is violated, and the LLM fails reliably.

---

## Where next

You have a working catalog of good fits. The pair-piece is **[Bad fits](bad-fits.html)**, which describes anti-patterns: combinations where an LLM looks plausible and reliably underperforms.
