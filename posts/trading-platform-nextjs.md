# Building a Full Trading Platform with Next.js and TradingView

**Category:** Development | **Reading time:** 12 min

## Introduction

We built FX Replay — a complete trading backtesting platform — using Next.js, TradingView charts, and Yahoo Finance data. Here's the technical deep dive.

## The Stack

- **Frontend**: Next.js 14, React, Tailwind CSS
- **Charts**: TradingView Lightweight Charts
- **Data**: Yahoo Finance API
- **State**: Zustand
- **Auth**: NextAuth.js
- **Database**: PostgreSQL (Neon)
- **Payments**: Stripe Subscriptions
- **Deploy**: Vercel

## Architecture

### Chart Component

The core of the platform is the TradingView chart:

```javascript
import { createChart } from 'lightweight-charts';

const chart = createChart(container, {
  width: container.clientWidth,
  height: 500,
  layout: {
    background: { color: '#0a0a1a' },
    textColor: '#d1d5db',
  },
  grid: {
    vertLines: { color: '#1f2937' },
    horzLines: { color: '#1f2937' },
  },
});

const candleSeries = chart.addCandlestickSeries();
candleSeries.setData(ohlcData);
```

### Data Pipeline

Yahoo Finance data flows through our API:

1. **Fetch**: `GET https://query1.finance.yahoo.com/v8/finance/chart/{symbol}`
2. **Parse**: Extract OHLCV data
3. **Transform**: Normalize format for TradingView
4. **Cache**: Redis for frequently accessed symbols

### Backtesting Engine

The backtesting engine runs client-side:

```javascript
function backtest(strategy, data) {
  const trades = [];
  let position = null;
  
  for (let i = 0; i < data.length; i++) {
    const signal = strategy.generateSignal(data, i);
    
    if (signal === 'BUY' && !position) {
      position = { entry: data[i].close, time: data[i].time };
    } else if (signal === 'SELL' && position) {
      trades.push({
        entry: position.entry,
        exit: data[i].close,
        pnl: data[i].close - position.entry,
      });
      position = null;
    }
  }
  
  return calculateMetrics(trades);
}
```

## Key Challenges

### 1. Chart Performance

Loading 10+ years of tick data kills the browser. Solution:
- Server-side pagination
- Web Workers for data processing
- Canvas rendering (TradingView handles this)

### 2. Real-Time Data

Yahoo Finance doesn't offer WebSocket. Solution:
- Polling every 5 seconds for active symbols
- Server-Sent Events for push updates
- Optimistic UI updates

### 3. State Management

Complex chart state (drawings, indicators, positions). Solution:
- Zustand for global state
- Local state for chart-specific data
- Undo/redo with command pattern

## Results

- **Load time**: < 2s for 1 year of daily data
- **Backtest speed**: 10 years of data in < 1s
- **Chart rendering**: 60fps even with 5,000+ candles
- **Bundle size**: 180KB gzipped (chart library is the biggest chunk)

## Lessons Learned

1. **Don't reinvent the wheel**. TradingView's library is battle-tested.
2. **Cache aggressively**. Financial data doesn't change every second.
3. **Web Workers are your friend**. Offload heavy computation.
4. **Test with real data**. Mock data lies.

## Try It Yourself

FX Replay is open source. Check it out on [GitHub](https://github.com/crazycompanyinc/fxreplay-clone).

---

*Built by the ZOO engineering team.*
