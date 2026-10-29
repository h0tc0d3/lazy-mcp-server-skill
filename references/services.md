# Documentation and memory

## Library documentation

For `context7`, call `resolve-library-id` with `libraryName` and a focused
`query`, then pass its returned `libraryId` to `query-docs`. Use an explicit
library ID from the user when provided. Follow the live tool's request limits
and include the requested library version when possible. Queries go to an
external API; keep credentials and private source out of them.

## Persistent memory

The `memory` server runs `gingugu` and stores data in
`$HOME/.gingugu/memory.db`. lazy-mcp expands `${HOME}` in the config's `env`
value. It exposes `memory_*` tools, rather than
the entity tools of the reference MCP memory server. Discover its schema;
start with `memory_context`, `memory_recall`, or `memory_stats`.

Use a project namespace to scope `memory_context` and `memory_recall`; request
compact results for broad queries. Store durable facts and decisions when
relevant to the task. Record their source and confidence, and update an
existing memory when correcting it. Avoid storing temporary test output or
credentials in prose memories. Recheck recalled claims against current files.
