# Signal Bots

## Overview

The Signals interface in ProfitView lets you build event-driven trading strategies in Python. Your strategy reacts to live market events and sends trading intent using `self.signal(...)`. Execution, risk management, and exchange connectivity are handled by ProfitView.

A Signal bot is built on an event-driven architecture. Market events trigger callback functions. These callbacks may handle public market data (quotes, trades, candles) and generate signals that downstream bots or processes can consume.

<div class="centered" style="margin-bottom: 20px;">
    <img src="/assets/images/trading-bot-architecture-white.png" alt="Signal Bot Architecture">
</div>


## Getting Started

- Sign up and verify your email address
- Choose a plan (Hobby $29, Active Trader $59, or Professional $299)
- **Signals** is Beta: request access from [ProfitView management](mailto:support@profitview.net).
- Open the Signals IDE: `https://profitview.net/trading/signals`
- Create a new file; ensure your main class is defined (see below)

**Note:** your script must define the main `Signals` class for the runtime to load it so that Trading Bots will function.


## Base Class

The base class `Link` provides helper properties and methods you can use in your Signal bot.

### Properties

> #### venues → `dict`
>>> Exchange venues configured for your account
>>> 
>>> **Example:**
>>> ```python
>>> print(self.venues)
>>> # {'BitMEX': {'id': 'abc123', 'kind': 'real', 'name': 'BitMEX', 'src': 'bitmex'}}
>>> ```

> #### trades → `dict`
>>> Latest trade data by exchange and symbol
>>> 
>>> **Example:**
>>> ```python
>>> latest_trade = self.trades['bitmex']['XBTUSD']
>>> print(f"Last trade: {latest_trade['price']} @ {latest_trade['size']}")
>>> ```

> #### quotes → `dict`
>>> Latest quote data by exchange and symbol
>>> 
>>> **Example:**
>>> ```python
>>> bid, ask = self.quotes['bitmex']['XBTUSD']['bid'][0], self.quotes['bitmex']['XBTUSD']['ask'][0]
>>> spread = ask - bid
>>> ```

> #### instruments → `dict`
>>> Instrument metadata by exchange and symbol
>>> 
>>> **Example:**
>>> ```python
>>> instrument = self.instruments['bitmex']['XBTUSD']
>>> print(f"Tick size: {instrument['tickSize']}, Min size: {instrument['lotSize']}")
>>> ```

> #### market_streams → `dict`
>>> Active market data subscriptions by exchange
>>> 
>>> **Example:**
>>> ```python
>>> print(self.market_streams)
>>> # {'bitmex': ['XBTUSD', 'ETHUSD'], 'coinbasepro': ['BTC-USD']}
>>> ```

> #### health → `dict`
>>> Connection health status for public and private feeds
>>> 
>>> **Example:**
>>> ```python
>>> health = self.health
>>> for feed in health['public']:
>>>     if not feed['live']:
>>>         logger.warning(f"Feed {feed['src']} is disconnected")
>>> ```

### Time Properties

> #### now → `datetime.datetime`
>>> Current UTC time
>>> 
>>> **Example:**
>>> ```python
>>> current_time = self.now
>>> print(f"Current time: {current_time}")
>>> ```

> #### unix_now → `int`
>>> Current unix time in seconds
>>> 
>>> **Example:**
>>> ```python
>>> timestamp = self.unix_now
>>> print(f"Unix timestamp: {timestamp}")
>>> ```

> #### epoch_now → `int`
>>> Current epoch time in milliseconds
>>> 
>>> **Example:**
>>> ```python
>>> epoch_ms = self.epoch_now
>>> print(f"Epoch milliseconds: {epoch_ms}")
>>> ```

> #### second → `float`
>>> Current second within the minute (high precision)
>>> 
>>> **Example:**
>>> ```python
>>> if self.second > 30:
>>>     logger.info("More than half way through the minute")
>>> ```

> #### iso_now → `str`
>>> Current UTC ISO 8601 string
>>> 
>>> **Example:**
>>> ```python
>>> iso_time = self.iso_now
>>> print(f"ISO time: {iso_time}")
>>> ```

### Methods

> #### get_bars(src, sym, level) → `Bars`
>>> Get candle data object for specified exchange, symbol, and timeframe
>>> 
>>> **Example:**
>>> ```python
>>> bars = self.get_bars('bitmex', 'XBTUSD', '1m')
>>> if bars:
>>>     latest_close = bars.closes[-1]
>>>     rsi = talib.RSI(bars.closes, timeperiod=14)[-1]
>>> ```

