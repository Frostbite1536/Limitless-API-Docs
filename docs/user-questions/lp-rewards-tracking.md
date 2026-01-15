# Tracking LP Rewards

**Question**: Is there a way to know if I'm winning LP rewards and how much?

## Answer

**Yes!** The Limitless API provides several endpoints to track your LP (Liquidity Provider) rewards, including current earnings, historical data, and epoch-by-epoch breakdowns.

## Endpoints for LP Rewards Tracking

### 1. GET /portfolio/positions (Authenticated)

Returns your portfolio positions with a rewards summary included.

```python
import requests

response = requests.get(
    "https://api.limitless.exchange/portfolio/positions",
    cookies={"limitless_session": session_cookie}
)
data = response.json()

# Access rewards summary
rewards = data['rewards']
print(f"Total Points: {rewards['totalPoints']}")

# Check LP tokens in AMM positions
for amm_pos in data['amm']:
    lp_tokens = amm_pos.get('lpTokens', '0')
    if int(lp_tokens) > 0:
        print(f"Market: {amm_pos['market']['title']}")
        print(f"LP Tokens: {int(lp_tokens) / 1e6}")
```

**Response Structure**:
```json
{
  "rewards": {
    "totalPoints": 15000,
    "epochs": [
      {
        "epoch": 1,
        "points": 5000,
        "tradingVolume": "50000000000"
      }
    ]
  },
  "amm": [
    {
      "market": {...},
      "lpTokens": "10000000000"
    }
  ],
  "clob": [...]
}
```

### 2. GET /portfolio/points (Authenticated)

Get detailed points/rewards breakdown by epoch, including liquidity provided.

```python
response = requests.get(
    "https://api.limitless.exchange/portfolio/points",
    cookies={"limitless_session": session_cookie}
)
points_data = response.json()

print(f"Total Points: {points_data['totalPoints']}")

for epoch in points_data['epochs']:
    print(f"\nEpoch {epoch['epoch']}:")
    print(f"  Points: {epoch['points']}")
    print(f"  Trading Volume: ${int(epoch['tradingVolume']) / 1e6:,.2f}")
    print(f"  Liquidity Provided: ${int(epoch['liquidityProvided']) / 1e6:,.2f}")
```

**Response Structure**:
```json
{
  "totalPoints": 15000,
  "epochs": [
    {
      "epoch": 1,
      "points": 5000,
      "tradingVolume": "50000000000",
      "liquidityProvided": "10000000000",
      "startDate": "2024-01-01T00:00:00.000Z",
      "endDate": "2024-01-07T00:00:00.000Z"
    },
    {
      "epoch": 2,
      "points": 10000,
      "tradingVolume": "100000000000",
      "liquidityProvided": "25000000000",
      "startDate": "2024-01-08T00:00:00.000Z",
      "endDate": "2024-01-14T00:00:00.000Z"
    }
  ]
}
```

## Key Fields Explained

| Field | Description |
|-------|-------------|
| `totalPoints` | Your cumulative points across all epochs |
| `points` | Points earned in a specific epoch |
| `tradingVolume` | Your trading volume in USDC (6 decimals) |
| `liquidityProvided` | Amount of liquidity you provided (6 decimals) |
| `lpTokens` | Your LP token balance in AMM positions |
| `startDate` / `endDate` | Epoch time boundaries |

## Complete Example: LP Rewards Dashboard

