# pycloudrestapi — Complete Guide

**Northeast B2C Trading API — Python Client**

---

## What Is This?

`pycloudrestapi` is a Python library that connects your code to the **Northeast B2C REST API** — a trading system used by IBT and NEBSL brokers, powered by the Odin/Wave trading engine.

Think of it as a remote control for your trading account. Instead of logging into a trading app manually, you write Python code that logs in, places orders, checks your portfolio, and receives live prices — all automatically.

---

## What Can You Do With It?

- **Account** — Login, validate your session, check your balance and margins, and logout.
- **Orders** — Place fresh buy or sell orders, modify pending orders (change price or quantity), and cancel orders you no longer want.
- **Order Types** — Regular limit (`RL`), market (`RL-MKT`), stop loss limit (`SL`), stop loss market (`SL-MKT`).
- **Advanced Orders** — Cover orders, bracket orders, and multileg orders.
- **Reports** — Fetch your full order book (all orders placed today), trade book (all executed trades), and order history (status timeline of a single order).
- **Portfolio** — View your open positions, convert positions from intraday to delivery (or vice versa), and check your demat holdings.
- **Live Data** — Subscribe to touchline (LTP) and best-five (market depth) data via WebSocket.

---

## Installation

```bash
pip install nebsl-b2c-api
```

---

## Credentials You Need

Your broker provides these when they onboard you:

| Credential | Description |
|---|---|
| **Base URL** | The server address where all API calls go. Looks like a long AWS or cloud URL. |
| **API Key** | Your application's identity key. Sent in the login request body. |
| **x-api-key** | Sent as a header (`x-api-key`) on every request for authentication. |
| **Second AUTH** | A PAN number sent in the login request body to authenticate your app. |
| **User ID (UCC)** | Your Unique Client Code / broker account number. Example: `NE2247` |

> ⚠️ **Never share these credentials or put them in your code directly.** Store them in environment variables or a `.env` file.

---

## Quick Start

```python
from pycloudrestapi import IBTConnect

client = IBTConnect({
    "baseurl": "<your_base_url>",
    "api_key": "<your_api_key>",
    "x-api-key": "<your_x_api_key>",
    "second_auth": "<your_pan_number>",
    "debug": True
})
```

---

## How Login Works

Every API call (except login itself) requires an **access token**. You get this token by logging in. The library stores it automatically and sends it with every subsequent request.

- **User ID** — Your Unique Client Code
- **Password** — Your normal password

```python
# Login
user = client.login({
    "userId": "<your_user_id>",
    "password": "<your_password>"
})
print(user)
```

### Session Management

```python
# Validate session (keep alive) — call periodically
client.validateSession()

# Check available margins
client.balance()

# Logout when done
client.logout()
```

---

## Understanding Orders

### `scrip_token` — What Is It?

Every stock, futures contract, or options contract on an exchange has a unique number called a **scrip token**. The API uses this number to identify what you want to trade. For example, RELIANCE on NSE_EQ has token `2885`, and IDEA has token `14366`.

You get the token from your broker's scrip master file.

### `exchange` — Which Market?

| Value | Description |
|---|---|
| `NSE_EQ` | NSE Equity (stocks on NSE) |
| `BSE_EQ` | BSE Equity (stocks on BSE) |
| `NSE_FO` | NSE Futures & Options |
| `BSE_FO` | BSE Futures & Options |
| `MCX_FO` | MCX Futures (commodities) |
| `NSE_COMM` | NSE Commodity Derivatives |
| `BSE_COMM` | BSE Commodity Derivatives |

### `product_type` — How Long You Hold It?

| Value | Description |
|---|---|
| `INTRADAY` | Also called MIS. Must close before market closes (~3:20 PM for NSE equity). Lower margin required. |
| `DELIVERY` | Also called CNC. Stock goes into your demat account. Hold as long as you want. Full margin required. |
| `BTST` | Buy Today Sell Tomorrow. |
| `MTF` | Margin Trading Facility. |

### `order_type` — How Does It Execute?

| Value | Description |
|---|---|
| `RL` | **Regular Limit** — Order sits at your specified price. Executes only when market reaches your price. |
| `RL-MKT` | **Market Order** — Executes immediately at the best available market price. Set `price` to `0`. |
| `SL` | **Stop Loss Limit** — Set a trigger price and a limit price. When market hits trigger, order activates at limit price. |
| `SL-MKT` | **Stop Loss Market** — Like SL but executes at market price once triggered. |

### `validity` — How Long Is the Order Active?

