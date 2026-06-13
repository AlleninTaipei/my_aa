# Agent Skills 入門

## 1. 什麼是 Skill？

Skills 是可重複使用的 Markdown 檔案，用來教導 Claude Code 如何自動處理特定任務。與其每次請 Claude 審查 PR 或撰寫 commit 訊息時重複說明，不如將指令寫成一個 skill，Claude 就會在每次遇到該任務時自動套用。

### Skill 的定義

Skill 是一個 Markdown 檔案，只需教導 Claude 一次如何完成某件事，Claude 便會在相關情境中自動套用這些知識。

Skills 是 Claude Code 可以探索並使用的指令與資源資料夾，幫助 Claude 更準確地處理特定任務。每個 skill 存放在一個 `SKILL.md` 檔案中，frontmatter 包含名稱與描述。

描述是 Claude 決定是否使用某個 skill 的依據。當你請 Claude 審查 PR 時，Claude 會將請求內容與所有可用 skill 的描述進行比對，找到最符合的一個。以下是 skill frontmatter 的範例：

```yaml
---
name: pr-review
description: Reviews pull requests for code quality. Use when reviewing PRs or checking code changes.
---
```

frontmatter 之後是實際指令——你的審查清單、格式偏好，或 Claude 執行該任務所需了解的任何事項。

### Skill 的存放位置

| 位置 | 路徑 | 適用範圍 |
|------|------|---------|
| 個人 | `~/.claude/skills` | 你所有的專案 |
| 專案 | `.claude/skills`（repo 根目錄） | 所有 clone 此 repo 的人 |

在 Windows 上，個人 skill 存放於 `C:/Users/<your-user>/.claude/skills`。

專案 skill 會隨程式碼一起提交至版本控制，整個團隊共同使用。

### Skills vs. CLAUDE.md vs. Slash Commands

| 功能 | 載入方式 | 最適合 |
|------|---------|-------|
| `CLAUDE.md` | 每次對話都載入 | 永遠適用的專案規範 |
| Skills | 依需求，符合條件時載入 | 特定任務的專業知識 |
| Slash commands | 明確呼叫 | 手動、一次性任務 |

### 何時使用 Skill

Skills 最適合用於特定任務的專業知識：

- 團隊的程式碼審查標準
- 個人偏好的 commit 訊息格式
- 組織的品牌規範
- 特定類型文件的文件範本
- 特定框架的除錯清單

判斷原則：如果你發現自己一再向 Claude 解釋同樣的事情，那就是一個等待被寫成 skill 的場景。

---

## 2. 建立第一個 Skill

本節從頭建立一個 skill——個人 PR 描述 skill，適用於所有專案。

### 建立 Skill

我們將建立一個個人 skill，教導 Claude 以一致的格式撰寫 PR 描述。由於是個人 skill，存放在 home 目錄，適用於所有專案。

首先，在 skills 資料夾內建立 skill 的目錄：

```bash
mkdir -p ~/.claude/skills/pr-description
```

然後在該目錄內建立 `SKILL.md` 檔案：

```markdown
---
name: pr-description
description: Writes pull request descriptions. Use when creating a PR, writing a PR, or when the user asks to summarize changes for a pull request.
---

When writing a PR description:

1. Run `git diff main...HEAD` to see all changes on this branch
2. Write a description following this format:

## What
One sentence explaining what this PR does.

## Why
Brief context on why this change is needed.

## Changes
- Bullet points of specific changes made
- Group related changes together
- Mention any files deleted or renamed
```

`name` 用來識別 skill；`description` 告訴 Claude 何時使用它，這是比對的依據。第二個分隔線之後的所有內容，就是 skill 啟動時 Claude 遵循的指令。

### 測試 Skill

Claude Code 在啟動時載入 skills，因此建立 skill 後需要重新啟動工作階段。可以透過查看可用 skills 清單確認是否成功載入。

測試方式：在分支上做一些變更後，輸入「write a PR description for my changes」之類的請求。Claude 會指出正在使用 PR description skill，檢查你的 diff，並依照你的範本撰寫描述——每次格式一致。

