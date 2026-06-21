---
name: lsai
description: "Semantic code navigation via LSAI MCP server. Use this skill whenever the user asks to explore code, find symbols, understand call graphs, trace dependencies, analyze impact of changes, navigate class hierarchies, or inspect code structure — especially in multi-language projects. Also use when you detect mcp__lsai__* tools are available and the user is doing code exploration that would normally require grep/glob. LSAI provides compiler-grade intelligence across 9 languages (C#, Python, TypeScript, JavaScript, Java, PHP, Rust, Go, C/C++) through 14 MCP tools. Prefer LSAI over grep/glob/find for ANY code navigation — it saves 90%+ tokens and returns richer data."
---

# LSAI — Semantic Code Navigation

**Skill version:** `v1.0.189` (matches the LSAI release that installed it — verify with
`lsai_server`; the placeholder is replaced with the real version at publish time).

LSAI is an MCP server that gives you compiler-grade code intelligence across 9 languages. It wraps real LSP servers (Roslyn, ty, tsserver, jdtls, intelephense, rust-analyzer, gopls, clangd) and exposes their capabilities as simple MCP tools.

## First: Check Availability

Before using LSAI, verify it's running:

```
lsai_server()  ->  shows version, plugins, open workspaces
```

If `mcp__lsai__lsai_server` is not available, LSAI is not configured for this session. Fall back to grep/glob.

## The Parasitic Model

LSAI is a **parasite** — it attaches to already-built projects and analyzes them. It never compiles, builds, or modifies project files (except `lsai_rename` which writes to disk).

- Projects MUST be pre-built before LSAI can analyze them (e.g. `mvn compile`, `npm install`, `cargo build`, `dotnet build`)
- LSAI auto-opens the workspace from the directory Claude Code was launched in
- The `.lsai/` directory it creates in your project is internal metadata — safe to gitignore

## Getting Your workspaceId

Every tool call needs a `workspaceId`. Get it once from `lsai_server`:

```
lsai_server()
->  Open workspaces: 1
      my-project-java-1 | Java | /path/to/project | Ready
```

Use `my-project-java-1` in all subsequent calls. If the workspace shows "Loading", wait and retry — LSP servers need warmup time (especially jdtls for Java: 30-60s first time).

For multi-language projects, LSAI auto-opens one workspace per detected language. You'll see multiple IDs — pick the one matching the language you're exploring.

## Readiness: Loading → Indexing → Ready (don't misread "empty")

**How to check:** call `lsai_server` or `lsai_workspace_list` — they report a status per workspace.
The workspace is **auto-opened** from `.lsai/mjsf.json` at startup; you do NOT open it yourself — never
call `lsai_workspace_open` (legacy path; it could block 88-164s on a big project). Just read the status
and wait for it. The status MATTERS — a query against a not-yet-ready workspace is handled honestly,
never with a silent empty result:

- **Loading** — the LSP server is still starting (no client yet). Wait and retry.
- **Indexing(done/total)** — the server is alive and building its index (e.g. clangd `Indexing(420/1731)`).
  - *Scoped* tools (`outline`/`info`/`diagnostics`/`source`/`usages`) already answer for files the server
    has parsed, but the result is annotated `⚠ workspace still indexing (done/total) — results may be
    incomplete`. Treat an empty/partial result here as "not indexed yet", NOT as "0 matches".
  - *Workspace-wide* tools (`search`/`hierarchy`) return an explicit `WorkspaceNotReady` with the counts
    until the index finishes — poll `lsai_workspace_list`.
- **Ready** — fully indexed. Results are complete.
- **Error / Unusable** — the workspace can't answer (e.g. a compile DB that doesn't cover the root). The
  error carries the reason; do not retry blindly.

**First open is a one-time indexing wait.** On a large native/C++ project the first open can take minutes
(roughly thousands of TUs at a few TU/s — e.g. ~10 min for ~1500 files). Subsequent opens are seconds: the
server reloads its on-disk index cache. Tell the user "first open indexes once, then it's fast" rather than
letting them think it hung. Never decide "X doesn't exist" from a query run while the workspace is Indexing.

## The 14 Tools — When to Use Each

### Discovery (start here)
| Tool | Use when you need to... | Example |
|------|------------------------|---------|
| `lsai_search` | Find a symbol by name | "Where is UserService defined?" |
| `lsai_info` | Get signature, docs, type details | "What does processOrder accept?" |
| `lsai_outline` | See all members of a class/file | "What methods does this class have?" |
| `lsai_source` | Read the implementation code | "Show me the body of validate()" |

