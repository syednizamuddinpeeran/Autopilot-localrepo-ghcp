# Agent Factory Template — Local Repo (GitHub Copilot CLI)

Variant of [autopilot](https://github.com/syednizamuddinpeeran/autopilot) for repositories with **no GitHub remote**:
no issues, pull requests, Actions, or cloud agent. A task file goes in, a reviewed local branch comes out,
and a human merges it with one script. Copilot CLI runs with full tool permissions inside guardrails, and every action is logged.

```
tasks/<id>.md
  └─ factory agent (orchestrator, small context)
       ├─ skill: task-intake         → .agent-work/<id>/brief.md
       ├─ planner subagent           → plan.md         (read-only on code)
       ├─ implementer subagent ×T    → 1 commit per task, test-first
       ├─ skill: verify-changes      → scripts/factory/check.sh
       ├─ reviewer subagent          → review.md       (independent, read-only)
       └─ skill: handoff             → handoff.md on branch agent/<id>
human: scripts/factory/accept.sh <id>   → guardrail diff check, check.sh, hooks self-test, merge --no-ff
Hooks on every step → .agent-logs/<session>.jsonl
```

## What changed vs. the GitHub version

| GitHub version | Local version |
|---|---|
| Issue + `agent-task` form | `tasks/<id>.md` (`tasks/_template.md`) |
| `run-issue.sh N` (uses `gh`) | `run-task.sh <id>` (git + Copilot CLI only) |
| `copilot/*` and `agent/*` branches | `agent/*` only |
| `open-pr` skill, PR template | `handoff` skill → `.agent-work/<id>/handoff.md` |
| CI (`verify`, `hooks-selftest`, `guardrails-unchanged`) | `accept.sh` runs the same three checks locally |
| Branch protection + merge button | Agents cannot merge; only a human runs `accept.sh` |
| Cloud agent surface, setup-steps workflow | Removed |
| `gh`, `git push` allowed (agent branch) | Denied outright; `git remote` changes denied |

## Safety layers

| Layer | What it stops | Where |
|---|---|---|
| Isolation | Agent touching your main checkout | git worktree per task + Copilot CLI sandbox |
| Guard hook (`preToolUse`) | Any push/remote change, commits or merges off `agent/*`, `gh`, `sudo`, secret reads, curl\|sh, deploys, editing hooks/agents/skills/scripts/tasks, log tampering | `.github/hooks/scripts/guard.sh` + `policy/*.txt` |
| Stop gate (`agentStop`) | Finishing with unverified changes | `stop-gate.sh` + `check.sh` marker |
| `accept.sh` | Unreviewed guardrail edits; merging unverified work | human-run, from the base checkout |
| No standing secrets | Credential theft / cloud damage | Don't give agents AWS or prod credentials |

The guard is fail-closed on errors, but a hook **timeout is fail-open** — keep the scripts fast.
Deny-lists reduce risk; they are not a complete sandbox. Isolation and the human-only merge are the real boundary.
Unlike the GitHub version, `accept.sh` is a convention, not an enforced server-side gate: anyone with shell access can merge by hand.

## One-time setup (WSL Ubuntu / macOS / Linux)
```bash
sudo apt-get install -y git jq
npm install -g @github/copilot      # Copilot CLI
copilot                             # sign in, then inside the session:
/sandbox enable                     # turn on local sandboxing (persists in settings)
/sandbox policy                     # deny ~/.ssh and ~/.aws; keep the working directory read/write
```
Keep repos in the WSL filesystem (`~/code/...`), not `/mnt/c/...` — faster, and hooks run as bash.

## Per-project changes (the only things you edit)
1. `scripts/factory/commands.env` — base branch, setup, format, lint, typecheck, test, build commands (presets included).
2. `AGENTS.md` → **Project** section — what the repo is, layout, conventions.

Install into an existing repo (needs a git repo with at least one commit; no remote required):
```bash
./install.sh ~/code/my-repo
cd ~/code/my-repo && bash scripts/factory/test-hooks.sh && bash scripts/factory/check.sh
git add -A && git commit -m "chore: add agent factory"
```

## Running
```bash
cp tasks/_template.md tasks/add-csv-export.md   # fill in, then commit it on the base branch
git add tasks && git commit -m "task: add-csv-export"

scripts/factory/run-task.sh add-csv-export          # autonomous run in ../<repo>-worktrees/add-csv-export
scripts/factory/run-task.sh add-csv-export --watch  # interactive; switch to autopilot yourself
```
Flag names (`--agent`, `-p`, `--allow-all-tools`, `--deny-tool`) can change between CLI versions —
check `copilot help permissions` once and adjust `run-task.sh` if needed.

Review and merge (from the base checkout, clean tree):
```bash
cat ../<repo>-worktrees/add-csv-export/.agent-work/add-csv-export/handoff.md
git diff main...agent/add-csv-export
scripts/factory/accept.sh add-csv-export                   # blocks if the branch touches guardrail files
scripts/factory/accept.sh add-csv-export --allow-guardrails   # after you reviewed those edits
```

## Logs
`.agent-logs/<sessionId>.jsonl` (one JSON line per event) and `.agent-logs/index.jsonl` (session start/end).
```bash
jq -c 'select(.event=="preToolUse") | {ts, tool: .toolName, decision, reason}' .agent-logs/*.jsonl
jq -c 'select(.decision=="deny")' .agent-logs/*.jsonl          # everything the guard blocked
jq -c 'select(.event|test("subagent")) | {ts, event, agentName}' .agent-logs/*.jsonl
```

## Files
```
AGENTS.md                         always-loaded rules (kept short)
tasks/_template.md                task file template
.github/agents/*.agent.md         factory, planner, implementer, reviewer
.github/skills/*/SKILL.md         task-intake, implementation-plan, verify-changes, handoff
.github/hooks/factory.json        hook wiring for all events
.github/hooks/scripts/            log.sh, guard.sh, stop-gate.sh, common.sh
.github/hooks/policy/             deny-commands.txt, deny-paths.txt
scripts/factory/                  commands.env, check.sh, setup.sh, run-task.sh, accept.sh, test-hooks.sh
```
(`.github/` is kept because Copilot CLI loads agents, skills, and hooks from there; nothing in it requires GitHub.)

## Known limits
- Edits made through shell commands (e.g. `sed -i`) bypass path rules, but the stop gate and `accept.sh` still catch unverified changes.
- `.env.example` is blocked by the `.env` rule; rename it (e.g. `env.example`) or edit `deny-paths.txt`.
- Native Windows needs PowerShell versions of the hooks; this template targets WSL, macOS, and Linux.
