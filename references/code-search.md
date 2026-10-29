# Code search

## Start with mcpls

1. Establish the project root and a source file in that project. Check
   `get_tool_support`, with `file_path` when known, to see active language
   servers and tool coverage. Empty `languages` or `coverage: none` means
   that successful MCP discovery has not made code analysis available.
2. Use `workspace_symbol_search` for names or partial names. For a known
   file, use `get_document_symbols` to locate its declarations.
3. Follow results with `get_definition`, `get_references`, `get_hover`, or
   call/type hierarchy tools after inspecting their schemas. Use absolute
   paths inside a configured workspace root. In the inspected mcpls release,
   tool line and character inputs are **1-based**; follow the live schema.
   Use a returned position or a supported `symbol_name` selector instead of
   guessing offsets. Check degraded-position and truncated-result markers.
4. Read the relevant source around the matches before drawing conclusions.
   Check `get_diagnostics` when the task concerns errors. A pending diagnostic
   result does not establish that a file is clean.
5. If a server is initializing, allow a bounded retry after it becomes ready.
   Inspect `get_server_logs` or `get_server_messages` for failures. Restart
   an affected language server only when the failure warrants it; restarting
   mcpls does not select a different project directory.

The current user configuration defines Rust, Python, TypeScript, Go, C/C++,
and Zig server entries. Their executables must be installed, and their project
markers must match the selected workspace. Language tags do not prove support.
For Rust projects, use the 120-second rust-analyzer startup timeout in
[INSTALL.md](../INSTALL.md) and merge the bundled
[rust-analyzer.toml](../rust-analyzer.toml) into the project's configuration.
It excludes `target` and limits Cargo target checks to reduce scanning work.
Formatting, rename, and code-action tools return edits; review and apply those
edits through the host's file tools within the requested task.

## Project selection

The supplied `servers.json` launches both code servers in
`LAZY_MCP_PROJECT_ROOT`, or in the proxy's launch directory if that variable
is unset. Select an absolute project path before starting the proxy. A shared
daemon keeps that directory across client sessions; a tool call cannot change
its process environment or working directory.

When the existing proxy was launched elsewhere, reconnect a proxy started for
the target project. The installed mcpls user config has `workspace.roots = []`,
so mcpls uses its launch directory. Explicit roots in another config take
precedence. See [INSTALL.md](../INSTALL.md) for startup and configuration
details.

## CocoIndex fallback

Use CocoIndex when mcpls lacks the needed language or capability, a symbol
search does not locate the implementation, or the query describes behavior
without a useful symbol name. Inspect its `search` schema, then invoke:

```json
{
  "server": "cocoindex-code",
  "command_name": "search",
  "parameters": {
    "query": "where failed requests are retried with backoff",
    "limit": 5,
    "refresh_index": true
  }
}
```

CocoIndex needs `ccc init` in the selected project before `ccc mcp` can start.
When initialization is missing, initialize that project with the existing
embedding settings and build the index with `ccc index`, if workspace access
permits. A first setup that needs an embedding provider choice must be resolved
before starting the noninteractive MCP process. Never initialize a home
directory to silence an error from a proxy launched in the wrong place.

Use `languages` and `paths` to narrow results when supported; paginate with
`offset`. Resolve returned relative paths against the selected project root.
Read the matching source and use mcpls to inspect symbols once located. Check
`success`, including when the outer MCP response reports no error. If both
code servers are unavailable, report why and use the host's text search when
available; a text match does not establish symbol references.
