# WebSocket Positions Subscription

**Question**: I want to subscribe to `positions` websocket (TypeScript). Do I use `websocket.emit("subscribe_positions")`? Do I need to provide other params?

## Answer

**Yes**, you use `socket.emit('subscribe_positions', ...)` but you **must provide the `marketAddresses` parameter** as an array of market contract addresses. Authentication is also required.

## What is `marketAddresses`?

The `marketAddresses` is the **market's own contract address** - specifically the `address` field returned from the market API endpoint.

**It is NOT:**
- ❌ YES/NO token addresses
- ❌ `venue.exchange` address (used for order signing)
- ❌ `conditionId`
- ❌ `collateralToken.address` (USDC token)

**It IS:**
- ✅ The `address` field from `GET /markets/{slug}` response

### How to Get the Market Address

```typescript
// Step 1: Fetch the market data
const response = await fetch('https://api.limitless.exchange/markets/your-market-slug');
const market = await response.json();

// Step 2: Use the 'address' field (NOT venue.exchange, NOT token addresses)
const marketAddress = market.address;  // e.g., "0x76d3e2098Be66Aa7E15138F467390f0Eb7349B9b"

// Step 3: Subscribe with the market address
socket.emit('subscribe_positions', {
  marketAddresses: [marketAddress]
});
```

### Full API Response Example (`GET /markets/{slug}`)

Here is a **complete** response from the markets API showing all fields. The `address` field (highlighted) is what you need for `marketAddresses`:

```json
{
  "type": "single-clob",
  "id": 7495,

  "address": "0x76d3e2098Be66Aa7E15138F467390f0Eb7349B9b",  // ✅ USE THIS for marketAddresses

  "title": "Will BTC reach $100k by end of 2024?",
  "slug": "btc-100k-2024",
  "description": "This market resolves YES if...",
  "status": "FUNDED",
  "deadline": "2024-12-31T23:59:59Z",

  "condition_id": "0x812f578a9b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e",  // ❌ NOT this
  "positionIds": [
    "19633204485790857949828516737993423758628930235371629943999544859324645414627",  // ❌ NOT this (YES token ID)
    "59488366814164541111277457205683376934541954961614829576144806470851104638710"   // ❌ NOT this (NO token ID)
  ],
  "outcome_slot_count": 2,
  "winning_index": null,

  "venue": {
    "exchange": "0xA1b2C3d4E5f6A1b2C3d4E5f6A1b2C3d4E5f6A1b2",  // ❌ NOT this (for EIP-712 signing)
    "adapter": "0xD4e5F6a7B8c9D4e5F6a7B8c9D4e5F6a7B8c9D4e5"    // ❌ NOT this (for NegRisk approvals)
  },

  "collateralToken": {
    "address": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",  // ❌ NOT this (USDC token address)
    "symbol": "USDC",
    "decimals": 6
  },

  "volume": "150000000000",
  "liquidity": "50000000000"
}
```

**Summary of fields that are NOT `marketAddresses`:**

| Field | What It Is | Why It's Not the Right One |
|-------|------------|---------------------------|
| `condition_id` | Gnosis conditional token ID | Used for CTF operations, not WebSocket |
| `positionIds[0]` | YES outcome token ID | Used for order placement |
| `positionIds[1]` | NO outcome token ID | Used for order placement |
| `venue.exchange` | Exchange contract | Used for EIP-712 order signing |
| `venue.adapter` | Adapter contract | Used for NegRisk sell approvals |
| `collateralToken.address` | USDC contract | The payment token address |

**The only correct field is `address`** - the market's own contract address at the top level of the response.

## Required Parameters

```typescript
socket.emit('subscribe_positions', {
  marketAddresses: string[]  // Array of market 'address' fields from API (required)
});
```

## Key Requirements

| Requirement | Details |
|-------------|---------|
| **Event Name** | `subscribe_positions` |
| **Namespace** | `/markets` |
| **Auth Required** | Yes (session cookie required) |
| **Required Params** | `marketAddresses: string[]` |

## TypeScript Example

