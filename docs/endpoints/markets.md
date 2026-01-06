# Markets Endpoints

Endpoints for browsing, searching, and retrieving prediction market data.

## Endpoints

### GET /markets/active

Browse all active (unresolved) markets with pagination.

**Authentication**: None required

**Query Parameters**:

| Parameter | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| `page` | number | No | Page number | `1` |
| `limit` | number | No | Items per page | `10` |
| `sortBy` | string | No | Sort order | `newest`, `volume`, `liquidity` |

**Response** (200):
```json
{
  "markets": [...],
  "groups": [...],
  "totalCount": 150,
  "page": 1,
  "limit": 10
}
```

**Usage**:
```python
response = requests.get(
    "https://api.limitless.exchange/markets/active",
    params={"page": 1, "limit": 10, "sortBy": "volume"}
)
markets = response.json()
```

---

### GET /markets/active/{categoryId}

Browse active markets filtered by category.

**Authentication**: None required

**Path Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `categoryId` | number | Category ID to filter by |

**Query Parameters**: Same as `/markets/active`

**Response** (200): Same structure as `/markets/active`

---

### GET /markets/categories/count

Get the count of active markets per category.

**Authentication**: None required

**Response** (200):
```json
{
  "categories": {
    "1": 25,
    "2": 15,
    "3": 42
  },
  "total": 82
}
```

---

### GET /markets/active/slugs

Get slugs and metadata for all active markets. Useful for building market indexes.

**Authentication**: None required

**Response** (200):
```json
[
  {
    "slug": "btc-price-prediction-2024",
    "strikePrice": "50000",
    "ticker": "BTC",
    "deadline": "2024-12-31T23:59:59Z"
  },
  {
    "slug": "crypto-predictions-2024",
    "strikePrice": null,
    "ticker": "USDC",
    "deadline": "2024-12-31T23:59:59Z",
    "markets": [
      {"slug": "btc-50k-prediction"},
      {"slug": "eth-5k-prediction"}
    ]
  }
]
```

**Notes**:
- Individual markets have `strikePrice` and optional `ticker`
- Group markets (NegRisk) have a nested `markets` array
- Groups do not have `strikePrice`

---

### GET /markets/{addressOrSlug}

Get detailed information about a specific market or group.

**Authentication**: None required

**Path Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `addressOrSlug` | string | Market address (`0x...`) or slug identifier |

**Response** (200):

Returns one of three response types based on market type:

#### CLOB Market Response
```json
{
  "type": "single-clob",
  "address": "0x...",
  "title": "Will BTC reach $100k?",
  "description": "Market description...",
  "slug": "btc-100k-2024",
  "status": "FUNDED",
  "deadline": "2024-12-31T23:59:59Z",
  "positionIds": ["123...", "456..."],
  "condition_id": "0x...",
  "outcome_slot_count": 2,
  "winning_index": null,
  "volume": "150000000000",
  "liquidity": "50000000000",
  "venue": {
    "exchange": "0xA1b2C3...",
    "adapter": "0xD4e5F6..."
  }
}
```

**Important**: The `venue` object contains contract addresses needed for order signing:
- `venue.exchange`: Use as `verifyingContract` in EIP-712 signing
- `venue.adapter`: Needed for SELL orders on NegRisk markets (approve CT to both)

Cache venue data per market - it's static and doesn't change.

#### NegRisk Group Response
```json
{
  "type": "group-negrisk",
  "slug": "election-predictions",
  "title": "2024 Election Markets",
  "markets": [
    {
      "slug": "candidate-a-wins",
      "title": "Candidate A wins",
      "positionIds": ["...", "..."]
    }
  ],
  "negRiskMarketId": "0x...",
  "venue": {
    "exchange": "0x...",
    "adapter": "0x..."
  }
}
```

#### AMM Market Response
```json
{
  "type": "amm",
  "address": "0x...",
  "title": "Market title",
  "liquidity": "100000000000",
  "volume": "500000000000"
}
```

**Error Responses**:
- `404`: Market or group not found
- `500`: Server error

---

### GET /markets/{slug}/get-feed-events

Get recent activity/events for a market.

**Authentication**: Bearer token required

**Path Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `slug` | string | Market slug |

**Query Parameters**:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `page` | number | No | Page number |
| `limit` | number | No | Events per page |

**Response** (200):
```json
{
  "events": [
    {
      "type": "trade",
      "timestamp": "2024-01-15T10:30:00Z",
      "data": {...}
    }
  ],
  "totalCount": 100
}
```

---

### GET /markets/search

Search markets by semantic similarity.

**Authentication**: None required

**Query Parameters**:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `q` | string | Yes | Search query |
| `limit` | number | No | Max results |

**Response** (200):
```json
{
  "results": [
    {
      "slug": "btc-price-prediction",
      "title": "Will Bitcoin reach...",
      "score": 0.95
    }
  ]
}
```

## Market Data Fields

### Common Fields

| Field | Type | Description |
|-------|------|-------------|
| `address` | string | Ethereum contract address |
| `title` | string | Market question/title |
| `slug` | string | URL-friendly identifier |
| `description` | string | Detailed market description |
| `status` | string | Market status (FUNDED, RESOLVED, etc.) |
| `deadline` | string | Resolution deadline (ISO 8601) |
| `condition_id` | string | Bytes32 condition identifier |
| `positionIds` | array | YES/NO position token IDs (`[0]` = YES, `[1]` = NO) |
| `outcome_slot_count` | number | Number of outcomes (usually 2) |
| `winning_index` | number | Winning outcome (0 or 1) after resolution |
| `venue` | object | Contract addresses for trading (CLOB markets) |
| `venue.exchange` | string | Exchange contract (use as verifyingContract) |
| `venue.adapter` | string | Adapter contract (for NegRisk SELL approvals) |

### Volume/Liquidity

All monetary values are in USDC with 6 decimals:
- `1000000` = 1 USDC
- `100000000` = 100 USDC
