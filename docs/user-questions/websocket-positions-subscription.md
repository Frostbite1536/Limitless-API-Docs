# WebSocket Positions Subscription

**Question**: I want to subscribe to `positions` websocket (TypeScript). Do I use `websocket.emit("subscribe_positions")`? Do I need to provide other params?

## Answer

**Yes**, you use `socket.emit('subscribe_positions', ...)` but you **must provide the `marketAddresses` parameter** as an array of market contract addresses. Authentication is also required.

## Important: AMM vs CLOB Markets

The `subscribe_positions` WebSocket event works differently depending on market type:

| Market Type | Has `address` field? | WebSocket Positions | How to Get Positions |
|-------------|---------------------|---------------------|---------------------|
| **AMM** (`type: "amm"`) | ✅ Yes | ✅ Works | Use `subscribe_positions` with `marketAddresses` |
| **CLOB** (`tradeType: "clob"`) | ❌ No | ⚠️ Limited | Use REST API `GET /portfolio/positions` |

### CLOB Markets (like hourly BTC markets)

**CLOB markets do NOT have an `address` field in their API response.** If your market response looks like this (no `address` field):

```json
{
  "id": 37715,
  "conditionId": "0x66485e0f49d01c71cf59262696860027c9616220c3b5228dc43d66c32f38b489",
  "slug": "dollarbtc-above-dollar8969874-on-jan-22-2100-utc-1769112006582",
  "tradeType": "clob",
  "venue": {
    "exchange": "0x05c748E2f4DcDe0ec9Fa8DDc40DE6b867f923fa5",
    "adapter": null
  },
  "tokens": {
    "yes": "107451767198812600366367451369309385652461274328912255994662576962885362377488",
    "no": "9406988567878049320164873863910186821044547671080951632674039568841521651984"
  },
  "collateralToken": {
    "address": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913"
  }
  // ... NO "address" field at top level!
}
```

#### What happens if you try `venue.exchange` for CLOB markets?

If you try using `venue.exchange` as the `marketAddress` for a CLOB market, you'll get a response but with **`positions: null`**:

```json
{
  "account": "0xYourWalletAddress...",
  "marketAddress": "0x05c748E2f4DcDe0ec9Fa8DDc40DE6b867f923fa5",
  "positions": null,   // ❌ No position data!
  "type": "AMM"        // Still says AMM - this event is designed for AMM markets
}
```

The `type: "AMM"` in the response indicates that `subscribe_positions` is designed for AMM markets, not CLOB markets. Even though the WebSocket accepts the subscription, it won't return actual position data for CLOB markets.

**For CLOB positions, use the REST API instead:**

```typescript
// For CLOB markets - use REST API
const response = await fetch('https://api.limitless.exchange/portfolio/positions', {
  headers: {
    'Cookie': `limitless_session=${sessionCookie}`
  }
});
const positions = await response.json();

// Access CLOB positions
for (const pos of positions.clob) {
  console.log(`Market: ${pos.market.title}`);
  console.log(`YES tokens: ${parseInt(pos.tokensBalance.yes) / 1e6}`);
  console.log(`NO tokens: ${parseInt(pos.tokensBalance.no) / 1e6}`);
}
```

### AMM Markets

**AMM markets have an `address` field** and work with `subscribe_positions`:

```json
{
  "type": "amm",
  "address": "0x76d3e2098Be66Aa7E15138F467390f0Eb7349B9b",  // ✅ USE THIS
  "title": "Will BTC reach $100k?",
  "slug": "btc-100k-2024",
  "liquidity": "100000000000",
  "volume": "500000000000"
}
```

## What is `marketAddresses`? (AMM Markets Only)

The `marketAddresses` is the **market's own contract address** - specifically the `address` field returned from the market API endpoint for **AMM markets**.

**It is NOT:**
- ❌ `tokens.yes` / `tokens.no` (position token IDs)
- ❌ `venue.exchange` address (used for order signing)
- ❌ `conditionId`
- ❌ `collateralToken.address` (USDC token)

**It IS:**
- ✅ The `address` field from `GET /markets/{slug}` response (AMM markets only)

### How to Get the Market Address (AMM Markets)

```typescript
// Step 1: Fetch the market data
const response = await fetch('https://api.limitless.exchange/markets/your-market-slug');
const market = await response.json();

// Step 2: Check if it's an AMM market with an address field
if (market.type === 'amm' && market.address) {
  // AMM market - can use WebSocket positions subscription
  socket.emit('subscribe_positions', {
    marketAddresses: [market.address]
  });
} else {
  // CLOB market - use REST API for positions instead
  console.log('CLOB market detected - use GET /portfolio/positions for positions');
}
```

### Fields That Are NOT `marketAddresses`

| Field | What It Is | Why It's Not the Right One |
|-------|------------|---------------------------|
| `conditionId` | Gnosis conditional token ID | Used for CTF operations, not WebSocket |
| `tokens.yes` | YES outcome token ID | Used for order placement |
| `tokens.no` | NO outcome token ID | Used for order placement |
| `venue.exchange` | Exchange contract | Used for EIP-712 order signing |
| `venue.adapter` | Adapter contract | Used for NegRisk sell approvals |
| `collateralToken.address` | USDC contract | The payment token address |

