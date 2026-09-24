import backtrader as bt
import pandas as pd 
import yfinance as yf

#downlaod data once
df =yf.download(
"NVDA",
start = "2023-01-01",
end = "2023-12-31",
group_by = "column",
auto_adjust=False      # keep raw OHLCV (silences the warning too)
)
#normalize columns for backtrader
if isinstance (df.columns, pd.MultiIndex):
    df.columns = df.columns.get_level_values(0)
    
#makes names case -insensitive, uniform    
df.rename(columns=lambda c: str(c).strip().lower(), inplace=True)

#map commnovariants to the exact name backtrader expects
cool_map = {
    'open': 'open',
    'high': 'high',
    'low': 'low',
    'close': 'close',
    'adj close': 'adj close',
    'adjclose': 'adj close',
    'volume': 'volume',
    'turnerover' : 'volume',
}
# drops rows with missing OHLC (some tickers start with NaNs)
df.rename(columns=cool_map, inplace=True)
df = df.dropna(subset=['open','high','low','close','volume'])

# My SMA strategy 
class testStrategy(bt.Strategy):
    def __init__(self):
        self.sma = bt.indicators.SimpleMovingAverage(self.data.close, period=30) # period changes time

    def next(self):
        if not self.position and self.data.close[0] > self.sma[0]:
            self.buy()
        elif self.position and self.data.close[0] < self.sma[0]:
            self.sell()

cerebro = bt.Cerebro()

data = bt.feeds.PandasData(
    dataname=df,
    open='open',
    high='high',
    low='low',
    close='close',
    volume='volume',
    openinterest=None
)

cerebro.adddata(data)
cerebro.addstrategy(testStrategy)
cerebro.broker.setcash(100000.0)
print("Starting value, ", cerebro.broker.getvalue())
cerebro.run()
print("Final Portfolio Value:", cerebro.broker.getvalue())

cerebro.plot()
