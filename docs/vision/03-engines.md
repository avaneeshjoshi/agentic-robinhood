# 03 — Engines Teardown (`brain/engines/`)

Sixteen engines. All are called from `orchestrator.py`; all persist through `db/repository.py`; all LLM calls go through `llm.py` (see doc 02 for the shared system prompt, `today_line`, and the gather→parse pattern).

## Quick reference

| Engine | LOC | LLM calls / run | Verdict |
|---|---|---|---|
| `monitor.py` | 176 | **0** | Robust. Pure `detect()` + IO wrapper; 9 rules; per-type cooldowns. |
| `memory.py` | 245 | 1 web_research + 1 parse per triggered name (uncapped) | Robust. Four free triggers gate one call; evidence ledger capped at 6. |
| `findings.py` | 101 | 1 | Solid, thin. A ranking *view* over the event stream. |
| `structural_risk.py` | 201 | 1 (+ rule fallback) | Robust concept, broken fallback. |
| `discovery.py` | 98 | 1 | Solid, narrow. Momentum-only; no judge gate. |
| `analyst.py` | 112 | 1 web + 1 parse + 1–3 judge | Robust. Three grounding sources + judge gate. |
| `judge.py` | 228 | 1–3 | Robust; the strongest idea. Self-judging bias. |
| `researcher.py` | 199 | up to 13 | Robust agentic loop. Most expensive path. |
| `deep_research.py` | 178 | ~15–18 total | Robust. Draft → self-critique → judge. |
| `autoresearch.py` | 80 | 0 (delegates) | Robust, small. Triple gating. |
| `missions.py` | 251 | 2 on seed, 1 on classify | Robust. Three cadence tiers. |
| `theme_scout.py` | 189 | **0** | Solid rules engine over 8 *hardcoded* themes. |
| `strategy_discovery.py` | 211 | **0** | Clever, deterministic. `theme:tactic:regime` keys. |
| `evaluation.py` | 286 | **0** | Excellent; most rigorous file. Untested. |
| `briefing.py` | 78 | 1 | Thin, dated. |
| `twin.py` | 1,309 | 1 decide + 1 low-effort per judged review | Most ambitious; layered defense-in-depth. |

---

## `monitor.py` — the always-on zero-LLM event detector

*"The proactive heartbeat the product is built around: cheap deterministic checks over data we already compute every refresh."* `detect()` is pure (testable); `run_monitors()` adds dedup + persistence.

Thresholds: `BIG_DRAWDOWN_PCT=15`, `BIG_GAIN_PCT=30`, `RSI_OVERBOUGHT=70`, `RSI_OVERSOLD=30`, `EARNINGS_SOON_DAYS=7`, `STALE_AFTER_DAYS=21`, cooldowns 12 h (default) / 48 h (earnings) / 168 h (stale).

| Rule | Event | Severity | Condition |
|---|---|---|---|
| 1 | `concentration` | warn | weight > `profile.max_single_position_pct` |
| 2 | `drawdown` | alert | unrealized ≤ −15% |
| 3 | `big_gain` | info | unrealized ≥ +30% |
| 4 | `below_200d` | warn | not above 200d |
| 5 | `overbought` / `oversold` | info | RSI ≥ 70 / 0 < RSI ≤ 30 (RSI 0 = no data, never trips) |
| 6 | `thesis_review` / `thesis_broken` | warn / alert | stored thesis status |
| 7 | `earnings_soon` | warn ≤2 d else info | 0–7 days out |
| 8 | `target_hit` | warn | unheld watch item at/below `target_entry` |
| 9 | `research_stale` | info | unheld active/review thesis ≥ 21 d old (held names are skipped — memory re-judges those) |

Weakness: thresholds don't scale with `profile.appetite`.

---

## `memory.py` — Living Memory (thesis re-underwriting)

Re-judges a held name's **stored thesis against its own invalidation condition** with one gated LLM call, then moves it active → review → broken and appends a dated note to `strengthens`/`weakens` (each capped at last 6) — "so the thesis genuinely *evolves*."

Four free triggers: `trigger_reason()` (≤ −15% from cost, below 200d, RSI ≤ 30); `_earnings_trigger` (today / in N days / just reported); `_news_trigger` (keywords from the **invalidation + strengthens/weakens only**, minus 40 stopwords, substring-matched against last 6 RSS headlines); `_stale_trigger` (≥ 21 d). Cooldown 24 h checked across all three outcome event types.

