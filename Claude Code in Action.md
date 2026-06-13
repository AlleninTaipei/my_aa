# Claude Code in Action

## 1. 什麼是 Coding Assistant

Coding Assistant 是使用語言模型來撰寫程式碼並完成開發任務的工具。其核心流程如下：

1. 接收任務（例如從錯誤訊息修復 bug）
2. 語言模型蒐集上下文（讀取檔案、理解程式碼庫）
3. 擬定解決方案
4. 執行動作（更新檔案、執行測試）

### 語言模型的限制與工具使用

語言模型本身只能處理文字輸入與輸出，無法直接讀取檔案、執行命令或與外部系統互動。工具使用（Tool Use）系統讓語言模型得以執行這些動作：

1. Assistant 在使用者請求後附加指令，說明格式化動作的規格（例如「讀取檔案：檔名」）
2. 語言模型回應格式化的動作請求
3. Assistant 執行實際動作（讀取檔案、執行命令）
4. 將結果傳回語言模型，產生最終回應

### Claude 的優勢

- 工具使用能力優於其他語言模型
- 更善於理解工具功能，並組合多個工具完成複雜任務
- Claude Code 具備擴充性，可輕鬆新增工具
- 透過直接程式碼搜尋（而非將整個程式碼庫傳送至外部伺服器的索引方式）提供更好的安全性

---

## 2. Claude Code 實際展示

### 效能優化

Claude 分析了 Chalk（JavaScript 第五大下載套件，每週 4.29 億次下載）。過程中使用基準測試、效能分析工具，建立待辦清單、找出瓶頸並實作修正，最終達到 3.9 倍的吞吐量提升。

### 資料分析

Claude 使用 Jupyter Notebook 對影音串流平台的 CSV 資料進行客戶流失分析。採用反覆執行程式碼格的方式，查看結果後依據發現進行後續分析。

### 瀏覽器自動化（Playwright）

Claude Code 可接受新的工具集。以 Playwright MCP 伺服器為例，Claude 開啟瀏覽器、擷取截圖、更新 UI 樣式，並持續疊代改進設計。

### GitHub Actions 自動審查

Claude Code 可在 GitHub Actions 中執行，由 Pull Request 或 Issue 觸發。在一個基礎建設審查範例中：Terraform 定義的 AWS 基礎建設（DynamoDB table 與 S3 bucket）與外部合作夥伴共享。開發者在 Lambda 函式輸出中加入使用者 email，Claude Code 自動分析基礎建設流程，偵測到 PII 外洩風險並在 PR 審查中標記。

Claude Code 的核心設計理念是：透過持續擴充工具而非固定功能，成為能隨團隊需求成長的彈性助手。

---

## 3. 管理上下文（Context）

上下文管理對 Claude Code 的效果至關重要——過多無關的資訊會降低效能。

### /init 指令與 CLAUDE.md

執行 `/init` 指令時，Claude Code 會分析整個程式碼庫，並建立包含專案摘要、架構與關鍵檔案的 `CLAUDE.md` 檔案。此檔案的內容會包含在每次請求中。

CLAUDE.md 有三種類型：

| 類型 | 說明 | 是否納入版本控制 |
|------|------|----------------|
| 專案層級 | 與團隊共享 | 是 |
| 本地層級 | 個人指令 | 否 |
| 機器層級 | 套用於所有專案的全域指令 | 否 |

### 提供上下文的方式

- `#` 符號（Memory 模式）：以自然語言請求智慧編輯 CLAUDE.md 檔案
- `@` 符號：在請求中明確指定要包含的檔案，提供精確上下文，而非讓 Claude 自行搜尋

最佳實踐是在 CLAUDE.md 中引用關鍵檔案（例如資料庫 schema），讓這些資訊在每次請求中都能作為上下文使用。目標是提供剛好足夠的相關資訊，讓 Claude 能有效完成任務。

---

## 4. 執行變更

### 截圖整合

按下 Control-V（macOS 上不是 Command-V）可貼上截圖，幫助 Claude 理解需要修改的特定 UI 元素。

### 提升效能的模式

| 模式 | 觸發方式 | 適用情境 |
|------|---------|---------|
| Plan Mode | Shift + Tab 兩次 | 需要廣泛了解程式碼庫的多步驟任務（廣度） |
| Thinking Mode | 輸入「Ultra think」等詞語 | 需要深入推理的複雜邏輯或除錯（深度） |

兩種模式可以組合使用，但都會消耗額外的 token（需考量成本）。

### Git 整合

