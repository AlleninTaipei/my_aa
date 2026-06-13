# Building with the Claude API

## 1. Claude Models

Claude 有三個模型系列，針對不同優先級優化：

| 模型 | 定位 | 特性 |
|------|------|------|
| Opus | 最高智能 | 適合複雜多步驟任務、深度推理與規劃；成本高、延遲較高 |
| Sonnet | 均衡 | 智能、速度、成本效率兼顧；程式碼能力強、精確編輯；適合大多數實際場景 |
| Haiku | 最快速 | 速度與成本效率最佳；無推理能力；適合即時互動與高量處理 |

選型框架：智能優先 → Opus；速度優先 → Haiku；均衡需求 → Sonnet。

實際應用中常見做法是在同一應用中依任務需求混用多個模型，而非統一使用單一模型。所有模型共享核心能力（文字生成、程式碼、圖像分析），主要差異在於優化方向。

---

## 2. API 基礎

### 2.1 API 存取流程

從使用者輸入到回應顯示共 5 個步驟：

1. 客戶端將使用者文字傳送至開發者的伺服器（絕不從客戶端直接存取 Anthropic API，以保護 API key）
2. 伺服器透過 SDK（Python、TypeScript、JavaScript、Go、Ruby）或純 HTTP 向 Anthropic API 發出請求，必要參數：API key、model name、messages list、max_tokens
3. 文字生成分 4 個階段：
   - Tokenization：將輸入拆分為 token（詞/詞的一部分/符號/空格）
   - Embedding：將 token 轉為代表所有可能詞義的數字列表
   - Contextualization：依相鄰 token 調整 embedding，確定精確語義
   - Generation：輸出層計算下一個詞的機率，依機率加隨機性選出詞，重複此過程
4. 達到 max_tokens 上限或生成特殊 end_of_sequence token 時停止
5. API 回傳生成文字、使用量計數、stop_reason 至伺服器，再由伺服器送至客戶端顯示

### 2.2 發出請求

環境設定步驟：

1. 安裝套件：`pip install anthropic python-dotenv`
2. 儲存 API key：建立 `.env` 文件寫入 `ANTHROPIC_API_KEY="your_key"`，並加入版本控制忽略清單
3. 載入環境變數：使用 python-dotenv 安全載入 API key
4. 建立客戶端：初始化 anthropic client 並定義 model 變數

API 請求結構：

```python
message = client.messages.create(
    model=model,
    max_tokens=1000,
    messages=[{"role": "user", "content": "What is quantum computing?"}]
)

# 取出純文字
text = message.content[0].text
```

- `max_tokens`：生成長度的安全上限，不是目標長度
- `messages`：包含對話交流的列表，每則訊息有 `role`（user/assistant）與 `content` 欄位

### 2.3 多輪對話

Anthropic API 不儲存任何訊息，每次請求都是獨立的，沒有上下文記憶。

解決方案：在程式碼中手動維護訊息列表，每次後續請求都附上完整對話歷史。

```python
def add_user_message(messages, text):
    messages.append({"role": "user", "content": text})

def add_assistant_message(messages, text):
    messages.append({"role": "assistant", "content": text})

def chat(messages):
    response = client.messages.create(model=model, max_tokens=1000, messages=messages)
    return response.content[0].text
```

對話流程：傳送初始訊息 → 接收回應 → 將回應加入歷史 → 加入新使用者訊息 → 附上完整歷史發送。

### 2.4 System Prompts

System Prompt 透過指定角色或行為模式來自訂 Claude 的回應風格，控制的是「如何回應」而非「回應什麼內容」。

```python
params = {"model": model, "max_tokens": 1000, "messages": messages}
if system_prompt:
    params["system"] = system_prompt
response = client.messages.create(**params)
```

結構：第一行通常指派角色（例如「You are a patient math tutor」），後續加入具體行為指令。若未提供 system，則完全省略該參數。

### 2.5 Temperature

Temperature（0–1）透過影響 token 選擇機率來控制生成的隨機程度。

| Temperature | 效果 | 適用場景 |
|-------------|------|----------|
| 接近 0 | 決定性輸出，永遠選最高機率的 token | 資料擷取、需要一致性的事實任務 |
| 接近 1 | 提高低機率 token 的選擇機會，輸出更有創意 | 腦力激盪、創意寫作、行銷文案 |

較高的值並不保證產生不同輸出，只是增加變化的機率。Temperature 直接操控下一個 token 選擇的機率分佈。

### 2.6 串流回應（Response Streaming）

串流可在 AI 生成回應的同時逐塊顯示，避免使用者等待 10–30 秒才看到任何內容。

運作方式：
1. 伺服器傳送訊息給 Claude
2. Claude 立即傳回初始回應（無文字，只是確認）
3. 後續串流事件陸續傳來，每個事件包含文字塊
4. 伺服器將文字塊轉發至前端即時顯示

主要事件類型：

| 事件 | 說明 |
|------|------|
| `message_start` | 初始確認 |
| `content_block_start` | 文字生成開始 |
| `content_block_delta` | 包含實際文字塊（最重要） |
| `content_block_stop` / `message_stop` | 生成完成 |