`_judge()`: `web_research(_revisit_task, max_searches=4)` → on failure degrade to RSS → add catalysts + sentiment blocks → `parse(ThesisVerdict, 1200)`. Verdict prompt core:

```
Re-judge this stored investment thesis against what just changed. Be conservative.
TICKER / STORED THESIS / INVALIDATION CONDITION / STRENGTHENS IT / WEAKENS IT
WHAT TRIGGERED THIS REVIEW: {trigger}
POSITION: unrealized {+N}% from cost.  CURRENT SIGNALS: …  {grounding} {catalysts} {sentiment}
Has the invalidation condition ACTUALLY been met, or is this normal volatility?
- status 'broken' ONLY if the evidence clearly matches the stated invalidation.
- status 'review' if the thesis is at genuine risk and warrants a human look.
- status 'active' if it still holds and this is noise.
Give the matching action label and ONE grounded sentence citing the specific evidence.
```

Persists `agent_runs(kind="rejudge")` + `evidence_items(engine="rejudge")`. Weaknesses: naive substring news match; **no per-cycle cap** (8 triggered names = 8 web-research loops in one pass); silent `None` on parse failure.

---

## `findings.py` — the curated feed

*"Findings is a view, not a second scan."* One LLM pass over: last 30 events (72 h) rendered `[severity] TICKER: title. summary` ("treat them as ground truth"), holdings framing (weight, P&L, stored thesis), mechanical concentration, news for the top-3 holdings, and the top-4 non-held names from a full universe screen ranked by `discovery._screen_score` (private cross-engine import). `parse(FindingsFeed, 2500)` → ≤6 findings of kind opportunity/risk/news/concentration. No error handling on the parse; untested.

---

## `structural_risk.py` — correlated-bet clustering

*"The model does the judgement (which names share a driver); the code does the math (summing the real portfolio weights of each cluster). We never trust the model's arithmetic."* — the single best design principle in the codebase.

`analyze()`: `parse(_ClusterPlan, 1500)` (prompt: group holdings by one underlying driver, "Do NOT compute weights or percentages") → keep only real tickers → `weight_pct = round(Σ weights)` in code → keep clusters with ≥2 names or a single name ≥40% → sort → `concentrated = top ≥ 40%` → headline. On any failure → `_fallback_plan`: six hardcoded rules (long-duration high-beta growth; AI/HPC data-center capex; US large-cap tech beta; defense/aerospace; consumer credit/fintech; crypto/Bitcoin proxy), each with a ticker set and keyword tuple matched by **unanchored substring** against thesis text.

🔴 The fallback's `"ai"` keyword matches "remain", "chair", "detail", "capital"; the ticker set contains `"DRAM"` (not a ticker); overlapping rules double-count so fallback weights can exceed 100%. `maybe_alert()` writes a 24 h-cooldowned `structural_risk` event with empty ticker.

---

## `discovery.py` — idea generation

Funnel: batched screen over the full universe (~660 names, one yfinance download) → `_flavor_ok` (stable: vol ≤ 35; volatile: vol ≥ 45) → sort by `_screen_score` → top 12 → `get_signals_many` for fundamentals → one `parse(DiscoveryResult, 3000)` → top_n → persist watch items → shadow-log each as a `buy` with `source="discovery"` (deduped by `has_open`).

```
_screen_score = ret_3m×1.0 + ret_6m×0.5 + (15 if above_50d else −10) + (15 if above_200d else −10)
              + (10 if 50≤RSI≤70) − (15 if RSI>80)
```

Weaknesses: purely momentum-biased (no value/quality/mean-reversion path); no web grounding; **not judge-gated**; untested.

---

## `analyst.py` — on-demand single-ticker recommendation

Flow: `get_signals` → `web_research(_research_task, max_searches=4, return_sources=True)` (fallback RSS) → `catalysts_prompt` + `sentiment_prompt` → `mandate_prompt` → `parse(TradeTicket, 2500)` → **`judge.gate_ticket`** → persist `agent_runs(kind="analyst")` + evidence → `judge.record` → `shadow.log_recommendation` + `research_state.update_from_ticket`.

The key prompt paragraph (verbatim):

