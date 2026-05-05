# Market Data Backend — Code Review

**Date:** 2026-05-05
**Scope:** `backend/app/market/` (8 modules, ~350 LOC) and `backend/tests/market/` (6 test files, 73 tests)
**Reviewer:** Claude Code

---

## Test Results

```
73 passed, 0 failed, 0 errors — 1.02s
```

All 73 tests pass. Ruff linting passes with zero violations.

### Coverage

| Module | Stmts | Miss | Cover | Missing Lines |
|--------|-------|------|-------|---------------|
| `__init__.py` | 6 | 0 | 100% | — |
| `models.py` | 26 | 0 | 100% | — |
| `cache.py` | 39 | 0 | 100% | — |
| `interface.py` | 13 | 0 | 100% | — |
| `seed_prices.py` | 8 | 0 | 100% | — |
| `factory.py` | 15 | 0 | 100% | — |
| `simulator.py` | 139 | 3 | 98% | 149, 268–269 |
| `massive_client.py` | 67 | 4 | 94% | 85–87, 125 |
| `stream.py` | 36 | 24 | 33% | 26–48, 62–87 |
| **TOTAL** | **349** | **31** | **91%** | |

---

## Issues Found

### Bug 1 — `cache.py`: `remove()` does not increment the version counter

**File:** `backend/app/market/cache.py`, line 59–63  
**Severity:** Medium

```python
def remove(self, ticker: str) -> None:
    with self._lock:
        self._prices.pop(ticker, None)
        # BUG: self._version += 1 is missing
```

The SSE streaming loop in `stream.py` uses `price_cache.version` as a change sentinel. It only sends an event when the version changes. When a ticker is removed from the watchlist, the cache state changes but the version is not bumped. The SSE stream will therefore continue including the removed ticker in the next event payload until a price update on any *other* ticker coincidentally increments the version.

The design doc (`MARKET_INTERFACE.md`) shows `remove()` *without* incrementing the version, so this may be intentional, but the consequence is that watchlist removal has a latency equal to the next price update (up to 15 seconds in Massive mode). Adding `self._version += 1` inside `remove()` would make removal propagate immediately.

---

### Bug 2 — `stream.py`: Module-level router object

**File:** `backend/app/market/stream.py`, line 17  
**Severity:** Low (latent)

```python
# Module level — created once when the module is imported
router = APIRouter(prefix="/api/stream", tags=["streaming"])

def create_stream_router(price_cache: PriceCache) -> APIRouter:
    @router.get("/prices")           # Registers on the shared module-level router
    async def stream_prices(...):
        ...
    return router                    # Returns the same object every time
```

The function is named `create_stream_router`, implying it creates a fresh router. It does not — it mutates and returns the single module-level instance. If called more than once (e.g., in tests or if the lifespan hook runs twice), the route is registered twice on the same router object, leading to duplicate routes and unpredictable FastAPI behavior.

The current application startup calls this exactly once, so there is no runtime impact today. But it is a design trap for future callers and test authors.

**Fix:** Move `APIRouter(...)` inside the factory function body.

---

### Warning — `conftest.py`: Deprecated `asyncio.DefaultEventLoopPolicy`

**File:** `backend/tests/conftest.py`, line 10  
**Severity:** Low (generates noise now; will break in Python 3.16)

```python
import asyncio
return asyncio.DefaultEventLoopPolicy()
```

`asyncio.DefaultEventLoopPolicy` was deprecated in Python 3.12 and is scheduled for removal in Python 3.16. The project already runs on Python 3.14.4, producing 73 deprecation warnings per test run (one per test, because `event_loop_policy` fires for every async test).

This fixture has no effect in the current configuration — `pytest-asyncio` in `auto` mode manages the event loop independently. The entire `conftest.py` file can simply be deleted.

---

### Gap — `stream.py`: No tests for the SSE endpoint

**File:** `backend/tests/market/` — no `test_stream.py`  
**Severity:** Low (for now; grows as complexity does)

`stream.py` has 33% coverage. Lines 26–48 (the FastAPI route handler) and 62–87 (the `_generate_events` async generator) are entirely untested. Key behaviors with no test coverage:

