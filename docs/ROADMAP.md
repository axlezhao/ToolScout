# ToolScout: Roadmap

*Planned as about 5 weeks of part-time work, starting the week of September 28, 2026. Dates are targets.*

## M0: Proposal (week of Sep 28) ✅

- [x] Problem statement and prior work ([PROPOSAL.md](PROPOSAL.md))
- [x] Architecture ([DESIGN.md](DESIGN.md))
- [x] Evaluation plan ([EVALUATION.md](EVALUATION.md))
- [x] Decision log ([DECISIONS.md](DECISIONS.md))

## M1: Gateway skeleton (target: Oct 5)

- [ ] Go module, CLI entry point (`serve`, `index`), MIT license, CI running `go vet` and tests
- [ ] YAML config with `${VAR}` expansion
- [ ] Upstream manager: stdio and Streamable HTTP connections with the official Go SDK, `tools/list` with pagination, `list_changed` handling, reconnect with backoff
- [ ] Tool catalog: `ToolRecord`, schema hashing, change detection
- [ ] Router server: `search_tools` (plain substring match for now) and `call_tool` (forwarding only)
- [ ] Check the namespaced tool-name format against the MCP spec's naming rules
- [ ] Try it with Wanderly's Go MCP server and one or two official reference servers

**Done when:** Claude Code can find and call a Wanderly tool through ToolScout.

## M2: Lexical search (target: Oct 12)

- [ ] Tokenizer: identifier splitting, stop words, Porter stemming, with unit tests
- [ ] BM25F index with per-field weights and length normalization
- [ ] `toolscout search` with per-field score breakdowns
- [ ] Export LiveMCPBench's 527 tools with `toolscout index`
- [ ] First retrieval numbers on a small hand-labeled query set
- [ ] Latency benchmark at 527 and ~2,500 tools

**Done when:** there's a first table of Recall@k and latency for BM25F in `evals/results/`.

## M3: Hybrid search and CI gate (target: Oct 19)

- [ ] Embedding provider interface: one local and one API implementation, vector cache keyed by schema hash
- [ ] Dense retriever (brute-force cosine) and hybrid (RRF)
- [ ] Labels for the 95 LiveMCPBench tasks, single-tool query set, look-alike query set
- [ ] Decoy tools for the ~2,500 tier, sampled with a fixed seed
- [ ] Dev/test split by tool; tune BM25F weights on dev only
- [ ] `toolscout eval --suite retrieval`
- [ ] GitHub Actions gate: Recall@10 and latency checks, with a job summary table
- [ ] Call proxy: schema checks, timeouts, result size cap
- [ ] Safety policy: allow/deny lists, hide destructive tools by default

**Done when:** a PR that lowers dev Recall@10 by more than 1 point fails CI, and the test-split table compares BM25F, dense and hybrid at three tool-count tiers.

## M4: End-to-end runs (target: Oct 26)

- [ ] ReAct agent runner that records JSONL traces
- [ ] LiveMCPBench environment in Docker, tool check, frozen task subset
- [ ] Pilot: 20 tasks × 4 conditions × 1 run, then check cost
- [ ] Full runs: frozen tasks × 4 conditions × 3 runs (if the pilot cost is within budget)
- [ ] Hand-label ~100 runs; judge agreement (κ)
- [ ] Failure attribution by rules, with LLM fallback and a hand-checked sample
- [ ] Paired statistics: McNemar for success, bootstrap CIs for tokens and cost

**Done when:** results for conditions A–D are in `evals/results/` with CIs and failure-cause charts.

## M5: v0.1 release (target: Nov 2)

- [ ] README results section: main table, 3–4 charts, honest limits
- [ ] Setup guides for Claude Code, Claude Desktop and one other MCP client
- [ ] Demo GIF: an agent finding and calling tools across many servers
- [ ] Short write-up of the findings (blog post or `docs/RESULTS.md`)
- [ ] Tag `v0.1.0` with release binaries

## Phase 2: research questions (optional, after v0.1)

- [ ] **Who writes the query:** one-shot retrieval vs. agent calls `search_tools` vs. dynamic exposure
- [ ] **doc2query for BM25:** generated example requests as an extra field
- [ ] **Look-alike tools:** failures at scale, split by near-duplicate vs. count
- [ ] **Choosing k:** success and cost as k goes from 1 to 20
- [ ] **Stronger model:** does a stronger model make up for weaker search?
- [ ] **Cross-lingual:** Chinese requests against English tool descriptions
- [ ] **Reranker:** cross-encoder or LLM rerank of the top 20

## Out of scope for v0.1

- Proxying resources, prompts or sampling
- Hosted service, accounts or web dashboard
- Training a retrieval model
