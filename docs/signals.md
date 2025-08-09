# Signal Bots

## Overview

A Signal bot is built on an event-driven architecture. Market events trigger callback functions. These callbacks may handle public market data (quotes, trades, candles) and generate signals that downstream bots or processes can consume.

<div class="centered" style="margin-bottom: 20px;">
    <img src="/assets/images/trading-bot-architecture-white.png" alt="Signal Bot Architecture">
</div>

## Getting Started

- Sign up for a plan (Hobby $29, Active Trader $59, or Professional $299)
- Open the Signals IDE: `https://profitview.net/trading/signals`
- Create a new file; ensure your main class is defined (see below)

**Note:** your script must define the main class for the runtime to load it so that Trading Bots will function.


## Base Class

The base class `Link` provides helper properties and methods you can use in your Signal bot.

[If desired, copy the same property/method list as Trading bots, or link to it.]

<div style="margin-bottom: 24px"></div>

#### class Link

> now → `datetime.datetime`

>>> Current UTC time

> unix_now → `int`

>>> Current unix time in seconds

> epoch_now → `int`

>>> Current epoch time in milliseconds

> second → `float`

>>> Current second within the minute (high precision)

> iso_now → `str`

>>> Current UTC ISO 8601 string

[Add any additional helpers you want to document here.]


## Event Callbacks

Event callbacks are triggered automatically by live market data.

### Quote Update

Receive public market top-of-book quote updates for all subscribed symbols.

```python
def quote_update(self, src: str, sym: str, data: dict)
```

Example `data`:

```python
{
    'bid': [21601, 170000],     # bid price, bid size
    'ask': [21601.5, 223000],   # ask price, ask size
    'time': 1678364572949       # epoch timestamp from exchange
}
```

### Trade Update

Receive public market trade updates for all subscribed symbols.

```python
def trade_update(self, src: str, sym: str, data: dict)
```

Example `data`:

```python
{
    'side': 'Buy',              # side of taker in trade
    'price': 21611.5,           # price of matched order
    'size': 200.0,              # size of matched order
    'time': 1678364683876       # epoch timestamp from exchange
}
```

### Candle Update Xm 

Receive rolling OHLCV bars for a given timeframe.

```python
def candle_update_1m(self, src: str, sym: str, bars)
```

- `bars.opens`, `bars.highs`, `bars.lows`, `bars.closes`, `bars.volumes` each hold 360 items
- `bars[i]` returns `{time, open, high, low, close, volume}`
- Called on every trade, not just at candle boundaries

Example RSI signal:

```python
import talib

def candle_update_1m(self, src, sym, bars):
    rsi = talib.RSI(bars.closes, timeperiod=14)[-1]
    if rsi > 70:
        self.signal(src, sym, size=-0.5)
```


## Emitting Signals

Use `self.signal(src, sym, bid=...)` (or similar) to emit signals for downstream consumers.

Examples:

```python
# Position signal
self.signal(src, sym, size=0.5)

# Neutral grid anchor
self.signal(src, sym, mid=30000)

# Maker quote fair value
self.signal(src, sym, quote=30000)
```


## Create Websocket Feeds

Signal bots can publish private websocket messages for external dashboards/tools.

```python
self.publish(topic, data=None)
```

- `topic`: name of the websocket topic
- `data`: any JSON-serializable payload

Connect to your private websocket feed:

```json
wss://profitview.net/stream?token=YOUR_API_KEY
```

Once connected you will receive messages in the format:

```python
< { "type": your_topic, "data": your_data }
```


## Create Webhooks

Expose HTTP endpoints to interact with your Signal bot.

```python
@http.route
```

### GET requests

```python
@http.route
def get_status(self, data):
    return {"ok": True}
```

### POST requests

```python
@http.route
def post_webhook(self, data):
    price = data.get("price")
    if price:
        self.signal("bitmex", "XBTUSD", mid=price)
```


## Bot Creation Process in UI

<div class="centered" style="margin-bottom: 20px;">
    <img src="/assets/images/create-bot.png" alt="Signal Bot Architecture">
</div>


1. Click **Bots** in the Signals IDE (bottom left of code window)
2. Click **Create** button
3. **Select Symbol** (currently there can only be one symbol traded per Bot)
4. **Select Bot Type** (see below for details)
    - **Position**
    - **Grid** (**Neutral**, **Long** or **Short**)
    - **Market Maker**
5. **Select Parameters**
    - possibly specify initial values
    - possibly specify **Defaults**
    - possibly make the parameter **Private** (therefore set, but not visible to the user)
6. Fill in 
    - **Bot Name** (so you can find it in the **Bot Marketplace**)
    - **Description** (mandatory)
7. Click **Create** to save and deploy


## Glossary

### Enum Definitions

#### level

- `1m`, `3m`, `5m`, `15m`, `30m` — minute
- `1h`, `2h`, `3h`, `4h`, `6h`, `12h` — hour
- `1d` — day
- `1w` — week


### Installed Libraries

- `numpy`: array/matrix calculations
- `pandas`: data structures and analysis
- `scikit-learn`: ML library
- `scipy`: scientific computing
- `talib`: TA-Lib - technical analysis


### Supported Exchanges

[Mirror the table from Trading docs if desired, or link to it.]


<!--
Notes for editors:
- Image paths mirror the trading doc structure; update to actual asset paths as needed.
- Keep heading order and formatting consistent with the Trading Bots doc.
- Replace bracketed sections with your actual Signals content.
-->



