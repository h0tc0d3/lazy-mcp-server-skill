# Shell access and recovery

When MCP tools are available in the host, use them. The one-shot CLI commands
connect to a running **HTTP** lazy-mcp daemon; a stdio connection alone does
not provide a CLI endpoint. Discover and describe before calling:

```bash
lazy-mcp list-commands mcpls-lsp
lazy-mcp describe mcpls-lsp workspace_symbol_search
lazy-mcp call mcpls-lsp workspace_symbol_search \
  '{"query":"parse_config","limit":10}'
```

An HTTP daemon started with `--cli` exposes endpoint discovery. Set `--url`
or `LAZY_MCP_URL` for a chosen daemon, and use `--timeout SECONDS` for a
bounded longer call. Keep stdout reserved for MCP messages in stdio mode.

If a server advertises `auth_required`, show its authorization URL to the user
and wait for authorization before retrying. Keep the Context7 secret in its
configured file; do not print its contents during diagnostics. After an
authorized config change, validate JSON and restart or send `SIGHUP` to the
identified proxy process, then repeat discovery and a small functional call.
Do not guess a PID or stop another client's daemon. A config reload does not
change a running daemon's inherited environment.

For a health check, test each supported server on its own: memory statistics,
a Context7 lookup, a known mcpls symbol, and a CocoIndex search with matches.
Report partial failures and prerequisites; tool counts alone are insufficient.
