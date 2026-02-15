# Claiming & Redeeming Positions

How to redeem winning positions for USDC after a market resolves on Limitless Exchange.

## Key Takeaway

**There is no REST API endpoint for claiming/redeeming winnings.** Redemption is a blockchain operation performed directly on smart contracts deployed on Base (Chain ID: 8453). The REST API helps you check resolution status and positions, but the actual redemption requires an on-chain transaction.

## Overview

| Step | Method | Description |
|------|--------|-------------|
| 1. Check resolution | REST API | Verify market is resolved and which outcome won |
| 2. Check positions | REST API | Confirm you hold winning outcome tokens |
| 3. Redeem tokens | Smart contract | Call `redeemPositions()` on-chain to receive USDC |

## Prerequisites

- A wallet with winning outcome tokens on Base
- ETH on Base for gas fees
- The market must have `status: "RESOLVED"`
- A library for contract interaction (ethers.js, viem, web3.py, etc.)

## Step 1: Check Market Resolution Status

Use the REST API to verify the market is resolved:

```python
import requests

response = requests.get("https://api.limitless.exchange/markets/{slug}")
market = response.json()

print(f"Status: {market['status']}")
print(f"Winning index: {market.get('winning_index')}")  # 0 = YES, 1 = NO
print(f"Condition ID: {market['condition_id']}")
```

Key fields from the response:

| Field | Description |
|-------|-------------|
| `status` | Must be `"RESOLVED"` for redemption |
| `winning_index` | `0` = YES won, `1` = NO won |
| `condition_id` | Required for the smart contract call |
| `venue` | Contains contract addresses for the market |

## Step 2: Check Your Positions

Verify you hold winning tokens:

```python
response = requests.get(
    "https://api.limitless.exchange/portfolio/positions",
    headers={"X-API-Key": "lmts_your_key_here"}
)
positions = response.json()

for pos in positions['clob']:
    if pos['market']['slug'] == slug:
        yes_balance = int(pos['tokensBalance']['yes'])
        no_balance = int(pos['tokensBalance']['no'])
        print(f"YES tokens: {yes_balance / 1e6}")
        print(f"NO tokens: {no_balance / 1e6}")
```

## Step 3: Redeem via Smart Contract

### Which Contract to Use

| Market Type | Contract | How to Identify |
|-------------|----------|-----------------|
| Simple CLOB (`single-clob`) | Conditional Tokens: `0xc9c98965297bc527861c898329ee280632b76e18` | `market.type == "single-clob"` |
| NegRisk (`group-negrisk`) | NegRisk Adapter from `market.venue.adapter` | `market.type == "group-negrisk"` |

### Standard CLOB Markets

Call `redeemPositions()` on the Conditional Tokens Framework (CTF) contract:

```typescript
import { ethers } from 'ethers';

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

### NegRisk Markets

NegRisk markets use the NegRisk Adapter contract instead of the CTF directly:

```typescript
// Fetch the adapter address from the market data
const response = await fetch(`https://api.limitless.exchange/markets/${slug}`);
const market = await response.json();
const adapterAddress = market.venue.adapter;

const NEGRISK_ADAPTER_ABI = [
  "function redeemPositions(bytes32 conditionId, uint[] calldata indexSets) external"
];

const adapter = new ethers.Contract(adapterAddress, NEGRISK_ADAPTER_ABI, signer);

const indexSets = winningIndex === 0 ? [1] : [2];
const tx = await adapter.redeemPositions(conditionId, indexSets);
await tx.wait();
console.log("NegRisk redemption successful:", tx.hash);
```

Note the NegRisk adapter's `redeemPositions` signature differs from the CTF -- it does not take `collateralToken` or `parentCollectionId` parameters.

### Python Example (web3.py)

```python
from web3 import Web3

w3 = Web3(Web3.HTTPProvider("https://mainnet.base.org"))

CTF_ADDRESS = "0xc9c98965297bc527861c898329ee280632b76e18"
USDC_ADDRESS = "0x833589fcd6edb6e08f4c7c32d4f71b54bda02913"

CTF_ABI = [
    {
        "name": "redeemPositions",
        "type": "function",
        "inputs": [
            {"name": "collateralToken", "type": "address"},
            {"name": "parentCollectionId", "type": "bytes32"},
            {"name": "conditionId", "type": "bytes32"},
            {"name": "indexSets", "type": "uint256[]"}
        ],
        "outputs": []
    }
]

