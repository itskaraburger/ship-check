# ship-check

> 一個給 Claude Code 用的**上線前永續體檢** skill。
> 它只問一個問題：**資料一直長大、時間一直過去，這東西會不會變慢、變肥、卡死？**

**目前版本：v2** —— 和 v1 差在哪裡，見 [v2 和 v1 的差異](#v2-和-v1-的差異)。

AI 幫你很快就做出一個「跑得動」的 MVP。但「在我電腦上、100 筆資料時很順」會安靜地爛成「三個月後 10 萬筆資料，整個卡死」——而且通常不會有任何錯誤訊息。

`ship-check` 就是補那一關的：在東西真正上線、或開始無人值守自己跑之前，先量數字、按「痛什麼時候來」排序、給出修法。

## 這個 skill 做什麼 / 不做什麼

|  | 做 |
|---|---|
| ✅ 做 | 資料成長與 retention、計算下推 / N+1、記憶體與 cache 安全、並發與背景工作重疊、payload 大小、前端抓取協調、誠實的 UI 狀態 |
| ❌ 不做 | 逐行的 correctness bug（off-by-one、null path、錯誤的邊界條件）——那是 Claude Code 內建的 `/code-review` 在做；也不做資安稽核（權限、注入、寫死的密鑰）——那是內建的 `/security-review` |

兩者問的是不同問題：

- **`/code-review`**：這段程式碼**現在**就寫錯了嗎？
- **`ship-check`**：**三個月後**資料長大了，這東西還活著嗎？

所以每次 ship-check 的結尾都會提醒你接著跑 `/code-review`——因為「加固」本身就是在改程式碼（加時間範圍、加 cache、加上限），而這種改動最容易順手弄壞別的東西。這個 skill 的起源就是一次加固之後多出一個「查不存在的 ID 會炸掉」的 bug，靠 review 才抓到。

## v2 和 v1 的差異

v1 是第一次公開的版本。v2 補上讀者回饋與實際試跑時發現的缺口。**七個維度沒有變**，變的是清單問得更細、報告的寫法更明確。

### v2 解決了 v1 的哪些問題

| v1 的限制 | v2 的處理方式 |
|---|---|
| 鐵律「未取得量測值不得提出發現」與清單中可由靜態分析判定的項目互相衝突 | 證據模型改為「量測值，或靜態證據（程式碼位置與推論）」。靜態可證明的缺陷標註「機制確定・代價未量」；無法量測的項目標註「需量測」並寫明量測方法，不得以假設值填補，也不得略過 |
| 無上限的回應（over-fetch）只被視為效能問題 | 增加資料暴露面檢查：誰能取得完整 payload（等同 bulk export）。涉及存取範圍或權限時，轉交 `/security-review` |
| 有 clamp 檢查，但未要求 server-side enforcement，也未處理 silent truncation | 分頁與查詢上限須由伺服器強制，不信任 client 傳入的 `limit`。結果被截斷時須回傳明確標記（如 `truncated`）或直接拒絕請求，且所有 consumer 都須讀取該標記（`stale` 同理） |
| 資料保留（retention）只涵蓋 append-only 表 | 納入 upsert-only 狀態表的列淘汰：失效的列不得永久殘留並進入聚合結果。查詢時間窗超過 retention 時，須拒絕請求或標明實際涵蓋範圍 |
| Cache 只檢查 TTL、eviction、cap、key space 是否有界 | 增加命中率檢查：cache key 含時間戳或亂數時命中率趨近零；hot key 與使用者衍生的 key 共用 cap 時可能被淘汰；cache-bypass 參數須受限。並明訂 query push-down 先於 caching |
| 背景工作只檢查 in-process 的 overlap guard | 增加四項：跨進程／跨實例的互斥（multi-replica、rolling deploy、手動補跑）、重跑的冪等性（idempotency）、外部呼叫的 timeout 須實際中斷連線（abort）、失敗語意（上游失敗不得寫成空值或 0，也不得回報為成功） |
| 未涵蓋外部服務的計量用量 | 檢查 quota（每輪呼叫數 × 每日輪數）與 rate limit（尖峰分鐘），計入請求路徑、排程、重試與重疊執行。按讀取列數計費的資料庫，另計每次請求的 rows read |
| 修法沒有驗證準則 | 每項修法附可重跑的驗證與通過條件，斷言量化上限或可觀察行為。baseline 另記錄專案現有的自動化測試與 CI 覆蓋 |
| 嚴重度分級未涵蓋條件式觸發的缺陷 | 須由外部事件觸發（上游失敗、重試堆積、多實例並行）且發生時間無法估計者，列為 🟠 並標註「條件觸發・時間未定」。觀察期（observation horizon）預設三個月，載明於報告開頭 |
| 清單項目沒有固定的判定值 | 每項判定為「通過／不通過／需量測／不適用／未檢查」之一並附證據。同一項有多處不通過須全數列出。沒有資料庫或沒有 UI 的專案，改以實際的資料源與 consumer 判定 |
| 交接只含 `/code-review`，且措辭預設修改已經完成 | 交接語改為「依建議修正後再執行」。體檢涉及資料存取範圍時，加接 `/security-review` |

### 清單條目數

| 清單 | v1 | v2 |
|---|---|---|
| A 新增 endpoint | 8 | 14 |
| B 新增前端頁面 | 6 | 6 |
| C 新增背景工作 | 7 | 10 |
| D 無人值守死機告警 | 5 | 5 |

### v2 拿掉的

- `checklists.md` 結尾那張重複的症狀表。排查步驟 0 的那一張保留。
- 清單 A 的「改既有 endpoint 會不會破壞其他消費者」。這件事交給 `/code-review`。

### v2 新增的說明

- README 的「寫的當下就預防」：五條可以直接貼進 `CLAUDE.md` 或 `AGENTS.md` 的規則。

### v2 怎麼驗證的，以及還沒驗證的

- 拿同一個實際專案（Node.js + Postgres 的儀表板後端），讓不知道預期答案的 AI 分別照 v1 和 v2 做體檢，比較兩邊抓到的問題；之後每改一輪就重跑一次。

還沒驗證的部分請留意：

- **只測過一個專案**，每個版本只跑一次。新條目在別種專案（例如沒有資料庫的服務、純排程腳本）上問不問得通，還沒有實測。
- **試跑全部是只讀程式碼**，沒有實際量任何數字。
- **報告會變長。** 清單變細、逐條結果不能省略，體檢報告會比 v1 長不少。

## 核心設計

**1. 鐵律：先量再猜。** 沒有數字，不准提出發現，也不准提修法。因為「看起來有風險所以先加個 cache」是拿新的複雜度去解決一個不存在的問題，而複雜度本身就是下一個壞掉的地方。

有兩種情況不算違反：光讀程式碼就能確定的結構性缺陷（cache 的 key 含現在時間、表沒有刪除路徑、排程沒有重疊保護）可以先報，但要標明「機制確定・代價未量」並寫出該量什麼；真的量不到的項目一律標「需量測」，不編數字、也不跳過。

體檢一開始一定先量這些：每張表幾筆 / 一天長幾筆、DB 檔案多大、每個 endpoint 回幾 KB（對比畫面真正顯示的量）、回應時間（先用不碰 DB 的 health-check 當基準線）、RSS 隨時間的曲線、cache 打兩次有沒有變快、背景工作單輪耗時 vs 觸發間隔。

**2. 七個維度。** 資料成長 / 計算下推 / 記憶體與 cache 四問 / 並發與背景工作 / payload / 前端抓取 / 誠實狀態。

**3. Cache 四問，缺一不可。** TTL（活多久）、Eviction（誰真的刪它）、Cap（最多幾筆）、Key 空間有界嗎。四個有一個答不出來就先不要上線——這就是「一個無害的小 cache」變成記憶體漏光、服務假死的完整過程。

**4. 分級按「痛什麼時候來」，不按抽象嚴重度。**

- 🔴 **已經在發生** —— 數字已經越線，或程式碼本身就能證明現在正在發生
- 🟠 **觀察期內會踩到** —— 現在沒事，但在你真正在意的時間範圍內會爆
- 🟡 **慢燒** —— 沒有上限，但短期活得下去

同一個寫法，資料 100 筆時是 🟡，長到一定量就變 🔴。**等級看的是問題什麼時候會真的影響系統**，不是寫法看起來危不危險——這也是為什麼「先量」是前提。要等外部出狀況才發作、時間估不出來的，放 🟠 並標「條件觸發・時間未定」。

**5. 也會列「檢查過、沒問題的」。** 只報壞消息會讓人去「修」本來就安全的東西，那是另一種製造脆弱的方式。

**6. 修法要留下可重跑的驗證。** 每個修法附一個可重跑的檢查和通過條件：有數字就斷言數字（回傳幾列、payload 幾 KB、cache 上限幾筆），沒有數字就斷言看得到的行為（上游卡住時工作會結束、同一批跑兩次只有一份結果）。體檢時只寫出檢查方法，你同意修了才真的寫。只用少量假資料檢查「結果對不對」的測試，100 筆和 10 萬筆時都會過，抓不到這類問題。

## 兩種模式

- **Mode A — 上線前主動體檢**：東西即將上線 / 要無人值守 / 新增一個對外介面。從頭跑一遍。
- **Mode B — 已上線排查**：已經在跑的東西變慢、假死、「我明明沒改 code」。這時**不要怪最近的 commit**——時間點對不上那個理論，要去找**隨時間長大的東西**（資料、cache 條目、記憶體），並且由外而內量（瀏覽器 → proxy → 後端 → DB），從不碰 DB 的 health-check 基準線開始。

## 什麼時候用（含什麼時候不要用）

- **MVP 原型階段：不要用。** 還在驗證想法行不行的時候，規模不重要，提早加固是純浪費。先讓它能動。
- **時機 1**：第一次真正部署 / 要讓它自己排程跑、沒人看著 → 完整跑一遍
- **時機 2**：要新增一個 endpoint / cache / 背景工作 → 只跑 `references/checklists.md` 裡對應那一張清單
- **時機 3**：賭注變大之前（開放給更多人、從測試轉到來真的）→ 當成閘門
- **時機 4**：已上線的東西變慢 / 假死 → Mode B 排查
- **每次體檢後** → 接 `/code-review`

## 寫的當下就預防

ship-check 是上線前的體檢。更便宜的做法是讓 AI 在寫的當下就避開——把下面這段貼進專案的 `CLAUDE.md`（或 `AGENTS.md`），它每次開工都會先讀到：

```markdown
## 資料存取規則
- 篩選、排序、取前 N、聚合、去重一律交給資料庫做，不要撈全部再用程式挑。
- 迴圈裡不發查詢（N+1），改用批次。
- 列表 endpoint 一律分頁，每頁上限由伺服器強制；結果被截斷時要在回應裡明講。
- 活得比單一請求久的 cache，都要有 TTL、真的刪除、筆數上限。
- 排程工作要有重疊保護：上一輪還沒跑完就跳過。
```

內建 `/code-review` 在 medium 以上會拿改動對照 `CLAUDE.md` 裡的明文規則（以撰寫時的 Claude Code 2.1 版為準，之後可能改變），所以寫進去之後，違反的改動有機會被點出來。

規則管得到寫法，管不到資料長大之後的實際數字，所以上線前還是要跑 ship-check。

## 安裝

Skill 就是一個資料夾裝幾個 `.md`，沒有依賴、沒有東西要編譯。

**個人用（所有專案都能用）**

```bash
git clone https://github.com/itskaraburger/ship-check.git
mkdir -p ~/.claude/skills
cp -r ship-check ~/.claude/skills/ship-check
```

**只給某個專案用**

```bash
mkdir -p .claude/skills
cp -r ship-check .claude/skills/ship-check
```

裝完重開 Claude Code。之後說「上線前幫我體檢」、「這個能不能上線」、「會不會越跑越慢」，或直接打 `/ship-check`，就會觸發。

安裝後的結構應該是這樣：

```
~/.claude/skills/ship-check/
├── SKILL.md
└── references/
    ├── playbook-principles.md   # 8 章通則：為什麼這樣判斷
    └── checklists.md            # 4 張 pre-ship 清單 + 排查手冊 + 症狀對照表
```

## 檔案說明

| 檔案 | 內容 |
|---|---|
| `SKILL.md` | 主流程：兩模式、鐵律、7 個維度、分級方式、輸出格式、使用時機 |
| `references/playbook-principles.md` | 知識底層。8 章通則，每章「核心原則 → 一句判準 → 自問清單 → 實戰案例」 |
| `references/checklists.md` | 清單 A 新增 endpoint／B 新增前端頁面／C 新增背景工作／D 無人值守死機告警，加上 Mode B 排查步驟（含症狀分類表） |

`references/` 裡的內容不會一直佔著 context，只有真的在跑體檢時才會被讀進來。

## 備註

裡面的實戰案例數字（26.8 萬列的查詢 7 秒 → 27 列 100ms、222KB → 22KB）都來自真實專案，已抽掉專案與環境的識別資訊。

---

**English:** `ship-check` is a Claude Code skill that runs a **pre-production sustainability audit** — the "will this still run in 3 months as the data grows?" layer that a fast AI-built MVP usually skips. It measures real numbers before diagnosing (no finding without a number, unless the code alone proves the defect), audits 7 growth-over-time dimensions (data growth & retention, computation push-down, memory & cache safety, concurrency, payload size, frontend fetching, honest UI state), and ranks findings by *when the pain lands* rather than abstract severity. It deliberately does **not** hunt line-level correctness bugs — that's what `/code-review` is for, and every audit ends by handing off to it. Drop the folder into `~/.claude/skills/` to install.
