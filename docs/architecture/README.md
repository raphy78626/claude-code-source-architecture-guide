# Claude Code Architecture -- Learning Path

> A fundamentals-first guide to understanding the [claude-code-source](https://github.com/AprilNEA/claude-code-source) codebase.
> Written for anyone -- beginner or experienced -- who wants to quickly understand how Claude Code works under the hood.

---

## How to read this guide

Start with the overview, then dive into whichever component interests you.

| # | Page | What you will learn |
|---|------|---------------------|
| 0 | [Architecture Overview](./CLAUDE_CODE_ARCHITECTURE_EASY.md) | The big picture: all 6 layers, end-to-end flow, slash commands, and key files |
| 1 | [Startup & Bootstrap Deep Dive](./01-startup-bootstrap-deep-dive.md) | How `claude` starts fast, then safely loads the full runtime |
| 2 | [Query Lifecycle Deep Dive](./02-query-lifecycle-deep-dive.md) | How one prompt becomes a streaming response, including tool loops |
| 3 | [Tools, Permissions & MCP Deep Dive](./03-tools-permissions-mcp-deep-dive.md) | How tools are found, approved, executed, and how MCP extends them |

---

## Visual overview

This diagram shows the end-to-end flow from user input to assistant response.
Open it in [Excalidraw](https://excalidraw.com) using the [source file](./claude-code-architecture.excalidraw) for interactive editing.

<p align="center">
  <img src="./claude-code-architecture.png" alt="Claude Code architecture overview diagram" width="100%" />
</p>

---

## Quick glossary

| Term | Meaning |
|------|---------|
| **REPL** | Read-Eval-Print Loop -- the interactive terminal UI powered by [Ink](https://github.com/vadimdemedes/ink) |
| **query loop** | The core agent cycle in `query.ts`: send messages to API, get response, run tools if needed, repeat |
| **tool_use / tool_result** | Anthropic API protocol: model asks to use a tool (`tool_use`), runtime returns the output (`tool_result`) |
| **MCP** | Model Context Protocol -- an open standard for connecting AI models to external tools and data sources |
| **bootstrap/state** | A singleton module holding session-wide runtime state (session ID, settings, flags) |
| **slash command** | User commands like `/help`, `/compact`, `/config` that are processed locally before the model sees them |

---

## Repository structure (key paths)

```
src/
├── entrypoints/cli.tsx    # Thin CLI bootstrap (fast path)
├── main.tsx               # Full orchestrator (args, init, mode selection)
├── entrypoints/init.ts    # Global initialization (config, env, telemetry)
├── bootstrap/state.ts     # Process-wide session state
├── replLauncher.tsx       # Mounts Ink REPL UI
├── screens/REPL.tsx       # Interactive REPL surface
├── query.ts               # Core agent loop (stream + tool execution)
├── services/
│   ├── api/claude.ts      # Model API streaming
│   ├── mcp/client.ts      # MCP server connections
│   └── tools/
│       ├── toolExecution.ts     # Individual tool runner
│       └── toolOrchestration.ts # Parallel/batch tool coordination
├── tools.ts               # Tool catalog (getAllBaseTools)
├── commands.ts            # Slash command registry
└── utils/
    ├── permissions/permissions.ts  # Allow / ask / deny engine
    ├── processUserInput/           # Input parsing pipeline
    └── slashCommandParsing.ts      # /command parser
```

---

## Next step

Start reading: **[Architecture Overview](./CLAUDE_CODE_ARCHITECTURE_EASY.md)**
