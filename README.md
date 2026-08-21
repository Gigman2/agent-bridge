# agent-bridge

Cross-CLI session continuity for **Claude Code, Cursor, and Cline**.

When one tool hits its usage limit, you can pick up the work in another without
re-explaining. Session state is captured **automatically** by hooks (zero tokens,
fires even if the model is cut off) and lives in `.agent-context/` inside each project.

## How it works

Three layers, each doing a job the one above cannot.

### Layer 0 — hooks capture everything, for free

Hooks are shell callbacks the CLI fires at lifecycle points. They cost the model
nothing and run even if it is cut off mid-turn. agent-bridge wires them for Claude
Code and Cursor:

| Hook event | What gets recorded |
|---|---|
| `SessionStart` | Session dir created; context injected (your dir + resumable sessions from other tools) |
| `UserPromptSubmit` | The prompt text |
| `PostToolUse` / `afterFileEdit` | Files edited, commands run (with exit codes) |
| `Stop` / `afterAgentResponse` | The agent's last statement → `last_said.md` |
| `PreCompact` | Compaction happened, so a resume knows the gap |
| `SessionEnd` | Marked ended; per-process mapping cleared |

All of this lands in `journal.md` — a plain-text log, one line per event.

**Cline has no public hook API.** It gets Layer 1 only (see below).

### Layer 1 — rules keep the handoff maintained

A global rule is installed into all three tools. It tells every agent, at session
start, to make a **small targeted edit** to its `handoff.md` after each milestone —
updating Current State and Next Actions, re-reading first, keeping it under ~40 lines.
This is the resume payload. It is the only thing the agent has to remember to do.

### Layer 2 — the resume path runs through the agent

This is the design decision that matters most.

`agent-bridge` is **not** the resume mechanism. It is a context feed. When you type
`resume` (or `continue`, `pick up`, `carry on`) in any tool, the `UserPromptSubmit`
hook detects it and injects:

- The **handoff score** (0–16, sufficient ≥ 8) — is the handoff good enough to resume
  from alone?
- What is **missing** (blank template, empty Next Actions, missing sections)
- The **fallback sources** to read: `handoff.md`, `last_said.md`, `journal.md`, and
  project-level material (AGENTS.md, superpowers ledgers, BRIEFS plans/specs, git log)

The agent then **reads and summarizes itself**. It has the conversation; it can be
accurate. agent-bridge does not write a handoff for it, because a mechanical
extraction cannot understand the work the way the agent can.

The CLI commands `agent-bridge resume` and `agent-bridge handoff` still exist — as
**helpers the agent can call** when it wants a mechanical starting point. They are
not the resume mechanism.

### Mid-session activation

If you run `agent-bridge init` while an agent is already working, that agent's
`SessionStart` hook already fired before `.agent-context/` existed — and rules only
load at session start, so they cannot reach it. `init` writes a transient `.activated`
marker; the hook consumes it on the agent's **next** event and injects a nudge to
build the handoff from the conversation so far. `init` also reports running agent
CLIs it detects in the project.

## Quick start

```sh
# 1. Wire all three tools (run once)
agent-bridge install

# 2. In any project you work in:
cd /my-project
agent-bridge init

# 3. Start working. State is captured automatically.
# 4. Hit a limit in one tool, open another, and say "resume".
agent-bridge status          # see live sessions
agent-bridge archive <key>   # finish a session -> .agent-context/log/
```

## Per-project layout

```
my-project/
├── AGENTS.md                    # durable project context (Cursor + Cline read it natively)
├── CLAUDE.md -> AGENTS.md       # symlink: Claude Code reads the same file
└── .agent-context/
    ├── index.json               # registry: id, tool, last_active, branch, last_line
    ├── sessions/
    │   ├── claude-<id>/
    │   │   ├── journal.md       # auto-written by hooks
    │   │   ├── handoff.md       # model-maintained at milestones
    │   │   └── last_said.md     # the agent's last statement
    │   ├── cursor-<id>/...
    │   └── cline-<id>/...
    └── log/                     # archived sessions
```

## Commands

```sh
agent-bridge init [path]              # scaffold .agent-context/ + AGENTS.md
agent-bridge status [--json] [--all]  # show sessions for this project
agent-bridge register <tool> <goal>   # create a session (Cline's entry point)
agent-bridge note [--session K] <txt> # append a journal line
agent-bridge resume [--json] [session]# score handoff sufficiency; list fallbacks
agent-bridge handoff [--json] [session]# synthesize a handoff from recorded material
agent-bridge archive [session] [--all-ended]
agent-bridge doctor                   # verify wiring across all tools
agent-bridge uninstall                # remove wiring (keeps data)
```

## Cline setup

Cline has no public hook API, so it gets Tier 1 (rule-driven) capture:

1. `agent-bridge install` links `~/Documents/Cline/Rules/agent-bridge.md` to the repo rule.
2. In an initialized project, on your first action run:
   `agent-bridge register cline "<one-line goal>"` — it prints your SESSION id and dir.
3. After each step: `agent-bridge note --session <SESSION> "<what you did + result>"`.
4. Keep your session's `handoff.md` updated at milestones.

## Safety

- Hooks are wrapped in try/except and always exit 0 — they can never break the host CLI.
- All writes are atomic (tmp + rename); two tools writing at once can't corrupt the index.
- No secrets or tokens are captured; only prompts, file paths, commands, and the model's
  last message. The journal is plain text in your repo.
- `uninstall` removes only the wiring; project data and `~/.agent-bridge` are left intact.

## Requirements

- Python 3 (stdlib only)
- macOS / Linux
