# Earnings Season Trading Statistics & Confidence Scoring System
## Architecture & Implementation Proposal (Pine Script v6)

---

## 1. Executive Summary

This document outlines the architecture for an automated earnings-season trading statistics tracker built as a TradingView Pine Script v6 indicator. The system implements the methodology described by the trader Vitaly: tracking gap-up/gap-down setups, "Green Turret" patterns, pre-market activity, market cap filtering, sector analysis, and computing real-time confidence scores with probability-based filtering.

**Key deliverables:**
- A Pine Script v6 indicator with on-chart dashboard
- Real-time confidence scoring for earnings setups
- Statistical aggregation with parameter-based filtering
- Alerting system for high-confidence setups
- Backtesting validation framework

---

## 2. System Architecture Overview

```
+------------------------------------------------------------------+
|                    EARNINGS TRACKER INDICATOR                      |
+------------------------------------------------------------------+
|                                                                    |
|  [INPUT LAYER]                                                     |
|    - User inputs (market cap thresholds, volume filters, sector)   |
|    - request.earnings() -> EPS actual/estimate data                |
|    - request.financial() -> market cap, shares outstanding         |
|    - request.security() -> multi-timeframe OHLCV data              |
|    - Built-in bar data -> gap detection, pattern recognition       |
|                                                                    |
|  [DETECTION LAYER]                                                 |
|    - Gap classifier (Gap Up / Gap Down + magnitude)                |
|    - Green Turret detector (first 5-min candle direction)          |
|    - Pre-market volume analyzer                                    |
|    - Daily breakout detector                                       |
|    - Price position classifier (at highs / middle / lows)          |
|    - Second movement detector (post-open continuation/reversal)    |
|                                                                    |
|  [SCORING ENGINE]                                                  |
|    - Base probability from historical binary outcomes              |
|    - Parameter-weighted confidence multipliers                     |
|    - Composite confidence score (0-100)                            |
|    - Rolling window statistics (season / 2-week / custom)          |
|                                                                    |
|  [STORAGE LAYER]                                                   |
|    - Arrays for historical setup outcomes                          |
|    - Maps for parameter->probability lookups                       |
|    - Persistent var state across bars                              |
|                                                                    |
|  [OUTPUT LAYER]                                                    |
|    - On-chart dashboard table (statistics summary)                 |
|    - Chart labels for detected setups                              |
|    - Background coloring for confidence levels                     |
|    - Alert conditions for high-confidence signals                  |
|    - Plotted score series                                          |
|                                                                    |
+------------------------------------------------------------------+
```

---

## 3. Data Sources & Technical Constraints

### 3.1 Available Data in Pine Script v6
| Data Source | Function | What It Provides |
|---|---|---|
| `request.earnings()` | Earnings data | Actual EPS, estimated EPS, report dates |
| `request.financial()` | Fundamentals | Market cap (calculated), shares outstanding |
| `request.security()` | Multi-TF data | OHLCV from any timeframe (now dynamic in v6) |
| Built-in variables | Bar data | `open`, `high`, `low`, `close`, `volume`, `time` |
| `syminfo.*` | Symbol info | Sector, industry, type, currency |

### 3.2 Key Pine Script v6 Limitations
- **40 dynamic `request.*()` calls** per execution
- **No external API access** (cannot pull from Finviz directly)
- **No file I/O** (all state must live in arrays/maps/vars)
- **~500 bars lookback** for most indicators (configurable with `max_bars_back`)
- **Memory limits** on arrays/maps (~100K elements)
- **No ML libraries** (must implement scoring algorithms manually)
- **Compilation timeout** of 2 minutes

### 3.3 Workarounds
- Market cap is computed as `close * shares_outstanding` via `request.financial()`
- Sector data available via `syminfo.sector`
- Pre-market data accessible via extended hours sessions
- Historical earnings dates detected via `request.earnings()` with gaps

---

## 4. Post-Open Setup Inventory

### 4.1 Setup Classification Taxonomy

