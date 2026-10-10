# Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `tasks/<id>.md is not committed on 'main'` | Task not committed on the base branch | `git add tasks && git commit` on the base branch |
| `task id must be a lowercase slug` | Invalid id | Use `[a-z0-9._-]`, starting alphanumeric |
| `Missing: copilot/jq/git` | Tool not installed | Install it (see usage-guide) |
| `No checks configured` (exit 3) | All commands empty in `commands.env` | Configure them, or `ALLOW_NO_CHECKS=1` for docs-only repos |
| Copilot rejects flags | CLI flag names changed | `copilot help permissions`; adjust `run-task.sh` (human edit) |
| "Blocked by factory guard" in the agent output | Policy matched a command/path | Check the `reason` in `.agent-logs`; if legitimate, change the task approach or adjust the policy files yourself — the agent cannot |
| Legit command blocked (e.g. `.env.example`, a word matching `\bgh\b`) | Regex too broad | Narrow the pattern in `policy/` and rerun `test-hooks.sh` |
| Agent keeps being told to verify | Stop gate: changes since last green `check.sh` | Run `check.sh` and fix failures; the marker is invalidated by any edit |
| `Branch modifies guardrail files` | `accept.sh` guardrail diff | Read the listed changes; if intended, `--allow-guardrails` |
| `Check out 'main' first` / `Working tree is not clean` | `accept.sh` preconditions | Switch to base, commit/stash |
| `No branch agent/<id>` | Run never started or branch deleted | Re-run `run-task.sh` |
| Handoff is DRAFT | High risk, failed verification, or open questions | Read Risks and questions; refine the task and rerun |
| Hooks seem not to run | Not in the repo/worktree where Copilot starts, or scripts not executable | Run `scripts/factory/setup.sh`; confirm `.github/hooks/factory.json` is present |
| Hook slow → a call slipped through | Timeout is fail-open | Keep policy files small; avoid heavy work in hooks |
| Sandbox blocks a legitimate write | Path outside allowed dirs | Add the path in `/sandbox config` → Filesystem, or move work inside the worktree |
| `/sandbox` says unavailable on Linux/WSL | Missing `bwrap` (≥ 0.5.0), `slirp4netns` or other requirements | Install them; see [sandboxing.md](sandboxing.md#platform-requirements) |
| No `.agent-logs`, nothing blocked | Repository hooks not loaded in `-p` mode | Use `run-task.sh` (sets `GITHUB_COPILOT_PROMPT_MODE_REPO_HOOKS=true`) or trust the folder |
| Different results on `/mnt/c` | Slow filesystem, line endings | Use the WSL filesystem |
