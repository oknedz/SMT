# ATR-Based Label Spacing Implementation

## Problem Solved

Labels need to adapt to different instruments with different price ranges and volatility:
- **Forex**: Small price values (e.g., 1.0500 for EURUSD)
- **Indices**: Medium values (e.g., 5000 for SPX, 19000 for NQ)
- **Crypto**: Large values (e.g., 50000 for BTC, 2500 for ETH)

Using percentage-based spacing (0.5%, 0.3%) doesn't work consistently across all instruments.

## Solution: ATR-Based Dynamic Spacing

### **What is ATR?**
Average True Range (ATR) measures market volatility and adapts to each instrument's price movement characteristics.

### **Benefits:**
1. ✅ **Adapts to volatility** - High volatility = more spacing, Low volatility = less spacing
2. ✅ **Works across all instruments** - Forex, indices, crypto automatically handled
3. ✅ **Adjusts to timeframe** - Different timeframes have different ATR values
4. ✅ **Price-independent** - Works for $1 forex or $50,000 crypto

## Implementation Details

### **Spacing Calculation:**
```pine
atrValue = ta.atr(14)  // 14-period ATR
baseOffset = 0.3 * atrValue      // First label: 30% of ATR away from line
labelSpacing = 0.2 * atrValue     // Each additional label: 20% of ATR apart
```

### **Stack Positions:**
- **90m label**: `linePrice + (0.3 * ATR)` for bearish, `linePrice - (0.3 * ATR)` for bullish
- **30m label**: `linePrice + (0.5 * ATR)` for bearish, `linePrice - (0.5 * ATR)` for bullish  
- **10m label**: `linePrice + (0.7 * ATR)` for bearish, `linePrice - (0.7 * ATR)` for bullish
- **3m label**: `linePrice + (0.9 * ATR)` for bearish, `linePrice - (0.9 * ATR)` for bullish

### **Examples:**

**Forex (EURUSD) - Low ATR:**
```
ATR = 0.0005
Base offset = 0.00015 (15 pips)
Stack spacing = 0.0001 (10 pips)
```

**Indices (NQ) - Medium ATR:**
```
ATR = 150 points
Base offset = 45 points
Stack spacing = 30 points
```

**Crypto (BTC) - High ATR:**
```
ATR = 1000
Base offset = 300
Stack spacing = 200
```

## Code Changes

### Both Functions Updated:

**1. `repositionAllLabels()` (lines 331-357)**
```pine
// Calculate ATR for dynamic spacing based on instrument volatility
atrValue = ta.atr(14)

// Use ATR-based spacing (adapts to instrument volatility and price range)
baseOffset = 0.3 * atrValue
labelSpacing = 0.2 * atrValue
stackOffset = stackPosition * labelSpacing
```

**2. `drawSMT()` - New label creation (lines 497-507)**
```pine
// Use ATR-based spacing (adapts to instrument volatility and price range)
atrValue = ta.atr(14)
baseOffset = 0.3 * atrValue
labelSpacing = 0.2 * atrValue
stackOffset = stackPosition * labelSpacing
```

## Fine-Tuning Options

If spacing needs adjustment, modify these multipliers:

**Tighter spacing:**
```pine
baseOffset = 0.2 * atrValue   // Closer to line
labelSpacing = 0.15 * atrValue // Labels closer together
```

**Wider spacing:**
```pine
baseOffset = 0.5 * atrValue   // Further from line
labelSpacing = 0.3 * atrValue // Labels further apart
```

## Testing Recommendations

Test on various instruments to verify spacing:
1. **Forex pairs** - EURUSD, GBPUSD (small price values)
2. **Indices** - SPX, NQ, ES (medium values)
3. **Crypto** - BTC, ETH (large values)
4. **Different timeframes** - 1m, 5m, 15m, 1h, 4h

All should show consistent, readable label spacing relative to the candles and line.
