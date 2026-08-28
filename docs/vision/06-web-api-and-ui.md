# 06 — Web API & Frontend Teardown

## `web/app.py` (797 lines) — FastAPI

A thin HTTP veneer over `brain.orchestrator` plus three background loops (see doc 01) and a static mount. `FastAPI(title="Signal Research Engine")` — no routers, no middleware, no dependencies, no auth.

### Request models
Plain Pydantic with **no validators or bounds**: `ChatBody(message, history=[])`, `AnalyzeBody(ticker, refresh)`, `DiscoverBody(flavor="any", top_n=5)`, `FeedbackBody`, `HoldingsBody(holdings: list[dict], cash)`, `BriefingBody(kind)`, `WatchTargetBody`, `MissionBody(title, mode)`, `MissionStatusBody`, `DeepResearchBody`, `ReconcileBody(trade_id, mode)`, `EvalLabelBody`, `MandateBody`. `top_n=100000` and a megabyte `message` are accepted.

### Route inventory (37 API routes + `/` + `/static`)

| Method | Path | Calls | Notes |
|---|---|---|---|
| GET | `/api/state` | portfolio + profile + research_state | the shared `_state()` payload |
| POST | `/api/refresh` | `refresh_live_state` | |
| POST | `/api/profile` | `update_profile` | body = `RiskProfile` domain model directly |
| POST | `/api/holdings` | `manual.save_portfolio` | `{"error"}` at 200 if source ≠ manual |
| POST | `/api/chat` | `chat` | persists both turns |
| POST | `/api/chat/stream` | `chat_stream` | **SSE** `text/event-stream`, `data: {json}\n\n`, ends with `{"type":"done"}`; errors in-band |
| GET | `/api/chat/history` | `chat_history` | `limit` 1–200 |
| POST | `/api/analyze` | `cached_analysis` or `analyze` | attaches `sources = evidence(ticker, 12)` |
| POST | `/api/deep_research` | `deep_research` | LLM-heavy |
| POST | `/api/discover` | `discover` | LLM |
| GET | `/api/feed` | `feed` | LLM (cached) |
| GET | `/api/events` | `today_events` | `limit` 1–200 |
| POST | `/api/briefing` | `create_briefing` | LLM |
| POST | `/api/watch/target` | `set_watch_target` | |
| GET | `/api/chart/{ticker}` | `portfolio_chart` if ticker ∈ {portfolio,total,account} else `stock_chart` | `span` is the **only regex-validated param in the API** |
| GET | `/api/scoreboard` · `/api/scorecard` · `/api/structural_risk` · `/api/agent_runs` · `/api/evals` | respective brain fns | `refresh=true` on structural_risk spends tokens |
| POST | `/api/shadow/reconcile` · `/api/evals/label` | | |
| GET/POST | `/api/mandate` | get + cached review / `set_mandate` | |
| POST | `/api/mandate/review` | `mandate_review(force=True)` | LLM, unauthenticated |
| GET | `/api/twin` · `/api/twin/ops` | `twin_compare` + `_autopilot_ops` | |
| POST | `/api/twin/start` · `/api/twin/decide` · `/api/twin/reset` | | **reset is destructive and unauthenticated** |
| GET/POST/DELETE | `/api/missions[/{id}[/run\|/status]]` | | DELETE returns `{"ok":true}` even for nonexistent ids |
| POST | `/api/feedback` · `/api/learn` | | |

### Notable helpers
- `_refresh_health()` — stale if the fast loop hasn't succeeded in `3 × AUTO_REFRESH_SECONDS`; drives the UI banner.
- `_worker_health()` / `_brain_timeline()` — classifies the brain loop (starting / running / working: `<step>` / stale) and renders all 16 steps as pending / running / ok / failed with durations, results, and `from_prev_cycle` flags ("the scend.ai-style timeline").
- `_step_summary()` — turns a step's return value into a sentence (`"3 items: NVDA, AMD, …"`, `"skipped (gated / nothing to do)"`).
- `_autopilot_ops()` — derives an operational narrative: decision status (waiting for fill / due now / scheduled), a four-way **`history_note`** explaining *why* History is empty, catch-up state. Unusually good product thinking.

