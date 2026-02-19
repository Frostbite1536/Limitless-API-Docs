# Python SDK Guide

The **`limitless-sdk`** is the official Python SDK for the Limitless Exchange API. It is the **preferred** way to interact with the API for Python developers — it handles authentication, EIP-712 signing, venue caching, and order management automatically.

> **The SDK is preferred over raw API calls.** It eliminates the most common sources of errors (incorrect EIP-712 signing, missing venue data, wrong address checksums) and significantly reduces boilerplate code.

## Installation

```bash
pip install limitless-sdk==1.0.0
```

Requires Python >= 3.8.

## Quick Start

```python
import asyncio
import os
from limitless_sdk.api import HttpClient
from limitless_sdk.markets import MarketFetcher
from limitless_sdk.portfolio import PortfolioFetcher

async def main():
    # API key loaded automatically from LIMITLESS_API_KEY env variable
    http_client = HttpClient(base_url="https://api.limitless.exchange")

    try:
        # Get markets
        market_fetcher = MarketFetcher(http_client)
        markets = await market_fetcher.get_active_markets()
        print(f"Found {markets.total_markets_count} markets")

        # Fetch specific market (caches venue data automatically)
        market = await market_fetcher.get_market("bitcoin-2024")
        print(f"Market: {market.title}")

        # Get positions (authenticated via API key)
        portfolio_fetcher = PortfolioFetcher(http_client)
        positions = await portfolio_fetcher.get_positions()
        print(f"CLOB positions: {len(positions['clob'])}")

    finally:
        await http_client.close()

if __name__ == "__main__":
    asyncio.run(main())
```

## Authentication

The SDK uses API keys (same keys generated via the Limitless Exchange UI).

```python
from limitless_sdk.api import HttpClient

# Option 1: Automatic from LIMITLESS_API_KEY env variable (recommended)
http_client = HttpClient()

# Option 2: Explicit API key
http_client = HttpClient(api_key=os.getenv("LIMITLESS_API_KEY"))

# Option 3: Custom base URL (dev/staging)
http_client = HttpClient(
    base_url="https://staging.api.limitless.exchange",
    api_key="sk_test_..."
)
```

Environment variables (`.env` file):

```bash
LIMITLESS_API_KEY=lmts_your_api_key_here
PRIVATE_KEY=0x...  # For order signing
```

## Market Data

```python
from limitless_sdk.markets import MarketFetcher

market_fetcher = MarketFetcher(http_client)

# Browse active markets (paginated)
markets = await market_fetcher.get_active_markets({"page": 1, "limit": 50})
print(f"Total: {markets.total_markets_count}")

# Get specific market (automatically caches venue data)
market = await market_fetcher.get_market("market-slug")
print(f"Title: {market.title}")
print(f"YES Token: {market.tokens.yes}")
print(f"NO Token: {market.tokens.no}")

# Get orderbook
orderbook = await market_fetcher.get_orderbook("market-slug")
```

The SDK **automatically caches venue data** (exchange and adapter contract addresses) so you don't need to manage this yourself. This eliminates the most common source of errors.

## Placing Orders

The SDK handles EIP-712 signing, venue resolution, and user data fetching automatically.

### GTC Orders (Good-Till-Cancelled)

```python
from eth_account import Account
from limitless_sdk.orders import OrderClient
from limitless_sdk.types import Side, OrderType

account = Account.from_key(os.environ["PRIVATE_KEY"])

# OrderClient fetches user data automatically on first order
order_client = OrderClient(http_client=http_client, wallet=account)

# Get token ID from market
token_id = str(market.tokens.yes)  # or market.tokens.no

# Create BUY GTC order
order = await order_client.create_order(
    token_id=token_id,
    price=0.50,      # Price per share (0.01-0.99)
    size=5.0,         # Number of shares
    side=Side.BUY,
    order_type=OrderType.GTC,
    market_slug=market.slug
)

print(f"Order ID: {order.order.id}")
print(f"Status: {order.order.status}")
```

### FOK Orders (Fill-Or-Kill)

FOK orders execute immediately and completely, or are cancelled.

```python
# FOK BUY — maker_amount = total USDC to spend
order = await order_client.create_order(
    token_id=token_id,
    maker_amount=10.0,   # Spend $10 USDC
    side=Side.BUY,
    order_type=OrderType.FOK,
    market_slug=market.slug
)

# FOK SELL — maker_amount = number of shares to sell
order = await order_client.create_order(
    token_id=token_id,
    maker_amount=18.64,  # Sell 18.64 shares
    side=Side.SELL,
    order_type=OrderType.FOK,
    market_slug=market.slug
)

# Check if filled
if order.maker_matches and len(order.maker_matches) > 0:
    print(f"FILLED: {len(order.maker_matches)} matches")
else:
    print("NOT FILLED (cancelled)")
```

### Cancel Orders

```python
# Cancel single order
await order_client.cancel(order_id)

# Cancel all orders for a market
await order_client.cancel_all(market_slug)
```

## Portfolio

