# 04 — Data Layer Teardown (`brain/data/` + `brain/portfolio/`)

Package docstring: *"Free sources (yfinance, RSS) behind narrow interfaces so paid providers (Polygon, Finnhub, …) can be swapped in without touching engines."* The stated design stance: **the LLM is never asked to recall or predict prices — it only interprets these grounded numbers.**

| File | LOC | Source | Verdict |
|---|---|---|---|
| `prices.py` | 493 | yfinance | Robust for free data; one-download screen; dividend-yield landmine |
| `news.py` | 81 | Yahoo RSS → Google News RSS | Thin; undated headlines |
| `catalysts.py` | 126 | Finnhub `company-news` | Robust, quarantined; cache key ignores `days` |
| `sentiment.py` | 129 | StockTwits + ApeWisdom | Sound, correctly subordinated |
| `edgar.py` | 262 | SEC submissions / XBRL / filing HTML | Strongest data file |
| `robinhood_charts.py` | 130 | `robin_stocks` | Thin; **no error handling** |
| `universe.py` + `sp500.json` | 87 | bundled JSON + hardcoded extras | Simple; static snapshot |
| `market_clock.py` | 64 | `zoneinfo` | Clean; holidays end 2027 |
| `portfolio/manual.py` | 39 | local JSON | Thin, unvalidated |
| `portfolio/robinhood.py` | 158 | `robin_stocks` (unofficial) | Careful thin wrapper on an unsafe idea |

---

## `prices.py` — the quantitative backbone (yfinance)

**Types.** `Quote(ticker, price, name, ok, error)`. `TrendSignals(ticker, price, name, sector, market_cap, pe, beta, dividend_yield, ret_1m/3m/6m_pct, above_50d, above_200d, vol_annualized_pct, rsi_14, notes)` with `.as_prompt()` → one line consumed by nearly every engine: `"NVDA (NVIDIA Corp, Technology): price $X, mktcap $YB, P/E Z, beta B, div yield D%, returns 1m +A% / 3m +B% / 6m +C%, above 50d MA, above 200d MA, RSI(14) R, annualized vol V%."` `ScreenRow` = the lightweight per-ticker row from batched history only.

**`SECTOR_ETF`** — 11 yfinance sector names → SPDR ETFs (Technology→XLK, Financial Services→XLF, Healthcare→XLV, Consumer Cyclical→XLY, Consumer Defensive→XLP, Energy→XLE, Industrials→XLI, Basic Materials→XLB, Utilities→XLU, Real Estate→XLRE, Communication Services→XLC). Powers sector-relative alpha in shadow and Twin.

**Functions.**
| Function | Behavior |
|---|---|
| `clean_ticker` | upper/strip; truncate at `^` (`VERV^`→`VERV`); `.`→`-` (`BRK.B`→`BRK-B`) |
| `get_earnings_date` | `yf.Ticker.calendar["Earnings Date"]`; earliest upcoming else latest past; **cached 24 h including negatives** |
| `clear_caches(include_signals=True)` | `include_signals=False` on the 2-min loop keeps daily-indicator caches warm |
| `get_quote` / `get_quotes(max_workers=8)` | `fast_info.last_price` → `history("5d")` close; 60 s TTL; threaded fan-out |
| `screen_universe(tickers)` | **One `yf.download(period="1y", group_by="ticker")` for the whole ~660-name universe**, momentum math in memory; 30-min TTL keyed on the joined ticker string; skips < 30 closes |
| `get_signals` / `get_signals_many` | `history("1y")` + `t.info` (shortName, sector, marketCap, trailingPE, beta, dividendYield); 15-min TTL |
| `get_chart(span)` | `1d→(1d,5m) 1w→(5d,15m) 1m→(1mo,1d) 3m→(3mo,1d) 6m→(6mo,1d) 1y→(1y,1wk)` |
| `get_portfolio_chart` | current quantities × historical closes + cash, rescaled to `target_latest`; docstring: *"This is not a broker statement history"* |

**Algorithms.** RSI = simple average of last 14 gains/losses (not Wilder); returns at 21/63/126 trading-day offsets; MAs over available window; vol = `sqrt(population var of daily log returns) × sqrt(252) × 100`.

**Defects.** 🔴 `dividend_yield = dy*100 if 0 < dy < 0.05 else dy` — a genuine 0.04% yield becomes 4%. Five module caches never evicted. `screen_universe`'s `yf.download` is unguarded. No tests.

