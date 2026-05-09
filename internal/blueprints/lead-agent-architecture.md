# Lead Agent Architecture

A Pydantic AI based personal orchestrator for Mac mini. The lead agent fields requests from you, handles trivial tasks itself, and delegates real work to Claude Code and Codex while supervising them.

## Design principles

1. **The harness is yours.** The framework provides primitives, not opinions. You can read every line of the orchestrator in an evening.
2. **Delegation is a tool call.** Spawning Claude Code or Codex is just another tool the agent can choose, with the same telemetry and approval surface as `web_search`.
3. **Durability is mandatory.** A delegated coding task may run 90 minutes. The orchestrator must survive process restarts, OS reboots, and API failures without losing track of in-flight work.
4. **Model flexibility.** Use cheap models (Haiku, GPT-5 Mini) for routing decisions. Use Opus for hard reasoning. The lead agent is the only place this choice lives.
5. **Observable by default.** Every tool call, every delegation, every model call is traced. You should be able to answer "what did the agent do at 3am last Tuesday?" in seconds.
6. **One process to rule, many to do work.** The lead agent is a single long-lived Python process. Subagents (Claude Code, Codex) are subprocess children with their own working directories.

## High-level architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                      Mac mini (always on)                        │
│                                                                  │
│  ┌────────────────┐    ┌──────────────────────────────────────┐  │
│  │   Messaging    │    │       Lead Agent (Pydantic AI)       │  │
│  │    Gateway     │───▶│                                      │  │
│  │  (Telegram,    │    │  ┌─────────┐    ┌────────────────┐   │  │
│  │   Slack, CLI)  │◀───│  │  Agent  │───▶│ Tools          │   │  │
│  └────────────────┘    │  │  Loop   │    │ ─ web_search   │   │  │
│                        │  │         │    │ ─ shell_exec   │   │  │
│                        │  └─────────┘    │ ─ delegate_*   │   │  │
│                        │       │         │ ─ check_status │   │  │
│                        │       ▼         └────────────────┘   │  │
│                        │  ┌──────────────────────────────┐    │  │
│                        │  │  DBOS durable workflow runtime│   │  │
│                        │  └──────────────────────────────┘    │  │
│                        │       │                              │  │
│                        │       ▼                              │  │
│                        │  ┌──────────────────────────────┐    │  │
│                        │  │  SQLite (state, memory, logs)│    │  │
│                        │  └──────────────────────────────┘    │  │
│                        └──────────────────────────────────────┘  │
│                                       │                          │
│              ┌────────────────────────┼─────────────────────┐    │
│              ▼                        ▼                     ▼    │
│      ┌──────────────┐         ┌──────────────┐      ┌──────────┐ │
│      │ Claude Code  │         │ Claude Code  │      │  Codex   │ │
│      │ subprocess A │         │ subprocess B │      │ subproc. │ │
│      │ (worktree-1) │         │ (worktree-2) │      │ (sandbox)│ │
│      └──────────────┘         └──────────────┘      └──────────┘ │
└──────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
                                  Logfire (cloud)
                              (traces, evals, costs)
```

The lead agent is a single Python process. It owns the conversation state, picks tools, and decides when to delegate. Delegation tools spawn subprocess children in isolated git worktrees, register them in SQLite, and (for long jobs) hand control to a DBOS workflow that polls and reports.

## Project structure

```
lead-agent/
├── pyproject.toml
├── README.md
├── .env                          # API keys (Anthropic, OpenAI, Logfire)
├── config.toml                   # Model choices, paths, defaults
├── lead_agent/
│   ├── __init__.py
│   ├── main.py                   # Entry point. Boots the agent, gateway, DBOS.
│   ├── agent.py                  # Pydantic AI Agent definition + system prompt
│   ├── deps.py                   # RunContext dependencies (db, paths, etc.)
│   ├── tools/
│   │   ├── __init__.py
│   │   ├── basic.py              # web_search, shell_exec, fetch_url
│   │   ├── claude_code.py        # delegate_to_claude_code, status, output
│   │   ├── codex.py              # delegate_to_codex, status, output
│   │   └── delegations.py        # list_delegations, kill_delegation
│   ├── workflows/
│   │   ├── __init__.py
│   │   └── delegation.py         # DBOS @workflow for long-running delegations
│   ├── memory/
│   │   ├── __init__.py
│   │   ├── store.py              # SQLite schema + queries
│   │   └── conversation.py       # Per-thread message history
│   ├── gateway/
│   │   ├── __init__.py
│   │   ├── cli.py                # Local terminal interface
│   │   └── telegram.py           # Telegram bot
│   └── prompts/
│       ├── system.md             # Lead agent system prompt
│       └── delegation_brief.md   # Template for briefing subagents
├── tests/
│   └── ...
└── scripts/
    └── com.you.leadagent.plist   # launchd template for Mac mini