Claude Code 可以暫存、提交變更，並自動撰寫描述性的提交訊息。

建議工作流程：截圖問題區域 → Control-V 貼上 → 描述需要的變更 → 視複雜度選擇開啟 Plan 或 Thinking 模式 → 審查並接受實作。

---

## 5. 控制上下文

| 功能 | 操作方式 | 適用情境 |
|------|---------|---------|
| Escape | 按一次 | 中斷 Claude 目前的輸出，重新引導對話方向 |
| Escape + Memory | 停止後用 `#` 快捷鍵新增記憶 | 防止 Claude 重複相同錯誤 |
| Double Escape | 按兩次 | 顯示所有先前訊息，可跳回較早的對話點，同時保留相關上下文 |
| Compact | `/compact` 指令 | 摘要整個對話歷史，同時保留 Claude 對當前任務的學習成果 |
| Clear | `/clear` 指令 | 刪除整個對話歷史，重新開始 |

這些功能有助於維持焦點、減少干擾上下文、保留相關知識，並防止重複錯誤。在長對話和任務切換時最為有效。

---

## 6. 自訂指令（Custom Commands）

Custom Commands 是使用者定義的自動化指令，可在 Claude Code 中以正斜線呼叫。

### 設定方式

- 位置：專案目錄下的 `.claude/commands/` 資料夾
- 命名：檔案名稱即為指令名稱（`audit.md` 建立 `/audit` 指令）
- 啟用：建立指令檔案後需重新啟動 Claude Code

### 指令結構

每個指令是一個包含 Claude 執行指引的 Markdown 檔案。若需在執行時傳入參數，在指令文字中使用 `$arguments` 占位符；參數可以是任何字串（檔案路徑、描述性文字等）。

### 使用方式

在 Claude Code 介面輸入 `/指令名稱`，視需要加上參數字串。

常見應用場景：自動化重複性任務，例如依賴項審查、測試生成、漏洞修正等。

---

## 7. 擴充 Claude Code：MCP 伺服器

MCP 伺服器是在本地或遠端執行的外部工具，可擴充 Claude Code 的功能。

### 安裝

```bash
claude mcp add [name] [start-command]
```

### 權限管理

首次使用工具時需要核准。若要自動核准，將 `"MCP__[servername]"` 加入 `settings.local.json` 的 allow 陣列中：

```json
{
  "allow": ["MCP__playwright"]
}
```

### Playwright 實際範例

以 Playwright MCP 伺服器為例，Claude 導航至 `localhost:3000`、生成 UI 元件、分析樣式品質，然後根據視覺回饋自動更新生成提示。自動化提示優化的結果顯著改善了元件樣式。

MCP 伺服器讓 Claude 能夠執行涉及外部系統的複雜多步驟任務，將能力從程式碼編輯擴展至完整的開發自動化。

---

## 8. GitHub 整合

Claude Code 官方整合可讓 Claude 在 GitHub Actions 中執行。

### 設定步驟

1. 執行 `/install GitHub app` 指令
2. 在 GitHub 上安裝 Claude Code app
3. 新增 API 金鑰
4. 系統自動生成的 PR 會新增兩個 GitHub Actions

### 預設 Actions

1. 提及支援：在 Issue 或 PR 中使用 `@Claude` 指派任務
2. PR 審查：針對新 PR 自動執行程式碼審查

### 自訂設定

Actions 可透過 `.github/workflows` 目錄下的設定檔自訂：
- 自訂指令：直接傳遞上下文與指引給 Claude
- MCP 伺服器整合：讓 Claude 存取外部工具（例如用 Playwright 進行瀏覽器自動化）

### 權限需求

- 必須明確列出所有 Claude Code Actions 所需的權限
- MCP 伺服器工具需要逐一列出權限，無法使用萬用設定

### 整合 Playwright 的範例流程

整合 Playwright MCP 伺服器後，可在 Claude 執行前先啟動開發伺服器。Claude 能在瀏覽器中瀏覽應用程式、測試功能、建立檢查清單，實現自動化測試與問題驗證。

---

## 9. Hooks 機制

Hooks 是在 Claude 執行工具前後運行的命令，用於攔截並控制工具調用。

### Hook 類型

| 類型 | 執行時機 | 是否可阻擋工具調用 |
|------|---------|-----------------|
| Pre-tool use hook | 工具執行前 | 可以（exit code 2） |
| Post-tool use hook | 工具執行後 | 不可以 |

### 設定方式

