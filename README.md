pycloudrestapi — Complete Guide
Northeast B2C Trading API — Python Client

What Is This?
pycloudrestapi is a Python library that connects your code to the Northeast B2C REST API — a trading system used by IBT and NEBSL brokers, powered by the Odin/Wave trading engine.

Think of it as a remote control for your trading account. Instead of logging into a trading app manually, you write Python code that logs in, places orders, checks your portfolio, and receives live prices — all automatically.


What Can You Do With It?
Account — Login, validate your session, check your balance and margins, and logout.

Orders — Place fresh buy or sell orders, modify pending orders (change price or quantity), and cancel orders you no longer want.

Order Types — Regular limit, stop loss limit.

Reports — Fetch your full order book (all orders placed today), trade book (all executed trades), and order history (status timeline of a single order).

Portfolio — View your open positions, convert positions from intraday to delivery (or vice versa), and check your demat holdings.


Credentials You Need
Your broker provides four things when they onboard you:

Base URL — The server address where all API calls go. It looks like a long AWS or cloud URL.

API Key — Your application's identity key. Sent in the request body during login.

Second AUTH — A PAN number sent in every request header to authenticate your app.

User ID (UCC) — Your Unique Client Code. This is your broker account number. Example: NE2247

Never share these credentials or put them in your code directly. Store them in environment variables or a .env file.


How Login Works
The Basic Idea
Every API call (except login itself) requires an access token. This token proves you are who you say you are. You get this token by logging in. The library stores it automatically and sends it with every subsequent request.
Login
Direct Login — You simply send your user ID, password, API key, and source (WEBAPI). The library stores the access token for you automatically.

User ID  — Login using Your Unique Client Code
PASSWORD — Login using your normal password
Session Management
After login, call validateSession() periodically to keep your session alive. Call balance() anytime to check your available margins. Call logout() when you are done trading.


Understanding Orders
scrip_token — What Is It?
Every stock, futures contract, or options contract on an exchange has a unique number called a scrip token. The API uses this number to identify what you want to trade. For example, RELIANCE on NSE_EQ has token 2885, and IDEA has token 14366.

You get the token from your broker's scrip master file, which lists every tradeable instrument and its corresponding token number.
exchange — Which Market?
The exchange field tells the API which market segment you are trading in.

NSE_EQ — NSE Equity (buying/selling stocks on NSE)
NSE_FO — NSE Futures and Options
product_type — How Long You Hold It?
This is one of the most important fields. It determines whether your trade is for today only or you want to carry it overnight.

INTRADAY — Also called MIS. You must close this position before market closes (usually 3:20 PM for NSE equity). If you forget, the broker squares it off automatically. Lower margin required.

DELIVERY — Also called CNC. The stock goes into your demat account. You can hold it for as long as you want — days, months, or years. Full margin required.
order_type — How Does It Execute?
RL (Regular Limit) — Your order sits in the exchange order book at the price you specify. It executes only when the market price reaches your price. You have full control over execution price but no guarantee it will fill.

SL (Stop Loss Limit) — You set two prices: a trigger price and a limit price. When the market reaches the trigger price, your order activates and enters the book at the limit price. Used to limit losses.
validity — How Long Is the Order Active?
DAY — The order is valid only for today. Automatically cancelled if not filled by end of day.
trigger_price — When to Use It?
Only needed for SL order types. This is the price that activates the order. For example, if you set trigger price to ₹490 on a SELL SL order, the system watches the market. When price drops to ₹490, your stop loss activates and places the sell order. For all non-SL orders, always set trigger_price to 0.
strike_price — Important Note for Options
When placing options orders, strike_price must be in paise, not rupees. This means you multiply the rupee value by 100. For example, ₹25000 strike becomes 2500000 in the API. This is one of the most common errors when working with options.


Order Lifecycle — What Happens After You Place an Order?
When you place an order, it goes through several stages. You can track these using get_order_history().

OMSXMITTED — Your order reached the OMS (Order Management System) and is being processed. Not yet sent to the exchange.

EXXMITTED — OMS has sent your order to the exchange. Waiting for exchange confirmation.

PENDING — Exchange confirmed the order. It is sitting in the exchange order book waiting to be matched with a counter-party.

EXECUTED — Your order was fully matched and filled. The trade is complete.

CANCELLED — Order was cancelled — either by you manually, or automatically by the system at end of day.

OMSREJECT — OMS rejected your order before it reached the exchange. Check the error_reason field to understand why.

ORDERERROR — Exchange rejected your order. Common reasons are insufficient margin, invalid symbol, or market being closed.

ADMINREJECT — Broker admin rejected the order.

You can only modify or cancel orders that are in PENDING or OMSXMITTED status. Executed or cancelled orders cannot be changed.