---

## `news.py` — RSS headlines

`get_news(ticker, limit=6)` tries `feeds.finance.yahoo.com/rss/2.0/headline?s=…`, falls back to `news.google.com/rss/search?q={ticker} stock`; 15-min TTL. `headlines_as_prompt()` → `"Recent {T} headlines:\n- title (publisher)"` — the degradation fallback used by analyst, memory, briefing, deep_research when web search fails. Defects: `published` captured but never used (headlines reach the LLM **undated**); empty results cached 15 min; ambiguous tickers (`ALL`, `IT`, `ON`) pull unrelated Google news; `time.sleep(0.1)` effectively unreachable.

---

## `catalysts.py` — Finnhub catalyst radar

Structured, **timestamped** company news — "the fix for RSS's weakness." `available()` = enabled ∧ key; with no key everything is a clean no-op. `get_company_news(ticker, days=7)` → `GET finnhub.io/api/v1/company-news`, `timeout=8`, sorted newest first, 15-min TTL; **on failure returns the stale cache rather than `[]`**. `fresh_items(within_hours)`, `latest_fresh`, `catalysts_prompt(limit=5, days=14)` → `"RECENT CATALYSTS (structured news feed, dated):\n- Jun 12 (Reuters): …"` or `""`.

🔴 **The cache key ignores `days`**: `catalysts_prompt` asks for 14 d, `fresh_items` for 7 d; whichever warms first wins for 15 min. Rate limits (60/min free tier) not modeled.

---

## `sentiment.py` — social crowd read

StockTwits `streams/symbol/{t}.json` (counts user-tagged Bullish/Bearish messages → `bullish_pct`, `tagged`) + ApeWisdom `filter/all-stocks/page/1` (**one call fetches the whole trending map**, cached, shared across the universe — Reddit mentions + 24h-ago count). `get_sentiment()` merges to `{bullish_pct, tagged, mentions, mentions_prev, mention_delta_pct, rank}`. `sentiment_prompt()` → `"SOCIAL SENTIMENT (secondary crowd context, not fact): StockTwits 72% bullish (140 tagged); Reddit mentions 310, +45% vs yesterday."` Correctly subordinated by both label and the caller-side guidance. Notes: `bullish_pct` is self-labeled retail tags over ~30 posts — very noisy; StockTwits v2 is undocumented and intermittently auth-gated.

---

## `edgar.py` — SEC primary sources ★

*"The layer that separates frontier-grade analysis from summarizing news about a company."* Free; needs only `EDGAR_USER_AGENT`. TTLs matched to volatility: ticker→CIK 24 h, submissions/companyfacts 6 h, filing text 24 h ("a specific filing's text never changes").

| Function | Behavior |
|---|---|
| `recent_filings(ticker, limit=8)` | `company_tickers.json` → CIK → `data.sec.gov/submissions/CIK{cik}.json` → `[{form, date, description, url}]` with direct Archives URLs (accession dashes stripped) |
| `financial_facts(ticker)` | `api/xbrl/companyfacts/CIK{cik}.json`; `_CONCEPTS` = Revenue (3 tag fallbacks), Net income, Operating income, Gross profit, Diluted EPS, Total assets, Total liabilities, Shareholders' equity, Cash; prefers USD then USD/shares; **de-dups by period end keeping latest-filed** (handles restatements); last 4 values |
| `filing_text(ticker, form="10-K", max_chars=16000)` | fetches the latest filing of that form, strips HTML (3-stage regex), and **anchors the excerpt on "risk factors" → "management's discussion"** (both apostrophe variants) |
| `filings_as_prompt`, `facts_as_prompt` (`$12.34B` formatting), `filing_text_as_prompt` | the researcher's tool outputs |

Defects: unbounded cache holding multi-MB companyfacts blobs; a failed CIK fetch caches an empty map for 24 h (silently disables EDGAR for a day); bails on non-`.htm/.html/.txt` documents; no 429 handling; no fiscal-period alignment. Well tested (`test_edgar.py`).

---

## `robinhood_charts.py` — broker-native charts

Active only when `PORTFOLIO_SOURCE=robinhood`. `get_stock_chart` → `rh.stocks.get_stock_historicals(interval, span, bounds)`; `get_portfolio_chart` → `rh.account.get_historical_portfolio`. Span map: `1d→(5minute, day, extended) 1w→(10minute, week) 1m→(hour, month) 3m→(day, 3month) 6m→(day, year, then client-filtered to 186 d) 1y→(week, year)`. Per-span TTLs 20 s → 900 s. 🔴 **The only file in `data/` with no error handling around its API calls**; reaches into the private `rh_portfolio._login()`; duplicates `prices.get_chart` logic. Untested.

