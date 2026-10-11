# ship-check

> 給 Claude Code 使用的**上線前永續性稽核（pre-production sustainability audit）** skill。
> 核心問題只有一個：**資料量與運行時間持續增加後，系統的效能與資源用量是否仍在可控範圍內？**

**目前版本：v2** —— 與 v1 的差異見 [v2 和 v1 的差異](#v2-和-v1-的差異)。

以 AI 快速產出的 MVP 通常只在小資料量下驗證過。資料量成長之後，查詢成本、記憶體占用與背景工作耗時會逐步惡化，最終造成服務無回應（假死），而且過程中通常不會產生任何錯誤訊息。

`ship-check` 處理的就是這一層：在系統正式上線或轉為無人值守運行之前，先建立量測基準，依「缺陷何時影響系統」排序，並提出修法與驗證方式。文件與觸發語中的「體檢」指的即是這項稽核。

## 涵蓋範圍

|  | 內容 |
|---|---|
| 涵蓋 | 資料成長與 retention、計算下推（query push-down）與 N+1、記憶體與 cache 安全、並發與背景工作重疊、payload 大小、前端資料抓取協調、UI 狀態的正確呈現 |
| 不涵蓋 | 逐行的 correctness bug（off-by-one、null path、邊界條件錯誤），由 Claude Code 內建的 `/code-review` 負責；資安稽核（權限、注入、寫死的密鑰），由內建的 `/security-review` 負責 |

兩者處理的問題不同：

- **`/code-review`**：這段程式碼**目前**的邏輯是否正確。
- **`ship-check`**：資料量成長**之後**，系統是否仍能維持運作。

每次稽核結束都會提示接續執行 `/code-review`。加固（hardening）本身就是修改程式碼（增加時間窗、cache、上限），這類修改容易引入 regression。本 skill 的起因即是一次加固之後，查詢不存在的 ID 會回傳 500，該問題是由 code review 發現的。

## v2 和 v1 的差異

v1 為首次公開版本。v2 依使用者回饋與實際試跑結果，擴充檢查項目並明確化報告規則。七個稽核維度不變。

### 主要差異

| 面向 | v1 | v2 |
|---|---|---|
| 證據規則 | 僅接受量測值，與可靜態判定的清單項目衝突 | 量測值或靜態證據；分別標註「機制確定・代價未量」與「需量測」 |
| 回應範圍 | Over-fetch 僅視為效能問題 | 增加資料暴露面檢查，涉及存取範圍時轉交 `/security-review` |
| 上限與截斷 | 有 clamp，未要求 server-side enforcement | 上限由伺服器強制；截斷須標記或拒絕，所有 consumer 須讀取標記 |
| 資料保留 | 僅涵蓋 append-only 表 | 納入 upsert-only 表的列淘汰；查詢時間窗須對齊 retention |
| Cache | TTL、eviction、cap、key space | 增加命中率、hot key 淘汰、bypass 參數；query push-down 先於 caching |
| 背景工作 | 僅 in-process overlap guard | 增加跨實例互斥、冪等性、可中斷的 timeout、失敗語意 |
| 外部用量 | 未涵蓋 | Quota 與尖峰 rate limit；計費型資料庫另計 rows read |
| 修法驗證 | 無 | 每項修法附可重跑的驗證與通過條件；baseline 記錄測試覆蓋 |
| 嚴重度 | 三級，未涵蓋條件式缺陷 | 增加「條件觸發・時間未定」（列 🟠）；觀察期預設三個月 |
| 清單判定 | 無固定判定值 | 五種固定判定值；同項多處不通過須全數列出 |
| 交接 | 僅 `/code-review` | 視情況加接 `/security-review`；交接語改為修正後執行 |

### 清單項目數

| 清單 | v1 | v2 |
|---|---|---|
| A 新增 endpoint | 8 | 14 |
| B 新增前端頁面 | 6 | 6 |
| C 新增背景工作 | 7 | 10 |
| D 無人值守工作的死機告警 | 5 | 5 |

### 移除項目

- `checklists.md` 結尾重複的症狀對照表。排查步驟 0 的版本保留。
- 清單 A 的「修改既有 endpoint 是否影響其他 consumer」，改由 `/code-review` 處理。

### 新增說明

- README 新增「開發階段的預防」一節：五條可直接加入 `CLAUDE.md` 或 `AGENTS.md` 的規則。

### 驗證方式與限制

驗證方式：以同一個實際專案（Node.js + Postgres 的儀表板後端）為對象，由未被告知預期結果的 AI 分別依 v1 與 v2 執行稽核並比較結果；其後每次修改皆重新執行。

限制：

- **僅在單一專案上驗證**，每個版本各執行一次。新增項目在其他類型專案（例如無資料庫的服務、純排程腳本）上的適用性尚未實測。
- **所有試跑皆為靜態分析**，未實際量測任何數值。
- **報告篇幅增加。** 清單項目增加，且逐項結果不得省略，報告較 v1 長。

## 核心設計

**1. 鐵律：先量測，再判斷（先量再猜）。** 未取得量測值之前，不提出發現，也不提出修法。未經量測就加入 cache 這類預防性修改，是以新增的複雜度處理尚未證實的問題，而複雜度本身就是後續的故障來源。

兩種例外：可由程式碼直接證明的結構性缺陷（cache key 含當下時間、資料表沒有刪除路徑、排程沒有重疊保護）可先行提出，須標註「機制確定・代價未量」並列出待量測項目；無法量測的項目一律標註「需量測」，不得以假設值填補，也不得略過。

稽核開始時先建立 baseline：各資料表的列數與每日成長量、資料庫檔案大小、各 endpoint 的回應大小（對照頁面實際顯示量）、回應時間（以不存取資料庫的 health-check 為基準）、RSS 隨時間的變化、cache 是否命中、背景工作單輪耗時與觸發間隔的比較、外部服務的計量用量，以及專案現有的自動化測試。

**2. 七個稽核維度。** 資料成長／計算下推／記憶體與 cache／並發與背景工作／payload／前端資料抓取／狀態的正確呈現。

**3. Cache 的四項必要條件。** TTL（存活時間）、Eviction（由誰、何時實際刪除）、Cap（筆數上限）、Key space 是否有界。任一項無法回答即不應上線；缺少其中一項，是 cache 演變為記憶體洩漏並導致服務假死的典型路徑。

**4. 嚴重度依「影響發生的時間」分級，而非抽象的嚴重程度。**

- 🔴 **已經在發生** —— 量測值已超過門檻，或程式碼可證明缺陷目前已觸發
- 🟠 **觀察期內會踩到** —— 目前正常，但在關注的時間範圍內會超過門檻
- 🟡 **慢燒** —— 沒有上限，但短期內可承受

同一種寫法在 100 筆資料時屬 🟡，資料量增加後會升為 🔴。分級依據是缺陷何時實際影響系統，而非寫法看起來是否危險，因此量測是分級的前提。須由外部事件觸發且時間無法估計的缺陷，列為 🟠 並標註「條件觸發・時間未定」。

**5. 同時列出已檢查且無問題的項目。** 只回報問題會導致對原本安全的部分進行不必要的修改，反而增加系統的脆弱性。

**6. 每項修法附可重跑的驗證。** 每項修法須附驗證方式與通過條件：可量化者斷言數值（回傳列數、payload 大小、cache 筆數上限），不可量化者斷言可觀察行為（上游無回應時工作會結束、同一批次執行兩次僅產生一份結果）。稽核階段僅描述驗證方式，取得同意後才實作。以小型 fixture 驗證結果正確性的測試，在 100 筆與 10 萬筆時都會通過，無法涵蓋此類問題。

## 兩種模式

- **Mode A — 上線前稽核（主動）**：系統即將上線、轉為無人值守，或新增對外介面時，完整執行一次。
- **Mode B — 已上線排查（被動）**：運行中的系統變慢、假死，或在未修改程式碼的情況下出現異常。此時不應先歸因於最近的 commit（時間點與該假設不符），而應檢查**隨時間增長的項目**（資料量、cache 項目數、記憶體），並由外而內分段量測（瀏覽器 → proxy → 後端 → 資料庫），以不存取資料庫的 health-check 作為基準。

## 使用時機

- **MVP 原型階段：不建議使用。** 仍在驗證概念可行性時，規模不是重點，過早加固沒有效益。
- **時機 1**：首次正式部署，或改為排程／無人值守運行之前 → 完整執行
- **時機 2**：新增 endpoint、cache 或背景工作之前 → 僅執行 `references/checklists.md` 中對應的清單
- **時機 3**：風險等級提高之前（開放更多使用者、由測試轉為正式）→ 作為上線閘門
- **時機 4**：已上線的系統變慢或假死 → Mode B 排查
- **每次稽核後** → 接續執行 `/code-review`

## 開發階段的預防

ship-check 屬於上線前的稽核。成本更低的做法是在開發階段就避免這些問題：將下列規則加入專案的 `CLAUDE.md`（或 `AGENTS.md`），AI 在每次工作開始時都會讀取。

```markdown
## 資料存取規則
- 篩選、排序、top-N、聚合、去重一律下推至資料庫，不得取回全部資料後在應用層處理。
- 迴圈內不得發出查詢（N+1），改用批次介面。
- 列表型 endpoint 一律分頁，每頁上限由伺服器強制；結果被截斷時須在回應中明確標示。
- 生命週期長於單一請求的 cache，須具備 TTL、實際刪除機制與筆數上限。
- 排程工作須有重疊保護：前一輪尚未結束時跳過本輪。
```

內建 `/code-review` 在 medium 以上的層級會將改動與 `CLAUDE.md` 中的明文規則比對（依撰寫時的 Claude Code 2.1 版，後續版本可能調整），因此寫入之後，違反規則的改動有機會被指出。

規則能約束寫法，但無法反映資料成長後的實際數值，因此上線前仍須執行 ship-check。

## 安裝

Skill 由一個資料夾與數個 `.md` 檔組成，沒有相依套件，也不需要編譯。

**個人層級（所有專案可用）**

```bash
git clone https://github.com/itskaraburger/ship-check.git
mkdir -p ~/.claude/skills
cp -r ship-check ~/.claude/skills/ship-check
```

**專案層級（僅單一專案）**

```bash
mkdir -p .claude/skills
cp -r ship-check .claude/skills/ship-check
```

安裝後重新啟動 Claude Code。輸入「上線前幫我體檢」、「這個能不能上線」、「會不會越跑越慢」，或直接輸入 `/ship-check`，即可觸發。

安裝後的目錄結構：

```
~/.claude/skills/ship-check/
├── SKILL.md
└── references/
    ├── playbook-principles.md   # 8 章通則：各項判斷的依據
    └── checklists.md            # 4 張 pre-ship 清單、排查手冊、症狀對照表
```

## 檔案說明

| 檔案 | 內容 |
|---|---|
| `SKILL.md` | 主流程：兩種模式、鐵律、7 個維度、嚴重度分級、輸出格式、使用時機 |
| `references/playbook-principles.md` | 知識基礎。8 章通則，每章包含核心原則、判準、自問清單與實例 |
| `references/checklists.md` | 清單 A（新增 endpoint）、B（新增前端頁面）、C（新增背景工作）、D（無人值守工作的死機告警），以及 Mode B 排查步驟（含症狀分類表） |

`references/` 的內容不會常駐於 context，僅在實際執行稽核時載入。

## 備註

文件中的實例數值（26.8 萬列的查詢耗時 7 秒 → 27 列 100ms；回應 222KB → 22KB）皆來自實際專案，已移除專案與環境的識別資訊。

---

**English:** `ship-check` is a Claude Code skill that runs a **pre-production sustainability audit** — the "will this still run in 3 months as the data grows?" layer that a fast AI-built MVP usually skips. It measures real numbers before diagnosing (no finding without a number, unless the code alone proves the defect), audits 7 growth-over-time dimensions (data growth & retention, computation push-down, memory & cache safety, concurrency, payload size, frontend fetching, honest UI state), and ranks findings by *when the pain lands* rather than abstract severity. It deliberately does **not** hunt line-level correctness bugs — that's what `/code-review` is for, and every audit ends by handing off to it. Drop the folder into `~/.claude/skills/` to install.
