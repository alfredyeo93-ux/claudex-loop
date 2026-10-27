# claudex-loop

**Claude plans, Codex audits. Then they swap: Codex builds, Claude audits.**

A free agent skill that makes Claude and OpenAI's Codex (GPT) work on the same task, so no single AI grades its own homework.

```
1. Claude writes PLAN.md ─────► 2. Codex audits the plan (read-only)
          ▲                                  │
          └──── revise until it holds up ◄───┘
3. Roles switch: Codex builds ──► 4. Claude audits the diff + runs the proof checks
          ▲                                  │
          └──── fix until it passes ◄────────┘
5. Report: what changed, proof results, the full audit trail
```

## Why

A model that plans, builds and reviews its own work tends to approve its own blind spots. Two frontier models from different labs catch different mistakes. claudex-loop makes them challenge each other at both checkpoints: the plan and the implementation.

## Install

You need **Claude Code** and the **Codex CLI**:

```bash
npm i -g @openai/codex
codex login
```

**Claude Code:** copy this folder to `~/.claude/skills/claudex-loop` (or `.claude/skills/claudex-loop` inside a project).

**Codex:** copy this folder to `~/.agents/skills/claudex-loop` (or `.agents/skills/claudex-loop` inside a project). Inside Codex the roles flip automatically: Codex plans and `claude -p` audits.

```bash
git clone https://github.com/alfredyeo93-ux/claudex-loop.git ~/.claude/skills/claudex-loop
```

## Use

In a git repository with a clean working tree, ask:

> claudex: add CSV export to the admin orders page

or "use claudex-loop to build …", or "have Codex audit the plan first".

The audit trail lives in `.claudex/` (excluded from git): `PLAN.md` with an audit log, each Codex verdict as JSON, the build report and the diff.

## What's in the box

| File | Purpose |
|---|---|
| `SKILL.md` | The workflow the agent follows |
| `prompts/plan-audit.md` | The auditor's brief for the plan |
| `prompts/build.md` | The builder's brief |
| `prompts/fix.md` | Sending implementation findings back to the builder |
| `schemas/audit.schema.json` | Structured APPROVE / REVISE verdict with findings (`codex exec --output-schema`) |

## Safety

- Audits run in Codex's **read-only** sandbox; builds run in **workspace-write**. Never full access.
- The skill never commits, pushes, stashes or discards your work.
- A failed or malformed audit stops the loop. It is never treated as an approval.
- Each run uses your own Codex sign-in or API key and Claude subscription, so it can cost money.

## Credits

Inspired by the "Claude plans, GPT audits, then they swap" workflow shown by [@nateherkai](https://www.instagram.com/nateherkai/), and by earlier community loops such as `codex-claude-loop` in [bear2u/my-skills](https://github.com/bear2u/my-skills) (MIT). This is an original implementation. Not affiliated with Anthropic or OpenAI.

## Licence

MIT. See [LICENSE](LICENSE).
