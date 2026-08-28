# Improvements Tracker

Living checklist. Every defect from `09-rating-and-verdict.md` and every new capability from docs 10–13, mapped to the phase in `14-roadmap-and-milestones.md`. Update `Status` as work lands (`todo` · `in-progress` · `done` · `dropped`).

Categories: SEC security · DATA data/broker · EXEC execution/risk · ENG engines/LLM · EVAL evaluation · DB persistence · WEB api/frontend · OPS ops/quality · PROD product.

## Fixes to the original's defects (D-xx ↔ doc 09 #)

| ID | Cat | Improvement | Phase | Status |
|---|---|---|---|---|
| D-01 | SEC | Authenticate every route; rate-limit and budget-guard LLM routes; re-auth on destructive/execution routes | 0 | todo |
| D-02 | SEC | No `innerHTML` with untrusted strings; DOMPurify for LLM markdown; CSP | 0 | todo |
| D-03 | DATA | Replace unofficial `robin_stocks` with the official Robinhood MCP adapter; no broker passwords in env | 0 | todo |
| D-04 | DB | Typed repository errors + structured logging; no silent `except: return` | 0 | todo |
| D-05 | DB | Alembic from commit one; Postgres + JSONB + real FKs | 0 | todo |
| D-06 | OPS | Scheduler with cancellable per-job timeouts (no `to_thread` leak) | 0 | todo |
| D-07 | EXEC | Paper fill model: spread, slippage, partial fills, commissions, market hours | 2 | todo |
| D-08 | EXEC | `edge_pct` apples-to-apples (marked value both sides) | 2 | todo |
| D-09 | ENG | Profile learning with decay/reversal (no one-way ratchets); sector can't be in favor and avoid | 1 | todo |
| D-10 | OPS | CLI regenerated from the API client; no dead commands | 1 | todo |
| D-11 | ENG | Structural-risk fallback with word-boundary matching, no fake tickers, no double counting; deterministic correlation clusters first | 1 | todo |
| D-12 | DATA | Fundamentals from RH MCP / EDGAR with explicit units; no dividend-yield heuristic | 0 | todo |
| D-13 | ENG | LLM cost ledger; per-engine daily budgets; global deadline + token budget on the researcher; per-cycle cap on memory re-judges | 1 | todo |
| D-14 | EVAL | Judge on a different model/persona; Cohen's κ vs human labels reported continuously | 1 | todo |
| D-15 | ENG | Deep-research critique receives dossier + signals; researcher threads source URLs to the judge | 1 | todo |
| D-16 | ENG | `decide()` best-effort for real (hold on model failure, persist attempt) | 2 | todo |
| D-17 | ENG | `dd_normal` used in grace verdict | 2 | todo |
| D-18 | ENG | One `market_regime`, one `screen_score`, shared module | 1 | todo |
| D-19 | EXEC | Transactional paper fills (cash + position per fill) | 2 | todo |
| D-20 | EXEC | Inception never destroys a paused fund | 2 | todo |
| D-21 | ENG | Dynamic theme discovery (correlation + news-entity clustering, LLM-named); tactics as declarative rules incl. below-200d and breakout | 2 / 5 | todo |
| D-22 | ENG | Validate mission tickers via `get_equity_tradability`; define "balanced"; fail-closed on bad timestamps | 2 | todo |
| D-23 | ENG | Discovery: multiple screens (value/quality/mean-reversion/dividend), judge-gated, web-grounded | 2 | todo |
| D-24 | EVAL | Evaluation v2: bootstrap CIs, sample-size weighting, horizon-aware maturity, drawdown/Sharpe/Sortino, reliability curve, tests | 2 | todo |
| D-25 | DATA | Catalyst cache keyed on (ticker, days) | 1 | todo |
| D-26 | DATA | Exchange calendar library instead of hardcoded holidays | 0 | todo |
| D-27 | DATA | All adapter calls wrapped with typed errors, retries, stale-on-error | 0 | todo |
| D-28 | DATA | EDGAR cache bounded (LRU, size cap); failed CIK fetch not cached | 1 | todo |
| D-29 | WEB | Proper HTTP status codes; typed error envelope; client checks `ok` | 0 | todo |
| D-30 | OPS | Briefings as calendar-aware scheduled jobs with logging, in ET | 1 | todo |
| D-31 | OPS | Lifespan startup; worker separate from API; multi-worker safe (Redis state) | 0 | todo |
| D-32 | WEB | Pydantic bounds on every body/param; no raw exception strings to clients | 0 | todo |
| D-33 | WEB | TanStack Query error/loading states; no leaks; visibility-aware polling | 1 | todo |
| D-34 | WEB | Accessibility: roles, focus management, keyboard paths, contrast ≥ 4.5:1 | 1 | todo |
| D-35 | WEB | Design tokens as a system; dark mode; one color scale for market movement | 1 | todo |
| D-36 | DB | Research state DB-only; no dual-write; explicit deletes | 1 | todo |
| D-37 | ENG | Chat agent on the shared `agent_loop`; bounded history; cached mandate block; loop tests | 1 | todo |
| D-38 | EVAL | Shadow ledger: real close semantics; win rate excludes hold/watch; indexed queries | 2 | todo |
| D-39 | ENG | Weights as fractions everywhere; every `DecisionLabel` reachable | 1 | todo |
| D-40 | OPS | `.env.example` committed; all deps declared; lockfile | 0 | todo |
| D-41 | DATA | Dynamic universe from RH scans + index feed; liquidity filter | 2 | todo |
| D-42 | DATA | News always dated (RH `get_equity_news`, Finnhub); no ambiguous-ticker fallbacks | 1 | todo |
| D-43 | OPS | pytest with per-test DB isolation, xdist, no hardcoded years, no vacuous assertions | 0 | todo |
| D-44 | OPS | Reset/maintenance scripts with backup + typed confirmation; tested | 1 | todo |
| D-45 | ENG | Persistent dive queue for autoresearch; config flags honored in-engine | 1 | todo |
| D-46 | ENG | Magic coefficients moved to a tunable, versioned config; backtest-validated | 5 | todo |
| D-47 | ENG | Briefing uses events/catalysts/sentiment/mandate; batched news; delivered as notification | 1 | todo |

