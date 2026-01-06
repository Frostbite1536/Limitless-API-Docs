# TypeScript/Node.js Quickstart Guide

Complete guide to trading on Limitless Exchange using TypeScript.

## Prerequisites

### Required Packages

```bash
npm install viem ethers socket.io-client cross-fetch
# or
pnpm add viem ethers socket.io-client cross-fetch
```

### Environment Variables

```bash
export PRIVATE_KEY="0x..."  # Your wallet private key
export API_URL="https://api.limitless.exchange"
```

## Project Setup

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "node",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "./dist"
  }
}
```

## Complete Trading Example

### 1. Authentication Module (auth.ts)

```typescript
import fetch from 'cross-fetch';
import { Wallet } from 'ethers';

const API_URL = process.env.API_URL || 'https://api.limitless.exchange';

export async function getSigningMessage(): Promise<string> {
  const res = await fetch(`${API_URL}/auth/signing-message`);
  if (!res.ok) throw new Error(`Failed: ${res.status}`);
  return res.text();
}

export async function authenticate(privateKey: string) {
  const wallet = new Wallet(privateKey);
  const address = await wallet.getAddress();

  // Get and sign message
  const message = await getSigningMessage();
  const signature = await wallet.signMessage(message);

  const headers = {
    'x-account': address,
    'x-signing-message': `0x${Buffer.from(message, 'utf8').toString('hex')}`,
    'x-signature': signature.startsWith('0x') ? signature : `0x${signature}`,
    'content-type': 'application/json',
    'accept': 'application/json',
  } as const;

  const res = await fetch(`${API_URL}/auth/login`, {
    method: 'POST',
    headers,
    body: JSON.stringify({ client: 'eoa' }),
  });

  if (!res.ok) {
    throw new Error(`Auth failed: ${res.status} ${await res.text()}`);
  }

  // Extract session cookie
  const setCookie = res.headers.get('set-cookie') || '';
  const match = /limitless_session=([^;]+)/i.exec(setCookie);
  const session = match?.[1];

  if (!session) throw new Error('Session cookie not found');

  const user = await res.json();
  return { session, user } as const;
}
```

### 2. Wallet Module (wallet.ts)

```typescript
import { createWalletClient, http } from 'viem';
import { privateKeyToAccount } from 'viem/accounts';
import { base } from 'viem/chains';

export function createWallet(privateKey: `0x${string}`) {
  const account = privateKeyToAccount(privateKey);

  const client = createWalletClient({
    account,
    chain: base,  // Chain ID 8453
    transport: http(),
  });

  return client;
}
```

### 3. Order Signing Module (orders.ts)

```typescript
import fetch from 'cross-fetch';
import { parseUnits, type WalletClient, type Account, type Transport, type Chain } from 'viem';

const API_URL = process.env.API_URL || 'https://api.limitless.exchange';

// EIP-712 Types
const EIP712_DOMAIN = [
  { name: 'name', type: 'string' },
  { name: 'version', type: 'string' },
  { name: 'chainId', type: 'uint256' },
  { name: 'verifyingContract', type: 'address' },
] as const;

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
  { name: 'signatureType', type: 'uint8' },
] as const;

interface OrderMessage {
  salt: string;
  maker: `0x${string}`;
  signer: `0x${string}`;
  taker: `0x${string}`;
  tokenId: string;
  makerAmount: string;
  takerAmount: string;
  expiration: string;
  nonce: string;
  feeRateBps: string;
  side: number;
  signatureType: number;
}

export async function createSignedOrder(
  client: WalletClient<Transport, Chain, Account>,
  options: {
    amount: string;          // USD amount (e.g., '65')
    side: 0 | 1;             // 0 = buy, 1 = sell
    tokenId: string;
    decimals?: number;       // Default 6
    chainId: number;
    verifyingContract: `0x${string}`;
  }
) {
  const maker = client.account!.address as `0x${string}`;
  const salt = String(Math.round(Math.random() * Date.now()));
  const makerAmount = parseUnits(options.amount, options.decimals ?? 6).toString();

  const order: OrderMessage = {
    salt,
    maker,
    signer: maker,
    taker: '0x0000000000000000000000000000000000000000',
    tokenId: options.tokenId,
    makerAmount,
    takerAmount: parseUnits('100', 6).toString(),  // 100 shares
    expiration: '0',
    nonce: '0',
    feeRateBps: '0',
    side: options.side,
    signatureType: 0,
  };

  const typedData = {
    primaryType: 'Order' as const,
    types: { EIP712Domain: EIP712_DOMAIN, Order: ORDER_STRUCTURE },
    domain: {
      name: 'Limitless CTF Exchange',
      version: '1',
      chainId: options.chainId,
      verifyingContract: options.verifyingContract,
    },
    message: order,
  };

  const signature = await client.signTypedData(typedData);
  return { order, signature } as const;
}

export async function submitOrder(
  payload: {
    order: OrderMessage & { price?: number; signature: string };
    ownerId: number;
    orderType: 'FOK' | 'GTC';
    marketSlug: string;
  },
  sessionCookie: string
) {
  const res = await fetch(`${API_URL}/orders`, {
    method: 'POST',
    headers: {
      'accept': 'application/json',
      'content-type': 'application/json',
      'cookie': `limitless_session=${sessionCookie}`,
    },
    body: JSON.stringify(payload),
  });

  if (!res.ok) {
    throw new Error(`Order failed: ${res.status} ${await res.text()}`);
  }

  return res.json();
}
```

### 4. Market Data Module (markets.ts)

```typescript
import fetch from 'cross-fetch';

const API_URL = process.env.API_URL || 'https://api.limitless.exchange';

