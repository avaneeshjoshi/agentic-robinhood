# 08 — The Original's Own Design Docs, and Roadmap vs Reality

The project ships five design documents. They are unusually concrete, and the shipped Phase-1 UI visibly follows them. Summaries below, then the shipped / partial / never-built table.

## `PRODUCT_VISION.md` (328 lines)

- **North star**: "a source-aware investing operating system" that tracks the real portfolio, remembers theses, learns style, monitors, warns, finds opportunities unprompted, explains with evidence, logs recommendations and whether they were right, measures edge, and keeps execution manual "unless a separate approved execution account is added later."
- **Surfaces**: Today · Portfolio · Research/Memory · Assistant · Settings. "Avoid adding many overlapping tabs."
- **Architecture direction**: `broker/prices/news/earnings/filings → cheap deterministic monitors → database → LLM only when useful → UI`. "Cheap checks should run often. LLM calls should be gated by importance."
- **Identity**: research copilot now; execution-with-approval "only if the system proves measurable edge." "Execution is not a UI feature. It is an identity shift that requires measured trust first."
- **Trust & evaluation**: the Scorecard must answer: did high-conviction beat low-conviction; which engines were right; which signals mattered; early/late/wrong; did the user act; vs SPY/QQQ/sector. "This is the bottleneck between 'interesting assistant' and 'something the user would trust with real money.'"
- **Known chart truth**: the portfolio history line is `current holdings × historical prices, anchored to current broker equity` — an estimate, because RH portfolio historicals fail through `robin_stocks`.
- **Living memory**: a thesis should be revisited on invalidation, earnings contradiction, price break, concentration change, driver news, staleness. Example: *"You said the AMD thesis breaks if data-center growth disappoints. New guidance weakened that exact segment. Decision: EXIT REVIEW."*
- **Recommendation labels** (mandatory): BUY CANDIDATE · WATCHLIST · WAIT FOR PULLBACK · HOLD · TRIM · EXIT REVIEW · REJECT · DO NOTHING. "No finance essays without a decision."
- **Execution preconditions**: calibrated edge, sizing rules, max-loss/allocation guardrails, human-approved order review, all actions logged, limit orders only at first.
- **Data ladder**: free now; first paid upgrade should be fundamentals + filings + transcripts, "because they improve thesis quality more than just faster prices."
- **Roadmap priority**: DB spine → evaluation → living memory → missions → data quality → deep research → execution-with-approval.

## `AUTOPILOT_README.md` (175 lines)

The Twin's contract: never places real orders; clones cash/positions/value once; never re-syncs; cannot invent cash; cannot sell more than owned; off-hours decisions queue; newest pending batch supersedes; history stays auditable. Candidate sources (holdings, watchlist, theses, missions, events, broad S&P 500 + curated universe, `universe_extra.json`, Theme Scout, Strategy Discovery); "The LLM may not make up tickers." Decision shape (action, dollars, tactic, horizon, thesis, exit rule, review window, attribution). Ordered execution plans with `depends_on`. Pre-trade critic (reject ungrounded, reject unowned sells, cap sells, scale buys, align tactic, size by reviewed results, `critic_note`) — "There is intentionally no hard single-position comfort cap." Pre-fill validation (chase, thesis break, gap-down, funding resize). Stage 3 self-review loop (fill anchors → mature windows → score vs SPY + sector → thesis state → lessons into prompts). Contextual bandit contexts (tactic, sector, theme, strategy, regime, and pairs) — "Full RL is parked until there is enough real paper-trade history and a backtesting/simulation layer." Strategy experiments (hypothesis, tactic, theme, regime, entry/exit, sizing, roster, evidence, score, status). Stage 4 shorting parked: needs margin model, borrow costs, squeeze limits, max loss, stop/cover rules, beta/hedge accounting — "The first step should be hedging behavior, not naked shorts."

## `design/product-roadmap.md` (531 lines)

