# Human Safety Checklist

The agent is guarded, not trusted. These are the things only you can do.

## Before a run
- [ ] The base branch is clean and committed; guardrails (`.github/`, `scripts/factory/`, `AGENTS.md`) are the versions you reviewed.
- [ ] `bash scripts/factory/test-hooks.sh` passes (39/39).
- [ ] Copilot CLI sandbox is on (`/sandbox policy` shows home read-only, `~/.ssh` and `~/.aws` denied). See [sandboxing.md](sandboxing.md).
- [ ] No cloud/prod credentials in your shell environment, `~/.aws`, or the repo; no `.env` with real secrets in the repo.
- [ ] The task file is yours. If it was pasted from an issue, email or web page, read it for hidden instructions.
- [ ] Risk is rated honestly (`high` for auth, payments, migrations, public APIs, infra).

## During / after a run
- [ ] Read `handoff.md`: Status (READY vs DRAFT), unmet ACs, **Assumptions**, **Risks / follow-ups**.
- [ ] Review denied actions — the agent was trying something:
  `jq -c 'select(.decision=="deny")' <worktree>/.agent-logs/*.jsonl`
- [ ] Skim what the agent ran:
  `jq -c 'select(.event=="preToolUse") | {tool: .toolName, args: .toolArgs}' <worktree>/.agent-logs/*.jsonl`

## Before merging (read the diff yourself)
- [ ] `git diff --stat main...agent/<id>` — only files the task needed.
- [ ] **Nothing unexpected under `.github/`, `scripts/`, `AGENTS.md`, `tasks/`** (`accept.sh` blocks the merge if any of these changed, unless you pass `--allow-guardrails`).
- [ ] No weakened tests: removed assertions, `skip`/`xfail`/`.only`, loosened lint config, changed thresholds.
- [ ] No new dependencies, lockfile changes, or install scripts you did not expect.
- [ ] No new network calls, telemetry, base64 blobs, obfuscated code, or hard-coded URLs/keys.
- [ ] Every acceptance criterion has a test that would fail without the change.
- [ ] `review.md` has no unresolved `BLOCKING:` items; read the `NIT:`s.
- [ ] Use `--allow-guardrails` only after reading each guardrail line changed.
- [ ] Run `accept.sh` from the base checkout; treat the `check.sh` run as executing the branch's code on your machine.

## Red flags — stop and investigate
- Edits to hooks, policy, `check.sh`, `commands.env`, `setup.sh`, or `tasks/`.
- Commands in the log touching `.env`, `.ssh`, `.aws`, tokens, `curl`, `base64`, `eval`.
- Many denied attempts at the same action.
- Handoff claims "all checks passed" but `verify.log` is missing or old.
- Changes far outside the task scope.

## Ongoing
- Review `deny-commands.txt` / `deny-paths.txt` when your stack changes; add project-specific dangers.
- Keep the Copilot CLI and sandbox up to date; re-check flag names after upgrades (`copilot help permissions`).
- Delete old worktrees and logs; logs may contain sensitive command output despite redaction.
- Remember `accept.sh` is not enforced: never merge agent branches by hand without the same checks.
