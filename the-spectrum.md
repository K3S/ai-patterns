---
title: The deterministic ↔ probabilistic spectrum
nav_order: 5
---

# The deterministic ↔ probabilistic spectrum

The most common framing in current AI discourse is that there are two kinds of systems — *deterministic* (rule-based, predictable, exact) and *probabilistic* (model-based, statistical, fluid) — and they sit in opposition. Use deterministic systems for the parts that need to be right, the framing goes, and probabilistic systems for the soft, fuzzy parts. Don't mix them.

This is *almost* useful. It produces correct first-cut judgments most of the time. It's also wrong in two specific ways that matter once you start designing real systems.

This chapter develops the more nuanced view, because the hybrid architectures in the next chapter only make sense if you've stopped seeing the world as "deterministic vs probabilistic."

---

## Why the binary fails: most "deterministic" systems are probabilistic

Take a forecasting engine. It runs in RPG. It's been in production for fifteen years. Same input, same output, every time. As deterministic as code gets.

But what *is* it computing? It's producing a number — say, "we'll need 2,140 units of SKU X next month" — that is itself a *probabilistic estimate of a future quantity*. The engine's exactness is in the *computation*. The thing being computed is a probability distribution that's been collapsed to a point estimate. The "right" answer next month, when it's revealed, will almost certainly not be exactly 2,140. That doesn't mean the forecast was wrong; it means the forecast represented a distribution and a single number was extracted from it.

So when this guide says "use deterministic systems for forecasting" — which earlier drafts of this kind of advice frequently do — what it really means is: *use deterministic computation to produce the probabilistic estimate, because deterministic computation is reproducible, auditable, and consistent.* The estimate is probabilistic in nature. The engine is deterministic in implementation.

The same is true of fraud scoring, credit decisions, demand sensing, retention modeling, and dozens of other operational outputs. They produce probabilities, scores, or distributions. They do so deterministically. The deterministic / probabilistic distinction isn't a property of the *output*; it's a property of the *computation*.

---

## Why the binary fails: most "probabilistic" systems are wrapped in deterministic scaffolding

Now take a customer-service chatbot built on an LLM. The actual generation step is probabilistic. The same query produces slightly different answers across runs.

But the system around it isn't probabilistic. The user authentication is deterministic. The session storage is deterministic. The retrieval augmentation queries a deterministic database. The output gets logged deterministically. The escalation rules ("if confidence < 0.7, route to human") are deterministic. The tool-calling bindings are deterministic. The post-processing that strips PII is deterministic.

What you actually have is a deterministic system *with one probabilistic node in the middle*. Pretending the whole thing is probabilistic obscures where the variability is and what to do about it.

Same observation, viewed from the other direction: a "deterministic" replenishment engine has supplier-behavior assumptions baked into it (assumed lead times, assumed reliability, assumed pricing curves). Those assumptions are probabilistic in nature. The engine treats them as fixed because that's what's tractable, but the system as a whole has variance whose source is not the code — it's the world.

---

## A more useful frame: the variance-source map

Instead of asking "is this deterministic or probabilistic?", ask: *where does the variance live?*

For any system, there are roughly three possible answers:

1. **In the code itself** — the same input genuinely produces different outputs. This is what people call "probabilistic." LLM generation is the canonical example. So is anything stochastic by design (Monte Carlo simulations, reinforcement-learning agents).
2. **In the inputs** — the code is deterministic, but the inputs come from a noisy process: the world. Same forecasting engine, different demand patterns each month. Same risk-scoring model, different transaction streams. The output varies because the input varies.
3. **In the assumptions** — the code is deterministic *given* certain assumptions (lead times, supplier reliability, conversion rates, seasonality), but the assumptions themselves are estimates of an underlying probabilistic reality.