### Skill 比對機制

Claude Code 啟動時會掃描四個位置尋找 skills，但只載入名稱與描述，不載入完整內容。當你送出請求時，Claude 會使用語意比對，將訊息與所有可用 skill 的描述進行比較。

一旦找到符合的 skill，Claude 會請你確認是否載入。確認後，Claude 讀取完整的 `SKILL.md` 檔案並按指令執行。

### Skill 優先順序

若兩個 skill 名稱相同，優先順序如下（從高到低）：

1. Enterprise——受管理的設定
2. 個人——`~/.claude/skills`
3. 專案——repository 內的 `.claude/skills`
4. Plugins——已安裝的 plugin

為避免衝突，請使用描述性名稱。比起單純的 `review`，使用 `frontend-review` 或 `backend-review` 之類的名稱更好。

### 更新與移除 Skill

- 更新 skill：編輯對應的 `SKILL.md` 檔案
- 移除 skill：刪除對應目錄
- 任何變更後需重新啟動 Claude Code 方可生效

---

## 3. 設定與多檔案 Skill

本節涵蓋進階技巧：完整的 metadata 欄位、如何撰寫可靠觸發的描述、限制特定工具存取以保護敏感工作流程，以及使用漸進式揭露（progressive disclosure）將較大的 skill 拆分為多個檔案。

### Skill Metadata 欄位

| 欄位 | 必填 | 說明 |
|------|------|------|
| `name` | 是 | 只允許小寫字母、數字、連字號，最多 64 字元，應與目錄名稱一致 |
| `description` | 是 | 告訴 Claude 何時使用此 skill，最多 1,024 字元 |
| `allowed-tools` | 否 | 限制 skill 啟動時 Claude 可使用的工具 |
| `model` | 否 | 指定此 skill 使用的 Claude 模型 |

### 撰寫有效的描述

好的描述要回答兩個問題：

1. 這個 skill 做什麼？
2. Claude 何時應該使用它？

如果 skill 沒有在預期時機觸發，請加入更多符合你實際請求措辭的關鍵字。

### 使用 `allowed-tools` 限制工具

有時你想要一個只能讀取檔案、無法修改的 skill。這對於安全敏感的工作流程、唯讀任務，或需要設定防護欄的情境特別有用。

```yaml
---
name: codebase-onboarding
description: Helps new developers understand how the system works.
allowed-tools: Read, Grep, Glob, Bash
model: sonnet
---
```

此 skill 啟動時，Claude 只能使用這些工具而無需額外申請權限——不能編輯、不能寫入。若完全省略 `allowed-tools`，skill 則不限制任何工具。

### 漸進式揭露（Progressive Disclosure）

Skills 與你的對話共用 Claude 的 context window。將所有內容塞進一個大型檔案有兩個問題：佔用太多 context window 空間，且難以維護。

漸進式揭露的解法：將核心指令保留在 `SKILL.md`，詳細的參考資料放在獨立檔案，讓 Claude 只在需要時讀取。

建議的目錄結構：

```
my-skill/
├── SKILL.md
├── scripts/      # 可執行程式碼
├── references/   # 補充文件
└── assets/       # 圖片、範本或其他資料檔案
```

在 `SKILL.md` 中連結到輔助檔案，並清楚說明何時載入。Claude 只在有人詢問系統設計時才讀取 `architecture-guide.md`——如果使用者只是問在哪裡新增元件，這個檔案永遠不會被載入。

參考原則：將 `SKILL.md` 控制在 500 行以內。若超過此限制，請將內容拆分至獨立的參考檔案。

### 有效使用腳本

Skill 目錄中的腳本無需將內容載入 context 即可執行。腳本執行後，只有輸出結果消耗 token。在 `SKILL.md` 中告訴 Claude 執行腳本，而非讀取腳本。

這對以下情境特別有用：

- 環境驗證
- 需要保持一致性的資料轉換
- 作為測試過的程式碼比即時生成更可靠的操作

---

## 4. Skills vs. 其他 Claude Code 功能

