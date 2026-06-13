# Anthropic Academy — 工作慣例

## 課程筆記整理原則

將課程影片逐字稿或學習筆記整理成 Markdown 參考文件時，遵循以下原則：

### 移除課程包裝內容

- 刪除 **Key Takeaways** 區塊：內容是正文的摘要，整理後的表格與清單已足夠精簡，摘要只會造成重複。
- 刪除 **Lesson Reflection** 區塊：引導問題屬於互動課程設計，沒有答案也不傳遞技術資訊，放在參考文件中只會打斷閱讀流程。

### 保留的課程元素

- 保留 **Learning Objectives**：說明該節教授什麼，有助於快速定位內容。
- 保留 **What's next** 導航提示（`> **What's next:**` 格式）：說明章節順序，對課程地圖有參考價值。
- 保留練習步驟內嵌的 **Reflect / Reflection**（例如 `#### Part III: Reflect`）：屬於練習流程的一部分，不是獨立的課程包裝區塊。

### 已處理的檔案

- `Introduction to agent skills.md` — 完整重新格式化 + 移除包裝內容
- `AI Fluency for Small Businesses.md` — 移除包裝內容
- `claude-101.md` — 移除包裝內容
- `Building with the Claude API.md` — 從 XML `<note>` 格式完整重寫為 Markdown；40 個 note 重組為 12 個章節；電報式寫法改為正常文句；比較性內容轉為表格；加入程式碼區塊語言標記
- `Introducing MCP.md` — 從 XML `<note>` 格式完整重寫為 Markdown；11 個 note 重組為 5 個章節；電報式寫法改為正常文句；比較性內容轉為表格；加入程式碼區塊語言標記
- `Claude Code in Action.md` — 從 XML `<note>` 格式完整重寫為 Markdown；13 個 note（含多個重複項）合併重組為 12 個章節；電報式寫法改為正常文句；比較性內容轉為表格；加入程式碼區塊語言標記
- `Introduction to agent skills.md` — 英文繁中化；移除粗體；保留原有結構（6 節）
- `AI Fluency for Small Businesses.md` — 英文繁中化；移除粗體；保留所有練習步驟內嵌的 Reflect 區塊與 What's next 提示
- `claude-101.md` — 英文繁中化；移除粗體；保留所有 Learning Objectives、What's next 與課程結構
- `Teaching AI Fluency.md` — 從 XML `<note>` 格式完整重寫為 Markdown；4 個 note 重組為 16 個章節；電報式寫法改為正常文句；比較性內容轉為表格；繁中化
- `Introduction to Claude Cowork.md` — 從 296KB Skilljar HTML 課程匯出完整重寫為 Markdown；剝除全部 CSS 與 JavaScript；14 課 4 模組重組為 15 個章節；比較性內容（Chat/Cowork/Code、權限模式、M365 vs Cowork 等）轉為表格；移除 Key Takeaways 與 Lesson Reflection；繁中化
- `MCP Advanced Topics.md` — 從 XML `<note>` 格式完整重寫為 Markdown；8 個 note 重組為 6 個章節；電報式寫法改為正常文句；比較性內容轉為表格；繁中化

### XML note 格式的處理原則

部分課程筆記以 XML `<note>` 格式儲存（設計為 AI RAG 上下文，非人類閱讀）。遇到此格式時，處理方式與一般 Markdown 筆記不同：

- 移除所有 XML 包裝（`<notes>`、`<note title="...">`、`<transcript>`、`<critical>` 等）
- 將 note title 轉為 Markdown 標題，依主題層次決定標題層級（`##` 或 `###`）
- 將平鋪的 note 列表依主題重組為有層次的章節，而非逐條轉換
- 電報式寫法（省略主詞、冠詞、動詞）改寫為正常句子與條列結構
- `<transcript>` 標籤內容視同 `<note>`，納入對應主題章節

### 格式慣例

- 章節用編號標題（`## 1.`、`## 2.`），子節用 `###`。
- 比較性說明（A vs B、何時用 X vs Y）改為表格呈現。
- 程式碼區塊加語言標記（`yaml`、`bash`、`json`、`markdown` 等）。
- 章節之間加 `---` 水平線分隔。
- 不使用粗體強調，改用標題層級或條列結構呈現層次。