```python
# 基本方式
with client.messages.stream(model=model, max_tokens=1000, messages=messages) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)

# 取得完整訊息（用於儲存）
final_message = stream.get_final_message()
```

---

## 3. 輸出控制

### 3.1 Pre-filling 與 Stop Sequences

兩種不修改核心 prompt 就能精確控制輸出的技術。

Pre-filling（預填助理訊息）：在對話的最後手動加入一條 assistant 訊息，Claude 會從該文字的結尾處繼續生成。需要將預填內容與生成內容拼接才能取得完整回應。

```python
messages = [
    {"role": "user", "content": "Coffee or tea?"},
    {"role": "assistant", "content": "Coffee is better because"}
]
# Claude 從 "Coffee is better because" 繼續生成
```

Stop Sequences（停止序列）：當 Claude 生成了特定字串時，立即停止生成，且生成的停止序列本身不包含在輸出中。

```python
response = client.messages.create(
    model=model,
    max_tokens=100,
    messages=messages,
    stop_sequences=["five"]  # 輸出在生成 "five" 前停止
)
```

### 3.2 結構化資料擷取

結合 pre-filling 與 stop sequences 可取得不含多餘說明文字的原始結構化輸出。

```python
messages = [
    {"role": "user", "content": "List 3 fruits as JSON"},
    {"role": "assistant", "content": "```json"}  # pre-fill
]
response = client.messages.create(
    model=model,
    max_tokens=500,
    messages=messages,
    stop_sequences=["```"]  # 在結尾分隔符停止
)
# 輸出：純 JSON，無多餘格式或說明
```

此模式適用於任何結構化資料類型（JSON、Python 程式碼、列表等）。

---

## 4. Prompt 評估

### 4.1 評估的必要性

Prompt 工程的三條路徑：

1. 測試一兩次就部署至生產環境（陷阱）
2. 針對自訂輸入測試，對邊界案例做小修改（陷阱）
3. 透過評估管線以客觀指標衡量效果（推薦）

工程師常見問題是對 prompt 的測試不足。應使用評估管線取得客觀分數，才能系統性迭代改善。

### 4.2 標準評估流程

6 步驟的迭代流程：

1. 撰寫初始 prompt 草稿，建立優化基準
2. 建立評估資料集（可從 3 個範例到數千個；可手寫或由 LLM 生成）
3. 生成 prompt 變體，將資料集中的每個輸入插入 prompt 模板
4. 取得 LLM 回應，將每個 prompt 變體送給 Claude 並收集輸出
5. 評分，使用評分系統對每個回應打分（如 1–10），平均分數即為整體 prompt 效能
6. 迭代，依據分數修改 prompt，重複整個流程，比較各版本

目前無標準方法論；有許多開源和付費工具可用；可從簡單的自訂實作開始；客觀評分使 A/B 比較成為可能。

### 4.3 生成測試資料集

使用 Claude 自動生成測試案例（使用速度較快的模型如 Haiku）：

```python
def generate_dataset():
    messages = [{"role": "user", "content": generate_prompt}]
    assistant_prefill = {"role": "assistant", "content": "```json"}
    messages.append(assistant_prefill)
    
    response = client.messages.create(
        model="claude-haiku-...",
        max_tokens=2000,
        messages=messages,
        stop_sequences=["```"]
    )
    dataset = json.loads(response.content[0].text)
    with open("dataset.json", "w") as f:
        json.dump(dataset, f)
```

資料集結構：包含 `task` 屬性的 JSON 物件陣列，每個物件描述一個使用者請求。

### 4.4 執行評估

三個核心函式：

```python
def run_prompt(test_case, prompt_template):
    prompt = prompt_template.replace("{task}", test_case["task"])
    messages = [{"role": "user", "content": prompt}]
    response = client.messages.create(model=model, max_tokens=1000, messages=messages)
    return response.content[0].text

def run_test_case(test_case, prompt_template, grader):
    output = run_prompt(test_case, prompt_template)
    score = grader(output, test_case)
    return {"output": output, "test_case": test_case, "score": score}

def run_eval(dataset, prompt_template, grader):
    return [run_test_case(tc, prompt_template, grader) for tc in dataset]
```

評估管線的核心要素：資料集 + prompt + LLM + 評分器。

### 4.5 模型評分

模型評分器透過額外的 API 呼叫對輸出進行評分，靈活度高但可能不一致。

三種評分器類型：

| 類型 | 說明 | 特性 |
|------|------|------|
| 程式碼評分器 | 程式化檢查（長度、詞彙存在性、語法驗證） | 確定性強 |
| 模型評分器 | 呼叫額外 LLM 評估輸出 | 靈活但可能不一致 |
| 人工評分器 | 人工評估回應 | 最靈活但耗時 |

模型評分器實作：要求輸出優缺點與推理過程（不要只要求分數，否則容易得到趨中的分數），使用 JSON 格式回應搭配 pre-fill 與 stop sequences。

### 4.6 程式碼評分

```python
def validate_json(output):
    try:
        json.loads(output)
        return 10
    except:
        return 0

def validate_python(output):
    try:
        ast.parse(output)
        return 10
    except:
        return 0

def validate_regex(output):
    try:
        re.compile(output)
        return 10
    except:
        return 0