```
GROUNDING DISCIPLINE (you are graded on this): every load-bearing claim — a number, a fact, a named catalyst — must be supported by the evidence above (the live web research, the quantitative signals, or the recent catalysts). If the evidence doesn't support a claim, do not assert it: drop it, or mark it explicitly as an assumption and lower your conviction. Never state a figure, date, or catalyst that isn't in the provided evidence. A thesis resting on uncited assertions is a weak thesis — price it at low conviction.
```

Telling the generator the exact criterion the judge scores on closes the generator↔evaluator loop. Weaknesses: no cooldown/cache on repeated `analyze()`; a DB failure silently loses the judgement.

---

## `judge.py` — LLM-as-judge + the self-repair gate ★

*"You judge the PROCESS, not the market outcome."* Judge prompt (near-verbatim):

```
You are the QUALITY JUDGE for this research engine. Score the reasoning below against the user's own failure taxonomy… regardless of whether the trade eventually wins or loses.
{mandate} INVESTOR PROFILE… THE CALL BEING JUDGED ({kind} on {ticker})… QUANTITATIVE SIGNALS… {evidence_block (≤6000 chars + ≤2000 chars of source titles)}
FAILURE TAXONOMY — score against exactly these modes (return the ids): {evals.taxonomy_prompt()}
Judge rigorously and specifically:
- Identify the load-bearing claims … An unsupported load-bearing claim is weak_grounding; an invented figure is hallucinated_fact; a source that doesn't say what's claimed is source_mismatch.
- A thesis with no concrete invalidation is not_falsifiable. Conviction out of line with the strength of the evidence is overconfident. A call that ignores the profile or mandate is ignored_profile.
- Be calibrated. Reserve a score of 85+ for genuinely rigorous, well-sourced, falsifiable calls. Most decent calls land 60-80. Only flag failure modes that actually apply — do not pad the list.
Return your verdict, the score, the failure-mode ids, the grounding checks, a short rationale, and — if it's not 'good' — the single most important fix.
```

`gate_ticket()` — the agentic loop:

```
a = assess_ticket(...)                       # effort = JUDGE_EFFORT (medium)
if SELF_CRITIQUE and a.verdict == "flawed" and is_load_bearing(a.failure_modes):
    fixed = repair_ticket(ticket, a, ...)    # "do NOT inflate conviction to compensate"
    a2 = assess_ticket(fixed, ...)
    if a2.score >= a.score: return (fixed, a2, revised=True)   # non-regression rule
return (ticket, a, False)
```

Worst case 3 calls. Background sweep (`judge_recent_traces`) reconstructs judgeable blocks from stored traces via `block_from_trace(kind)`. `judge_summary()` computes **judge-vs-human agreement %** where both exist.

Weaknesses: **self-judging bias** — same model, same system prompt as the generator; judge scores never validated against human labels; `deep_research` passes `sources=[]`; every failure is `except: return None`.

---

## `researcher.py` — the agentic multi-step investigation

`MAX_STEPS=12`, `max_tokens=4000`. System prompt = `SYSTEM_PROMPT` + (verbatim):

```
You are operating as a DEEP RESEARCH analyst with tools. This is the heavy, careful pass — act like an analyst who will be graded on being right, not on sounding confident.
Method:
- Start by planning the few specific questions that actually decide this call.
- Investigate them with the tools. Go to PRIMARY SOURCES: read the company's SEC filings (read_sec_filing) and reported financials (get_company_financials), not just news about them. Use web_search for current events, the latest quarter, analyst views, catalysts, and risks.
- Follow leads. If an IPO, deal, or macro/policy event matters, trace its ripple to the names it actually affects (including ones the investor doesn't hold).
- CORROBORATE load-bearing claims across independent sources before you rely on them. When sources DISAGREE, say so explicitly and don't paper over it. Tag each key conclusion with a confidence (high / medium / low) and name what would raise it.
- Be honest about gaps: if the evidence is thin or stale, say that plainly.
Do NOT issue a final buy/sell/size here — a separate synthesis step does that.
End your turn with a written RESEARCH DOSSIER in this shape:
- Questions / Findings (with sources + confidence) / Primary sources (figures) / Disagreements & open questions / Narrative vs fundamentals
```

