# Binance-Crypto-Trader

import os
import math
from binance.client import Client
import pandas as pd
import time

# Set your Binance API key and secret
api_key = ''
api_secret = ''
client = Client(api_key, api_secret)

# Set trading parameters
symbol = 'BTCBRL'
interval = '1m'
short_window = 10
long_window = 30
fee_percentage = 0.001  # Assuming a 0.1% fee, adjust as needed

def get_historical_data():
    # Fetch historical klines
    klines = client.get_klines(symbol=symbol, interval=interval)
    
    # Convert klines to DataFrame for analysis
    df = pd.DataFrame(klines, columns=['timestamp', 'open', 'high', 'low', 'close', 'volume', 'close_time', 'quote_asset_volume', 'number_of_trades', 'taker_buy_base_asset_volume', 'taker_buy_quote_asset_volume', 'ignore'])
    
    # Convert timestamp to datetime
    df['timestamp'] = pd.to_datetime(df['timestamp'], unit='ms')
    
    return df

def calculate_moving_averages(df):
    # Calculate short and long-term moving averages
    df['short_mavg'] = df['close'].rolling(window=short_window, min_periods=1, center=False).mean()
    df['long_mavg'] = df['close'].rolling(window=long_window, min_periods=1, center=False).mean()

def execute_trade_action(balance, current_price, position):
    if position == 'none':
        return 'buy' if balance > 0 else 'none'
    elif position == 'long' and current_price < df['short_mavg'].iloc[-1]:
        return 'sell'
    elif position == 'none' and current_price > df['long_mavg'].iloc[-1]:
        return 'buy'
    else:
        return 'none'

# Main trading loop
balance = 95 # Updated balance in BRL
position = 'none'  # Initial position
while True:
    df = get_historical_data()
    calculate_moving_averages(df)
    
    current_price = float(df['close'].iloc[-1])
    
    action = execute_trade_action(balance, current_price, position)
    
    if action == 'buy' and balance > 0:
        # Get the lot size filter from the exchange info
        lot_size_filter = next((filter for filter in client.get_symbol_info(symbol)['filters'] if filter['filterType'] == 'LOT_SIZE'), None)
        if lot_size_filter:
            step_size = float(lot_size_filter['stepSize'])
            quantity_to_buy = round(balance / current_price, int(-1 * round(math.log10(step_size))))
        else:
            # Default rounding to 6 decimal places if lot size information is not available
            quantity_to_buy = round(balance / current_price, 6)
        
        order = client.order_market_buy(symbol=symbol, quantity=quantity_to_buy)
        print(f"Buying {quantity_to_buy} BTC at {current_price} BRL")
        balance = 0
        position = 'long'
    elif action == 'sell':
        quantity_to_sell = round(balance / current_price, int(-1 * round(math.log10(step_size))))
        quantity_to_sell_after_fee = quantity_to_sell * (1 - fee_percentage)
        quantity_to_sell_after_fee = max(quantity_to_sell_after_fee, 0)  # Ensure quantity is not negative
        order = client.order_market_sell(symbol=symbol, quantity=quantity_to_sell_after_fee)
        balance = current_price * quantity_to_sell_after_fee
        position = 'none'
        print(f"Selling {quantity_to_sell_after_fee} BTC at {current_price} BRL after fees")
    
    time.sleep(60)  # Wait for 1 minute before the next iteration
