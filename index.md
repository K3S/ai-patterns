---
title: Home
nav_order: 1
---

# AI Patterns for Business Systems

This is a guide to thinking about AI in operational software. It is not a tutorial on calling APIs, building agents, or fine-tuning models. Those subjects age out in months. This guide is about the parts that don't.

[Get started](#start-here){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 }
[View on GitHub](https://github.com/K3S/AI-On-IBM-i){: .btn .fs-5 .mb-4 .mb-md-0 }

---

## AI as a collection of tools

The premise is simple. AI is not a hammer; it's a hardware store. Some jobs match what an LLM is genuinely good at, and an LLM doing them unattended is the right answer. Other jobs don't, and an LLM doing them produces output that looks correct and is operationally wrong — sometimes harmlessly, sometimes catastrophically. The ability to tell the difference, and to design systems that respect it, is the part that matters.

Most of what's published about AI in 2026 — model leaderboards, agent frameworks, prompt-engineering tips, demos of the same five toy use cases — is downstream of a single architectural assumption: that AI is undifferentiated, a thing you either "use" or "don't use," a checkbox on a feature list. That assumption produces bad systems. Replenishment recommendations that look reasonable and are quietly wrong by 18% on edge cases. Customer-service bots that are confidently incorrect about your return policy. Forecast explanations that have nothing to do with the actual forecast. AI integrations that work in the demo and crumble in the audit.

This guide describes a different approach. AI is a *category* of capabilities — drafting, classifying, summarizing, extracting, explaining, fuzzy-matching, translating between formal languages — and the work of integrating it well is the work of recognizing which capability matches your problem, which doesn't, and how to assemble a system that uses each thing for what it's actually good at.

---

## Who this is for

- **Developers** asked to add AI capabilities to systems they already maintain — typically systems that have been running and getting things right for fifteen years. The questions you ask are sharper than the questions a startup founder asks: *will this break my reconciliation? will my auditor accept this? what happens at month-end close?* This guide is written with you in mind.
- **Tech leadership** trying to evaluate whether the AI integration your team is proposing is sound, half-sound, or the kind of thing that wins a demo and loses an audit. The pattern catalog and risk framework here give you the vocabulary to ask the right questions.
- **People in operational domains** — distribution, manufacturing, logistics, healthcare, financial services — where the cost of "the model was confidently wrong" is measured in dollars, lawsuits, or lives. The framing here is built around that cost being real.

It is not a beginner's introduction to AI. It assumes you've used an LLM, you've seen the demos, and you're looking for architectural and judgment vocabulary you can use at work.

---

## What this guide claims

A short version, before the long one:

1. **AI is a category, not a tool.** Treating it as one thing — and either embracing or rejecting "AI" as a whole — is the underlying mistake behind both the maximalist and the dismissive camps.
2. **Capability-fit beats model selection.** Picking the right *class* of work for an LLM matters far more than picking the latest model.
3. **Probabilistic systems behave fundamentally differently from deterministic ones.** The differences cannot be hidden behind a clever prompt.
4. **Hybrid is the architecture that survives production.** Pure-LLM systems work in demos; hybrid systems work at month-end.
5. **The risk question isn't "could this be wrong?" — it's "what happens when it is?"** Reversibility, blast radius, and verifiability are the dimensions that matter, not raw accuracy.
6. **Explanations from an LLM are not causes.** A model can produce a fluent justification for any output, including outputs it produced for completely different reasons. Treating LLM-generated explanations as causal is one of the deepest traps in this space.
7. **The hardest part of an operational system is invariably invisible.** Years of accumulated edge-case handling, exception logic, and embedded heuristics live in the code that produces the obvious-looking output. AI doesn't replace that; if it pretends to, it's wrong about what the work actually is.

Each of these is a chapter.

---

## Start here

The chapters are designed to be read in order, but they're also reference-shaped — you can come back to any chapter for a specific decision and find what you need.

- **[The capability frame](capability-thinking.html)** — the central reframe. Read first.
- **[Good fits](good-fits.html)** — the named patterns where LLMs do real work.
- **[Bad fits](bad-fits.html)** — the named anti-patterns where LLMs reliably fail.
- **[The deterministic ↔ probabilistic spectrum](the-spectrum.html)** — why pure either-or is wrong, and why hybrid is the answer.
- **[Hybrid architectures](hybrid-architectures.html)** — concrete combinations: LLM-front deterministic-back, deterministic-first LLM-last, triage-and-decide, augmentation, side-by-side verification.
- **[Operational risk](operational-risk.html)** — reversibility, blast radius, verifiability. The framework.
- **[Supervision and explainability](supervision-and-explainability.html)** — humans in the loop, when to gate, what an LLM-produced explanation actually demonstrates.
- **[Invisible complexity](invisible-complexity.html)** — Chesterton's fence applied to forecasting and replenishment.
- **[What good looks like](what-good-looks-like.html)** — a worked example showing every principle in action.
- **[Reference](reference.html)** — links, further reading, acknowledgments.

---

## What this guide is not

It is not a defense of the status quo. K3S builds AI-powered systems. They run on K3S hardware against K3S databases serving K3S customers. The view here is informed by what works, what doesn't, and what's quietly broken in production a year after the press release.

It is not anti-AI. The maximalist view is wrong because it overgeneralizes; the minimalist view is wrong for the same reason. The position this guide takes is that AI does what it does, and the engineering job is to figure out *what* and *where*.

It is not specific to IBM i. The examples are drawn from the K3S domain — distribution, replenishment, ERP-adjacent workflows on top of IBM i — because that's where the credibility is. But the patterns travel. If you're integrating an LLM into a Salesforce workflow, a Java microservices architecture, or a Postgres-backed Rails app, the capability questions, the risk framework, and the hybrid patterns are the same. The IBM i shows up where it's structurally relevant — when talking about deterministic systems that have been running for decades, or when describing concrete examples of hybrid integration — and stays out of the way otherwise.

---

## Where next

Start with **[The capability frame](capability-thinking.html)**. Everything else assumes it.
