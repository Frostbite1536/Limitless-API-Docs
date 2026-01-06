# Portfolio Endpoints

Endpoints for accessing user portfolio data including positions, trades, history, and rewards.

## Authenticated Endpoints

All portfolio endpoints require authentication via session cookie.

### GET /portfolio/positions

Get all active positions with profit/loss data.

**Authentication**: Required (session cookie)

**Response** (200):
```json
{
  "rewards": {
    "totalPoints": 1500,
    "epochs": [
      {
        "epoch": 1,
        "points": 500,
        "tradingVolume": "10000000000"
      }
    ]
  },
  "amm": [
    {
      "market": {
        "slug": "btc-50k",
        "title": "Will BTC hit $50k?"
      },
      "positions": {...},
      "tokensBalance": {...}
    }
  ],
  "clob": [
    {
      "market": {
        "address": "0x...",
        "slug": "btc-100k",
        "title": "Will BTC hit $100k?"
      },
      "positions": {
        "yes": {
          "cost": "75000000",
          "fillPrice": "750000",
          "realisedPnl": "0",
          "unrealizedPnl": "25000000",
          "marketValue": "100000000"
        },
        "no": {
          "cost": "25000000",
          "fillPrice": "250000",
          "realisedPnl": "0",
          "unrealizedPnl": "-5000000",
          "marketValue": "20000000"
        }
      },
      "tokensBalance": {
        "yes": "100000000",
        "no": "0"
      },
      "orders": {...},
      "rewards": {...}
    }
  ]
}
```

**Position Data Fields**:

| Field | Type | Description |
|-------|------|-------------|
| `cost` | string | Cost basis in USDC (6 decimals) |
| `fillPrice` | string | Average entry price (6 decimals) |
| `realisedPnl` | string | Realized P&L from closed positions |
| `unrealizedPnl` | string | Unrealized P&L at current price |
| `marketValue` | string | Current market value |

**Usage**:
```python
response = requests.get(
    "https://api.limitless.exchange/portfolio/positions",
    cookies={"limitless_session": session_cookie}
)
positions = response.json()

for position in positions['clob']:
    print(f"Market: {position['market']['title']}")
    print(f"  YES tokens: {int(position['tokensBalance']['yes']) / 1e6}")
    print(f"  Unrealized P&L: ${int(position['positions']['yes']['unrealizedPnl']) / 1e6}")
```

---

### GET /portfolio/trades

Get all trades for the authenticated user.

**Authentication**: Required (session cookie)

**Response** (200):
```json
{
  "trades": [
    {
      "market": {
        "slug": "btc-100k",
        "title": "Will BTC hit $100k?"
      },
      "side": "BUY",
      "outcome": "YES",
      "price": 0.65,
      "size": "100000000",
      "total": "65000000",
      "timestamp": "2024-01-15T10:30:00Z",
      "txHash": "0x..."
    }
  ]
}
```

---

### GET /portfolio/history

Get paginated transaction history.

**Authentication**: Required (session cookie)

**Query Parameters**:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `page` | number | No | Page number |
| `limit` | number | No | Items per page |

**Response** (200):
```json
{
  "data": [
    {
      "blockTimestamp": 1744115608,
      "collateralAmount": "65000000",
      "market": {
        "id": 980,
        "slug": "btc-100k",
        "title": "Will BTC hit $100k?",
        "deadline": "2024-12-31T23:59:59Z",
        "closed": false,
        "collateral": {
          "symbol": "USDC",
          "id": 7,
          "decimals": 6
        },
        "group": null
      },
      "outcomeTokenAmount": "100000000",
      "outcomeTokenAmounts": ["100000000", "0"],
      "outcomeIndex": 0,
      "outcomeTokenPrice": 0.65,
      "strategy": "Limit Buy",
      "transactionHash": "0x..."
    }
  ],
  "totalCount": 50
}
```

**Strategy Types**:

| Strategy | Description |
|----------|-------------|
| `Buy` | AMM buy |
| `Sell` | AMM sell |
| `Limit Buy` | CLOB limit buy order filled |
| `Limit Sell` | CLOB limit sell order filled |
| `Market Buy` | Market order buy |
| `Market Sell` | Market order sell |
| `Split` | Split collateral into YES/NO tokens |
| `Merge` | Merge YES/NO tokens into collateral |
| `Convert` | Convert positions (NegRisk) |

---

### GET /portfolio/points

Get points breakdown and rewards data.

**Authentication**: Required (session cookie)

**Response** (200):
```json
{
  "totalPoints": 15000,
  "epochs": [
    {
      "epoch": 1,
      "points": 5000,
      "tradingVolume": "50000000000",
      "liquidityProvided": "10000000000"
    },
    {
      "epoch": 2,
      "points": 10000,
      "tradingVolume": "100000000000",
      "liquidityProvided": "25000000000"
    }
  ]
}
```

---

### GET /portfolio/trading/allowance

Check USDC spending allowance for the CTF Exchange contract.

**Authentication**: Required (session cookie)

**Response** (200):
```json
{
  "allowance": "1000000000000",
  "isApproved": true
}
```

**Notes**:
- `allowance` is in USDC (6 decimals)
- `isApproved` indicates if allowance is sufficient for trading

## Public Portfolio Endpoints

These endpoints allow viewing other users' public portfolio data.

### GET /portfolio/{account}/traded-volume

Get total traded volume for a user.

**Authentication**: None required

**Path Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `account` | string | Ethereum address |

**Response** (200):
```json
{
  "totalVolume": "500000000000",
  "volumeByMarket": [
    {
      "marketSlug": "btc-100k",
      "volume": "100000000000"
    }
  ]
}
```

---

### GET /portfolio/{account}/positions

Get all positions for a specific user (public data only).

**Authentication**: None required

**Path Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `account` | string | Ethereum address |

**Response** (200):
```json
{
  "positions": [
    {
      "market": {
        "slug": "btc-100k",
        "title": "Will BTC hit $100k?"
      },
      "outcome": "YES",
      "balance": "100000000"
    }
  ]
}
```

## Understanding P&L

### Cost Basis
The total amount spent to acquire tokens, including fees.

### Fill Price
Average price per token across all fills:
```
fillPrice = cost / tokensAcquired
```

### Unrealized P&L
Profit/loss if you sold at current market price:
```
unrealizedPnl = (currentPrice - fillPrice) * tokensBalance
```

### Realized P&L
Actual profit/loss from closed positions:
```
realisedPnl = sellAmount - costBasisOfSoldTokens
```

### Market Value
Current value of position at market prices:
```
marketValue = currentPrice * tokensBalance
```

## Example: Calculating Total Portfolio Value

```python
response = requests.get(
    "https://api.limitless.exchange/portfolio/positions",
    cookies={"limitless_session": session_cookie}
)
positions = response.json()

total_value = 0
total_unrealized_pnl = 0

for pos in positions['clob']:
    for outcome in ['yes', 'no']:
        data = pos['positions'][outcome]
        total_value += int(data['marketValue'])
        total_unrealized_pnl += int(data['unrealizedPnl'])

print(f"Total Portfolio Value: ${total_value / 1e6:.2f}")
print(f"Total Unrealized P&L: ${total_unrealized_pnl / 1e6:.2f}")
```