# 綜合評分
final_score = (model_score + syntax_score) / 2
```

資料集需包含 `format` 欄位指定預期輸出類型（JSON/Python/RegEx），Prompt 需明確指示模型只輸出原始程式碼，不附任何說明。

---

## 5. Prompt 工程

### 5.1 流程概述

起點：撰寫初始 prompt（通常效果差）→ 逐步套用技術 → 每次技術套用後評估改善幅度 → 觀察效能隨時間的提升。

技術評估框架：更新後的評估管線支援並發處理（依速率限制調整 `max_concurrent_tasks`），輸出 HTML 格式的評估報告顯示測試案例結果與分數。

### 5.2 清楚直接（Clear and Direct）

Prompt 最重要的部分是第一行，它奠定 AI 回應的基礎。

結構：動詞 + 明確任務描述 + 輸出規格

```
# 差
"Tell me about solar panels"

# 好
"Write three paragraphs about how solar panels work"
"Identify three countries that use geothermal energy and for each include generation stats"
```

效果：分數範例從 2.32 提升至 3.92。

### 5.3 具體明確（Being Specific）

兩種指引類型：

| 類型 | 用途 | 適用時機 |
|------|------|----------|
| Type A（屬性）| 列出輸出的所需特性（長度、結構、格式）| 幾乎所有 prompt 都推薦使用 |
| Type B（步驟）| 提供推理過程的具體步驟 | 複雜問題，需要模型考慮更廣泛視角時 |

效果：範例中加入指引後分數從 3.92 跳至 7.86。

### 5.4 XML 標籤結構化

當 prompt 中需插入大量內容時，用 XML 標籤區分不同類型的資訊，幫助模型理解文字分組。

```
<sales_records>
  ...實際銷售資料...
</sales_records>

<my_code>
  ...程式碼...
</my_code>

<docs>
  ...說明文件...
</docs>
```

標籤命名使用描述性名稱（`sales_records` 比 `data` 好）。即使插入內容較短，也可以用標籤說明這是需要被考量的外部輸入。

### 5.5 提供範例（One-shot / Multi-shot）

One-shot = 一個範例，Multi-shot = 多個範例。

```
<example>
  <input>使用者輸入</input>
  <ideal_output>
    理想輸出（附推理說明）
  </ideal_output>
</example>
```

最佳實踐：
- 說明範例為何是理想輸出，不只提供結果
- 針對邊界案例加上說明（「對諷刺語氣要特別注意」）
- 使用評估中分數最高的範例作為範本
- 範例放在主要指令與指引之後

---

## 6. Tool Use

### 6.1 概念

工具讓 Claude 能存取訓練資料以外的外部資訊。

工具使用流程：

1. 傳送初始請求給 Claude，含存取外部資料的指令
2. Claude 判斷是否需要外部資料，要求特定資訊
3. 伺服器執行程式碼，從外部來源擷取資料
4. 傳送後續請求給 Claude，附上擷取到的資料
5. Claude 依原始 prompt 與外部資料生成最終回應

### 6.2 工具函式

工具函式是 Claude 判斷需要額外資訊時呼叫的 Python 函式。

```python
def get_current_datetime(date_format="%Y%m%d %H:%M:%S"):
    if not date_format:
        raise ValueError("date format cannot be empty")
    return datetime.now().strftime(date_format)
```

最佳實踐：
- 使用描述性函式名稱與參數名稱
- 對無效輸入立即拋出錯誤，附上有意義的錯誤訊息
- 錯誤訊息對 Claude 可見，Claude 可依此更正參數重試

### 6.3 Tool Schemas

Tool Schema 是描述工具函式及其參數的 JSON schema 規格，讓 Claude 了解可用工具、所需參數與使用情境。

結構：
```json
{
  "name": "get_current_datetime",
  "description": "3-4 句說明工具做什麼、何時使用、回傳什麼資料",
  "input_schema": {
    "type": "object",
    "properties": {
      "date_format": {
        "type": "string",
        "description": "strftime 格式字串"
      }
    },
    "required": []
  }
}
```

快速生成 schema 的方式：將工具函式與 Anthropic API 文件一起提供給 Claude，請 Claude 生成符合規格的 JSON schema。

實作：使用 `ToolParam` 包裝 schema 字典以避免型別錯誤。

### 6.4 訊息區塊處理

加入工具後，訊息內容從單純的文字區塊變為多個區塊。

工具回應格式（assistant 訊息）：
- 文字區塊：給使用者看的說明
- 工具使用區塊：包含函式名稱與執行參數

關鍵：append 至訊息歷史時，必須 append 整個 `response.content`（所有區塊），而非只取文字。

### 6.5 傳送工具結果

```python
# 工具結果區塊結構
tool_result = {
    "type": "tool_result",
    "tool_use_id": tool_use_block.id,   # 對應原始工具使用區塊的 ID
    "content": json.dumps(tool_output),  # 轉為字串
    "is_error": False                    # 執行失敗時設為 True
}

