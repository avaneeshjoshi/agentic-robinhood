# 05 — Database Schema & Repository Teardown (`brain/db/`)

SQLAlchemy 2.0 declarative. Default `sqlite:///data_store/brain.db`; `postgresql://` URLs are rewritten to `postgresql+psycopg://`. 24 tables. Engine created at import (`pool_pre_ping=True`, `check_same_thread=False` for sqlite). Sessions: `autoflush=False, expire_on_commit=False` (repository reads ORM attributes after commit).

## Migrations — the main technical debt

`init_db()` = `Base.metadata.create_all()` + `_ensure_columns()`, where `_ADDITIVE_COLUMNS` is a list of **29 hand-written `ALTER TABLE … ADD COLUMN IF NOT EXISTS`** statements (missions.last_seeded_at; the whole twin_trades / twin_trade_reviews expansion). Postgres-only (sqlite gets whatever `create_all` builds), all 29 run in **one transaction inside a bare `try/except: return`** — one failure rolls back all and is swallowed. **Alembic is declared in `requirements.txt` with zero migration files.** Type changes, renames, drops, and index additions are unsupported.

## Tables by domain

### Portfolio snapshots
| Table | Columns | Notes |
|---|---|---|
| `portfolio_snapshots` | id, source(idx), total_value, cash, buying_power, reported_equity, pricing_source, pricing_warning, sync_ok, sync_message, captured_at(idx), created_at | one row per refresh cycle (~120 s) |
| `position_snapshots` | id, snapshot_id(FK cascade, idx), ticker(idx), quantity, avg_cost, current_price, market_value, weight | one row per holding per snapshot; composite idx (ticker, snapshot_id) |

### Research memory
| Table | Columns |
|---|---|
| `theses` | id, ticker(unique), thesis, status(idx), strengthens_json, weakens_json, invalidation, last_decision(idx), updated_at(idx), created_at |
| `watchlist_items` | id, ticker(unique), reason, mode(idx), target_entry, max_allocation_pct, added_at, updated_at(idx) |
| `briefings` | id(str PK), kind(idx), title, summary, bullets_json, actions_json, created_at(idx) |
| `ticker_research` | ticker(PK), action_label(idx), confidence, thesis, bull_case, bear_case, risks, source, refreshed_at(idx) |

### Events & evidence (the universal bus)
| Table | Columns |
|---|---|
| `research_events` | id, ticker(idx), event_type(idx), severity(idx), title, summary, source, created_at(idx); composite (ticker, created_at) |
| `evidence_items` | id, ticker(idx), url(700, idx), title, source, snippet(≤2000), kind(idx: web\|catalyst\|filing), engine(idx), first_seen, last_seen(idx); composite (ticker, url) — "so citations aren't trapped inside each run's JSON" |

### Track record
| Table | Columns |
|---|---|
| `shadow_trades` | id(str), ticker(idx), action, decision_label(idx), conviction, risk_mode(idx), flavor, sector, thesis, source(idx), entry_price, entry_at(idx), entry_signals_json, bench_symbol, bench_entry_price, sector_etf, sector_etf_entry_price, last_price, last_at, bench_last_price, sector_etf_last_price, closed(idx), closed_at, close_reason, user_executed; composite (source, closed) |
| `agent_runs` | id(str), kind(idx), query(≤8000), answer(≤20000), steps_json(≤200000), tools_used(255, comma-joined), model, created_at(idx) |

### Mandate
| Table | Columns |
|---|---|
| `mandates` | single row id='default': statement, horizon, risk, style, favor_json, avoid_json, summary, updated_at |
| `mandate_plan_state` | single row: signature_json `[[ticker, weight_pct], …]`, updated_at — durable drift baseline |

### Eval layer
| Table | Columns |
|---|---|
| `eval_labels` (human) | id, run_id(idx), kind(idx), ticker(idx), verdict(idx), failure_modes_json, note(≤4000), created_at, updated_at |
| `eval_judgements` (machine) | same + score, grounding_json, rationale(≤4000), fix(≤2000), revised, model; run_id **unique** — separate table so judge-vs-human agreement is computable |
| `chat_messages` | id, role(idx), content, created_at(idx) — final text only; traces live in agent_runs |

