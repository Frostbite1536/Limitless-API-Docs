# Trading Endpoints

Endpoints for accessing market trading data including historical prices, orderbook, and events.

## Endpoints

### GET /markets/{slug}/historical-price

Retrieve historical price data for a market with configurable time intervals.

**Authentication**: None required

**Path Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `slug` | string | Market slug identifier |

**Query Parameters**:

| Parameter | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| `from` | string | No | Start date (ISO 8601) | `2024-01-01T00:00:00Z` |
| `to` | string | No | End date (ISO 8601) | `2024-01-31T23:59:59Z` |
| `interval` | string | No | Time interval for data points | `1h`, `6h`, `1d`, `1w`, `1m`, `all` |

**Response** (200):
```json
[
  {
    "title": "YES Token",
    "prices": [
      {
        "price": 0.75,
        "timestamp": "2024-01-15T10:30:00Z"
      },
      {
        "price": 0.72,
        "timestamp": "2024-01-15T11:30:00Z"
      }
    ]
  },
  {
    "title": "NO Token",
    "prices": [
      {
        "price": 0.25,
        "timestamp": "2024-01-15T10:30:00Z"
      },
      {
        "price": 0.28,
        "timestamp": "2024-01-15T11:30:00Z"
      }
    ]
  }
]
```

**Usage**:
```python
response = requests.get(
    "https://api.limitless.exchange/markets/btc-100k-2024/historical-price",
    params={
        "from": "2024-01-01T00:00:00Z",
        "to": "2024-01-31T23:59:59Z",
        "interval": "1d"
    }
)
price_history = response.json()
```

**Interval Options**:

| Interval | Description |
|----------|-------------|
| `1h` | 1-hour candles |
| `6h` | 6-hour candles |
| `1d` | Daily candles |
| `1w` | Weekly candles |
| `1m` | Monthly candles |
| `all` | All available data |

---

### GET /markets/{slug}/orderbook

Get the current orderbook showing all open buy and sell orders.

**Authentication**: None required

**Path Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `slug` | string | Market slug identifier |

**Response** (200):
```json
{
  "adjustedMidpoint": 0.75,
  "asks": [
    {
      "price": 0.76,
      "size": "100000000",
      "total": "100000000"
    },
    {
      "price": 0.77,
      "size": "250000000",
      "total": "350000000"
    }
  ],
  "bids": [
    {
      "price": 0.74,
      "size": "150000000",
      "total": "150000000"
    },
    {
      "price": 0.73,
      "size": "200000000",
      "total": "350000000"
    }
  ]
}
```

**Response Fields**:

| Field | Type | Description |
|-------|------|-------------|
| `adjustedMidpoint` | number | Mid-market price |
| `asks` | array | Sell orders (ascending by price) |
| `bids` | array | Buy orders (descending by price) |
| `price` | number | Order price (0.01-0.99) |
| `size` | string | Order size in token decimals |
| `total` | string | Cumulative size at this price level |

**Usage**:
```python
response = requests.get(
    "https://api.limitless.exchange/markets/btc-100k-2024/orderbook"
)
orderbook = response.json()

best_bid = orderbook['bids'][0]['price'] if orderbook['bids'] else None
best_ask = orderbook['asks'][0]['price'] if orderbook['asks'] else None
spread = best_ask - best_bid if best_bid and best_ask else None
```

---

### GET /markets/{slug}/locked-balance

Get the amount of funds locked in open orders for the authenticated user.

**Authentication**: Required (session cookie)

**Path Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `slug` | string | Market slug identifier |

**Response** (200):
```json
{
  "lockedBalance": "50000000"
}
```

**Notes**:
- Balance is in USDC with 6 decimals
- `50000000` = 50 USDC locked

---

### GET /markets/{slug}/user-orders

Get all open orders for the authenticated user in a specific market.

**Authentication**: Required (session cookie)

**Path Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `slug` | string | Market slug identifier |

**Response** (200):
```json
{
  "orders": [
    {
      "orderId": "6f52b6d2-6c9e-4a5c-8a4f-28ab4b7ff203",
      "side": 0,
      "price": 0.65,
      "size": "100000000",
      "filledSize": "25000000",
      "status": "OPEN",
      "createdAt": "2024-01-15T10:30:00Z"
    }
  ]
}
```

**Order Fields**:

| Field | Type | Description |
|-------|------|-------------|
| `orderId` | string | Unique order identifier |
| `side` | number | 0 = BUY, 1 = SELL |
| `price` | number | Order price |
| `size` | string | Total order size |
| `filledSize` | string | Amount already filled |
| `status` | string | Order status (OPEN, FILLED, CANCELLED) |

---

### GET /markets/{slug}/events

Get recent trading events for a market.

**Authentication**: None required

**Path Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `slug` | string | Market slug identifier |

**Query Parameters**:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `limit` | number | No | Max events to return |

**Response** (200):
```json
{
  "events": [
    {
      "type": "trade",
      "price": 0.75,
      "size": "10000000",
      "side": 0,
      "timestamp": "2024-01-15T10:30:00Z",
      "txHash": "0x..."
    }
  ]
}
```

## Price Understanding

### Price Representation
- Prices are decimals between 0.01 and 0.99
- Price represents probability of YES outcome
- YES price + NO price = 1.00 (approximately)

### Example
- YES price: 0.75 (75% probability)
- NO price: 0.25 (25% probability)
- If YES wins, YES holders receive 1.00 per share
- If NO wins, NO holders receive 1.00 per share

### Size/Amount Representation
- All sizes are in USDC with 6 decimals
- `1000000` = 1 USDC
- `100000000` = 100 USDC
- `1000000000000` = 1,000,000 USDC
