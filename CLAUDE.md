# CLAUDE.md

## Project Overview

AutoAgent is a meta-agent engineering framework that autonomously improves AI agent harnesses. A meta-agent iteratively modifies the agent harness (`agent.py`), runs benchmarks via Harbor, and keeps or discards changes based on score improvements.

## Tech Stack

- Python 3.12+ (uv-managed)
- OpenAI Agents SDK (`agent.py`) / Claude Agent SDK (`agent-claude.py`)
- Harbor benchmarking framework
- Docker for task isolation

## Setup

```bash
uv sync
docker build -f Dockerfile.base -t autoagent-base .
```

Requires a `.env` file with API keys (OpenAI, Anthropic, etc.).

## Running Benchmarks

```bash
# Single task
rm -rf jobs; mkdir -p jobs && \
uv run harbor run -p tasks/ --task-name "<task-name>" -l 1 -n 1 \
  --agent-import-path agent:AutoAgent -o jobs --job-name latest > run.log 2>&1

# All tasks (n = concurrency)
rm -rf jobs; mkdir -p jobs && \
uv run harbor run -p tasks/ -n 100 \
  --agent-import-path agent:AutoAgent -o jobs --job-name latest > run.log 2>&1
```

There are no pytest tests or linters configured. Evaluation is done through Harbor benchmark scores.

## Key Files

- `agent.py` — Main agent harness (OpenAI Agents SDK, gpt-5). Has an **editable section** (system prompt, model config, tools, orchestration) and a **fixed adapter section** (Harbor integration, ATIF trajectory conversion).
- `agent-claude.py` — Claude Agent SDK variant (Haiku 4.5, Claude Code tools preset).
- `program.md` — Human-written directive that guides the meta-agent's improvement strategy.
- `Dockerfile.base` — Base container image for running agents in isolation.
- `results.tsv` — Experiment log tracking commits, scores, and keep/discard decisions (gitignored).

## Code Conventions

- Async/await throughout (agents SDK is async-native)
- Tools are async functions decorated with `@function_tool`, defined inside `create_tools(environment)`
- Only modify the editable section of `agent.py` (above the "FIXED ADAPTER" boundary)
- Prefer specialized tools over generic shell execution
- No task-specific hacks — changes should generalize across tasks
- Results logged in TSV format: `commit | avg_score | passed | task_scores | cost_usd | status | description`
