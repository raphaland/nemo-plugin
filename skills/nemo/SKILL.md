---
name: nemo
description: "Control Nemo, a macOS terminal with agent-aware sessions. Use only when the user explicitly mentions Nemo or asks to inspect or control its terminals, panes, or agents. Do not use merely because a task could benefit from a background terminal or parallel work."
---

# Nemo

Nemo organizes terminals into spaces and panes, detects coding agents
running inside them (working / blocked / idle), and exposes the current
session through a control socket and the `nemo-cli` binary.

Before issuing any control command, verify Nemo is running and the
socket exists:

```bash
CLI="/Applications/Nemo.app/Contents/MacOS/nemo-cli"
test -x "$CLI" && test -S "$HOME/Library/Application Support/nemo/control.sock"
```

If the check fails, say that Nemo is not available and stop.

## Commands

```bash
"$CLI" list
```

Every terminal and pane, with short ids, the running process, and the
detected agent state. Pane ids (8-hex prefixes) address the other
commands.

```bash
"$CLI" read --panel <id> --lines 40      # a pane's visible screen
"$CLI" send --panel <id> "npm test"      # type + run in a pane
"$CLI" send --panel <id> --raw "y"       # type without newline
"$CLI" split [--down]                    # split the focused terminal
"$CLI" new --space Work --cwd ~/api      # new terminal in a space
"$CLI" new --worktree                    # new git worktree + terminal
"$CLI" new --cwd ~/api --command "npm run dev"  # new terminal running a command
"$CLI" wait --panel <id> --until blocked --timeout 300
"$CLI" wait --panel <id> --until command # exits with the command's status
"$CLI" notify                            # ask for the user's attention
"$CLI" icon [--use theme] [--case-top '#RRGGBB'] [--reset]  # Dock icon and its colors
"$CLI" install-cli                       # symlink as `nemo` on PATH
"$CLI" install-hooks                     # exact Claude Code state via hooks
"$CLI" install-mcp                       # register nemo as MCP tools

```

`new --worktree` creates a git worktree on a fresh branch beside the
repository (`<repo>-worktrees/<branch>`) and opens a terminal in it, so
parallel agents do not fight over one checkout. It prints
`branch<TAB>path` before the panel id.

`new --command CMD` spawns the fresh terminal's shell already running
CMD (then drops into an interactive shell). Because it runs code, it
clears the same one-per-run approval prompt as `send`; a denial creates
no terminal.

The first `send` of each app run asks the user to allow it, since any
local process can reach the socket. Expect a prompt, and do not retry in
a loop if it is denied.

`wait` blocks until the pane's detected agent state matches
(`blocked` | `working` | `idle`); exit code 2 means timeout, and the
final state is printed either way.

`wait --until command` waits for a shell command in the pane to finish,
prints its exit status (`none` if the shell sent none) and exits with
it, or 124 on timeout. After a `send`, it waits for the command that
send typed, even if it already finished, never one that was already
running. It needs shell integration in that pane (on by default for zsh
and fish); without it, it times out.

## Typical patterns

Watch a sibling agent and take over when it needs input:

```bash
"$CLI" wait --panel 9d6bcc93 --until blocked && "$CLI" read --panel 9d6bcc93 --lines 20
```

Fan work out to a new pane and collect the result:

```bash
panel=$("$CLI" split --down)
"$CLI" send --panel "$panel" "cargo test 2>&1 | tail -5"
if "$CLI" wait --panel "$panel" --until command --timeout 600; then
    echo "tests passed"
fi
"$CLI" read --panel "$panel" --lines 10
```

## Wire protocol

The CLI is a thin client over a unix socket at
`~/Library/Application Support/nemo/control.sock`; talk to it directly
when a shell-out is awkward. Two formats, chosen by the first byte of the
connection:

- **JSON lines** (first byte `{`): one JSON object per line, replies are
  newline-terminated JSON.
- **MessagePack** (anything else): each message is a 4-byte big-endian
  length prefix followed by a MessagePack map; replies use the same
  framing.

Request keys mirror the CLI: `method`, `panel`, `terminal`, `direction`,
`space`, `cwd`, `text`, `lines`, `worktree`, `until`, `timeout`. Errors
come back as `{"error": "..."}`.

Two additional surfaces exist and are usually better than raw sockets:

- **MCP**: `nemo-mcp` (same directory as the CLI) exposes every
  command above as MCP tools (`nemo_list`, `nemo_send`, …);
  `install-mcp` registers it with Claude Code. Prefer these tools over
  shelling out when they are available in your session.
- **`agent-event`** (`agent`, `event`, `panel`, `session`, `cwd`,
  `message`, `detail`): how Claude Code hooks report lifecycle state;
  wired by `install-hooks`, not meant to be called by agents. Shells
  spawned by Nemo carry `NEMO_PANE_ID` — your own pane's id —
  which those hooks use for exact attribution.

## Cautions

- `send` types into a live terminal the user can see; never send
  destructive commands without the user's explicit request.
- `read` returns whatever is on screen, which may include secrets the
  user has displayed; do not exfiltrate it.
- Pane ids change across app restarts; always start from `list`.
- Worktrees are removed only with the user's confirmation, and never
  when they contain uncommitted work.
