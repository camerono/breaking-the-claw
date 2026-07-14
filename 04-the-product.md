# Building the product

*With the platform decided, the work turned into an actual product: make retrieval good, then point the monitor-and-react loop at something real. CVE advisories were the warmup. Email triage was the target.*

← prev: [The pivot](03-the-pivot.md) · [overview](README.md) · next: [The cluster grows up](05-the-cluster.md)

## The Good

**A 10× latency win on the most-exercised path, for free.** Swapping the embedding model from a larger one to a small 137M embedder dropped p50 query latency from ~4.7s to ~488ms. Both run CPU-only on this box, but the small one's CPU path is fast enough that the embed step stops being the bottleneck. Latency held flat across 100 / 500 / 1000-entry collections, which told me the retrieval search itself wasn't the cost — the embedder was.

**The document handler shipped with zero third-party dependencies.** PDF via a poppler tool already on the box, DOCX via the standard library (it's just a zip of XML). I'd started to reach for a convenience package, remembered the rule about auditing anything I pull in before I chain it, and realized the stdlib got me all the way there. End-to-end: a real paper extracted to ~62K chars, embedded, queryable. Smallest possible new attack surface.

**A cross-encoder reranker bought a 30× speedup at the rerank stage.** The obvious container image for a reranking service had no ARM64 manifest — a dead end on this hardware. Dropping one level down to the underlying libraries (PyTorch + transformers running the reranker model directly) worked on the first try: loads in ~40s, ~3 GB resident, runs as a small local service. **~0.25s per 12-candidate rerank versus 5–10s for an LLM doing the reranking.** The lesson worth keeping: when the convenience wrapper doesn't fit your hardware, drop a level — the underlying library usually has the breadth the wrapper lacks.

**The CVE warmup transferred into the email pipeline almost verbatim.** That was the entire point of doing the CVE loop first. The advisory fetcher became the IMAP poller; the embedding matcher became an LLM classify-and-draft step; the alert log became a dashboard. The same idempotency pattern — move processed inputs to a dated subdir, check before re-emitting — dropped straight in. Fresh code, but the shape was decided ahead of time, so it was authoring rather than fork-and-modify.

**Structured-output classification is genuinely good at this.** An eight-class intent enum with subtle boundaries (service-issue vs complaint, referral vs admin, billing vs billing-dispute) got 12/12 right after two prompt rounds on the small local model. The model isn't asked to *format* JSON — it's asked to fill valid values into an enforced schema, and that constraint shrinks the cognitive surface a lot.

## The Bad

**Retrieval quality is a three-layer problem, and I kept trying to fix it with one lever.** The corpus was a 1978-era TV series. First the agent quoted a character from the *2004 reboot* — because I'd quietly over-broadened the corpus from "the 1978 series only" to "all eras" during a re-chunk. Fixed the corpus. Next it quoted a passage that only name-dropped the subject in passing — because the embedder couldn't tell "this chunk is *about* X" from "this chunk *mentions* X." The answer needed all three layers aligned: the **data** (strict-era corpus), the **retrieval** (a two-stage vector → lexical-rerank → cross-encoder pipeline), and the **agent's behavior** (explicit query-construction guidance, plus a hard rule: don't quote a passage that doesn't directly address the question). Any one alone left the other two looking broken.

**"Rechunk" is not "add sources."** The era-bleed above was self-inflicted: I treated a re-ingest as license to broaden scope. Rechunk means re-ingesting the *same* material with better structure. If broader scope seems warranted, that's a question to ask first, not a thing to slip into a chunking pass.

**The first prompt-tuning round for email triage was a net regression.** I tried to fix three misclassifications at once and broke two unrelated cases — safety-urgency and empty-draft rules bled into the intent boundaries. Two more rounds to land it, and the final prompt was *shorter* than the first patch attempt. Crisp intent definitions beat layered rules; adding a top-level "EXCEPTION: do X for Y" just teaches the model to go looking for places to apply it.

## The Ugly

**`exec` is an SSRF sieve, and the model works out the bypass on its own.** The web-fetch tool enforces a real URL blocklist — it refuses loopback and private ranges, which is correct. But any agent that also has `exec` (most of them) can reach the same addresses via `curl`, and across four of five local-IP probe cells the agent *escalated to `exec(curl …)` by itself* when web-fetch refused. So the SSRF guard is real on paper and cosmetic in practice. You can inspect `exec` argv for URL patterns, but that's a deny-list game the model beats with `nc`, a Python one-liner, a base64'd URL. The categorical fix is a network-isolated exec — which means waiting on the sandbox. Filed upstream (openclaw#76260); the most security-relevant thing I found. Two months later it's still open, flagged P1 with security impact — and also labeled `stale`, with no maintainer comment. The [scorecard](06-upstream-scorecard.md) has the rest of that story.

**A careless harness left ghost duplicates I didn't catch for a while.** A drop-then-recreate on a collection assumed an atomicity it didn't have; an orphaned child process kept writing after its parent died and raced the recreate, leaving 36 phantom entries. Two minutes of paranoid checking would have caught it. I didn't do the two minutes, and only noticed when a count came out wrong. The ledger has it now — the ugly part isn't the race, it's that I shipped a sloppy harness and trusted it.

**My first instinct under weird output is always to blame the model.** Twice — the missing-tool bug and the email regression — I spent real time suspecting the model or the infra when the actual cause was something I'd just changed in the prompt or the config. When an agent narrates "I will now use the tool" and then doesn't, the likeliest explanation isn't confusion — it's that **the tool isn't actually attached**, or the prompt asked for the wrong thing. Take the narration literally: it's the agent telling you it wanted a tool it couldn't reach.

## Net read on the product

By the end of this stretch there's a real monitor-and-react system: good retrieval behind a proper reranker, a CVE-advisory matcher tuned to catch true positives and suppress the near-misses, and an email-triage pipeline that classifies intent and drafts replies with structured-output enforcement. The warmup-then-target sequencing worked exactly as intended — build the disposable version on an easy feed, then transfer the shape to the one that matters.
