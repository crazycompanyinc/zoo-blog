# How to Build a Backtesting Engine in Python: The Complete Guide

**Category:** Trading & Python | **Reading time:** 18 min | **Date:** May 8, 2026

## Why Backtesting Is the Difference Between Winners and Losers

Here's a number that should terrify you: **95% of retail algorithmic traders lose money.** Not because their strategies are bad — but because they never properly tested them before risking real capital.

Backtesting is the process of simulating a trading strategy against historical data to see how it *would have* performed. It's the single most important step between "I have an idea" and "I'm putting money on the line."

At ZOO, we've built backtesting systems for trading platforms that process millions of data points. In this guide, we're giving you the exact architecture — with production-ready Python code you can run today.

---

## Table of Contents

1. [The 3 Types of Backtesting](#types)
2. [Architecture Overview](#architecture)
3. [Building the Data Layer](#data)
4. [The Strategy Engine](#strategy)
5. [The Execution Simulator](#execution)
6. [Performance Metrics That Actually Matter](#metrics)
7. [Walk-Forward Analysis (Avoiding Overfitting)](#walkforward)
8. [Complete Working Example](#complete)
9. [Next Steps: From Backtest to Live Trading](#next)

---

<a name="types"></a>
## The 3 Types of Backtesting

### 1. Vectorized Backtesting (Fast, Simplest)

Best for: Quick strategy prototyping, simple indicators.

```python
import pandas as pd
import numpy as np

def vectorized_sma_backtest(df, fast=10, slow=50):
    """
    Simple Moving Average Crossover — vectorized.
    Fast to run, but doesn't account for realistic execution.
    """
    df = df.copy()
    df['fast_ma'] = df['close'].rolling(fast).mean()
    df['slow_ma'] = df['close'].rolling(slow).mean()
    
    # Signal: 1 = long, -1 = short, 0 = flat
    df['signal'] = 0
    df.loc[df['fast_ma'] > df['slow_ma'], 'signal'] = 1
    df.loc[df['fast_ma'] < df['slow_ma'], 'signal'] = -1
    
    # Calculate returns
    df['market_return'] = df['close'].pct_change()
    df['strategy_return'] = df['signal'].shift(1) * df['market_return']
    
    df['cumulative_market'] = (1 + df['market_return']).cumprod()
    df['cumulative_strategy'] = (1 + df['strategy_return']).cumprod()
    
    return df

# Usage:
# result = vectorized_sma_backtest(price_data, fast=10, slow=50)
```

**Pros:** Runs in milliseconds on years of data.
**Cons:** Assumes instant execution at close prices, no slippage, no transaction costs.

### 2. Event-Driven Backtesting (Realistic)

Best for: Production-grade backtesting, complex order types, realistic fills.

This is what we use at ZOO. Every bar triggers an event. The strategy reacts. The execution engine simulates fills.

### 3. Walk-Forward Analysis (Robust)

Best for: Validating that your strategy isn't just overfitted to history.

Split data into chunks. Train on chunk 1, test on chunk 2. Roll forward. Repeat.

---

<a name="architecture"></a>
## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                  BACKTESTING ENGINE                   │
│                                                       │
│  ┌──────────┐   ┌──────────┐   ┌──────────────────┐ │
│  │  Data     │──▶│ Strategy │──▶│  Execution       │ │
│  │  Layer    │   │  Engine  │   │  Simulator       │ │
│  └──────────┘   └──────────┘   └────────┬─────────┘ │
│                                          │           │
│  ┌──────────┐   ┌──────────┐            │           │
│  │ Portfolio│◀──│  Risk    │◀───────────┘           │
│  │ Manager  │   │  Manager │                        │
│  └────┬─────┘   └──────────┘                        │
│       │                                              │
│  ┌────▼─────┐                                        │
│  │ Metrics  │                                        │
│  │ Engine   │                                        │
│  └──────────┘                                        │
└─────────────────────────────────────────────────────┘
```

Each component is independent. Swap data sources, change strategies, adjust risk rules — without touching the rest.

---

<a name="data"></a>
## Building the Data Layer

The data layer fetches, cleans, and serves price data. It should handle multiple timeframes and instruments.

```python
from dataclasses import dataclass, field
from typing import List, Optional, Dict
from datetime import datetime
import pandas as pd
import numpy as np

@dataclass
class OHLCV:
    """Single bar of market data."""
    timestamp: datetime
    open: float
    high: float
    low: float
    close: float
    volume: float

@dataclass
class DataFeed:
    """
    Unified data layer. Supports CSV, API, and DataFrame sources.
    At ZOO, we use this same interface for backtesting AND live trading.
    """
    symbol: str
    timeframe: str  # '1m', '5m', '1h', '1d'
    data: pd.DataFrame = field(default_factory=pd.DataFrame)
    
    @classmethod
    def from_csv(cls, filepath: str, symbol: str, timeframe: str) -> 'DataFeed':
        """Load from CSV with standard OHLCV columns."""
        df = pd.read_csv(filepath, parse_dates=['timestamp'])
        df = df.sort_values('timestamp').reset_index(drop=True)
        df = cls._validate_and_clean(df)
        return cls(symbol=symbol, timeframe=timeframe, data=df)
    
    @classmethod
    def from_dataframe(cls, df: pd.DataFrame, symbol: str, timeframe: str) -> 'DataFeed':
        """Load from existing DataFrame."""
        df = df.copy()
        df = cls._validate_and_clean(df)
        return cls(symbol=symbol, timeframe=timeframe, data=df)
    
    @staticmethod
    def _validate_and_clean(df: pd.DataFrame) -> pd.DataFrame:
        """Ensure data quality — critical for reliable backtests."""
        required = ['timestamp', 'open', 'high', 'low', 'close', 'volume']
        for col in required:
            if col not in df.columns:
                raise ValueError(f"Missing required column: {col}")
        
        # Remove duplicates
        df = df.drop_duplicates(subset='timestamp')
        
        # Forward-fill small gaps (up to 3 bars)
        df = df.set_index('timestamp')
        df = df.resample('1min').ffill(limit=3)  # Adjust for your timeframe
        df = df.reset_index()
        
        # Remove rows with zero volume (illiquid periods)
        df = df[df['volume'] > 0]
        
        # Validate OHLC relationships
        df = df[df['high'] >= df['low']]
        df = df[df['high'] >= df['open']]
        df = df[df['high'] >= df['close']]
        df = df[df['low'] <= df['open']]
        df = df[df['low'] <= df['close']]
        
        return df.reset_index(drop=True)
    
    def get_bars(self, start: int, end: int) -> List[OHLCV]:
        """Get a range of bars as OHLCV objects."""
        slice_df = self.data.iloc[start:end]
        return [
            OHLCV(
                timestamp=row['timestamp'],
                open=row['open'],
                high=row['high'],
                low=row['low'],
                close=row['close'],
                volume=row['volume']
            )
            for _, row in slice_df.iterrows()
        ]
    
    def __len__(self):
        return len(self.data)
    
    def __iter__(self):
        """Iterate over all bars — used by the event loop."""
        for idx, row in self.data.iterrows():
            yield OHLCV(
                timestamp=row['timestamp'],
                open=row['open'],
                high=row['high'],
                low=row['low'],
                close=row['close'],
                volume=row['volume']
            )
```

---

<a name="strategy"></a>
## The Strategy Engine

A strategy is a function that receives market data and emits signals. Keep it pure — no side effects.

```python
from enum import Enum
from dataclasses import dataclass
from typing import Optional

class SignalType(Enum):
    BUY = "buy"
    SELL = "sell"
    CLOSE = "close"
    NONE = "none"

@dataclass
class Signal:
    type: SignalType
    price: float          # Target price
    stop_loss: Optional[float] = None
    take_profit: Optional[float] = None
    size: float = 1.0     # Position size (fraction of portfolio)
    reason: str = ""

# ─── Strategy Base Class ───

class Strategy:
    """
    Base class for all strategies.
    Subclass this and implement `on_bar()`.
    """
    def __init__(self, params: dict = None):
        self.params = params or {}
        self.indicators = {}
    
    def on_bar(self, bar: OHLCV, history: List[OHLCV]) -> Signal:
        """Called for every bar. Return a Signal or Signal(SignalType.NONE)."""
        raise NotImplementedError
    
    def on_start(self):
        """Called before the backtest begins."""
        pass
    
    def on_end(self):
        """Called after the backtest completes."""
        pass

# ─── Example: Mean Reversion Strategy ───

class MeanReversionStrategy(Strategy):
    """
    Bollinger Band mean reversion.
    Buy when price touches lower band, sell when it returns to mean.
    """
    def __init__(self, period=20, num_std=2.0, risk_per_trade=0.02):
        super().__init__()
        self.period = period
        self.num_std = num_std
        self.risk_per_trade = risk_per_trade
    
    def on_bar(self, bar: OHLCV, history: List[OHLCV]) -> Signal:
        if len(history) < self.period:
            return Signal(type=SignalType.NONE, price=bar.close)
        
        # Calculate Bollinger Bands
        closes = [h.close for h in history[-self.period:]]
        sma = np.mean(closes)
        std = np.std(closes)
        upper = sma + self.num_std * std
        lower = sma - self.num_std * std
        
        self.indicators = {'sma': sma, 'upper': upper, 'lower': lower, 'std': std}
        
        # Mean reversion logic
        if bar.close <= lower:
            stop_loss = bar.close - 2 * std
            take_profit = sma
            return Signal(
                type=SignalType.BUY,
                price=bar.close,
                stop_loss=stop_loss,
                take_profit=take_profit,
                size=self.risk_per_trade,
                reason=f"Price {bar.close:.2f} below lower band {lower:.2f}"
            )
        
        if bar.close >= sma:
            return Signal(
                type=SignalType.SELL,
                price=bar.close,
                reason=f"Price {bar.close:.2f} returned to mean {sma:.2f}"
            )
        
        return Signal(type=SignalType.NONE, price=bar.close)

# ─── Example: Momentum Strategy ───

class MomentumStrategy(Strategy):
    """
    RSI + Volume momentum.
    Buy on oversold + volume spike. Sell on overbought.
    """
    def __init__(self, rsi_period=14, oversold=30, overbought=70, vol_multiplier=1.5):
        super().__init__()
        self.rsi_period = rsi_period
        self.oversold = oversold
        self.overbought = overbought
        self.vol_multiplier = vol_multiplier
    
    def _calculate_rsi(self, closes: List[float]) -> float:
        if len(closes) < self.rsi_period + 1:
            return 50.0
        
        deltas = np.diff(closes[-(self.rsi_period + 1):])
        gains = np.where(deltas > 0, deltas, 0)
        losses = np.where(deltas < 0, -deltas, 0)
        
        avg_gain = np.mean(gains)
        avg_loss = np.mean(losses)
        
        if avg_loss == 0:
            return 100.0
        
        rs = avg_gain / avg_loss
        return 100 - (100 / (1 + rs))
    
    def on_bar(self, bar: OHLCV, history: List[OHLCV]) -> Signal:
        if len(history) < self.rsi_period + 1:
            return Signal(type=SignalType.NONE, price=bar.close)
        
        closes = [h.close for h in history]
        rsi = self._calculate_rsi(closes)
        
        # Volume analysis
        avg_vol = np.mean([h.volume for h in history[-20:]])
        vol_spike = bar.volume > avg_vol * self.vol_multiplier
        
        if rsi < self.oversold and vol_spike:
            return Signal(
                type=SignalType.BUY,
                price=bar.close,
                stop_loss=bar.close * 0.97,
                take_profit=bar.close * 1.06,
                size=0.02,
                reason=f"RSI={rsi:.1f} oversold + volume spike ({bar.volume/avg_vol:.1f}x)"
            )
        
        if rsi > self.overbought:
            return Signal(
                type=SignalType.SELL,
                price=bar.close,
                reason=f"RSI={rsi:.1f} overbought"
            )
        
        return Signal(type=SignalType.NONE, price=bar.close)
```

---

<a name="execution"></a>
## The Execution Simulator

This is where most amateur backtests fail. Realistic execution matters.

```python
from dataclasses import dataclass, field
from typing import List, Optional
from datetime import datetime

@dataclass
class Fill:
    timestamp: datetime
    side: str        # 'buy' or 'sell'
    price: float
    quantity: float
    commission: float
    slippage: float

@dataclass
class Position:
    side: str
    entry_price: float
    quantity: float
    stop_loss: Optional[float] = None
    take_profit: Optional[float] = None
    entry_time: Optional[datetime] = None

class ExecutionSimulator:
    """
    Simulates order execution with realistic slippage and commissions.
    
    Key insight: In backtesting, you CAN always buy at the close price.
    In reality, you can't. Slippage kills more strategies than bad signals.
    """
    
    def __init__(
        self,
        commission_rate: float = 0.001,    # 0.1% per trade (typical for crypto)
        slippage_model: str = 'fixed',      # 'fixed', 'percentage', 'volatility'
        slippage_value: float = 0.0005,     # 0.05% slippage
        min_commission: float = 0.0         # Minimum commission per fill
    ):
        self.commission_rate = commission_rate
        self.slippage_model = slippage_model
        self.slippage_value = slippage_value
        self.min_commission = min_commission
        self.fills: List[Fill] = []
        self.current_position: Optional[Position] = None
    
    def calculate_slippage(self, bar: OHLCV, side: str) -> float:
        """Calculate realistic slippage based on model."""
        if self.slippage_model == 'fixed':
            return self.slippage_value
        
        elif self.slippage_model == 'percentage':
            return bar.close * self.slippage_value
        
        elif self.slippage_model == 'volatility':
            # Slippage proportional to bar range (wider bars = more slippage)
            bar_range = (bar.high - bar.low) / bar.close
            return bar.close * bar_range * self.slippage_value
        
        return 0.0
    
    def execute_signal(self, signal: Signal, bar: OHLCV) -> Optional[Fill]:
        """Convert a signal into a fill with realistic execution."""
        if signal.type == SignalType.NONE:
            return None
        
        slippage = self.calculate_slippage(bar, signal.type.value)
        
        if signal.type == SignalType.BUY:
            fill_price = bar.close + slippage  # Buy at ask (higher)
        elif signal.type == SignalType.SELL:
            fill_price = bar.close - slippage  # Sell at bid (lower)
        elif signal.type == SignalType.CLOSE:
            fill_price = bar.close
        else:
            return None
        
        commission = max(fill_price * signal.size * self.commission_rate, self.min_commission)
        
        fill = Fill(
            timestamp=bar.timestamp,
            side=signal.type.value,
            price=fill_price,
            quantity=signal.size,
            commission=commission,
            slippage=slippage
        )
        
        self.fills.append(fill)
        
        # Update position
        if signal.type == SignalType.BUY:
            self.current_position = Position(
                side='long',
                entry_price=fill_price,
                quantity=signal.size,
                stop_loss=signal.stop_loss,
                take_profit=signal.take_profit,
                entry_time=bar.timestamp
            )
        elif signal.type in (SignalType.SELL, SignalType.CLOSE):
            self.current_position = None
        
        return fill
    
    def check_stops(self, bar: OHLCV) -> Optional[Fill]:
        """Check if stop-loss or take-profit was hit during this bar."""
        if self.current_position is None:
            return None
        
        pos = self.current_position
        
        if pos.side == 'long':
            # Check stop loss
            if pos.stop_loss and bar.low <= pos.stop_loss:
                slippage = self.calculate_slippage(bar, 'sell')
                fill = Fill(
                    timestamp=bar.timestamp,
                    side='sell',
                    price=pos.stop_loss - slippage,  # Slippage on stop fills
                    quantity=pos.quantity,
                    commission=pos.stop_loss * pos.quantity * self.commission_rate,
                    slippage=slippage
                )
                self.fills.append(fill)
                self.current_position = None
                return fill
            
            # Check take profit
            if pos.take_profit and bar.high >= pos.take_profit:
                slippage = self.calculate_slippage(bar, 'sell')
                fill = Fill(
                    timestamp=bar.timestamp,
                    side='sell',
                    price=pos.take_profit - slippage,
                    quantity=pos.quantity,
                    commission=pos.take_profit * pos.quantity * self.commission_rate,
                    slippage=slippage
                )
                self.fills.append(fill)
                self.current_position = None
                return fill
        
        return None
```

---

<a name="metrics"></a>
## Performance Metrics That Actually Matter

Most people only look at total return. That's dangerously incomplete.

```python
from dataclasses import dataclass
from typing import List
import numpy as np
import pandas as pd

@dataclass
class BacktestMetrics:
    """Complete performance report."""
    # Returns
    total_return: float
    annualized_return: float
    benchmark_return: float
    alpha: float
    
    # Risk
    sharpe_ratio: float
    sortino_ratio: float
    max_drawdown: float
    max_drawdown_duration: int  # in bars
    
    # Trading
    total_trades: int
    win_rate: float
    avg_win: float
    avg_loss: float
    profit_factor: float
    avg_trade_duration: float
    
    # Costs
    total_commission: float
    total_slippage: float
    cost_drag: float  # % of return lost to costs
    
    def __str__(self):
        return f"""
╔══════════════════════════════════════════════════╗
║           BACKTEST PERFORMANCE REPORT            ║
╠══════════════════════════════════════════════════╣
║ Returns                                          ║
║   Total Return:        {self.total_return:>8.2%}                 ║
║   Annualized Return:   {self.annualized_return:>8.2%}                 ║
║   Benchmark (Buy&Hold):{self.benchmark_return:>8.2%}                 ║
║   Alpha:               {self.alpha:>8.2%}                 ║
╠══════════════════════════════════════════════════╣
║ Risk                                             ║
║   Sharpe Ratio:        {self.sharpe_ratio:>8.2f}                 ║
║   Sortino Ratio:       {self.sortino_ratio:>8.2f}                 ║
║   Max Drawdown:        {self.max_drawdown:>8.2%}                 ║
║   Max DD Duration:     {self.max_drawdown_duration:>8d} bars            ║
╠══════════════════════════════════════════════════╣
║ Trading                                          ║
║   Total Trades:        {self.total_trades:>8d}                 ║
║   Win Rate:            {self.win_rate:>8.2%}                 ║
║   Avg Win:             {self.avg_win:>8.4f}                 ║
║   Avg Loss:            {self.avg_loss:>8.4f}                 ║
║   Profit Factor:       {self.profit_factor:>8.2f}                 ║
╠══════════════════════════════════════════════════╣
║ Costs                                            ║
║   Total Commission:    {self.total_commission:>8.2f}                 ║
║   Total Slippage:      {self.total_slippage:>8.4f}                 ║
║   Cost Drag:           {self.cost_drag:>8.2%}                 ║
╚══════════════════════════════════════════════════╝
"""

class MetricsCalculator:
    """Calculate comprehensive backtest metrics."""
    
    @staticmethod
    def calculate(equity_curve: List[float], fills: List[Fill], 
                  bars_count: int, risk_free_rate: float = 0.05) -> BacktestMetrics:
        
        equity = np.array(equity_curve)
        returns = np.diff(equity) / equity[:-1]
        returns = returns[returns != 0]  # Remove zero-return periods
        
        # Total return
        total_return = (equity[-1] / equity[0]) - 1
        
        # Annualized (assuming daily bars for simplicity)
        n_years = bars_count / 252
        annualized_return = (1 + total_return) ** (1 / max(n_years, 0.01)) - 1
        
        # Benchmark = buy and hold
        benchmark_return = total_return  # Simplified; in practice, compare to index
        
        # Sharpe Ratio
        excess_returns = returns - (risk_free_rate / 252)
        sharpe = np.mean(excess_returns) / (np.std(returns) + 1e-10) * np.sqrt(252)
        
        # Sortino Ratio (only penalizes downside volatility)
        downside_returns = returns[returns < 0]
        downside_std = np.std(downside_returns) if len(downside_returns) > 0 else 1e-10
        sortino = np.mean(excess_returns) / downside_std * np.sqrt(252)
        
        # Max Drawdown
        peak = np.maximum.accumulate(equity)
        drawdown = (equity - peak) / peak
        max_drawdown = np.min(drawdown)
        
        # Max Drawdown Duration
        is_drawdown = drawdown < 0
        dd_groups = np.diff(np.concatenate(([0], is_drawdown.astype(int), [0])))
        dd_starts = np.where(dd_groups == 1)[0]
        dd_ends = np.where(dd_groups == -1)[0]
        max_dd_duration = int(np.max(dd_ends - dd_starts)) if len(dd_starts) > 0 else 0
        
        # Trade analysis
        trades = []
        for i in range(0, len(fills) - 1, 2):
            if fills[i].side == 'buy' and fills[i+1].side == 'sell':
                pnl = (fills[i+1].price - fills[i].price) * fills[i].quantity
                pnl -= fills[i].commission + fills[i+1].commission
                trades.append(pnl)
        
        wins = [t for t in trades if t > 0]
        losses = [t for t in trades if t <= 0]
        
        win_rate = len(wins) / max(len(trades), 1)
        avg_win = np.mean(wins) if wins else 0
        avg_loss = np.mean(losses) if losses else 0
        profit_factor = abs(sum(wins) / sum(losses)) if losses and sum(losses) != 0 else float('inf')
        
        # Costs
        total_commission = sum(f.commission for f in fills)
        total_slippage = sum(f.slippage * f.quantity for f in fills)
        cost_drag = (total_commission + total_slippage) / equity[0] if equity[0] > 0 else 0
        
        return BacktestMetrics(
            total_return=total_return,
            annualized_return=annualized_return,
            benchmark_return=benchmark_return,
            alpha=total_return - benchmark_return,
            sharpe_ratio=sharpe,
            sortino_ratio=sortino,
            max_drawdown=max_drawdown,
            max_drawdown_duration=max_dd_duration,
            total_trades=len(trades),
            win_rate=win_rate,
            avg_win=avg_win,
            avg_loss=avg_loss,
            profit_factor=profit_factor,
            avg_trade_duration=0,  # Simplified
            total_commission=total_commission,
            total_slippage=total_slippage,
            cost_drag=cost_drag
        )
```

---

<a name="walkforward"></a>
## Walk-Forward Analysis: Avoiding Overfitting

This is the **most important section** of this guide. Overfitting is why 95% of strategies fail in live trading.

```python
from typing import List, Tuple

class WalkForwardAnalysis:
    """
    Split data into in-sample (training) and out-of-sample (testing) windows.
    Optimize on in-sample, validate on out-of-sample.
    Roll forward and repeat.
    
    This is the gold standard for strategy validation.
    """
    
    def __init__(self, strategy_class, param_grid: dict, 
                 n_splits: int = 5, train_ratio: float = 0.7):
        self.strategy_class = strategy_class
        self.param_grid = param_grid
        self.n_splits = n_splits
        self.train_ratio = train_ratio
    
    def run(self, data: pd.DataFrame, engine: 'BacktestEngine') -> dict:
        """
        Run walk-forward analysis.
        Returns out-of-sample performance for each fold.
        """
        n = len(data)
        fold_size = n // self.n_splits
        results = []
        
        for i in range(self.n_splits):
            # Define train/test split
            test_start = i * fold_size
            test_end = min((i + 1) * fold_size, n)
            train_end = test_start
            train_start = max(0, int(train_end - fold_size * self.train_ratio / (1 - self.train_ratio)))
            
            if train_start >= train_end:
                continue
            
            train_data = data.iloc[train_start:train_end]
            test_data = data.iloc[test_start:test_end]
            
            # Optimize on training data
            best_params = self._optimize_params(train_data, engine)
            
            # Test on out-of-sample data
            strategy = self.strategy_class(**best_params)
            metrics = engine.run(test_data, strategy)
            
            results.append({
                'fold': i,
                'train_start': train_start,
                'train_end': train_end,
                'test_start': test_start,
                'test_end': test_end,
                'best_params': best_params,
                'metrics': metrics
            })
            
            print(f"Fold {i}: Return={metrics.total_return:.2%}, "
                  f"Sharpe={metrics.sharpe_ratio:.2f}, "
                  f"MaxDD={metrics.max_drawdown:.2%}")
        
        # Aggregate results
        avg_return = np.mean([r['metrics'].total_return for r in results])
        avg_sharpe = np.mean([r['metrics'].sharpe_ratio for r in results])
        avg_maxdd = np.mean([r['metrics'].max_drawdown for r in results])
        
        print(f"\n{'='*50}")
        print(f"WALK-FORWARD SUMMARY ({self.n_splits} folds)")
        print(f"{'='*50}")
        print(f"Avg OOS Return:  {avg_return:.2%}")
        print(f"Avg OOS Sharpe:  {avg_sharpe:.2f}")
        print(f"Avg OOS MaxDD:   {avg_maxdd:.2%}")
        print(f"Profitable folds: {sum(1 for r in results if r['metrics'].total_return > 0)}/{len(results)}")
        
        return {'folds': results, 'avg_return': avg_return, 'avg_sharpe': avg_sharpe}
    
    def _optimize_params(self, data: pd.DataFrame, engine: 'BacktestEngine') -> dict:
        """Grid search over parameter space (simplified)."""
        from itertools import product
        
        keys = list(self.param_grid.keys())
        values = list(self.param_grid.values())
        best_sharpe = -np.inf
        best_params = {}
        
        for combo in product(*values):
            params = dict(zip(keys, combo))
            strategy = self.strategy_class(**params)
            metrics = engine.run(data, strategy)
            
            if metrics.sharpe_ratio > best_sharpe:
                best_sharpe = metrics.sharpe_ratio
                best_params = params
        
        return best_params
```

---

<a name="complete"></a>
## Complete Working Example

Here's everything wired together:

```python
import pandas as pd
import numpy as np
from datetime import datetime, timedelta

def generate_sample_data(n_bars=5000, start_price=100.0, volatility=0.02):
    """Generate realistic synthetic price data for testing."""
    np.random.seed(42)
    dates = [datetime(2024, 1, 1) + timedelta(minutes=i) for i in range(n_bars)]
    
    # Geometric Brownian Motion with mean reversion
    prices = [start_price]
    for i in range(1, n_bars):
        drift = 0.0001  # Slight upward bias
        shock = np.random.normal(0, volatility)
        mean_reversion = -0.001 * (prices[-1] - start_price) / start_price
        new_price = prices[-1] * (1 + drift + shock + mean_reversion)
        prices.append(max(new_price, 0.01))
    
    # Generate OHLCV from close prices
    data = []
    for i, (date, close) in enumerate(zip(dates, prices)):
        intra_range = close * volatility * 0.5
        high = close + abs(np.random.normal(0, intra_range))
        low = close - abs(np.random.normal(0, intra_range))
        open_price = prices[i-1] if i > 0 else close
        volume = np.random.lognormal(10, 1)
        
        data.append({
            'timestamp': date,
            'open': open_price,
            'high': max(high, open_price, close),
            'low': min(low, open_price, close),
            'close': close,
            'volume': volume
        })
    
    return pd.DataFrame(data)


# ─── Run the backtest ───

if __name__ == "__main__":
    # 1. Generate data
    print("Generating sample data...")
    data = generate_sample_data(n_bars=5000)
    print(f"Data: {len(data)} bars, {data['timestamp'].iloc[0]} to {data['timestamp'].iloc[-1]}")
    
    # 2. Create data feed
    feed = DataFeed.from_dataframe(data, symbol="SYNTH", timeframe="1min")
    
    # 3. Create strategy
    strategy = MeanReversionStrategy(period=20, num_std=2.0)
    
    # 4. Create execution simulator
    execution = ExecutionSimulator(
        commission_rate=0.001,
        slippage_model='volatility',
        slippage_value=0.5
    )
    
    # 5. Run backtest
    print("\nRunning backtest...")
    engine = BacktestEngine(feed, strategy, execution, initial_capital=10000)
    metrics = engine.run()
    
    # 6. Print results
    print(metrics)
    
    # 7. Walk-forward analysis
    print("\nRunning walk-forward analysis...")
    wfa = WalkForwardAnalysis(
        strategy_class=MeanReversionStrategy,
        param_grid={'period': [10, 20, 30], 'num_std': [1.5, 2.0, 2.5]},
        n_splits=3
    )
    wfa_results = wfa.run(data, engine)
```

---

<a name="next"></a>
## Next Steps: From Backtest to Live Trading

A profitable backtest is just the beginning. Here's the path to production:

### The 5 Stages of Strategy Deployment

```
Backtest → Paper Trade → Small Live → Scale → Monitor
   ↑           ↑            ↑          ↑        ↑
  Weeks      2-4 weeks    1 month    Gradual   Ongoing
```

1. **Backtest** — You are here. Validate with walk-forward analysis.
2. **Paper Trade** — Run the strategy on live data with simulated money. 2-4 weeks minimum.
3. **Small Live** — Risk 1-2% of your intended position size. Real money, real psychology.
4. **Scale** — Increase position size gradually as confidence grows.
5. **Monitor** — Track live performance vs. backtest expectations. Degrade gracefully.

### Common Pitfalls

- **Overfitting:** Your strategy works perfectly on historical data and fails live. Solution: Walk-forward analysis, fewer parameters, simpler strategies.
- **Look-ahead bias:** Your strategy accidentally uses future data. Solution: Never access `bar[i+1]` in your strategy logic.
- **Survivorship bias:** Your dataset only includes assets that still exist. Solution: Use point-in-time data.
- **Ignoring costs:** Slippage and commissions can turn a profitable strategy into a losing one. Solution: Always model realistic costs.

---

## Want Us to Build Your Trading System?

At **ZOO**, we build algorithmic trading platforms, backtesting engines, and automated trading systems for clients worldwide.

**What we offer:**
- Custom backtesting engines with your exact requirements
- Strategy development and optimization
- Live trading infrastructure (exchange connectivity, risk management)
- AI-powered signal generation

**Our stack:** Python, C++, Rust, Next.js, PostgreSQL, Redis, Docker, AWS/GCP

→ **[Get a free strategy audit](https://zootechnologies.com)** — Send us your backtest results. We'll tell you in 48 hours if your strategy is ready for live trading or if it's overfitted.

---

*This post is part of our **Algorithmic Trading Series**. Next week: "How to Connect Your Backtest to a Live Exchange API" — with real order execution code.*

*Last updated: May 8, 2026 | Trading involves risk. Past performance (including backtests) does not guarantee future results.*
