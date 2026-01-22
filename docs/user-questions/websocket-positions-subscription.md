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

### Example API Response

```json
{
  "id": 7495,
  "address": "0x76d3e2098Be66Aa7E15138F467390f0Eb7349B9b",  // ✅ THIS is marketAddresses
  "conditionId": "0x812f578...",                            // ❌ NOT this
  "title": "$DOGE above $0.21652...",
  "venue": {
    "exchange": "0xABC123...",                              // ❌ NOT this (used for order signing)
    "adapter": "0xDEF456..."
  },
  "collateralToken": {
    "address": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913"  // ❌ NOT this (USDC token)
  }
}
```

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
