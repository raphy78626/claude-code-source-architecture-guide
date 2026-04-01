# Claude Code Architecture Learning Path

This is a fundamentals-first architecture guide for `claude-code-source`.

If you are new, read in this order:

1. `CLAUDE_CODE_ARCHITECTURE_EASY.md` (big picture)
2. `01-startup-bootstrap-deep-dive.md`
3. `02-query-lifecycle-deep-dive.md`
4. `03-tools-permissions-mcp-deep-dive.md`

---

## Visual map (overview)

![Claude Code architecture overview](./claude-code-architecture.png)

---

## What each child page teaches

- `01-startup-bootstrap-deep-dive.md`: how `claude` starts fast, then loads full runtime safely.
- `02-query-lifecycle-deep-dive.md`: how one prompt becomes streaming output, including tool loops.
- `03-tools-permissions-mcp-deep-dive.md`: how tools are selected, approved, executed, and integrated with MCP.

Each child page includes:
- easy explanations,
- practical mental models,
- one deeper component PNG,
- key files to read in source.
