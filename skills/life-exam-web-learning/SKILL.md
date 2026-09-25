---
name: life-exam-web-learning
description: >-
  建立與維護人壽保險歷屆試題網頁學習平台：整理題目／答案 PDF、建立可修正的 Excel 題庫、產生 JSON 題庫、測驗與學習模式、LocalStorage 設定保存、PDF 頁碼定位與高亮、右側 AI 解題抽屜、SQLite AI 解答快取、Qwen3-TTS 普通話朗讀，以及 Rust 可攜式啟動程式。當使用者要求「考古題網頁考試」「答題後 AI 解釋」「PDF 題目答案配對」「Excel 修正答案」「右側 AI 解題」「題目朗讀」「可攜式 Rust 考試程式」時使用。
---

# 人壽保險歷屆試題網頁學習平台

## 核心架構

```text
PDF 題目／答案
  ↓
questions.json + 題目答案題庫.xlsx
  ↓
index.html（測驗／學習模式）
  ↓
highlight_server.py（PDF 高亮、AI、TTS、本地服務）
  ├─ /questions.json：載入 Excel 修正後答案
  ├─ /highlight：產生指定頁面的高亮 PDF
  ├─ /ai-explain：呼叫本機 OmniRoute
  └─ /tts：代理遠端 Qwen3-TTS
  ↓
Rust portable EXE（啟動本機伺服器與瀏覽器）
```

目前平台的主要路徑範例：

```text
D:\中華民國人壽保險管理學會\exam_web\
D:\中華民國人壽保險管理學會\portable_exam\
```

## 題庫資料流程

### 原始資料

- 題目與答案 PDF 依考試梯次／科目配對。
- 每筆題目保留 `source_question`、`source_question_page`、`source_answer`、`source_answer_page`。
- PDF 頁碼不可固定寫成第 1 頁；必須依實際題號／答案表位置記錄。

### JSON

`questions.json` 至少包含：

```json
{
  "id": "172-48",
  "session": "107秋季",
  "subject": "壽險數學",
  "number": 48,
  "stem": "...",
  "options": [{"label":"A","text":"..."}],
  "answer": "B",
  "source_question": "...pdf",
  "source_question_page": 12,
  "source_answer": "...pdf",
  "source_answer_page": 1
}
```

### Excel 答案管理

交付檔：

```text
exam_web\題目答案題庫.xlsx
```

欄位：

- `官方答案`：原始答案 PDF 擷取結果，不覆蓋。
- `修正答案`：人工發現官方答案疑義時填 A/B/C/D/E。
- `最終答案`：網站實際使用的答案；若有值優先於官方答案。
- 題目／答案 PDF 相對路徑與頁碼。

不要把有疑義的官方答案靜默改掉；應保留官方答案，另填修正答案／最終答案，並在備註記錄原因。

建立或更新 Excel：

```powershell
uv run --with openpyxl python exam_web\export_question_bank_xlsx.py
```

將 Excel 的修正答案寫回 JSON（可選）：

```powershell
uv run --with openpyxl python exam_web\sync_answers_from_excel.py
```

網站伺服器也會在提供 `/questions.json` 時動態讀取 Excel 的最終答案，因此修改 Excel 後重啟／重新載入即可生效。

## PDF 文字與數學符號

舊 PDF 常把數學符號存成私有字碼，例如 `\uf03d`、`\uf0a3`、`\uf0b3`，直接顯示會變成方框或亂碼。清理時至少處理：

```text
\uf03d → =
\uf02b → +
\uf02d → −
\uf03c → <
\uf03e → >
\uf0a3 → ≤
\uf0b3 → ≥
\uf0b4 / \uf0d7 → ×
\uf0bc → ⋯
```

題目解析的選項偵測只把大寫 `(A)`～`(E)`、`A.`～`E.` 視為選項；題幹內的小寫 `a/b/c` 敘述不可誤切成選項。

若 PDF 文字層本身已遺失公式，保留原始 PDF 供人工回查，不要臆造公式。

## 網頁功能

### 測驗模式

- 交卷後批改。
- 顯示正確／錯誤選項狀態。
- 答案來源以 Excel 最終答案為準。

### 學習模式

- 題目旁提供：
  - `🔊 朗讀`
  - `🤖 解釋`
