# _Breaking the Claw! Breaking the Claw!_
<sub>_(that's a Judas Priest reference, btw)_</sub>
### OpenClaw + NemoClaw, Honest Impressions

*Cam, Updated 2026-05-01.*


![Judas Priest What if?](api_zimage_00070_.png)


I've spent about 36 hours getting a small autonomous monitor-and-react agent off the ground on this stack — on a DGX Spark / GB10 / ARM64 box, with vLLM serving Hermes-3-Llama-3.1-8B locally. My use case goal is eventually to host dozens of agentic workflows authored by non-technical family members. Numerous alternatives have been considered and POC'd (N8N with Ollama, LangChain, LlamaIndex, etc.) but with this shiny Spark/Blackwell device and the hype on OpenClaw/NemoClaw, I dove in.

Here's the honest read on what's good, what's bad, and what's ugly about OpenClaw and NemoClaw.



## The Good

**OpenClaw's architecture is genuinely well-designed.** The skill model (compact registry, on-demand load), the cron subsystem (cron expressions, intervals, isolated/main session targets, error-backoff, run history with summaries), the wire-level observability (session jsonls capture every toolCall + toolResult), the schema-aware config (`openclaw config set` with dry-run + atomic writes + integrity hash) — these are the bones of a serious agent framework, not a hobby project. The 25+ registered tools cover the right primitives. The "inject a compact list of skills + let the model load on demand" pattern is exactly right for keeping prompts lean.

**NemoClaw maintainers actually engage.** I filed six bugs Wednesday night. By Thursday afternoon, one was closed COMPLETED with a shipped fix ([NemoClaw#2728](https://github.com/NVIDIA/NemoClaw/issues/2728) — onboard --resume defaulting to the wrong provider), one had concrete root-cause analysis from a maintainer ([openclaw#41304](https://github.com/openclaw/openclaw/issues/41304) closed as superseded), and one ([openclaw#22438](https://github.com/openclaw/openclaw/issues/22438) / [PR #22439](https://github.com/openclaw/openclaw/pull/22439)) got a direct "thank you, this is exactly the kind of evidence we need" reply. That's faster engagement than I expected from a project this size. clawsweeper's Codex auto-review validated all three of my new OpenClaw filings as real bugs and queued them for maintainer triage.

**The fixes ARE shipping.** [NemoClaw PR #2642](https://github.com/NVIDIA/NemoClaw/pull/2642) (auto-proxy token self-heal), [NemoClaw PR #2411](https://github.com/NVIDIA/NemoClaw/pull/2411) (multi-sandbox port allocation), the bootstrap-truncation orphaned-user-turn fix ([openclaw#42084](https://github.com/openclaw/openclaw/issues/42084) closed COMPLETED) — these are real shipped fixes for real reported bugs. The community is a learning organism, not a wishlist box.

**Wire-level evidence is everywhere.** Session jsonls, gateway logs, runs history, `systemPromptReport` metadata exposed through `openclaw agent --json` — once you know where to look, every piece of agent behavior is inspectable. That's not a small thing. It's what made it possible to distinguish "the agent dispatched a tool" from "the agent claimed to dispatch a tool." On most frameworks I've evaluated, that distinction takes external instrumentation.

## The Bad

**The defaults are tuned for cloud/frontier models, not local-inference users.** `bootstrapMaxChars=20000`, auto-generated AGENTS.md with personality content at the top and tool-use rules at the bottom, prompt totaling ~27K chars before a single user message — that's fine on Sonnet 4.6 with its 200K context, but on Hermes-3 8B it pushes the model below the threshold where tool dispatch is reliable. OpenClaw onboarding nudges users toward Hermes-3 as the local-inference choice, which leads them straight into the trap. I filed three bugs ([openclaw#75184](https://github.com/openclaw/openclaw/issues/75184), [openclaw#75187](https://github.com/openclaw/openclaw/issues/75187), [openclaw#75189](https://github.com/openclaw/openclaw/issues/75189)) on this configuration cluster.

**The supported config surface has dead flags and undocumented gotchas.** `agents.defaults.skipBootstrap: true` does nothing for workspace bootstrap files ([openclaw#75184](https://github.com/openclaw/openclaw/issues/75184)). `agents.defaults.heartbeat.lightContext` works only on heartbeats. `--session main` requires `--system-event` payload type, which doesn't reliably execute as an agent turn. `--no-deliver` on cron doesn't suppress the agent's own message-tool invocation. None of these are documented next to the schema entries that hint at them.

**`write` is overwrite-only.** The single biggest discovery from the BSG monitor build: cron firings that say "alert written" lose history because `write` calls `fs.writeFile`. The append wrapper exists in source (`wrapToolMemoryFlushAppendOnlyWrite`), gated to a memory-flush subsystem nobody can reach from cron. The workaround is `exec` with shell redirect (`>>`), which broadens policy surface unnecessarily. Filed as [openclaw#75321](https://github.com/openclaw/openclaw/issues/75321).

**v0.0.29's Dockerfile patches are shape-tied to OpenClaw 2026.4.9, but the bundled base image carries 2026.4.24.** Build fails on first install. There's no automated test for "patches still apply against the bundled base." The assignee couldn't reproduce on his environment, removed himself, and the issue is stalled ([NemoClaw#2689](https://github.com/NVIDIA/NemoClaw/issues/2689)). I have a local Dockerfile workaround running. The whole Dockerfile-monkey-patches-the-OpenClaw-source pattern is fragile by construction — every bundled OpenClaw version requires hand-tuned patch sets, and there's no obvious way to detect drift before users hit it.

## The Ugly

**The sandboxing layer is the entire reason to use NemoClaw vs bare OpenClaw — and that exact layer is broken.** [NemoClaw#719](https://github.com/NVIDIA/NemoClaw/issues/719): workspace files inside the sandbox pod are root-owned, the gateway runs as uid 998, and the gateway can't write to its own workspace directory. The fix is being actively rewritten upstream (jyaunches' refactor moves config + writable state to a PVC-backed `/sandbox/.openclaw-data/config/`), but it hasn't landed. So today: either run un-sandboxed on bare OpenClaw (production-unsafe for non-technical users), or wait. There is no third option.

**Tool dispatch hallucination is a confidence-inducing silent failure.** The first time we noticed: agent says "I searched memory and found no matches" — sounds completely correct. Gateway log: zero tool calls. Session jsonl: zero toolCall events. The model fabricated a plausible "I did the work" reply. On a frontier model this is rare; on Hermes-3 8B with default OpenClaw config it's the *common* case. For an agent that takes external actions (sending messages, modifying files), this failure mode is genuinely dangerous — the agent claims success while doing nothing. I'd argue OpenClaw should refuse to run agent-loops with tool surfaces that exceed the model's reliable dispatch ceiling, or at minimum surface a startup warning. Right now nothing protects users from it.

**The framework's smartest patterns are locked behind hardcoded gates.** `wrapToolMemoryFlushAppendOnlyWrite` already implements the right append semantics with path validation. The "compact skill registry" pattern keeps base prompts lean. Tiered bootstrap loading exists in [PR #22439](https://github.com/openclaw/openclaw/pull/22439) with `bootstrapTier: minimal | standard | full`. All of these are *almost-shipped* features that would solve real problems if they were exposed to general agent runs. They sit in the source as architectural ghosts of a more thoughtful design that lost its way somewhere between "build this for one specific subsystem" and "expose it as a general primitive."

**OpenClaw's auto-generated AGENTS.md template is anthropomorphic by default.** The literal line from SOUL.md: *"You're not a chatbot. You're becoming someone."* For a serious agent platform aimed at hosting dozens of operational workflows, the prose-poem identity content competes for attention budget with the rules that actually matter. I had to rewrite my own AGENTS.md to put the load-bearing instructions ("when user asks you to fetch X, call web_fetch with url=X") at the top. The default template optimizes for personality at the expense of behavior. Filed as [openclaw#75187](https://github.com/openclaw/openclaw/issues/75187).

## Net read

**OpenClaw is closer to "production ready for technical users" than I expected.** It's farther from "production ready for non-technical family members hosting dozens of workflows" than I'd hoped. The gap is mostly fixable — every bug I filed has a clean structural fix on the table — but nothing in flight will close it inside a week.

**NemoClaw is right on the cusp.** The sandboxing model is sound; the multi-sandbox port allocation is there; the maintainer team is responsive. [NemoClaw#719](https://github.com/NVIDIA/NemoClaw/issues/719) is the load-bearing fix. Once it lands, Phase L (port the working bare-OpenClaw stack into the sandbox) becomes ~1 day of work, not 4.

**My BSG-Wikipedia monitor is running on the bare-OpenClaw stack right now**, accumulating real history every 5 minutes, dispatching real tools, hitting a real Wikipedia API endpoint. That's enough to call the foundational use case validated — the architecture works, the components compose, and the lights stay on overnight. Everything from here is layering: more skills, the K-2 reliability/scalability test plan, the sandbox port when #719 ships, eventually a multi-tenant UX layer for family members.

**What I'd tell someone evaluating this stack today**: file your bugs (the maintainers will engage), expect to rewrite AGENTS.md, expect to use `exec` for append, and don't deploy to anyone-but-yourself until [NemoClaw#719](https://github.com/NVIDIA/NemoClaw/issues/719) ships. After that, this is a real platform — not a polished one yet, but real.

---

*Filed bugs (10 total):*

- *NVIDIA/NemoClaw:* [#2689](https://github.com/NVIDIA/NemoClaw/issues/2689), [#2701](https://github.com/NVIDIA/NemoClaw/issues/2701), [#2727](https://github.com/NVIDIA/NemoClaw/issues/2727), [#2728](https://github.com/NVIDIA/NemoClaw/issues/2728), [#2731](https://github.com/NVIDIA/NemoClaw/issues/2731), [#2733](https://github.com/NVIDIA/NemoClaw/issues/2733)
- *openclaw/openclaw:* [#75184](https://github.com/openclaw/openclaw/issues/75184), [#75187](https://github.com/openclaw/openclaw/issues/75187), [#75189](https://github.com/openclaw/openclaw/issues/75189), [#75321](https://github.com/openclaw/openclaw/issues/75321)
- *Comments-with-evidence:* [openclaw#22438](https://github.com/openclaw/openclaw/issues/22438), [openclaw#41304](https://github.com/openclaw/openclaw/issues/41304)