- The `retry: 1000\n\n` preamble is always yielded first
- Version-based change detection skips sending when nothing has changed
- Client disconnect terminates the loop
- `CancelledError` is handled gracefully
- Empty cache produces no `data:` events

These are testable with `pytest-asyncio` and a mock `Request` object. E2E tests will exercise this path, but unit tests here would catch regressions much earlier.

---

### Minor — `cache.py`: `version` property reads without the lock

**File:** `backend/app/market/cache.py`, line 65–67  
**Severity:** Very low (CPython-safe in practice)

```python
@property
def version(self) -> int:
    return self._version   # No lock acquired
```

All other read methods on `PriceCache` hold `self._lock`. The `version` property is the sole exception. Under CPython, reading a Python `int` is effectively atomic due to the GIL, so this is safe in practice. Under alternative Python runtimes (PyPy, CPython 3.13+ free-threaded mode with `--disable-gil`) this could race.

The design doc (`MARKET_DATA_DESIGN.md`) shows this property holding the lock: `with self._lock: return self._version`. Aligning the implementation to the design doc costs nothing and removes the inconsistency.

---

## Coverage Notes (Explained)

The three uncovered regions are expected and not concerns:

- **`simulator.py` line 149** — the `return` branch in `_add_ticker_internal` when the ticker already exists. This path is unreachable via `add_ticker()` because the outer guard at line 122 (`if ticker in self._prices: return`) fires first. Dead code that could be removed.

- **`simulator.py` lines 268–269** — the `except Exception: logger.exception(...)` block in `_run_loop`. Intentionally resilient error handling; testing it would require injecting an exception into `GBMSimulator.step()` which is feasible but low value.

- **`massive_client.py` lines 85–87** — the body of `_poll_loop` after the initial `await asyncio.sleep(...)`. Tests stop the source before the interval elapses, so the second poll call never fires. The behaviour is correct and tested indirectly via `test_stop_cancels_task`.

- **`massive_client.py` line 125** — the body of `_fetch_snapshots`. All tests mock `_fetch_snapshots` via `patch.object`, so the actual SDK call never executes. This is the correct approach for a test suite that shouldn't require a live API key.

---

## Architecture Assessment

The implementation is well-structured and conforms to the design specification:

**Strategy pattern with `MarketDataSource` ABC** — clean separation. Both implementations are source-agnostic downstream consumers. Adding a third data source (e.g., a WebSocket provider) requires only a new class implementing the ABC.

**`PriceCache` as single point of truth** — correct. The push model (sources write, consumers read) decouples update frequency from consumption frequency. `threading.Lock` is the right choice given `asyncio.to_thread()` usage in `MassiveDataSource`.

**GBM with Cholesky-correlated moves** — correctly implemented. The correlation matrix construction, Cholesky decomposition, and correlated normal draw application are all mathematically sound. TSLA's independent treatment is explicit and well-reasoned.

**Error resilience** — both the simulator loop and Massive poll loop catch and log exceptions without propagating them. Cache retains stale prices on Massive failures. Both sources handle empty ticker lists gracefully.

**Immediate cache seeding** — both `start()` and `add_ticker()` seed the cache before the first loop tick. This prevents the frontend from receiving an empty watchlist on page load.

**SSE version-based change detection** — avoids redundant serialization and transmission when Massive API data is stale between polls (15-second intervals vs 500ms SSE cadence).

---

## Summary

| # | Severity | File | Issue |
|---|----------|------|-------|
| 1 | Medium | `cache.py:63` | `remove()` missing `self._version += 1` — removed tickers lag in SSE stream |
| 2 | Low (latent) | `stream.py:17` | Module-level router leaked from factory — breaks on second call |
| 3 | Low | `conftest.py:10` | Deprecated `asyncio.DefaultEventLoopPolicy` — 73 warnings, removes in Py 3.16 |
| 4 | Low | `stream.py` | No unit tests for SSE route or event generator (33% coverage) |
| 5 | Very low | `cache.py:67` | `version` reads without lock — inconsistent with class threading discipline |

The codebase is production-quality for a single-user application. Issue #1 is the only one with a user-visible consequence (delayed watchlist removal in the SSE stream). Issues #2–5 are preventive — they create risk for future changes rather than immediate breakage.