Tools: server `web_search` + client `get_company_filings`, `get_company_financials` ("Primary-source numbers — prefer these over figures quoted in articles"), `read_sec_filing(form enum)`, `get_stock_signals`, `get_stock_chart`. Kickoff frames the prior thesis as *"test it, don't assume it."* Handles `pause_turn`, container propagation, tool errors as content, and a forced tool-free final call on step exhaustion. Weaknesses: no cost ceiling beyond 12 steps; sources captured only as summaries ("5 results"), not URL lists.

---

## `deep_research.py` — draft → self-critique → judge

`run()`: signals + RSS + 6m chart summary → prior thesis block → `researcher.investigate()` (best-effort) → `parse(DeepResearchDraft, 3500)` (with GROUNDING DISCIPLINE and "**Be willing to conclude the boring or negative answer**") → `parse(DeepResearchCritique, 2000)` ("be your own toughest critic… what is the strongest version of the OPPOSITE call?… **Intellectual honesty over consistency**") → fold into a `TradeTicket` → `judge.gate_ticket` → memory + shadow (`source="deep_research"`) → `agent_runs(kind="deep_research", steps=[{"type":"report", …}])`. Reports `changed = final ≠ draft` — a measure of how often self-critique moves the call.

Weaknesses: the critique sees **only the draft's summary, not the dossier or signals**; draft/critique parses unguarded (an expensive `investigate` is thrown away on schema failure); `sources=[]` to the judge.

---

## `autoresearch.py` — unprompted deep dives

Triggers `{thesis_broken: 3, thesis_review: 2, mission_update(warn only): 1}` from the last 3 h of events; `AUTO_DIVE_COOLDOWN_HOURS=72` per ticker; `AUTO_DIVE_MAX_PER_CYCLE=1`. The completed `deep_dive` event is simultaneously the ping and the cooldown marker. Severity `warn` for buy/add/sell/trim dives, `info` for hold/watch. Gap: a third simultaneous trigger can fall out of the 3 h lookback and never be dived.

---

## `missions.py` — standing theme trackers

Three cost tiers: free deterministic monitoring every cycle; LLM classify at `CLASSIFY_COOLDOWN_HOURS=20`; LLM reseed at `RESEED_COOLDOWN_HOURS=168`. `MAX_ROSTER=15`.

- `_seed_roster` = `web_research` ("find the real, liquid, US-listed stocks that genuinely best fit this theme today… don't restrict yourself to the familiar names from memory… equally, don't reach for obscure names for novelty's sake") → `parse(MissionSeed)`.
- `_classify` → `parse(MissionRoster)` with labels **BUY / WATCH / WAIT ("a good name at the wrong price/time") / REJECT** — "Be willing to REJECT names that have drifted off-theme."
- `run_mission` merges per-field with previous values, preserves `first_seen`, emits `mission_update` (warn on promotion to BUY → that severity is what triggers autoresearch). Roster grows **only** in `reseed_mission`; `_cap_roster` sheds REJECTs (lowest conviction first) then lowest conviction overall.

Weaknesses: `mode="balanced"` accepted but never explained in prompts; **no validation that LLM-proposed tickers exist**; `_hours_since` returns 1e9 on parse failure (fail-open on cost).

---

## `theme_scout.py` — autonomous market radar (no LLM)

8 hardcoded `ThemeRule`s: `ai_compute` (18 names), `ai_power_infra` (15), `defense_space` (16), `fintech_crypto` (12), `software_security` (14), `biotech_platforms` (11), `nuclear_uranium` (12), `quantum_nextgen` (6). Overlaps are deliberate.

```
candidate = 0.7·r1m + 0.55·r3m + 0.25·r6m + (12|−8 above_50d) + (14|−10 above_200d)
          + (8 if 35≤RSI≤72) − (16 if RSI>82) − max(0, vol−80)×0.08 + min(event_hits,4)×5
theme    = 20 + 0.7·avg_1m(top5) + 0.45·avg_3m(top5) + 18·breadth_50 + 14·breadth_200 + 14·pos_3m + 3·min(ev,8)
confidence = clamp(30 + 3·n + 25·breadth_50 + 4·min(ev,6))
feedback  += clamp(2.0·avg_sector_alpha + 0.15·(win_rate−50) − 0.25·break_rate, −15, +15)   # from Twin reviews
status = active if score ≥ 45 else cooling
```

