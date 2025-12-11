# SMT V3 Bug Fixes Summary

## Bugs Fixed

### 1. **Label Mislabeling Bug - 90min labels on 30min/10min SMTs**

**Root Cause:**
- The `isForCurrentCycle` flag was never set to `false` when a cycle completed
- This caused old lines from completed cycles (e.g., 90min) to remain marked as "current"
- When `updateExistingLines()` ran, it would incorrectly match and update lines from different cycles

**Fix Implemented:**
- Added `markLinesAsHistorical()` function (lines 473-480) that marks all lines from a completed cycle as historical
- Called this function in `completeCycle()` BEFORE starting the new cycle (line 330)
- This ensures old cycle lines are properly flagged as historical and won't be confused with new cycles

### 2. **Incorrect historyCycleStart Tracking**

**Root Cause:**
- Line 463 (old 460) was passing `bar2` (current pivot bar) instead of cycle's `startBar`
- This caused incorrect tracking of which cycle a line belonged to

**Fix Implemented:**
- Modified `drawSMT()` function signature to accept `cycleStartBar` parameter (line 376)
- Updated line 463 to use `cycleStartBar` instead of `bar2`
- Updated all 8 calls to `drawSMT()` in `processSMT()` to pass `curr.startBar` (lines 560, 565, 570, 575, 582, 587, 592, 597)
- Updated all 14 calls to `drawSMT()` in `compareSessionSMT()` to pass `sessionStartBar` (lines 669-710)

### 3. **Session SMT Lines Missing Historical Marking**

**Root Cause:**
- Session SMT lines also use `isForCurrentCycle = true` but sessions don't call `markLinesAsHistorical()`
- Session lines would remain marked as current indefinitely

**Fix Implemented:**
- Sessions now pass `bar_index` as `sessionStartBar` parameter (line 663)
- Session lines will be marked historical when daily reset occurs (existing logic at line 756)

## Code Changes Summary

### New Function Added:
```pine
markLinesAsHistorical(cycleLabel, cycleStartBar) =>
    // Mark all lines from a completed cycle as historical
    if array.size(lines) > 0
        for i = 0 to array.size(lines) - 1
            meta = array.get(lines, i)
            // Mark lines from this specific cycle as historical
            if meta.isForCurrentCycle and meta.cycleLabel == cycleLabel and meta.historyCycleStart == cycleStartBar
                meta.isForCurrentCycle := false
```

### Modified Functions:
1. **completeCycle()** - Now calls `markLinesAsHistorical()` before starting new cycle
2. **drawSMT()** - Added `cycleStartBar` parameter, uses it for `LineMetadata` creation
3. **processSMT()** - All 8 `drawSMT()` calls now pass `curr.startBar`
4. **compareSessionSMT()** - All 14 `drawSMT()` calls now pass `sessionStartBar`

## Testing Recommendations

1. **Verify cycle separation:** Ensure 90min SMT labels only appear on 90min cycles, 30min on 30min, etc.
2. **Test cycle transitions:** Check label behavior when cycles complete and new ones start
3. **Test multiple timeframes:** Run all 4 cycles (90m, 30m, 10m, 3m) simultaneously
4. **Test session SMTs:** Verify ASIA, LONDON, NY-AM, NY-PM labels work correctly
5. **Test daily reset:** Confirm all lines clear properly at day change

## Expected Behavior After Fix

- Each cycle's SMT lines will have correct labels matching the cycle duration
- Lines from completed cycles will be marked as historical and won't update
- Only current active cycle lines will update in real-time
- No label confusion between different timeframes
