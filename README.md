# Note for Reviewer

My complete trading infrastructure, including both the cross platform semantic arbitrage system described below and the temperature trading system mentioned in my cover letter is maintained in a single private repo.

Because the temperature strategy contained in the repo is making a solid amount of money, I cannot make it public. Below is a detailed architectural overview of the cross-platform semantic arbitrage system.

# Cross-Platform Prediction Market Arbitrage

Automated system for finding and exploiting price differences between prediction market platforms (Polymarket, Kalshi, Opinion). When two platforms price the same event differently, buying one side on each platform locks in a guaranteed profit regardless of outcome.

**Example:** PM prices "Will X happen?" at 60c YES. Kalshi prices the same event at 35c YES (= 65c NO). Buy YES on PM (60c) + NO on Kalshi (65c) = $1.25 total cost. One side always pays $1.00, so you lose 25c. But if PM prices it at 40c YES and Kalshi at 50c YES (= 50c NO), you pay 40c + 50c = 90c for a guaranteed $1.00 payout = 10c profit per contract.

## Pipeline Overview

There are 3 scripts you actually run, in this order:

```
SCAN  ──>  MONITOR  ──>  POSITION BUILDER
match       find arbs     execute trades
markets     track prices  manage portfolio
```

---

## Scripts

### 1. `scan_category.py` — Match Markets Across Platforms

This is the foundation. It finds which markets on Polymarket correspond to which markets on Kalshi (or Opinion).

**What it does:**
1. Fetches all open PM markets via the CLOB API (~30k markets, cached for 6 hours)
2. Fetches all Kalshi markets for a category via the series API (e.g., ~2800 politics series)
3. Filters both sides by category (politics, crypto, sports, finance, tech, culture, climate)
4. Generates text embeddings for every market title using Gemini (`gemini-embedding-001`, 768 dimensions)
5. Computes cosine similarity between all PM and Kalshi embeddings to find candidate matches
6. Sends top candidates to Gemini AI for validation ("Are these the same bet?")
7. Caches results so reruns only process new/changed markets

**How to run:**
```bash
python scripts/scan_category.py --category politics           # One category
python scripts/scan_category.py --category all                # All 7 categories
python scripts/scan_category.py --category politics --target opinion  # Match against Opinion instead
python scripts/scan_category.py --category sports --dry-run   # Just count markets, don't match
python scripts/scan_category.py --category all --refresh-clob # Force re-fetch PM markets
```

**Output:** Writes matches to `data/mapper_cache.json` (or `data/pm_opinion_cache.json` for Opinion). Also caches embeddings as `.npz` files per category so subsequent runs are fast.

**Typical run:** ~5 min for politics (embeds are cached), ~20 min for `--category all` on first run.

---

### 2. `monitor_prices.py` — Find and Track Arb Opportunities

Takes the matched market pairs from step 1 and continuously checks if any have profitable price differences.

**What it does:**
1. Loads all matched pairs from the mapper cache
2. Every 3.5 minutes, fetches live ask prices from both PM and Kalshi
3. Computes the arb edge (profit per contract after fees) and annualized return
4. Filters by annualized return (default >15%)
5. For new opportunities, runs AI resolution screening — checks if the two markets actually resolve the same way (e.g., different deadlines, different resolution sources = FAIL)
6. Caches resolution screening results so the same pair isn't re-screened
7. Beeps when new high-value arbs appear
8. Serves an auto-refreshing HTML dashboard at `localhost:8080`

**How to run:**
```bash
python scripts/monitor_prices.py                  # Default: >15% annualized filter
python scripts/monitor_prices.py --min-ann 30     # Only show >30% annualized
python scripts/monitor_prices.py --min-edge 5     # Also require >5c raw edge
```

**Output:** Live terminal output + HTML dashboard. Writes results to `data/arb_monitor.html`.

---

### 3. `position_builder.py` — Execute and Manage Arb Positions

Once you've identified good arbs and manually opened initial positions on both platforms, this script manages and grows those positions.

**What it does:**
1. Queries your live PM wallet positions and Kalshi account positions
2. Cross-references with the mapper cache to auto-discover which positions are paired arbs
3. Writes discovered pairs to `data/positions_config.json` (inactive by default — you must approve each one)
4. Launches a web dashboard at `localhost:8765` where you activate/deactivate pairs, view portfolio stats
5. In continuous mode, monitors live order books for active pairs
6. When annualized return exceeds threshold (default 40%), executes trades: buys on both platforms simultaneously
7. Tracks cost basis, trade history, and P&L from real platform data

**How to run:**
```bash
python scripts/position_builder.py --paper --once   # Discover pairs, compute edges, don't trade
python scripts/position_builder.py --paper           # Continuous paper mode (no real trades)
python scripts/position_builder.py --once            # Single real execution cycle
python scripts/position_builder.py                   # Production: continuous real trading
```

**Dashboard:** Opens automatically at `localhost:8765`. Shows all pairs, live edges, portfolio tab with cost basis and P&L.

---

### Utilities

