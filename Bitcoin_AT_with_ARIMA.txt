import time
import pyupbit
import datetime
import schedule
import pandas as pd
import numpy as np
from statsmodels.tsa.arima.model import ARIMA

access = "vNFxtArjVTw2XrFaZsgINAc5ts8ma2d9A4rDNtta"
secret = "PZN9tryzoVYfMgD6olNliUxdq2KHyOEF3z9he4MD"

def get_target_price(ticker, k):
    """변동성 돌파 전략으로 매수 목표가 조회"""
    df = pyupbit.get_ohlcv(ticker, count=600, interval="minute60")
    df["range"] = df["high"] - df["low"]
    target_price = df.iloc[-1]["close"] + (df.iloc[-1]["range"] * k)
    return target_price

def get_start_time(ticker):
    """시작 시간 조회"""
    df = pyupbit.get_ohlcv(ticker, interval="day", count=1)
    start_time = df.index[0].replace(hour=21, minute=34, second=0, microsecond=0)
    return start_time

def get_balance(ticker):
    """잔고 조회"""
    balances = upbit.get_balances()
    for b in balances:
        if b["currency"] == ticker:
            if b["balance"] is not None:
                return float(b["balance"])
            else:
                return 0
    return 0

def get_current_price(ticker):
    """현재가 조회"""
    return pyupbit.get_orderbook(tickers=ticker)[0]["orderbook_units"][0]["ask_price"]

def predict_price(ticker):
    """ARIMA 모델로 당일 종가 가격 예측"""
    df = pyupbit.get_ohlcv(ticker, interval="minute60")
    prices = df["close"].values
    model = ARIMA(prices, order=(1, 0, 1))
    model_fit = model.fit()
    forecast = model_fit.forecast(steps=24)[0]
    close_value = forecast[-1]
    global predicted_close_price
    predicted_close_price = close_value

predict_price("KRW-BTC")
schedule.every().hour.do(lambda: predict_price("KRW-BTC"))

# 로그인
upbit = pyupbit.Upbit(access, secret)
print("autotrade start")

# 자동매매 시작
while True:
    try:
        now = datetime.datetime.now()
        start_time = get_start_time("KRW-BTC")
        end_time = start_time + datetime.timedelta(days=1)
        schedule.run_pending()

        if start_time < now < end_time - datetime.timedelta(seconds=10):
            target_price = get_target_price("KRW-BTC", 0.7)
            current_price = get_current_price("KRW-BTC")
            if target_price < current_price and current_price < predicted_close_price:
                krw = get_balance("KRW")
                if krw > 5000:
                    upbit.buy_market_order("KRW-BTC", krw*0.9995)
        else:
            btc = get_balance("BTC")
            if btc > 0.00008:
                upbit.sell_market_order("KRW-BTC", btc*0.9995)
        time.sleep(1)
    except Exception as e:
        print(e)
        time.sleep(1)
