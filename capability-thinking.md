---
title: The capability frame
nav_order: 2
---

# The capability frame

The single most consequential idea in this guide is the simplest: AI is not a tool, it's a category of tools. The vocabulary most teams use — "should we use AI for this?", "let's add AI to that," "we're building an AI feature" — flattens that category into a single thing, and the flattening is where the bad architecture comes from.

If you can replace "AI" in a sentence with "a database" and the sentence still reads naturally, you've identified one of these flattening sentences. *"We're going to use a database for this."* For what? Storing user records? Running analytical queries? Coordinating distributed transactions? The answer matters because databases aren't all the same. SQLite, Cassandra, Neo4j, Redis, and DuckDB all answer to "database" and are appropriate for completely different problems. Anyone designing a system around "we'll use a database" without specifying which kind, for what, and where in the architecture, is doing the work badly.

The same is true of AI. The category called "AI" in 2026 contains:

- **Generative LLMs** that produce text, code, structured data, or images — what most people mean when they say "AI."
- **Embedding models** that turn text into vectors, used for semantic search and similarity.
- **Classifier and extractor models**, often LLMs prompted into structured-output mode, but sometimes purpose-built smaller models.
- **Speech and vision models** for transcription, OCR, and visual understanding.
- **Reinforcement-learning systems** for control problems and game-playing.
- **Time-series and forecasting models** that predict numerical values from historical data.
- **Constraint solvers and operations-research tools** marketed as AI but operating on completely different principles.

These are different things. They have different strengths, different failure modes, different cost profiles, different supervision requirements, and different appropriate uses. A team that talks about "the AI strategy" without distinguishing among them is going to produce architecture-by-press-release.

This guide focuses primarily on **generative LLMs and the patterns built on top of them**, because that's where most of the current confusion lives. Most of what's marketed as "AI" in business software in 2026 is an LLM, often with retrieval, sometimes with structured output, sometimes wrapped in agent frameworks. When this guide talks about "the model" or "an LLM," that's what it means.

---

## "AI is a hardware store"

The metaphor that helps most is this: AI is a hardware store, and an LLM is one product on one aisle of it. A hardware store contains hammers, drills, levels, glue, sandpaper, paint, and tape measures. They share a building, but you don't pick "the hardware store" to solve a problem — you pick the right product for the job and you ignore the rest. *"I need to use the hardware store on this"* is not a sentence anyone says.

When teams say *"we should use AI here,"* what they almost always mean is *"we should ask GPT-4 / Claude / Llama a question about this."* But the question of whether that's a good idea is wildly different depending on what *kind* of question, what *kind* of answer would be acceptable, and what *kind* of follow-up the answer would trigger.

A useful exercise: when someone proposes an AI integration, force the next sentence to be about a *capability*, not about AI. "We should use AI to draft replies to inbound supplier emails" is a capability statement: drafting. "We should use AI to flag anomalous purchase orders" is a capability statement: classification. "We should use AI to generate the actual purchase orders" is also a capability statement, but a different one: combinatorial decision-making. Forcing the capability into view immediately tells you whether you're discussing something an LLM is good at, something an LLM is bad at, or something where the answer depends.

---

## What this changes about how you build

Once you accept the capability frame, several common architectural patterns reveal themselves as mistakes:

**Picking a model first.** Saying "we're standardizing on Claude" or "we're standardizing on Gemini" before knowing what capabilities you need is like saying "we're standardizing on Craftsman" without knowing whether you're building a cabinet or wiring a house. The choice may end up reasonable, but only by accident. The right path runs the other direction: identify the capability, then pick a model and a deployment pattern that fits.

**Picking a framework first.** "We're building this on LangChain / LlamaIndex / AutoGPT / [whichever framework launched last week]" before identifying what the system needs to do is downstream of the same mistake. Frameworks bundle assumptions about which capabilities you'll need, in what order, with what wiring. Those assumptions are sometimes right, but you can't tell until you know what you're building.

