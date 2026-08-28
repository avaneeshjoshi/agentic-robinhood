# 12 — Execution & Risk Engine

The original's Twin already had the right shape for the *paper* case: LLM proposes → deterministic critic → queue → preflight → clamp at fill → review. This doc generalizes that into one pipeline that serves all four execution modes, and specifies the **notify-only** mode in detail.

## Pipeline

```
Decision run (any engine: Autopilot, analyst, mission, you via chat)
   │  produces Proposal(s): ticker, side, qty|usd, order_type, limit, tactic, horizon,
   │  thesis, exit_rule, review_after, conviction, source attribution, plan_step, depends_on
   ▼
Critic (ported from twin._critic): grounding, ownership, tactic normalization, attribution,
   │  bandit sizing pressure, fixed-capital scaling, ordering. Writes critic_note.
   ▼
RiskGate (new, deterministic, mode-independent) ──► rejected / resized / approved (+ reasons persisted)
   ▼
ModeRouter: paper | notify | approval | autonomous   (per account, from Settings)
   ▼
Order lifecycle: proposed → gated → sent/notified → approved|declined|expired → placed → partially_filled|filled|canceled → reconciled
   ▼
Fill events → review windows scheduled (port _windows_for) → scorecard / bandit / lessons
```

Every transition is a row in `orders`/`order_events` with actor (`system`, `risk_gate`, `user`, `broker`) and reason. Nothing is ever silently dropped — the original's "write rejections as canceled rows" rule becomes universal.

## RiskGate (hard limits, all in code)

| Check | Default | Notes |
|---|---|---|
| Max single position | 15% of equity (profile-tunable; **hard cap 35%**) | The Twin's "no cap by design" stays only in paper mode as an experiment flag |
| Max sector exposure | 40% | Sector from fundamentals; lens clusters from structural risk count as soft warnings |
| Max order size | 5% of equity per order; 10% per day per symbol | |
| Daily loss limit | −3% of equity realized+unrealized → freeze new buys for the day | Circuit breaker |
| Max daily turnover | 20% of equity | Anti-churn |
| Cash buffer | keep ≥ 2% cash | |
| Liquidity | order ≤ 1% of 20-day ADV; spread ≤ 1% | From RH quotes/price book |
| Tradability | `get_equity_tradability` must be tradable; no halted names | |
| Earnings blackout | no new buys inside 24 h before earnings unless tactic = catalyst_trade and flagged | From `get_earnings_calendar` |
| Event veto | cancel buys if a `thesis_broken` event landed since the decision (port preflight) | |
| Price drift | cancel buy if > +4% vs decision price; cancel sell if < −8% unless urgent tactic (port preflight) | |
| PDT | if equity < $25k, track day trades and block the 4th in 5 days | |
| Fractional / lot rules | round to broker rules; min order $1 | |
| Kill switch | Redis flag `execution:halt` → everything rejected with reason `halted` | Checked at every step |
| Mode ceiling | autonomous mode additionally requires: no rejections in last 24 h, scorecard gate green | Auto-demotes on breach |

Outputs: `approved | resized(new_qty, reason) | rejected(reason)`, persisted to `risk_decisions`. Property-tested (e.g. "sum of buys never exceeds available cash + accepted sells", "no order exceeds hard cap under any sequence").

## ModeRouter

### Paper
`PaperBroker` fills against live quotes with a **fill model**: mid + half-spread × side, plus slippage = `k · (order_$ / ADV_$)^0.5` (k tunable, default 0.1%), commission $0, partial fills when order > 5% of interval volume, no fills outside market hours, corporate actions applied from historicals. Cash and positions update **in one transaction per fill**. Port everything else from `twin.py` (inception, stale-batch cancel, review windows, compare) — and fix `edge_pct` to use marked value on both sides.

### Notify-only (the ToS-safe autonomous mode)

**Payload** (`notifications` row + rendered card):

