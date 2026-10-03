# bin

Machine-wide guard for memory-heavy commands, shared by every project, session and agent on this box (and by OMP). Heavy work is limited at launch time by concurrency and free memory, not by agent count.

## heavy-gate

```
heavy-gate [-n SLOTS] [-m MEMORY_MAX] [-l LABEL] -- cmd args...
heavy-gate --status    # slot holders + free memory
```

- 2 slots machine-wide (flock files in `/tmp/heavy-gate`, same as OMP). `-n` is the most browsers running at once, not a worker count. Playwright Test runs one browser per worker (default: half the CPU cores), so `--workers=1` keeps it to one; tests that call `launch()` themselves add more. Other runners use their own flag (Jest `--runInBand`, etc.).
- Starts a command only while MemAvailable >= 4500 MiB, with launches spaced 20 s apart.
- Runs the command in its own systemd scope capped at 6G (`-m`), with no swap and `oom_score_adj` 500. Refuses to run if the user systemd manager or isolation is unavailable.
- Uses `dbus-run-session` inside that scope so Chromium cannot ask the host user systemd manager to migrate it into an uncapped sibling scope. Requires `dbus-run-session`/`dbus-daemon`; missing or failed session isolation must not launch the payload. The gate supervisor keeps the host bus for scope management.
- Waits for capacity instead of failing: run it in the background, never under `timeout`.
- Use it even when a project has its own gate.
- A slot stays held until every process in the scope exits. Stop only your own holders; after a SIGKILL, check for leftover processes.
- Host session D-Bus services are intentionally not inherited. This stops automatic Chromium migration, not deliberate same-user/root escape. Per-command caps and the launch-time memory check are not a machine-wide aggregate memory budget.

## heavy-gate-hook

PreToolUse(Bash) hook registered in `../settings.json`. Denies ungated browser/Electron launches it recognizes (Playwright CLI, browser binaries, scripts importing playwright/puppeteer/selenium, package scripts). It is a best-effort static check, not a sandbox.

## Orchestration

The gate already caps browser work at 2 wide, so extra parallel agents only queue. Run browser work in batches of <= 2 or one serial verifier, give other agents static checks and unit tests, and check `free -m` / `heavy-gate --status` before fanning out.

Guard other shared resources (ports, GPU, a shared dev server) the same way: a lock or check at the point of use, not a headcount rule in a prompt. Don't add new per-project locks.

## Maintenance

- After changing the gate or hook: `python3 -m unittest discover -s ~/.claude/tests -p '*_test.py' -v`
- On the shared OMP installation, also run the real migration regression: `HEAVY_GATE_INTEGRATION=1 HEAVY_GATE_TEST_GATE="$HOME/.claude/bin/heavy-gate" python3 -m unittest discover -s "$HOME/.omp/tests" -p 'heavy_gate_runtime_test.py' -v`. It uses the real shared slots with a low-memory process; allow it to wait for capacity, never wrap it in `timeout`.
- Keep Claude's PreToolUse JSON protocol; OMP's extension/Eval APIs and timeout fields differ.
- The hook's deny message points at `~/.claude/CLAUDE.md`; keep the two consistent.
