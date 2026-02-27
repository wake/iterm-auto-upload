# iterm-auto-upload

Drag-and-drop files into an iTerm2 window to automatically SCP upload them to a remote host and paste the remote path into [Claude Code](https://claude.ai/code) CLI — with auto-attach support.

## How It Works

```
Drag file into iTerm2
  → iTerm2 triggers fileDropCoprocess
  → Script detects SSH + Claude Code environment
  → scp uploads file to remote /tmp/iterm-upload/
  → Outputs absolute path with bracketed paste sequence
  → Claude Code auto-attaches the file ✓
```

When not in an SSH + Claude Code session, files are pasted as local paths (default iTerm2 behavior).

### Environment Detection

| Scenario | jobName | autoName | Behavior |
|----------|---------|----------|----------|
| Local shell | `bash`/`zsh` | `bash` | Paste local path |
| Local Claude Code | `node` | `✳ Claude Code` | Paste local path |
| SSH → shell | `ssh` | `bash` | Paste local path |
| SSH → Claude Code | `ssh` | `✳ Claude Code` | **Upload + paste remote path** |
| SSH → tmux → Claude Code | `ssh` | `✳ Claude Code` | **Upload + paste remote path** |

### Claude Code Auto-Attach Mechanism

Claude Code's TUI detects file paths through **bracketed paste mode** — a terminal protocol where pasted content is wrapped in `\e[200~...\e[201~` escape sequences. When Claude Code receives a bracketed paste containing an **absolute file path** that exists on the filesystem, it automatically attaches the file as context.

Key requirements for auto-attach:
- Path must be **absolute** (e.g., `/home/user/file.png`). Paths starting with `~/` will **not** trigger attach.
- The file must **exist** on the remote filesystem at the time of paste.
- Content must arrive via **bracketed paste**, not as keyboard input.

The coprocess stdout is normally injected as keyboard input (which would not trigger attach). By wrapping the output in bracketed paste sequences, we simulate a paste event that Claude Code recognizes.

## Setup

### 1. Clone the repository

```bash
git clone https://github.com/wake/iterm-auto-upload.git
```

### 2. Configure iTerm2 fileDropCoprocess

Go to **iTerm2 → Settings → Advanced** → search for **"file drop"**, and set the value to:

```
/path/to/iterm-auto-upload/bin/iterm-upload \(jobName) "\(autoName)" \(tty) /tmp/iterm-upload \(filenames)
```

Replace `/path/to/iterm-auto-upload` with the actual path where you cloned the repository. You can customize `/tmp/iterm-upload` to any remote directory you prefer.

Or via `defaults write` (quit iTerm2 first):

```bash
defaults write com.googlecode.iterm2 fileDropCoprocess \
  -string '/path/to/iterm-auto-upload/bin/iterm-upload \(jobName) "\(autoName)" \(tty) /tmp/iterm-upload \(filenames)'
```

### 3. tmux title passthrough (if using tmux on remote)

For Claude Code detection to work through tmux, add to the remote `~/.tmux.conf`:

```bash
set -g set-titles on
set -g set-titles-string "#{pane_title}"
set -g allow-rename on
```

## Known Limitations

### SSH target parsing

The script parses the SSH command from `ps -t <tty>`. This may fail if:
- The SSH process has exited or is not visible in `ps`
- SSH is invoked through a wrapper script (the command won't start with `ssh `)
- ProxyJump or ProxyCommand chains where the final target isn't directly visible

### SSH options parsing

The script skips known SSH options that take arguments (`-o`, `-p`, `-i`, `-l`, `-L`, `-R`, `-D`, `-J`, `-W`, `-F`, `-e`, `-b`, `-c`, `-m`, `-S`, `-w`, `-E`). Unrecognized options with arguments could cause the target to be misidentified.

### Port detection

Custom ports are only detected from explicit `-p PORT` in the SSH command. If the port is defined in `~/.ssh/config`, scp will rely on the same config (which usually works correctly).

### File overwrite

Files uploaded to `/tmp/iterm-upload/` are not namespaced. Uploading a file with the same name as an existing file will overwrite it silently.

### SSH authentication

The script makes two SSH connections (one for `mkdir -p` + `pwd`, one for `scp`). If your SSH setup requires interactive authentication (password prompt, 2FA), these will fail silently and fall back to local paths. Using SSH key authentication or `ControlMaster` multiplexing is recommended.

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

---

# iterm-auto-upload（中文）

拖曳檔案到 iTerm2 視窗，自動 SCP 上傳到遠端主機，並在 [Claude Code](https://claude.ai/code) CLI 中自動附加（auto-attach）為上下文。

## 運作原理

```
拖曳檔案到 iTerm2
  → iTerm2 觸發 fileDropCoprocess
  → 腳本偵測 SSH + Claude Code 環境
  → scp 上傳檔案到遠端 /tmp/iterm-upload/
  → 輸出絕對路徑 + bracketed paste 序列
  → Claude Code 自動附加檔案 ✓
```

非 SSH + Claude Code 環境時，直接貼上本地路徑（iTerm2 預設行為）。

### Claude Code 自動附加機制

Claude Code 的 TUI 透過 **bracketed paste mode** 偵測檔案路徑 — 終端協議會將貼上的內容包在 `\e[200~...\e[201~` 跳脫序列中。當 Claude Code 收到包含**絕對檔案路徑**的 bracketed paste，且該檔案存在於檔案系統上，就會自動附加為 context。

自動附加的條件：
- 路徑必須是**絕對路徑**（如 `/home/user/file.png`）。`~/` 開頭**不會**觸發。
- 檔案在貼上時必須已**存在**於遠端。
- 內容必須透過 **bracketed paste** 到達，而非鍵盤輸入。

Coprocess 的 stdout 預設以鍵盤輸入方式注入（不會觸發 attach）。透過在輸出中加上 bracketed paste 序列，我們模擬了貼上事件，讓 Claude Code 能夠辨識。

## 設定方式

### 1. Clone 本專案

```bash
git clone https://github.com/wake/iterm-auto-upload.git
```

### 2. 設定 iTerm2 fileDropCoprocess

前往 **iTerm2 → Settings → Advanced** → 搜尋 **"file drop"**，設定為：

```
/path/to/iterm-auto-upload/bin/iterm-upload \(jobName) "\(autoName)" \(tty) /tmp/iterm-upload \(filenames)
```

將 `/path/to/iterm-auto-upload` 替換為實際 clone 路徑。`/tmp/iterm-upload` 可替換為任何你偏好的遠端目錄。

### 3. tmux 標題穿透（遠端使用 tmux 時）

在遠端 `~/.tmux.conf` 加入：

```bash
set -g set-titles on
set -g set-titles-string "#{pane_title}"
set -g allow-rename on
```

## 已知限制

- **SSH target 解析**：從 `ps -t <tty>` 解析，透過 wrapper script 啟動的 SSH 可能無法辨識。
- **SSH options 解析**：未列入的帶參數 option 可能導致 target 誤判。
- **Port 偵測**：僅從 SSH command 中的 `-p PORT` 偵測。`~/.ssh/config` 中定義的 port 會由 scp 自動使用。
- **檔案覆蓋**：同名檔案會直接覆蓋，無命名空間隔離。
- **SSH 驗證**：需要兩次 SSH 連線（mkdir + scp），互動式驗證（密碼、2FA）會導致失敗並 fallback。建議使用 SSH key 或 `ControlMaster`。

## 除錯

所有操作記錄在 `/tmp/iterm-upload.log`：

```bash
cat /tmp/iterm-upload.log
```