```bash
python scripts/validate_pipeline.py       # Health check: verify caches, matches, and pricing are intact
python scripts/config_dashboard.py        # Standalone dashboard (normally launched by position_builder)
```

---

## Source Code (`src/`)

### `src/mapper/` — Market Matching Engine

The core logic for figuring out which PM market = which Kalshi market.

| File | What it does |
|------|-------------|
| `embeddings.py` | Wraps the Gemini embedding API. Takes a list of market title strings, returns 768-dimensional vectors. Handles batching (100 texts per API call) and rate limiting (backs off on 429s). |
| `ai.py` | Wraps Gemini 2 Pro for two tasks: **(1) Match validation** — given a PM title and Kalshi candidates, asks "are these the same bet?" Returns the matching ticker or null. **(2) Resolution screening** — given a matched pair, checks if they'd resolve the same way (same deadline, same criteria). Returns PASS/FAIL with a resolve date. |
| `cache.py` | JSON-based cache for match results. Stores both matches (PM title -> Kalshi ticker + metadata) and explicit no-matches (so we don't re-check known non-matches). Supports multiple target platforms (Kalshi, Opinion, etc.) with separate cache files. Has TTL-based invalidation for no-matches when new markets appear. |

**How matching works:**
1. Every market title gets turned into a 768-dim vector via Gemini embeddings
2. Cosine similarity finds the closest Kalshi titles for each PM title
3. Top candidates (similarity > 0.75) go to Gemini AI for validation
4. AI confirms or rejects the match, checking for subtle differences (different time periods, different metrics, etc.)
5. Results are cached so subsequent scans only process new markets

### `src/kalshi/` — Kalshi API Client

| File | What it does |
|------|-------------|
| `client.py` | REST client for the Kalshi API. Handles RSA-PSS request signing for authentication. Methods for: fetching series/markets/order books, placing/canceling orders, querying positions and settlements, getting account balance. Supports both production and demo environments. |

### `src/polymarket/` — Polymarket Clients

| File | What it does |
|------|-------------|
| `client.py` | Async client for the PM CLOB API. Fetches markets with cursor pagination (the CLOB has 400k+ markets total, ~30k active). This is the only reliable way to get all PM markets — the Gamma API is broken and misses ~40% of open markets. |
| `trading.py` | Trading client using `py-clob-client`. Handles the Gnosis Safe proxy address derivation from your MetaMask private key. Methods for: placing limit orders, getting order books, checking balance and positions. Used by position_builder for execution. |
| `categories.py` | Maps PM's 471 flat tags to 7 normalized categories (politics, crypto, sports, finance, tech, culture, climate). Used by scan_category to filter markets before embedding. |

### `src/opinion/` — Opinion.trade Client

| File | What it does |
|------|-------------|
| `client.py` | REST client for Opinion.trade. Handles paginated market fetching (max 20 per page), order books, and price queries. Requires API key. |
| `categories.py` | Keyword-based category filter for Opinion markets (they don't have a tag system like PM). |

---

## Data Files (gitignored, generated at runtime)

| File | What it stores |
|------|---------------|
| `data/mapper_cache.json` | PM <-> Kalshi match cache. ~2k matches + ~4k explicit no-matches. |
| `data/pm_opinion_cache.json` | PM <-> Opinion match cache. Same format. |
| `data/resolution_cache.json` | AI resolution screening results (PASS/FAIL + resolve dates). |
| `data/pm_clob_cache.json` | Full PM CLOB market dump (~30k active markets). 6-hour TTL. |
| `data/positions_config.json` | Position builder config: pair list, active/inactive status, global settings. |
| `data/live_edges.json` | Latest edge data from position builder (prices, cost basis, balances). |
| `data/trade_log.json` | Trade execution history from position builder. |
| `data/*_emb.npz` + `*_emb_meta.json` | Embedding caches per category (e.g., `kalshi_politics_emb.npz`). |
| `data/arb_monitor.html` | Latest monitor dashboard snapshot. |

---

## Setup

### Install Dependencies

```bash
pip install -r requirements.txt
```

### API Keys

| Key | Location | How to get |
|-----|----------|------------|
| Gemini API | `.env` as `GEMINI_API_KEY` | [Google AI Studio](https://aistudio.google.com/apikey) |
| Kalshi API | `kalshiapikey/kalshiapikey.txt` | [Kalshi API settings](https://kalshi.com/settings/api) |
| PM private key | `polymarketkey/private_key.txt` | Export from MetaMask (only needed for trading) |
| Opinion API | `opinionkey/apikey.txt` | Apply via Google Form (only needed for Opinion scanning) |

### Platform Fees

| Platform | Taker Fee | Maker Fee |
|----------|-----------|-----------|
| Kalshi | `ceil(7 * P * (1-P))` cents per contract | `ceil(1.75 * P * (1-P))` cents |
| Polymarket | 0.10% flat | 0.10% flat |
| Opinion | max(50c, 2% of P*(1-P)) per contract | 0% |

Fees are highest at 50/50 prices and approach zero at extreme prices (>90c or <10c). At extreme prices, Kalshi rounds up to a minimum of 1c per contract.