Cadence via the last `theme_scout` agent_run (`THEME_SCOUT_HOURS=6`). Verdict: **it ranks pre-listed themes; it cannot discover an unlisted one** — the name oversells it.

---

## `strategy_discovery.py` — tactic experiments

*"Theme Scout answers 'what areas are alive?' Strategy Discovery answers 'what repeatable tactic should the Twin test there?'"*

`_market_regime()` over SPY+QQQ → `risk_on_overextended` (avg RSI ≥ 72 & both > 200d) | `risk_on` | `risk_off` | `oversold` (avg RSI ≤ 35) | `mixed` | `unknown`. **Duplicated verbatim in `twin.py`.**

| Tactic | Entry filter over theme candidates |
|---|---|
| `pullback_in_uptrend` | above_200d ∧ RSI ≤ 58 ∧ r3m ≥ 0 |
| `momentum_continuation` | above_50d ∧ above_200d ∧ 45 ≤ RSI ≤ 72 ∧ r1m > 0 |
| `valuation_mean_reversion` | above_200d ∧ RSI ≤ 38 |

```
score = 0.55·theme + 0.45·avg_candidate  (−12 momentum in overextended; +5 pullback in mixed/oversold)
      + clamp(2.0·alpha + 0.12·(win−50) − 0.25·break, ±18)   keyed on f"{theme}:{tactic}:{regime}"
status = retired if break_rate ≥ 34 and tested ≥ 2 | active if score ≥ 68 ∧ conf ≥ 45 | exploring if ≥ 45 | cooling
```

Defects: dead `regime`/`candidates` params in `_strategy_template`; scoring before the empty-candidate guard; N+1 feedback queries; all three tactics require `above_200d` (no falling-knife/value path).

---

## `evaluation.py` — the outcome Scorecard (no LLM) ★

*"The numbers are the argument."* `MATURE_DAYS=5`: calls younger than this are reported as **forming** and excluded from the headline win rate/alpha. `_agg` averages alpha **only over `has_benchmark()` trades**. Cuts over matured trades: calibration by conviction bucket (high ≥7 / medium 4–6 / low ≤3), by source engine, by action label, by risk mode. Leaderboard ranked by **alpha, not return**. `_theme_signal` groups by the user's **mission rosters** (falling back to GICS sector — "coarse GICS sector alone is useless on a tech-heavy book"), flags `graded=False` for provisional reads. `_narrative` leads with maturity, then says plainly when *"Conviction is inverted: high-conviction calls aren't beating low-conviction ones."* Duplicates flagged, not dropped.

Weaknesses: 5 days is a noise floor; a theme is "graded" on one matured call; **no statistical significance anywhere**; **no tests**.

---

## `briefing.py` — morning/evening briefings

One `parse(BriefingDraft, 2500)` over holdings (position line + signals + thesis + 2 RSS headlines each, serially) and the last 12 watch items. "Mention if 'do nothing' is the right action." Does **not** use catalysts, sentiment, mandate, structural risk, or events — less informed than `findings`. No error handling; untested.

---

## `twin.py` — Autopilot / the Twin ★★★

*"An autonomous paper fund cloned once from the real account… Fixed capital — to buy anything it must sell something. The user races their real account against it."* It is at once a competence demo, an idea generator, and the labeled-outcome source that trains theme_scout and strategy_discovery.

Constants: `_VALID_TACTICS` = rebalance, risk_reduction, momentum_continuation, pullback_in_uptrend, valuation_mean_reversion, catalyst_trade, long_term_compounder, theme_exposure, defensive_rotation, liquidity_cleanup; `_HYGIENE_TACTICS` = rebalance, risk_reduction, liquidity_cleanup, defensive_rotation (scored "executed", never alpha-judged); `_MIN_ORDER_USD=1`; windows `1d/1w/1m/3m/6m`.

`_TwinMoveDraft` — a narrow 10-field LLM schema because "the rich internal TwinMove schema… makes Anthropic reject the structured-output schema as too complex."

### Lifecycle

**0. Inception** — once, ever. Copies cash and every position at `avg_cost = current_price` ("Inherited from your real book at inception"); no-op if running so a restart never re-clones.

**1. Valuation** — `value()` = cash + Σ shares×price, falling back to avg_cost on quote miss; `_mark_real_portfolio` marks the *real* book with the same quote source for an apples-to-apples race.

