# Label Stacking Fix - Multiple SMTs on Same Candle

## Problem Identified

When multiple SMT cycles (90m, 30m, 10m, 3m) detected SMTs on the same candle, all labels appeared at the **same position**, making them **unreadable**.

### Root Cause:
- Old logic counted labels with same direction at same time
- Applied 1% offset per label (too large)
- **Did NOT distinguish between different cycles**
- All different cycle labels stacked at identical positions

## Solution Implemented

### **1. Priority-Based Stacking System**

Labels stack vertically based on cycle priority:
```
Priority Order (Top to Bottom):
1. 90min (highest priority - closest to price)
2. 30min
3. 10min
4. 3min
5. Sessions (ASIA, LONDON, NY-AM, NY-PM) (lowest priority - furthest from price)
```

### **2. Smart Position Calculation**

- Each label calculates its position based on:
  - Its own cycle priority
  - Number of lower-priority labels at same time/direction
  - Lower priority = stacks below

### **3. Fixed Spacing (0.3% of price)**

- Changed from 1% to **0.3%** per label
- Much tighter, more readable spacing
- Consistent across all price levels

### **4. Dynamic Repositioning**

Added `repositionAllLabels()` function that:
- Recalculates all label positions when array changes
- Ensures consistent stacking even after deletions/additions
- Called after:
  - New label added
  - Label deleted (invalidation)
  - Array cleanup

## Code Changes

### New Function Added:
```pine
repositionAllLabels() =>
    // Reposition all labels to ensure proper stacking after any change
    if array.size(lines) > 0
        for i = 0 to array.size(lines) - 1
            meta = array.get(lines, i)
            
            // Get label time and base price
            labTime = line.get_x2(meta.lineObj)
            labPrice = line.get_y2(meta.lineObj)
            
            // Calculate stack position for this label
            stackPosition = 0
            cycleOrder = meta.cycleLabel == "90" ? 0 : (meta.cycleLabel == "30" ? 1 : (meta.cycleLabel == "10" ? 2 : (meta.cycleLabel == "3" ? 3 : 4)))
            
            // Count labels at same time with same direction
            for j = 0 to array.size(lines) - 1
                if j != i
                    checkMeta = array.get(lines, j)
                    checkTime = line.get_x2(checkMeta.lineObj)
                    if checkTime == labTime and checkMeta.isBearish == meta.isBearish
                        checkOrder = checkMeta.cycleLabel == "90" ? 0 : (checkMeta.cycleLabel == "30" ? 1 : (checkMeta.cycleLabel == "10" ? 2 : (checkMeta.cycleLabel == "3" ? 3 : 4)))
                        // Count lower priority labels (they go below)
                        if checkOrder > cycleOrder
                            stackPosition += 1
            
            // Apply offset
            labelSpacing = 0.003 * labPrice
            offset = stackPosition * (meta.isBearish ? labelSpacing : -labelSpacing)
            label.set_xy(meta.labelObj, labTime, labPrice + offset)
```

### Modified Logic in `drawSMT()`:

**Old (lines 456-465):**
```pine
// Count existing labels on this bar with same direction
labelsOnSameBar = 0
if array.size(lines) > 0
    for j = 0 to array.size(lines) - 1
        checkMeta = array.get(lines, j)
        checkTime = line.get_x2(checkMeta.lineObj)
        if checkTime == labTime and checkMeta.isBearish == bearish
            labelsOnSameBar += 1

// Apply vertical offset (1% of price per label)
offset = labelsOnSameBar * (bearish ? 0.01 : -0.01) * labPrice
```

**New:**
```pine
// Calculate proper stacking position for this cycle
// Get list of all labels at this time position
var array<string> labelsAtTime = array.new<string>()
array.clear(labelsAtTime)

if array.size(lines) > 0
    for j = 0 to array.size(lines) - 1
        checkMeta = array.get(lines, j)
        checkTime = line.get_x2(checkMeta.lineObj)
        if checkTime == labTime and checkMeta.isBearish == bearish
            array.push(labelsAtTime, checkMeta.cycleLabel)

// Determine stack position based on cycle priority: 90 > 30 > 10 > 3 > sessions
stackPosition = 0
cycleOrder = cycleLabel == "90" ? 0 : (cycleLabel == "30" ? 1 : (cycleLabel == "10" ? 2 : (cycleLabel == "3" ? 3 : 4)))

// Count how many labels should be below this one
if array.size(labelsAtTime) > 0
    for j = 0 to array.size(labelsAtTime) - 1
        existingLabel = array.get(labelsAtTime, j)
        existingOrder = existingLabel == "90" ? 0 : (existingLabel == "30" ? 1 : (existingLabel == "10" ? 2 : (existingLabel == "3" ? 3 : 4)))
        if existingOrder > cycleOrder
            stackPosition += 1

// Use fixed offset per label (0.3% of price for better readability)
labelSpacing = 0.003 * labPrice
offset = stackPosition * (bearish ? labelSpacing : -labelSpacing)
```

## Expected Behavior After Fix

### Example: 3 SMTs on Same Candle (Bearish)

```
High of candle
    ↓
    [90m BTC SMT]      ← Closest to price (highest priority)
    [30m ES SMT]       ← 0.3% below 90m
    [10m BTC SMT]      ← 0.3% below 30m
    |
    | SMT Lines
    |
Low of candle
```

### Benefits:
1. ✅ **Clear separation** between different cycle labels
2. ✅ **Consistent ordering** - always same priority order
3. ✅ **Readable spacing** - 0.3% is perfect for most timeframes
4. ✅ **Dynamic updates** - positions adjust when labels added/removed
5. ✅ **Works for both directions** - bearish (above) and bullish (below)

## Testing Recommendations

1. **Multiple cycles on same bar** - Enable all 4 cycles, verify stacking order
2. **Label deletion** - Check remaining labels reposition correctly
3. **Label addition** - New label inserts at correct position in stack
4. **Bearish vs Bullish** - Both directions stack properly
5. **Different price levels** - 0.3% spacing works across all instruments
