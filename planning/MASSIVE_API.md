# Massive API Reference (formerly Polygon.io)

Reference documentation for the Massive (formerly Polygon.io) REST API as used in FinAlly.

## Overview

Polygon.io rebranded to **Massive** on October 30, 2025. Both base URLs are supported:

- `https://api.massive.com` (current)
- `https://api.polygon.io` (legacy, still functional)

**Python package**: `massive` (`uv add massive`)
**Min Python**: 3.9+
**Auth**: API key via `MASSIVE_API_KEY` env var, or pass `api_key=` to `RESTClient`. The client sends it as an `Authorization: Bearer <key>` header automatically.

## Rate Limits

| Tier | Limit |
|------|-------|
| Free | 5 requests/minute — EOD data only |
| Paid (all) | Unlimited |

For FinAlly: free tier → poll every 15s. Paid → poll every 2–5s.

Free tier also restricts access to **end-of-day** data only. During market hours the snapshot endpoint still returns the last known trade price, but it may lag behind real-time.

## Client Initialization

```python
from massive import RESTClient

# Reads MASSIVE_API_KEY from environment automatically
client = RESTClient()

# Or pass explicitly
client = RESTClient(api_key="your_key_here")
```

The client is **synchronous**. In an async FastAPI app, run it in a thread:

```python
import asyncio
snapshots = await asyncio.to_thread(client.get_snapshot_all, ...)
```

## Endpoints Used in FinAlly

### 1. Snapshot — Multiple Tickers (Primary Endpoint)

Gets current prices for a list of tickers in **one API call**. This is the only endpoint FinAlly calls in its poll loop.

**REST**: `GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,GOOGL,MSFT`

**Python**:
```python
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient()

snapshots = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA"],
)

for snap in snapshots:
    price = snap.last_trade.price
    # Timestamp is Unix milliseconds — convert to seconds
    ts = snap.last_trade.timestamp / 1000.0
    prev_close = snap.day.previous_close
    change_pct = snap.day.change_percent

    print(f"{snap.ticker}: ${price:.2f}  ({change_pct:+.2f}% today)")
```

**Response structure per ticker** (simplified):
```json
{
  "ticker": "AAPL",
  "day": {
    "open": 189.20,
    "high": 192.10,
    "low": 188.50,
    "close": 190.85,
    "volume": 68432100,
    "volume_weighted_average_price": 190.44,
    "previous_close": 188.30,
    "change": 2.55,
    "change_percent": 1.35
  },
  "last_trade": {
    "price": 190.85,
    "size": 100,
    "exchange": "XNAS",
    "timestamp": 1707939600000
  },
  "last_quote": {
    "bid_price": 190.84,
    "ask_price": 190.86,
    "bid_size": 300,
    "ask_size": 200,
    "timestamp": 1707939600500
  },
  "prev_daily_bar": {
    "open": 187.10,
    "high": 189.90,
    "low": 186.80,
    "close": 188.30,
    "volume": 71200000
  }
}
```

**Fields we extract**:
| Field | Use |
|-------|-----|
| `last_trade.price` | Current price for trading and display |
| `last_trade.timestamp` | When the price was recorded (ms → convert to seconds) |
| `day.previous_close` | Day change baseline |
| `day.change_percent` | Day change % for display |

### 2. Single Ticker Snapshot

For detailed data on one ticker (e.g., user clicks for detail view).

```python
snapshot = client.get_snapshot_ticker(
    market_type=SnapshotMarketType.STOCKS,
    ticker="AAPL",
)

print(f"Price:    ${snapshot.last_trade.price}")
print(f"Bid/Ask:  ${snapshot.last_quote.bid_price} / ${snapshot.last_quote.ask_price}")
print(f"Day range: ${snapshot.day.low} – ${snapshot.day.high}")
```

### 3. Previous Close

Previous trading day's OHLCV for a ticker. Useful for seeding prices on startup.

**REST**: `GET /v2/aggs/ticker/{ticker}/prev`

```python
prev = client.get_previous_close_agg(ticker="AAPL")

for agg in prev:
    print(f"Prev close: ${agg.close}")
    print(f"OHLCV: O={agg.open} H={agg.high} L={agg.low} C={agg.close} V={agg.volume}")
```

