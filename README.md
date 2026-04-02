# claude-code-rc-monitor

Keep a [Claude Code](https://docs.anthropic.com/en/docs/claude-code) remote control (`/rc`) session alive automatically.

Claude Code's [Remote Control](https://code.claude.com/docs/en/remote-control) feature lets you access a local CLI session from `claude.ai/code` or a mobile device. However, RC sessions frequently disconnect — sometimes every few minutes — with no reliable way to reconnect. This is a [known server-side issue](#why-this-exists) affecting all platforms. This utility works around it by monitoring a Claude Code session inside tmux and automatically reconnecting `/rc` with the same session name.

## Requirements

- **Python 3** (standard library only — no pip installs)
- **tmux** (the Claude Code session must be running inside a tmux pane)
- **Claude Code CLI** (`claude` binary on PATH)
- **macOS or Linux**

## Installation

### One-liner (curl)

Download and run directly — no clone needed:

```bash
curl -fsSL https://raw.githubusercontent.com/iceraj/cloude-code-remote-control-restart/main/claude-code-rc-monitor -o claude-code-rc-monitor && chmod +x claude-code-rc-monitor
```

Or install to your PATH in one step:

```bash
curl -fsSL https://raw.githubusercontent.com/iceraj/cloude-code-remote-control-restart/main/claude-code-rc-monitor -o ~/.local/bin/claude-code-rc-monitor && chmod +x ~/.local/bin/claude-code-rc-monitor
```

### Clone the repo

```bash
git clone https://github.com/iceraj/cloude-code-remote-control-restart.git
cd cloude-code-remote-control-restart

# Make it executable (already set, but just in case)
chmod +x claude-code-rc-monitor

# Optionally symlink to a directory on your PATH
ln -s "$(pwd)/claude-code-rc-monitor" ~/.local/bin/claude-code-rc-monitor
```

## Usage

### Quick start

1. Start a tmux session and launch Claude Code in one pane:
   ```bash
   tmux new-session -s myproject
   claude --permission-mode acceptEdits
   ```

2. Open a second pane in the same tmux session and run the monitor:
   ```bash
   # Ctrl+B % (split pane), then:
   ./claude-code-rc-monitor my-session-name
   ```

3. The monitor will:
   - Find the Claude Code instance in the tmux session
   - Disconnect any existing `/rc` session
   - Start `/rc my-session-name`
   - Sleep for 1 hour, then repeat

### Refreshing on demand

While the monitor is running:

| Trigger | How |
|---------|-----|
| **Enter** | Press Enter in the monitor's terminal |
| **SIGUSR1** | `kill -USR1 <monitor-pid>` |
| **SIGHUP** | `kill -HUP <monitor-pid>` |
| **Timeout** | Automatic after 1 hour |

Any trigger disconnects the current `/rc` session and reconnects with the same name.

### Stopping

Press `Ctrl+C` in the monitor's terminal.

## How it works

```
┌─────────────────────────────────────────────┐
│ tmux session                                │
│                                             │
│  ┌─────────────────┐  ┌──────────────────┐  │
│  │ Pane 0          │  │ Pane 1           │  │
│  │ claude           │  │ rc-monitor       │  │
│  │ (interactive)   │◄─│ (sends /rc via   │  │
│  │                 │  │  tmux send-keys) │  │
│  └─────────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────┘
```

1. **Detect** — Enumerates all panes in the current tmux session and matches shell child processes against `claude` using `ps`.
2. **Validate** — Exits with an error if zero or more than one Claude instance is found.
3. **Wait for idle** — Reads the pane via `tmux capture-pane` and polls until Claude is not actively thinking (checks for absence of "esc to interrupt" in the status bar). Times out after 5 minutes.
4. **Disconnect** — Sends `/remote-control` to the Claude pane and reads the pane output via `tmux capture-pane` to determine state:
   - **RC was active** → Menu appears with "Disconnect this session" — navigates Up+Up+Enter to disconnect, then verifies "Remote Control disconnected" in pane output.
   - **RC was not active** → `/remote-control` auto-starts a session ("Remote Control connecting") — detects this, disconnects the auto-started session, then proceeds.
   - Logs warnings if expected output is not found.
5. **Connect** — Sends `/rc <session-name>` to start a fresh remote control session, then reads the pane to verify "Remote Control connecting" or "Remote Control active".
6. **Wait** — Sleeps up to 1 hour, waking on Enter, SIGUSR1, or SIGHUP.
7. **Loop** — Returns to step 1.

## Constraints

- **One Claude per tmux session** — The monitor expects exactly one `claude` process in the tmux session. If you have multiple, it exits with an error listing them.
- **tmux required** — The monitor uses `tmux send-keys` to interact with Claude's TUI. It cannot operate outside tmux.
- **No code execution** — The monitor only sends slash commands (`/remote-control`, `/rc`). It never executes code or modifies files in your project.

## Examples

### Dedicated remote control session

```bash
# Terminal 1: Start Claude in tmux
tmux new-session -s rc-server -d
tmux send-keys -t rc-server 'claude --permission-mode acceptEdits' Enter

# Terminal 2: Monitor from another pane
tmux split-window -t rc-server
tmux send-keys -t rc-server './claude-code-rc-monitor my-project' Enter
```

### Refresh from a cron job or script

```bash
# Find the monitor PID and signal it
pkill -USR1 -f 'claude-code-rc-monitor'
```

## Why this exists

Claude Code's Remote Control feature has a known reliability problem: sessions disconnect frequently and unpredictably, with no built-in auto-reconnect. This has been confirmed across macOS, Linux, and Windows/WSL2 — users have ruled out client-side causes (TCP keepalive tuning, DNS, VPN, different ISPs) and the consensus is that this is a server-side issue requiring a fix from Anthropic.

Making matters worse, after a disconnect the session often vanishes from the session list entirely, so there is no way to reconnect — only a full restart of `/rc` works. This tool automates that restart.

### Related issues

- [anthropics/claude-code#33041](https://github.com/anthropics/claude-code/issues/33041) — **"Remote Control disconnects frequently"** — Detailed bug report with 16+ upvotes. Confirmed across all platforms. Users tried TCP keepalive, DNS changes, VPNs — none helped.
- [anthropics/claude-code#28402](https://github.com/anthropics/claude-code/issues/28402) — **"Remote Control session not visible in session list, cannot reconnect"** — 22+ upvotes. After disconnect, sessions vanish from the list with no way to reconnect.
- [anthropics/claude-code#32819](https://github.com/anthropics/claude-code/issues/32819) — **"Remote Control keeps disconnecting unexpectedly"** — Same pattern on Linux: connection drops intermittently, reconnects briefly, then drops again.

## License

[MIT](LICENSE)