# 工具結果放在 user 訊息中，不是 assistant 訊息
messages.append({"role": "user", "content": [tool_result]})
```

後續請求必須包含完整訊息歷史（原始使用者訊息 + 含工具使用的 assistant 訊息 + 含工具結果的新 user 訊息），且即使不再使用工具，也必須附上原始 tool schemas。

### 6.6 多輪工具對話

```python
def run_conversation(messages, tools):
    while True:
        response = chat(messages, tools)
        add_assistant_message(messages, response.content)
        
        if response.stop_reason != "tool_use":
            break
        
        tool_results = run_tools(response)
        add_user_message(messages, tool_results)
    
    return text_from_message(response)

def run_tools(message):
    tool_use_blocks = [b for b in message.content if b.type == "tool_use"]
    results = []
    for block in tool_use_blocks:
        try:
            output = run_tool(block.name, block.input)
            results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": json.dumps(output),
                "is_error": False
            })
        except Exception as e:
            results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": str(e),
                "is_error": True
            })
    return results
```

`stop_reason = "tool_use"` 表示 Claude 需要呼叫工具；其他值表示已完成。

### 6.7 新增多個工具

加入新工具只需 3 步：

1. 將 tool schema 加入 `run_conversation` 的 tools 列表
2. 在 `run_tool` 中加入對應新工具名稱的 if 分支
3. 實作實際的工具函式

工具鏈：Claude 可在單次對話中依序使用多個工具（例如先計算日期，再設定提醒）。

### 6.8 Batch Tool

Batch Tool 讓 Claude 在單一 assistant 訊息中並行執行多個工具，減少不必要的順序呼叫。

```python
# Batch tool schema
batch_schema = {
    "name": "batch",
    "description": "同時執行多個工具",
    "input_schema": {
        "type": "object",
        "properties": {
            "invocations": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "tool_name": {"type": "string"},
                        "arguments": {"type": "object"}
                    }
                }
            }
        }
    }
}

def run_batch(invocations):
    return [run_tool(inv["tool_name"], inv["arguments"]) for inv in invocations]
```

原理：提供更高層次的抽象，讓 Claude 透過 batch 工具手動實現多個工具使用區塊的效果，達到單次請求-回應循環完成多個工具執行。

### 6.9 工具擷取結構化資料

相較於 pre-fill + stop sequences，使用工具擷取結構化資料更可靠，但設定較複雜。

```python
# 強制工具呼叫
response = client.messages.create(
    model=model,
    max_tokens=1000,
    messages=messages,
    tools=[extraction_schema],
    tool_choice={"type": "tool", "name": "extract_data"}
)

