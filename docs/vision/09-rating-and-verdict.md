# 09 — Rating & Verdict

## Overall

| As a… | Score | Why |
|---|---|---|
| **Research prototype** | **7 / 10** | The LLM engineering, evaluation layer, and Autopilot design are unusually thoughtful and mostly tested. Several ideas (entry-time benchmark anchoring, model-judges/code-computes, judge gate with non-regression, maturity-gated scorecard, fixed-capital enforcement) are better than most commercial "AI trading" products. |
| **Product you could rely on** | **3 / 10** | No auth, no migrations, no cost accounting, silent data-layer failures, free/unofficial data with a ToS problem, paper fills that ignore slippage, a frontend with XSS and 0% tests. It proves the brain can reason; it does not prove it has edge or that it can be operated safely. |

## Sub-scores

| Area | Score | Evidence |
|---|---|---|
| Architecture | **8** | Two decoupled loops; pipeline-as-data; events as bus; narrow data interfaces; gate discipline (TTL + signature + DB cooldown). Minus: god-module orchestrator, process-local caches, `while True` loops instead of a scheduler. |
| LLM engineering | **9** | Cache-stable system prompt + date anchoring in the message; gather→parse around a real API constraint; `container_id` carry; structured outputs everywhere; narrow-schema workaround; deterministic arithmetic kept out of the model; "GROUNDING DISCIPLINE (you are graded on this)". Minus: no token/cost ledger; same model judges itself. |
| Evaluation / trust layer | **8** | One taxonomy shared by human labels, LLM judge, and self-repair; judge-vs-human agreement %; process eval (judge) vs outcome eval (scorecard) separated; maturity bar; alpha only over anchored trades; inverted-calibration sentence. Minus: no significance testing; `MATURE_DAYS=5`; scorecard never feeds back into prompts; `evaluation.py` untested. |
| Autopilot / Twin design | **8** | LLM → critic → preflight → fill clamps → horizon-matched multi-window sector-relative review → policy memory + contextual bandit → next prompt. Rejections written as canceled rows with reasons. Minus: no slippage/spread/commissions/liquidity; `edge_pct` not apples-to-apples; `dd_normal` dead; regime/score duplication; docstring lies about best-effort. |
| Data quality | **4** | yfinance (unofficial, flaky), undated RSS, hardcoded 503-ticker universe, holidays through 2027, dividend-yield ×100 heuristic, unbounded caches. EDGAR integration is the bright spot. |
| Broker integration | **3** | Unofficial `robin_stocks` with password + TOTP secret in env, session pickled locally, against ToS, breaks on app updates, no re-auth on expiry. Read-only discipline is real. |
| Execution realism | **3** | Paper only; fills at last quote; unlimited fractional shares; no partial fills; cash updated non-transactionally. |
| Security | **1** | Zero authentication; unauthenticated LLM spend and destructive endpoints; XSS in the frontend; raw exception strings to the browser. |
| Persistence & migrations | **4** | Good domain vocabulary and indexes; but Alembic with no migrations, 29 hand-written ALTERs, JSON-as-Text, missing FKs, ~60 silent excepts with no logging. |
| Frontend engineering | **5** | Clever chart pinning, caching, adaptive polling, honest empty states — in 2,500 lines of global-scope innerHTML with XSS, leaks, no tests, no a11y. |
| Frontend design | **7** | Restrained palette, tabular nums, unboxed sections, responsive to 560 px; matches its own design spec. `var(--card)` undefined; low-contrast metadata; dead CSS. |
| Tests | **6** | Strong on engines (call-count discipline, fallback paths, sign conventions, an end-to-end theme→trade→review→feedback test); 0% on web and frontend; order-coupled shared DB. |
| Ops / observability | **3** | Heartbeats and a timeline UI are good; but no logging in the data layer, no metrics, no cost dashboard, thread leak on step timeout, silently-failing briefing loop, no multi-worker safety. |
| Documentation | **8** | README, vision, Autopilot contract, roadmap, UI vision, logo notes; docstrings that record the specific incident a design fixes. Minus: `.env.example` missing; naming drift (Brain/Signal, twin/Autopilot, shadow/Scorecard). |

