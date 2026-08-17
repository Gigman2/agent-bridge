# agent-bridge — cross-tool session continuity

This project uses **agent-bridge**: session state lives in `.agent-context/` so work can
continue across Claude Code, Cursor, and Cline when one tool hits its usage limit.

These instructions apply ONLY when `.agent-context/` exists at the project root (or an
`[agent-bridge]` block was injected at session start). Otherwise ignore this file entirely.

## Your session

- **Claude Code / Cursor**: hooks have already created your session dir
  (`.agent-context/sessions/<tool>-<id>/`). If an `[agent-bridge]` block was injected at
  session start, it names your dir — follow it. If unsure, run `agent-bridge status --json`.
- **Cline (no hooks)**: on your FIRST action in an initialized project, run:
  `agent-bridge register cline "<one-line goal>"` — it prints your SESSION id and dir.
  If that command fails, the project is not initialized: proceed normally, ignore these rules.

## During work (all tools)

- After each milestone (a working change, a test result, a decision), make a SMALL edit to
  your session's `handoff.md`: update **Current State** and **Next Actions**. Keep the file
  under ~40 lines. Always re-read it before editing. Never include secrets or tokens.
- This is what lets another tool resume after an abrupt limit cutoff — the hook-captured
  `journal.md` plus your `handoff.md` are the whole handoff. No user action required.
- **Cline additionally**: append one journal line per significant step:
  `agent-bridge note --session <SESSION> "<what you did + result>"`

## When the user says "handoff"

Fully refresh your session's `handoff.md`, keeping the section structure already in it
(Goal / Current State / Key Decisions / Next Actions / Files in Play / Gotchas).
Make Next Actions ordered, specific, and verifiable. Then tell the user which session key
you wrote so they can pick it up anywhere.

## When the user says "resume" (or a resumable session exists)

1. Run `agent-bridge status --json` (or read `.agent-context/index.json`).
2. One active session → read its `handoff.md` (fall back to `journal.md` tail and
   `last_said.md`). Several sessions → list them (tool, age, last line) and ask which,
   unless the user named one.
3. Confirm your understanding in 2–3 sentences, then continue from Next Actions.

You may resume ANY session regardless of which tool created it.
