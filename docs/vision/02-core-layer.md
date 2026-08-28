# 02 — Core Layer Teardown

Files: `cli.py`, `brain/config.py`, `brain/models.py`, `brain/llm.py`, `brain/agent.py`, `brain/orchestrator.py`, `brain/mandate.py`, `brain/shadow.py`, `brain/research_state.py`, `brain/evals.py`, `brain/profile_store.py`, `brain/profile_learning.py`, `scripts/reset_state.py`. (`db/` and `portfolio/` are in docs 05 and 04.)

Depth legend: **Robust** = complete, defended, iterated · **Thin** = works, minimal · **Stub/Broken** = incomplete or non-functional.

---

## `cli.py` (94 lines) — Thin, partly broken

Argparse front-end over `brain.orchestrator`. Subcommands: `analyze <ticker>`, `discover --flavor {any,stable,volatile} --n 5`, `digest`, `briefing --kind`, `score`, `refresh`, `db-init`, `db-stats`, `portfolio`, `memory`, `ask "<message>"`. The `ask` renderer consumes the streaming trace and prints `🔧 tool(input)`, `↳ result[:120]`, `💭 note`, then the answer.

Defects:
- 🔴 `digest` calls `brain.daily_digest()` — **not defined** in `orchestrator.py`.
- 🔴 `db-stats` references `db_models.QuoteSnapshot`, `PriceAlertRecord`, `AgentRun` — **none exist** (the real name is `AgentRunRecord`; the others were removed). Raises `AttributeError`.
- No error handling, exit codes, or logging config.

---

## `brain/config.py` (104 lines) — Robust

Pure env→constants module (`python-dotenv`), with excellent inline rationale for each knob. Side effects on import: creates `data_store/` and `data_store/digests/`. Full variable table is in `01-original-overview.md`.

Defects: `.env.example` is referenced by the README and by `require_api_key()`'s error text but **is gitignored and does not exist**; malformed `int()` env values crash at import; `MODEL`/`EFFORT` are snapshotted at import in `llm.py` so runtime changes don't apply.

---

## `brain/models.py` (609 lines) — Robust, unusually thoughtful

Pydantic models that **double as the JSON-schema contract for LLM structured outputs** — every `Field(description=…)` is prompt surface.

### Risk / personality
- `RiskAppetite` = conservative | balanced | aggressive; `Horizon` = short | medium | long.
- `FeedbackEvent(ticker, accepted, beta, sector, flavor, dividend_yield, at)` — stores the stock's *characteristics* with each accept/reject so patterns can be inferred, not just symbols.
- `RiskProfile(appetite=balanced, horizon=medium, max_single_position_pct=15, prefers_dividends, avoid_sectors, favor_sectors, notes, feedback_events, investor_signature, learning_log, updated_at)` with `.describe()` → the compact NL block fed to every prompt (includes last 10 liked/passed tickers).

### Portfolio
- `Holding(ticker, quantity, avg_cost, current_price)` → `market_value`, `unrealized_pct`.
- `Portfolio(holdings, cash, buying_power, reported_equity, pricing_source, pricing_warning, source, sync_ok, sync_message, as_of)`; `total_value` prefers `reported_equity` when > 0; **`weights()` returns percentages 0–100, not fractions** (a documented footgun).

### Recommendations
- `Action = buy|sell|trim|add|hold|watch`; `DecisionLabel = BUY CANDIDATE | WATCHLIST | WAIT FOR PULLBACK | HOLD | TRIM | EXIT REVIEW | REJECT | DO NOTHING`.
- `TradeTicket(ticker, action, conviction 1–10, thesis, catalyst, risks, suggested_size_pct, fits_profile_because)` + computed `decision_label` (buy/add→BUY CANDIDATE, watch→WATCHLIST, hold→HOLD, trim→TRIM, sell→EXIT REVIEW, else DO NOTHING — so WAIT FOR PULLBACK and REJECT are unreachable from a ticket).
- `StockIdea`/`DiscoveryResult`, `Finding(kind: opportunity|risk|news|concentration)`/`FindingsFeed`, `RiskCluster`/`StructuralRisk` (weight_pct "computed deterministically from actual portfolio weights, never trusted to the model's arithmetic"), `ChartPoint`/`StockChart`.