# 從工具使用區塊取出結構化資料（不需要工具結果）
structured_data = response.content[0].input
```

| 方式 | 可靠性 | 複雜度 | 適用場景 |
|------|--------|--------|----------|
| Pre-fill + Stop sequences | 一般 | 低 | 快速/簡單擷取 |
| 工具 | 高 | 較高 | 複雜/需要可靠性的擷取 |

### 6.10 Fine-Grained Tool Calling 與串流

工具串流會新增 `input_json_delta` 事件，包含 `partial_json`（塊）與 `snapshot`（累計總和）。

| 模式 | 行為 | 優缺點 |
|------|------|--------|
| 預設 | API 緩衝 JSON 塊，驗證後再傳送 | 較慢但 JSON 已驗證 |
| Fine-grained (`fine_grained: true`) | 立即傳送，跳過 API 端驗證 | 更快但可能有無效 JSON |

Fine-grained 適用於需要即時 UI 更新或提前處理工具參數的場景。

---

## 7. 進階功能

### 7.1 Text Edit Tool

Text Edit Tool 是 Claude 的內建工具，支援檔案/文字操作（讀取、寫入、建立、取代、復原）。

特性：
- JSON schema 已內建於 Claude，只需傳送精簡的 schema stub，Claude 會自動展開為完整 schema
- Claude 3.5 與 3.7 的 schema type 字串（含日期）不同
- 實際的檔案系統操作需自行實作

```python
# Schema stub（Claude 自動展開）
text_editor_schema = {
    "type": "text_editor_20250124",  # 版本依 Claude model 調整
    "name": "str_replace_editor"
}
```

使用情境：複製 AI 程式碼編輯器功能、自動化程式碼生成/重構、多檔案專案操作。

### 7.2 Web Search Tool

Web Search Tool 是 Claude 的內建工具，讓 Claude 能搜尋網路取得最新資訊，不需要自行實作任何程式碼。

```python
web_search_schema = {
    "type": "web_search_20250305",
    "name": "web_search",
    "max_uses": 5,                          # 限制搜尋次數
    "allowed_domains": ["nih.gov"]          # 可選：限制搜尋網域
}
```

回應結構包含四種區塊：文字區塊、工具使用區塊（搜尋查詢）、網頁搜尋結果區塊（標題、URL）、引用區塊（支撐特定陳述的文字）。

### 7.3 Extended Thinking

Extended Thinking 讓 Claude 在生成最終回應前先進行推理。

```python
response = client.messages.create(
    model=model,
    max_tokens=16000,
    thinking={
        "type": "enabled",
        "budget_tokens": 10000  # 最少 1024 tokens，max_tokens 必須大於此值
    },
    messages=messages
)
```

回應結構：
- 思考區塊：包含推理文字 + 密碼學簽章（防止篡改）
- 文字區塊：最終回應

特殊情況：被安全系統標記的思考內容會以加密的 redacted thinking block 呈現，提供對話連續性而不洩漏內容。

使用時機：prompt 優化後仍無法達到所需準確度時才啟用；使用 prompt 評估確認是否必要。

### 7.4 圖像支援

```python
message = client.messages.create(
    model=model,
    max_tokens=1000,
    messages=[{
        "role": "user",
        "content": [
            {
                "type": "image",
                "source": {
                    "type": "base64",
                    "media_type": "image/png",
                    "data": base64.standard_b64encode(image_bytes).decode("utf-8")
                }
            },
            {"type": "text", "text": "詳細分析這張圖片..."}
        ]
    }]
)
```

限制：每次請求最多 100 張圖片；有尺寸限制；圖片依像素高寬計算消耗 token。

圖像準確度完全取決於 prompt 的品質：逐步分析指令、one-shot/multi-shot 範例、明確的指引與驗證步驟都能顯著提升結果。

### 7.5 PDF 支援

```python
{
    "type": "document",
    "source": {
        "type": "base64",
        "media_type": "application/pdf",
        "data": base64.standard_b64encode(pdf_bytes).decode("utf-8")
    }
}
```

與圖像處理程式碼幾乎相同，只需將 `"image"` 改為 `"document"`，media type 改為 `"application/pdf"`。Claude 可同時讀取 PDF 中的文字、圖片、圖表與表格。

### 7.6 Citations

```python
response = client.messages.create(
    model=model,
    max_tokens=1000,
    messages=[{
        "role": "user",
        "content": [
            {
                "type": "document",
                "title": "來源文件標題",
                "source": {"type": "base64", "media_type": "application/pdf", "data": ...},
                "citations": {"enabled": True}
            },
            {"type": "text", "text": "問題..."}
        ]
    }]
)
```

引用類型：
- `citation_page_location`：PDF 文件，含文件索引/標題/起始頁/結束頁/引用文字
- `citation_char_location`：純文字，含字元位置

用途：讓使用者能驗證 Claude 的資訊來源、確認解讀準確性；UI 可實作引用彈出視窗，顯示來源頁碼與原文。

### 7.7 Prompt Caching

Prompt Caching 透過重用前次請求的計算結果來加速回應並降低成本。

運作原理：一般請求完成後，Claude 丟棄所有處理工作。Prompt Caching 將輸入訊息的處理結果存入暫時快取，後續含有相同輸入的請求直接取用快取，無需重新處理。

規則：
- 快取有效期：最長 1 小時
- 需手動在訊息區塊加入快取斷點（`cache_control`）
- 快取範圍：斷點前的所有內容
- 斷點前任何內容變動都會使快取失效
- 處理順序：tools → system prompt → messages
- 每次請求最多 4 個斷點
- 最低快取門檻：1024 tokens

```python
# 文字區塊的完整格式（才能加入 cache_control）
{
    "type": "text",
    "text": "內容...",
    "cache_control": {"type": "ephemeral"}
}
```

```python
# 工具 schema 快取：在最後一個工具加入 cache_control
tools_with_cache = tools.copy()
last_tool = dict(tools_with_cache[-1])
last_tool["cache_control"] = {"type": "ephemeral"}
tools_with_cache[-1] = last_tool

# System prompt 快取
system_with_cache = [{"type": "text", "text": system_prompt, "cache_control": {"type": "ephemeral"}}]
```

Token 使用模式：
- `cache_creation_input_tokens`：首次使用時寫入快取的 token 數
- `cache_read_input_tokens`：後續相同請求從快取讀取的 token 數

### 7.8 Code Execution 與 Files API

Files API：事先上傳檔案，之後透過 file ID 引用，不需要在每次請求中帶入原始檔案資料。

```python
# 上傳檔案
with open("data.csv", "rb") as f:
    file_metadata = client.beta.files.upload(file=("data.csv", f, "text/csv"))
file_id = file_metadata.id

