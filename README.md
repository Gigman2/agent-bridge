# agent-bridge

Cross-CLI session continuity for **Claude Code, Cursor, and Cline**.

When one tool hits its usage limit, you can pick up the work in another without
re-explaining. Session state is captured **automatically** by hooks (zero tokens,
fires even if the model is cut off) and lives in `.agent-context/` inside each project.

## Why

- **No manual handoff needed.** Hooks journal every prompt, edit, command, and the
  model's last words — so the handoff exists *before* you need it.
- **Multi-session aware.** Each tool gets its own session dir; `agent-bridge status`
  shows all live threads so you (or the resuming tool) can pick the right one.
- **Any tool can resume any session.** The state is plain markdown — no vendor lock-in.
- **Cline gets Tier 1 only** (it has no public hook API); Claude Code and Cursor get
  full automatic capture.

## Quick start

```sh
# 1. Wire all three tools (run once)
agent-bridge install

# 2. In any project you work in:
cd /my-project
agent-bridge init

# 3. Start working normally. State is captured automatically.
agent-bridge status          # see live sessions
agent-bridge archive <key>   # finish a session -> .agent-context/log/
```

## What happens

| Trigger | Effect |
|---|---|
| Session starts (Claude Code / Cursor) | A session dir is created and context is injected: your dir, plus any resumable sessions from other tools. |
| Prompt / edit / command | Logged to `journal.md`; index updated. |
| Agent stops | Last assistant text captured to `last_said.md` — usually the plan/next steps. |
| Context compacts | Recorded so a resume still knows what happened. |
| Session ends | Marked ended; per-process mapping cleared. |

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
    │   │   └── last_said.md
    │   ├── cursor-<id>/...
    │   └── cline-<id>/...
    └── log/                     # archived sessions
```

## Commands

```sh
agent-bridge init [path]        # scaffold .agent-context/ + AGENTS.md in a project
agent-bridge status [--json] [--all]
agent-bridge register <tool> <goal...>   # create a session (Cline's entry point)
agent-bridge note [--session K] <text>   # append a journal line
agent-bridge archive [session] [--all-ended]
agent-bridge doctor              # verify wiring across all tools
agent-bridge uninstall           # remove wiring (keeps data)
```

## Resuming

Open any tool in an initialized project and say **"resume"** (or just continue —
the injected context lists resumable sessions). The tool reads `handoff.md` (falling
back to `journal.md` and `last_said.md`), confirms its understanding in 2-3 sentences,
and continues from Next Actions.

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