### Research memory
- `Thesis(ticker, thesis, status: active|review|broken|archived, strengthens[], weakens[], invalidation, last_decision, updated_at)`.
- `ThesisVerdict(status, decision_label, reason)` with rules embedded in field descriptions ("'broken' only if the invalidation condition is clearly met").
- `WatchItem(ticker, reason, mode: stable|balanced|volatile, target_entry, max_allocation_pct, …)`, `Briefing`, `ResearchState(watchlist, theses{}, briefings)`.

### `ShadowTrade` (the evaluation substrate)
Fields: id, ticker, action, decision_label, conviction, thesis, entry_price, entry_at, source (which engine), risk_mode, flavor, sector, entry_signals dict, **bench_symbol="SPY", bench_entry_price, sector_etf, sector_etf_entry_price**, last_price/last_at, bench_last_price, sector_etf_last_price, closed/closed_at/close_reason, user_executed.
Methods: `_sign()` = −1 for sell/trim ("a sell call is right when the name falls"), `return_pct()` = sign × stock change, `alpha_pct()` = sign × (stock − bench), **0.0 when no anchor** so legacy rows can't pollute stats; `has_benchmark()`; `sector_alpha_pct()`.

### Missions, deep research, mandate, eval, Twin
- `Mission(id, title verbatim, theme, mode, status, candidates[], last_run_at, last_classified_at, last_seeded_at)`; `MissionCandidate(ticker, label: BUY|WATCH|WAIT|REJECT, conviction, reason, sector, signals, first_seen)`.
- `DeepResearchDraft(plan, bull_case, bear_case, evidence, thesis, catalyst, risks, action, conviction, suggested_size_pct)`; `DeepResearchCritique(critique[], holds_up, final_action, final_conviction, note)`.
- `Mandate(statement, horizon, risk, style, favor[], avoid[], summary)`; `.describe()` renders `USER'S MANDATE (their stated goal — align every call to it): …`; `MandateExtract`, `PlanMove`, `PlanReview(alignment, moves 1–3, note)`.
- `GroundingCheck(claim, supported, note)`; `JudgeAssessment(verdict good|mixed|flawed, score 0–100 "reserve 85+", failure_modes[], grounding[], rationale, fix)`.
- `TwinMove(ticker, action buy|add|trim|sell|hold, usd, reasoning, conviction, tactic, thesis, horizon, exit_rule, source_theme_key/name, source_strategy_key/name, plan_step, depends_on[], review_after_days)`; `TwinDecision(summary, moves)`; `TwinThesisReview(state active|weakening|broken|stronger, drawdown_normal, reason)` — *"a long-term name that's merely down… is 'active', not a failure."*

---

## `brain/llm.py` (169 lines) — Robust; the strongest file in the codebase

### The frozen system prompt (verbatim)

```
You are the analytical core of a personal stock-research engine for a single retail investor.

Your job is rigor, not hype. You are explicitly NOT a price predictor and NOT a hype machine. You synthesize grounded data (price/trend signals, fundamentals, recent news) into clear, honest, decision-useful reasoning tailored to one specific person's risk personality.

Operating principles:
- Ground every claim in the data provided. If the data is thin, say so and lower your conviction. Never invent numbers, catalysts, or headlines.
- Match recommendations to the user's stated risk profile. "Stable" means low beta, large cap, durable cash flows, often dividends. "Volatile" means high beta, smaller/mid cap, momentum or story-driven names. Calibrate accordingly.
- Be specific and falsifiable. A thesis names a concrete driver and what would break it. "Strong company" is not a thesis.
- Conviction is earned. Reserve 8-10 for genuinely compelling, multi-signal setups. Most ideas are 4-6.
- Surface risk plainly. The user is trusting you with real money decisions; downside and uncertainty come first, not as fine print.
- You do not place trades. You produce reasoning and recommendations the user executes themselves. Write for someone who will act on your words.
- Use a consistent answer shape for portfolio/investing questions:
  1. Start with a one-line decision or state-of-portfolio read.
  2. Then give 2-4 bullets of evidence from the provided data.
  3. Then give concrete actions or non-actions.
  4. End with a data caveat only when the provided data has a real limitation.
- Avoid long finance essays. Prefer crisp, scannable, decision-useful output.

Tone: direct, concise, opinionated but honest about uncertainty. No filler, no disclaimers theater.
```

