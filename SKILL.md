---
name: ship-check
description: Pre-production sustainability & hardening audit for a web service, dashboard, or background job — the "will this still run in 3 months as the data grows?" layer that a quick AI-built MVP usually skips. Use this whenever the user says anything like "上線前幫我體檢", "這個能不能上線", "要掛 cron / 部署了幫我檢查", "幫我加固 / 穩定度體檢", "會不會越跑越慢 / 越跑越肥 / 假死", "撐得住成長嗎", "ship check this", "is this production ready?", "harden this before it runs unattended" — or before deploying anything, scheduling a cron/timer, or adding a new API endpoint, cache, or background job. Also use in reverse (triage mode) when something already live got slow, hangs, or broke with no code change. Measures real numbers first (先量再猜), audits the growth-over-time dimensions, reports findings by severity with evidence, proposes fixes, and hands off to /code-review for line-level correctness bugs (and /security-review when the change touches who can reach data).
---

# ship-check

A **pre-production sustainability audit**. AI builds a working MVP fast, but "works on my machine with 100 rows" quietly rots into "假死 with 100k rows three months later." This skill is the gate that catches that *before* it goes live — the layer that asks **"does this get slower, fatter, or leakier as data and time grow?"**

The condensed principles it reasons from live in `references/playbook-principles.md`; the pre-ship tick-lists and troubleshooting steps in `references/checklists.md`. Read those when you actually run an audit.

## What this is / is NOT

- **IS:** the *sustainability / hardening* layer — data growth & retention, computation push-down / N+1, memory & cache safety, concurrency & background-job overlap, payload size, frontend fetch coordination, honest UI state.
- **is NOT:** a line-by-line correctness bug hunt. That is the built-in **`/code-review`**. The two are complementary and must not be conflated. **Every ship-check ends by telling the user to run `/code-review`** — because hardening edits (adding windows, caches, retention, clamps) can themselves introduce regressions. In the pass that motivated this skill, a hardening change introduced a `500 on a nonexistent id` that only `/code-review` caught. That is exactly why the handoff exists.
- **is also NOT:** a security audit (auth, injection, hard-coded secrets). That is the built-in **`/security-review`**. ship-check asks only the one exposure question its own measurements raise — *who can pull this whole payload?* — and hands the rest off.

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

**No finding and no fix before you have a number — with exactly the two exceptions listed below.** Every hypothesis is verified by measured evidence: rows returned, payload KB, latency seconds, RSS bytes, cache entry count, job runtime. State the number in the finding.

Two things that rule does **not** mean:

- **Structural defects the code alone proves** — a cache key that changes on every request, a table with no delete path, a timer with no overlap guard — are reportable without a measurement. Label them **「機制確定・代價未量」** and state the one measurement that would size them. Rank them by when the mechanism bites.
- **When you cannot measure** (read-only access, no credentials, nothing running): mark the item **「需量測」** and say exactly what you would measure. Never fill the gap with an invented or assumed number, and never quietly skip the item.

Before auditing, gather the baseline (stack-neutral — adapt the commands to the project). For a SQLite / Python / FastAPI project, for example:

- **Table sizes + growth rate** — row count per key table, and rows/day. Which tables are append-only (logs, price snapshots, history)? Do they have retention?
  - e.g. `SELECT COUNT(*) FROM some_history_table;` plus the span of a timestamp column → rows/day.
- **DB file size on disk**, and `output/`-style artefact dirs (count × avg size × per-day = per-year).
- **Payload bytes per endpoint** vs what the page actually shows — `curl -s <url> | wc -c`.
- **Response time**, health-check first: hit a no-DB, no-auth endpoint for a baseline, then the business endpoint. Health-check *also* slow → process layer (memory/CPU/locks). Health-check fast, business slow → that query.
- **RSS over time** — long rise, never drops = leak / unbounded accumulation; saw-tooth = hitting a ceiling and restarting.
- **Cache** — same request twice: second faster? (A hint, not proof — confirm with hit logs or the count of underlying queries.) Does the key count grow with user input (unbounded key space)?
- **Background job** — single-round runtime vs its trigger interval. Can a round outlast its interval?
- **Metered usage** — calls to external APIs per round × rounds/day vs the provider's quota, and the busiest minute vs its rate limit (requests, schedules, retries and overlapping runs all count). On a database billed per row *read*, count rows read per request, not rows returned.
- **Safety net** — does the project have automated tests or CI at all? Count the test files and note what they cover. If there are none, say so in the baseline: every fix this audit proposes will land unguarded.

Put the numbers in a short table before you diagnose. Numbers first, opinions second.

---

## Audit dimensions (7)

Walk these seven. Each has a one-sentence test; the reasoning and self-questions are in `references/playbook-principles.md` (chapter numbers in brackets). Only flag a dimension when a **measured number** backs it, or the code alone proves the mechanism (see the iron rule).

