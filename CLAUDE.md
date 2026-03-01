# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

pty-mgr is a PTY session manager for programmatic terminal control. It spawns
commands in real pseudo-terminals, emulates the screen buffer with xterm, and
exposes session management via a daemon over Unix sockets.

Two-layer architecture:
- `lib/pty-bridge.py` - Python script that allocates a real PTY via `pty.openpty()`,
  forks the child process, and bridges stdin/stdout. Zero native deps, just stdlib.
  Reports child PID on stderr as `PID:<n>`, exit code as `EXIT:<n>`.
- `lib/pty-manager.mjs` - Node.js module (~1400 lines, single file). Contains:
  - `PtySession` class: wraps a bridge process + `@xterm/headless` Terminal.
    Feeds PTY output into xterm so `capture()` returns rendered screen state
    (escape codes resolved, cursor movements applied).
  - `PtyManager` class: session registry. spawn/sendKeys/capture/kill/waitFor/waitForExit.
    Exported for library use: `import { PtyManager } from 'pty-mgr'`
  - Daemon: Unix socket server (JSON-over-newline protocol). Holds sessions persistently.
    Socket at `~/.pty-manager/<name>.sock`. Supports `attach` (raw streaming mode).
  - CLI: full command parser with aliases. Entry point `bin/pty-mgr` (also `p`).

## Commands

```
npm install                          # install deps (@xterm/headless)
node bin/pty-mgr demo                # self-test, no daemon needed
npm run demo                         # same
npm run build                        # bun compile to dist/pty-mgr
```

No test framework is set up. The `demo` command is the current smoke test.

## CLI Usage (after `npm link` or direct)

```
p daemon                             # start daemon (forks to background)
p daemon @myproject                  # named daemon (isolated sessions)
p spawn <name> [cmd] [args...]       # create session
p send <name> <text>                 # send text + enter
p capture <name> [lines]             # get rendered screen
p attach <name>                      # interactive mode (ctrl-] detach)
p list                               # list sessions
p kill <name|all|glob*>              # kill sessions
p stop [all]                         # stop daemon(s)
```

Aliases: n/new=spawn, s=send, c/cap=capture, k=kill, l/ls=list,
a=attach, st=status, r/rm=remove, d=daemon, cfg=config, x=stop

## Key Design Decisions

- Single .mjs file for the entire Node side. No build step for dev, just run it.
- Python bridge instead of node-pty: avoids native addon compilation entirely.
  Requires python3 on PATH. Bridge path resolved via `findBridge()` or `PTY_BRIDGE` env.
- `@xterm/headless` does terminal emulation so capture() returns what you'd see on
  screen, not raw bytes. Scrollback default 5000 lines.
- Daemon uses newline-delimited JSON over Unix socket. `attach` command switches
  connection to raw streaming mode (bidirectional).
- Session names support glob patterns for bulk operations: `kill refa*`, `capture all`.
- `cap-on-send` config: when enabled, every `send` command returns a capture after 1s delay.
- Env var whitelist in daemon spawn to prevent injection (PATH, HOME, API keys, etc).
- Socket permissions set to 0o600 (owner-only).
- ESM throughout (`"type": "module"` in package.json).
