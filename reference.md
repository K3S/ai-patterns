---
title: Reference
nav_order: 11
---

# Reference

Links, further reading, and acknowledgments worth coming back to.

---

## Companion guides

Three K3S-published guides cover related ground; together they describe a coherent architectural worldview for modern operational software on top of IBM i.

- **[IBM i AI Workers](https://ibmi-ai-workers.k3s.com)** — the integration layer. How to call models, scale them, handle failure, run a worker pool, queue work, manage rate limits. *This guide is about deciding what to call them for; that one is about how the calls happen.*
- **[PHP / PDO / ODBC Toolkit Setup](https://odbcphp.k3s.com)** — running PHP on IBM i, connecting PHP to DB2, calling RPG from PHP. The plumbing layer the AI Workers guide assumes.
- **[IBM i Open Source Setup](https://ospm.k3s.com)** — `yum`, RPMs, SSH, CCSID, service users. The foundation everything else sits on.
- **[RPG Tutorial](https://rpgtutorial.k3s.com)** — practical RPG programming through VS Code. The deterministic-engine layer.

The K3S architectural worldview, in one sentence: *RPG and SQL own the operationally-critical, deterministic core; PHP and modern web frameworks own presentation, integration, and queries; AI is a worker pattern integrated where its capabilities actually fit; the binding decisions live with humans or deterministic rules.*

---

## Reading worth doing

A short list of reading that informed this guide. None of these are AI tutorials. They're foundations.

- **G.K. Chesterton, "The Thing"** — the original fence parable. Short essay; eight pages. Worth re-reading whenever a stakeholder proposes tearing down something they don't understand.
- **Donella Meadows, *Thinking in Systems*** — the canonical introduction to thinking about systems as wholes, including the ones with feedback, accumulated state, and invisible structure. Especially useful for thinking about operational software where the visible behavior is the surface of an iceberg.
- **James Reason, *Human Error: Models and Management*** — the classic on supervision and how it fails. The "Swiss cheese" model of accident causation maps almost directly onto layered AI supervision.
- **Don Norman, *The Design of Everyday Things*** — for understanding why "humans in the loop" frequently devolves into rubber-stamping. Affordances, signifiers, and the cost of friction.
- **Nassim Taleb, *Antifragile*** — for the framework around bounded vs unbounded risk, and the deep difference between systems that suffer from variance and systems that benefit from it.
- **The original *No Free Lunch* and *Curse of Dimensionality* literature in machine learning** — for understanding why no model is universally good, and why combinatorial problems can't be brute-forced into LLMs by training harder.

There is also a vast literature in operations research on solver-based approaches to the problems that LLMs are bad at. If you're tempted to use an LLM for replenishment, scheduling, routing, or planning, spend a week reading the OR literature instead. Tools like Gurobi, CPLEX, OR-Tools, OptaPlanner, and CP-SAT have spent decades getting good at the problems LLMs are structurally unsuited for.

---

## Frequently-asked pushbacks

Counterpoints that come up often enough to address briefly:

**"But the next model will be better."** Possibly. Some of the patterns in this guide may shift as models improve. The structural ones — variance source, reversibility, blast radius, verifiability, post-hoc rationalization, invisible complexity — are not properties of model quality. They're properties of probabilistic systems and operational software. A more capable model produces fewer errors per call, but the *kind* of errors it produces and the *categories* of work it's structurally suited for don't change with scale. The same pattern catalog applies to whatever model is current.

**"You're underrating agents."** Possibly. The skepticism here isn't about whether agentic systems are useful (they are, for some problem classes); it's about whether they're appropriate for *operational* systems where the consequences of errors are real. The math of compounding error in long agent loops doesn't go away with better models. Agents are great for exploration and human-in-the-loop assistance; they're risky for unattended operational decisions. That hasn't been an empirical claim about today's models — it's been a structural claim about loops with stochastic nodes.

**"This is too cautious; it'll get K3S left behind."** The opposite case is harder to make than it sounds. Companies that ship reliable AI features that respect operational complexity end up with deployments that survive. Companies that ship impressive AI features that don't respect operational complexity end up with rollbacks, audit problems, and customer trust damage. Which path leaves you "behind" depends on the timescale; over a year or two, the careful approach almost always wins on actual deployed value.

**"Your competitors are doing more aggressive AI integrations."** Sometimes. Watching competitors closely is useful; copying them isn't. The right question is whether the competitor's AI integration is producing *operational value* — measurable, sustained, surviving an audit — or whether it's producing *demos and press releases*. The two are not the same. K3S prefers the first kind.

**"What about [latest framework / agent toolkit / AI product]?"** The architectures and patterns in this guide outlive specific frameworks. If a framework helps implement a Hybrid Pattern 4 with bounded risk and good supervision, it's a fine framework. If it pushes toward LLM-as-controller architectures with weak verification, it's the wrong fit, regardless of how popular it is this quarter.

**"Doesn't this just confirm AI is overhyped?"** No. AI is genuinely transformative for the kinds of work it's good at. The patterns in [Good fits](good-fits.html) describe real, valuable, durable applications. The argument here is for *accurate* expectations and *appropriate* deployment — not for skepticism. K3S's own production AI work, including the exception-review system in [What good looks like](what-good-looks-like.html), reflects the position that AI is valuable when applied well. The maximalist and minimalist positions are both wrong, and for the same reason: they treat AI as one thing.

---

## A short glossary

Terms used throughout the guide, defined in one place:

- **Capability frame.** The view that AI is a category of capabilities (drafting, classifying, extracting, etc.) rather than a single tool. The architectural orientation that follows: pick the capability that fits the problem, not the model that's fashionable.
- **Hybrid architecture.** A system that uses an LLM as one node in a deterministic workflow, with the LLM doing what it's good at and the deterministic system handling state, control flow, and consequential operations.
- **Reversibility.** How hard it is to undo the consequences of a wrong action. Combined with blast radius and verifiability, the framework for deciding what's safe to automate.
- **Blast radius.** How many things a single wrong action affects. From "one row" to "system-wide" to "industry-wide." Combines with reversibility and verifiability.
- **Verifiability.** How quickly and cheaply a wrong output is detectable. Determines whether automation is operationally safe — wrong outputs that take a quarter to detect are different from wrong outputs the next user immediately notices.
- **Variance source.** Where unpredictability enters a system: in the code, in the inputs, or in the assumptions. A more useful frame than "deterministic vs probabilistic."
- **Post-hoc rationalization.** A plausible-sounding explanation produced after the fact, regardless of whether it matches the actual cause. The default behavior of an LLM asked to explain its own outputs.
- **Communicative explanation.** A translation of structured causal data into prose. The Explainer pattern.
- **Invisible complexity.** The 95% of accumulated logic in a long-running operational system that doesn't appear in any demo. The reason AI replacements of operational systems frequently underperform.
- **Structural vs functional supervision.** Structural supervision means the workflow has a human-review step. Functional supervision means the human is actually catching errors. The two are often not the same.

---

## Acknowledgments

The thinking in this guide is informed by:

- **The K3S team and customers** whose AI feature requests, deployments, and post-deployment debugging produced the patterns described here.
- **Alan Seiden** and the **Seiden Group** team for ongoing dialogue about the right place for AI in IBM i operational software.
- **The IBM i community** for the constant grounding effect of asking *"yes, but how does this hold up at month-end close?"* — a question worth asking in any AI-architecture conversation.
- **The OpenAI, Anthropic, IBM Research, and academic ML communities** for producing the actual capabilities this guide depends on, and for being honest in their own publications about what those capabilities can and cannot do.
- **The operations research and constraint-programming communities**, who have been quietly solving the problems LLMs are bad at for decades and who will continue to do so.

Errors in this guide are my own. Corrections, additions, and dissenting examples are welcome via the **Edit this page on GitHub** link at the bottom of any chapter.

---

*Maintained by [King III Solutions, Inc.](https://k3s.com)*
