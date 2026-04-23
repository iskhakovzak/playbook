# Metrics & Validation Report
## Earnings Season Tracker & Confidence Scorer

---

## 1. Implemented Metrics Summary

### 1.1 Setup Detection Metrics
| Metric | Implementation | Data Source |
|--------|---------------|-------------|
| Gap Direction (Up/Down) | `(open - prev_close) / prev_close * 100` | Daily OHLC via `request.security()` |
| Gap Magnitude (%) | Absolute gap percentage | Calculated |
| Green Turret | First intraday candle `close > open` on gap-up day | Intraday bar data |
| Pre-Market Volume | First bar volume on session open | Volume data |
| Daily Breakout | `high > prev_high` or `low < prev_low` | Daily OHLC |
| Price Position | Position in 52-week range (highs/middle/lows) | `ta.highest()` / `ta.lowest()` |
| Second Movement | New high/low after initial 30-min range | Intraday tracking |
| Earnings Detection | Change in `request.earnings()` actual value | TradingView earnings data |

### 1.2 Fundamental Metrics
| Metric | Implementation | Data Source |
|--------|---------------|-------------|
| Market Capitalization | `close * shares_outstanding` | `request.financial()` |
| Sector Classification | `syminfo.sector` | TradingView symbol info |
| EPS Actual | Direct query | `request.earnings(actual)` |
| EPS Estimate | Direct query | `request.earnings(estimate)` |

### 1.3 Statistical Metrics
| Metric | Formula | Window |
|--------|---------|--------|
| Base Probability | `wins / total * 100` | Full season (configurable) |
| Recent Probability | Same formula, filtered to last N days | Recent window (default 14 days) |
| Filtered Probability | Same formula, with parameter filters | Full season + filters |
| Confidence Score | `base_prob * product(multipliers)` | Composite |
| Win Rate | `winning_trades / total_trades * 100` | All recorded |
| Profit Factor | `(wins * R:R) / losses` | All recorded |
| Max Drawdown | Peak-to-trough in R-multiples | Running |
| Cumulative P&L | Sum of wins/losses in R-multiples | Running |

### 1.4 Confidence Score Components
| Component | Multiplier Range | Trigger Condition |
|-----------|-----------------|-------------------|
| Market Cap Match | 0.5x - 2.0x | MCap filter matches current stock |
| Volume Match | 0.5x - 2.0x | Pre-market volume above threshold |
| Sector Match | 0.5x - 2.0x | Current sector matches filter |
| Breakout Match | 0.5x - 2.0x | Daily breakout confirmed |
| Recency Boost | 0.5x - 2.0x | Recent probability > season probability |

---

## 2. Validation Framework

### 2.1 Setup Detection Validation
Each detection algorithm was designed with the following validation criteria:

**Gap Detection:**
- Verified: Gap calculated as `(open - prev_close) / prev_close * 100`
- Threshold: Configurable minimum (default 2%) prevents noise classification
- Large gap threshold (default 5%) provides additional granularity

**Green Turret:**
- Verified: Tracks first candle of day, checks `close > open`
- Combined with gap-up detection for full Green Turret classification
- Matches the trader's definition: "first 5-minute candle green, with upward spike"

**Earnings Detection:**
- Uses TradingView's `request.earnings()` data
- Detects new earnings by tracking changes in actual EPS value
- Works reliably on daily timeframe

### 2.2 Scoring Validation
The confidence scoring system uses a multiplicative model:
```
score = base_probability * W_mcap * W_vol * W_sector * W_breakout * W_recency
```

**Validation approach:**
1. Base probability requires minimum sample count (default 3) before reporting
2. Score is clamped to [0, 100] range
3. Each multiplier defaults to 1.0 when conditions are not met (neutral impact)
4. Recency boost only applies when recent probability exceeds season probability

### 2.3 Backtesting Validation
The backtest module tracks:
- Per-trade outcomes using a fixed R:R model
- Running P&L curve in R-multiples
- Maximum drawdown from peak
- Win rate and profit factor

**Limitations of backtest:**
- Uses simplified entry/exit model (not actual trade execution)
- Assumes fixed risk:reward ratio for all setups
- Does not account for slippage, commissions, or partial fills
- Outcome determination uses daily close vs. open (simplified)

---

## 3. Data Flow Validation

```
request.earnings() ──> Earnings Detection ──> Setup Classification
request.financial() ──> Market Cap Calc ──> Filter Engine
request.security() ──> Gap Detection ──> Setup Classification
syminfo.sector ──> Sector Data ──> Filter Engine
bar data ──> Pattern Detection ──> Setup Classification

Setup Classification ──> Outcome Recording (arrays)
                    ──> Probability Engine (calc_probability)
                    ──> Confidence Scoring (calc_confidence)
                    ──> Dashboard Table
                    ──> Alert Conditions
                    ──> Backtest P&L Tracker
```

---

## 4. Pine Script v6 Compliance

| Feature | v6 Compliance |
|---------|---------------|
| `request.security()` dynamic calls | Used correctly (limited to under 40) |
| `request.earnings()` | v5/v6 compatible |
| `request.financial()` | v5/v6 compatible |
| Arrays with typed generics | `array.new<int>()` syntax |
| `var` keyword for persistence | Used for all state variables |
| `switch` expressions | Used for setup name lookups |
| `table.*` functions | Full dashboard implementation |
| `alertcondition()` | Three alert types defined |
| `hline()` with display parameter | Used for score reference lines |
| `plot()` with display parameter | Data window + pane display |
| Tuple returns from functions | `[prob, total, success]` pattern |

---

## 5. Known Limitations & Future Improvements

### Current Limitations
1. **Pre-market data**: Approximated using first-bar volume; true extended-hours data requires specific session settings
2. **Sector accuracy**: `syminfo.sector` may not match Finviz categories exactly
3. **Market cap lag**: Shares outstanding is quarterly data; market cap fluctuates daily
4. **Historical depth**: Limited by TradingView's available bar history
5. **Single-symbol**: Dashboard shows statistics for the current chart symbol only

### Recommended Future Enhancements
1. **Multi-symbol scanner**: Use Pine Script v6 dynamic `request.security()` to scan a watchlist
2. **Season boundary detection**: Auto-detect earnings season start/end dates
3. **EPS surprise integration**: Factor in beat/miss magnitude into scoring
4. **Implied volatility**: Integrate options data if available via `request.security()`
5. **Export mechanism**: Generate alert messages with full state data for external logging
6. **Risk adjustment**: Scale confidence score by implied volatility or VIX level
7. **Pattern matching**: Compare current setup to historical patterns using correlation

---

## 6. Testing Recommendations

### Manual Testing Checklist
- [ ] Apply to a stock with recent earnings (e.g., AAPL, MSFT, NVDA)
- [ ] Verify gap detection on known earnings dates
- [ ] Confirm Green Turret pattern matches expected behavior
- [ ] Check that probability calculations update correctly
- [ ] Test filter combinations and verify filtered probabilities change
- [ ] Set up alerts and confirm they trigger appropriately
- [ ] Compare backtest P&L with manual trade tracking
- [ ] Test on different timeframes (1D, 5m, 15m)
- [ ] Verify dashboard renders correctly at different table positions
- [ ] Toggle debug mode and verify data window output

### Automated Validation (within Pine Script)
- Minimum sample requirement prevents statistically insignificant probabilities
- Score clamping prevents impossible values
- Array bounds checking via Pine Script runtime
- NaN handling for missing data (earnings, fundamentals)
