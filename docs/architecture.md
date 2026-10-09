# Architecture

## End-to-end flow

```
tasks/<id>.md  (committed on the base branch by a human)
      │
scripts/factory/run-task.sh <id>
      │  creates git worktree ../<repo>-worktrees/<id> on new branch agent/<id>
      │  runs setup.sh, then: copilot --agent factory -p "<prompt>" --allow-all-tools --deny-tool …
      ▼
factory agent (orchestrator)
  1. skill task-intake      → .agent-work/<id>/brief.md
  2. planner subagent       → plan.md        (writes only the plan)
  3. implementer subagent   → one commit per plan task, test-first (looped over T1…Tn)
  4. skill verify-changes   → scripts/factory/check.sh → verify.log (≤3 fix loops)
  5. reviewer subagent      → review.md      (BLOCKING / NIT, VERDICT) (≤2 fix loops)
  6. skill handoff          → handoff.md   (READY or DRAFT)
      ▼
human: scripts/factory/accept.sh <id>
      guardrail-diff check → check.sh in the branch worktree → hook self-test → confirm → git merge --no-ff
```

Hooks wrap every step (see [security-model.md](security-model.md)) and write an audit trail to `.agent-logs/`.

## Key design ideas

- **Small orchestrator context.** The factory passes file paths between steps, not content. Heavy work runs in subagents.
- **Files are the interface.** Each step reads and writes files in `.agent-work/<id>/`, so any step can be inspected or resumed.
- **One verification gate.** `check.sh` is the single definition of "good"; agents, humans and `accept.sh` all use it.
- **Humans own the rules.** Agents, skills, hooks, policy, scripts and tasks cannot be edited by the agent (enforced by the guard hook).
- **Merge is human-only.** There is no remote and no CI; `accept.sh` is the local equivalent of CI + branch protection + the merge button.

## Where things live

| Path | Purpose | Git-tracked |
|---|---|---|
| `AGENTS.md` | Always-loaded rules and project description | yes |
| `tasks/<id>.md` | Task input | yes (base branch) |
| `.github/agents/*.agent.md` | Agent definitions | yes |
| `.github/skills/*/SKILL.md` | Step procedures loaded on demand | yes |
| `.github/hooks/factory.json` | Hook wiring (11 events) | yes |
| `.github/hooks/scripts/` | `guard.sh`, `stop-gate.sh`, `log.sh`, `common.sh` | yes |
| `.github/hooks/policy/` | `deny-commands.txt`, `deny-paths.txt` | yes |
| `scripts/factory/` | `commands.env`, `check.sh`, `setup.sh`, `run-task.sh`, `accept.sh`, `test-hooks.sh` | yes |
| `.agent-work/<id>/` | brief, plan, verify.log, review, decisions, handoff | no (gitignored) |
| `.agent-logs/` | `<session>.jsonl`, `index.jsonl`, `.verified` marker | no (gitignored) |
| `../<repo>-worktrees/<id>/` | Isolated checkout per task | outside repo |

## Isolation model

Each task gets its own git worktree on its own `agent/<id>` branch. Your main checkout is never touched. `.agent-work/` and `.agent-logs/` are per-worktree, so the audit log for a run lives in that run's worktree.

## The verification marker

`check.sh` computes a hash of the working state (HEAD + tracked diff + untracked file hashes) and stores it in `.agent-logs/.verified` only if every configured check passes. The `agentStop` hook recomputes the hash; any edit after the last green run changes the hash and the agent is forced to re-verify before it can finish.
