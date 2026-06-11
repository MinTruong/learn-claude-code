---
name: s02-tool-use-example
description: Worked examples for S02 tool dispatch — parallel, sequential, and mixed tool execution patterns
metadata:
  type: reference
---

# S02 Tool Use — 3 Worked Examples

## Example 1: Parallel — Independent tools in one response

**Prompt:** *"Đọc file README.md và tìm tất cả file Python"*

Model returns 2 `tool_use` blocks in a single response:
- `read_file(path="README.md")` + `glob(pattern="**/*.py")` — independent, no ordering needed
- Harness dispatches both via `TOOL_HANDLERS[name](**input)`, collects results, sends 1 follow-up
- 1 API call for tool execution, 1 for final response

**Flow:**
```
read_file ──┐
            ├──> gom kết quả → LLM → end_turn
glob ───────┘
```

## Example 2: Sequential — Tool output determines next tool

**Prompt:** *"Đọc code.py, tìm hàm run_bash, sửa timeout 120→60"*

3 API calls, each with 1 tool:
1. `read_file(path="code.py")` → model sees `timeout=120`
2. `edit_file(path="code.py", old_text="timeout=120", new_text="timeout=60")` → confirms edit
3. `end_turn` — done

**Flow:**
```
read_file → LLM → edit_file → LLM → end_turn
```

## Example 3: Mixed — Parallel read then sequential write

**Prompt:** *"Dựa vào 2 ví dụ trên viết file example.md"*

2 API calls:
1. (Parallel) `read_file(ex1)` + `read_file(ex2)` → model learns structure
2. (Sequential) `write_file(path, content)` → model writes new file

## Key Insight

- **Parallel** = model guesses which tools it needs without seeing results first (independent tasks)
- **Sequential** = model must see tool output before deciding next step (dependent tasks)
- **Mixed** = some read tools can be parallelized, then the write depends on what was read

## Comparison Table

| Pattern | Example | Tool calls | API calls | Order matters? | Token cost |
|---|---|---|---|---|---|
| Parallel | read + glob | 2 | 1 | No | ~1,600 |
| Sequential | read → edit | 2 | 2 | Yes | ~1,800 |
| Mixed | 2×read → write | 3 | 2 | Partial | ~2,000 |

## How to apply:
- When reviewing tool dispatch patterns, reference these 3 examples to distinguish parallel vs sequential execution
- Parallel in one response reduces API call count and latency
- Sequential is unavoidable when later tools depend on earlier results
