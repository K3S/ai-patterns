# AI Patterns for Business Systems

A guide to *thinking about* AI in operational software, not a tutorial on calling APIs.

The premise is simple: AI is not a tool, it's a hardware store. Some jobs match what an LLM is genuinely good at — drafting, classifying, summarizing, explaining — and an LLM running unattended is not just acceptable but often the right choice. Other jobs — combinatorial optimization, deterministic reproducibility, financial reconciliation, autonomous high-stakes decisions — match what an LLM is genuinely bad at, and using one anyway produces output that *looks* right and is *operationally* wrong.

This guide is about telling the difference, designing systems that respect it, and avoiding the most common failure mode of the moment: treating AI as a single thing that either works or doesn't, rather than a category of capabilities each with its own appropriate use.

**Read it at: [aipatterns.k3s.com](https://aipatterns.k3s.com)**

---

## What's here

- **The capability frame** — why "AI is a hammer" thinking produces bad systems, and what to do instead.
- **A pattern catalog** — named patterns for the seven things LLMs do well and the seven they reliably fail at, with one-page sketches.
- **The deterministic ↔ probabilistic spectrum** — most real systems aren't pure either; the interesting question is where the seam goes.
- **Hybrid architectures** — concrete patterns for combining LLMs with deterministic engines so each does what it's good at.
- **An operational-risk framework** — reversibility, blast radius, verifiability. The framework that decides what's safe to automate.
- **Supervision and explainability** — humans in the loop, when to gate, what an LLM-generated explanation actually demonstrates (and what it doesn't).
- **Invisible complexity** — Chesterton's fence applied to forecasting and replenishment, and why the easy-looking output is the hard part.
- **What good looks like** — a worked example showing all the principles together.

The site is a Jekyll project using the [just-the-docs](https://just-the-docs.github.io/just-the-docs/) theme, served from GitHub Pages with a custom domain.

---

## Companion guides

This is one of several K3S-published guides on building modern systems on top of IBM i:

- **[IBM i AI Workers](https://ibmi-ai-workers.k3s.com)** — the integration layer. How to call models, scale them, handle failure, run a worker pool. *This guide is about what to call them for.*
- **[PHP / PDO / ODBC Toolkit Setup](https://odbcphp.k3s.com)** — running PHP on IBM i, connecting to DB2, calling RPG from PHP.
- **[IBM i Open Source Setup](https://ospm.k3s.com)** — `yum`, RPMs, SSH, the editor workflow, CCSID sanity, service users.
- **[RPG Tutorial](https://rpgtutorial.k3s.com)** — practical RPG programming through VS Code.

Each is independent, but they share a worldview: *RPG and SQL own the operationally-critical, deterministic core of the business; PHP and modern web frameworks own presentation, integration, and queries; AI is a worker pattern integrated where its capabilities actually fit; and what binds all of it together is being deliberate about which tool does which job.*

---

## Who this is for

- **IBM i developers** asked to add AI capabilities to existing systems.
- **Tech leadership** evaluating whether the AI integration their team is proposing is a good idea, a bad idea, or a good idea designed badly.
- **People in distribution, manufacturing, healthcare, and other operational-software domains** who see AI demos and wonder why the demos never seem to land in production.

It is not a beginner's introduction to AI. It assumes you've used an LLM, you've seen the demos, and you're looking for the architectural and judgment vocabulary to do something serious with one.

---

## Contributing

Issues and pull requests welcome. Use the **Edit this page on GitHub** link at the bottom of any chapter on the live site.

---

## License

- **Prose** — [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/). See `LICENSE-PROSE`.
- **Code samples** — [MIT License](https://opensource.org/licenses/MIT). See `LICENSE`.

---

*Maintained by [King III Solutions, Inc.](https://k3s.com)*
