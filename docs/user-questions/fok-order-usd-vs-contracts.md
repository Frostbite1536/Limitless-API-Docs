# FOK Orders: USD-Denominated vs Contract-Denominated

## Question

> FOK orders on Limitless don't seem to guarantee that the exact number of contracts requested will be filled or cancelled. Instead, the order executes using the USD amount I send, filling whatever quantity of contracts that amount can buy. Is this the intended behavior?

## Short Answer

**Yes, this is the intended behavior.** FOK orders on Limitless are **USD-denominated**, not contract-denominated. The "fill or kill" guarantee applies to the **dollar amount** (`makerAmount`), not to a specific number of contracts.

## How It Works

When you submit a FOK BUY order on Limitless:

1. You specify `makerAmount` = total USDC to spend
2. You set `takerAmount` to `'1'` (market order semantics)
3. The system attempts to spend your entire `makerAmount` against available orderbook liquidity
4. If the full USDC amount can be matched, the order fills — you receive however many contracts that amount buys at current prices
5. If the full USDC amount **cannot** be matched (insufficient liquidity), the entire order is cancelled

The number of contracts you receive depends on the prices available in the orderbook at execution time.

### FOK SELL orders

For FOK SELL orders, the logic is inverted:

- `makerAmount` = number of shares to sell
- The guarantee is: sell all of those shares or cancel entirely

## Comparison with Other Platforms

| Platform | FOK Guarantee | You Specify |
|----------|---------------|-------------|
| **Limitless** | Spend all X USDC or cancel | USD amount (`makerAmount`) |
| **Polymarket** | Fill exactly N contracts at price P or cancel | Contract quantity + price |

On Polymarket, you specify both a contract quantity and a limit price. The FOK guarantee is: fill that exact quantity at that price or better, or cancel.

On Limitless, you specify a USD amount. The FOK guarantee is: spend that entire USD amount or cancel. The contract quantity is a *result* of execution, not an input.

## Example

```python
# Limitless FOK BUY: "Spend $50 or nothing"
order = await order_client.create_order(
    token_id=token_id,
    maker_amount=50.0,   # Spend $50 USDC
    side=Side.BUY,
    order_type=OrderType.FOK,
    market_slug=market.slug
)
# Result: You get however many contracts $50 buys at current orderbook prices
# (e.g., 76.9 contracts if average fill price is ~$0.65)
```

```python
# If you want a specific number of contracts at a specific price,
# use a GTC limit order instead:
order = await order_client.create_order(
    token_id=token_id,
    price=0.65,
    size=100.0,          # Exactly 100 contracts
    side=Side.BUY,
    order_type=OrderType.GTC,
    market_slug=market.slug
)
```

## Implications for Bot Developers

1. **You cannot guarantee an exact contract quantity with FOK.** If you need exactly N contracts, use a GTC limit order with a specific price and size.

2. **Check the orderbook first** to estimate how many contracts your USD amount will buy:
   ```python
   orderbook = requests.get(f"{API_URL}/markets/{slug}/orderbook").json()
   # Walk the asks to estimate fill quantity for your USD budget
   ```

3. **The `makerMatches` array in the response** tells you exactly what was filled:
   ```python
   if order.maker_matches and len(order.maker_matches) > 0:
       print(f"Filled {len(order.maker_matches)} matches")
   else:
       print("Order killed — insufficient liquidity")
   ```

4. **For precise contract quantities**, calculate the required `makerAmount` from the orderbook:
   ```python
   # To buy ~100 contracts, estimate cost from orderbook
   target_contracts = 100
   estimated_price = best_ask  # or walk the book for large orders
   maker_amount = target_contracts * estimated_price
   ```

## Related

- [FAQ: GTC vs FOK](../guides/faq.md#whats-the-difference-between-gtc-and-fok-orders)
- [Order Schema](../schemas/order.md)
- [Python SDK Guide](../guides/sdk.md)
- [Order Not Filling](order-not-filling.md)
