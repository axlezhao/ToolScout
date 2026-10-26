# ToolScout: Roadmap

*Planned as about 5 weeks of part-time work. Re-planned October 25, 2026 for Python, LangChain and LangGraph, restarting from M1. Dates are targets.*

## M0: Proposal (Sep 28) ✅

- [x] Problem statement and prior work ([PROPOSAL.md](PROPOSAL.md))
- [x] Architecture ([DESIGN.md](DESIGN.md))
- [x] Evaluation plan ([EVALUATION.md](EVALUATION.md))
- [x] Decision log ([DECISIONS.md](DECISIONS.md))

## M0.5: Re-plan on LangChain and LangGraph (Oct 25) ✅

- [x] Move the design from Go to Python, LangChain and LangGraph ([D1](DECISIONS.md#d1-python-on-langchain-and-langgraph))
- [x] Add the LangChain middleware surface and the LangChain-native baselines

## M1: Gateway skeleton (target: Nov 1)

- [ ] `uv` project, Typer CLI (`serve`, `index`), MIT license, CI running `ruff`, `mypy` and `pytest`
- [ ] Pin `langchain[mcp]`, `langgraph` and `fastmcp` versions
- [ ] YAML config (Pydantic) with `${VAR}` expansion, converted to an `MCPConfig`
- [ ] Upstream manager: one `MCPAdapter` over stdio and Streamable HTTP servers, `list_tools()`, refresh, health per server
- [ ] Check what `MCPAdapter` exposes: tool annotations, raw `inputSchema`, `list_changed`, per-call session cost. Fill gaps from FastMCP's client
- [ ] Tool catalog: `ToolRecord`, schema hashing, change detection
- [ ] FastMCP server: `search_tools` (plain substring match for now) and `call_tool` (forwarding only)
- [ ] Settle the tool-name format (Q1)
- [ ] Try it with Wanderly's MCP server and one or two official reference servers

**Done when:** Claude Code can find and call a Wanderly tool through ToolScout.

## M2: Lexical search (target: Nov 8)

- [ ] Tokenizer: identifier splitting, stop words, Porter stemming, with unit tests (reused from the earlier search engine)
- [ ] `BM25FRetriever(BaseRetriever)` with per-field weights and length normalization
- [ ] Stock `BM25Retriever` wired up as an ablation
- [ ] `toolscout search` with per-field score breakdowns
- [ ] Export LiveMCPBench's 527 tools with `toolscout index`
- [ ] First retrieval numbers on a small hand-labeled query set
- [ ] Latency benchmark at 527 and ~2,500 tools

**Done when:** there's a first table of Recall@k and latency for BM25F and stock BM25 in `evals/results/`.

## M3: Hybrid search, middleware and CI gate (target: Nov 15)

- [ ] Dense retriever: `Embeddings` + `CacheBackedEmbeddings` + `InMemoryVectorStore`, one local (Ollama) and one API model
- [ ] Hybrid: `EnsembleRetriever` (RRF) over BM25F and dense
- [ ] `ToolScoutMiddleware` with `retrieve` and `gateway` modes, tested with `create_agent`
- [ ] Labels for the 95 LiveMCPBench tasks, single-tool query set, look-alike query set
- [ ] Decoy tools for the ~2,500 tier, sampled with a fixed seed
- [ ] Dev/test split by tool; tune BM25F weights and ensemble weights on dev only
- [ ] `toolscout eval --suite retrieval`
- [ ] GitHub Actions gate: Recall@10 and latency checks, with a job summary table
- [ ] Call proxy: schema checks, timeouts, result size cap
- [ ] Safety policy: allow/deny lists, hide destructive tools by default

**Done when:** a PR that lowers dev Recall@10 by more than 1 point fails CI, the test-split table compares stock BM25, BM25F, dense and hybrid at three tool-count tiers, and a `create_agent` agent with the middleware completes the Wanderly demo task.

## M4: End-to-end runs (target: Nov 22)

- [ ] LangGraph runner: one `create_agent` agent, conditions as config, JSONL traces from middleware and a callback handler
- [ ] Cost guards: `ModelCallLimitMiddleware` and `ToolCallLimitMiddleware` with the same limits in every condition
- [ ] LiveMCPBench environment in Docker, tool check, frozen task subset
- [ ] Pilot: 20 tasks × core conditions (A–D) × 1 run, then check cost
- [ ] Full runs: frozen tasks × A–D × 3 runs (if the pilot cost is within budget)
- [ ] Ecosystem baselines E–G, 1 run each, if budget allows
- [ ] Hand-label ~100 runs; judge agreement (κ)
- [ ] Failure attribution by rules, with LLM fallback and a hand-checked sample
- [ ] Paired statistics: McNemar for success, bootstrap CIs for tokens and cost

**Done when:** results for conditions A–D are in `evals/results/` with CIs and failure-cause charts.

## M5: v0.1 release (target: Nov 29)

- [ ] README results section: main table, 3–4 charts, honest limits
- [ ] Setup guides for Claude Code, Claude Desktop, one other MCP client, and LangChain `create_agent`
- [ ] Demo GIF: an agent finding and calling tools across many servers
- [ ] Short write-up of the findings (blog post or `docs/RESULTS.md`)
- [ ] Tag `v0.1.0` and publish `toolscout` to PyPI

## M6: A small trained judge (optional, after v0.1, ~3–4 weeks)

Plan and equations: [JUDGE.md](JUDGE.md). Needs the M4 traces.

- [ ] Raise hand labels from ~100 to ~300, stratified by condition and judge verdict
- [ ] Render traces for the judge without condition-revealing steps
- [ ] 5-fold cross-fitting split by task, fixed seed
- [ ] Stage 1: SSL on unlabeled traces (language modeling + next tool call)
- [ ] Stage 2: SFT on silver labels and rationales, with class weights
- [ ] Stage 3: GRPO with gold labels as the reward, KL to the SFT model
- [ ] Ablations: base, SFT, SSL+SFT, SFT+RL, full, and the large judge
- [ ] H1–H3 with task-clustered bootstrap CIs

**Done when:** a table of κ for every judge variant with CIs, and the H3 check for C vs. D and C/D vs. A, are in `evals/results/judge/`.

## Phase 2: research questions (optional, after v0.1)

- [ ] **Who writes the query:** middleware `retrieve` vs. `search` (dynamic exposure) vs. `gateway`
- [ ] **doc2query for BM25:** generated example requests as an extra field
- [ ] **Look-alike tools:** failures at scale, split by near-duplicate vs. count
- [ ] **Choosing k:** success and cost as k goes from 1 to 20
- [ ] **Stronger model:** does a stronger model make up for weaker search?
- [ ] **Cross-lingual:** Chinese requests against English tool descriptions
- [ ] **Reranker:** `ContextualCompressionRetriever` with a cross-encoder or LLM rerank of the top 20

## Out of scope for v0.1

- Proxying resources, prompts or sampling
- Hosted service, accounts, web dashboard or LangGraph Platform deployment
- Training a retrieval model
- A LangChain.js port