**Building "an AI strategy."** Companies that produce AI strategy documents that don't decompose into specific capabilities mapped to specific business problems are producing executive theater. The useful artifact is *"here are eight specific capabilities we plan to use; here's why each fits its target problem; here's how each will be supervised; here's what failure looks like for each."* Anything more abstract is unfalsifiable, which means it's not an architectural document — it's a brand statement.

**The "agent" reflex.** Wrapping every problem in an agent — a loop that calls an LLM, interprets the output, calls a tool, calls the LLM again — is currently in fashion. Sometimes that's the right pattern. Often it's a way to disguise the fact that nobody decided what the LLM is supposed to be *good at* in this system. If a workflow could be expressed as a deterministic state machine that calls an LLM at one or two specific points, that's almost always the simpler and more reliable architecture. *Agents are state machines that have been talked into thinking they're free will.*

---

## A capability-driven design conversation

Compare two ways the same conversation can go.

**The flat version.** *"We want to add AI to our supplier portal. Customers want it. Let's get something live by Q3. Mike, can you spike on it?"*

What happens next: Mike picks the model that won the most recent benchmark, picks the framework his friend at another company uses, builds a chat interface, demos it, ships it to a few customers, and then over the next year the team discovers — slowly, painfully, in the form of small bug reports that accumulate into systemic mistrust — that the chat interface is confidently wrong about pricing, occasionally invents supplier policies, and quietly hallucinates SKU numbers in numerical contexts where it sounds authoritative. The product is technically AI-powered. It's not, in any operationally useful sense, working.

**The capability-driven version.** *"We want to add AI to our supplier portal. Before we start: which specific capabilities are we considering, on which specific user journeys, with what specific accept criteria? Let's name them and then evaluate each one separately."* The team comes back with a list. *Drafting replies to supplier email — the AI writes a draft, the human edits and sends. Classifying inbound POs by urgency. Summarizing the change history of a supplier relationship. Extracting line items from PDF invoices into structured form. Helping users find information by typing a question instead of navigating a menu.* Each gets evaluated against capability, risk, supervision, and reversibility. Three pass. Two get reshaped to be safer. One gets dropped because no LLM is going to do that job well.

That second conversation is what this guide trains you to have.

---

## What about "agentic" systems?

The current discourse treats "agents" as a separate, more advanced pattern. They're not exactly that. An agent is a loop:

1. Send the LLM a prompt and a list of available tools.
2. The LLM produces text saying which tool to call and with what arguments.
3. The system executes that tool.
4. The result is fed back into the LLM.
5. Repeat until the LLM declares the task done, or until you hit a step limit.

That's a useful pattern for some problems — exploratory data analysis, deep research, code-writing assistants where the developer wants to delegate a multi-step task. It's a *terrible* pattern for production operational systems, for one specific reason: each iteration of the loop multiplies error probability. If the LLM is right 90% of the time on any given step, a five-step loop is right 0.9⁵ = 59% of the time, and a ten-step loop is right 35% of the time. By twenty steps you're below random.

This isn't a bug to fix; it's a structural property of the architecture. Agentic systems are exploration tools. They can produce excellent suggestions and useful drafts and impressive demos. They cannot produce reliable operational decisions, because the loop's compounding error makes reliability mathematically inaccessible.

The operational pattern that *does* work is to use an LLM at one or two specific points in a deterministic workflow — never as the controller of the workflow itself. Code is the agent; the LLM is the linguist. Most of the chapters that follow describe variations on that pattern.

{: .note }
> If you find yourself defending an agentic architecture for an operational system, ask: *what is the deterministic state machine I'm trying to avoid writing?* Sometimes the answer is "the state machine is genuinely too complex to write." That's a real case, and agents can be appropriate for it. More often the answer is "we just like agents," and you'd be much better served writing the state machine and letting the LLM do specific subtasks within it.

---

## Where next

The frame is clear: AI is a category, the work is matching capability to problem. Now the catalog. **[Good fits](good-fits.html)** describes the patterns where LLMs do useful, often unattended work. **[Bad fits](bad-fits.html)** describes the anti-patterns where they reliably fail.
