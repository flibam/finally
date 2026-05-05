# Market Simulator Design

Approach and code structure for simulating realistic stock prices when `MASSIVE_API_KEY` is not set.

**Implementation**: `backend/app/market/simulator.py` + `backend/app/market/seed_prices.py`

## Overview

The simulator uses **Geometric Brownian Motion (GBM)** — the same stochastic model underlying Black-Scholes option pricing. Prices evolve continuously with correlated random noise:

- Can never go negative (GBM is multiplicative — `exp()` is always positive)
- Exhibit lognormal distribution seen in real markets
- Are correlated across stocks in the same sector
- Produce occasional large random "events" for visual drama

Updates run every **500ms**, giving a continuous live feed indistinguishable from real market data at a glance.

## GBM Mathematics

At each time step, each stock price evolves as:

```
S(t+dt) = S(t) * exp((mu - sigma^2/2) * dt + sigma * sqrt(dt) * Z)
```

Where:
- `S(t)` — current price
- `mu` — annualized drift (expected return), e.g. `0.05` for 5%
- `sigma` — annualized volatility, e.g. `0.20` for 20%
- `dt` — time step as a fraction of a trading year
- `Z` — standard normal random variable drawn from N(0,1)

**Time step calculation for 500ms ticks**:
```
trading_seconds_per_year = 252 days × 6.5 hours/day × 3600 s/hour = 5,896,800 s/year
dt = 0.5 / 5,896,800 ≈ 8.48 × 10⁻⁸
```

This tiny `dt` produces sub-cent moves per tick that accumulate naturally — a stock with `sigma=0.25` will see roughly the right intraday range over a simulated trading day.

## Correlated Moves

Real stocks don't move independently — tech stocks move together, financials move together, etc. The simulator uses **Cholesky decomposition** of a correlation matrix to generate correlated random draws.

Given a correlation matrix `C`, compute its lower-triangular Cholesky factor `L` such that `C = L @ L.T`. Then for a vector of independent standard normal draws `Z_ind`:

```
Z_correlated = L @ Z_ind
```

The resulting `Z_correlated` has the correct pairwise correlations. These are fed into each ticker's GBM step.

**The Cholesky matrix is rebuilt whenever tickers are added or removed** (O(n²) but n < 50).

### Correlation Groups

Defined in `seed_prices.py`:

```python
CORRELATION_GROUPS = {
    "tech":    {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}

INTRA_TECH_CORR    = 0.6   # Tech stocks move together
INTRA_FINANCE_CORR = 0.5   # Finance stocks move together
CROSS_GROUP_CORR   = 0.3   # Between sectors or unknown tickers
TSLA_CORR          = 0.3   # TSLA does its own thing (in tech set but treated as independent)
```

Pairwise correlation lookup:

```python
@staticmethod
def _pairwise_correlation(t1: str, t2: str) -> float:
    tech = CORRELATION_GROUPS["tech"]
    finance = CORRELATION_GROUPS["finance"]

    if t1 == "TSLA" or t2 == "TSLA":
        return TSLA_CORR  # 0.3

    if t1 in tech and t2 in tech:
        return INTRA_TECH_CORR  # 0.6
    if t1 in finance and t2 in finance:
        return INTRA_FINANCE_CORR  # 0.5

    return CROSS_GROUP_CORR  # 0.3
```

## Random Events

Every tick, each ticker has a small probability of a sudden "event" — a 2–5% shock in either direction:

```python
EVENT_PROBABILITY = 0.001  # 0.1% per tick per ticker

if random.random() < EVENT_PROBABILITY:
    shock_magnitude = random.uniform(0.02, 0.05)
    shock_sign = random.choice([-1, 1])
    price *= 1 + shock_magnitude * shock_sign
```

With 10 tickers at 2 ticks/second, expect a random event somewhere in the watchlist roughly every **50 seconds** — enough to keep the dashboard visually interesting.

## Seed Prices and Per-Ticker Parameters

Defined in `seed_prices.py`. Realistic starting prices and volatility parameters for the default watchlist:

```python
SEED_PRICES: dict[str, float] = {
    "AAPL": 190.00,
    "GOOGL": 175.00,
    "MSFT": 420.00,
    "AMZN": 185.00,
    "TSLA": 250.00,
    "NVDA": 800.00,
    "META": 500.00,
    "JPM":  195.00,
    "V":    280.00,
    "NFLX": 600.00,
}

TICKER_PARAMS: dict[str, dict[str, float]] = {
    "AAPL":  {"sigma": 0.22, "mu": 0.05},
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT":  {"sigma": 0.20, "mu": 0.05},
    "AMZN":  {"sigma": 0.28, "mu": 0.05},
    "TSLA":  {"sigma": 0.50, "mu": 0.03},  # High volatility
    "NVDA":  {"sigma": 0.40, "mu": 0.08},  # High vol, strong drift
    "META":  {"sigma": 0.30, "mu": 0.05},
    "JPM":   {"sigma": 0.18, "mu": 0.04},  # Low volatility (bank)
    "V":     {"sigma": 0.17, "mu": 0.04},  # Low volatility (payments)
    "NFLX":  {"sigma": 0.35, "mu": 0.05},
}

DEFAULT_PARAMS: dict[str, float] = {"sigma": 0.25, "mu": 0.05}
```

