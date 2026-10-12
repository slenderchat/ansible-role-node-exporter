# Agent Instructions

Shared instructions for all coding agents (Claude Code, OpenAI Codex, etc.) working in this repository.

## Project context

- For substantial work, read `/PROJECT_CONTEXT.md` first.
- Keep `PROJECT_CONTEXT.md` current during long or multi-step work, and update it before handoff or completion when durable handoff state has changed.
- Store only non-obvious information useful to a fresh agent: important decisions and their rationale, non-obvious constraints, current or incomplete state, blockers, next steps, and relevant non-obvious run/verification commands.
- Remove or revise stale information.
- Do not store conversation history, trivial edits, information easily rediscovered from the repository, documentation already maintained elsewhere (e.g. role `README.md` files), or secrets/credentials.

## Working rules

- Follow the existing project architecture, conventions, and tooling; keep changes scoped to the task.
- Run applicable existing validation; never claim checks that were not run or did not pass.
- Do not put task-specific instructions in this file.