| Value | Description |
|---|---|
| `DAY` | Valid only for today. Automatically cancelled if not filled by end of day. |
| `IOC` | Immediate or Cancel. |
| `GTD` | Good Till Date. |
| `GTC` | Good Till Cancelled. |

### `trigger_price` — When to Use It?

Only needed for `SL` and `SL-MKT` order types. For all other orders, always set `trigger_price` to `0`.

### `strike_price` — Important Note for Options

When placing options orders, `strike_price` must be in **paise, not rupees**. Multiply the rupee value by 100.

> Example: ₹56,000 strike → `5600000` in the API (`56000 × 100 = 5600000`)

---

## Placing Orders

### Equity Order

```python
res = client.place_order({
    "scrip_info": {
        "exchange": "NSE_EQ",
        "scrip_token": 14366,
        "symbol": "",
        "series": "EQ"
    },
    "transaction_type": "BUY",
    "product_type": "DELIVERY",
    "order_type": "RL",
    "quantity": 1,
    "price": 8.59,
    "trigger_price": 0,
    "disclosed_quantity": 0,
    "validity": "DAY",
    "validity_days": 0,
    "is_amo": False,
    "order_identifier": "",
    "part_code": "",
    "algo_id": "",
    "strategy_id": "",
    "vender_code": ""
})

order_id = res["data"]["orderId"]
print("Order ID:", order_id)
```

### Futures Order

```python
res = client.place_order({
    "scrip_info": {
        "exchange": "NSE_FO",
        "scrip_token": 66688,
        "symbol": "BANKNIFTY",
        "expiry_date": "2025-10-31",
        "strike_price": 0,
        "option_type": ""
    },
    "transaction_type": "BUY",
    "product_type": "INTRADAY",
    "order_type": "RL",
    "quantity": 50,
    "price": 56000.00,
    "trigger_price": 0,
    "validity": "DAY"
})
```

### Options Order

```python
res = client.place_order({
    "scrip_info": {
        "exchange": "NSE_FO",
        "scrip_token": 67528,
        "symbol": "BANKNIFTY",
        "expiry_date": "2026-06-30",
        "strike_price": 5600000,   # In paise: ₹56000 × 100 = 5600000
        "option_type": "CE"        # CE = Call, PE = Put
    },
    "transaction_type": "SELL",
    "product_type": "INTRADAY",
    "order_type": "RL",
    "quantity": 30,
    "price": 120.00,
    "trigger_price": 0,
    "validity": "DAY"
})
```

---

## Modify Order

> ⚠️ The API requires you to send **all fields**, not just the ones you want to change. Fetch the order from `get_order_book()` first to get current values. The `traded_quantity` field **must** be fresh from the order book every time.

```python
res = client.modify_order({
    "exchange": "NSE_EQ",
    "order_id": "NXHCO00006A4",
    "order_type": "RL",
    "quantity": 1,
    "traded_quantity": 0,       # Required — get fresh from order book
    "price": 9.00,              # New price
    "trigger_price": 0,
    "disclosed_quantity": 0,
    "validity": "DAY",
    "validity_days": 0
})
```

---

## Cancel Order

```python
res = client.cancel_order({
    "exchange": "NSE_EQ",
    "order_id": "NXHCO00006A4"
})
```

---

## Cover Orders

```python
# Place cover order
res = client.place_cover_order({...})

# Modify cover order
res = client.modify_cover_order({
    "exchange": "NSE_EQ",
    "order_id": "NXHCO00006A4",
    ...
})

# Cancel cover order
res = client.cancel_cover_order({
    "exchange": "NSE_EQ",
    "order_id": "NXHCO00006A4"
})
```

---

## Bracket Orders

```python
# Place bracket order
res = client.place_bracket_order({...})

# Modify bracket order
res = client.modify_bracket_order({
    "exchange": "NSE_EQ",
    "order_id": "NXHCO00006A4",
    ...
})

# Delete bracket order
res = client.delete_bracket_order({
    "exchange": "NSE_EQ",
    "order_id": "NXHCO00006A4"
})
```

---

## Multileg Orders

```python
# Place multileg order
res = client.place_multileg_order({...})

# Cancel multileg order
res = client.cancel_multileg_order({
    "order_flag": "<order_flag>",
    "gateway_order_no": "<gateway_order_no>",
    ...
})
```

---

## Reports

### Order Book

Shows every order you placed today, regardless of status. Supports pagination and filtering.