1. **資料成長 [Ch.1]** — Does query cost scale with the *result* size (safe) or the *table* size (dangerous)? Do append-only tables have an explicit retention contract? Do upsert-only state tables have an exit path for rows that no longer apply? Do the limits agree with each other — a query window must not exceed the retention of the data it reads. Will this query still run at 3× the rows?
2. **計算下推 [Ch.2]** — Any "撈全部再用程式挑一小把"? Are filter / sort / top-N / aggregation / dedup pushed into the DB? Any N+1 (a query per item in a loop)? Push-down comes *before* caching — a cache laid over a full-table pull hides the cost instead of removing it.
3. **記憶體 + Cache 四問 [Ch.3–4]** — For every structure that outlives one request, answer all four: **TTL** (活多久) / **Eviction** (誰真的刪它) / **Cap** (最多幾筆) / **Key space bounded?** (key 空間有界嗎). Lazy-TTL that never deletes is a leak. A key that differs between requests that should have shared a result (it embeds the current time or a random id) almost never hits — a cache in name only. Expensive compute needs in-flight dedup (or 50 concurrent misses recompute the same thing).
4. **並發 + 背景工作 [Ch.5]** — "同一時刻最多幾份在跑?" must have an explicit number, not "unbounded." Timers need overlap protection ("上一輪還沒跑完就跳過"). An in-process flag does not cover a second instance (replicas, rolling deploys, manual reruns) — that needs a cross-process lock. Reruns must be idempotent. Every external call needs a timeout that actually aborts the connection. Background + request paths triggering the same compute must share dedup. Metered external calls per round × rounds/day must fit the quota with headroom — overlap and retries multiply that number.
5. **Payload [Ch.6]** — Is the response = "what the UI actually shows," not "everything the DB has"? Lists paginated? Sorting/filtering server-side so the client isn't forced to receive everything? Is the page-size cap enforced by the server (a client-supplied `limit` is a request, not a rule)? Who can pull this whole payload — an endpoint that returns everything is also a bulk-export door. Check every consumer of the endpoint, not just the main page — a badge that fetches 500 rows to show one number is the same defect.
6. **前端抓取 [Ch.7]** — Shared fetch layer or every component fetching its own copy? Poll frequency matched to how often data *actually* changes? Background tab pauses polling? Timers cleaned on unmount? Permanent UI (sidebar light) must use a lightweight summary endpoint, not fetch whole lists.
7. **誠實狀態 [Ch.8]** — Are Loading / Success / Failure / No-data four *distinct* states? Does the UI ever fake a "normal" default (green light, 0, blank) when it actually has no data or failed? When a cap truncates a result, does the response say so (or refuse the request) instead of passing a slice off as the whole? A `truncated` / `stale` flag set by the backend only counts if every place that displays the data reads it.

---

## Severity — tie it to the growth curve

Rank findings by *when the pain lands*, not by abstract badness. State the observation horizon first — three months unless the user names another:

- 🔴 **已經在發生** — the number already crossed the line, or the code alone proves the mechanism fires today (then it carries the 「機制確定・代價未量」 label). (e.g. "the '7-day trend' is silently a *whole-history* average because the query has no time window, and there are already 17 days / 68 keys in the table.")
- 🟠 **觀察期內會踩到** — fine now, breaks within the horizon that matters. (e.g. "the calendar embeds `limit=60` rows = only 20 days visible; the 30-day observation window will drop the earliest dates.")
- 🟡 **慢燒** — unbounded but survives the short term. (e.g. "cron.log isn't rotated — ~4MB/year, slow but capless.")

A defect that only fires when something outside goes wrong (an upstream fails, retries pile up, two instances run at once) and whose timing you cannot estimate goes under 🟠, labelled **「條件觸發・時間未定」**, with the trigger and the consequence spelled out. Knowing that a defect exists and knowing when it will hurt are two separate judgements — make both.

Each finding = **evidence + one-line plain-language "why it bites" + concrete fix + how to verify the fix.** Evidence is a measured number, or — for a 「機制確定・代價未量」 finding — the code location and the reasoning that proves it, plus what remains to be measured. That label stays next to the colour until it is measured.

---

## Output format

1. **量到的數字** — the baseline table (rows, growth/day, payload, RSS, etc.). Always lead with this. Rows you could not measure stay in the table, marked 「需量測」 with what you would measure.
2. **發現 (ranked 🔴→🟠→🟡)** — each with evidence + everyday-language impact + fix + how to verify the fix. This ranked summary may be capped (say so when you cap it); the per-item checklist results are never cut — every failing location stays listed, and one root cause can be explained once for all of them.
3. **✅ 檢查過、沒問題的** — list what you verified is safe (bounded queries, existing retention, stable RSS). Don't only report bad news — the clean bill is half the value and stops needless "fixes."
4. **修法建議** — group into one focused PR when possible; note anything that's VM-only / infra (logrotate, systemd) vs in-repo code. Each fix comes with a re-runnable check and its pass condition, so a later change can't quietly undo it: a number where there is one (rows returned, payload bytes, cache entry cap), otherwise an observable behaviour (the job ends when the upstream hangs; running the same batch twice leaves one result; the truncation flag reaches every place that shows the data). During the audit you only *describe* the check — write or run it after the user green-lights the fix. A correctness test on a small fixture passes at 100 rows and at 100k rows alike; it does not cover this.
5. **接力 (always, verbatim intent):** end with — *"永續層體檢完成。照建議修完之後，請接著跑 `/code-review` 抓逐行 correctness bug——加固修改本身可能引入 regression。"* If the audit looked at or raised anything about who can reach data (a new or widened endpoint, auth, exposed fields, a secret with a built-in default), add — *"這次體檢碰到資料對外範圍的問題，請另外跑 `/security-review`。"*

Do **not** start editing during an audit unless the user says so — Mode A's job is to *surface and rank*, then let the user green-light fixes (typical flow: 開分支 → PR → lint → 合併). In Mode B, measuring may be enough to point at the fix; still confirm before changing a live system.

---

## When to actually use this (dev-flow timing)

- **MVP 原型階段：不要用。** While proving the idea works, scale doesn't matter — premature hardening is wasted effort. Make it work first.
- **時機 1 — 第一次真正部署 / 要無人值守前。** The moment it goes from "runs when I press play" to "runs on cron / stays up as a server." → full Mode A audit.
- **時機 2 — 新增 endpoint / cache / 背景工作前。** Run only the matching tick-list in `references/checklists.md`. Cheap, targeted — no need for a full sweep. Report it in the same output format, scoped to that list; baseline rows you could not measure stay in the table marked 「需量測」.
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
