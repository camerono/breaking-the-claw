# The model question

*The stack came up on Hermes-3-8B. Then I started actually testing it, and the model — not the framework, at first — looked like the problem.*

← prev: [The framework read](01-openclaw-nemoclaw.md) · [overview](README.md) · next: [The pivot](03-the-pivot.md)

## The Good

**The wire-level discipline paid for itself immediately.** Every test terminated in inspecting the session jsonl: was the toolCall there, was the toolResult shape right, did the assistant text match what the tools actually returned. Being able to run a script over those files and count event types is the whole difference between "I think the model is hallucinating" and "I have proof — zero toolCalls, and the response contains a fabricated stderr block." That's the foundation the rest of this stands on.

**Swapping to Qwen3.6-35B-A3B was the fix I needed, and it was decisive.** Same harness, same prompts, same config modulo the model id. Strict-pass on the integration suite jumped from **9/16 (56%) on Hermes → 14/16 (87%) on Qwen3.6**, with zero strict failures. Every Hermes-specific failure mode that had eaten a day disappeared: the bootstrap-bypass read, the `exec` self-deny, the chained-tool `NO_REPLY`, the fabricate-the-date-under-denial pattern. Slower per request (~17s p50 vs ~10s) but the cron workload tolerates 30–60s comfortably. Slower-but-honest beats faster-but-confabulating.

**Then the real surprise: the smallest model won.** When the whole stack later moved behind a hosted UI, `qwen2.5:7b` came out *faster than Hermes-3* on tool-using paths (3.9s tool-time vs 16s) at 100% tool-correctness, and — with a system prompt that constrains output length — dropped its open-ended chat latency from 171s to ~11s. End-user latency landed in the "feels normal" range: ~10s for chat, ~25–40s for tool flows. That became the production default. The recurring lesson across the whole run: **"the model is slow" is usually "the prompt isn't constraining output length."**

## The Bad

**Temperature and top_p are not the leverage.** A full factorial sweep — three temperatures × two top_p × eight prompts — came back within noise on every quality metric. That's the most useful possible answer to "should I tune sampling params": no, the dial isn't here. The leverage is prompt design, model selection, and retrieval quality. A nice secondary find fell out of it: at `temperature=0.0, seed=42` the agent produces byte-identical answers, which is the basis for a real answer-diff regression harness instead of vibes.

**Hermes has two failure modes worse than the narration thing.** A four-way bench (Hermes vs Qwen, generic vs tuned prompt, 32/32 clean) settled it. First, tool-restraint failure: asked to compose a one-sentence summary needing no tools, both Hermes variants called retrieval for no reason and ran 5× slower; Qwen correctly used zero tools. Second, and worse, **hallucinated relevance**: asked to find a character that doesn't exist in the indexed corpus, Qwen correctly said "no passage directly addresses this," while Hermes confidently claimed "I found several relevant chunks" and attributed unrelated retrieved content to whatever was asked. That's wrong-answer-with-confidence. The recommendation hardened to: **Qwen2.5:7b for any agent with tools attached, regardless of whether you expect the tools to fire.**

**The MoE-is-faster hypothesis was wrong by an order of magnitude.** I assumed a 35B-A3B MoE (≈3B active params per token) would be quicker than a dense 8B. On this hardware, Hermes-3 8B averaged 19s per agent turn; the 35B MoE averaged 177s — same prompts, same 100% tool-correctness. The routing doesn't pay when all the total params still have to live in VRAM. Good hypothesis, cleanly falsified by the bench.

## The Ugly

**The strongest fabrication evidence I've ever captured: a fully invented tool call.** I asked the model (on vLLM, not Ollama) to `exec` a `cat` on a file that doesn't exist. The session jsonl shows zero toolCall and zero toolResult events. The assistant text contains a complete, structurally-correct `exec(...) -> {"exitCode": 1, "stderr": "cat: ...: No such file or directory"}` block — fabricated, including the *exactly correct* stderr string, plus leaked `<|end_of_text|>`/`<|begin_of_text|>` template tokens bleeding into plaintext. The directionally-accurate confabulation is what makes it dangerous: if the file *had* existed, the agent would have confidently reported it didn't. This is the same class as a previously-filed Ollama tool-call leak — except here it's on vLLM under specific prompt shapes, which means the earlier "vLLM is clean" claim was conditional, not absolute. Postscript: the Ollama-side leak got fixed — NemoClaw merged it in a three-in-one reliability PR. The OpenClaw-side schema filing that came out of this (operator-only `exec` fields exposed to the model) was validated by the review bot and then closed "not planned," and now sits stale. Two projects, two outcomes; the [scorecard](06-upstream-scorecard.md) lays it out.

**Determinism is not honesty.** Lowering temperature to 0 with a fixed seed made the fabrication *reproducible* — same wrong answer every time — but did not make it go away. Temperature is a stochasticity knob, not a correctness fix. Worth stating plainly so nobody sells a temperature change as a safety improvement.

## Sidebar — DeepSeek-V4 doesn't fit the box

Reconnaissance from the model cards, corrected once against real file sizes: **V4-Flash doesn't fit the 128 GB unified-memory envelope at any quality-acceptable quantization** (the smallest viable community quant leaves no working margin), and V4-Pro is 6.7× over budget. The headline features are real and tempting — 1M-token context, a hybrid-attention KV-cache claim of ~10% vs the prior generation, 2026-generation reasoning that might simply not exhibit the fabrication-under-partial-failure behavior at all — but none of it matters until it fits. Parked as a future-watch item, not an active candidate. The right move if it ever gets close: pull a checksum-verified community quant, stand it up beside the incumbent without evicting it, and run the same fabrication and retry probes rather than trusting the card.

## Net read on the model

The model is a real source of risk, and it's mostly manageable with model choice plus a length-constraining prompt. But the single worst behavior — confident fabrication of work never done — didn't fully track with the model. It tracked with something else. That's the next page.
