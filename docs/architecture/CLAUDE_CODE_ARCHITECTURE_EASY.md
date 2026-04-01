# Claude Code Source Architecture (Easy Guide)

This guide explains how this codebase works in simple terms.

The short version:
- `claude` starts in a lightweight CLI bootstrap.
- The app initializes config, auth, state, and integrations.
- It launches either an interactive REPL UI (Ink) or a non-interactive print mode.
- User input is parsed (normal prompt or slash command), then sent through the query loop.
- The model may request tools; tools are permission-checked, executed, and results are fed back.
- The loop continues until the assistant reaches a final response.

---

## 1) Big Picture

Think of the system as 6 layers:

1. **Entrypoint layer**: starts fast and routes to the right runtime path.
2. **Bootstrap layer**: loads settings, policies, telemetry, auth, and environment.
3. **Interaction layer**: REPL UI and input handling.
4. **Agent loop layer**: message/query loop with streaming model responses.
5. **Tool layer**: built-in tools, MCP tools, permission gating, execution.
6. **State + observability layer**: session state, transcripts, analytics, telemetry.

```mermaid
flowchart TD
    A["User runs `claude`"] --> B["CLI bootstrap<br/>src/entrypoints/cli.tsx"]
    B --> C["Main orchestration<br/>src/main.tsx"]
    C --> D["Init + bootstrap state<br/>src/entrypoints/init.ts + src/bootstrap/state.ts"]
    D --> E{"Mode?"}
    E -->|Interactive| F["Ink REPL UI<br/>src/replLauncher.tsx + src/screens/REPL.tsx"]
    E -->|Non-interactive / print| G["Headless print path<br/>src/cli/print.ts"]
    F --> H["Query loop<br/>src/query.ts"]
    G --> H
    H --> I["Claude API streaming<br/>src/services/api/claude.ts"]
    I --> J{"Tool requested?"}
    J -->|Yes| K["Permissions + tool execution<br/>src/utils/permissions/permissions.ts + src/services/tools/toolExecution.ts"]
    K --> H
    J -->|No| L["Final assistant response"]
```

---

## 2) Startup Fundamentals

### Step A: Fast bootstrap

`src/entrypoints/cli.tsx` is intentionally small and fast:
- handles quick paths like `--version`,
- handles feature-gated special modes,
- then imports and hands off to `main()`.

This design avoids loading the full app for cheap commands.

### Step B: Main orchestration

`src/main.tsx` is the orchestrator:
- parses CLI options and determines interactive vs non-interactive mode,
- initializes environment and services (`init`),
- sets process/session-level state in `bootstrap/state.ts`,
- loads tools, commands, MCP resources, plugins, and skills,
- launches REPL (interactive) or print/headless flow.

### Step C: Global init and trust-sensitive setup

`src/entrypoints/init.ts` performs global setup (config, safe env, cleanup, telemetry plumbing).

In simple terms: **startup is split into "fast route" and "full setup"** so the app can be both responsive and feature-rich.

---

## 3) Input to Output Lifecycle

This is the core "how Claude Code works" loop.

```mermaid
sequenceDiagram
    participant U as User
    participant REPL as REPL/UI
    participant Input as Input Processor
    participant Q as Query Loop
    participant API as Claude API
    participant PERM as Permission Engine
    participant TOOL as Tool Executor

    U->>REPL: Type prompt or /slash command
    REPL->>Input: processUserInput(...)
    Input->>Q: Normalized messages + context
    Q->>API: queryModel(...) stream request
    API-->>Q: Assistant deltas (text/thinking/tool_use)
    alt Model requested tool_use
        Q->>PERM: canUseTool(...) decision
        PERM-->>Q: allow/ask/deny
        Q->>TOOL: runToolUse/runTools
        TOOL-->>Q: tool_result blocks
        Q->>API: Continue turn with tool results
    end
    API-->>Q: Final assistant response
    Q-->>REPL: Render output
```

Key files in this path:
- UI entry: `src/replLauncher.tsx`, `src/screens/REPL.tsx`
- Input handling: `src/utils/processUserInput/processUserInput.ts`, `src/utils/processUserInput/processSlashCommand.tsx`
- Command parsing: `src/utils/slashCommandParsing.ts`
- Core loop: `src/query.ts`
- API call path: `src/services/api/claude.ts`

---

## 4) Slash Commands (Simple Mental Model)

