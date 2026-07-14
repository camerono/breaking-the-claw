# _Breaking the Claw! Breaking the Claw!_
<sub>_(that's a Judas Priest reference, btw)_</sub>
### OpenClaw + NemoClaw, Honest Impressions

*Cam. Compiled from ten days of build logs, 2026-04-30 → 2026-05-09.*


![Judas Priest What if?](api_zimage_00070_.png)


I set out to get a small autonomous monitor-and-react agent off the ground — on a DGX Spark / GB10 / ARM64 box, serving open-weight models locally, with the goal of eventually hosting dozens of agentic workflows authored by non-technical family members. I'd POC'd the usual alternatives (n8n + Ollama, LangChain, LlamaIndex) but the shiny Spark/Blackwell hardware plus the hype around OpenClaw/NemoClaw pulled me in.

The first writeup was the morning-after read on standing the stack up. What follows is ten days: the stack came up, the model lied to me, the framework turned out to be the thing lying, I swapped the whole UX layer, and by the end there was a real product monitoring real feeds. This is the honest version — what's good, what's bad, and what's ugly, updated as the picture kept changing under me.

## The pages

1. **[OpenClaw + NemoClaw — the framework read](01-openclaw-nemoclaw.md)** — the architecture is genuinely well-designed, the maintainers actually engage, and the defaults are tuned for frontier models in ways that quietly wreck local inference. Where I started.
2. **[The model question](02-the-model-question.md)** — Hermes-3 hallucinates tool calls it never made. Qwen3.6 fixed most of it, Qwen2.5:7b turned out faster *and* honest, and the MoE-is-faster hypothesis was wrong by an order of magnitude. Plus why DeepSeek-V4 doesn't fit the box.
3. **[The pivot](03-the-pivot.md)** — the framework was the source of the worst failure mode, not the model. A LangGraph spike proved it in one evening, and the multi-tenant UX problem got solved by not writing a frontend at all.
4. **[Building the product](04-the-product.md)** — retrieval quality is a three-layer problem, a cross-encoder reranker bought a 30× speedup, and the CVE warmup loop transferred almost verbatim into the real email-triage pipeline.
5. **[The cluster grows up](05-the-cluster.md)** — a two-instance prod/dev split, a fifth host, and a runtime bench that came out 3–6× off from what I expected.
6. **[Did anyone fix it? — the upstream scorecard](06-upstream-scorecard.md)** — two months later, which bugs actually moved. NemoClaw fixes things, OpenClaw labels things and stalls, and Dify's community fixes things while the merge waits on code owners.

## Net read, ten days in

**OpenClaw is closer to "production-ready for technical users" than I expected, and further from "production-ready for non-technical family members" than I'd hoped.** Every bug I filed had a clean structural fix on the table. Nothing in flight closed the gap inside a week.

**The two decisions that mattered:** move production off Hermes-3 (it fabricates tool calls with total confidence), and stop trying to hand-author skills as bare framework config. The second one is what pushed me from bare OpenClaw → a LangGraph spike → hosting the whole multi-tenant surface on Dify, wrapping my own retrieval skills behind an OpenAPI bridge. That let me keep the parts of OpenClaw/NemoClaw that were good (the observability discipline, the testing muscle) without betting the family-facing product on the parts that weren't ready.

**By day ten there's a real thing running.** A retrieval agent with a cross-encoder reranker, a CVE-advisory monitor that matches new entries against a tracked-tech list, and an email-triage pipeline that classifies and drafts replies — the actual monitor-and-react shape I was after, on local inference, on hardware I own.

**The upstream verdict, two months on, is the tiebreaker.** I filed across all three projects and let the results come in ([the scorecard](06-upstream-scorecard.md) has the receipts). NemoClaw fixed the load-bearing sandbox blocker and merged a three-in-one reliability PR — that's a project you can build on. OpenClaw's bot triages beautifully and then stalls: P1 security bugs sit `stale`, a validated filing got auto-closed, and my complaint about a bad bot-closure was closed and locked with no human reply. Dify's one bug drew a community-written fix PR that's still waiting on code-owner review. Responsiveness, not architecture, is what set the timeline I could actually plan around — and it's most of why the product routed off OpenClaw's own roadmap.

**What I'd tell someone evaluating this stack today:** file your bugs — but know that "the maintainers will engage" is true for NemoClaw and only half-true for OpenClaw, where a beautifully-labeled P1 can sit untouched for months. Don't run local inference on Hermes-3 with a tool surface. Expect to rewrite the default AGENTS.md. And treat "the agent said it did the work" as a claim to verify at the wire, not a fact — on a small local model it is the common failure case, not the rare one.

---

*Filed upstream across the run (OpenClaw, NemoClaw, and langgenius/dify): 15+ issues and evidence-comments. NemoClaw shipped several as merged fixes; most OpenClaw filings remain open or bot-closed; Dify's has a community fix PR in review. The [scorecard](06-upstream-scorecard.md) tracks each one's current state.*