```typescript
import { io, Socket } from 'socket.io-client';

// Connect with authentication
const socket: Socket = io('wss://ws.limitless.exchange/markets', {
  transports: ['websocket'],
  extraHeaders: {
    'Cookie': `limitless_session=${sessionCookie}`
  }
});

// Subscribe to positions for specific markets
socket.emit('subscribe_positions', {
  marketAddresses: ['0xE082AF5a25f5D3904fae514CD03dC99F9Ff39fBc']
});

// Listen for position updates
socket.on('positions', (data) => {
  console.log('Account:', data.account);
  console.log('Market:', data.marketAddress);
  console.log('Positions:', data.positions);
  console.log('Type:', data.type);
});
```

## Response Data Structure

When you receive a `positions` event, the data has this structure:

```typescript
interface PositionsData {
  account: string;        // User's wallet address
  marketAddress: string;  // The market contract address
  positions: Array<{
    tokenId: string;      // Position token identifier
    balance: string;      // Position balance amount
    outcomeIndex: number; // Index of the outcome (0 = YES, 1 = NO)
  }>;
  type: string;           // Position type (e.g., "AMM")
}
```

**Example response**:
```json
{
  "account": "0xabcd...",
  "marketAddress": "0x1234...",
  "positions": [
    {
      "tokenId": "123456",
      "balance": "1000000",
      "outcomeIndex": 0
    }
  ],
  "type": "AMM"
}
```

## Complete Example with Authentication

```typescript
import { io, Socket } from 'socket.io-client';

class PositionsSubscriber {
  private socket: Socket;
  private sessionCookie: string;
  private subscribedMarkets: string[] = [];

  constructor(sessionCookie: string) {
    this.sessionCookie = sessionCookie;
    this.socket = io('wss://ws.limitless.exchange/markets', {
      transports: ['websocket'],
      extraHeaders: {
        'Cookie': `limitless_session=${sessionCookie}`
      }
    });

    this.setupEventHandlers();
  }

  private setupEventHandlers(): void {
    this.socket.on('connect', () => {
      console.log('Connected to WebSocket');

      // Re-subscribe on reconnect
      if (this.subscribedMarkets.length > 0) {
        this.subscribe(this.subscribedMarkets);
      }
    });

    this.socket.on('positions', (data) => {
      console.log(`Position update for ${data.account}:`);
      for (const pos of data.positions) {
        const outcome = pos.outcomeIndex === 0 ? 'YES' : 'NO';
        console.log(`  ${outcome}: ${pos.balance}`);
      }
    });

    this.socket.on('disconnect', () => {
      console.log('Disconnected from WebSocket');
    });
  }

  subscribe(marketAddresses: string[]): void {
    this.subscribedMarkets = marketAddresses;
    this.socket.emit('subscribe_positions', { marketAddresses });
    console.log(`Subscribed to ${marketAddresses.length} market(s)`);
  }
}

// Usage
const subscriber = new PositionsSubscriber('your_session_cookie');
subscriber.subscribe(['0xE082AF5a25f5D3904fae514CD03dC99F9Ff39fBc']);
```

## Important Notes

1. **Authentication is required** - You must include your session cookie in the connection headers. Without authentication, you won't receive position updates.

2. **New subscriptions replace old ones** - If you call `subscribe_positions` again, the new subscription replaces the previous one. To subscribe to multiple markets, include all addresses in a single call:
   ```typescript
   // Correct: subscribe to multiple markets at once
   socket.emit('subscribe_positions', {
     marketAddresses: ['0xMarket1...', '0xMarket2...', '0xMarket3...']
   });
   ```

3. **Re-subscribe on reconnect** - If the WebSocket disconnects and reconnects, you need to re-subscribe. Track your subscribed markets and re-emit on the `connect` event.

## Related

- [WebSocket Guide](../guides/websockets.md) - Complete WebSocket documentation
- [Authentication Guide](../guides/authentication.md) - How to get a session cookie
- [TypeScript Quickstart](../quickstart/typescript.md) - Getting started with TypeScript
