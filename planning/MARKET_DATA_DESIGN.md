# Market Data Backend — Design & Implementation Guide

Complete implementation reference for the FinAlly market data subsystem. Covers the unified interface, price cache, GBM simulator, Massive API client, SSE streaming endpoint, FastAPI lifecycle integration, and testing. All code lives under `backend/app/market/`.

> **Status:** Fully implemented, tested, and reviewed. See `MARKET_DATA_SUMMARY.md` for a high-level summary. This document is the authoritative implementation reference incorporating all post-review fixes.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [File Structure](#2-file-structure)
3. [Data Model — `models.py`](#3-data-model--modelspy)
4. [Price Cache — `cache.py`](#4-price-cache--cachepy)
5. [Abstract Interface — `interface.py`](#5-abstract-interface--interfacepy)
6. [Seed Prices & Parameters — `seed_prices.py`](#6-seed-prices--parameters--seed_pricespy)
7. [GBM Simulator — `simulator.py`](#7-gbm-simulator--simulatorpy)
8. [Massive API Client — `massive_client.py`](#8-massive-api-client--massive_clientpy)
9. [Factory — `factory.py`](#9-factory--factorypy)
10. [SSE Streaming Endpoint — `stream.py`](#10-sse-streaming-endpoint--streampy)
11. [Package Init — `__init__.py`](#11-package-init--initpy)
12. [FastAPI Lifecycle Integration](#12-fastapi-lifecycle-integration)
13. [Watchlist Coordination](#13-watchlist-coordination)
14. [Error Handling & Edge Cases](#14-error-handling--edge-cases)
15. [Testing Strategy](#15-testing-strategy)
16. [Configuration Reference](#16-configuration-reference)

---

## 1. Architecture Overview

```
MarketDataSource (ABC)
├── SimulatorDataSource   ← GBM simulator (default, no API key)
└── MassiveDataSource     ← Polygon.io REST poller (when MASSIVE_API_KEY is set)
        │
        ▼
   PriceCache (thread-safe, in-memory)
        │
        ├──→ SSE stream endpoint  (/api/stream/prices)
        ├──→ Portfolio valuation  (current price × quantity)
        └──→ Trade execution      (market fill at current price)
```

**Key design principles:**

- **Strategy pattern** — both data sources implement the same ABC; all downstream code is source-agnostic.
- **Push model** — data sources write to the cache on their own schedule. Consumers (SSE, portfolio) read from the cache and never call the data source directly.
- **Single point of truth** — `PriceCache` is the only place live prices are stored. No globals, no singletons beyond what FastAPI `app.state` holds.
- **Thread safety via `threading.Lock`** — not `asyncio.Lock`, because the Massive client's synchronous SDK call runs inside `asyncio.to_thread()` (a real OS thread), which `asyncio.Lock` does not protect against.

---

## 2. File Structure

```
backend/
  app/
    market/
      __init__.py           # Re-exports public API
      models.py             # PriceUpdate frozen dataclass
      interface.py          # MarketDataSource ABC
      cache.py              # PriceCache (thread-safe in-memory store)
      seed_prices.py        # Constants: SEED_PRICES, TICKER_PARAMS, CORRELATION_GROUPS
      simulator.py          # GBMSimulator + SimulatorDataSource
      massive_client.py     # MassiveDataSource (Polygon.io REST poller)
      factory.py            # create_market_data_source() factory
      stream.py             # create_stream_router() SSE endpoint factory
  tests/
    market/
      test_models.py
      test_cache.py
      test_simulator.py
      test_simulator_source.py
      test_factory.py
      test_massive.py
```

---

## 3. Data Model — `models.py`

`PriceUpdate` is the **only** type that leaves the market data layer. Every downstream consumer works with this type.

```python
# backend/app/market/models.py
from __future__ import annotations

import time
from dataclasses import dataclass, field


@dataclass(frozen=True, slots=True)
class PriceUpdate:
    """Immutable snapshot of a single ticker's price at a point in time."""

    ticker: str
    price: float
    previous_price: float
    timestamp: float = field(default_factory=time.time)  # Unix seconds

    @property
    def change(self) -> float:
        return round(self.price - self.previous_price, 4)

    @property
    def change_percent(self) -> float:
        if self.previous_price == 0:
            return 0.0
        return round((self.price - self.previous_price) / self.previous_price * 100, 4)

    @property
    def direction(self) -> str:
        """'up', 'down', or 'flat'."""
        if self.price > self.previous_price:
            return "up"
        elif self.price < self.previous_price:
            return "down"
        return "flat"

    def to_dict(self) -> dict:
        return {
            "ticker": self.ticker,
            "price": self.price,
            "previous_price": self.previous_price,
            "timestamp": self.timestamp,
            "change": self.change,
            "change_percent": self.change_percent,
            "direction": self.direction,
        }
```

**Design decisions:**
- `frozen=True` — immutable value objects, safe to share across async tasks without copying.
- `slots=True` — minor memory optimization; many of these are created per second.
- Computed properties (`change`, `direction`, `change_percent`) — derived from stored fields so they can never be inconsistent with each other.
- `to_dict()` — single serialization point used by the SSE endpoint and REST responses.

---

## 4. Price Cache — `cache.py`

The central data hub. Data sources write; SSE and portfolio code read. Thread-safe because the Massive client runs inside `asyncio.to_thread()`.

```python
# backend/app/market/cache.py
from __future__ import annotations

import time
from threading import Lock

from .models import PriceUpdate


class PriceCache:
    """Thread-safe in-memory cache of the latest price per ticker.

    Writers: SimulatorDataSource or MassiveDataSource (one at a time).
    Readers: SSE streaming endpoint, portfolio valuation, trade execution.
    """

    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._lock = Lock()
        self._version: int = 0  # Monotonically increasing; bumped on every update

    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
        """Record a new price. Returns the created PriceUpdate.

        First update for a ticker sets previous_price == price (direction='flat').
        """
        with self._lock:
            ts = timestamp or time.time()
            prev = self._prices.get(ticker)
            previous_price = prev.price if prev else price

            update = PriceUpdate(
                ticker=ticker,
                price=round(price, 2),
                previous_price=round(previous_price, 2),
                timestamp=ts,
            )
            self._prices[ticker] = update
            self._version += 1
            return update

    def get(self, ticker: str) -> PriceUpdate | None:
        with self._lock:
            return self._prices.get(ticker)

    def get_all(self) -> dict[str, PriceUpdate]:
        """Snapshot of all current prices. Returns a shallow copy."""
        with self._lock:
            return dict(self._prices)

    def get_price(self, ticker: str) -> float | None:
        """Convenience: get just the price float, or None."""
        update = self.get(ticker)
        return update.price if update else None

    def remove(self, ticker: str) -> None:
        with self._lock:
            self._prices.pop(ticker, None)
            self._version += 1

    @property
    def version(self) -> int:
        """Monotonic counter. SSE endpoint uses this to detect changes."""
        with self._lock:
            return self._version

    def __len__(self) -> int:
        with self._lock:
            return len(self._prices)

    def __contains__(self, ticker: str) -> bool:
        with self._lock:
            return ticker in self._prices
```

**Why a version counter?**

The SSE streaming loop polls the cache every 500ms. Without a version counter, it would serialize and send all prices on every iteration even when nothing changed (e.g., Massive API updates only every 15s). The version counter lets the SSE loop skip sends cheaply:

```python
last_version = -1
while True:
    if price_cache.version != last_version:
        last_version = price_cache.version
        yield format_sse(price_cache.get_all())
    await asyncio.sleep(0.5)
```

**Why `threading.Lock` not `asyncio.Lock`?**

`asyncio.Lock` only synchronizes within a single OS thread (the event loop thread). The Massive client's `get_snapshot_all()` is synchronous and runs via `asyncio.to_thread()` in a worker thread from the default thread pool. A `threading.Lock` is required to protect against concurrent access from that thread.

---

## 5. Abstract Interface — `interface.py`

```python
# backend/app/market/interface.py
from __future__ import annotations

from abc import ABC, abstractmethod


class MarketDataSource(ABC):
    """Contract for market data providers.

    Implementations push price updates into a shared PriceCache on their own
    schedule. Downstream code never calls the data source directly for prices —
    it reads from the cache.

    Lifecycle:
        source = create_market_data_source(cache)
        await source.start(["AAPL", "GOOGL", ...])
        await source.add_ticker("TSLA")
        await source.remove_ticker("GOOGL")
        await source.stop()
    """

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Begin producing price updates. Starts a background task.
        Must be called exactly once."""

    @abstractmethod
    async def stop(self) -> None:
        """Stop the background task. Safe to call multiple times."""

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Add a ticker to the active set. No-op if already present."""

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker. Also removes it from the PriceCache."""

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Return the current list of actively tracked tickers."""
```

**Why the source writes to the cache instead of returning prices:**

This push model decouples update timing from consumption. The simulator ticks at 500ms; Massive polls at 15s; the SSE endpoint reads at its own 500ms cadence. The SSE layer has no knowledge of which source is active or its update frequency.

---

## 6. Seed Prices & Parameters — `seed_prices.py`

Constants only — no logic, no imports beyond stdlib.

```python
# backend/app/market/seed_prices.py
"""Seed prices and per-ticker parameters for the market simulator."""

# Realistic starting prices for the default watchlist
SEED_PRICES: dict[str, float] = {
    "AAPL":  190.00,
    "GOOGL": 175.00,
    "MSFT":  420.00,
    "AMZN":  185.00,
    "TSLA":  250.00,
    "NVDA":  800.00,
    "META":  500.00,
    "JPM":   195.00,
    "V":     280.00,
    "NFLX":  600.00,
}

# Per-ticker GBM parameters
# sigma: annualized volatility  |  mu: annualized drift / expected return
TICKER_PARAMS: dict[str, dict[str, float]] = {
    "AAPL":  {"sigma": 0.22, "mu": 0.05},
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT":  {"sigma": 0.20, "mu": 0.05},
    "AMZN":  {"sigma": 0.28, "mu": 0.05},
    "TSLA":  {"sigma": 0.50, "mu": 0.03},  # High volatility
    "NVDA":  {"sigma": 0.40, "mu": 0.08},  # High volatility, strong drift
    "META":  {"sigma": 0.30, "mu": 0.05},
    "JPM":   {"sigma": 0.18, "mu": 0.04},  # Low volatility (bank)
    "V":     {"sigma": 0.17, "mu": 0.04},  # Low volatility (payments)
    "NFLX":  {"sigma": 0.35, "mu": 0.05},
}

# Fallback for dynamically added tickers not in the table above
DEFAULT_PARAMS: dict[str, float] = {"sigma": 0.25, "mu": 0.05}

# Sector groups for Cholesky correlation matrix
CORRELATION_GROUPS: dict[str, set[str]] = {
    "tech":    {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}

INTRA_TECH_CORR    = 0.6   # Tech stocks move together
INTRA_FINANCE_CORR = 0.5   # Finance stocks move together
CROSS_GROUP_CORR   = 0.3   # Between sectors, or unknown tickers
TSLA_CORR          = 0.3   # TSLA is in the tech set but treated as independent
```

Tickers added dynamically (not in `SEED_PRICES`) start at a random price between $50–$300 using `DEFAULT_PARAMS`.

---

## 7. GBM Simulator — `simulator.py`

Two classes in one file:

- **`GBMSimulator`** — pure math engine; stateful; holds current prices; advances them one tick at a time.
- **`SimulatorDataSource`** — `MarketDataSource` implementation that runs `GBMSimulator` in an async loop and writes to `PriceCache`.

### 7.1 GBM Mathematics

At each time step, each stock price evolves as:

```
S(t+dt) = S(t) * exp((mu - sigma²/2) * dt + sigma * sqrt(dt) * Z)
```

Where:
- `S(t)` — current price
- `mu` — annualized drift (expected return), e.g. `0.05` for 5%
- `sigma` — annualized volatility, e.g. `0.20` for 20%
- `dt` — time step as fraction of a trading year
- `Z` — standard normal random variable (correlated across tickers via Cholesky)

**Time step for 500ms ticks:**
```
trading_seconds_per_year = 252 days × 6.5 hours × 3600 s = 5,896,800 s/year
dt = 0.5 / 5,896,800 ≈ 8.48 × 10⁻⁸
```

This tiny `dt` produces sub-cent moves per tick that accumulate naturally. A stock with `sigma=0.25` will see roughly the correct intraday range over a simulated trading day.

### 7.2 Correlated Moves via Cholesky Decomposition

Real stocks don't move independently — tech stocks correlate. The simulator applies Cholesky decomposition to a sector-based correlation matrix to generate correlated random draws.

Given correlation matrix `C`, compute lower-triangular Cholesky factor `L` where `C = L @ Lᵀ`. Then for independent standard-normal draws `Z_ind`:

```
Z_correlated = L @ Z_ind
```

The resulting `Z_correlated` has the correct pairwise correlations. The Cholesky matrix is rebuilt whenever tickers are added or removed — O(n²) but n < 50 in practice.

### 7.3 GBMSimulator Implementation

```python
# backend/app/market/simulator.py  (part 1 of 2)
from __future__ import annotations

import asyncio
import logging
import math
import random

import numpy as np

from .cache import PriceCache
from .interface import MarketDataSource
from .seed_prices import (
    CORRELATION_GROUPS,
    CROSS_GROUP_CORR,
    DEFAULT_PARAMS,
    INTRA_FINANCE_CORR,
    INTRA_TECH_CORR,
    SEED_PRICES,
    TICKER_PARAMS,
    TSLA_CORR,
)

logger = logging.getLogger(__name__)


class GBMSimulator:
    """Geometric Brownian Motion simulator for correlated stock prices.

    S(t+dt) = S(t) * exp((mu - sigma²/2)*dt + sigma*sqrt(dt)*Z)
    """

    # 500ms as a fraction of a trading year (252 days * 6.5 h/day * 3600 s/h)
    TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # 5,896,800
    DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR   # ~8.48e-8

    def __init__(
        self,
        tickers: list[str],
        dt: float = DEFAULT_DT,
        event_probability: float = 0.001,
    ) -> None:
        self._dt = dt
        self._event_prob = event_probability
        self._tickers: list[str] = []
        self._prices: dict[str, float] = {}
        self._params: dict[str, dict[str, float]] = {}
        self._cholesky: np.ndarray | None = None

        for ticker in tickers:
            self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    def step(self) -> dict[str, float]:
        """Advance all tickers one time step. Returns {ticker: new_price}.
        Called every 500ms — keep it fast."""
        n = len(self._tickers)
        if n == 0:
            return {}

        z_ind = np.random.standard_normal(n)
        z = self._cholesky @ z_ind if self._cholesky is not None else z_ind

        result: dict[str, float] = {}
        for i, ticker in enumerate(self._tickers):
            mu = self._params[ticker]["mu"]
            sigma = self._params[ticker]["sigma"]

            drift = (mu - 0.5 * sigma ** 2) * self._dt
            diffusion = sigma * math.sqrt(self._dt) * z[i]
            self._prices[ticker] *= math.exp(drift + diffusion)

            # Random shock event: ~0.1% chance per tick per ticker.
            # With 10 tickers at 2 ticks/sec → roughly one event every 50s.
            if random.random() < self._event_prob:
                shock = random.uniform(0.02, 0.05) * random.choice([-1, 1])
                self._prices[ticker] *= (1 + shock)
                logger.debug("Random event on %s: %+.1f%%", ticker, shock * 100)

            result[ticker] = round(self._prices[ticker], 2)

        return result

    def add_ticker(self, ticker: str) -> None:
        """Add a ticker. Rebuilds the Cholesky matrix."""
        if ticker in self._prices:
            return
        self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker. Rebuilds the Cholesky matrix."""
        if ticker not in self._prices:
            return
        self._tickers.remove(ticker)
        del self._prices[ticker]
        del self._params[ticker]
        self._rebuild_cholesky()

    def get_price(self, ticker: str) -> float | None:
        return self._prices.get(ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    def _add_ticker_internal(self, ticker: str) -> None:
        """Add without rebuilding Cholesky — for batch initialization."""
        if ticker in self._prices:
            return
        self._tickers.append(ticker)
        self._prices[ticker] = SEED_PRICES.get(ticker, random.uniform(50.0, 300.0))
        self._params[ticker] = TICKER_PARAMS.get(ticker, dict(DEFAULT_PARAMS))

    def _rebuild_cholesky(self) -> None:
        """Rebuild Cholesky decomposition. O(n²), called on add/remove."""
        n = len(self._tickers)
        if n <= 1:
            self._cholesky = None
            return

        corr = np.eye(n)
        for i in range(n):
            for j in range(i + 1, n):
                rho = self._pairwise_correlation(self._tickers[i], self._tickers[j])
                corr[i, j] = rho
                corr[j, i] = rho

        self._cholesky = np.linalg.cholesky(corr)

    @staticmethod
    def _pairwise_correlation(t1: str, t2: str) -> float:
        tech = CORRELATION_GROUPS["tech"]
        finance = CORRELATION_GROUPS["finance"]

        if t1 == "TSLA" or t2 == "TSLA":
            return TSLA_CORR         # 0.3 — TSLA does its own thing
        if t1 in tech and t2 in tech:
            return INTRA_TECH_CORR   # 0.6
        if t1 in finance and t2 in finance:
            return INTRA_FINANCE_CORR  # 0.5
        return CROSS_GROUP_CORR      # 0.3
```

### 7.4 SimulatorDataSource — Async Wrapper

```python
# backend/app/market/simulator.py  (part 2 of 2)

class SimulatorDataSource(MarketDataSource):
    """MarketDataSource backed by the GBM simulator.

    Runs a background asyncio task calling GBMSimulator.step() every
    `update_interval` seconds and writing results to PriceCache.
    """

    def __init__(
        self,
        price_cache: PriceCache,
        update_interval: float = 0.5,
        event_probability: float = 0.001,
    ) -> None:
        self._cache = price_cache
        self._interval = update_interval
        self._event_prob = event_probability
        self._sim: GBMSimulator | None = None
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(tickers=tickers, event_probability=self._event_prob)
        # Seed cache before the loop starts so SSE has data on its first tick.
        for ticker in tickers:
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)
        self._task = asyncio.create_task(self._run_loop(), name="simulator-loop")
        logger.info("Simulator started with %d tickers", len(tickers))

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        logger.info("Simulator stopped")

    async def add_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.add_ticker(ticker)
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)  # Seed immediately

    async def remove_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.remove_ticker(ticker)
        self._cache.remove(ticker)

    def get_tickers(self) -> list[str]:
        return self._sim.get_tickers() if self._sim else []

    async def _run_loop(self) -> None:
        while True:
            try:
                if self._sim:
                    prices = self._sim.step()
                    for ticker, price in prices.items():
                        self._cache.update(ticker=ticker, price=price)
            except Exception:
                logger.exception("Simulator step failed")
            await asyncio.sleep(self._interval)
```

**Key behaviors:**
- **Immediate seeding** — `start()` populates the cache with seed prices *before* the loop starts; the frontend never sees a blank watchlist.
- **Graceful cancellation** — `stop()` cancels and awaits the task, catching `CancelledError`; clean shutdown during FastAPI lifespan teardown.
- **Exception resilience** — the loop catches and logs exceptions per-step; a single bad tick doesn't kill the feed.

---

## 8. Massive API Client — `massive_client.py`

Polls the Massive (formerly Polygon.io) REST API snapshot endpoint. The synchronous Massive SDK runs inside `asyncio.to_thread()` to avoid blocking the event loop.

```python
# backend/app/market/massive_client.py
from __future__ import annotations

import asyncio
import logging
from typing import Any

from massive import RESTClient
from massive.rest.models import SnapshotMarketType

from .cache import PriceCache
from .interface import MarketDataSource

logger = logging.getLogger(__name__)


class MassiveDataSource(MarketDataSource):
    """MarketDataSource backed by the Massive (Polygon.io) REST API.

    Polls GET /v2/snapshot/locale/us/markets/stocks/tickers for all watched
    tickers in a single API call, then writes results to PriceCache.

    Rate limits:
      Free tier: 5 req/min  → poll_interval=15.0 (default)
      Paid tiers: unlimited → poll_interval=2.0–5.0
    """

    def __init__(
        self,
        api_key: str,
        price_cache: PriceCache,
        poll_interval: float = 15.0,
    ) -> None:
        self._api_key = api_key
        self._cache = price_cache
        self._interval = poll_interval
        self._tickers: list[str] = []
        self._task: asyncio.Task | None = None
        self._client: RESTClient | None = None

    async def start(self, tickers: list[str]) -> None:
        self._client = RESTClient(api_key=self._api_key)
        self._tickers = list(tickers)
        await self._poll_once()  # Immediate first poll — cache has data before SSE clients connect
        self._task = asyncio.create_task(self._poll_loop(), name="massive-poller")
        logger.info("Massive poller started: %d tickers, %.1fs interval", len(tickers), self._interval)

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        self._client = None
        logger.info("Massive poller stopped")

    async def add_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        if ticker not in self._tickers:
            self._tickers.append(ticker)
            logger.info("Massive: added ticker %s (appears on next poll)", ticker)

    async def remove_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        self._tickers = [t for t in self._tickers if t != ticker]
        self._cache.remove(ticker)
        logger.info("Massive: removed ticker %s", ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    async def _poll_loop(self) -> None:
        while True:
            await asyncio.sleep(self._interval)
            await self._poll_once()

    async def _poll_once(self) -> None:
        if not self._tickers or not self._client:
            return
        try:
            # RESTClient is synchronous — run in thread to avoid blocking event loop
            snapshots = await asyncio.to_thread(self._fetch_snapshots)
            processed = 0
            for snap in snapshots:
                try:
                    self._cache.update(
                        ticker=snap.ticker,
                        price=snap.last_trade.price,
                        timestamp=snap.last_trade.timestamp / 1000.0,  # ms → seconds
                    )
                    processed += 1
                except (AttributeError, TypeError) as e:
                    logger.warning("Skipping snapshot for %s: %s", getattr(snap, "ticker", "?"), e)
            logger.debug("Massive poll: updated %d/%d tickers", processed, len(self._tickers))
        except Exception as e:
            logger.error("Massive poll failed: %s", e)
            # Don't re-raise — loop retries on next interval.
            # Handles: 401 (bad key), 429 (rate limit), network failures.

    def _fetch_snapshots(self) -> list:
        """Synchronous REST call — runs in a worker thread via asyncio.to_thread."""
        return self._client.get_snapshot_all(
            market_type=SnapshotMarketType.STOCKS,
            tickers=self._tickers,
        )
```

**Error handling philosophy:**

| Error | Behavior |
|-------|----------|
| 401 Unauthorized | Logged. Poller keeps running — user can fix `.env` and restart. |
| 429 Rate Limited | Logged. Next poll retries after `poll_interval` seconds. |
| Network timeout | Logged. Auto-retries on next cycle. |
| Malformed snapshot | Individual ticker skipped with warning. Other tickers still process. |
| All tickers fail | Cache retains last-known prices. SSE keeps streaming stale (better than nothing). |

**Snapshot response structure** (Massive REST API):

```json
{
  "ticker": "AAPL",
  "last_trade": {
    "price": 190.85,
    "size": 100,
    "exchange": "XNAS",
    "timestamp": 1707939600000
  },
  "day": {
    "open": 189.20,
    "high": 192.10,
    "low": 188.50,
    "close": 190.85,
    "volume": 68432100,
    "previous_close": 188.30,
    "change_percent": 1.35
  }
}
```

Fields extracted by FinAlly:
| Field | Usage |
|-------|-------|
| `last_trade.price` | Current price for trading and display |
| `last_trade.timestamp` | Price timestamp (milliseconds → divide by 1000 for seconds) |

---

## 9. Factory — `factory.py`

```python
# backend/app/market/factory.py
from __future__ import annotations

import logging
import os

from .cache import PriceCache
from .interface import MarketDataSource

logger = logging.getLogger(__name__)


def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    """Select market data source based on environment.

    - MASSIVE_API_KEY set and non-empty → MassiveDataSource (real market data)
    - Otherwise → SimulatorDataSource (GBM simulation, default)

    Returns an unstarted source. Caller must await source.start(tickers).
    """
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()

    if api_key:
        from .massive_client import MassiveDataSource
        logger.info("Market data source: Massive API (real data)")
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    else:
        from .simulator import SimulatorDataSource
        logger.info("Market data source: GBM Simulator")
        return SimulatorDataSource(price_cache=price_cache)
```

---

## 10. SSE Streaming Endpoint — `stream.py`

The SSE endpoint holds a long-lived HTTP connection open and pushes price updates to connected clients as `text/event-stream`.

```python
# backend/app/market/stream.py
from __future__ import annotations

import asyncio
import json
import logging
from collections.abc import AsyncGenerator

from fastapi import APIRouter, Request
from fastapi.responses import StreamingResponse

from .cache import PriceCache

logger = logging.getLogger(__name__)


def create_stream_router(price_cache: PriceCache) -> APIRouter:
    """Create the SSE streaming router with an injected PriceCache reference."""
    router = APIRouter(prefix="/api/stream", tags=["streaming"])

    @router.get("/prices")
    async def stream_prices(request: Request) -> StreamingResponse:
        """SSE endpoint for live price updates.

        All tracked ticker prices are pushed every ~500ms. Client connects
        with the browser EventSource API and receives:

            data: {"AAPL": {"ticker":"AAPL","price":190.50,...}, "GOOGL":{...}}

        EventSource auto-reconnects on disconnection (built-in retry behavior).
        """
        return StreamingResponse(
            _generate_events(price_cache, request),
            media_type="text/event-stream",
            headers={
                "Cache-Control": "no-cache",
                "Connection": "keep-alive",
                "X-Accel-Buffering": "no",  # Disable nginx buffering if proxied
            },
        )

    return router


async def _generate_events(
    price_cache: PriceCache,
    request: Request,
    interval: float = 0.5,
) -> AsyncGenerator[str, None]:
    """Yield SSE-formatted price events until client disconnects."""
    yield "retry: 1000\n\n"  # Tell browser to reconnect after 1s if dropped

    last_version = -1
    client_ip = request.client.host if request.client else "unknown"
    logger.info("SSE client connected: %s", client_ip)

    try:
        while True:
            if await request.is_disconnected():
                logger.info("SSE client disconnected: %s", client_ip)
                break

            current_version = price_cache.version
            if current_version != last_version:
                last_version = current_version
                prices = price_cache.get_all()
                if prices:
                    data = {ticker: update.to_dict() for ticker, update in prices.items()}
                    yield f"data: {json.dumps(data)}\n\n"

            await asyncio.sleep(interval)
    except asyncio.CancelledError:
        logger.info("SSE stream cancelled for: %s", client_ip)
```

**SSE wire format** — each event the client receives:

```
retry: 1000

data: {"AAPL":{"ticker":"AAPL","price":190.50,"previous_price":190.42,"timestamp":1707580800.5,"change":0.08,"change_percent":0.042,"direction":"up"},"GOOGL":{"ticker":"GOOGL","price":175.12,...}}

```

**Frontend consumption:**

```typescript
const eventSource = new EventSource('/api/stream/prices');

eventSource.onmessage = (event) => {
    const prices: Record<string, PriceUpdate> = JSON.parse(event.data);
    // prices["AAPL"].price, prices["AAPL"].direction, etc.
};

eventSource.onerror = () => {
    // EventSource auto-reconnects; update connection status indicator
};
```

**Why version-based change detection?**

Polling on a fixed interval is simpler and produces evenly-spaced updates, which is important for sparkline visualization. The version counter avoids sending duplicate payloads when the source hasn't updated (e.g., Massive API at 15s intervals while SSE polls at 500ms).

---

## 11. Package Init — `__init__.py`

```python
# backend/app/market/__init__.py
"""Market data subsystem for FinAlly.

Public API:
    PriceUpdate              — Immutable price snapshot
    PriceCache               — Thread-safe in-memory price store
    MarketDataSource         — Abstract interface for data providers
    create_market_data_source — Factory: selects simulator or Massive
    create_stream_router     — FastAPI SSE endpoint factory
"""

from .cache import PriceCache
from .factory import create_market_data_source
from .interface import MarketDataSource
from .models import PriceUpdate
from .stream import create_stream_router

__all__ = [
    "PriceUpdate",
    "PriceCache",
    "MarketDataSource",
    "create_market_data_source",
    "create_stream_router",
]
```

All downstream code imports from `app.market` — never from submodules directly.

---

## 12. FastAPI Lifecycle Integration

Market data starts and stops with the application via FastAPI's `lifespan` context manager.

```python
# backend/app/main.py  (relevant sections)
from contextlib import asynccontextmanager

from fastapi import FastAPI

from app.market import PriceCache, MarketDataSource, create_market_data_source, create_stream_router


@asynccontextmanager
async def lifespan(app: FastAPI):
    # --- STARTUP ---

    price_cache = PriceCache()
    app.state.price_cache = price_cache

    source = create_market_data_source(price_cache)
    app.state.market_source = source

    # Load initial tickers from the database
    initial_tickers = await load_watchlist_tickers()  # reads from SQLite
    await source.start(initial_tickers)

    # Register SSE route after creating the cache/source
    app.include_router(create_stream_router(price_cache))

    yield  # App is running

    # --- SHUTDOWN ---
    await source.stop()


app = FastAPI(title="FinAlly", lifespan=lifespan)


# FastAPI dependencies for injecting into route handlers
def get_price_cache() -> PriceCache:
    return app.state.price_cache

def get_market_source() -> MarketDataSource:
    return app.state.market_source
```

**Accessing market data in route handlers:**

```python
from fastapi import APIRouter, Depends, HTTPException

router = APIRouter(prefix="/api")

@router.post("/portfolio/trade")
async def execute_trade(
    trade: TradeRequest,
    price_cache: PriceCache = Depends(get_price_cache),
):
    current_price = price_cache.get_price(trade.ticker)
    if current_price is None:
        raise HTTPException(400, f"No price available for {trade.ticker}. Try again in a moment.")
    # ... execute trade at current_price ...


@router.get("/watchlist")
async def get_watchlist(
    price_cache: PriceCache = Depends(get_price_cache),
):
    # Return watchlist with current prices from cache
    all_prices = price_cache.get_all()
    # ... build response ...
```

---

## 13. Watchlist Coordination

When the watchlist changes via the REST API or LLM chat, the data source must be notified to track the updated ticker set.

### Adding a Ticker

```
POST /api/watchlist {ticker: "PYPL"}
  → Validate ticker format
  → Insert into watchlist table (SQLite)
  → await source.add_ticker("PYPL")
      Simulator: adds to GBMSimulator, rebuilds Cholesky, seeds cache immediately
      Massive:   appends to ticker list, appears on next poll cycle
  → Return {ticker, price} (price available immediately for simulator; may be null for Massive)
```

```python
@router.post("/watchlist")
async def add_to_watchlist(
    payload: WatchlistAdd,
    source: MarketDataSource = Depends(get_market_source),
    price_cache: PriceCache = Depends(get_price_cache),
):
    ticker = payload.ticker.upper().strip()
    await db.insert_watchlist_entry(ticker)
    await source.add_ticker(ticker)
    price = price_cache.get_price(ticker)
    return {"ticker": ticker, "price": price}
```

### Removing a Ticker

```
DELETE /api/watchlist/{ticker}
  → Delete from watchlist table (SQLite)
  → Check for open position — if held, keep tracking for portfolio valuation
  → await source.remove_ticker(ticker)  (only if no open position)
  → Return {status: "ok"}
```

```python
@router.delete("/watchlist/{ticker}")
async def remove_from_watchlist(
    ticker: str,
    source: MarketDataSource = Depends(get_market_source),
):
    ticker = ticker.upper()
    await db.delete_watchlist_entry(ticker)

    # Keep tracking if user still holds shares (needed for P&L valuation)
    position = await db.get_position(ticker)
    if not position or position.quantity == 0:
        await source.remove_ticker(ticker)

    return {"status": "ok"}
```

---

## 14. Error Handling & Edge Cases

### Empty Watchlist at Startup

Both sources handle an empty ticker list gracefully — the simulator produces no prices, the Massive poller skips its API call. The SSE endpoint sends empty events. When the user adds the first ticker, tracking begins immediately.

### Price Cache Miss During Trade Execution

```python
price = price_cache.get_price(ticker)
if price is None:
    raise HTTPException(
        status_code=400,
        detail=f"Price not yet available for {ticker}. Please wait a moment.",
    )
```

The simulator avoids this by seeding the cache in `add_ticker()`. The Massive client may have a brief gap between `add_ticker()` and the next poll cycle — the HTTP 400 with a clear message is the correct response.

### Invalid Massive API Key

The first poll fails with HTTP 401. The poller logs the error and keeps retrying on each interval. The SSE endpoint streams empty data. The fix is to correct `MASSIVE_API_KEY` in `.env` and restart the container.

### Ticker Not Found by Massive API

If a user adds an invalid ticker (e.g., "AAAP" typo), Massive returns no snapshot for it. The poller silently skips it with a warning log. The cache will have no entry for the ticker, and any trade attempt returns HTTP 400.

### Thread Safety Under Load

`PriceCache` uses `threading.Lock` — a mutex with no contention risk at 10 tickers × 2 updates/second. The critical section is tiny (dict lookup + assignment). If the project ever scaled to hundreds of tickers with many concurrent SSE readers, a reader-writer lock would be the appropriate upgrade.

---

## 15. Testing Strategy

### Unit Tests — GBMSimulator

```python
# backend/tests/market/test_simulator.py
import pytest
from app.market.simulator import GBMSimulator
from app.market.seed_prices import SEED_PRICES


class TestGBMSimulator:

    def test_step_returns_all_tickers(self):
        sim = GBMSimulator(tickers=["AAPL", "GOOGL"])
        result = sim.step()
        assert set(result.keys()) == {"AAPL", "GOOGL"}

    def test_prices_always_positive(self):
        """GBM uses exp() — prices can never go negative."""
        sim = GBMSimulator(tickers=["AAPL"])
        for _ in range(10_000):
            prices = sim.step()
            assert prices["AAPL"] > 0

    def test_initial_prices_match_seeds(self):
        sim = GBMSimulator(tickers=["AAPL"])
        assert sim.get_price("AAPL") == SEED_PRICES["AAPL"]

    def test_unknown_ticker_gets_random_price_in_range(self):
        sim = GBMSimulator(tickers=["ZZZZ"])
        assert 50.0 <= sim.get_price("ZZZZ") <= 300.0

    def test_add_ticker(self):
        sim = GBMSimulator(tickers=["AAPL"])
        sim.add_ticker("TSLA")
        assert "TSLA" in sim.step()

    def test_remove_ticker(self):
        sim = GBMSimulator(tickers=["AAPL", "GOOGL"])
        sim.remove_ticker("GOOGL")
        result = sim.step()
        assert "GOOGL" not in result
        assert "AAPL" in result

    def test_add_duplicate_is_noop(self):
        sim = GBMSimulator(tickers=["AAPL"])
        sim.add_ticker("AAPL")
        assert len(sim.get_tickers()) == 1

    def test_remove_nonexistent_is_noop(self):
        sim = GBMSimulator(tickers=["AAPL"])
        sim.remove_ticker("NOPE")  # Must not raise

    def test_empty_simulator_returns_empty_dict(self):
        sim = GBMSimulator(tickers=[])
        assert sim.step() == {}

    def test_cholesky_none_for_single_ticker(self):
        sim = GBMSimulator(tickers=["AAPL"])
        assert sim._cholesky is None

    def test_cholesky_built_for_two_tickers(self):
        sim = GBMSimulator(tickers=["AAPL", "GOOGL"])
        assert sim._cholesky is not None

    def test_cholesky_rebuilt_on_add(self):
        sim = GBMSimulator(tickers=["AAPL"])
        assert sim._cholesky is None
        sim.add_ticker("GOOGL")
        assert sim._cholesky is not None

    def test_full_default_watchlist_cholesky_succeeds(self):
        """Cholesky should succeed for all 10 default tickers."""
        tickers = ["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA", "NVDA", "META", "JPM", "V", "NFLX"]
        sim = GBMSimulator(tickers=tickers)
        assert sim._cholesky is not None
        assert sim.step()  # No exception
```

### Unit Tests — PriceCache

```python
# backend/tests/market/test_cache.py
from app.market.cache import PriceCache


class TestPriceCache:

    def test_update_and_get(self):
        cache = PriceCache()
        update = cache.update("AAPL", 190.50)
        assert update.ticker == "AAPL"
        assert update.price == 190.50
        assert cache.get("AAPL") == update

    def test_first_update_direction_is_flat(self):
        cache = PriceCache()
        update = cache.update("AAPL", 190.50)
        assert update.direction == "flat"
        assert update.previous_price == 190.50

    def test_direction_up(self):
        cache = PriceCache()
        cache.update("AAPL", 190.00)
        update = cache.update("AAPL", 191.00)
        assert update.direction == "up"
        assert update.change == 1.00

    def test_direction_down(self):
        cache = PriceCache()
        cache.update("AAPL", 190.00)
        update = cache.update("AAPL", 189.00)
        assert update.direction == "down"
        assert update.change == -1.00

    def test_remove(self):
        cache = PriceCache()
        cache.update("AAPL", 190.00)
        cache.remove("AAPL")
        assert cache.get("AAPL") is None

    def test_get_all_returns_snapshot(self):
        cache = PriceCache()
        cache.update("AAPL", 190.00)
        cache.update("GOOGL", 175.00)
        assert set(cache.get_all().keys()) == {"AAPL", "GOOGL"}

    def test_version_increments_on_update(self):
        cache = PriceCache()
        v0 = cache.version
        cache.update("AAPL", 190.00)
        assert cache.version == v0 + 1

    def test_get_price_returns_none_for_unknown(self):
        cache = PriceCache()
        assert cache.get_price("NOPE") is None
```

### Integration Tests — SimulatorDataSource

```python
# backend/tests/market/test_simulator_source.py
import asyncio
import pytest
from app.market.cache import PriceCache
from app.market.simulator import SimulatorDataSource


@pytest.mark.asyncio
class TestSimulatorDataSource:

    async def test_start_populates_cache_immediately(self):
        cache = PriceCache()
        source = SimulatorDataSource(price_cache=cache, update_interval=0.1)
        await source.start(["AAPL", "GOOGL"])
        # Cache seeded before first loop tick
        assert cache.get("AAPL") is not None
        assert cache.get("GOOGL") is not None
        await source.stop()

    async def test_stop_is_idempotent(self):
        cache = PriceCache()
        source = SimulatorDataSource(price_cache=cache, update_interval=0.1)
        await source.start(["AAPL"])
        await source.stop()
        await source.stop()  # Second stop must not raise

    async def test_add_ticker_seeds_cache(self):
        cache = PriceCache()
        source = SimulatorDataSource(price_cache=cache, update_interval=0.5)
        await source.start(["AAPL"])
        await source.add_ticker("TSLA")
        assert cache.get("TSLA") is not None
        await source.stop()

    async def test_remove_ticker_clears_cache(self):
        cache = PriceCache()
        source = SimulatorDataSource(price_cache=cache, update_interval=0.5)
        await source.start(["AAPL", "TSLA"])
        await source.remove_ticker("TSLA")
        assert cache.get("TSLA") is None
        assert "TSLA" not in source.get_tickers()
        await source.stop()
```

### Unit Tests — MassiveDataSource (Mocked)

```python
# backend/tests/market/test_massive.py
from unittest.mock import MagicMock, patch
import pytest
from app.market.cache import PriceCache
from app.market.massive_client import MassiveDataSource


def _make_snapshot(ticker: str, price: float, timestamp_ms: int = 1707580800000) -> MagicMock:
    snap = MagicMock()
    snap.ticker = ticker
    snap.last_trade.price = price
    snap.last_trade.timestamp = timestamp_ms
    return snap


@pytest.mark.asyncio
class TestMassiveDataSource:

    async def test_poll_updates_cache(self):
        cache = PriceCache()
        source = MassiveDataSource(api_key="test", price_cache=cache, poll_interval=60.0)
        source._tickers = ["AAPL", "GOOGL"]

        with patch.object(source, "_fetch_snapshots", return_value=[
            _make_snapshot("AAPL", 190.50),
            _make_snapshot("GOOGL", 175.25),
        ]):
            await source._poll_once()

        assert cache.get_price("AAPL") == 190.50
        assert cache.get_price("GOOGL") == 175.25

    async def test_malformed_snapshot_is_skipped(self):
        cache = PriceCache()
        source = MassiveDataSource(api_key="test", price_cache=cache, poll_interval=60.0)
        source._tickers = ["AAPL", "BAD"]

        bad_snap = MagicMock()
        bad_snap.ticker = "BAD"
        bad_snap.last_trade = None  # Causes AttributeError

        with patch.object(source, "_fetch_snapshots", return_value=[
            _make_snapshot("AAPL", 190.50),
            bad_snap,
        ]):
            await source._poll_once()  # Must not raise

        assert cache.get_price("AAPL") == 190.50
        assert cache.get_price("BAD") is None

    async def test_api_error_does_not_crash(self):
        cache = PriceCache()
        source = MassiveDataSource(api_key="test", price_cache=cache, poll_interval=60.0)
        source._tickers = ["AAPL"]

        with patch.object(source, "_fetch_snapshots", side_effect=Exception("network error")):
            await source._poll_once()  # Must not raise

    async def test_timestamp_converted_from_ms_to_seconds(self):
        cache = PriceCache()
        source = MassiveDataSource(api_key="test", price_cache=cache, poll_interval=60.0)
        source._tickers = ["AAPL"]
        ts_ms = 1707580800000

        with patch.object(source, "_fetch_snapshots", return_value=[_make_snapshot("AAPL", 190.0, ts_ms)]):
            await source._poll_once()

        update = cache.get("AAPL")
        assert update is not None
        assert abs(update.timestamp - ts_ms / 1000.0) < 0.001

    async def test_add_and_remove_ticker(self):
        cache = PriceCache()
        source = MassiveDataSource(api_key="test", price_cache=cache, poll_interval=60.0)
        source._tickers = ["AAPL"]
        source._client = MagicMock()

        await source.add_ticker("TSLA")
        assert "TSLA" in source.get_tickers()

        # Seed a price so remove can clear it
        cache.update("TSLA", 250.0)
        await source.remove_ticker("TSLA")
        assert "TSLA" not in source.get_tickers()
        assert cache.get("TSLA") is None
```

**Running the tests:**

```bash
cd backend
uv run --extra dev pytest tests/market/ -v --cov=app/market
```

---

## 16. Configuration Reference

All tunable parameters with their defaults:

| Parameter | Location | Default | Description |
|-----------|----------|---------|-------------|
| `MASSIVE_API_KEY` | Environment variable | `""` | If set, use Massive API; otherwise use simulator |
| `update_interval` | `SimulatorDataSource.__init__` | `0.5` s | Time between simulator ticks |
| `poll_interval` | `MassiveDataSource.__init__` | `15.0` s | Time between Massive API polls |
| `event_probability` | `GBMSimulator.__init__` | `0.001` | Chance of random shock per ticker per tick |
| `dt` | `GBMSimulator.__init__` | `~8.5e-8` | GBM time step (fraction of trading year) |
| SSE push interval | `_generate_events()` | `0.5` s | Time between SSE sends to each client |
| SSE retry directive | `_generate_events()` | `1000` ms | Browser EventSource reconnection delay |

**GBM volatility reference** (for calibration):

| Ticker | sigma | Notes |
|--------|-------|-------|
| V, JPM | 0.17–0.18 | Low-volatility large-cap finance |
| MSFT, AAPL | 0.20–0.22 | Steady large-cap tech |
| GOOGL, AMZN | 0.25–0.28 | Mid-volatility tech |
| META, NFLX | 0.30–0.35 | Higher-volatility tech |
| NVDA | 0.40 | High-volatility semiconductor |
| TSLA | 0.50 | Very high volatility; independent behavior |
