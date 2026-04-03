# Architecture

This document describes the architecture of AutoAgent, a meta-agent engineering
framework that autonomously improves AI agent harnesses through iterative
benchmarking.

## System Overview

AutoAgent implements a hill-climbing optimization loop for agent harnesses:

```
┌──────────────────────────────────────────────────────────────────┐
│                        META-AGENT LOOP                           │
│                                                                  │
│   ┌───────────┐    ┌──────────┐    ┌───────────┐    ┌────────┐  │
│   │  Diagnose  │───▶│  Modify  │───▶│  Benchmark │───▶│ Decide │ │
│   │  failures  │    │ agent.py │    │  (Harbor)  │    │keep/   │ │
│   └───────────┘    └──────────┘    └───────────┘    │discard │ │
│         ▲                                            └───┬────┘ │
│         └────────────────────────────────────────────────┘      │
└──────────────────────────────────────────────────────────────────┘
```

A coding agent (the "meta-agent") reads `program.md` for its strategy, modifies
the agent harness (`agent.py`), runs benchmarks via Harbor, evaluates the
results, and repeats. The human programs the meta-agent through `program.md`
rather than editing the harness directly.

## Core Components

### 1. Agent Harness (`agent.py`)

The harness is a single-file design with two clearly demarcated sections:

**Editable Section** (lines 24–73):
- `SYSTEM_PROMPT` — natural-language instructions for the agent
- `MODEL` — model identifier (default: `gpt-5`)
- `MAX_TURNS` — maximum conversation turns before stopping
- `create_tools(environment)` — tool factory; returns a list of `FunctionTool`
  instances decorated with `@function_tool`
- `create_agent(environment)` — constructs the `Agent` object with prompt,
  tools, and optional handoffs/sub-agents
- `run_task(environment, instruction)` — orchestration entry point; creates the
  agent, runs it via `Runner.run()`, and returns the result with timing

**Fixed Adapter Section** (lines 76–242):
- `to_atif()` — converts OpenAI Agents SDK `RunResult` into an ATIF v1.6
  trajectory dictionary
- `AutoAgent` — Harbor `BaseAgent` subclass that wires the agent into Harbor's
  evaluation framework

The boundary is enforced by convention (comments), not code. The meta-agent is
instructed never to modify the fixed section.

### 2. Claude Variant (`agent-claude.py`)

An alternative harness using the Claude Agent SDK instead of OpenAI Agents SDK.

**Editable Section**: Similar config surface — `SYSTEM_PROMPT`, `MODEL`
(default: `haiku`), `TOOLS_PRESET` (uses `claude_code` preset for Bash, file
operations, etc.), `CUSTOM_TOOLS`, `EXTERNAL_MCP_SERVERS`, `SUBAGENTS`, `HOOKS`,
and `get_options()` which assembles a `ClaudeAgentOptions` object.

**Fixed Adapter Section**: Two parts:
1. **Harbor Adapter** — `AutoAgent` class that runs the agent inside a Docker
   container by executing `python agent.py` via `environment.exec()`, then
   reads the trajectory from the container's filesystem.
2. **Container Entrypoint** — `_run_in_container()` reads
   `/task/instruction.md`, creates a `ClaudeSDKClient`, streams the response,
   and writes the ATIF v1.2 trajectory to `/logs/agent/trajectory.json`.

Key architectural difference: the OpenAI variant runs the agent host-side and
proxies shell commands into the container. The Claude variant runs the entire
agent inside the container.

### 3. Meta-Agent Directive (`program.md`)

A Markdown file that serves as the "program" for the meta-agent. It defines:

- **Directive** — what kind of agent to build (general-purpose coding/terminal
  agent)
- **Modification scope** — what the meta-agent may and must not change
- **Tool strategy** — guidance on specialized tools vs. generic shell
- **Keep/discard rules** — strict decision criteria based on `passed` count
- **Experiment loop** — the 10-step iteration process
- **Failure analysis patterns** — taxonomy of common failure modes
- **Overfitting rule** — test to prevent task-specific hacks