# 後續請求引用 file ID
```

Code Execution：伺服器端工具，Claude 在隔離的 Docker 容器中執行 Python 程式碼，不需要自行實作，只需包含預定義的工具 schema。

限制：Docker 容器無網路存取，資料輸入/輸出依賴 Files API 整合。

```python
# Claude 可在容器中生成檔案（圖表、報告），透過回應中的 file ID 下載
```

---

## 8. Retrieval-Augmented Generation（RAG）

### 8.1 概念

RAG 解決如何用 Claude 查詢大型文件（100–1000+ 頁）而不超出 context 限制的問題。

| 方法 | 說明 | 限制 |
|------|------|------|
| 直接放入 prompt | 將整份文件放入 prompt | Token 上限、長 prompt 效果下降、成本高、速度慢 |
| RAG | 將文件切塊，只取最相關的塊放入 prompt | 較複雜、需前處理、需搜尋機制 |

RAG 流程：切塊 → 使用者提問 → 找到相關塊 → 將相關塊加入 prompt → 送給 LLM。

RAG 以複雜度換取可擴展性與效率，需要仔細實作與評估。

### 8.2 文字切塊策略

切塊品質直接影響 RAG 效能。差的切塊會導致擷取到不相關的內容。

| 策略 | 說明 | 優缺點 |
|------|------|--------|
| 大小切塊 | 切成等長字串 | 易實作、常見；可能切斷詞彙、缺乏上下文 |
| 結構切塊 | 依文件結構（標題、段落）切分 | 適合結構化文件；需保證格式一致 |
| 語義切塊 | 用 NLP 依語義相似度分組 | 最精確；實作最複雜 |

大小切塊的重疊策略：在相鄰塊之間保留部分字元以保存上下文，以文字重複換取塊的完整性。

沒有通用的最佳切塊方法，取決於文件類型保證與具體用途。

### 8.3 Text Embeddings

Embedding 是文字語義的數值表示，由 embedding 模型生成（輸出 -1 到 +1 範圍的長數字列表）。

語義搜尋使用 embedding 找到與使用者問題相關的文字塊，實現語義相似度比對而非關鍵字比對。

Anthropic 推薦使用 Voyage AI 生成 embedding（需另外申請帳號與 API key，免費開始使用）。

### 8.4 完整 RAG 流程

```
前處理（一次性）：
Step 1: 文字切塊 → Step 2: 生成 embedding → Step 3: 正規化 → Step 4: 存入向量資料庫

即時查詢：
Step 5: 查詢轉 embedding → Step 6: 相似度搜尋（餘弦相似度）→ Step 7: 組裝 prompt → LLM 回應
```

關鍵數學概念：
- 餘弦相似度：向量夾角的餘弦值，-1 到 1，越接近 1 越相似
- 餘弦距離：1 減餘弦相似度，越接近 0 相似度越高

```python
# 向量資料庫實作
store = VectorIndex()
for chunk, embedding in zip(chunks, embeddings):
    store.add_vector(embedding, {"content": chunk})

# 查詢
user_embedding = generate_embedding(user_query)
results = store.search(user_embedding, top_k=2)
```

### 8.5 BM25 詞彙搜尋

BM25（Best Match 25）是 RAG 管線中補充語義搜尋的詞彙搜尋演算法。

語義搜尋的缺點：可能錯過精確的詞彙比對，即使某特定詞彙頻繁出現在某文件中也可能返回不相關結果。

BM25 演算法步驟：
1. 將查詢拆分為獨立詞彙（去除標點、空格分隔）
2. 統計每個詞彙在所有塊中的出現頻率
3. 依頻率賦予相對重要性（罕見詞 = 高重要性；常見詞如 "a" = 低重要性）
4. 依塊中高權重詞彙的出現次數排名

```python
# 語義搜尋 + BM25 使用相似 API（add_document, search）
vector_results = vector_index.search(query_embedding, top_k=5)
bm25_results = bm25_index.search(query_tokens, top_k=5)
```

### 8.6 Multi-Index RAG Pipeline

結合向量索引（語義搜尋）與 BM25 索引（詞彙搜尋）的混合搜尋系統。

Reciprocal Rank Fusion（RRF）合併兩個索引的結果：

```
RRF_score = sum(1 / (rank + 1)) 對每個文件跨所有搜尋方法求和
```

範例：向量搜尋回傳 [doc2, doc7, doc6]，BM25 回傳 [doc6, doc2, doc7]，RRF 最終排名：[doc2, doc6, doc7]（doc2 在兩種方法中都排前）。

### 8.7 重新排名（Reranking）

Reranking 是初步擷取後，使用 LLM 依相關性重新排序結果的後處理步驟。

```python
# 使用文件 ID 而非完整文字（效率更高）
rerank_prompt = f"""
使用者查詢：{user_query}

候選文件：
{format_candidates(candidates)}

請依相關性從高到低排列文件 ID，以 JSON 陣列回傳。
"""
# assistant prefill + stop sequence 確保結構化輸出
```

優缺點：提升搜尋準確度（特別是「ENG team」vs「engineering team」這類細微查詢），但增加一次 LLM 呼叫的延遲。

### 8.8 Contextual Retrieval

文件切塊後，個別塊會失去原始文件的上下文，降低擷取準確度。

解決方案：在切塊插入向量/BM25 索引前，先讓 LLM 為每個塊生成情境說明。

```python
def add_context(chunk, source_document):
    # 若原始文件太大，使用選擇性上下文策略：
    # - 包含開頭的 1-3 個塊（摘要/概述）
    # - 包含目標塊前的相鄰塊（局部上下文）
    # - 跳過中間不相關的塊
    
    context = generate_context(chunk, source_document)
    return f"{context}\n\n{chunk}"  # 情境化塊
