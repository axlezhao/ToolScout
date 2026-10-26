# ToolScout: Evaluation Plan

*Status: plan. Numbers in this document are targets or budgets, not results. Revised October 25, 2026: the agent runs on LangGraph, and LangChain-native baselines are added.*

The evaluation has three layers. The retrieval benchmark is cheap and runs on every change. The end-to-end runs cost money and run at milestones. Failure attribution explains the end-to-end results.

## 1. Retrieval benchmark (offline, runs in CI)

### 1.1 Tool corpus

| Tier | Tools | Source |
|---|---|---|
| Small | ~50 | A few servers from LiveMCPBench, plus Wanderly's MCP server |
| Main | 527 | All of LiveMCPBench's tools (70 servers), exported with `toolscout index` from their Docker image |
| Large | ~2,500 | Main + decoy tools from public MCP registries and the ToolRet corpus |

**Decoys** are tool *definitions* only. They're indexed but never executed. In end-to-end runs, calling a decoy returns an "unavailable" error and counts as a failure. Each decoy set is sampled with a fixed seed and saved in the repo, so runs can be reproduced.

### 1.2 Queries and labels

| Set | Size (target) | How it's made |
|---|---|---|
| **Task queries** | 95 | LiveMCPBench tasks, each labeled with the tools it needs. Use their annotations if `annotated_data/` has them, otherwise label by hand |
| **Single-tool queries** | ~500 | An LLM writes requests from tool descriptions and is told not to reuse the tool's name words. A random 10% is checked by hand |
| **Look-alike queries** | ~100 | Requests whose correct tool has a near-duplicate on another server |

Labels live in `evals/labels/*.jsonl`, one query per line with its correct tool IDs.

**Split:** 30% dev, used for tuning field weights, `k1`, `b` and fusion settings. 70% test, used only for reported numbers. The split is by *tool*, not by query, so tuning can't learn from test tools.

### 1.3 Retrievers compared

| Retriever | Built with |
|---|---|
| Stock BM25 (ablation) | `BM25Retriever` over concatenated fields, ToolScout tokenizer |
| BM25F | ToolScout's `BM25FRetriever` |
| Dense | `InMemoryVectorStore` + one local and one API embedding model |
| Hybrid | `EnsembleRetriever` (RRF) over BM25F and dense |

All four share the `BaseRetriever` interface, so the benchmark runner treats them the same way.

### 1.4 Metrics

- **Recall@k** for k = 1, 5, 10, 20: is a correct tool in the top k?
- **Complete-recall@k:** for tasks that need several tools, are *all* of them in the top k?
- **MRR** and **nDCG@10**
- **Latency** p50 and p95 per retriever and tier, plus index build time

### 1.5 CI gate

GitHub Actions runs the retrieval suite on every pull request:

- **Lexical:** always, with no API keys needed.
- **Dense and hybrid:** from cached tool vectors and cached query vectors (pinned embedding model, `CacheBackedEmbeddings` store committed to the repo), so CI needs no keys either.
- **Fails the PR if**:
  - Recall@10 on the dev split drops more than 1 point below `main`, or
  - lexical p95 latency goes above budget.
- **Posts a summary table** as a job summary.

## 2. End-to-end agent runs

### 2.1 Setup

- **Environment:** LiveMCPBench's Docker image with its 70 servers. Use their tool-check script to freeze a subset of tasks whose tools work, and record the date.
- **Agent:** one LangGraph agent built with `langchain.agents.create_agent`, with the same system prompt, model settings and limits everywhere. A condition is a config that sets only two things: the tools passed in and the tool-selection middleware. Everything else is shared:
  - `ModelCallLimitMiddleware` and `ToolCallLimitMiddleware` with the same caps, so a looping run can't burn budget
  - a tracing middleware and callback handler that write the JSONL trace (searches, visible tools per step, calls, tokens)
- **Model:** one small, cheap model as the main agent. Record the exact model ID and dates. Optional: repeat the main comparison with a stronger model.
- **Judge:** LiveMCPBench's LLM-judge setup (LiveMCPEval), with the judge model recorded.
- **Runner:** a small LangGraph graph per run, `agent → judge → attribute`, so a run's verdict and failure cause are written next to its trace. Runs execute concurrently with a fixed concurrency limit.

### 2.2 Conditions

