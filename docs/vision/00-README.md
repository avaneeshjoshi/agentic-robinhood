# Signal / Brain — Teardown & 100x Product Vision

This folder is the complete written teardown of the open-source project **"Brain" / "Signal"** (`robinhood-agentic-main`) — an agentic stock-research assistant plus an autonomous paper-trading fund ("Autopilot"/"the Twin") layered on a read-only Robinhood feed — and the product vision for rebuilding it, from scratch, as something 100x better in **this** repository.

Nothing in this folder is code. It is the spec you build against.

## How to read this

| Read when… | Doc |
|---|---|
| You want the 5-minute picture | `01-original-overview.md` → `09-rating-and-verdict.md` → `10-product-vision.md` |
| You are about to port/rewrite a specific subsystem | the matching teardown doc (02–07) + `14-roadmap-and-milestones.md` for the phase it lands in |
| You are designing the new system | `10` → `11` → `12` → `13` → `14` |
| You want to know what to fix from the original | `09-rating-and-verdict.md` (defect list) + `improvements-tracker.md` |

### Part I — Teardown of the original (what it does, how, how deep)

| # | File | What it covers |
|---|---|---|
| 01 | [01-original-overview.md](01-original-overview.md) | System architecture, the three background loops, the 16-step brain pipeline, the three learning loops, config, dependencies, how to run |
| 02 | [02-core-layer.md](02-core-layer.md) | `cli.py`, `config`, `models`, `llm` (prompts verbatim), `agent` (the chat agent loop + 16 tools), `orchestrator`, `mandate`, `shadow`, `research_state`, `evals`, `profile_*` |
| 03 | [03-engines.md](03-engines.md) | All 16 engines including the 1,309-line Twin/Autopilot, every LLM prompt, every threshold |
| 04 | [04-data-layer.md](04-data-layer.md) | yfinance prices, RSS news, Finnhub catalysts, StockTwits/ApeWisdom sentiment, SEC EDGAR, Robinhood charts, universe, market clock, and the two portfolio sources (manual / unofficial `robin_stocks`) |
| 05 | [05-database-schema.md](05-database-schema.md) | All 24 tables, the hand-rolled migration mechanism, the repository layer, the contextual-bandit and lesson-book algorithms |
| 06 | [06-web-api-and-ui.md](06-web-api-and-ui.md) | All 37 API routes, SSE streaming, the 8-screen vanilla-JS frontend, hand-rolled SVG charts, security findings |
| 07 | [07-tests-and-quality.md](07-tests-and-quality.md) | The 23 test files, what they prove, what is untested |
| 08 | [08-design-docs-and-roadmap.md](08-design-docs-and-roadmap.md) | The original's own vision/roadmap docs and a shipped-vs-planned reality table |
| 09 | [09-rating-and-verdict.md](09-rating-and-verdict.md) | **The rating.** Sub-scores, what is genuinely great (keep), ~45 concrete defects ranked by severity |

### Part II — The rebuild

| # | File | What it covers |
|---|---|---|
| 10 | [10-product-vision.md](10-product-vision.md) | Product thesis, the four execution modes and the trust ladder, what "100x" means on 8 axes, north-star screens, non-goals |
| 11 | [11-architecture-blueprint.md](11-architecture-blueprint.md) | Target stack, `BrokerAdapter` / `DataProvider` interfaces, Robinhood MCP mapping, LLM layer, event bus, notifications, auth, observability |
| 12 | [12-execution-and-risk-engine.md](12-execution-and-risk-engine.md) | Proposal → RiskGate → mode router → fill → reconcile; the **notify-only** design; kill switches; realistic paper fills |
| 13 | [13-research-and-evaluation-engine.md](13-research-and-evaluation-engine.md) | Research engines v2, multi-model judging, dynamic theme discovery, options-aware research, statistical evaluation, **backtesting engine** |
| 14 | [14-roadmap-and-milestones.md](14-roadmap-and-milestones.md) | Phases 0–7 with acceptance criteria and port-vs-rewrite decisions |
| — | [improvements-tracker.md](improvements-tracker.md) | Living checklist of every improvement, mapped to phase and status |

## The original in one paragraph

