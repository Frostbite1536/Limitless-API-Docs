# Guide: Authentication Flow

Complete guide to authenticating with the Limitless Exchange API.

> **Use an official SDK?** Both the [Python SDK](sdk.md) (`pip install limitless-sdk`) and [TypeScript SDK](sdk-typescript.md) (`npm install @limitless-exchange/sdk`) handle authentication automatically — just set the `LIMITLESS_API_KEY` environment variable.

## Overview

Limitless Exchange supports two authentication methods:

| Method | Header | Status |
|--------|--------|--------|
| API Key | `X-API-Key: lmts_...` | **Required** for programmatic access |
| Cookie Session | `Cookie: limitless_session=...` | **Deprecated** (removal imminent) |

> **DEPRECATION NOTICE**: Cookie-based session authentication is deprecated and will be removed within weeks. Please migrate to API keys immediately.

## API Key Authentication (Recommended)

API keys are the simplest and recommended way to authenticate with the Limitless Exchange API.

### Getting an API Key

1. Log in to [limitless.exchange](https://limitless.exchange) using your wallet
2. Click your profile menu (top right)
3. Select "Api keys"
4. Generate a new key

### Using Your API Key

Include the `X-API-Key` header in all requests:

```python
import requests

API_URL = "https://api.limitless.exchange"

# All authenticated requests use the X-API-Key header
headers = {"X-API-Key": "lmts_your_key_here"}

# Example: Get positions
response = requests.get(f"{API_URL}/portfolio/positions", headers=headers)
positions = response.json()

# Example: Submit an order
response = requests.post(f"{API_URL}/orders", json=order_payload, headers=headers)
```

```javascript
// JavaScript/TypeScript
const headers = { 'X-API-Key': 'lmts_your_key_here' };

const res = await fetch('https://api.limitless.exchange/portfolio/positions', { headers });
const positions = await res.json();
```

### API Key Authentication Flow Diagram

```
┌─────────┐                      ┌─────────┐
│  Client │                      │   API   │
└────┬────┘                      └────┬────┘
     │                                │
     │  Any request                   │
     │  Header: X-API-Key: lmts_...  │
     │ ─────────────────────────────> │
     │                                │
     │  Response data                 │
     │ <───────────────────────────── │
     │                                │
```

No login flow, no session management, no cookie handling needed.

---

## Cookie-Based Authentication (Deprecated)

> **WARNING**: Cookie-based session authentication is deprecated and will be removed within weeks. Migrate to API keys immediately.

### Legacy Authentication Flow

1. Request a signing message with unique nonce
2. Sign the message with your Ethereum wallet
3. Submit the signature to login
4. Use the session cookie for authenticated requests

### Legacy Flow Diagram

```
┌─────────┐                      ┌─────────┐
│  Client │                      │   API   │
└────┬────┘                      └────┬────┘
     │                                │
     │  GET /auth/signing-message     │
     │ ─────────────────────────────> │
     │                                │
     │  "Welcome to Limitless...      │
     │   Nonce: 0xa1b2c3..."          │
     │ <───────────────────────────── │
     │                                │
     │  [Sign message with wallet]    │
     │                                │
     │  POST /auth/login              │
     │  Headers: x-account,           │
     │           x-signing-message,   │
     │           x-signature          │
     │ ─────────────────────────────> │
     │                                │
     │  Set-Cookie: limitless_session │
     │  { account, id, ... }          │
     │ <───────────────────────────── │
     │                                │
     │  [Use cookie for requests]     │
     │                                │
```

## Step 1: Get Signing Message

Request a unique signing message from the API.

### Request

```http
GET /auth/signing-message
```

### Response

```
Welcome to Limitless.exchange! Please sign this message to verify your identity.

Nonce: 0xa1b2c3d4e5f67890abcdef1234567890abcdef1234567890abcdef1234567890
```

### Code Example

```python
import requests

def get_signing_message():
    response = requests.get("https://api.limitless.exchange/auth/signing-message")
    response.raise_for_status()
    return response.text
```

## Step 2: Sign the Message

Sign the message using your Ethereum wallet.

### Python (eth-account)

```python
from eth_account import Account
from eth_account.messages import encode_defunct

def sign_message(private_key, message):
    account = Account.from_key(private_key)

    # Encode as personal message
    signable = encode_defunct(text=message)

    # Sign
    signed = account.sign_message(signable)

    return signed.signature.hex()

# Usage
signing_message = get_signing_message()
signature = sign_message(private_key, signing_message)
```

### JavaScript (ethers)

```javascript
import { Wallet } from 'ethers';

async function signMessage(privateKey, message) {
    const wallet = new Wallet(privateKey);
    const signature = await wallet.signMessage(message);
    return signature;
}
```

### JavaScript (viem)

```javascript
import { createWalletClient, http } from 'viem';
import { privateKeyToAccount } from 'viem/accounts';

async function signMessage(privateKey, message) {
    const account = privateKeyToAccount(privateKey);
    const client = createWalletClient({
        account,
        transport: http()
    });

    return await client.signMessage({ message });
}
```

## Step 3: Login

Submit the signature to authenticate.

### Request

```http
POST /auth/login
Content-Type: application/json

Headers:
  x-account: 0x742d35Cc6634C0532925a3b844Bc454e4438f44e
  x-signing-message: 0x57656c636f6d6520746f204c696d69746c6573732e657863...
  x-signature: 0x123abc456def789...

Body:
{
  "client": "eoa"
}
```

### Header Preparation

| Header | Format | Example |
|--------|--------|---------|
| `x-account` | Checksummed address | `0x742d35Cc6634C0532925a3b844Bc454e4438f44e` |
| `x-signing-message` | Hex-encoded message | `0x57656c636f6d65...` |
| `x-signature` | Hex signature with 0x prefix | `0xabc123...` |

### Code Example

```python
def login(private_key, signing_message):
    account = Account.from_key(private_key)

    # Sign message
    message = encode_defunct(text=signing_message)
    signed = account.sign_message(message)

    # Prepare headers
    headers = {
        'x-account': account.address,
        'x-signing-message': '0x' + signing_message.encode('utf-8').hex(),
        'x-signature': '0x' + signed.signature.hex(),
        'Content-Type': 'application/json'
    }

    # Login
    response = requests.post(
        "https://api.limitless.exchange/auth/login",
        headers=headers,
        json={"client": "eoa"}
    )
    response.raise_for_status()

    # Extract session
    session_cookie = response.cookies.get('limitless_session')
    user_data = response.json()

    return session_cookie, user_data
```

### Response

```json
{
  "account": "0x742d35Cc6634C0532925a3b844Bc454e4438f44e",
  "displayName": "0x742d...f44e",
  "smartWallet": null,
  "client": "eoa",
  "id": 12345
}
```

The response includes a `Set-Cookie` header with `limitless_session`.

## Step 4: Use Session Cookie (Deprecated)

> **Deprecated**: Use API key authentication instead. See [API Key Authentication](#api-key-authentication-recommended) above.

Include the session cookie in authenticated requests.

```python
# DEPRECATED - use API key instead
def get_positions(session_cookie):
    response = requests.get(
        "https://api.limitless.exchange/portfolio/positions",
        cookies={"limitless_session": session_cookie}
    )
    return response.json()

# RECOMMENDED - use API key
def get_positions(api_key):
    response = requests.get(
        "https://api.limitless.exchange/portfolio/positions",
        headers={"X-API-Key": api_key}
    )
    return response.json()
```

## Complete Authentication Module

### Using API Key (Recommended)

```python
import os
import requests

class LimitlessClient:
    """Simple API client using API key authentication."""
    API_URL = "https://api.limitless.exchange"

    def __init__(self, api_key=None):
        self.api_key = api_key or os.environ.get('LIMITLESS_API_KEY')
        if not self.api_key:
            raise ValueError("API key required. Set LIMITLESS_API_KEY env var or pass api_key parameter.")

    def _headers(self):
        """Get headers with API key."""
        return {"X-API-Key": self.api_key}

    def get_positions(self):
        """Get all portfolio positions."""
        response = requests.get(f"{self.API_URL}/portfolio/positions", headers=self._headers())
        response.raise_for_status()
        return response.json()

    def get_trades(self):
        """Get trade history."""
        response = requests.get(f"{self.API_URL}/portfolio/trades", headers=self._headers())
        response.raise_for_status()
        return response.json()

    def submit_order(self, order_payload):
        """Submit a signed order."""
        response = requests.post(f"{self.API_URL}/orders", json=order_payload, headers=self._headers())
        response.raise_for_status()
        return response.json()

    def cancel_order(self, order_id):
        """Cancel a specific order."""
        response = requests.delete(f"{self.API_URL}/orders/{order_id}", headers=self._headers())
        response.raise_for_status()
        return response.json()

# Usage
client = LimitlessClient()  # Uses LIMITLESS_API_KEY env var
positions = client.get_positions()
```

### Using Cookie Session (Deprecated)

> **WARNING**: This method is deprecated. Migrate to API keys.

```python
import os
import requests
from eth_account import Account
from eth_account.messages import encode_defunct

class LimitlessAuth:
    """DEPRECATED: Use LimitlessClient with API key instead."""
    API_URL = "https://api.limitless.exchange"

    def __init__(self, private_key=None):
        self.private_key = private_key or os.environ.get('PRIVATE_KEY')
        if not self.private_key:
            raise ValueError("Private key required")

        self.account = Account.from_key(self.private_key)
        self.session = None
        self.user = None

    def get_signing_message(self):
        """Get signing message with nonce."""
        response = requests.get(f"{self.API_URL}/auth/signing-message")
        response.raise_for_status()
        return response.text

    def sign_message(self, message):
        """Sign message with wallet."""
        signable = encode_defunct(text=message)
        signed = self.account.sign_message(signable)
        return '0x' + signed.signature.hex()

    def login(self):
        """Authenticate and store session."""
        # Get and sign message
        message = self.get_signing_message()
        signature = self.sign_message(message)

        # Prepare headers
        headers = {
            'x-account': self.account.address,
            'x-signing-message': '0x' + message.encode('utf-8').hex(),
            'x-signature': signature,
            'Content-Type': 'application/json'
        }

        # Login
        response = requests.post(
            f"{self.API_URL}/auth/login",
            headers=headers,
            json={"client": "eoa"}
        )
        response.raise_for_status()

        self.session = response.cookies.get('limitless_session')
        self.user = response.json()

        return self.session, self.user

    def verify(self):
        """Verify current session is valid."""
        if not self.session:
            return False

        response = requests.get(
            f"{self.API_URL}/auth/verify-auth",
            cookies={"limitless_session": self.session}
        )
        return response.ok

    def logout(self):
        """End session."""
        if not self.session:
            return

        requests.post(
            f"{self.API_URL}/auth/logout",
            cookies={"limitless_session": self.session}
        )
        self.session = None
        self.user = None

    def get_cookies(self):
        """Get cookies dict for requests."""
        return {"limitless_session": self.session} if self.session else {}
```

## Smart Wallet Support

For smart contract wallets, use `client: "smart_wallet"` and provide the smart wallet address.

```python
headers = {
    'x-account': signer_address,          # EOA signer
    'x-signing-message': hex_message,
    'x-signature': signature,
}

response = requests.post(
    f"{API_URL}/auth/login",
    headers=headers,
    json={
        "client": "smart_wallet",
        "smartWallet": smart_wallet_address   # Contract address
    }
)
```

## Migration from Cookie to API Key

If you're currently using cookie-based authentication, migrate by:

1. **Generate an API key** via the UI (profile menu → Api keys)
2. **Replace cookie header with API key header**:
```diff
# Before (deprecated)
- Cookie: limitless_session=your_session_token

# After
+ X-API-Key: lmts_your_key_here
```
3. **Remove session management code** - no more login flow or cookie handling needed

### Code Migration Example

```python
# BEFORE (deprecated cookie auth)
session_cookie, user = authenticate(private_key)
response = requests.get(
    f"{API_URL}/portfolio/positions",
    cookies={"limitless_session": session_cookie}
)

# AFTER (API key auth)
response = requests.get(
    f"{API_URL}/portfolio/positions",
    headers={"X-API-Key": "lmts_your_key_here"}
)
```

## Session Management (Deprecated)

> **Deprecated**: API key authentication does not require session management.

### Session Expiration

Sessions expire after inactivity. Handle 401 responses:

```python
def authenticated_request(url, session, auth):
    response = requests.get(url, cookies={"limitless_session": session})

    if response.status_code == 401:
        # Re-authenticate
        session, user = auth.login()
        response = requests.get(url, cookies={"limitless_session": session})

    return response
```

### Verify Before Requests

```python
def ensure_authenticated(auth):
    if not auth.verify():
        auth.login()
    return auth.session
```

## Security Best Practices

1. **Protect API keys**
   - Store in environment variables (`LIMITLESS_API_KEY`)
   - Never commit to version control
   - Rotate keys periodically via the Limitless Exchange UI

2. **Never expose private keys**
   - Use environment variables
   - Never commit to version control

3. **Validate addresses**
   - Use checksummed addresses (EIP-55)
   - Verify address format before sending

4. **Error handling**
   - Handle authentication failures gracefully
   - Implement retry logic for network issues

## Troubleshooting

### "Invalid API key"

- Verify the key starts with `lmts_`
- Check the key was copied correctly without extra whitespace
- Ensure the key has not been revoked in the UI

### "Invalid signature"

- Verify message encoding matches API expectation
- Check signature format has 0x prefix
- Ensure address is checksummed

### "Session not found" (Deprecated)

- Session may have expired
- Cookie may not be included in request
- Re-authenticate or migrate to API key authentication

### "Smart wallet required"

- Using `smart_wallet` client but missing address
- Provide `smartWallet` in request body
