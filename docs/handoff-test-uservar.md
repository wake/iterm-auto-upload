# 測試 iTerm2 User-Defined Variable 作為 Claude Code 偵測機制

## 背景

`iterm-auto-upload` 透過 iTerm2 fileDropCoprocess 攔截拖曳檔案，在 SSH + Claude Code 環境中自動 scp 上傳。

目前偵測 Claude Code 的方式是檢查 `\(autoName)` 是否包含 `"Claude Code"`，但 Claude Code 在第一次對話後會自動改名標題（如 `✳ TDD Refactor tmux-ags`），導致偵測失敗。

## 新方案

改用 iTerm2 的 **User-Defined Variable**（OSC 1337 SetUserVar），在遠端 claude wrapper 中設定，iTerm2 端統一偵測。

好處：不依賴標題內容，不管有沒有 tmux 都能用，一種判斷就夠。

## 需要測試的事情

### 測試 1：確認 `\(user.is_claude)` 在 fileDropCoprocess 中可用

1. 專案位置：`~/Workspace/wake/iterm-auto-upload`（需 clone 或確認路徑）

2. 已建立測試腳本 `bin/test-uservar`，內容：

```bash
#!/bin/bash
LOG="/tmp/iterm-upload-test.log"
echo "[$(date '+%Y-%m-%d %H:%M:%S')] === TEST DROP ===" >> "$LOG"
echo "  user.is_claude = [$1]" >> "$LOG"
echo "  jobName        = [$2]" >> "$LOG"
echo "  autoName       = [$3]" >> "$LOG"
echo "  tty            = [$4]" >> "$LOG"
shift 4
echo "  files          = [$*]" >> "$LOG"
printf '%s' "$*"
```

3. 把 iTerm2 的 fileDropCoprocess 設定改為：

```
/path/to/bin/test-uservar "\(user.is_claude)" \(jobName) "\(autoName)" \(tty) \(filenames)
```

4. 在終端執行（設定 user variable）：

```bash
printf '\033]1337;SetUserVar=%s=%s\007' "is_claude" "$(echo -n 1 | base64)"
```

5. 拖一個檔案進該終端

6. 檢查結果：

```bash
cat /tmp/iterm-upload-test.log
```

重點看 `user.is_claude = [???]` 是 `1` 還是空的。

### 測試 2：若測試 1 成功，驗證清除也能運作

```bash
printf '\033]1337;SetUserVar=%s=%s\007' "is_claude" "$(echo -n 0 | base64)"
```

再拖一次檔案，確認 `user.is_claude` 變成 `0`。

### 測試 3：若前兩個都成功，測試 tmux 穿透

在 tmux 中執行（DCS 包裝版）：

```bash
printf '\ePtmux;\e\033]1337;SetUserVar=%s=%s\007\e\\' "is_claude" "$(echo -n 1 | base64)"
```

tmux.conf 需要加：

```
set -g allow-passthrough on
```

## 測試結果回報

回報以下資訊即可：
- 測試 1：`user.is_claude` 的值是什麼？還是 fileDropCoprocess 根本沒執行？
- 測試 2：清除後值是否變成 `0`？
- 測試 3：tmux 中是否也能穿透？

## 後續

確認可用後，修改 `bin/iterm-upload` 主腳本，將偵測邏輯從 `autoName` 改為 `user.is_claude`，並更新 README 中的 wrapper 和 tmux 設定說明。
