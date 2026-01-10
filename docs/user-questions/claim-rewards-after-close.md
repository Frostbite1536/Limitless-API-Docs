# Claiming Rewards After Markets Close

**Question**: Is it possible to claim rewards after markets have closed using the Limitless API?

## Answer

**No, there is no direct API endpoint for claiming/redeeming winnings.** Redemption of winning positions is a blockchain operation that must be performed directly on smart contracts, not through the REST API.

## How Redemption Works

After a market resolves, your winning outcome tokens can be redeemed for collateral (USDC) by interacting with the Conditional Tokens Framework (CTF) smart contract on Base.

### Relevant Contracts

| Contract | Address | Use Case |
|----------|---------|----------|
| Conditional Tokens | `0xc9c98965297bc527861c898329ee280632b76e18` | Standard CLOB markets |
| NegRisk Adapter (v1) | `0xb8DAA4C8C9f690396f671BB601727A4c3741340C` | NegRisk markets (v1) |
| NegRisk Adapter (v2) | `0x7afeB946986211950d17f24176039F12c2aB2436` | NegRisk markets (v2) |
| NegRisk Adapter (v3) | `0x6151EF8368b6316c1aa3C68453EF083ad31E712D` | NegRisk markets (v3) |

## Step-by-Step Guide

### 1. Check Market Resolution Status

First, verify the market is resolved using the API:

```python
import requests

response = requests.get("https://api.limitless.exchange/markets/{slug}")
market = response.json()

if market['status'] == 'RESOLVED':
    winning_index = market['winning_index']  # 0 = YES won, 1 = NO won
    condition_id = market['condition_id']
    print(f"Market resolved! Winning outcome: {'YES' if winning_index == 0 else 'NO'}")
```

### 2. Check Your Position

Verify you hold winning tokens:

```python
response = requests.get(
    "https://api.limitless.exchange/portfolio/positions",
    cookies={"limitless_session": session_cookie}
)
positions = response.json()

# Find your position in the resolved market
for pos in positions['clob']:
    if pos['market']['slug'] == slug:
        yes_balance = int(pos['tokensBalance']['yes'])
        no_balance = int(pos['tokensBalance']['no'])
        print(f"YES tokens: {yes_balance / 1e6}")
        print(f"NO tokens: {no_balance / 1e6}")
```

### 3. Redeem via Smart Contract

Use the Conditional Tokens contract's `redeemPositions` function:

```typescript
import { ethers } from 'ethers';

// Conditional Tokens ABI (partial)
const CTF_ABI = [
  "function redeemPositions(address collateralToken, bytes32 parentCollectionId, bytes32 conditionId, uint[] calldata indexSets) external"
];

const CTF_ADDRESS = "0xc9c98965297bc527861c898329ee280632b76e18";
const USDC_ADDRESS = "0x833589fcd6edb6e08f4c7c32d4f71b54bda02913"; // USDC on Base

async function redeemWinnings(
  signer: ethers.Signer,
  conditionId: string,
  winningIndex: number
) {
  const ctf = new ethers.Contract(CTF_ADDRESS, CTF_ABI, signer);

  // Index sets: [1] for YES (index 0), [2] for NO (index 1)
  const indexSets = winningIndex === 0 ? [1] : [2];

  // Parent collection ID is bytes32(0) for standard markets
  const parentCollectionId = ethers.ZeroHash;

  const tx = await ctf.redeemPositions(
    USDC_ADDRESS,
    parentCollectionId,
    conditionId,
    indexSets
  );

  await tx.wait();
  console.log("Redemption successful:", tx.hash);
}
```

### 4. For NegRisk Markets

NegRisk markets require using the NegRisk Adapter instead:

```typescript
// Get the correct adapter from market venue
const market = await fetch(`https://api.limitless.exchange/markets/${slug}`);
const adapterAddress = market.venue.adapter;

// Use NegRisk Adapter for redemption
const NEGRISK_ADAPTER_ABI = [
  "function redeemPositions(bytes32 conditionId, uint[] calldata indexSets) external"
];

const adapter = new ethers.Contract(adapterAddress, NEGRISK_ADAPTER_ABI, signer);
await adapter.redeemPositions(conditionId, indexSets);
```

## What the API Helps With

While claiming isn't available via API, these endpoints support the redemption workflow:

| Endpoint | Purpose |
|----------|---------|
| `GET /markets/{slug}` | Check `status`, `winning_index`, `condition_id` |
| `GET /portfolio/positions` | View your token balances |
| `GET /portfolio/history` | Track redemption transactions |

## Alternative: Merge Positions

If a market is still active, you can merge equal amounts of YES and NO tokens back into USDC using the `Merge` operation on the CTF contract:

```typescript
// Merge function signature
"function mergePositions(address collateralToken, bytes32 parentCollectionId, bytes32 conditionId, uint[] calldata partition, uint amount) external"
```

This is useful for exiting positions without selling on the order book.

## Common Issues

### "Market not resolved"
The market must have `status: "RESOLVED"` before redemption is possible.

### "No winning tokens"
You must hold the winning outcome's tokens. If `winning_index` is 0 (YES won), you need YES tokens to redeem.

### "Wrong contract version"
For NegRisk markets, always use the adapter address from `market.venue.adapter`, not the base CTF contract.

## Summary

| Action | Method |
|--------|--------|
| Check if market resolved | REST API: `GET /markets/{slug}` |
| View your positions | REST API: `GET /portfolio/positions` |
| Redeem winning tokens | Smart contract: `redeemPositions()` |
| Merge YES+NO to USDC | Smart contract: `mergePositions()` |

The Limitless web interface handles redemption automatically, but for programmatic access you'll need to interact with the smart contracts directly using a library like ethers.js or viem.
