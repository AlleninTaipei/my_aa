# MCP Advanced Topics

## 1. JSON 訊息類型基礎

MCP 的所有通訊都以 JSON-RPC 格式在客戶端與伺服器之間傳遞。訊息分為兩大類別：

| 類別 | 特性 | 範例 |
|------|------|------|
| 請求／結果配對 | 永遠成對出現；請求方需等待回應 | call_tool_request + call_tool_result、initialize_request + initialize_result |
| 通知 | 單向事件，不需要回應 | progress_notification、logging_message_notification、tool_change_notification |

訊息方向分為兩類：

- 客戶端訊息：由 MCP 客戶端發送至伺服器
- 伺服器訊息：由 MCP 伺服器發送至客戶端

關鍵認知：伺服器也可以主動向客戶端發起請求（server requests）與發送通知（server notifications）。這種雙向特性在 StreamableHTTP 傳輸中會成為核心限制。

Schema 定義位於 MCP 規格儲存庫的 TypeScript 檔案（schema.ts），內容是型別描述，非可執行程式碼。

---

## 2. Stdio 傳輸

Stdio 是 MCP 的本地傳輸機制：客戶端以子程序方式啟動伺服器，透過標準輸入／輸出串流進行通訊。

通訊機制：

- 客戶端寫入伺服器的 stdin，從 stdout 讀取回應
- 伺服器寫入 stdout，從 stdin 讀取請求

| 面向 | 內容 |
|------|------|
| 優點 | 完整雙向通訊——客戶端或伺服器均可隨時主動發起請求 |
| 限制 | 僅適用於客戶端與伺服器在同一台實體機器上的情況 |

初始化序列（必須按順序執行）：

1. Initialize request（客戶端 → 伺服器）
2. Initialize result（伺服器 → 客戶端）
3. Initialized notification（客戶端 → 伺服器，不需要回應）

訊息類型說明：

- Requests：期待回應
- Notifications：不需要回應
- Results：對 request 的回應

Stdio 傳輸完整支援伺服器發起請求的場景，這是與 HTTP 傳輸的關鍵差異。

---

## 3. StreamableHTTP 傳輸

### 基本概念

StreamableHTTP 傳輸讓 MCP 伺服器可以透過 HTTP 連線運作，不再受限於同一台機器，可以部署在公開可存取的 URL（如 mcpserver.com）。

| 面向 | 內容 |
|------|------|
| 主要優勢 | 遠端部署能力——伺服器可公開存取 |
| 關鍵限制 | HTTP 的單向特性使伺服器難以主動向客戶端發送請求 |

受此限制影響的功能：

- Sampling 請求
- Listing routes
- 進度通知
- 日誌通知

常見部署陷阱：應用程式在本地使用 Stdio 傳輸運作正常，部署到 HTTP 後卻出現功能異常，原因正是上述訊息方向的限制。

### 深入機制：SSE 如何解決雙向通訊問題

StreamableHTTP 使用 Server-Sent Events（SSE）讓伺服器能主動串流訊息給客戶端。

Session ID：初始化時伺服器指派一個隨機識別符，後續所有請求都在 HTTP 標頭中帶上這個 ID。

初始化流程：

1. 客戶端發送 initialize request
2. 伺服器回應 result，並附帶 MCP session ID 標頭
3. 客戶端發送含 session ID 的 initialized notification
4. 客戶端（可選）以 GET 請求加上 session ID 建立 SSE 連線

兩種 SSE 連線：

| 連線類型 | 用途 |
|----------|------|
| 長效 SSE 連線 | 處理伺服器發起的請求（sampling、通知） |
| 短效 SSE 連線 | 處理特定工具呼叫的回應，傳回結果後自動關閉 |

訊息路由規則：

- 進度通知 → 長效 SSE 連線
- 日誌訊息與工具結果 → 短效 SSE 連線（綁定至特定請求）

### Stateless HTTP 與 JSON Response 旗標

StreamableHTTP 有兩個旗標，預設值均為 `false`，設為 `true` 會顯著改變伺服器行為。

Stateless HTTP（`stateless=true`）：

適用場景：需要橫向擴展至多個伺服器實例搭配負載平衡器的環境。

問題根源：多個伺服器實例時，客戶端需要兩條連線——GET SSE 用於接收伺服器訊息、POST 用於發送請求。負載平衡器可能將這兩條連線路由到不同的伺服器實例。若工具在 Server A 上執行但 SSE 連線在 Server B，協調邏輯會變得非常複雜。

