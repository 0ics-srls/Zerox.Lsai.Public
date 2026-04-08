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

## Supported Languages

| Language | LSP Server | Status |
|----------|-----------|--------|
| Python | [ty-lsai](https://pypi.org/project/ty-lsai/) | 9/10 tools |
| Java | jdtls | 9/10 tools |
| TypeScript | typescript-language-server | 7/10 tools |
| JavaScript | typescript-language-server | 8/10 tools |
| C# | Roslyn | 10/10 tools |

## Install

```bash
curl -fsSL https://github.com/0ics-srls/Zerox.Lsai.Public/releases/latest/download/lsai-install.sh | bash
```

Or download manually from [Releases](https://github.com/0ics-srls/Zerox.Lsai.Public/releases).

### Requirements

- [.NET 10 runtime](https://dotnet.microsoft.com/download)
- Python 3.8+ (for ty-lsai — Python LSP)
- Node.js (for typescript-language-server — TS/JS LSP)
- Java (for jdtls — Java LSP)

The installer auto-detects what you have and downloads missing LSP servers.

### Configure

After install, add to your `.mcp.json` (or Claude Code settings):

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

## How it works

```
AI Assistant  ←→  MCP (stdio)  ←→  LSAI Server  ←→  LSP Servers
                                        ↓
                                   ty (Python)
                                   jdtls (Java)
                                   tsserver (TS/JS)
                                   Roslyn (C#)
```

LSAI sits between the AI assistant and language-specific LSP servers. It translates MCP tool calls into LSP requests, manages workspace lifecycle, and returns structured results optimized for AI consumption.

## Issues

Report bugs and feature requests in [Issues](https://github.com/0ics-srls/Zerox.Lsai.Public/issues).

## License

Proprietary — © 0ics s.r.l.s.
