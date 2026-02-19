# Smart Wallet Signature Type Error

> **Tip**: The [`limitless-sdk`](../guides/sdk.md) handles signature type selection automatically for EOA wallets. For the simplest API trading experience, use a fresh EOA wallet with the SDK: `pip install limitless-sdk`

## Symptoms

- Order submission fails with "Invalid signature" or "contract mismatch" error
- Error message suggests wrong exchange address, but you've verified `venue.exchange` is correct
- You're using `smartWallet` and `embeddedAccount` from user data
- Error persists despite correct `verifyingContract` in EIP-712 domain

## Common Scenario

**Using the Limitless frontend website even once can link your wallet to a smart wallet.** This makes the wallet unsuitable for simple EOA API trading unless you use the correct signature type.

If you've ever connected your wallet to the Limitless web app, your account may have been automatically linked to a trade wallet (smart wallet + embedded account). This is a common source of confusion for developers who later try to use the same wallet for API trading.

## Root Cause

When using Limitless trade wallets (smart wallet + embedded account), you have different addresses for `maker` and `signer`. This requires a **non-zero signature type**.

Using `signatureType: 0` (EOA) with different maker/signer addresses will fail:

```python
# WRONG - signatureType 0 requires maker == signer
order = {
    "maker": user_data['smartWallet'],      # Smart wallet address
    "signer": user_data['embeddedAccount'], # Different address!
    "signatureType": 0,                      # EOA - expects same address
    # ...
}
```

## Solutions

### Option 1: Use EOA with matching addresses (Recommended for API usage)

For standard EOA signing, use the same address for both `maker` and `signer`:

```python
# CORRECT - EOA with matching addresses
maker_address = user_data['account']

order = {
    "maker": maker_address,
    "signer": maker_address,  # Same as maker
    "signatureType": 0,       # EOA signature
    # ...
}
```

**Important**: If your wallet was ever used with the Limitless frontend, it may have been linked to a smart wallet. In this case, **create a new wallet specifically for API trading**. This is the cleanest solution and avoids signature type complexity.

### Option 2: Use correct signature type for smart wallets

If you need to use the smart wallet flow, use the appropriate signature type:

| signatureType | Description | maker/signer requirement |
|---------------|-------------|--------------------------|
| 0 | EOA | maker == signer |
| 1 | POLY_PROXY | maker != signer allowed |
| 2 | POLY_GNOSIS_SAFE | maker != signer allowed |
| 3 | EIP1271 | Contract signature |

```python
# For smart wallet flow (if needed)
order = {
    "maker": user_data['smartWallet'],
    "signer": user_data['embeddedAccount'],
    "signatureType": 1,  # Non-zero for smart wallet
    # ...
}
```

## How to Check Your Account Type

After authentication, inspect your user data:

```python
session, user_data = authenticate(private_key)

account = user_data.get('account')           # Your EOA address
smart_wallet = user_data.get('smartWallet')  # Smart wallet (if linked)
embedded = user_data.get('embeddedAccount')  # Embedded account (if linked)

print(f"Account: {account}")
print(f"Smart Wallet: {smart_wallet}")
print(f"Embedded Account: {embedded}")

# If smartWallet exists and differs from account,
# your account is linked to a trade wallet
if smart_wallet and smart_wallet != account:
    print("Account is linked to a smart wallet")
    print("Use a new wallet for simple EOA trading, or use correct signatureType")
```

## Debugging Checklist

1. [ ] Check if `smartWallet` exists in user data
2. [ ] If using `signatureType: 0`, ensure `maker == signer`
3. [ ] If `maker != signer`, use appropriate non-zero `signatureType`
4. [ ] Consider creating a fresh wallet for API-only trading

## Real-World Example

A developer reported "contract mismatch" errors even with the correct `venue.exchange` address. Their code:

```python
# Their problematic code
order = {
    "maker": user_data.get('smartWallet') or user_data['account'],
    "signer": user_data.get('embeddedAccount'),  # Different from maker!
    "signatureType": 0,  # EOA - but addresses don't match
    # ...
}
```

The fix was to use matching addresses:

```python
# Fixed code
maker_address = user_data['account']
order = {
    "maker": maker_address,
    "signer": maker_address,  # Same address
    "signatureType": 0,
    # ...
}
```

Or create a new account not linked to a smart wallet.

## Related

- [Smart Wallet Signer Mismatch](smart-wallet-signer-mismatch.md)
- [Signature Verification Failed](signature-verification-failed.md)
- [Placing Orders Guide](../guides/placing-orders.md)
- [Order Endpoint Documentation](../endpoints/orders.md#signature-types)
