# Authentication Endpoints

The Limitless Exchange uses wallet-based authentication with EIP-712 signatures. Users sign a message with their Ethereum wallet to prove ownership.

## Authentication Flow

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

## Security Notes

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
