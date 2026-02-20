# Order Signature Verification Failed

> **Tip**: The official SDKs handle EIP-712 signing automatically and eliminate most signature errors. Python: `pip install limitless-sdk` ([guide](../guides/sdk.md)). TypeScript: `npm install @limitless-exchange/sdk` ([guide](../guides/sdk-typescript.md)).

## Symptoms

- Order submission returns signature error
- API rejects your signed order
- "Invalid signature" or similar error messages

## Common Causes

### 1. Wrong verifyingContract

You're using a hardcoded address instead of `venue.exchange`:

```python
# WRONG - hardcoded address
domain = {
    "verifyingContract": "0xa4409D988CA2218d956BeEFD3874100F444f0DC3"
}

# CORRECT - fetch from market data
market = get_market(slug)
domain = {
    "verifyingContract": market['venue']['exchange']
}
```

**This is the #1 cause of signature failures.** Each market has its own venue contract address.

### 2. Address not checksummed

All addresses must use EIP-55 mixed-case format:

```python
# WRONG - all lowercase
maker = "0x742d35cc6634c0532925a3b844bc454e4438f44e"

# CORRECT - checksummed
maker = "0x742d35Cc6634C0532925a3b844Bc454e4438f44e"

# Use web3 to checksum
from web3 import Web3
maker = Web3.to_checksum_address(address)
```

### 3. Wrong chain ID

Must be 8453 (Base mainnet):

```python
domain = {
    "name": "Limitless CTF Exchange",
    "version": "1",
    "chainId": 8453,  # Base mainnet - NOT 1, NOT 137
    "verifyingContract": venue_exchange
}
```

### 4. Field type mismatches

Amounts must be strings in the signed message:

```python
order = {
    "makerAmount": str(65000000),   # String, not int
    "takerAmount": str(100000000),  # String, not int
    "salt": str(salt_value),        # String
    "nonce": "0",                   # String
    "feeRateBps": "0",              # String
    # ...
}
```

### 5. Wrong EIP-712 type structure

Make sure your Order type matches exactly:

```python
types = {
    "Order": [
        {"name": "salt", "type": "uint256"},
        {"name": "maker", "type": "address"},
        {"name": "signer", "type": "address"},
        {"name": "taker", "type": "address"},
        {"name": "tokenId", "type": "uint256"},
        {"name": "makerAmount", "type": "uint256"},
        {"name": "takerAmount", "type": "uint256"},
        {"name": "expiration", "type": "uint256"},
        {"name": "nonce", "type": "uint256"},
        {"name": "feeRateBps", "type": "uint256"},
        {"name": "side", "type": "uint8"},
        {"name": "signatureType", "type": "uint8"}
    ]
}
```

## Debugging Checklist

1. [ ] Fetched `venue.exchange` from market data (not hardcoded)
2. [ ] Using checksummed addresses (EIP-55)
3. [ ] Chain ID is 8453
4. [ ] All numeric fields are strings
5. [ ] Domain name is exactly `"Limitless CTF Exchange"`
6. [ ] Domain version is exactly `"1"`
7. [ ] Using the correct private key for the maker address

## Related

- [Placing Orders Guide](../guides/placing-orders.md)
- [EIP-712 Signing](../endpoints/orders.md#eip-712-order-signing)