---

## `universe.py` + `sp500.json`

`sp500.json` = a plain JSON array of **503** uppercase tickers (multi-class listings), 3.6 KB, static — no refresh mechanism. `_EXTRAS` ≈ 180 hardcoded names by section (story stocks, AI/chips, defense/space, fintech/crypto, software/security, biotech, energy/uranium/nuclear, broad + sector ETFs "Autopilot may use as liquid exposure"). Optional `data_store/universe_extra.json`. `full_universe()` is `lru_cache`d → ~660 symbols; editing the extras file needs a restart. `screening_universe(exclude)` is the entry point for discovery, findings, theme_scout, twin. No liquidity/market-cap filter; some ambiguous symbols (`TOOL`, `S`, `AI`, `U`, `ON`).

---

## `market_clock.py`

`ZoneInfo("America/New_York")`, open 09:30, close 16:00 (half-open). `_HOLIDAYS` = 20 hardcoded NYSE closures for **2026–2027 only**; early closes ignored. `is_market_open`, `session_phase`, `next_open` (walks forward past weekends/holidays). Well tested. 🔴 After 2027-12-24 the holiday set is empty → the Twin will "fill" on holidays at stale quotes with no warning.

---

## `portfolio/__init__.py` — the swappable source with a 3-tier read-through

`get_portfolio(refresh=False)`: in-process cache (30 s) → DB snapshot → JSON snapshot → live fetch (`robinhood.fetch_portfolio` lazily imported, else `manual.load_portfolio`) → on exception return `Portfolio(sync_ok=False, sync_message=e)` → **persist only if `sync_ok and holdings`** so a bad read never overwrites a good snapshot. Module-global cache is not thread-safe.

## `portfolio/manual.py`

`data_store/holdings_manual.json` `{"holdings":[{ticker, quantity, avg_cost}], "cash"}`; batch quotes; `buying_power = cash`. No input validation, non-atomic write.

## `portfolio/robinhood.py` — the unofficial read-only path

Module docstring is a safety statement: *"It NEVER calls any order-placement function — there are none imported here. This path is against Robinhood's ToS and stores a session token locally; use it knowingly."* Verified: no order function is imported anywhere in the repo.

| Aspect | Implementation |
|---|---|
| Library | `robin_stocks.robinhood` |
| Auth | `rh.login(username, password, store_session=True)` + optional `pyotp.TOTP(RH_MFA).now()`; a module `threading.Lock` with double-checked `_logged_in` (without it every thread started a device-approval challenge → 429) |
| Endpoints | `profiles.load_account_profile()` (cash, buying power), `profiles.load_portfolio_profile()` (equity — first non-zero of extended_hours_equity, extended_hours_portfolio_equity, equity, portfolio_equity, market_value), `account.build_holdings()`, raw `request_get(historicals_url(), {symbols, interval:"5minute", span:"day", bounds:"24_7"})` |
| Price precedence | 24_7 historical close → RH quote → yfinance |
| Output | `Portfolio(pricing_source="Robinhood API 24_7 historicals / portfolio profile", pricing_warning=…)` |

Defects: `build_holdings()` and the historicals call unguarded; `_logged_in` never resets on session expiry; plaintext creds; token pickled to `~/.tokens/`; `pyotp` undeclared.

---

## What this layer teaches the rebuild

- Keep the **narrow-interface** idea (`DataProvider`) — it's why swapping to the Robinhood MCP is feasible.
- Keep: one-download screening, tiered TTLs, "return stale cache on failure", the EDGAR anchoring + XBRL de-dup, the `SECTOR_ETF` map, `clean_ticker`.
- Replace: yfinance quotes/fundamentals/historicals → RH MCP (`get_equity_quotes`, `get_equity_fundamentals`, `get_equity_historicals`, `get_financials`, `get_equity_technical_indicators`, `get_earnings_calendar`, `get_equity_news`); `robin_stocks` → RH MCP (`get_portfolio`, `get_equity_positions`, `get_accounts`); hardcoded holidays → a real exchange calendar; static `sp500.json` → RH `run_scan` / a maintained index feed.