### Relationships (understand connections)
| Tool | Use when you need to... | Example |
|------|------------------------|---------|
| `lsai_usages` | Find all references to a symbol | "Who uses this constant?" |
| `lsai_callers` | See who calls a method | "What calls processOrder?" |
| `lsai_callees` | See what a method calls | "What does initialize() do internally?" |
| `lsai_hierarchy` | See inheritance chain | "What implements IRepository?" |
| `lsai_file_refs` | Cross-file reference map | "What files depend on config.ts?" |

### Analysis (assess risk and structure)
| Tool | Use when you need to... | Example |
|------|------------------------|---------|
| `lsai_impact` | Assess change risk | "Is it safe to modify this method?" |
| `lsai_deps` | File-level imports/dependencies | "What does this file import?" |
| `lsai_context` | Composite overview (outline+diag+usages+callers+risk) | "Give me full context for this file" |
| `lsai_diagnostics` | Compiler errors/warnings | "Are there any errors in the project?" |

### Mutation (changes disk)
| Tool | Use when you need to... | Example |
|------|------------------------|---------|
| `lsai_rename` | Rename a symbol everywhere | "Rename getUserById to findUser" |

## Efficient Exploration Pattern

When exploring unfamiliar code, follow this sequence:

```
1. lsai_search("UserService")        -> find it, get location
2. lsai_info("UserService")          -> signature, docs, member count
3. lsai_outline("UserService")       -> list all methods
4. lsai_usages("processOrder")       -> who references it?
5. lsai_callers("processOrder")      -> call graph: who calls it?
6. lsai_callees("processOrder")      -> what does it call?
7. lsai_hierarchy("BaseService")     -> inheritance chain
8. lsai_impact("processOrder")       -> risk if we change it
```

You don't always need all 8 steps. For a quick lookup, steps 1-2 suffice. For refactoring assessment, steps 4-8 are essential.

## Querying Effectively — Scope Your Searches