Claude Code 提供多種自訂選項，選擇錯誤的方式可能導致不必要的複雜性。

### CLAUDE.md vs. Skills

`CLAUDE.md` 在每次對話中都會載入。Skills 則依需求，符合條件時才載入。

| 使用 `CLAUDE.md` 的情境 | 使用 Skills 的情境 |
|------------------------|------------------|
| 永遠適用的全專案規範 | 特定任務的專業知識 |
| 「永不修改資料庫 schema」之類的限制 | 只在部分情況適用的知識 |
| 框架偏好與程式碼風格 | 若出現在每次對話中會造成干擾的詳細流程 |

### Skills vs. 子代理（Subagents）

Skills 為你目前的對話新增知識——skill 啟動時，其指令會加入既有的 context。

子代理（Subagents）在獨立的 context 中執行，接收任務、獨立處理，並回傳結果。

| 使用子代理的時機 | 使用 Skills 的時機 |
|----------------|------------------|
| 想將任務委派至獨立的執行 context | 想強化 Claude 在當前任務的知識 |
| 需要與主對話不同的工具存取權限 | 專業知識適用於整個對話過程 |
| 想隔離委派工作與主 context | |

### Skills vs. Hooks

Hooks 依事件觸發。Skills 由請求驅動——根據你的提問決定是否啟動。

| 使用 Hooks 的時機 | 使用 Skills 的時機 |
|----------------|------------------|
| 每次儲存檔案都應執行的操作 | 能影響 Claude 處理請求方式的知識 |
| 特定工具呼叫前的驗證 | 影響 Claude 推理過程的指引 |
| Claude 動作的自動化副作用 | |

### 整合運用

典型的設置可能包含：

- `CLAUDE.md`——永遠啟用的專案規範
- Skills——依需求載入的特定任務專業知識
- Hooks——由事件觸發的自動化操作
- 子代理——委派工作的隔離執行 context
- MCP 伺服器——外部工具與整合

---

## 5. 分享 Skills

跨團隊或整個組織分享 skills，能讓它們的價值大幅提升。

### 將 Skills 提交至 Repository

最簡單的分享方式是直接將 skills 提交至 repository。將 skills 放在 `.claude/skills`，任何 clone 此 repo 的人都能自動取得這些 skills——無需額外安裝。

適用情境：

- 團隊程式碼規範
- 專案特定的工作流程
- 參考了程式碼庫結構的 skills

`.claude` 目錄包含你的 agents、hooks、skills 與設定——所有內容都透過一般的 Git 工作流程進行版本控制與分享。

### 透過 Plugins 分發 Skills

Plugins 是一種擴充 Claude Code 自訂功能的方式，設計用於跨團隊與專案分享。在 plugin 專案中，建立一個 `skills` 目錄，其結構與 `.claude` 相同——每個 skill 都有自己的資料夾，內含 `SKILL.md` 檔案。

當你的 skills 不侷限於特定專案，且能對更廣泛的社群成員有所助益時，這個方式最為適合。

### 透過受管理設定進行企業部署

管理員可透過受管理設定（managed settings）在整個組織中部署 skills。Enterprise skills 具有最高優先順序——它們會覆蓋同名的個人、專案與 plugin skills。

受管理設定檔支援 `strictKnownMarketplaces` 等功能，用於控制 plugin 的安裝來源：

```json
"strictKnownMarketplaces": [
  {
    "source": "github",
    "repo": "acme-corp/approved-plugins"
  },
  {
    "source": "npm",
    "package": "@acme-corp/compliance-plugins"
  }
]
```

這是強制規範、安全要求、合規工作流程，以及必須在整個組織保持一致的程式碼實踐的最佳選擇。

### Skills 與子代理

子代理不會自動看到你的 skills。當你將任務委派給子代理時，它從全新、乾淨的 context 開始。

- 內建代理（Explorer、Plan、Verify）完全無法存取 skills
- 你自訂的子代理可以使用 skills，但只在明確列出的情況下
- Skills 在子代理啟動時載入，而非依需求載入