export async function getMarket(slug: string) {
  const res = await fetch(`${API_URL}/markets/${slug}`);
  if (!res.ok) throw new Error(`Failed: ${res.status}`);
  return res.json();
}

export async function getOrderbook(slug: string) {
  const res = await fetch(`${API_URL}/markets/${slug}/orderbook`);
  if (!res.ok) throw new Error(`Failed: ${res.status}`);
  return res.json();
}

export async function getActiveMarkets(page = 1, limit = 10) {
  const res = await fetch(
    `${API_URL}/markets/active?page=${page}&limit=${limit}`
  );
  if (!res.ok) throw new Error(`Failed: ${res.status}`);
  return res.json();
}
```

### 5. WebSocket Module (websocket.ts)

```typescript
import { io, Socket } from 'socket.io-client';

export function connectToMarkets(session?: string): Socket {
  const url = 'wss://ws.limitless.exchange/markets';

  const opts: Parameters<typeof io>[1] = {
    transports: ['websocket'],
  };

  if (session) {
    opts.extraHeaders = { Cookie: `limitless_session=${session}` };
  }

  return io(url, opts);
}

export function subscribeToOrderbook(socket: Socket, marketSlugs: string[]) {
  socket.emit('subscribe_market_prices', { marketSlugs });

  socket.on('orderbookUpdate', (data: unknown) => {
    console.log('Orderbook update:', data);
  });
}

export function subscribeToAmmPrices(socket: Socket, marketAddresses: string[]) {
  socket.emit('subscribe_market_prices', { marketAddresses });

  socket.on('newPriceData', (data: unknown) => {
    console.log('AMM price update:', data);
  });
}
```

### 6. Main Application (main.ts)

```typescript
import { authenticate } from './auth';
import { createWallet } from './wallet';
import { createSignedOrder, submitOrder } from './orders';
import { getMarket } from './markets';
import { connectToMarkets, subscribeToOrderbook } from './websocket';

// Cache for market data (venue is static per market)
const marketCache = new Map<string, any>();

async function getMarketCached(slug: string) {
  if (marketCache.has(slug)) {
    return marketCache.get(slug);
  }
  const market = await getMarket(slug);
  marketCache.set(slug, market);
  return market;
}

async function main() {
  const privateKey = process.env.PRIVATE_KEY as `0x${string}`;
  if (!privateKey) {
    throw new Error('PRIVATE_KEY environment variable required');
  }

  const API_URL = process.env.API_URL || 'https://api.limitless.exchange';

  // 1. Authenticate
  console.log('Authenticating...');
  const { session, user } = await authenticate(privateKey);
  console.log('Authenticated as:', user.account);

  // 2. Create wallet client
  const wallet = createWallet(privateKey);

  // 3. Get market data (cached per market)
  const marketSlug = 'btc-100k-2024';
  const market = await getMarketCached(marketSlug);

  // Get venue exchange for EIP-712 signing (CRITICAL)
  const venueExchange = market.venue.exchange as `0x${string}`;
  console.log('Using venue exchange:', venueExchange);

  // Get token ID (positionIds[0] = YES, positionIds[1] = NO)
  const tokenId = market.positionIds[0];

  // 4. Connect WebSocket for real-time updates
  const socket = connectToMarkets(session);
  subscribeToOrderbook(socket, [marketSlug]);

  // 5. Create and sign order using venue's exchange address
  console.log('Creating order...');
  const { order, signature } = await createSignedOrder(wallet, {
    amount: '65',  // $65 total
    side: 0,       // BUY
    tokenId,
    decimals: 6,
    chainId: 8453,
    verifyingContract: venueExchange,  // From market's venue data
  });

  // 6. Submit order
  const orderPayload = {
    order: { ...order, price: 0.65, signature },
    ownerId: user.id,
    orderType: 'GTC' as const,
    marketSlug,
  };

  const result = await submitOrder(orderPayload, session);
  console.log('Order created:', result);

  // Keep WebSocket connection alive
  socket.on('disconnect', () => {
    console.log('WebSocket disconnected');
  });
}

main().catch(console.error);
```

## Running the Application

```bash
# Compile TypeScript
npx tsc

# Run with environment variables
PRIVATE_KEY="0x..." node dist/main.js
```

## Common Patterns

### Error Handling

```typescript
async function safeApiCall<T>(fn: () => Promise<T>): Promise<T | null> {
  try {
    return await fn();
  } catch (error) {
    if (error instanceof Error) {
      console.error('API Error:', error.message);
    }
    return null;
  }
}
```

### Retry Logic

```typescript
async function withRetry<T>(
  fn: () => Promise<T>,
  retries = 3,
  delay = 1000
): Promise<T> {
  for (let i = 0; i < retries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (i === retries - 1) throw error;
      await new Promise(r => setTimeout(r, delay * (i + 1)));
    }
  }
  throw new Error('Retry failed');
}
```

### Type Definitions

```typescript
interface Market {
  address: string;
  slug: string;
  title: string;
  positionIds: [string, string];  // [YES, NO] position token IDs
  status: 'FUNDED' | 'RESOLVED' | 'DISPUTED';
  deadline: string;
  venue: {
    exchange: string;  // Use as verifyingContract in EIP-712
    adapter: string;   // For NegRisk SELL approvals
  };
}

interface OrderResponse {
  order: {
    orderId: string;
    price: number;
    side: 0 | 1;
    status: string;
  };
  makerMatches: Array<{
    fillAmount: string;
    price: number;
  }>;
}
```

## Disclaimer

This code is for educational purposes only. Limitless Labs is not responsible for any losses. Always test with small amounts first.