**2. Decide** — bail if orders pending; grade due reviews first; build `_candidate_universe(held)` from 7 sources with `setdefault` precedence (watchlist → active theses → mission rosters (non-REJECT) → strategy experiments → autonomous themes → recent events (96 h) → broad screen top 40), truncated to 60; `_signals_block`, `_events_block`, `_policy_memory()` (per-tactic stats "lean in modestly" / "size down", per-sector lesson book, contextual bandit lean-in/avoid contexts), `_market_regime()`; `parse(_TwinDecisionDraft, 2400)`. Decision prompt (verbatim core):

```
You are AUTOPILOT — an autonomous fund manager running a real paper portfolio. You decide the trades and nobody approves them. Your job: pursue the mandate below and beat the user's real account.
FIXED CAPITAL — to buy you must use cash already in the Twin or sell/trim something first. You may not invent deposits, assume extra buying power, or spend money that is not shown. Long-only (no shorts).
{mandate | 'No mandate is set — manage prudently toward steady long-term growth and capital preservation.'}
INVESTOR PROFILE (secondary to the mandate): …
YOUR BOOK: Total $X · cash $Y · N positions  - TICKER: sh, $, (+% since you bought) · thesis; horizon; exit
CANDIDATES you may buy (… choose from this list only; never invented tickers): …
QUANTITATIVE SIGNALS / RECENT CATALYSTS / AUTOPILOT POLICY MEMORY (reviewed paper trades only)
Decide your moves for this cycle. You set the pace — trade as much or as little as warranted, including nothing at all. Size each move in DOLLARS… Treat this as an ORDERED execution plan: put funding/risk-reduction legs before the buys they enable, and use depends_on… Position sizing and concentration are YOUR call — you may concentrate in a high-conviction name beyond the profile's comfort cap if you judge it worth the risk; treat that cap as a preference, not a hard limit…
```