## WebSocket Positions (AMM Markets)

### Required Parameters

```typescript
socket.emit('subscribe_positions', {
  marketAddresses: string[]  // Array of AMM market 'address' fields (required)
});
```

### Key Requirements

| Requirement | Details |
|-------------|---------|
| **Event Name** | `subscribe_positions` |
| **Namespace** | `/markets` |
| **Auth Required** | Yes (session cookie required) |
| **Required Params** | `marketAddresses: string[]` |
| **Market Type** | AMM markets only (markets with `type: "amm"`) |

### TypeScript Example (AMM Markets)

```typescript
import { io, Socket } from 'socket.io-client';

// Connect with authentication
const socket: Socket = io('wss://ws.limitless.exchange/markets', {
  transports: ['websocket'],
  extraHeaders: {
    'Cookie': `limitless_session=${sessionCookie}`
  }
});

// Subscribe to positions for AMM markets (must have 'address' field)
socket.emit('subscribe_positions', {
  marketAddresses: ['0xE082AF5a25f5D3904fae514CD03dC99F9Ff39fBc']
});

// Listen for position updates
socket.on('positions', (data) => {
  console.log('Account:', data.account);
  console.log('Market:', data.marketAddress);
  console.log('Positions:', data.positions);
  console.log('Type:', data.type);  // Will be "AMM"
});
```

### Response Data Structure (AMM)

```typescript
interface PositionsData {
  account: string;        // User's wallet address
  marketAddress: string;  // The market contract address
  positions: Array<{
    tokenId: string;      // Position token identifier
    balance: string;      // Position balance amount
    outcomeIndex: number; // Index of the outcome (0 = YES, 1 = NO)
  }>;
  type: "AMM";            // Position type
}
```

---

## REST API Positions (All Markets, Including CLOB)

For **CLOB markets** (like hourly BTC/ETH markets), use the REST API endpoint:

### GET /portfolio/positions

```typescript
// Fetch all positions via REST API
async function getPositions(sessionCookie: string) {
  const response = await fetch('https://api.limitless.exchange/portfolio/positions', {
    headers: {
      'Cookie': `limitless_session=${sessionCookie}`
    }
  });
  return response.json();
}

// Usage
const positions = await getPositions(sessionCookie);

// Access CLOB positions
console.log('CLOB Positions:');
for (const pos of positions.clob) {
  console.log(`  Market: ${pos.market.title}`);
  console.log(`  Slug: ${pos.market.slug}`);
  console.log(`  YES tokens: ${parseInt(pos.tokensBalance.yes) / 1e6}`);
  console.log(`  NO tokens: ${parseInt(pos.tokensBalance.no) / 1e6}`);
  console.log(`  YES market value: $${parseInt(pos.positions.yes.marketValue) / 1e6}`);
  console.log(`  Unrealized P&L: $${parseInt(pos.positions.yes.unrealizedPnl) / 1e6}`);
}

// Access AMM positions
console.log('AMM Positions:');
for (const pos of positions.amm) {
  console.log(`  Market: ${pos.market.title}`);
}
```

### Response Structure

```typescript
interface PortfolioPositions {
  clob: ClobPosition[];
  amm: AmmPosition[];
  rewards: RewardInfo[];
}

interface ClobPosition {
  market: {
    title: string;
    slug: string;
    status: string;
  };
  tokensBalance: {
    yes: string;  // Raw balance (divide by 1e6 for display)
    no: string;
  };
  positions: {
    yes: {
      marketValue: string;
      unrealizedPnl: string;
    };
    no: {
      marketValue: string;
      unrealizedPnl: string;
    };
  };
}
```

---

## Summary: Which Method to Use

| Your Market Type | API Response | Method to Use |
|-----------------|--------------|---------------|
| AMM market | Has `type: "amm"` and `address` field | WebSocket `subscribe_positions` |
| CLOB market | Has `tradeType: "clob"`, NO `address` field | REST API `GET /portfolio/positions` |
| Any market | - | REST API works for all market types |

**Recommendation**: If you're working with CLOB markets (hourly markets, etc.), use the REST API endpoint `GET /portfolio/positions` to fetch your positions. Poll periodically if you need updates.

## Important Notes

1. **Authentication is required** for both methods - Include your session cookie.

2. **CLOB markets don't have `address` field** - The `subscribe_positions` WebSocket requires `marketAddresses`, which only AMM markets provide.

3. **New subscriptions replace old ones** - If using WebSocket, send all market addresses in a single call.

4. **Re-subscribe on reconnect** - Track subscribed markets and re-emit on the `connect` event.

## Related

- [WebSocket Guide](../guides/websockets.md) - Complete WebSocket documentation
- [Portfolio Endpoints](../endpoints/portfolio.md) - REST API for positions
- [Authentication Guide](../guides/authentication.md) - How to get a session cookie
- [TypeScript Quickstart](../quickstart/typescript.md) - Getting started with TypeScript
