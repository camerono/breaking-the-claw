# Did anyone fix it? — the upstream scorecard

*I filed a lot of bugs across three projects. Two months on, here's what actually moved. Checked mid-July 2026; states below are live as of then, and the honest read is that the three projects behave very differently when you hand them evidence.*

← prev: [The cluster grows up](05-the-cluster.md) · [overview](README.md)

The short version: **NemoClaw fixes things, OpenClaw labels things, and Dify's community fixes things while the merge waits on code owners.** A filed bug is a good probe for a project's health, and this run turned into an accidental test of all three.

## NemoClaw — the project that moved

This is the one that earns the goodwill. Real people, real merges, on a timescale that made a difference to the build.

- **The load-bearing blocker got fixed.** The sandbox's config file being created root-owned and read-only — the single issue that made "run this safely for my family" impossible — is **closed and resolved**, with a human assignee. That was the fix the whole sandboxed path was waiting on.
- **The reliability PR landed.** One merged pull request (early May) closed three issues at once: the Ollama tool-call template leak, a silently-ignored policy-config field, and a rebuild that aborted on root-owned files. It was merged by a maintainer and then referenced by follow-up PRs adding release notes and end-to-end tests — which is what a healthy fix pipeline looks like.
- **Even the stalled one recovered.** The Dockerfile-patch-drift build failure that an assignee couldn't reproduce and walked away from in April is now **closed with a linked fix PR**. It took the long way around, but it got there.
- **The fast one stayed fast.** The very first batch included an issue closed COMPLETED with a shipped fix inside a day, plus two earlier PRs (proxy token self-heal, multi-sandbox port allocation) that were already merged when I arrived.

**Mobility: high.** Human maintainers, evidence gets read, fixes ship, and the hardest problem on my list is closed. If NemoClaw were the whole stack I'd have fewer reservations about it.

## OpenClaw — brilliant intake, stalled throughput

OpenClaw has the most impressive *automated* triage layer I've seen on an open-source project, and it is also where my best-evidenced bugs went to sit. Both of those are true at once, and the combination is the story.

The bot (its Codex-backed reviewer) is genuinely good: it reproduces issues at the source level, assigns priority, tags security impact, and rates evidence quality. Several of my filings came back flagged P1 with "source-repro" and "fix-shape-clear." The problem is what happens *after* that.

- **Validated security bugs are going stale, not getting fixed.** The `exec`-bypasses-the-SSRF-block finding is **still open**, flagged P1 with security impact — and also carries a `stale` label, with no maintainer comment. The fabrication-under-tool-failure issue is the same: **open, P1, security impact, stale, bot-only**. The data-loss `write`-has-no-append-mode tracker has been **open since March**, P1, data-loss, no maintainer plan.
- **A validated filing got auto-closed anyway.** My cleanest security filing — the `exec` schema exposing operator-only fields to the model — is **closed "not planned" and marked stale**, despite the bot's own line-by-line review endorsing the fix direction.
- **The bot closed a real bug against its own advice, and the complaint got locked.** The cron-runId issue was closed `completed` by the same bot whose Codex review said "keep open." I filed a meta-issue about exactly that failure mode; it was **closed "not planned" and then locked by a bot, with no human ever responding.** That's the moment that colored my read of the whole project.
- **Community fixes wait too.** The tiered-bootstrap feature I proposed *and wrote a PR for* is still **open and unmerged**, P2, sitting behind "needs product decision." Same shape as the `skipBootstrap`-is-a-no-op bug: linked PR open, awaiting a human call that hasn't come.

To be fair, it isn't all stall — an earlier bootstrap-truncation bug did close COMPLETED with a real fix, and one thread got a genuinely warm maintainer reply when I brought wire-level evidence. But the pattern across two months is clear.

**Mobility: fast to triage, slow to fix.** The automation is simultaneously the best and the worst thing about the project. It means nothing rots unlabeled — and it also means a P1 security bug can sit "stale, needs-maintainer-review" indefinitely while a bot occasionally closes the correct ones. For a platform I'd hand to non-technical users, "the security queue depends on a maintainer who isn't showing up" is the load-bearing worry, and it's why I stopped betting the product on OpenClaw's own timeline.

## Dify — community-driven, merge-gated

Only one filing here, so treat this as a smaller sample. The bug — a cloned agent app silently carrying a stale tool attachment — is **still open**, but it prompted the healthiest-looking response of the three in one specific way: **a community contributor wrote a fix PR** (a UI warning when an app exposes only a subset of a tool's operations), and I tested and approved the approach in mid-May. The PR is **still open**, waiting on the named code owners to review.

**Mobility: engaged ecosystem, official merge is the bottleneck.** Somebody who isn't a core maintainer cared enough to build the fix, which says good things about the community. Whether it ships is a code-owner-bandwidth question, same genus as OpenClaw's — just without the automation theater on top.

## Net read on the three projects

If I rank them by "what happens when you hand them a well-evidenced bug":

1. **NemoClaw** — reads it, fixes it, merges it. The blocker that gated my whole project is closed.
2. **Dify** — the community may fix it faster than the core team merges it. One data point, but a good one.
3. **OpenClaw** — labels it beautifully, then it depends. Real fixes exist; so do P1 security bugs sitting stale for months and a bot that closes valid issues and locks the complaint.

None of this changes the architecture verdicts on the earlier pages — OpenClaw's bones are still good, the sandbox model is still sound. What it changes is the *timeline* you can plan around. NemoClaw's responsiveness is something you can build on. OpenClaw's is something you route around, which is most of why the product ended up where it did.
