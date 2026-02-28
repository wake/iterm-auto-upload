# iterm-auto-upload

## Problem

- No built-in way to share local files (images, documents) with Claude Code running on a remote machine over SSH.
- Want drag-and-drop upload **only** for SSH + Claude Code sessions; all other scenarios should behave as default (paste local path).

## How It Works

```
Drag file onto iTerm2 window
  → iTerm2 fires a custom coprocess (replaces default paste behavior)
  → Script checks: is this SSH? Is user.is_claude set to "1"?
  → Yes: scp upload to remote, output remote path in auto-attach format
  → No:  paste local path as usual
```

## Step 1. Intercept the Drop Event

iTerm2's **fileDropCoprocess** setting lets you replace the default drag-and-drop behavior with a custom script. When a file is dropped, iTerm2 launches the script as a **coprocess** — its stdout is injected into the terminal as input.

iTerm2 passes session metadata via [interpolated string](https://iterm2.com/documentation-scripting-fundamentals.html) variables:

| Variable | Example | Purpose |
|----------|---------|---------|
| `\(user.is_claude)` | `1` | User-defined variable — detect Claude Code |
| `\(jobName)` | `ssh` | Foreground process — detect SSH |
| `\(autoName)` | `✳ Claude Code` | Terminal title (kept for logging) |
| `\(tty)` | `/dev/ttys003` | TTY — find SSH process via `ps -t` |
| `\(filenames)` | `/Users/me/img.png` | Dropped file paths |

### Configuration (local machine)

**iTerm2 → Settings → Advanced** → search `"file drop"`, set:

```
/path/to/iterm-auto-upload/bin/iterm-upload "\(user.is_claude)" /tmp/iterm-upload \(jobName) "\(autoName)" \(tty) \(filenames)
```

Or via `defaults write` (quit iTerm2 first):

```bash
defaults write com.googlecode.iterm2 fileDropCoprocess \
  -string '/path/to/iterm-auto-upload/bin/iterm-upload "\(user.is_claude)" /tmp/iterm-upload \(jobName) "\(autoName)" \(tty) \(filenames)'
```

Replace `/path/to/iterm-auto-upload` with your clone path. `/tmp/iterm-upload` is the remote upload directory.

## Step 2. Upload Script

The script (`bin/iterm-upload`) runs on the **local machine** as an iTerm2 coprocess:

1. Checks `jobName == "ssh"` and `user.is_claude == "1"`
2. Parses SSH target (user@host, port) from `ps -t <tty>`
3. Runs `ssh mkdir -p` to ensure the remote directory exists
4. Uploads files via `scp`
5. Outputs remote absolute paths wrapped in bracketed paste sequences

On any failure, falls back to pasting local paths. All operations logged to `/tmp/iterm-upload.log`.

### Installation

```bash
git clone https://github.com/wake/iterm-auto-upload.git
```

The script requires **non-interactive SSH access** (key auth or ControlMaster). It makes two connections per upload (mkdir + scp). Recommended `~/.ssh/config`:

```
Host *
  ControlMaster auto
  ControlPath ~/.ssh/sockets/%r@%h-%p
  ControlPersist 600
```

## Step 3. Set Up Claude Code Wrapper

The script detects Claude Code by checking `user.is_claude`, an iTerm2 [user-defined variable](https://iterm2.com/documentation-scripting-fundamentals.html). This is set via an OSC 1337 escape sequence from the remote shell, and works reliably in both direct SSH and tmux sessions.

Add a shell wrapper on the **remote machine** (`~/.zshrc` or `~/.bashrc`):

```bash
claude() {
  printf '\033]1337;SetUserVar=%s=%s\007' "is_claude" "$(echo -n 1 | base64)"
  command claude "$@"
  printf '\033]1337;SetUserVar=%s=%s\007' "is_claude" "$(echo -n 0 | base64)"
}
```

For **tmux** sessions, wrap the escape sequence with DCS passthrough:

```bash
claude() {
  if [ -n "$TMUX" ]; then
    printf '\033Ptmux;\033\033]1337;SetUserVar=%s=%s\007\033\\' "is_claude" "$(echo -n 1 | base64)"
  else
    printf '\033]1337;SetUserVar=%s=%s\007' "is_claude" "$(echo -n 1 | base64)"
  fi
  command claude "$@"
  if [ -n "$TMUX" ]; then
    printf '\033Ptmux;\033\033]1337;SetUserVar=%s=%s\007\033\\' "is_claude" "$(echo -n 0 | base64)"
  else
    printf '\033]1337;SetUserVar=%s=%s\007' "is_claude" "$(echo -n 0 | base64)"
  fi
}
```

This sets `is_claude=1` when Claude Code starts and resets to `0` when it exits. iTerm2 makes this available as `\(user.is_claude)` in the fileDropCoprocess interpolated string.

## Step 4. Auto-Attach

Claude Code auto-attaches files when it receives an **absolute path** via **bracketed paste** (`\e[200~...\e[201~`).

- Path must be absolute (`/home/user/file.png`). `~/` won't work.
- File must exist on the remote filesystem.
- Must arrive as bracketed paste, not keyboard input.

The coprocess stdout is normally treated as keyboard input. The script wraps the path in `\e[200~...\e[201~` to simulate a paste event, triggering auto-attach.

## Known Limitations

| Issue | Detail |
|-------|--------|
| SSH parsing | Parses `ps -t <tty>`. Wrapper scripts or unusual SSH invocations may fail. |
| Port detection | Only from `-p PORT` flag. `~/.ssh/config` ports work via scp inheritance. |
| File overwrite | Same-name files overwrite silently. |
| SSH auth | Requires non-interactive auth (keys or ControlMaster). |

## Troubleshooting

```bash
cat /tmp/iterm-upload.log
```

| Symptom | Cause | Fix |
|---------|-------|-----|
| Always pastes local paths | `is_claude` not set | Check Claude wrapper (step 3) |
| Works in SSH but not tmux | DCS passthrough missing | Use tmux version of wrapper (step 3) |
| Upload fails silently | SSH needs password/2FA | Use SSH keys or ControlMaster |
| "Cannot find ssh process" | SSH behind a wrapper | Check `ps -t <tty>` manually |

---

# iterm-auto-upload（中文）

## 問題

- SSH 遠端使用 Claude Code 時，無法直接拖曳上傳圖片等檔案。
- 希望只在 SSH + Claude Code 環境下觸發上傳，其餘情境維持 iTerm2 預設行為（貼上本地路徑）。

## 運作方式

```
拖曳檔案到 iTerm2
  → iTerm2 觸發自訂 coprocess（取代預設貼上行為）
  → 腳本檢查：是否在 SSH 中？user.is_claude 是否為 "1"？
  → 是：scp 上傳到遠端，以 auto-attach 格式輸出遠端路徑
  → 否：照常貼上本地路徑
```

## Step 1. 攔截拖曳事件

iTerm2 的 **fileDropCoprocess** 設定可將拖曳行為替換為自訂腳本。腳本以 **coprocess** 執行，stdout 會作為輸入注入終端。

iTerm2 透過 interpolated string 變數傳入 session 資訊：

| 變數 | 範例 | 用途 |
|------|------|------|
| `\(user.is_claude)` | `1` | User-defined variable — 偵測 Claude Code |
| `\(jobName)` | `ssh` | 前景程序名 — 偵測 SSH |
| `\(autoName)` | `✳ Claude Code` | 終端標題（僅用於 log） |
| `\(tty)` | `/dev/ttys003` | TTY — 透過 `ps -t` 找到 SSH process |
| `\(filenames)` | `/Users/me/img.png` | 拖曳的檔案路徑 |

### 設定（本機）

**iTerm2 → Settings → Advanced** → 搜尋 `"file drop"`，設定為：

```
/path/to/iterm-auto-upload/bin/iterm-upload "\(user.is_claude)" /tmp/iterm-upload \(jobName) "\(autoName)" \(tty) \(filenames)
```

或用 `defaults write`（需先關閉 iTerm2）：

```bash
defaults write com.googlecode.iterm2 fileDropCoprocess \
  -string '/path/to/iterm-auto-upload/bin/iterm-upload "\(user.is_claude)" /tmp/iterm-upload \(jobName) "\(autoName)" \(tty) \(filenames)'
```

## Step 2. 上傳腳本

`bin/iterm-upload` 在**本機**以 iTerm2 coprocess 執行：

1. 檢查 `jobName == "ssh"` 且 `user.is_claude == "1"`
2. 從 `ps -t <tty>` 解析 SSH target（user@host、port）
3. `ssh mkdir -p` 建立遠端目錄
4. `scp` 上傳檔案
5. 輸出遠端絕對路徑，以 bracketed paste 格式包裝

失敗時 fallback 貼上本地路徑。日誌寫入 `/tmp/iterm-upload.log`。

### 安裝

```bash
git clone https://github.com/wake/iterm-auto-upload.git
```

需要**非互動式 SSH 連線**（key 認證或 ControlMaster），每次上傳會建立兩次 SSH 連線。建議 `~/.ssh/config`：

```
Host *
  ControlMaster auto
  ControlPath ~/.ssh/sockets/%r@%h-%p
  ControlPersist 600
```

## Step 3. 設定 Claude Code Wrapper

偵測邏輯：檢查 `user.is_claude` 這個 iTerm2 [user-defined variable](https://iterm2.com/documentation-scripting-fundamentals.html)。它透過 OSC 1337 escape sequence 從遠端 shell 設定，在直接 SSH 和 tmux 環境下都能正常運作。

在**遠端機器**的 `~/.zshrc` 或 `~/.bashrc` 加入 wrapper：

```bash
claude() {
  printf '\033]1337;SetUserVar=%s=%s\007' "is_claude" "$(echo -n 1 | base64)"
  command claude "$@"
  printf '\033]1337;SetUserVar=%s=%s\007' "is_claude" "$(echo -n 0 | base64)"
}
```

**tmux** 環境需用 DCS passthrough 包裝：

```bash
claude() {
  if [ -n "$TMUX" ]; then
    printf '\033Ptmux;\033\033]1337;SetUserVar=%s=%s\007\033\\' "is_claude" "$(echo -n 1 | base64)"
  else
    printf '\033]1337;SetUserVar=%s=%s\007' "is_claude" "$(echo -n 1 | base64)"
  fi
  command claude "$@"
  if [ -n "$TMUX" ]; then
    printf '\033Ptmux;\033\033]1337;SetUserVar=%s=%s\007\033\\' "is_claude" "$(echo -n 0 | base64)"
  else
    printf '\033]1337;SetUserVar=%s=%s\007' "is_claude" "$(echo -n 0 | base64)"
  fi
}
```

啟動 Claude Code 時設定 `is_claude=1`，結束時重設為 `0`。iTerm2 會將此變數以 `\(user.is_claude)` 提供給 fileDropCoprocess interpolated string。

## Step 4. 自動 Attach

Claude Code 在收到 **bracketed paste**（`\e[200~...\e[201~`）包裝的**絕對路徑**時，會自動 attach 該檔案。

- 必須是絕對路徑（`/home/user/file.png`），`~/` 無效。
- 檔案必須已存在於遠端。
- 必須是 bracketed paste，非鍵盤輸入。

Coprocess stdout 預設作為鍵盤輸入。腳本用 `\e[200~...\e[201~` 包裝路徑來模擬貼上事件，觸發 auto-attach。

## 已知限制

| 問題 | 說明 |
|------|------|
| SSH 解析 | 透過 `ps -t <tty>` 解析，wrapper script 啟動的 SSH 可能無法偵測 |
| Port 偵測 | 僅從 `-p PORT` 旗標偵測。`~/.ssh/config` 的 port 由 scp 繼承 |
| 檔案覆蓋 | 同名檔案直接覆蓋 |
| SSH 認證 | 需非互動式認證（key 或 ControlMaster） |

## 除錯

```bash
cat /tmp/iterm-upload.log
```

| 症狀 | 原因 | 修正 |
|------|------|------|
| 總是貼本地路徑 | `is_claude` 未設定 | 檢查 Claude wrapper（step 3）|
| SSH 可用但 tmux 不行 | 缺少 DCS passthrough | 使用 tmux 版 wrapper（step 3）|
| 上傳無聲失敗 | SSH 需要密碼/2FA | 使用 SSH key 或 ControlMaster |
| "Cannot find ssh process" | SSH 被 wrapper 包裝 | 手動檢查 `ps -t <tty>` |