### Security findings (severity order)
1. **No authentication on any route.** Anyone reaching the port reads the full portfolio and chat history, and can set the mandate, save holdings, start/reset Autopilot, delete missions.
2. **Unauthenticated LLM spend**: chat, chat/stream, deep_research, discover, mandate/review, analyze?refresh, twin/decide, missions create, structural_risk?refresh — no quota, no rate limit, no prompt size cap.
3. **Destructive endpoints** (`twin/reset`, `DELETE missions`) guarded only by a browser `confirm()`.
4. No CORS policy (browser same-origin default; JSON bodies force preflight — mitigated by accident).
5. Secrets: correct at rest (env + gitignore), but RH creds are plaintext env and a session token is pickled locally.
6. Raw exception strings shipped to the browser via worker/refresh health.
7. No input validation on tickers, ids, statements, `top_n`.

### Error handling
Every error is `{"error": "..."}` with **HTTP 200**; not a single `HTTPException`. `app.js`'s `api()` never checks `res.ok`. `brain.init_database()` failure at startup is swallowed. `_briefing_loop` is wrapped in bare `except: pass` with no logging; it compares `"HH:MM"` strings in **server local time** while everything else is UTC. `@app.on_event("startup")` is the deprecated hook. In-memory `_REFRESH`/`_BRAIN` dicts break under `--workers > 1`.

---

## Frontend (`web/static/`) — no framework, no build step

`index.html` (247 lines): one page, inline SVG logo (the "folded vector" mark from the design notes, animated on load), header with `#totVal`, a stale banner, an 8-tab nav (Home · Portfolio · Activity · Memory · Scorecard · Evals · Autopilot · Settings — DOM ids still `shadow`/`profile`), 8 mostly-empty `<section class="panel">` shells filled by JS, a modal, a toast. Assets cache-busted with `?v=72`.

`app.js` (2,513 lines): global-scope script, ~20 functions re-exported to `window` for inline `onclick`. **State model: fetch → assign a global → `renderX()` rebuilds `innerHTML` from scratch.** No diffing; `CHAT_BUSY` is a manual lock so a streaming answer isn't wiped by a re-render.

### Chart system — hand-rolled SVG (no library)
`svgPath()` (index-spaced, autoscaled, no axes), `chartBlock()` (hero: big price, delta, dashed baseline, crosshair + tooltip, touch support), `sparkSvg()` (80×26 per-holding), `dualLine()` (You vs Autopilot, timestamp-normalized), `chartHTML()` (in-chat 520×96). Three different green/red hex pairs across them.

**The best frontend idea — `withLiveTip` / `paintChart`:** the chart's rightmost point is pinned to the price in `STATE` (the same numbers the header and holdings list render) so the tip can never disagree with the header. Per-span TTLs (1d 20 s → 1y 15 min), `prefetchChartSet()` warms all six spans, refresh repaints the cached body with a fresh tip and a pulsing dot, and a fetch failure **never wipes a working chart**. Leak: `CHART_STORE` keyed by `Math.random()` is never cleaned.

