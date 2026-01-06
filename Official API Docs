Open Search Search Keyboard Shortcut: `CTRL⌃ k`

- Open Group



Limitless Exchange Trading API

- Close Group



Authentication


  - Get signing message
    HTTP Method:  GET
  - Verify authentication
    HTTP Method:  GET
  - User login
    HTTP Method:  POST
  - User logout
    HTTP Method:  POST
- Open Group



Markets

- Open Group



Trading

- Open Group



Portfolio

- Open Group



Public Portfolio

- Open Group



Models


[Open API Client](https://client.scalar.com/?url=https%3A%2F%2Fapi.limitless.exchange%2Fapi-v1&integration=html&utm_source=api-reference&utm_medium=button&utm_campaign=sidebar)

[Powered by Scalar](https://www.scalar.com/)

v1.0

OAS 3.0.0

# Limitless Exchange API

[API Support](mailto:hey@limitless.network)

Download OpenAPI Document
json
Download OpenAPI Document
yaml

# Limitless Exchange Trading API

_Production-ready API for prediction market trading, portfolio management, and market data_

> 🎯 **Quick Navigation**: [Authentication](https://api.limitless.exchange/api-v1#tag/authentication) \| [Markets](https://api.limitless.exchange/api-v1#tag/markets) \| [Trading](https://api.limitless.exchange/api-v1#tag/trading) \| [Portfolio](https://api.limitless.exchange/api-v1#tag/portfolio)

* * *

## 🚀 Quick Start

Choose your preferred programming language for complete end-to-end implementation:

### Overview

The Limitless Exchange API offers both REST and WebSocket integration:

**REST API (Trading)**:

1. **🔐 Authentication**: Sign a message with your wallet to get authenticated
2. **📊 Fetch Market Data**: Get market info including venue contract addresses (once per market)
3. **📋 Order Creation**: Build and sign orders using EIP-712 structured data
4. **🚀 Order Submission**: Submit signed orders and receive confirmations

**WebSocket API (Real-Time Data)**:

1. **🔌 Connection**: Connect to `/markets` namespace for real-time updates
2. **📊 Subscriptions**: Subscribe to market prices and position changes
3. **📡 Events**: Handle live market data and transaction updates

### Important: Venue System for CLOB Markets

CLOB markets use a **venue system** where each market is associated with specific contract addresses. Before placing orders:

1. **Fetch market data once**: `GET /markets/:slug` returns venue information
2. **Use venue.exchange**: This is the `verifyingContract` for EIP-712 order signing
3. **Cache the venue**: Venue data is static per market - fetch once and reuse

**Sample venue response:**

```json
{
  "venue": {
    "exchange": "0xA1b2C3...",
    "adapter": "0xD4e5F6..."
  }
}
```

### Required Approvals

Before trading, set up token approvals based on order type:

| Order Type | Market Type | Approve To |
| --- | --- | --- |
| BUY | All CLOB | USDC → `venue.exchange` |
| SELL | Simple CLOB | CT → `venue.exchange` |
| SELL | NegRisk/Grouped | CT → `venue.exchange` AND `venue.adapter` |

### Checksummed Addresses

All addresses must use **checksummed format** (EIP-55 mixed-case):

- Authentication: `x-account` header
- Orders: `maker` and `signer` fields
- Example: `0x5aAeb6053F3E94C9b9A09f33669435E7Ef1BeAed`

### Implementation Guides

**[🐍 Python Quick Start](https://api.limitless.exchange/api-v1#description/-python-quick-start)**

- REST API: eth-account, requests, and web3.py libraries
- WebSocket: python-socketio, asyncio integration

**[☕ Java Quick Start](https://api.limitless.exchange/api-v1#description/-java-quick-start)**

- REST API: Web3j, OkHttp3, and Jackson libraries

**[📦 Node.js/TypeScript Quick Start](https://api.limitless.exchange/api-v1#description/-nodejs-typescript-quick-start)**

- REST API: viem, ethers, and cross-fetch libraries
- WebSocket: socket.io-client for real-time trading
- Full TypeScript support with end-to-end examples

**[🔌 WebSocket Integration](https://api.limitless.exchange/api-v1#description/-websocket-integration)**

- Real-time market data and position updates
- Production-ready Python client with authentication

* * *

## 🐍 Python Quick Start

Complete end-to-end Python implementation for Limitless Exchange API integration.

#### Python E2E Order Creation Guide

⚠️ **IMPORTANT DISCLAIMER** ✨ ⚠️

```properties
This is an example script for educational purposes only.
Limitless Labs is not responsible for any losses or mistakes.
This script should be adjusted to your personal needs and risk tolerance.
Always test with small amounts first and understand the code before using it.
USE AT YOUR OWN RISK.
```

This guide provides a complete walkthrough of creating orders on Limitless Exchange using Python, from authentication to order signing and submission.

#### Overview

The order creation process involves:

1. **Authentication** ✨: Get a signing message and authenticate with your wallet (use checksummed addresses)
2. **Fetch Market Data** ✨: Get venue information from `GET /markets/:slug` (cache per market)
3. **Order Construction** ✨: Build the order payload with proper parameters
4. **EIP-712 Signing** ✨: Sign the order using venue's exchange address as `verifyingContract`
5. **Order Submission** ✨: Submit the signed order to the API

#### Important: Venue System

CLOB markets use a **venue system** ✨ where each market has specific contract addresses. You must:

- Fetch market data via `GET /markets/:slug` to get `venue.exchange` and `venue.adapter`
- Use `venue.exchange` as the `verifyingContract` in EIP-712 domain
- Cache venue data per market (it's static and doesn't change)

#### Required Approvals

Before trading, set up token approvals:

- **BUY orders** ✨: Approve USDC to `venue.exchange`
- **SELL orders (simple CLOB)** ✨: Approve conditional tokens to `venue.exchange`
- **SELL orders (NegRisk/grouped)** ✨: Approve conditional tokens to **both** ✨ `venue.exchange` AND `venue.adapter`

#### Prerequisites

#### Required Python Packages

```bash
# 💻 Run in your terminal
// 🎯 Copy and customize this example
pip install eth-account requests web3
```

#### Complete Implementation

```python
// 🎯 Copy and customize this example
#!/usr/bin/env python3
"""
Limitless Exchange Trading Script
WARNING: This is a demo script. Use at your own risk.
Limitless Labs is not responsible for any losses or mistakes.
"""

import requests
import json
import time
import os
from eth_account import Account
from eth_account.messages import encode_defunct

def sign_message(self, message: str) -> str:
    """Sign a message using the private key."""
    if not self.account:
        raise Exception("Private key not provided. Cannot sign message.")

    print(f"🖊️  Signing message for account: {self.account.address}")

    message_hash = encode_defunct(text=message)
    signed_message = self.account.sign_message(message_hash)
    signature_hex = signed_message.signature.hex()

    print(
        f"   Generated signature: {signature_hex[:10]}... (length: {len(signature_hex)})"
    )

    return signature_hex

#### ============================================================================
#### DISCLAIMER AND WARNING
#### ============================================================================
print("⚠️  WARNING: This is an example trading script for educational purposes.")
print("⚠️  Limitless Labs is not responsible for any losses or mistakes.")
print("⚠️  Always test with small amounts first and understand the code.")
print("⚠️  USE AT YOUR OWN RISK.\n")

#### ============================================================================
#### Import Compatibility Layer
#### ============================================================================

try:
    from eth_account.messages import encode_typed_data

    print("✅ Successfully imported encode_typed_data from eth_account.messages")
except ImportError:
    try:
        from eth_account.messages import encode_structured_data as encode_typed_data

        print("✅ Using encode_structured_data (fallback)")
    except ImportError as e:
        print(f"❌ Import error: {e}")

#### ============================================================================
#### Configuration
#### ============================================================================

API_BASE_URL = os.getenv("API_URL", "https://api.limitless.exchange")

#### Market cache - stores venue data per market slug (venue is static per market)
MARKET_CACHE = {}

#### EIP-712 Type Definitions
ORDER_TYPES = {
    "Order": [\
        {"name": "salt", "type": "uint256"},\
        {"name": "maker", "type": "address"},\
        {"name": "signer", "type": "address"},\
        {"name": "taker", "type": "address"},\
        {"name": "tokenId", "type": "uint256"},\
        {"name": "makerAmount", "type": "uint256"},\
        {"name": "takerAmount", "type": "uint256"},\
        {"name": "expiration", "type": "uint256"},\
        {"name": "nonce", "type": "uint256"},\
        {"name": "feeRateBps", "type": "uint256"},\
        {"name": "side", "type": "uint8"},\
        {"name": "signatureType", "type": "uint8"},\
    ]
}

#### ============================================================================
#### Utility Functions
#### ============================================================================

def string_to_hex(text):
    """Convert string to hex representation with 0x prefix."""
    return "0x" + text.encode("utf-8").hex()

#### ============================================================================
#### Market Data Functions
#### ============================================================================

def get_market_data(slug):
    """
    Fetch market data including venue information.
    Uses cache to avoid repeated API calls (venue is static per market).

    Args:
        slug: The market slug

    Returns:
        dict: Market data including venue, tokens, etc.
    """
    if slug in MARKET_CACHE:
        print(f"📦 Using cached market data for: {slug}")
        return MARKET_CACHE[slug]

    print(f"📊 Fetching market data for: {slug}")
    response = requests.get(f"{API_BASE_URL}/markets/{slug}")
    if response.status_code == 200:
        market = response.json()
        MARKET_CACHE[slug] = market
        print(f"✅ Market data cached. Venue exchange: {market.get('venue', {}).get('exchange')}")
        return market
    else:
        raise Exception(f"Failed to get market data: {response.status_code} - {response.text}")

def get_venue_exchange(slug):
    """
    Get the venue exchange address for a market (used as verifyingContract).

    Args:
        slug: The market slug

    Returns:
        str: The venue exchange address (checksummed)
    """
    market = get_market_data(slug)
    venue = market.get("venue")
    if not venue or not venue.get("exchange"):
        raise Exception(f"Market {slug} does not have venue data")
    return venue["exchange"]

#### ============================================================================
#### Authentication Functions
#### ============================================================================

def get_signing_message():
    """
    Fetch the signing message from the API.

    Returns:
        str: The signing message to be signed
    """
    response = requests.get(f"{API_BASE_URL}/auth/signing-message")
    if response.status_code == 200:
        return response.text
    else:
        raise Exception(f"Failed to get signing message: {response.status_code}")

def authenticate(private_key, signing_message):
    """
    Authenticate with the Limitless Exchange API.

    Args:
        private_key: Your wallet's private key
        signing_message: The message to sign for authentication

    Returns:
        tuple: (session_cookie, user_data)

    Note:
        The address must be checksummed (EIP-55 format).
        eth_account.Account.from_key() returns checksummed addresses by default.
    """
    hex_message = "0x" + signing_message.encode("utf-8").hex()

    # Get the Ethereum address from the private key
    if not private_key.startswith("0x"):
        private_key = "0x" + private_key
    account = Account.from_key(private_key)
    # account.address is already checksummed (EIP-55 format)
    ethereum_address = account.address

    print(f"Using address (checksummed): {ethereum_address}")
    print(f"Signing message: {repr(signing_message)}")

    # Sign the message
    message = encode_defunct(text=signing_message)
    signature = account.sign_message(message)
    sig_hex = signature.signature.hex()
    if not sig_hex.startswith("0x"):
        sig_hex = "0x" + sig_hex

    # x-account must be checksummed address
    headers = {
        "x-account": ethereum_address,
        "x-signing-message": hex_message,
        "x-signature": sig_hex,
        "Content-Type": "application/json",
    }

    response = requests.post(
        f"{API_BASE_URL}/auth/login", headers=headers, json={"client": "eoa"}
    )

    if response.status_code == 200:
        session_cookie = response.cookies.get("limitless_session")
        return session_cookie, response.json()
    else:
        raise Exception(
            f"Authentication failed: {response.status_code} - {response.text}"
        )

#### ============================================================================
#### Order Creation Functions
#### ============================================================================

def create_order_payload_without_signature(
    maker_address, token_id, maker_amount, taker_amount, fee_rate_bps
):
    """
    Create the base order payload without signature.

    Args:
        maker_address: The maker's wallet address
        token_id: The token ID to trade (YES or NO token)
        maker_amount: Amount the maker is offering (scaled)
        taker_amount: Amount the maker wants in return (scaled)
        fee_rate_bps: Fee rate in basis points

    Returns:
        dict: Order payload ready for signing
    """
    salt = int(time.time() * 1000) + (24 * 60 * 60 * 1000)  # Current time + 24h in ms

    return {
        "salt": salt,
        "maker": maker_address,
        "signer": maker_address,
        "taker": "0x0000000000000000000000000000000000000000",  # Open to any taker
        "tokenId": str(token_id),  # Keep as string for API
        "makerAmount": maker_amount,
        "takerAmount": taker_amount,
        "expiration": "0",  # No expiration
        "nonce": 0,
        "feeRateBps": fee_rate_bps,
        "side": 0,  # 0 = BUY
        "signatureType": 0,  # 0 = EOA signature
    }

def get_eip712_domain(venue_exchange_address):
    """
    Get the EIP-712 domain for signing.

    Args:
        venue_exchange_address: The venue's exchange contract address from market data

    Returns:
        dict: EIP-712 domain object
    """
    return {
        "name": "Limitless CTF Exchange",
        "version": "1",
        "chainId": 8453,  # Base chain ID
        "verifyingContract": venue_exchange_address,
    }

def create_signature_for_order_payload(venue_exchange_address, order_payload, private_key):
    """
    Sign an order payload using EIP-712.

    Args:
        venue_exchange_address: The venue's exchange address (verifyingContract)
        order_payload: The order data to sign
        private_key: Private key for signing

    Returns:
        str: Hex-encoded signature
    """
    # Remove '0x' prefix if present
    if private_key.startswith("0x"):
        private_key = private_key[2:]

    # Create account
    account = Account.from_key("0x" + private_key)

    # Get domain data using venue's exchange address
    domain_data = get_eip712_domain(venue_exchange_address)

    # Convert string fields to int for signing
    message_data = {
        "salt": order_payload["salt"],
        "maker": order_payload["maker"],
        "signer": order_payload["signer"],
        "taker": order_payload["taker"],
        "tokenId": int(order_payload["tokenId"]),
        "makerAmount": order_payload["makerAmount"],
        "takerAmount": order_payload["takerAmount"],
        "expiration": int(order_payload["expiration"])
        if order_payload["expiration"]
        else 0,
        "nonce": order_payload["nonce"],
        "feeRateBps": order_payload["feeRateBps"],
        "side": order_payload["side"],
        "signatureType": order_payload["signatureType"],
    }

    print(f"Domain: {json.dumps(domain_data, indent=2)}")
    print(f"Types: {json.dumps(ORDER_TYPES, indent=2)}")
    print(f"Message: {json.dumps(message_data, indent=2)}")

    # Sign using eth_account's implementation of EIP-712
    encoded_message = encode_typed_data(domain_data, ORDER_TYPES, message_data)
    signed_message = account.sign_message(encoded_message)

    # Extract signature
    signature = signed_message.signature.hex()

    print(f"Successfully generated EIP-712 Signature: {signature}")
    return "0x" + signature

def create_order_api(order_payload, session_cookie):
    """
    Submit an order to the API.

    Args:
        order_payload: Complete order payload with signature
        session_cookie: Authentication session cookie

    Returns:
        dict: API response with order details
    """
    api_url = f"{API_BASE_URL}/orders"

    headers = {
        "cookie": f"limitless_session={session_cookie}",
        "Content-Type": "application/json",
        "Accept": "application/json",
    }

    print(f"Order payload: {json.dumps(order_payload, indent=2)}")

    try:
        response = requests.post(
            api_url, headers=headers, json=order_payload, timeout=35
        )

        if response.status_code != 201:
            print(f"Failed to create order. Status: {response.status_code}")
            print(f"Response: {response.text}")
            raise Exception(f"API Error {response.status_code}: {response.text}")

        result = response.json()
        print(f"Order created successfully: {json.dumps(result, indent=2)}")
        return result

    except Exception as error:
        print(f"Error creating order: {error}")
        raise error

#### ============================================================================
#### Main Trading Function
#### ============================================================================

def execute_trade(trading_params, market_slug, private_key):
    """
    Main function to execute a trade on Limitless Exchange.

    Args:
        trading_params: Dictionary containing:
            - sharePrice: price in cents (e.g., 65 for 65¢)
            - amount: amount in shares (e.g., 100)
            - firstType: "YES" or "NO"

        market_slug: The market slug (used to fetch market data including venue)

        private_key: Your wallet's private key

    Returns:
        dict: Order execution result
    """

    # Step 1: Authenticate
    signing_message = get_signing_message()
    session_cookie, user_data = authenticate(private_key, signing_message)
    print(f"Authenticated as: {user_data['account']}")

    # Step 2: Fetch market data (cached per market)
    # This gets venue info needed for EIP-712 signing
    market_data = get_market_data(market_slug)
    venue_exchange = market_data["venue"]["exchange"]
    print(f"Using venue exchange: {venue_exchange}")

    # Step 3: Calculate amounts
    price_in_cents = trading_params["sharePrice"]
    amount = trading_params["amount"]
    trade_type = "GTC"  # Good Till Cancelled
    scaling_factor = 1000000  # 1e6 for USDC (6 decimals)

    # Get token IDs from market data
    # positionIds[0] = YES token, positionIds[1] = NO token
    position_ids = market_data.get("positionIds", [])
    if len(position_ids) < 2:
        raise Exception("Market does not have valid position IDs")

    first_user_token = (
        position_ids[0]
        if trading_params["firstType"] == "YES"
        else position_ids[1]
    )

    # Calculate amounts
    price_in_dollars = price_in_cents / 100  # Convert cents to dollars
    total_cost = price_in_dollars * amount  # Total cost in dollars
    maker_amount = round(total_cost * scaling_factor)
    taker_amount = round(amount * scaling_factor)

    print(
        f"Trading: {amount} shares at {price_in_cents} cents ({price_in_dollars} dollars) each"
    )
    print(f"Total cost: {total_cost} dollars")
    print(f"Maker amount: {maker_amount}")
    print(f"Taker amount: {taker_amount}")

    # Step 4: Create order payload without signature
    # user_data["account"] is already checksummed
    order_payload_without_signature = create_order_payload_without_signature(
        user_data["account"],
        first_user_token,
        maker_amount,
        taker_amount,
        user_data.get("rank", {}).get("feeRateBps", 0),
    )

    # Step 5: Sign the order using venue's exchange address
    signature = create_signature_for_order_payload(
        venue_exchange, order_payload_without_signature, private_key
    )

    # Step 6: Create final order payload
    final_order_payload = {
        "order": {
            **order_payload_without_signature,
            "price": price_in_dollars,  # Send as decimal
            "signature": signature,
        },
        "ownerId": user_data["id"],
        "orderType": trade_type,
        "marketSlug": market_slug,
    }

    print(
        f"Order placed for ${price_in_cents} cents, amount: {amount}, share type: {trading_params['firstType']}"
    )

    # Step 7: Submit to API
    result = create_order_api(final_order_payload, session_cookie)
    return result

#### ============================================================================
#### Example Usage
#### ============================================================================

if __name__ == "__main__":
    print("=" * 80)
    print("LIMITLESS EXCHANGE TRADING SCRIPT - DEMO")
    print("=" * 80)
    print("\n⚠️  This is a demonstration script. Please modify for your needs.")
    print("⚠️  Never share your private key with anyone.")
    print("⚠️  Always test with small amounts first.\n")
    print("=" * 80)

    # Example parameters - REPLACE THESE WITH YOUR ACTUAL VALUES
    trading_params = {
        "sharePrice": 50,  # Price in cents (50¢)
        "amount": 2,  # Number of shares
        "firstType": "YES",  # 'YES' or 'NO'
    }

    # Market slug - the script will fetch market data (including venue and tokens) automatically
    market_slug = "example-market-slug"  # Replace with actual market slug

    # SECURITY WARNING: Never hardcode your private key!
    # Use environment variables or secure key management
    private_key = os.getenv("PRIVATE_KEY", "<FALLBACK_VALUE_FOR_P_KEY>")

    if private_key == "<FALLBACK_VALUE_FOR_P_KEY>":
        print(
            "❌ ERROR: Please set your private key in the environment variable PRIVATE_KEY"
        )
        print("Example: export PRIVATE_KEY='0x...'")
        exit(1)

    print("🚀 Starting trading script...")
    print(f"Trading parameters: {trading_params}")
    print(f"Market slug: {market_slug}")

    try:
        # Execute the trade
        # Market data (including venue) is fetched and cached automatically
        result = execute_trade(trading_params, market_slug, private_key)
        print("\n✅ Trade executed successfully!")
        print(f"Result: {json.dumps(result, indent=2)}")

    except Exception as e:
        print(f"\n❌ Error executing trade: {e}")
        import traceback

        traceback.print_exc()
```

#### Important Notes

#### ⚠️ Security Considerations

- **NEVER** ✨ share your private key with anyone
- **NEVER** ✨ commit your private key to version control
- Use environment variables to store sensitive information
- Always test with small amounts first
- Understand the code before using it with real funds

#### Order Types

- **GTC (Good Till Cancelled)** ✨: Order remains active until filled or cancelled
- **FOK (Fill or Kill)** ✨: Fill completely or cancel

#### Price Calculation

- Prices are in cents (e.g., 65 = 65¢ = 0.65 USD)
- YES price = probability of outcome occurring
- NO price = 1 - YES price

#### Amount Calculation

- USDC has 6 decimals (1 USDC = 1,000,000 units)
- Shares are scaled by 1e6 for precision
- Always verify calculations before submitting orders

#### Market Types

- **single-clob** ✨: Standard central limit order book markets
- **group-negrisk** ✨: Grouped markets with multiple related outcomes

#### Error Handling

Always implement proper error handling for:

- Network failures
- Authentication errors
- Insufficient balance
- Market closed/resolved
- Invalid order parameters

#### Environment Variables

Set these environment variables for production use:

```bash
# 💻 Run in your terminal
// 🎯 Copy and customize this example
#### Required
export PRIVATE_KEY="0x..."  # Your wallet private key

#### Optional (defaults shown)
export API_URL="https://api.limitless.exchange"
```

**Note** ✨: Contract addresses (exchange, adapter) are now fetched dynamically from the market's venue data via `GET /markets/:slug`. You no longer need to configure them as environment variables.

#### Troubleshooting

#### Common Issues

1. **ImportError for encode\_typed\_data** ✨
   - Solution: Upgrade eth-account: `pip install eth-account==0.10.0`
2. **Authentication Failed** ✨
   - Check that your private key is correct
   - Ensure you're using the correct format (with or without '0x' prefix)
3. **Order Creation Failed** ✨
   - Verify market slug and token IDs are correct
   - Check that you have sufficient balance
   - Ensure the market is still open
4. **Signature Verification Failed** ✨
   - Ensure you're using the venue's exchange address from market data
   - Verify the chain ID matches (8453 for Base)
   - Make sure addresses are checksummed (EIP-55 format)

#### Disclaimer

**This script is provided for educational purposes only. Limitless Labs is not responsible for any losses, mistakes, or unintended consequences from using this code. Trading involves risk, and you should never trade more than you can afford to lose. Always understand the code you're running and test with small amounts first.** ✨

#### Support

For API support and questions:

- Documentation: [https://limitlesslabs.notion.site/](https://limitlesslabs.notion.site/)
- Support: [hey@limitless.network](mailto:hey@limitless.network)

* * *

_Last updated: 2025-12-16_

## ☕ Java Quick Start

Complete end-to-end Java implementation for Limitless Exchange API integration.

#### Java E2E Order Creation Guide

⚠️ **IMPORTANT DISCLAIMER** ✨ ⚠️

```properties
This is an example implementation for educational purposes only.
Limitless Labs is not responsible for any losses or mistakes.
This code should be adjusted to your personal needs and risk tolerance.
Always test with small amounts first and understand the code before using it.
USE AT YOUR OWN RISK.
```

This guide provides a complete Java implementation for creating orders on Limitless Exchange, from authentication to order signing and submission.

#### Overview

The order creation process involves:

1. **Authentication** ✨: Get a signing message and authenticate with your wallet (use checksummed addresses)
2. **Fetch Market Data** ✨: Get venue information from `GET /markets/:slug` (cache per market)
3. **Order Construction** ✨: Build the order payload with proper parameters
4. **EIP-712 Signing** ✨: Sign the order using venue's exchange address as `verifyingContract`
5. **Order Submission** ✨: Submit the signed order to the API

#### Important: Venue System

CLOB markets use a **venue system** ✨ where each market has specific contract addresses. You must:

- Fetch market data via `GET /markets/:slug` to get `venue.exchange` and `venue.adapter`
- Use `venue.exchange` as the `verifyingContract` in EIP-712 domain
- Cache venue data per market (it's static and doesn't change)

#### Required Approvals

Before trading, set up token approvals:

- **BUY orders** ✨: Approve USDC to `venue.exchange`
- **SELL orders (simple CLOB)** ✨: Approve conditional tokens to `venue.exchange`
- **SELL orders (NegRisk/grouped)** ✨: Approve conditional tokens to **both** ✨ `venue.exchange` AND `venue.adapter`

#### Project Setup

#### Project Structure

Create the following directory structure:

```properties
limitless-trading-example/
├── build.gradle
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
└── src/
    └── main/
        └── java/
            └── com/
                └── limitless/
                    └── example/
                        └── LimitlessTradingExample.java
```

#### Initialize Gradle Project

```bash
# 💻 Run in your terminal
// 🎯 Copy and customize this example
mkdir limitless-trading-example
cd limitless-trading-example
gradle init --type java-application
```

#### Dependencies

#### build.gradle

```gradle
// 🎯 Copy and customize this example
plugins {
    id 'java'
    id 'application'
}

group = 'com.limitless.example'
version = '1.0.0'
sourceCompatibility = '11'

repositories {
    mavenCentral()
}

dependencies {
    // Web3j for Ethereum operations
    implementation 'org.web3j:core:4.9.8'
    implementation 'org.web3j:crypto:4.9.8'

    // OkHttp for HTTP operations
    implementation 'com.squareup.okhttp3:okhttp:4.11.0'
    implementation 'com.squareup.okhttp3:okhttp-urlconnection:4.11.0'

    // Jackson for JSON processing
    implementation 'com.fasterxml.jackson.core:jackson-databind:2.15.2'

    // Bouncy Castle for additional crypto support
    implementation 'org.bouncycastle:bcprov-jdk15on:1.70'

    // SLF4J for logging
    implementation 'org.slf4j:slf4j-api:2.0.7'
    implementation 'ch.qos.logback:logback-classic:1.4.8'

    // Testing
    testImplementation 'junit:junit:4.13.2'
}

application {
    mainClass = 'com.limitless.example.LimitlessTradingExample'
}

// Create a fat JAR with all dependencies
task fatJar(type: Jar) {
    manifest {
        attributes 'Main-Class': 'com.limitless.example.LimitlessTradingExample'
    }
    archiveClassifier = 'all'
    from {
        configurations.runtimeClasspath.collect { it.isDirectory() ? it : zipTree(it) }
    }
    with jar
}
```

#### Complete Implementation

#### LimitlessTradingExample.java

```java
// 🎯 Copy and customize this example
package com.limitless.example;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import com.fasterxml.jackson.databind.node.ArrayNode;

import okhttp3.CookieJar;
import okhttp3.JavaNetCookieJar;
import okhttp3.MediaType;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.RequestBody;
import okhttp3.Response;

import org.web3j.crypto.Credentials;
import org.web3j.crypto.Sign;
import org.web3j.crypto.StructuredDataEncoder;
import org.web3j.crypto.Keys;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.web3j.utils.Numeric;

import java.io.IOException;
import java.math.BigInteger;
import java.net.CookieManager;
import java.net.HttpCookie;
import java.nio.charset.StandardCharsets;
import java.util.*;
import java.util.concurrent.TimeUnit;

/**
 * Limitless Exchange Trading Example
 *
 * WARNING: This is a demo implementation. Use at your own risk.
 * Limitless Labs is not responsible for any losses or mistakes.
 *
 * This example demonstrates how to:
 * - Authenticate with the Limitless Exchange API
 * - Create and sign orders using EIP-712
 * - Submit orders to the exchange
 */
public class LimitlessTradingExample {

    private static final Logger logger = LoggerFactory.getLogger(LimitlessTradingExample.class);
    private static final ObjectMapper objectMapper = new ObjectMapper();

    // ============================================================================
    // Configuration
    // ============================================================================

    private static final String API_BASE_URL = System.getenv("API_URL") != null ?
        System.getenv("API_URL") : "https://api.limitless.exchange";

    private static final int CHAIN_ID = 8453; // Base chain ID

    // Market cache - stores market data per slug (venue is static per market)
    private static final Map<String, JsonNode> MARKET_CACHE = new HashMap<>();

    // HTTP client with cookie support
    private static final CookieManager cookieManager = new CookieManager();
    private static final OkHttpClient httpClient = new OkHttpClient.Builder()
        .cookieJar(new JavaNetCookieJar(cookieManager))
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .build();

    // ============================================================================
    // Data Classes
    // ============================================================================

    public static class TradingParams {
        public final int firstPrice;  // Price in cents
        public final int amount;      // Number of shares
        public final String firstType; // "YES" or "NO"

        public TradingParams(int firstPrice, int amount, String firstType) {
            this.firstPrice = firstPrice;
            this.amount = amount;
            this.firstType = firstType;
        }
    }

    public static class AuthResult {
        public final String sessionCookie;
        public final JsonNode userData;

        public AuthResult(String sessionCookie, JsonNode userData) {
            this.sessionCookie = sessionCookie;
            this.userData = userData;
        }
    }

    // ============================================================================
    // Utility Functions
    // ============================================================================

    private static String stringToHex(String text) {
        return "0x" + Numeric.toHexStringNoPrefix(text.getBytes(StandardCharsets.UTF_8));
    }

    // ============================================================================
    // Market Data Functions
    // ============================================================================

    /**
     * Fetch market data including venue information.
     * Uses cache to avoid repeated API calls (venue is static per market).
     */
    private static JsonNode getMarketData(String slug) throws IOException {
        if (MARKET_CACHE.containsKey(slug)) {
            logger.info("Using cached market data for: {}", slug);
            return MARKET_CACHE.get(slug);
        }

        logger.info("Fetching market data for: {}", slug);

        Request request = new Request.Builder()
            .url(API_BASE_URL + "/markets/" + slug)
            .get()
            .build();

        try (Response response = httpClient.newCall(request).execute()) {
            if (!response.isSuccessful()) {
                throw new IOException("Failed to get market data: " + response.code() + " - " + response.body().string());
            }

            JsonNode market = objectMapper.readTree(response.body().string());
            MARKET_CACHE.put(slug, market);

            String venueExchange = market.path("venue").path("exchange").asText(null);
            logger.info("Market data cached. Venue exchange: {}", venueExchange);

            return market;
        }
    }

    /**
     * Get the venue exchange address for a market (used as verifyingContract).
     */
    private static String getVenueExchange(String slug) throws IOException {
        JsonNode market = getMarketData(slug);
        JsonNode venue = market.get("venue");
        if (venue == null || venue.get("exchange") == null) {
            throw new IOException("Market " + slug + " does not have venue data");
        }
        return venue.get("exchange").asText();
    }

    // ============================================================================
    // Authentication
    // ============================================================================

    private static String getSigningMessage() throws IOException {
        logger.info("Fetching signing message from API...");

        Request request = new Request.Builder()
            .url(API_BASE_URL + "/auth/signing-message")
            .get()
            .build();

        try (Response response = httpClient.newCall(request).execute()) {
            if (!response.isSuccessful()) {
                throw new IOException("Failed to get signing message: " + response.code());
            }

            String signingMessage = response.body().string();
            logger.info("Signing message received: {}", signingMessage);
            return signingMessage;
        }
    }

    private static AuthResult authenticate(String privateKey, String signingMessage) throws Exception {
        logger.info("Authenticating with private key...");

        // Create credentials from private key
        Credentials credentials = Credentials.create(privateKey);
        String address = credentials.getAddress();
        logger.info("Using address: {}", address);

        // Sign the message
        byte[] messageBytes = signingMessage.getBytes(StandardCharsets.UTF_8);
        Sign.SignatureData signature = Sign.signPrefixedMessage(messageBytes, credentials.getEcKeyPair());

        // Convert signature to hex
        String signatureHex = Numeric.toHexString(signature.getR()) +
                              Numeric.toHexStringNoPrefix(signature.getS()) +
                              Numeric.toHexStringNoPrefix(signature.getV());

        // Prepare request
        String hexMessage = stringToHex(signingMessage);

        ObjectNode requestBody = objectMapper.createObjectNode();
        requestBody.put("client", "eoa");

        RequestBody body = RequestBody.create(
            objectMapper.writeValueAsString(requestBody),
            MediaType.parse("application/json")
        );

        String checksumAddress = Keys.toChecksumAddress(credentials.getAddress());

        Request request = new Request.Builder()
            .url(API_BASE_URL + "/auth/login")
            .addHeader("x-account", checksumAddress)
            .addHeader("x-signing-message", hexMessage)
            .addHeader("x-signature", signatureHex)
            .addHeader("Content-Type", "application/json")
            .addHeader("Accept", "application/json")
            .post(body)
            .build();

        try (Response response = httpClient.newCall(request).execute()) {
            if (!response.isSuccessful()) {
                throw new Exception("Authentication failed: " + response.code() + " - " + response.body().string());
            }

            // Extract session cookie
            String sessionCookie = null;
            for (HttpCookie cookie : cookieManager.getCookieStore().getCookies()) {
                if ("limitless_session".equals(cookie.getName())) {
                    sessionCookie = cookie.getValue();
                    break;
                }
            }

            if (sessionCookie == null) {
                throw new Exception("Session cookie not found in response");
            }

            JsonNode userData = objectMapper.readTree(response.body().string());
            logger.info("Authenticated as: {}", userData.get("account").asText());

            return new AuthResult(sessionCookie, userData);
        }
    }

    // ============================================================================
    // Order Creation
    // ============================================================================

    private static Map<String, Object> createOrderPayloadWithoutSignature(
            String makerAddress,
            String tokenId,
            long makerAmount,
            long takerAmount,
            int feeRateBps) {

        long salt = System.currentTimeMillis() + (24L * 60 * 60 * 1000); // Current time + 24h

        Map<String, Object> payload = new LinkedHashMap<>();
        payload.put("salt", salt);
        payload.put("maker", Keys.toChecksumAddress(makerAddress));
        payload.put("signer", Keys.toChecksumAddress(makerAddress));
        payload.put("taker", "0x0000000000000000000000000000000000000000");
        payload.put("tokenId", tokenId); // Keep as string for API
        payload.put("makerAmount", makerAmount);
        payload.put("takerAmount", takerAmount);
        payload.put("expiration", "0");
        payload.put("nonce", 0);
        payload.put("feeRateBps", feeRateBps);
        payload.put("side", 0); // 0 = BUY
        payload.put("signatureType", 0);

        return payload;
    }

    // ============================================================================
    // EIP-712 Signing
    // ============================================================================

    /**
     * Get the EIP-712 domain for signing.
     * @param venueExchangeAddress The venue's exchange contract address from market data
     */
    private static Map<String, Object> getEip712Domain(String venueExchangeAddress) {
        Map<String, Object> domain = new LinkedHashMap<>();
        domain.put("name", "Limitless CTF Exchange");
        domain.put("version", "1");
        domain.put("chainId", CHAIN_ID);
        domain.put("verifyingContract", venueExchangeAddress);

        return domain;
    }

    /**
     * Sign an order payload using EIP-712.
     * @param venueExchangeAddress The venue's exchange address (verifyingContract)
     * @param orderPayload The order data to sign
     * @param privateKey Private key for signing
     */
    private static String createSignatureForOrderPayload(
            String venueExchangeAddress,
            Map<String, Object> orderPayload,
            String privateKey) throws Exception {

        logger.info("Creating EIP-712 signature...");

        Credentials credentials = Credentials.create(privateKey);
        Map<String, Object> domain = getEip712Domain(venueExchangeAddress);

        // Convert for signing (tokenId and expiration to BigInteger)
        Map<String, Object> signingPayload = new LinkedHashMap<>(orderPayload);
        signingPayload.put("tokenId", new BigInteger(orderPayload.get("tokenId").toString()));
        signingPayload.put("expiration", new BigInteger(orderPayload.get("expiration").toString()));

        // Create EIP-712 typed data structure
        ObjectNode typedData = objectMapper.createObjectNode();

        // Add domain
        ObjectNode domainNode = objectMapper.createObjectNode();
        domainNode.put("name", domain.get("name").toString());
        domainNode.put("version", domain.get("version").toString());
        domainNode.put("chainId", (Integer) domain.get("chainId"));
        domainNode.put("verifyingContract", domain.get("verifyingContract").toString());
        typedData.set("domain", domainNode);

        // Add types
        ObjectNode types = objectMapper.createObjectNode();

        // EIP712Domain
        ArrayNode domainTypeArray = objectMapper.createArrayNode();
        domainTypeArray.add(objectMapper.createObjectNode().put("name", "name").put("type", "string"));
        domainTypeArray.add(objectMapper.createObjectNode().put("name", "version").put("type", "string"));
        domainTypeArray.add(objectMapper.createObjectNode().put("name", "chainId").put("type", "uint256"));
        domainTypeArray.add(objectMapper.createObjectNode().put("name", "verifyingContract").put("type", "address"));
        types.set("EIP712Domain", domainTypeArray);

        // Order
        ArrayNode orderTypeArray = objectMapper.createArrayNode();
        orderTypeArray.add(objectMapper.createObjectNode().put("name", "salt").put("type", "uint256"));
        orderTypeArray.add(objectMapper.createObjectNode().put("name", "maker").put("type", "address"));
        orderTypeArray.add(objectMapper.createObjectNode().put("name", "signer").put("type", "address"));
        orderTypeArray.add(objectMapper.createObjectNode().put("name", "taker").put("type", "address"));
        orderTypeArray.add(objectMapper.createObjectNode().put("name", "tokenId").put("type", "uint256"));
        orderTypeArray.add(objectMapper.createObjectNode().put("name", "makerAmount").put("type", "uint256"));
        orderTypeArray.add(objectMapper.createObjectNode().put("name", "takerAmount").put("type", "uint256"));
        orderTypeArray.add(objectMapper.createObjectNode().put("name", "expiration").put("type", "uint256"));
        orderTypeArray.add(objectMapper.createObjectNode().put("name", "nonce").put("type", "uint256"));
        orderTypeArray.add(objectMapper.createObjectNode().put("name", "feeRateBps").put("type", "uint256"));
        orderTypeArray.add(objectMapper.createObjectNode().put("name", "side").put("type", "uint8"));
        orderTypeArray.add(objectMapper.createObjectNode().put("name", "signatureType").put("type", "uint8"));
        types.set("Order", orderTypeArray);

        typedData.set("types", types);
        typedData.put("primaryType", "Order");

        // Add message
        ObjectNode message = objectMapper.valueToTree(signingPayload);
        typedData.set("message", message);
        typedData.put("primaryType", "Order");

        // Sign using Web3j's structured data signing
        StructuredDataEncoder encoder = new StructuredDataEncoder(typedData.toString());
        byte[] structuredData = encoder.getStructuredData();

        Sign.SignatureData signature = Sign.signMessage(structuredData, credentials.getEcKeyPair());

        String signatureHex = Numeric.toHexString(signature.getR()) +
                             Numeric.toHexStringNoPrefix(signature.getS()) +
                             Numeric.toHexStringNoPrefix(signature.getV());

        logger.info("Successfully generated EIP-712 signature: {}", signatureHex);
        return signatureHex;
    }

    // ============================================================================
    // Order Submission
    // ============================================================================

    private static JsonNode createOrderApi(Map<String, Object> orderPayload, String sessionCookie) throws Exception {
        logger.info("Submitting order to API...");

        String jsonPayload = objectMapper.writeValueAsString(orderPayload);
        logger.debug("Order payload: {}", jsonPayload);

        RequestBody body = RequestBody.create(
            jsonPayload,
            MediaType.parse("application/json")
        );

        Request request = new Request.Builder()
            .url(API_BASE_URL + "/orders")
            .addHeader("Cookie", "limitless_session=" + sessionCookie)
            .addHeader("Content-Type", "application/json")
            .addHeader("Accept", "application/json")
            .post(body)
            .build();

        try (Response response = httpClient.newCall(request).execute()) {
            String responseBody = response.body().string();

            if (response.code() != 201) {
                logger.error("Failed to create order. Status: {}", response.code());
                logger.error("Response: {}", responseBody);
                throw new Exception("API Error " + response.code() + ": " + responseBody);
            }

            JsonNode result = objectMapper.readTree(responseBody);
            logger.info("Order created successfully: {}", result);
            return result;
        }
    }

    // ============================================================================
    // Main Trading Function
    // ============================================================================

    /**
     * Main function to execute a trade on Limitless Exchange.
     * @param tradingParams Trading parameters (price, amount, type)
     * @param marketSlug The market slug (used to fetch market data including venue)
     * @param privateKey Your wallet's private key
     */
    public static JsonNode executeTrade(TradingParams tradingParams, String marketSlug, String privateKey)
            throws Exception {

        logger.info("=" + "=".repeat(79));
        logger.info("Starting trade execution...");
        logger.info("=" + "=".repeat(79));

        // Step 1: Authenticate
        String signingMessage = getSigningMessage();
        AuthResult authResult = authenticate(privateKey, signingMessage);

        // Step 2: Fetch market data (cached per market)
        // This gets venue info needed for EIP-712 signing
        JsonNode marketData = getMarketData(marketSlug);
        String venueExchange = marketData.path("venue").path("exchange").asText();
        logger.info("Using venue exchange: {}", venueExchange);

        // Step 3: Calculate amounts
        int priceInCents = tradingParams.firstPrice;
        int amount = tradingParams.amount;
        String tradeType = "GTC"; // Good Till Cancelled
        long scalingFactor = 1000000L; // 1e6 for USDC (6 decimals)

        // Get token IDs from market data
        // positionIds[0] = YES token, positionIds[1] = NO token
        JsonNode positionIds = marketData.get("positionIds");
        if (positionIds == null || positionIds.size() < 2) {
            throw new Exception("Market does not have valid position IDs");
        }

        String selectedToken = "YES".equals(tradingParams.firstType) ?
            positionIds.get(0).asText() : positionIds.get(1).asText();

        // Calculate amounts
        double priceInDollars = priceInCents / 100.0;
        double totalCost = priceInDollars * amount;
        long makerAmount = Math.round(totalCost * scalingFactor);
        long takerAmount = Math.round(amount * scalingFactor);

        logger.info("Trading: {} shares at {} cents ({} dollars) each",
            amount, priceInCents, priceInDollars);
        logger.info("Total cost: {} dollars", totalCost);
        logger.info("Maker amount: {}", makerAmount);
        logger.info("Taker amount: {}", takerAmount);

        // Step 4: Create order payload without signature
        // authResult.userData.get("account") is already checksummed
        String makerAddress = authResult.userData.get("account").asText();
        int feeRateBps = authResult.userData.path("rank").path("feeRateBps").asInt(0);

        Map<String, Object> orderPayload = createOrderPayloadWithoutSignature(
            makerAddress,
            selectedToken,
            makerAmount,
            takerAmount,
            feeRateBps
        );

        // Step 5: Sign the order using venue's exchange address
        String signature = createSignatureForOrderPayload(venueExchange, orderPayload, privateKey);

        // Step 6: Create final order payload
        Map<String, Object> finalOrderPayload = new LinkedHashMap<>();
        Map<String, Object> orderWithSignature = new LinkedHashMap<>(orderPayload);
        orderWithSignature.put("price", priceInDollars);
        orderWithSignature.put("signature", signature);

        finalOrderPayload.put("order", orderWithSignature);
        finalOrderPayload.put("ownerId", authResult.userData.get("id").asInt());
        finalOrderPayload.put("orderType", tradeType);
        finalOrderPayload.put("marketSlug", marketSlug);

        logger.info("Order placed for {} cents, amount: {}, share type: {}",
            priceInCents, amount, tradingParams.firstType);

        // Step 7: Submit to API
        return createOrderApi(finalOrderPayload, authResult.sessionCookie);
    }

    // ============================================================================
    // Main Method - Example Usage
    // ============================================================================

    public static void main(String[] args) {
        System.out.println("=" + "=".repeat(79));
        System.out.println("LIMITLESS EXCHANGE TRADING EXAMPLE - JAVA");
        System.out.println("=" + "=".repeat(79));
        System.out.println("\n⚠️  WARNING: This is an example trading application for educational purposes.");
        System.out.println("⚠️  Limitless Labs is not responsible for any losses or mistakes.");
        System.out.println("⚠️  Always test with small amounts first and understand the code.");
        System.out.println("⚠️  USE AT YOUR OWN RISK.\n");
        System.out.println("=" + "=".repeat(79));

        try {
            // Example parameters - REPLACE THESE WITH YOUR ACTUAL VALUES
            TradingParams tradingParams = new TradingParams(
                50,    // Price in cents (50¢)
                2,     // Number of shares
                "YES"  // "YES" or "NO"
            );

            // Market slug - the script will fetch market data (including venue and tokens) automatically
            String marketSlug = "example-market-slug";  // Replace with actual market slug

            // SECURITY WARNING: Never hardcode your private key!
            // Use environment variables or secure key management
            String privateKey = System.getenv("PRIVATE_KEY");

            if (privateKey == null || privateKey.isEmpty()) {
                System.err.println("❌ ERROR: Please set your private key in the PRIVATE_KEY environment variable");
                System.err.println("Example: export PRIVATE_KEY='0x...'");
                System.exit(1);
            }

            System.out.println("🚀 Starting trading example...");
            System.out.println("Trading parameters: " +
                tradingParams.amount + " shares at " +
                tradingParams.firstPrice + " cents for " +
                tradingParams.firstType);
            System.out.println("Market slug: " + marketSlug);

            // Execute the trade
            // Market data (including venue) is fetched and cached automatically
            JsonNode result = executeTrade(tradingParams, marketSlug, privateKey);

            System.out.println("\n✅ Trade executed successfully!");
            System.out.println("Result: " + objectMapper.writerWithDefaultPrettyPrinter()
                .writeValueAsString(result));

        } catch (Exception e) {
            System.err.println("\n❌ Error executing trade: " + e.getMessage());
            e.printStackTrace();
        }
    }
}
```

#### Important Notes

#### ⚠️ Security Considerations

- **NEVER** ✨ share your private key with anyone
- **NEVER** ✨ commit your private key to version control
- Use environment variables to store sensitive information
- Always test with small amounts first
- Understand the code before using it with real funds

#### Order Types

- **GTC (Good Till Cancelled)** ✨: Order remains active until filled or cancelled
- **FOK (Fill or Kill)** ✨: Fill completely or cancel

#### Price Calculation

- Prices are in cents (e.g., 65 = 65¢ = 0.65 USD)
- YES price = probability of outcome occurring
- NO price = 1 - YES price

#### Amount Calculation

- USDC has 6 decimals (1 USDC = 1,000,000 units)
- Shares are scaled by 1e6 for precision
- Always verify calculations before submitting orders

#### Market Types

- **single-clob** ✨: Standard central limit order book markets
- **group-negrisk** ✨: Grouped markets with multiple related outcomes

#### Build and Run

#### Install Dependencies

```bash
# 💻 Run in your terminal
// 🎯 Copy and customize this example
./gradlew build
```

#### Run the Application

```bash
# 💻 Run in your terminal
// 🎯 Copy and customize this example
#### With environment variable
export PRIVATE_KEY="0x..."
./gradlew run

#### Or with system property
./gradlew run -DPRIVATE_KEY="0x..."
```

#### Create Executable JAR

```bash
# 💻 Run in your terminal
// 🎯 Copy and customize this example
./gradlew fatJar
java -jar build/libs/limitless-trading-example-1.0.0-all.jar
```

#### Environment Variables

Set these environment variables for production use:

```bash
# 💻 Run in your terminal
// 🎯 Copy and customize this example
#### Required
export PRIVATE_KEY="0x..."  # Your wallet private key

#### Optional (defaults shown)
export API_URL="https://api.limitless.exchange"
```

**Note** ✨: Contract addresses (exchange, adapter) are now fetched dynamically from the market's venue data via `GET /markets/:slug`. You no longer need to configure them as environment variables.

#### Troubleshooting

#### Common Issues

1. **Build Failures** ✨
   - Ensure Java 11+ is installed
   - Check internet connection for dependency downloads
   - Clear Gradle cache: `./gradlew clean`
2. **Authentication Failed** ✨
   - Verify private key format (with or without '0x' prefix)
   - Check API endpoint accessibility
   - Ensure signing message is current
3. **Order Creation Failed** ✨
   - Verify market slug and token IDs
   - Check account balance
   - Ensure market is still open
4. **Signature Verification Failed** ✨
   - Ensure you're using the venue's exchange address from market data
   - Check chain ID (8453 for Base)
   - Make sure addresses are checksummed (use `Keys.toChecksumAddress()`)
   - Ensure Web3j version compatibility

#### Key Features

✅ **Authentication** ✨: Message signing with Ethereum private key

✅ **EIP-712 Signing** ✨: Complete EIP-712 typed data signing for orders

✅ **Order Creation** ✨: Same payload structure as Python/TypeScript

✅ **API Integration** ✨: HTTP requests with proper headers and cookies

✅ **Error Handling** ✨: Comprehensive exception handling

✅ **Logging** ✨: SLF4J with Logback for proper logging

#### Disclaimer

**This code is provided for educational purposes only. Limitless Labs is not responsible for any losses, mistakes, or unintended consequences from using this code. Trading involves risk, and you should never trade more than you can afford to lose. Always understand the code you're running and test with small amounts first.** ✨

#### Support

For API support and questions:

- Documentation: [https://limitlesslabs.notion.site/](https://limitlesslabs.notion.site/)
- Support: [hey@limitless.network](mailto:hey@limitless.network)

* * *

_Last updated: 2025-12-16_

## 📦 Node.js/TypeScript Quick Start

Complete end-to-end Node.js/TypeScript implementation for trading and WebSocket subscriptions.

#### Node.js (TypeScript) – Place Order + Subscribe to Updates

```properties
This is an example for educational purposes only.
Limitless Labs is not responsible for any losses or mistakes.
Use at your own risk.
```

#### Overview

- Place an order via REST (EIP-712 signing required)
- Subscribe via WebSocket to:
  - AMM price updates for specific market addresses
  - CLOB orderbook updates for specific market slugs

Important: subscriptions replace previous ones. If you want AMM prices and CLOB orderbook at the same time, send both `marketAddresses` and `marketSlugs` together in a single `subscribe_market_prices` call.

Production WS URL: `wss://ws.limitless.exchange` namespace `/markets`
Production API URL: `https://api.limitless.exchange`

#### Important: Venue System for CLOB Markets

CLOB markets use a **venue system** ✨ where each market has specific contract addresses. Before placing orders:

1. **Fetch market data once** ✨: `GET /markets/:slug` returns venue information
2. **Use venue.exchange** ✨: This is the `verifyingContract` for EIP-712 order signing
3. **Cache the venue** ✨: Venue data is static per market - fetch once and reuse

**Required Approvals:** ✨

- **BUY orders** ✨: Approve USDC to `venue.exchange`
- **SELL orders (simple CLOB)** ✨: Approve conditional tokens to `venue.exchange`
- **SELL orders (NegRisk/grouped)** ✨: Approve conditional tokens to **both** ✨ `venue.exchange` AND `venue.adapter`

**Checksummed Addresses:** ✨
All addresses must use checksummed format (EIP-55). Use `getAddress()` from viem to ensure proper formatting.

#### Prerequisites

- Node 18+ (or any V8/Node LTS)
- Packages:

```bash
# 💻 Run in your terminal
// 🎯 Copy and customize this example
pnpm add socket.io-client cross-fetch ethers viem
```

- Env variables (example):

```bash
# 💻 Run in your terminal
// 🎯 Copy and customize this example
export PRIVATE_KEY="0x..."     # Your wallet private key
export API_URL="https://api.limitless.exchange"
```

**⚠️ Security Warning:** ✨

Storing private keys in `.env` files is **dangerous** ✨ and should only be used for development/testing purposes. For production environments, use proper secret management services.

* * *

#### TypeScript Helpers (minimal)

```ts
// 🎯 Copy and customize this example
// market.ts - Market data fetching with caching
import fetch from 'cross-fetch';

type MarketVenue = {
  exchange: `0x${string}`;
  adapter?: `0x${string}`;
};

type MarketData = {
  id: number;
  slug: string;
  positionIds: string[];
  venue: MarketVenue;
};

// Cache market data per slug (venue is static per market)
const marketCache = new Map<string, MarketData>();

export async function getMarketData(apiBase: string, slug: string): Promise<MarketData> {
  const cached = marketCache.get(slug);
  if (cached) {
    console.log(`Using cached market data for: ${slug}`);
    return cached;
  }

  console.log(`Fetching market data for: ${slug}`);
  const res = await fetch(`${apiBase}/markets/${slug}`);
  if (!res.ok) throw new Error(`Failed to get market: ${res.status} ${await res.text()}`);

  const market = (await res.json()) as MarketData;
  marketCache.set(slug, market);
  console.log(`Market data cached. Venue exchange: ${market.venue?.exchange}`);

  return market;
}

export function getVenueExchange(market: MarketData): `0x${string}` {
  if (!market.venue?.exchange) {
    throw new Error(`Market ${market.slug} does not have venue data`);
  }
  return market.venue.exchange;
}
```

```ts
// 🎯 Copy and customize this example
// auth.ts
import fetch from 'cross-fetch';
import { Wallet } from 'ethers';

export async function getSigningMessage(apiBase: string): Promise<string> {
  const res = await fetch(`${apiBase}/auth/signing-message`);
  if (!res.ok) throw new Error(`Failed signing-message: ${res.status}`);
  return res.text();
}

export async function loginWithPrivateKey(apiBase: string, privateKey: string) {
  const wallet = new Wallet(privateKey);
  // wallet.address is already checksummed
  const address = await wallet.getAddress();

  const message = await getSigningMessage(apiBase);
  const signature = await wallet.signMessage(message);

  // x-account must be checksummed address
  const headers = {
    'x-account': address,
    'x-signing-message': `0x${Buffer.from(message, 'utf8').toString('hex')}`,
    'x-signature': signature.startsWith('0x') ? signature : `0x${signature}`,
    'content-type': 'application/json',
    accept: 'application/json',
  } as const;

  const res = await fetch(`${apiBase}/auth/login`, {
    method: 'POST',
    headers,
    body: JSON.stringify({ client: 'eoa' }),
  });
  if (!res.ok) throw new Error(`Auth failed: ${res.status} ${await res.text()}`);

  const setCookie = res.headers.get('set-cookie') || '';
  const match = /limitless_session=([^;]+)/i.exec(setCookie);
  const session = match?.[1];
  if (!session) throw new Error('Session cookie not found');

  const user = await res.json();
  return { session, user } as const;
}
```

```ts
// 🎯 Copy and customize this example
// orders.ts (viem-based FOK market order example)
import fetch from 'cross-fetch';
import { parseUnits, type WalletClient, type Account, type Transport, type Chain } from 'viem';

type OrderMessage = {
  salt: string;
  maker: `0x${string}`;
  signer: `0x${string}`;
  taker: `0x${string}`;
  tokenId: string;
  makerAmount: string;
  takerAmount: string;
  expiration: string;
  nonce: string;
  feeRateBps: string;
  side: number; // 0 buy, 1 sell
  signatureType: number; // 0 EOA
};

const EIP712_DOMAIN = [\
  { name: 'name', type: 'string' },\
  { name: 'version', type: 'string' },\
  { name: 'chainId', type: 'uint256' },\
  { name: 'verifyingContract', type: 'address' },\
] as const;

const ORDER_STRUCTURE = [\
  { name: 'salt', type: 'uint256' },\
  { name: 'maker', type: 'address' },\
  { name: 'signer', type: 'address' },\
  { name: 'taker', type: 'address' },\
  { name: 'tokenId', type: 'uint256' },\
  { name: 'makerAmount', type: 'uint256' },\
  { name: 'takerAmount', type: 'uint256' },\
  { name: 'expiration', type: 'uint256' },\
  { name: 'nonce', type: 'uint256' },\
  { name: 'feeRateBps', type: 'uint256' },\
  { name: 'side', type: 'uint8' },\
  { name: 'signatureType', type: 'uint8' },\
] as const;

/**
 * Create a signed order for CLOB trading.
 * @param client - viem WalletClient with account
 * @param options.verifyingContract - The venue's exchange address from market data (GET /markets/:slug)
 */
export async function createSignedOrder(
  client: WalletClient<Transport, Chain, Account>,
  options: {
    amount: string; // USD amount in decimal, e.g., '1.5'
    side: 0 | 1; // 0 buy, 1 sell
    tokenId: string;
    decimals?: number; // default 6
    chainId: number; // e.g., 8453
    verifyingContract: `0x${string}`; // venue.exchange from market data
  },
) {
  const maker = client.account!.address as `0x${string}`;
  const salt = String(Math.round(Math.random() * Date.now()));
  const makerAmount = parseUnits(options.amount, options.decimals ?? 6).toString();

  const order: OrderMessage = {
    salt,
    maker,
    signer: maker,
    taker: '0x0000000000000000000000000000000000000000',
    tokenId: options.tokenId,
    makerAmount,
    takerAmount: '1', // FOK market order semantics
    expiration: '0',
    nonce: '0',
    feeRateBps: '300',
    side: options.side,
    signatureType: 0,
  };

  const typedData = {
    primaryType: 'Order',
    types: { EIP712Domain: EIP712_DOMAIN, Order: ORDER_STRUCTURE },
    domain: {
      name: 'Limitless CTF Exchange',
      version: '1',
      chainId: options.chainId,
      verifyingContract: options.verifyingContract,
    },
    message: order,
  } as const;

  const signature = await client.signTypedData(typedData);
  return { order, signature } as const;
}

export async function submitOrder(
  apiBase: string,
  payload: {
    order: OrderMessage & { price?: number; signature: string };
    ownerId: number;
    orderType: 'FOK' | 'GTC';
    marketSlug: string;
  },
  extraHeaders?: Record<string, string>,
) {
  const res = await fetch(`${apiBase}/orders`, {
    method: 'POST',
    headers: {
      accept: 'application/json',
      'content-type': 'application/json',
      ...(extraHeaders || {}),
    },
    body: JSON.stringify(payload),
  });
  if (!res.ok) throw new Error(`Order failed: ${res.status} ${await res.text()}`);
  return res.json();
}
```

```ts
// 🎯 Copy and customize this example
// ws.ts
import { io, Socket } from 'socket.io-client';

export function connectMarkets(session?: string) {
  const url = 'wss://ws.limitless.exchange/markets';
  const transports = ['websocket'];

  const opts: Parameters<typeof io>[1] = { transports };
  if (session) {
    // Send cookie via extraHeaders; supported on Node
    opts.extraHeaders = { Cookie: `limitless_session=${session}` };
  }

  const socket: Socket = io(url, opts);
  return socket;
}

// If need to only subscribe to AMM prices, use this function
export function subscribeAmmPrices(socket: Socket, marketAddresses: string[]) {
  socket.emit('subscribe_market_prices', { marketAddresses });

  // Server emits 'newPriceData' to the same room
  socket.on('newPriceData', (data: unknown) => {
    // Shape may include: { marketAddress?, updatedPrices, blockNumber, timestamp }
    console.log('AMM price update:', data);
  });
}

// If need to only subscribe to CLOB orderbook, use this function
export function subscribeClobOrderbook(socket: Socket, marketSlugs: string[]) {
  socket.emit('subscribe_market_prices', { marketSlugs });

  // Server emits 'orderbookUpdate' per slug room
  socket.on('orderbookUpdate', (data: unknown) => {
    // Shape: { marketSlug, orderbook, timestamp }
    console.log('Orderbook update:', data);
  });
}
```

```ts
// 🎯 Copy and customize this example
// wallet.ts
import { createWalletClient, http } from 'viem';
import { privateKeyToAccount } from 'viem/accounts';
import { base } from 'viem/chains';

export function createWalletFromPrivateKey(privateKey: `0x${string}`) {
  const account = privateKeyToAccount(privateKey);

  const client = createWalletClient({
    account,
    chain: base, // Base mainnet (chain ID 8453)
    transport: http(),
  });

  return client;
}
```

* * *

#### End-to-end example

```ts
// 🎯 Copy and customize this example
// main.ts
import { loginWithPrivateKey } from './auth';
import { getMarketData, getVenueExchange } from './market';
import { createSignedOrder, submitOrder } from './orders';
import { connectMarkets } from './ws';
import { createWalletFromPrivateKey } from './wallet';

async function main() {
  const API_URL = process.env.API_URL || 'https://api.limitless.exchange';

  // Warning: storing private keys in .env files is dangerous, use proper secret management services.
  const PRIVATE_KEY = process.env.PRIVATE_KEY || '';

  // Market slug for CLOB trading
  const marketSlug = 'example-market-slug'; // Replace with actual market slug

  // 1) Authenticate (optional for public WS; required to place orders)
  const { session, user } = await loginWithPrivateKey(API_URL, PRIVATE_KEY);
  console.log('Authenticated as', user.account);

  // 2) Fetch market data (cached per market) to get venue and token info
  const market = await getMarketData(API_URL, marketSlug);
  const verifyingContract = getVenueExchange(market);
  console.log('Using venue exchange:', verifyingContract);

  // Get token IDs from market data
  // positionIds[0] = YES token, positionIds[1] = NO token
  const tokenId = market.positionIds[0]; // YES token for this example

  // 3) Connect WS and subscribe
  const socket = connectMarkets(session);

  // Combined subscription (send both arrays together to avoid replacing previous subscriptions)
  const marketAddresses = ['0xE082AF5a25f5D3904fae514CD03dC99F9Ff39fBc']; // AMM markets array
  const marketSlugs = [marketSlug]; // CLOB markets array
  socket.emit('subscribe_market_prices', { marketAddresses, marketSlugs });

  // Handlers
  socket.on('newPriceData', (data: unknown) => console.log('AMM price update:', data)); // AMM markets only
  socket.on('orderbookUpdate', (data: unknown) => console.log('Orderbook update:', data)); // CLOB markets only

  // 4) Create wallet client from private key
  const client = createWalletFromPrivateKey(PRIVATE_KEY as `0x${string}`);

  // 5) Prepare and place CLOB order using venue's exchange address
  const { order, signature } = await createSignedOrder(client, {
    amount: '1.5',
    side: 0, // 0 = BUY
    tokenId,
    decimals: 6,
    chainId: 8453,
    verifyingContract, // From venue data
  });

  const orderPayload = {
    order: { ...order, signature },
    ownerId: user.id,
    orderType: 'FOK' as const,
    marketSlug,
  };

  // 6) Submit order
  const result = await submitOrder(API_URL, orderPayload, {
    Cookie: `limitless_session=${session}`,
  });
  console.log('Order created:', result);
}

main().catch((e) => {
  console.error(e);
  process.exit(1);
});
```

* * *

#### Event reference (server-side)

- Namespace: `/markets`
- Subscribe:
  - `subscribe_market_prices` with `{ marketAddresses?: string[]; marketSlugs?: string[] }`
- Emits:
  - `newPriceData` for AMM prices
  - `orderbookUpdate` for CLOB orderbook

Payload shapes are minimal and may evolve; rely on keys shown in console in the snippets.

* * *

#### Notes

- Use cookies (session) in WS headers to receive authenticated streams if/when required for certain features.
- For public AMM prices or orderbook, authentication is not required.
- Scale USDC to 6 decimals when computing amounts.
- **Venue data is static** ✨: Fetch market data once per market and cache it. Do not fetch before every trade.
- **Checksummed addresses** ✨: All addresses must be in EIP-55 checksummed format. viem's `getAddress()` and `privateKeyToAccount()` return checksummed addresses by default.
- Contract addresses (exchange, adapter) are fetched dynamically from the market's venue data via `GET /markets/:slug`.

_Last updated: 2025-12-16_

## 🔌 WebSocket Integration

Real-time market data and position updates using WebSocket connections.

##### Python WebSocket Real-Time Data Guide

⚠️ **IMPORTANT DISCLAIMER** ⚠️

```properties
This is an example script for educational purposes only.
Limitless Labs is not responsible for any losses or mistakes.
This script should be adjusted to your personal needs and risk tolerance.
Always test with small amounts first and understand the code before using it.
USE AT YOUR OWN RISK.
```

This guide provides a complete walkthrough of connecting to Limitless Exchange WebSocket for real-time market data and position updates using Python.

#### Overview

The Socket.io integration process involves:

1. **Authentication**: Get a signing message and authenticate with your wallet
2. **WebSocket Connection**: Connect to the `markets`
3. **Event Subscription**: Subscribe to market prices and position updates
4. **Real-Time Events**: Handle incoming market data and position changes

#### Prerequisites

##### Required Python Packages

```bash
# Connection Commands
pip install python-socketio[asyncio] eth-account==0.10.0 requests asyncio
```

**Important**: Make sure you have compatible versions of these packages. The script has been tested with the specified dependencies.

##### Environment Variables

```bash
# Connection Commands
##### Required for authenticated features (positions, transactions)
export PRIVATE_KEY="0x..."  # Your wallet private key

##### Optional: Enable debug logging
export DEBUG=1
```

#### Complete Implementation

##### 1\. Common Utilities (`common_utils.py`)

```python
# WebSocket Client Implementation
#!/usr/bin/env python3
"""
Common Utilities for Limitless Exchange
Shared authentication and utility functions
"""

import requests
from eth_account import Account
from eth_account.messages import encode_defunct

##### Import encode_structured_data with version compatibility

try:
    from eth_account.messages import encode_typed_data
    print("✅ Successfully imported encode_typed_data from eth_account.messages")
except ImportError:
try:
    from eth_account.messages import encode_structured_data as encode_typed_data
    print("✅ Using encode_structured_data (fallback)")
except ImportError as e:
    print(f"❌ Import error: {e}")
    print("Please install the correct version:")
    print("pip install eth-account=0.10.0")
raise ImportError("encode_typed_data not available. Please upgrade eth-account: pip install eth-account==0.10.0")

def string_to_hex(text):
"""Convert string to hex representation with 0x prefix."""
return '0x' + text.encode('utf-8').hex()

def authenticate(private_key, signing_message):
"""
Authenticate with Limitless Exchange API using private key

    Args:
        private_key: Private key for signing
        signing_message: Message to sign from get_signing_message()

    Returns:
        tuple: (session_cookie, user_data)
    """
    account = Account.from_key(private_key)
    address = account.address
    print(f"Using address: {address}")

    print(f"Signing message: {repr(signing_message)}")
    message = encode_defunct(text=signing_message)
    signature = account.sign_message(message)

    hex_message = string_to_hex(signing_message)
    sig_hex = signature.signature.hex()
    if not sig_hex.startswith('0x'):
        sig_hex = '0x' + sig_hex

    headers = {
        'x-account': address,
        'x-signing-message': hex_message,
        'x-signature': sig_hex,
        'Content-Type': 'application/json',
        'Accept': 'application/json'
    }

    response = requests.post(
        'https://api.limitless.exchange/auth/login',
        headers=headers,
        json={'client': 'eoa'}
    )

    if response.status_code == 200:
        session_cookie = response.cookies.get('limitless_session')
        return session_cookie, response.json()
    else:
        raise Exception(f"Authentication failed: {response.status_code} - {response.text}")

def get_signing_message():
"""
Get signing message from Limitless Exchange API

    Returns:
        str: Signing message to be used for authentication
    """
    response = requests.get('https://api.limitless.exchange/auth/signing-message')
    if response.status_code == 200:
        return response.text
    else:
        raise Exception(f"Failed to get signing message: {response.status_code}")

##### Re-export encode_typed_data for backward compatibility

__all__ = ['authenticate', 'get_signing_message', 'string_to_hex', 'encode_typed_data']
```

##### 2\. WebSocket Client (`socket-subs.py`)

```css
In below script you need to pass MARKET_ADDRESSES, please read code.
```

```python
# WebSocket Client Implementation
#!/usr/bin/env python3
"""
Limitless Exchange WebSocket Client - Streamlined Example
Perfect for quick setup and real-time data handling
Connect to real-time market data and position updates
"""

import asyncio
import json
import logging
import os
from typing import Optional, List

import socketio
from common_utils import authenticate, get_signing_message

logging.basicConfig(level=logging.WARNING, format='%(message)s')
logger = logging.getLogger(__name__)

##### Global configuration

MARKET_ADDRESSES = ["<MARKET_ADDRESS>"] # Hourly market address fetched from Active markets

class LimitlessWebSocket:
"""
Streamlined WebSocket client for Limitless Exchange
Essential functionality: authentication, connection, subscription, events
"""

    def __init__(self, websocket_url: str = "wss://ws.limitless.exchange", private_key: Optional[str] = None):
        self.websocket_url = websocket_url
        self.private_key = private_key
        self.session_cookie = None
        self.connected = False
        self.subscribed_markets: List[str] = []

        # Socket.IO client with minimal logging
        self.sio = socketio.AsyncClient(logger=False, engineio_logger=False)
        self._setup_handlers()

    def _setup_handlers(self):
        """Setup essential event handlers"""

        @self.sio.event(namespace='/markets')
        async def connect():
            self.connected = True
            print("✅ Connected to /markets")

            # Send authentication if available
            if self.session_cookie:
                await self.sio.emit('authenticate', f'Bearer {self.session_cookie}', namespace='/markets')

            # Re-subscribe to markets after reconnection
            if self.subscribed_markets:
                await asyncio.sleep(1)
                await self._resubscribe()

        @self.sio.event(namespace='/markets')
        async def disconnect():
            self.connected = False
            print("❌ Disconnected from /markets")

        @self.sio.event(namespace='/markets')
        async def authenticated(data):
            print(f"Received packet MESSAGE data 2/markets, [\"authenticated\", {json.dumps(data)}]")

        @self.sio.event(namespace='/markets')
        async def newPriceData(data):
            """Print raw newPriceData packet"""
            print(f"Received packet MESSAGE data 2/markets, [\"newPriceData\", {json.dumps(data)}]")

        @self.sio.event(namespace='/markets')
        async def positions(data):
            """Print raw positions packet"""
            print(f"Received packet MESSAGE data 2/markets, [\"positions\", {json.dumps(data)}]")

        @self.sio.event(namespace='/markets')
        async def system(data):
            print(f"Received packet MESSAGE data 2/markets, [\"system\", {json.dumps(data)}]")

        @self.sio.event(namespace='/markets')
        async def exception(data):
            """Print raw exception packet"""
            print(f"Received packet MESSAGE data 2/markets, [\"exception\", {json.dumps(data)}]")

    async def authenticate(self):
        """Get session cookie for authentication"""
        if not self.private_key:
            print("💡 No private key - running in public mode")
            return

        try:
            print("🔐 Authenticating with private key...")
            signing_message = get_signing_message()
            self.session_cookie, user_data = authenticate(self.private_key, signing_message)
            print(f"✅ Authenticated as: {user_data['account']}")
        except Exception as e:
            print(f"❌ Authentication failed: {e}")

    async def connect(self):
        """Connect to WebSocket with working configuration"""
        try:
            # Authenticate first if private key provided
            await self.authenticate()

            # Connect with same options as working version
            print(f"🔌 Connecting to {self.websocket_url}...")

            # Prepare connection options with authentication headers if available
            connect_options = {'transports': ['websocket']}
            if self.session_cookie:
                connect_options['headers'] = {
                    'Cookie': f'limitless_session={self.session_cookie}'
                }
                print("🍪 Adding session cookie to connection headers")

            await self.sio.connect(
                self.websocket_url,
                namespaces=['/markets'],
                **connect_options
            )

            # Wait for connection to establish
            max_retries = 10
            for _ in range(max_retries):
                if self.connected:
                    break
                await asyncio.sleep(0.2)

            if self.connected:
                print("✅ Successfully connected")
            else:
                print("❌ Connection failed")

        except Exception as e:
            print(f"❌ Connection error: {e}")
            raise

    async def subscribe_markets(self, market_addresses: List[str]):
        """Subscribe to market price updates"""
        if not self.connected:
            print("❌ Not connected - call connect() first")
            return

        print(f"📊 Subscribing to {len(market_addresses)} markets")
        payload = {'marketAddresses': market_addresses}

        # Subscribe to price updates
        await self.sio.emit('subscribe_market_prices', payload, namespace='/markets')
        print("✅ Subscribed to market prices")

        # Subscribe to positions if authenticated
        if self.session_cookie:
            await self.sio.emit('subscribe_positions', payload, namespace='/markets')
            print("✅ Subscribed to positions")

        # Track subscribed markets for reconnection
        self.subscribed_markets.extend(
            addr for addr in market_addresses if addr not in self.subscribed_markets
        )

    async def _resubscribe(self):
        """Re-subscribe to markets after reconnection"""
        if self.subscribed_markets:
            await self.subscribe_markets(self.subscribed_markets)

    async def disconnect(self):
        """Disconnect from WebSocket"""
        if self.connected:
            await self.sio.disconnect()
            print("👋 Disconnected")

    async def wait(self):
        """Keep connection alive and listen for events"""
        await self.sio.wait()

def get_default_markets():
"""Helper function to access global market addresses"""
return MARKET_ADDRESSES

async def main():
"""Main execution function"""
global MARKET_ADDRESSES

    # Check environment variable for additional markets

    private_key = os.getenv('PRIVATE_KEY')

    print("=" * 50)
    print("Limitless Exchange WebSocket Client")
    print("=" * 50)

    if private_key:
        print("🔐 Private key detected - full market data mode")
    else:
        print("📊 Public mode - price data only")

    # Create and run client
    client = LimitlessWebSocket(private_key=private_key)

    try:
        # Connect to WebSocket
        await client.connect()

        # Subscribe to markets
        if MARKET_ADDRESSES:
            await client.subscribe_markets(MARKET_ADDRESSES)
        else:
            print("⚠️ No market addresses configured")
            return

        print("📡 Listening for events... Press Ctrl+C to stop")

        # Keep connection alive
        await client.wait()

    except KeyboardInterrupt:
        print("\n🛑 Shutting down...")
    except Exception as e:
        print(f"Error: {e}")
    finally:
        await client.disconnect()

async def simple_usage_example():
"""Simple example for API docs""" # Basic usage
client = LimitlessWebSocket(private_key=os.getenv('PRIVATE_KEY'))
await client.connect()
await client.subscribe_markets(MARKET_ADDRESSES)
await client.wait()

if __name__ == "__main__": # Set debug logging if needed
if os.getenv('DEBUG'):
logging.getLogger().setLevel(logging.DEBUG)

    asyncio.run(main())
```

#### Key Features

##### 🔐 Authentication Modes

The WebSocket client supports two modes:

- **Public Mode**: Access to market prices without authentication
- **Authenticated Mode**: Full access including position updates and transactions

##### 📡 Event Types

- **`newPriceData`**: Real-time price updates for markets - Public
- **`positions`**: User position changes - Required
- **`system`**: Connection status and notifications - Public
- **`authenticated`**: Authentication confirmation - Required
- **`exception`**: Error messages and exceptions - Public

##### 🎯 Market Subscription

```python
# WebSocket Client Implementation
##### Subscribe to specific markets
await client.subscribe_markets([\
    "0x1234...",  # Market address 1\
    "0x5678..."   # Market address 2\
])
```

#### Usage Examples

##### Basic Connection (Public Data)

```python
# WebSocket Client Implementation
import asyncio
from socket_subs import LimitlessWebSocket

async def public_data_example():
    client = LimitlessWebSocket()
    await client.connect()
    await client.subscribe_markets(["0x1234..."])
    await client.wait()

asyncio.run(public_data_example())
```

##### Authenticated Connection (Full Data)

```python
# WebSocket Client Implementation
import os
import asyncio
from socket_subs import LimitlessWebSocket

async def authenticated_example():
    private_key = os.getenv('PRIVATE_KEY') ##please define on shell level export PRIVATE_KEY='0x...'
    client = LimitlessWebSocket(private_key=private_key)

    await client.connect()
    await client.subscribe_markets(["0x1234..."]) # on script level is passed automatically
    await client.wait()

asyncio.run(authenticated_example())
```

##### Custom Event Handling

```python
# WebSocket Client Implementation
class CustomWebSocket(LimitlessWebSocket):
    def _setup_handlers(self):
        super()._setup_handlers()

        @self.sio.event(namespace='/markets')
        async def newPriceData(data):
            """Custom price data handler"""
            market_address = data.get('marketAddress')
            prices = data.get('updatedPrices', {})
            print(f"Market {market_address}: YES={prices.get('yes')}, NO={prices.get('no')}")

        @self.sio.event(namespace='/markets')
        async def positions(data):
            """Custom position handler"""
            account = data.get('account')
            positions = data.get('positions', [])
            print(f"User {account} has {len(positions)} positions")
```

#### Important Notes

##### ⚠️ Security Considerations

- **NEVER** share your private key with anyone
- **NEVER** commit your private key to version control
- Use environment variables to store sensitive information
- Test with small amounts first
- Understand the code before using it with real funds

##### WebSocket Configuration

- **URL**: `wss://ws.limitless.exchange` (production)
- **Namespace**: `/markets` for trading data
- **Transport**: WebSocket only (no polling fallback)
- **Authentication**: JWT session cookies

##### Connection Management

- **Auto-reconnection**: Built-in reconnection logic
- **Market re-subscription**: Automatic re-subscription after reconnection
- **Error handling**: Comprehensive error logging and recovery

##### Market Address Format

Market addresses are contract addresses in hexadecimal format, example:

```nginx
0x1234567890123456789012345678901234567890
```

Get active market addresses from:

- API endpoint: `/markets`
- WebSocket events
- Market browser

#### Event Data Schemas

##### Price Update Event

```json
{
  "marketAddress": "0x1234...",
  "updatedPrices": {
    "yes": "0.65",
    "no": "0.35"
  },
  "blockNumber": 12345678,
  "timestamp": "2024-01-01T00:00:00.000Z"
}
```

##### Position Update Event

```json
{
  "account": "0xabcd...",
  "marketAddress": "0x1234...",
  "positions": [\
    {\
      "tokenId": "123456",\
      "balance": "1000000",\
      "outcomeIndex": 0\
    }\
  ],
  "type": "AMM"
}
```

##### System Message Event

```json
{
  "message": "Successfully subscribed to market price updates",
  "markets": ["0x1234...", "0x5678..."]
}
```

#### Troubleshooting

##### Common Issues

1. **Import Error for socketio**
   - Solution: Install with async support: `pip install python-socketio[asyncio]`
2. **Connection Failed**
   - Check WebSocket URL is correct
   - Verify network connectivity
   - Check if market addresses are valid
3. **Authentication Failed**
   - Verify private key format (with or without '0x' prefix)
   - Check that signing message request succeeds
   - Ensure wallet has sufficient permissions
4. **No Data Received**
   - Verify market addresses are active
   - Check subscription was successful
   - Enable debug logging to see raw events

##### Debug Mode

Enable detailed logging:

```bash
# Connection Commands
export DEBUG=1
python socket-subs.py
```

#### Environment Variables

Set these environment variables for production use:

```bash
# Connection Commands
##### Required for authenticated features
export PRIVATE_KEY="0x..."  # Your wallet private key

##### Optional debugging
export DEBUG=1  # Enable debug logging
```

#### WebSocket Events Reference

- **`connect`**: Server → Client - None - Connection established
- **`disconnect`**: Server → Client - None - Connection lost
- **`subscribe_market_prices`**: Client → Server - None - Subscribe to price updates
- **`subscribe_positions`**: Client → Server - Required - Subscribe to position updates
- **`newPriceData`**: Server → Client - None - Market price update
- **`positions`**: Server → Client - Required - Position balance update
- **`system`**: Server → Client - None - System notifications
- **`authenticated`**: Server → Client - Required - Authentication confirmation
- **`exception`**: Server → Client - None - Error notifications

#### Performance Considerations

- **Connection pooling**: Reuse connections when possible
- **Event batching**: Handle multiple events efficiently
- **Memory usage**: Monitor memory consumption for long-running processes
- **Reconnection limits**: Implement exponential backoff for failed connections

#### Disclaimer

**This script is provided for educational purposes only. Limitless Labs is not responsible for any losses, mistakes, or unintended consequences from using this code. Trading involves risk, and you should never trade more than you can afford to lose. Always understand the code you're running and test with small amounts first.**

#### Support

For API support and questions:

- Documentation: [https://limitlesslabs.notion.site/](https://limitlesslabs.notion.site/)
- Support: [hey@limitless.network](mailto:hey@limitless.network)

* * *

_Last updated: 2025-09-05_

Server

Server:https://api.limitless.exchange

Production API

## AuthenticationOptional

Select Auth Type

|
|

No authentication selected

Client Libraries

Shell

Ruby

Node.js

PHP

Python

More Select from all clients

Shell Curl

## Authentication

​Copy link

User authentication and session management

Authentication Operations

- get/auth/signing-message
- get/auth/verify-auth
- post/auth/login
- post/auth/logout

### Get signing message

​Copy link

Returns a signing message with a randomly generated nonce for authentication purposes.

Responses

- 200







A signing message containing a nonce has been successfully generated







application/json


Request Example for get/auth/signing-message

Shell Curl

```curl
curl https://api.limitless.exchange/auth/signing-message
```

Test Request(get /auth/signing-message)

Status: 200

Show Schema

```json
Welcome to Limitless.exchange! Please sign this message to verify your identity.

Nonce: 0xa1b2c3d4e5f67890...
```

A signing message containing a nonce has been successfully generated

### Verify authentication

​Copy link

Verifies if the user is authenticated by checking the session cookie

Responses

- 200







User is authenticated







application/json

- 401







User is not authenticated







application/json


Request Example for get/auth/verify-auth

Shell Curl

```curl
curl https://api.limitless.exchange/auth/verify-auth
```

Test Request(get /auth/verify-auth)

Status: 200Status: 401

Show Schema

```json
0x1234567890123456789012345678901234567890
```

User is authenticated

### User login

​Copy link

Authenticates a user with a signed message and creates a session

Headers

- x-accountCopy link to x-account



Type: string

required









The Ethereum address of the user

- x-signing-messageCopy link to x-signing-message



Type: string

required









The signing message generated by the server

- x-signatureCopy link to x-signature



Type: string

required









The signature generated by signing the message with the user's wallet


Body

required

application/json

- clientCopy link to client



Type: stringenum

required



Example

eoa









Client type for authentication







  - eoa

  - etherspot

  - base


- rCopy link to r



Type: string





Referral code associated with the user who referred (invited) this user

- smartWalletCopy link to smartWallet



Type: string

Example

0x1234567890123456789012345678901234567890









Smart wallet address (required for Smart Wallet client)


Responses

- 200







User has been successfully logged in and a session cookie has been set







application/json

- 400







Bad request (missing or invalid parameters)







application/json

- 500







Internal server error







application/json


Request Example for post/auth/login

Shell Curl

```curl
curl https://api.limitless.exchange/auth/login \
  --request POST \
  --header 'x-account: ' \
  --header 'x-signing-message: ' \
  --header 'x-signature: ' \
  --header 'Content-Type: application/json' \
  --data '{
  "client": "eoa",
  "smartWallet": "0x1234567890123456789012345678901234567890",
  "r": ""
}'
```

Test Request(post /auth/login)

Status: 200Status: 400Status: 500

Show Schema

```json
{
  "account": "0x1234567890123456789012345678901234567890",
  "displayName": "0x1234...7890",
  "smartWallet": "0x0987654321098765432109876543210987654321",
  "client": "eoa"
}
```

User has been successfully logged in and a session cookie has been set

### User logout

​Copy link

Logs out the user by clearing the session cookie

Responses

- 200







User has been successfully logged out







application/json


Request Example for post/auth/logout

Shell Curl

```curl
curl https://api.limitless.exchange/auth/logout \
  --request POST
```

Test Request(post /auth/logout)

Status: 200

Show Schema

```json
{
  "message": "Logged out successfully"
}
```

User has been successfully logged out

## Markets  (Collapsed)

​Copy link

Browse, search, and analyze prediction markets

Markets Operations

- get/markets/active/{categoryId}
- get/markets/active
- get/markets/categories/count
- get/markets/active/slugs
- get/markets/{addressOrSlug}
- get/markets/{slug}/get-feed-events
- get/markets/search

Show More

## Trading  (Collapsed)

​Copy link

Create, manage, and cancel orders

Trading Operations

- get/markets/{slug}/historical-price
- get/markets/{slug}/orderbook
- get/markets/{slug}/locked-balance
- get/markets/{slug}/user-orders
- get/markets/{slug}/events
- post/orders
- delete/orders/{orderId}
- post/orders/cancel-batch
- delete/orders/all/{slug}

Show More

## Portfolio  (Collapsed)

​Copy link

Position tracking, trade history, and performance

Portfolio Operations

- get/portfolio/trades
- get/portfolio/positions
- get/portfolio/history
- get/portfolio/points
- get/portfolio/trading/allowance

Show More

## Public Portfolio  (Collapsed)

​Copy link

Public Portfolio Operations

- get/portfolio/{account}/traded-volume
- get/portfolio/{account}/positions

Show More

## Models

Show More

Show sidebar

Show search

- Close Group

Authentication










  - [Get signing message\\
    \\
    HTTP Method:\\
    GET](https://api.limitless.exchange/workspace/default/request/kAFCTzc_d4Fv5xBMM90s0)

  - [Verify authentication\\
    \\
    HTTP Method:\\
    GET](https://api.limitless.exchange/workspace/default/request/g7DY6B3kYqUz1UxeJTqfg)

  - [User login\\
    \\
    HTTP Method:\\
    POST](https://api.limitless.exchange/workspace/default/request/TLAbGoJ-NXvPL1xuXz4is)

  - [User logout\\
    \\
    HTTP Method:\\
    POST](https://api.limitless.exchange/workspace/default/request/1_j3ZNy6ixvjuomXNSrq0)


- Open Group

Markets

- Open Group

Trading

- Open Group

Portfolio

- Open Group

Public Portfolio


GET

Server: https://api.limitless.exchange

/auth/signing-message

Send Send get request to https://api.limitless.exchange/auth/signing-message

Close ClientClose Client

Get signing message

AllAuthCookiesHeadersQuery

All

## AuthenticationOptional

Select Auth Type

No authentication selected

## Path Parameters

## Cookies

| Cookie Enabled | Cookie Key | Cookie Value |
| --- | --- | --- |
|  | Key | Value |

## Headers

Clear All Headers

| Header Enabled | Header Key | Header Value |
| --- | --- | --- |
|  | Accept | \*/\* |
|  | Key | Value |

## Query Parameters

| Parameter Enabled | Parameter Key | Parameter Value |
| --- | --- | --- |
|  | Key | Value |

## Code Snippet (Collapsed)

Curl

Response

AllCookiesHeadersBody

All

[Powered By Scalar.com](https://www.scalar.com/)

.,,uod8B8bou,,. ..,uod8BBBBBBBBBBBBBBBBRPFT?l!i:. \|\|\|\|\|\|\|\|\|\|\|\|\|\|!?TFPRBBBBBBBBBBBBBBB8m=, \|\|\|\| '""^^!!\|\|\|\|\|\|\|\|\|\|TFPRBBBVT!:...! \|\|\|\| '""^^!!\|\|\|\|\|?!:.......! \|\|\|\| \|\|\|\|.........! \|\|\|\| \|\|\|\|.........! \|\|\|\| \|\|\|\|.........! \|\|\|\| \|\|\|\|.........! \|\|\|\| \|\|\|\|.........! \|\|\|\| \|\|\|\|.........! \|\|\|\|, \|\|\|\|.........\` \|\|\|\|\|!!-.\_ \|\|\|\|.......;. ':!\|\|\|\|\|\|\|\|\|!!-.\_ \|\|\|\|.....bBBBBWdou,. bBBBBB86foi!\|\|\|\|\|\|\|!!-..:\|\|\|!..bBBBBBBBBBBBBBBY! ::!?TFPRBBBBBB86foi!\|\|\|\|\|\|\|\|!!bBBBBBBBBBBBBBBY..! :::::::::!?TFPRBBBBBB86ftiaabBBBBBBBBBBBBBBY....! :::;\`"^!:;::::::!?TFPRBBBBBBBBBBBBBBBBBBBY......! ;::::::...''^::::::::::!?TFPRBBBBBBBBBBY........! .ob86foi;::::::::::::::::::::::::!?TFPRBY..........\` .b888888888886foi;:::::::::::::::::::::::..........\` .b888888888888888888886foi;::::::::::::::::...........b888888888888888888888888888886foi;:::::::::......\`!Tf998888888888888888888888888888888886foi;:::....\` '"^!\|Tf9988888888888888888888888888888888!::..\` '"^!\|Tf998888888888888888888888889!! '\` '"^!\|Tf9988888888888888888!!\` iBBbo. '"^!\|Tf998888888889!\` WBBBBbo. '"^!\|Tf9989!\` YBBBP^' '"^!\` \`

Send Request

ctrlControl

↵Enter