- 只有使用者按 `🤖 解釋` 才呼叫 AI，不因每次答錯自動產生。
- AI 解答顯示在瀏覽器最右側滑出式抽屜，不放在題目卡片內。
- 題目卡片下方不要重複顯示答錯／正確答案／AI 解答。
- 右側抽屜應包含題目、AI 解答、PDF 參考連結與關閉按鈕。

### LocalStorage

使用 `localStorage` 保存使用者設定，不能每次初始化時重設：

```text
life-exam-settings-v1
```

至少保存：

- 科目
- 考試梯次
- 題數
- 作答模式

載入題庫後重新建立 select 選項時，先保存舊值，再恢復仍存在的選項值。

### PDF 參考資料

- 題目 PDF 與答案 PDF 連結必須使用正確的相對路徑、中文 URL encoding 與頁碼。
- `/highlight` 只輸出指定 PDF 頁面並加黃色 annotation。
- 不要把不可精確對應的泛用網路連結當作本題來源；Google 搜尋可作額外查證，但不能取代原始 PDF。

## AI 解題與快取

### API

OmniRoute：

```text
http://127.0.0.1:20128/v1/chat/completions
```

使用 OpenAI-compatible JSON，模型由目前本機設定決定；不得把 API key 寫進程式或 log。

### 解題規則

送給 AI 的內容包含：

- 題目 ID
- 科目
- 題目
- 選項
- Excel／JSON 最終答案

Prompt 必須要求：

- 繁體中文
- 說明正確答案與判斷理由
- 說明其他選項為何不對
- 官方答案與題意矛盾時，明確標記「題庫答案與題意可能有落差」
- 不可把 AI 替代推論冒充官方答案
- 不輸出隱藏 chain-of-thought

### SQLite 快取

資料庫：

```text
exam_web\ai_explanations.sqlite3
```

表：

```sql
CREATE TABLE ai_explanations (
  question_id TEXT PRIMARY KEY,
  question_hash TEXT NOT NULL,
  answer TEXT NOT NULL,
  content TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
```

`question_hash` 必須由題目、選項與最終答案計算。相同 ID 且 hash 未變時直接使用 SQLite；題目、選項或答案變更才重新呼叫 AI。

## Qwen3-TTS

遠端服務：

```text
http://10.145.119.19:7862/v1/audio/speech
```

使用 Qwen3-TTS、`language: Chinese`、speaker `Serena`，並指定標準普通話／國語 instruction。不要誤用已停止或不存在的 `7861` audiocpp endpoint。

本機 `/tts` 代理回傳 `audio/wav`，前端使用 `Audio` 播放；TTS 失敗要顯示實際 HTTP 錯誤，不要假稱成功。

## Rust 可攜式程式

Rust 主程式：

```text
portable_exam\src\main.rs
```

交付 EXE 範例：

```text
portable_exam\LifeExamPortable_v2\LifeExamPortable.exe
```

可攜式版本至少需要整個資料夾，不可只複製 EXE，因為需要：

```text
exam_web\
題目／答案資料夾
questions.json
題目答案題庫.xlsx
SQLite 快取
```

Rust EXE 負責啟動本地 `highlight_server.py`、等待 8765、再開啟瀏覽器。若要真正零 Python 依賴，必須另行將 PDF 高亮改為 Rust 原生 PDF 引擎；不可宣稱目前版本是單一零依賴 EXE。

## 驗證清單

1. `questions.json` 可載入，題數大於 0。
2. Excel 存在，且有 `官方答案`、`修正答案`、`最終答案`。
3. 修改 Excel 後，`/questions.json` 反映最終答案。
4. Node `--check` 通過，尤其是新增按鈕、側邊抽屜與 Markdown renderer 後。
5. `/ai-explain` 可取得 HTTP 200；相同題目第二次應命中 SQLite，而不是重新呼叫模型。
6. `ai_explanations.sqlite3` 存在且有快取資料。
7. `/tts` 回傳 WAV。
8. PDF 參考連結回到正確題目／答案頁碼並有高亮。
9. `localStorage` 在重新整理後恢復科目、梯次、題數與作答模式。
10. Rust release EXE 可以啟動本地服務與考試頁面。

## 重要限制

- AI 解答不是官方答案；官方／Excel 最終答案優先。
- 網路查證若要加入，必須明確使用可驗證的搜尋／瀏覽工具；不可把模型臆測稱為已上網查證。
- PDF 可能有舊版答案錯誤或題意／答案版本不一致，必須保留疑義標記。
- 原始題目與答案可能有著作權限制；不要公開散布未授權資料或可大量還原原文的模型。