```

## Core components

### The lead agent

The lead agent is a single Pydantic AI `Agent` with a small, carefully chosen tool surface. It does not try to be clever. Its only job is routing, brief-writing, and reporting.

```python
# lead_agent/agent.py
from dataclasses import dataclass
from pathlib import Path
from pydantic_ai import Agent, RunContext
from .deps import LeadDeps
from .tools import basic, claude_code, codex, delegations

SYSTEM_PROMPT = Path(__file__).parent.joinpath("prompts/system.md").read_text()

lead_agent = Agent(
    "anthropic:claude-sonnet-4-6",        # Default; override per-run for cheap routing
    deps_type=LeadDeps,
    instructions=SYSTEM_PROMPT,
)

# Register tools
basic.register(lead_agent)
claude_code.register(lead_agent)
codex.register(lead_agent)
delegations.register(lead_agent)
```

The system prompt is the contract. It describes available tools, when to delegate vs handle directly, how to brief subagents, and how to report results back to you. Keep it under 1500 tokens.

A sketch of the routing logic in the system prompt:

> Handle yourself: questions answerable from one or two tool calls, simple file reads, status checks on existing delegations, anything under ~5 minutes.
>
> Delegate to Claude Code: any task involving editing, refactoring, or creating code in a real repo; multi-file changes; anything that needs to run tests.
>
> Delegate to Codex: tasks where OpenAI's models perform better (specific languages, specific framework knowledge), or when you want a second opinion on a Claude Code result.
>
> When delegating, write a focused brief: goal, constraints, success criteria, what files matter, what to ignore. Do not paste the whole conversation.

### Dependencies via RunContext

Pydantic AI's dependency injection is the right place for shared infrastructure: the SQLite connection, the path to your projects directory, the messaging gateway handle. This keeps tools testable and avoids global state.

```python
# lead_agent/deps.py
from dataclasses import dataclass
from pathlib import Path
import sqlite3

@dataclass
class LeadDeps:
    db: sqlite3.Connection
    workspaces_root: Path     # Where subagent worktrees live
    projects_root: Path       # Where your real projects live
    user_id: str              # Used for memory partitioning
```

### Tool: delegate to Claude Code

This is the heart of the orchestrator. The tool spawns Claude Code as a subprocess in a fresh git worktree, captures its session ID, persists everything, and returns a delegation handle to the lead agent.

```python
# lead_agent/tools/claude_code.py
import asyncio
import json
import uuid
from pathlib import Path
from pydantic import BaseModel, Field
from pydantic_ai import RunContext
from ..deps import LeadDeps

class DelegationHandle(BaseModel):
    delegation_id: str
    agent: str = "claude-code"
    workspace: Path
    status: str = "running"
    notes: str = ""

class ClaudeCodeBrief(BaseModel):
    goal: str = Field(..., description="One sentence: what should be done.")
    repo_path: str = Field(..., description="Absolute path to the repo or project.")
    constraints: list[str] = Field(default_factory=list)
    success_criteria: list[str] = Field(default_factory=list)
    relevant_files: list[str] = Field(
        default_factory=list,
        description="Hints for the subagent. Optional but reduces wasted exploration.",
    )
    create_worktree: bool = Field(
        default=True,
        description="If true, work in a new git worktree to avoid stomping main.",
    )

