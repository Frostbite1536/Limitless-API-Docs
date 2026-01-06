# Python Quickstart Guide

Complete guide to trading on Limitless Exchange using Python.

## Prerequisites

### Required Packages

```bash
pip install requests eth-account web3
```

### Environment Variables

```bash
export PRIVATE_KEY="0x..."  # Your wallet private key
```

**Security Warning**: Never commit private keys to version control.

## Complete Trading Example

### 1. Authentication Module

```python
import requests
from eth_account import Account
from eth_account.messages import encode_defunct

API_URL = "https://api.limitless.exchange"

def string_to_hex(text):
    """Convert string to hex with 0x prefix."""
    return '0x' + text.encode('utf-8').hex()

def get_signing_message():
    """Get signing message with nonce from API."""
    response = requests.get(f"{API_URL}/auth/signing-message")
    response.raise_for_status()
    return response.text

def authenticate(private_key):
    """Authenticate and return session cookie + user data."""
    account = Account.from_key(private_key)
    address = account.address

    # Get and sign message
    signing_message = get_signing_message()
    message = encode_defunct(text=signing_message)
    signature = account.sign_message(message)

    # Prepare headers
    headers = {
        'x-account': address,
        'x-signing-message': string_to_hex(signing_message),
        'x-signature': '0x' + signature.signature.hex(),
        'Content-Type': 'application/json'
    }

    # Login request
    response = requests.post(
        f"{API_URL}/auth/login",
        headers=headers,
        json={"client": "eoa"}
    )
    response.raise_for_status()

    # Extract session cookie
    session_cookie = response.cookies.get('limitless_session')
    user_data = response.json()

    return session_cookie, user_data
```

### 2. Market Data Module

```python
# Cache for market data (venue is static per market)
MARKET_CACHE = {}

def get_market(slug):
    """Fetch market details by slug (cached)."""
    if slug in MARKET_CACHE:
        return MARKET_CACHE[slug]

    response = requests.get(f"{API_URL}/markets/{slug}")
    response.raise_for_status()
    market = response.json()
    MARKET_CACHE[slug] = market
    return market

def get_venue_exchange(slug):
    """Get venue exchange address for EIP-712 signing."""
    market = get_market(slug)
    return market["venue"]["exchange"]

def get_orderbook(slug):
    """Fetch current orderbook."""
    response = requests.get(f"{API_URL}/markets/{slug}/orderbook")
    response.raise_for_status()
    return response.json()

def get_active_markets(page=1, limit=10, sort_by="volume"):
    """Browse active markets."""
    response = requests.get(
        f"{API_URL}/markets/active",
        params={"page": page, "limit": limit, "sortBy": sort_by}
    )
    response.raise_for_status()
    return response.json()
```

### 3. EIP-712 Order Signing

```python
from eth_account.messages import encode_typed_data
import time
import random

CHAIN_ID = 8453  # Base mainnet

def create_order_payload(maker, token_id, maker_amount, taker_amount, fee_rate_bps=0):
    """Create order payload without signature."""
    return {
        "salt": int(time.time() * 1000) + random.randint(0, 1000),
        "maker": maker,
        "signer": maker,
        "taker": "0x0000000000000000000000000000000000000000",
        "tokenId": token_id,
        "makerAmount": str(maker_amount),
        "takerAmount": str(taker_amount),
        "expiration": "0",
        "nonce": "0",
        "feeRateBps": str(fee_rate_bps),
        "side": 0,  # 0 = BUY
        "signatureType": 0  # EOA
    }

def sign_order(private_key, order_payload, venue_exchange):
    """Sign order using EIP-712 with venue's exchange address."""
    account = Account.from_key(private_key)

    # IMPORTANT: Use venue.exchange from market data as verifyingContract
    domain = {
        "name": "Limitless CTF Exchange",
        "version": "1",
        "chainId": CHAIN_ID,
        "verifyingContract": venue_exchange  # From market's venue data
    }

    types = {
        "Order": [
            {"name": "salt", "type": "uint256"},
            {"name": "maker", "type": "address"},
            {"name": "signer", "type": "address"},
            {"name": "taker", "type": "address"},
            {"name": "tokenId", "type": "uint256"},
            {"name": "makerAmount", "type": "uint256"},
            {"name": "takerAmount", "type": "uint256"},
            {"name": "expiration", "type": "uint256"},
            {"name": "nonce", "type": "uint256"},
            {"name": "feeRateBps", "type": "uint256"},
            {"name": "side", "type": "uint8"},
            {"name": "signatureType", "type": "uint8"}
        ]
    }

    # Sign typed data
    signable = encode_typed_data(domain, types, order_payload)
    signed = account.sign_message(signable)

    return '0x' + signed.signature.hex()
```

### 4. Order Management

