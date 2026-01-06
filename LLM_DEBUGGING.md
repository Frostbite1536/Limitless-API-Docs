# LLM Debugging: Decision Trees for Troubleshooting

Use these decision trees to systematically diagnose user issues. Start with the error category, then follow the tree.

---

## Error Category Index

| Error Type | Jump To |
|------------|---------|
| Signature/signing errors | [Signature Errors](#signature-errors) |
| Authentication errors | [Authentication Errors](#authentication-errors) |
| Order submission errors | [Order Submission Errors](#order-submission-errors) |
| Order not filling | [Order Not Filling](#order-not-filling) |
| Market/token errors | [Market Errors](#market-errors) |

---

## Signature Errors

**Symptoms**: "Invalid signature", "contract mismatch", "signature verification failed"

```
START: User gets signature error
│
├─► Q1: Is verifyingContract from venue.exchange?
│   │
│   ├─► NO: "Fetch venue.exchange from GET /markets/{slug}"
│   │       See: docs/user-questions/signature-verification-failed.md
│   │
│   └─► YES: Continue to Q2
│
├─► Q2: Is maker == signer in the order?
│   │
│   ├─► NO: What is signatureType?
│   │   │
│   │   ├─► signatureType = 0
│   │   │   "EOA (type 0) requires maker == signer"
│   │   │   "Either use matching addresses OR use correct signatureType for smart wallet"
│   │   │   See: docs/user-questions/smart-wallet-signature-type.md
│   │   │
│   │   └─► signatureType != 0
│   │       Continue to Q3
│   │
│   └─► YES: Continue to Q3
│
├─► Q3: Has wallet been used on Limitless frontend?
│   │
│   ├─► YES or UNSURE:
│   │   "Frontend usage links wallet to smart wallet"
│   │   "Check user_data for smartWallet and embeddedAccount fields"
│   │   "Recommend: Create new wallet for API-only trading"
│   │   See: docs/user-questions/smart-wallet-signature-type.md
│   │
│   └─► NO (fresh wallet): Continue to Q4
│
├─► Q4: Is chainId exactly 8453?
│   │
│   ├─► NO: "Must be 8453 (Base mainnet)"
│   │
│   └─► YES: Continue to Q5
│
├─► Q5: Are all addresses checksummed (EIP-55)?
│   │
│   ├─► NO: "Use Web3.to_checksum_address() or equivalent"
│   │
│   └─► YES: Continue to Q6
│
├─► Q6: Is domain.name exactly "Limitless CTF Exchange"?
│   │
│   ├─► NO: "Must be exact: 'Limitless CTF Exchange'"
│   │
│   └─► YES: Continue to Q7
│
├─► Q7: Is domain.version exactly "1" (string)?
│   │
│   ├─► NO: "Must be string '1', not integer 1"
│   │
│   └─► YES: Continue to Q8
│
├─► Q8: Is salt ≤ 2^53 - 1?
│   │
│   ├─► NO or UNSURE: "Salt must not exceed 9007199254740991"
│   │                  "Use int(time.time() * 1000) for safe values"
│   │
│   └─► YES: Continue to Q9
│
└─► Q9: Is the EIP-712 type structure correct?
    │
    ├─► UNSURE: "Verify Order type matches exactly:"
    │           See: docs/endpoints/orders.md (EIP-712 Order Signing section)
    │
    └─► YES: "Issue may be in signing library or key mismatch"
            "Ask user to verify they're using correct private key"
            "Check if signing library handles typed data correctly"
```

---

## Authentication Errors

**Symptoms**: 401 errors, "not authenticated", session issues

```
START: User gets authentication error
│
├─► Q1: Did they complete the login flow?
│   │
│   ├─► NO: "Must call GET /auth/signing-message, sign it, then POST /auth/login"
│   │       See: docs/guides/authentication.md
│   │
│   └─► YES: Continue to Q2
│
├─► Q2: Are they storing and sending the session cookie?
│   │
│   ├─► NO: "Session cookie 'limitless_session' must be sent with requests"
│   │
│   └─► YES: Continue to Q3
│
├─► Q3: Is x-account header checksummed?
│   │
│   ├─► NO: "x-account header must use EIP-55 checksum"
│   │
│   └─► YES: Continue to Q4
│
├─► Q4: Did they use client: "eoa" in login?
│   │
│   ├─► NO or UNSURE: "For API trading, use client: 'eoa' in login request"
│   │
│   └─► YES: "Check if session expired - try re-authenticating"
```

---

## Order Submission Errors

**Symptoms**: 400 errors on POST /orders, validation errors

```
START: User gets order submission error
│
├─► Error: "Invalid signature"
│   └─► Go to: Signature Errors section above
│
├─► Error: "Invalid price"
│   └─► "GTC orders require price between 0.01 and 0.99"
│       "Check: Is orderType GTC? Is price field present and in range?"
│
├─► Error: "Invalid token ID"
│   └─► "tokenId must come from market.positionIds[0] or [1]"
│       "Fetch fresh market data via GET /markets/{slug}"
│       See: docs/user-questions/invalid-token-id.md
│
├─► Error: "Signer does not match"
│   └─► "Smart wallet mismatch"
│       See: docs/user-questions/smart-wallet-signer-mismatch.md
│
├─► Error: "Market not found" or "Invalid market"
│   └─► "Check marketSlug is correct"
│       "Verify market exists and is active (not resolved)"
│
├─► Error: "Insufficient balance"
│   └─► "Check USDC balance for BUY orders"
│       "Check token balance for SELL orders"
│       "Verify approvals are set for venue.exchange"
│
└─► Other 400 error
    └─► "Ask user for exact error message"
        "Check all fields match schema in docs/schemas/order.md"
```

---

## Order Not Filling

**Symptoms**: Order accepted (201) but no trade executes

```
START: Order submitted successfully but nothing happens
│
├─► Q1: What orderType was used?
│   │
│   ├─► GTC (Good Till Cancelled)
│   │   "GTC orders wait for a matching counterparty"
│   │   "Order will fill when someone takes the other side"
│   │   "This is normal behavior, not an error"
│   │   See: docs/user-questions/order-not-filling.md
│   │
│   └─► FOK (Fill Or Kill)
│       └─► Q2: Did it return success or error?
│           │
│           ├─► Success but no fill
│           │   "FOK should fail if no match - this is unexpected"
│           │   "Check response for 'makerMatches' array"
│           │
│           └─► Error
│               "No matching liquidity at that price"
│               "Try GTC order or adjust price"
│
├─► Q2: Is the price competitive?
│   │
│   ├─► UNSURE: "Check current orderbook via GET /markets/{slug}/orderbook"
│   │           "User's price may be too far from market"
│   │
│   └─► YES: Continue to Q3
│
└─► Q3: Is the market still active?
    │
    ├─► NO (resolved): "Cannot trade on resolved markets"
    │
    └─► YES: "Order is in queue, waiting for match"
            "User can check open orders via GET /orders"
```

---

## Market Errors

**Symptoms**: Can't fetch market, wrong data, position ID issues

```
START: User has market-related issues
│
├─► Error: "Market not found"
│   └─► "Check slug is correct (case-sensitive)"
│       "Market may have been removed or resolved"
│
├─► Issue: Wrong position IDs
│   └─► "Always fetch fresh from GET /markets/{slug}"
│       "positionIds[0] = YES, positionIds[1] = NO"
│       "IDs are market-specific, never hardcode"
│
├─► Issue: Venue data missing
│   └─► "Market may not be a CLOB market"
│       "Check market.type - only CLOB markets have venue"
│
└─► Issue: Stale data
    └─► "Cache market data per slug, but refresh periodically"
        "Venue addresses are static but position IDs are market-specific"
```

---

## Questions to Ask Users

When debugging, gather this information:

### For Signature Errors
1. "What exact error message do you see?"
2. "How are you getting the verifyingContract address?"
3. "Can you share your domain object for EIP-712 signing?"
4. "Have you ever used this wallet on the Limitless website?"
5. "What are the values of maker, signer, and signatureType?"

### For Authentication Errors
1. "Are you storing the session cookie from login?"
2. "What client type did you use in login? ('eoa' or 'smart_wallet')"
3. "Is your x-account header checksummed?"

### For Order Errors
1. "Can you share your order payload (redact sensitive data)?"
2. "What does GET /markets/{slug} return for venue and positionIds?"
3. "What's your orderType (GTC or FOK)?"

### For General Debugging
1. "What programming language and libraries are you using?"
2. "Can you share the relevant code snippet?"
3. "Is this a new integration or did it work before?"

---

## Documented User Issues

Always check `docs/user-questions/` for similar issues:

| File | Issue |
|------|-------|
| `smart-wallet-signer-mismatch.md` | Signer address mismatch |
| `smart-wallet-signature-type.md` | Wrong signatureType with smart wallet |
| `signature-verification-failed.md` | General signature debugging |
| `order-not-filling.md` | GTC orders waiting for match |
| `invalid-token-id.md` | Token/position ID errors |
