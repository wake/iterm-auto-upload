# iterm-auto-upload

Drag-and-drop files into an iTerm2 window to automatically SCP upload them to a remote host and paste the remote path into [Claude Code](https://claude.ai/code) CLI — with auto-attach support.

## Problem

When using Claude Code over SSH (with or without tmux), there's no built-in way to share local files with the remote Claude Code session. You'd have to manually `scp` files, then type the remote path into Claude Code.

This project makes it a single drag-and-drop: files are uploaded and auto-attached as context in Claude Code.

## How It Works

```
Drag file into iTerm2
  → iTerm2 triggers fileDropCoprocess
  → Script detects SSH + Claude Code environment
  → scp uploads file to remote /tmp/iterm-upload/
  → Outputs absolute path wrapped in bracketed paste sequence
  → Claude Code auto-attaches the file ✓
```

When not in an SSH + Claude Code session, files are pasted as local paths (default iTerm2 behavior).

### Detection Logic

| Scenario | jobName | autoName | Behavior |
|----------|---------|----------|----------|
| Local shell | `bash`/`zsh` | `bash` | Paste local path |
| Local Claude Code | `node` | `✳ Claude Code` | Paste local path |
| SSH → shell | `ssh` | `bash` | Paste local path |
| SSH → Claude Code | `ssh` | `✳ topic name` | **Upload + paste remote path** |
| SSH → tmux → Claude Code | `ssh` | `✳ topic name` | **Upload + paste remote path** |

## Prerequisites

Four conditions must be met for auto-upload to work:

1. **iTerm2 fileDropCoprocess** — local hook that intercepts drag-and-drop events
2. **SSH session** — detected via `jobName == "ssh"`
3. **Claude Code running** — detected via `autoName` containing `"Claude Code"`
4. **tmux title passthrough** (if using tmux) — allows Claude Code's terminal title to propagate through tmux to iTerm2

### Why these conditions?

The script uses iTerm2's interpolated string variables to determine the environment:

- **`\(jobName)`** — the foreground process name. When SSH is active, this is `"ssh"`.
- **`\(autoName)`** — the session's automatic name, derived from the terminal title set by the running application. Claude Code sets this via OSC 0 escape sequence (`\x1B]0;<title>\x07`) to something like `✳ Claude Code` or `✳ <topic>`.
- **`\(tty)`** — the local TTY device, used to find the SSH process via `ps -t` and parse the remote host.

When all conditions are met (SSH session + Claude Code detected), the script uploads via `scp` and outputs the remote path. Otherwise, it falls back to pasting local paths.

### How Claude Code sets the terminal title

Claude Code automatically manages the terminal title using OSC 0 escape sequences:

| Event | Title set to |
|-------|-------------|
| Session start | `✳ Claude Code` |
| AI detects new topic | `✳ <2-3 word topic>` (e.g., `✳ Fix auth bug`) |
| `/rename xxx` | `✳ xxx` |
| Thinking (spinner) | `⠂ <topic>` / `⠐ <topic>` (alternating at 960ms) |

This title is what iTerm2 sees as `autoName`. The detection script matches on `"Claude Code"` in the initial title. Note that once the AI auto-renames the topic, `autoName` changes — but by then the session is already established.

**Relevant environment variable:** `CLAUDE_CODE_DISABLE_TERMINAL_TITLE=1` disables all title updates.

### How Claude Code auto-attaches files

Claude Code's TUI detects file paths through **bracketed paste mode** — a terminal protocol where pasted content is wrapped in `\e[200~...\e[201~` escape sequences. When Claude Code receives a bracketed paste containing an **absolute file path** that exists on the filesystem, it automatically attaches the file as context.

Requirements for auto-attach:
- Path must be **absolute** (e.g., `/home/user/file.png`). `~/` paths will **not** trigger it.
- The file must **exist** on the remote filesystem at the time of paste.
- Content must arrive via **bracketed paste**, not as regular keyboard input.

The coprocess stdout is normally injected as keyboard input (which would not trigger attach). By wrapping the output in bracketed paste sequences, we simulate a paste event that Claude Code recognizes.

## Setup

### 1. Clone the repository (local machine)

```bash
git clone https://github.com/wake/iterm-auto-upload.git
```

### 2. Configure iTerm2 fileDropCoprocess (local machine)

This is the hook that intercepts drag-and-drop events in iTerm2. It replaces the default "paste file path" behavior with our upload script.

Go to **iTerm2 → Settings → Advanced** → search for **"file drop"**, and set the value to:

```
/path/to/iterm-auto-upload/bin/iterm-upload /tmp/iterm-upload \(jobName) "\(autoName)" \(tty) \(filenames)
```

Replace `/path/to/iterm-auto-upload` with the actual path where you cloned the repository. `/tmp/iterm-upload` is the remote upload directory — customize as needed.

Or via `defaults write` (quit iTerm2 first):

```bash
defaults write com.googlecode.iterm2 fileDropCoprocess \
  -string '/path/to/iterm-auto-upload/bin/iterm-upload /tmp/iterm-upload \(jobName) "\(autoName)" \(tty) \(filenames)'
```

**How it works:** When you drag a file onto an iTerm2 window, iTerm2 launches the script as a coprocess, passing session metadata (`jobName`, `autoName`, `tty`) and the dropped file paths. The script's stdout is injected into the terminal as input.

### 3. tmux title passthrough (remote machine, if using tmux)

Claude Code sets the terminal title via OSC 0 escape sequences. For this title to reach iTerm2 through tmux, tmux must be configured to forward pane titles.

Add to the remote `~/.tmux.conf`:

```bash
set -g set-titles on
set -g set-titles-string "#{pane_title}"
set -g allow-rename on
```

Then reload: `tmux source ~/.tmux.conf`

**Why each setting:**

| Setting | Purpose |
|---------|---------|
| `set-titles on` | Enables tmux to update the outer terminal's title |
| `set-titles-string "#{pane_title}"` | Forwards the pane's title (set by Claude Code via OSC 0) to the outer terminal |
| `allow-rename on` | Allows applications inside tmux to set the pane title via escape sequences |

Without these settings, tmux absorbs the title change and iTerm2 never sees `autoName` containing "Claude Code", causing the script to fall back to local paths.

### 4. SSH key authentication (recommended)

The script makes two SSH connections per upload (one for `mkdir -p`, one for `scp`). Interactive authentication (password prompts, 2FA) will cause these to fail silently.

Recommended setup:
- Use SSH key authentication, or
- Enable `ControlMaster` multiplexing in `~/.ssh/config`:

```
Host *
  ControlMaster auto
  ControlPath ~/.ssh/sockets/%r@%h-%p
  ControlPersist 600
```

This reuses your existing SSH connection for the upload, avoiding extra authentication prompts.

## Known Limitations

- **SSH target parsing** — The script parses `ps -t <tty>` output. SSH invoked through wrapper scripts may not be detected.
- **SSH options parsing** — Unrecognized options with arguments could cause the target to be misidentified.
- **Port detection** — Only detected from explicit `-p PORT` in the SSH command. Ports in `~/.ssh/config` are handled by scp automatically.
- **File overwrite** — Same-name files are overwritten silently (no namespacing).
- **autoName timing** — After Claude Code auto-renames the topic, `autoName` may no longer contain "Claude Code". This only affects new drag-and-drop events; the initial session detection works because the default title is `✳ Claude Code`.

## Troubleshooting

All operations are logged to `/tmp/iterm-upload.log`:

```bash
cat /tmp/iterm-upload.log
```

On any error (SSH target parsing, mkdir, scp), the script falls back to pasting local file paths.

## Comparison with iTerm2 Built-in Option+Drag SCP

| | iterm-auto-upload | Built-in Option+Drag SCP |
|---|---|---|
| Trigger | Direct drag (no modifier key) | Hold Option + drag |
| Prerequisite | Local script only | Remote Shell Integration required |
| Upload directory | `/tmp/iterm-upload/` | Remote current working directory |
| Path pasted | Yes (auto-attach in Claude Code) | No path pasted |
| Environment detection | Auto-detect SSH + Claude Code | None |
| Non-SSH fallback | Paste local path | No action |
