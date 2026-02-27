# iterm-auto-upload

Drag-and-drop files into an iTerm2 window to automatically SCP upload them to a remote host and paste the remote path into [Claude Code](https://claude.ai/code) CLI — with auto-attach support.

## Problem

When using Claude Code over SSH (with or without tmux), there's no built-in way to share local files with the remote session. You'd have to manually `scp` files, then type the remote path into Claude Code.

This project makes it a single drag-and-drop: files are uploaded and auto-attached as context in Claude Code.

## How It Works

A typical session looks like this:

```
Local: iTerm2 ── SSH ──▶ Remote: tmux ──▶ Claude Code
```

When you drag a file onto the iTerm2 window:

```
1. iTerm2 intercepts the drop, launches bin/iterm-upload as a coprocess
2. The script checks two iTerm2 variables:
   - jobName == "ssh"           → confirms this is an SSH session
   - autoName contains "Claude Code"  → confirms Claude Code is running
3. Parses the SSH target from `ps -t <tty>`
4. Uploads the file via scp to the remote /tmp/iterm-upload/
5. Outputs the remote absolute path wrapped in bracketed paste sequences
6. Claude Code receives the bracketed paste and auto-attaches the file ✓
```

When not in an SSH + Claude Code session, the script falls back to pasting local file paths (default iTerm2 behavior).

### The detection challenge: `autoName` and terminal titles

The key to Claude Code detection is iTerm2's `autoName` variable, which reflects the terminal title. Claude Code sets the terminal title via **OSC 0** escape sequences (`\x1B]0;<title>\x07`):

| Event | Title (= autoName) |
|-------|---------------------|
| Session start | `✳ Claude Code` |
| AI detects conversation topic | `✳ <2-3 word summary>` (e.g., `✳ Fix auth bug`) |
| User runs `/rename xxx` | `✳ xxx` |
| Thinking animation | `⠂ <topic>` / `⠐ <topic>` (960ms cycle) |

**The problem:** The detection script matches `"Claude Code"` in `autoName`. This works at session start (`✳ Claude Code`), but Claude Code quickly auto-renames the title to a topic summary. Once renamed to something like `✳ Fix auth bug`, `autoName` no longer contains "Claude Code" and all subsequent drag-and-drop events fall back to local paths.

**The solution (for tmux users):** A shell wrapper marks the tmux pane when Claude Code launches, and tmux's `set-titles-string` conditionally appends `- Claude Code` to the title. This ensures `autoName` always contains "Claude Code" — for example, `✳ Fix auth bug - Claude Code` — regardless of topic changes. See [step 3](#3-tmux-title-with-claude-code-suffix-remote-machine) for details.

### How auto-attach works

Claude Code's TUI uses **bracketed paste mode** to detect file paths. Bracketed paste wraps pasted content in `\e[200~...\e[201~` escape sequences. When Claude Code receives a bracketed paste containing an **absolute file path** that exists on the filesystem, it auto-attaches the file as context.

Requirements:
- Path must be **absolute** (e.g., `/home/user/file.png`). `~/` paths won't trigger it.
- File must **exist** on the remote filesystem at the time of paste.
- Must arrive via **bracketed paste**, not keyboard input.

The coprocess stdout is normally injected as keyboard input. By wrapping the path in bracketed paste sequences, we simulate a paste event that Claude Code recognizes.

## Setup

### 1. Clone the repository (local machine)

```bash
git clone https://github.com/wake/iterm-auto-upload.git
```

### 2. Configure iTerm2 fileDropCoprocess (local machine)

This replaces iTerm2's default "paste file path on drop" with our upload script.

Go to **iTerm2 → Settings → Advanced** → search for **"file drop"**, set:

```
/path/to/iterm-auto-upload/bin/iterm-upload /tmp/iterm-upload \(jobName) "\(autoName)" \(tty) \(filenames)
```

Replace `/path/to/iterm-auto-upload` with your clone path. `/tmp/iterm-upload` is the remote upload directory — customize as needed.

Or via `defaults write` (quit iTerm2 first):

```bash
defaults write com.googlecode.iterm2 fileDropCoprocess \
  -string '/path/to/iterm-auto-upload/bin/iterm-upload /tmp/iterm-upload \(jobName) "\(autoName)" \(tty) \(filenames)'
```

**How it works:** iTerm2 launches the script as a coprocess on drag-and-drop, passing interpolated string variables:

| Variable | Value | Purpose |
|----------|-------|---------|
| `\(jobName)` | Foreground process name (e.g., `ssh`) | Detect SSH session |
| `\(autoName)` | Session name from terminal title | Detect Claude Code |
| `\(tty)` | Local TTY device path | Find SSH process via `ps -t` |
| `\(filenames)` | Shell-quoted dropped file paths | Files to upload |

The script's stdout is injected into the terminal as input — this is how the remote path reaches Claude Code.

### 3. tmux title with "Claude Code" suffix (remote machine)

> **Skip this step if you don't use tmux.** Without tmux, Claude Code's OSC 0 title reaches iTerm2 directly and detection works at session start. However, detection will still break after topic auto-rename — see [Known Limitations](#known-limitations).

This step solves two problems at once:

1. **Title passthrough** — tmux absorbs OSC 0 by default; these settings forward pane titles to iTerm2
2. **Persistent detection** — appends `- Claude Code` to the title so `autoName` always matches, even after topic rename

#### 3a. Shell wrapper (remote `~/.zshrc` or `~/.bashrc`)

Replace your `claude` alias with a function:

```bash
unalias claude 2>/dev/null

claude() {
  [ -n "$TMUX" ] && tmux set-option -p @is_claude 1
  /path/to/claude "$@"
  [ -n "$TMUX" ] && tmux set-option -pu @is_claude
}
```

Replace `/path/to/claude` with your Claude Code binary path (e.g., `~/.local/bin/claude`).

This sets a tmux pane-level user option `@is_claude=1` while Claude Code is running, and unsets it on exit. The `[ -n "$TMUX" ]` guard makes the function safe to use outside tmux.

#### 3b. tmux configuration (remote `~/.tmux.conf`)

```bash
set -g set-titles on
set -g set-titles-string '#{?#{==:#{@is_claude},1},#{pane_title} - Claude Code,#{pane_title}}'
set -g allow-rename on
```

Then reload: `tmux source ~/.tmux.conf`

| Setting | Purpose |
|---------|---------|
| `set-titles on` | Enables tmux to forward titles to the outer terminal (iTerm2) |
| `set-titles-string` | Formats the title before forwarding — adds `- Claude Code` suffix when `@is_claude` is set |
| `allow-rename on` | Allows Claude Code's OSC 0 to update the pane title inside tmux |

The `set-titles-string` uses tmux conditional format:

```
#{?#{==:#{@is_claude},1}, #{pane_title} - Claude Code, #{pane_title}}
     ╰─ condition ─╯      ╰──── if true ────╯          ╰─ if false ─╯
```

The full title flow:

```
Claude Code sends OSC 0 "✳ Fix auth bug"
  → tmux stores it as pane_title
  → set-titles-string appends " - Claude Code" (because @is_claude=1)
  → tmux sends OSC 0 "✳ Fix auth bug - Claude Code" to outer terminal
  → SSH forwards it to iTerm2
  → iTerm2 sets autoName = "✳ Fix auth bug - Claude Code"
  → autoName contains "Claude Code" ✓ → detection works
```

Result in the iTerm2 tab bar:

| Pane state | Tab title |
|------------|-----------|
| Claude Code idle | `✳ Fix auth bug - Claude Code` |
| Claude Code thinking | `⠂ Fix auth bug - Claude Code` |
| Normal shell (same tmux) | `bash` |
| After Claude Code exits | `bash` |

### 4. SSH key authentication (recommended)

The script makes two SSH connections per upload (`ssh` for `mkdir -p`, `scp` for the file). Interactive authentication (passwords, 2FA) will fail silently.

Use SSH key authentication, or enable `ControlMaster` in `~/.ssh/config`:

```
Host *
  ControlMaster auto
  ControlPath ~/.ssh/sockets/%r@%h-%p
  ControlPersist 600
```

This reuses your existing SSH connection for uploads.

## Known Limitations

- **SSH target parsing** — Parses `ps -t <tty>` output. SSH via wrapper scripts may not be detected.
- **SSH options** — Unrecognized options with arguments could misidentify the target.
- **Port detection** — Only from explicit `-p PORT`. Ports in `~/.ssh/config` work via scp config inheritance.
- **File overwrite** — Same-name files overwrite silently (no namespacing).
- **Non-tmux title rename** — Without tmux step 3, detection breaks after Claude Code auto-renames the topic. There is no workaround for direct SSH (no tmux) because iTerm2 provides no way to intercept or transform OSC 0 titles before they become `autoName`.

## Troubleshooting

All operations are logged to `/tmp/iterm-upload.log`:

```bash
cat /tmp/iterm-upload.log
```

Common issues:

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Always pastes local paths | `autoName` doesn't contain "Claude Code" | Check tmux title settings (step 3) |
| Upload fails silently | SSH auth requires interaction | Use SSH keys or ControlMaster (step 4) |
| Works once then stops | Claude Code renamed the topic | Apply the tmux suffix wrapper (step 3) |
| Log shows "Cannot find ssh process" | SSH not visible in `ps -t` | Check if SSH is behind a wrapper |

## Comparison with iTerm2 Built-in Option+Drag SCP

| | iterm-auto-upload | Built-in Option+Drag SCP |
|---|---|---|
| Trigger | Direct drag (no modifier key) | Hold Option + drag |
| Prerequisite | Local script only | Remote Shell Integration required |
| Upload directory | `/tmp/iterm-upload/` | Remote working directory |
| Path pasted | Yes (auto-attach in Claude Code) | No path pasted |
| Environment detection | Auto-detect SSH + Claude Code | None |
| Non-SSH fallback | Paste local path | No action |