def register(agent):
    @agent.tool
    async def delegate_to_claude_code(
        ctx: RunContext[LeadDeps],
        brief: ClaudeCodeBrief,
    ) -> DelegationHandle:
        """Hand a coding task to Claude Code. Returns a delegation handle. Poll
        for status and output via check_delegation_status / get_delegation_output.
        Use this for anything involving editing real code in a real repo."""

        delegation_id = f"cc-{uuid.uuid4().hex[:8]}"
        workspace = ctx.deps.workspaces_root / delegation_id
        workspace.mkdir(parents=True, exist_ok=True)

        if brief.create_worktree:
            await _create_worktree(brief.repo_path, workspace, branch=delegation_id)
            cwd = workspace
        else:
            cwd = Path(brief.repo_path)

        # Render the brief into a single prompt for Claude Code
        prompt = _render_brief(brief)

        # Spawn Claude Code in headless / non-interactive mode
        process = await asyncio.create_subprocess_exec(
            "claude",
            "-p", prompt,
            "--output-format", "json",
            "--permission-mode", "acceptEdits",
            cwd=str(cwd),
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
        )

        # Persist the delegation immediately so a crash mid-spawn doesn't lose it
        ctx.deps.db.execute(
            "INSERT INTO delegations (id, agent, workspace, pid, brief, status) "
            "VALUES (?, 'claude-code', ?, ?, ?, 'running')",
            (delegation_id, str(workspace), process.pid, brief.model_dump_json()),
        )
        ctx.deps.db.commit()

        # Hand off to a background DBOS workflow that monitors and persists output
        from ..workflows.delegation import monitor_delegation
        monitor_delegation.start(delegation_id, process.pid)

        return DelegationHandle(
            delegation_id=delegation_id,
            workspace=workspace,
            notes=f"Claude Code spawned, pid={process.pid}",
        )
```

A few details worth flagging:

- **Headless mode.** Use `claude -p "<prompt>"` with `--output-format json` so Claude Code emits structured events to stdout. The orchestrator can stream and persist these.
- **Worktrees, not branches.** Each delegation gets its own git worktree under `~/.lead-agent/workspaces/<id>`. This is how Conductor and similar Mac tools handle parallel coding agents. You can run three Claude Codes against the same repo without merge conflicts.
- **Permission mode.** `acceptEdits` is permissive. For risky repos, use `default` and surface approval requests through the messaging gateway.
- **Persist before yielding.** The row goes into SQLite before the tool returns. If the lead agent crashes between `subprocess_exec` and the next line, the delegation still gets reaped and tracked.

### Tool: delegate to Codex

Same shape, different binary. Codex CLI's App Server protocol gives you JSON-RPC for fine-grained streaming, but for v1 you can just shell out.

```python
# lead_agent/tools/codex.py (sketch)

class CodexBrief(BaseModel):
    goal: str
    repo_path: str
    sandbox: str = Field(default="modal", description="local | modal | e2b | daytona")
    constraints: list[str] = Field(default_factory=list)

@agent.tool
async def delegate_to_codex(
    ctx: RunContext[LeadDeps],
    brief: CodexBrief,
) -> DelegationHandle:
    """Hand a coding task to Codex. Prefer this when GPT-5 family models are
    likely to do better, when you want a second opinion, or when the sandbox
    options Codex supports (Modal, E2B) matter."""
    ...
```

### Tool: check status, get output, kill

These three are how the lead agent supervises its delegations. They're cheap, side-effect-free, and called frequently.

```python
@agent.tool
async def check_delegation_status(
    ctx: RunContext[LeadDeps],
    delegation_id: str,
) -> dict:
    """Returns status (running | completed | failed | killed), runtime, last
    activity timestamp, and a one-line summary. Cheap. Call this freely."""

@agent.tool
async def get_delegation_output(
    ctx: RunContext[LeadDeps],
    delegation_id: str,
    tail_lines: int = 200,
) -> str:
    """Returns the latest output from a delegation. Use tail_lines to control
    how much you pull into your context."""

@agent.tool
async def kill_delegation(
    ctx: RunContext[LeadDeps],
    delegation_id: str,
    reason: str,
) -> dict:
    """Stop a running delegation. Use sparingly; prefer letting them finish."""