Hook 加入 Claude 設定檔（全域、專案或個人層級），可透過手動編輯或 `/hooks` 指令設定。設定結構分為兩個區塊（pre-tool use 和 post-tool use），每個區塊包含：
- Matcher：指定要監控哪些工具
- Commands：要執行的命令

### 實作流程

1. 選擇 hook 類型（pre 或 post）
2. 確認要監控的工具名稱
3. 撰寫命令，透過 stdin 以 JSON 接收工具調用資料
4. 解析包含 `tool_name` 和輸入參數的 JSON
5. 以適當的 exit code 回應

### Exit Code 說明

| Exit Code | 意義 |
|-----------|------|
| 0 | 允許工具調用繼續 |
| 2 | 阻擋工具調用（僅限 pre-tool use） |
| stderr 輸出 | 阻擋時傳送給 Claude 的回饋訊息 |

工具調用資料的 JSON 格式包含 `tool_name`（例如 `"read"`、`"grep"`）以及 `input` 參數（例如 `file_path`）。如需查詢可用的工具名稱，可直接詢問 Claude。

---

## 10. 實作 Hook 範例：封鎖 .env 讀取

以下範例示範如何建立 pre-tool use hook，防止 Claude 讀取 `.env` 檔案的內容。

### 設定（settings.local.json）

```json
{
  "preToolUseHooks": [
    {
      "matcher": "read|grep",
      "command": "node ./hooks/read_hook.js"
    }
  ]
}
```

### Hook 腳本邏輯（read_hook.js）

Hook 透過 stdin 接收 JSON 物件，包含 session ID、工具名稱、工具輸入及檔案路徑：

```javascript
// 從 stdin 讀取 JSON 後：
if (toolInput.path && toolInput.path.includes('.env')) {
  console.error('Blocked: access to .env file is not allowed.');
  process.exit(2);
}
```

重點說明：
- 使用 `console.error()` 將回饋訊息透過 stderr 傳送給 Claude
- Exit code 2 表示阻擋操作
- 透過 `tool_input.path` 搭配備援處理取得檔案路徑
- 修改 hook 後必須重新啟動 Claude

### 測試結果

成功阻擋 `.env` 檔案的讀取，Claude 能識別是由 read hook 攔截的操作，且對 read 和 grep 工具均有效。

---

## 11. 實用 Hook 範例

### TypeScript 型別檢查 Hook

Claude 修改函式簽名時，往往不會同步更新所有呼叫端，導致型別錯誤。解決方式是在 TypeScript 檔案被編輯後，透過 post-tool-use hook 執行 `tsc --no-emit`：

```bash
tsc --no-emit
```

流程：偵測到型別錯誤 → 將錯誤回饋給 Claude → Claude 自動修正呼叫端。

此方式適用於任何具有型別檢查器的強型別語言；對無型別語言則可改用測試替代。

### 重複程式碼防範 Hook

在複雜任務中，Claude 有時會建立新的 query 或函式，而非重用已有的程式碼。解決方式是監控特定目錄（如 `queries/` 資料夾）的變更，並透過 TypeScript SDK 啟動另一個 Claude 實例進行審查：

1. 偵測受監控目錄有檔案被編輯
2. 透過 TypeScript SDK 啟動新的 Claude 實例
3. 比對新增程式碼與既有程式碼
4. 若發現重複，以 exit code 2 回傳回饋訊息
5. 原始 Claude 接收到回饋後改為重用既有程式碼

| 考量面向 | 說明 |
|---------|------|
| 優點 | 程式碼庫更整潔，減少重複 |
| 缺點 | 增加時間與 API 成本 |
| 建議 | 僅對關鍵目錄啟用，降低額外負擔 |

---

## 12. Claude Code SDK

Claude Code SDK 提供以程式化方式使用 Claude Code 的介面，支援 CLI、TypeScript 與 Python 三種函式庫，工具組與終端機版本相同。

主要應用場景是整合至現有的較大型 pipeline 或工作流程，為既有流程加入智慧處理能力。

### 權限設定

| 權限類型 | 說明 |
|---------|------|
| 預設 | 唯讀（檔案、目錄、grep 操作） |
| 寫入權限 | 需手動設定，透過 `options.allowTools` 陣列或 `.claude` 目錄設定啟用 |

```typescript
// 啟用寫入權限的範例
query({
  options: {
    allowTools: ['edit']
  }
})
```

SDK 執行時會顯示本地 Claude Code 與語言模型之間的原始對話，最終回應為最後一則訊息。

SDK 最適合用於現有專案內的輔助指令、腳本與 hooks，而非作為獨立工具使用。
