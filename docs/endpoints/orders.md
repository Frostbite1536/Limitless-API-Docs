# Orders Endpoints

Endpoints for creating and managing buy/sell orders on CLOB markets.

## Prerequisites

Before placing orders:

1. **Authentication**: Must be logged in with session cookie
2. **USDC Approval**: Approve USDC spending for the CTF Exchange contract
3. **Conditional Token Approval**: For selling, approve conditional tokens

### Contract Addresses (Base Mainnet)

| Contract | Address |
|----------|---------|
| CLOB CTF Exchange | `0xa4409D988CA2218d956BeEFD3874100F444f0DC3` |
| NegRisk CTF Exchange | `0x5a38afc17F7E97ad8d6C547ddb837E40B4aEDfC6` |

## Endpoints

### POST /orders

Create a new signed buy or sell order.

**Authentication**: Required (session cookie)

**Request Body**:
```json
{
  "order": {
    "salt": 1234567890,
    "maker": "0x742d35Cc6634C0532925a3b844Bc454e4438f44e",
    "signer": "0x742d35Cc6634C0532925a3b844Bc454e4438f44e",
    "taker": "0x0000000000000000000000000000000000000000",
    "tokenId": "19633204485790857949828516737993423758628930235371629943999544859324645414627",
    "makerAmount": 65000000,
    "takerAmount": 100000000,
    "expiration": "0",
    "nonce": 0,
    "feeRateBps": 300,
    "side": 0,
    "signature": "0x123abc...",
    "signatureType": 0,
    "price": 0.65
  },
  "ownerId": 12345,
  "orderType": "GTC",
  "marketSlug": "btc-100k-2024"
}
```

**Order Fields**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `salt` | number | Yes | Random value for signature uniqueness |
| `maker` | string | Yes | Your Ethereum address |
| `signer` | string | Yes | Address that signed (usually same as maker) |
| `taker` | string | No | Specific taker address (use zero address for open orders) |
| `tokenId` | string | Yes | YES or NO position ID from market data |
| `makerAmount` | number | Yes | Amount you're offering (USDC, 6 decimals) |
| `takerAmount` | number | Yes | Amount you want in return (shares, 6 decimals) |
| `expiration` | string | No | Expiration timestamp (0 = no expiration) |
| `nonce` | number | No | Order nonce for cancellation |
| `feeRateBps` | number | Yes | Fee rate in basis points (100 = 1%) |
| `side` | number | Yes | 0 = BUY, 1 = SELL |
| `signature` | string | Yes | EIP-712 signature |
| `signatureType` | number | Yes | 0 = EOA, 1-3 = other types |
| `price` | number | GTC only | Price for GTC orders (0.01-0.99) |

**Order Types**:

| Type | Description |
|------|-------------|
| `GTC` | Good Till Cancelled - stays open until filled or cancelled |
| `FOK` | Fill Or Kill - must fill completely or fails |

**Response** (200):
```json
{
  "order": {
    "orderId": "6f52b6d2-6c9e-4a5c-8a4f-28ab4b7ff203",
    "salt": 1234567890,
    "maker": "0x742d35Cc6634C0532925a3b844Bc454e4438f44e",
    "price": 0.65,
    "side": 0,
    "status": "OPEN"
  },
  "makerMatches": []
}
```

**Error Responses**:
- `400`: Invalid order data
- `401`: Not authenticated
- `500`: Server error

---

### DELETE /orders/{orderId}

Cancel a specific open order.

**Authentication**: Required (session cookie)

**Path Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `orderId` | string | UUID of the order to cancel |

**Response** (200):
```json
{
  "message": "Order canceled successfully"
}
```

**Error Responses**:
- `404`: Order not found or already cancelled
- `401`: Not authenticated or not order owner

---

### POST /orders/cancel-batch

Cancel multiple orders in a single request.

**Authentication**: Required (session cookie)

**Request Body**:
```json
{
  "orderIds": [
    "6f52b6d2-6c9e-4a5c-8a4f-28ab4b7ff203",
    "9e31c452-8a2b-42d1-b327-65f18d07dc96"
  ]
}
```

**Response** (200):
```json
{
  "message": "Orders canceled successfully",
  "canceled": [
    "6f52b6d2-6c9e-4a5c-8a4f-28ab4b7ff203"
  ],
  "failed": [
    {
      "orderId": "9e31c452-8a2b-42d1-b327-65f18d07dc96",
      "reason": "ORDER_NOT_FOUND",
      "message": "Order not found or already canceled"
    }
  ]
}
```

**Failure Reasons**:

| Reason | Description |
|--------|-------------|
| `ORDER_NOT_FOUND` | Order doesn't exist or already cancelled |
| `UNKNOWN_ERROR` | Unexpected error during cancellation |

---

### DELETE /orders/all/{slug}

Cancel all open orders for a specific market.

**Authentication**: Required (session cookie)

**Path Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `slug` | string | Market slug identifier |

**Response** (200):
```json
{
  "message": "Orders canceled successfully",
  "canceled": [
    "6f52b6d2-6c9e-4a5c-8a4f-28ab4b7ff203",
    "9e31c452-8a2b-42d1-b327-65f18d07dc96"
  ],
  "failed": []
}
```

## EIP-712 Order Signing

Orders must be signed using EIP-712 typed data signing.

### Domain

```javascript
{
  name: 'Limitless CTF Exchange',
  version: '1',
  chainId: 8453,  // Base mainnet
  verifyingContract: '0xa4409D988CA2218d956BeEFD3874100F444f0DC3'  // or NegRisk address
}
```

### Order Type Structure

```javascript
const ORDER_STRUCTURE = [
  { name: 'salt', type: 'uint256' },
  { name: 'maker', type: 'address' },
  { name: 'signer', type: 'address' },
  { name: 'taker', type: 'address' },
  { name: 'tokenId', type: 'uint256' },
  { name: 'makerAmount', type: 'uint256' },
  { name: 'takerAmount', type: 'uint256' },
  { name: 'expiration', type: 'uint256' },
  { name: 'nonce', type: 'uint256' },
  { name: 'feeRateBps', type: 'uint256' },
  { name: 'side', type: 'uint8' },
  { name: 'signatureType', type: 'uint8' }
];
```

### Signature Types

| Value | Type | Description |
|-------|------|-------------|
| 0 | EOA | Externally Owned Account signature |
| 1 | POLY_PROXY | Polymarket proxy signature |
| 2 | POLY_GNOSIS_SAFE | Gnosis Safe signature |
| 3 | EIP1271 | Contract signature (EIP-1271) |

## Amount Calculations

### Buying YES tokens at 65 cents

```python
price_cents = 65
shares = 100
scaling_factor = 1_000_000  # USDC has 6 decimals

price_dollars = price_cents / 100  # 0.65
total_cost = price_dollars * shares  # 65 USDC

maker_amount = int(total_cost * scaling_factor)  # 65,000,000
taker_amount = int(shares * scaling_factor)  # 100,000,000
```

### Key Points

- `makerAmount`: What you're paying (USDC)
- `takerAmount`: What you're receiving (outcome tokens)
- For BUY orders: `price = makerAmount / takerAmount`
- All amounts use 6 decimal places (USDC standard)
