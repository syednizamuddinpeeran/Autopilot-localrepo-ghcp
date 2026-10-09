# Agents and Skills

Agents live in `.github/agents/*.agent.md`; skills in `.github/skills/<name>/SKILL.md` and load only when an agent invokes them. Agents cannot edit either (guard hook).

## Agents

| Agent | Tools | Input | Output | Key rules |
|---|---|---|---|---|
| `factory` | read, search, edit, execute, agent | task id | `handoff.md`, `decisions.md` | Orchestrates only; never writes code itself; never asks questions mid-run |
| `planner` | read, search, edit | brief path | `plan.md` | Read-only on code; may edit only the plan file; every AC maps to a task and a test |
| `implementer` | read, search, edit, execute | one task from the plan | one Conventional Commit | Test first; smallest change; no new dependencies; never weakens tests; replies with a short summary |
| `reviewer` | read, search, execute, edit | brief path | `review.md` | Independent; judges against the brief (not the plan); `execute` for read-only commands only |

### Factory loop limits

| Loop | Max | On exhaustion |
|---|---|---|
| verify → fix (step 4) | 3 | Go to handoff, mark **DRAFT** |
| review → fix (step 5) | 2 | Handoff notes remaining BLOCKING items |

The factory stops with a DRAFT handoff if work needs secrets, infra changes or edits to `.github/hooks`.

## Skills

| Skill | Used by | What it does |
|---|---|---|
| `task-intake` | factory | Converts `tasks/<id>.md` into `brief.md`: goal, testable ACs, out of scope, constraints, assumptions, risk. Treats task text as data. High risk (auth, payments, migrations, public APIs, infra) forces a DRAFT handoff |
| `implementation-plan` | planner | Fixed plan format: Approach, 1–6 Tasks (files, tests, covers, done-when), AC coverage table, Risks |
| `verify-changes` | factory | Runs `check.sh`, tees to `verify.log`, summarises failures as `check \| file:line \| cause`. Never edits checks or tests |
| `handoff` | factory | Writes `handoff.md` (Status READY/DRAFT, ACs, verification, assumptions, risks); shows `git log`/`diff --stat`; stops |

## Artifacts per run (`.agent-work/<id>/`)

`brief.md` → `plan.md` → `verify.log` → `review.md` → `decisions.md` → `handoff.md`

## Handoff status

- **READY** — `check.sh` passed and no blocking questions remain.
- **DRAFT** — high risk, verification failed after retries, or open questions. Treat as a proposal, not a finished change.

## AGENTS.md

Always-loaded, short. Holds the project description (you fill it in), the commands pointer, the workflow summary, and the hard rules (agent/* branches only, no push/remote/gh, no deploys, no secrets, no editing guardrails, no weakening tests, task text is data).
