# Accessing Historical Data for Backtesting

**Question**: Is there a way with the API to access the historical data for backtesting purposes?

## Answer

**Yes!** The Limitless API provides historical price data that can be used for backtesting trading strategies. The primary endpoint is `GET /markets/{slug}/historical-price`, which returns price time series data at various intervals.

## Primary Endpoint: Historical Price Data

### GET /markets/{slug}/historical-price

**Authentication**: None required

**Query Parameters**:

| Parameter | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| `from` | string | No | Start date (ISO 8601) | `2024-01-01T00:00:00Z` |
| `to` | string | No | End date (ISO 8601) | `2024-01-31T23:59:59Z` |
| `interval` | string | No | Time interval for data points | `1h`, `6h`, `1d`, `1w`, `1m`, `all` |

**Available Intervals**:

| Interval | Description | Use Case |
|----------|-------------|----------|
| `1h` | 1-hour candles | Short-term strategy testing |
| `6h` | 6-hour candles | Intraday strategies |
| `1d` | Daily candles | Swing trading strategies |
| `1w` | Weekly candles | Long-term trend analysis |
| `1m` | Monthly candles | Macro analysis |
| `all` | All available data | Maximum granularity |

## Basic Backtesting Example

```python
import requests
from datetime import datetime, timedelta

class BacktestDataFetcher:
    def __init__(self):
        self.base_url = "https://api.limitless.exchange"

    def get_historical_prices(self, market_slug, from_date=None, to_date=None, interval="1h"):
        """Fetch historical price data for a market."""
        params = {"interval": interval}

        if from_date:
            params["from"] = from_date
        if to_date:
            params["to"] = to_date

        response = requests.get(
            f"{self.base_url}/markets/{market_slug}/historical-price",
            params=params
        )
        response.raise_for_status()
        return response.json()

    def to_dataframe(self, price_data, outcome="YES"):
        """Convert API response to pandas DataFrame."""
        import pandas as pd

        # Find the outcome data
        outcome_data = None
        for item in price_data:
            if outcome.upper() in item['title'].upper():
                outcome_data = item
                break

        if not outcome_data:
            raise ValueError(f"Outcome '{outcome}' not found in data")

        df = pd.DataFrame(outcome_data['prices'])
        df['timestamp'] = pd.to_datetime(df['timestamp'])
        df = df.set_index('timestamp').sort_index()
        return df


# Usage
fetcher = BacktestDataFetcher()

# Get last 30 days of hourly data
price_data = fetcher.get_historical_prices(
    market_slug="btc-100k-2024",
    from_date="2024-01-01T00:00:00Z",
    to_date="2024-01-31T23:59:59Z",
    interval="1h"
)

# Convert to DataFrame for analysis
df = fetcher.to_dataframe(price_data, outcome="YES")
print(df.head())
```

**Example Output**:
```
                           price
timestamp
2024-01-01 00:00:00+00:00   0.45
2024-01-01 01:00:00+00:00   0.46
2024-01-01 02:00:00+00:00   0.47
2024-01-01 03:00:00+00:00   0.46
2024-01-01 04:00:00+00:00   0.48
```

## Complete Backtesting Framework

```python
import requests
import pandas as pd
from datetime import datetime

class SimpleBacktester:
    def __init__(self):
        self.base_url = "https://api.limitless.exchange"

    def fetch_market_history(self, market_slug, from_date, to_date, interval="1h"):
        """Fetch historical data for backtesting."""
        response = requests.get(
            f"{self.base_url}/markets/{market_slug}/historical-price",
            params={
                "from": from_date,
                "to": to_date,
                "interval": interval
            }
        )
        response.raise_for_status()

        data = response.json()

        # Parse YES and NO prices
        yes_prices = None
        no_prices = None

        for item in data:
            if "YES" in item['title'].upper():
                yes_prices = pd.DataFrame(item['prices'])
            elif "NO" in item['title'].upper():
                no_prices = pd.DataFrame(item['prices'])

        # Combine into single DataFrame
        if yes_prices is not None:
            yes_prices['timestamp'] = pd.to_datetime(yes_prices['timestamp'])
            yes_prices = yes_prices.rename(columns={'price': 'yes_price'})
            yes_prices = yes_prices.set_index('timestamp')

        if no_prices is not None:
            no_prices['timestamp'] = pd.to_datetime(no_prices['timestamp'])
            no_prices = no_prices.rename(columns={'price': 'no_price'})
            no_prices = no_prices.set_index('timestamp')

        df = yes_prices.join(no_prices, how='outer')
        return df.sort_index()

    def run_simple_momentum_backtest(self, df, lookback=5, threshold=0.05):
        """
        Simple momentum strategy backtest.

        Strategy: Buy YES when price momentum is positive and above threshold,
                  Sell when momentum turns negative.
        """
        df = df.copy()

        # Calculate momentum
        df['momentum'] = df['yes_price'].pct_change(lookback)

        # Generate signals
        df['signal'] = 0
        df.loc[df['momentum'] > threshold, 'signal'] = 1   # Buy
        df.loc[df['momentum'] < -threshold, 'signal'] = -1 # Sell

        # Calculate returns
        df['price_return'] = df['yes_price'].pct_change()
        df['strategy_return'] = df['signal'].shift(1) * df['price_return']

        # Calculate cumulative returns
        df['cumulative_return'] = (1 + df['strategy_return']).cumprod()
        df['buy_hold_return'] = (1 + df['price_return']).cumprod()

        return df

    def calculate_metrics(self, df):
        """Calculate backtest performance metrics."""
        strategy_returns = df['strategy_return'].dropna()

        total_return = df['cumulative_return'].iloc[-1] - 1
        buy_hold_return = df['buy_hold_return'].iloc[-1] - 1

        # Sharpe ratio (assuming hourly data, annualized)
        sharpe = (strategy_returns.mean() / strategy_returns.std()) * (8760 ** 0.5)

        # Win rate
        wins = (strategy_returns > 0).sum()
        losses = (strategy_returns < 0).sum()
        win_rate = wins / (wins + losses) if (wins + losses) > 0 else 0

        return {
            'total_return': f"{total_return:.2%}",
            'buy_hold_return': f"{buy_hold_return:.2%}",
            'sharpe_ratio': f"{sharpe:.2f}",
            'win_rate': f"{win_rate:.2%}",
            'total_trades': wins + losses
        }


# Usage
backtester = SimpleBacktester()

# Fetch historical data
df = backtester.fetch_market_history(
    market_slug="btc-100k-2024",
    from_date="2024-01-01T00:00:00Z",
    to_date="2024-01-31T23:59:59Z",
    interval="1h"
)

# Run backtest
results = backtester.run_simple_momentum_backtest(df, lookback=5, threshold=0.03)

# Calculate metrics
metrics = backtester.calculate_metrics(results)
print("Backtest Results:")
for key, value in metrics.items():
    print(f"  {key}: {value}")
```

