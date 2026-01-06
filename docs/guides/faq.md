# Frequently Asked Questions (FAQ)

Common questions about the Limitless Exchange API.

## Authentication

### How do I authenticate with the API?

Use wallet-based authentication with EIP-712 signatures:

1. Get signing message: `GET /auth/signing-message`
2. Sign the message with your wallet
3. Login: `POST /auth/login` with signed message in headers
4. Use returned session cookie for authenticated requests

See [Authentication Guide](../endpoints/authentication.md) for details.

### What wallets are supported?

- **EOA wallets**: Standard Ethereum wallets (MetaMask, etc.)
- **Smart wallets**: Contract-based wallets (set `client: "smart_wallet"`)

### How long does a session last?

Sessions expire after a period of inactivity. If you receive a 401 error, re-authenticate.

### Do I need authentication for all endpoints?

No. Public endpoints (markets, orderbook) don't require authentication. Order management and portfolio endpoints require authentication.

---

## Trading

### How do I place an order?

1. Authenticate and get session cookie
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
- For YES tokens: Use `position_ids[0]` from market data
- For NO tokens: Use `position_ids[1]` from market data

### Why did my order fail?

Common reasons:
- **Invalid signature**: Check EIP-712 signing implementation
- **Insufficient balance**: Check USDC balance
- **Market closed**: Market may be resolved or expired
- **Price out of range**: Prices must be between 0.01 and 0.99
- **Not approved**: Approve USDC spending for CTF contract

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

No, public data streams don't require authentication. Include session cookie only if authenticated streams become available.

---

## Portfolio

### How do I get my positions?

```python
GET /portfolio/positions
# Requires: session cookie authentication
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

| Contract | Address |
|----------|---------|
| CLOB CTF Exchange | `0xa4409D988CA2218d956BeEFD3874100F444f0DC3` |
| NegRisk CTF Exchange | `0x5a38afc17F7E97ad8d6C547ddb837E40B4aEDfC6` |

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

Unauthorized - your session has expired or you're accessing an authenticated endpoint without a session cookie. Re-authenticate.

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

## Best Practices

### Security

- **Never** expose private keys in code
- Use environment variables for sensitive data
- Test with small amounts first
- Verify contract addresses before approving

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
        # Re-authenticate
        session = authenticate()
        result = api_call()
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
