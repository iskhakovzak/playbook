# Earnings Season Tracker & Confidence Scorer v2

Pine Script v6 indicator for TradingView that automates tracking of post-earnings trading setups, confidence scoring, and risk management.

## Features

### 8 Setup Types Tracked
- **Gap Up Long/Short** — continuation/reversal after upward earnings gap
- **Gap Down Long/Short** — continuation/reversal after downward earnings gap
- **Green Turret** — first candle green after gap up (short setup)
- **Pre-Market Move** — high pre-market volume signals
- **Second Movement** — continuation beyond initial 30-min range
- **Daily Breakout** — price breaks prior day's high/low

### EPS Surprise Analysis
- Calculates `(actual - estimate) / |estimate|` surprise percentage
- Tracks consecutive beat/miss streaks
- Displays EPS data on chart labels and dashboard
- EPS surprise boosts confidence scoring

### Gap Analysis
- **Gap fill detection** — tracks if gaps fill within configurable N bars
- **Expected move** — calculates average ± stdev of historical earnings gaps
- **Large gap classification** — separate tracking for gaps above threshold

### Confidence Scoring (0–100)
Multi-factor composite score using parameter-weighted multipliers:
- Market cap match
- Volume threshold
- Sector alignment
- Daily breakout presence
- Recency boost (recent > season probability)
- **EPS surprise boost** (new)
- **Large gap bonus** (new)
- **Gap fill rate penalty** (new)

### Enhanced Backtesting
- Win rate, profit factor, cumulative P&L
- **Expectancy** (avg P&L per trade)
- **Kelly criterion** (optimal bet fraction)
- **Max consecutive wins/losses**
- **3 position sizing modes**: Fixed, Confidence-Based, Kelly

### Dashboard Table (10 columns × 16 rows)
- Per-setup statistics: total, wins, probability, recent probability, **trend arrows**, filtered probability, confidence score, **avg EPS surprise**, filtered total
- **Rich tooltips** on all column headers explaining each metric
- Active signal display with confidence color-coding
- Filter status showing all active criteria
- **Enhanced stats row**: expectancy, Kelly %, max streaks, suggested risk %
- **Market context row**: price position, weekly trend, EPS streak, expected move, gap fill rate, relative volume, historical volatility
- **EPS & Gap info row**: current earnings data with beat/miss highlighting

### Filters
- Market cap (above/below threshold)
- Sector
- Pre-market volume
- **Gap size** (small / large relative to threshold) — new
- **Price position** (at 52w highs / middle / at 52w lows) — new

### Alerts
- High confidence setup detected (≥70)
- Earnings bar detected
- Probability shift (>10%)
- **EPS beat detected** — new

### Visual Enhancements
- **Compact label mode** — short setup names for cleaner charts
- **Trend arrows** (▲/▼/─) showing probability direction
- **Gap fill markers** — X-cross plotted when gap fills
- **Continuation/Reversal labels** on earnings bars
- **Large gap indicator** (⚡) on chart labels
- EPS surprise columns in score pane

## Requirements

- TradingView **Pro+** subscription (for `request.earnings()` and `request.financial()`)
- Daily timeframe recommended

## Installation

1. Open TradingView → Pine Editor
2. Create new indicator
3. Paste contents of `src/earnings_tracker.pine`
4. Click "Add to chart"
5. Configure inputs in Settings panel

## File Structure

```
src/
  earnings_tracker.pine   — Main indicator (960 lines, Pine Script v6)
```

## Changelog

### v2 (Current)
- Added EPS surprise analysis with beat/miss streak tracking
- Added gap fill detection and tracking
- Added expected move calculation from historical gaps
- Enhanced confidence scoring with EPS, gap fill, and large gap factors
- Added Kelly criterion and confidence-based position sizing
- Added expectancy, max consecutive wins/losses to backtest engine
- Added gap size and price position filters
- Added trend arrows showing probability direction
- Added rich tooltips to all dashboard columns
- Added market context row (weekly trend, relative volume, historical volatility)
- Added EPS & gap detail row with beat/miss highlighting
- Added compact label mode
- Added gap fill markers on chart
- Added EPS beat alert condition
- Cleaned up unused variables and reduced validator warnings
- Refactored setup recording with helper function (DRY)
- Code grew from 803 → 960 lines with significantly more functionality

### v1
- Initial implementation with 8 setup types
- Basic confidence scoring with 5 multipliers
- Dashboard table with probabilities and backtest summary
- 3 alert conditions
