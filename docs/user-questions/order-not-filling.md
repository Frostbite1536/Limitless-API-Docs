# Order Accepted But Not Filling

> **Auth Update**: Code examples below use cookie-based auth which is **deprecated**. Replace `cookies={"limitless_session": session}` with `headers={"X-API-Key": "lmts_your_key_here"}`. See [Authentication Guide](../../docs/guides/authentication.md).

## Symptoms

- Order submission returns success
- Order ID is returned
- But no trade executes
- Position balance doesn't change

## Possible Causes

### 1. Price too far from market

Your limit price may not match any existing orders on the other side.

**Example**: If the best ask is $0.75 and you submit a buy at $0.65, your order will sit in the book waiting for someone to sell at $0.65.

```python
# Check the orderbook first
orderbook = get_orderbook(slug)
best_bid = orderbook['bids'][0]['price'] if orderbook['bids'] else None
best_ask = orderbook['asks'][0]['price'] if orderbook['asks'] else None

print(f"Best bid: {best_bid}, Best ask: {best_ask}")
# Then decide your price accordingly
```

### 2. Insufficient liquidity

Not enough counter-orders to fill yours, even at matching prices.

### 3. GTC order queued

Good Till Cancelled orders are designed to wait:
- They stay open until filled or manually cancelled
- This is expected behavior for limit orders

### 4. Using wrong order type

| Type | Behavior |
|------|----------|
| `GTC` | Waits for match - may not fill immediately |
| `FOK` | Fill Or Kill - fails if can't fill completely |

If you want immediate execution, consider:
- Using `FOK` order type
- Pricing at or better than the best bid/ask

## How to Check Your Orders

```python
# Check your open orders in the market
response = requests.get(
    f"{API_URL}/markets/{slug}/user-orders",
    cookies={"limitless_session": session}
)
orders = response.json()
print(orders)
```

## How to Cancel Unfilled Orders

```python
# Cancel a specific order
response = requests.delete(
    f"{API_URL}/orders/{order_id}",
    cookies={"limitless_session": session}
)

# Or cancel all orders in a market
response = requests.delete(
    f"{API_URL}/orders/all/{slug}",
    cookies={"limitless_session": session}
)
```

## Tips for Better Fills

1. **Check the orderbook** before placing orders
2. **Price competitively** - at or near the spread
3. **Use FOK** for immediate-or-nothing execution
4. **Monitor via WebSocket** for real-time orderbook updates

## Related

- [Trading Endpoints](../endpoints/trading.md)
- [Orders Endpoints](../endpoints/orders.md)
- [WebSocket Guide](../guides/websockets.md)
