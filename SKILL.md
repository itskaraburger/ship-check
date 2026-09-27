---
name: ship-check
description: Pre-production sustainability & hardening audit for a web service, dashboard, or background job — the "will this still run in 3 months as the data grows?" layer that a quick AI-built MVP usually skips. Use this whenever the user says anything like "上線前幫我體檢", "這個能不能上線", "要掛 cron / 部署了幫我檢查", "幫我加固 / 穩定度體檢", "會不會越跑越慢 / 越跑越肥 / 假死", "撐得住成長嗎", "ship check this", "is this production ready?", "harden this before it runs unattended" — or before deploying anything, scheduling a cron/timer, or adding a new API endpoint, cache, or background job. Also use in reverse (triage mode) when something already live got slow, hangs, or broke with no code change. Measures real numbers first (先量再猜), audits the growth-over-time dimensions, reports findings by severity with evidence, proposes fixes, and hands off to /code-review for line-level correctness bugs.
---

# ship-check

A **pre-production sustainability audit**. AI builds a working MVP fast, but "works on my machine with 100 rows" quietly rots into "假死 with 100k rows three months later." This skill is the gate that catches that *before* it goes live — the layer that asks **"does this get slower, fatter, or leakier as data and time grow?"**

The condensed principles it reasons from live in `references/playbook-principles.md`; the pre-ship tick-lists and troubleshooting steps in `references/checklists.md`. Read those when you actually run an audit.

## What this is / is NOT

- **IS:** the *sustainability / hardening* layer — data growth & retention, computation push-down / N+1, memory & cache safety, concurrency & background-job overlap, payload size, frontend fetch coordination, honest UI state.
- **is NOT:** a line-by-line correctness bug hunt. That is the built-in **`/code-review`**. The two are complementary and must not be conflated. **Every ship-check ends by telling the user to run `/code-review`** — because hardening edits (adding windows, caches, retention, clamps) can themselves introduce regressions. In the pass that motivated this skill, a hardening change introduced a `500 on a nonexistent id` that only `/code-review` caught. That is exactly why the handoff exists.

## How to talk about findings

Assume the user ships fast and is technically capable, but may be newer to infra. Explain findings plainly with numbers, not jargon. When they ask "why is this a problem," answer in 生活化 (everyday) terms. Do the technical work directly — don't over-ask; only pause for privacy, spending, or big-direction calls.

---

## Two modes

Pick the mode from how the request arrives.

### Mode A — Pre-ship audit (proactive)
The thing is *about to* go live / run unattended / gain a new surface. Run the audit below top-to-bottom (or, for a single new endpoint/job, just the matching tick-list in `references/checklists.md`).

### Mode B — Triage (reactive)
Something *already live* is slow, hangs, or "沒改 code 卻壞了." **Do not guess and do not blame the last commit** — the timing is wrong for that theory; look for things that *grew* (data, cache entries, memory). Follow Part B in `references/checklists.md`: classify the symptom → measure outward-in (browser → proxy → backend → DB), starting from a no-DB health-check baseline.

---

## The iron rule: 先量再猜 (measure before guessing)

**No finding and no fix is allowed before you have a number.** Every hypothesis is verified by measured evidence: rows returned, payload KB, latency seconds, RSS bytes, cache entry count, job runtime. State the number in the finding.

Before auditing, gather the baseline (stack-neutral — adapt the commands to the project). For a SQLite / Python / FastAPI project, for example:

- **Table sizes + growth rate** — row count per key table, and rows/day. Which tables are append-only (logs, price snapshots, history)? Do they have retention?
  - e.g. `SELECT COUNT(*) FROM some_history_table;` plus the span of a timestamp column → rows/day.
- **DB file size on disk**, and `output/`-style artefact dirs (count × avg size × per-day = per-year).
- **Payload bytes per endpoint** vs what the page actually shows — `curl -s <url> | wc -c`.
- **Response time**, health-check first: hit a no-DB, no-auth endpoint for a baseline, then the business endpoint. Health-check *also* slow → process layer (memory/CPU/locks). Health-check fast, business slow → that query.
- **RSS over time** — long rise, never drops = leak / unbounded accumulation; saw-tooth = hitting a ceiling and restarting.
- **Cache** — same request twice: second faster? Does the key count grow with user input (unbounded key space)?
- **Background job** — single-round runtime vs its trigger interval. Can a round outlast its interval?

Put the numbers in a short table before you diagnose. Numbers first, opinions second.

---

## Audit dimensions (7)

Walk these seven. Each has a one-sentence test; the reasoning and self-questions are in `references/playbook-principles.md` (chapter numbers in brackets). Only flag a dimension when a **measured number** backs it.