This is the human's primary control surface. Instead of editing Python, the
human edits strategy in Markdown.

### 4. Harbor Integration

[Harbor](https://github.com/laude-institute/harbor) is the benchmarking
framework that orchestrates task execution:

```
┌─────────────────────────────────────────────────────┐
│                    Harbor Runner                      │
│                                                      │
│  ┌──────────┐     ┌─────────────────────────────┐   │
│  │ Task Def │     │      Docker Container        │   │
│  │  tasks/  │────▶│  ┌────────┐   ┌──────────┐  │   │
│  │  *.toml  │     │  │ Agent  │   │ Task Env │  │   │
│  └──────────┘     │  │ Code   │   │ (files,  │  │   │
│                   │  └───┬────┘   │  deps)   │  │   │
│                   │      │        └──────────┘  │   │
│                   │      ▼                       │   │
│                   │  ┌──────────┐                │   │
│                   │  │ Verifier │─▶ score 0.0-1.0│   │
│                   │  └──────────┘                │   │
│                   └─────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

**Task format** (each subdirectory under `tasks/`):
- `task.toml` — config (timeouts, metadata)
- `instruction.md` — natural-language prompt sent to the agent
- `tests/test.sh` + `test.py` — verifier that produces a score (0.0–1.0)
- `environment/Dockerfile` — task container (`FROM autoagent-base`)
- `files/` — reference files mounted into the container

**Agent adapter** (`AutoAgent` class):
- Implements Harbor's `BaseAgent` interface
- `setup()` — no-op (no pre-run initialization needed)
- `run()` — uploads the instruction, invokes the agent, saves the ATIF
  trajectory, and reports token usage back to Harbor's `AgentContext`

Harbor runs tasks in parallel (controlled by `-n` flag) in isolated Docker
containers. Each task produces a numeric score that feeds the meta-agent's
keep/discard decision.

### 5. ATIF Trajectory Format

Agent Trajectory Interchange Format — a structured JSON schema for recording
agent execution traces.

```json
{
  "schema_version": "ATIF-v1.6",
  "session_id": "...",
  "agent": { "name": "autoagent", "version": "0.1.0", "model_name": "gpt-5" },
  "steps": [
    {
      "step_id": 1,
      "timestamp": "2026-04-03T...",
      "source": "agent",
      "message": "Tool: run_shell",
      "tool_calls": [{ "tool_call_id": "...", "function_name": "run_shell", "arguments": {...} }],
      "observation": { "results": [{ "source_call_id": "...", "content": "..." }] }
    }
  ],
  "final_metrics": {
    "total_prompt_tokens": 1234,
    "total_completion_tokens": 567,
    "total_cost_usd": null,
    "total_steps": 10,
    "extra": { "duration_ms": 45000, "num_turns": 8 }
  }
}
```

Step types captured:
- **Agent messages** — text output from the model
- **Reasoning** — chain-of-thought / thinking content
- **Tool calls + observations** — tool invocation with arguments and results

The OpenAI variant uses ATIF v1.6; the Claude variant uses ATIF v1.2.

## Data Flow

### OpenAI Variant (`agent.py`)

```
Host Machine                          Docker Container
─────────────                         ────────────────
Harbor Runner
  └─▶ AutoAgent.run()
        ├─ Upload instruction.md ──────▶ /task/instruction.md
        ├─ run_task()
        │    ├─ create_agent()
        │    │    └─ create_tools()
        │    │         └─ run_shell() ──▶ environment.exec() ──▶ shell command
        │    │                          ◀── stdout/stderr ◀──
        │    └─ Runner.run()
        │         └─ (agent loop: prompt ↔ tools × N turns)
        ├─ to_atif() → trajectory.json
        └─ Report metrics to AgentContext
```

The agent runs on the host; only shell commands execute inside the container
via `environment.exec()`.

### Claude Variant (`agent-claude.py`)

```
Host Machine                          Docker Container
─────────────                         ────────────────
Harbor Runner
  └─▶ AutoAgent.run()
        ├─ Upload instruction.md ──────▶ /task/instruction.md
        ├─ environment.exec(               ┌────────────────────┐
        │    "python agent.py") ──────────▶│ _run_in_container() │
        │                                  │   ClaudeSDKClient   │
        │                                  │   ├─ query()        │
        │                                  │   ├─ receive_response()
        │                                  │   └─ trajectory.json│
        │                                  └────────────────────┘
        ├─ Read trajectory.json ◀──────────
        └─ Report metrics to AgentContext
```

The entire agent (SDK client + tool execution) runs inside the container.
The host only launches the process and reads results.

## Docker Architecture

```
┌───────────────────────────────────────┐
│          autoagent-base               │
│  FROM uv:python3.12-bookworm-slim    │
│  ├─ git, ca-certificates             │
│  ├─ Python deps (pyproject.toml)     │
│  ├─ agent.py copied to /app          │
│  ├─ /logs (trajectory output)        │
│  └─ /app/output (agent artifacts)    │
└───────────────────┬───────────────────┘
                    │ FROM autoagent-base
        ┌───────────┴───────────┐
        │    Task Container     │
        │  environment/Dockerfile│
        │  ├─ Task-specific deps│
        │  ├─ files/ mounted    │
        │  └─ /task/instruction │
        └───────────────────────┘
```

Each task builds its own Docker image extending `autoagent-base`. This gives
tasks their own dependencies while sharing the agent code and Python runtime.

## Experiment Tracking

Results are logged to `results.tsv` (gitignored):

| Column | Description |
|--------|-------------|
| `commit` | Short git commit hash |
| `avg_score` | Aggregate benchmark score (0.0–1.0) |
| `passed` | Passed/total (e.g. `20/58`) |
| `task_scores` | Per-task breakdown |
| `cost_usd` | API cost if available |
| `status` | `keep`, `discard`, or `crash` |
| `description` | What was changed |

The meta-agent appends a row after every experiment run. The same commit may
appear multiple times if rerun for variance testing.

## Decision Logic

The meta-agent follows strict keep/discard rules:

```
if passed_improved:
    keep (commit stays)
elif passed_same and harness_simpler:
    keep (simplification win)
else:
    discard (git revert)
```

An additional overfitting guard: every change must pass the test "If this exact
task disappeared, would this still be a worthwhile harness improvement?"

## Key Design Decisions

1. **Single-file harness** — All agent logic lives in one file to minimize
   complexity for the meta-agent. It only needs to understand and edit one file.

2. **Editable/fixed boundary** — The adapter code is protected by convention,
   ensuring the meta-agent can't accidentally break Harbor integration or
   trajectory serialization.

3. **Human programs Markdown, not Python** — `program.md` is the human's
   control surface. Strategy, constraints, and rules are expressed in natural
   language rather than code.

4. **Score-driven hill-climbing** — Simple evolutionary strategy: try a change,
   measure, keep if better. No gradient, no learned optimizer — just
   disciplined iteration.

5. **Docker isolation** — Agents run in containers and cannot damage the host.
   Each task gets its own isolated environment.

6. **Dual SDK support** — Both OpenAI Agents SDK and Claude Agent SDK variants
   exist, with different execution models (host-side vs. in-container) but the
   same Harbor adapter interface.

7. **No test suite** — Correctness is measured through benchmark scores, not
   unit tests. The benchmarks *are* the tests.

## File Dependency Graph

```
program.md ──────────▶ Meta-Agent (external coding agent)
                            │
                            ▼
                        agent.py ◀──── pyproject.toml (deps)
                            │              │
                            ▼              ▼
                       Dockerfile.base ──▶ autoagent-base image
                            │
                            ▼
                    tasks/*/Dockerfile ──▶ task images
                    tasks/*/instruction.md
                    tasks/*/tests/
                            │
                            ▼
                      Harbor Runner
                            │
                            ▼
                    jobs/ (output) + results.tsv (log)
```