> #### last_trade(src, sym, key=None) → `dict` or `value`
>>> Get latest trade data for exchange/symbol, optionally filter by key
>>> 
>>> **Example:**
>>> ```python
>>> # Get full trade data
>>> trade = self.last_trade('bitmex', 'XBTUSD')
>>> 
>>> # Get specific field
>>> price = self.last_trade('bitmex', 'XBTUSD', 'price')
>>> size = self.last_trade('bitmex', 'XBTUSD', 'size')
>>> ```

> #### last_quote(src, sym, key=None) → `dict` or `value`
>>> Get latest quote data for exchange/symbol, optionally filter by key
>>> 
>>> **Example:**
>>> ```python
>>> # Get full quote data
>>> quote = self.last_quote('bitmex', 'XBTUSD')
>>> 
>>> # Get specific bid/ask prices
>>> bid_price = self.last_quote('bitmex', 'XBTUSD', 'bid')[0]
>>> ask_price = self.last_quote('bitmex', 'XBTUSD', 'ask')[0]
>>> ```

> #### publish(topic, data=None) → `None`
>>> Publish message to private websocket stream (see [WebSocket Documentation](https://profitview.net/docs/websocket))
>>> 
>>> **Example:**
>>> ```python
>>> # Simple status update
>>> self.publish('status', {'active': True, 'timestamp': self.epoch_now})
>>> 
>>> # Strategy metrics
>>> self.publish('metrics', {
>>>     'rsi': 65.5,
>>>     'signal_strength': 0.8,
>>>     'position': 'long'
>>> })
>>> ```

> #### signal(src, sym, **intents) → `None`
>>> Send trading intent signals to execution system (see [Signal Documentation](https://profitview.net/docs/signals))
>>> 
>>> **Example:**
>>> ```python
>>> # Position signal
>>> self.signal('bitmex', 'XBTUSD', size=0.5)
>>> 
>>> # Grid signals
>>> self.signal('bitmex', 'XBTUSD', mid=30000)
>>> self.signal('bitmex', 'XBTUSD', bid=29500)
>>> 
>>> # Market maker
>>> self.signal('bitmex', 'XBTUSD', quote=[29950, 30050])
>>> ```

> #### stop_trading(msg=None) → `None`
>>> Terminate the trading script
>>> 
>>> **Example:**
>>> ```python
>>> # Emergency stop
>>> if error_condition:
>>>     self.stop_trading("Critical error detected")
>>> 
>>> # Scheduled shutdown
>>> if self.now.hour == 23:
>>>     self.stop_trading("End of trading day")
>>> ```

For additional documentation on these methods, see the [Trading docs](https://profitview.net/docs/trading/#base-class).

<div style="margin-bottom: 24px"></div>

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
    'price': 121611.5,          # price of matched order
    'size': 200.0,              # size of matched order
    'time': 1678364683876       # epoch timestamp from exchange
}
```

### Candle Update

Receive rolling OHLCV bars for a given timeframe.

Called on each trade thereby updating candle data: `candle_update_XX()` for any XX in

`1m`, `3m`, `5m`, `15m`, `30m`, `1h`, `2h`, `3h`, `4h`, `6h`, `12h`, `1d` and `1w`

```python
def candle_update_1m(self, src: str, sym: str, bars)
```

- `bars.opens`, `bars.highs`, `bars.lows`, `bars.closes`, `bars.volumes` each hold 360 elements
- `bars[i]` returns `{time, open, high, low, close, volume}`
- Called on every trade, not just at candle boundaries
- `bars.closes[-1]` and `bars.volumes[-1]` update on every trade; `bars.highs[-1]` and `bars.lows[-1]` may update intra-candle; `bars.opens[-1]` updates once per candle
- `bars.closes` format is compatible with TA-Lib functions

Example RSI signal:

```python
import talib

def candle_update_1m(self, src, sym, bars):
    rsi = talib.RSI(bars.closes, timeperiod=14)[-1]
    if rsi > 70:
        self.signal(src, sym, size=-0.5)
```

### Initialization

- `on_start(self)` — runs once when `Link` infrastructure is ready. Note **callbacks may trigger before this**.
- `__init__(self, *args, **kwargs)` — if you override this, you must call `Link.__init__(self, *args, **kwargs)` at some point in `__init__()` to initialize event handling.


## Scheduled Tasks (@cron)

Use the `@cron` decorator to run methods on a schedule. Decorate no-arg instance methods of your `Signals` class.

- Exactly one option is required:
  - `every=<seconds>`: positive int/float seconds
  - `on='<period>'`: one of `1m, 3m, 5m, 15m, 30m, 1h, 2h, 3h, 4h, 6h, 12h, 1d, 1w`
  - `expr='<cron expression>'`: standard cron syntax (UTC). See [UNIX cron format](https://www.ibm.com/docs/en/db2/11.5.x?topic=task-unix-cron-format).

Behavior:
- Runs in background threads; long tasks do not block others
- First run triggers immediately after script start, then follows schedule
- Methods must only accept `self` (no additional parameters)

Examples:

```python
from profitview import Link, logger, cron

class Signals(Link):
  @cron.run(every=60)
  def refresh(self):
    logger.info('refreshing...')

  @cron.run(on='1m')
  def minutely(self):
    self.signal('bitmex', 'XBTUSD', size=0.1)

  @cron.run(expr='0 0 * * *')  # every day at 00:00 UTC
  def daily(self):
    ...
```


## Emitting Signals

Use `self.signal(src, sym, bid=...)` (or similar) to emit signals for downstream consumers.

Examples:

```python
# Position signal
self.signal(src, sym, size=0.5)

# Neutral grid anchor
self.signal(src, sym, mid=30000)

# Maker quote both sides
self.signal(src, sym, quote=[2300, 2305])
```


## Bot Types & Signal Formats

Choose a bot type that matches the signal format you will emit. Parameters are configured in the bot creation UI and are not accessible from Python code.

### Position

- Signal: `self.signal(src, sym, size=value)`
- Description: Expresses directional intent between -1.0 (max short) and 1.0 (max long). Execution respects the user's position limits.
- Mandatory params: `max_position_size`
- Optional params: `stop_loss_pct`, `take_profit_pct`, `close_on_stop`, `trading_start_time`, `trading_stop_time`, `order_fill_delay`, `order_min_pct_change`, `limit_post_only`

### Neutral Grid

- Signal: `self.signal(src, sym, mid=value)`
- Description: Places symmetric grid orders above and below mid price.
- Mandatory params: `grid_count`, `grid_spread`
- Optional params: `min_spread`, `max_order_size`, `order_refresh_delay`, `take_profit_pct`, `stop_loss_pct`, `trail_stop_loss_pct`, `trail_stop_loss_entry_pct`, `limit_post_only`

### Short Grid

- Signal: `self.signal(src, sym, ask=value)`
- Description: Sell-side grid above ask price.
- Mandatory params: `grid_count`, `grid_spread`, `grid_side`
- Optional params: same as Neutral Grid

### Long Grid

- Signal: `self.signal(src, sym, bid=value)`
- Description: Buy-side grid below bid price.
- Mandatory params: `grid_count`, `grid_spread`, `grid_side`
- Optional params: same as Neutral Grid

### Market Maker

- Signal: `self.signal(src, sym, quote=value)`
- Description: Makes a market around a quote price.
- Mandatory params: `max_position_size`
- Optional params: `min_spread`, `limit_post_only`, `order_refresh_delay`, `order_min_pct_change`


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

### Test with wscat

1. Install wscat (Node.js):

```bash
npm i -g wscat
```

2. Connect to your private stream:

```bash
wscat -c "wss://profitview.net/stream?token=YOUR_API_KEY"
```
- **Note** - API Key found in SETTINGS -> Account
- Tip: avoid putting secrets in shell history:

```bash
export PV_TOKEN=YOUR_API_KEY
wscat -c "wss://profitview.net/stream?token=$PV_TOKEN"
```

3. From your Signal bot, publish a test message:

```python
self.publish("test", {"hello": "world"})
```

4. In wscat, you should see:

```json
{ "type": "test", "data": { "hello": "world" } }
```

Once connected you will receive messages in the format:

```python
< { "type": your_topic, "data": your_data }
```


## Create Webhooks

Use HTTP routes to receive external inputs (e.g., TradingView alerts). See [Create Webhooks](https://profitview.net/docs/trading#create-webhooks) in the Trading docs for details.

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
    <img src="/assets/images/create-bot.png" alt="Create Signal Bot">
</div>

1. Click **Bots** in the Signals IDE (bottom left of code window)
2. Click **Create** button
3. **Select Symbol** (currently there can only be one symbol traded per Bot)
4. **Select Bot Type** (see above for details)
   - **Position**
   - **Grid** (**Neutral**, **Long** or **Short**)
   - **Market Maker**
5. **Select Parameters**
   - optionally specify initial values
   - optionally specify **Defaults**
   - optionally make a parameter **Private** (set but not visible to the user)
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

Use the [Supported Exchanges list](https://profitview.net/docs/trading/#supported-exchanges) in the Trading docs to obtain valid `src` strings for `self.signal(...)`.


<!--
Notes for editors:
- Keep heading order and formatting consistent with the Trading Bots doc.
- Replace or localize image paths as needed.
-->