1. **資料成長 [Ch.1]** — Does query cost scale with the *result* size (safe) or the *table* size (dangerous)? Do append-only tables have an explicit retention contract? Will this query still run at 3× the rows?
2. **計算下推 [Ch.2]** — Any "撈全部再用程式挑一小把"? Are filter / sort / top-N / aggregation / dedup pushed into the DB? Any N+1 (a query per item in a loop)?
3. **記憶體 + Cache 四問 [Ch.3–4]** — For every structure that outlives one request, answer all four: **TTL** (活多久) / **Eviction** (誰真的刪它) / **Cap** (最多幾筆) / **Key space bounded?** (key 空間有界嗎). Lazy-TTL that never deletes is a leak. Expensive compute needs in-flight dedup (or 50 concurrent misses recompute the same thing).
4. **並發 + 背景工作 [Ch.5]** — "同一時刻最多幾份在跑?" must have an explicit number, not "unbounded." Timers need overlap protection ("上一輪還沒跑完就跳過"). Background + request paths triggering the same compute must share dedup.
5. **Payload [Ch.6]** — Is the response = "what the UI actually shows," not "everything the DB has"? Lists paginated? Sorting/filtering server-side so the client isn't forced to receive everything?
6. **前端抓取 [Ch.7]** — Shared fetch layer or every component fetching its own copy? Poll frequency matched to how often data *actually* changes? Background tab pauses polling? Timers cleaned on unmount? Permanent UI (sidebar light) must use a lightweight summary endpoint, not fetch whole lists.
7. **誠實狀態 [Ch.8]** — Are Loading / Success / Failure / No-data four *distinct* states? Does the UI ever fake a "normal" default (green light, 0, blank) when it actually has no data or failed?

---

## Severity — tie it to the growth curve

Rank findings by *when the pain lands*, not by abstract badness:

- 🔴 **已經在發生** — the number already crossed the line. (e.g. "the '7-day trend' is silently a *whole-history* average because the query has no time window, and there are already 17 days / 68 keys in the table.")
- 🟠 **觀察期內會踩到** — fine now, breaks within the horizon that matters. (e.g. "the calendar embeds `limit=60` rows = only 20 days visible; the 30-day observation window will drop the earliest dates.")
- 🟡 **慢燒** — unbounded but survives the short term. (e.g. "cron.log isn't rotated — ~4MB/year, slow but capless.")

Each finding = **measured evidence + one-line plain-language "why it bites" + concrete fix.**

---

## Output format

1. **量到的數字** — the baseline table (rows, growth/day, payload, RSS, etc.). Always lead with this.
2. **發現 (ranked 🔴→🟠→🟡)** — each with evidence + everyday-language impact + fix. Cap the list; if you truncate, say so.
3. **✅ 檢查過、沒問題的** — list what you verified is safe (bounded queries, existing retention, stable RSS). Don't only report bad news — the clean bill is half the value and stops needless "fixes."
4. **修法建議** — group into one focused PR when possible; note anything that's VM-only / infra (logrotate, systemd) vs in-repo code.
5. **接力 (always, verbatim intent):** end with — *"永續層體檢完成。逐行 correctness bug 請接著跑 `/code-review`，因為這次的加固修改本身可能引入 regression。"*

Do **not** start editing during an audit unless the user says so — Mode A's job is to *surface and rank*, then let the user green-light fixes (typical flow: 開分支 → PR → lint → 合併). In Mode B, measuring may be enough to point at the fix; still confirm before changing a live system.

---

## When to actually use this (dev-flow timing)

- **MVP 原型階段：不要用。** While proving the idea works, scale doesn't matter — premature hardening is wasted effort. Make it work first.
- **時機 1 — 第一次真正部署 / 要無人值守前。** The moment it goes from "runs when I press play" to "runs on cron / stays up as a server." → full Mode A audit.
- **時機 2 — 新增 endpoint / cache / 背景工作前。** Run only the matching tick-list in `references/checklists.md`. Cheap, targeted — no need for a full sweep.
- **時機 3 — 里程碑升級、賭注變大前。** e.g. opening to more users, or switching from a test run to the real thing. Use it as a gate.
- **時機 4 — 已上線的東西變慢 / 假死 / 沒改 code 卻壞了。** → Mode B triage.
- **⭐ 無人值守系統的收尾條件 — 只要這次改動碰到排程 / 執行 / 背景工作的程式，收尾一定順帶跑 `references/checklists.md` 的清單 D。** 死機告警（dead man's switch）是無人值守的最後一道防線，卻可能被無關的改動悄悄弄斷，或更糟——弄成「報假平安」（例外被吞、卻還報成功）。這條防線斷了不會有任何錯誤訊息，只能靠每次改完主動重驗。
- **每次體檢後 → 接 `/code-review`。** Sustainability + correctness together = a complete gate.

---

## Why each piece matters

- **先量再猜**: without a number, a "fix" is a guess that adds complexity for no proven gain. The number is what separates a real 🔴 from a hypothetical one.
- **The four cache questions**: skipping any one is how a "harmless little cache" becomes the memory leak that causes 假死. All four or don't ship the cache.
- **Retention on append-only tables**: these grow fastest and are the classic "sudden crash, no code change" — the code didn't change, the row count crossed a threshold.
- **Severity tied to the growth curve**: it tells the user *what to fix now vs. later*, so a 🟡 doesn't get treated like a 🔴 and vice-versa.
- **The `/code-review` handoff**: hardening edits touch query shapes, caches, and guards — prime territory for off-by-one and null-path regressions. The audit that *introduces* a bug and doesn't catch it is worse than none.
- **Listing the clean bill**: prevents the user from "fixing" things that were already safe, which is its own way of adding fragility.
