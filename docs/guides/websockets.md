# WebSocket Integration Guide

Real-time market data and position updates using Socket.io WebSocket connections.

## Connection Details

| Environment | URL |
|-------------|-----|
| Production | `wss://ws.limitless.exchange` |
| Namespace | `/markets` |

## Quick Start

### JavaScript/TypeScript

```typescript
import { io } from 'socket.io-client';

const socket = io('wss://ws.limitless.exchange/markets', {
  transports: ['websocket']
});

socket.on('connect', () => {
  console.log('Connected to Limitless WebSocket');
});
```

### Python

```python
import socketio

sio = socketio.Client()

@sio.event
def connect():
    print('Connected to Limitless WebSocket')

sio.connect('wss://ws.limitless.exchange', namespaces=['/markets'])
```

## Subscribing to Data

### Subscribe to Market Prices

Subscribe to real-time price updates for specific markets.

```typescript
// Subscribe to AMM prices (by address)
socket.emit('subscribe_market_prices', {
  marketAddresses: ['0xE082AF5a25f5D3904fae514CD03dC99F9Ff39fBc']
});

// Subscribe to CLOB orderbook (by slug)
socket.emit('subscribe_market_prices', {
  marketSlugs: ['btc-100k-2024']
});

// Subscribe to both (combined subscription)
socket.emit('subscribe_market_prices', {
  marketAddresses: ['0xE082AF5a25f5D3904fae514CD03dC99F9Ff39fBc'],
  marketSlugs: ['btc-100k-2024']
});
```

**Important**: Subscriptions replace previous ones. To subscribe to both AMM and CLOB data, send both arrays in a single call.

## Event Handlers

### AMM Price Updates

Receive real-time AMM market prices.

```typescript
socket.on('newPriceData', (data) => {
  console.log('AMM price update:', data);
  // data shape:
  // {
  //   marketAddress: '0x...',
  //   updatedPrices: { yes: 0.75, no: 0.25 },
  //   blockNumber: 123456,
  //   timestamp: '2024-01-15T10:30:00Z'
  // }
});
```

### CLOB Orderbook Updates

Receive real-time orderbook changes.

```typescript
socket.on('orderbookUpdate', (data) => {
  console.log('Orderbook update:', data);
  // data shape:
  // {
  //   marketSlug: 'btc-100k-2024',
  //   orderbook: {
  //     bids: [{ price: 0.74, size: '100000000' }],
  //     asks: [{ price: 0.76, size: '150000000' }]
  //   },
  //   timestamp: '2024-01-15T10:30:00Z'
  // }
});
```

## Authenticated Connections

For private data streams (if/when available), include session cookie:

### JavaScript/TypeScript

```typescript
const socket = io('wss://ws.limitless.exchange/markets', {
  transports: ['websocket'],
  extraHeaders: {
    Cookie: `limitless_session=${sessionCookie}`
  }
});
```

### Python

```python
sio = socketio.Client()

sio.connect(
    'wss://ws.limitless.exchange',
    namespaces=['/markets'],
    headers={'Cookie': f'limitless_session={session_cookie}'}
)
```

## Complete Python Example

```python
#!/usr/bin/env python3
"""
Limitless Exchange WebSocket Client
Real-time market data subscription
"""

import asyncio
import socketio
import os

# Configuration
WS_URL = 'wss://ws.limitless.exchange'
NAMESPACE = '/markets'

class LimitlessWebSocket:
    def __init__(self, session_cookie=None):
        self.sio = socketio.AsyncClient()
        self.session_cookie = session_cookie
        self._setup_handlers()

    def _setup_handlers(self):
        @self.sio.event(namespace=NAMESPACE)
        async def connect():
            print('Connected to Limitless WebSocket')

        @self.sio.event(namespace=NAMESPACE)
        async def disconnect():
            print('Disconnected from Limitless WebSocket')

        @self.sio.on('newPriceData', namespace=NAMESPACE)
        async def on_price_data(data):
            print(f'AMM Price Update: {data}')

        @self.sio.on('orderbookUpdate', namespace=NAMESPACE)
        async def on_orderbook_update(data):
            print(f'Orderbook Update: {data}')

    async def connect(self):
        headers = {}
        if self.session_cookie:
            headers['Cookie'] = f'limitless_session={self.session_cookie}'

        await self.sio.connect(
            WS_URL,
            namespaces=[NAMESPACE],
            headers=headers
        )

    async def subscribe_markets(self, addresses=None, slugs=None):
        """Subscribe to market updates."""
        payload = {}
        if addresses:
            payload['marketAddresses'] = addresses
        if slugs:
            payload['marketSlugs'] = slugs

        await self.sio.emit(
            'subscribe_market_prices',
            payload,
            namespace=NAMESPACE
        )

    async def disconnect(self):
        await self.sio.disconnect()

    async def wait(self):
        """Wait for messages."""
        await self.sio.wait()


async def main():
    # Create client
    client = LimitlessWebSocket()

    # Connect
    await client.connect()

    # Subscribe to markets
    await client.subscribe_markets(
        addresses=['0xE082AF5a25f5D3904fae514CD03dC99F9Ff39fBc'],
        slugs=['btc-100k-2024']
    )

    # Wait for updates
    try:
        await client.wait()
    except KeyboardInterrupt:
        await client.disconnect()


if __name__ == '__main__':
    asyncio.run(main())
```

