# LLM Context: Limitless Exchange API

You are assisting users with the Limitless Exchange API - a prediction market trading platform on Base (chain ID 8453). Users range from confused novices to seasoned blockchain developers.

## What This API Does

Limitless Exchange allows users to:
- Trade YES/NO outcome tokens on prediction markets
- Place limit orders (GTC) or immediate-fill orders (FOK)
- Manage positions and view portfolio data

## Essential Reading Order

When helping a user, consult these files based on their problem:

| User Problem | Read First |
|--------------|------------|
| Getting started | `docs/overview.md`, `docs/guides/placing-orders.md` |
| Authentication issues | `docs/guides/authentication.md`, `docs/endpoints/authentication.md` |
| Order/signature errors | `LLM_DEBUGGING.md`, `docs/user-questions/` |
| Understanding markets | `docs/endpoints/markets.md`, `docs/schemas/market.md` |
| Portfolio questions | `docs/endpoints/portfolio.md` |
| Language-specific help | `docs/quickstart/{python,typescript,java}.md` |

## Authentication

### API Key Authentication (Recommended)

| Method | Header | Status |
|--------|--------|--------|
| API Key | `X-API-Key: lmts_...` | **Required** for programmatic access |
| Cookie Session | `Cookie: limitless_session=...` | **Deprecated** (removal imminent) |

Generate API keys via the Limitless Exchange UI (profile menu → Api keys). No login flow needed.

```python
headers = {"X-API-Key": "lmts_your_key_here"}
response = requests.get(f"{API_URL}/portfolio/positions", headers=headers)
```

> **DEPRECATION NOTICE**: Cookie-based session authentication is deprecated and will be removed within weeks. Migrate to API keys immediately.

## The Order Placement Flow

Every successful order follows this sequence:

```
1. AUTHENTICATE
   Use X-API-Key header with your API key
   (Legacy: POST /auth/login with wallet signature - DEPRECATED)

2. FETCH MARKET DATA (cache this per market)
   GET /markets/{slug}
   → Returns: positionIds, venue.exchange, venue.adapter

3. BUILD ORDER
   Use positionIds for tokenId
   Use user.feeRateBps for feeRateBps

4. SIGN ORDER (EIP-712)
   Domain must use venue.exchange as verifyingContract
   Chain ID must be 8453

5. SUBMIT ORDER
   POST /orders with signed order payload + X-API-Key header
```

## Critical Rules (See LLM_INVARIANTS.md)

Before diagnosing any issue, verify these are true:
- Authentication uses `X-API-Key` header (not deprecated cookie)
- `verifyingContract` comes from `market.venue.exchange` (never hardcoded)
- `maker == signer` when using `signatureType: 0`
- All addresses are checksummed (EIP-55 mixed-case)
- Chain ID is 8453 (Base mainnet)

## Common User Personas

### The Novice
- May not understand EIP-712 signing
- May not know what checksummed addresses are
- Needs step-by-step guidance
- Point them to `docs/quickstart/` for their language

### The Experienced Dev with Signature Errors
- Usually has a subtle configuration issue
- Walk through `LLM_DEBUGGING.md` decision tree
- Check if they used the Limitless frontend (links wallet to smart wallet)
- Verify they're fetching venue.exchange dynamically

### The Integration Developer
- Building automated trading systems
- Needs to understand rate limits, caching strategies
- Point them to `docs/guides/faq.md` for best practices

## How to Debug Issues

1. **Read `LLM_DEBUGGING.md`** for structured decision trees
2. **Check `docs/user-questions/`** for documented issues matching their symptoms
3. **Verify invariants** from `LLM_INVARIANTS.md`
4. **Ask clarifying questions** about their setup:
   - What language/library are they using?
   - Have they ever used the Limitless frontend with this wallet?
   - What exact error message do they see?
   - Can they share their order-building code?

## Key API Concepts

### Venue System
Each CLOB market has a `venue` object with contract addresses:
- `venue.exchange` - Used as `verifyingContract` for EIP-712 signing
- `venue.adapter` - Needed for NegRisk SELL order approvals

**This is market-specific.** Always fetch from `GET /markets/{slug}`.

### Position IDs (Token IDs)
Markets have two positions:
- `positionIds[0]` = YES token
- `positionIds[1]` = NO token

Use these as `tokenId` in orders.

### Signature Types
| Value | Type | When to Use |
|-------|------|-------------|
| 0 | EOA | Standard wallet, maker == signer |
| 1 | POLY_PROXY | Polymarket proxy |
| 2 | POLY_GNOSIS_SAFE | Gnosis Safe |
| 3 | EIP1271 | Contract wallets |

### Smart Wallets vs EOA
- **EOA (Externally Owned Account)**: Regular wallet, signs directly
- **Smart Wallet**: Contract-based wallet linked via Limitless frontend

**Critical**: Using the Limitless web frontend can link a wallet to a smart wallet, making it incompatible with simple EOA API trading. Recommend users create a fresh wallet for API-only trading.

## Files in This Repository

```
/
├── LLM_CONTEXT.md          # This file - start here
├── LLM_INVARIANTS.md       # Critical rules that must be true
├── LLM_DEBUGGING.md        # Decision trees for troubleshooting
├── README.md               # Human-readable overview
├── Official_API_Docs.md    # Comprehensive API reference
├── api-1.yaml              # OpenAPI specification
├── docs/
│   ├── overview.md         # API overview
│   ├── endpoints/          # Endpoint documentation
│   ├── guides/             # How-to guides
│   ├── schemas/            # Data structure documentation
│   ├── quickstart/         # Language-specific tutorials
│   └── user-questions/     # Real user issues and solutions
```

## When You Don't Know

If you cannot find the answer in this documentation:
1. Say so honestly
2. Suggest the user contact Limitless support or check their Discord
3. If it seems like a bug, suggest they report it with:
   - Exact error message
   - Their order-building code
   - What `GET /markets/{slug}` returns for venue data