| Setup ID | Setup Name | Description | Detection Logic |
|---|---|---|---|
| `GAP_UP_LONG` | Gap Up Long | Gap up on earnings, trade continuation (long) | `open > prev_close` after earnings |
| `GAP_UP_SHORT` | Gap Up Short | Gap up on earnings, trade reversal (short) | `open > prev_close` + reversal pattern |
| `GAP_DOWN_LONG` | Gap Down Long | Gap down on earnings, trade reversal (long) | `open < prev_close` + reversal pattern |
| `GAP_DOWN_SHORT` | Gap Down Short | Gap down on earnings, trade continuation (short) | `open < prev_close` after earnings |
| `GREEN_TURRET` | Green Turret | First 5-min candle green after earnings open | First intraday candle `close > open` |
| `PREMARKET_MOVE` | Pre-market Move | Significant pre-market price action | Extended hours volume + price change |
| `SECOND_MOVE` | Second Movement | Post-open continuation after initial move | Price makes new high/low after pullback |
| `DAILY_BREAKOUT` | Daily Breakout | Price breaks previous day's range | `high > prev_high` or `low < prev_low` |

### 4.2 Filtering Parameters

| Parameter | Type | Values | Source |
|---|---|---|---|
| Market Cap | Float | Threshold (e.g., 100B) | `request.financial()` |
| Sector | String | Technology, Healthcare, etc. | `syminfo.sector` |
| Pre-market Volume | Float | Min threshold (e.g., 100K, 200K) | `request.security()` extended |
| Gap Size (%) | Float | Magnitude of gap | Calculated from OHLC |
| Price Position | Enum | AT_HIGHS, MIDDLE, AT_LOWS | Relative to 52-week range |
| Daily Breakout | Bool | Yes/No | Previous day's range comparison |

---

## 5. Confidence Scoring Schema

### 5.1 Base Score Calculation
```
base_probability = successful_setups / total_setups
```
Each setup type maintains its own running tally of outcomes (0/1).

### 5.2 Parameter-Weighted Confidence
```
confidence_score = base_probability
    * mcap_multiplier          // e.g., 1.2 if >100B and historically better
    * volume_multiplier        // e.g., 1.15 if PM vol > 200K
    * sector_multiplier        // e.g., 1.1 if tech sector performs well
    * breakout_multiplier      // e.g., 1.1 if daily breakout confirmed
    * recency_multiplier       // e.g., 1.2 if last 2 weeks show strong trend
```

### 5.3 Composite Score Output
```
final_score = clamp(confidence_score * 100, 0, 100)

Confidence Levels:
  - HIGH (>=70):   Strong signal, consider increased risk
  - MEDIUM (50-69): Standard signal, base risk
  - LOW (30-49):   Weak signal, reduced risk or skip
  - AVOID (<30):   Poor probability, do not trade
```

### 5.4 Rolling Window Statistics
- **Full Season**: All earnings in current quarter
- **Last 2 Weeks**: Most recent 10 trading days
- **Custom Window**: User-configurable lookback period

---

## 6. Storage Schema (In-Script)

### 6.1 Core Data Structures (Pine Script v6)
```pine
// Setup outcome tracking
var array<int>   setup_types     = array.new<int>()       // Setup type enum
var array<int>   setup_outcomes  = array.new<int>()       // 0 or 1
var array<float> setup_gaps      = array.new<float>()     // Gap %
var array<float> setup_mcaps     = array.new<float>()     // Market cap at time
var array<int>   setup_times     = array.new<int>()       // Bar timestamps
var array<float> setup_volumes   = array.new<float>()     // Pre-market volume
var array<int>   setup_sectors   = array.new<int>()       // Sector encoded

// Probability cache (map-based)
var map<string, float> prob_cache = map.new<string, float>()

// Rolling statistics
var map<string, int>   stat_total   = map.new<string, int>()
var map<string, int>   stat_success = map.new<string, int>()
```

### 6.2 State Management
- `var` keyword ensures persistence across bars
- Maps provide O(1) lookup for probability queries
- Arrays store raw data for recalculation with different filters

---

## 7. Aggregation & Reporting Logic

