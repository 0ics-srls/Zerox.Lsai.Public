---
name: lsai
description: "Semantic code navigation via LSAI MCP server. Use this skill whenever the user asks to explore code, find symbols, understand call graphs, trace dependencies, analyze impact of changes, navigate class hierarchies, or inspect code structure — especially in multi-language projects. Also use when you detect mcp__lsai__* tools are available and the user is doing code exploration that would normally require grep/glob. LSAI provides compiler-grade intelligence across 9 languages (C#, Python, TypeScript, JavaScript, Java, PHP, Rust, Go, C/C++) through 14 MCP tools. Prefer LSAI over grep/glob/find for ANY code navigation — it saves 90%+ tokens and returns richer data."
---

# LSAI — Semantic Code Navigation

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

## Installation

**For any AI assistant that supports MCP (Claude Code, Cursor, Windsurf, etc.):**

1. Install LSAI (once per machine):
   - Linux/macOS/WSL: `curl -fsSL https://github.com/0ics-srls/Zerox.Lsai.Public/releases/latest/download/install.sh | bash`
   - Windows: `iwr https://github.com/0ics-srls/Zerox.Lsai.Public/releases/latest/download/install.ps1 -OutFile $env:TEMP\lsai.ps1; & $env:TEMP\lsai.ps1`

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