```python
def create_order(session_cookie, order_payload, owner_id, market_slug, order_type="GTC"):
    """Submit order to API."""
    payload = {
        "order": order_payload,
        "ownerId": owner_id,
        "orderType": order_type,
        "marketSlug": market_slug
    }

    response = requests.post(
        f"{API_URL}/orders",
        json=payload,
        cookies={"limitless_session": session_cookie}
    )
    response.raise_for_status()
    return response.json()

def cancel_order(session_cookie, order_id):
    """Cancel a specific order."""
    response = requests.delete(
        f"{API_URL}/orders/{order_id}",
        cookies={"limitless_session": session_cookie}
    )
    response.raise_for_status()
    return response.json()

def cancel_all_orders(session_cookie, market_slug):
    """Cancel all orders in a market."""
    response = requests.delete(
        f"{API_URL}/orders/all/{market_slug}",
        cookies={"limitless_session": session_cookie}
    )
    response.raise_for_status()
    return response.json()
```

### 5. Portfolio Functions

```python
def get_positions(session_cookie):
    """Get all portfolio positions."""
    response = requests.get(
        f"{API_URL}/portfolio/positions",
        cookies={"limitless_session": session_cookie}
    )
    response.raise_for_status()
    return response.json()

def get_trades(session_cookie):
    """Get trade history."""
    response = requests.get(
        f"{API_URL}/portfolio/trades",
        cookies={"limitless_session": session_cookie}
    )
    response.raise_for_status()
    return response.json()
```

### 6. Main Trading Function

```python
import os

def execute_trade(market_slug, token_type, price_cents, amount):
    """
    Execute a complete trade.

    Args:
        market_slug: Market identifier
        token_type: "YES" or "NO"
        price_cents: Price in cents (e.g., 65 = $0.65)
        amount: Number of shares
    """
    private_key = os.environ.get("PRIVATE_KEY")
    if not private_key:
        raise ValueError("PRIVATE_KEY environment variable not set")

    # Step 1: Authenticate
    print("Authenticating...")
    session_cookie, user_data = authenticate(private_key)
    owner_id = user_data.get("id")
    maker_address = user_data.get("account")  # Already checksummed
    fee_rate_bps = user_data.get("rank", {}).get("feeRateBps", 0)

    # Step 2: Get market data (cached per market)
    print(f"Fetching market: {market_slug}")
    market = get_market(market_slug)

    # Get venue exchange for EIP-712 signing
    venue_exchange = market["venue"]["exchange"]
    print(f"Using venue exchange: {venue_exchange}")

    # Get token ID (positionIds[0] = YES, positionIds[1] = NO)
    position_ids = market.get("positionIds", [])
    token_id = position_ids[0] if token_type == "YES" else position_ids[1]

    # Step 3: Calculate amounts
    price_dollars = price_cents / 100
    total_cost = price_dollars * amount
    scaling_factor = 1_000_000  # USDC decimals

    maker_amount = int(total_cost * scaling_factor)
    taker_amount = int(amount * scaling_factor)

    print(f"Buying {amount} {token_type} at ${price_dollars}")
    print(f"Total cost: ${total_cost}")

    # Step 4: Create and sign order
    order_payload = create_order_payload(
        maker=maker_address,
        token_id=token_id,
        maker_amount=maker_amount,
        taker_amount=taker_amount,
        fee_rate_bps=fee_rate_bps
    )

    # Sign using venue's exchange address as verifyingContract
    signature = sign_order(private_key, order_payload, venue_exchange)
    order_payload["signature"] = signature
    order_payload["price"] = price_dollars

    # Step 5: Submit order
    print("Submitting order...")
    result = create_order(
        session_cookie=session_cookie,
        order_payload=order_payload,
        owner_id=owner_id,
        market_slug=market_slug,
        order_type="GTC"
    )

    print(f"Order created: {result}")
    return result

# Example usage
if __name__ == "__main__":
    result = execute_trade(
        market_slug="btc-100k-2024",
        token_type="YES",
        price_cents=65,
        amount=100
    )
```

## Common Operations

### Get Best Bid/Ask

```python
def get_best_prices(slug):
    orderbook = get_orderbook(slug)
    best_bid = orderbook['bids'][0]['price'] if orderbook['bids'] else None
    best_ask = orderbook['asks'][0]['price'] if orderbook['asks'] else None
    spread = best_ask - best_bid if best_bid and best_ask else None
    return {"bid": best_bid, "ask": best_ask, "spread": spread}
```

### Calculate Portfolio Value

```python
def calculate_portfolio_value(session_cookie):
    positions = get_positions(session_cookie)
    total_value = 0

    for pos in positions['clob']:
        for outcome in ['yes', 'no']:
            market_value = int(pos['positions'][outcome]['marketValue'])
            total_value += market_value

    return total_value / 1_000_000  # Convert to dollars
```

## Error Handling

```python
import requests

def safe_api_call(func, *args, **kwargs):
    """Wrapper for API calls with error handling."""
    try:
        return func(*args, **kwargs)
    except requests.exceptions.HTTPError as e:
        print(f"HTTP Error: {e.response.status_code}")
        print(f"Response: {e.response.text}")
        raise
    except requests.exceptions.ConnectionError:
        print("Connection error - check your internet")
        raise
    except Exception as e:
        print(f"Unexpected error: {e}")
        raise
```

## Disclaimer

This code is for educational purposes only. Limitless Labs is not responsible for any losses. Always test with small amounts first.