```python
res = client.get_order_book({
    "offset": "1",
    "limit": "20",
    "orderStatus": None,
    "order_id": None
})

if res and "data" in res:
    for o in res["data"]:
        print(
            o["order_id"],
            o["symbol"],
            o["exchange"],
            o["transaction_type"],
            o["status"],
            o["order_type"],
            o["product_type"],
            o["total_quantity"],
            o["traded_quantity"],
            o["pending_quantity"],
            o["order_price"],
            o["trigger_price"],
            o["order_timestamp"],
            o["exchange_timestamp"],
            o["error_reason"],
        )
    print("Total orders:", res["metadata"]["total_records"])
```

### Trade Book

Shows all executed trades for the day.

```python
res = client.get_trade_book({
    "offset": "1",
    "limit": "20"
})
```

### Order History

Shows the complete status timeline of a single order. Useful for debugging rejected or stuck orders.

```python
res = client.get_order_history({"orderId": "NXHCO00002A4"})

if res and "data" in res:
    for h in res["data"]:
        print(
            h["order_id"],
            h["status"],
            h["order_timestamp"],
            h["exchange_timestamp"],
            h["traded_quantity"],
            h["pending_quantity"],
            h["order_price"],
        )
```

---

## Portfolio

### Positions

```python
# type: "NET" or "DAY"
res = client.get_positions({"type": "NET"})
```

### Position Conversion

Convert positions from intraday to delivery or vice versa.

```python
res = client.position_conversion({...})
```

### Holdings

```python
res = client.get_holdings()
```

---

## Order Lifecycle

When you place an order, it goes through several stages. Track these using `get_order_history()`.

| Status | Description |
|---|---|
| `OMS_XMITTED` | Order reached the OMS and is being processed. Not yet sent to exchange. |
| `EXCHANGE_XMITTED` | OMS sent your order to the exchange. Waiting for confirmation. |
| `PENDING` | Exchange confirmed. Sitting in order book waiting to be matched. |
| `EXECUTED` | Fully matched and filled. Trade is complete. |
| `CANCELLED` | Cancelled by you or automatically at end of day. |
| `OMS_REJECT` | OMS rejected before reaching exchange. Check `error_reason`. |
| `ORDER_ERROR` | Exchange rejected. Common reasons: insufficient margin, invalid symbol, market closed. |
| `ADMIN_REJECT` | Broker admin rejected the order. |

> You can only modify or cancel orders in `PENDING` or `OMS_XMITTED` status. Executed or cancelled orders cannot be changed.

---

## Order Book — Field Reference

| Field | Description |
|---|---|
| `order_id` | Unique ID like `NXHCO00003A4` |
| `exchange` | Segment like `NSE_EQ` |
| `scrip_token` | Instrument token number |
| `symbol` | Stock symbol like `IDEA` |
| `series` | Series like `EQ` |
| `transaction_type` | `BUY` or `SELL` |
| `product_type` | `INTRADAY`, `DELIVERY`, etc. |
| `order_type` | `RL`, `RL-MKT`, `SL`, `SL-MKT` |
| `status` | `EXECUTED`, `PENDING`, etc. |
| `total_quantity` | Total shares ordered |
| `traded_quantity` | Shares filled so far |
| `pending_quantity` | Shares still waiting |
| `disclosed_quantity` | Shares visible in market |
| `order_price` | Price as string like `"9.56"` |
| `trigger_price` | SL trigger price as string like `"0.00"` |
| `validity` | `DAY`, `IOC`, etc. |
| `validity_days` | Number of days for GTD orders |
| `order_timestamp` | When placed, e.g. `"2026-04-17 12:46:34"` |
| `exchange_timestamp` | When exchange confirmed, e.g. `"2026-04-17 12:46:33"` |
| `error_reason` | Blank if no error, filled if rejected |
| `is_amo_order` | `True` if after market order |
| `exchange_order_no` | Exchange's own order number |
| `client_id` | Your UCC like `NE1234` |
| `bracket_details` | Nested object with bracket/cover order details |

---

## WebSocket — Live Market Data

### Broadcast Socket (Touchline & Market Depth)

```python
import asyncio

async def main():
    # ... login first ...

    async def on_open_broadcast_socket(message):
        print("Broadcast socket opened:", message)
        await client.touchline_subscription([
            {"MktSegId": "1", "token": "26009"}
        ])

    async def on_touchline(message):
        print("Touchline:", message)

    async def on_bestfive(message):
        print("Best Five:", message)

    async def on_close_broadcast_socket(close_msg):
        print("Broadcast socket closed:", close_msg)

    async def on_error_broadcast_socket(error):
        print("Broadcast socket error:", error)

    client.on_open_broadcast_socket = on_open_broadcast_socket
    client.on_close_broadcast_socket = on_close_broadcast_socket
    client.on_error_broadcast_socket = on_error_broadcast_socket
    client.on_touchline = on_touchline
    client.on_bestfive = on_bestfive

    await client.connect_broadcast_socket()

asyncio.run(main())
```

