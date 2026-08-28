# 10 — Product Vision: the 100x Rebuild

## Thesis

> A personal market operating system that **researches, remembers, measures itself, and then acts** — climbing a trust ladder from paper trading to autonomous execution, one proven rung at a time.

The original proved that an LLM brain can reason over grounded data, grade its own reasoning, and run a paper fund with real discipline. It stopped there on purpose. The rebuild keeps that brain, replaces the prototype foundations under it, and adds the thing the original explicitly withheld: a **trustworthy path from recommendation to execution** — with you in the loop exactly as much as you choose.

Working name: keep **Signal** (the original's own choice, well argued: "separating signal from noise… serious without sounding like hedge-fund cosplay"). Alternatives if you want your own identity: **Ledger**, **Conviction**, **Northstar**, **Helm** (the last fits the "you steer, it navigates" framing of the execution ladder).

## Who it is for

You. One serious retail investor with a Robinhood account who wants a system that watches the book continuously, does frontier-grade research on demand and unprompted, tells the truth about its own track record, and — once it has earned it — tells you exactly what to trade and when, or trades for you within limits you set. Multi-user is an expansion phase, not the design center.

## The four execution modes and the trust ladder

| Mode | Who decides | Who executes | Unlock criterion |
|---|---|---|---|
| **1. Paper** | The agent (Autopilot) | Nobody — simulated fills with realistic slippage/spread | Day one |
| **2. Notify-only** | The agent, fully autonomously | **You**, after a push notification: "BUY 25 NVDA @ ≤ $131.40 (2.1% of book) — thesis… exit rule… expires 15:45 ET" with one-tap **Approve → I did it / Decline / Snooze** | Scorecard shows ≥ N matured calls with positive, significant sector alpha and calibration not inverted; risk gate green |
| **3. Approval-gated execution** | The agent | The system via the **official Robinhood MCP** (`review_equity_order` → you approve → `place_equity_order`), limit orders only at first | Notify-only has run ≥ M weeks with your approvals tracked; hard limits configured; kill switch tested |
| **4. Autonomous-within-limits** | The agent | The system, no per-trade approval, inside hard caps (per-trade %, daily loss, max turnover, sector caps, blackout windows); every trade still notified after the fact | Approval-gated has run ≥ K weeks with ≥ X% of proposals approved unchanged and no risk-gate breaches; explicit opt-in |

Modes are **per account and reversible**; a breach of any hard limit drops the account one rung automatically. Notify-only is the ToS-safe autonomous path: the logic is fully autonomous, the hands are yours. Mode 3/4 legitimacy rests on the official MCP — never on scraping.

## What "100x better" means, concretely

| Axis | Original | Rebuild |
|---|---|---|
| **Data** | yfinance (unofficial), undated RSS, static 503-ticker universe, holidays hardcoded to 2027, unofficial RH scraping | Official RH MCP for quotes/historicals/fundamentals/news/earnings/technicals/options/scans/positions/P&L; EDGAR kept; Finnhub optional; exchange calendar library; dynamic universe from RH scans + index feed; every datum stamped with source + freshness |
| **Research depth** | Analyst / deep research / researcher loop with one model judging itself | Same loops plus: independent critic **with evidence access**, multi-model judging, cited sources threaded end to end, semantic thesis↔news matching, dynamic theme discovery, value/quality/mean-reversion screens, options-aware research (chains, IV, Greeks), earnings-transcript ingestion |
| **Execution** | None (paper at last quote) | Four-mode ladder; realistic paper fills; notify-only with approve links; MCP order pipeline with review → approve → place → fill tracking → reconciliation; limit orders first |
| **Risk** | Prompt-level "preference"; Twin critic; preflight | A deterministic **RiskGate** in front of every order regardless of mode: position/sector caps, daily loss, turnover, liquidity/ADV, earnings blackout, PDT, cash buffer, kill switch, circuit breakers |
| **Evaluation** | Maturity-gated scorecard; judge; no significance; no feedback to prompts | Confidence intervals and sample-size weighting; Sharpe/Sortino/max drawdown; calibration curves; judge validated against human labels; **scorecard outcomes feed back into engine prompts**; backtesting engine to test strategies against history before the Twin tests them live |
| **Memory** | Theses + watchlist + missions; keyword triggers | Same objects, plus lineage (event → research → recommendation → order → fill → outcome), semantic retrieval over evidence, versioned theses, user decisions (approved/declined/edited) as first-class training signal |
| **UX** | Vanilla JS, hand-rolled SVG, no auth, no mobile push | Next.js + TypeScript, lightweight-charts, shadcn, typed API; **Today** with a real no-action confirmation; global command bar; notification inbox + mobile push; Settings as a control plane (modes, limits, cadence, LLM budget, sources, accounts) |
| **Ops** | `while True` loops, no logging in the DB layer, no cost ledger, no migrations, no CI | Durable scheduler with per-job timeouts that actually cancel; structured logs + OpenTelemetry; **LLM cost ledger with per-engine budgets**; Alembic; CI with coverage gates; single-command dev env |

## Product principles (inherited and sharpened)

1. **Ground everything.** The model never recalls or predicts a number; code fetches it, stamps it, and the model interprets it. Every load-bearing claim carries a source.
2. **Model judges, code computes.** Clustering, ranking, and prose are the model's; arithmetic, capital, risk limits, and fills are code's.
3. **A calm book spends nothing.** Every LLM call is gated by a trigger, a signature, a cooldown, or a user action — and now also by a **budget**.
4. **Measure before trusting.** No execution rung unlocks on vibes; the scorecard and the risk gate decide, and both are honest about sample size.
5. **Decisions, not essays.** Every insight ends in an action label with an invalidation condition and a freshness stamp.
6. **Degrade, never fail.** A missing source lowers conviction; it never blanks a screen or silently invents.
7. **Persist the reasoning.** Every decision is auditable: what the model proposed, what the governor changed, what you did, what happened.
8. **You set the pace.** Modes, limits, cadences, and budgets are yours; the system never quietly widens its own authority.

## North-star screens (6 tabs — engine tabs are gone)

| Tab | Question it answers | Notes |
|---|---|---|
| **Today** | Do I need to care right now? | State-of-book read, urgent alerts, **explicit "No action needed" card**, pending proposals awaiting your approval (in notify/approval modes), research queue, composer with the command bar |
| **Portfolio** | What do I own and how exposed am I? | Chart + rail, allocation, factor/lens exposure, concentration, per-position health, options positions and Greeks |
| **Memory** | What do we believe and what would break it? | Case files with versioned theses, evidence ledger, missions, deep dives; Autopilot's book as a second "believer" |
| **Scorecard** | Is this thing actually good? | Human-engine and Autopilot track records with CIs, drawdown, calibration, attribution; the trust-ladder gauge showing what's unlocked and why |
| **Activity** | Show me the audit trail. | Events, decisions, orders, fills, approvals, judge reads, cost — filterable, with lineage links |
| **Settings** | Control it. | Execution mode per account, hard limits, notification channels/cadence, LLM budget, data sources and their health, connected accounts, kill switch |

Autopilot and Evals become *views inside* Scorecard/Memory/Activity rather than tabs — honoring the original roadmap's own rule.

## Non-goals (for the core; some return as expansion phases)

- Not a general chatbot; not a Robinhood clone; not a signals-for-sale service.
- No shorting, margin, or leverage in the core (the original parked this correctly; needs a margin model, borrow costs, squeeze limits first).
- No crypto/futures/forex in the core.
- No "predict the price" features, ever.
- Multi-user/SaaS, options strategies, and backtesting are documented expansion phases (docs 13, 14), not v1.

## Success criteria for v1 (Phases 0–3 in doc 14)

- Real portfolio synced from the official MCP every N minutes with zero credentials in env files.
- Every recommendation has a benchmark-anchored paper record; the scorecard reports CIs.
- Notify-only mode delivers a proposal to your phone within 60 s of a decision, with approve/decline captured and scored.
- LLM spend is visible per engine per day and capped.
- A kill switch stops all proposals and orders in < 1 s.
- Zero unauthenticated routes.
