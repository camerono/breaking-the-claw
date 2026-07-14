# OpenClaw + NemoClaw — the framework read

*Where I started. The honest read on the framework itself, before the model and framework questions pulled the project somewhere else.*

← [back to the overview](README.md) · next: [The model question](02-the-model-question.md)

## The Good

**The architecture is genuinely well-designed.** The skill model (compact registry, load-on-demand), the cron subsystem (cron expressions, intervals, isolated/main session targets, error-backoff, run history with summaries), the wire-level observability (session jsonls that capture every toolCall and toolResult), the schema-aware config with dry-run, atomic writes and an integrity hash — these are the bones of a serious agent framework, not a hobby project. The 25+ registered tools cover the right primitives. "Inject a compact list of skills, let the model load them on demand" is exactly the right pattern for keeping prompts lean.

**The maintainers actually engage — at first.** I filed six bugs one night. By the next afternoon one was closed COMPLETED with a shipped fix (NemoClaw#2728 — onboard `--resume` defaulting to the wrong provider), one had concrete root-cause from a maintainer, and one got a direct "this is exactly the kind of evidence we need." The auto-review bot validated three fresh filings as real bugs and queued them for triage. That's faster engagement than I expected from a project this size. (Two months later that first impression had split hard between the two projects — NemoClaw kept shipping fixes, OpenClaw's queue stalled. The [scorecard](06-upstream-scorecard.md) is the honest follow-up.)

**The fixes ARE shipping.** NemoClaw PR#2642 (auto-proxy token self-heal), PR#2411 (multi-sandbox port allocation), the bootstrap-truncation orphaned-user-turn fix (openclaw#42084, closed COMPLETED) — real shipped fixes for real reported bugs. The community behaves like a learning organism, not a wishlist box.

**Wire-level evidence is everywhere.** Session jsonls, gateway logs, run history, `systemPromptReport` metadata through `openclaw agent --json` — once you know where to look, every piece of agent behavior is inspectable. That is not a small thing. It is what made it possible to distinguish "the agent dispatched a tool" from "the agent *claimed* to dispatch a tool." On most frameworks I've evaluated, that distinction takes external instrumentation. Here it's on disk by default, and it became the workhorse of everything that followed.

## The Bad

**The defaults are tuned for cloud/frontier models, not local inference.** `bootstrapMaxChars=20000`, an auto-generated AGENTS.md with personality content at the top and tool-use rules at the bottom, a prompt totaling ~27K chars before the first user message — fine on a 200K-context frontier model, fatal on a local 8B, where it pushes the model below the threshold where tool dispatch stays reliable. Onboarding nudges local users toward Hermes-3, which walks them straight into the trap. I filed three bugs on that configuration cluster (openclaw#75184, #75187, #75189).

**The config surface has dead flags and undocumented gotchas.** `agents.defaults.skipBootstrap: true` does nothing for workspace bootstrap files. `heartbeat.lightContext` works only on heartbeats. `--session main` requires a `--system-event` payload that doesn't reliably execute as an agent turn. `--no-deliver` on cron doesn't suppress the agent's own message-tool call. None of these are documented next to the schema entries that hint at them.

**`write` is overwrite-only.** The single biggest gotcha from the first monitor build: cron firings that report "alert written" silently lose history, because `write` calls `fs.writeFile`. An append wrapper exists in the source, gated to a memory-flush subsystem nothing else can reach. The workaround is `exec` with a shell redirect, which broadens the policy surface for no good reason. It turned out to be a known data-loss class with a long-open canonical tracker (openclaw#40001), and I added fresh repro from a current stack.

**The Dockerfile patch model is fragile by construction.** NemoClaw monkey-patches OpenClaw's source via Dockerfile patches that are shape-tied to a specific OpenClaw version, but the bundled base image drifted to a newer one — so the build fails on first install with no automated test guarding "do the patches still apply." Every bundled version needs a hand-tuned patch set, and there's no obvious way to detect drift before a user hits it.

## The Ugly

**The sandboxing layer is the entire reason to choose NemoClaw over bare OpenClaw — and that exact layer was broken.** Workspace files inside the sandbox pod are root-owned, the gateway runs as an unprivileged uid, and the gateway can't write to its own workspace directory (NemoClaw#719). A refactor moving config and writable state to a PVC-backed data dir was in flight upstream but hadn't landed. So the choice was: run un-sandboxed on bare OpenClaw (unsafe for the non-technical users this whole thing is for), or wait. There was no third option — and that fact is what eventually reframed the whole project.

**Tool-dispatch hallucination is a confidence-inducing silent failure.** The first time it bit: the agent said "I searched memory and found no matches." Sounds completely correct. Gateway log: zero tool calls. Session jsonl: zero toolCall events. The model fabricated a plausible "I did the work" reply. On a frontier model this is rare; on a local 8B with default config it's the *common* case. For an agent that takes real actions — sends messages, edits files — this is genuinely dangerous: it claims success while doing nothing. OpenClaw should refuse to run agent-loops whose tool surface exceeds the model's reliable dispatch ceiling, or at least warn at startup. Right now nothing protects a user from it. (The next page is largely about chasing this one down.)

**The smartest patterns are locked behind hardcoded gates.** The append-with-path-validation semantics exist — for one subsystem. Tiered bootstrap loading exists in an open PR. The compact skill registry keeps base prompts lean. These are almost-shipped features that would each solve a real problem if they were exposed to general agent runs. They sit in the source as architectural ghosts of a more thoughtful design that lost its way between "build this for one subsystem" and "expose it as a primitive."

**The default AGENTS.md is anthropomorphic.** The literal line from the identity template: *"You're not a chatbot. You're becoming someone."* For a platform meant to host dozens of operational workflows, prose-poem identity content competes for attention budget with the rules that actually matter. I rewrote my own to put the load-bearing instructions at the top, and filed the default as a bug (openclaw#75187). The template optimizes for personality at the expense of behavior.

## Net read on the framework

The bones are good and the people are responsive. The gap between "works for me, a technical user, today" and "safe to hand to my family" is real, and every piece of it is fixable — but on someone else's merge schedule, not mine. That gap, plus the broken sandbox being the one non-negotiable, is what turned the next week from "harden this stack" into "question this stack."