```

效益：切塊保留與大型文件結構及交叉引用的連結，提升複雜文件的擷取準確度。

---

## 9. Model Context Protocol（MCP）

### 9.1 概念

MCP 是為 Claude 提供上下文與工具的通訊層，讓開發者不需要撰寫繁瑣的工具整合程式碼。

| 面向 | 說明 |
|------|------|
| 架構 | MCP client 連接 MCP server；server 包含 tools、resources、prompts |
| 解決的問題 | 消除為每個服務整合撰寫工具 schema 和函式的負擔 |
| 與 tool use 的關係 | 互補，不是替代——MCP 決定「誰」執行工作（server vs 開發者），兩者都涉及工具 |
| MCP server 由誰建立 | 任何人；服務提供商常建立官方實作（如 AWS） |

核心價值：將整合負擔從應用開發者轉移至 MCP server 維護者。

### 9.2 MCP Client 運作流程

```
使用者查詢 → 伺服器向 MCP client 請求工具列表 → MCP client 向 MCP server 請求
→ MCP server 回傳工具列表 → 伺服器傳送查詢 + 工具給 Claude
→ Claude 請求工具執行 → 伺服器請求 MCP client 執行工具
→ MCP client 請求 MCP server → MCP server 執行工具（如 GitHub API 呼叫）
→ 結果沿鏈回傳：MCP server → MCP client → 伺服器 → Claude → 使用者
```

MCP client 採用傳輸無關設計（可透過 stdio、HTTP、WebSockets 通訊）。

### 9.3 用 MCP Python SDK 定義工具

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

SDK 優點：從 Python 函式定義和裝飾器自動生成 tool JSON schema，無需手動撰寫。

### 9.4 MCP Inspector

MCP Inspector 是在瀏覽器中測試 MCP server 的除錯工具，不需要連接完整應用程式。

```bash
mcp dev server.py
# 在瀏覽器開啟提供的 URL
```

功能：連接 server → 瀏覽可用工具 → 手動輸入參數執行工具 → 驗證輸出。

### 9.5 實作 MCP Client

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

常見模式：將 ClientSession 包裝在較大的類別中進行資源管理，而非直接使用 session。

### 9.6 Resources

MCP Resources 讓 MCP server 向 client 暴露資料供讀取操作。

| 類型 | URI 範例 | 說明 |
|------|----------|------|
| 直接 resource | `docs://documents` | 靜態 URI |
| 樣板 resource | `docs://documents/{doc_id}` | 參數化 URI |

```python
@mcp.resource("docs://documents", mime_type="application/json")
def list_documents() -> dict:
    return docs

@mcp.resource("docs://documents/{doc_id}", mime_type="text/plain")
def get_document(doc_id: str) -> str:
    return docs.get(doc_id, "")
```

Resources vs Tools：Resources 主動提供資料（如 @ 提及時擷取文件）；Tools 被動等待 Claude 決定呼叫。

Client 端讀取：

```python
async def read_resource(self, uri):
    result = await self.session.read_resource(AnyURL(uri))
    resource = result.contents[0]
    if resource.mime_type == "application/json":
        return json.loads(resource.text)
    return resource.text
```

### 9.7 Prompts

MCP Prompts 是 MCP server 預先定義、測試過的 prompt 模板，供 client 應用程式使用。

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

Client 端：

```python
async def list_prompts(self):
    result = await self.session.list_prompts()
    return result.prompts

async def get_prompt(self, prompt_name, arguments):
    result = await self.session.get_prompt(prompt_name, arguments)
    return result.messages  # 可直接送給 LLM
```

在 client 應用程式中，Prompts 通常以自動完成選項（slash commands）呈現，提示使用者輸入所需參數，然後執行預建的 prompt 工作流程。

---

## 10. Claude Code

### 10.1 概述

Claude Code 是終端機介面的程式碼助理，可搜尋/讀取/編輯檔案、使用進階工具（網頁擷取、終端機存取）、並支援 MCP client 功能。

```bash
npm install -g @anthropic-ai/claude-code
claude  # 登入 Anthropic 帳號
```

### 10.2 操作方式

初始化工作流程：

```bash
# 讓 Claude 讀取 README 並執行設定
> 請讀取 README 並執行設定步驟

# 初始化 Claude 掃描程式庫，建立 claude.md 記錄架構與程式碼風格
/init
```

`claude.md` 會自動包含在後續請求的上下文中。可透過 `#` 符號新增筆記至記憶，或手動編輯 `claude.md`。

有效的 prompting 策略：

方法一（三步驟工作流程）：
1. 識別相關檔案，請 Claude 分析
2. 描述功能，請 Claude 規劃解決方案（先不寫程式碼）
3. 請 Claude 實作計劃

方法二（測試驅動開發）：
1. 提供相關上下文
2. 請 Claude 為功能建議測試案例
3. 選定並實作測試
4. 請 Claude 撰寫程式碼直到測試通過

核心原則：Claude Code 是效率倍增器。指令越詳細，結果越好。把 Claude 當作協作工程師，而非單純的程式碼生成工具。

### 10.3 透過 MCP Server 擴展功能

```bash
# 將 MCP server 加入 Claude Code
claude mcp add document-server "uv run main.py"

# 重啟 Claude Code 以使用新功能
```

常見用途：生產環境監控（Sentry）、專案管理（Jira）、通訊（Slack）、自訂開發工作流程工具。

### 10.4 並行化 Claude Code

核心問題：多個 Claude 實例同時修改相同檔案會造成衝突。

解決方案：Git worktrees 為每個 Claude 實例提供隔離的工作空間。

```bash
# 建立 worktree
git worktree add ../feature-branch feature-branch

# 在獨立目錄中指派任務給 Claude 實例
```

