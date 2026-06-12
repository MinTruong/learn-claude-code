---
name: s03-permission-example
description: Worked examples for S03 permission pipeline — auto-allow, hard deny, and user approval flows
metadata:
  type: reference
---

# S01-S03 Worked Examples — Code-path tracing style

## S01 Agent Loop (2026-06-12)
- **File:** `s01_agent_loop/example.md`
- **3 examples:** 1 tool (create file), 3 tools parallel (mkdir+write+run), no-tool (text only)
- **Tracing:** `agent_loop()` dong 85-113, `run_bash()` dong 69-81, stop_reason check dong 96, subprocess.run dong 74-75
- **Key finding:** S01 hardcodes `run_bash(block.input["command"])` — no dispatch map yet

## S02 Tool Dispatch Map (2026-06-12)
- **File:** `s02_tool_use/example.md`
- **3 examples:** parallel (read+glob), sequential (read→edit), mixed (parallel read→sequential write)
- **Tracing:** `TOOL_HANDLERS.get(block.name)` dong 165, `handler(**block.input)` dong 166, `safe_path()` dong 66-70
- **Key finding:** S02 replaced S01's hardcoded run_bash with dispatch map — line 165 is the critical difference

# S03 Permission — 3 Worked Examples

## Example 1: Auto-allow — Safe tool, no user interruption

**Prompt:** *"Đọc file README.md"*

- Tool: `read_file(path="README.md")`
- Gate 1: skip (not bash)
- Gate 2: no rule matches
- Gate 3: not called
- **Runs immediately — 0 delay**

## Example 2: Hard Deny — Blacklisted command blocked

**Prompt:** *"Xóa toàn bộ file"*

- Tool: `bash(command="rm -rf *")`
- Gate 1: `"rm -rf /"` is substring of `"rm -rf *"` → **blocked**
- `run_bash` **never called**
- Model receives `"Permission denied."` instead of real output

## Example 3: User Ask — Writing outside workspace

**Prompt:** *"Ghi file ra C:\temp\output.txt"*

- Tool: `write_file(path="C:\\temp\\output.txt", content="...")`
- Gate 1: skip (not bash)
- Gate 2: Rule "Writing outside workspace" matches
- Gate 3: User prompted `Allow? [y/N]`
  - `y` → tool runs (but `safe_path` still blocks as second layer)
  - `N`/Enter → `"Permission denied."` returned to model

## Key Design Principle

> Permission pipeline is **layer 1**, `safe_path` is **layer 2**.
> Defense in depth — not either/or, but both.

## How to apply:
- Reference these 3 examples when explaining permission pipeline: auto-allow (fast path), hard deny (no execution), user ask (interactive approval)
- The 3-gate pattern (deny list → rule matching → user approval) is reusable for any access control system
