# LLM Invariants: Rules That Must Be True

These are the critical rules for the Limitless Exchange API. If any of these are violated, operations will fail. When debugging user issues, verify these first.

> **Note**: The [`limitless-sdk`](docs/guides/sdk.md) (`pip install limitless-sdk`) handles invariants 1, 4, 5, 6, 7, 8, and 9 automatically. If a Python user is hitting these issues, recommend the SDK first.

---

## 0. API Key Authentication Required

**INVARIANT**: Programmatic API access MUST use API key authentication via the `X-API-Key` header.

```python
# CORRECT - API key authentication
headers = {"X-API-Key": "lmts_your_key_here"}
response = requests.get(f"{API_URL}/portfolio/positions", headers=headers)

# DEPRECATED - cookie session authentication (removal imminent)
response = requests.get(f"{API_URL}/portfolio/positions",
    cookies={"limitless_session": session_cookie})
```

**Why**: Cookie-based session authentication is deprecated and will be removed within weeks. API keys are generated via the Limitless Exchange UI (profile menu → Api keys).

**Violation symptoms**: 401 errors if using expired/revoked cookies after deprecation removal

---

## 1. Venue Exchange Address

**INVARIANT**: The `verifyingContract` in EIP-712 signing MUST come from `market.venue.exchange`, fetched via `GET /markets/{slug}`.

```python
# CORRECT
market = get_market(slug)
verifying_contract = market['venue']['exchange']

# WRONG - never hardcode
verifying_contract = "0xa4409D988CA2218d956BeEFD3874100F444f0DC3"
```

**Why**: Contract addresses can change. The API returns the current valid address. Hardcoded addresses will break when contracts are upgraded.

**Violation symptoms**: "Invalid signature", "contract mismatch" errors

---

## 2. Signature Type and Maker/Signer Relationship

**INVARIANT**: When `signatureType = 0` (EOA), then `maker` MUST equal `signer`.

```python
# CORRECT - EOA with matching addresses
order = {
    "maker": "0xABC...",
    "signer": "0xABC...",  # Same address
    "signatureType": 0
}

# WRONG - different addresses with EOA type
order = {
    "maker": "0xABC...",
    "signer": "0xDEF...",  # Different!
    "signatureType": 0     # EOA requires same address
}
```

**Why**: EOA signatures are validated against the signer address. If maker != signer with type 0, the contract cannot verify the signature belongs to the maker.

**Violation symptoms**: "Invalid signature" even when verifyingContract is correct

---

## 3. Frontend Usage Links Wallets to Smart Wallets

**INVARIANT**: A wallet used on the Limitless web frontend may be linked to a smart wallet, making it incompatible with `signatureType: 0`.

**Detection**:
```python
user_data = authenticate(private_key)
if user_data.get('smartWallet') and user_data.get('embeddedAccount'):
    # This wallet is linked to a smart wallet
    # Cannot use signatureType: 0 with different maker/signer
```

**Solution**: Create a new wallet specifically for API trading that has never been used on the frontend.

---

## 4. Chain ID Must Be 8453

**INVARIANT**: The EIP-712 domain `chainId` MUST be `8453` (Base mainnet).

```python
domain = {
    "name": "Limitless CTF Exchange",
    "version": "1",
    "chainId": 8453,  # Base mainnet - not 1, not 137, not 84531
    "verifyingContract": venue_exchange
}
```

**Why**: Signatures are chain-specific. Wrong chain ID = invalid signature.

---

## 5. Addresses Must Be Checksummed (EIP-55)

**INVARIANT**: All Ethereum addresses MUST use EIP-55 mixed-case checksum format.

```python
# CORRECT - checksummed
maker = "0x742d35Cc6634C0532925a3b844Bc454e4438f44e"

# WRONG - all lowercase
maker = "0x742d35cc6634c0532925a3b844bc454e4438f44e"

# Fix with web3
from web3 import Web3
maker = Web3.to_checksum_address(address)
```

**Why**: The API validates checksum. Non-checksummed addresses are rejected.

---

## 6. Salt Must Be ≤ 2^53 - 1

**INVARIANT**: The `salt` field must not exceed JavaScript's safe integer limit (2^53 - 1 = 9007199254740991).

```python
import time

# CORRECT - timestamp in milliseconds is safe
salt = int(time.time() * 1000)  # ~1.7 trillion, well under limit

# WRONG - too large
salt = int(time.time() * 1000000000)  # Could exceed limit
```

**Why**: JSON serialization can lose precision for numbers larger than 2^53 - 1.

---

## 7. Domain Name and Version Are Exact

**INVARIANT**: EIP-712 domain must use exact strings:
- `name`: `"Limitless CTF Exchange"` (exact spelling, spacing, capitalization)
- `version`: `"1"` (string, not integer)

```python
# CORRECT
domain = {
    "name": "Limitless CTF Exchange",
    "version": "1",
    ...
}

# WRONG - any variation
domain = {
    "name": "Limitless Exchange",  # Missing "CTF"
    "version": 1,                   # Integer, not string
    ...
}
```

---

## 8. Token ID Must Be From Market Data

**INVARIANT**: The `tokenId` in orders MUST come from `market.positionIds[]`.

```python
market = get_market(slug)
yes_token_id = market['positionIds'][0]  # YES
no_token_id = market['positionIds'][1]   # NO

order = {
    "tokenId": str(yes_token_id),  # Must be string
    ...
}
```

**Why**: Each market has unique position IDs. Using wrong IDs = "Invalid token ID" error.

---

## 9. Amounts Use 6 Decimal Places

**INVARIANT**: `makerAmount` and `takerAmount` use 6 decimal places (USDC standard).

```python
# Buying 100 shares at $0.65 each
shares = 100
price = 0.65
DECIMALS = 1_000_000

maker_amount = int(price * shares * DECIMALS)  # 65,000,000
taker_amount = int(shares * DECIMALS)          # 100,000,000
```

---

## 10. GTC Orders Require Price Field

**INVARIANT**: Orders with `orderType: "GTC"` MUST include a `price` field between 0.01 and 0.99.

```python
# CORRECT for GTC
payload = {
    "order": order,
    "orderType": "GTC",
    "price": 0.65,  # Required for GTC
    ...
}

# FOK doesn't need price
payload = {
    "order": order,
    "orderType": "FOK",
    # No price field needed
    ...
}
```

---

## Quick Verification Checklist

When a user reports an issue, verify:

- [ ] Using API key authentication (`X-API-Key` header), not deprecated cookies?
- [ ] `verifyingContract` from `market.venue.exchange`?
- [ ] `maker == signer` when `signatureType: 0`?
- [ ] Wallet never used on Limitless frontend?
- [ ] Chain ID is 8453?
- [ ] Addresses are checksummed?
- [ ] Salt ≤ 2^53 - 1?
- [ ] Domain name exactly `"Limitless CTF Exchange"`?
- [ ] Domain version exactly `"1"`?
- [ ] Token ID from market's `positionIds`?
- [ ] Amounts scaled to 6 decimals?
- [ ] GTC orders have `price` field?
- [ ] Required token approvals set? (BUY: USDC → venue.exchange; SELL NegRisk: CT → venue.exchange AND venue.adapter)