```
SIGNAL · Proposal #8412 · expires 15:45 ET (in 2h 10m)
BUY  NVDA   25 sh   ≈ $3,285   (2.1% of book)   limit ≤ $131.40
Tactic: pullback_in_uptrend · Horizon: 1–3 months · Conviction 7/10
Thesis: … (2 lines)
Exit rule: … (1 line)
Funding: from TRIM AMD (plan step 1, $3,300)     Risk gate: approved (resized from $4,000: sector cap)
Sources: Q2 10-Q (EDGAR) · Reuters 08-27 · RH fundamentals
[ Approve — I placed it ]  [ Approve with changes ]  [ Decline ]  [ Snooze 1h ]
```

- **Channels**: web push + Telegram (recommended: instant, free, supports inline buttons), optional email/SMS. Links carry a signed, single-use, expiring token; approval also possible in the Today tab.
- **Expiry**: default end of session or 4 h, whichever first; an expired proposal is scored as `expired` (not a market outcome) and the decision engine may re-propose next cycle at fresh prices.
- **Approve → reconcile**: after approval, the system polls `get_equity_orders`/positions via the MCP and matches your real fill (symbol, side, qty ± tolerance, time window); records `executed_price` and slippage vs the proposal's price. If nothing matches in 24 h it asks once, then marks `approved_unfilled`.
- **Decline** captures a reason (chips: too big / don't like the name / timing / other + text) — stored in `user_decisions` and fed to the judge and the bandit as context.
- **"I'm late"**: if you open a proposal after expiry, the card shows the current price vs the proposal and a one-tap "re-evaluate now" that runs the risk gate at fresh prices.
- **Quiet hours** and **max proposals/day** in Settings; urgent sells (risk_reduction) may bypass quiet hours if you allow it.
- **Paper twin runs in parallel**: every notify proposal is also filled in the paper book so you can see what would have happened if you had followed every proposal, vs what you actually did.

### Approval-gated (official MCP)
`review_equity_order` → store the review (cost, fees, warnings) → notification with the same card plus the broker's review → on approve: `place_equity_order` (limit orders only until Phase 6; market orders allowed only for urgent risk_reduction with your opt-in) → poll order status → fills → reconcile positions and cash from the MCP, not from local math → notify result. Cancel path: expired approvals cancel the review; open limit orders auto-cancel at session end unless GTC is enabled.

### Autonomous-within-limits
Same as approval-gated minus the wait; every placed order notifies immediately with a one-tap **Cancel**. Extra guards: proposals above a per-trade "notify-first" threshold ($ or %) still require approval; N consecutive risk-gate rejections or one circuit-breaker trip demotes the account to approval-gated and notifies.

## Reconciliation & truth

The broker is the source of truth for positions/cash; the local book is a cache. `sync_portfolio` diffs MCP positions against local, emits `position_drift` events (you traded outside the system, dividends, splits), and never lets a local fill overwrite broker truth. The Twin/paper book is separate and never re-syncs (port the original contract).

## Audit & safety

- `order_events` is append-only; the UI's Activity tab shows the full lineage for any order.
- Kill switch UI in the header (single click, confirm) and via a Telegram command; also flips automatically on: broker auth failure, data staleness > 10 min during market hours, risk-gate bug exception, LLM budget exhausted.
- Daily "execution report" notification: proposals, approvals, fills, slippage vs proposal, risk-gate rejections, cost.
- Dry-run switch for every mode (routes to paper) for testing new engines.

## What ports from the original (and what changes)

| Original | Rebuild |
|---|---|
| `twin._critic` | `Critic` (unchanged logic; deduplicated `_market_regime`, shared `screen_score`) |
| `twin._preflight_pending` | folded into `RiskGate` (event veto, price drift, funding rescale) |
| `twin.execute_pending` clamps | `PaperBroker.fill` with a fill model and transactional cash |
| `twin._windows_for`, `_grace_verdict`, review windows | `ReviewScheduler` — unchanged, plus `dd_normal` actually used, plus applied to **real** fills too |
| bandit / lesson book | kept; now includes `user_decisions` context (declined-by-user as a signal) |
| no execution | the four-mode pipeline above |