### Screens
| Screen | What it does |
|---|---|
| **Home** | One time-sorted stream merging persisted chat, chat-worthy pings (judgements or warn/alert), the last 6 briefings, and session ephemerals. Bot messages carry a collapsible research trace. An **opener** shows book value, concentration callout, unread count, and up to 4 findings ("On my radar right now"). **Onboarding** (money → goal → plan) keyed off real state, with an inline mandate textarea whose payoff is the first plan card. Composer chips: Plan / Ideas / Brief. `sendChat` streams via `fetch` + `ReadableStream` (POST, so not `EventSource`), renders tool/chart/note/answer events live, collapses the trace on completion, keeps the last 12 turns as model context. Scroll pinning. |
| **Portfolio** | Two-column: hero chart (span control, `← Portfolio` reset, `.linked` pulsing dot when a holding drives it) + **structural-risk lens map** below (overlapping factor lenses, not pie slices; tickers in ≥2 lenses flagged as "doubled-up names"; self-heals with `?refresh=true`). Right rail: holdings with sparklines and a hover-revealed Analyze button; an SVG **donut** built from stroke-dasharray with cash as residual and selection state shared with the rows and legend. Manual-holdings editor when `source === manual`. |
| **Activity** | Terminal-style log; `actKind` classifies judgement vs signal; filters; deep-links a thesis event to its re-judge run by **nearest timestamp within 10 min** (no FK); `judge good 87 →` chips open the auto-judge's read. |
| **Memory** | Missions composer + cards (roster sorted BUY<WATCH<WAIT<REJECT); **Dossier** master/detail for theses (thesis headline, "Tracking for" reason, amber "Breaks if", Supports/Pressures ledger, price-alert input, metadata); recent deep dives reconstructed from two storage formats. |
| **Scorecard** | **Maturity honesty**: with 0 matured calls it shows "None yet" and deliberately no win rate ("don't dress up noise"); `sc-locked` placeholders until calls clear the 5-day bar; `thin` rows (n<3) dimmed. Statline → narrative → calibration + by-engine → Beating/Lagging SPY → Themes that are working (mission-grouped, expandable) → every-call ledger with `repeat` tags and a `replace older` reconcile button. |
| **Evals** | Auto-judge suite (avg score, verdict chips, self-revised count, **agrees with you X%**) beside Your labels; filterable trace list; per-trace body + evidence + sources + judge block (grounding checks with green/red dots, suggested fix) + a label form whose **taxonomy grows from what the user types**. |
| **Autopilot** | 11 sections: pre-launch card, You-vs-Autopilot hero, status tiles, ops tiles, the **16-step brain timeline** (expandable rows), the decision trace (Model draft vs Governor, final ordered plan with `depends_on`), six lessons panels (tactics, sectors, policy + bandit arms, strategy experiments, themes, reviews), controls (run a cycle / reset), dual equity curve, side-by-side books with diverged-from-you diff and health badges, day-grouped history with preflight badges parsed from note text. Polls every 3 s **only while a step is running and the tab is active**. |
| **Settings** | Learned signature + last 6 adjustments; a plain form (appetite, horizon, max position %, dividends, favor/avoid sectors, notes); Learn/Save. The least-developed screen — none of the roadmap's "control plane" exists. |

Notifications: `PING_SEEN` in localStorage (first load marks the backlog as seen); unread badges; opt-in browser notifications (max 3, alert/warn only, skipped on first load). Boot is staggered (250 ms / 800 ms); a global 60 s poll refetches everything regardless of tab; `#hash` deep-links a tab on load but tabs never write the hash.

### Frontend defects
- 🔴 **XSS**: `esc()` escapes only `& < >`, yet is used inside attributes (`href="${esc(url)}"`, `onclick="showJudge('${esc(id)}')"`, `title="${esc(note)}"`) carrying LLM-generated URLs/titles and broker tickers; several sites skip `esc` entirely (`ticketHTML`, `showAnalysisModal`, `renderHoldings`).
- `api()` never checks `res.ok`; `analyze()`, `loadScore()`, `loadState()` unguarded — a failed boot leaves "Waking Signal up…" forever; `busy()` without `finally`.
- `CHART_STORE` and `EPHEMERAL` grow unbounded; no `visibilitychange` handling.
- Accessibility: no tab roles, no `aria-live`, no labels, modal has no focus trap / Escape / `role="dialog"`, dozens of `onclick` on non-focusable spans, hover-only affordances, `--faint` contrast ≈ 2.6:1.
- `mdLite` is a 5-line markdown subset (no links, code, nested lists).

## `style.css` (1,061 lines)

Tokens match the roadmap's palette almost exactly: `--bg:#fbfbfa --txt:#17191c --muted:#686f77 --faint:#9aa0a6 --green:#10a348 --red:#d9432f --amber:#b7791f --blue:#2563eb --brain:#6e56cf`, `tabular-nums` on every numeric column, custom `cubic-bezier` logo intro with `prefers-reduced-motion`, unboxed sections with hairlines, the violet left-border motif unifying agent voice, chat pinning via `.stream > :first-child{margin-top:auto}`, 8 breakpoints (980→560) — genuinely responsive. Defects: `var(--card)` used ~20× but **never defined** (Evals/Autopilot cards render flat); ~10–15% dead CSS (`.wl-*`, `.feed`, `.mandate-*`, `.pf-band`, `.cards`); ~60 raw hex literals outside the token system; no dark mode; `.dossier` grid has no mobile breakpoint.

## What the rebuild keeps

The information architecture (Today/Home, Portfolio, Activity, Memory, Scorecard, Evals, Autopilot, Settings), the live-tip chart pinning, per-span TTL + prefetch + never-wipe, maturity honesty, self-explaining empty states, the timeline-as-data, adaptive polling, first-load backlog suppression, state-keyed onboarding, the restrained palette. Everything else moves to Next.js + TypeScript + TanStack Query + lightweight-charts + shadcn with generated API types (doc 11).
