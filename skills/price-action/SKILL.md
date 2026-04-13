# Price Action Trading Analysis

## When to Use
- Zen asks "how's the trading bot?", "any signals?", "check price action"
- Zen asks about BTC/ETH market structure, support/resistance
- During daily earning routine checks

## Current System
The Trader agent (`conductor/agents/trader/exec.py`) runs every 20 minutes with a hybrid approach:

### Strategy 1: Momentum (guarded)
- 24h price change threshold (3%, raised from 2% to filter noise)
- BUY if ETH up >3%, SELL if down >3%, HOLD otherwise
- **Position guard**: SELL signals suppressed when held qty < minimum trade qty (prevents sell-spam on empty/dust positions)

### Strategy 2: Price Action with Indicator Confirmation (hybrid)
- **S/R Zones**: Detected from 4H and 1D swing highs/lows
- **Trend Structure**: HH/HL (uptrend), LH/LL (downtrend), range via MA20/MA10
- **Entry Patterns**:
  - Breakout-retest long: price breaks resistance, retests from above
  - Support bounce: bullish reversal candle at support in uptrend/range
  - Resistance rejection: bearish reversal candle at resistance in downtrend/range
- **Indicator Confirmation Layer** (new):
  - RSI(14) filter: longs rejected if RSI >= 70 (overbought), shorts rejected if RSI <= 30 (oversold)
  - Volume filter: signal requires current volume > 1.2x average (20-period)
- **Multi-Factor Confidence Scoring** (new):
  - Pattern quality: breakout-retest (0.30), support-bounce (0.25), resistance-rejection (0.20)
  - Trend alignment: with-trend (0.25), range (0.10), counter-trend (0.00)
  - RSI confirmation: favorable RSI (0.20), neutral RSI (0.10)
  - Volume confirmation: 2x+ (0.25), 1.5x+ (0.15), 1.2x+ (0.10)
  - **Minimum threshold: 0.50** — signals below this are logged but not traded
- **Risk Management**: Stop beyond S/R zone (0.2% buffer), min 2:1 R:R

### Trade Pairs
BTCUSDT, ETHUSDT, SOLUSDT, BNBUSDT, AVAXUSDT, LINKUSDT

## Check Trader Status
```bash
cat ~/claw/conductor/agents/trader/state.json
```

## Check Signals
```bash
cat ~/claw/conductor/agents/trader/signals.json
```

## Check Trade History
```bash
tail -20 ~/claw/conductor/agents/trader/trade-history.jsonl
```

## Check Paper P&L
```bash
cat ~/claw/conductor/agents/trader/paper-pnl.json
```

## Configuration
- Mode: Live trading on Binance spot
- Pairs: BTCUSDT, ETHUSDT, SOLUSDT, BNBUSDT, AVAXUSDT, LINKUSDT
- Trade amount: $7-$10 per symbol (see SYMBOL_QTY map)
- Max open PA positions: 3
- Min R:R: 2.0 (PA only)
- Min PA confidence: 0.50
- Momentum threshold: 3%
- PA signals require: S/R proximity + candlestick pattern + trend alignment + RSI confirmation + volume confirmation + confidence >= 0.50