> **INVARIANT — scope EVERY query. This is a per-query rule, not a one-time gate.** A lookup you scoped
> a minute ago does NOT make the next one safe. Re-scope every time. (Skipping this after the first
> query is the #1 way to get fooled by dependency/system noise.)

**The index covers the WHOLE build graph, not just your code.** LSAI indexes everything the language
server sees — your own code PLUS vendored dependencies (third-party libs pulled into the build) and
system headers. On a large native project that can be hundreds of entries in `.lsai/mjsf.json`. So a
`lsai_search` for a common term (`Result`, `Mesh`, `expected`) is DOMINATED by dependency/system noise
BY DESIGN — not a bug. Scope it: by `projectName=` (the language-server project for your own module),
or by keeping only your own-code paths and discarding vendored / build / system paths in the result.

**Pick the tool that matches the question:**

| Question | Use |
|----------|-----|
| "Does X exist? signature / members?" | `lsai_info` / `lsai_outline` on the known name — precise, zero noise |
| "Who uses / calls a REAL symbol?" | `lsai_usages` / `lsai_callers` |
| Fuzzy discovery | `lsai_search` — always scoped |
| (C/C++) macro uses, non-code files (CMake/JSON/YAML) | grep |
| Third-party library API | xmp4 (library index), if available |

**Negative-verdict discipline:** "X is absent / 0 uses" is trustworthy ONLY from a precise `info`/`outline`
query or a path-scoped `search`. NEVER conclude "absent" from an unscoped search (or a macro lookup —
see Known Limitations). **Division of labor:** LSAI = your own code · xmp4 = third-party libs · grep =
non-code files + macros.

Project-specific scoping (your own-code roots, vendored exclude globs, project-name prefix) belongs in
your project's settings, not in this skill.

## Token Efficiency

LSAI saves 90%+ tokens compared to grep/glob. The data is structured and compact:

```
lsai_search("User")
->  models.py:14 class User
    services.py:7 class UserService
    main.py:7 function main

vs. grep -rn "class User" (returns full lines, duplicates, noise from node_modules...)
```

**Rules:**
- NEVER use grep/glob/find for symbol lookup when LSAI is available
- Use `lsai_source` instead of `Read` when you need a method body (returns just that method, not the whole file)
- Use `lsai_outline` instead of reading a file to understand its structure
- Use `lsai_usages` instead of grep to find references

## Live Editing Awareness

LSAI tracks file changes in real-time via file system watchers. When you edit a file with the Edit/Write tools:

- LSAI detects the change automatically
- The next tool call reflects the updated state
- No manual refresh or workspace reopen needed

This means you can: edit a file -> immediately search for the new symbol -> LSAI finds it.

## Multi-Language Workspaces

LSAI detects all languages in your project automatically. A monorepo with Java backend + TypeScript frontend gets two workspaces:

```
lsai_server()
->  my-app-java-1     | Java       | /path/to/app | Ready
    my-app-typescript-2 | TypeScript | /path/to/app | Ready
```

Use the Java workspace ID for backend queries, TypeScript for frontend.

## Known Limitations

- **Java**: project must be built first (`mvn compile` / `gradle build`). jdtls cold-start takes 30-60s.
- **PHP rename**: intelephense free tier doesn't support rename. Tool returns N/A.
- **JavaScript workspace/symbol**: tsserver doesn't support workspace/symbol for CommonJS. Use `lsai_outline` per-file instead.
- **Python hierarchy**: ty/pyright may not populate full supertype chain.
- **Macros are NOT indexed (C/C++)** — *measured*: a macro is a preprocessor construct, expanded before
  the AST exists, so clangd's symbol index doesn't carry it like a declaration. `lsai_search`/`lsai_usages`
  on a `#define` return empty / SymbolNotFound — that is NOT "0 uses". Go-to-definition within a file works;
  count macro uses with grep, not LSAI.
- **Compiler-expanded / generated constructs may be under-indexed (general)**: the same blind spot — code
  that isn't a declaration in the AST until it's expanded/generated — can affect Rust proc-macros / `derive`
  and C# source generators. Treat an empty result on such code as "may be under-indexed", and confirm with
  grep or a build with the generators run. (Only the C/C++ macro case above is field-confirmed; the rest is
  expected — verify on a real Rust/C# repo before relying on it.)
- **C/C++ needs a compile database**: generate `compile_commands.json` (CMake `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON`,
  or Bear for Make). LSAI points clangd at the DB directory at launch (`--compile-commands-dir`, resolved at
  spawn — you'll see it on the clangd *process* command line, not in `.lsai/mjsf.json`, which holds only the
  base args), so a missing or malformed project `.clangd` can't suppress indexing. No DB → clangd falls back
  to generic flags and indexes nothing.
- **C/C++ on Windows**: launch your MCP client from the MSVC developer environment (x64 Native Tools Command
  Prompt) so clangd resolves the MSVC stdlib + Windows SDK — otherwise every file fails ("no type named
  'mutex' in namespace std"). One clangd serves both C and C++ from the same compile database.

## Installation

**For any AI assistant that supports MCP (Claude Code, Cursor, Windsurf, etc.):**

1. Install LSAI (once per machine):
   - Linux/macOS/WSL: `curl -fsSL https://github.com/0ics-srls/Zerox.Lsai.Public/releases/latest/download/install.sh | bash`
   - Windows (PowerShell): `irm https://github.com/0ics-srls/Zerox.Lsai.Public/releases/latest/download/install.ps1 | iex`

2. Add to MCP config:
   ```json
   {"mcpServers":{"lsai":{"command":"~/.lsai/run","args":["--stdio"]}}}
   ```

3. Launch your AI assistant from the project directory (`cd /path/to/project && claude`)

Requirements: .NET 10 runtime + language toolchains for the languages you work with (the installer tells you what's missing).

## Quick Reference Card

```
FIND     lsai_search(query="MyClass")
INSPECT  lsai_info(symbolName="MyClass")
MEMBERS  lsai_outline(typeName="MyClass")  OR  lsai_outline(filePath="src/my.ts")
REFS     lsai_usages(symbolName="myMethod")
CALLERS  lsai_callers(methodName="myMethod")
CALLEES  lsai_callees(methodName="myMethod")
INHERIT  lsai_hierarchy(typeName="MyClass")
RISK     lsai_impact(symbolName="myMethod")
IMPORTS  lsai_deps()
ERRORS   lsai_diagnostics()
CODE     lsai_source(symbolName="myMethod")
FILES    lsai_file_refs(filePath="src/my.ts")
CONTEXT  lsai_context(filePath="src/my.ts")
RENAME   lsai_rename(symbolName="old", newName="new")
```

All tools accept `workspaceId` (required) and optional `filePath` to narrow scope.
