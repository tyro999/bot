import time
import datetime
import oandapyV20
from oandapyV20 import API
import oandapyV20.endpoints.instruments as instruments
import talib

# Define your OANDA account credentials and trading parameters
accountID = "YOUR_ACCOUNT_ID"
access_token = "YOUR_ACCESS_TOKEN"
base_currency = "USD"
quote_currency = "EUR"
trade_units = 1000
stop_loss_distance = 0.005  # Stop loss distance as a percentage
take_profit_distance = 0.01  # Take profit distance as a percentage
moving_average_window = 20  # Moving average window size
rsi_period = 14  # RSI period
rsi_overbought_threshold = 70  # RSI overbought threshold
rsi_oversold_threshold = 30  # RSI oversold threshold
risk_percentage = 0.02  # Percentage of account balance to risk per trade

# Connect to the OANDA API
api = API(access_token=access_token, environment="practice")  # Use "live" for real trading

def get_current_price():
    # Fetch the current bid and ask prices for the currency pair
    params = {
        "instruments": f"{base_currency}_{quote_currency}",
        "granularity": "M1",  # Adjust granularity as per your strategy
        "count": 2  # Retrieve the latest two candles for bid and ask prices
    }
    request = instruments.InstrumentsCandles(instrument=params["instruments"], params=params)
    response = api.request(request)
    candles = response["candles"]
    current_price = (candles[0]["bid"]["c"] + candles[0]["ask"]["c"]) / 2
    return current_price

def calculate_moving_average():
    # Calculate the moving average over a window of specified size
    params = {
        "instruments": f"{base_currency}_{quote_currency}",
        "granularity": "M1",
        "count": moving_average_window
    }
    request = instruments.InstrumentsCandles(instrument=params["instruments"], params=params)
    response = api.request(request)
    candles = response["candles"]
    close_prices = [float(candle["mid"]["c"]) for candle in candles]
    moving_average = sum(close_prices) / len(close_prices)
    return moving_average

def calculate_rsi():
    # Calculate the RSI (Relative Strength Index) indicator
    params = {
        "instruments": f"{base_currency}_{quote_currency}",
        "granularity": "M1",
        "count": rsi_period
    }
    request = instruments.InstrumentsCandles(instrument=params["instruments"], params=params)
    response = api.request(request)
    candles = response["candles"]
    close_prices = [float(candle["mid"]["c"]) for candle in candles]
    rsi = talib.RSI(close_prices, timeperiod=rsi_period)[-1]
    return rsi

def calculate_position_size(balance):
    # Calculate the position size based on the risk percentage and account balance
    risk_amount = balance * risk_percentage
    position_size = int(risk_amount / stop_loss_distance)
    return position_size

def enter_trade():
    # Implement your entry criteria here
    current_price = get_current_price()
    moving_average = calculate_moving_average()
    rsi = calculate_rsi()
    # Add your conditions to enter a trade
    if current_price > moving_average and rsi < rsi_oversold_threshold:
        # Place a buy order
        stop_loss_price = current_price - (current_price * stop_loss_distance)
        take_profit_price = current_price + (current_price * take_profit_distance)
        account_summary = api.account.summary(accountID)
        balance = float(account_summary["account"]["balance"])
        position_size = calculate_position_size(balance)
        order = {
            "order": {
                "price": "MARKET",
                "instrument": f"{base_currency}_{quote_currency}",
                "units": position_size,
                "type": "MARKET",
                "positionFill": "DEFAULT",
                "stopLossOnFill": {
                    "price": stop_loss_price
                },
                "takeProfitOnFill": {
                    "price": take_profit_price
                }
            }
        }
        request = oandapyV20.endpoints.orders.OrderCreate(accountID, data=order)
        response = api.request(request)
        if "orderFillTransaction" in response:
            trade_id = response["orderFillTransaction"]["tradeOpened"]["tradeID"]
            print(f"Buy order placed. Trade ID: {trade_id}")

def exit_trade():
    # Implement your exit criteria here
    current_price = get_current_price()
    moving_average = calculate_moving_average()
    rsi = calculate_rsi()
    # Add your conditions to exit a trade
    if current_price < moving_average or rsi > rsi_overbought_threshold:
        # Close the trade
        trade_id = "YOUR_TRADE_ID"  # Replace with the trade ID of the open trade
        order = {
            "order": {
                "units": str(-trade_units)
            }
        }
        request = oandapyV20.endpoints.trades.TradeClose(accountID=accountID, tradeID=trade_id, data=order)
        api.request(request)
        print("Trade closed")

def run_bot():
    while True:
        now = datetime.datetime.now()
        # Define your trading hours or frequency of trades
        if now.weekday() < 5 and now.hour == 9 and now.minute == 0:
            enter_trade()
        if now.weekday() < 5 and now.hour == 16 and now.minute == 0:
            exit_trade()
        time.sleep(60)  # Sleep for one minute before checking conditions again

# Run the bot
run_bot()