### Missions & autonomous discovery
| Table | Columns |
|---|---|
| `missions` | id, title, theme, mode, status(idx), created_at, updated_at, last_run_at, last_classified_at, last_seeded_at |
| `mission_candidates` | id, mission_id(FK cascade), ticker, label, conviction, reason, sector, signals_json, first_seen, updated_at; composite (mission_id, ticker) |
| `autonomous_themes` | key(PK), name, status(idx), score(idx), confidence, evidence_json(≤20000), candidates_json(≤60000), source, discovered_at, updated_at, last_seen_at; composite (status, score) |
| `autonomous_strategies` | key(PK), title, status, score, confidence, tactic(idx), horizon, theme_key(idx), theme_name, market_regime(idx), hypothesis, entry_rule, exit_rule, sizing_note, evidence_json, candidates_json, source, timestamps; composite (status, score) |

### Twin / Autopilot
| Table | Columns |
|---|---|
| `twin_fund` | single row: status(""\|running\|paused), inception_at, inception_value, cash, mandate_statement |
| `twin_positions` | ticker, shares, avg_cost, **thesis, horizon, exit_rule** ("what makes the holding a decision, not just a number") |
| `twin_trades` | 33 cols: action, shares, price, decision_price, value, reasoning, conviction, critic_note, preflight_note, tactic, source_theme_key/name, source_strategy_key/name, market_regime, plan_step, depends_on_json, horizon, thesis, exit_rule, review_after_days, bench prices, review_* fields, status(pending\|filled\|canceled), decided_at, filled_at |
| `twin_equity` | at, value, cash, positions_value |
| `twin_trade_reviews` | trade_id, window(1d\|1w\|1m\|3m\|6m), judged, due_at, entry/bench/sector anchors, price/bench_last/sector_last, return_pct, spy_alpha_pct, sector_alpha_pct, drawdown_pct, thesis_state, verdict(monitoring\|intact\|worked\|lagged\|weak\|failed\|executed), note, reviewed_at, theme/strategy/regime attribution |

Schema defects: `*_json Text` columns instead of JSON/JSONB (unqueryable); no FKs `eval_*.run_id → agent_runs.id` or `twin_trade_reviews.trade_id → twin_trades.id`; `tools_used` as a 255-char comma string; all timestamps set in Python.

## `repository.py` (2,127 lines) — the data access layer

Every function has the shape:

```python
def f(...):
    if not _ensure_ready(): return <empty>
    try:
        with db_session() as session: ...
    except Exception:
        return <empty>
```

🔴 **~60 blanket `except Exception: return <empty>` with zero logging.** A schema mismatch, a connection failure, and "no rows" are indistinguishable; `save_agent_run` even returns a valid-looking id when the write failed. `_parse_dt` substitutes `now()` for unparseable timestamps.