`stateless=true` 的效果：

| 效果 | 說明 |
|------|------|
| 不指派 Session ID | 伺服器無法追蹤個別客戶端 |
| GET SSE 連線停用 | 伺服器無法向客戶端發送請求 |
| 功能限制 | Sampling、進度日誌、資源訂閱全部失效 |
| 略過初始化 | 不需要 initialize request + notification，可減少伺服器流量 |

JSON Response（`json_response=true`）：

設為 `true` 後，POST 請求的回應只返回最終 JSON 結果，不包含串流訊息。執行過程中的進度與日誌陳述不再傳送，客戶端必須等待工具完整執行完畢才能收到回應。

關鍵原則：開發環境應使用與正式環境相同的傳輸方式，避免部署時才發現功能差異。

---

## 4. Sampling（取樣）

Sampling 是一種讓 MCP 伺服器委託客戶端代為呼叫語言模型的技術，伺服器本身不需要直接持有 LLM 存取權限。

運作架構：

1. 伺服器呼叫 `create_message()`，附上訊息列表
2. 客戶端透過 sampling callback 接收請求
3. 客戶端使用自己的 API 金鑰與配額呼叫 LLM
4. 客戶端將生成結果以 `create_message_result` 回傳伺服器

| 角色 | 實作方式 |
|------|----------|
| 伺服器端 | 呼叫 `create_message()`，附上訊息列表 |
| 客戶端端 | 實作 sampling callback，處理 LLM 請求並回傳 `create_message_result` |

主要優點：

- 伺服器不需要持有 API 金鑰或處理認證邏輯
- 伺服器不需要承擔 token 費用
- 公開存取的伺服器不會被濫用來消耗 LLM 配額

最適用場景：需要 LLM 能力、但又必須公開存取的 MCP 伺服器——尤其是不應持有 API 金鑰的情境。

---

## 5. 日誌與進度通知

日誌與進度通知讓伺服器在工具執行過程中向客戶端提供即時回饋，改善使用者體驗。

伺服器端實作：

- 工具函式自動以 context 作為最後一個參數
- Context 物件提供兩個方法：
  - `info()`：發送日誌訊息
  - `report_progress()`：發送進度更新
- 呼叫這些方法後，訊息會自動傳送至客戶端

客戶端實作：

- 建立日誌陳述的 callback 函式
- 建立進度更新的獨立 callback
- 將日誌 callback 傳入 client session
- 將進度 callback 傳入 `call_tool` 函式
- Callback 負責決定如何向使用者呈現資訊（終端機輸出、網頁 UI 等）

主要效益：

- 避免使用者誤以為工具呼叫卡住或失敗
- 提供長時間執行操作的可見度
- 工具執行期間的即時回饋

此功能為可選項目，若不需要可以省略，純屬 UX 改善用途。

---

## 6. Roots（根目錄）

Roots 是讓使用者授予伺服器存取特定檔案或資料夾的正式授權機制。

問題情境：使用者說「轉換 bikin.mp4」，但 Claude 面對複雜的檔案系統無法定位該檔案；若要求使用者每次提供完整路徑又不方便。

解決方式：在 MCP 伺服器中加入三個工具：

| 工具 | 功能 |
|------|------|
| ConvertVideo | 原有的核心工具 |
| ReadDirectory | 列出指定目錄下的檔案與資料夾 |
| ListRoots | 回傳目前可用的根目錄清單 |

Root 定義：使用者透過啟動伺服器時的命令列參數，事先授權伺服器可以存取的檔案或資料夾。

實作要點：每個工具在存取檔案或資料夾前，必須驗證路徑確實位於已授權的根目錄範圍內（透過類似 `is_path_allowed()` 的函式實作）。

Roots 的兩項主要效益：

| 效益 | 說明 |
|------|------|
| 權限控制 | 限制伺服器只能存取已授權的區域 |
| 自主探索 | Claude 可以主動搜尋可用根目錄來定位檔案，使用者不必提供完整路徑 |

重要限制：MCP SDK 不會自動強制執行根目錄限制，伺服器開發者必須手動實作存取檢查邏輯。

ListRoots 工具為可選項——另一種做法是直接在提示詞中列出根目錄清單。工具模式的優點是讓 Claude 能在需要時動態查詢可用根目錄。
