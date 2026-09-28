# ToolScout: Decisions

A running log of design decisions and the reasons for them. Each entry says what would make us revisit it.

## Decided

### D1: Go, on the official MCP Go SDK
*Sep 28, 2026*

- A single static binary is easy to install next to any MCP client.
- Goroutines suit holding many upstream connections open at once.
- It's the same SDK used in [Wanderly's Go MCP server](https://github.com/axlezhao/Wanderly-AI-Travel-Agent/tree/main/mcp-go).

**Revisit if:** the SDK lacks a client feature ToolScout needs, such as list-changed handling on the client side.

### D2: A two-tool surface by default (`search_tools` + `call_tool`)
*Sep 28, 2026*

- It works with every MCP client, because it doesn't depend on how a client handles `notifications/tools/list_changed`.
- The agent's context holds two small tool definitions instead of hundreds.
- Dynamic exposure (adding found tools to the tool list) stays as a Phase 2 experiment.

**Revisit if:** end-to-end runs show agents misuse `call_tool` much more than native tool calls.

### D3: BM25F is the default retriever; hybrid is opt-in
*Sep 28, 2026*

- It needs no API keys, no model download and no network access, and it runs in milliseconds.
- Published comparisons show hybrid beating plain BM25 on retrieval accuracy. What's not known is how much of that gap remains end to end when the agent writes its own queries, and how much field weighting and identifier splitting recover.

**Revisit after M3/M4:** if hybrid wins clearly on the test split *and* end to end, make it the default when an embedding provider is configured.

### D4: No vector database
*Sep 28, 2026*

- Brute-force cosine over 10,000 vectors of 1,024 dimensions is about 10M multiply-adds per query, which takes milliseconds. The float32 vectors take about 40 MB of memory.
- A vector database would add a dependency and a service to run, with no benefit at this scale.

**Revisit if:** catalogs grow past about 100,000 tools.

### D5: An in-memory index, not a disk-backed one
*Sep 28, 2026*

- My earlier search engine needed SPIMI, external merging and compressed postings for 51,480 web pages. A few thousand short tool documents fit in memory.
- What carries over is the ranking design: BM25 with field boosts applied to term frequency before saturation.

### D6: The benchmark lives in the repo and gates CI
*Sep 28, 2026*

- Search quality can silently get worse with changes to tokenization or weights. The gate catches that before merge.
- Every number in the README can be traced to a config and a data version.

## Open questions

| # | Question | Options | Decide by |
|---|---|---|---|
| Q1 | Namespaced tool-name format | `server__tool`, `server.tool`, `server/tool`. Must fit the MCP spec's name rules and common clients | M1 |
| Q2 | Should search results include full input schemas? | Full schemas in results (one fewer round trip, more tokens) vs. a third `describe_tool` call | M4, from token and success data |
| Q3 | Default handling of destructive tools | Hide by default vs. show with a warning | M3 |
| Q4 | Default embedding model for hybrid | Local (no key, slower on CPU) vs. API (fast, needs a key) | M3, from the benchmark |
| Q5 | Proxy resources and prompts too? | Tools only (v0.1) vs. full proxy | After v0.1 |
| Q6 | Default `k` | 5, 10, or adaptive by score gap | M4 |
