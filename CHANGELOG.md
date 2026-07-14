語言 Language： **繁體中文** ｜ [English](#english)

# Changelog

本專案的重大變更紀錄。版本語意：`vMAJOR.MINOR`。

## v2.0.1 (2026-07-14) — 文件：下游 repo 更名

- 閱讀端 repo `claude-paper-tools` 更名為 **`paper-review-and-digest`**（讓名字直接說明它做什麼）。README 中英版的互連連結同步更新；舊 GitHub 網址仍會自動轉址，既有連結不會斷。

## v2.0 (2026-07-07) — PRPM：個人研究偏好模型（Personal Research Preference Model）

把 v1 的「keyword 排序器」重寫成一個會自我學習的**個人研究偏好模型**。設計文件：[`docs/DESIGN-PRPM.md`](docs/DESIGN-PRPM.md)。

- **每晚自我訓練迴圈**（`train_model.py`，`sync_actions.py`）：無狀態設計，每晚從 D1 全量事件重算 decayed Beta 後驗，idempotent、無漂移、可完整重建。
- **所有訊號都學**：投入（🔬 品質／📚 內容／📎 上傳，+2）、👍（+1）、😐 中立（弱負 −0.3）、👎（−1.5）、僅看過（−0.1）、曝光未互動（impression-only，−0.05）；套 90 天半衰期讓偏好隨時間漂移。
- **Thompson 抽樣排序 ＋ 變異阻尼**：每特徵每天抽一次 θ，不確定的特徵自然輪到高位＝內建探索；展示分數用穩定的 posterior mean。
- **MMR 多樣性重排**：打散同主題連發。
- **🧭 探索槽**：固定位置注入冷門特徵／語意鄰接／隨機驚喜，回饋帶 exploration context，可與 exploit 回饋區分。
- **LLM facet 抽取**（`extract_facets.py`）：以 Groq 免費模型把每篇分類到 study design／setting／population／methods（enum 白名單，丟幻覺）。
- **語意 embedding profile**（`embed_papers.py`）：bge-small（384 維）建立語意偏好向量。
- **why 分解 UI**：點分數展開每個特徵的貢獻，可 debug 自己的模型。
- **偏好儀表板**（`site/profile.html`）：最偏好／想避開特徵、近 30 天漂移、探索成效。
- **activity-based 等待同步 banner**：點擊即時增減、可切換只看待同步。
- **ETag 快取修正**：`papers.json` 改走 304，不再每次強制重載整包。
- **關鍵字比對 word-boundary 修正**：短縮寫（SCI/FMS/OA…）用字界比對，避免子字串誤命中污染評分與訓練資料。
- **D1 `action_log` 事件歷史**：worker 於 `/api/action`、`/api/upload` 雙寫 append-only 事件（含 ctx），未建表時容錯不 500。

## v1.0 (2026-07-01) — 首次公開

- 抓取 ＋ 興趣評分（`fetch_and_score.py`）、全文三層加值（`enrich.py`）、私密網頁動作層（`site/` ＋ Cloudflare D1/R2）、興趣訓練迴圈（`train_interest.py`）、ntfy 推播。

---

<a id="english"></a>
語言 Language： [繁體中文](#changelog) ｜ **English**

# Changelog

Notable changes. Versioning: `vMAJOR.MINOR`.

## v2.0.1 (2026-07-14) — Docs: downstream repo renamed

- The reading-end repo `claude-paper-tools` was renamed to **`paper-review-and-digest`** (the name now says what it does). Cross-links in both README variants updated; the old GitHub URL still redirects, so existing links keep working.

## v2.0 (2026-07-07) — PRPM: Personal Research Preference Model

Rewrote v1's keyword sorter into a self-learning **Personal Research Preference Model**. Design doc: [`docs/DESIGN-PRPM.md`](docs/DESIGN-PRPM.md).

- **Nightly self-training loop** (`train_model.py`, `sync_actions.py`): stateless — recomputes decayed Beta posteriors from the full D1 history each night; idempotent, drift-free, fully rebuildable.
- **Learns from all signals**: engagement (🔬 quality / 📚 content / 📎 upload, +2), 👍 (+1), 😐 neutral (weak negative −0.3), 👎 (−1.5), seen-only (−0.1), impression-only (−0.05); 90-day half-life so preferences drift over time.
- **Thompson-sampling ranking + variance damping**: one θ drawn per feature per day so uncertain features surface (built-in exploration); the displayed score is the stable posterior mean.
- **MMR diversity re-rank**: breaks same-topic streaks.
- **🧭 Explore slots**: fixed positions seeded with cold features / semantically adjacent / serendipity picks, with context-tagged feedback distinguishable from exploit feedback.
- **LLM facet extraction** (`extract_facets.py`): a free Groq model classifies each paper by study design / setting / population / methods (enum whitelist, hallucinations dropped).
- **Embedding profile** (`embed_papers.py`): bge-small (384-d) semantic preference vector.
- **Why-breakdown UI**: click a score to expand each feature's contribution and debug your own model.
- **Preference dashboard** (`site/profile.html`): top / avoided features, 30-day drift, explore effectiveness.
- **Activity-based sync banner**: live counting, toggles to show only pending items.
- **ETag caching fix**: `papers.json` now uses 304 instead of a forced full reload every visit.
- **Word-boundary keyword fix**: short acronyms (SCI/FMS/OA…) match on word boundaries, avoiding substring false-positives polluting scoring and training data.
- **D1 `action_log` event history**: the worker dual-writes append-only events (with ctx) on `/api/action` and `/api/upload`, tolerating a missing table without a 500.

## v1.0 (2026-07-01) — Initial public release

- Fetch + interest scoring (`fetch_and_score.py`), three-tier full-text enrichment (`enrich.py`), private web action layer (`site/` + Cloudflare D1/R2), interest training loop (`train_interest.py`), ntfy push.