ctf = w3.eth.contract(address=CTF_ADDRESS, abi=CTF_ABI)

def redeem_winnings(account, condition_id: str, winning_index: int):
    index_sets = [1] if winning_index == 0 else [2]
    parent_collection_id = b'\x00' * 32

    tx = ctf.functions.redeemPositions(
        USDC_ADDRESS,
        parent_collection_id,
        bytes.fromhex(condition_id[2:]),  # Remove 0x prefix
        index_sets
    ).build_transaction({
        'from': account.address,
        'nonce': w3.eth.get_transaction_count(account.address),
    })

    signed = account.sign_transaction(tx)
    tx_hash = w3.eth.send_raw_transaction(signed.raw_transaction)
    receipt = w3.eth.wait_for_transaction_receipt(tx_hash)
    print(f"Redemption tx: {receipt.transactionHash.hex()}")
```

## Alternative: Merge Positions (Active Markets Only)

If a market is **still active** (not resolved), you can exit a position by merging equal amounts of YES and NO tokens back into USDC. This burns both token types and returns the collateral.

```typescript
const CTF_ABI = [
  "function mergePositions(address collateralToken, bytes32 parentCollectionId, bytes32 conditionId, uint[] calldata partition, uint amount) external"
];

const ctf = new ethers.Contract(CTF_ADDRESS, CTF_ABI, signer);

// Merge requires equal YES and NO token balances
// partition [1, 2] represents both outcomes
const tx = await ctf.mergePositions(
  USDC_ADDRESS,
  ethers.ZeroHash,          // parentCollectionId
  conditionId,
  [1, 2],                   // partition: both outcomes
  amount                    // amount of each token to merge (in 6-decimal USDC units)
);

await tx.wait();
```

This is useful when you hold both YES and NO tokens and want to recover USDC without selling on the order book.

## Relevant Contract Addresses

See [Smart Contract Addresses](../contracts.md) for the full list. Key contracts for redemption:

| Contract | Address | Purpose |
|----------|---------|---------|
| Conditional Tokens | `0xc9c98965297bc527861c898329ee280632b76e18` | Redeem standard CLOB markets |
| USDC (Base) | `0x833589fcd6edb6e08f4c7c32d4f71b54bda02913` | Collateral token |
| NegRisk Adapter (v1) | `0xb8DAA4C8C9f690396f671BB601727A4c3741340C` | Redeem NegRisk v1 markets |
| NegRisk Adapter (v2) | `0x7afeB946986211950d17f24176039F12c2aB2436` | Redeem NegRisk v2 markets |
| NegRisk Adapter (v3) | `0x6151EF8368b6316c1aa3C68453EF083ad31E712D` | Redeem NegRisk v3 markets |

Always use `market.venue.adapter` from the API response rather than hardcoding adapter addresses.

## API Endpoints That Support Redemption

| Endpoint | Purpose |
|----------|---------|
| `GET /markets/{slug}` | Check `status`, `winning_index`, `condition_id`, `venue` |
| `GET /portfolio/positions` | View your token balances for resolved markets |
| `GET /portfolio/history` | Track redemption transactions after they complete |

## Troubleshooting

### "Market not resolved"

The market must have `status: "RESOLVED"` before you can redeem. Check the status via `GET /markets/{slug}`.

### "No winning tokens"

You must hold the winning outcome's tokens. If `winning_index` is `0` (YES won), you need YES tokens to redeem. Holding only losing tokens yields nothing on redemption.

### "Wrong contract version"

For NegRisk markets, always use the adapter address from `market.venue.adapter`, not the base CTF contract. Using the wrong contract will fail.

### "Transaction reverted"

- Verify you have ETH on Base for gas
- Confirm the `conditionId` matches the market
- Ensure `indexSets` is correct: `[1]` for YES (index 0), `[2]` for NO (index 1)
- Check that you haven't already redeemed these tokens

## Summary

Claiming/redeeming on Limitless is an **on-chain operation**, not an API call. The workflow is:

1. **API**: Check market resolution status and your positions
2. **On-chain**: Call `redeemPositions()` on the appropriate smart contract
3. **Result**: Winning outcome tokens are burned and USDC is returned to your wallet

The Limitless web interface handles this automatically, but for programmatic access you interact with the smart contracts directly using ethers.js, viem, web3.py, or similar libraries.