Wrapped as `[{"type":"text","text":SYSTEM_PROMPT,"cache_control":{"type":"ephemeral"}}]`.

### Functions
| Function | Behavior |
|---|---|
| `today_line()` | `"Today's date is {%A, %B %-d, %Y}. Treat this as the present: ground every 'recent', 'this week', 'latest', or 'now' judgement — and every web search query — in this date. Do NOT assume an earlier year; your training data predates today."` Injected into the **user message**, never the system block, so the cache stays byte-stable. |
| `client()` | Lazy singleton `anthropic.Anthropic(api_key, timeout=120.0, max_retries=1)`. |
| `ask(prompt, max_tokens=4000, effort)` | Free-form text. `thinking=adaptive`, `output_config={"effort"}`. |
| `parse(prompt, schema, max_tokens=4000, effort)` | `messages.parse(output_format=schema)`; raises `RuntimeError` if `parsed_output is None`. |
| `WEB_SEARCH_TOOL` | `{"type":"web_search_20260209","name":"web_search","max_uses":6,"blocked_domains":["zacks.com","fool.com","investorplace.com","stocktwits.com"]}` — a small blocklist; trust steering is by prompt tiers, not an allowlist. |
| `web_research(task, max_searches=5, max_tokens=2500, max_steps=6, return_sources=False)` | The **gather** half of gather→parse. Loops on `stop_reason == "pause_turn"`, **carries `container_id`** across requests (the search-result filtering runs in a server-side code-execution container; omitting it 400s), keeps the last text as the brief, and extracts sources two ways: inline `citations` on text blocks **and** raw hits inside `web_search_tool_result.content`. Dedupes by URL. |

Gaps: no retry/backoff beyond `max_retries=1`; no token/cost accounting; polymorphic return type on `web_research`.

---

## `brain/agent.py` (587 lines) — Robust; the real "agentic" surface

A tool-use loop where the model drives its own research. `MAX_STEPS = 12`, `max_tokens = 4000`.

### The agent system prompt (appended to `SYSTEM_PROMPT`, verbatim)

```
You are operating in AGENT MODE with tools. Work like a real analyst:
- Investigate before concluding. Pull the data you need — don't guess.
- When asked about the portfolio, read it, then dig into the specific holdings that matter.
- When hunting for ideas, screen the market, then pull signals/news on the standouts.
- When price action matters, fetch a chart and use it in the answer.
- Save only researched, genuinely useful ideas to watchlist memory; do not save every ticker mentioned.
- When you reach a real, decision-useful call on a ticker (a buy/add/hold/trim/sell with a conviction level), log it once with log_recommendation so the brain builds an honest, measurable track record. Never log passing mentions or hypotheticals.
- Chain tools as needed. Be efficient: don't pull data you won't use.
- End with a clear, decision-useful answer grounded in what the tools returned.
- Format final answers as:
  **Read:** one sentence.
  **Evidence:** 2-4 bullets.
  **Action:** 1-4 concrete actions or non-actions.
  **Watch:** optional, only if there is a specific trigger/level/event.
- You proactively surface findings the user didn't explicitly ask for but should know
  (a concentration risk, a holding breaking down, a standout opportunity).

MANAGING THE BRAIN (control tools — you can change the user's tracked state):
- You can add/remove watchlist names, set entry-price alerts, drop stored theses, and start /
  pause / resume / archive / delete strategy missions. This lets the user run the app by talking.
- When the user states their overall investing goal ("I want long-term holds I can keep a year+,
  nothing too speculative"), persist it with set_mandate so the whole system aligns to it, then
  confirm what you understood. Only on an explicit goal statement — never infer one from a
  passing question or a single trade idea.
- Only mutate state on a CLEAR, EXPLICIT user request ("add NVDA to my watchlist", "stop tracking
  the defense mission", "drop the RKLB thesis"). Never delete or remove something on your own
  initiative or as a side effect of analysis.
- For removing a mission, prefer 'archive' (keeps the history) unless the user clearly wants it
  gone for good — then 'delete'. When unsure which name they mean, ask rather than guess.
- After any change, state plainly what you did (the tool already returns a confirmation).

USING WEB SEARCH:
- The quantitative tools (signals, screen, chart) give you numbers; web_search gives you the
  live story. Use web_search whenever the answer depends on what's happening *now* — recent
  news, an earnings reaction, a policy/Fed/Trump comment, an IPO and its ripple to related
  names, a sector catalyst, or anything time-sensitive. Don't answer from memory when current
  information would change the call; search first.
- Weight sources by trust, and say which tier a claim rests on:
  TIER 1 (treat as ground truth): SEC filings / company IR / official releases.
  TIER 2 (reputable reporting): Reuters, Bloomberg, WSJ, FT, CNBC, Barron's, AP.
  TIER 3 (sentiment/color only, never a standalone fact): blogs, forums, Seeking Alpha, smaller outlets.
- Never treat a single headline or post as fact. Corroborate, and cite the source for any
  claim that moves your conclusion so it can be checked.
```

