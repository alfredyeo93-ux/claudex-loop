---
name: claudex-loop
description: Two-model build loop. Claude writes the plan and OpenAI Codex (GPT) audits it until it holds up; then the roles switch, Codex builds and Claude audits the implementation. Use when the user says "claudex", asks for a second model to check a plan or code, wants a cross-model review, or wants a build where one AI doesn't grade its own homework.
---

# claudex-loop

One model plans, the other attacks the plan. Then they swap: the auditor builds, the planner audits. Nothing ships until both have signed off.

| Phase | Writes | Audits |
|---|---|---|
| 1. Plan | Claude → `.claudex/PLAN.md` | Codex (read-only) |
| 2. Build | Codex (workspace-write) | Claude reviews the diff and runs the proof checks |

When this skill runs **inside Codex** instead of Claude Code, swap the tools: Codex writes the plan and calls `claude -p` as the plan auditor, then `claude -p` builds and Codex audits. The steps below are written from Claude Code's side.

## 0. Preflight

1. `codex --version`. If it's missing, tell the user to install it (`npm i -g @openai/codex`) and sign in (`codex login`), then stop.
2. The working folder must be a git repository so the build can be diffed. If `git status --porcelain` shows uncommitted changes, ask the user to commit or stash them first. Never stash, reset or discard their work yourself.
3. Create `.claudex/` and add `.claudex/` to `.git/info/exclude` so the audit trail never gets committed by accident.
4. Read the user's task. If it is ambiguous, ask one clarifying question now, not halfway through the build.

`<skill>` in the commands below means this skill's own folder.

Defaults: up to **3 plan-audit rounds** and **3 build-fix rounds**. The user can change these ("claudex with 5 rounds").

## 1. Claude plans

Inspect the repository (structure, conventions, tests, how to run it), then write `.claudex/PLAN.md` with exactly these sections:

```markdown
# PLAN: <task in one line>
## Scope        what changes and what explicitly does not
## Requirements locked requirements and acceptance criteria, numbered (R1, R2, ...)
## Plan         ordered implementation steps with the files each step touches
## Proof checks commands that prove it works (tests, build, lint, a curl, a script), each with its expected result
## Risks        what could break, and how the plan prevents it
## Audit log    (filled in during the loop)
```

Every requirement must map to at least one step and one proof check.

## 2. Codex audits the plan (read-only)

Run Codex in its default read-only sandbox with the audit prompt and the verdict schema. The prompt is `prompts/plan-audit.md` with `{{PLAN}}` replaced by the contents of `.claudex/PLAN.md`; save it as `.claudex/plan-audit-prompt.md` first.

```bash
codex exec --sandbox read-only --output-schema <skill>/schemas/audit.schema.json \
  -o .claudex/plan-audit-<N>.json - < .claudex/plan-audit-prompt.md
```

PowerShell: `Get-Content .claudex/plan-audit-prompt.md -Raw | codex exec --sandbox read-only --output-schema <skill>/schemas/audit.schema.json -o .claudex/plan-audit-<N>.json -`

Read the JSON. For each finding, either change the plan or reject it with a one-line reason. Record both in **Audit log** (round number, finding, accepted/rejected, why). Never silently drop a blocker.

- `verdict: "APPROVE"` with no open blockers: go to phase 3.
- `verdict: "REVISE"`: update the plan and re-audit. For later rounds, keep Codex's context with `codex exec resume --last` and send only the revised plan plus your responses.
- Still REVISE after the round limit: stop, show the user the open findings, and ask how to proceed.

If Codex exits non-zero, times out, or returns output that doesn't match the schema, stop and tell the user. Never treat a failed audit as an approval.

## 3. Switch roles: Codex builds

Fill `prompts/build.md` with the approved plan and run Codex with write access to the workspace only:

```bash
codex exec --sandbox workspace-write -o .claudex/build-report.md - < .claudex/build-prompt.md
```

Codex implements the plan, runs the proof checks and reports. It must not commit, push, install global packages or touch files outside the repo. Afterwards, save `git diff --stat` and `git diff` to `.claudex/build.diff`.

## 4. Claude audits the implementation

Claude does not write the code in this phase. It reviews:

1. **Spec match:** every requirement R1..Rn is implemented, and nothing outside Scope changed.
2. **Proof:** run every proof check yourself. Don't trust the build report's claims.
3. **Correctness:** read the diff for bugs, edge cases, error handling, security (injection, secrets, unsafe input) and leftover debug code.
4. **Fit:** matches the repo's conventions; no needless dependencies or abstractions.

Write findings to `.claudex/impl-audit-<N>.md` with a severity (blocker / major / minor) and the exact file and line.

- No blockers or majors and all proof checks pass: go to phase 5.
- Otherwise send the findings back to Codex with `prompts/fix.md`: `codex exec resume --last --sandbox workspace-write - < .claudex/fix-prompt.md`. Then re-audit. Stop at the round limit and hand the open findings to the user.

## 5. Report

Tell the user, briefly:
- what was built and which files changed (`git diff --stat`);
- proof checks and their results, from your own run;
- the audit trail: plan rounds and build rounds, with what each model caught;
- anything left open, plus minor findings you chose not to fix.

Leave the changes uncommitted unless the user asks you to commit.

## Rules

- One model never approves its own work: the planner doesn't approve its plan, and the builder doesn't approve its build.
- Read-only for audits, workspace-write for builds; never `danger-full-access`.
- Keep secrets out of prompts and `.claudex/` files.
- Each `codex exec` uses the user's own Codex sign-in or API key and may cost money on their plan. Mention it once at the start of a run.
