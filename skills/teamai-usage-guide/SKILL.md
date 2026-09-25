---
name: teamai-usage-guide
description: >-
  用繁體中文說明與操作 TeamAI CLI：解釋 TeamAI 與 Claude Code 的差異、初始化團隊、同步與發布 skills、檢查狀態、診斷 hooks，以及處理常見設定問題。當使用者詢問「TeamAI 怎麼用」「TeamAI 跟 Claude Code 的差異」「如何分享 skill」「teamai init/pull/push/doctor/status」時使用。
---

# TeamAI 使用指南

## 核心概念

TeamAI 不是 AI 模型，也不是 Claude Code 的替代品：

- **Claude Code**：實際與使用者對話、讀寫程式碼、執行命令的 AI 開發工具。
- **TeamAI**：團隊資源同步工具，讓 Claude Code、Cursor、OpenCode 等工具共用 skills、rules、docs、MCP 與團隊工作規範。

簡單理解：

```text
Claude Code = AI 程式設計師
TeamAI     = 團隊 AI 資源同步與管理系統
```

TeamAI 主要透過團隊 Git 儲存庫分發資源；使用者通常只需要操作 `teamai` 指令，不需要自行處理底層同步細節。

## 回覆原則

1. 使用繁體中文回答；命令、參數、URL、路徑與程式識別字保持原樣。
2. 先確認目前是在 Claude Code、OpenCode 或其他 AI 工具中，再說明 hooks 的效果。
3. 不要把 TeamAI 描述成聊天機器人、模型或 Claude Code 替代品。
4. 不要要求或回顯 GitHub token、API key、密碼等敏感資料。
5. 涉及初始化、同步、發布或移除前，先執行對應的唯讀檢查；不要臆測不存在的參數。
6. 完成設定或排錯後，執行 `teamai doctor` 驗證。
7. 不要在沒有確認內容前發布本機資源；可先使用 `teamai status` 與 `teamai list skills --source local`。

## 目前團隊設定

目前已設定的團隊儲存庫：

```text
https://github.com/HuangChienHsiung590902/teams.git
```

目前採 user scope，主要資料位置：

```text
C:\Users\Administrator\.teamai
```

## 常用指令

```powershell
teamai status
teamai list
teamai list skills --source local
teamai pull
teamai push
teamai doctor
teamai --help
```

用途：

- `teamai status`：查看目前團隊設定、同步狀態與尚未發布的本機資源。
- `teamai list`：列出團隊資源。
- `teamai list skills --source local`：列出各 AI 工具本機的 skills。
- `teamai pull`：立即同步團隊資源；正常情況下，AI 工具的新工作階段也會透過 hook 自動同步。
- `teamai push`：把確認過的本機 skills、rules 等資源發布到團隊。
- `teamai doctor`：檢查 GitHub 登入、團隊儲存庫、設定與 hooks。

## 常見工作流程

### 第一次加入或建立團隊

```powershell
teamai init https://github.com/HuangChienHsiung590902/teams.git
teamai doctor
```

若在 home directory 執行，TeamAI 可能自動選擇 user scope；這是為了避免把專案資源直接寫入家目錄造成衝突。

### 取得團隊最新資源

```powershell
teamai pull
teamai status
```

在 Claude Code 中，通常重新開啟一個工作階段後會由 TeamAI hook 自動執行同步；若要立即同步，直接執行 `teamai pull`。

### 發布本機 skill

先確認本機有哪些 skill：

```powershell
teamai list skills --source local
teamai status
```

確認內容與名稱後，再發布指定 skill：

```powershell
teamai push --skill "C:\Users\Administrator\.claude\skills\<skill-name>"
```

若要讓 TeamAI 掃描並列出所有待發布資源：

```powershell
teamai push
```

發布後可用以下命令確認：

```powershell
teamai pull
teamai status
teamai doctor
```

不要把 secrets、token、密碼或含有機密的設定檔放進 skill。

## 在 Claude Code 中使用

TeamAI 負責同步；Claude Code 負責實際使用同步後的 skill。

典型流程：

1. 執行 `teamai pull` 或重新開啟 Claude Code。
2. 在 Claude Code 中提出與 skill 對應的任務。
3. Claude Code 依 skill 的 `SKILL.md` 執行工作。
4. 若修改了可共享的 skill，先檢查差異，再執行 `teamai push`。

若目前版本與設定支援 `/teamai`，可在 Claude Code 中使用：

```text
/teamai
```

但 `/teamai` 是 AI 工具內的互動入口，不等於 `teamai` PowerShell CLI。若 slash command 不可用，直接使用 PowerShell 指令即可。

## 常見問題

### TeamAI 是否等於 Claude Code？

不是。Claude Code 是 AI 開發工具；TeamAI 是讓多個 AI 開發工具共用團隊資源的同步與管理層。

### 為什麼初始化後看不到團隊 skills？

初始化只建立連線與 hooks；團隊儲存庫若尚未有資源，清單會是空的。先確認：

```powershell
teamai status
teamai list
teamai pull
```

### `teamai init` 出現 Author identity unknown

這表示本機 Git 尚未有提交者名稱或 email。不要使用虛構的帳號資料；應使用使用者指定的名稱與 GitHub 可接受的 email，再重新執行初始化或完成未發布的初始化動作。設定完成後執行：

```powershell
teamai doctor
teamai status
```

### hooks 是否會影響所有 AI 工具？

不一定。`teamai doctor` 會列出已檢查的 hooks。某些工具需要 shell 或特定 hook 介面；若某個工具被跳過，不要假設它已自動同步，改用該工具的支援方式或手動執行 `teamai pull`。

### 發布前要做什麼？

```powershell
teamai list skills --source local
teamai status
```

確認 skill 內容沒有密碼、token 或私人資料，再執行 `teamai push`。完成後一定執行：

```powershell
teamai doctor
```
