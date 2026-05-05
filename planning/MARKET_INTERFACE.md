# Market Data Interface Design

Unified Python interface for stock prices in FinAlly. Two implementations — simulator and Massive REST API — behind one abstract interface. All downstream code (SSE streaming, portfolio valuation, trade execution) is source-agnostic.

**Implementation lives in**: `backend/app/market/`

## Architecture

```
MarketDataSource (ABC)
├── SimulatorDataSource   ← GBM simulator (default, no API key needed)
└── MassiveDataSource     ← Polygon.io REST poller (when MASSIVE_API_KEY is set)
        │
        ▼
   PriceCache (thread-safe, in-memory)
        │
        ├──→ SSE stream endpoint  (/api/stream/prices)
        ├──→ Portfolio valuation  (current price × quantity)
        └──→ Trade execution      (fill at current price)
```

## Core Data Model

Defined in `app/market/models.py`. This is the only type that leaves the market data layer.

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class PriceUpdate:
    ticker: str
    price: float
    previous_price: float
    timestamp: float          # Unix seconds

    @property
    def change(self) -> float:
        return round(self.price - self.previous_price, 2)

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

## Abstract Interface

Defined in `app/market/interface.py`.

```python
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
        Call exactly once."""

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

Both implementations write to the `PriceCache`. The interface does **not** return prices directly — it pushes updates into the cache on its own schedule.

## PriceCache

Defined in `app/market/cache.py`. Thread-safe because the data source background task (asyncio thread) and web request handlers may read concurrently.

```python
import time
from threading import Lock
from .models import PriceUpdate

class PriceCache:
    """Thread-safe in-memory cache of latest prices per ticker.

    Writers: SimulatorDataSource or MassiveDataSource (one at a time).
    Readers: SSE endpoint, portfolio valuation, trade execution.
    """

    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._lock = Lock()
        self._version: int = 0  # Bumped on every update — SSE change detection

    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
        """Record a new price. Returns the created PriceUpdate.
        First update for a ticker sets previous_price == price (direction='flat')."""
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

    def get_price(self, ticker: str) -> float | None:
        """Convenience: returns just the price float, or None."""
        update = self.get(ticker)
        return update.price if update else None

    def get_all(self) -> dict[str, PriceUpdate]:
        """Snapshot of all current prices (shallow copy)."""
        with self._lock:
            return dict(self._prices)

    def remove(self, ticker: str) -> None:
        with self._lock:
            self._prices.pop(ticker, None)

    @property
    def version(self) -> int:
        """Monotonic counter. SSE endpoint uses this for change detection."""
        return self._version
```

## Factory Function

Defined in `app/market/factory.py`. Selects the data source at startup.

```python
import os
from .cache import PriceCache
from .interface import MarketDataSource

def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    """Return MassiveDataSource if MASSIVE_API_KEY is set, else SimulatorDataSource."""
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()
    if api_key:
        from .massive_client import MassiveDataSource
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    else:
        from .simulator import SimulatorDataSource
        return SimulatorDataSource(price_cache=price_cache)
```

## MassiveDataSource Implementation

Defined in `app/market/massive_client.py`.

```python
import asyncio
import logging
from massive import RESTClient
from massive.rest.models import SnapshotMarketType
from .cache import PriceCache
from .interface import MarketDataSource

logger = logging.getLogger(__name__)

class MassiveDataSource(MarketDataSource):
    """REST polling client for Polygon.io via the massive package.

    Polls /v2/snapshot/locale/us/markets/stocks/tickers — all tickers in one
    API call. Default interval is 15s (free tier: 5 req/min).
    """

    def __init__(self, api_key: str, price_cache: PriceCache, poll_interval: float = 15.0) -> None:
        self._api_key = api_key
        self._cache = price_cache
        self._interval = poll_interval
        self._tickers: list[str] = []
        self._task: asyncio.Task | None = None
        self._client: RESTClient | None = None

    async def start(self, tickers: list[str]) -> None:
        self._client = RESTClient(api_key=self._api_key)
        self._tickers = list(tickers)
        await self._poll_once()  # Immediate first poll — cache has data right away
        self._task = asyncio.create_task(self._poll_loop(), name="massive-poller")

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        self._client = None

    async def add_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        if ticker not in self._tickers:
            self._tickers.append(ticker)

    async def remove_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        self._tickers = [t for t in self._tickers if t != ticker]
        self._cache.remove(ticker)

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
            for snap in snapshots:
                try:
                    self._cache.update(
                        ticker=snap.ticker,
                        price=snap.last_trade.price,
                        timestamp=snap.last_trade.timestamp / 1000.0,  # ms → seconds
                    )
                except (AttributeError, TypeError) as e:
                    logger.warning("Skipping snapshot for %s: %s", getattr(snap, "ticker", "?"), e)
        except Exception as e:
            logger.error("Massive poll failed: %s", e)
            # Don't re-raise — loop retries on next interval

    def _fetch_snapshots(self) -> list:
        return self._client.get_snapshot_all(
            market_type=SnapshotMarketType.STOCKS,
            tickers=self._tickers,
        )
