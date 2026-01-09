# Smart Contract Addresses

This document contains all smart contract addresses for the Limitless Exchange on Base (Chain ID: 8453).

## Conditional Tokens

The Conditional Tokens Framework (CTF) contract that manages outcome token positions:

| Contract | Address |
|----------|---------|
| Conditional Tokens | `0xc9c98965297bc527861c898329ee280632b76e18` |

---

## Trading Venues

Limitless supports multiple trading venue versions for both simple CLOB markets and NegRisk grouped markets. Each market's `venue` object (from `GET /markets/{slug}`) indicates which version to use.

### Simple CLOB Markets

Standard prediction markets with YES/NO outcomes.

#### Version 1 (v1)

| Contract | Address |
|----------|---------|
| CTF Exchange | `0xa4409D988CA2218d956BeEFD3874100F444f0DC3` |
| Fee Module | `0x6d8A7D1898306CA129a74c296d14e55e20aaE87D` |

#### Version 2 (v2)

| Contract | Address |
|----------|---------|
| CTF Exchange | `0xF1De958F8641448A5ba78c01f434085385Af096D` |
| Fee Module | `0xEECD2Cf0FF29D712648fC328be4EE02FC7931c7A` |

#### Version 3 (v3)

| Contract | Address |
|----------|---------|
| CTF Exchange | `0x05c748E2f4DcDe0ec9Fa8DDc40DE6b867f923fa5` |
| Fee Module | `0x5130c2c398F930c4f43B15635410047cBEa9D6EB` |

---

### NegRisk Markets

Grouped markets with multiple related outcomes sharing collateral. NegRisk markets require additional contracts for the adapter, operator, vault, and wrapped collateral.

#### Version 1 (v1)

| Contract | Address |
|----------|---------|
| NegRisk Adapter | `0xb8DAA4C8C9f690396f671BB601727A4c3741340C` |
| NegRisk CTF Exchange | `0x5a38afc17F7E97ad8d6C547ddb837E40B4aEDfC6` |
| NegRisk Fee Module | `0x73FC1B1395bA964FEa8705BfF7eF8EA5C23CC661` |
| NegRisk Operator | `0xAE363abc7B264755e8706D81475C3586D4543992` |
| NegRisk Vault | `0x80307da4d8EA92Cd7a13bBf6B3309431Ca7a1c0E` |
| NegRisk Wrapped Collateral | `0x5d6c6a4fea600e0b1a3ab3ef711060310e27886a` |

#### Version 2 (v2)

| Contract | Address |
|----------|---------|
| NegRisk Adapter | `0x7afeB946986211950d17f24176039F12c2aB2436` |
| NegRisk CTF Exchange | `0x46e607D3f4a8494B0aB9b304d1463e2F4848891d` |
| NegRisk Fee Module | `0x18B3E1192c01286050A0994Bc26f7226Ae4A483d` |
| NegRisk Operator | `0xd2e4a23E57F67a90Bfc999d420fDA16De0Fb2751` |
| NegRisk Vault | `0xd7d245CB2cbE55633e270aF8379E5D4AbA87BD58` |
| NegRisk Wrapped Collateral | `0x428F0FC93221A9957dC667baa07E62d50c6b8c03` |

#### Version 3 (v3)

| Contract | Address |
|----------|---------|
| NegRisk Adapter | `0x6151EF8368b6316c1aa3C68453EF083ad31E712D` |
| NegRisk CTF Exchange | `0xe3E00BA3a9888d1DE4834269f62ac008b4BB5C47` |
| NegRisk Fee Module | `0xfeb646D32a2A558359419a1C9c5dfb47fD92dADb` |
| NegRisk Operator | `0xB7D463037836cFf84FA9ddC25c1136756B4b5F61` |
| NegRisk Vault | `0x2eC22ee9381D0b3570cCb5887960DDFd05d210b3` |
| NegRisk Wrapped Collateral | `0xBd8Ff5Ac78A3739037FEaA18278cC157C4798B01` |

---

## Usage Guide

### Determining the Correct Exchange

When placing orders, you must use the correct exchange address as the `verifyingContract` in EIP-712 signing:

1. **Fetch market data**: `GET /markets/{slug}`
2. **Check market type**: Look at `type` field (`single-clob` vs `group-negrisk`)
3. **Use venue from response**: The `venue.exchange` field contains the correct exchange address

```typescript
const market = await fetch(`https://api.limitless.exchange/markets/${slug}`);
const verifyingContract = market.venue.exchange;
```

### Token Approvals

| Order Type | Market Type | Approve Token | Approve To |
|------------|-------------|---------------|------------|
| BUY | Simple CLOB | USDC | CTF Exchange |
| BUY | NegRisk | USDC | NegRisk CTF Exchange |
| SELL | Simple CLOB | Conditional Token | CTF Exchange |
| SELL | NegRisk | Conditional Token | NegRisk CTF Exchange AND NegRisk Adapter |

### JSON Configuration

For programmatic use, here is the complete configuration:

```json
{
  "conditionalTokens": "0xc9c98965297bc527861c898329ee280632b76e18",
  "tradingVenues": {
    "simple": {
      "v1": {
        "ctfExchange": "0xa4409D988CA2218d956BeEFD3874100F444f0DC3",
        "feeModule": "0x6d8A7D1898306CA129a74c296d14e55e20aaE87D"
      },
      "v2": {
        "ctfExchange": "0xF1De958F8641448A5ba78c01f434085385Af096D",
        "feeModule": "0xEECD2Cf0FF29D712648fC328be4EE02FC7931c7A"
      },
      "v3": {
        "ctfExchange": "0x05c748E2f4DcDe0ec9Fa8DDc40DE6b867f923fa5",
        "feeModule": "0x5130c2c398F930c4f43B15635410047cBEa9D6EB"
      }
    },
    "negrisk": {
      "v1": {
        "negRiskAdapter": "0xb8DAA4C8C9f690396f671BB601727A4c3741340C",
        "negRiskCtfExchange": "0x5a38afc17F7E97ad8d6C547ddb837E40B4aEDfC6",
        "negRiskFeeModule": "0x73FC1B1395bA964FEa8705BfF7eF8EA5C23CC661",
        "negRiskOperator": "0xAE363abc7B264755e8706D81475C3586D4543992",
        "negRiskVault": "0x80307da4d8EA92Cd7a13bBf6B3309431Ca7a1c0E",
        "negRiskWrappedCollateral": "0x5d6c6a4fea600e0b1a3ab3ef711060310e27886a"
      },
      "v2": {
        "negRiskAdapter": "0x7afeB946986211950d17f24176039F12c2aB2436",
        "negRiskCtfExchange": "0x46e607D3f4a8494B0aB9b304d1463e2F4848891d",
        "negRiskFeeModule": "0x18B3E1192c01286050A0994Bc26f7226Ae4A483d",
        "negRiskOperator": "0xd2e4a23E57F67a90Bfc999d420fDA16De0Fb2751",
        "negRiskVault": "0xd7d245CB2cbE55633e270aF8379E5D4AbA87BD58",
        "negRiskWrappedCollateral": "0x428F0FC93221A9957dC667baa07E62d50c6b8c03"
      },
      "v3": {
        "negRiskAdapter": "0x6151EF8368b6316c1aa3C68453EF083ad31E712D",
        "negRiskCtfExchange": "0xe3E00BA3a9888d1DE4834269f62ac008b4BB5C47",
        "negRiskFeeModule": "0xfeb646D32a2A558359419a1C9c5dfb47fD92dADb",
        "negRiskOperator": "0xB7D463037836cFf84FA9ddC25c1136756B4b5F61",
        "negRiskVault": "0x2eC22ee9381D0b3570cCb5887960DDFd05d210b3",
        "negRiskWrappedCollateral": "0xBd8Ff5Ac78A3739037FEaA18278cC157C4798B01"
      }
    }
  }
}
```

---

## Important Notes

1. **Always use `venue` from API**: While these addresses are provided for reference, always fetch the current `venue` from the market endpoint to ensure you're using the correct version.

2. **Checksummed addresses**: All addresses must be EIP-55 checksummed when used in API requests.

3. **Chain ID**: All contracts are deployed on Base mainnet (Chain ID: 8453).

4. **Version selection**: The API automatically provides the correct venue version for each market. You don't need to manually select a version.
