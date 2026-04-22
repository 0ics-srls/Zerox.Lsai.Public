# LSAI — Language Server AI

Semantic code navigation for AI assistants via MCP protocol.

LSAI gives AI coding assistants (Claude Code, Cursor, etc.) deep understanding of your codebase through LSP-powered semantic analysis — not grep, not embeddings, but real compiler-grade intelligence.

## What it does

| Tool | Description |
|------|-------------|
| `lsai_search` | Find symbols by name across the workspace |
| `lsai_info` | Get symbol details — type, signature, documentation |
| `lsai_outline` | Document structure with names and signatures |
| `lsai_usages` | Find all references to a symbol |
| `lsai_source` | Read a symbol's implementation |
| `lsai_callers` | Who calls this function/method? |
| `lsai_callees` | What does this function/method call? |
| `lsai_hierarchy` | Type inheritance chain |
| `lsai_diagnostics` | Compiler errors and warnings |
| `lsai_rename` | Rename a symbol across the workspace (writes to disk) |

## Supported Languages

| Language | LSP Server | Notes |
|----------|-----------|-------|
| C# | Roslyn (in-process) | Full coverage incl. `deps` and `impact` |
| Python | [ty-lsai](https://pypi.org/project/ty-lsai/) | Rename limited upstream |
| Java | jdtls | Project must be built (`mvn compile` / `gradle build`) |
| TypeScript | typescript-language-server | — |
| JavaScript | typescript-language-server | — |
| PHP | intelephense | `rename` returns `ToolNotSupported` (upstream limit) |
| Rust | rust-analyzer | — |
| Go | gopls | — |
| C / C++ | clangd | — |

Tool-vs-language coverage varies where the upstream LSP doesn't expose a capability (e.g. call hierarchy on PHP); in those cases LSAI returns a clean `N/A` response instead of failing.

---

## Getting Started — 3 Steps

### Step 1: Install (once per machine)

You install LSAI **once**, from any directory. The installer puts everything in `~/.lsai/` so it works for every project on your machine.

**Linux / macOS / WSL:**

```bash
curl -fsSL https://github.com/0ics-srls/Zerox.Lsai.Public/releases/latest/download/install.sh | bash
```

**Windows (PowerShell):**

```powershell
iwr https://github.com/0ics-srls/Zerox.Lsai.Public/releases/latest/download/install.ps1 -OutFile $env:TEMP\lsai-install.ps1
& $env:TEMP\lsai-install.ps1
```

**Requirements** (the installer checks and tells you what's missing):
- [.NET 10 runtime](https://dotnet.microsoft.com/download) — must be on `PATH` (`dotnet --version` works)
- Python 3.8+ (for Python/ty-lsai)
- Node.js (for TypeScript, JavaScript, PHP/intelephense)
- JDK 21+ (for Java/jdtls) — `JAVA_HOME` is auto-resolved from `which java` if not set
- Go toolchain (for Go/gopls), rustup (for Rust/rust-analyzer), clangd (for C/C++) — only if you work in those languages

The installer downloads any missing LSP servers (jdtls, ty-lsai, typescript-language-server, …) into `~/.lsai/servers/` and reports which languages are ready. Your project is never touched.

What gets installed:
```
~/.lsai/
├── run                    ← launcher wrapper (use this in .mcp.json)
├── server/                ← LSAI .NET server
├── servers/               ← LSP servers (jdtls, ty, typescript-language-server)
└── config.json            ← generated discovery info
```

### Step 2: Configure Claude Code (once)

Add LSAI to Claude Code's MCP config. The exact file depends on your platform:

- **User-wide** (all projects): `~/.claude/mcp.json`
- **Per-project**: `.mcp.json` in the project root
- **Cursor**: `~/.cursor/mcp.json`

```json
{
  "mcpServers": {
    "lsai": {
      "command": "~/.lsai/run",
      "args": ["--stdio"]
    }
  }
}
```

That's it. No path to your project here — LSAI figures that out automatically from step 3.

### Step 3: Use — the CWD rule

**LSAI analyzes whatever directory Claude Code is running from.** This is the single most important thing to understand.

```
              cwd when you launch Claude Code
                        │
                        ▼
              ┌───────────────────┐
              │   Claude Code     │
              └─────────┬─────────┘
                        │ spawns ~/.lsai/run (inherits cwd)
                        ▼
              ┌───────────────────┐
              │   LSAI Server     │  ← workspace root = this cwd
              └───────────────────┘
```

**The correct flow:**

```bash
cd /path/to/your/project    # ← workspace is decided HERE
claude                      # (or cursor, etc.)
```

Claude Code inherits this directory as its working directory. When it spawns the LSAI server via `.mcp.json`, the server inherits the same directory and **auto-opens it as the workspace**. No tool calls needed from the AI — by the time the first `lsai_search` arrives, the workspace is already loading.

---

## For AI Assistants

**Your workspace is implicit.** When the LSAI MCP server starts, it automatically opens the directory it was launched from (which is Claude Code's cwd, which is the project the user opened Claude Code in). You do **not** need to call `lsai_workspace_open` in the normal case.

**Standard workflow:**

1. Call `lsai_server` to see available plugins and the auto-opened workspace. You'll see something like:
   ```
   Open workspaces: 1
     my-project-java-1 | Java | /path/to/project | Ready
   ```
2. Use the `workspaceId` from step 1 (e.g. `my-project-java-1`) in all tool calls:
   ```json
   {"name":"lsai_search","arguments":{"query":"Calculator","workspaceId":"my-project-java-1"}}
   ```
3. For Java specifically: the **project must be built** before analysis (e.g. `mvn compile` or `gradle build`). LSAI is an analyzer — it reads build artifacts but does not build anything.

**When to call `lsai_workspace_open`:**

Only when you need to analyze a project **outside** the user's cwd (e.g. a dependency in another directory). LSAI will generate the metadata for that path and open it:

```json
{"name":"lsai_workspace_open","arguments":{"path":"/other/path","language":"Java"}}
```

**What LSAI does NOT do:**

- ❌ Build, compile, or modify the user's project
- ❌ Require manual configuration files in the project
- ❌ Need environment variables set by the user (JAVA_HOME is auto-resolved)

---

## How it works

```
  User cd's to /path/to/project && runs claude
                        │
                        ▼
  Claude Code (cwd = /path/to/project)
                        │
                        │ reads .mcp.json → spawns:
                        ▼
  ~/.lsai/run --stdio   (cwd inherited = /path/to/project)
                        │
                        ▼
  LSAI Server
   ├─ reads cwd = /path/to/project
   ├─ scans files, detects languages
   ├─ writes /path/to/project/.lsai/mjsf.json (internal metadata)
   ├─ starts matching LSP servers (jdtls / ty / tsserver / Roslyn)
   └─ exposes MCP tools (lsai_search, lsai_info, etc.)
                        │
                        ▼
  AI calls lsai_* tools → structured semantic results
```

The `.lsai/` directory is auto-generated inside your project for caching. It's safe to commit or `.gitignore` — LSAI regenerates it as needed.

---

## Troubleshooting

**"No workspace open" when calling tools**
LSAI auto-opens from cwd. If you launched Claude Code outside your project, either:
- Close Claude Code, `cd` into the project, relaunch, OR
- Call `lsai_workspace_open(path="/full/project/path", language="Java")`

**Java fails with "target/classes missing — run mvn compile first"**
LSAI never builds. Run `mvn compile` (or `gradle build`) in the project once, then retry.

**jdtls not found / JAVA_HOME issues**
The installer puts jdtls in `~/.lsai/servers/jdtls/`. `JAVA_HOME` is auto-resolved from `which java` if not in your environment. If resolution fails, set `JAVA_HOME` explicitly.

**LSP server shows NOT INSTALLED despite being there**
Re-run the installer:
- Linux/macOS: `curl -fsSL https://github.com/0ics-srls/Zerox.Lsai.Public/releases/latest/download/install.sh | bash`
- Windows: re-run `install.ps1` from the latest release.

---

## Uninstall

```bash
~/.lsai/run --uninstall
```

Or just `rm -rf ~/.lsai` and remove the `lsai` entry from your `.mcp.json`.

---

## Issues

Report bugs and feature requests in [Issues](https://github.com/0ics-srls/Zerox.Lsai.Public/issues).

## License

Proprietary — © 0ics s.r.l.s.
