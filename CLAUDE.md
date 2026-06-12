# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```sh
# Python setup
pip install -r requirements.txt                          # install deps (anthropic, python-dotenv, pyyaml)
cp .env.example .env                                     # configure ANTHROPIC_API_KEY + MODEL_ID

# Run a specific lesson chapter
python s01_agent_loop/code.py                            # chapter 1: agent loop
python s20_comprehensive/code.py                         # chapter 20: all mechanisms combined

# Run legacy agent
python agents/s_full.py                                  # full reference agent (legacy 12-lesson)

# Run tests
python -m pytest tests -q                                # all Python smoke tests
python -m pytest tests/test_agents_smoke.py -q           # legacy agent compile tests
python -m pytest tests/test_compaction_tool_pairs.py -q  # compaction logic tests (snip, reactive)
python tests/test_todo_write_string_input.py              # unittest-based todo_write tests
python tests/test_s_full_background.py                    # unittest-based background manager tests

# Run web app
cd web && npm install && npm run dev                      # http://localhost:3000
cd web && npm run build                                   # production build
cd web && npx tsc --noEmit                                # TypeScript check

# Environment
# Required: ANTHROPIC_API_KEY, MODEL_ID (see .env.example for provider options)
# Optional: ANTHROPIC_BASE_URL (for Anthropic-compatible providers like DeepSeek, GLM, etc.)
```

## Project Overview

This is a **harness engineering teaching repository** — 20 progressive Python lessons (s01–s20) that build an AI coding agent's operational environment (the "harness") layer by layer around a fixed Anthropic API agent loop. The core idea: *agency comes from model training; the harness gives agency a place to land.*

Each lesson chapter is a standalone folder with a full-narrative README, translations, a runnable `code.py`, and optional SVG diagrams in `images/`.

## Architecture: The Agent Harness Stack

The project teaches one mechanism per chapter, all converging in `s20_comprehensive/code.py`. The invariant across every chapter is the agent loop itself:

```
while stop_reason == "tool_use":
    response = LLM(messages, tools)
    execute tools
    append results (loop back)
```

### 20 Mechanisms (in learning order)

| Phase | Chapters | What it adds |
|---|---|---|
| **Core** | s01–s04 | Agent loop, tool dispatch map, permission pipeline, hook system |
| **Planning** | s05–s08 | Todo write (plan-first), subagent isolation, skill loading, context compaction (4 layers) |
| **Persistence** | s09–s11 | Memory system (select/extract/consolidate), runtime system prompt assembly, error recovery (retry+fallback) |
| **Long-running** | s12–s14 | File-backed task graph with deps, background tasks with notification queue, cron scheduler |
| **Multi-agent** | s15–s18 | MessageBus JSONL mailboxes, team protocols (shutdown handshake, plan approval gate), autonomous idle-loop agents, git worktree isolation |
| **Extend** | s19–s20 | MCP plugin model (late-bound tool discovery), all mechanisms composed in one loop |

### Key Design Patterns

- **Dispatch map**: Tools are registered in a dict (`name → handler`), loop never changes when adding tools
- **Hooks around the loop, not in it**: PreToolUse, PostToolUse, UserPromptSubmit, Stop — extension points outside the loop body
- **Cheap first, expensive last**: Context compaction runs 3 zero-API passes (snip, micro, budget) before the LLM summary pass
- **Disk-backed state**: Tasks (`task_*.json`), mailboxes (`*.jsonl`), cron jobs (`.scheduled_tasks.json`), transcripts all persist to `.`-prefixed dirs
- **Teammate protocol**: Request-reply via request_id — plan approval, shutdown handshake — prevents cross-talk
- **Worktree isolation**: Each task claims a git worktree by ID; file tools transparently resolve `cwd` to the worktree path

### Directory Layout

```
learn-claude-code/
  s01_agent_loop/ ... s20_comprehensive/   # 20 teaching chapters
    code.py           # standalone runnable
    README.md         # full narrative (Chinese source)
    README.en.md      # English translation
    images/           # SVG diagrams
  agents/             # legacy 12-lesson runnable copies (s_full.py, etc.)
  skills/             # skill files with YAML frontmatter, loaded in s07+
  tests/              # Python smoke tests (unittest + pytest)
  web/                # Next.js 16 web platform (renders legacy docs)
  docs/               # legacy 12-lesson markdown (transitional)
  .github/workflows/  # CI: pytest smoke tests + web type-check/build
```

### Important Runtime Conventions

- `.env` config: uses `MODEL_ID` (not `ANTHROPIC_MODEL`), supports `ANTHROPIC_BASE_URL` for provider-agnostic API calls
- `code.py` files import `anthropic.Anthropic` and `dotenv.load_dotenv` at module level — tests mock these
- Tests import chapter code via `importlib.util.spec_from_file_location` with fake module substitution
- All persistent agent state lives in `.tasks/`, `.mailboxes/`, `.worktrees/`, `.transcripts/`, `.memory/` — these are gitignored
