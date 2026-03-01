# pty-mgr

PTY session manager with terminal emulation for programmatic session control.

Spawn commands in real pseudo-terminals, capture rendered screen output (not raw
bytes), and manage sessions through a persistent daemon. Single binary, no
external dependencies.

## How It Works

`Bun.spawn({ terminal })` allocates a native PTY for each session. An
`@xterm/headless` terminal emulator parses escape codes, cursor movements, and
screen redraws so `capture()` returns exactly what you'd see on screen.

Compiles to a single self-contained binary via `bun build --compile`.

## Install

### Binary (no dependencies)

```
curl -fsSL https://raw.githubusercontent.com/maarco/pty-mgr/main/install.sh | sh
```

Installs to `~/.pty-mgr/bin/` and adds to PATH. Works on Linux and macOS (x64, arm64).

### Bun package (for library use)

```
bun add pty-mgr
```

Requires [Bun](https://bun.sh) runtime.

## Quick Start

### As a library

```js
import { PtyManager } from 'pty-mgr';

const mgr = new PtyManager();
mgr.spawn('my-session', 'zsh', [], { cols: 120, rows: 30 });
mgr.sendKeys('my-session', 'echo hello\r');

// wait for output, then capture rendered screen
setTimeout(() => {
  console.log(mgr.capture('my-session', 5));
  mgr.kill('my-session');
}, 1000);
```

### As a CLI

```
# start the daemon (forks to background)
p daemon

# named daemon for isolated environments
p daemon @myproject

# spawn a session
p spawn agent-1 claude --print

# send keystrokes
p send agent-1 "fix the login bug"

# capture rendered screen (last 20 lines)
p capture agent-1 20

# attach interactively (ctrl-] to detach)
p attach agent-1

# bulk operations with globs
p capture all 50
p kill refa*

# stop daemon
p stop
```

### CLI Aliases

| Short | Full    | Short | Full    |
|-------|---------|-------|---------|
| n/new | spawn   | k     | kill    |
| s     | send    | l/ls  | list    |
| c/cap | capture | r/rm  | remove  |
| a     | attach  | d     | daemon  |
| st    | status  | cfg   | config  |
| i     | info    | x     | stop    |

## Example: Orchestrating Claude

Spawn Claude inside a managed PTY and control it programmatically.

### CLI

```bash
p daemon
p spawn claude-1 claude
p send claude-1 "fix the login bug in src/auth.ts"
sleep 5
p capture claude-1 50       # see what Claude is doing
p attach claude-1            # jump in interactively (ctrl-] to detach)
p kill claude-1
```

### Programmatic (parallel agents)

```js
import { PtyManager } from 'pty-mgr';

const mgr = new PtyManager();

// launch 3 Claude agents in parallel
const agents = ['auth-fix', 'api-tests', 'docs-update'];
for (const name of agents) {
  mgr.spawn(name, 'claude', ['--print'], { cols: 120, rows: 40 });
}

mgr.sendKeys('auth-fix', 'fix the login bug in src/auth.ts\r');
mgr.sendKeys('api-tests', 'write tests for the /users endpoint\r');
mgr.sendKeys('docs-update', 'update the API docs in README.md\r');

// poll until all agents finish
for (const name of agents) {
  await mgr.waitFor(name, /[✔✓]|completed|done/i, 120000);
  console.log(`${name} done:\n${mgr.capture(name, 10)}\n`);
}

mgr.destroyAll();
```

## Daemon Protocol

The daemon listens on a Unix socket at `~/.pty-manager/<name>.sock`.
Communication is newline-delimited JSON:

```json
{"cmd": "spawn", "name": "agent-1", "args": {"cmd": "zsh"}}
{"cmd": "send", "name": "agent-1", "args": {"text": "echo hi\r"}}
{"cmd": "capture", "name": "agent-1", "args": {"lines": 20}}
{"cmd": "list"}
{"cmd": "kill", "name": "agent-1"}
{"cmd": "shutdown"}
```

The `attach` command switches the connection to raw streaming mode for
interactive use.

## Configuration

```
p config screen 120x40       # default terminal size for new sessions
p config cap-on-send on      # return capture with every send command
```

## Logging

```
p spawn agent-1 --log claude   # spawn with auto-logging (jsonl)
p log agent-1 on jsonl         # start logging an existing session
p log agent-1 off              # stop logging
```

Formats: `jsonl` (timestamped events), `raw` (PTY bytes), `rendered` (screen snapshots).

## Build

```
bun run build    # compiles to dist/pty-mgr (single binary, ~60MB)
```

## License

MIT