Tickers added dynamically (not in `SEED_PRICES`) start at a random price between $50–$300 using `DEFAULT_PARAMS`.

## GBMSimulator Implementation

```python
import math
import random
import logging
import numpy as np

from .seed_prices import (
    SEED_PRICES, TICKER_PARAMS, DEFAULT_PARAMS,
    CORRELATION_GROUPS, INTRA_TECH_CORR, INTRA_FINANCE_CORR,
    CROSS_GROUP_CORR, TSLA_CORR,
)

logger = logging.getLogger(__name__)

class GBMSimulator:
    """Generates correlated GBM price paths for multiple tickers."""

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
        """Advance all tickers by one time step. Returns {ticker: new_price}.
        This is the hot path — called every 500ms."""
        n = len(self._tickers)
        if n == 0:
            return {}

        z_independent = np.random.standard_normal(n)
        z_correlated = self._cholesky @ z_independent if self._cholesky is not None else z_independent

        result: dict[str, float] = {}
        for i, ticker in enumerate(self._tickers):
            mu = self._params[ticker]["mu"]
            sigma = self._params[ticker]["sigma"]

            # GBM: S(t+dt) = S(t) * exp((mu - 0.5*sigma^2)*dt + sigma*sqrt(dt)*Z)
            drift = (mu - 0.5 * sigma**2) * self._dt
            diffusion = sigma * math.sqrt(self._dt) * z_correlated[i]
            self._prices[ticker] *= math.exp(drift + diffusion)

            # Random event: ~0.1% chance per tick
            if random.random() < self._event_prob:
                shock = random.uniform(0.02, 0.05) * random.choice([-1, 1])
                self._prices[ticker] *= (1 + shock)
                logger.debug("Random event on %s: %.1f%%", ticker, shock * 100)

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

    # --- Internals ---

    def _add_ticker_internal(self, ticker: str) -> None:
        """Add without rebuilding Cholesky — for batch initialization."""
        self._tickers.append(ticker)
        self._prices[ticker] = SEED_PRICES.get(ticker, random.uniform(50.0, 300.0))
        self._params[ticker] = TICKER_PARAMS.get(ticker, dict(DEFAULT_PARAMS))

    def _rebuild_cholesky(self) -> None:
        """Rebuild Cholesky decomposition of the correlation matrix.
        Called on add/remove. O(n^2), n < 50 in practice."""
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
            return TSLA_CORR
        if t1 in tech and t2 in tech:
            return INTRA_TECH_CORR
        if t1 in finance and t2 in finance:
            return INTRA_FINANCE_CORR
        return CROSS_GROUP_CORR
```

## File Structure

```
backend/
  app/
    market/
      simulator.py      # GBMSimulator class + SimulatorDataSource wrapper
      seed_prices.py    # SEED_PRICES, TICKER_PARAMS, DEFAULT_PARAMS, CORRELATION_GROUPS constants
```

`seed_prices.py` contains only constants. `simulator.py` contains both the core `GBMSimulator` class and the `SimulatorDataSource` (`MarketDataSource` implementation that wraps `GBMSimulator` in an async loop — see `MARKET_INTERFACE.md`).

## Behavior Notes

- **Prices never go negative** — GBM uses `exp()`, which is always positive
- **Sub-cent moves per tick** — the tiny `dt` produces small, realistic per-tick moves that accumulate naturally
- **Calibrated volatility** — with `sigma=0.50` (TSLA), simulated intraday ranges match roughly what you'd see in real TSLA
- **Positive semi-definite guarantee** — the correlation matrix is valid (all diagonal 1.0, off-diagonal values in a consistent range), so Cholesky always succeeds
- **Event frequency** — with 10 tickers at 2 ticks/sec: `0.001 × 10 × 2 = 0.02 events/sec` → roughly one event every **50 seconds** across the watchlist
- **Dynamic ticker add** — when a new ticker is added mid-session, Cholesky is rebuilt. The new ticker gets seeded from `SEED_PRICES` (or a random price) and appears in the cache immediately. Cost: O(n²), negligible for n < 50
- **No clock drift** — `dt` is a fixed constant, not derived from wall-clock time between steps. This keeps the math simple and deterministic regardless of event loop jitter

## Testing

The simulator is tested in `backend/tests/market/`:

- `test_simulator.py` — 17 tests: GBM math correctness, event probability, add/remove ticker, Cholesky rebuild, price bounds
- `test_simulator_source.py` — 10 integration tests: start/stop lifecycle, async add/remove, cache seeding

Run: `cd backend && uv run --extra dev pytest tests/market/ -v`
