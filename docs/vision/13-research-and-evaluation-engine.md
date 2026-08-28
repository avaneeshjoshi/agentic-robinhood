# 13 — Research & Evaluation Engine v2

Keep the original's engine set and its three loops (thesis, quality, outcome). Fix the identified weaknesses, close the one loop the original left open (scorecard → prompts), and add the expansion capabilities you asked for: options, backtesting.

## Research engines: keep / fix / add

| Engine | Keep | Fix | Add |
|---|---|---|---|
| monitor | 9 rules, pure core, cooldowns | thresholds scale with profile appetite | options-position rules (assignment risk, expiry ≤ 3 d, IV crush before earnings), position-drift rule |
| memory | four triggers, evidence ledger, "broken ONLY if…" | **semantic** thesis↔news matching (embed invalidation + drivers; cosine ≥ τ) instead of substring; per-cycle cap; log parse failures | thesis versioning (every re-judge is a version with diff); earnings-transcript trigger |
| findings | "view not scan" | shared `screen_score`; error handling | explicit **"No action needed"** finding kind so Today can say so |
| structural_risk | model-judges/code-computes | word-boundary fallback; drop `DRAM`; no double counting | factor exposure from returns regression (beta to SPY/QQQ/sector ETFs), correlation clusters from 60-day co-movement as a deterministic first pass the LLM then labels |
| discovery | funnel design, shadow-log dedup | judge-gate it; ground with web | **multiple screens**: momentum, value (P/E, EV/EBITDA vs sector), quality (margins, FCF, ROIC from EDGAR facts), mean-reversion, dividend growth; RH `run_scan` as a screen source; user-defined screens |
| analyst | three grounding sources + GROUNDING DISCIPLINE + judge gate | cache repeated calls (5 min); never lose the judgement on DB failure | options context when relevant (IV rank, skew, upcoming expiries) |
| researcher | the loop, tools, corroboration prompt | **capture source URLs** from tool results and thread them to the judge; global deadline + token budget | tools: `get_earnings_transcript`, `get_option_chain_summary`, `get_insider_transactions` (EDGAR Form 4), `get_peer_comparison` |
| deep_research | draft → critique → judge | critique gets the **dossier + signals**, not just the draft; guard parses; per-ticker cooldown in-engine | "what would change my mind" watch triggers auto-registered from the invalidation |
| autoresearch | triple gating | lookback covers the queue (persist a `dive_queue` instead of re-scanning 3 h) | |
| missions | 3 cadence tiers, WAIT label, single growth point | validate tickers exist (`get_equity_tradability`); define "balanced" | mission-level P&L in the scorecard (already partly there) |
| theme_scout | breadth + leadership + event support; Twin feedback | | **dynamic theme discovery**: cluster the tradable universe by 60-day return correlation + shared news entities (NER over RH/Finnhub news) → LLM names the clusters → persist as themes; hardcoded themes become seeds, not the ceiling |
| strategy_discovery | `theme:tactic:regime` keys; retirement rule | dedupe regime; hoist feedback query; drop dead params | tactics as **declarative rules** (entry/exit predicates over signals) so new tactics are data, not code; include mean-reversion below 200d and breakout tactics; every candidate strategy is **backtested first** (below) |
| judge | taxonomy, gate, non-regression | **different model (or at least a different prompt persona) for the judge**; validate judge scores against human labels continuously (report Cohen's κ) | a second, adversarial judge lens ("argue this is wrong") on high-conviction calls only |
| briefing | "do nothing is valid" | use events, catalysts, sentiment, mandate; batched news | delivered as a notification, with the day's pending proposals |
| evaluation | maturity bar, anchored alpha, mission themes, honest narrative | see below | see below |
| twin | everything | see doc 12 | applies to real fills too |

## Closing the open loop: scorecard → prompts

The original's Twin learns from outcomes; its advisory engines don't. Add `engine_feedback()` that renders, per engine, a compact block from the scorecard — *"Your last 40 matured BUY calls: alpha +1.8% (CI −0.4…+4.0), high-conviction calls underperform medium (inverted), momentum-sourced ideas lag sector by 2.1%"* — and inject it into analyst/discovery/deep-research prompts the same way `_policy_memory()` feeds Autopilot. Track whether it changes calibration.

## Evaluation v2

| Metric | Now | v2 |
|---|---|---|
| Win rate / avg return / alpha | point estimates | **bootstrap 90% CIs**, sample-size-weighted ranking (never name a "best engine" on n<10) |
| Benchmarks | SPY + sector ETF | + QQQ, + equal-weight sector, + user's own real account |
| Risk-adjusted | none | Sharpe, Sortino, **max drawdown**, hit-rate vs magnitude (payoff ratio) |
| Calibration | 3 buckets | reliability curve (conviction decile vs realized hit rate), Brier score |
| Maturity | 5-day bar | horizon-aware bars (port `_windows_for` to advisory calls too): a 3-month idea isn't graded at 5 days |
| Judge validity | agreement % | Cohen's κ vs human labels; drift alerts when κ drops |
| User behavior | none | approve/decline/edit rates, slippage between proposal and your fill, "followed vs not followed" counterfactual P&L |
| Attribution | by engine / label / mode / mission | + by signal (which screen surfaced it), by regime, by data source freshness |
| Lineage | none | click any number → the calls behind it → the evidence behind each |

All computed in SQL / pandas from the ledgers; tested with fixtures (the original's untested `evaluation.py` was pure arithmetic — no excuse).

## Backtesting engine (expansion phase)

Purpose: test strategy experiments and engine changes against history **before** the paper Twin tests them live, and replay the system's own past decisions with new prompts.

- **Data**: daily bars for the universe from RH historicals (+ yfinance/Polygon backfill), stored in Postgres/Parquet; corporate actions; the exchange calendar; **point-in-time** fundamentals from EDGAR XBRL (use `filed` dates, never restated values) to avoid lookahead.
- **Engine**: event-driven simulator over daily (later intraday) bars; the same `PaperBroker` fill model; the same `RiskGate`; strategies as declarative rules (the strategy_discovery tactics) or as Python callables.
- **Modes**: (1) rule backtest — no LLM; (2) **decision replay** — re-run stored decision contexts (`agent_runs`) through a new prompt/model and score the counterfactual; (3) walk-forward with rolling re-fit of bandit priors.
- **Outputs**: equity curve, CAGR, Sharpe, max DD, turnover, per-tactic/theme/regime attribution, and a `backtest_runs` row linked to the strategy experiment. Strategy status can only reach `active` if its backtest Sharpe > threshold over ≥ 2 regimes.
- **Guardrails**: no survivorship bias (delisted names kept), transaction costs on, lookahead checks in CI (a test that shifts data by one day and asserts results change).

## Options (expansion phase)

- Data: RH `get_option_chains`, `get_option_instruments`, `get_option_quotes`, `get_option_historicals`, `get_option_positions`; compute IV rank, skew, expected move, Greeks (py_vollib or QuantLib).
- Research: an `OptionsStrategist` engine that, for a **stock thesis the system already holds**, proposes the expression — covered call (income on a HOLD), cash-secured put (WAIT FOR PULLBACK with a target), protective put (concentration/thesis-at-risk), debit spread (BUY CANDIDATE with capped risk). Options are an *expression layer* over equity theses, never a standalone speculation engine.
- Risk gate additions: max notional, max defined-risk per position, no naked short options, assignment-aware cash reserve, earnings/expiry blackout.
- Execution: `review_option_order` / `place_option_order` through the same ladder; options start one rung lower than equities (paper → notify only until proven).
- Scorecard: options P&L attributed to the underlying thesis; Greeks exposure on Portfolio.

## Multi-user / SaaS (expansion phase, design notes only)

- `users`, `accounts` (per broker connection, encrypted tokens), per-tenant LLM budgets and rate limits, row-level tenancy on every table, background jobs partitioned by tenant, audit logs per tenant, billing via Stripe, legal review before any execution mode is offered to others (investment-advice exposure; per-jurisdiction rules). Not in v1.
