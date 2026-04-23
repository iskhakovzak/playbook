# Earnings Season Tracker & Confidence Scorer

A TradingView Pine Script v6 indicator that automatically tracks trading statistics for earnings-season setups, computes real-time confidence scores, and provides a comprehensive on-chart dashboard with alerting capabilities.

## Overview

This system implements a methodology for systematically tracking and scoring earnings-season trading setups, inspired by statistical approaches to gap trading on earnings reports. It answers three core questions:
- **What to trade?** (which setups have the highest probability)
- **How to trade?** (which parameter combinations improve odds)
- **When to trade?** (recent trend vs. season-long statistics)

## Features

### Setup Detection (8 Types)
| Setup | Description |
|-------|-------------|
| **Gap Up Long** | Gap up on earnings, trade continuation (long) |
| **Gap Up Short** | Gap up on earnings, trade reversal (short) |
| **Gap Down Long** | Gap down on earnings, trade reversal (long) |
| **Gap Down Short** | Gap down on earnings, trade continuation (short) |
| **Green Turret** | First candle after earnings open is green (reversal short) |
| **Pre-Market Move** | Significant pre-market volume activity |
| **Second Movement** | Post-open continuation after initial move and pullback |
| **Daily Breakout** | Price breaks previous day's high/low range |

### Filtering Parameters
- **Market Capitalization** (above/below threshold, e.g., 100B)
- **Pre-Market Volume** (minimum threshold)
- **Sector** (Technology, Healthcare, Financial, etc.)
- **Gap Size** (minimum % for classification)
- **Price Position** (at 52-week highs, middle, or lows)
- **Daily Breakout** (confirmed breakout of previous range)

### Confidence Scoring (0-100)
- Base probability from historical binary outcomes
- Parameter-weighted multipliers (market cap, volume, sector, breakout, recency)
- Rolling window statistics (full season + configurable recent window)
- Four confidence levels: HIGH (>=70), MEDIUM (50-69), LOW (30-49), AVOID (<30)

### Dashboard
- Real-time on-chart table showing all setup statistics
- Per-setup: total count, wins, probability %, recent probability, filtered probability, confidence score
- Active signal display with confidence level
- Backtest summary (win rate, profit factor, P&L in R-multiples, max drawdown)
- Current stock metadata (ticker, sector, market cap, gap %, price position)

### Alerting
- High confidence setup detection (score >= 70)
- New earnings report detection
- Significant probability shift (>10% change)

### Backtesting
- Hypothetical P&L tracking with configurable risk:reward ratio
- Win rate and profit factor calculation
- Maximum drawdown tracking

## Installation

1. Open TradingView and navigate to the Pine Editor
2. Create a new indicator script
3. Copy the contents of `src/earnings_tracker.pine` into the editor
4. Click "Add to Chart"
5. Configure inputs via the indicator settings panel

## Configuration

### Key Settings

| Setting | Default | Description |
|---------|---------|-------------|
| Lookback Bars | 500 | Historical bars for statistics |
| Season Length | 45 days | Earnings season window |
| Recent Window | 14 days | Short-term trend window |
| Min Samples | 3 | Minimum setups before showing probability |
| Min Gap % | 2.0% | Threshold to classify as a gap |
| MCap Threshold | 100B | Market cap filter threshold |
| Min PM Volume | 100K | Pre-market volume minimum |
| Risk:Reward | 2.0 | Backtesting R:R ratio |

### Scoring Multipliers

Each parameter can boost the confidence score when conditions are met:
- Market Cap Multiplier: 1.2x (default)
- Volume Multiplier: 1.15x (default)
- Sector Multiplier: 1.1x (default)
- Breakout Multiplier: 1.1x (default)
- Recency Multiplier: 1.2x (default)

## Usage Guide

### Daily Workflow
1. Apply indicator to stocks with upcoming/recent earnings
2. Check the dashboard table for setup probabilities
3. Apply filters (market cap, sector, volume) to refine probabilities
4. Monitor confidence score for the active setup
5. Set alerts for high-confidence signals

### Interpreting the Dashboard
- **Prob%**: Overall probability for the full season
- **14D Prob%**: Probability in the last 14 days (detect trend shifts)
- **Filt Prob%**: Probability with all active filters applied
- **Score**: Composite confidence score (0-100) with level indicator (H/M/L/X)

### Best Practices
- Compare full-season vs. recent probabilities to detect changing dynamics
- Use market cap + sector filters to find the best parameter combinations
- Monitor the backtest row to validate that high-confidence setups are profitable
- Adjust the recent window (14D default) based on how quickly setups change

## Technical Requirements

- TradingView Pro/Pro+/Premium account (for `request.earnings()` and `request.financial()`)
- Pine Script v6 runtime
- Recommended timeframe: Daily (1D) for most accurate gap and earnings detection

## File Structure

```
earnings-tracker/
  src/
    earnings_tracker.pine    # Main indicator script
  docs/
    architecture.md          # System architecture document
    metrics_report.md        # Metrics and validation report
  README.md                  # This file
```

## Limitations

- Pine Script cannot access external APIs (no Finviz integration)
- Limited to 40 dynamic `request.*()` calls per execution
- Historical data depends on TradingView's available bar history
- Pre-market volume detection approximates using first-bar volume
- Market cap calculated from shares outstanding (quarterly data) x current price
- No true machine learning; uses weighted parameter scoring instead

## License

This Pine Script code is subject to the terms of the Mozilla Public License 2.0.
