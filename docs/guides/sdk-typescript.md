# TypeScript SDK Guide

The **`@limitless-exchange/sdk`** is the official TypeScript SDK for the Limitless Exchange API. It is the **preferred** way to interact with the API for TypeScript/JavaScript developers — it handles authentication, EIP-712 signing, venue caching, order management, and provides full type safety.

> **The SDK is preferred over raw API calls.** It eliminates the most common sources of errors (incorrect EIP-712 signing, missing venue data, wrong address checksums) and provides comprehensive TypeScript type definitions.

## Installation

```bash
npm install @limitless-exchange/sdk
# or
yarn add @limitless-exchange/sdk
# or
pnpm add @limitless-exchange/sdk
```

## Quick Start

```typescript
import { HttpClient, MarketFetcher, OrderClient, Side, OrderType } from '@limitless-exchange/sdk';
import { ethers } from 'ethers';

// API key loaded automatically from LIMITLESS_API_KEY env var
const httpClient = new HttpClient({
  baseURL: 'https://api.limitless.exchange',
});

const marketFetcher = new MarketFetcher(httpClient);

// Get active markets
const markets = await marketFetcher.getActiveMarkets({ limit: 10, sortBy: 'newest' });
console.log(`Found ${markets.totalMarketsCount} markets`);

// Fetch specific market (caches venue data automatically)
const market = await marketFetcher.getMarket('bitcoin-2024');
console.log(`Market: ${market.title}`);

// Place an order
const wallet = new ethers.Wallet(process.env.PRIVATE_KEY!);
const orderClient = new OrderClient({ httpClient, wallet });

const order = await orderClient.createOrder({
  tokenId: market.tokens.yes,
  price: 0.50,
  size: 10,
  side: Side.BUY,
  orderType: OrderType.GTC,
  marketSlug: market.slug,
});

console.log(`Order ID: ${order.order.id}`);
```

## Authentication

The SDK uses API keys (same keys generated via the Limitless Exchange UI).

```typescript
import { HttpClient } from '@limitless-exchange/sdk';

// Option 1: Automatic from LIMITLESS_API_KEY env var (recommended)
const httpClient = new HttpClient({
  baseURL: 'https://api.limitless.exchange',
});

// Option 2: Explicit API key
const httpClient = new HttpClient({
  baseURL: 'https://api.limitless.exchange',
  apiKey: process.env.LIMITLESS_API_KEY,
});

// Optional: Custom headers for all requests
const httpClient = new HttpClient({
  baseURL: 'https://api.limitless.exchange',
  additionalHeaders: {
    'X-Custom-Header': 'value',
  },
});
```

Environment variables (`.env` file):

```bash
LIMITLESS_API_KEY=lmts_your_api_key_here
PRIVATE_KEY=0x...  # For order signing
```

## Market Data

```typescript
import { MarketFetcher } from '@limitless-exchange/sdk';

const marketFetcher = new MarketFetcher(httpClient);

// Browse active markets with sorting and pagination
const markets = await marketFetcher.getActiveMarkets({
  limit: 8,
  sortBy: 'lp_rewards', // 'lp_rewards' | 'ending_soon' | 'newest' | 'high_value'
  page: 1,
});
console.log(`Total: ${markets.totalMarketsCount}`);

// Get specific market (automatically caches venue data)
const market = await marketFetcher.getMarket('market-slug');
console.log(`Title: ${market.title}`);
console.log(`YES Token: ${market.tokens.yes}`);
console.log(`NO Token: ${market.tokens.no}`);

// Get orderbook
const orderbook = await marketFetcher.getOrderbook('market-slug');
```

The SDK **automatically caches venue data** (exchange and adapter contract addresses) so you don't need to manage this yourself.

## Placing Orders

The SDK handles EIP-712 signing, venue resolution, and user data fetching automatically.

### GTC Orders (Good-Till-Cancelled)

```typescript
import { OrderClient, Side, OrderType } from '@limitless-exchange/sdk';

// OrderClient fetches user data automatically on first order
const orderClient = new OrderClient({ httpClient, wallet });

const order = await orderClient.createOrder({
  tokenId: market.tokens.yes,
  price: 0.50,      // Price per share (0.01-0.99)
  size: 10,          // Number of shares
  side: Side.BUY,
  orderType: OrderType.GTC,
  marketSlug: market.slug,
});

console.log(`Order ID: ${order.order.id}`);
```

### FOK Orders (Fill-Or-Kill)

FOK orders execute immediately and completely, or are cancelled. They use `makerAmount` instead of `price` + `size`.

```typescript
// FOK BUY — makerAmount = total USDC to spend
const buyOrder = await orderClient.createOrder({
  tokenId: market.tokens.yes,
  makerAmount: 50,   // Spend $50 USDC
  side: Side.BUY,
  orderType: OrderType.FOK,
  marketSlug: market.slug,
});

// FOK SELL — makerAmount = number of shares to sell
const sellOrder = await orderClient.createOrder({
  tokenId: market.tokens.no,
  makerAmount: 120,  // Sell 120 shares
  side: Side.SELL,
  orderType: OrderType.FOK,
  marketSlug: market.slug,
});

// Check if filled
if (buyOrder.makerMatches && buyOrder.makerMatches.length > 0) {
  console.log(`FILLED: ${buyOrder.makerMatches.length} matches`);
} else {
  console.log('NOT FILLED (cancelled)');
}
```