```

## SimulatorDataSource Implementation

Defined in `app/market/simulator.py`. Wraps `GBMSimulator` in an async loop.

```python
import asyncio
import logging
from .cache import PriceCache
from .interface import MarketDataSource
from .simulator import GBMSimulator

logger = logging.getLogger(__name__)

class SimulatorDataSource(MarketDataSource):
    """MarketDataSource backed by the GBM simulator.

    Calls GBMSimulator.step() every update_interval seconds and writes
    to the PriceCache. Default: 500ms intervals.
    """

    def __init__(self, price_cache: PriceCache, update_interval: float = 0.5) -> None:
        self._cache = price_cache
        self._interval = update_interval
        self._sim: GBMSimulator | None = None
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(tickers=tickers)
        # Seed cache with initial prices immediately
        for ticker in tickers:
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)
        self._task = asyncio.create_task(self._run_loop(), name="simulator-loop")

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None

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

## SSE Integration

Defined in `app/market/stream.py`. The SSE endpoint reads from `PriceCache` using version-based change detection — it only sends an event when the version counter has changed, avoiding redundant pushes.

```python
import asyncio
import json
from fastapi import APIRouter
from fastapi.responses import StreamingResponse
from .cache import PriceCache

def create_stream_router(price_cache: PriceCache) -> APIRouter:
    router = APIRouter()

    @router.get("/api/stream/prices")
    async def stream_prices():
        return StreamingResponse(
            _generate_events(price_cache),
            media_type="text/event-stream",
            headers={
                "Cache-Control": "no-cache",
                "X-Accel-Buffering": "no",
            },
        )

    return router

async def _generate_events(cache: PriceCache):
    last_version = -1
    while True:
        current_version = cache.version
        if current_version != last_version:
            last_version = current_version
            prices = cache.get_all()
            data = {ticker: update.to_dict() for ticker, update in prices.items()}
            yield f"data: {json.dumps(data)}\n\n"
        await asyncio.sleep(0.1)  # 100ms polling on server side; client gets ~10 events/sec max
```

## File Structure

```
backend/
  app/
    market/
      __init__.py           # Re-exports: PriceUpdate, PriceCache, MarketDataSource, create_market_data_source, create_stream_router
      models.py             # PriceUpdate frozen dataclass
      interface.py          # MarketDataSource ABC
      cache.py              # PriceCache
      factory.py            # create_market_data_source()
      massive_client.py     # MassiveDataSource
      simulator.py          # GBMSimulator + SimulatorDataSource
      seed_prices.py        # SEED_PRICES, TICKER_PARAMS, CORRELATION_GROUPS constants
      stream.py             # create_stream_router() — FastAPI SSE endpoint
```

## Usage in FastAPI App Startup

```python
from app.market import PriceCache, create_market_data_source, create_stream_router

# At startup
cache = PriceCache()
source = create_market_data_source(cache)        # Reads MASSIVE_API_KEY
await source.start(["AAPL", "GOOGL", "MSFT"])   # Begins background task

# Register SSE endpoint
app.include_router(create_stream_router(cache))

# Read prices anywhere in request handlers
update = cache.get("AAPL")           # PriceUpdate | None
price = cache.get_price("AAPL")      # float | None
all_prices = cache.get_all()         # dict[str, PriceUpdate]

# Dynamic watchlist
await source.add_ticker("TSLA")
await source.remove_ticker("GOOGL")

# At shutdown
await source.stop()
```

## Design Decisions

| Decision | Rationale |
|----------|-----------|
| Strategy pattern (ABC) | Both sources implement the same interface; all downstream code is source-agnostic |
| PriceCache as single point of truth | Producers write, consumers read; no direct coupling between data source and API routes |
| Version counter on PriceCache | SSE endpoint detects changes without polling timestamps or diffing dicts |
| `asyncio.to_thread` for Massive client | The `massive` SDK is synchronous; running it in a thread avoids blocking the event loop |
| Immediate first poll in `MassiveDataSource.start()` | Ensures the cache has data before the first SSE client connects |
| Seed cache on `add_ticker` in simulator | New watchlist additions appear immediately in the SSE stream |