Modifying an Order — What You Need to Know
When you modify an order, the API requires you to send all fields in the request body, not just the ones you want to change. This means you must first fetch the order from get_order_book() to get the current values, then pass them all in the modify request along with your new price or quantity.

The most important field is traded_quantity — this tells the API how much of the order has already been filled. You must get this fresh from the order book response every time before modifying. If you hardcode it or get it wrong, the modification will fail.


Reports — What Each One Shows
Order Book — Shows every order you placed today, regardless of status. Includes pending, executed, cancelled, and rejected orders. Supports pagination using offset and limit. You can filter by status using orderStatus. Returns a metadata object with total record counts.

Order History — Shows the complete status timeline of a single specific order. For example, it shows when it was OMSXMITTED, when it became PENDING, and when it finally became EXECUTED. Very useful for debugging rejected or stuck orders.


Order Book — Real Field Names
When you receive orders from the API, these are the actual field names in each order object. Understanding these is critical because using wrong names causes KeyError crashes.

order_id — unique ID like NXHCO00003A4
exchange — segment like NSE_EQ
scrip_token — instrument token number
symbol — stock symbol like IDEA
series — series like EQ
transaction_type — BUY or SELL
product_type — INTRADAY or DELIVERY etc.
order_type — RL, RL-MKT, SL etc.
status — current status like EXECUTED or PENDING
total_quantity — how many shares you ordered
traded_quantity — how many have been filled so far
pending_quantity — how many are still waiting
disclosed_quantity — how many are visible in market
order_price — the price as a string like "9.56"
trigger_price — SL trigger price as string like "0.00"
validity — DAY, IOC etc.
validity_days — number of days for GTD orders
order_timestamp — when you placed the order, like "2026-04-17 12:46:34"
exchange_timestamp — when exchange confirmed, like "2026-04-17 12:46:33"
error_reason — blank if no error, filled if rejected
is_amo_order — True if it is an after market order
exchange_order_no — the exchange's own order number
client_id — your UCC like NE1234
bracket_details — nested object with bracket/cover order details


TOKENS Repsotry
Kindly use below scrip master API for Tokens : https://odinscripmaster.s3.ap-south-1.amazonaws.com/scripfiles/NSE_EQ.json

replace exchange name, like NSE_FO.json, etc...with the list below. Another version file is at https://odinscripmaster.s3.ap-south-1.amazonaws.com/scripfiles/v2/NSE_FO.json

Kindly use below scrip master API for Tokens : https://odinscripmaster.s3.ap-south-1.amazonaws.com/scripfiles/BSE_EQ.json

replace exchange name, like BSE_FO.json, etc...with the list below. Another version file is at https://odinscripmaster.s3.ap-south-1.amazonaws.com/scripfiles/v2/BSE_FO.json
Official Documentation
Full API reference with all endpoints, request schemas, and response structures:

https://b2c-api-docs.northeastltd.in/docs/B2CAPI/nebsl-b2c-api-docs.html

pip install nebsl-b2c-api

from pycloudrestapi.connect import IBTConnect

client = IBTConnect({

    "baseurl": "",

    "api_key": "",

    "second_auth" :"",

    "debug": True

})

#--------------------------- LOGIN----------------------

user = client.login({

    "userId": "",

    "password": ""

})

print(user)

# -------------Validate session (keep alive)-------------

client.validateSession()

#--------------- Balance---------------------------

client.balance()

#------------------------------EQUITY--------------------------

res = client.place_order({

    "scrip_info": {

        "exchange":    "NSE_EQ",   # NSE_EQ / BSE_EQ / NSE_FO / BSE_FO / MCX_FO

        "scrip_token": 14366,      # Token from scrip master (INFY = 10243)

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


# --------------------------FUTURES---------------------------

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


# ----------------------OPTION------------------------------------------------

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


# ------------------------------MODIFY ORDER — ------------------------------

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


#----------------------------CANCEL ORDER--------------------------------------

res = client.cancel_order({

    "exchange": "NSE_EQ",        # Exchange segment

    "order_id": "NXHCO00006A4"  # Order ID from place_order / order book

})


# ---------------------------------ORDER BOOK-------------------------------------


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

 

# ----------------------------------------ORDER HISTORY OF SPECIFIC ID's ----------------------------------------

res = client.get_order_history({"orderId": "NXHCO00002A4"})

if res and "data" in res:

    for h in res["data"]:

        print(

            h["order_id"],

            h["status"],             # OMSXMITTED / EXECUTED etc.

            h["order_timestamp"],    # ✅ NOT "timestamp"

            h["exchange_timestamp"],

            h["traded_quantity"],

            h["pending_quantity"],

            h["order_price"],

        )


#-----------------------------------LOGOUT----------------------------------------------------

client.logout()



