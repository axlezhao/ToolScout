# ToolScout: Decisions

A running log of design decisions and the reasons for them. Each entry says what would make us revisit it.

## Decided

### D1: Python, on LangChain and LangGraph
*Oct 25, 2026. Supersedes the Sep 28 decision to use Go on the official MCP Go SDK.*

- `langchain.mcp.MCPAdapter` (built on FastMCP) already handles multi-server configs, stdio and Streamable HTTP, and tool namespacing, and it returns LangChain tools. That's most of M1's upstream work.
- LangChain's `BaseRetriever`, `Embeddings`, `CacheBackedEmbeddings`, `InMemoryVectorStore` and `EnsembleRetriever` cover the dense and hybrid side, so the custom code is the part worth measuring: BM25F.
- `create_agent` (on LangGraph) gives the end-to-end evaluation one agent loop for every condition, plus middleware for call limits and tracing.
- It opens a second surface, `ToolScoutMiddleware`, for LangChain agents, and makes the LangChain-native options (`langgraph-bigtool`, `LLMToolSelectorMiddleware`, provider tool search) easy to run as baselines.
- My earlier search engine's BM25 code is already Python, so it's reused rather than ported.

**What we give up:** a single static binary (now `uvx toolscout`), and some speed on the lexical path. Latency targets in [DESIGN.md](DESIGN.md) were adjusted for Python.

**Revisit if:** `langchain.mcp` changes in ways that break us more than once during v0.1 (fall back to FastMCP's `Client` directly), or lexical p95 misses its budget by a wide margin after profiling.

### D2: A two-tool surface by default (`search_tools` + `call_tool`)
*Sep 28, 2026*

- It works with every MCP client, because it doesn't depend on how a client handles `notifications/tools/list_changed`.
- The agent's context holds two small tool definitions instead of hundreds.
- Dynamic exposure (adding found tools to the tool list) stays a Phase 2 experiment on the MCP surface. On the LangChain surface it's the middleware's `search` mode (see D8).

**Revisit if:** end-to-end runs show agents misuse `call_tool` much more than native tool calls.

### D3: BM25F is the default retriever; hybrid is opt-in
*Sep 28, 2026. Updated Oct 25, 2026.*

- It needs no API keys, no model download and no network access, and it runs in milliseconds.
- Published comparisons show hybrid beating plain BM25 on retrieval accuracy. What's not known is how much of that gap remains end to end when the agent writes its own queries, and how much field weighting and identifier splitting recover.
- BM25F is ToolScout's own `BaseRetriever`, not LangChain's `BM25Retriever`, which scores one concatenated field with no field weights. The stock retriever stays in the benchmark as an ablation, so the value of field weighting is measured rather than assumed.

**Revisit after M3/M4:** if hybrid wins clearly on the test split *and* end to end, make it the default when an embedding model is configured.

### D4: No vector database
*Sep 28, 2026. Updated Oct 25, 2026.*

- Brute-force cosine over 10,000 vectors of 1,024 dimensions is about 10M multiply-adds per query, which takes milliseconds. The float32 vectors take about 40 MB of memory.
- `InMemoryVectorStore` does exactly this. A vector database would add a dependency and a service to run, with no benefit at this scale.

**Revisit if:** catalogs grow past about 100,000 tools.

### D5: An in-memory index, not a disk-backed one
*Sep 28, 2026*

- My earlier search engine needed SPIMI, external merging and compressed postings for 51,480 web pages. A few thousand short tool documents fit in memory.
- What carries over is the ranking design: BM25 with field boosts applied to term frequency before saturation.

### D6: The benchmark lives in the repo and gates CI
*Sep 28, 2026*

- Search quality can silently get worse with changes to tokenization or weights. The gate catches that before merge.
- Every number in the README can be traced to a config and a data version.

### D7: FastMCP for the agent-facing MCP server
*Oct 25, 2026*

- LangChain consumes MCP but doesn't serve it. FastMCP is what `MCPAdapter` is built on, so the client and server sides share one MCP library and one set of protocol behaviors.

**Revisit if:** FastMCP lags the MCP spec on something the server surface needs. The fallback is the official MCP Python SDK.

### D8: LangChain middleware as a second surface
*Oct 25, 2026*

- `wrap_model_call` can replace `request.tools` on each model call, so a `create_agent` agent can hold every upstream tool while the model only sees a few.
- The same retrievers, safety policy and trace format back both surfaces, so results from one apply to the other.
- `retrieve`, `search` and `gateway` modes map directly onto Phase 2's "who writes the query" question.

**Revisit if:** keeping both surfaces in sync slows down v0.1. The MCP surface comes first.

### D9: JSONL traces are the source of truth; LangSmith is optional
*Oct 25, 2026*

- Results must be reproducible from files committed to the repo, by anyone, without an account.
- LangSmith tracing still works when its environment variables are set, which helps with debugging runs.

## Open questions

| # | Question | Options | Decide by |
|---|---|---|---|
| Q1 | Namespaced tool-name format | `MCPAdapter`'s `server_tool` (matches LangChain) vs. `server__tool` (unambiguous). Must fit the MCP spec's name rules and common clients | M1 |
| Q2 | Should search results include full input schemas? | Full schemas in results (one fewer round trip, more tokens) vs. a third `describe_tool` call | M4, from token and success data |
| Q3 | Default handling of destructive tools | Hide by default vs. show with a warning (plus `HumanInTheLoopMiddleware` on the LangChain surface) | M3 |
| Q4 | Default embedding model for hybrid | Local through Ollama (no key, slower on CPU) vs. API (fast, needs a key) | M3, from the benchmark |
| Q5 | Proxy resources and prompts too? | Tools only (v0.1) vs. full proxy | After v0.1 |
| Q6 | Default `k` | 5, 10, or adaptive by score gap | M4 |
| Q7 | Gaps in `MCPAdapter` | Does it expose tool annotations, raw `inputSchema`, `list_changed`, and acceptable per-call session cost? If not, read those through FastMCP's client | M1 |
| Q8 | Default middleware query | Latest user message only vs. user message + recent tool results vs. a model-written query | M4 |
| Q9 | Base model for the trained judge ([JUDGE.md](JUDGE.md)) | Which 1–3B open instruct model, and whether it fits the trace length after rendering | Start of M6 |
| Q10 | Hardware for M6 | Local Apple Silicon (fine for 0.5–1.5B LoRA SFT, slow for GRPO) vs. a rented GPU | Start of M6, from the cost of one RL run |
