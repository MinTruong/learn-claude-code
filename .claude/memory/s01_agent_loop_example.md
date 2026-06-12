---
name: s01-agent-loop-example
description: Worked examples for S01 agent loop — single tool, multiple tools, and no-tool flows with code-path tracing
metadata:
  type: reference
---

# S01 Agent Loop — 3 Worked Examples (2026-06-12)

Rewritten in S03-style code-path tracing format. Each example traces from model response to function execution to message state.

## Example 1: Single tool — create hello.py
- Loop runs 2 iterations (tool_use → run_bash → end_turn)
- `run_bash("echo 'print(...)' > hello.py")` at line 104
- `subprocess.run(shell=True, capture_output=True, timeout=120)` at line 74-75
- Returns `"(no output)"` — stdout is empty due to shell redirect
- [Full trace](s01_agent_loop/example.md) — lines 69-81, 86-113

## Example 2: Three tools in one response — mkdir + cat + python
- Model returns 3 `tool_use` blocks in 1 response (parallel)
- Loop iterates over all 3, calls `run_bash()` 3 times sequentially
- All 3 `tool_result` items appended at once — single API call for 3 tools
- Model sees all results in next turn → `end_turn`

## Example 3: No tool — text-only response
- `stop_reason = "end_turn"` on first API call → loop returns immediately at line 96-97
- Only 1 API call, no tools executed

## Key difference from S02+
- S01 hardcodes `run_bash(block.input["command"])` — no dispatch map
- Only `bash` tool available (no read_file/write_file/glob)
- No `safe_path()` validation

## How to apply:
- Reference these 3 examples when explaining the agent loop invariant: `while stop_reason == "tool_use"`
- Example 2 is the most important — shows parallel tool execution in the harness
- Compare with S02's dispatch map to show the evolution
