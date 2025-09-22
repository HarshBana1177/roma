<p align="center">
  <span style="font-size:40px; font-weight:bold;">Build Your First ROMA Agent</span>
</p>

A clear, community-focused walkthrough to set up ROMA, build an Asset Snapshot Agent, and run it right in your browser. It taps the CoinPaprika public API for live prices and top movers, with a Binance fallback tool for quick exchange quotes.


🚀 What You’ll Actually Do 

- Install ROMA locally (Docker)

- Create a custom Asset Snapshot Agent

- Add four production‑ready tools:

   ✅ search_asset (smart ID lookup)

   ✅ get_price (live price + % change)

   ✅ top_movers (top gainers/losers)

   ✅ spot_price (Binance direct ticker price)

- Run it in your browser and chat with your agent
  

**Setup & Installation**

1. Clone ROMA
   ```bash
   git clone https://github.com/sentient-agi/ROMA.git
   cd ROMA
   ```
2. Setup with Docker
   ```bash
   ./setup.sh --docker
   cd docker
   docker compose up -d
   ```
 3. Open in Browser

    Go to 👉 http://localhost:3000

    You should now see the ROMA dashboard running locally.  

**Creating Your Agent**

Open this:
```bash
ROMA/agent_configs/agents.yaml
```
Add this in your yaml file:
```bash
agents:
- id: asset_snapshot
name: Asset Snapshot Agent
description: "Fetches prices, top movers, and quick market snapshots."
role: researcher
goals:
- Resolve user asset names/tickers to canonical IDs
- Provide live price with 24h change
- List top movers (gainers/losers)
- Provide Binance spot price if requested
model: openrouter/deepseek/deepseek-chat-v3.1:free
tools:
- search_asset
- get_price
- top_movers
- spot_price
```
If your file already starts with agents: make sure to add this as another list item with consistent indentation.

**Add Tools (CoinPaprika + Binance)**

First Create a new file:
```bash
ROMA/tools/coinpaprika.py
```
```bash
import requests
r.raise_for_status()
return r.json()
except Exception as e:
return {"error": str(e)}

def search_asset(query: str) -> str:
data = _http_get(f"{BASE}/search", params={"q": query, "c": "currencies", "limit": 5})
if isinstance(data, dict) and data.get("error"):
return f"Error: {data['error']}"
currencies = data.get("currencies", [])
if not currencies:
return f"No matches for '{query}'."
best = currencies[0]
return f"{best.get('id')}|symbol={best.get('symbol')}|name={best.get('name')}"

def get_price(asset: str) -> str:
coin_id = asset.lower().strip()
if "-" not in coin_id:
res = search_asset(coin_id)
if "|" in res:
coin_id = res.split("|")[0]
else:
return res
data = _http_get(f"{BASE}/tickers/{coin_id}")
if isinstance(data, dict) and data.get("error"):
return f"Error: {data['error']}"
quotes = data.get("quotes", {}).get("USD", {})
price = quotes.get("price")
chg24 = quotes.get("percent_change_24h")
vol24 = quotes.get("volume_24h")
mcap = quotes.get("market_cap")
if price is None:
return f"Could not fetch price for '{asset}'."
def _fmt(n):
return f"{n:,.2f}" if isinstance(n, (int, float)) else str(n)
return (
f"{data.get('name')} ({data.get('symbol')})\n"
f"Price: ${_fmt(price)}\n"
f"24h Change: {chg24:+.2f}%\n"
f"24h Volume: ${_fmt(vol24)}\n"
f"Market Cap: ${_fmt(mcap)}\n"
f"Source: CoinPaprika"
)

def top_movers(limit: int = 10) -> str:
tickers = _http_get(f"{BASE}/tickers")
if isinstance(tickers, dict) and tickers.get("error"):
return f"Error: {tickers['error']}"
rows = []
for t in tickers[:200]:
q = t.get("quotes", {}).get("USD", {})
chg = q.get("percent_change_24h")
price = q.get("price")
mcap = q.get("market_cap")
if price and chg is not None and mcap and mcap > 1e8:
rows.append({"name": t.get("name"), "symbol": t.get("symbol"), "chg": chg, "price": price})
if not rows:
return "No market data available."
gainers = sorted(rows, key=lambda r: r["chg"], reverse=True)[:limit]
losers = sorted(rows, key=lambda r: r["chg"])[:limit]
def block(title, items):
out = [title]
for r in items:
out.append(f"- {r['name']} ({r['symbol']}): {r['chg']:+.2f}% @ ${r['price']:,.4f}")
return "\n".join(out)
return block("Top Gainers (24h):", gainers) + "\n\n" + block("Top Losers (24h):", losers)
```
Then Create this:
```bash
ROMA/tools/binance_public.py
```
```bash
import requests

def spot_price(symbol: str = "BTCUSDT") -> str:
"""
Fetch spot price from Binance public API.
Example: spot_price("ETHUSDT")
"""
url = "https://api.binance.com/api/v3/ticker/price"
try:
r = requests.get(url, params={"symbol": symbol.upper()}, timeout=10)
r.raise_for_status()
data = r.json()
return f"{symbol.upper()} = {float(data['price']):,.2f} USDT (Binance)"
except Exception as e:
return f"Error fetching Binance price: {str(e)}"
```
Then Update :
```bash
ROMA/tools/__init__.py
```
And add at bottom:
```bash
from .coinpaprika import search_asset, get_price, top_movers
from .binance_public import spot_price
```

**Run & Test**

Restart ROMA so it picks up your new tools:
```bash
cd docker
docker compose down
docker compose up -d
```
Open 👉 http://localhost:3000 and select Asset Snapshot Agent.

Try prompts like:

- BTC price
- price of solana
- top movers
- spot_price("ETHUSDT")

**ROMA Flow Example:**
```bash
User Query
↓
Planner → interprets query (e.g., "price of solana")
↓
Executor → search_asset("solana") → coin ID
↓
Executor → get_price("solana") → live price + stats
↓
Executor → spot_price("BTCUSDT") → Binance quote
↓
Aggregator → returns final response
```

**Next Steps that you can do**

- Add a news summarizer tool via RSS feeds

- Experiment with multi-agent flows: Data Collector → Analyst → Reporter

- Add a lightweight frontend widget to embed agent responses

**✨ Start small, build bold.. Let your first ROMA agent spark bigger ideas.**


