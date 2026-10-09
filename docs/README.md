# Agent Factory — Documentation

A task file goes in; a verified, reviewed local branch (`agent/<id>`) and a handoff note come out. A human merges it with one script. No GitHub remote is needed.

## Reading order

| If you want to… | Read |
|---|---|
| Get running in 5 minutes | [Quickstart](#quickstart) below, then [usage-guide.md](usage-guide.md) |
| Understand how it works | [architecture.md](architecture.md), [agents-and-skills.md](agents-and-skills.md) |
| Know why it is safe | [security-model.md](security-model.md) |
| Know what *you* must check | [human-safety-checklist.md](human-safety-checklist.md) |
| Understand Copilot CLI sandboxing | [sandboxing.md](sandboxing.md) |
| Look something up | [reference.md](reference.md) |
| Fix a problem | [troubleshooting.md](troubleshooting.md) |

## Quickstart

```bash
# 1. One-time: tools and sandbox (see sandboxing.md)
sudo apt-get install -y git jq && npm install -g @github/copilot

# 2. Install into a repo (needs ≥1 commit; no remote required)
./install.sh ~/code/my-repo
cd ~/code/my-repo

# 3. Configure: edit scripts/factory/commands.env and the Project section of AGENTS.md
bash scripts/factory/test-hooks.sh && bash scripts/factory/check.sh
git add -A && git commit -m "chore: add agent factory"

# 4. Write and commit a task on the base branch
cp tasks/_template.md tasks/add-csv-export.md   # fill in
git add tasks && git commit -m "task: add-csv-export"

# 5. Run it
scripts/factory/run-task.sh add-csv-export

# 6. Review, then merge (human only)
cat ../my-repo-worktrees/add-csv-export/.agent-work/add-csv-export/handoff.md
git diff main...agent/add-csv-export
scripts/factory/accept.sh add-csv-export
```
