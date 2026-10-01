# CLAUDE.md

## Project lessons

- During your interaction with the user, if you find anything reusable in this project (e.g. version of a library, model name), especially about a fix to a mistake you made or a correction you received, you should take note in the `Lessons` section in the **repo-level** `AGENTS.md` (at the repository root, NOT this global file) so you will not make the same mistake again.

## Network exposure

- Docker-published Compose ports can bypass expected UFW `deny incoming` behavior through Docker iptables chains; local/dev service ports should bind explicitly to `127.0.0.1` in `ports` mappings on cloud hosts unless public exposure is intended.

## Resource limits

- Memory is shared machine-wide and one OOM can kill every session. Run heavy commands (any browser launch incl. screenshot/smoke/e2e scripts, Electron, emulators, anything over ~2 GiB) as `~/.claude/bin/heavy-gate [-n N] [-l label] -- <cmd>` with `run_in_background: true`. `-n` is the most browsers running at once. The PreToolUse hook only blocks the browser/Electron launches it recognizes; the rule covers everything else too.
- Prefer one browser at a time and serial browser verification. Playwright Test runs one browser per worker and defaults to half the CPU cores, so use `--workers=1` with `-n 1`; tests that launch extra browsers themselves need a higher `-n`.
- If the gate refuses, don't run the command uncapped, don't change `HEAVY_GATE_*`, and never disguise a command to get past the hook.
- In code you write: one browser per process, closed in `finally`. Tell subagents that might launch browsers to use the gate.
- Before fanning out agents or parallel heavy work, check `free -m` and `~/.claude/bin/heavy-gate --status`.

Consult `~/.claude/bin/README.md` for gate behavior, orchestration, or gate/hook changes; run its regression checks after changes.
