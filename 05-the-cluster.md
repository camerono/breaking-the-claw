# The cluster grows up

*Nothing user-visible shipped in this stretch, but the infrastructure underneath it went from "one box I keep poking" to "a prod/dev split with a real deploy path." Plus a runtime bench that came out well off what I expected.*

← prev: [Building the product](04-the-product.md) · [overview](README.md)

## The Good

**A two-instance prod/dev split came together fast.** Dev stays on the GPU box (the existing setup, relabeled); prod stands up clean on a separate machine; a single branch is the source of truth between them. The deploy tool takes `--target dev|prod` and treats each environment's declared apps and tools as a real allowlist — it fails closed if you try to push something off-list. The smoke test round-tripped clean and idempotent on re-apply. Those are the bones I want for the next year of authoring against this thing.

**The undocumented-Console-API deploy path was less scary than feared.** Dify's Console API has no public docs and prior experience said "expect surprises," so I'd flagged it as the risky piece. It came in at ~250 lines total, with the OpenAPI spec kept as readable YAML in the repo and JSON-stringified only on the wire — so diffs stay honest about what Dify actually stores. Renaming a tool requires delete-then-recreate (now documented); everything else just works.

**A fifth machine joined the cluster** and the cross-host load-balancing writeup forced me to actually think placement through instead of hand-waving. The v1 answer is deliberately boring: round-robin in front, a small placement registry, a puller cron — debuggable, with room to grow into model-aware routing if and when cold-start tax actually shows up in the dashboards. Worth having that doc as a target *before* the second GPU box comes online, so it's not designed under pressure.

## The Bad

**The shared memory bus got flaky on larger writes.** Three of the day's longer coordination writes timed out — all on entries past a few KB. I worked around it by shortening them on retry, which means the canonical shared record is thinner than my local docs. Not the first time it's been finicky on bigger writes; it needs a real look at its write timeouts rather than another round of me trimming entries to fit.

**"It's 3–6× faster" is a long way from "so switch."** A runtime bench (below) showed real headroom, but a real switch is a project, not a config flip: the model stores don't share namespaces, the Bearer-auth plugin I built for the incumbent has no analog on the alternative, and every consumer that hardcodes the old endpoint would need a config fan-out. Knowing the headroom exists is useful even if I don't act on it — and the more interesting near-term move is probably tuning the incumbent (KV-cache dtype, more parallel slots) and re-benching before committing to anything.

## The Ugly

**Auth-path proliferation is turning into tech debt.** Getting one host to pull a branch meant working around three different auth shapes for the *same* internal git server — an SSH key that works from one machine, credential-less HTTP on another, no clone at all on a third. I ended up copying the changed files onto the box, running the test, and reverting. It worked, but the friction compounds, and it'll bite the moment a runbook needs to land a real install across several hosts at once. Consolidate before then, not during.

**I left state on the box I hadn't decided to keep.** The runtime bench pulled a couple of extra GB of images and model weights, and I documented the cleanup commands but didn't run them — because I hadn't decided whether the alternative runtime stays. Not a real problem on a big NVMe, but exactly the kind of "decide later" residue that compounds if I'm not deliberate about it.

## Sidebar — the runtime bench that surprised me

I expected the alternative model runtime to be "comparable, maybe a hair faster on small models." It came in **3.2× faster on a 3B model and 5.9× faster on a 360M model**, with byte-identical output (same weights, deterministic seed), and the incumbent showed 30%+ run-to-run jitter on identical prompts. That's a genuine surprise and real headroom. But it's a narrow slice — single-stream, small models — and the question that actually matters for this workload is whether the gap holds at 7B and under concurrent load. That's a separate hour-long spike, worth queuing only when something is actually pushing the throughput ceiling. Not today.

## Net read on the cluster

The platform grew up: a prod/dev split with a fail-closed deploy path, a documented (if undocumented-upstream) way to ship Custom Tools, a second GPU box's worth of routing designed before it's needed, and a runtime option with measured headroom sitting in the queue. The debt is honest and named — auth-path sprawl, a flaky coordination bus, undecided leftover state — which is the right place for it to live until it's worth paying down.
