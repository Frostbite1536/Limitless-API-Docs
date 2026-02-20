# Claiming Rewards After Markets Close

**Question**: Is it possible to claim rewards after markets have closed using the Limitless API?

> **See also**: [Claiming & Redeeming Guide](../guides/claiming-redeeming.md) for a comprehensive walkthrough with full TypeScript and Python examples.

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
    headers={"X-API-Key": "lmts_your_key_here"}
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

## Alternative: Merge Positions (Active Markets)

If a market is **still active** (not resolved), you can merge equal amounts of YES and NO tokens back into USDC using `mergePositions()` on the CTF contract. This burns both token types and returns collateral without selling on the order book.

```typescript
const CTF_ABI = [
  "function mergePositions(address collateralToken, bytes32 parentCollectionId, bytes32 conditionId, uint[] calldata partition, uint amount) external"
];

const ctf = new ethers.Contract(CTF_ADDRESS, CTF_ABI, signer);

const tx = await ctf.mergePositions(
  USDC_ADDRESS,
  ethers.ZeroHash,          // parentCollectionId = bytes32(0)
  conditionId,
  [1, 2],                   // partition: both outcomes
  amount                    // amount of each token to merge (6-decimal USDC units)
);
await tx.wait();
```

### Does `mergePositions` Work with Embedded / Smart Wallets (Safe)?

Both `mergePositions` and `redeemPositions` are functions on the CTF contract — they accept calls from any wallet type. The requirement is that the **calling wallet holds the tokens** and the transaction is **signed by an authorized key** (the wallet's own key, or the Safe owner's key).

When using the Limitless frontend, the platform creates:

| Component | What It Is | Role |
|-----------|-----------|------|
| **Embedded account** (`embeddedAccount`) | Privy EOA, stored in the browser/enclave | **Signer** — signs transactions |
| **Smart wallet** (`smartWallet`) | Gnosis Safe contract on Base | **Position holder** — holds outcome tokens and USDC |

Your **positions live in the Safe**, not in the embedded account. The Safe must be the caller of `mergePositions` / `redeemPositions`, which requires a transaction signed by the embedded account (the Safe's owner).

**However, Limitless does not allow export of the Privy-managed embedded wallet private key.** This means you **cannot** programmatically call `mergePositions` through the Safe from your own scripts — the key never leaves the Privy enclave.

#### What this means in practice

| Scenario | Can you `mergePositions`? | How |
|----------|--------------------------|-----|
| **Positions in Safe** (used Limitless frontend) | **Not programmatically** — use the Limitless web interface, which has access to the Privy signer | Via the frontend UI |
| **Positions in your own EOA** (API-only trading, `signatureType: 0`, fresh wallet) | **Yes** — call the contract directly with your own private key | See code example below |

> **Recommendation**: If you need programmatic access to merge/redeem, trade via a **fresh EOA wallet** that you control (not linked to the Limitless frontend). Positions will be held directly in your EOA, and you can call the CTF contract without needing the Privy key.

"Base vs Safe" is **not** an either/or — the Safe *is* deployed on Base. The real distinction is whether you control the signing key.

#### Merging via plain EOA (programmatic)

```python
tx = ctf.functions.mergePositions(
    USDC_ADDRESS,
    b'\x00' * 32,
    condition_id,
    [1, 2],
    merge_amount
).build_transaction({
    'from': your_eoa_address,
    'nonce': w3.eth.get_transaction_count(your_eoa_address),
})
signed = account.sign_transaction(tx)
tx_hash = w3.eth.send_raw_transaction(signed.raw_transaction)
```

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