A single-user Python app (~20k lines). A FastAPI server runs three `asyncio` loops: a fast loop (every 120 s) that refreshes prices/portfolio and runs zero-LLM monitors; a slow loop (every 180 s) that walks a fixed 16-step pipeline of LLM-spending engines (sentiment, catalysts, thesis re-judging, missions, autonomous deep research, structural risk, mandate review/drift, theme scout, strategy discovery, Autopilot review/decide/fill/snapshot, self-grading judge, feed pre-warm), each self-gated by cooldowns so "a calm book spends nothing"; and a briefing loop. The LLM is Claude Opus 4.7 via the Anthropic SDK with a frozen prompt-cached system prompt, adaptive thinking, structured outputs, and server-side web search. Portfolio data comes from the **unofficial** `robin_stocks` library (username/password, read-only, against RH ToS) or a manual JSON file; prices from yfinance; news from RSS/Finnhub; filings from SEC EDGAR. Every recommendation is paper-logged with SPY and sector-ETF anchors so it can be graded later; an LLM-as-judge scores every reasoning trace against a 12-mode failure taxonomy and repairs flawed calls once; the "Twin" clones your real account and trades it autonomously in paper under fixed capital, with a deterministic critic, pre-fill validation, multi-window thesis-aware reviews, and a contextual bandit that feeds lessons back into the next decision prompt. The frontend is one hand-written HTML file + 2,500 lines of vanilla JS with hand-rolled SVG charts. There is no authentication, no migrations, and no real order execution.

## The rating in one line

**7/10 as a research prototype, ~3/10 as a product.** The LLM engineering, evaluation layer, and Autopilot design are unusually thoughtful; the data quality, execution realism, security, and operational maturity are prototype-grade. Full breakdown in `09-rating-and-verdict.md`.

## The rebuild in one line

A personal market operating system that **researches, remembers, measures itself, and then acts** — through a trust ladder of execution modes (paper → notify-only → approval-gated Robinhood MCP → autonomous-within-limits) — built on Python + Next.js with the **official Robinhood MCP** as the broker/data backbone instead of scraping.

## Vocabulary map (the original uses different names in code vs UI)

| Code name | UI name | What it is |
|---|---|---|
| `brain` / "Brain" | **Signal** | The whole product |
| `twin` | **Autopilot** | The autonomous paper fund cloned from your real book |
| `shadow` | **Scorecard** | Paper-logged recommendations and their graded outcomes |
| `profile` (tab id) | **Settings** | Risk profile / investor personality |
| `mandate` | "your plan" / "your goal" | The user's standing investing goal in plain words |
| `research_state` (watchlist + theses) | **Memory** / "Tracked theses" | Durable research memory |
| `findings` / `feed` | "On my radar right now" | The curated LLM-ranked view over the event stream |
| `research_events` | **Activity** | The persisted event stream / audit trail |
| `agent_runs` | "Research trace" / **Evals** | Every LLM reasoning trace, persisted |
| `missions` | Strategy missions | Standing theme trackers ("track defense stocks") |

## Assumed execution/data surface for the rebuild

The official **Robinhood MCP** toolset was available in the session that produced these docs and was inspected by name. It exposes, among others: `get_accounts`, `get_portfolio`, `get_equity_positions`, `get_equity_quotes`, `get_equity_historicals`, `get_equity_fundamentals`, `get_equity_news`, `get_equity_technical_indicators`, `get_equity_price_book`, `get_equity_tax_lots`, `get_equity_tradability`, `get_financials`, `get_earnings_calendar`, `get_earnings_results`, `get_option_chains`, `get_option_instruments`, `get_option_quotes`, `get_option_historicals`, `get_option_positions`, `get_scanner_filter_specs`, `create_scan`/`run_scan`, watchlist CRUD, `get_pnl_trade_history`, `get_realized_pnl`, `get_index_quotes`/`get_index_historicals`, and — crucially — `review_equity_order`, `place_equity_order`, `cancel_equity_order`, `review_option_order`, `place_option_order`. That surface is what `11-architecture-blueprint.md` maps the `BrokerAdapter` onto, and it is what makes approval-gated execution legitimate where the original's `robin_stocks` scraping was not.

## Provenance

These docs were produced by reading every file in `robinhood-agentic-main` (89 files, ~20.5k lines). Quotes marked as verbatim are copied from the source. Line references are to the original project's files.