自訂指令（`.claude/commands/` 目錄中的 Markdown 檔案）：

```markdown
<!-- .claude/commands/new-feature.md -->
請在新的 worktree 中實作以下功能：$ARGUMENTS
建立功能分支，完成後提交並準備合併至主分支。
```

工作流程：建立 worktree → 指派任務給 Claude 實例 → 各自在隔離環境工作 → 提交變更 → 合併回主分支（Claude 自動處理合併衝突與 worktree 清理）。

### 10.5 自動化除錯

使用 AI 自動偵測、分析並修正生產環境錯誤：

```yaml
# GitHub Action（每日執行）
- 從 CloudWatch 擷取過去 24 小時的日誌
- Claude 識別錯誤並去重
- Claude 分析每個錯誤並生成修正方案
- 建立含修正方案的 Pull Request
```

優點：捕捉生產環境特有的錯誤（本地開發環境不存在的問題）、減少手動查找日誌的時間、提供帶說明的修正方案、建立可審查的 PR。

---

## 11. Computer Use

### 11.1 概念

Computer Use 讓 Claude 能透過視覺觀察與控制操作電腦介面。

能力：截圖應用程式/瀏覽器、點擊按鈕、輸入文字、自主完成多步驟指令。

主要用途：自動化 QA 測試、UI 互動測試、重複性電腦操作的時間節省、系統性錯誤識別。

### 11.2 運作原理

Computer Use 是工具系統的實作，Claude 不直接操控電腦。

```
特殊 tool schema（精簡版，自動展開）→ Claude 傳送工具使用請求
→ 開發者在運算環境（Docker 容器）中執行
→ 容器執行程式化的按鍵/滑鼠動作
→ 結果（截圖）回傳給 Claude
```

Anthropic 提供 reference implementation（含預建滑鼠/鍵盤執行程式碼的 Docker 容器）。

---

## 12. Agents 與 Workflows

### 12.1 基本區別

| 方式 | 定義 | 適用時機 |
|------|------|----------|
| Workflow | 預先定義的一系列 Claude 呼叫 | 任務細節明確、步驟順序已知 |
| Agent | 使用工具動態規劃完成任務 | 任務細節不明確、需要靈活應對 |

建議：優先使用 workflow 確保可靠性；只在確實需要靈活性時才使用 agent。使用者需要 100% 正常運作的產品，而非酷炫的 agent。

### 12.2 Evaluator-Optimizer Pattern

```
生成 → 評估 → 若不符合則重新生成（帶回饋）→ 直到評估者接受
```

範例：圖片轉 3D 模型工作流程
1. Claude 詳細描述上傳的圖片
2. Claude 使用 CADQuery 依描述建立 3D 模型
3. 渲染模型
4. Claude 比對渲染結果與原圖
5. 若不準確，帶回饋從步驟 2 重試

### 12.3 並行化 Workflow

將一個複雜任務分解為多個同時進行的子任務，再聚合結果。

```
輸入 → [子任務 A | 子任務 B | 子任務 C]（並行）→ 聚合 → 最終輸出
```

優點：每個子任務專注於單一分析、個別 prompt 可獨立評估改善、新增子任務不影響現有子任務。

### 12.4 Chaining Workflow

將大型任務拆分為一系列依序執行的步驟，而非單一複雜的 prompt。

主要用途：當 Claude 在複雜 prompt 中持續忽略約束條件時（常見於含有多個「不要做 X」要求的長 prompt）。

```
步驟 1：傳送初始 prompt，接受不完美的輸出
步驟 2：後續 prompt 請 Claude 依發現的違規重新修改
```

### 12.5 Routing Workflow

將使用者輸入分類，依分類結果路由至對應的處理管線。

```
使用者輸入 → Claude 分類（如：教育/娛樂/技術）→ 對應的專用 prompt 模板 → 輸出
```

不同類別使用不同的語氣與結構：程式設計主題使用教育性語氣（定義/說明），娛樂主題使用引人入勝的語言與鉤子。

### 12.6 Agent 工具設計原則

工具抽象原則：提供通用/抽象工具，而非超特化的工具。

```
# 好（抽象）：bash, web_fetch, file_write
# 差（特化）：refactor_tool, install_dependencies
```

工具組合範例：`get_current_datetime` + `add_duration` + `set_reminder` 透過不同組合可解決各種時間相關任務。

Agent 可在需要時請求額外資訊，創造性地組合工具以達成目標，在任務細節不明確的情況下比 workflow 更靈活。

### 12.7 環境觀察（Environment Inspection）

Agent 在每次行動後需要回饋機制，以了解進度並處理錯誤。

Computer Use 範例：Claude 在每次行動後截圖，以查看環境變化（因為無法預測點擊按鈕的確切結果）。

程式碼編輯範例：修改檔案前，agent 必須先讀取現有內容。

影片製作 agent 範例：
- 使用 Whisper CPP 透過 bash 生成帶時間戳的字幕，驗證對話位置
- 使用 FFmpeg 在時間點截取影片截圖，檢查視覺結果
- 在發布前驗證影片符合預期

環境觀察讓 agent 能評估任務進度、偵測錯誤，並適應意外結果，而非盲目執行。
