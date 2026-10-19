# Installing the lazy-mcp servers

This guide installs the seven servers in [servers.json](servers.json) and the
lazy-mcp proxy. The file is based on the inspected user configuration, with
home-directory variables and example filesystem roots. Configure those roots
for your system before applying it. Installing the skill itself is covered in
[README.md](README.md).

## Prerequisites

Use a POSIX shell on Linux for the supplied configuration. Install Git,
Node.js with npm/npx, Rust with Cargo, and [uv](https://docs.astral.sh/uv/).
Use current Rust and Python toolchains and a Node.js release supported by the
latest server packages. uv can manage the Python interpreter. Check the
binaries in the environment that launches your MCP client, since GUI
applications may inherit a different `PATH`.

```bash
export PATH="$HOME/.cargo/bin:$HOME/.local/bin:$HOME/go/bin:$PATH"
rustc --version
cargo --version
uv --version
node --version
npm --version
git --version
```

Package installation and first use need network access. `uvx` and `npx -y`
download their configured packages into caches; they need writable cache
directories even when a package has been downloaded before.

## Install the proxy and code servers

Install **lazy-mcp and mcpls through Cargo**, and **cocoindex-code through uv**.
These commands select the latest published releases without version pins:

```bash
cargo install --locked lazy-mcp
cargo install --locked mcpls
uv tool install --upgrade cocoindex-code
```

The uv package provides the `ccc` executable. Check that the launcher can find
it and the Cargo binaries:

```bash
command -v lazy-mcp mcpls ccc uvx npx
lazy-mcp --help
mcpls --version
ccc version
```

Rerun these commands to update the installations, then repeat the functional
checks below. `--locked` uses a Cargo package's dependency lockfile; it does
not pin the installed server release.

The upstream [lazy-mcp installation instructions](https://gitlab.com/gitlab-org/ai/lazy-mcp/-/blob/main/README.md)
and [mcpls installation guide](https://bug-ops.github.io/mcpls/getting-started/installation.html)
describe their packages. The
[CocoIndex Code repository](https://github.com/cocoindex-io/cocoindex-code)
documents the `ccc` command and project setup.

## Install the remaining server packages

The configured `uvx` and `npx` commands install these packages on first use.
Warm their caches before starting an MCP client if you want to separate package
downloads from MCP startup:

```bash
uvx --upgrade gingugu --help
uvx --upgrade mcp-server-fetch --help
uvx --upgrade mcp-server-git --help
npx -y --package @modelcontextprotocol/server-filesystem@latest -- node --version
npx -y --package @upstash/context7-mcp@latest -- node --version
```

These commands confirm package resolution, rather than MCP tool functionality.
Use the checks below to exercise tools. `uvx --upgrade` refreshes the Python
tool environments, and npm's `@latest` selects its latest release tag. Rerun
these commands when updating; an unversioned `uvx` launch can reuse its cache.
The configuration keeps these packages unpinned.

| Server | Package / executable | Runtime |
| --- | --- | --- |
| `memory` | `gingugu` via `uvx` | Python managed by uv |
| `filesystem` | `@modelcontextprotocol/server-filesystem` via `npx` | Node.js |
| `fetch-server` | `mcp-server-fetch` via `uvx` | Python managed by uv |
| `git` | `mcp-server-git` via `uvx` | Python managed by uv, Git |
| `context7` | `@upstash/context7-mcp` via `npx` | Node.js, Context7 API access |
| `cocoindex-code` | `ccc mcp` from `cocoindex-code` | uv tool environment, embedding model |
| `mcpls-lsp` | `mcpls` | Installed language servers |

The memory package is [gingugu](https://pypi.org/project/gingugu/).
The [filesystem](https://github.com/modelcontextprotocol/servers/tree/main/src/filesystem),
[fetch](https://github.com/modelcontextprotocol/servers/tree/main/src/fetch),
and [Git](https://github.com/modelcontextprotocol/servers/tree/main/src/git)
packages are the reference MCP servers. Context7's
[setup instructions](https://github.com/upstash/context7) cover API access.

## Prepare paths and the Context7 secret

Create the memory directory and the private configuration directory:

```bash
install -d -m 700 "$HOME/.gingugu" "$HOME/.config/lazy-mcp/secrets"
```

The filesystem roots are system-specific. `$HOME/go/pkg`, `/usr/include`,
`/usr/lib64/gcc`, `/usr/lib64/clang`, and `/tmp` are examples from the inspected
Linux installation. Go settings, compiler packages, architecture, and the OS
can change their locations. Find the directories you need:

| Purpose | How to find the path |
| --- | --- |
| Go module cache | `go env GOMODCACHE`; do not assume `$HOME/go/pkg` |
| System C/C++ headers | Inspect the search list from `cc -E -v -x c /dev/null` |
| GCC builtin headers | `gcc -print-file-name=include`, if GCC is installed |
| Clang builtin headers | The `include` subdirectory of `clang -print-resource-dir` |
| Temporary files | Use `$TMPDIR` when set; otherwise check the OS's temporary directory |

Edit the filesystem server's shell command in `servers.json` to pass the
chosen absolute paths as separate quoted arguments. Its supplied command is
an example, not a universal allowlist. Each root must exist; validate your
chosen roots before starting the server. For example, replace these paths:

```bash
for dir in /absolute/path/to/module-cache /absolute/path/to/headers /absolute/path/to/scratch; do
  test -d "$dir" || printf 'Missing filesystem root: %s\n' "$dir"
done
```

User-specific path examples use `$USER` in place of a login name, such as
`/home/$USER`; `$HOME` also handles a home directory outside `/home`.
The bundled memory setting uses `${HOME}` because lazy-mcp expands `${VAR}`
in JSON `env` values. Command arrays need a shell launcher to expand `$HOME`
or `$USER`; the filesystem launcher provides one. The secret reference uses
`{file:~/.config/lazy-mcp/secrets/context7-api.key}`, whose `~` lazy-mcp expands.

Do not add broad home-directory access to fix a missing path. The filesystem
server can write within its allowlist where OS permissions permit; the
description is not an access-control setting.

The config expects a Context7 API key in
`~/.config/lazy-mcp/secrets/context7-api.key`. Keep an existing working key.
If the file is missing, obtain a key from Context7 and enter it without putting
it in shell history. In Bash:

```bash
secret_file="$HOME/.config/lazy-mcp/secrets/context7-api.key"
if [ ! -s "$secret_file" ]; then
  (umask 077
    read -r -s -p 'Context7 API key: ' context7_key
    printf '\n'
    printf '%s\n' "$context7_key" > "$secret_file"
    unset context7_key
  )
fi
chmod 600 "$secret_file"
```

The config stores a `{file:~/...}` reference; lazy-mcp reads the key
without requiring it in the JSON. A missing referenced secret can prevent
proxy startup. If Context7 is not being configured, set its `enabled` field
to `false` and report that it is unavailable.

## Configure language servers for mcpls

mcpls provides the MCP bridge; language servers do the analysis. Install the
servers needed by the project's languages:

| Language | Install | Executable |
| --- | --- | --- |
| Rust | `rustup component add rust-analyzer clippy` | `rust-analyzer` |
| Go | `go install golang.org/x/tools/gopls@latest` | `gopls` |
| C/C++ | Distribution LLVM package, such as `sudo dnf install clang-tools-extra` or `sudo apt install clangd` | `clangd` |
| Python | `npm install -g pyright` | `pyright-langserver` |
| TypeScript / JavaScript | `npm install -g typescript-language-server typescript` | `typescript-language-server` |
| Zig | A `zls` release compatible with your Zig compiler | `zls` |

### Install gopls for Go

Install a current [Go toolchain](https://go.dev/doc/install) first. Then install
the latest gopls release and add its actual installation directory to `PATH`:

```bash
go version
go install golang.org/x/tools/gopls@latest
gopls_bin_dir="$(go env GOBIN)"
if [ -z "$gopls_bin_dir" ]; then
  gopls_gopath="$(go env GOPATH)"
  gopls_bin_dir="${gopls_gopath%%:*}/bin"
fi
export PATH="$gopls_bin_dir:$PATH"
command -v gopls
gopls version
```

Go installs executables into `GOBIN` when set, or `bin` under the first
`GOPATH` entry. Persist the resolved directory in your shell's `PATH` and in
the environment that launches the MCP client. gopls also needs `go` on that
`PATH`. See the [official gopls installation guide](https://go.dev/gopls/).

The mcpls example below starts `gopls serve` for a project with `go.mod`.
After an update, restart the affected mcpls language server and check
`get_tool_support` and a known Go symbol.

### Configure mcpls

The inspected user config at `~/.config/mcpls/mcpls.toml` defines Rust, Python,
TypeScript, Go, C/C++, and Zig. Python, TypeScript, and Zig executables were
absent from the verification shell; install them before using those entries.
The TypeScript entry's current patterns cover `.ts` and `.tsx`; add `.js`
and `.jsx` patterns if you want JavaScript routed through that server.

Preserve an existing mcpls config. On a new machine, create
`~/.config/mcpls/mcpls.toml` with entries such as these for the languages you
installed:

```toml
[workspace]
roots = []

[[lsp_servers]]
language_id = "rust"
command = "rust-analyzer"
timeout_seconds = 120
file_patterns = ["**/*.rs"]
[lsp_servers.heuristics]
project_markers = ["Cargo.toml", "rust-toolchain.toml"]

[[lsp_servers]]
language_id = "go"
command = "gopls"
args = ["serve"]
file_patterns = ["**/*.go"]
[lsp_servers.heuristics]
project_markers = ["go.mod"]

[[lsp_servers]]
language_id = "cpp"
command = "clangd"
file_patterns = ["**/*.c", "**/*.h", "**/*.cpp", "**/*.hpp"]
[lsp_servers.heuristics]
project_markers = ["compile_commands.json", "CMakeLists.txt", "Makefile", ".clangd"]
```

With `roots = []`, mcpls analyzes its launch directory. It starts an entry
when its project markers match. A standalone source directory without those
markers needs an appropriate marker or adjusted heuristics. C/C++ projects
often need `compile_commands.json` for correct includes and compiler options.
See the [mcpls configuration guide](https://bug-ops.github.io/mcpls/guide/configuration.html).

### Limit Rust project scanning

Copy the bundled [rust-analyzer.toml](rust-analyzer.toml) into the Rust project
root, beside `Cargo.toml`, or merge it into an existing file:

```toml
[check]
allTargets = false
command = "clippy"
```

These settings limit Cargo's target checks and exclude build output from
rust-analyzer's file scanning. `command = "clippy"` selects Clippy for
diagnostics. The mcpls `timeout_seconds = 120` setting allows up to two minutes
for rust-analyzer's startup handshake; it does not make scanning faster.

TOML places `check.allTargets` under `[check]`: putting that dotted key after
`[cargo]` would produce `cargo.check.allTargets`. The example also sets
`cargo.allTargets` to limit Cargo loading. Current rust-analyzer accepts
`excludeDirs` as an alias for `files.exclude`. See the
[rust-analyzer configuration guide](https://rust-analyzer.github.io/book/configuration.html)
and its [configuration definitions](https://github.com/rust-lang/rust-analyzer/blob/master/crates/rust-analyzer/src/config.rs).

Restart rust-analyzer after changing these settings. Support for configuration
files varies by setting; check the effective configuration if they appear
ignored. For user-wide settings, merge into
`~/.config/rust-analyzer/rust-analyzer.toml` instead of copying into each project.

## Initialize the CocoIndex fallback

Run setup in the project you intend to search:

```bash
cd /absolute/path/to/project
ccc init
ccc index
ccc status
ccc search 'a known behavior implemented in this project'
```

On first setup, `ccc init` prompts for an embedding provider/model and tests
it. The inspected installation uses local `sentence-transformers` embeddings
with `Snowflake/snowflake-arctic-embed-xs`. A fresh local setup may download
the model; a remote provider needs its credentials. Reuse the existing model
choice unless you intend to rebuild the index with a different one.

User settings live in `~/.cocoindex_code/global_settings.yml`; project settings
live in `<project>/.cocoindex_code/settings.yml`. The CLI manages the local
index and daemon. Review the `.gitignore` change that `ccc init` can make.
`ccc mcp` requires initialization and starts indexing in the background.
An MCP handshake alone does not confirm that embeddings and search work.

## Install the improved configuration

From this skill's directory, review `servers.json`, adjust host paths if
needed, and validate its JSON. Back up an existing config before replacing it:

```bash
python3 -m json.tool servers.json >/dev/null
config_dir="$HOME/.config/lazy-mcp"
install -d -m 700 "$config_dir"
if [ -f "$config_dir/servers.json" ]; then
  cp -p "$config_dir/servers.json" \
    "$config_dir/servers.json.bak.$(date +%Y%m%dT%H%M%S)"
fi
install -m 600 servers.json "$config_dir/servers.json"
```

The improved file corrects capability descriptions, gives both code servers a
project-selecting launcher, normalizes command arrays, and sets a 120-second
request timeout for startup/indexing. It keeps the seven server names and
uses home-directory variables. Customize the filesystem example for this
machine. There is no unsupported `cwd` config field: the two code-server
shell launchers change directory before `exec`.

Select the project before starting lazy-mcp:

```bash
export LAZY_MCP_PROJECT_ROOT=/absolute/path/to/project
lazy-mcp --config "$HOME/.config/lazy-mcp/servers.json"
```

This starts a stdio MCP process. An MCP client must connect to its stdin and
stdout. Add a single lazy-mcp entry to the client's MCP settings, using its
expected schema. A common `mcpServers` format is:

```json
{
  "mcpServers": {
    "lazy-mcp": {
      "command": "lazy-mcp",
      "args": ["--config", "~/.config/lazy-mcp/servers.json"],
      "env": {
        "LAZY_MCP_PROJECT_ROOT": "/absolute/path/to/project"
      }
    }
  }
}
```

Ensure the client can find the Cargo binary on `PATH`. lazy-mcp expands the
leading `~` in its `--config` argument. Replace the project placeholder and
match the client's own settings format. When the
variable is absent, both code servers inherit the proxy's launch directory.
Start a separate proxy per project, or reconnect after changing the selected
root. Changing a client environment setting or sending `SIGHUP` does not
change an existing daemon's inherited environment. Restart that daemon to
select a different root, unless you update the launch commands themselves.

For shell CLI access, run an HTTP daemon in a terminal you control:

```bash
LAZY_MCP_PROJECT_ROOT=/absolute/path/to/project \
  lazy-mcp --config "$HOME/.config/lazy-mcp/servers.json" \
  --transport http --host 127.0.0.1 --cli
```

The default port is 8080. Use `--port` for another port if needed. From a
second terminal, `lazy-mcp list-commands memory` should connect to it.
The one-shot CLI does not connect to a stdio-only proxy. For a chosen HTTP
endpoint, supply `--url http://127.0.0.1:PORT/mcp` on each CLI command.

## Verify all seven servers

In an MCP client, call `list_servers`, discover each server with
`list_commands`, and inspect its input schemas with `describe_commands`.
Run each discovery on its own if one failure obscures a batch's results.

With the HTTP CLI, discover each server and stop on a failure:

```bash
for server in memory filesystem fetch-server git context7 mcpls-lsp cocoindex-code; do
  lazy-mcp list-commands "$server" --timeout 150 || break
done
```

Then exercise the following tools through `invoke_command`, or through
`lazy-mcp describe SERVER TOOL` followed by `lazy-mcp call SERVER TOOL JSON`:

| Server | Functional check | Passing result |
| --- | --- | --- |
| `memory` | `memory_stats` with `{}` | Parsed statistics with `ok: true` |
| `filesystem` | `list_allowed_directories`, then write/read a unique file under `/tmp` | Expected roots and exact file content; remove the test file afterward |
| `fetch-server` | `fetch` for a reachable public documentation page | Retrieved content, with no downstream error |
| `git` | `git_status` with an existing repository's `repo_path` | Repository status |
| `context7` | `resolve-library-id`, then `query-docs` for that ID | A library match and documentation |
| `mcpls-lsp` | `get_tool_support`, `get_document_symbols`, and `workspace_symbol_search` for known code | Active language support and the expected symbol |
| `cocoindex-code` | `search` for a behavior in the initialized project | `success: true` and relevant nonempty results |

For example, after discovery and schema inspection:

```bash
lazy-mcp describe mcpls-lsp workspace_symbol_search
lazy-mcp call mcpls-lsp workspace_symbol_search \
  '{"query":"KnownSymbol","limit":5}' --timeout 150
lazy-mcp describe cocoindex-code search
lazy-mcp call cocoindex-code search \
  '{"query":"a known project behavior","limit":3}' --timeout 150
```

Check nested `isError` values and domain results as described in
[SKILL.md](SKILL.md). Report missing executables, authentication failures, and
indexing errors as failures. The default health probe has its own timeout;
repeat discovery after initial setup if a cold start exceeded that probe.
