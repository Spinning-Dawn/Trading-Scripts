import backtrader as bt
import yfinance as yf
import pandas as pd

# Runs a 30 day SMA(Simple moving average) test for apple stocks for complete 2023

# ---------- 1) Download once ----------
df = yf.download(
    "AAPL",
    start="2023-01-01",
    end="2023-12-31",
    group_by="column",     # ask for single-level columns
    auto_adjust=False      # keep raw OHLCV (silences the warning too)
)

# ---------- 2) Normalize columns for Backtrader ----------
# If MultiIndex, take the first level (Open/High/Low/Close/Adj Close/Volume)
if isinstance(df.columns, pd.MultiIndex):
    df.columns = df.columns.get_level_values(0)

# Make names case-insensitive, uniform
df.rename(columns=lambda c: str(c).strip().lower(), inplace=True)

# Map common variants to the exact names Backtrader expects
col_map = {
    'open': 'open',
    'high': 'high',
    'low': 'low',
    'close': 'close',
    'adj close': 'adj close',
    'adjclose': 'adj close',
    'volume': 'volume',
    'turnover': 'volume',   # just in case
}
df.rename(columns=col_map, inplace=True)

# Drop rows with missing OHLC (some tickers start with NaNs)
df = df.dropna(subset=['open','high','low','close','volume'])



# ---------- 3 Define a tiny test strategy ----------
class TestStrategy(bt.Strategy):
    def __init__(self):
        self.sma = bt.indicators.SimpleMovingAverage(self.data.close, period=30) # change value for SMA day control 

    def next(self):
        if not self.position and self.data.close[0] > self.sma[0]:
            self.buy()
        elif self.position and self.data.close[0] < self.sma[0]:
            self.sell()

# ---------- 4 Backtrader wiring ----------
cerebro = bt.Cerebro()

data = bt.feeds.PandasData(
    dataname=df,
    open='open',
    high='high',
    low='low',
    close='close',
    volume='volume',
    openinterest=None,
)

cerebro.adddata(data)
cerebro.addstrategy(TestStrategy)
cerebro.broker.setcash(100000)

print("Starting Portfolio Value:", cerebro.broker.getvalue())
cerebro.run()
print("Final Portfolio Value:", cerebro.broker.getvalue())


cerebro.plot()