**Core conditions** (required for v0.1):

| ID | Condition | What the agent sees |
|---|---|---|
| A | All tools in prompt | Every tool definition (if it fits; otherwise report that it doesn't) |
| B | LiveMCPBench baseline | Their MCP Copilot agent's embedding retrieval, run in their harness with the same model |
| C | ToolScout, lexical | `search_tools` + `call_tool` over MCP, BM25F (middleware `gateway` mode gives the same surface in-process) |
| D | ToolScout, hybrid | Same as C, with BM25F + dense (`EnsembleRetriever`) |

**Ecosystem baselines** (run if the pilot shows budget is left; 1 run each):

| ID | Condition | What the agent sees |
|---|---|---|
| E | `LLMToolSelectorMiddleware` | Tools picked by a small selector model before each call (`max_tools` = ToolScout's `k`) |
| F | `langgraph-bigtool` | Its `retrieve_tools` step over an `InMemoryStore` with the same embedding model as D |
| G | `ProviderToolSearchMiddleware` | Provider-native tool search (only if the main model is from a supported provider) |

E–G answer the question a LangChain user would ask first: "why not just use what LangChain already has?" Their token counts include the selector model's tokens (E) and any provider-side search tokens reported (G).

### 2.3 Metrics

- **Task success rate** (judge)
- **Input and output tokens per task**, from each message's `usage_metadata`, including any auxiliary model calls (selector, judge excluded)
- **Cost per successful task** = total cost ÷ successful tasks
- **Wall-clock time per task** and **searches per task**
- **Failure causes** (section 3)

### 2.4 Runs and cost control

1. **Pilot:** 20 tasks × conditions A–D × 1 run, to measure real cost and time per run.
2. **Full:** all frozen tasks × A–D × 3 runs, only after the pilot cost is known and within budget.
3. **Ecosystem baselines:** all frozen tasks × E–G × 1 run, only if budget is left after the full runs.
4. Record every run as JSONL: config hash, package versions (`langchain`, `langgraph`, `fastmcp`), model ID, seed, trace, tokens, judge verdict. LangSmith can mirror the runs for browsing, but the committed JSONL is what results are computed from.

### 2.5 Statistics

- All conditions run on the same tasks, so comparisons are **paired**.
- Success: McNemar's test on per-task outcomes, with runs aggregated per task.
- Tokens and cost: paired bootstrap, 10,000 resamples, 95% CIs.
- Differences whose CI includes zero are reported as "no clear difference", never as a win.

### 2.6 Checking the judge

- Hand-label about 100 runs, stratified across conditions and verdicts.
- Report agreement with the judge (Cohen's κ) and the cases where they disagree.
- If κ < 0.7, report success rates from the hand labels next to the judge's, and treat the judge's numbers as rough.
- Optional follow-up (M6): train a small judge on these traces with SSL, SFT and RL, and test whether it can replace the large one. See [JUDGE.md](JUDGE.md).

## 3. Failure attribution

Every failed run gets one main cause, assigned by rules over the JSONL traces first. An LLM classifier handles only the ambiguous cases, and a sample is checked by hand.

| Cause | How it's detected |
|---|---|
| **Never retrieved** | A needed tool never appeared in any `search_tools` result or middleware selection |
| **Retrieved, not called** | The needed tool was returned but the agent didn't call it |
| **Look-alike chosen** | The agent called a near-duplicate from another server instead |
| **Bad arguments** | The schema check or the upstream server rejected the arguments |
| **Upstream error** | The server failed or timed out with valid arguments |
| **Gave up early** | The agent stopped before calling the needed tools, with no error |
| **Other** | None of the above; reviewed by hand |

The output is a stacked bar chart of causes per condition. That shows *where* each search setup loses tasks, not just how many.

## 4. Rules for honest results

- Tune only on the dev split. Report only the test split.
- doc2query examples (Phase 2) are generated from tool definitions only, never from benchmark tasks.
- Commit every reported number with the config and data version that produced it (`evals/results/`).
- Report the conditions where ToolScout does worse, not only the ones where it does better.
- Never compare against a baseline number copied from a paper when the setup differs. Rerun it or leave it out.
- Run the ecosystem baselines (E–G) with their documented defaults, changing only what's needed to match `k` and the embedding model. If ToolScout gets tuned on dev, say how much the baselines were tuned too.
