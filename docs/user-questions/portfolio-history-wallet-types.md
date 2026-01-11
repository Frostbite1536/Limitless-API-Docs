# Portfolio History Endpoint and Wallet Types

## Symptoms

- You're using a smart wallet and wondering if `/portfolio/history` will work
- You want to know if the endpoint supports non-EOA wallets
- You're only seeing EOA-related documentation

## Possible Causes

### 1. Documentation emphasis on EOA

Most example code in the docs shows EOA authentication, which can make it seem like smart wallets aren't supported.

### 2. Different authentication flows

EOA and smart wallet authentication are slightly different, but both produce valid session cookies.

### 3. Wallet type confusion

There's a difference between the **signer address** (EOA) and the **smart wallet address** (contract), which can cause confusion.

## Solution: Both Wallet Types Are Supported

The `/portfolio/history` endpoint works with **both EOA wallets and smart contract wallets**. The wallet type is determined during authentication, not by the endpoint itself.

### How It Works

When you authenticate, you specify your wallet type:

**For EOA Wallets:**
```python
import requests
from eth_account import Account
from eth_account.messages import encode_defunct

# Get signing message
response = requests.get("https://api.limitless.exchange/auth/signing-message")
message = response.text

# Sign with EOA
account = Account.from_key(private_key)
signable = encode_defunct(text=message)
signed = account.sign_message(signable)

# Login as EOA
headers = {
    'x-account': account.address,
    'x-signing-message': '0x' + message.encode('utf-8').hex(),
    'x-signature': signed.signature.hex(),
}

response = requests.post(
    "https://api.limitless.exchange/auth/login",
    headers=headers,
    json={"client": "eoa"}  # Specify EOA
)

session_cookie = response.cookies.get('limitless_session')
```

**For Smart Wallets:**
```python
# Same signing process with EOA signer, but different login body
response = requests.post(
    "https://api.limitless.exchange/auth/login",
    headers=headers,
    json={
        "client": "smart_wallet",
        "smartWallet": "0x1234...5678"  # Contract address
    }
)

session_cookie = response.cookies.get('limitless_session')
```

### Fetch History (Works for Both)

Once authenticated, use the same endpoint regardless of wallet type:

```python
# This works for both EOA and smart wallet sessions
response = requests.get(
    "https://api.limitless.exchange/portfolio/history",
    params={"page": 1, "limit": 50},
    cookies={"limitless_session": session_cookie}
)

history = response.json()

for entry in history['data']:
    print(f"Strategy: {entry['strategy']}")
    print(f"Market: {entry['market']['title']}")
    print(f"Collateral: {int(entry['collateralAmount']) / 1e6} USDC")
    print(f"Price: {entry['outcomeTokenPrice']}")
    print("---")
```

## Key Differences

| Aspect | EOA | Smart Wallet |
|--------|-----|--------------|
| `client` parameter | `"eoa"` | `"smart_wallet"` |
| Signer address | The wallet address | EOA that controls the wallet |
| `smartWallet` parameter | Not needed | Required (contract address) |
| `/portfolio/history` support | ✅ Yes | ✅ Yes |
| Session cookie | ✅ Produced | ✅ Produced |

## Response Format

Both wallet types return the same history format:

```json
{
  "data": [
    {
      "blockTimestamp": 1744115608,
      "collateralAmount": "65000000",
      "market": {
        "id": 980,
        "slug": "btc-100k",
        "title": "Will BTC hit $100k?",
        "deadline": "2024-12-31T23:59:59Z",
        "closed": false
      },
      "outcomeTokenAmount": "100000000",
      "outcomeTokenAmounts": ["100000000", "0"],
      "outcomeIndex": 0,
      "outcomeTokenPrice": 0.65,
      "strategy": "Limit Buy",
      "transactionHash": "0x..."
    }
  ],
  "totalCount": 50
}
```

## Complete Example with Smart Wallet

```python
import requests
from eth_account import Account
from eth_account.messages import encode_defunct

class SmartWalletAuth:
    def __init__(self, signer_private_key, smart_wallet_address):
        self.signer_private_key = signer_private_key
        self.smart_wallet_address = smart_wallet_address
        self.session = None

    def login(self):
        # Get message
        response = requests.get(
            "https://api.limitless.exchange/auth/signing-message"
        )
        message = response.text

        # Sign with EOA signer
        account = Account.from_key(self.signer_private_key)
        signable = encode_defunct(text=message)
        signed = account.sign_message(signable)

        # Login as smart wallet
        headers = {
            'x-account': account.address,  # Signer EOA
            'x-signing-message': '0x' + message.encode('utf-8').hex(),
            'x-signature': signed.signature.hex(),
        }

        response = requests.post(
            "https://api.limitless.exchange/auth/login",
            headers=headers,
            json={
                "client": "smart_wallet",
                "smartWallet": self.smart_wallet_address  # Contract
            }
        )
        response.raise_for_status()

        self.session = response.cookies.get('limitless_session')
        return response.json()

    def get_history(self, page=1, limit=50):
        response = requests.get(
            "https://api.limitless.exchange/portfolio/history",
            params={"page": page, "limit": limit},
            cookies={"limitless_session": self.session}
        )
        response.raise_for_status()
        return response.json()

# Usage
auth = SmartWalletAuth(
    signer_private_key="0x...",
    smart_wallet_address="0x1234...5678"
)

user_data = auth.login()
print(f"Smart Wallet: {user_data['smartWallet']}")

history = auth.get_history()
print(f"Total transactions: {history['totalCount']}")

for entry in history['data']:
    print(f"{entry['strategy']}: {entry['market']['title']}")
```

## Tips

1. **Session cookie contains wallet context** - Once authenticated, the API knows whether you're an EOA or smart wallet user
2. **No wallet type check needed** - `/portfolio/history` doesn't require specifying wallet type; it's already in your session
3. **Signer vs. smart wallet address** - For smart wallets, you sign with the EOA but specify the contract address in `smartWallet`
4. **History is always personal** - The endpoint returns history for the authenticated wallet (EOA or smart wallet), not for all wallets

## Related

- [Authentication Guide](../guides/authentication.md) - Complete auth flow with wallet types
- [Portfolio Endpoints](../endpoints/portfolio.md) - All portfolio endpoints
- [Smart Wallet Signer Mismatch](smart-wallet-signer-mismatch.md) - Common smart wallet issues
