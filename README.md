# pycloudrestapi — Complete Guide

**Northeast B2C Trading API — Python Client**

---

## What Is This?

`pycloudrestapi` is a Python library that connects your code to the **Northeast B2C REST API** — a trading system used by IBT and NEBSL brokers, powered by the Odin/Wave trading engine.

Think of it as a remote control for your trading account. Instead of logging into a trading app manually, you write Python code that logs in, places orders, checks your portfolio, and receives live prices — all automatically.

---

## What Can You Do With It?

**Account** — Login, validate your session, check your balance and margins, and logout.

**Orders** — Place fresh buy or sell orders, modify pending orders (change price or quantity), and cancel orders you no longer want.

**Order Types** — Regular limit, stop loss limit.

**Reports** — Fetch your full order book (all orders placed today), trade book (all executed trades), and order history (status timeline of a single order).

**Portfolio** — View your open positions, convert positions from intraday to delivery (or vice versa), and check your demat holdings.



---

## Official Documentation ( Recommended )

Full API reference with all endpoints, request schemas, and response structures:

🔗 [https://developer.synapsewave.com/docs/category/b2c-api](https://developer.synapsewave.com/docs/category/b2c-api)




---

## Installation

```bash
pip install nebsl-b2c-api
```

---

## Credentials You Need

Your broker provides four things when they onboard you:

**Base URL** — The server address where all API calls go. It looks like a long AWS or cloud URL.

**API Key** — Your application's identity key. Sent in the request body during login.

**Second AUTH** — A PAN number sent in every request header to authenticate your app.

**User ID (UCC)** — Your Unique Client Code. This is your broker account number. Example: `NE2247`

> ⚠️ **Never share these credentials or put them in your code directly.** Store them in environment variables or a `.env` file.

---

## Quick Start

```python
from pycloudrestapi import IBTConnect

client = IBTConnect({
    "baseurl": "<your_base_url>",
    "api_key": "<your_api_key>",
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

Every stock, futures contract, or options contract on an exchange has a unique number called a **scrip token**. The API uses this number to identify what you want to trade. For example, IDEA on NSE_EQ has token `14366`, and IDEA  on BSE_EQ has token `532822`.

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

### `order_type` — How Does It Execute?

| Value | Description |
|---|---|
| `RL` | **Regular Limit** — Order sits at your specified price. Executes only when market reaches your price. |
| `SL` | **Stop Loss Limit** — Set a trigger price and a limit price. When market hits trigger, order activates at limit price. |


### `validity` — How Long Is the Order Active?

| Value | Description |
|---|---|
| `DAY` | Valid only for today. Automatically cancelled if not filled by end of day. |

### `trigger_price` — When to Use It?

Only needed for `SL` order types. For all other orders, always set `trigger_price` to `0`.

### `strike_price` — Important Note for Options

When placing options orders, `strike_price` must be in **paise, not rupees**. Multiply the rupee value by 100.

> Example: ₹56,000 strike → `5600000` in the API (`56000 × 100 = 5600000`)

---

## Placing Orders

### Equity Order

```python
res = client.place_order({
    "scrip_info": {
        "exchange":    "NSE_EQ",   # NSE_EQ / BSE_EQ / NSE_FO / BSE_FO / MCX_FO
        "scrip_token": 14366,      # BSE token = 532822
        "symbol":      "",     # Optional if scrip_token given
        "series":      "EQ"        # EQ for equity
    },
    "transaction_type":  "BUY",        # BUY or SELL
    "product_type":      "DELIVERY",   # INTRADAY / DELIVERY / BTST / MTF
    "order_type":        "RL",         # RL / SL / S
    "quantity":          1,
    "price":             8.59,      # Set 0 for market order 
    "trigger_price":     0,            # Only for SL orders
    "disclosed_quantity": 0,           # 0 = show full quantity
    "validity":          "DAY",        # DAY / IOC / GTD / GTC
  
})
order_id = res["data"]["orderId"]         # To know order ID 
print("Order ID:", order_id)
```

### Futures Order

```python
res = client.place_order({
    "scrip_info": {
        "exchange":     "NSE_FO",
        "scrip_token":  66688,          # Token for the futures contract
        "symbol":       "BANKNIFTY",
        "expiry_date":  "2025-10-31",   # Format: yyyy-mm-dd
        "strike_price": 0,              # 0 for futures
        "option_type":  ""              # Blank for futures
    },
    "transaction_type": "BUY",
    "product_type":     "INTRADAY",
    "order_type":       "RL",       # Market order
    "quantity":         50,             # 1 lot NIFTY = 50
    "price":            56000.00,              # 0 for market
    "trigger_price":    0,                   # Only for SL orders
    "validity":         "DAY"
})
```

### Options Order

```python
res = client.place_order({
    "scrip_info": {
        "exchange":     "NSE_FO",
        "scrip_token":  67528,
        "symbol":       "BANKNIFTY",
        "expiry_date":  "2026-06-30",
        "strike_price": 5600000,   # In PAISE: ₹25000 × 100 = 2500000
        "option_type":  "CE"       # CE = Call Option, PE = Put Option
    },
    "transaction_type": "SELL",
    "product_type":     "INTRADAY",
    "order_type":       "RL",
    "quantity":         30,
    "price":            120.00,
    "trigger_price":    0,                                # Only for SL orders
    "validity":         "DAY"
})
```

---

## Modify Order

> ⚠️ The API requires you to send **all fields**, not just the ones you want to change. Fetch the order from `get_order_book()` first to get current values. The `traded_quantity` field **must** be fresh from the order book every time.

```python
res = client.modify_order({
    "exchange":        "NSE_EQ",        
    "order_id":        "NXHCO00006A4",  # 
    "order_type":      "RL",            # 
    "quantity":   1,   
    "traded_quantity":    0,  # required from order book
    "price":              9.00,                   # new price
    "trigger_price":      0,
    "disclosed_quantity": 0,
    "validity":           "DAY",         # DAY
    "validity_days":      0     # 0
})
```

---

## Cancel Order

```python
res = client.cancel_order({
    "exchange": "NSE_EQ",        # Exchange segment
    "order_id": "NXHCO00006A4"  # Order ID from place_order / order book
})
```

---

## Reports

### Order Book

Shows every order you placed today, regardless of status. Supports pagination and filtering.

```python
res = client.get_order_book({
    "offset":      "1",
    "limit":       "20",
    "orderStatus": None,
    "order_id":    None
})

if res and "data" in res:
    for o in res["data"]:
        print(
            o["order_id"],           # e.g. NXHCO00003A4
            o["symbol"],             # e.g. IDEA
            o["exchange"],           # e.g. NSE_EQ
            o["transaction_type"],   # BUY or SELL
            o["status"],             # EXECUTED / PENDING / CANCELLED
            o["order_type"],         # RL / SL
            o["product_type"],       # INTRADAY / DELIVERY
            o["total_quantity"],     # total qty placed
            o["traded_quantity"],    # qty filled
            o["pending_quantity"],   # qty remaining
            o["order_price"],        # price (string)
            o["trigger_price"],      # trigger price (string)
            o["order_timestamp"],    # "2026-04-17 12:46:34"
            o["exchange_timestamp"], # exchange confirm time
            o["error_reason"],       # blank if no error
        )
    print("Total orders:", res["metadata"]["total_records"])
```

### Order History

Shows the complete status timeline of a single order. Useful for debugging rejected or stuck orders.

```python
res = client.get_order_history({"orderId": "NXHCO00002A4"})

if res and "data" in res:
    for h in res["data"]:
        print(
            h["order_id"],
            h["status"],             # OMSXMITTED / EXECUTED etc.
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
# type: "DAY"
res = client.get_positions({"type": "DAY"})
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
| `order_type` | `RL`, `SL`|
| `status` | `EXECUTED`, `PENDING`, etc. |
| `total_quantity` | Total shares ordered |
| `traded_quantity` | Shares filled so far |
| `pending_quantity` | Shares still waiting |
| `disclosed_quantity` | Shares visible in market |
| `order_price` | Price as string like `"9.56"` |
| `trigger_price` | SL trigger price as string like `"0.00"` |
| `validity` | `DAY`, etc. |
| `validity_days` | Number of days for GTD orders |
| `order_timestamp` | When placed, e.g. `"2026-04-17 12:46:34"` |
| `exchange_timestamp` | When exchange confirmed, e.g. `"2026-04-17 12:46:33"` |
| `error_reason` | Blank if no error, filled if rejected |
| `exchange_order_no` | Exchange's own order number |
| `client_id` | Your UCC like `NE1234` |
| `bracket_details` | Nested object with bracket/cover order details |

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
| `client.place_bracket_order({...})` | Place a bracket order |
| `client.modify_bracket_order({...})` | Modify a bracket order |
| `client.delete_bracket_order({...})` | Delete a bracket order |
| `client.get_order_book({...})` | Fetch order book |
| `client.get_trade_book({...})` | Fetch trade book |
| `client.get_order_history({...})` | Fetch order status timeline |
| `client.get_positions({...})` | Fetch open positions |
| `client.position_conversion({...})` | Convert position type |
| `client.get_holdings()` | Fetch demat holdings |