Slash commands are not magic. They follow a clean pipeline:

1. detect `/...` input,
2. parse command name + args (`parseSlashCommand`),
3. resolve command from registry (`commands.ts`),
4. dispatch by command type:
   - local,
   - local-jsx,
   - prompt/fork.

```mermaid
flowchart LR
    A["Input starts with /"] --> B["parseSlashCommand()<br/>src/utils/slashCommandParsing.ts"]
    B --> C["Command lookup<br/>src/commands.ts"]
    C --> D{"Command type"}
    D -->|local| E["Run local command"]
    D -->|local-jsx| F["Run local JSX command"]
    D -->|prompt| G["Generate prompt/forked flow"]
    E --> H["Messages/Result"]
    F --> H
    G --> H
```

---

## 5) Tools, Permissions, and Safety

### Tool registration

`src/tools.ts` defines the tool catalog (`getAllBaseTools()`):
- file tools (`ReadFile`, `FileEdit`, `Glob`, etc.),
- shell tools (`Bash`, `PowerShell`),
- web tools (`WebFetch`, `WebSearch`),
- planning/task tools,
- MCP resource/tools bridge,
- many feature-gated tools.

### Permission decisions

Before a tool runs, permission logic evaluates:
- allow rules,
- deny rules,
- ask rules,
- mode/classifier/hook constraints.

This logic lives in `src/utils/permissions/permissions.ts`.

### Execution

If approved, execution flows through `src/services/tools/toolExecution.ts` (with hooks and telemetry), and orchestration helpers under `src/services/tools/toolOrchestration.ts`.

```mermaid
flowchart TD
    A["Model emits tool_use"] --> B["Find tool by name"]
    B --> C["Permission evaluation<br/>allow / ask / deny"]
    C -->|deny| D["Return tool_result error"]
    C -->|ask| E["Prompt user / approval path"]
    E --> F{"Approved?"}
    F -->|No| D
    F -->|Yes| G["Execute tool call"]
    C -->|allow| G
    G --> H["Post-hooks + telemetry"]
    H --> I["tool_result returned to query loop"]
```

---

## 6) MCP Integration (Model Context Protocol)

`src/services/mcp/client.ts` manages MCP servers and connections:
- transport types (stdio/SSE/streamable HTTP/websocket),
- listing MCP tools/resources,
- MCP tool call execution + error handling,
- auth flows (including MCP auth tool),
- resource reads and result handling.

Beginner mental model:
- MCP lets Claude Code treat external systems as tools/resources.
- The MCP client converts remote capabilities into local "tool-like" interfaces.

---

## 7) State and Observability Fundamentals

### State
- `src/bootstrap/state.ts`: process/session-wide runtime state.
- `src/state/*`: app/repl state store and transitions.
- `src/utils/sessionStorage.ts`: transcript/session persistence helpers.

### Observability
- `src/services/analytics/index.ts`: product analytics events.
- `src/utils/telemetry/*`: tracing and telemetry internals.

Practical meaning: the app can recover context, track behavior, and debug performance over long sessions.

---

## 8) "One Prompt" Walkthrough (In Plain Words)

When you type one request:
1. UI receives text.
2. Parser checks if it is slash command or normal message.
3. Query loop sends structured messages to the model API.
4. Model streams output.
5. If model needs tools, permission engine decides and tool executor runs approved tools.
6. Tool results are added back to the conversation.
7. Model continues and returns final answer.
8. REPL renders final response and stores session artifacts.

---

## 9) Most Important Files to Read First

If you want to learn this codebase quickly, read in this order:

1. `src/entrypoints/cli.tsx`
2. `src/main.tsx`
3. `src/entrypoints/init.ts`
4. `src/replLauncher.tsx`
5. `src/screens/REPL.tsx`
6. `src/query.ts`
7. `src/services/api/claude.ts`
8. `src/tools.ts`
9. `src/services/tools/toolExecution.ts`
10. `src/utils/permissions/permissions.ts`
11. `src/services/mcp/client.ts`
12. `src/commands.ts`

---

## 10) Why This Architecture Works

In simple terms, this architecture separates concerns clearly:
- startup is optimized for speed,
- UI loop is independent from model API details,
- tool system is modular and permission-gated,
- external integrations (MCP/plugins/skills) are pluggable,
- state and telemetry are centralized.

That separation makes a very complex agent system maintainable in practice.
