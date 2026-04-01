# Query Lifecycle Deep Dive

This page explains what happens after a user submits one prompt.

---

## Fundamental idea

`query.ts` is the heart of the agent loop.

It repeatedly does:
1. send structured context to model API,
2. stream response,
3. execute requested tools if needed,
4. feed tool results back,
5. continue until final answer.

---

## Component diagram (deeper view)

![Query lifecycle deep dive](./query-lifecycle-deep-dive.png)

---

## Step-by-step walkthrough

### Step 1: Input enters REPL pipeline

`src/screens/REPL.tsx` receives user input and starts query handling.

Input can be:
- normal prompt text,
- slash command path (`/...`) resolved earlier in input processing.

### Step 2: Query loop starts

`src/query.ts` normalizes messages and prepares request context:
- user/system context,
- model choice and token budget,
- tool definitions and permission callbacks.

### Step 3: API streaming

`src/services/api/claude.ts` streams assistant output chunks.

Assistant output may contain:
- text,
- thinking blocks,
- `tool_use` blocks.

### Step 4: Tool turn (if requested)

If `tool_use` appears:
- query loop routes to tool executor/orchestrator,
- tool results are converted into `tool_result` messages,
- loop calls model again with updated context.

### Step 5: Final answer and render

When no more tool calls are needed:
- final assistant response is produced,
- REPL renders output,
- session/transcript utilities store relevant turn state.

---

## Mental model

Think of this as a **conversation engine with pit stops**:
- the model “drives” the plan,
- tools are “pit stops” for fetching/changing real-world context,
- loop resumes driving until destination (final answer).

---

## Key source files

- `src/screens/REPL.tsx`
- `src/query.ts`
- `src/services/api/claude.ts`
- `src/services/tools/toolOrchestration.ts`
- `src/services/tools/toolExecution.ts`
- `src/utils/messageQueueManager.ts`
