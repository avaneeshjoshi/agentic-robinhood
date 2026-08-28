# 07 — Tests & Quality Teardown (`tests/`)

23 files, 2,876 lines, **stdlib `unittest`** (no pytest, no conftest, no fixtures library, no coverage config, no CI). Run: `.venv/bin/python -m unittest discover -s tests`.

## Fixture pattern

Fifteen files set a temp SQLite DB **before importing `brain`** because `config.py` reads env at import:

```python
_TMP = tempfile.NamedTemporaryFile(suffix=".db", delete=False); _TMP.close()
os.environ["DATABASE_URL"] = f"sqlite:///{_TMP.name}"
from brain.db import repository as repo  # noqa: E402
```

Consequences, which the authors know and work around: **whichever module imports `brain` first wins, and the whole suite shares one SQLite file** — tests are order-coupled and non-parallelizable (unique ticker prefixes, `reset_twin()` in setUp, intentionally-empty `tearDownModule` "so we don't pull the rug out from under later modules"). Monkeypatching of module globals (`memory.get_news = lambda …`) restored in `tearDown`/`importlib.reload`; `unittest.mock` used in exactly one file. Two files capture the real function at import time (`_REAL_DR_RUN`, `_REAL_INVESTIGATE`) to prevent cross-module patch leakage — unusually careful.

**The dominant assertion style is call counting**: "this path made 0 / 1 / 3 LLM calls." For an LLM-spending system that is exactly the right discipline.

## Per-file coverage

| File | Lines | What it proves |
|---|---|---|
| `test_sentiment.py` | 40 | prompt formatter; empty → `""`; never raises |
| `test_mandate.py` | 48 | `is_set`/`describe`; save/load round-trip (extract/review not tested) |
| `test_market_clock.py` | 55 | open/closed/weekend/holiday/next_open — **all dates hardcoded to 2026** |
| `test_agent_controls.py` | 57 | regression for the upsert-only bug: removed watch items/theses stay removed across reload |
| `test_chat_home.py` | 58 | chat persistence order; `set_mandate` tool registered and dispatched (the one `mock.patch` user) |
| `test_catalysts.py` | 59 | dated headlines; `latest_fresh` window; error swallowing |
| `test_evals.py` | 60 | taxonomy, `normalize_tag`, `pretty`, label upsert semantics |
| `test_events_repo.py` | 63 | save/recent/dedup window (incl. 0-hour edge); `today_events` ranking (assertion is conditional — can vacuously pass) |
| `test_autoresearch.py` | 93 | trigger priority, non-triggers ignored, 72 h cooldown, per-cycle cap |
| `test_memory_triggers.py` | 105 | thesis term extraction, news/earnings/stale triggers; judge degrades to RSS on search failure |
| `test_theme_scout.py` | 107 | **end-to-end**: fake screen → theme persisted → flows into Twin candidate universe → critic attributes `source_theme_key` → review window → `autonomous_theme_feedback` reports it. The most valuable test. |
| `test_researcher.py` | 108 | scripted fake Anthropic client: EDGAR tool → `pause_turn` → resume; **container id carried forward** |
| `test_deep_research.py` | 110 | final call comes from the critique pass; four side effects (thesis, shadow, audit, `tools_used` exact string — brittle) |
| `test_edgar.py` | 110 | fully mocked HTTP: CIK, filings URL construction, XBRL newest-first, `$400.00B`, Risk-Factors anchoring, graceful failure. Most thorough single-module test. |
| `test_structural_risk.py` | 110 | weights computed by code not model; single-name clusters dropped; alert cooldown; **LLM failure falls back to deterministic clusterer** |
| `test_strategy_discovery.py` | 111 | experiments persist; feed Autopilot; critic attributes `source_strategy_key` |
| `test_memory.py` | 118 | trigger reasons; active→broken with event; cooldown blocks a second judgement; unheld skipped |
| `test_mandate_drift.py` | 128 | pure drift detector (7 cases with human-readable reasons); flow: baseline-only first run, fires once, **absorbs drift within cooldown** |
| `test_missions.py` | 143 | seed→classify→persist; web failure degrades; BUY promotion event; classify cooldown; additive reseed preserving `first_seen` |
| `test_monitor.py` | 162 | 12 pure tests over `detect()`: every rule, RSI-0 guard, event-dict contract, held names skipped for stale |
| `test_judge.py` | 212 | `StubLLM` context manager; good=1 call; flawed load-bearing → 3 calls and the repair ships; soft flaw doesn't gate; **worse revision discarded**; flags off → 0 calls; trace reconstruction; persistence + agreement % |
| `test_shadow.py` | 236 | SPY+XLK anchors captured and round-tripped; alpha math; **trim inverts sign**; legacy JSONL rows load with alpha 0; scorecard keys; matured-only headline; mission supersedes sector for theme attribution |
| `test_twin.py` | 583 | ~40 tests: once-only inception; mark-to-market; clone hygiene; fills clamped to cash/shares with weighted cost; edge math; critic rejects ungrounded/unowned, scales to cash, no position cap, orders funding legs, **bandit halves size in a bad context**; plan_step ordering; stale batch cancel; 5 preflight cases (chase, thesis break, gap-down, urgent gap-down allowed, resize); `_windows_for`; grace verdicts; lessons; bandit arms. Effectively the Autopilot spec. |

## Coverage estimate

- `brain/engines` + `brain/data`: **~70–80%** of the logic that matters, and aimed at the right things (gates, fallbacks, sign conventions, ordering, idempotence).
- `brain/` top level: ~40%.
- `web/app.py`: **0%** — no `TestClient`, no route tests, no tests for the pure `_refresh_health` / `_worker_health` / `_brain_timeline` / `_autopilot_ops` state machines.
- Frontend: **0%** — 2,513 lines of JS including cache/TTL logic, SSE parsing, SVG math, stream merging.

### Untested modules
`findings.py`, `discovery.py`, `analyst.py`, `briefing.py`, **`evaluation.py`** (pure arithmetic — the easiest and most valuable thing to test), `llm.py`, `agent.py`'s loop (only `set_mandate` touched), `prices.py`, `news.py`, `universe.py`, `robinhood_charts.py`, both `portfolio/` sources, `profile_store.py`, `profile_learning.py`, `scripts/reset_state.py` (irreversible, untested).

### Structural fragilities
- Shared-DB order coupling; cannot run in parallel.
- 2026-hardcoded market-clock dates need annual maintenance.
- `test_twin.test_off_hours_does_not_fill` is conditional on wall-clock time.
- `test_deep_research` asserts an exact `tools_used` string.
- No error-path tests for HTTP; no timeout-path tests for `_run_brain_step`.

## What the rebuild should adopt

- Keep the **call-count discipline** as a first-class fixture (`assert_llm_calls(n)`), the scripted fake LLM client, and the pure-core/IO-wrapper split.
- Move to pytest with per-test DB isolation (transactional fixtures or `testcontainers` Postgres), `pytest-xdist`, coverage gates in CI.
- Add: API contract tests (schemathesis against the OpenAPI spec), frontend unit tests (vitest) + Playwright smoke, property tests for the risk gate and paper-fill math, a golden-file harness for prompts (so a prompt change is a visible diff), and an evaluation harness that replays stored traces through the judge to detect drift.