### Tools (16 client + server web_search)

| Read | Write / control |
|---|---|
| `get_stock_signals`, `get_stock_news`, `screen_market(flavor, limit≤15)`, `get_my_portfolio`, `get_my_profile`, `get_research_memory`, `get_recent_activity(limit≤40, 168 h)`, `get_stock_chart` (renders an inline chart in chat) | `save_watchlist_item` (idempotent, never clobbers on re-add), `log_recommendation` (refuses to double-log via `shadow.has_open(source="assistant")`), `remove_watchlist_item`, `set_watch_target`, `drop_thesis`, `start_mission`, `manage_mission(pause/resume/archive/delete)` (title resolved exact-then-substring), `set_mandate` |

`_execute` wraps every handler: errors become `"Tool error (name): e"` fed back to the model rather than crashing the loop.

### `run_stream(message, history)` flow
1. Append user turn with `today_line()`.
2. System = `[AGENT_SYSTEM (cached)]` + **a second uncached block** with `mandate_prompt()` if set ("Keep this mandate front of mind…"), split so the mandate can change without busting the cache.
3. Up to 12 × `messages.create(tools=TOOLS, container=container_id)`; carry `container_id`; narrate server-side `web_search` as `tool`/`tool_result` events; `continue` on `pause_turn`.
4. On non-tool stop: persist to `agent_runs(kind="chat", steps, tools_used, model)` and yield `answer`.
5. Else execute tools, yield `tool` / `chart` / `tool_result` (`out[:240]`), append results as one user message.
6. Step-limit exhaustion yields `"(Reached the step limit — here's what I found above.)"`.

Defects: `get_stock_chart` fetches twice per call; unbounded `history`; the mandate block is never cached; the loop itself is untested.

---

## `brain/orchestrator.py` (839 lines) — Robust; the god-module

Single import surface for web/CLI. Groups:

