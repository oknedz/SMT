# Manual Label Offset Input

## Implementation Summary

### **New Input Setting Added:**

```pine
labelOffset = input.int(50, "Label Offset (Pips/Points)", minval=10, maxval=200, 
    tooltip="Distance of labels from SMT lines. Uses mintick multiplication (50 = 5 pips for forex, ~12.5 pts for NQ/ES)", 
    group="Display")
```

### **How It Works:**

**Base Offset:** `labelOffset × syminfo.mintick`
- User sets single value (default: 50)
- Applies to both bearish (above) and bullish (below) labels

**Stack Spacing:** `(labelOffset × 0.6) × syminfo.mintick`
- Automatically calculated as 60% of base offset
- Ensures proportional spacing between stacked labels

### **Examples Across Instruments:**

| Setting | Instrument | mintick | Base Offset | Stack Spacing |
|---------|------------|---------|-------------|---------------|
| **50** | EURUSD | 0.00001 | 5 pips | 3 pips |
| **50** | NQ | 0.25 | 12.5 pts | 7.5 pts |
| **50** | BTC | 1.0 | 50 pts | 30 pts |
| **100** | EURUSD | 0.00001 | 10 pips | 6 pips |
| **100** | NQ | 0.25 | 25 pts | 15 pts |
| **100** | BTC | 1.0 | 100 pts | 60 pts |

### **User Adjustment Guide:**

**Labels too close to line?**
- Increase value: 50 → 75 → 100

**Labels too far from line?**
- Decrease value: 50 → 35 → 25

**Labels overlapping when stacked?**
- Increase value (increases both base and stack spacing proportionally)

### **Benefits:**

1. ✅ **Single setting** - Controls both directions (bearish/bullish)
2. ✅ **Simple & intuitive** - One number to adjust
3. ✅ **Works across all instruments** - Auto-adapts via mintick
4. ✅ **No complex calculations** - No ATR, just fixed multiplier
5. ✅ **Proportional stacking** - Stack spacing scales with base offset

### **Visual Example (Setting = 50):**

**Bearish:**
```
[90m BTC SMT]  ← 50 × mintick above
[30m ES SMT]   ← 80 × mintick above (50 + 30)
[10m BTC SMT]  ← 110 × mintick above (50 + 60)
─────────────  ← SMT line
```

**Bullish:**
```
─────────────  ← SMT line
[90m BTC SMT]  ← 50 × mintick below
[30m ES SMT]   ← 80 × mintick below
[10m BTC SMT]  ← 110 × mintick below
```

### **ATR Logic Completely Removed:**
- ✅ No `ta.atr()` calls
- ✅ No ATR variables
- ✅ Cleaner, faster code
- ✅ More predictable behavior
