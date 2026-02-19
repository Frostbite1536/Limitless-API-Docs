# mergePositions with Embedded / Smart Wallet

## User Question

> Can I use `mergePositions` to take a complete set back to collateral with an embedded wallet? Do positions need to be on Base (not Safe)? Is `redeemPositions` only available "on the contract level" — referring to the Safe?

## Short Answer

**Yes, `mergePositions` works** — it is a function on the CTF (Conditional Tokens Framework) contract deployed on Base. It does not care whether the caller is an EOA, a smart wallet (Safe), or an embedded wallet. As long as the wallet holds both YES and NO outcome tokens and calls the contract correctly, the merge will succeed and return USDC.

## Key Clarifications

### 1. "Embedded wallet" vs "Safe" vs "Base wallet"

In the Limitless context, these terms map to a specific architecture:

| Term | What It Actually Is | Where Tokens Live |
|------|--------------------|--------------------|
| **Embedded wallet** (`embeddedAccount`) | An EOA created by Privy, stored in the browser/enclave. This is the **signer**. | Does NOT hold outcome tokens directly |
| **Smart wallet** / **Safe** (`smartWallet`) | A Gnosis Safe contract wallet on Base. This is the **maker** / position holder. | **Holds the outcome tokens and USDC** |
| **"Base wallet"** (informal) | Usually means a plain EOA on Base (no Safe wrapper) | Holds tokens directly |

When you use the Limitless frontend, it automatically creates a Safe (smart wallet) and an embedded account (Privy EOA signer). Your **positions (outcome tokens) live in the Safe**, not in the embedded account.

### 2. Can the Safe call `mergePositions`?

**Yes.** The CTF contract at `0xc9c98965297bc527861c898329ee280632b76e18` exposes `mergePositions` to any caller. The Safe holds the tokens, so the Safe must be the one calling `mergePositions` (or the call must be executed *through* the Safe).

The function signature on the CTF contract:
```solidity
function mergePositions(
    address collateralToken,
    bytes32 parentCollectionId,
    bytes32 conditionId,
    uint[] calldata partition,
    uint amount
) external
```

### 3. What about `redeemPositions`?

Same story — `redeemPositions` is also on the CTF contract. Both functions are "on the contract level" meaning they are **on-chain smart contract calls**, not REST API endpoints. There is no Limitless REST API for merge or redeem.

| Operation | When to Use | Contract |
|-----------|-------------|----------|
| `mergePositions` | Market still **active** — burn equal YES + NO → get USDC back | CTF: `0xc9c9...` |
| `redeemPositions` | Market **resolved** — burn winning tokens → get USDC back | CTF (standard CLOB) or NegRisk Adapter |

### 4. Positions on Safe vs EOA — does it matter?

**The tokens must be in the wallet that calls the contract.** If your positions are in the Safe (which they are when using the Limitless frontend), then:

- The **Safe** must call `mergePositions` / `redeemPositions`
- You execute this as a Safe transaction, signed by the embedded account (the Safe's owner/signer)
- You can use a Safe SDK or a library like `safe-eth-py` / `@safe-global/protocol-kit` to build and execute Safe transactions

If your positions are in a plain EOA (e.g., you traded purely via API with `signatureType: 0` and your own wallet), then the EOA calls the contract directly — no Safe involvement needed.

## How to Merge via a Safe (Programmatic)

### Step 1: Confirm your token balances

```python
import requests

response = requests.get(
    "https://api.limitless.exchange/portfolio/positions",
    headers={"X-API-Key": "lmts_your_key_here"}
)
positions = response.json()

for pos in positions['clob']:
    yes_bal = int(pos['tokensBalance']['yes'])
    no_bal = int(pos['tokensBalance']['no'])
    mergeable = min(yes_bal, no_bal)
    if mergeable > 0:
        print(f"Market: {pos['market']['slug']}")
        print(f"  YES: {yes_bal / 1e6}, NO: {no_bal / 1e6}")
        print(f"  Mergeable: {mergeable / 1e6} USDC worth")
```

### Step 2: Build the merge transaction

```python
from web3 import Web3

w3 = Web3(Web3.HTTPProvider("https://mainnet.base.org"))

CTF_ADDRESS = "0xc9c98965297bc527861c898329ee280632b76e18"
USDC_ADDRESS = "0x833589fcd6edb6e08f4c7c32d4f71b54bda02913"

CTF_ABI = [{
    "name": "mergePositions",
    "type": "function",
    "inputs": [
        {"name": "collateralToken", "type": "address"},
        {"name": "parentCollectionId", "type": "bytes32"},
        {"name": "conditionId", "type": "bytes32"},
        {"name": "partition", "type": "uint256[]"},
        {"name": "amount", "type": "uint256"}
    ],
    "outputs": []
}]

ctf = w3.eth.contract(address=CTF_ADDRESS, abi=CTF_ABI)

# Build the calldata (this is what gets executed by the Safe)
condition_id = bytes.fromhex("YOUR_CONDITION_ID_WITHOUT_0x")
merge_amount = 1_000_000  # 1 USDC worth of each token

calldata = ctf.functions.mergePositions(
    USDC_ADDRESS,
    b'\x00' * 32,       # parentCollectionId = bytes32(0) for standard markets
    condition_id,
    [1, 2],              # partition: both outcomes
    merge_amount
).build_transaction({'from': '0x0000000000000000000000000000000000000000'})['data']
```

### Step 3: Execute through the Safe

If your tokens are in a Safe, you need to execute the merge as a Safe transaction. This requires the Safe SDK or a multisig transaction service:

```python
# Pseudocode — use @safe-global/protocol-kit or safe-eth-py
from safe_eth.safe import Safe

safe = Safe(safe_address, ethereum_client)
safe_tx = safe.build_multisig_tx(
    to=CTF_ADDRESS,
    value=0,
    data=calldata  # from Step 2
)
safe_tx.sign(embedded_account_private_key)
tx_hash, _ = safe_tx.execute(embedded_account_private_key)
```

For a plain EOA (positions held directly), skip the Safe layer and send the transaction normally:

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

## NegRisk Markets

For NegRisk markets, `mergePositions` works differently. Use the NegRisk Adapter address from `market.venue.adapter` and consult the NegRisk-specific ABI. The adapter's merge function signature may differ from the base CTF.

## Summary

| Question | Answer |
|----------|--------|
| Can I `mergePositions` with an embedded wallet? | Yes — execute the contract call through the Safe that holds the tokens |
| Do positions need to be on "Base" (EOA) not "Safe"? | No — merge works for either. The key is that the **caller must hold the tokens**. |
| Is redeem/merge only "on the contract level"? | Yes — both are on-chain smart contract calls. No REST API endpoint exists for them. |
| Which contract has `mergePositions`? | CTF at `0xc9c98965297bc527861c898329ee280632b76e18` (standard CLOB) |

## Related

- [Claiming & Redeeming Guide](../guides/claiming-redeeming.md) — full merge and redeem documentation
- [Smart Wallet Signature Type](smart-wallet-signature-type.md) — understanding maker/signer with smart wallets
- [Smart Contract Addresses](../contracts.md) — all contract addresses by version