```

### Durable execution with DBOS

The job of DBOS in this architecture is narrow but critical: monitor each running delegation in a workflow that survives process restarts. If you reboot the Mac mini at hour two of a three-hour Claude Code run, the workflow picks up where it left off when the process restarts.

```python
# lead_agent/workflows/delegation.py
from dbos import DBOS, WorkflowContext

@DBOS.workflow()
def monitor_delegation(ctx: WorkflowContext, delegation_id: str, pid: int) -> dict:
    """Long-running workflow that polls a subprocess, captures its output,
    persists progress, and reports back to the lead agent on completion.

    Survives crashes because DBOS checkpoints state to Postgres / SQLite
    after every step."""

    while True:
        status = poll_subprocess(pid)            # @DBOS.step
        capture_output_chunk(delegation_id)      # @DBOS.step
        if status in ("completed", "failed", "killed"):
            break
        DBOS.sleep(10)  # Durable sleep. Survives restarts.

    summary = summarize_run(delegation_id)       # @DBOS.step
    notify_user(delegation_id, summary)          # @DBOS.step
    return {"delegation_id": delegation_id, "status": status, "summary": summary}
```

Pydantic AI's docs explicitly call out DBOS, Temporal, and Prefect as supported durable execution adapters. For personal use, DBOS is the lightest option; it can use SQLite as its durable store, so you don't need Postgres.

Why not just rely on Pydantic AI's own agent durability? Because it covers the agent loop (model calls, tool execution) but not unbounded subprocess monitoring. You want the workflow layer for "this delegation will run for an hour or more."

### Memory and state

Two stores, both in the same SQLite file:

**Conversation memory.** Per-thread message history, keyed by gateway and channel. Used by the lead agent to maintain context with you across messages. Implement as a simple table with `(thread_id, ts, role, content)` and feed the last N messages into each agent run via `message_history`.

**Delegation state.** The source of truth for "what is the agent currently working on." Schema:

```sql
CREATE TABLE delegations (
    id TEXT PRIMARY KEY,
    agent TEXT NOT NULL,            -- 'claude-code' | 'codex'
    workspace TEXT NOT NULL,
    pid INTEGER,
    brief TEXT NOT NULL,             -- JSON
    status TEXT NOT NULL,            -- running | completed | failed | killed
    started_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ended_at TIMESTAMP,
    parent_thread_id TEXT,           -- The conversation that spawned this
    summary TEXT
);

CREATE TABLE delegation_output (
    delegation_id TEXT,
    ts TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    stream TEXT,                     -- stdout | stderr | event
    chunk TEXT,
    FOREIGN KEY (delegation_id) REFERENCES delegations(id)
);
```

For project-specific knowledge (preferences, conventions, "this repo uses pnpm not npm"), start with a flat markdown file per project loaded into the system prompt at run time. Avoid a vector database until you have a concrete reason for one. Hermes does FTS5 full-text search across past conversations, which is a reasonable next step if simple recall isn't enough.

## Messaging gateway

You want to talk to this agent from your phone. Two reasonable paths:

**Path A (simplest): Telegram bot.** Run a tiny `aiogram` or `python-telegram-bot` process in the same Python service. On message, look up the thread, append to conversation memory, run the agent, stream the response back. About 100 lines of code.

**Path B (more flexible): MCP messaging servers.** Hermes-style. Run third-party MCP servers for Telegram, Slack, Discord, etc., and let the lead agent treat them as tools (`send_telegram_message`, `read_slack_dms`). More moving parts, but you don't write or maintain the integrations.

Start with Path A. Add Path B if you outgrow it.

```python
# lead_agent/gateway/telegram.py (sketch)
async def on_message(msg):
    thread_id = f"tg-{msg.chat.id}"
    history = load_history(thread_id, limit=20)

    result = await lead_agent.run(
        msg.text,
        deps=deps,
        message_history=history,
    )

    save_message(thread_id, "user", msg.text)
    save_message(thread_id, "assistant", result.output)
    await msg.reply(result.output)
```

## Configuration

Use a single `config.toml` plus environment variables. No magic, no startup wizards.

```toml
# config.toml
[models]
default = "anthropic:claude-sonnet-4-6"
router = "anthropic:claude-haiku-4-5"      # Cheap model for trivial routing
heavy  = "anthropic:claude-opus-4-7"       # Hard reasoning, escalations

