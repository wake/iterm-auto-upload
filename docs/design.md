# iterm-auto-upload 設計文件

## 目標

在 iTerm2 中 SSH 到遠端時，拖曳檔案自動上傳並貼上遠端路徑。
專為 Claude Code CLI 場景設計。

## 使用情境

```
iTerm2 → SSH → tmux (可能) → Claude Code CLI
```

拖曳檔案後，Claude Code 輸入框直接出現遠端路徑，可立即使用。

## 核心機制：iTerm2 `fileDropCoprocess`

iTerm2 進階設定 `fileDropCoprocess`，在檔案拖曳到視窗時觸發自訂腳本，
完全取代預設的「貼上路徑」行為。

### 設定

```bash
# iTerm2 → Settings → Advanced → 搜尋 "file drop"
# 或用 defaults write（需先關閉 iTerm2）：
defaults write com.googlecode.iterm2 fileDropCoprocess \
  -string '/path/to/bin/iterm-upload \(jobName) "\(autoName)" \(tty) \(filenames)'
```

### iTerm2 Interpolated String 變數

| 變數 | 說明 | 用途 |
|------|------|------|
| `\(jobName)` | 當前 foreground job 名稱 | 偵測 SSH（值為 `ssh`） |
| `\(autoName)` | Session 自動名稱（含終端標題） | 偵測 Claude Code |
| `\(tty)` | 本機 TTY 裝置 | 用於 `ps -t` 解析 SSH process |
| `\(filenames)` | Shell-quoted 空格分隔的完整檔案路徑 | 上傳對象 |

### Coprocess 特性

- stdout 直接作為鍵盤輸入送到終端
- 設定後原始「貼上路徑」行為被完全取代
- 每個 session 同時只能有一個 coprocess

## 環境偵測邏輯

```bash
if [ "$jobName" = "ssh" ] && echo "$autoName" | grep -q "Claude Code"; then
    # SSH + Claude Code → scp 上傳，輸出遠端路徑
else
    # 本機或非 CC → 直接輸出本地路徑（原始行為）
fi
```

### 偵測結果

| 情境 | jobName | autoName | 行為 |
|------|---------|----------|------|
| 本機 shell | `tcsh`/`bash` | `tcsh` | 貼本地路徑 |
| 本機 Claude Code | `node`/`claude` | `✳ Claude Code` | 貼本地路徑 |
| SSH → shell | `ssh` | `bash` | 貼本地路徑 |
| SSH → Claude Code | `ssh` | `✳ Claude Code` | **上傳 + 貼遠端路徑** |
| SSH → tmux → Claude Code | `ssh` | `✳ Claude Code` | **上傳 + 貼遠端路徑** |

> 注意：tmux 需設定標題穿透才能正確偵測 Claude Code：
> ```bash
> # ~/.tmux.conf
> set -g set-titles on
> set -g set-titles-string "#{pane_title}"
> set -g allow-rename on
> ```

## 上傳流程

```
1. 拖曳檔案到 iTerm2 視窗
2. iTerm2 攔截 drop（不貼入任何文字）
3. 執行 bin/iterm-upload（作為 silent coprocess）
4. 腳本判斷環境：
   a. 非 SSH 或非 Claude Code → stdout 輸出本地路徑 → 結束
   b. SSH + Claude Code：
      - 從 ps process tree 解析 SSH 連線的 user@host:port
      - ssh mkdir -p ~/tmp/iterm-upload/
      - scp 檔案到遠端 ~/tmp/iterm-upload/
      - stdout 輸出遠端路徑（空格分隔）
5. coprocess stdout 自動送入終端作為鍵盤輸入
```

## 上傳目的地

- 固定路徑：`~/tmp/iterm-upload/`
- 上傳方式：`scp`

## 多檔案支援

iTerm2 的 `\(filenames)` 會提供 shell-quoted、空格分隔的所有拖曳檔案路徑。
腳本以 `$1 $2 ... $N` 各自取得每個檔案。

## 元件

### `bin/iterm-upload`（本機腳本）

- 輸入：`$1=jobName`, `$2=autoName`, `$3=tty`, `shift 3`, `$@=files`
- 偵測環境（SSH + Claude Code）
- 從 `ps -t <tty>` 解析 SSH 連線資訊（target + port）
- scp 上傳
- stdout 輸出路徑
- 錯誤時 fallback 輸出本地路徑
- 所有日誌寫入 `/tmp/iterm-upload.log`

### 安裝腳本 / 說明

- 設定 iTerm2 `fileDropCoprocess`
- 遠端 tmux 標題穿透設定

## 與 iTerm2 內建 Option+Drag SCP 的差異

| | 本方案 (fileDropCoprocess) | 內建 Option+Drag SCP |
|---|---|---|
| 觸發方式 | 直接拖曳（無需按鍵） | 按住 Option 拖曳 |
| 前提條件 | 無（只需本機腳本） | 遠端需安裝 Shell Integration |
| 上傳目錄 | 固定 `~/tmp/iterm-upload/` | 遠端當前工作目錄 |
| 路徑貼入 | 自動貼入終端 | 不會貼入路徑 |
| 環境偵測 | 自動判斷 SSH + CC | 無 |
| 非 SSH 時 | 退回貼本地路徑 | 無反應 |

## 已驗證事項

- [x] `fileDropCoprocess` 正確觸發（iTerm2 3.6.4）
- [x] 單檔 / 多檔皆正確傳入
- [x] `\(jobName)` 可偵測 SSH（值為 `ssh`）
- [x] `\(autoName)` 可偵測 Claude Code（包含 `Claude Code`）
- [x] tmux 標題穿透後 autoName 正確顯示
- [x] coprocess stdout 正確送入終端作為輸入

## 待實作

- [x] `bin/iterm-upload` 正式腳本
- [x] SSH 連線資訊解析（從 ps process tree）
- [x] scp 上傳邏輯
- [ ] 安裝說明 / README
