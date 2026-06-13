# Introducing MCP

## 1. 概念

MCP（Model Context Protocol）是為 Claude 提供上下文與工具的通訊層，解決傳統工具整合方式的維護負擔問題。

傳統做法需要開發者為每個服務整合手動撰寫 tool schema 和函式（例如 GitHub API 工具），當服務功能複雜時，維護負擔極高。MCP 將工具定義與執行的工作從開發者的伺服器移至專用的 MCP server，MCP server 作為外部服務的介面，將功能包裝成預建工具供他人使用。

| 面向 | 說明 |
|------|------|
| 架構 | MCP client 連接 MCP server；server 包含 tools、resources、prompts |
| 解決的問題 | 消除為每個服務整合撰寫工具 schema 和函式的負擔 |
| 與 tool use 的關係 | 互補，不是替代——MCP 決定「誰」執行工作（server vs 開發者），兩者都涉及工具 |
| MCP server 由誰建立 | 任何人；服務提供商常建立官方實作（如 AWS、GitHub） |

核心價值：將整合負擔從應用開發者轉移至 MCP server 維護者，開發者不再需要自行撰寫或維護工具 schema 與函式實作。

---

## 2. MCP Server

### 2.1 工具（Tools）

工具是模型控制的原語，由 Claude 決定何時執行。使用 MCP Python SDK，可以透過裝飾器從 Python 函式自動生成 tool JSON schema，無需手動撰寫。

```python
from mcp import FastMCP
from pydantic import Field

mcp = FastMCP("my_server")

@mcp.tool(name="read_doc_contents", description="讀取文件內容")
def read_doc_contents(doc_id: str = Field(description="文件 ID")) -> str:
    if doc_id not in docs:
        raise ValueError(f"文件 {doc_id} 不存在")
    return docs[doc_id]

@mcp.tool(name="edit_document", description="編輯文件內容")
def edit_document(
    doc_id: str = Field(description="文件 ID"),
    old_string: str = Field(description="要替換的文字"),
    new_string: str = Field(description="替換後的文字")
) -> str:
    if doc_id not in docs:
        raise ValueError(f"文件 {doc_id} 不存在")
    docs[doc_id] = docs[doc_id].replace(old_string, new_string)
    return docs[doc_id]
```

實作要點：
- 使用 `@mcp.tool` 裝飾器加上 `name` 和 `description`
- 參數使用 pydantic 的 `Field()` 附上描述
- 在操作前驗證資料是否存在，對無效輸入拋出 `ValueError` 並附上有意義的錯誤訊息（錯誤訊息對 Claude 可見，Claude 可依此更正參數重試）
- SDK 自動從裝飾函式生成 JSON schema，無需手動撰寫

### 2.2 資源（Resources）

資源是應用程式控制的原語，由應用程式程式碼決定何時擷取資料，用於取得資料供 UI 顯示或 prompt 擴充。

| 類型 | URI 範例 | 說明 |
|------|----------|------|
| 直接資源 | `docs://documents` | 靜態 URI |
| 樣板資源 | `docs://documents/{doc_id}` | 參數化 URI，URI 參數會成為函式的關鍵字參數 |

```python
@mcp.resource("docs://documents", mime_type="application/json")
def list_documents() -> dict:
    return docs

@mcp.resource("docs://documents/{doc_id}", mime_type="text/plain")
def get_document(doc_id: str) -> str:
    return docs.get(doc_id, "")
```

MIME type 作為提示讓 client 知道回傳的資料格式以便正確反序列化。Python SDK 會自動將回傳值序列化為字串。

### 2.3 提示（Prompts）

提示是使用者控制的原語，由使用者動作（如點擊按鈕或 slash command）觸發，用於預定義的工作流程。

```python
@mcp.prompt(name="format_document", description="將文件格式化為 Markdown")
def format_document_prompt(document_id: str):
    return [
        base.UserMessage(
            f"請讀取文件 {document_id}，使用 read_doc_contents 工具，"
            f"然後將內容重新格式化為清晰的 Markdown，並用 edit_document 工具儲存。"
        )
    ]
```

