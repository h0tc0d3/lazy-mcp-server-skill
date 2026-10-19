# lazy-mcp-server-skill

An Agent Skill for using the seven MCP servers behind lazy-mcp. It teaches
discovery, schema inspection, and invocation for code search, documentation,
fetching, files, Git, and persistent memory. For code search, it prioritizes
mcpls and uses cocoindex-code as a fallback.

The skill follows the current server names in
`~/.config/lazy-mcp/servers.json`. The bundled configuration corrects server
descriptions and lets both code servers start in a selected project directory.

## Layout

```text
.
├── SKILL.md             # discovery metadata and server workflows
├── INSTALL.md           # server installation, project setup, and checks
├── README.md            # skill installation and usage
├── servers.json         # configuration with example filesystem roots
└── rust-analyzer.toml   # Rust target and file-scanning settings
```

The [Agent Skills format](https://agentskills.io/specification) lets tools read
the name and description for discovery and load the procedure when needed.
Keep `SKILL.md` with its linked installation guide when copying the skill.

## Install

Clone the repository into your tool's skills directory. For Codex and other tools
that read `~/.agents/skills/`:

```bash
mkdir -p ~/.agents/skills
git clone https://github.com/h0tc0d3/lazy-mcp-server-skill.git ~/.agents/skills/lazy-mcp-server-skill
```

Name the containing directory `lazy-mcp-server-skill`, matching the `name` in
the frontmatter. Keep the linked files beside `SKILL.md` when updating a copy.

Installing the skill does not install or register MCP servers. Follow
[INSTALL.md](INSTALL.md) to install lazy-mcp and mcpls with Cargo, install
cocoindex-code with uv, and prepare the other five services. Review host paths
before applying [servers.json](servers.json); it includes a secret-file
reference, without the API key itself.

## Where each tool looks

Put the skill inside one of these directories so its entry point lands at
`<skills dir>/lazy-mcp-server-skill/SKILL.md`. The tool names link to their
skill documentation.

| Tool | Project skills dir | Personal skills dir |
| --- | --- | --- |
| [Codex](https://developers.openai.com/codex/skills/) | `.agents/skills/` | `~/.agents/skills/` |
| [Claude Code](https://code.claude.com/docs/en/skills) | `.claude/skills/` | `~/.claude/skills/` |
| [OpenCode](https://opencode.ai/docs/skills/) | `.opencode/skills/`, `.agents/skills/`, `.claude/skills/` | `~/.config/opencode/skills/`, `~/.agents/skills/`, `~/.claude/skills/` |
| [Cursor](https://cursor.com/docs/skills) | `.cursor/skills/`, `.agents/skills/` | `~/.cursor/skills/`, `~/.agents/skills/` |
| [Gemini CLI](https://geminicli.com/docs/cli/skills/) | `.gemini/skills/`, `.agents/skills/` | `~/.gemini/skills/`, `~/.agents/skills/` |

Project paths are relative to the project root. `~` means your home directory.
The supplied server launch commands and filesystem paths target Linux; adapt
them before using the server configuration on another platform.

## One copy, several tools

The `.agents/skills/` installation above serves tools that search that path.
To share it with Claude Code, link the same directory:

```bash
mkdir -p "$HOME/.claude/skills"
ln -s "$HOME/.agents/skills/lazy-mcp-server-skill" \
  "$HOME/.claude/skills/lazy-mcp-server-skill"
```

Use an unused destination for the link. If it contains an older copy, move
that copy aside first. Where symlinks are unavailable, copy the directory and
refresh each copy after updates.

## Using it

Tools can match MCP work against the skill's description. To request it
explicitly, name it in your prompt:

```text
Use lazy-mcp-server-skill to find the parser's callers with mcpls.
```

```text
Use lazy-mcp-server-skill to check all configured servers and report failures.
```

Once active, the agent finds the relevant server, lists its commands, inspects
the schemas, and invokes the needed tools. It starts code search with
`mcpls-lsp`. If language support is missing or a symbol query is insufficient,
it uses `cocoindex-code` in the initialized project. A healthy MCP process
still needs a functional tool call before the agent reports it working.

## Confirm it loaded

Ask your agent to list the skills it can see and look for
`lazy-mcp-server-skill`. Then ask it to find a known source symbol. Check that
it reads `SKILL.md`, discovers `mcpls-lsp`, inspects tool support and schemas,
and verifies the result against source. If mcpls cannot answer, it should
explain the fallback and inspect CocoIndex's `search` schema.

## No skill support?

Copy the body of `SKILL.md`, without its YAML frontmatter, into the rules file
your tool reads, such as `AGENTS.md` or `CLAUDE.md`. Keep `INSTALL.md`
accessible and update its link in the copied instructions. Always-loaded rules
use context on every turn instead of loading for MCP tasks alone.

For a single task, attach `SKILL.md` and ask the agent to apply it with the
configured lazy-mcp tools. Follow the checks in [INSTALL.md](INSTALL.md) in
the environment that will run the servers.
