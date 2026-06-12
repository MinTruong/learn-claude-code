---
name: s02-tool-use-example
description: Worked examples for S02 tool dispatch map — parallel, sequential, and mixed tool execution with code-path tracing
metadata:
  type: reference
---

# S02 Tool Dispatch Map — 3 Worked Examples (2026-06-12)

Rewritten in S03-style code-path tracing format. Each example traces dispatch map lookup, handler execution, and message state.

## Example 1: Parallel — read_file + glob
- 2 independent tools in 1 response
- `TOOL_HANDLERS.get("read_file")` → `run_read` at line 165
- `TOOL_HANDLERS.get("glob")` → `run_glob` at line 165
- Both execute, results collected together → single API call round-trip
- [Full trace](s02_tool_use/example.md) — lines 165-166, 73-80, 105-114

## Example 2: Sequential — read_file → edit_file
- 2 API calls required: cannot edit without knowing content
- Call 1: `run_read(path="code.py")` returns file content with `timeout=120`
- Call 2: `run_edit(path="...", old_text="timeout=120", new_text="timeout=60")` at line 93-102
  - `safe_path()` at line 66-70 validates path
  - `read_text()` then `replace()` then `write_text()`
- Call 3: model confirms → `end_turn`

## Example 3: Mixed — parallel read → sequential write
- Call 1: 2 `read_file` tools in parallel for example.md + README.md
- Call 2: 1 `write_file` tool writes summary based on both readings
  - `run_write()` at line 83-90: `safe_path()` → `mkdir(parents=True)` → `write_text()`
- Call 3: model confirms → `end_turn`

## Key difference from S01
- Line 165: `handler = TOOL_HANDLERS.get(block.name)` replaces S01's hardcoded `run_bash()`
- Line 166: `handler(**block.input)` unpacking — replaces `run_bash(block.input["command"])`
- `safe_path()` at line 66-70 prevents path traversal attacks
- 5 tools instead of 1 (bash, read_file, write_file, edit_file, glob)

## How to apply:
- Reference Example 1 for parallel execution (most efficient token usage)
- Reference Example 2 for demonstrating why sequential flows are necessary
- Reference Example 3 for real-world mixed patterns (read-then-write)
- The dispatch map pattern (TOOL_HANDLERS dict) is reused in every subsequent chapter
