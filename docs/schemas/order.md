# Order Data Schemas

Reference documentation for order-related data structures.

## Order

The core order structure used for creating and managing trades.

```typescript
interface Order {
  // Signature Components
  salt: number;              // Random value for uniqueness
  signature: string;         // EIP-712 signature (0x...)
  signatureType: SignatureType;

  // Addresses
  maker: string;             // Order creator address
  signer: string;            // Signing address (usually same as maker)
  taker: string;             // Specific taker or zero address

  // Order Details
  tokenId: string;           // YES or NO position ID
  makerAmount: number;       // Amount maker offers (6 decimals)
  takerAmount: number;       // Amount maker wants (6 decimals)
  side: OrderSide;           // 0 = BUY, 1 = SELL

  // Optional Fields
  expiration: string;        // Unix timestamp or "0" for no expiration
  nonce: number;             // Cancellation nonce
  feeRateBps: number;        // Fee in basis points
  price: number;             // Required for GTC orders (0.01-0.99)
}
```

### SignatureType

| Value | Name | Description |
|-------|------|-------------|
| 0 | `EOA` | Standard Externally Owned Account |
| 1 | `POLY_PROXY` | Polymarket proxy wallet |
| 2 | `POLY_GNOSIS_SAFE` | Gnosis Safe multisig |
| 3 | `EIP1271` | Contract signature (EIP-1271) |

### OrderSide

| Value | Name | Description |
|-------|------|-------------|
| 0 | `BUY` | Buying outcome tokens |
| 1 | `SELL` | Selling outcome tokens |

---

## CreateOrderDto

Request body for `POST /orders`.

```typescript
interface CreateOrderDto {
  order: Order;              // Signed order data
  ownerId: number;           // User's profile ID
  orderType: OrderType;      // GTC or FOK
  marketSlug: string;        // Target market slug
}
```

### OrderType

| Type | Name | Description |
|------|------|-------------|
| `GTC` | Good Till Cancelled | Stays open until filled or cancelled |
| `FOK` | Fill Or Kill | Must fill completely or fails |

**FOK Order Semantics**: For FOK market orders, set `takerAmount: '1'` to indicate market order semantics where the amount is derived from `makerAmount`.

---

## OrderResponseDto

Response from successful order creation.

```typescript
interface OrderResponseDto {
  order: {
    orderId: string;         // UUID for the order
    salt: number;
    maker: string;
    price: number;
    side: OrderSide;
    status: OrderStatus;
    createdAt: string;
  };
  makerMatches: MakerMatch[]; // Immediate matches (if any)
}
```

### OrderStatus

| Status | Description |
|--------|-------------|
| `OPEN` | Order is active and waiting for fills |
| `FILLED` | Order completely filled |
| `PARTIALLY_FILLED` | Order partially filled |
| `CANCELLED` | Order cancelled by user |
| `EXPIRED` | Order expired |

---

## MakerMatch

Represents a match with an existing maker order.

```typescript
interface MakerMatch {
  makerOrderId: string;      // Matched order ID
  fillAmount: string;        // Amount filled
  price: number;             // Fill price
  timestamp: string;         // Match timestamp
}
```

---

## CancelOrderResponseDto

Response from order cancellation.

```typescript
interface CancelOrderResponseDto {
  message: string;           // "Order canceled successfully"
}
```

---

## DeleteOrderBatchDto

Request body for batch cancellation.

```typescript
interface DeleteOrderBatchDto {
  orderIds: string[];        // Array of order UUIDs
}
```

---

## CancelOrderBatchResponseDto

Response from batch cancellation.

```typescript
interface CancelOrderBatchResponseDto {
  message: string;
  canceled: string[];        // Successfully cancelled order IDs
  failed: CancelOrderFailure[];
}

interface CancelOrderFailure {
  orderId: string;
  reason: 'ORDER_NOT_FOUND' | 'UNKNOWN_ERROR';
  message: string;           // Human-readable error
}
```

---

## CancelAllOrdersResponseDto

Response from cancelling all orders in a market.

```typescript
interface CancelAllOrdersResponseDto {
  message: string;
  canceled: string[];
  failed: CancelOrderFailure[];
}
```

---

## ErrorResponseDto

Standard error response structure.

```typescript
interface ErrorResponseDto {
  message: string;           // Error description
}
```

---

## EIP-712 Order Structure

For signing orders, use this EIP-712 typed data structure:

```typescript
// Get verifyingContract from market's venue.exchange
// See docs/contracts.md for all contract addresses
const domain = {
  name: 'Limitless CTF Exchange',
  version: '1',
  chainId: 8453,             // Base mainnet
  verifyingContract: market.venue.exchange  // e.g., '0xa4409D988CA2218d956BeEFD3874100F444f0DC3' for simple v1
};

const types = {
  Order: [
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
  ]
};
```

---

## Order Examples

### Buy 100 YES tokens at 65 cents

```json
{
  "order": {
    "salt": 1704931200000,
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
    "signature": "0x...",
    "signatureType": 0,
    "price": 0.65
  },
  "ownerId": 12345,
  "orderType": "GTC",
  "marketSlug": "btc-100k-2024"
}
```

### Sell 50 NO tokens at 30 cents

```json
{
  "order": {
    "salt": 1704931300000,
    "maker": "0x742d35Cc6634C0532925a3b844Bc454e4438f44e",
    "signer": "0x742d35Cc6634C0532925a3b844Bc454e4438f44e",
    "taker": "0x0000000000000000000000000000000000000000",
    "tokenId": "59488366814164541111277457205683376934541954961614829576144806470851104638710",
    "makerAmount": 50000000,
    "takerAmount": 15000000,
    "expiration": "0",
    "nonce": 0,
    "feeRateBps": 300,
    "side": 1,
    "signature": "0x...",
    "signatureType": 0,
    "price": 0.30
  },
  "ownerId": 12345,
  "orderType": "GTC",
  "marketSlug": "btc-100k-2024"
}
```

---

## Price Calculation

### For BUY orders
```
price = makerAmount / takerAmount
```
- `makerAmount`: USDC you're paying
- `takerAmount`: Tokens you're receiving

### For SELL orders
```
price = takerAmount / makerAmount
```
- `makerAmount`: Tokens you're selling
- `takerAmount`: USDC you're receiving

### Example
Buying 100 YES tokens at $0.65:
- `makerAmount = 65 * 1e6 = 65000000` (65 USDC)
- `takerAmount = 100 * 1e6 = 100000000` (100 tokens)
- `price = 65000000 / 100000000 = 0.65`
