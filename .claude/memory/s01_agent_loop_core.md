---
name: s01-agent-loop-core
description: Core agent loop pattern from S01 — while tool_use loop, response structure, tool execution flow
metadata:
  type: reference
---

# S01: Agent Loop — Core Concepts

## The Agent Loop Pattern

The entire secret of an AI coding agent is a simple while loop:

```python
while stop_reason == "tool_use":
    response = LLM(messages, tools)
    execute tools
    append results back to messages
```

**Why:** LLMs can only output text — they can't run commands, see results, or iterate. The agent loop bridges this gap by feeding tool execution results back into the model's context so it can reason about the outcome and decide the next step.

## 3 Key Message Components

| Component | Role | Description |
|---|---|---|
| `response` (from LLM) | Decision | Contains `text` blocks and/or `tool_use` blocks. `stop_reason` tells harness what to do next |
| `messages.append({"role": "assistant", "content": response})` | Memory | Stores the model's output so the conversation history is preserved |
| `messages.append({"role": "user", "content": results})` | Feedback | Feeds tool execution results back to the model as user-role content |

## stop_reason Values

| Value | Meaning | Action |
|---|---|---|
| `"tool_use"` | Model wants to call a tool | Continue loop — execute tools, feed results back |
| `"end_turn"` | Model finished answering | Exit loop — print final text response |
| `"max_tokens"` | Hit token limit | Exit loop (response may be truncated) |

## Trade-off: Token Cost vs Autonomy

- **Without loop (1 API call, ~80 tokens)**: Model only suggests commands — user must manually execute and handle errors.
- **With loop (2+ API calls, ~565 tokens)**: Model executes, sees results, and self-corrects. Costs 2-3x more for simple tasks but is essential for complex multi-step tasks (debugging, refactoring, feature implementation).

**Why the extra cost is worthwhile:** The model doesn't guess — it **reacts to actual outcomes**. A failed command leads to a different next command, not a repeat of the same suggestion.

## Parallel Tool Calls

Claude can generate multiple `tool_use` blocks in a single response (3-5+ concurrent calls). The harness executes all of them, collects results, and sends them back in one follow-up call. This is key for complex workflows like "create a directory, write a file, then run it."

## Example

Full worked example with "Tạo file hello.py in Hello World" showing both API calls, messages state at each step, and token usage: [[s01-agent-loop-example]]

## How to apply:
- The agent loop is the foundation everything else builds on (policy, hooks, lifecycle, permissions).
- Simple tasks cost ~2-3x more tokens with loop, but complex tasks are impossible without it.
- Parallel tool calls reduce the number of loop iterations for multi-step tasks.

**Why:** This is the minimal harness that connects an LLM to the real world. Without it, the model is just a text generator.