### 7.1 Probability Calculation Engine
```
For each setup_type in [GAP_UP_LONG, GAP_UP_SHORT, ...]:
    total = count(outcomes where type == setup_type)
    success = count(outcomes where type == setup_type AND outcome == 1)
    probability = success / total

    // With filters applied:
    For each filter_combo in [mcap_filter, volume_filter, sector_filter, ...]:
        filtered_total = count(outcomes matching filter_combo)
        filtered_success = count(successful outcomes matching filter_combo)
        filtered_probability = filtered_success / filtered_total
```

### 7.2 Dashboard Table Layout
```
+---------------------------------------------------------------+
| EARNINGS SETUP TRACKER           Season: Q1 2026              |
+---------------------------------------------------------------+
| Setup Type      | Total | Won | Prob% | 2W Prob% | Score     |
|-----------------|-------|-----|-------|----------|-----------|
| Gap Up Long     |  25   | 13  | 52%   | 60%      | 62 MED    |
| Gap Up Short    |  25   | 10  | 40%   | 35%      | 38 LOW    |
| Gap Down Long   |  18   |  7  | 39%   | 45%      | 42 LOW    |
| Gap Down Short  |  18   | 11  | 61%   | 58%      | 65 MED    |
| Green Turret    |  20   | 14  | 70%   | 72%      | 78 HIGH   |
| Pre-mkt Move    |  30   | 19  | 63%   | 67%      | 71 HIGH   |
+---------------------------------------------------------------+
| FILTERS: MCap>100B | Vol>100K | Sector: All                  |
+---------------------------------------------------------------+
| Active Signal: GREEN TURRET | Confidence: 78 HIGH             |
+---------------------------------------------------------------+
```

---

## 8. Visualization Components

1. **Dashboard Table** (top-right): Statistics summary with color-coded cells
2. **Chart Labels**: Markers at earnings bars showing setup type + score
3. **Background Highlighting**: Color bands behind bars based on confidence level
4. **Score Plot**: Separate pane showing confidence score over time
5. **Gap Visualization**: Lines showing gap magnitude
6. **Filter Status**: Display of active filters and their impact

---

## 9. Alerting System

| Alert Condition | Trigger | Data Included |
|---|---|---|
| High Confidence Setup | Score >= 70 | Setup type, score, probability |
| New Earnings Detected | Earnings bar identified | Ticker, gap %, direction |
| Probability Shift | 2-week prob changes >10% from season | Old prob, new prob, setup |
| Filter Match | Setup matches all active filters | Full parameter set |

---

## 10. Backtesting & Validation Plan

### 10.1 Within Pine Script
- Track hypothetical P&L using fixed risk/reward ratios (1:2, 1:3)
- Compare setup score vs actual outcome correlation
- Calculate Sharpe-like ratio for each setup type
- Measure score accuracy: `correct_predictions / total_predictions`

### 10.2 Parameter Optimization
- User-adjustable inputs for all thresholds
- Score calibration via lookback period adjustment
- A/B comparison of filter combinations

### 10.3 ML Approximation (within Pine Script constraints)
- Simple logistic regression approximation using weighted parameters
- Feature importance ranking based on historical correlation
- No neural networks possible, but weighted scoring achieves similar outcome

---

## 11. Telemetry & Logging

Within Pine Script, "logging" is limited to:
- **Table cells** showing diagnostic data (toggle-able)
- **Labels** on chart with debug information
- **Plot values** for score components (visible in data window)
- **Tooltips** on table cells with detailed breakdowns
- **Alert messages** containing full state dumps

---

## 12. Implementation Plan

| Phase | Description | Deliverable |
|---|---|---|
| Phase 1 | Core detection engine (gaps, patterns) | Detection functions |
| Phase 2 | Scoring engine + probability calculations | Scoring system |
| Phase 3 | Dashboard table + visualizations | UI components |
| Phase 4 | Alerting system | Alert conditions |
| Phase 5 | Backtesting validation | P&L tracking |
| Phase 6 | Documentation + report | Final report |

---

## 13. File Structure

```
earnings-tracker/
  +-- src/
  |   +-- earnings_tracker.pine       # Main indicator script
  +-- docs/
  |   +-- architecture.md             # This document
  |   +-- metrics_report.md           # Validation results
  +-- README.md                       # Usage guide
```
