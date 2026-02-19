# Enabling API Trading on a New Account

## Question

> I have an issue with API, do I need to activate something for a new account / new address to be able to use API for trading? I have an API key already. Can you provide a working Python example for generating the EIP-712 signature?

## Short Answer

Having an API key is necessary but **not sufficient** for trading. You also need:

1. **USDC balance** on Base (chain ID 8453) in the wallet whose private key you use for signing
2. **Token approvals** — approve USDC spending to the venue's exchange contract
3. **Correct EIP-712 signing** — use the venue's `exchange` address from market data as `verifyingContract`

There is no separate "enable trading" toggle. If you have an API key and a funded wallet with approvals, you can trade immediately.

> **Recommended**: Use the [`limitless-sdk`](../guides/sdk.md) (`pip install limitless-sdk`) — it handles EIP-712 signing, venue resolution, and order management automatically. See the [SDK Guide](../guides/sdk.md) for a complete quickstart.

## Checklist for New Accounts

| Step | What | How |
|------|------|-----|
| 1 | Generate API key | Profile menu → Api keys on [limitless.exchange](https://limitless.exchange) |
| 2 | Fund wallet with USDC on Base | Bridge or transfer USDC to your wallet on Base (chain 8453) |
| 3 | Approve USDC for the venue contract | On-chain approval (see below) |
| 4 | Use correct EIP-712 signing | Fetch `venue.exchange` from market data for `verifyingContract` |

### Why Orders Might Fail on a New Account

| Symptom | Cause | Fix |
|---------|-------|-----|
| `401 Unauthorized` | API key missing or invalid | Check `X-API-Key` header, ensure key starts with `lmts_` |
| `Invalid signature` | Wrong `verifyingContract` | Fetch `venue.exchange` from `GET /markets/{slug}` |
| `Invalid signature` | Address not checksummed | Use `Web3.to_checksum_address()` |
| `Insufficient allowance` | USDC not approved for venue contract | Run the approval script below (one-time per venue) |
| `Insufficient balance` | No USDC on Base | Bridge or transfer USDC to your wallet on Base mainnet |
| Order accepted but no fill | GTC order waiting for a match | This is normal — your limit order is on the book |
| `Signer does not match` | Wallet linked to smart wallet via frontend | Use a fresh wallet for API-only trading (see note below) |

> **Important**: If you have ever logged into the Limitless web frontend with this wallet, it may have been linked to a smart wallet. This makes simple EOA signing fail. **Use a dedicated wallet for API trading** that has never been used on the web frontend.

## Token Approval (On-Chain)

Before your first BUY order on a market, you must approve USDC spending to the venue's exchange contract. This is a one-time on-chain transaction per venue contract.

```python
from web3 import Web3

# Connect to Base
w3 = Web3(Web3.HTTPProvider("https://mainnet.base.org"))

# USDC on Base
USDC_ADDRESS = "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913"
USDC_ABI = [
    {
        "name": "approve",
        "type": "function",
        "inputs": [
            {"name": "spender", "type": "address"},
            {"name": "amount", "type": "uint256"}
        ],
        "outputs": [{"name": "", "type": "bool"}]
    }
]

def approve_usdc(private_key, spender_address, amount=2**256 - 1):
    """
    Approve USDC spending for a venue exchange contract.
    Call once per venue contract before placing BUY orders.

    Args:
        private_key: Your wallet private key
        spender_address: venue.exchange address from market data
        amount: Approval amount (default: max uint256 for unlimited)
    """
    account = w3.eth.account.from_key(private_key)
    usdc = w3.eth.contract(address=USDC_ADDRESS, abi=USDC_ABI)

    tx = usdc.functions.approve(
        Web3.to_checksum_address(spender_address),
        amount
    ).build_transaction({
        "from": account.address,
        "nonce": w3.eth.get_transaction_count(account.address),
        "gas": 100000,
        "gasPrice": w3.eth.gas_price,
        "chainId": 8453,
    })

    signed_tx = w3.eth.account.sign_transaction(tx, private_key)
    tx_hash = w3.eth.send_raw_transaction(signed_tx.raw_transaction)
    receipt = w3.eth.wait_for_transaction_receipt(tx_hash)
    print(f"USDC approval tx: {tx_hash.hex()} (status: {receipt['status']})")
    return receipt
```

## Complete Working Python Example

This is a self-contained script that fetches market data, builds, signs, and submits an order.

### Prerequisites

```bash
pip install requests eth-account web3
```

```bash
export LIMITLESS_API_KEY="lmts_..."   # Your API key
export PRIVATE_KEY="0x..."            # Wallet private key (for signing)
```

### Full Script

```python
import os
import time
import random
import requests
from eth_account import Account
from eth_account.messages import encode_typed_data

# ─── Configuration ────────────────────────────────────────────
API_URL = "https://api.limitless.exchange"
API_KEY = os.environ["LIMITLESS_API_KEY"]
PRIVATE_KEY = os.environ["PRIVATE_KEY"]
CHAIN_ID = 8453  # Base mainnet

ACCOUNT = Account.from_key(PRIVATE_KEY)
MAKER_ADDRESS = ACCOUNT.address  # Already checksummed


# ─── 1. Authentication ────────────────────────────────────────
def auth_headers():
    return {"X-API-Key": API_KEY}


# ─── 2. Market Data (cached) ─────────────────────────────────
MARKET_CACHE = {}

def get_market(slug):
    """Fetch and cache market data. Venue info is static per market."""
    if slug not in MARKET_CACHE:
        resp = requests.get(f"{API_URL}/markets/{slug}")
        resp.raise_for_status()
        MARKET_CACHE[slug] = resp.json()
    return MARKET_CACHE[slug]


# ─── 3. EIP-712 Signature Generation ─────────────────────────
EIP712_ORDER_TYPES = {
    "Order": [
        {"name": "salt",          "type": "uint256"},
        {"name": "maker",         "type": "address"},
        {"name": "signer",        "type": "address"},
        {"name": "taker",         "type": "address"},
        {"name": "tokenId",       "type": "uint256"},
        {"name": "makerAmount",   "type": "uint256"},
        {"name": "takerAmount",   "type": "uint256"},
        {"name": "expiration",    "type": "uint256"},
        {"name": "nonce",         "type": "uint256"},
        {"name": "feeRateBps",    "type": "uint256"},
        {"name": "side",          "type": "uint8"},
        {"name": "signatureType", "type": "uint8"},
    ]
}

def sign_order(order_payload, venue_exchange):
    """
    Sign an order using EIP-712 typed data.

    Args:
        order_payload: Dict with all order fields (salt, maker, signer, etc.)
        venue_exchange: The venue's exchange contract address from market data.
                        This MUST come from market['venue']['exchange'].

    Returns:
        Hex signature string starting with '0x'.
    """
    domain = {
        "name": "Limitless CTF Exchange",
        "version": "1",
        "chainId": CHAIN_ID,
        "verifyingContract": venue_exchange,  # CRITICAL: from market data
    }

    signable = encode_typed_data(domain, EIP712_ORDER_TYPES, order_payload)
    signed = ACCOUNT.sign_message(signable)
    return "0x" + signed.signature.hex()


# ─── 4. Order Building ───────────────────────────────────────
def build_order(token_id, maker_amount, taker_amount, side=0, fee_rate_bps=0):
    """
    Build an unsigned order payload.

    Args:
        token_id: Position ID (from market['positionIds'][0] for YES, [1] for NO)
        maker_amount: USDC amount in 6 decimals (what you pay)
        taker_amount: Shares amount in 6 decimals (what you receive)
        side: 0 = BUY, 1 = SELL
        fee_rate_bps: Fee in basis points (0 unless you know your fee tier)

    Returns:
        Order payload dict ready for signing.
    """
    return {
        "salt": int(time.time() * 1000) + random.randint(0, 1000),
        "maker": MAKER_ADDRESS,
        "signer": MAKER_ADDRESS,
        "taker": "0x0000000000000000000000000000000000000000",
        "tokenId": str(token_id),
        "makerAmount": str(maker_amount),
        "takerAmount": str(taker_amount),
        "expiration": "0",
        "nonce": "0",
        "feeRateBps": str(fee_rate_bps),
        "side": side,
        "signatureType": 0,  # 0 = EOA wallet
    }


# ─── 5. Submit Order ─────────────────────────────────────────
def submit_order(order_payload, market_slug, order_type="GTC"):
    """Submit a signed order to the API."""
    body = {
        "order": order_payload,
        "ownerId": 0,  # Replace with your user ID if known
        "orderType": order_type,
        "marketSlug": market_slug,
    }
    resp = requests.post(f"{API_URL}/orders", json=body, headers=auth_headers())
    resp.raise_for_status()
    return resp.json()


# ─── 6. Main: Place a Trade ──────────────────────────────────
def place_trade(market_slug, token_type, price_cents, num_shares):
    """
    End-to-end trade placement.

    Args:
        market_slug: e.g. "btc-100k-2024"
        token_type: "YES" or "NO"
        price_cents: Price per share in cents (1-99)
        num_shares: Number of shares to buy
    """
    # Step 1 — Fetch market data (cached)
    market = get_market(market_slug)
    venue_exchange = market["venue"]["exchange"]
    position_ids = market["positionIds"]
    token_id = position_ids[0] if token_type == "YES" else position_ids[1]

    print(f"Market:   {market_slug}")
    print(f"Venue:    {venue_exchange}")
    print(f"Token:    {token_type} (ID: {token_id})")

    # Step 2 — Calculate amounts (USDC has 6 decimals)
    price = price_cents / 100
    maker_amount = int(price * num_shares * 1_000_000)
    taker_amount = int(num_shares * 1_000_000)

    print(f"Price:    ${price:.2f}")
    print(f"Cost:     ${price * num_shares:.2f} USDC")

    # Step 3 — Build order
    order = build_order(token_id, maker_amount, taker_amount)

    # Step 4 — Sign with EIP-712 (venue.exchange as verifyingContract)
    order["signature"] = sign_order(order, venue_exchange)
    order["price"] = price  # Required for GTC orders

    # Step 5 — Submit
    print("Submitting order...")
    result = submit_order(order, market_slug, order_type="GTC")
    print(f"Order ID: {result['order']['orderId']}")
    return result


# ─── Entry Point ──────────────────────────────────────────────
if __name__ == "__main__":
    # Example: Buy 10 YES shares at $0.50 on a market
    place_trade(
        market_slug="btc-100k-2024",  # Replace with a real active market slug
        token_type="YES",
        price_cents=50,
        num_shares=10,
    )
```

### How EIP-712 Signing Works (Step by Step)

1. **Fetch `venue.exchange`** from market data — this is the smart contract address that verifies your signature
2. **Build the EIP-712 domain** with `name="Limitless CTF Exchange"`, `version="1"`, `chainId=8453`, and `verifyingContract=venue.exchange`
3. **Define the `Order` type structure** — this must match exactly (see `EIP712_ORDER_TYPES` above)
4. **Encode the typed data** using `eth_account.messages.encode_typed_data(domain, types, order_payload)`
5. **Sign** with `account.sign_message(signable)` to produce the signature
6. **Attach** the hex signature to your order payload before submitting

### Common Mistakes

| Mistake | What Happens | Fix |
|---------|-------------|-----|
| Hardcoding `verifyingContract` | `Invalid signature` error | Always fetch `venue.exchange` from `GET /markets/{slug}` |
| Using lowercase address | `Invalid signature` error | Use `Account.from_key(key).address` (already checksummed) |
| Wrong chain ID | `Invalid signature` error | Must be `8453` (Base mainnet) |
| Missing `price` field | Order rejected | Add `order["price"] = price` for GTC orders |
| Used wallet on web frontend | `Signer does not match` | Use a fresh wallet that was never used on limitless.exchange |

## Pre-Flight Checklist

Before placing your first order, confirm every item:

- [ ] API key generated at limitless.exchange (profile → Api keys)
- [ ] `LIMITLESS_API_KEY` and `PRIVATE_KEY` exported as environment variables
- [ ] USDC funded on Base mainnet (chain 8453)
- [ ] USDC approved for the venue's exchange contract (one-time per venue)
- [ ] `venue.exchange` fetched from `GET /markets/{slug}` (never hardcoded)
- [ ] Chain ID is `8453` in EIP-712 domain
- [ ] Wallet has **never** been used on the Limitless web frontend (to avoid smart wallet linking)

## If You Still Can't Trade

If you've confirmed all the above and orders still fail:

1. Open the support chat on [limitless.exchange](https://limitless.exchange) and provide:
   - Your wallet address
   - The exact error message from the API response
   - The market slug you're trying to trade on
2. Check that the market is still active (not resolved)
3. Verify your USDC balance on Base using a block explorer

## Related

- [Python Quickstart](../quickstart/python.md) — Full Python trading guide
- [Placing Orders Guide](../guides/placing-orders.md) — Step-by-step order flow
- [Signature Verification Failed](signature-verification-failed.md) — Debugging signature errors
- [Authentication Guide](../guides/authentication.md) — API key setup