### Touchline Subscription / Unsubscription

```python
# Subscribe
await client.touchline_subscription([
    {"MktSegId": "1", "token": "26009"}
])

# Unsubscribe
await client.touchline_unsubscription([
    {"MktSegId": "1", "token": "26009"}
])
```

### Best Five (Market Depth) Subscription / Unsubscription

```python
# Subscribe
await client.bestfive_subscription({"MktSegId": "1", "token": "22"})

# Unsubscribe
await client.bestfive_unsubscription({"MktSegId": "1", "token": "22"})
```

### Message Socket (Order Updates)

```python
async def on_ready_message_socket(response):
    print("Message socket ready:", response)

async def on_msg_message_socket(response):
    print("Order update:", response)

async def on_close_message_socket(close_msg):
    print("Message socket closed:", close_msg)

async def on_error_message_socket(error):
    print("Message socket error:", error)

client.on_ready_message_socket = on_ready_message_socket
client.on_msg_message_socket = on_msg_message_socket
client.on_close_message_socket = on_close_message_socket
client.on_error_message_socket = on_error_message_socket

await client.connect_message_socket()
```

### Running Both Sockets Together

```python
await asyncio.gather(
    client.connect_broadcast_socket(),
    client.connect_message_socket(),
)
```

---

## Scrip Master (Token Repository)

Get scrip tokens from the scrip master files:

| Exchange | URL |
|---|---|
| NSE Equity | `https://odinscripmaster.s3.ap-south-1.amazonaws.com/scripfiles/NSE_EQ.json` |
| NSE F&O | `https://odinscripmaster.s3.ap-south-1.amazonaws.com/scripfiles/NSE_FO.json` |
| BSE Equity | `https://odinscripmaster.s3.ap-south-1.amazonaws.com/scripfiles/BSE_EQ.json` |
| BSE F&O | `https://odinscripmaster.s3.ap-south-1.amazonaws.com/scripfiles/BSE_FO.json` |

**v2 URLs** (alternate version):

Replace the path with `/scripfiles/v2/<EXCHANGE>.json`, e.g.:
`https://odinscripmaster.s3.ap-south-1.amazonaws.com/scripfiles/v2/NSE_FO.json`

---

## Official Documentation

Full API reference with all endpoints, request schemas, and response structures:

🔗 [https://b2c-api-docs.northeastltd.in/docs/B2CAPI/nebsl-b2c-api-docs.html](https://b2c-api-docs.northeastltd.in/docs/B2CAPI/nebsl-b2c-api-docs.html)

---

## Complete API Reference

| Method | Description |
|---|---|
| `client.login({...})` | Login and get access token |
| `client.validateSession()` | Keep session alive |
| `client.balance()` | Check available margins |
| `client.logout()` | End session |
| `client.place_order({...})` | Place a regular order |
| `client.modify_order({...})` | Modify a pending order |
| `client.cancel_order({...})` | Cancel a pending order |
| `client.place_cover_order({...})` | Place a cover order |
| `client.modify_cover_order({...})` | Modify a cover order |
| `client.cancel_cover_order({...})` | Cancel a cover order |
| `client.place_bracket_order({...})` | Place a bracket order |
| `client.modify_bracket_order({...})` | Modify a bracket order |
| `client.delete_bracket_order({...})` | Delete a bracket order |
| `client.place_multileg_order({...})` | Place a multileg order |
| `client.cancel_multileg_order({...})` | Cancel a multileg order |
| `client.get_order_book({...})` | Fetch order book |
| `client.get_trade_book({...})` | Fetch trade book |
| `client.get_order_history({...})` | Fetch order status timeline |
| `client.get_positions({...})` | Fetch open positions |
| `client.position_conversion({...})` | Convert position type |
| `client.get_holdings()` | Fetch demat holdings |
| `client.connect_broadcast_socket()` | Connect to live data WebSocket |
| `client.connect_message_socket()` | Connect to order updates WebSocket |
| `client.touchline_subscription([...])` | Subscribe to LTP data |
| `client.touchline_unsubscription([...])` | Unsubscribe from LTP data |
| `client.bestfive_subscription({...})` | Subscribe to market depth |
| `client.bestfive_unsubscription({...})` | Unsubscribe from market depth |