**Raw response**:
```json
{
  "ticker": "AAPL",
  "results": [
    {
      "o": 187.10,
      "h": 189.90,
      "l": 186.80,
      "c": 188.30,
      "v": 71200000,
      "t": 1707868800000
    }
  ]
}
```

### 4. Historical Bars (Aggregates)

OHLCV bars over a date range. Not used in the live polling loop, but available for historical chart views.

**REST**: `GET /v2/aggs/ticker/{ticker}/range/{multiplier}/{timespan}/{from}/{to}`

```python
bars = []
for bar in client.list_aggs(
    ticker="AAPL",
    multiplier=1,
    timespan="day",
    from_="2024-01-01",
    to="2024-01-31",
    limit=50000,
):
    bars.append(bar)

for bar in bars:
    print(f"{bar.timestamp}: O={bar.open} H={bar.high} L={bar.low} C={bar.close} V={bar.volume}")
```

### 5. Last Trade / Last Quote

Individual endpoints for the most recent trade or NBBO quote (one ticker at a time).

```python
# Last trade
trade = client.get_last_trade(ticker="AAPL")
print(f"Last trade: ${trade.price} x {trade.size} shares")

# Last NBBO quote
quote = client.get_last_quote(ticker="AAPL")
print(f"Bid: ${quote.bid} x {quote.bid_size}")
print(f"Ask: ${quote.ask} x {quote.ask_size}")
```

Note: prefer `get_snapshot_all()` for multiple tickers — it's one API call vs. N calls.

## How FinAlly Uses the API

The `MassiveDataSource` runs as an asyncio background task:

1. Collect all current watchlist tickers
2. Call `get_snapshot_all()` with those tickers (one API call)
3. Extract `last_trade.price` and `last_trade.timestamp` from each snapshot
4. Write to the shared `PriceCache`
5. Sleep for `poll_interval` seconds, then repeat

```python
import asyncio
from massive import RESTClient
from massive.rest.models import SnapshotMarketType
from app.market.cache import PriceCache

async def poll_once(client: RESTClient, tickers: list[str], cache: PriceCache) -> None:
    """One poll cycle — wraps synchronous client in thread."""
    snapshots = await asyncio.to_thread(
        client.get_snapshot_all,
        market_type=SnapshotMarketType.STOCKS,
        tickers=tickers,
    )
    for snap in snapshots:
        cache.update(
            ticker=snap.ticker,
            price=snap.last_trade.price,
            timestamp=snap.last_trade.timestamp / 1000.0,  # ms → seconds
        )
```

See `backend/app/market/massive_client.py` for the full implementation.

## Error Handling

The client raises exceptions for HTTP errors:

| Status | Meaning |
|--------|---------|
| 401 | Invalid API key |
| 403 | Plan doesn't include this endpoint |
| 429 | Rate limit exceeded — free tier: 5 req/min |
| 5xx | Server error (client retries up to 3 times by default) |

FinAlly catches all exceptions in `_poll_once()` and logs them rather than crashing. The poller continues on the next interval. This handles:
- Transient network failures
- Temporary rate limit hits
- Invalid tickers (silently skipped with a warning)

## Raw REST Calls (without the SDK)

If you need to call the API without the `massive` package, use `requests` directly. Authentication is the same: `apiKey` query param or `Authorization: Bearer` header.

```python
import requests

API_KEY = "your_key"

# Snapshot for specific tickers
url = (
    "https://api.polygon.io/v2/snapshot/locale/us/markets/stocks/tickers"
    f"?tickers=AAPL,GOOGL,MSFT&apiKey={API_KEY}"
)
resp = requests.get(url)
data = resp.json()

for ticker_data in data.get("tickers", []):
    ticker = ticker_data["ticker"]
    price = ticker_data["lastTrade"]["p"]
    print(f"{ticker}: ${price}")
```

Note the raw REST response uses different field names than the Python SDK (`lastTrade.p` vs `last_trade.price`). The SDK normalizes everything to snake_case objects.

## Notes

- Snapshot endpoint fetches **all requested tickers in one call** — critical for staying within free-tier rate limits
- Timestamps from the API are Unix milliseconds; convert to seconds with `/ 1000.0`
- During market-closed hours, `last_trade.price` is the last traded price (may include after-hours)
- The `day` object resets at market open; during pre-market, values may be from the previous session
- Tickers must be uppercase (e.g., `"AAPL"` not `"aapl"`)