## Complete TypeScript Example

```typescript
import { io, Socket } from 'socket.io-client';

interface PriceData {
  marketAddress?: string;
  updatedPrices: { yes: number; no: number };
  blockNumber: number;
  timestamp: string;
}

interface OrderbookData {
  marketSlug: string;
  orderbook: {
    bids: Array<{ price: number; size: string }>;
    asks: Array<{ price: number; size: string }>;
  };
  timestamp: string;
}

class LimitlessWebSocket {
  private socket: Socket;
  private onPriceUpdate?: (data: PriceData) => void;
  private onOrderbookUpdate?: (data: OrderbookData) => void;

  constructor(sessionCookie?: string) {
    const opts: Parameters<typeof io>[1] = {
      transports: ['websocket'],
    };

    if (sessionCookie) {
      opts.extraHeaders = {
        Cookie: `limitless_session=${sessionCookie}`,
      };
    }

    this.socket = io('wss://ws.limitless.exchange/markets', opts);
    this.setupHandlers();
  }

  private setupHandlers() {
    this.socket.on('connect', () => {
      console.log('Connected to Limitless WebSocket');
    });

    this.socket.on('disconnect', () => {
      console.log('Disconnected from Limitless WebSocket');
    });

    this.socket.on('newPriceData', (data: PriceData) => {
      console.log('AMM Price Update:', data);
      this.onPriceUpdate?.(data);
    });

    this.socket.on('orderbookUpdate', (data: OrderbookData) => {
      console.log('Orderbook Update:', data);
      this.onOrderbookUpdate?.(data);
    });
  }

  subscribe(addresses?: string[], slugs?: string[]) {
    const payload: {
      marketAddresses?: string[];
      marketSlugs?: string[];
    } = {};

    if (addresses?.length) payload.marketAddresses = addresses;
    if (slugs?.length) payload.marketSlugs = slugs;

    this.socket.emit('subscribe_market_prices', payload);
  }

  onPrice(callback: (data: PriceData) => void) {
    this.onPriceUpdate = callback;
  }

  onOrderbook(callback: (data: OrderbookData) => void) {
    this.onOrderbookUpdate = callback;
  }

  disconnect() {
    this.socket.disconnect();
  }
}

// Usage
const ws = new LimitlessWebSocket();

ws.onPrice((data) => {
  console.log(`YES: ${data.updatedPrices.yes}, NO: ${data.updatedPrices.no}`);
});

ws.onOrderbook((data) => {
  const bestBid = data.orderbook.bids[0]?.price;
  const bestAsk = data.orderbook.asks[0]?.price;
  console.log(`Bid: ${bestBid}, Ask: ${bestAsk}`);
});

ws.subscribe(
  ['0xE082AF5a25f5D3904fae514CD03dC99F9Ff39fBc'],
  ['btc-100k-2024']
);
```

## Event Reference

### Client Events (emit)

| Event | Payload | Description |
|-------|---------|-------------|
| `subscribe_market_prices` | `{ marketAddresses?: string[], marketSlugs?: string[] }` | Subscribe to market updates |

### Server Events (on)

| Event | Payload | Description |
|-------|---------|-------------|
| `newPriceData` | PriceData object | AMM price update |
| `orderbookUpdate` | OrderbookData object | CLOB orderbook update |

## Best Practices

### Reconnection Handling

```typescript
socket.on('disconnect', () => {
  console.log('Disconnected, attempting reconnect...');
});

socket.io.on('reconnect', (attempt) => {
  console.log(`Reconnected after ${attempt} attempts`);
  // Re-subscribe to markets
  socket.emit('subscribe_market_prices', { marketSlugs: ['btc-100k-2024'] });
});
```

### Error Handling

```typescript
socket.on('error', (error) => {
  console.error('WebSocket error:', error);
});

socket.on('connect_error', (error) => {
  console.error('Connection error:', error.message);
});
```

### Heartbeat/Ping

Socket.io handles heartbeat automatically. Default ping interval is 25 seconds.

## Common Issues

### Connection Refused
- Check firewall settings
- Verify WebSocket URL
- Ensure `transports: ['websocket']` is set

### Missing Updates
- Verify subscription was sent after connect
- Check if subscription payload is correct
- Remember: new subscriptions replace old ones

### Authentication Errors
- Session cookie may be expired
- Re-authenticate and reconnect with new session