## What is genuinely great — keep these in the rebuild

1. **Entry-time benchmark anchoring** (`shadow.py`): SPY + sector-ETF prices captured the moment a call is made. Cannot be retrofitted.
2. **Model judges, code computes** (`structural_risk.py`, Twin critic, evaluation): the model clusters/proposes; code sums weights, enforces capital, computes every number.
3. **The judge gate with a non-regression rule** (`judge.py`): flawed on a load-bearing mode → repair once → keep only if re-score ≥ original.
4. **One failure taxonomy** driving human labels, the LLM judge prompt, the self-repair gate, and judge-vs-human agreement.
5. **Maturity-gated scorecard** that refuses to show a win rate on unmatured calls and flags thin samples.
6. **Fixed capital enforced four times** (prompt, critic rescale, preflight rescale, fill clamp) and **grounded-universe rejection** (a ticker not in the list cannot be bought).
7. **`_windows_for`**: judging horizon matched to stated intent, with unjudged monitoring windows; **`_grace_verdict`**: a drawdown alone can never produce "failed".
8. **Sector-relative alpha** as the scoring metric; **sign inversion** so sells are graded on decline avoided.
9. **Cache-stable system prompt + `today_line()` in the user message**; **gather→parse** two-step; `container_id` carry on `pause_turn`.
10. **Two decoupled loops** (fast data / slow LLM) with per-step named timeouts and a heartbeat-driven staleness banner.
11. **Pipeline as data** (`_BRAIN_STEPS`) rendering a pending/running/done/failed timeline with "what it actually did" summaries.
12. **Gate discipline** — "a calm book spends nothing" — TTL + content signature + DB cooldown, with the display event doubling as the rate-limit marker.
13. **Degrade, never fail** — web → RSS; researcher → signals; clustering → rules; judge → original; quote → leave queued; thesis check → intact.
14. **Persist the reasoning** — `agent_runs` + `evidence_items` on every engine; the Twin's three-part `decision_context / model_draft / governor_review` trace; rejected moves stored as canceled rows with reasons.
15. **Living memory**: evidence ledger (strengthens/weakens, capped) that evolves; news triggers from the invalidation clause rather than the thesis prose; "broken ONLY if the evidence clearly matches the stated invalidation."
16. **Mission label set** with WAIT ("a good name at the wrong price/time"); roster grows in exactly one place; REJECT-first eviction.
17. **Three-tier source trust** (filings/IR → Reuters/Bloomberg/WSJ → blogs/forums) and "only mutate state on an explicit request" in the agent prompt.
18. **EDGAR**: Risk-Factors/MD&A anchoring, latest-filed-per-period XBRL de-dup, TTLs matched to artifact volatility.
19. **`withLiveTip`/`paintChart`**: chart tip pinned to the same numbers as the header; per-span TTL + prefetch; never wipe a working chart.
20. **Self-explaining empty states** (`history_note`), adaptive 3 s polling only while a step runs, first-load backlog suppression, state-keyed onboarding.
21. **Drift absorption** in mandate re-planning (a shift inside the cooldown is baselined so it can't re-fire).
22. **`_about_another_company()`** — parenthesized-ticker regex to skip off-target Finnhub headlines.

## Consolidated defect list (severity-ranked)

| # | Sev | Area | Defect | Where |
|---|---|---|---|---|
| 1 | 🔴 | Security | No authentication on any of 37 routes; unauthenticated LLM spend; unauthenticated destructive `twin/reset`, `DELETE missions` | `web/app.py` |
| 2 | 🔴 | Security | `esc()` escapes only `& < >` but is used inside attributes/onclick with LLM-generated URLs/titles → XSS; several sites skip `esc` | `web/static/app.js:11`, `ticketHTML`, `showAnalysisModal`, `renderHoldings` |
| 3 | 🔴 | Broker | Unofficial `robin_stocks` with password + TOTP secret; against ToS; session pickled; `_logged_in` never resets | `brain/portfolio/robinhood.py` |
| 4 | 🔴 | Persistence | ~60 `except Exception: return <empty>` with zero logging; writes fail invisibly (`save_agent_run` returns an id anyway) | `brain/db/repository.py` |
| 5 | 🔴 | Persistence | Alembic declared, no migrations; 29 hardcoded ALTERs, Postgres-only, one silently-swallowed transaction | `brain/db/session.py` |
| 6 | 🔴 | Ops | `asyncio.wait_for` + `to_thread` cannot kill a stalled thread → executor exhaustion after repeated timeouts | `web/app.py:_run_brain_step` |
| 7 | 🔴 | Execution | Paper fills at last quote: no slippage, spread, commissions, liquidity, partial fills → `edge_pct` is optimistic vs the real account it races | `brain/engines/twin.py` |
| 8 | 🔴 | Execution | `compare()` uses broker equity for real value but quote-map for twin value; `marked_value` computed but `edge_pct` ignores it | `twin.py:compare` |
| 9 | 🔴 | Learning | Profile ratchets are one-way: sectors only appended, `prefers_dividends` never reverts, appetite never returns to balanced | `brain/profile_learning.py` |
| 10 | 🔴 | CLI | `digest` calls nonexistent `brain.daily_digest`; `db-stats` references removed models (`QuoteSnapshot`, `PriceAlertRecord`, `AgentRun`) | `cli.py` |
| 11 | 🔴 | Engines | `_fallback_plan` uses unanchored substring match; keyword `"ai"` matches "remain/chair/capital"; `"DRAM"` not a ticker; overlapping rules double-count | `structural_risk.py` |
| 12 | 🔴 | Data | Dividend yield `dy*100 if 0<dy<0.05` turns a 0.04% yield into 4% | `brain/data/prices.py` |
| 13 | 🟠 | Cost | No token/cost accounting anywhere; `researcher` up to 13 × 4k tokens at high effort, triggerable unattended; `memory` has no per-cycle cap | `llm.py`, `researcher.py`, `memory.py` |
| 14 | 🟠 | Engines | Judge is the same model + system prompt as the generator (self-judging bias); judge scores never validated against human labels | `judge.py` |
| 15 | 🟠 | Engines | Deep-research critique sees only the draft summary, not the dossier/signals; `sources=[]` passed to the judge | `deep_research.py` |
| 16 | 🟠 | Engines | `twin.decide()` docstring claims best-effort but raises `RuntimeError` | `twin.py` |
| 17 | 🟠 | Engines | `_grace_verdict` ignores `dd_normal` (computed, stored, prompted, unused) | `twin.py` |
| 18 | 🟠 | Engines | `_market_regime` duplicated verbatim; three variants of `_screen_score`; `findings` imports a private symbol | `twin.py`, `strategy_discovery.py`, `discovery.py`, `findings.py` |
| 19 | 🟠 | Engines | Twin cash updated once at loop end — crash mid-fill leaves positions without decremented cash | `twin.py:execute_pending` |
| 20 | 🟠 | Engines | `inception()` calls `reset_twin()` — destroys a `paused` fund | `twin.py` |
| 21 | 🟠 | Engines | Theme scout / strategy discovery cannot discover anything outside 8 hardcoded themes and 3 tactics; all three tactics require `above_200d` | `theme_scout.py`, `strategy_discovery.py` |
| 22 | 🟠 | Engines | Mission tickers from the LLM are never validated to exist; `mode="balanced"` undefined in prompts; `_hours_since` fails open | `missions.py` |
| 23 | 🟠 | Engines | Discovery is momentum-only, un-judged, no web grounding | `discovery.py` |
| 24 | 🟠 | Engines | `evaluation.py`: no significance testing; theme "graded" on 1 call; `MATURE_DAYS=5`; scorecard never feeds back into prompts; untested | `evaluation.py` |
| 25 | 🟠 | Data | `catalysts._cache` key ignores `days` (7-day vs 14-day window race) | `catalysts.py` |
| 26 | 🟠 | Data | Holidays hardcoded through 2027; silent degradation after | `market_clock.py` |
| 27 | 🟠 | Data | `robinhood_charts.py` has no error handling around `rh.*` calls | `robinhood_charts.py` |
| 28 | 🟠 | Data | EDGAR: unbounded multi-MB cache; failed CIK fetch caches an empty map for 24 h | `edgar.py` |
| 29 | 🟠 | Web | Every error is `{"error"}` at HTTP 200; `api()` never checks `res.ok`; `init_database()` failure swallowed at startup | `web/app.py`, `app.js` |
| 30 | 🟠 | Web | `_briefing_loop`: bare `except: pass`, local-time `"HH:MM"` string compare, no logging | `web/app.py` |
| 31 | 🟠 | Web | Deprecated `@app.on_event`; process-local state breaks under `--workers>1`; `AUTO_REFRESH_SECONDS<=0` silently disables the brain loop too | `web/app.py` |
| 32 | 🟠 | Web | No input validation (`top_n`, `message`, ticker, kind, mode, flavor); raw exception strings to browser | `web/app.py` |
| 33 | 🟠 | Frontend | `analyze()`/`loadScore()`/`loadState()` unguarded; `busy()` without `finally`; `CHART_STORE`/`EPHEMERAL` leak; no `visibilitychange` | `app.js` |
| 34 | 🟠 | Frontend | Accessibility: no tab roles/aria-live/labels; modal without focus trap/Escape; onclick on non-focusable spans; `--faint` ≈ 2.6:1 | `index.html`, `app.js`, `style.css` |
| 35 | 🟡 | Frontend | `var(--card)` used ~20× but never defined; 10–15% dead CSS; 3 green/red pairs; no dark mode | `style.css`, `app.js` |
| 36 | 🟡 | Core | Five module-level caches not thread-safe; `research_state` dual-writes JSON + DB on every mutation; upsert-only save needs companion deletes | `orchestrator.py`, `research_state.py` |
| 37 | 🟡 | Core | `agent.py`: `get_stock_chart` fetched twice; unbounded history; mandate block uncached; loop untested | `agent.py` |
| 38 | 🟡 | Core | `shadow.py`: nothing sets `closed`; win rate includes hold/watch; O(n) table loads; unlocked migration flag | `shadow.py` |
| 39 | 🟡 | Core | `Portfolio.weights()` returns 0–100 percentages (footgun); `WAIT FOR PULLBACK`/`REJECT` unreachable from a ticket | `models.py` |
| 40 | 🟡 | Repo | `.env.example` gitignored and missing though README/config point to it; `requests` and `pyotp` undeclared | `.gitignore`, `requirements.txt` |
| 41 | 🟡 | Data | Static `sp500.json`; `lru_cache`d extras need a restart; ambiguous symbols; no liquidity filter | `universe.py` |
| 42 | 🟡 | Data | RSS headlines undated; empty results cached 15 min; ambiguous-ticker Google fallback | `news.py` |
| 43 | 🟡 | Tests | Shared SQLite across modules (order-coupled); 2026-hardcoded dates; conditional/vacuous assertions; exact `tools_used` string | `tests/` |
| 44 | 🟡 | Scripts | `reset_state.py` deletes manual holdings irrecoverably; reflection drops any table in a shared DB; untested | `scripts/reset_state.py` |
| 45 | 🟡 | Engines | `autoresearch`: a third simultaneous trigger can fall out of the 3 h lookback and never be dived; `AUTO_DEEP_RESEARCH` not checked in-module | `autoresearch.py` |
| 46 | 🟡 | Engines | ~40 untuned magic coefficients across theme_scout / strategy_discovery / twin critic / bandit | various |
| 47 | 🟡 | Engines | `briefing.py` ignores catalysts/sentiment/mandate/events; serial RSS per holding | `briefing.py` |

Every item above is tracked in `improvements-tracker.md` with the phase it is addressed in.

## The verdict in three sentences

This is the best-designed open-source "AI investing brain" teardown target you could hope for: its authors understood that the hard part is grounding, evaluation, and cost discipline, and they built real machinery for all three. It is also unmistakably a single-user prototype with prototype-grade data, security, execution, and operations. The rebuild's job is to keep the brain's ideas nearly intact, replace the foundations underneath them, and add the one thing the original deliberately withheld — a trustworthy path from recommendation to execution.