Server 端定義的 prompt 模板讓 server 作者能封裝領域專業知識，不需要依賴使用者自行撰寫 prompt。在 client 應用程式中，prompts 通常以 slash commands 形式呈現，提示使用者輸入所需參數後執行預建工作流程。

---

## 3. MCP Inspector

MCP Inspector 是在瀏覽器中測試 MCP server 的除錯工具，可在不連接完整應用程式的情況下驗證工具行為。

```bash
mcp dev server.py
# 在瀏覽器開啟提供的 URL
```

使用流程：連接 server → 瀏覽可用工具 → 手動輸入參數執行工具 → 驗證輸出與錯誤訊息。Inspector 介面左側邊欄有連接按鈕，上方導航列顯示 Resources、Prompts、Tools 三個區塊，點選工具後可在右側面板手動測試。

---

## 4. MCP Client

### 4.1 架構與通訊

MCP client 採用傳輸無關設計，可透過 stdio、HTTP、WebSockets 等多種協定通訊。常見設定是兩端在同一台機器上使用 stdin/stdout 通訊。MCP client 作為應用程式與 MCP server 之間的中介，本身不執行工具，只負責協助通訊。

主要訊息類型：

| 訊息類型 | 說明 |
|---------|------|
| list tools request/result | client 向 server 請求可用工具列表，server 回傳工具清單 |
| call tool request/result | client 要求 server 以指定參數執行工具，server 回傳執行結果 |

完整對話流程：
```
使用者查詢 → 伺服器向 MCP client 請求工具列表 → MCP client 向 MCP server 請求
→ MCP server 回傳工具列表 → 伺服器傳送查詢 + 工具給 Claude
→ Claude 請求工具執行 → 伺服器請求 MCP client 執行工具
→ MCP client 請求 MCP server → MCP server 執行工具（如 GitHub API 呼叫）
→ 結果沿鏈回傳：MCP server → MCP client → 伺服器 → Claude → 使用者
```

### 4.2 實作 MCP Client

常見做法是將 MCP Python SDK 的 `ClientSession` 包裝在較大的類別中進行資源管理（連接、關閉、非同步進入/退出），而非直接使用 session。

```python
class MCPClient:
    def __init__(self, session):
        self.session = session  # ClientSession from MCP Python SDK

    async def list_tools(self):
        result = await self.session.list_tools()
        return result.tools

    async def call_tool(self, tool_name, tool_input):
        return await self.session.call_tool(tool_name, tool_input)
```

### 4.3 存取資源

```python
async def read_resource(self, uri):
    result = await self.session.read_resource(AnyUrl(uri))
    resource = result.contents[0]
    if resource.mime_type == "application/json":
        return json.loads(resource.text)
    return resource.text
```

資源整合後，選取的資源內容可自動納入 LLM prompt，不需要 Claude 再透過工具呼叫讀取文件。

### 4.4 使用提示

```python
async def list_prompts(self):
    result = await self.session.list_prompts()
    return result.prompts

async def get_prompt(self, prompt_name, arguments):
    result = await self.session.get_prompt(prompt_name, arguments)
    return result.messages  # 可直接送給 LLM
```

呼叫提示時，client 傳入的 `arguments` 會作為關鍵字參數插入 server 端的 prompt 模板。回傳的 messages 陣列可直接送給 AI 模型。

---

## 5. 三種 Server 原語

MCP server 包含三種原語，由不同角色控制，服務不同的用途：

| 原語 | 控制方 | 用途 | 實例 |
|------|--------|------|------|
| Tools | 模型（Claude 決定何時執行） | 為 Claude 新增執行能力 | 程式碼執行、API 呼叫 |
| Resources | 應用程式（程式碼決定何時擷取） | 取得資料供 UI 顯示或 prompt 擴充 | Google Drive 文件列表、自動完成選項 |
| Prompts | 使用者（由使用者動作觸發） | 預定義工作流程 | Claude 介面的對話起始按鈕、slash commands |

選擇依據：需要 Claude 執行能力 → 實作工具；需要應用程式資料 → 使用資源；需要使用者工作流程 → 建立提示。
