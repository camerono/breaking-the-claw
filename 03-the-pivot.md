# The pivot

*Two questions I didn't expect to answer this fast: is the worst failure the framework's fault, and how do I give non-technical people a UI without building one? A LangGraph spike answered the first in an evening. Dify answered the second.*

← prev: [The model question](02-the-model-question.md) · [overview](README.md) · next: [Building the product](04-the-product.md)

## The Good

**The architectural diagnosis was right, and I got to prove it.** My best guess was that OpenClaw's bootstrap-content injection — not the model — was the cause of the worst fabrication case (under an ambiguous HTTP 503, the agent replays bootstrap context and answers a *different* prompt than the one it was given). So I ran the controlled version: same model, same vLLM, same 503, on a framework with no bootstrap injection. The agent reported the 503 cleanly and fabricated nothing. **Same upstream conditions, different framework, different behavior** — the strongest causation signal you can get without a lab. That one result flipped the recommendation from "spike, don't migrate" to "switch in slices."

**The spike was half the budgeted time.** Two-day estimate; the first slice — vuln-audit, install, three smoke tests, the controlled 503 reproducer — landed in about ninety minutes. The prebuilt-agent path is genuinely productive: `create_agent` plus a handful of `@tool`-decorated functions is the entire skill, no graph definition needed for this shape. And a lot of it was fast because the testing harness discipline from the framework phase carried straight over.

**Wire-level observability transferred for ~120 lines.** A `BaseCallbackHandler` writing JSONL to disk gives `chat_model_start` / `llm_end` / `tool_start` / `tool_end` with full content capture — close enough to the OpenClaw session jsonls that every analysis script I'd built (count tool calls, scan for fabrication signals, match assistant text against tool results) worked without a rewrite. Losing observability was my biggest migration fear; it turned out to be a half-day problem.

**Failure visibility is *cleaner* off OpenClaw.** Under an outage, the alternative failed in 3 seconds (vs ~22) with a named exception and a full traceback (vs an opaque error string), and — the part that matters more than I'd credited — **rc=1 at the OS level**, which every standard cron/systemd/process monitor already understands. OpenClaw's `status=error` is visible only through its own CLI. An OS-level non-zero exit is visible to tooling a Linux operator already has.

**Then the bigger structural win: I didn't have to build a frontend.** The multi-tenant UX problem — admin authors a small catalog of skills, non-technical users consume them — got answered by hosting the whole surface on Dify. It talks to local Ollama, its plugin does tool-calling, and the load-bearing question (can it wrap my own retrieval skills without a rewrite?) came back green: I stood up a stdlib-only HTTP wrapper for the retrieval skill, served an OpenAPI spec, registered it as a Dify Custom API tool, and the agent called it correctly and synthesized a sensible answer. Every skill I want to expose is now ~30 minutes of surface on the same bridge.

## The Bad

**My first recommendation was too cautious.** "Spike, don't migrate" was the right epistemic posture — reconnaissance, not field test — but once the spike ran, the finding was strong enough that I wished I'd pushed past my own training-data confidence flags sooner. Several "medium confidence" alternatives collapsed to "high" in one evening. A sharper "test it, decide" call would have arrived at the same place faster.

**The API churn is real even past 1.0.** The spike hit a deprecation warning pointing from one agent-construction entry point to another; both work today, one is the path forward. At an early-1.x version the API is still moving. Not a dealbreaker — every framework moves — but the "post-1.0 should be stable" read needed tempering.

**The perf story is a wash on the same model.** I half-expected the leaner framework to be materially faster from less prompt overhead. On short prompts it was 2–4× faster; on the real skill round-trip it was the same order, maybe a touch slower. The real performance picture lives in the skill prompts, not framework overhead — worth saying so the migration isn't sold on a perf claim that won't survive a long run.

**Hosting on Dify has its own friction.** Its bundled Ollama plugin shipped with zero auth support — no way to put a key in front of an auth-proxied Ollama — so credential validation 401'd and the form died with an unhelpful "Failed to parse." And the whole "author a skill" ergonomic regressed: the framework version of a skill is a short markdown file plus a tiny cron payload; the ported version is one ~150-line Python file. Less total surface, more lines in one place — and for a non-technical author who'd rather hand-edit markdown than read Python, that's a real cost.

## The Ugly

**The Custom Tool path is gated behind an undocumented forward proxy, and it fought me for hours.** Dify routes all custom-tool calls through a Squid SSRF proxy that, by default, couldn't resolve the host gateway (it hadn't been given the same extra-hosts entry as the other containers) and only allows one outbound host in its ACL. The failure surfaced as a Python "reached maximum retries (0)" message that looks exactly like a connection-refused — when the real response was a 503 DNS-fail from Squid. Two hours of chasing a connection bug that was actually a proxy ACL. The fix is one line; finding it meant reading the Squid response headers I should have read first.

**And a phantom process ate an afternoon.** Real Ollama runs in a container behind the auth proxy — but a stale host install with its own service unit was hot-looping trying to bind a port the container already owned. The "address already in use" errors looked like the override I'd just added; they were the old install fighting the container. One line of `pgrep` recon up front would have caught both the phantom and the proxy at once. I didn't run it. I do now.

## Net read on the pivot

Two of the hardest questions on the roadmap — is the framework the problem, and how do family members get a UI — both got answered inside a couple of sessions, and both answers were "not the way you were planning." Keep OpenClaw/NemoClaw's good bones (the observability, the testing discipline) as the thing that taught me how to test. Move the actual product onto a leaner agent runtime, behind a UI I didn't have to write, with my own skills wrapped as OpenAPI tools. That's the shape everything after this is built on.
