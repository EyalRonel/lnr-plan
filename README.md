# /lnr-plan

A Claude Code slash command that turns a Linear (or any tracker) issue — or
just a free-text ask — into a single, self-contained, no-build HTML plan:
an executive summary, a security/risks tab, an open-questions tab with
inline suggestion-picking, a domain-by-domain impact breakdown (UI / API /
Modules / Services / Models·schemas / Cloud), a mermaid chain diagram, and a
full printable plan — plus a highlight-to-comment review flow for whoever
reads it.

It's generic: you are not authoring HTML/CSS/JS when you use it. The
engine (`lnr-plan.template.html`) is fixed and already written; the command
(`lnr-plan.md`) just tells an agent how to read your issue and your code and
fill in the plan-data blocks.

## What's here

- `commands/lnr-plan.md` — the command itself.
- `commands/lnr-plan.template.html` — the engine: a complete, working HTML
  page with fenced `▸▸ PLAN DATA` regions the command fills in.
- `commands/lnr-plan.worldmap.json` — the fixed domain/module registry,
  mirrored inside the template's `WORLD_MAP` constant.
- `SETUP.md` — a one-time prompt to generate your own `WORLD_MAP` from
  your own repo, since the one that ships here is placeholder data.

## Install

Copy the three files in `commands/` into your repo's `.claude/commands/`
folder, then follow `SETUP.md` to replace the placeholder domain data with
your own.