- Meaning: "a personal market operating system… an instrument panel for judgment, not a dashboard of widgets."
- Brand: `Signal` chosen over Thesis/Conviction/Ledger/Vector/North/Axis. Logo: "a quiet directional glyph… The mark is not 'AI is alive.' The mark is 'attention is being directed.'" Connection state secondary (`Signal    Robinhood read-only`).
- Shell: top bar with mark + name + connection pill; **center: global command/search**; right: value, sync, notifications, settings. Final IA: Today · Activity · Portfolio · Memory · Scorecard · Settings (+ optional Research). "Do not add tabs for every engine."
- Surfaces: Today = "an edited front page" with **no-action confirmation**; Activity = "not for beauty. It is for trust"; Portfolio = account truth + structure, "The AI should not take over this page"; Memory = case files separating Latest analysis / Stored thesis / Watch reason / Evidence ledger; Scorecard = hit rate, alpha, **max drawdown**, calibration, by label, by engine; Settings = "a control plane": limits, sources, **connected accounts, OAuth, notification cadence, LLM spend limits, execution mode**.
- Broker-agnostic: durable objects = user, connected account, snapshot, position, thesis, event, recommendation, scorecard result; brokers listed: Robinhood, Schwab, Fidelity, Coinbase, Alpaca, IBKR, manual.
- Insight shape: `Decision / Evidence / Risk / What would change the call / Action label / Source · freshness`.
- Visual system: warm off-white, ink, soft lines, muted metadata, green/red only for market movement, blue/violet for assistant state, amber for caution; unboxed sections, thin borders, dense data, large charts.
- Phases: 1 clarify current app (renames, shell) → 2 component system (tokens, primitives) → 3 frontend migration (Next.js/React, TypeScript, TanStack Query, generated OpenAPI types, lightweight-charts/visx, Radix/shadcn; "Keep the FastAPI backend initially") → 4 productized proactivity (pre-warmed Today, no-action confirmations, inbox, cadence controls) → 5 trust layer (calibrated conviction, attribution, backtest/replay, **lineage event → research → action → outcome**).
- North-star screen order: Today (do I need to care?) → Portfolio → Memory → Scorecard (is this assistant actually good?) → Activity → Settings. "Do not show the user the machinery unless they asked for it."

## `design/production-ui-vision.md` (114 lines)

The earlier draft the roadmap expanded. Same moves; adds `alpha vs SPY / QQQ / sector ETF`, `event IDs / citations`, and `execution mode: manual / approval-only / off`. "The important migration is not 'HTML to React' by itself. It is moving from one-off screens to a consistent app shell, design tokens, reusable components, typed API state, and clearer product surfaces."

## `design/signal-logo-notes.md` + the two SVGs

Six concepts; **01 Folded vector chosen** ("attention being routed… a decision path without using a chart/candlestick"); 02 Decision path as runner-up. Lockup: `[mark] Signal` with connection state as a separate string. "The brokerage is a data source, not the brand." `signal-logo-concepts.svg` (1800×1600) and `production-ui-vision.svg` (1600×2200) are hand-authored mockups; the mockups use blue `#3f5fdb` while the shipped CSS uses violet `#6e56cf`.

## Roadmap vs reality

| Roadmap item | Status |
|---|---|
| Rename Brain → Today | ✅ shipped as **Home**, a chat-first cockpit (arguably stronger than "edited front page") |
| Rename Shadow → Scorecard, Profile → Settings | ✅ labels only; DOM ids unchanged |
| Remove green-dot brand; folded-vector mark; connection pill | ✅ |
| Portfolio two-column, AI below the chart | ✅ hero chart + rail + structural-risk lenses |
| Activity as raw audit trail with severity/source/evidence links | ✅ |
| Memory case-file-first with the four-way separation | ✅ the Dossier |
| Scorecard: calibration, by engine, by label, ledger, alpha vs SPY | ✅ plus mission-grouped themes |
| Restrained palette, unboxed sections | ✅ CSS tokens match the spec |
| Consistent insight shape | ✅ in dossier/deep report/ticket |
| Living memory, missions, deep research, event engine, DB spine | ✅ all built (the vision's roadmap priorities 1–6) |
| **Global command/search bar** | ❌ |
| **No-action confirmation** on Today | ❌ (opener describes the book; never says "nothing to do") |
| Notification inbox | ◐ badges + browser notifications, no inbox surface |
| Factor exposure on Portfolio | ◐ the lens map is a cousin |
| **Max drawdown** on Scorecard | ❌ |
| Alpha vs QQQ / sector in the Scorecard UI | ◐ sector alpha exists in shadow/Twin; UI shows SPY only |
| **Settings as a control plane** (sources, accounts, OAuth, cadence, LLM spend limits, execution mode) | ❌ still a form |
| **User login / accounts / encrypted tokens / multi-broker** | ❌ entirely absent — the largest gap |
| Phase 2 component system | ◐ tokens exist; styling is still per-tab; 10–15% dead CSS |
| Phase 3 React/Next + TS + TanStack + OpenAPI types + chart lib | ❌ still vanilla JS with hand-rolled SVG |
| Phase 4 user-controllable cadence | ❌ env vars only |
| Phase 5 backtest/replay; lineage event → research → action → outcome | ◐ pieces exist separately; no lineage view; no backtest |
| Execution-with-approval (roadmap priority 7) | ❌ by design |
| "Do not add tabs for every engine" | ⚠️ violated: `Evals` and `Autopilot` are engine-named tabs (8 tabs vs the target 6) |

The original built its brain roadmap almost completely and its *product* roadmap barely at all. The rebuild inverts nothing — it keeps the brain ideas and finally builds the product layer, plus the execution ladder the original explicitly deferred.
