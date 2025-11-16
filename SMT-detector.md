# SMT Detector - Technical Documentation

## Overview

The SMT (Smart Money Technique) Detector is a sophisticated TradingView Pine Script indicator designed to identify divergences between correlated financial instruments. It tracks multiple time-based cycles and detects when the main chart instrument and comparison symbols move in opposing directions, potentially signaling important market turning points.

---

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [Architecture](#architecture)
3. [Configuration](#configuration)
4. [Cycle Detection Logic](#cycle-detection-logic)
5. [SMT Detection Types](#smt-detection-types)
6. [Visual Display](#visual-display)
7. [Alert System](#alert-system)
8. [Code Structure](#code-structure)

---

## Core Concepts

### What is SMT (Smart Money Technique)?

SMT occurs when two correlated instruments make diverging highs or lows. For example:
- **Bearish SMT**: Main chart makes a higher high, but the comparison symbol makes a lower high
- **Bullish SMT**: Main chart makes a lower low, but the comparison symbol makes a higher low

This divergence can indicate:
- Liquidity hunting by institutional traders
- Potential reversal points
- Weakness in the current trend

### Why Multiple Timeframes?

The indicator tracks four cycle durations:
- **90-minute cycles**: Captures intraday institutional rotation
- **30-minute cycles**: Medium-term intraday movements
- **10-minute cycles**: Short-term scalping opportunities
- **3-minute cycles**: Ultra-short-term (1-minute charts only)

---

## Architecture

### Data Structure

The indicator uses a custom type to track cycle data:

```pine
type CycleData
    int startTime        // When the cycle began
    int startBar         // Bar index of cycle start
    float mainHigh       // Highest price on main chart
    float mainLow        // Lowest price on main chart
    int mainHighBar      // Bar index of main chart high
    int mainLowBar       // Bar index of main chart low
    float s1High         // Highest price on Symbol 1
    float s1Low          // Lowest price on Symbol 1
    int s1HighBar        // Bar index of Symbol 1 high
    int s1LowBar         // Bar index of Symbol 1 low
    float s2High         // Highest price on Symbol 2
    float s2Low          // Lowest price on Symbol 2
    int s2HighBar        // Bar index of Symbol 2 high
    int s2LowBar         // Bar index of Symbol 2 low
    bool active          // Whether the cycle is currently running
```

### State Management

The indicator maintains:
- **Active cycles**: Currently running cycles for each timeframe
- **Previous cycles**: Completed cycles stored for comparison (cleared daily)
- **Line history**: Up to 4 SMT lines per detection type
- **Label history**: Text labels for each SMT line

---

## Configuration

### Settings Panel Organization

#### 1. Comparison Symbols
```pine
enableSymbol1 = input.bool(true, "Enable Symbol 1", group="Comparison Symbols", inline="sym1")
symbol1 = input.symbol("MEXC:BTCUSDT.P", "", confirm=false, group="Comparison Symbols", inline="sym1")
enableSymbol2 = input.bool(false, "Enable Symbol 2", group="Comparison Symbols", inline="sym2")
symbol2 = input.symbol("ES1!", "", confirm=false, group="Comparison Symbols", inline="sym2")
```

**Features:**
- Two comparison symbols available
- Inline layout (checkbox + label + input on one line)
- Symbol picker integration with `input.symbol()`
- Individual enable/disable per symbol

#### 2. Cycle Timeframes
```pine
enable90min = input.bool(true, "90-Minute Cycle", group="Cycle Timeframes")
enable30min = input.bool(true, "30-Minute Cycle", group="Cycle Timeframes")
enable10min = input.bool(true, "10-Minute Cycle", group="Cycle Timeframes")
enable3min = input.bool(true, "3-Minute Cycle", group="Cycle Timeframes")
```

**Special Condition for 3-Minute Cycle:**
```pine
isValidTimeframeFor3Min = timeframe.in_seconds(timeframe.period) <= 60
```
The 3-minute cycle only activates on 1-minute or seconds-based charts.

#### 3. Session Filter
```pine
enableSession = input.bool(true, "Enable Session Filter", group="Session Filter")
sessionStart = input.session("0230-1600", "Session Time", group="Session Filter")
sessionTimezone = input.string("America/New_York", "Timezone", 
    options=["America/New_York", "America/Chicago", "Europe/London"], 
    group="Session Filter")
```

**Purpose:** Restricts SMT detection to specific trading hours (e.g., New York session).

---

## Cycle Detection Logic

### Starting a New Cycle

```pine
startNewCycle(cycle) =>
    cycle.startTime := time
    cycle.startBar := bar_index
    cycle.mainHigh := high
    cycle.mainLow := low
    cycle.mainHighBar := bar_index
    cycle.mainLowBar := bar_index
    cycle.s1High := s1_high
    cycle.s1Low := s1_low
    // ... initialize all comparison symbol data
    cycle.active := true
```

**Trigger:** When not currently in an active cycle and within session hours.

### Updating a Cycle

```pine
updateCycle(cycle) =>
    if high > cycle.mainHigh
        cycle.mainHigh := high
        cycle.mainHighBar := bar_index
    if low < cycle.mainLow
        cycle.mainLow := low
        cycle.mainLowBar := bar_index
    // ... update comparison symbols similarly
```

**Process:** Continuously tracks highest high and lowest low for all symbols during the cycle.

### Completing a Cycle

```pine
shouldCompleteCycle(cycle, duration) =>
    cycle.active and (time - cycle.startTime) >= duration

completeCycle(cycle, prevCycles) =>
    if array.size(prevCycles) >= 10
        array.shift(prevCycles)  // Remove oldest cycle
    array.push(prevCycles, cycle)  // Store completed cycle
    cycle.active := false
```

**Trigger:** When the cycle has been active for its full duration (90, 30, 10, or 3 minutes).

### Daily Reset

```pine
newDay = dayofmonth(time) != dayofmonth(time[1])
if newDay and not na(time[1])
    array.clear(prevCycles90)
    array.clear(prevCycles30)
    array.clear(prevCycles10)
    array.clear(prevCycles3)
    
    // Reset active cycles to prevent cross-day comparisons
    cycle90.active := false
    cycle30.active := false
    cycle10.active := false
    cycle3.active := false
    
    // Clear all line and label history
    while array.size(lineHistory90_0) > 0
        line.delete(array.shift(lineHistory90_0))
        label.delete(array.shift(labelHistory90_0))
    // ... repeat for all history arrays
```

**Purpose:** 
- Prevents SMT detection across day boundaries
- Clears visual elements from previous day
- Ensures fresh start each trading day

---

## SMT Detection Types

The indicator detects **8 different SMT patterns** per cycle:

### Standard SMT (Indices 0-3)

#### Bearish SMT (Index 0, 2)
```pine
detectBearishSMT(mainCurr, mainPrev, compCurr, compPrev) =>
    mainCurr > mainPrev and compCurr < compPrev
```

**Interpretation:** Main chart makes higher high, comparison makes lower high → Bearish divergence

#### Bullish SMT (Index 1, 3)
```pine
detectBullishSMT(mainCurr, mainPrev, compCurr, compPrev) =>
    mainCurr < mainPrev and compCurr > compPrev
```

**Interpretation:** Main chart makes lower low, comparison makes higher low → Bullish divergence

### Inverse SMT (Indices 4-7)

#### Bearish Inverse SMT (Index 4, 6)
```pine
detectBearishInverseSMT(mainCurr, mainPrev, compCurr, compPrev) =>
    mainCurr < mainPrev and compCurr > compPrev
```

**Interpretation:** Main chart makes lower high, comparison makes higher high → Alternative bearish signal

#### Bullish Inverse SMT (Index 5, 7)
```pine
detectBullishInverseSMT(mainCurr, mainPrev, compCurr, compPrev) =>
    mainCurr > mainPrev and compCurr < compPrev
```

**Interpretation:** Main chart makes higher low, comparison makes lower low → Alternative bullish signal

### Index Mapping

| Index | Symbol | Type | Direction |
|-------|--------|------|-----------|
| 0 | Symbol 1 | Standard | Bearish |
| 1 | Symbol 1 | Standard | Bullish |
| 2 | Symbol 2 | Standard | Bearish |
| 3 | Symbol 2 | Standard | Bullish |
| 4 | Symbol 1 | Inverse | Bearish |
| 5 | Symbol 1 | Inverse | Bullish |
| 6 | Symbol 2 | Inverse | Bearish |
| 7 | Symbol 2 | Inverse | Bullish |

---

## Visual Display

### Line Drawing

```pine
drawOrUpdateSMT(mainBar, mainPrice, compBar, compPrice, labelText, lineArray, labelArray, isBearish) =>
    line newLine = na
    label newLabel = na
    if not na(mainBar) and not na(compBar)
        // Draw line connecting the two diverging points
        newLine := line.new(mainBar, mainPrice, compBar, compPrice, 
            color=color.black, width=1, style=line.style_solid)
        
        // Position label on the right side
        labelBar = math.max(mainBar, compBar)
        labelPrice = mainBar > compBar ? mainPrice : compPrice
        textYloc = isBearish ? yloc.abovebar : yloc.belowbar
        
        newLabel := label.new(labelBar, labelPrice, labelText, 
            style=label.style_none, color=color.black, textcolor=color.black, 
            size=size.small, yloc=textYloc, xloc=xloc.bar_index)
```

**Visual Elements:**
- **Black line** connects the diverging points
- **Text label** positioned at the rightmost end
  - Above the line for bearish SMT
  - Below the line for bullish SMT
- **Label format**: "SMT-{SYMBOL} {TIMEFRAME}"
  - Example: "SMT-BTC.P 90"

### Line History Management

```pine
// Add to history
array.push(lineArray, newLine)
array.push(labelArray, newLabel)

// Remove oldest if exceeds limit (4 lines per type)
if array.size(lineArray) > MAX_LINES_PER_TYPE
    oldLine = array.shift(lineArray)
    oldLabel = array.shift(labelArray)
    if not na(oldLine)
        line.delete(oldLine)
    if not na(oldLabel)
        label.delete(oldLabel)
```

**Limit:** Maximum of 4 SMT lines per detection type to keep chart clean.

---

## Alert System

### Alert Implementation (90-min and 30-min only)

```pine
processSMT90(cycle, prevCycles, timeLabel) =>
    alertTriggered = false
    alertMessage = ""
    
    if cycle.active and array.size(prevCycles) > 0
        // SMT detection logic...
        if detectBearishSMT(...)
            alertTriggered := true
            alertMessage := "Bearish SMT 90-min - " + getSymbolAbbr(symbol1)
        // ... check all other SMT types
    
    // Trigger alert
    if alertTriggered
        alert(alertMessage, alert.freq_once_per_bar)
    
    alertTriggered
```

### Alert Messages

**90-Minute Cycle:**
- "Bearish SMT 90-min - {SYMBOL}"
- "Bullish SMT 90-min - {SYMBOL}"
- "Bearish Inverse SMT 90-min - {SYMBOL}"
- "Bullish Inverse SMT 90-min - {SYMBOL}"

**30-Minute Cycle:**
- "Bearish SMT 30-min - {SYMBOL}"
- "Bullish SMT 30-min - {SYMBOL}"
- "Bearish Inverse SMT 30-min - {SYMBOL}"
- "Bullish Inverse SMT 30-min - {SYMBOL}"

**Alert Frequency:** `alert.freq_once_per_bar` prevents multiple alerts on the same bar.

**Note:** 10-minute and 3-minute cycles do not trigger alerts.

---

## Code Structure

### File Organization

```
SMT.pine
├── Metadata (//@version=5, indicator(), etc.)
├── Input Configuration
│   ├── Comparison Symbols
│   ├── Cycle Timeframes
│   └── Session Filter
├── Data Fetching (request.security)
├── Type Definitions
│   └── CycleData
├── State Variables
│   ├── Active cycles (cycle90, cycle30, etc.)
│   ├── Previous cycles arrays
│   └── Line/label history arrays
├── Helper Functions
│   ├── getSymbolAbbr()
│   ├── inSession()
│   └── getCycleDuration()
├── SMT Detection Functions
│   ├── detectBearishSMT()
│   ├── detectBullishSMT()
│   ├── detectBearishInverseSMT()
│   └── detectBullishInverseSMT()
├── Cycle Management Functions
│   ├── startNewCycle()
│   ├── updateCycle()
│   ├── shouldCompleteCycle()
│   └── completeCycle()
├── Drawing Functions
│   └── drawOrUpdateSMT()
├── Processing Functions
│   ├── processSMT90()
│   ├── processSMT30()
│   ├── processSMT10()
│   └── processSMT3()
└── Main Execution Logic
    ├── Daily reset
    ├── Cycle start/update/complete
    └── SMT processing and display
```

### Execution Flow

1. **Bar-by-Bar Processing**
   ```
   For each new bar:
   ├── Check if new day → Reset if true
   ├── Check session filter → Skip if outside hours
   ├── For each cycle timeframe:
   │   ├── Start new cycle if inactive
   │   ├── Update cycle if active
   │   └── Complete cycle if duration reached
   └── Process SMT detection for all cycles
   ```

2. **SMT Detection Process**
   ```
   For each active cycle:
   ├── Loop through previous cycles
   ├── Check for SMT patterns (8 types)
   ├── Store matching cycles
   ├── Draw lines and labels
   ├── Manage line history (max 4 per type)
   └── Trigger alerts (90-min and 30-min only)
   ```

### Key Variables

| Variable | Type | Purpose |
|----------|------|---------|
| `cycle90`, `cycle30`, `cycle10`, `cycle3` | CycleData | Active cycle trackers |
| `prevCycles90`, etc. | array<CycleData> | Completed cycles for comparison |
| `lineHistory90_0` to `lineHistory90_7` | array<line> | Line storage per SMT type |
| `labelHistory90_0` to `labelHistory90_7` | array<label> | Label storage per SMT type |
| `MAX_LINES_PER_TYPE` | int | Maximum lines per SMT type (4) |

---

## Performance Considerations

### Memory Management

1. **Cycle Cleanup**
   - Maximum 10 previous cycles stored per timeframe
   - Oldest cycles removed when limit exceeded
   - Daily reset clears all historical data

2. **Visual Element Limits**
   - 4 lines maximum per SMT type
   - 8 SMT types × 4 cycles = 32 line types total
   - Maximum ~128 lines on chart (4 × 32)

3. **Daily Reset Strategy**
   ```pine
   // Prevents memory buildup across days
   if newDay and not na(time[1])
       array.clear(prevCycles90)
       // ... clear all arrays
       cycle90.active := false
       // ... reset all cycles
   ```

### Optimization Techniques

1. **Early Exit Conditions**
   ```pine
   if not cycle.active
       return  // Skip processing if cycle not active
   
   if not inSession()
       return  // Skip if outside trading hours
   ```

2. **Efficient Array Operations**
   ```pine
   // Shift instead of recreating arrays
   if array.size(lineArray) > MAX_LINES_PER_TYPE
       array.shift(lineArray)  // O(n) but faster than rebuild
   ```

3. **Single Pass Detection**
   - Checks all 8 SMT types in one loop through previous cycles
   - Stops checking a type once a match is found

---

## Usage Guide

### Basic Setup

1. **Add Indicator to Chart**
   - Add to desired chart (works on any timeframe for 90, 30, 10-min cycles)
   - Use 1-minute or seconds chart for 3-minute cycle

2. **Configure Comparison Symbols**
   - Click symbol input to open symbol picker
   - Select correlated instruments (e.g., BTC futures vs spot)
   - Enable/disable second symbol as needed

3. **Select Active Cycles**
   - Enable relevant timeframes for your trading style
   - Higher timeframes (90-min) for swing trades
   - Lower timeframes (10-min, 3-min) for scalping

4. **Set Session Filter** (Optional)
   - Enable session filter to focus on liquid hours
   - Select timezone matching your trading hours
   - Adjust session time range as needed

### Interpretation

**Strong Signals:**
- SMT on 90-minute cycle = Major institutional divergence
- Multiple SMT types at once = Higher confidence
- SMT + price action confirmation = Best entries

**Weak Signals:**
- SMT during low volume = Less reliable
- Conflicting SMT signals = Wait for clarity
- SMT against strong trend = Use caution

### Alert Configuration

1. Click **"Create Alert"** button in TradingView
2. Select **"SMT Detector"** as condition
3. Configure notifications:
   - Email alerts for 90-minute cycles
   - Push notifications for 30-minute cycles
   - Webhook integration for automated trading
4. Set expiration and frequency as desired

---

## Advanced Concepts

### Symbol Abbreviation Logic

```pine
getSymbolAbbr(sym) =>
    parts = str.split(sym, ":")
    symbolPart = array.size(parts) > 1 ? array.get(parts, 1) : array.get(parts, 0)
    cleanSymbol = str.replace_all(str.replace_all(symbolPart, "1!", ""), "USDT", "")
    str.substring(cleanSymbol, 0, math.min(str.length(cleanSymbol), 6))
```

**Process:**
1. Split by ":" to separate exchange from symbol
2. Remove common suffixes ("1!" for continuous contracts, "USDT")
3. Truncate to 6 characters for label brevity

**Examples:**
- "MEXC:BTCUSDT.P" → "BTC.P"
- "ES1!" → "ES"
- "BINANCE:ETHUSDT" → "ETH"

### Cycle Duration Calculation

```pine
getCycleDuration(cycleMinutes) => cycleMinutes * 60 * 1000
```

**Explanation:**
- Input: Minutes (90, 30, 10, 3)
- Output: Milliseconds
- Conversion: Minutes → Seconds (×60) → Milliseconds (×1000)

### Session Filter Logic

```pine
inSession() => not enableSession or not na(time(timeframe.period, sessionStart, sessionTimezone))
```

**Behavior:**
- Returns `true` if session filter disabled
- Returns `true` if current time within session range
- Returns `false` if outside session hours and filter enabled

---

## Troubleshooting

### Common Issues

**Issue:** No SMT lines appearing
- **Check:** Ensure comparison symbols are enabled
- **Check:** Verify cycles are enabled
- **Check:** Confirm you're within session hours (if filter enabled)
- **Check:** Wait for at least 2 complete cycles to compare

**Issue:** Too many lines on chart
- **Solution:** Disable unused cycles (10-min, 3-min)
- **Solution:** Disable Symbol 2 if not needed
- **Note:** Max 4 lines per type is already enforced

**Issue:** Cross-day SMT detection
- **Fixed:** Code resets all cycles and clears history at day boundaries
- **Verify:** Check that you see clean slate at market open

**Issue:** 3-minute cycle not working
- **Check:** Are you on 1-minute or seconds chart?
- **Reason:** 3-minute cycle only available on sub-minute timeframes

**Issue:** Alerts not triggering
- **Check:** Alerts only work for 90-min and 30-min cycles
- **Check:** Ensure alert is properly configured in TradingView
- **Check:** Verify alert frequency is not being throttled

### Performance Issues

**Slow Chart Loading:**
- Reduce enabled cycles (disable 3-min and 10-min)
- Disable Symbol 2 if not needed
- Reduce historical bars loaded in TradingView settings

**Memory Warnings:**
- Normal on longer timeframes with deep history
- Daily reset should prevent memory buildup
- Consider upgrading TradingView plan for more resources

---

## Future Enhancements

### Potential Additions

1. **Customizable Cycle Durations**
   - Allow user-defined cycle lengths
   - Support for 60-min, 120-min cycles

2. **Additional SMT Types**
   - Triple divergence detection (3+ symbols)
   - Volume-weighted SMT
   - Percentage-based divergence thresholds

3. **Enhanced Visualization**
   - Color-coded lines by strength
   - Gradient fills between diverging lines
   - Dashboard summary panel

4. **Statistical Analysis**
   - SMT success rate tracking
   - Historical performance metrics
   - Heatmap of SMT frequency

5. **Machine Learning Integration**
   - Pattern recognition for SMT quality
   - Predictive strength scoring
   - Adaptive threshold adjustment

---

## Best Practices

### Trading with SMT

1. **Confirmation Required**
   - Don't trade SMT alone
   - Wait for price action confirmation
   - Use with support/resistance levels

2. **Multiple Timeframe Analysis**
   - Check higher timeframe SMT first
   - Use lower timeframes for entry timing
   - Align SMT across timeframes for strength

3. **Risk Management**
   - Set stop losses beyond SMT points
   - Position size based on SMT strength
   - Consider multiple SMT confluences

4. **Session Awareness**
   - Most reliable during liquid hours
   - Avoid overnight gaps
   - Be cautious during news events

### Development Best Practices

1. **Code Maintenance**
   - Comment complex logic
   - Use descriptive variable names
   - Maintain consistent naming conventions

2. **Testing Strategy**
   - Test on multiple symbols
   - Verify across different timeframes
   - Backtest with historical data

3. **Version Control**
   - Document changes in comments
   - Maintain changelog
   - Keep backup of working versions

---

## Conclusion

The SMT Detector is a comprehensive tool for identifying institutional-level divergences across multiple timeframes. By tracking cycle-based price relationships between correlated instruments, it provides traders with early warning signals for potential reversals and trend changes.

Key strengths:
- ✅ Multi-timeframe analysis (90, 30, 10, 3-minute cycles)
- ✅ Dual symbol comparison capability
- ✅ Clean visual presentation with line history management
- ✅ Automated alerts for major signals
- ✅ Session-based filtering for optimal signal quality
- ✅ Daily reset prevents cross-day false signals

The indicator is designed for both discretionary traders seeking visual signals and systematic traders who can integrate the alerts into automated strategies.

---

## Version History

**Current Version:** 5 (Pine Script v5)

### Changelog

- Added 3-minute cycle detection for 1-minute charts
- Implemented line history management (4 lines per type)
- Added daily reset to prevent cross-day comparisons
- Integrated alert system for 90-min and 30-min cycles
- Enhanced settings panel with symbol picker
- Optimized memory management and performance

---

## Support & Resources

### Documentation
- Pine Script Reference: https://www.tradingview.com/pine-script-docs/
- TradingView Community: https://www.tradingview.com/community/

### Contact
For bugs, feature requests, or questions about the SMT Detector, please refer to the original code repository or TradingView publishing page.

---

**Disclaimer:** This indicator is for educational and informational purposes only. It does not constitute financial advice. Always perform your own analysis and risk assessment before making trading decisions.
