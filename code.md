IMPORT TEST DATA 

import yfinance as yf
import pandas as pd

dataF = yf.download("EURUSD=X", start="2023-10-10", end="2023-12-10", interval='1h')
dataF.iloc[:,:]
#dataF.Open.iloc

DEFINE YOUR SIGNALS FUNCTIONS

def signal_generator(df):
    open = df.Open.iloc[-1]
    close = df.Close.iloc[-1]
    previous_open = df.Open.iloc[-2]
    previous_close = df.Close.iloc[-2]
    
    # Bearish Pattern
    if (open>close and 
    previous_open<previous_close and 
    close<previous_open and
    open>=previous_close):
        return 1

    # Bullish Pattern
    elif (open<close and 
        previous_open>previous_close and 
        close>previous_open and
        open<=previous_close):
        return 2
    
    # No clear pattern
    else:
        return 0

signal = []
signal.append(0)
for i in range(1,len(dataF)):
    df = dataF[i-1:i+1]
    signal.append(signal_generator(df))
#signal_generator(data)
dataF["signal"] = signal

dataF.signal.value_counts()
#dataF.iloc[:, :]

CONNECT TO MARKETS AND EXECUTE THE TRADES

from apscheduler.schedulers.blocking import BlockingScheduler
from pymexc import spot, futures
from time import time

api_key = "mx0vglootnjuVZaGgz"
api_secret = "7360d73af7d14e7d8d1b4e41cabe97ed"
symbol = 'BTC_USDT'  # Replace with the actual trading symbol you want to use

# SPOT V3
spot_client = spot.HTTP(api_key=api_key, api_secret=api_secret)
spot_client.recv_window = 5000  # Set the recv_window after initialization
ws_spot_client = spot.WebSocket(api_key=api_key, api_secret=api_secret)

def handle_spot_message(message):
    print("Spot:", message)

print(spot_client.exchange_info())
ws_spot_client.deals_stream(handle_spot_message, "BTC_USDT")

# FUTURES V1
futures_client = futures.HTTP(api_key=api_key, api_secret=api_secret)
ws_futures_client = futures.WebSocket(api_key=api_key, api_secret=api_secret)

def handle_futures_message(message):
    print("Futures:", message)

print(futures_client.index_price("MX_USDT"))
ws_futures_client.tickers_stream(handle_futures_message)

# MXC Trading Job Logic
def trading_job():
    candles = spot_client.klines(symbol, '15m', 3)  # Fetching candlestick data

    # Extract necessary data from candles and perform your trading logic

    # Example order placement (replace with MXC-specific order placement logic)
    timestamp = int(time() * 1000)
    # Ensure the timestamp falls within the acceptable range
    timestamp = max(timestamp, spot_client.recv_window)
    order_request = spot_client.market_order(symbol, 'BUY', 100, timestamp=timestamp)  # Replace with your order details
    spot_client.place_order(order_request)

# Set up scheduler
scheduler = BlockingScheduler()

# Schedule trading job every 15 minutes (adjust as needed)
scheduler.add_job(trading_job, 'interval', minutes=15)

# Start the scheduler
scheduler.start()

from config import access_token, accountID
def get_candles(n):
    #access_token='XXXXXXX'#you need token here generated from OANDA account
    client = CandleClient(access_token, real=False)
    collector = client.get_collector(Pair.EUR_USD, Gran.M15)
    candles = collector.grab(n)
    return candles

candles = get_candles(3)
for candle in candles:
    print(float(str(candle.bid.o))>1)

def trading_job():
    candles = get_candles(3)
    dfstream = pd.DataFrame(columns=['Open','Close','High','Low'])
    
    i=0
    for candle in candles:
        dfstream.loc[i, ['Open']] = float(str(candle.bid.o))
        dfstream.loc[i, ['Close']] = float(str(candle.bid.c))
        dfstream.loc[i, ['High']] = float(str(candle.bid.h))
        dfstream.loc[i, ['Low']] = float(str(candle.bid.l))
        i=i+1

    dfstream['Open'] = dfstream['Open'].astype(float)
    dfstream['Close'] = dfstream['Close'].astype(float)
    dfstream['High'] = dfstream['High'].astype(float)
    dfstream['Low'] = dfstream['Low'].astype(float)

    signal = signal_generator(dfstream.iloc[:-1,:])#
    
    # EXECUTING ORDERS
    #accountID = "XXXXXXX" #your account ID here
    client = API(access_token)
         
    SLTPRatio = 2.
    previous_candleR = abs(dfstream['High'].iloc[-2]-dfstream['Low'].iloc[-2])
    
    SLBuy = float(str(candle.bid.o))-previous_candleR
    SLSell = float(str(candle.bid.o))+previous_candleR

    TPBuy = float(str(candle.bid.o))+previous_candleR*SLTPRatio
    TPSell = float(str(candle.bid.o))-previous_candleR*SLTPRatio
    
    print(dfstream.iloc[:-1,:])
    print(TPBuy, "  ", SLBuy, "  ", TPSell, "  ", SLSell)
    signal = 2
    #Sell
    if signal == 1:
        mo = MarketOrderRequest(instrument="EUR_USD", units=-1000, takeProfitOnFill=TakeProfitDetails(price=TPSell).data, stopLossOnFill=StopLossDetails(price=SLSell).data)
        r = orders.OrderCreate(accountID, data=mo.data)
        rv = client.request(r)
        print(rv)
    #Buy
    elif signal == 2:
        mo = MarketOrderRequest(instrument="EUR_USD", units=1000, takeProfitOnFill=TakeProfitDetails(price=TPBuy).data, stopLossOnFill=StopLossDetails(price=SLBuy).data)
        r = orders.OrderCreate(accountID, data=mo.data)
        rv = client.request(r)
        print(rv)

EXECUTING ORDERS AUTOMATICALLY WITH A SCHEDULER

trading_job()

#scheduler = BlockingScheduler()
#scheduler.add_job(trading_job, 'cron', day_of_week='mon-fri', hour='00-23', minute='1,16,31,46', start_date='2022-01-12 12:00:00', timezone='America/Chicago')
#scheduler.start()


