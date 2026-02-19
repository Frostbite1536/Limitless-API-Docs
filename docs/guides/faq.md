# Frequently Asked Questions (FAQ)

Common questions about the Limitless Exchange API.

## Authentication

### How do I authenticate with the API?

Use **API key authentication** (recommended):

1. Log in to [limitless.exchange](https://limitless.exchange) using your wallet
2. Go to profile menu → "Api keys"
3. Generate a new key
4. Include `X-API-Key: lmts_your_key_here` header in all requests

```bash
curl -H "X-API-Key: lmts_your_key_here" https://api.limitless.exchange/portfolio/positions
```

> **Note**: Cookie-based session authentication is **deprecated** and will be removed within weeks. Migrate to API keys immediately.

See [Authentication Guide](../endpoints/authentication.md) for details.

### What wallets are supported?

- **EOA wallets**: Standard Ethereum wallets (MetaMask, etc.)
- **Smart wallets**: Contract-based wallets (set `client: "smart_wallet"`)

### Do I need authentication for all endpoints?

No. Public endpoints (markets, orderbook) don't require authentication. Order management and portfolio endpoints require API key authentication.

### How do I migrate from cookie auth to API key?

1. Generate an API key via the UI (profile menu → Api keys)
2. Replace cookie header with API key header:
   - Before: `Cookie: limitless_session=your_session_token`
   - After: `X-API-Key: lmts_your_key_here`
3. Remove session management code - no more login flow needed

---

## Trading

### How do I place an order?

1. Set up API key authentication
2. Get market data to obtain position IDs
3. Calculate maker/taker amounts
4. Sign order using EIP-712
5. Submit to `POST /orders`

See [Python Quickstart](../quickstart/python.md) for complete example.

### What's the difference between GTC and FOK orders?

| Type | Name | Behavior |
|------|------|----------|
| GTC | Good Till Cancelled | Stays open until filled or cancelled |
| FOK | Fill Or Kill | Must fill completely immediately or fails |

### How do I calculate the order price?

```python
price_cents = 65  # 65 cents = $0.65
shares = 100
scaling_factor = 1_000_000  # USDC decimals

price_dollars = price_cents / 100  # 0.65
total_cost = price_dollars * shares  # 65

maker_amount = int(total_cost * scaling_factor)  # 65000000
taker_amount = int(shares * scaling_factor)  # 100000000
```

### What is the tokenId in orders?

The `tokenId` is the position ID for the outcome you're trading:
- For YES tokens: Use `positionIds[0]` from market data
- For NO tokens: Use `positionIds[1]` from market data

### Why did my order fail?

Common reasons:
- **Invalid signature**: Check EIP-712 signing implementation
- **Insufficient balance**: Check USDC balance
- **Market closed**: Market may be resolved or expired
- **Price out of range**: Prices must be between 0.01 and 0.99
- **Not approved**: Approve USDC spending for `venue.exchange` (see [Required Approvals](#what-are-the-contract-addresses))

---

## Markets

### How do I get market data?

```python
# Get single market
GET /markets/{slug}

# Get all active markets
GET /markets/active?page=1&limit=10&sortBy=volume
```

### What's the difference between CLOB and AMM markets?

| Type | Description |
|------|-------------|
| CLOB | Central Limit Order Book - users place limit orders |
| AMM | Automated Market Maker - trades against liquidity pool |

### What is a NegRisk group?

A NegRisk group contains multiple related markets sharing collateral. For example, "Election Predictions" group might contain markets for each candidate.

### How do I get historical price data?

```python
GET /markets/{slug}/historical-price?from=2024-01-01&to=2024-01-31&interval=1d
```

Intervals: `1h`, `6h`, `1d`, `1w`, `1m`, `all`

---

## WebSocket

### How do I connect to real-time data?

```typescript
import { io } from 'socket.io-client';

const socket = io('wss://ws.limitless.exchange/markets', {
  transports: ['websocket']
});

socket.emit('subscribe_market_prices', {
  marketSlugs: ['btc-100k-2024']
});

socket.on('orderbookUpdate', (data) => console.log(data));
```

### Why am I not receiving updates?

- Make sure you subscribe after connection is established
- Subscriptions replace previous ones - include all markets in one call
- Check if the market has activity

### Do I need authentication for WebSocket?

No, public data streams don't require authentication. Pass the `X-API-Key` header during the connection handshake for authenticated streams (e.g., position updates).

---

## Claiming & Redeeming

### How do I claim/redeem my winnings after a market resolves?

There is **no REST API endpoint** for claiming/redeeming. Redemption is an on-chain operation. You must call `redeemPositions()` on the appropriate smart contract on Base.

1. Check market resolution via `GET /markets/{slug}` (look for `status: "RESOLVED"`)
2. Verify your winning token balance via `GET /portfolio/positions`
3. Call `redeemPositions()` on the Conditional Tokens contract (standard CLOB) or the NegRisk Adapter (NegRisk markets)

See the full [Claiming & Redeeming Guide](claiming-redeeming.md) for code examples in TypeScript and Python.

### Which smart contract do I use for redemption?

| Market Type | Contract |
|-------------|----------|
| Standard CLOB | Conditional Tokens: `0xc9c98965297bc527861c898329ee280632b76e18` |
| NegRisk | Use `market.venue.adapter` from market data |

### Can I exit a position before the market resolves?

Yes, two options:
1. **Sell on the order book** using `POST /orders` (creates a SELL order)
2. **Merge positions** on-chain: if you hold equal YES and NO tokens, call `mergePositions()` on the CTF contract to get USDC back without selling

### Does `mergePositions` work with embedded / smart wallets (Safe)?

`mergePositions` is a function on the CTF contract and works regardless of wallet type — the requirement is that the **calling wallet holds the tokens**. However, if your positions are in a Gnosis Safe (which happens when you use the Limitless frontend), you **cannot** call `mergePositions` programmatically because Limitless does not allow export of the Privy-managed embedded wallet key. Use the Limitless web interface for Safe-held positions, or trade via a fresh EOA you control for programmatic merge/redeem access. See the [Claiming Rewards After Close](../user-questions/claim-rewards-after-close.md#does-mergepositions-work-with-embedded--smart-wallets-safe) guide for full details.

---

## Portfolio

### How do I get my positions?

```python
GET /portfolio/positions
# Requires: X-API-Key header
```

### How is P&L calculated?

| Metric | Formula |
|--------|---------|
| Cost | Total USDC spent to acquire tokens |
| Fill Price | Average entry price (cost / quantity) |
| Unrealized P&L | (current_price - fill_price) * balance |
| Realized P&L | Proceeds from sales - cost basis of sold tokens |
| Market Value | current_price * balance |

### How do I see my trade history?

```python
GET /portfolio/history?page=1&limit=20
# Returns paginated transaction history
```

---

## Technical

### What blockchain is Limitless on?

Base mainnet (Chain ID: 8453)

### What are the contract addresses?

**Do NOT hardcode contract addresses.** Each market has its own venue with contract addresses. Fetch them from market data:

```python
market = get_market("btc-100k-2024")
venue_exchange = market['venue']['exchange']  # Use as verifyingContract in EIP-712
venue_adapter = market['venue']['adapter']    # For NegRisk SELL orders
```

Cache venue data per market - it's static and doesn't change.

### What collateral is used?

USDC with 6 decimals. 1 USDC = 1,000,000 raw units.

### How do I handle large numbers?

Use string representation for large integers (like tokenIds) to avoid precision loss:
```javascript
// Wrong - loses precision
const tokenId = 19633204485790857949828516737993423758628930235371629943999544859324645414627;

// Correct - use string
const tokenId = "19633204485790857949828516737993423758628930235371629943999544859324645414627";
```

### What's the API rate limit?

Rate limits are enforced. Check response headers for current limits. Implement exponential backoff for retries.

---

## Errors

### What does error 401 mean?

Unauthorized - your API key is invalid/revoked, or you're accessing an authenticated endpoint without an `X-API-Key` header. Generate a new API key at limitless.exchange if needed.

### What does error 400 mean?

Bad request - check your request parameters. Common causes:
- Missing required fields
- Invalid address format
- Invalid price range

### What does error 404 mean?

Resource not found - the market, order, or other resource doesn't exist or has been removed.

### What does error 500 mean?

Server error - try again later. If persistent, contact support.

---

## User Questions

For detailed troubleshooting of specific issues, see the [User Questions](../user-questions/) section:

| Issue | Link |
|-------|------|
| "Signer does not match" error | [Smart Wallet Signer Mismatch](../user-questions/smart-wallet-signer-mismatch.md) |
| Signature verification fails | [Signature Verification Failed](../user-questions/signature-verification-failed.md) |
| Order accepted but not filling | [Order Not Filling](../user-questions/order-not-filling.md) |
| Invalid token ID error | [Invalid Token ID](../user-questions/invalid-token-id.md) |

---

## Best Practices

### Security

- **Never** expose API keys or private keys in code
- Use environment variables for sensitive data (`LIMITLESS_API_KEY`, `PRIVATE_KEY`)
- Test with small amounts first
- Verify contract addresses before approving
- Rotate API keys periodically via the Limitless Exchange UI

### Performance

- Cache market data when appropriate
- Use WebSocket for real-time data instead of polling
- Batch order cancellations when possible

### Error Handling

```python
try:
    result = api_call()
except HTTPError as e:
    if e.response.status_code == 401:
        # Check API key is valid, generate new one if needed
        raise Exception("Authentication failed - check your API key")
    elif e.response.status_code == 429:
        # Rate limited - back off
        time.sleep(5)
        result = api_call()
    else:
        raise
```

---

## Support

- **Documentation**: https://limitlesslabs.notion.site/
- **Email**: hey@limitless.network
- **Issues**: Report bugs or feature requests

## Disclaimer

This documentation is for educational purposes. Limitless Labs is not responsible for any losses from trading. Always understand the code and test with small amounts first.
