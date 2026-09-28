# ToolScout: Project Proposal

*Author: Yize (Axle) Zhao · Drafted September 28, 2026 · Status: proposal*

## 1. Summary

ToolScout is an MCP gateway written in Go. It connects to many MCP servers, indexes all of their tools, and gives the agent two tools instead of hundreds: `search_tools` to find the right tools, and `call_tool` to use them. The project's main claim will be a measured one. For a fixed agent and task set, ToolScout should cut prompt tokens per task sharply compared with loading every tool, without lowering task success. Every search design choice is backed by a benchmark that runs in CI.

## 2. Motivation

MCP made it easy to connect an agent to many tool servers. Most clients still send every tool definition to the model on every request. That causes three problems:

1. **Cost and context limits.** In Stacklok's test, 2,792 tool definitions came to about 206K tokens, which is more than many models can accept.
2. **Accuracy.** LiveMCPBench (70 servers, 527 tools, 95 tasks) found that retrieval errors cause nearly half of agent failures. Most models solved only 30–50% of tasks.
3. **Look-alike tools.** Different servers often expose tools with nearly the same name and purpose. The model has to choose between them from short descriptions.

Searching over tools instead of loading all of them is a known fix. RAG-MCP cut prompt tokens by more than half and roughly tripled tool-selection accuracy (43% vs. 14%). Anthropic's tool search tool reports large token savings and higher accuracy on its MCP evaluations.

## 3. What already exists, and what's missing

| Work | What it shows | What it doesn't cover |
|---|---|---|
| [Anthropic tool search](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool) | Agent-driven search with regex or BM25, and `defer_loading` for tools | Only for the Claude API; not an open, client-side component |
| [Stacklok MCP Optimizer](https://stacklok.com/blog/stackloks-mcp-optimizer-vs-anthropics-tool-search-tool-a-head-to-head-comparison/) | Hybrid (semantic + BM25) search: 94% vs. 34% for BM25 on 2,792 tools | Measures retrieval accuracy, not end-to-end task success |
| [StackOne](https://www.stackone.com/blog/mcp-tool-search-bm25-tfidf-hybrid/), [MCPProxy](https://mcpproxy.app/blog/2026-03-19-bm25-accuracy-hybrid-search/) | BM25, TF-IDF and hybrid compared on 916+ tools | Same: retrieval accuracy on single queries |
| [ToolRet](https://aclanthology.org/2025.findings-acl.1258/) | Strong general search models do poorly at finding tools (43k tools) | Not specific to MCP gateways |
| [LiveMCPBench](https://arxiv.org/abs/2508.01780) | End-to-end tasks over 527 real tools, with a judge | Uses one fixed retriever (embedding-based) |
| [RAG-MCP](https://arxiv.org/abs/2505.03275) | One-shot retrieval before the agent runs | One retriever, one search mode |

**The gap ToolScout fills:** an open, client-agnostic gateway whose search choices are tested **end to end**, on task success and cost per successful task, not only on retrieval accuracy. The benchmark is part of the repository and gates every change.

## 4. Goals

1. **G1: A working gateway.** Any MCP client can use ToolScout in front of 50+ servers and 500+ tools.
2. **G2: Measured search.** BM25F is the default, hybrid is opt-in, and a benchmark in the repo backs both.
3. **G3: Measured impact.** An end-to-end comparison on LiveMCPBench against (a) all tools in the prompt and (b) LiveMCPBench's own retrieval baseline.
4. **G4: Easy to explain.** Every major design choice is recorded in [DECISIONS.md](DECISIONS.md) with the evidence behind it.

## 5. Non-goals (v0.1)

- Proxying MCP resources, prompts or sampling. Tools only.
- Training or fine-tuning a retrieval model.
- A hosted service, multi-user accounts or a web dashboard.
- Handling authentication for upstream servers beyond passing configured headers and environment variables.

## 6. Approach

**Build (details in [DESIGN.md](DESIGN.md)):**
- Go, on the official MCP Go SDK. ToolScout is an MCP client to upstream servers and an MCP server to the agent.
- A tool catalog that stays current as servers connect, disconnect or change their tools.
- Search:
  - **BM25F:** field-weighted BM25 over tool name, server, description and parameters. It carries over the field-boosted BM25 design from my earlier [search engine](https://github.com/axlezhao/search-engine), ported from Python to Go.
  - **Dense:** embeddings with an in-memory vector cache.
  - **Hybrid:** reciprocal rank fusion of both.
- Checks argument schemas before forwarding calls, applies timeouts, and keeps traces for every search and call.

**Measure (details in [EVALUATION.md](EVALUATION.md)):**
- A retrieval benchmark on LiveMCPBench's 527 tools plus decoy tools (up to about 2,500), with a dev/test split. It runs in CI.
- End-to-end agent runs on LiveMCPBench tasks, with paired statistics and a check of how often the judge agrees with hand labels.
- Failure attribution from traces.

## 7. Success criteria

These are **targets to test**, not results.

| Criterion | Target |
|---|---|
| Works with real clients | Claude Code and at least one other MCP client can search for and call tools through ToolScout |
| Retrieval (hybrid, test split, ~2,500 tools) | Recall@10 ≥ 0.90 |
| Retrieval (BM25F, test split, ~2,500 tools) | Report honestly, whatever it is. Goal: within 10 points of hybrid |
| Search latency, lexical | p95 < 10 ms at ~2,500 tools |
| End-to-end tokens | ≥ 70% fewer input tokens per task than all tools in the prompt |
| End-to-end success | Not worse than all tools in the prompt (paired 95% CI includes or exceeds 0) |
| CI gate | Blocks a change that lowers Recall@10 on the dev split by more than 1 point |

## 8. Phase 2: optional research questions

If v0.1 ships on time, these turn the gateway into a small study:

1. **Who writes the query?** One search on the user's request, the agent calling `search_tools` itself, or dynamic exposure (found tools are added to the tool list). Does the gap between BM25 and hybrid shrink when the agent writes its own queries?
2. **Does doc2query close the gap for BM25?** This means adding LLM-generated example requests to each tool's document before indexing.
3. **Look-alike tools.** How much of the failure at scale comes from near-duplicate tools, rather than the total number of tools?
4. **Choosing k.** Where does success peak between missing tools and confusion?
5. **Cross-lingual queries.** Chinese requests against English tool descriptions: BM25 against multilingual embeddings.

## 9. Risks and mitigations

| Risk | Mitigation |
|---|---|
| LiveMCPBench servers depend on live APIs and some break | Use their Docker image and tool-check script. Freeze a subset that runs reliably and report it with dates |
| 95 tasks is small, so differences may be noise | 3 runs per task, paired comparisons, bootstrap CIs. Don't claim differences inside the noise |
| The LLM judge is wrong | Hand-label about 100 runs and report agreement (Cohen's κ) |
| API cost grows | Run a 20-task pilot to measure cost per run before full runs. Use a small model as the main agent |
| Similar products already exist | Positioned as open, measured and client-agnostic, not "first" |
| Clients handle `tools/list_changed` differently | The default two-tool surface doesn't depend on it |

## 10. Deliverables

1. The `toolscout` binary and its source (Go), MIT licensed
2. A retrieval benchmark with labels, decoys and a CI gate
3. End-to-end results with charts in the README
4. A short write-up of the findings