- **Profile**: `get_profile`, `update_profile`, `feedback(ticker, accepted)` → `refresh_learning()` (re-derives the investor signature from holdings).
- **Portfolio/charts**: `portfolio`, `refresh_live_state()` (clears portfolio + price caches with `include_signals=False` + news), `stock_chart` (RH first, yfinance fallback, **anchored to the warm quote, never a forced re-fetch** — forcing quotes per poll rate-limited to 0 and froze the line), `portfolio_chart` (anchored to `pf.total_value` so the chart tip can't disagree with the header), `_anchor_to_now`.
- **Engines**: `analyze`, `cached_analysis` (reverse-maps `action_label`→action from `ticker_research`), `discover` (excludes held), `deep_research`, `feed()` (3 h TTL + `(holdings, latest event id)` signature), `structural_risk()` (6 h TTL + `(ticker, rounded weight)` signature).
- **Quarantined signals**: `ingest_sentiment()` (ApeWisdom map, fires `social_buzz` when Δmentions ≥ 50% and ≥ 25 mentions, 12 h dedup), `ingest_catalysts()` (Finnhub; per-name cooldown; cross-name headline dedup; **`_about_another_company()`** — regex `\(([A-Z]{1,5})\)` extracts parenthesized tickers from a headline and skips items about a different company; folds into `evidence_items`).
- **Events**: `run_monitors`, `revisit_memory`, `today_events()` (stable two-pass sort: newest first, then alert < warn < info; batched judgement attachment via `_EVENT_TRACE_KIND`).
- **Eval**: `scoreboard`, `scorecard`, `agent_runs`, `evidence`, `eval_taxonomy/traces/summary`, `save_eval_label`, `judge_summary`, `judge_recent_traces()` (sweep of `unjudged_run_ids`).
- **Mandate**: `set_mandate` (invalidates cached review), `mandate_review()` cached on `(mandate.updated_at, weight signature)`; `_mandate_drift(old, new, threshold)` → material if any weight moved ≥ threshold points or a new/exited position ≥ `max(5, threshold//2)`; `run_mandate_review()` (once per 7 d via `mandate_plan` event cooldown); `run_mandate_drift()` (first run only baselines; within cooldown it **absorbs** the drift so the same shift can't re-fire; durable baseline in `mandate_plan_state`).
- **Twin**: `twin_start`, `twin_compare`, `twin_execute_pending`, `twin_snapshot`, `twin_review_windows`, `run_twin_decision()` (skips if last `twin_decision` run < `TWIN_DECIDE_HOURS`; emits "Autopilot rebalanced" event), `twin_decide_now`, `twin_reset`, `run_theme_scout`, `run_strategy_discovery`.
- **Missions/chat**: CRUD + `run_due_missions`, `run_autoresearch` (hard-gated on `AUTO_DEEP_RESEARCH`), `chat`, `chat_stream`, `chat_history(80)`, `save_chat_message`.

Defects: `daily_digest` missing (CLI); five module-level mutable caches not thread-safe; silent `except: pass` in universe helpers; convoluted `run_twin_decision` scan.

---

## `brain/mandate.py` (104 lines) — Robust, small

- `set_mandate(text)` → `llm.parse(_extract_prompt, MandateExtract, 800)`; on **any** exception falls back to `MandateExtract(summary=text[:200])` — "never lose the user's words to a parse failure."
- `mandate_prompt()` → `Mandate.describe()` or `""`; injected into analyst, discovery, judge, Twin, chat.
- `_holdings_block(pf)` → deterministic per-holding line (`weight%, ±% from cost, above/below 200d, RSI, 3m, P/E`) via batched `get_signals_many`.
- `review(pf, profile)` → `PlanReview` with the prompt: *"You are the user's investing advisor. Judge their portfolio against their stated mandate… INVESTOR PROFILE (secondary to the mandate)… give 1-3 concrete moves… if the book already fits well, say so and give zero or one move."*

Defects: both try/excepts swallow real API failures indistinguishably from "no mandate."

---

## `brain/shadow.py` (247 lines) — Robust; the honesty layer

*"Benchmark-relative scoring of a past recommendation is impossible to reconstruct after the fact, so the anchor price must be stored the moment the call is made."*

- `_migrate_jsonl_once()` — one-shot legacy `shadow_ledger.jsonl` → DB, renames to `.migrated`.
- `log_recommendation(ticket, source, profile, flavor, signals)` — captures entry price, **SPY price, sector ETF + its price**, 12-key signal snapshot, `risk_mode`; keeps duplicates on purpose ("a re-call at a new price is real information") and fires a cooldowned `shadow_dup` event for later `reconcile_duplicate(trade_id, "replace"|"keep")`.
- `mark_to_market(refresh)` — one quote per unique symbol across all open trades' tickers/benchmarks; only updates non-zero prices.
- `scoreboard()` → `{count, win_rate, avg_return_pct, best, worst, trades[]}`.

Defects: nothing ever sets `closed`; win rate includes hold/watch calls; `set_user_executed`/`reconcile_duplicate` load the whole table; `_MIGRATED` unlocked.

---

## `brain/research_state.py` (212 lines) — Robust with a smell

Dual-write: every save writes **both** `research_state.json` and the DB; `load_state()` prefers the DB only if it has content, else the JSON (and back-fills). Hence `reset_state.py` must delete the JSON too.

- `update_from_ticket(ticket)` — upserts `Thesis` (status `review` for sell/trim else `active`, `invalidation = ticket.risks`), `ticker_research`, a `ticker_research` event, and (for buy/add/watch) a watch item whose `reason = _watch_reason(ticket)` = the **forward catalyst** (≤200 chars), not the thesis — "keeps the three research tables meaning three different things."
- `add_briefing` caps at 50; `upsert_watch_item` caps watchlist at 100; `_mode_from_size`: ≤3% → volatile, ≥8% → stable.
- `remove_watch_item` / `remove_thesis` must delete DB rows explicitly because `save_research_state` is upsert-only (a bug that was clearly learned the hard way; guarded by `tests/test_agent_controls.py`).
- `summarize_for_prompt(20)` renders the memory block the agent's `get_research_memory` tool returns.

---

## `brain/evals.py` (~100 lines) — Small, clean, the best single idea

12 seed failure modes: `hallucinated_fact, weak_grounding, source_mismatch, missed_catalyst, not_falsifiable, overconfident, ignored_profile, stale_data, contradiction, tool_misuse, vague, generic`. `VERDICTS = good|mixed|flawed`.

`LOAD_BEARING = {hallucinated_fact, weak_grounding, source_mismatch, not_falsifiable, overconfident, ignored_profile, contradiction}` — the modes severe enough to trigger a self-repair. `normalize_tag()` (`"_".join(lower split)[:40]`) lets free-typed human tags accrete into stable ids. `taxonomy_prompt()` renders `- id: Label — desc` lines that are embedded **verbatim** in the judge prompt, so human labeling UI, LLM judge, and self-repair gate share one rubric.

---

## `brain/profile_store.py` (33 lines) — Thin

JSON file at `data_store/profile.json`; creates a default `RiskProfile()` on first run. No DB backing, no locking; a corrupt file raises into every prompt builder.

## `brain/profile_learning.py` (123 lines) — Robust, deterministic, no LLM

- `record_feedback` → `FeedbackEvent` with beta/sector/flavor/yield (capped 100) → `_infer_from_feedback` (needs ≥3 accepts): mean accepted beta ≥1.3 → aggressive, ≤0.9 → conservative; top-2 accepted sectors → `favor_sectors`; sector rejected ≥2× and never accepted → `avoid_sectors`; ≥60% of accepts yielding ≥1.5% → `prefers_dividends`. Every change writes a dated reason into `learning_log` (cap 25).
- `learn_from_holdings` → weight-weighted beta, dividend-payer weight, sector concentration → `investor_signature` string (e.g. "actual book runs ~1.34 beta (aggressive/high-beta); concentrated in Technology…").

Defects: 🔴 **one-way ratchets** — sectors only ever appended, `prefers_dividends` never reverts, appetite never returns to balanced; a sector can land in both favor and avoid; all thresholds hardcoded; per-holding `get_signals` in a loop.

---

## `scripts/reset_state.py` (72 lines) — Small, careful

Dry-run by default; `--yes` drops **every reflected table** (orphans included), recreates the schema, and deletes the five `data_store` JSON files + digests. Password-redacted URL printout. Defects: deletes `holdings_manual.json` irrecoverably for manual-mode users; reflection drops any table in a shared Postgres; no backup option; untested.

---

## Cross-cutting: hardcoded values worth knowing

`MAX_STEPS=12`, `max_tokens=4000`, 240-char tool-result truncation, 168 h activity window, `_FEED_TTL=3h`, `_RISK_TTL=6h`, `_MIN_EVENTS=3`, `_DIV_THRESHOLD=1.5`, beta 1.3/0.9, dividend fraction 0.6, caps 25/100/100/50, `_mode_from_size` ≤3/≥8, screen score (`3m×1.0 + 6m×0.5`, ±15 MAs, +10 RSI 50–70, −15 RSI>80), flavor vol bands 35/45, judge "85+" guidance. **No TODO/FIXME comments exist anywhere** — dead code is of the "references a removed feature" kind.
