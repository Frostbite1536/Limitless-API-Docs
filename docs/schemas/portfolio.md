# Portfolio Data Schemas

Reference documentation for portfolio-related data structures.

## PortfolioPositionsDto

Main response from `/portfolio/positions`.

```typescript
interface PortfolioPositionsDto {
  rewards: PortfolioRewardsDto;
  amm: AmmPositionDto[];
  clob: ClobPositionDto[];
}
```

---

## PositionDataDto

Core position metrics for a single outcome (YES or NO).

```typescript
interface PositionDataDto {
  cost: string;              // Total cost basis (6 decimals)
  fillPrice: string;         // Average fill price (6 decimals)
  realisedPnl: string;       // Realized profit/loss (6 decimals)
  unrealizedPnl: string;     // Unrealized P&L at current price
  marketValue: string;       // Current market value
}
```

### Example

```json
{
  "cost": "75000000",        // 75 USDC spent
  "fillPrice": "750000",     // 0.75 average price
  "realisedPnl": "0",        // No closed positions yet
  "unrealizedPnl": "25000000", // +25 USDC unrealized
  "marketValue": "100000000" // Worth 100 USDC now
}
```

---

## MarketPositionDataDto

Combined position data for both outcomes.

```typescript
interface MarketPositionDataDto {
  yes: PositionDataDto;
  no: PositionDataDto;
}
```

---

## ClobPositionDto

Position in a CLOB market.

```typescript
interface ClobPositionDto {
  market: Market;            // Market details
  positions: MarketPositionDataDto;
  latestTrade: {
    yes: number | null;      // Latest YES trade price
    no: number | null;       // Latest NO trade price
  };
  tokensBalance: {
    yes: string;             // YES token balance (6 decimals)
    no: string;              // NO token balance (6 decimals)
  };
  orders: {
    open: number;            // Count of open orders
    filled: number;          // Count of filled orders
  };
  rewards: object;           // Position-specific rewards
}
```

---

## AmmPositionDto

Position in an AMM market.

```typescript
interface AmmPositionDto {
  market: Market;
  positions: MarketPositionDataDto;
  tokensBalance: {
    yes: string;
    no: string;
  };
  lpTokens: string;          // Liquidity provider tokens (if any)
}
```

---

## PortfolioRewardsDto

Rewards summary for the portfolio.

```typescript
interface PortfolioRewardsDto {
  totalPoints: number;
  epochs: EpochRewardDataDto[];
}
```

---

## EpochRewardDataDto

Rewards data for a specific epoch.

```typescript
interface EpochRewardDataDto {
  epoch: number;             // Epoch identifier
  points: number;            // Points earned
  tradingVolume: string;     // Volume traded (6 decimals)
  liquidityProvided: string; // Liquidity added (6 decimals)
  startDate: string;         // Epoch start (ISO 8601)
  endDate: string;           // Epoch end (ISO 8601)
}
```

---

## HistoryResponseDto

Response from `/portfolio/history`.

```typescript
interface HistoryResponseDto {
  data: HistoryEntryDto[];
  totalCount: number;
}
```

---

## HistoryEntryDto

Single entry in transaction history.

```typescript
interface HistoryEntryDto {
  blockTimestamp: number;    // Unix timestamp
  collateralAmount: string;  // USDC involved (6 decimals)
  market: HistoryMarketDto;
  outcomeTokenAmount: string;
  outcomeTokenAmounts: string[]; // Per-outcome amounts
  outcomeIndex: number;      // 0 = YES, 1 = NO
  outcomeTokenPrice: number; // Price at transaction
  strategy: TransactionStrategy;
  transactionHash: string;   // Blockchain tx hash
}
```

### TransactionStrategy

| Strategy | Description |
|----------|-------------|
| `Buy` | AMM purchase |
| `Sell` | AMM sale |
| `Limit Buy` | CLOB limit buy filled |
| `Limit Sell` | CLOB limit sell filled |
| `Market Buy` | Market order buy |
| `Market Sell` | Market order sell |
| `Split` | Split collateral into tokens |
| `Merge` | Merge tokens into collateral |
| `Convert` | Convert between positions |

---

## HistoryMarketDto

Market info in history entries.

```typescript
interface HistoryMarketDto {
  closed: boolean;
  collateral: {
    symbol: string;          // "USDC"
    id: number;
    decimals: number;        // 6
  };
  group: HistoryMarketGroupDto | null;
  condition_id: string;
  funding: number;
  id: number;
  slug: string;
  title: string;
  deadline: string;
}
```

---

## HistoryMarketGroupDto

Group info for NegRisk markets in history.

```typescript
interface HistoryMarketGroupDto {
  id: number;
  slug: string;
  title: string;
  status: string;
  deadline: string;
  hidden: boolean;
  txHash: string | null;
  resolutionTxHash: string | null;
  priorityIndex: number;
  metadata: {
    isBannered: boolean;
  };
  negRiskMarketId: string;
  createdAt: string;
  updatedAt: string;
}
```

---

## P&L Calculations

### Cost Basis
Total amount spent acquiring tokens:
```typescript
cost = sum(fillPrice * quantity) for all buys
```

### Average Fill Price
Weighted average price:
```typescript
fillPrice = totalCost / totalQuantity
```

### Unrealized P&L
Profit if sold at current price:
```typescript
unrealizedPnl = (currentPrice - fillPrice) * tokensBalance
```

### Realized P&L
Profit from closed positions:
```typescript
realisedPnl = sellProceeds - costBasisOfSoldTokens
```

### Market Value
Current value at market price:
```typescript
marketValue = currentPrice * tokensBalance
```

---

## Example: Full Position Response

```json
{
  "rewards": {
    "totalPoints": 15000,
    "epochs": [
      {
        "epoch": 1,
        "points": 5000,
        "tradingVolume": "50000000000"
      }
    ]
  },
  "amm": [],
  "clob": [
    {
      "market": {
        "address": "0x...",
        "slug": "btc-100k-2024",
        "title": "Will BTC reach $100k in 2024?",
        "deadline": "2024-12-31T23:59:59Z",
        "status": "FUNDED"
      },
      "positions": {
        "yes": {
          "cost": "75000000",
          "fillPrice": "750000",
          "realisedPnl": "0",
          "unrealizedPnl": "25000000",
          "marketValue": "100000000"
        },
        "no": {
          "cost": "0",
          "fillPrice": "0",
          "realisedPnl": "0",
          "unrealizedPnl": "0",
          "marketValue": "0"
        }
      },
      "latestTrade": {
        "yes": 0.80,
        "no": 0.20
      },
      "tokensBalance": {
        "yes": "100000000",
        "no": "0"
      },
      "orders": {
        "open": 2,
        "filled": 5
      },
      "rewards": {}
    }
  ]
}
```

### Interpretation
- Bought 100 YES tokens at $0.75 average (cost: $75)
- Current YES price: $0.80
- Unrealized profit: $25 (100 tokens × $0.05 gain)
- Market value: $100 (100 tokens × $0.80)