### Function inventory
- **Portfolio**: `save_portfolio_snapshot` (no-op without holdings), `latest_portfolio_snapshot(source)`, `portfolio_equity_history(span)` (`1d:1, 1m:31, 3m:93, 6m:186, 1y:366` days).
- **Research state**: `load_research_state` (re-validates Literals), `save_research_state` (upsert-only), `delete_watchlist_item`, `delete_thesis`.
- **Chat**: `save_chat_message`, `recent_chat_messages(80, 168 h)`.
- **Events**: `save_research_event`, **`event_exists_recent(type|types, ticker, within_hours)`** (the universal cooldown primitive), `recent_events(limit, within_hours, event_types)`.
- **Ticker research**: `upsert_ticker_research`, `get_ticker_research`.
- **Shadow**: `save_shadow_trade(s)`, `delete_shadow_trades`, `all_shadow_trades`, `shadow_trade_count`, `open_shadow_tickers(source)`.
- **Agent runs**: `save_agent_run` (truncates: query 8k, answer 20k, steps 200k), `recent_agent_runs(limit, kind)`.
- **Mandate**: `load_mandate`, `save_mandate`, `load_mandate_plan_sig` (None if never set), `save_mandate_plan_sig`.
- **Eval**: `save_eval_label` (upsert by run_id), `eval_labels_by_run`, `eval_summary`, `save_eval_judgement`, `eval_judgements_by_run`, `unjudged_run_ids(limit, kinds=(analyst, rejudge, deep_research))`, `judge_summary` (verdict split, avg score, failure modes, revised count, **judge-vs-human agreement %**), `judgements_for_tickers`.
- **Evidence**: `record_evidence` (dedup on (ticker, url), refreshes last_seen), `evidence_for(ticker, 30)`.
- **Missions**: `save_mission` (full roster replace), `all_missions(status)`, `get_mission`, `set_mission_status`, `delete_mission`.
- **Autonomous**: `upsert_autonomous_theme/strategy`, `autonomous_themes/strategies(status, limit, min_score)`, `autonomous_theme/strategy_feedback()` (group judged windows → tested_count, avg_return, avg_spy_alpha, avg_sector_alpha, win_rate, break_rate).
- **Twin**: fund load/save/update_cash; positions upsert (None intent fields keep existing) / delete / set_intent; `add_twin_trade`, `pending_twin_trades`, `cancel_twin_trades(ids, reason|dict)`, `resize_twin_trade`, `fill_twin_trade`, `recent_twin_trades(60)`; `schedule_twin_reviews(windows)`, `due_twin_reviews`, `save_twin_review_window`, `twin_reviews_for_trades`, `twin_window_policy`, `latest_twin_review`; `add_twin_equity_point`, `twin_equity_curve(400)`, `real_equity_curve`; `reset_twin()` (deletes all five tables); `twin_lesson_book()`; `twin_contextual_bandit()`.

### `twin_contextual_bandit()` — the learning primitive

Built **only from done + judged review windows** (never simulated).

```
reward = (sector_alpha if sector anchor else spy_alpha)
       + {worked:+1, lagged/weak:−1, failed:−4}[verdict]
       + {stronger:+2, weakening:−2, broken:−5}[thesis_state]

arms keyed on 13 contexts: tactic · sector · regime · tactic|sector · tactic|regime · sector|regime
                           + (if attributed) theme · tactic|theme · theme|regime · strategy · strategy|regime · tactic|strategy
confidence = min(0.95, n/(n+4))
stance: n<2 → explore | avg>2 ∧ conf≥0.33 ∧ break<0.34 → lean_in | avg<−3 ∨ break≥0.34 → avoid | avg<−1 → size_down | else neutral
returns {arms, by_key, top(8), bottom(8)}   (top/bottom require count ≥ 2)
```

### `twin_lesson_book()`
Per-tactic and per-sector aggregates (with `best_tactic` per sector by avg sector alpha), the 12 most recent reviews, active themes (score ≥ 45) and strategies (≥ 40) decorated with feedback and a stance word (testing / lean in / cooling / back off / mixed; exploring / scale / cooling / retire / mixed). "Normal drawdowns" = `verdict ∈ {intact, monitor}` or `(return < 0 ∧ sector_alpha ≥ −2 ∧ thesis_state == active)`.

### Other defects
- In-Python aggregation everywhere (`SELECT *` then loop) — fine at hobby scale.
- `judgements_for_tickers` / `latest_twin_review` rely on `ORDER BY created_at DESC` + first-wins instead of window functions.
- `save_mission` deletes and recreates every candidate row on each run.
- Magic truncation limits scattered inline.

## What the rebuild keeps and changes

Keep: the table *vocabulary* (snapshot, position, thesis, watch item, event, evidence, recommendation/shadow trade, agent run, judgement, mission, theme, strategy, fund/position/trade/equity/review) — it is a good domain model. Change: Postgres-only with JSONB and real FKs; Alembic from day one; typed errors + structured logging instead of silent empties; SQL aggregation (or materialized views) for the scorecard and bandit; add `orders`, `fills`, `notifications`, `approvals`, `llm_calls` (cost ledger), `users`/`accounts` (for the SaaS phase). See `11-architecture-blueprint.md`.
