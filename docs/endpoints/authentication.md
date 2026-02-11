# Authentication Endpoints

The Limitless Exchange supports two authentication methods for API access.

## Authentication Methods

| Method | Header | Status |
|--------|--------|--------|
| API Key | `X-API-Key: lmts_...` | **Required** for programmatic access |
| Cookie Session | `Cookie: limitless_session=...` | **Deprecated** (removal imminent) |

> **DEPRECATION NOTICE**: Cookie-based session authentication is deprecated and will be removed within weeks. Please migrate to API keys immediately. See [Migration Guide](#migration-from-cookie-to-api-key) below.

## API Key Authentication (Recommended)

API keys are the required method for programmatic access to the Limitless Exchange API.

### Getting an API Key

API keys can only be created via the Limitless Exchange UI:

1. Log in to [limitless.exchange](https://limitless.exchange) using your wallet
2. Click your profile menu (top right)
3. Select "Api keys"
4. Generate a new key

### Using Your API Key

Include in all requests via the `X-API-Key` header:

```bash
# REST API
curl -H "X-API-Key: lmts_your_key_here" https://api.limitless.exchange/markets

# WebSocket - pass X-API-Key header during connection handshake
```

```python
import requests

# All authenticated requests use the X-API-Key header
response = requests.get(
    "https://api.limitless.exchange/portfolio/positions",
    headers={"X-API-Key": "lmts_your_key_here"}
)
```

## Cookie-Based Authentication (Deprecated)

> **WARNING**: Cookie-based session authentication is deprecated and will be removed within weeks. Migrate to API keys immediately.

### Legacy Authentication Flow

1. Request a signing message with nonce
2. Sign the message with your wallet
3. Submit signature to login endpoint
4. Receive session cookie for authenticated requests

## Endpoints

### GET /auth/signing-message

Get a signing message containing a unique nonce for authentication.

**Authentication**: None required

**Parameters**: None

**Response** (200):
```json
"Welcome to Limitless.exchange! Please sign this message to verify your identity.\n\nNonce: 0xa1b2c3d4e5f67890..."
```

**Usage**:
```python
import requests

response = requests.get("https://api.limitless.exchange/auth/signing-message")
signing_message = response.text
```

---

### POST /auth/login

Authenticate with a signed message and create a session.

**Authentication**: None required

**Headers** (Required):

| Header | Description | Example |
|--------|-------------|---------|
| `x-account` | User's Ethereum address | `0x1234...7890` |
| `x-signing-message` | Hex-encoded signing message | `0x57656c636f6d65...` |
| `x-signature` | Wallet signature of the message | `0xabc123...` |

**Request Body**:
```json
{
  "client": "eoa"  // or "smart_wallet"
}
```

**Response** (200):
```json
{
  "account": "0x1234567890123456789012345678901234567890",
  "displayName": "0x1234...7890",
  "smartWallet": "0x0987654321098765432109876543210987654321",
  "client": "eoa"
}
```

**Response Headers**:
- `Set-Cookie`: Contains `limitless_session` cookie for subsequent requests

**Error Responses**:
- `400`: Missing or invalid parameters
- `500`: Error creating JWT token

**Usage**:
```python
from eth_account import Account
from eth_account.messages import encode_defunct

def login(private_key, signing_message):
    account = Account.from_key(private_key)
    message = encode_defunct(text=signing_message)
    signature = account.sign_message(message)

    headers = {
        'x-account': account.address,
        'x-signing-message': '0x' + signing_message.encode('utf-8').hex(),
        'x-signature': signature.signature.hex()
    }

    response = requests.post(
        "https://api.limitless.exchange/auth/login",
        headers=headers,
        json={"client": "eoa"}
    )

    session_cookie = response.cookies.get('limitless_session')
    return session_cookie, response.json()
```

---

### GET /auth/verify-auth

Verify if the current session is authenticated.

**Authentication**: Required (session cookie)

**Parameters**: None

**Response** (200):
```json
"0x1234567890123456789012345678901234567890"
```

**Error Response** (401):
```json
{
  "message": "The token cookie is required"
}
```

**Usage**:
```python
response = requests.get(
    "https://api.limitless.exchange/auth/verify-auth",
    cookies={"limitless_session": session_cookie}
)
```

---

### POST /auth/logout

Log out and clear the session cookie.

**Authentication**: Required (session cookie)

**Parameters**: None

**Response** (200):
```json
{
  "message": "Logged out successfully"
}
```

**Usage**:
```python
response = requests.post(
    "https://api.limitless.exchange/auth/logout",
    cookies={"limitless_session": session_cookie}
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

## Security Notes

### API Key Security
- Store API keys securely using environment variables
- Never commit API keys to version control
- Rotate keys periodically via the Limitless Exchange UI
- Each key provides full account access - treat it like a password

### Address Format
- Use checksummed Ethereum addresses (EIP-55)
- Example: `0x742d35Cc6634C0532925a3b844Bc454e4438f44e`

### Smart Wallet Support
- Smart contract wallets are supported
- Set `client: "smart_wallet"` in login body
- Provide smart wallet address in request

### Private Key Security
- Never expose private keys in code
- Use environment variables for sensitive data
- Never commit keys to version control