### Cancel Orders

```typescript
// Cancel single order by ID
await orderClient.cancel(orderId);

// Cancel all orders for a market
await orderClient.cancelAll(marketSlug);
```

## NegRisk Markets

NegRisk markets are group markets with multiple related outcomes:

```typescript
// 1. Fetch the group market
const groupMarket = await marketFetcher.getMarket('group-market-slug');

// 2. Select a submarket
const submarket = groupMarket.markets[0];
const marketDetails = await marketFetcher.getMarket(submarket.slug);

// 3. Place order on the SUBMARKET slug (not the group!)
const order = await orderClient.createOrder({
  tokenId: marketDetails.tokens.yes,
  price: 0.5,
  size: 10,
  side: Side.BUY,
  orderType: OrderType.GTC,
  marketSlug: submarket.slug, // Use submarket slug!
});
```

## Token Approvals

Before placing orders, you must approve tokens for the exchange contracts. This is a one-time setup per wallet.

| Order Type | Market Type | Approve To |
|------------|-------------|------------|
| BUY | All CLOB | USDC → `venue.exchange` |
| SELL | Simple CLOB | CT → `venue.exchange` |
| SELL | NegRisk | CT → `venue.exchange` AND `venue.adapter` |

Use `ethers.js` or `viem` to approve the exchange contract to spend your tokens:

```typescript
import { ethers } from 'ethers';

const usdc = new ethers.Contract(USDC_ADDRESS, ERC20_ABI, wallet);
await usdc.approve(venue.exchange, ethers.MaxUint256);
```

## WebSocket Support

```typescript
import { WebSocketClient } from '@limitless-exchange/sdk';

const wsClient = new WebSocketClient({
  url: 'wss://ws.limitless.exchange',
  autoReconnect: true,
});

wsClient.on('orderbookUpdate', (data) => {
  const bestBid = data.bids[0]?.price;
  const bestAsk = data.asks[0]?.price;
  console.log(`Bid: ${bestBid} | Ask: ${bestAsk}`);
});

await wsClient.connect();
await wsClient.subscribe('subscribe_market_prices', {
  marketSlugs: ['market-slug'],
});
```

## Error Handling & Retry

```typescript
import { withRetry } from '@limitless-exchange/sdk';

const result = await withRetry(
  async () => await orderClient.createOrder(orderData),
  {
    statusCodes: [429, 500, 503],
    maxRetries: 3,
    delays: [2, 5, 10],
    onRetry: (attempt, error, delay) => {
      console.log(`Retry ${attempt + 1} after ${delay}s: ${error.message}`);
    },
  }
);
```

## SDK vs Raw API

| Aspect | SDK (`@limitless-exchange/sdk`) | Raw API (`fetch`) |
|--------|-------------------------------|-------------------|
| EIP-712 signing | Automatic | Manual (error-prone) |
| Venue caching | Automatic | Must implement yourself |
| Authentication | Automatic header injection | Manual header management |
| Type safety | Full TypeScript types | None |
| NegRisk support | Built-in | Manual submarket handling |
| Order construction | `createOrder()` with simple params | Build full payload manually |
| WebSocket | Built-in client with reconnect | Manual socket.io setup |
| Error types | Typed API errors | Raw HTTP status codes |
| Retry logic | Built-in `withRetry` + decorators | Must implement |

## What the SDK Does NOT Cover

- **On-chain operations**: `mergePositions()` and `redeemPositions()` are smart contract calls — use `ethers.js` or `viem` directly. See [Claiming & Redeeming Guide](claiming-redeeming.md).
- **Token approvals**: One-time setup using `ethers.js` or `viem` (see Token Approvals section above).

## Examples

The SDK ships with production-ready code samples:

| Example | Description |
|---------|-------------|
| `basic-auth.ts` | API key authentication |
| `clob-gtc-order.ts` | GTC limit orders |
| `clob-fok-order.ts` | FOK market orders |
| `negrisk-gtc-trading-example.ts` | Complete NegRisk trading |
| `negrisk-fok-order.ts` | FOK on group markets |
| `get-active-markets.ts` | Market discovery with sorting |
| `orderbook.ts` | Orderbook analysis |
| `positions.ts` | Portfolio tracking |
| `websocket-trading.ts` | Real-time order monitoring |
| `websocket-orderbook.ts` | Live orderbook streaming |
| `setup-approvals.ts` | Token approval setup |

## Links

- **npm**: https://www.npmjs.com/package/@limitless-exchange/sdk
- **Source**: https://github.com/limitless-labs-group/limitless-exchange-ts-sdk

## Disclaimer

USE AT YOUR OWN RISK. Trading on prediction markets involves financial risk. Test all functionality with small amounts first. Limitless restricts order placement from US locations due to regulatory requirements.
