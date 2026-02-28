# Changelog

## v1.0.0

iTerm2 drag-and-drop auto upload for SSH + Claude Code sessions.

### Features

- **fileDropCoprocess 攔截** — 拖曳檔案到 iTerm2 時，自動判斷是否為 SSH + Claude Code 環境
- **SCP 自動上傳** — 偵測到 Claude Code 後，透過 scp 上傳到遠端指定目錄
- **Bracketed paste 輸出** — 上傳完成後以 bracketed paste 格式輸出遠端絕對路徑，觸發 Claude Code auto-attach
- **SSH 自動解析** — 從 `ps -t <tty>` 解析 SSH target 和 port，支援 `~/.ssh/config` alias
- **user.is_claude 偵測** — 透過 iTerm2 user-defined variable（OSC 1337 SetUserVar）偵測 Claude Code，不依賴終端標題
- **Fallback 機制** — 非 SSH 或非 Claude Code 環境自動貼上本地路徑，不影響正常操作
- **tmux 支援** — 透過 `allow-passthrough` 或 DCS 手動包裝支援 tmux 環境
