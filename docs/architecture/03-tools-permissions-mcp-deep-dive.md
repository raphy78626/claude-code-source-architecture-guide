# Tools, Permissions & MCP Deep Dive

> **[Back to Learning Path](./README.md)** | **Prev:** [Query Lifecycle](./02-query-lifecycle-deep-dive.md) | **[Back to Overview](./CLAUDE_CODE_ARCHITECTURE_EASY.md)**

This page explains how Claude Code finds, approves, and executes tools -- and how MCP extends the system with external servers.

---

## Table of contents

- [The fundamental idea](#the-fundamental-idea)
- [Visual diagram](#visual-diagram)
- [Step 1: Tool catalog](#step-1-tool-catalog)
- [Step 2: How tools reach the API](#step-2-how-tools-reach-the-api)
- [Step 3: Permission decision](#step-3-permission-decision)
- [Step 4: Tool execution](#step-4-tool-execution)
- [Step 5: MCP integration path](#step-5-mcp-integration-path)
- [Subagents and the AgentTool](#subagents-and-the-agenttool)
- [Mental model](#mental-model)
- [Key source files](#key-source-files)

---

## The fundamental idea

Tool usage follows a strict, security-conscious path:

```
model asks for tool → runtime finds definition → permission engine decides → approved tools execute → results return
```

Every single tool call goes through permission checking. There are no backdoors.

MCP extends this same pipeline by making external server capabilities look like local tools.

---

## Visual diagram

This diagram shows the tool lifecycle from model request to result, including the MCP path:

<p align="center">
  <img src="./tools-permissions-mcp-deep-dive.png" alt="Tools permissions and MCP deep dive diagram" width="100%" />
</p>

> Open the [Excalidraw source](./tools-permissions-mcp-deep-dive.excalidraw) in [excalidraw.com](https://excalidraw.com) to explore interactively.

---

## Step 1: Tool catalog

**File:** [`src/tools.ts`](../../src/tools.ts)

`getAllBaseTools()` returns the complete tool inventory. Here is how the tools are organized:

| Category | Tools | Always available? |
|----------|-------|-------------------|
| **File operations** | `FileRead`, `FileEdit`, `FileWrite`, `Glob`, `Grep`, `NotebookEdit` | Yes |
| **Shell** | `Bash`, `PowerShell` | Bash always; PowerShell on Windows |
| **Web** | `WebFetch`, `WebSearch` | Yes |
| **Planning** | `TodoWrite`, `EnterPlanMode`, `ExitPlanMode` | Yes |
| **Agent** | `AgentTool` (sub-agents), `SkillTool` (model-invokable skills) | Yes |
| **User interaction** | `AskUserQuestion`, `SendMessage` | Yes |
| **MCP bridge** | `MCPTool`, `ListMcpResources`, `ReadMcpResource` | When MCP servers configured |
| **Task management** | `TaskCreate`, `TaskGet`, `TaskUpdate`, `TaskList`, `TaskStop` | Feature-gated |
| **Worktree** | `EnterWorktree`, `ExitWorktree` | Feature-gated |
| **Advanced** | `WebBrowser`, `TerminalCapture`, `ToolSearch`, `Snip` | Feature-gated |

Each tool is a TypeScript object conforming to the [`Tool`](../../src/Tool.ts) type:

```typescript
type Tool = {
  name: string
  description: string
  inputSchema: ZodType          // Zod schema for validation
  inputJSONSchema?: JSONSchema  // JSON Schema for the API
  call(args, context, ...): Promise<ToolResult>
  isEnabled(): boolean
  isConcurrencySafe?: boolean
}
```

---

## Step 2: How tools reach the API

**File:** [`src/utils/api.ts`](../../src/utils/api.ts) (`toolToAPISchema`)

Before each API call, tools are converted to the wire format:

1. `getTools()` filters the catalog -- removes denied tools, applies REPL/simple-mode filtering.
2. `toolToAPISchema()` converts each tool's schema to the Anthropic API format:
   - uses `inputJSONSchema` if provided (MCP tools),
   - otherwise converts Zod to JSON Schema via `zodToJsonSchema()`.
3. Schemas are cached per session for performance.

The model sees tool names and descriptions, then decides which to call based on the user's request.

---

## Step 3: Permission decision

**File:** [`src/utils/permissions/permissions.ts`](../../src/utils/permissions/permissions.ts)

When the model emits a `tool_use` block, the permission engine evaluates:

### Rule sources (checked in order)

| Source | Example | Persistence |
|--------|---------|-------------|
| CLI args | `--allowedTools Bash` | Session only |
| Policy settings | Enterprise-managed deny lists | Server-managed |
| Project settings | `.claude/settings.json` allow rules | Per-project |
| User settings | `~/.claude/settings.json` preferences | Per-user |
| Session grants | "Allow once" / "Allow for session" | Session only |

### Decision outcomes

| Outcome | What happens |
|---------|-------------|
| **allow** | Tool executes immediately, no prompt |
| **ask** | User sees a permission prompt; can approve once, for session, or deny |
| **deny** | Tool returns an error to the model with explanation |

### Additional safety layers

- **Sandbox mode** -- Bash/PowerShell can run inside a sandbox (macOS sandbox-exec, Linux containers).
- **Classifier** -- optional ML classifier that evaluates bash commands for safety.
- **Hooks** -- pre-tool-use and post-tool-use hooks can intercept or modify behavior.

---

## Step 4: Tool execution

**File:** [`src/services/tools/toolExecution.ts`](../../src/services/tools/toolExecution.ts)

Once approved, `runToolUse()` handles execution:

1. **Validate input** -- parse against the tool's Zod schema; format validation errors clearly.
2. **Start telemetry span** -- OTel span tracks timing and metadata.
3. **Run pre-hooks** -- plugin hooks that can modify or block execution.
4. **Execute `tool.call()`** -- the actual tool logic runs.
5. **Process result** -- format output, handle large results (truncation, file persistence).
6. **Run post-hooks** -- hooks that can react to results.
7. **Return `tool_result`** -- formatted message goes back to the query loop.

### Concurrency

[`toolOrchestration.ts`](../../src/services/tools/toolOrchestration.ts) decides execution strategy:

- Tools marked `isConcurrencySafe: true` can run in **parallel** (e.g., multiple `FileRead` calls).
- Other tools run **sequentially** to avoid conflicts (e.g., `Bash` commands).

---

## Step 5: MCP integration path

**File:** [`src/services/mcp/client.ts`](../../src/services/mcp/client.ts)

[MCP (Model Context Protocol)](https://modelcontextprotocol.io/) extends Claude Code with external server capabilities.

### How MCP servers connect

MCP supports multiple transport types:

| Transport | When used |
|-----------|----------|
| **stdio** | Local processes (most common for dev tools) |
| **SSE** | HTTP-based servers with server-sent events |
| **Streamable HTTP** | Modern HTTP transport with bidirectional streaming |
| **WebSocket** | Real-time connections |

### How MCP tools become available

1. MCP configs are read from `.claude/settings.json`, project config, and Claude.ai sync.
2. For each server, the client:
   - establishes a connection via the appropriate transport,
   - calls `listTools()` to discover available tools,
   - wraps each in an `MCPTool` object with the same `Tool` interface.
3. These MCP tools are **merged into the main tool catalog** -- the model sees them alongside built-in tools.

### How MCP tool calls work

When the model calls an MCP tool:

1. `MCPTool.call()` serializes the input and sends it to the MCP server.
2. Server processes the request and returns a result.
3. Result is normalized (text, images, binary handled differently).
4. The normalized result flows back through the same `tool_result` path as built-in tools.

### MCP auth

Some MCP servers require authentication:
- `McpAuthTool` handles OAuth flows,
- `ClaudeAuthProvider` manages token refresh,
- Session-expired errors trigger automatic reconnection.

---

## Subagents and the AgentTool

**Files:** [`src/tools/AgentTool/AgentTool.tsx`](../../src/tools/AgentTool/AgentTool.tsx), [`src/tools/AgentTool/runAgent.ts`](../../src/tools/AgentTool/runAgent.ts)

The `AgentTool` lets the model spawn **sub-agents** -- child query loops that run independently:

- Each sub-agent gets its own conversation context and tool set.
- Built-in agents include: general-purpose, explore/plan, and guide agents.
- Custom agents can be defined in `.claude/agents/` directory.
- Sub-agents share the parent's permission context but run with restricted tool access.

This is how features like "Task" tool and forked slash commands work under the hood.

---

## Mental model

Think of the tool system as a factory with a security checkpoint:

| Factory concept | Claude Code equivalent |
|-----------------|----------------------|
| **Parts catalog** | `tools.ts` -- lists every machine that exists |
| **Order form** | Model's `tool_use` block -- "I need machine X with these settings" |
| **Security gate** | `permissions.ts` -- checks if this machine is approved for this job |
| **Machine room** | `toolExecution.ts` -- runs the machine, monitors output |
| **External suppliers** | `mcp/client.ts` -- orders compatible parts from other factories |
| **Quality check** | Post-hooks and telemetry -- verify output before shipping |

Nothing leaves the factory without passing through security. External suppliers follow the same quality standards as internal machines.

---

## Key source files

| File | Role |
|------|------|
| [`src/Tool.ts`](../../src/Tool.ts) | Tool type definition and utilities |
| [`src/tools.ts`](../../src/tools.ts) | Tool catalog (`getAllBaseTools`) |
| [`src/utils/permissions/permissions.ts`](../../src/utils/permissions/permissions.ts) | Permission allow/ask/deny engine |
| [`src/services/tools/toolExecution.ts`](../../src/services/tools/toolExecution.ts) | Single tool runner with hooks + telemetry |
| [`src/services/tools/toolOrchestration.ts`](../../src/services/tools/toolOrchestration.ts) | Parallel/sequential execution strategy |
| [`src/services/tools/StreamingToolExecutor.ts`](../../src/services/tools/StreamingToolExecutor.ts) | Collects tool use blocks from API stream |
| [`src/services/mcp/client.ts`](../../src/services/mcp/client.ts) | MCP server connections and tool calls |
| [`src/services/mcp/config.ts`](../../src/services/mcp/config.ts) | MCP configuration loading and parsing |
| [`src/tools/AgentTool/AgentTool.tsx`](../../src/tools/AgentTool/AgentTool.tsx) | Sub-agent tool definition |
| [`src/commands.ts`](../../src/commands.ts) | Slash command registry (related: SkillTool) |

---

**[Back to Learning Path](./README.md)** | **[Back to Overview](./CLAUDE_CODE_ARCHITECTURE_EASY.md)**