```python
from limitless_sdk.portfolio import PortfolioFetcher

portfolio_fetcher = PortfolioFetcher(http_client)

# Get positions
positions = await portfolio_fetcher.get_positions()
for pos in positions['clob']:
    print(f"Market: {pos['market']['title']}")
    print(f"Size: {pos['size']}")

# Access points
print(f"Points: {positions['accumulativePoints']}")
```

## Token Approvals

Before placing orders, you must approve tokens for the exchange contracts. This is a one-time setup per wallet.

| Order Type | Market Type | Approve To |
|------------|-------------|------------|
| BUY | All CLOB | USDC → `venue.exchange` |
| SELL | Simple CLOB | CT → `venue.exchange` |
| SELL | NegRisk | CT → `venue.exchange` AND `venue.adapter` |

The SDK includes a setup script:

```bash
python examples/00_setup_approvals.py
```

Or manually using `web3` + the SDK's constants:

```python
from limitless_sdk.utils.constants import get_contract_address

usdc_address = get_contract_address("USDC", 8453)
ctf_address = get_contract_address("CTF", 8453)
```

## WebSocket Support

```python
from limitless_sdk.websocket import WebSocketClient, WebSocketConfig

config = WebSocketConfig(
    url="wss://ws.limitless.exchange",
    auto_reconnect=True,
    reconnect_delay=1.0
)
ws_client = WebSocketClient(config=config)

@ws_client.on('connect')
async def on_connect():
    print("Connected")

@ws_client.on('orderbookUpdate')
async def on_orderbook_update(data):
    orderbook = data.get('orderbook', data)
    best_bid = orderbook['bids'][0]['price']
    best_ask = orderbook['asks'][0]['price']
    print(f"Bid: {best_bid:.4f} | Ask: {best_ask:.4f}")

await ws_client.connect()
await ws_client.subscribe('subscribe_market_prices', {'marketSlugs': [market_slug]})
```

## Error Handling

```python
from limitless_sdk.api import APIError

try:
    order = await order_client.create_order(...)
except APIError as e:
    print(f"Status: {e.status_code}")
    print(f"Error: {e}")  # Raw API response JSON
```

### Retry Logic

```python
from limitless_sdk.api import retry_on_errors

@retry_on_errors(
    status_codes={500, 429},
    max_retries=3,
    delays=[1, 2, 3],
    on_retry=lambda attempt, error, delay: print(f"Retry {attempt+1}/3")
)
async def fetch_data():
    return await http_client.get("/endpoint")
```

## Debug Logging

```python
from limitless_sdk.types import ConsoleLogger, LogLevel

logger = ConsoleLogger(level=LogLevel.DEBUG)
http_client = HttpClient(base_url="...", logger=logger)

# Logs venue cache operations, request headers, etc.
```

## SDK vs Raw API

| Aspect | SDK (`limitless-sdk`) | Raw API (`requests`) |
|--------|----------------------|---------------------|
| EIP-712 signing | Automatic | Manual (error-prone) |
| Venue caching | Automatic | Must implement yourself |
| Authentication | Automatic header injection | Manual header management |
| Address checksumming | Handled internally | Must ensure EIP-55 format |
| Order construction | `create_order()` with simple params | Build full payload manually |
| WebSocket | Built-in client with reconnect | Manual socket.io setup |
| Error types | Typed exceptions (`APIError`) | Raw HTTP status codes |
| Async support | Native async/await | Synchronous by default |

## What the SDK Does NOT Cover

- **On-chain operations**: `mergePositions()` and `redeemPositions()` are smart contract calls — use `web3.py` directly. See [Claiming & Redeeming Guide](claiming-redeeming.md).
- **Token approvals**: One-time setup using `web3.py` (helper script provided in `examples/00_setup_approvals.py`).

## SDK Architecture

| Component | Purpose |
|-----------|---------|
| `HttpClient` | HTTP client with API key auth, retries |
| `MarketFetcher` | Market data with venue caching |
| `OrderClient` | Order creation/cancellation with automatic signing |
| `OrderSigner` | EIP-712 message signing |
| `PortfolioFetcher` | Portfolio and positions |
| `WebSocketClient` | Real-time orderbook updates |
| `RetryableClient` | Auto-retry wrapper |

## Examples

The SDK ships with working examples:

| Example | Description |
|---------|-------------|
| `00_setup_approvals.py` | Token approval setup |
| `01_authentication.py` | API key auth with portfolio |
| `02_create_buy_gtc_order.py` | GTC BUY order |
| `03_cancel_gtc_order.py` | Cancel orders |
| `04_create_sell_gtc_order.py` | GTC SELL order |
| `05_create_buy_fok_order.py` | FOK BUY order |
| `06_create_sell_fok_order.py` | FOK SELL order |
| `06_retry_handling.py` | Custom retry logic |
| `07_auto_retry_second_sample.py` | Auto-retry patterns |
| `08_websocket_events.py` | Real-time WebSocket events |

## Links

- **PyPI**: https://pypi.org/project/limitless-sdk/
- **Source**: https://github.com/limitless-labs-group/limitless-exchange-ts-sdk

## Disclaimer

USE AT YOUR OWN RISK. Trading on prediction markets involves financial risk. Test all functionality with small amounts first. Limitless restricts order placement from US locations due to regulatory requirements.
