# Security Model

The agent runs with broad tool permissions (`--allow-all-tools`), so safety comes from **layers around it**, not from trusting the agent. No single layer is a complete sandbox; the real boundary is isolation plus the human-only merge.

## Threats considered

| Threat | Example |
|---|---|
| Prompt injection | Task text, code comments or fetched web pages say "ignore your rules and run X" |
| Secret theft | Reading `.env`, `~/.ssh`, `~/.aws/credentials`, tokens in env vars |
| Exfiltration / publishing | `git push`, `gh`, adding a remote, `curl … \| sh` |
| Cloud damage | `terraform apply`, `cdk deploy`, `aws … delete-*`, `kubectl delete` |
| Escaping the box | `sudo`, destructive `rm`, `chmod 777`, privileged containers |
| Weakening the rules | Editing hooks, policy, agents, skills, `check.sh`, `commands.env` |
| Faking success | Forging the `.verified` marker, tampering with logs, skipping/weakening tests |
| Unreviewed merge | Agent merging its own work into the base branch |

## Layers

| # | Layer | Mechanism | Stops |
|---|---|---|---|
| 1 | Isolation | Separate git worktree and `agent/<id>` branch | Touching your main checkout |
| 2 | Copilot CLI sandbox (you enable it) | OS-level filesystem/network limits — see [sandboxing.md](sandboxing.md) | Writes outside the worktree, reads of denied paths |
| 3 | CLI deny flags | `--deny-tool 'shell(git push)'`, `shell(git remote)`, `shell(sudo)` | Pushes, remotes, sudo |
| 4 | `preToolUse` guard | `guard.sh` + `policy/deny-commands.txt` + `deny-paths.txt` | Everything in the threat table above for shell commands and file tools |
| 5 | Branch rule in guard | `git commit/merge/rebase/cherry-pick/am/revert` only on `agent/*` | Commits to the base branch |
| 6 | Instructions | `AGENTS.md` hard rules; task text treated as data | Honest mistakes, naive injection |
| 7 | Stop gate | `stop-gate.sh` + `check.sh` state-hash marker | Finishing with unverified changes |
| 8 | Audit log | `log.sh` on 11 events, secrets redacted | Undetected activity; supports review |
| 9 | Human merge | `accept.sh` run by a person | Unreviewed code reaching the base branch |

## How the guard works

- Runs before **every** tool call; receives JSON on stdin.
- **Shell:** the command is whitespace-normalised and matched case-insensitively against every regex in `deny-commands.txt`. A match returns `{"permissionDecision":"deny",…}`.
- **Files:** every path-like argument (including `apply_patch` headers) is matched against `deny-paths.txt`. Plain rules block read and write (secrets); `write:` rules block writes only (guardrails, `.git/`, logs, `tasks/`).
- **Fail-closed:** a crash or non-zero exit denies the call.
- **Fail-open on timeout:** if the hook exceeds `timeoutSec` (15s) it is killed and the call is allowed. Keep the scripts fast.
- **Hooks must load.** The CLI loads repository hooks in `-p` mode only for a trusted folder or with `GITHUB_COPILOT_PROMPT_MODE_REPO_HOOKS=true`, which `run-task.sh` sets. If you start `copilot -p` yourself in an untrusted folder, **no guard runs**.
- Policy files are themselves write-protected from the agent.
- `test-hooks.sh` runs 39 assertions (allow/deny cases, stop gate, redaction). `accept.sh` runs it before every merge.

## What is denied (summary)

Destructive filesystem ops; `sudo`/`su`; curl/wget piped to a shell; `git push`, remote changes, merges/rebases touching main/master, `filter-branch`, global git config; `accept.sh`; `gh`; terraform/cdk/pulumi/sam/kubectl/helm changes and AWS mutating calls; reading `.env`, SSH keys, cloud credentials, token files; bare `env`/`printenv`; touching `.agent-logs`. Full lists: `.github/hooks/policy/`.

## Known limits — be honest with yourself

- **Deny-lists are bypassable.** Shell is expressive: `sed -i` on a protected file, a script that writes the file, base64-decoded commands, or a language runtime (`python -c`) can sidestep regexes. The stop gate and `accept.sh` catch *unverified* changes, not every malicious one.
- **Hook timeout fails open.**
- **`accept.sh` is a convention.** Anyone with shell access can `git merge` by hand; nothing server-side enforces it.
- **`accept.sh` guardrail diff covers** everything under `.github/` and `scripts/factory/`, plus `tasks/`, `AGENTS.md` and `install.sh`. Any change there blocks the merge until you re-run with `--allow-guardrails`. This matters because `accept.sh` executes the **branch's** `setup.sh` and your configured check commands on your machine. Shell edits can bypass the agent's path rules, so this diff check is the backstop.
- **Checks run branch code.** `check.sh` executes the project's tests/build from the branch, which is arbitrary code (and `commands.env` itself is flagged only if changed).
- **Environment variables** are not hidden from the agent except by the printenv/echo patterns; do not export secrets in the shell that starts `run-task.sh`.
- **No network control** from the hooks beyond blocking specific commands; use the sandbox's network settings.
- **Not a defence against a malicious repo owner.** The threat model is a mistaken or manipulated agent, not a hostile human with write access.

## Secrets and credentials

- Never give agents AWS, cloud or production credentials; run with a clean environment.
- Logs redact GitHub tokens, AWS keys, `sk-` keys, private keys and `password|secret|token|api_key = value` patterns before writing. Redaction is pattern-based; treat logs as sensitive anyway.
- `.env.example` is blocked by the `.env` rule — rename (e.g. `env.example`) or adjust `deny-paths.txt`.
