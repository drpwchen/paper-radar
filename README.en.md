Language： [繁體中文](README.md) ｜ **English**

# paper-radar

🔭 **Discovery** paper-radar ｜ ⬇️ **Download** [paper-fetch](https://github.com/drpwchen/paper-fetch) ｜ 📖 **Reading** [paper-review-and-digest](https://github.com/drpwchen/paper-review-and-digest) — papers the radar picks are fetched in full text by paper-fetch's route ladder, then handed to `/paper-review` (appraisal) and `/paper-digest` (absorption); together the three repos form the full paper-reading pipeline.

> A **personal literature-tracking & learning radar**. It pulls dozens of journal RSS / PubMed-search feeds, scores and ranks them against *your* research interests, and pushes them to a **private, just-for-you web page** to swipe and triage; the papers you pick then flow back into your own note system. I originally built it to keep up with new papers while studying for the **PM&R (Physical Medicine & Rehabilitation)** board exam — this is the cleaned-up, self-hostable open-source version.

🔒 My own instance runs behind Cloudflare Access (private, personal-interest data), so there is no public demo. Screenshots below.

---

## What it is · the problem it solves

Dozens of journals update daily, plus a few authors and topics I want to follow. The usual approach — dumping all those RSS feeds into a note app — turns into an unreadable pile fast. I wanted a radar that **filters and ranks first**:

- Pull **dozens of sources** at once (journal RSS + author/topic PubMed searches), de-duplicated
- Score & rank by **my interest model** — high-signal new papers float up, noise sinks
- Push everything to a **private web page** (works on mobile); mark each one ✅ seen / 🔬 want a quality appraisal / 📚 want a content digest / 👍😐👎
- Auto-tag **full-text availability** per paper (open access? institutional subscription? fetch yourself?)
- Whatever I mark gets pulled into my note system with one command
- My votes **train the interest model back** — the radar gets sharper with use

> A solo side project. The **radar itself is reusable and self-hostable**; the "flow back into notes" stage is wired to my personal Obsidian + LLM toolchain — treat it as an example and wire your own downstream.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Host (24/7 cron — I use a small Oracle Cloud Free Tier box)  │
│                                                               │
│  fetch_and_score.py                                           │
│    dozens of feeds (rss + pubmed_search) → SQLite dedup       │
│    → interest_model scoring → papers.json                     │
│  enrich.py                                                     │
│    per-DOI → Unpaywall (open access)                          │
│            + institutional SFX / link resolver (optional)     │
│  notify_digest.py  daily high-score new papers → ntfy push    │
│             │ wrangler pages deploy                            │
└─────────────┼─────────────────────────────────────────────────┘
              ▼
┌─────────────────────────────────────────────────────────────┐
│  Cloudflare Pages (locked behind Cloudflare Access — only you)│
│    site/        private page: topic toggles / filters /       │
│                 action buttons / full-text badges             │
│    _worker.js   POST /api/action → D1                         │
│                 POST /api/upload → R2 (external PDFs)          │
│                 GET  /api/state  → downstream pulls unsynced   │
│    D1 actions table = every action you take (source of truth, │
│                       synced across devices)                  │
└─────────────┬─────────────────────────────────────────────────┘
              ▼
        Your downstream (example: flow back into notes)
        read unsynced D1 actions → fetch full text →
        route by 🔬/📚 → write into notes; votes → train model
```

## Core features

### 1 · Fetch + interest scoring (`fetch_and_score.py`)
- Two feed types: `rss` (feedparser) and `pubmed_search` (NCBI E-utilities query — bypasses broken journal RSS, and enables author/topic tracking).
- Everything lands in SQLite, de-duped by DOI/title.
- Each paper is scored via `interest_model.json` (keyword/MeSH weights); the front-end defaults to highest-score-first.
- Hard-won notes live in the comments: some publisher CDNs block automated requests (LWW), some `search.rss` endpoints serve broken XML (some Springer journals) → route those through PubMed `[ta]` instead.

### 2 · Three full-text tiers (`enrich.py`)
For every paper with a DOI, tag how hard it is to obtain:

| Tier | Badge | How |
|---|---|---|
| Open access | 🟢 OA | [Unpaywall](https://unpaywall.org/) lookup (automatic, free) |
| Institutional | 🏥 subscription | **Optional**: via your institution's SFX / link resolver — does *this* paper resolve to full text right now |
| Fetch yourself | 🔒 | deep link provided; you fetch / upload |

> ⚠️ **The institutional tier is OFF by default.** It uses your institution's link resolver (standard library tech, e.g. ExLibris SFX) only to determine *whether a paper is currently obtainable* and to produce a deep link — **accessing and downloading still must comply with your institution's license and each publisher's Terms of Service.** No institutional access? Leave it off and use the OA tier only.

### 3 · Private web action layer (`site/` + Cloudflare D1/R2)
- The whole site sits behind **Cloudflare Access** — only you get in (email OTP / IdP). No SEO, no public RSS, no byline.
- Per-paper actions: ✅ seen, 🔬 quality appraisal, 📚 content digest, 👍😐👎 vote, 📎 upload full text.
- Actions are written to **Cloudflare D1** (SQLite), so **what you checked shows up across phones/browsers** (D1 is the source of truth; localStorage is just a cache).
- Upload **external PDFs** (not from a feed) into R2 — the worker enforces a monthly quota and per-file size cap so you can't blow past free limits.
- Mobile-friendly: collapsible settings, persistent search, seen items hidden by default.

### 4 · Interest training loop (`train_interest.py`)
- Your 👍👎 votes → aggregate the topic tags each paper matched → nudge `interest_model.json` weights.
- Designed as a **pure, idempotent function** (`effective = clamp(base + delta, 1, 5)`); dry-run by default, `--apply` writes with a backup.

> As of v2, ranking is driven by **PRPM v2**'s `train_model.py` (below); `train_interest.py` is kept as a local manual inspector (see [CHANGELOG](CHANGELOG.md)).

### 5 · Push notifications (`notify_digest.py` / `notify_pending.py`)
- A daily digest of "new enough + high enough score + not yet seen" papers to [ntfy](https://ntfy.sh/) (de-duped, never re-pushed).
- A second one nags you that "N papers are still marked for processing on the page."

### 6 · Downstream: flow back into your notes (conceptual)
My own downstream is an on-demand process that reads unsynced D1 actions → shared prep (DOI verify, add to reference manager, fetch full text) → routes by badge: **🔬 quality** goes through a credibility appraisal, **📚 content** goes through a fast content digest → writes into my Obsidian notes; votes go to a training log.

> That stage is **tightly coupled to my personal Obsidian + LLM toolchain and is not in this repo.** `_worker.js`'s `GET /api/state?unsynced=1` is the hook for any downstream — wire it to whatever you want (store to Notion, hand to an LLM, email yourself…). Think of it as: the radar already filtered, ranked, and tagged full-text for you — do whatever you like with the result.

> 🔗 My own "🔬 quality appraisal / 📚 content digest" downstream is open-sourced as
> [**paper-review-and-digest**](https://github.com/drpwchen/paper-review-and-digest) (`/paper-review` + `/paper-digest`,
> with deterministic GRADE recompute and a CrossRef citation gate). paper-radar is the *discovery* end,
> paper-review-and-digest is the *reading* end — together they form the full paper-reading pipeline.

## PRPM v2 · Personal Research Preference Model

v1's ranking was a single hand-written keyword weight table, vote-only, with a manual training step — essentially a sorter. v2 rewrites the whole recommendation engine into a **self-learning personal preference model**: every ✅/🔬/📚/👍😐👎/seen event sharpens its answers to "what do you like (topic × methodology × setting), what do you dislike, which features actually drive decisions, and how are your preferences drifting over time." Full math and constants live in [`docs/DESIGN-PRPM.md`](docs/DESIGN-PRPM.md).

Layered architecture:

```
Event store    D1 actions (current state) + action_log (append-only history)
Feature store  SQLite papers.{tags, facets, embedding}
Model          model_state.json — decayed Beta pseudo-counts per feature + profile vector
Serving        rank.py — Thompson-sampled score, MMR diversity, explore slots, why-breakdown
Introspection  site/profile.json + preference dashboard profile.html
```

- **Nightly self-training (stateless)**: `train_model.py` recomputes from the full D1 event history every night → idempotent, no drift, fully rebuildable. All signals count: engagement (🔬/📚/📎, +2), 👍 (+1), 😐 (weak negative −0.3), 👎 (−1.5), seen-only (−0.1), with a 90-day half-life so preferences can drift.
- **Thompson-sampling ranking + damping**: one Beta θ drawn per feature per day as the ranking score — uncertain features naturally surface (built-in exploration); the displayed score uses the stable posterior mean.
- **MMR diversity**: greedy re-rank of the top 60 to break same-topic streaks.
- **🧭 Explore slots**: fixed positions (6/12/18/24/30/36) seeded with cold features / semantically adjacent / serendipity picks; feedback carries exploration context so explore hit-rate is measurable.
- **LLM facet extraction** (`extract_facets.py`): classifies each paper by study design / setting / population / sample size (enum whitelist, hallucinations dropped), adding the axes hand-written tags can't. Skips automatically if `GROQ_API_KEY` is unset.
- **Semantic embedding profile** (`embed_papers.py`): bge-small (384-d) builds your semantic preference vector, capturing the long tail no tag/facet encodes.
- **Why-breakdown UI**: click a score to expand each feature's contribution and debug your own model; the preference dashboard (📈 profile.html) shows top / avoided features, 30-day drift, and explore effectiveness.

Every stage degrades gracefully: if any PRPM stage fails, ranking falls back to the previous day's keyword order without blocking the pipeline. See [`CHANGELOG.md`](CHANGELOG.md) for the full history.

## Stack & dependencies

| Layer | Tech |
|---|---|
| Fetch / score / enrich | Python 3.11+ · feedparser · requests · pyyaml · SQLite |
| Full text | Unpaywall API (free, needs an email) · optional institutional SFX/link resolver |
| Web | Static HTML/CSS/JS (no framework) · Cloudflare Pages |
| Action backend | Cloudflare Workers (Pages advanced mode) · D1 · R2 |
| Push | ntfy |

### External services I use (swap in equivalents)
This project is built on a few services that **mostly have free tiers**. Use the same, or swap in equivalents:

- **A 24/7 host for cron** — I use an [Oracle Cloud Free Tier](https://www.oracle.com/cloud/free/) Always-Free box. **Alternatives**: any VPS, a home Raspberry Pi/NAS, or even a scheduled GitHub Action — anything that can run Python on a timer and invoke `wrangler`.
- **Static site + edge DB/storage** — I use **Cloudflare Pages + D1 + R2 + Access** (all within free tiers). **Alternatives**: Vercel/Netlify + any SQLite/Postgres, or a self-hosted reverse proxy + basic auth. The action layer just needs a backend that can store rows and an auth layer that locks the site down.
- **Push** — I use **ntfy** (self-hosted or public ntfy.sh). **Alternatives**: Telegram bot, Discord webhook, email.

> Swapping mostly means changing endpoints in `enrich.py`/`notify_*.py` and the deploy scripts (`run.sh`/`deploy.sh`). The core fetch + scoring (`fetch_and_score.py`) needs no cloud service and runs entirely locally.

## Screenshots

> The site is behind Cloudflare Access; these are captures of the real UI (data is public literature, nothing sensitive).

![demo](docs/screenshots/demo.gif)

| Paper list (full-text badges 🟢/🏥 + action buttons ✅🔬📚👍) | Topic toggles / filter row |
|---|---|
| ![list](docs/screenshots/list.png) | ![filter](docs/screenshots/filter.png) |

## Quick start (run fetch + scoring locally)

Requires Python 3.11+.

```bash
git clone https://github.com/<you>/paper-radar.git
cd paper-radar
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt

cp config.example.yaml config.yaml          # edit your feeds / email / site domain
cp interest_model.example.json interest_model.json
cp env.example .env                          # fill CF / ntfy (only needed to deploy)

# Local only: fetch + score (no cloud service required)
python fetch_and_score.py                    # all feeds → papers.json
python fetch_and_score.py --only eswt,pain --limit 8   # just a few feeds
python enrich.py --limit 20                  # enrich first 20 (OA / institutional)
```

Open `site/index.html` (or serve it with `python -m http.server`) to see the front-end render `papers.json`. `site/papers.sample.json` is a bundled synthetic sample so the UI renders right after cloning.

## Deploy to Cloudflare (private site)

Full steps in [`docs/DEPLOY.md`](docs/DEPLOY.md); summary:

1. **D1**: `wrangler d1 create paper-radar-db` → `wrangler d1 execute paper-radar-db --remote --file=schema.sql`; put the returned database_id into `wrangler.toml` (copy from `wrangler.toml.example`).
2. **R2** (optional, only for uploads): `wrangler r2 bucket create paper-radar-pdfs`.
3. **Pages**: `wrangler pages deploy site --project-name=paper-radar`, bind your custom domain.
4. **Cloudflare Access**: in Zero Trust, create a self-hosted application covering your domain, policy allowing only your own email. **This step is what makes the site private — set it up before putting any data online.**
5. **Host cron**: scp the project to your host, create a venv, set `.env`, cron `run.sh`.

## ⚠️ Security & permissions before you self-host

Even a just-for-me site must be defended like a public service once it's on the internet:

- **Lock before you expose**: set up Cloudflare Access (or equivalent auth) *at the same time* you attach the public domain. Don't "go live now, lock it later" — in between it's world-readable.
- **Least-privilege API tokens**: the host-cron Cloudflare token only needs `Pages:Edit` + `D1:Read` (it doesn't even need D1 write). For schema changes (ALTER), use a separate `D1:Edit` token from a trusted machine — don't keep write access resident on the host.
- **Never commit secrets**: `.env`, `*.dpapi`, `wrangler.toml` (with the real database_id), generated `*.db` / `papers.json` are all in `.gitignore`. Run `git status` once more before committing.
- **Keep the worker's quota caps** (monthly upload count, per-file size) to avoid abuse or accidentally blowing the free tier.
- **ntfy**: use a private, token-protected topic, not a guessable public name.
- **Respect full-text access**: OA tier is fair game; institutional access and downloads must follow your institution's license and each publisher's ToS — this tool only determines *availability* and links out, it never bypasses a paywall.

## License

[MIT](LICENSE). Fork it and retarget it to your own field — not just medicine, any literature source with RSS/an API works.

---

*Built by 陳柏威 (Po-Wei Chen) — a PM&R physician. It started as "stop missing the good papers." If it helps you keep up with the literature, a star ⭐ is welcome.*

## 🧋 Support

If this tool saves you time, you can buy me a boba — it keeps the servers running 🧡

[![Buy me a drink](https://img.shields.io/badge/🧋_Buy_me_a_drink-8a6d4e?style=for-the-badge)](https://drpwchen.com/en/support/)