A system can have variance from all three sources at once. A modern replenishment system has deterministic core logic (#1: low variance), runs on noisy demand and supplier-behavior data (#2: high variance), parameterized by assumptions that are themselves estimates (#3: medium variance). The total system behavior varies a lot — but only one of the three sources is what people mean by "AI variance."

When you're designing a system, you want to:

- Push the *code-level* variance (#1) into the smallest possible places, because it's the hardest to audit. An LLM call should be a node in the graph, not the graph itself.
- Make the *input* variance (#2) explicit and measured. You can't eliminate it, but you can track it.
- Make the *assumption* variance (#3) parameterized and reviewable. Domain experts should be able to inspect the assumptions and update them.

This perspective unlocks better design choices than "deterministic vs probabilistic" does. The question stops being "should this part be deterministic or probabilistic?" and starts being "where does variance enter, and how do I bound it?"

---

## The corollary: bounded variance is the actual goal

Once you frame it as *where does variance live*, the design problem becomes *how do I bound the variance at each source?*

For LLM-generation variance (#1), the available techniques are:

- **Restrict the output space.** Force JSON. Force a label from a set. Force a citation. Constrained outputs have lower variance than free-form text by orders of magnitude.
- **Lower the temperature.** Doesn't make outputs deterministic, but reduces stylistic variance.
- **Use the same model version.** Specifying `claude-opus-4-7` is different from specifying `claude-opus-4`. Pin the version.
- **Validate the output.** A schema check, a structural check, a logical consistency check. Reject and retry on failure.
- **Cache responses for stable inputs.** If the same input is asked many times and you don't need novelty, cache the first answer.

For input variance (#2), the techniques are mostly statistical:

- **Track the distribution of inputs.** Log everything. Produce dashboards. When the distribution shifts (which it always eventually does), you should know.
- **Robust algorithms.** Trim outliers. Use medians, not means. Add safety stock. Don't bet the system's correctness on the input distribution staying stable.

For assumption variance (#3), the techniques are organizational:

- **Make assumptions explicit and parameterized.** Don't bake "lead time = 14 days" into the code. Read it from a table. Let domain experts update it.
- **Versioned assumption sets.** Last quarter's assumptions are different from this quarter's. Don't lose the ability to reproduce last quarter's results.
- **Sensitivity analysis.** Know which assumptions matter. If the system's output barely moves when an assumption swings 30%, that assumption is robust. If the output moves a lot, that assumption needs more care.

A well-designed operational system bounds variance at all three sources. A typical "AI-powered" demo bounds none of them and works only on the happy paths.

---

## Where the LLM belongs on the spectrum

The most useful position for an LLM in an operational system is *as a node within a deterministic graph*. The graph orchestrates. The graph defines the entry points, the validation, the exits, the supervision points, the audit trail. The LLM is asked one specific question at one specific point and its answer is checked before it propagates.

This isn't "AI plus a wrapper" — it's "deterministic system that uses AI for one capability." The framing matters because the design effort goes into the deterministic part, not the LLM part. The deterministic part is what makes the system reliable; the LLM is borrowed only for the specific capability where its strengths apply.

Compare two architectures:

**Architecture A (LLM-as-controller).** A user submits a request. An agent loop calls an LLM. The LLM decides what to do, calls tools, gets results, decides again. Eventually it produces a final result. The user sees it.

**Architecture B (deterministic-controller).** A user submits a request. The system validates and parses it. Based on the request type, it routes to a specific workflow. The workflow does its work, and at one or two specific points calls an LLM with a tightly-scoped question (classify this, draft this, summarize this). The LLM's answer is structurally validated and incorporated into the workflow's state. The workflow eventually produces a final result. The user sees it.

Both architectures can use the same LLM, the same retrievals, even the same prompt fragments. The difference is who's in charge of the workflow. In A, the LLM is the brain; in B, the code is the brain and the LLM is a translator. A is brittle; B is solid. The architectural choice is the most important one in the system.

{: .note }
> Agent frameworks try to make architecture A look like architecture B by adding scaffolding, retries, and tool-call validation. Some of this scaffolding helps. But none of it changes the fundamental property: in A, the LLM is in the loop on every step, and the compounding error is intrinsic. The most robust agent frameworks are increasingly the ones that make the agent "smaller" — fewer iterations, more deterministic state, the LLM doing one specific job per call.

---

## When pure-LLM systems make sense

For completeness: there are systems where the LLM-as-controller pattern is appropriate.

- **Exploration and ideation tools.** Brainstorming partners, deep-research assistants, creative collaborators. The output is suggestion, not commitment. The user is in the loop on every step and is using the LLM as a thinking partner.
- **Coding assistants and copilots.** The developer reviews every suggestion before accepting it. The blast radius of any single suggestion is one keystroke or one diff hunk. The compounding error doesn't matter because the human is the loop closer.
- **High-volume, low-stakes tasks where errors are catchable.** Email drafting, content moderation first-pass, internal-document summarization. If 5% of outputs are subtly wrong and the cost of a wrong output is small, automation is still net-positive.

Most operational software in distribution, manufacturing, healthcare, finance, and government does not look like any of these. The patterns in this guide are written for the systems that *don't* fit those exceptions.

---

## Where next

The variance-source view sets up the architectural answer: hybrid systems where deterministic code orchestrates and an LLM is borrowed for specific capabilities. **[Hybrid architectures](hybrid-architectures.html)** describes the concrete patterns.
