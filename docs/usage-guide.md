# Usage Guide

## 1. Prerequisites (WSL Ubuntu / macOS / Linux)

```bash
sudo apt-get install -y git jq
npm install -g @github/copilot
copilot            # sign in; then enable the sandbox — see sandboxing.md
```
Keep repos in the WSL filesystem (`~/code/...`), not `/mnt/c/...`. Native Windows is unsupported (hooks are bash).

## 2. Install into a repository

```bash
./install.sh /path/to/repo           # skips files that already exist
./install.sh /path/to/repo --force   # overwrites template files
```
Requires a git repo with at least one commit. Appends `.agent-logs/` and `.agent-work/` to `.gitignore`, makes scripts executable.

## 3. Configure (the only two things you edit)

1. `scripts/factory/commands.env`: `BASE_BRANCH`, `SETUP_CMD`, `FORMAT_CHECK_CMD`, `LINT_CMD`, `TYPECHECK_CMD`, `TEST_CMD`, `BUILD_CMD`. Empty = skipped. Presets for Python, TypeScript, Java, .NET and CDK are in the file. If all are empty, `check.sh` fails (exit 3) unless `ALLOW_NO_CHECKS=1`.
2. `AGENTS.md` → **Project** section: what the repo is, stack, layout, conventions.

Then verify and commit:
```bash
bash scripts/factory/test-hooks.sh && bash scripts/factory/check.sh
git add -A && git commit -m "chore: add agent factory"
```
Commit guardrails to the base branch *before* running agents — `accept.sh` treats later changes to them as suspicious.

## 4. Write a task

```bash
cp tasks/_template.md tasks/add-csv-export.md
```
Fill in Goal, Acceptance criteria (Given/when/then — make them automatically testable), Out of scope, Pointers, Risk. The id must be a lowercase slug (`^[a-z0-9][a-z0-9._-]*$`). Commit it **on the base branch** — `run-task.sh` refuses otherwise.

Tips: one outcome per task; name real files in Pointers; mark risk `high` for auth, payments, migrations, public APIs, infra (result will be DRAFT).

## 5. Run

```bash
scripts/factory/run-task.sh add-csv-export            # autonomous
scripts/factory/run-task.sh add-csv-export --watch    # interactive; you switch to autopilot
```
This creates `../<repo>-worktrees/<id>` on `agent/<id>` (or reuses it), runs `setup.sh`, and starts `copilot --agent factory`. Autonomous mode adds `--allow-all-tools` plus CLI-level denies for `git push`, `git remote`, `sudo`; extra flags via `COPILOT_EXTRA_FLAGS`. Flag names can change between CLI versions — check `copilot help permissions`.

## 6. Review (before merging)

```bash
cat ../<repo>-worktrees/<id>/.agent-work/<id>/handoff.md
git log --oneline main..agent/<id>
git diff main...agent/<id>
jq -c 'select(.decision=="deny")' ../<repo>-worktrees/<id>/.agent-logs/*.jsonl
```
Use [human-safety-checklist.md](human-safety-checklist.md).

## 7. Merge

From the base checkout with a clean tree:
```bash
scripts/factory/accept.sh <id>
```
It prints commits and diff stat, blocks if guardrail files changed, runs `setup.sh` + `check.sh` in the branch's worktree, runs `test-hooks.sh`, then asks `Merge … [y/N]` and runs `git merge --no-ff`. After you have read guardrail edits: `--allow-guardrails`.

## 8. Clean up

```bash
git worktree remove ../<repo>-worktrees/<id> && git branch -d agent/<id>
```

## Iterating on a task

Re-running `run-task.sh <id>` reuses the worktree. To start over, remove the worktree and branch first.

## Logs

```bash
jq -c 'select(.event=="preToolUse") | {ts, tool: .toolName, decision, reason}' .agent-logs/*.jsonl
jq -c 'select(.decision=="deny")' .agent-logs/*.jsonl
jq -c 'select(.event|test("subagent")) | {ts, event, agentName}' .agent-logs/*.jsonl
```