若要建立帶有 skills 的自訂子代理，在 `.claude/agents` 中新增一個 agent markdown 檔案。使用 Claude Code 中的 `/agents` 指令以互動方式建立。生成的 agent 檔案包含 `skills` 欄位：

```yaml
---
name: frontend-security-accessibility-reviewer
description: "Use this agent when you need to review frontend code for accessibility..."
tools: Bash, Glob, Grep, Read, WebFetch, WebSearch, Skill
model: sonnet
color: blue
skills: accessibility-audit, performance-check
---
```

此模式適用於以下情境：

- 想要帶有特定專業知識的隔離任務委派
- 不同子代理需要不同的 skills（前端審查員 vs. 後端審查員）
- 想在委派工作中強制執行規範而不依賴提示詞

---

## 6. Skills 疑難排解

Skills 無法如預期運作時，問題通常落在幾個可預測的類別中。

### 使用 Skills 驗證器

首先嘗試 agent skills 驗證指令。使用 `uv` 是最快的設定方式。安裝後，切換至 skill 目錄或從任意位置執行指令，驗證器會在你花時間除錯其他問題之前，先找出結構性問題。

### Skill 未觸發

原因幾乎都是描述的問題。Claude 使用語意比對，因此你的請求需要與描述的含義重疊。

處理方式：

- 對照你實際措辭的請求，檢查你的描述
- 加入使用者實際會說的觸發詞
- 用「help me profile this」、「why is this slow?」、「make this faster」等變化測試
- 若任何變化未能觸發，將這些關鍵字加入描述

### Skill 未載入

若詢問 Claude「what skills are available」時沒有出現你的 skill，請檢查以下結構要求：

- `SKILL.md` 必須在已命名的目錄內，不能直接放在 skills 根目錄
- 檔案名稱必須正確為 `SKILL.md`——`SKILL` 全部大寫，`md` 小寫

執行 `claude --debug` 查看載入錯誤，尋找包含你 skill 名稱的訊息。

### 使用了錯誤的 Skill

若 Claude 使用了錯誤的 skill，或在 skills 之間感到困惑，你的描述可能太過相似。讓它們更有區別。描述越具體，不僅能幫助 Claude 決定何時使用你的 skill，也能防止與其他類似 skill 的衝突。

### Skill 優先順序衝突

若你的個人 skill 被忽略，可能是 enterprise 或更高優先順序的 skill 具有相同名稱。例如，若存在 enterprise 的 `code-review` skill，而你也有個人的 `code-review` skill，enterprise 的版本永遠優先。解決方式：

- 將你的 skill 重新命名為更有特色的名稱（通常較容易）
- 與管理員討論 enterprise skill 的問題

### Plugin Skills 未顯示

若你安裝了 plugin 但看不到其 skills，清除快取、重新啟動 Claude Code 並重新安裝。若之後 skills 仍未顯示，plugin 結構可能有問題——這時驗證器工具就能派上用場。

### 執行時期錯誤

Skill 已載入但執行時失敗。常見原因：

- 缺少依賴：外部套件必須已安裝；在 skill 描述中加入依賴資訊，讓 Claude 知道需要什麼
- 權限問題：腳本需要執行權限；對 skill 引用的腳本執行 `chmod +x`
- 路徑分隔符：到處使用正斜線，即使在 Windows 上也一樣

### 快速疑難排解清單

| 症狀 | 解決方式 |
|------|---------|
| 未觸發 | 改善描述，加入觸發詞 |
| 未載入 | 檢查路徑、檔案名稱與 YAML 語法 |
| 使用了錯誤的 skill | 讓各 skill 描述更有區別 |
| 被覆蓋 | 檢查優先順序層次，必要時重新命名 |
| Plugin skills 未顯示 | 清除快取並重新安裝 |
| 執行時期失敗 | 檢查依賴、權限與路徑 |

---

## 課程結語

你已學會在 Claude Code 中建立、設定、分享與疑難排解 skills。當你開始為自己的工作流程建立 skills 時，請記住，最好的 skills 來自真實的痛點——從你最常重複說明的指令開始寫起。