[paths]
workspaces_root = "~/.lead-agent/workspaces"
projects_root   = "~/code"
db_path         = "~/.lead-agent/state.db"

[agents.claude_code]
binary = "claude"
default_permission_mode = "acceptEdits"
worktree_base_branch = "main"

[agents.codex]
binary = "codex"
default_sandbox = "modal"

[gateway]
telegram_enabled = true
cli_enabled = true

[observability]
logfire_enabled = true
```

API keys (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `LOGFIRE_TOKEN`, `TELEGRAM_BOT_TOKEN`) live in `.env` and never in `config.toml`.

## Observability

Pydantic Logfire is the path of least resistance. Two lines in `main.py`:

```python
import logfire
logfire.configure()
logfire.instrument_pydantic_ai()
```

You get a web UI showing every agent run, every tool call, every model request, token costs, and latency. Worth the integration even before production. If you don't want a hosted backend, Pydantic AI's instrumentation is OpenTelemetry-native, so any OTel collector works.

For delegations specifically, log a custom span per delegation lifecycle so you can answer "how long did Claude Code take on the auth refactor?" without digging through SQLite.

## Deployment on Mac mini

The lead agent is one long-lived Python process plus a couple of subprocess gateways. `launchd` is the right tool. Skip `tmux` or `screen`; they don't survive reboots cleanly.

```xml
<!-- ~/Library/LaunchAgents/com.you.leadagent.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.you.leadagent</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/you/.lead-agent/.venv/bin/python</string>
        <string>-m</string>
        <string>lead_agent.main</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/Users/you/.lead-agent</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/usr/local/bin:/usr/bin:/bin:/Users/you/.lead-agent/.venv/bin</string>
    </dict>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <dict>
        <key>SuccessfulExit</key>
        <false/>
    </dict>
    <key>StandardOutPath</key>
    <string>/Users/you/.lead-agent/logs/stdout.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/you/.lead-agent/logs/stderr.log</string>
</dict>
</plist>
```

Load with `launchctl load ~/Library/LaunchAgents/com.you.leadagent.plist`. The `KeepAlive` block restarts the process if it crashes, and `RunAtLoad` starts it when you log in.

A few Mac mini specifics worth getting right:

- **Disable App Nap** for the venv's Python, or long-running runs may throttle when the user session is inactive.
- **Caffeinate when delegations are active.** A small helper that runs `caffeinate -i` while any delegation is in `running` state prevents the machine from sleeping mid-task.
- **Filesystem.** Worktrees on the SSD, not on iCloud Drive. Git worktrees in iCloud are a recipe for corrupted repos.

## Iteration roadmap

A realistic build order:

1. **v0 (one weekend).** Pydantic AI agent with `web_search`, `shell_exec`, `delegate_to_claude_code`. CLI gateway only. SQLite for delegation state. Synchronous delegation with a hard timeout. No DBOS yet. Goal: prove the loop works.

2. **v1 (next weekend).** Add `delegate_to_codex`, `check_delegation_status`, `get_delegation_output`, `kill_delegation`. Add DBOS-backed `monitor_delegation` workflow. Add Logfire. Goal: 90-minute delegations work and survive a `kill -9` of the lead agent.

3. **v2.** Telegram gateway. Per-project memory files. Multi-delegation summaries. Approval flows for risky tool calls (Pydantic AI supports tool approval natively).

4. **v3+.** MCP messaging servers, calendar awareness, scheduled tasks, evals (Pydantic Evals) for the routing decisions, second opinions across delegations.

Resist building v3 features into v0. The harness is small on purpose.

## Why this works for your case

- The lead agent stays small (~500 lines including tools). You can read and modify it.
- Pydantic AI gives you the agent loop, structured tools, dependency injection, durable execution adapters, observability, and model flexibility for free.
- DBOS handles the "long-running task survives restarts" problem that no agent SDK solves cleanly on its own.
- Claude Code and Codex stay in the roles they're best at (coding subagents) and never have to be your orchestrator.
- The whole thing runs on a Mac mini, costs only the API tokens you actually use, and you own every line.
