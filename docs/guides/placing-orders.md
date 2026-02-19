# Guide: Placing Orders

Step-by-step guide to placing orders on the Limitless Exchange.

> **Using Python?** The [`limitless-sdk`](sdk.md) handles all of this automatically — venue caching, EIP-712 signing, and order submission in a single call. See the [SDK Guide](sdk.md#placing-orders) for examples.

## Overview

Placing an order on Limitless Exchange requires:

1. **Authentication** - Use API key (recommended) or wallet login (deprecated)
2. **Market Data** - Get position IDs for the market
3. **Amount Calculation** - Calculate maker/taker amounts
4. **Order Signing** - Sign order with EIP-712
5. **Submission** - Submit to the API

The SDK reduces this to a single `create_order()` call. The manual steps below are for non-Python languages or when you need direct API access.

## Step 1: Authentication

Before placing orders, set up authentication. API keys are recommended.

### Using API Key (Recommended)

```python
import requests

API_URL = "https://api.limitless.exchange"
API_KEY = "lmts_your_key_here"  # Generate at limitless.exchange (profile → Api keys)

# All authenticated requests use this header
auth_headers = {"X-API-Key": API_KEY}
```

No login flow needed. Include `auth_headers` in all authenticated requests.

### Using Cookie Session (Deprecated)

> **WARNING**: Cookie-based authentication is deprecated and will be removed within weeks.

```python
from eth_account import Account
from eth_account.messages import encode_defunct

def authenticate(private_key):
    account = Account.from_key(private_key)

    # Get signing message
    response = requests.get(f"{API_URL}/auth/signing-message")
    signing_message = response.text

    # Sign message
    message = encode_defunct(text=signing_message)
    signature = account.sign_message(message)

    # Login
    headers = {
        'x-account': account.address,
        'x-signing-message': '0x' + signing_message.encode().hex(),
        'x-signature': '0x' + signature.signature.hex()
    }

    response = requests.post(
        f"{API_URL}/auth/login",
        headers=headers,
        json={"client": "eoa"}
    )

    session = response.cookies.get('limitless_session')
    user = response.json()

    return session, user
```

## Step 2: Get Market Data

Fetch market details to get position IDs and venue information.

```python
# Cache for market data (venue is static per market)
MARKET_CACHE = {}

def get_market(slug):
    if slug in MARKET_CACHE:
        return MARKET_CACHE[slug]

    response = requests.get(f"{API_URL}/markets/{slug}")
    market = response.json()
    MARKET_CACHE[slug] = market
    return market

market = get_market("btc-100k-2024")

# Get venue exchange for EIP-712 signing (CRITICAL)
venue_exchange = market['venue']['exchange']

# Extract position IDs (positionIds[0] = YES, positionIds[1] = NO)
position_ids = market['positionIds']
yes_token_id = position_ids[0]
no_token_id = position_ids[1]
```

**Key Fields**:
- `positionIds[0]` - YES token ID
- `positionIds[1]` - NO token ID
- `venue.exchange` - Use as `verifyingContract` in EIP-712 signing
- `venue.adapter` - Needed for SELL orders on NegRisk markets
- `slug` - Market identifier

**IMPORTANT**: Cache market data per slug - venue info is static and doesn't change.

## Step 3: Calculate Amounts

Calculate maker and taker amounts based on your trade.

```python
def calculate_amounts(price_cents, shares):
    """
    Calculate order amounts.

    Args:
        price_cents: Price per share in cents (e.g., 65 = $0.65)
        shares: Number of shares to buy/sell

    Returns:
        maker_amount: USDC you're paying (in 6 decimals)
        taker_amount: Shares you're receiving (in 6 decimals)
    """
    SCALING_FACTOR = 1_000_000  # USDC has 6 decimals

    price_dollars = price_cents / 100
    total_cost = price_dollars * shares

    maker_amount = int(total_cost * SCALING_FACTOR)
    taker_amount = int(shares * SCALING_FACTOR)

    return maker_amount, taker_amount

# Example: Buy 100 YES shares at 65 cents each
maker_amount, taker_amount = calculate_amounts(65, 100)
# maker_amount = 65,000,000 (65 USDC)
# taker_amount = 100,000,000 (100 shares)
```

### Price Calculation

| You Want | Price | Shares | Maker Amount | Taker Amount |
|----------|-------|--------|--------------|--------------|
| Buy 100 YES @ $0.65 | 65c | 100 | 65,000,000 | 100,000,000 |
| Buy 50 YES @ $0.80 | 80c | 50 | 40,000,000 | 50,000,000 |
| Buy 200 NO @ $0.25 | 25c | 200 | 50,000,000 | 200,000,000 |

## Step 4: Create Order Payload

Build the order structure (without signature).

```python
import time
import random

def create_order_payload(maker, token_id, maker_amount, taker_amount, side=0):
    """
    Create order payload for signing.

    Args:
        maker: Your Ethereum address
        token_id: YES or NO position ID
        maker_amount: USDC amount (6 decimals)
        taker_amount: Shares amount (6 decimals)
        side: 0 = BUY, 1 = SELL

    Returns:
        Order payload dict
    """
    return {
        "salt": int(time.time() * 1000) + random.randint(0, 1000),
        "maker": maker,
        "signer": maker,
        "taker": "0x0000000000000000000000000000000000000000",
        "tokenId": str(token_id),
        "makerAmount": str(maker_amount),
        "takerAmount": str(taker_amount),
        "expiration": "0",
        "nonce": "0",
        "feeRateBps": "0",
        "side": side,
        "signatureType": 0
    }
```

### Order Fields Explained

| Field | Description |
|-------|-------------|
| `salt` | Random value for signature uniqueness |
| `maker` | Your wallet address |
| `signer` | Address signing the order (same as maker for EOA) |
| `taker` | Zero address for open orders |
| `tokenId` | Position ID from market data |
| `makerAmount` | What you're paying |
| `takerAmount` | What you're receiving |
| `expiration` | "0" for no expiration |
| `nonce` | Cancellation nonce |
| `feeRateBps` | Fee rate in basis points |
| `side` | 0 = BUY, 1 = SELL |
| `signatureType` | 0 = EOA |

## Step 5: Sign Order (EIP-712)

Sign the order using EIP-712 typed data with the **venue's exchange address**.

```python
from eth_account.messages import encode_typed_data

CHAIN_ID = 8453

def sign_order(private_key, order_payload, venue_exchange):
    """
    Sign order using EIP-712 with venue's exchange address.

    IMPORTANT: venue_exchange must be fetched from market data via:
    venue_exchange = market['venue']['exchange']
    """
    account = Account.from_key(private_key)

    # Use venue.exchange from market data as verifyingContract
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

    signable = encode_typed_data(domain, types, order_payload)
    signed = account.sign_message(signable)

    return '0x' + signed.signature.hex()
```

### Venue System (CRITICAL)

Each CLOB market has a `venue` object with contract addresses. You **must** fetch this from market data:

```python
market = get_market("btc-100k-2024")
venue_exchange = market['venue']['exchange']  # Use as verifyingContract
venue_adapter = market['venue']['adapter']    # For NegRisk SELL orders
```

**Do NOT hardcode contract addresses** - always fetch them from the market's venue data.

## Step 6: Submit Order

Submit the signed order to the API.

```python
def submit_order(api_key, order_payload, owner_id, market_slug, order_type="GTC"):
    """
    Submit order to API.

    Args:
        api_key: API key (lmts_...)
        order_payload: Signed order payload
        owner_id: User profile ID
        market_slug: Market identifier
        order_type: "GTC" or "FOK"

    Returns:
        Order response
    """
    payload = {
        "order": order_payload,
        "ownerId": owner_id,
        "orderType": order_type,
        "marketSlug": market_slug
    }

    response = requests.post(
        f"{API_URL}/orders",
        json=payload,
        headers={"X-API-Key": api_key}
    )

    if not response.ok:
        raise Exception(f"Order failed: {response.status_code} {response.text}")

    return response.json()
```

### Order Types

| Type | Behavior |
|------|----------|
| GTC | Good Till Cancelled - stays open until filled or cancelled |
| FOK | Fill Or Kill - must fill completely or fails |

## Complete Example

```python
import os

def place_order(market_slug, token_type, price_cents, shares):
    """
    Place a complete order.

    Args:
        market_slug: Market identifier
        token_type: "YES" or "NO"
        price_cents: Price in cents
        shares: Number of shares
    """
    api_key = os.environ['LIMITLESS_API_KEY']
    private_key = os.environ['PRIVATE_KEY']
    maker = Account.from_key(private_key).address

    # Step 1: Get market data (cached per market)
    market = get_market(market_slug)

    # Get venue exchange for EIP-712 signing (CRITICAL)
    venue_exchange = market['venue']['exchange']

    # Get token ID (positionIds[0] = YES, positionIds[1] = NO)
    token_id = market['positionIds'][0 if token_type == "YES" else 1]

    # Step 2: Calculate amounts
    maker_amount, taker_amount = calculate_amounts(price_cents, shares)

    # Step 3: Create order
    order = create_order_payload(maker, token_id, maker_amount, taker_amount)

    # Step 4: Sign order using venue's exchange address
    signature = sign_order(private_key, order, venue_exchange)
    order['signature'] = signature
    order['price'] = price_cents / 100  # Required for GTC

    # Step 5: Submit with API key
    result = submit_order(api_key, order, owner_id, market_slug, "GTC")

    return result

# Usage
result = place_order(
    market_slug="btc-100k-2024",
    token_type="YES",
    price_cents=65,
    shares=100
)
print(f"Order ID: {result['order']['orderId']}")
```

## Troubleshooting

### Signature Verification Failed

- Verify you're using `venue.exchange` from market data (not a hardcoded address)
- Check chain ID is 8453 (Base)
- Ensure all fields are properly converted to strings
- Make sure you fetched market data to get the correct venue

### Order Not Found

- Check market slug is correct
- Verify market is still active (not resolved)

### Insufficient Balance

- Check USDC balance
- Verify USDC approval for CTF contract

### Invalid Price

- Price must be between 0.01 and 0.99
- For GTC orders, price field is required

## Next Steps

- [Cancel Orders](../endpoints/orders.md#delete-ordersorderid)
- [View Positions](../endpoints/portfolio.md#get-portfoliopositions)
- [Real-time Updates](websockets.md)
