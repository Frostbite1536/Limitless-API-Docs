# Market Data Schemas

Reference documentation for market-related data structures.

## Market

The core market data structure.

```typescript
interface Market {
  // Identifiers
  address: string;           // Ethereum contract address (0x...)
  title: string;             // Market question/title (max 70 chars)
  proxyTitle: string | null; // Alternative title (max 70 chars)
  slug: string;              // URL-friendly identifier

  // Description
  description: string;       // Detailed market description

  // Blockchain Data
  question_id: string;       // Bytes32 parsed title (66 chars)
  condition_id: string;      // Bytes32 condition ID (66 chars)
  position_ids: string[];    // YES/NO position token IDs (max 2)

  // Market State
  outcome_slot_count: 2;     // Always 2 for binary markets
  winning_index: 0 | 1 | null; // Winner (0=YES, 1=NO, null=unresolved)
  payout_numerators: string[] | null; // Oracle payout data
  status: MarketStatus;      // Current market status

  // Metadata
  og_url: string | null;     // OG image URL for SEO
  image_url: string | null;  // Market logo URL
  deadline: string;          // Resolution deadline (ISO 8601)
  hidden: boolean;           // Visibility flag (default: false)
  priority_index: number | null; // Display priority

  // Transaction Hashes
  txHash: string | null;     // Creation transaction
  resolutionTxHash: string | null; // Resolution transaction
}
```

### MarketStatus Values

| Status | Description |
|--------|-------------|
| `CREATED` | Market created but not funded |
| `FUNDED` | Market is active and trading |
| `RESOLVED` | Market has been resolved |
| `DISPUTED` | Resolution is being disputed |

---

## BrowseActiveMarketsResponseDto

Response from `/markets/active` endpoint.

```typescript
interface BrowseActiveMarketsResponseDto {
  markets: Market[];         // List of individual markets
  groups: MarketGroup[];     // List of grouped markets
  totalCount: number;        // Total available
  page: number;              // Current page
  limit: number;             // Items per page
}
```

---

## CategoryCountResponseDto

Response from `/markets/categories/count` endpoint.

```typescript
interface CategoryCountResponseDto {
  categories: {
    [categoryId: string]: number; // Category ID -> market count
  };
  total: number;             // Total markets across all categories
}
```

---

## ClobMarketResponseDto

Detailed CLOB market response.

```typescript
interface ClobMarketResponseDto {
  type: 'single-clob';
  address: string;
  title: string;
  description: string;
  slug: string;
  status: MarketStatus;
  deadline: string;

  // Trading Data
  position_ids: [string, string]; // [YES_ID, NO_ID]
  condition_id: string;
  outcome_slot_count: 2;
  winning_index: number | null;

  // Volume & Liquidity
  volume: string;            // Total traded volume (6 decimals)
  liquidity: string;         // Current liquidity (6 decimals)

  // Current Prices
  yesPrice: number;          // Current YES price (0-1)
  noPrice: number;           // Current NO price (0-1)
}
```

---

## NegRiskGroupResponseDto

Response for NegRisk grouped markets.

```typescript
interface NegRiskGroupResponseDto {
  type: 'group-negrisk';
  slug: string;
  title: string;
  description: string;
  deadline: string;
  status: MarketStatus;

  // NegRisk Specific
  negRiskMarketId: string;   // Onchain market ID (bytes32)

  // Nested Markets
  markets: NegRiskMarket[];
}

interface NegRiskMarket {
  slug: string;
  title: string;
  position_ids: [string, string];
  condition_id: string;
  yesPrice: number;
  noPrice: number;
}
```

---

## AmmMarketResponseDto

Response for AMM markets.

```typescript
interface AmmMarketResponseDto {
  type: 'amm';
  address: string;
  title: string;
  description: string;
  slug: string;
  status: MarketStatus;
  deadline: string;

  // AMM Specific
  liquidity: string;         // Pool liquidity
  volume: string;            // Total volume

  // Current Prices
  yesPrice: number;
  noPrice: number;
}
```

---

## FeedEventsResponseDto

Response from `/markets/{slug}/get-feed-events`.

```typescript
interface FeedEventsResponseDto {
  events: FeedEvent[];
  totalCount: number;
  page: number;
  limit: number;
}

interface FeedEvent {
  type: 'trade' | 'resolution' | 'comment';
  timestamp: string;
  data: object;              // Event-specific data
}
```

---

## Market Slug Format

Market slugs follow these patterns:

```
# Individual markets
btc-price-100k-2024
presidential-election-2024
eth-merge-success

# Group markets
crypto-predictions-2024
election-results-2024
```

**Rules**:
- Lowercase alphanumeric characters
- Words separated by hyphens
- No special characters
- Typically includes topic and date

---

## Position IDs

Position IDs are large integers representing conditional token positions:

```typescript
// Example YES token ID
"19633204485790857949828516737993423758628930235371629943999544859324645414627"

// Example NO token ID
"59488366814164541111277457205683376934541954961614829576144806470851104638710"
```

**Notes**:
- Position IDs are derived from the condition ID and outcome index
- They are used when placing orders and tracking balances
- Always use string representation to avoid precision loss

---

## Monetary Values

All monetary values use USDC with 6 decimal places:

| Display | Raw Value |
|---------|-----------|
| 1 USDC | `1000000` |
| 0.50 USDC | `500000` |
| 100 USDC | `100000000` |
| 1,000,000 USDC | `1000000000000` |

```python
# Convert raw to display
display_value = raw_value / 1_000_000

# Convert display to raw
raw_value = int(display_value * 1_000_000)
```