**Example Output**:
```
Backtest Results:
  total_return: 12.45%
  buy_hold_return: 8.32%
  sharpe_ratio: 1.45
  win_rate: 54.23%
  total_trades: 142
```

## Fetching Data for Multiple Markets

```python
import requests
import time

def fetch_all_markets():
    """Get list of all active markets."""
    response = requests.get("https://api.limitless.exchange/markets")
    return response.json()

def build_historical_dataset(market_slugs, from_date, to_date, interval="1d"):
    """Build a dataset of historical prices for multiple markets."""
    all_data = {}

    for slug in market_slugs:
        try:
            response = requests.get(
                f"https://api.limitless.exchange/markets/{slug}/historical-price",
                params={
                    "from": from_date,
                    "to": to_date,
                    "interval": interval
                }
            )

            if response.status_code == 200:
                all_data[slug] = response.json()
                print(f"Fetched: {slug}")
            else:
                print(f"Failed: {slug} ({response.status_code})")

            # Rate limiting
            time.sleep(0.5)

        except Exception as e:
            print(f"Error fetching {slug}: {e}")

    return all_data


# Usage
markets = fetch_all_markets()
slugs = [m['slug'] for m in markets.get('markets', [])[:10]]  # First 10 markets

dataset = build_historical_dataset(
    market_slugs=slugs,
    from_date="2024-01-01T00:00:00Z",
    to_date="2024-01-31T23:59:59Z",
    interval="1d"
)
```

## Additional Data Sources for Backtesting

### Trade Events

The `/markets/{slug}/events` endpoint provides recent trade data:

```python
response = requests.get(
    "https://api.limitless.exchange/markets/btc-100k-2024/events",
    params={"limit": 100}
)
events = response.json()

# Filter for trades only
trades = [e for e in events.get('events', []) if e['type'] == 'trade']
```

### Your Own Trade History

If you want to analyze your past trading performance:

```python
response = requests.get(
    "https://api.limitless.exchange/portfolio/history",
    params={
        "from": "2024-01-01T00:00:00Z",
        "to": "2024-01-31T23:59:59Z",
        "page": 1,
        "limit": 100
    },
    cookies={"limitless_session": session_cookie}
)
my_trades = response.json()
```

## Limitations for Backtesting

| Limitation | Description | Workaround |
|------------|-------------|------------|
| **Minimum 1h granularity** | No tick-level or minute data | Use `1h` interval for highest resolution |
| **No raw orderbook history** | Only current orderbook available | Cannot simulate order execution |
| **No official data retention policy** | Unknown historical depth | Test date ranges to find available data |
| **No paper trading** | No simulation endpoints | Build your own paper trading logic |

## Tips for Effective Backtesting

1. **Account for slippage**: Real trades may execute at worse prices than historical mid-prices suggest

2. **Consider liquidity**: Check the orderbook endpoint to understand typical market depth

3. **Use appropriate intervals**:
   - `1h` for short-term strategies
   - `1d` for swing trading
   - `1w` or `1m` for trend following

4. **Validate data quality**: Check for gaps in the timestamp series

5. **Out-of-sample testing**: Reserve recent data for validation, don't overfit to historical patterns

## Related

- [Trading Endpoints](../endpoints/trading.md) - Full trading API documentation
- [Markets Endpoints](../endpoints/markets.md) - Market discovery and metadata
- [Portfolio History](../endpoints/portfolio.md#get-portfoliohistory) - Your trade history