## New capabilities (N-xx)

| ID | Cat | Improvement | Phase | Status |
|---|---|---|---|---|
| N-01 | EXEC | `Proposal` → Critic → `RiskGate` → `ModeRouter` pipeline with append-only order events | 3 | todo |
| N-02 | EXEC | `RiskGate` full rule table (position/sector caps, daily loss, turnover, cash buffer, liquidity, tradability, earnings blackout, event veto, price drift, PDT, kill switch) — property-tested | 3 | todo |
| N-03 | PROD | **Notify-only mode**: Telegram + web push cards, signed approve/decline/snooze links, expiry, quiet hours, max/day | 3 | todo |
| N-04 | EXEC | Reconcile real fills via MCP orders; slippage vs proposal; approved-unfilled handling | 3 | todo |
| N-05 | EVAL | `user_decisions` as training signal; followed-vs-not counterfactual P&L | 3 | todo |
| N-06 | EXEC | Kill switch (UI, Telegram, auto-triggers) + daily execution report | 3 | todo |
| N-07 | EXEC | Approval-gated execution via `review_equity_order` → `place_equity_order`; limit-only; lineage | 4 | todo |
| N-08 | PROD | Trust-ladder unlock criteria enforced in code and shown in Scorecard | 2–4 | todo |
| N-09 | EXEC | Autonomous-within-limits with post-trade notify, notify-first threshold, auto-demotion | 6 | todo |
| N-10 | EVAL | Scorecard → prompt feedback (`engine_feedback()` block in analyst/discovery/deep research) | 2 | todo |
| N-11 | ENG | Semantic thesis↔news matching (pgvector) and semantic evidence retrieval | 1 | todo |
| N-12 | DB | Lineage edges: event → research → proposal → order → fill → outcome; Activity lineage view | 3 | todo |
| N-13 | ENG | Thesis versioning with diffs | 1 | todo |
| N-14 | ENG | ModelRouter (generator/judge/critic/cheap roles); prompt registry with golden tests | 1 | todo |
| N-15 | PROD | Today: explicit "No action needed" card; pending proposals; global command bar | 1 / 3 | todo |
| N-16 | PROD | Settings as control plane: modes, hard limits, channels/cadence, LLM budget, sources health, accounts, kill switch | 1–3 | todo |
| N-17 | PROD | Notification inbox + mobile push | 3 | todo |
| N-18 | ENG | Factor exposure (regression betas) and correlation clusters on Portfolio | 2 | todo |
| N-19 | ENG | Research tools: earnings transcripts, insider transactions (Form 4), peer comparison, option-chain summary | 5 | todo |
| N-20 | EVAL | **Backtesting engine**: bar store, point-in-time fundamentals, event-driven sim on `PaperBroker` + `RiskGate`, decision replay, walk-forward, lookahead CI test | 5 | todo |
| N-21 | ENG | Strategy experiments require a passing backtest before `active` | 5 | todo |
| N-22 | ENG | **Options**: chains/Greeks/IV via MCP; `OptionsStrategist` as an expression layer over equity theses; options risk rules; paper/notify first | 5 | todo |
| N-23 | PROD | Options positions + Greeks on Portfolio; options P&L attributed to theses in Scorecard | 5 | todo |
| N-24 | OPS | Job timeline (port `_brain_timeline`), cost dashboard, staleness heartbeat, OTel traces | 0–1 | todo |
| N-25 | OPS | Test stack: call-count fixtures, scripted fake LLM, schemathesis, vitest + Playwright, judge-drift replay harness | 0–2 | todo |
| N-26 | PROD | **Multi-user/SaaS**: tenancy, encrypted broker tokens, per-tenant budgets, billing, legal review | 7 | todo |
| N-27 | DATA | `DataProvider` with source/as_of/ttl stamps on every datum; Redis tiered caches; stale-on-error | 0 | todo |
| N-28 | PROD | Paper twin runs alongside notify/approval modes ("what if you'd followed everything") | 3 | todo |