```python
import requests

class LPRewardsTracker:
    def __init__(self, session_cookie):
        self.session_cookie = session_cookie
        self.base_url = "https://api.limitless.exchange"

    def get_rewards_summary(self):
        """Get overall rewards summary."""
        response = requests.get(
            f"{self.base_url}/portfolio/positions",
            cookies={"limitless_session": self.session_cookie}
        )
        response.raise_for_status()
        return response.json()['rewards']

    def get_epoch_breakdown(self):
        """Get detailed epoch-by-epoch breakdown."""
        response = requests.get(
            f"{self.base_url}/portfolio/points",
            cookies={"limitless_session": self.session_cookie}
        )
        response.raise_for_status()
        return response.json()

    def get_lp_positions(self):
        """Get all LP token positions."""
        response = requests.get(
            f"{self.base_url}/portfolio/positions",
            cookies={"limitless_session": self.session_cookie}
        )
        response.raise_for_status()
        data = response.json()

        lp_positions = []
        for amm_pos in data.get('amm', []):
            lp_tokens = amm_pos.get('lpTokens', '0')
            if int(lp_tokens) > 0:
                lp_positions.append({
                    'market': amm_pos['market']['title'],
                    'slug': amm_pos['market']['slug'],
                    'lpTokens': int(lp_tokens) / 1e6
                })
        return lp_positions

    def print_dashboard(self):
        """Print a complete LP rewards dashboard."""
        # Get all data
        rewards = self.get_rewards_summary()
        epochs = self.get_epoch_breakdown()
        lp_positions = self.get_lp_positions()

        print("=" * 50)
        print("LP REWARDS DASHBOARD")
        print("=" * 50)

        # Summary
        print(f"\nTotal Points: {rewards['totalPoints']:,}")

        # LP Positions
        if lp_positions:
            print(f"\nActive LP Positions ({len(lp_positions)}):")
            for pos in lp_positions:
                print(f"  - {pos['market']}: {pos['lpTokens']:,.2f} LP tokens")
        else:
            print("\nNo active LP positions")

        # Epoch breakdown
        print(f"\nEpoch Breakdown ({len(epochs['epochs'])} epochs):")
        print("-" * 50)

        total_lp = 0
        for epoch in epochs['epochs']:
            lp_provided = int(epoch.get('liquidityProvided', 0)) / 1e6
            total_lp += lp_provided
            print(f"Epoch {epoch['epoch']}:")
            print(f"  Points:     {epoch['points']:,}")
            print(f"  Volume:     ${int(epoch['tradingVolume']) / 1e6:,.2f}")
            print(f"  LP Amount:  ${lp_provided:,.2f}")

        print("-" * 50)
        print(f"Total Liquidity Provided: ${total_lp:,.2f}")


# Usage
tracker = LPRewardsTracker(session_cookie="your_session_cookie")
tracker.print_dashboard()
```

**Example Output**:
```
==================================================
LP REWARDS DASHBOARD
==================================================

Total Points: 15,000

Active LP Positions (2):
  - Will BTC hit $100k?: 5,000.00 LP tokens
  - ETH price at end of month: 2,500.00 LP tokens

Epoch Breakdown (2 epochs):
--------------------------------------------------
Epoch 1:
  Points:     5,000
  Volume:     $50,000.00
  LP Amount:  $10,000.00
Epoch 2:
  Points:     10,000
  Volume:     $100,000.00
  LP Amount:  $25,000.00
--------------------------------------------------
Total Liquidity Provided: $35,000.00
```

## Understanding Rewards

### How Points Are Earned

Points are typically earned through:
1. **Trading volume** - Buying and selling outcome tokens
2. **Liquidity provision** - Adding liquidity to AMM pools
3. **Market participation** - Active engagement in markets

### Epoch System

Rewards are tracked in **epochs** (time periods). Each epoch tracks:
- Points earned during that period
- Trading volume contributed
- Liquidity provided

### LP Tokens

When you provide liquidity to an AMM pool, you receive **LP tokens** representing your share of the pool. These are visible in the `lpTokens` field of AMM positions.

## Tips

1. **Check regularly** - Rewards accrue over time, so check periodically
2. **Track by epoch** - Use epoch data to see how your rewards grow
3. **Monitor LP positions** - The `lpTokens` field shows your active liquidity contributions
4. **All values in 6 decimals** - Divide by 1e6 to get human-readable USDC amounts

## Related

- [Portfolio Endpoints](../endpoints/portfolio.md) - Full portfolio API documentation
- [Portfolio Schemas](../schemas/portfolio.md) - Data structure reference
- [Claiming Rewards After Close](claim-rewards-after-close.md) - How to redeem winnings