Persists a three-part trace: `decision_context` / `model_draft` / `governor_review`. On LLM failure writes a `twin_decision_attempt` run and **raises** (the docstring claims "best-effort… simply holds" — it doesn't).

**3. Critic (`_critic`)** — deterministic governor. Normalizes tactic (unknown → theme_exposure) and `review_after_days` (from horizon text: compounder → 90, months → 45, swing → 14, catalyst → 5). Hard rejections (written to DB as `canceled` rows with reasons): `usd < $1`; buy of a ticker **not in universe ∪ held** (anti-hallucination); sell of a name not held; tactic with `count ≥ 3 ∧ alpha < −6 ∧ win < 34`; bandit arm `count ≥ 3 ∧ stance == avoid`. Attribution: stamps `source_theme_*`/`source_strategy_*`, adopts the strategy's tactic if the move's is generic. Sizing: sells capped to owned value; tactic `count ≥ 2 ∧ alpha < −2` → ×0.5; bandit `size_down/avoid` → ×0.5; `lean_in ∧ conf ≥ 0.33` → ×1.15. `_bandit_match` backoff: `strategy|regime → tactic|strategy → strategy → theme|regime → tactic|theme → tactic|sector → tactic|regime → sector|regime → tactic` (needs `count ≥ 2`). Fixed capital: `available = cash + Σ accepted sells`; scale all buys by `available/total_buy`, or reject all if `available ≤ 0`. Orders sell → trim → add → buy; assigns `plan_step`; buys get `depends_on` = the funding legs. **No position cap by design.**

**4. Apply** — writes canceled rows first, then pending rows with `usd` (shares 0), `decision_price`, `market_regime`, intent fields; `set_twin_intent` for every move including holds. "Orders are priced at fill, so a transient quote failure never loses a decision."

**5. Fill (`execute_pending`)** — gated on `market_clock.is_market_open()` unless forced. `_current_pending_trades` clusters pending rows by `decided_at` with a 120 s gap and **cancels every batch but the newest**. `_preflight_pending`: cancel buy if a `thesis_broken` event for the ticker appeared in 24 h; cancel buy if price > decision +4% ("not chasing stale entry"); cancel sell if price < decision −8% ("avoiding stale gap-down sale") **unless** tactic is risk_reduction/defensive_rotation; rescale buys to `cash + expected sell proceeds`. Fill loop: sorted by `(plan_step, sell<trim<add<buy)`; price ≤ 0 → leave queued; **buy `shares = min(want, cash/price)`** with weighted-average cost; **sell `shares = min(want, held)`**; SPY anchor captured once; `_schedule_fill_reviews` per fill.

**6. Multi-window review** — `_windows_for(horizon, tactic)`:

| Condition | Windows (judged?) |
|---|---|
| hygiene tactic | 1w (yes, scored "executed") |
| long_term_compounder / horizon mentions core, long, multi, compound, year | 1w(no) 1m(no) 3m(yes) 6m(yes) |
| horizon "swing" | 1d(no) 1w(no) 1m(yes) |
| catalyst_trade / horizon trade, catalyst, day | 1d(no) 1w(yes) |
| default | 1w(no) 1m(yes) 3m(yes) |

At fill, capture SPY + sector-ETF anchors and theme/strategy/regime attribution. `review_windows()`: `sign = −1` for sells; `spy_alpha = sign·(stock − spy)`, `sector_alpha = sign·(stock − sector)`; drawdown since entry; `_assess_thesis` (skipped for 1d/hygiene; `parse(TwinThesisReview, 600, effort="low")`, prompt: *"A drawdown alone is NOT a thesis break… Mark 'broken' ONLY if the exit rule clearly fired"*; fails toward "intact"); `_grace_verdict`: hygiene → executed; broken → failed; weakening → weak; active/stronger → judged ? (sector_alpha > 0 ? worked : lagged) : intact. `dd_normal` is computed, stored, prompted for, and **never used**.

**7. Compare** — `edge_pct = twin_return − real_return`, both books, equity curves since inception, pending summary, market phase, last 200 trades, lessons, decision traces. Note `real.value` uses broker equity while `twin.value` uses the quote map — `marked_value` is exposed but `edge_pct` doesn't use it.

### Twin defects
1. `decide()` docstring contradicts behavior (raises). 2. `dd_normal` dead. 3. `_market_regime` / `_screen_score` duplicated. 4. **No slippage, spread, commissions, liquidity, or partial fills** — the Twin's returns are optimistic vs the real account it races. 5. Cash updated once at loop end — a crash mid-fill leaves positions without decremented cash. 6. `edge_pct` not apples-to-apples. 7. `inception()` resets a `paused` fund. 8. ~15 untuned magic constants. 9. Four concerns in one 1,309-line file.

---

## The 21 prompts (index)

| # | Prompt | File | Output |
|---|---|---|---|
| 1 | Brain system prompt | llm.py | — (cached, all calls) |
| 2 | `today_line()` | llm.py | — (user message) |
| 3 | Deep-research system extension | researcher.py | — (cached) |
| 4 | Research kickoff | researcher.py | dossier prose |
| 5 | Analyst web-research task | analyst.py | brief |
| 6 | Analyst recommendation (+GROUNDING DISCIPLINE) | analyst.py | TradeTicket |
| 7 | Deep-research draft | deep_research.py | DeepResearchDraft |
| 8 | Self-critique | deep_research.py | DeepResearchCritique |
| 9 | Quality judge | judge.py | JudgeAssessment |
| 10 | Repair | judge.py | TradeTicket |
| 11 | Thesis re-underwrite web task | memory.py | brief |
| 12 | Thesis re-judge | memory.py | ThesisVerdict |
| 13 | Mission seed web task | missions.py | brief |
| 14 | Mission seed structure | missions.py | MissionSeed |
| 15 | Mission classify | missions.py | MissionRoster |
| 16 | Findings curation | findings.py | FindingsFeed |
| 17 | Structural clustering | structural_risk.py | _ClusterPlan |
| 18 | Discovery ranking | discovery.py | DiscoveryResult |
| 19 | Briefing | briefing.py | BriefingDraft |
| 20 | Autopilot decision | twin.py | _TwinDecisionDraft |
| 21 | Twin thesis review | twin.py | TwinThesisReview |

## Recurring patterns worth copying

1. **Gather then structure** (web search can't share a request with structured output).
2. **Model judges, code computes.**
3. **Gate everything on cost** — "a calm book spends nothing."
4. **Degrade, never fail** (web → RSS; researcher → signals; clustering → rules; judge → ship original; quote → avg_cost/leave queued; thesis check → intact).
5. **Persist the reasoning, not just the answer** (`agent_runs` + `evidence_items`).
6. **Events as the universal bus.**
7. **Pure core + IO wrapper** for testability.
