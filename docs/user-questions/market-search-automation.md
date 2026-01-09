# Automating Market Search by Category for AI Analysis

## Symptoms

- You want to analyze all markets in a category (e.g., "off-the-pitch") to determine which can be resolved
- Sending a link directly to ChatGPT doesn't work because it can't access client-side rendered pages
- You have to manually copy/paste markets or create a spreadsheet as a workaround
- Markets update frequently but your spreadsheet gets out of sync

## Possible Causes

### 1. Web pages are client-side rendered

The Limitless Exchange UI (e.g., `https://limitless.exchange/pro/cat/off-the-pitch`) renders entirely in the browser using JavaScript. Web crawlers and API-based readers (like ChatGPT's Web Browser) see an empty document with no market data.

### 2. No built-in export from the UI

There's no "Export CSV" or "Copy Markets" button visible in the UI, so you're forced to copy markets manually one at a time.

### 3. Spreadsheets become stale

Even if you export a spreadsheet, markets update constantly. New markets are added, old ones resolve—making your spreadsheet outdated quickly.

## Solution: Use the API Endpoint

The Limitless Exchange API provides a direct endpoint to fetch all markets in a category as structured JSON data.

### Get Markets from Any Category

```python
import requests

# Fetch all active markets from category 50 (off-the-pitch)
response = requests.get(
    "https://api.limitless.exchange/markets/active/50",
    params={"limit": 100}  # Adjust limit as needed
)

markets = response.json()

# Now you have structured data:
# markets['markets'] contains list of market objects with:
# - title: Market question
# - description: Full market details
# - status: FUNDED, RESOLVED, etc.
# - deadline: Resolution deadline
# - winning_index: Outcome (if resolved)
# - volume: Trading volume
# - liquidity: Available liquidity

print(markets)
```

### Feed to ChatGPT

**Option A: Copy and paste the JSON**
1. Run the Python script above to get the JSON
2. Copy the entire JSON response
3. Paste into ChatGPT with your request:
   ```
   Go through all these markets and analyze which ones can already be resolved based on their titles and descriptions.

   [PASTE JSON HERE]
   ```

**Option B: Use the API URL directly (if ChatGPT can access it)**
```
https://api.limitless.exchange/markets/active/50?limit=100

Can you analyze all these markets and tell me which ones can be resolved?
```

**Option C: Format for readability**
```python
import requests
import json

response = requests.get(
    "https://api.limitless.exchange/markets/active/50",
    params={"limit": 100}
)

markets = response.json()

# Format nicely for ChatGPT
output = "Markets to analyze:\n\n"
for market in markets['markets']:
    output += f"Title: {market['title']}\n"
    output += f"Description: {market.get('description', 'N/A')}\n"
    output += f"Status: {market['status']}\n"
    output += f"Deadline: {market.get('deadline', 'N/A')}\n"
    output += f"---\n"

print(output)
# Copy this output and paste into ChatGPT
```

## Why This Works Better Than Spreadsheets

| Method | Real-time | Complete Data | Scalable | Current Status |
|--------|-----------|---------------|----------|---|
| Spreadsheet | ❌ Manual | ✅ Yes | ❌ Hard | ❌ Stale |
| Screenshots | ❌ Manual | ✅ Yes | ❌ Very hard | ❌ Manual |
| API | ✅ Automatic | ✅ Yes | ✅ Easy | ✅ Always fresh |

## Finding Your Category ID

If you don't know your category ID:

```python
import requests

# Get count of markets per category
response = requests.get(
    "https://api.limitless.exchange/markets/categories/count"
)

categories = response.json()
print(categories)

# Look for your category and use its ID
# Example: Category 50 is "off-the-pitch"
```

## Tips

1. **Adjust the limit**: The `limit` parameter controls how many markets are returned. Start with 100, increase if you have more markets.

2. **Pagination**: If you have more than 100 markets, use pagination:
   ```python
   response = requests.get(
       "https://api.limitless.exchange/markets/active/50",
       params={"page": 1, "limit": 100}
   )
   ```

3. **Auto-refresh**: Wrap this in a scheduled task to refresh the data before each ChatGPT analysis:
   ```python
   import schedule
   import time

   def fetch_and_analyze():
       # Fetch data
       # Save to file or database
       # Use in ChatGPT analysis
       pass

   schedule.every(1).hours.do(fetch_and_analyze)
   ```

4. **Filter by status**: The API returns all active markets. If you need only unresolved ones, filter client-side:
   ```python
   unresolved = [m for m in markets['markets'] if m['status'] == 'FUNDED']
   ```

## Related

- [Markets Endpoints](../endpoints/markets.md) - Full market browsing API
- [Market Schemas](../schemas/market.md) - Complete field definitions
- [Authentication Guide](../guides/authentication.md) - For protected endpoints
