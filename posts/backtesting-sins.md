# The 7 Deadly Sins of Backtesting

**Category:** Trading | **Reading time:** 15 min

## Introduction

You built a strategy. Backtested it. 95% win rate. 50% annual return.

Then you went live and lost money.

What happened? You committed one (or more) of the 7 deadly sins of backtesting.

## Sin #1: Overfitting

**The Sin**: Your strategy is too perfectly tuned to historical data.

**The Result**: Amazing backtest. Terrible live performance.

**How to Detect It**:
- Win rate > 80%
- Profit factor > 5
- Smooth equity curve (no drawdowns)
- Too many optimized parameters

**The Fix**:
- Walk-Forward Optimization (70/30 split)
- Monte Carlo simulation (1,000+ runs)
- Parameter sensitivity: ±10% change → <30% performance change
- Maximum 3 optimization rounds

## Sin #2: Look-Ahead Bias

**The Sin**: Your strategy uses information from the future.

**The Result**: Impossible-to-replicate results.

**Common Causes**:
- Using close price for entry signals
- Not shifting indicators by 1 bar
- Using future data in calculations
- Survivorship bias in data

**The Fix**:
```python
# WRONG: Using current bar's close for signal
if close > sma(close, 20):
    buy()

# RIGHT: Using previous bar's close
if close[1] > sma(close[1], 20):
    buy()
```

## Sin #3: Survivorship Bias

**The Sin**: You only test against assets that still exist.

**The Result**: Your backtest ignores the stocks that went bankrupt.

**Example**: Testing only current S&P 500 companies ignores the ones that were removed.

**The Fix**:
- Use point-in-time data
- Include delisted assets
- Test across multiple markets
- Don't trust backtests that only show winners

## Sin #4: Ignoring Transaction Costs

**The Sin**: Your backtest assumes perfect execution at mid-price.

**The Result**: Real performance is 20-40% worse than backtest.

**Costs to Include**:
- Spread: 1-3 pips (forex), $0.01-0.05 (stocks)
- Slippage: 0.5-1.0 pips
- Commission: $3-10 per trade
- Swap/rollover: For overnight positions

**The Fix**:
```python
# Add realistic costs
entry_price = close + (spread / 2) + slippage
exit_price = close - (spread / 2) - slippage
commission = 7  # $7 round trip
```

## Sin #5: Curve Fitting

**The Sin**: You added indicators until the backtest looked good.

**The Result**: A strategy that perfectly fits the past but fails in the future.

**How to Detect It**:
- More than 3-4 indicators
- Complex entry/exit rules
- Different parameters for different pairs
- Works on one timeframe but not others

**The Fix**:
- Keep it simple (2-3 indicators max)
- Use the same parameters across all instruments
- Test on multiple timeframes
- If it needs 10 conditions to work, it doesn't work

## Sin #6: Insufficient Data

**The Sin**: You backtested on too little data.

**The Result**: Your strategy hasn't seen enough market conditions.

**Minimum Data Requirements**:
- Trend following: 10+ years
- Mean reversion: 5+ years
- Day trading: 2+ years of tick data
- At least 1,000 trades in backtest

**The Fix**:
- Get more data
- Test across multiple market regimes
- Include crisis periods (2008, 2020, 2022)

## Sin #7: No Out-of-Sample Testing

**The Sin**: You optimized and tested on the same data.

**The Result**: You don't know if your strategy actually works.

**The Fix**:
1. Split data: 70% optimization, 30% testing
2. Optimize on training set
3. Test on unseen data
4. If OOS results are < 50% of IS results, reject the strategy

```python
split_point = int(len(data) * 0.7)
train_data = data[:split_point]
test_data = data[split_point:]

# Optimize on train
best_params = optimize(strategy, train_data)

# Test on unseen data
results = backtest(strategy, best_params, test_data)
```

## The Checklist

Before going live, verify:

- [ ] Walk-Forward Optimization passed
- [ ] Monte Carlo 1,000+ simulations passed
- [ ] Parameter sensitivity < 30% change
- [ ] No look-ahead bias
- [ ] Includes transaction costs (2x spread)
- [ ] Tested on 10+ years of data
- [ ] Out-of-sample results > 50% of in-sample
- [ ] Works on 3+ market regimes
- [ ] Maximum 3-4 indicators
- [ ] At least 1,000 trades in backtest

If you can't check all 10 boxes, don't go live.

---

*Building trading systems at [ZOO](https://zootechnologies.com). Our TITAN platform handles all 10 checks automatically.*
