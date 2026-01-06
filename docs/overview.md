# Limitless Exchange API Overview

## Introduction

The Limitless Exchange API provides programmatic access to a decentralized prediction market platform built on the Base blockchain (Chain ID: 8453). The API enables trading, portfolio management, and real-time market data access.

## Base URL

```
Production: https://api.limitless.exchange
WebSocket: wss://ws.limitless.exchange
```

## API Categories

| Category | Description | Authentication |
|----------|-------------|----------------|
| [Authentication](endpoints/authentication.md) | Wallet-based login/logout | None |
| [Markets](endpoints/markets.md) | Browse and search prediction markets | Optional |
| [Trading](endpoints/trading.md) | Historical prices, orderbook, market events | Optional |
| [Orders](endpoints/orders.md) | Create and manage buy/sell orders | Required |
| [Portfolio](endpoints/portfolio.md) | User positions, trades, and rewards | Required |

## Key Concepts

### Market Types

1. **CLOB Markets (single-clob)**: Central Limit Order Book markets where users place limit orders
2. **AMM Markets**: Automated Market Maker markets with liquidity pools
3. **NegRisk Groups (group-negrisk)**: Grouped markets with multiple related outcomes sharing collateral

### Trading Mechanics

- **YES/NO Tokens**: Each market has two outcome tokens (`positionIds[0]` = YES, `positionIds[1]` = NO)
- **Prices**: Expressed as decimals (0.01-0.99), representing probability
- **USDC Collateral**: All trades settled in USDC (6 decimals)
- **EIP-712 Signing**: Orders require cryptographic signatures
- **Checksummed Addresses**: All addresses must be EIP-55 checksummed

### Venue System (CRITICAL for Trading)

Each CLOB market has a `venue` object containing contract addresses. You must:

1. **Fetch market data** via `GET /markets/{slug}` to get `venue.exchange` and `venue.adapter`
2. **Use `venue.exchange`** as the `verifyingContract` in EIP-712 order signing
3. **Cache venue data** per market (it's static and doesn't change)

### Required Token Approvals

| Order Type | Market Type | Approve To |
|------------|-------------|------------|
| BUY | All CLOB | USDC → `venue.exchange` |
| SELL | Simple CLOB | CT → `venue.exchange` |
| SELL | NegRisk/Grouped | CT → `venue.exchange` AND `venue.adapter` |

## Authentication Methods

1. **Session Cookie (`limitless_session`)**: Obtained via `/auth/login` using EIP-712 wallet signature
2. **Bearer JWT Token**: Alternative HTTP Bearer authentication

## Quick Reference

### Endpoint Summary

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/auth/signing-message` | Get nonce for authentication |
| POST | `/auth/login` | Authenticate with wallet signature |
| POST | `/auth/logout` | End session |
| GET | `/markets/active` | Browse active markets |
| GET | `/markets/{slug}` | Get market details |
| GET | `/markets/{slug}/orderbook` | Get orderbook |
| POST | `/orders` | Create signed order |
| DELETE | `/orders/{orderId}` | Cancel order |
| GET | `/portfolio/positions` | Get user positions |
| GET | `/portfolio/trades` | Get trade history |

## Documentation Structure

```
docs/
├── overview.md              # This file
├── endpoints/
│   ├── authentication.md    # Auth endpoints
│   ├── markets.md           # Market browsing/search
│   ├── trading.md           # Trading data endpoints
│   ├── orders.md            # Order management
│   └── portfolio.md         # Portfolio & history
├── schemas/
│   ├── market.md            # Market data structures
│   ├── order.md             # Order data structures
│   └── portfolio.md         # Portfolio data structures
├── quickstart/
│   ├── python.md            # Python implementation
│   ├── java.md              # Java implementation
│   └── typescript.md        # TypeScript implementation
└── guides/
    ├── authentication.md    # Auth flow guide
    ├── placing-orders.md    # Order creation guide
    ├── websockets.md        # Real-time data guide
    └── faq.md               # Common questions
```

## Rate Limits

The API implements rate limiting. Check response headers for current limits.

## Error Handling

Standard HTTP status codes:
- `200`: Success
- `400`: Bad request (invalid parameters)
- `401`: Unauthorized (authentication required)
- `404`: Resource not found
- `500`: Server error

Error responses include a `message` field with details.

## Support

- Documentation: https://limitlesslabs.notion.site/
- Support: hey@limitless.network
