# SMT Detection System - Complete Implementation Plan

## Executive Summary

This document outlines the complete implementation plan for upgrading `SMT-dev.pine` to support **dual-layer SMT detection**:
1. **Global Time Cycles**: Continuous 90min, 30min, 10min, 3min cycles across entire trading day
2. **Session-Scoped Cycles**: Multiple 90-minute cycles within each trading session (Asia, London, NY AM, NY PM)

**Goal**: Real-time SMT detection that provides both macro (global) and micro (session-specific) divergence signals.

---

## Table of Contents
1. [Current State Analysis](#current-state-analysis)
2. [Architecture Overview](#architecture-overview)
3. [Data Structures](#data-structures)
4. [Implementation Phases](#implementation-phases)
5. [Core Algorithm Logic](#core-algorithm-logic)
6. [Label & Visualization Strategy](#label--visualization-strategy)
7. [Testing Strategy](#testing-strategy)
8. [Risk Mitigation](#risk-mitigation)

---

## Current State Analysis

### What SMT-dev.pine Currently Has

#### ✅ Global Time Cycles (Working)
- 90-minute cycles: Run continuously, 9 cycles per day
- 30-minute cycles: Run continuously, 27 cycles per day
- 10-minute cycles: Run continuously, 81 cycles per day
- 3-minute cycles: Run continuously, 243 cycles per day

**Implementation**: `CycleManager` type with rolling pivot tracking

#### ✅ Session SMT (Basic Implementation)
- Asia Session (19:00-02:30)
- London Session (02:30-07:00)
- NY AM Session (07:00-11:30)
- NY PM Session (11:30-16:00)

**Current Limitation**: Each session tracks **only ONE pivot pair** (one high, one low per symbol). No sub-cycle detection.

### What SMT_V-3.0.pine Has (Superior Features)

#### Key Improvements to Port:
1. **Better Symbol Merging**: Uses `" + "` separator (e.g., "BTC + ES")
2. **isForCurrentCycle Flag**: Better tracking of active vs historical lines
3. **Length-Based Line Updates**: Replace lines if new merge produces longer symbol string
4. **Enhanced Invalidation**: Checks both `currComp` AND `currMain` for inverse SMTs
5. **Cleaner Code Organization**: Better function naming and structure

---

## Architecture Overview

### System Design: Dual-Layer Detection

```
┌─────────────────────────────────────────────────────────────────┐
│                     SMT DETECTION SYSTEM                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  LAYER 1: GLOBAL TIME CYCLES (Continuous)                       │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ • 90-min cycles (9 per day)                            │    │
│  │ • 30-min cycles (27 per day)                           │    │
│  │ • 10-min cycles (81 per day)                           │    │
│  │ • 3-min cycles (243 per day)                           │    │
│  │                                                         │    │
│  │ Purpose: Detect SMTs across entire trading day         │    │
│  │ Label Format: "SMT-BTC 90", "SMT-ES 30"               │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  LAYER 2: SESSION-SCOPED CYCLES (New)                           │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ London (02:30-07:00): 3x 90-min cycles                 │    │
│  │ NY AM (07:00-11:30): 3x 90-min cycles                  │    │
│  │ NY PM (11:30-16:00): 3x 90-min cycles                  │    │
│  │ Asia (19:00-02:30): 5x 90-min cycles                   │    │
│  │                                                         │    │
│  │ Purpose: Detect SMTs within session context            │    │
│  │ Label Format: "SMT-BTC London-C2", "SMT-ES NY AM-C1"  │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Key Architectural Principles

1. **Independence**: Both layers run simultaneously without interference
2. **Code Reuse**: Session cycles use same `processSMT()` and `drawSMT()` functions
3. **Real-Time Updates**: Dynamic lines update as cycles progress
4. **Smart Invalidation**: Each layer tracks invalidation independently
5. **Scalability**: Easy to add more session types or cycle durations


---

## Data Structures

### New Type: SessionCycleData

```pine
type SessionCycleData
    int cycleNumber          // Which cycle within session (1, 2, 3, etc.)
    int startTime            // Timestamp when cycle started
    int startBar             // Bar index when cycle started
    bool active              // Is this cycle currently running?
    
    // Main symbol pivots
    PivotPoint mainPivotHigh
    PivotPoint mainPivotLow
    
    // Comparison symbol 1 pivots
    PivotPoint s1PivotHigh
    PivotPoint s1PivotLow
    
    // Comparison symbol 2 pivots
    PivotPoint s2PivotHigh
    PivotPoint s2PivotLow
```

**Purpose**: Tracks pivot data for ONE 90-minute cycle within a session.

### New Type: SessionCycleManager

```pine
type SessionCycleManager
    string sessionName           // "London", "NY AM", "NY PM", "Asia"
    string sessionTime           // "0230-0700", "0700-1130", etc.
    int cycleDurationMin         // 90 minutes
    int cyclesPerSession         // 3 for London/NY, 5 for Asia
    bool sessionActive           // Is session currently running?
    int currentCycleIndex        // Which cycle we're on (0-based: 0, 1, 2)
    
    // Current cycle being tracked in real-time
    SessionCycleData current
    
    // Completed cycles (for SMT comparison)
    array<SessionCycleData> history
    
    // Line management (same as CycleManager)
    array<LineMetadata> lines
```

**Purpose**: Manages all cycles within one trading session.

### Modified Type: LineMetadata

```pine
type LineMetadata
    line smtLine
    label smtLabel
    string cycleLabel        // Renamed from smtLabel: "90", "30", "London-C2", etc.
    int smtTypeId           // 0-7 (bearish/bullish, normal/inverse, high/low)
    int mainBar
    float mainPrice
    int compBar
    float compPrice
    int length              // For symbol merging priority
    bool isForCurrentCycle  // Renamed from isDynamic: true if line is for active cycle
```

**Changes from current SMT-dev.pine**:
- `smtLabel` → `cycleLabel` (clearer naming)
- `isDynamic` → `isForCurrentCycle` (matches V-3.0 naming)


---

## Implementation Phases

### Phase 1: Port V-3.0 Improvements (Foundation)

**Goal**: Upgrade existing code quality before adding session cycles.

#### Step 1.1: Rename Fields for Consistency
- **File**: SMT-dev.pine
- **Lines**: 117-128 (LineMetadata type)

**Changes**:
```pine
// OLD
type LineMetadata
    ...
    string smtLabel
    bool isDynamic

// NEW
type LineMetadata
    ...
    string cycleLabel           // More descriptive name
    bool isForCurrentCycle      // Clearer intent
```

**Impact**: All references must be updated throughout file.

#### Step 1.2: Upgrade drawSMT() Function
- **File**: SMT-dev.pine
- **Lines**: 301-348

**Key Improvements from V-3.0**:

1. **Better Symbol Merging Logic**
```pine
// OLD: Uses "+" separator
newSymbol = existing + "+" + newSymbol

// NEW: Uses " + " separator (more readable)
newSymbol = existing + " + " + newSymbol
```

2. **Merge Only for Current Cycle**
```pine
// OLD: Merges based on isDynamic
if meta.isDynamic and sameBearish and sameLabel

// NEW: More explicit naming
if meta.isForCurrentCycle and sameBearish and sameLabel
```

3. **Length-Based Line Updating**
```pine
// NEW: Always update if new line has more symbols
if mergeIdx >= 0
    existingMeta = array.get(manager.lines, mergeIdx)
    newLen = calcLen(newSymbol)
    
    if newLen > existingMeta.length
        // Update existing line with better data
        line.set_xy1(existingMeta.smtLine, bar2, price2)
        line.set_xy2(existingMeta.smtLine, bar1, price1)
        // ... update all metadata
```

4. **Add Symbol Only If Not Present**
```pine
// NEW: Check if symbol already in merged string
if str.contains(existingMeta.cycleLabel, symbolStr)
    // Don't add duplicate
else
    // Add to merge
    newSymbol = existingMeta.cycleLabel + " + " + symbolStr
```

**Function Signature Change**:
```pine
// OLD
drawSMT(..., labelStr, isBearish, typeId, compPrice, isDynamicLine)

// NEW
drawSMT(..., cycleLabel, bearish, typeId, compPrice, isForCurrentCycle)
```

#### Step 1.3: Enhance Invalidation Logic
- **File**: SMT-dev.pine
- **Lines**: 292-300

**Port V-3.0 Logic**:
```pine
// NEW: Better inverse SMT invalidation
isInvalidated(meta, isS1, isHigh, bearish, inverse, currMain, currComp) =>
    if na(currComp) or na(currMain)
        false
    else
        if inverse
            // V-3.0: Check BOTH main and comp prices
            if bearish
                currComp < meta.compPrice or currMain > meta.mainPrice
            else
                currComp > meta.compPrice or currMain < meta.mainPrice
        else
            // Normal SMT: Only check comp price
            if bearish
                currComp > meta.compPrice
            else
                currComp < meta.compPrice
```

**Why This Matters**: Inverse SMTs (when main makes new high but comp doesn't) need to check if EITHER condition breaks, not just comp.

#### Step 1.4: Update All Function Calls
- Update `processSMT()` parameter: `isDynamic` → `isForCurrentCycle`
- Update `updateDynamicLines()` to use new field names
- Update all references to `meta.smtLabel` → `meta.cycleLabel`
- Update all references to `meta.isDynamic` → `meta.isForCurrentCycle`

**Files to Update**:
- Line 376-393: `updateDynamicLines()`
- Line 395-403: `processCycleSMT()`
- Line 405-407: `processSessionSMT()`
- Line 409-428: `processSMT()`

#### Step 1.5: Testing Phase 1
- Verify global time cycles still work (90, 30, 10, 3 min)
- Verify symbol merging with " + " separator
- Verify invalidation works correctly
- Verify existing session SMT still works
- Verify line updates happen correctly

**Expected Result**: All existing functionality works with improved code quality.


---

### Phase 2: Add Session Cycle Infrastructure

**Goal**: Build the data structures and initialization for session-scoped cycles.

#### Step 2.1: Add SessionCycleData Type
- **Location**: After `CycleManager` type definition (around line 115)

```pine
type SessionCycleData
    int cycleNumber
    int startTime
    int startBar
    bool active
    PivotPoint mainPivotHigh
    PivotPoint mainPivotLow
    PivotPoint s1PivotHigh
    PivotPoint s1PivotLow
    PivotPoint s2PivotHigh
    PivotPoint s2PivotLow
```

#### Step 2.2: Add SessionCycleManager Type
- **Location**: After `SessionCycleData` type

```pine
type SessionCycleManager
    string sessionName
    string sessionTime
    int cycleDurationMin
    int cyclesPerSession
    bool sessionActive
    int currentCycleIndex
    SessionCycleData current
    array<SessionCycleData> history
    array<LineMetadata> lines
```

#### Step 2.3: Initialize Session Managers Array
- **Location**: In variable declarations (around line 160)

```pine
// Global array to hold all session cycle managers
var array<SessionCycleManager> sessionCycleManagers = array.new<SessionCycleManager>()
```

#### Step 2.4: Create Session Manager Initialization Function
- **Location**: After helper functions (around line 225)

```pine
initSessionCycleData() =>
    SessionCycleData.new(
        0,              // cycleNumber
        na,             // startTime
        na,             // startBar
        false,          // active
        na,             // mainPivotHigh
        na,             // mainPivotLow
        na,             // s1PivotHigh
        na,             // s1PivotLow
        na,             // s2PivotHigh
        na              // s2PivotLow
    )

createSessionManager(name, timeStr, duration, numCycles) =>
    SessionCycleManager.new(
        name,                           // sessionName
        timeStr,                        // sessionTime
        duration,                       // cycleDurationMin
        numCycles,                      // cyclesPerSession
        false,                          // sessionActive
        0,                              // currentCycleIndex
        initSessionCycleData(),         // current
        array.new<SessionCycleData>(),  // history
        array.new<LineMetadata>()       // lines
    )
```

#### Step 2.5: Initialize All Session Managers
- **Location**: In `if barstate.isfirst` block (around line 178)

```pine
if barstate.isfirst
    // ... existing code ...
    
    // Initialize session cycle managers
    if enableSessionSmt
        // London: 02:30-07:00 (270 min = 3 cycles)
        array.push(sessionCycleManagers, 
            createSessionManager("London", "0230-0700", 90, 3))
        
        // NY AM: 07:00-11:30 (270 min = 3 cycles)
        array.push(sessionCycleManagers, 
            createSessionManager("NY AM", "0700-1130", 90, 3))
        
        // NY PM: 11:30-16:00 (270 min = 3 cycles)
        array.push(sessionCycleManagers, 
            createSessionManager("NY PM", "1130-1600", 90, 3))
        
        // Asia: 19:00-02:30 (450 min = 5 cycles)
        array.push(sessionCycleManagers, 
            createSessionManager("Asia", "1900-0230", 90, 5))
```

**Note**: Times are in Exchange Timezone (ET for US markets).

#### Step 2.6: Testing Phase 2
- Verify managers initialize correctly
- Verify no errors on script load
- No visual changes yet (infrastructure only)

**Expected Result**: Script loads without errors, no functional changes yet.

---

### Phase 3: Implement Session Cycle Detection Logic

**Goal**: Make session managers track 90-minute cycles within their sessions.

#### Step 3.1: Create Session Detection Function
- **Location**: After time helper functions (around line 245)

```pine
detectActiveSession() =>
    // Returns index of active session manager, or -1 if none active
    var int activeIdx = -1
    
    if enableSessionSmt and array.size(sessionCycleManagers) > 0
        for i = 0 to array.size(sessionCycleManagers) - 1
            manager = array.get(sessionCycleManagers, i)
            inSession = time(timeframe.period, manager.sessionTime, "America/New_York") != 0
            
            if inSession
                activeIdx := i
                break
    
    activeIdx
```

#### Step 3.2: Create Session Cycle Start Function
- **Location**: After `detectActiveSession()`

```pine
startSessionCycle(cycleData, cycleNum) =>
    cycleData.cycleNumber := cycleNum
    cycleData.startTime := time
    cycleData.startBar := bar_index
    cycleData.active := true
    
    // Reset pivots
    cycleData.mainPivotHigh := na
    cycleData.mainPivotLow := na
    cycleData.s1PivotHigh := na
    cycleData.s1PivotLow := na
    cycleData.s2PivotHigh := na
    cycleData.s2PivotLow := na
```

#### Step 3.3: Create Session Cycle Completion Check
- **Location**: After `startSessionCycle()`

```pine
shouldCompleteSessionCycle(manager) =>
    // Check if 90 minutes have elapsed since cycle start
    bool shouldComplete = false
    
    if not na(manager.current.startTime) and manager.current.active
        elapsedMin = (time - manager.current.startTime) / 60000
        shouldComplete := elapsedMin >= manager.cycleDurationMin
    
    shouldComplete
```

#### Step 3.4: Create Session Cycle Completion Function
- **Location**: After `shouldCompleteSessionCycle()`

```pine
completeSessionCycle(manager) =>
    if manager.current.active
        // Save current cycle to history
        savedCycle = SessionCycleData.new(
            manager.current.cycleNumber,
            manager.current.startTime,
            manager.current.startBar,
            false,  // no longer active
            manager.current.mainPivotHigh,
            manager.current.mainPivotLow,
            manager.current.s1PivotHigh,
            manager.current.s1PivotLow,
            manager.current.s2PivotHigh,
            manager.current.s2PivotLow
        )
        
        array.push(manager.history, savedCycle)
        
        // Start next cycle if session continues
        manager.currentCycleIndex += 1
        
        if manager.currentCycleIndex < manager.cyclesPerSession
            startSessionCycle(manager.current, manager.currentCycleIndex + 1)
        else
            // Session ending, mark current as inactive
            manager.current.active := false
```

#### Step 3.5: Create Pivot Update Function for Session Cycles
- **Location**: After `completeSessionCycle()`

```pine
updateSessionCyclePivots(cycleData, mainHigh, mainLow, s1High, s1Low, s2High, s2Low) =>
    // Update main symbol pivots
    if not na(mainHigh)
        if na(cycleData.mainPivotHigh) or mainHigh.price > cycleData.mainPivotHigh.price
            cycleData.mainPivotHigh := mainHigh
    
    if not na(mainLow)
        if na(cycleData.mainPivotLow) or mainLow.price < cycleData.mainPivotLow.price
            cycleData.mainPivotLow := mainLow
    
    // Update S1 pivots
    if not na(s1High)
        if na(cycleData.s1PivotHigh) or s1High.price > cycleData.s1PivotHigh.price
            cycleData.s1PivotHigh := s1High
    
    if not na(s1Low)
        if na(cycleData.s1PivotLow) or s1Low.price < cycleData.s1PivotLow.price
            cycleData.s1PivotLow := s1Low
    
    // Update S2 pivots
    if not na(s2High)
        if na(cycleData.s2PivotHigh) or s2High.price > cycleData.s2PivotHigh.price
            cycleData.s2PivotHigh := s2High
    
    if not na(s2Low)
        if na(cycleData.s2PivotLow) or s2Low.price < cycleData.s2PivotLow.price
            cycleData.s2PivotLow := s2Low
```

#### Step 3.6: Testing Phase 3
- Verify session detection works correctly
- Verify cycles start/stop at correct times
- Add debug labels to show cycle numbers
- Verify pivot updates happen correctly

**Expected Result**: Session cycles track correctly, but no SMT lines drawn yet.


---

### Phase 4: Integrate Session Cycle SMT Processing

**Goal**: Connect session cycles to existing SMT detection and drawing logic.

#### Step 4.1: Create Session Cycle SMT Processor
- **Location**: After `processSMT()` function (around line 430)

```pine
processSessionCycleSMT(manager, s1Abbr, s2Abbr) =>
    // Only process if we have at least one completed cycle
    if array.size(manager.history) > 0 and manager.current.active
        curr = manager.current
        prev = array.get(manager.history, array.size(manager.history) - 1)
        
        // Create cycle label: "London-C2", "NY AM-C3", etc.
        cycleLabel = manager.sessionName + "-C" + str.tostring(curr.cycleNumber)
        
        // Reuse existing processSMT with session-specific label
        processSMT(
            curr,                  // current cycle data
            prev,                  // previous cycle data
            cycleLabel,            // "London-C2", etc.
            s1Abbr,               // "BTC", "ES", etc.
            s2Abbr,               // comparison symbol
            true,                  // isForCurrentCycle = true
            prev.startBar,         // histBarStart for comparison
            manager.lines          // session-specific line array
        )
```

**Key Points**:
- Compares current cycle vs previous cycle within SAME session
- Uses session-specific label format
- Passes `manager.lines` so session SMTs tracked separately

#### Step 4.2: Modify processSMT() to Accept Line Array
- **Location**: Line 409 (processSMT function signature)

**Current**:
```pine
processSMT(curr, prev, label, s1, s2, isDynamic, histBarStart) =>
    // Uses implicit global line manager
```

**New**:
```pine
processSMT(curr, prev, label, s1, s2, isForCurrentCycle, histBarStart, lineManager=na) =>
    // Accept optional line manager parameter
    // If na, use default cycle manager lines
    // If provided, use session manager lines
    
    manager = na(lineManager) ? cycleManager : lineManager
```

**Impact**: All existing calls still work (default behavior), but session cycles can pass their own line manager.

#### Step 4.3: Add Session Cycle Main Loop
- **Location**: In main execution block (around line 475-510)

```pine
// After time cycle SMT logic
// Add session cycle logic

if enableSessionSmt and array.size(sessionCycleManagers) > 0
    // Detect which session is active
    activeSessionIdx = detectActiveSession()
    
    // Check for session transitions
    for i = 0 to array.size(sessionCycleManagers) - 1
        manager = array.get(sessionCycleManagers, i)
        wasActive = manager.sessionActive
        isActive = (i == activeSessionIdx)
        
        if isActive and not wasActive
            // Session just started
            manager.sessionActive := true
            manager.currentCycleIndex := 0
            array.clear(manager.history)  // Clear previous session history
            startSessionCycle(manager.current, 1)  // Start Cycle 1
        
        else if not isActive and wasActive
            // Session just ended
            manager.sessionActive := false
            manager.current.active := false
        
        // Update manager in array
        array.set(sessionCycleManagers, i, manager)
    
    // Process active session
    if activeSessionIdx >= 0
        activeManager = array.get(sessionCycleManagers, activeSessionIdx)
        
        if activeManager.current.active
            // Update pivots for current cycle
            updateSessionCyclePivots(
                activeManager.current,
                mainPivotHigh, mainPivotLow,
                s1PivotHigh, s1PivotLow,
                s2PivotHigh, s2PivotLow
            )
            
            // Check if cycle should complete
            if shouldCompleteSessionCycle(activeManager)
                completeSessionCycle(activeManager)
            
            // Process SMT (compare current vs previous cycle)
            if activeManager.current.active
                processSessionCycleSMT(activeManager, s1Abbr, s2Abbr)
                
                // Update dynamic lines for session SMTs
                updateDynamicLines(activeManager, true)
        
        // Update manager in array
        array.set(sessionCycleManagers, activeSessionIdx, activeManager)
```

#### Step 4.4: Extend Invalidation Check to Session Lines
- **Location**: At start of main execution (around line 476)

```pine
// Existing: Check global cycle line invalidation
checkInvalidation()

// NEW: Check session cycle line invalidation
if enableSessionSmt
    for i = 0 to array.size(sessionCycleManagers) - 1
        manager = array.get(sessionCycleManagers, i)
        checkInvalidation(manager.lines)  // Pass session line array
        array.set(sessionCycleManagers, i, manager)
```

**Note**: Need to modify `checkInvalidation()` to accept optional line array parameter.

#### Step 4.5: Modify checkInvalidation() Function
- **Location**: Line 355

**Current**:
```pine
checkInvalidation() =>
    // Uses implicit global manager
```

**New**:
```pine
checkInvalidation(lineArray=na) =>
    // Accept optional line array
    // If na, use default cycle manager lines
    linesToCheck = na(lineArray) ? cycleManagers : lineArray
    
    // Rest of logic stays the same, but operate on linesToCheck
```

#### Step 4.6: Testing Phase 4
- Verify session SMTs draw correctly
- Verify labels show "London-C2", "NY AM-C3", etc.
- Verify lines update in real-time during active cycle
- Verify invalidation works for session SMTs
- Verify global cycle SMTs still work independently

**Expected Result**: Both global and session SMTs visible on chart, working independently.


---

### Phase 5: Optimization & Fine-Tuning

**Goal**: Polish the implementation for performance and usability.

#### Step 5.1: Add Input Controls
- **Location**: Input section (around line 80)

```pine
// Existing
enableSessionSmt = input.bool(false, "Enable Session SMT", group="Session Filter")

// NEW: Add granular controls
enableSessionCycles = input.bool(true, "Enable Session Cycle Detection", 
    tooltip="Detects multiple 90-min cycles within each session", 
    group="Session Filter")

showLondonCycles = input.bool(true, "Show London Cycles", group="Session Filter")
showNyAmCycles = input.bool(true, "Show NY AM Cycles", group="Session Filter")
showNyPmCycles = input.bool(true, "Show NY PM Cycles", group="Session Filter")
showAsiaCycles = input.bool(true, "Show Asia Cycles", group="Session Filter")

maxSessionLines = input.int(20, "Max Session SMT Lines", minval=5, maxval=50,
    tooltip="Maximum number of session cycle SMT lines to keep", 
    group="Session Filter")
```

#### Step 5.2: Implement Session Filtering
- **Location**: In `processSessionCycleSMT()` function

```pine
processSessionCycleSMT(manager, s1Abbr, s2Abbr) =>
    // Check if this session is enabled
    bool enabled = switch manager.sessionName
        "London" => showLondonCycles
        "NY AM" => showNyAmCycles
        "NY PM" => showNyPmCycles
        "Asia" => showAsiaCycles
        => false
    
    if not enabled
        return  // Skip processing
    
    // Rest of existing logic...
```

#### Step 5.3: Add Line Limit Management
- **Location**: In `drawSMT()` function

```pine
drawSMT(..., lineManager) =>
    // ... existing merge logic ...
    
    // Check line limit (different for session vs cycle managers)
    maxLines = na(lineManager) ? maxLines : maxSessionLines
    
    if array.size(lineManager.lines) >= maxLines
        // Remove oldest non-current line
        // ... existing FIFO logic ...
```

#### Step 5.4: Add Debug Mode
- **Location**: Input section

```pine
debugMode = input.bool(false, "Debug Mode", 
    tooltip="Shows cycle start/end markers and cycle numbers",
    group="Developer")

showCycleMarkers = input.bool(false, "Show Cycle Start Markers",
    group="Developer")
```

- **Location**: In session cycle logic

```pine
if debugMode and manager.current.active
    // Draw debug label showing current cycle info
    label.new(
        bar_index, high,
        manager.sessionName + " C" + str.tostring(manager.current.cycleNumber),
        style=label.style_label_down,
        color=color.yellow,
        textcolor=color.black,
        size=size.tiny
    )

if showCycleMarkers and manager.current.startBar == bar_index
    // Mark cycle start
    label.new(
        bar_index, low,
        "▲ Start C" + str.tostring(manager.current.cycleNumber),
        style=label.style_label_up,
        color=color.green,
        textcolor=color.white,
        size=size.tiny
    )
```

#### Step 5.5: Optimize Array Operations
- **Location**: Throughout session cycle code

**Optimization Tips**:
1. Cache array size checks instead of repeated calls
2. Use `array.get()` once and store reference
3. Avoid unnecessary array.set() calls
4. Clear history arrays when sessions reset

```pine
// BEFORE (inefficient)
for i = 0 to array.size(sessionCycleManagers) - 1
    m = array.get(sessionCycleManagers, i)
    // ... logic ...
    array.set(sessionCycleManagers, i, m)

// AFTER (optimized)
size = array.size(sessionCycleManagers)
for i = 0 to size - 1
    m = array.get(sessionCycleManagers, i)
    // ... logic ...
    // Only set if modified
    if wasModified
        array.set(sessionCycleManagers, i, m)
```

#### Step 5.6: Add Session Transition Logging
- **Location**: In session transition detection

```pine
if debugMode
    if isActive and not wasActive
        log.info("Session started: " + manager.sessionName)
    else if not isActive and wasActive
        log.info("Session ended: " + manager.sessionName + 
            " | Cycles: " + str.tostring(array.size(manager.history)))
```

#### Step 5.7: Testing Phase 5
- Test with different input configurations
- Verify performance with all sessions enabled
- Test debug mode features
- Verify line limits work correctly
- Test across different timeframes

**Expected Result**: Polished, configurable system with good performance.


---

## Core Algorithm Logic

### Real-Time Execution Flow (Every Bar)

```
START OF BAR
│
├─► 1. INVALIDATION CHECK
│   ├─ Check global cycle SMT lines
│   │  └─ Remove lines where price invalidates divergence
│   └─ Check session cycle SMT lines
│      └─ Remove lines where price invalidates divergence
│
├─► 2. GLOBAL TIME CYCLE PROCESSING
│   │
│   └─ FOR EACH time cycle (90min, 30min, 10min, 3min)
│       │
│       ├─ IF cycle not active
│       │  └─ startCycle() → Initialize pivots
│       │
│       ├─ Update pivots (highs/lows) for current cycle
│       │
│       ├─ IF shouldComplete() → Cycle duration reached
│       │  ├─ Save current cycle to history
│       │  ├─ processSMT() → Compare with previous cycle
│       │  │  ├─ Check bearish SMT (main higher high, comp lower high)
│       │  │  ├─ Check bullish SMT (main lower low, comp higher low)
│       │  │  ├─ Check inverse SMTs
│       │  │  └─ drawSMT() → Create/update lines and labels
│       │  └─ Start new cycle
│       │
│       └─ updateDynamicLines() → Update lines for active cycle
│
├─► 3. SESSION DETECTION
│   │
│   ├─ detectActiveSession() → Which session is running?
│   │  ├─ Check London (02:30-07:00)
│   │  ├─ Check NY AM (07:00-11:30)
│   │  ├─ Check NY PM (11:30-16:00)
│   │  └─ Check Asia (19:00-02:30)
│   │
│   └─ FOR EACH session manager
│       │
│       ├─ IF session just started (transition)
│       │  ├─ sessionActive = true
│       │  ├─ Clear history from previous day
│       │  ├─ currentCycleIndex = 0
│       │  └─ startSessionCycle(Cycle 1)
│       │
│       ├─ IF session just ended (transition)
│       │  └─ sessionActive = false, current.active = false
│       │
│       └─ IF session is active
│           │
│           ├─ Update pivots for current cycle
│           │
│           ├─ IF shouldCompleteSessionCycle()
│           │  ├─ Save current cycle to history
│           │  ├─ currentCycleIndex++
│           │  └─ IF more cycles remaining
│           │     └─ startSessionCycle(next cycle)
│           │
│           ├─ IF array.size(history) > 0
│           │  └─ processSessionCycleSMT()
│           │     ├─ Compare current cycle vs previous cycle
│           │     ├─ Label: "London-C2", "NY AM-C3", etc.
│           │     └─ drawSMT() with session line manager
│           │
│           └─ updateDynamicLines() → Update session SMT lines
│
END OF BAR
```

### Cycle Completion Logic

#### Global Time Cycles
```
shouldComplete(cycleManager, durationMin):
    IF cycleManager.active:
        elapsedMin = (time - cycleManager.startTime) / 60000
        RETURN elapsedMin >= durationMin
    RETURN false

Example: 90-min cycle
    Start: 09:00
    Check: 10:30
    Elapsed: 90 minutes
    Result: Complete → Process SMT → Start new cycle
```

#### Session Cycles
```
shouldCompleteSessionCycle(sessionManager):
    IF sessionManager.current.active:
        elapsedMin = (time - sessionManager.current.startTime) / 60000
        RETURN elapsedMin >= sessionManager.cycleDurationMin
    RETURN false

Example: London Cycle 2
    Session: London (02:30-07:00)
    Cycle 2: 04:00-05:30
    Check: 05:30
    Elapsed: 90 minutes
    Result: Complete → Process SMT → Start Cycle 3
```

### SMT Detection Algorithm

```
processSMT(current, previous, label, s1Abbr, s2Abbr, isForCurrentCycle, histBarStart, lineManager):
    
    // Extract pivot data
    currMainHigh = current.mainPivotHigh
    currMainLow = current.mainPivotLow
    prevMainHigh = previous.mainPivotHigh
    prevMainLow = previous.mainPivotLow
    
    currS1High = current.s1PivotHigh
    currS1Low = current.s1PivotLow
    prevS1High = previous.s1PivotHigh
    prevS1Low = previous.s1PivotLow
    
    // Check all 8 SMT types:
    
    // 1. BEARISH NORMAL HIGH (Main higher high, S1 lower high)
    IF currMainHigh > prevMainHigh AND currS1High < prevS1High:
        drawSMT(prevMainHigh, currMainHigh, prevS1High, currS1High, 
                s1Abbr, label, true, 0, currS1High.price, isForCurrentCycle, lineManager)
    
    // 2. BULLISH NORMAL LOW (Main lower low, S1 higher low)
    IF currMainLow < prevMainLow AND currS1Low > prevS1Low:
        drawSMT(prevMainLow, currMainLow, prevS1Low, currS1Low,
                s1Abbr, label, false, 1, currS1Low.price, isForCurrentCycle, lineManager)
    
    // 3. BEARISH INVERSE HIGH (Main lower high, S1 higher high)
    IF currMainHigh < prevMainHigh AND currS1High > prevS1High:
        drawSMT(prevMainHigh, currMainHigh, prevS1High, currS1High,
                s1Abbr, label, true, 4, currS1High.price, isForCurrentCycle, lineManager)
    
    // 4. BULLISH INVERSE LOW (Main higher low, S1 lower low)
    IF currMainLow > prevMainLow AND currS1Low < prevS1Low:
        drawSMT(prevMainLow, currMainLow, prevS1Low, currS1Low,
                s1Abbr, label, false, 5, currS1Low.price, isForCurrentCycle, lineManager)
    
    // ... repeat for S2 (types 2, 3, 6, 7)
```

### Symbol Merging Logic

```
drawSMT(..., symbolStr, cycleLabel, bearish, typeId, compPrice, isForCurrentCycle, lineManager):
    
    // 1. Check for existing line to merge
    mergeIdx = -1
    
    IF isForCurrentCycle:  // Only merge lines for active cycles
        FOR i = 0 to array.size(lineManager.lines) - 1:
            meta = array.get(lineManager.lines, i)
            
            sameBearish = (meta.smtTypeId in {0,2,4,6}) == bearish
            sameCycle = meta.cycleLabel == cycleLabel
            isForCurrent = meta.isForCurrentCycle
            
            IF sameBearish AND sameCycle AND isForCurrent:
                mergeIdx = i
                BREAK
    
    // 2. If merge found, update existing line
    IF mergeIdx >= 0:
        existingMeta = array.get(lineManager.lines, mergeIdx)
        
        // Check if symbol already in label
        IF NOT str.contains(existingMeta.cycleLabel, symbolStr):
            newSymbol = existingMeta.cycleLabel + " + " + symbolStr
            newLen = calcLen(newSymbol)
            
            // Update if new line is longer (more symbols)
            IF newLen > existingMeta.length:
                line.set_xy1(existingMeta.smtLine, bar2, price2)
                line.set_xy2(existingMeta.smtLine, bar1, price1)
                label.set_xy(existingMeta.smtLabel, bar1, price1)
                label.set_text(existingMeta.smtLabel, newSymbol + " " + cycleLabel)
                existingMeta.length := newLen
                // ... update all metadata
        
        RETURN true  // Merged successfully
    
    // 3. No merge, add new line
    ELSE:
        // Check line limit
        IF array.size(lineManager.lines) >= maxLines:
            // Remove oldest non-current line (FIFO)
            removeOldestLine(lineManager)
        
        // Create new line and label
        newLine = line.new(...)
        newLabel = label.new(...)
        newMeta = LineMetadata.new(...)
        
        array.push(lineManager.lines, newMeta)
        
        RETURN true
```

### Invalidation Logic

```
isInvalidated(meta, isS1, isHigh, bearish, inverse, currMain, currComp):
    
    IF na(currComp) OR na(currMain):
        RETURN false
    
    IF inverse:
        // Inverse SMTs check BOTH main and comp
        IF bearish:
            // Bearish inverse: invalidated if comp drops below OR main rises above
            RETURN currComp < meta.compPrice OR currMain > meta.mainPrice
        ELSE:
            // Bullish inverse: invalidated if comp rises above OR main drops below
            RETURN currComp > meta.compPrice OR currMain < meta.mainPrice
    
    ELSE:
        // Normal SMTs check only comp
        IF bearish:
            // Bearish normal: invalidated if comp rises above
            RETURN currComp > meta.compPrice
        ELSE:
            // Bullish normal: invalidated if comp drops below
            RETURN currComp < meta.compPrice

checkInvalidation(lineArray):
    toRemove = array.new<int>()
    
    FOR i = 0 to array.size(lineArray) - 1:
        meta = array.get(lineArray, i)
        
        // Get current prices
        currMain = getCurrentPrice(isHigh, true)
        currS1 = getCurrentPrice(isHigh, false, 1)
        currS2 = getCurrentPrice(isHigh, false, 2)
        
        // Check if invalidated
        IF isInvalidated(meta, ...):
            array.push(toRemove, i)
            line.delete(meta.smtLine)
            label.delete(meta.smtLabel)
    
    // Remove from array (reverse order to preserve indices)
    FOR i = array.size(toRemove) - 1 to 0:
        idx = array.get(toRemove, i)
        array.remove(lineArray, idx)
```


---

## Label & Visualization Strategy

### Label Naming Convention

#### Global Time Cycles
```
Format: "SMT-{SYMBOL} {DURATION}"

Examples:
- "SMT-BTC 90"       → Bitcoin 90-minute cycle SMT
- "SMT-ES 30"        → S&P 500 30-minute cycle SMT
- "SMT-BTC+ES 10"    → Merged Bitcoin & S&P 10-minute cycle SMT
- "SMT-BTC+ES+NQ 3"  → Three-symbol merged 3-minute cycle SMT
```

#### Session Cycle SMTs
```
Format: "SMT-{SYMBOL} {SESSION}-C{NUMBER}"

Examples:
- "SMT-BTC London-C1"      → Bitcoin, London session, Cycle 1 (02:30-04:00)
- "SMT-BTC London-C2"      → Bitcoin, London session, Cycle 2 (04:00-05:30)
- "SMT-ES NY AM-C1"        → S&P 500, NY AM session, Cycle 1 (07:00-08:30)
- "SMT-BTC+ES NY PM-C3"    → Merged, NY PM session, Cycle 3 (14:30-16:00)
- "SMT-BTC Asia-C5"        → Bitcoin, Asia session, Cycle 5 (01:00-02:30)
```

### Visual Differentiation

#### Color Schemes

**Global Time Cycles:**
```pine
// Bearish SMT (main higher high, comp lower high)
Bearish Normal: color.red (lines) + color.red (labels)
Bearish Inverse: color.orange (lines) + color.orange (labels)

// Bullish SMT (main lower low, comp higher low)
Bullish Normal: color.lime (lines) + color.lime (labels)
Bullish Inverse: color.aqua (lines) + color.aqua (labels)
```

**Session Cycle SMTs:**
```pine
// Use slightly different shades for distinction

// Bearish SMTs
Bearish Normal: color.new(color.red, 20) (semi-transparent red)
Bearish Inverse: color.new(color.orange, 20)

// Bullish SMTs
Bullish Normal: color.new(color.lime, 20)
Bullish Inverse: color.new(color.aqua, 20)
```

#### Line Styles

**Global Time Cycles:**
```pine
line.style_solid → Solid lines for global cycles
```

**Session Cycle SMTs:**
```pine
line.style_dashed → Dashed lines for session cycles
```

This visual distinction makes it easy to identify at a glance whether an SMT is from a global cycle or session cycle.

#### Label Positions

```pine
// Bearish SMTs (high pivots)
Label position: label.style_label_down
Placement: Above the main symbol high

// Bullish SMTs (low pivots)
Label position: label.style_label_up
Placement: Below the main symbol low
```

### Chart Display Example

```
Time: 02:30 ───────────────────────────────────────────────────────► 16:00
      │                                                               │
      London          │        NY AM         │       NY PM           │
      02:30-07:00     │     07:00-11:30      │    11:30-16:00        │
      │               │                      │                       │
      ├─ C1 (02:30-04:00)                                            │
      ├─ C2 (04:00-05:30) ← SMT-BTC London-C2 [DASHED RED LINE]     │
      └─ C3 (05:30-07:00)                                            │
                      │                                               │
                      ├─ C1 (07:00-08:30)                            │
                      ├─ C2 (08:30-10:00) ← SMT-ES NY AM-C2 [DASHED LIME LINE]
                      └─ C3 (10:00-11:30)                            │
                                           │                          │
                                           ├─ C1 (11:30-13:00)       │
                                           ├─ C2 (13:00-14:30)       │
                                           └─ C3 (14:30-16:00)       │

Plus overlaid:
- Global 90-min cycle SMTs [SOLID LINES]
- Global 30-min cycle SMTs [SOLID LINES]
- Global 10-min cycle SMTs [SOLID LINES]
```

### Label Size & Tooltip Strategy

```pine
// Global cycle labels (more prominent)
label.new(
    ...,
    size = size.normal,
    tooltip = "Bearish SMT: ES higher high, BTC lower high\n" +
              "Main: " + str.tostring(mainPrice) + "\n" +
              "Comp: " + str.tostring(compPrice) + "\n" +
              "Cycle: 90-minute"
)

// Session cycle labels (slightly smaller)
label.new(
    ...,
    size = size.small,
    tooltip = "Bearish SMT: ES higher high, BTC lower high\n" +
              "Main: " + str.tostring(mainPrice) + "\n" +
              "Comp: " + str.tostring(compPrice) + "\n" +
              "Session: London, Cycle 2\n" +
              "Time: 04:00-05:30"
)
```

### Dynamic Line Updates (Real-Time)

```
During Active Cycle:
─────────────────────────────────────────────
Current Cycle: London-C3 (05:30-07:00)
Previous Cycle: London-C2 (04:00-05:30) - completed

05:35 → New high on Main, draw SMT line (dashed)
05:42 → Even higher high on Main, extend line endpoint
05:50 → BTC makes new high → INVALIDATE, delete line
05:58 → Another divergence forms → Draw new line
07:00 → Cycle completes → Line becomes historical (solid)
```

**Implementation:**
- Active cycle lines: `isForCurrentCycle = true` → Can be updated/merged
- Historical cycle lines: `isForCurrentCycle = false` → Fixed, no updates

---

## Testing Strategy

### Phase 1: Unit Testing (Per Phase)

#### Phase 1 Tests: V-3.0 Improvements
```
✓ Verify LineMetadata field renames don't break existing code
✓ Test symbol merging with " + " separator
✓ Test length-based line updates
✓ Test enhanced invalidation for inverse SMTs
✓ Verify global time cycles still work correctly
```

#### Phase 2 Tests: Infrastructure
```
✓ Verify SessionCycleData type initializes correctly
✓ Verify SessionCycleManager type initializes correctly
✓ Verify all 4 session managers created on barstate.isfirst
✓ Check no script errors on load
✓ Verify memory usage is acceptable
```

#### Phase 3 Tests: Detection Logic
```
✓ Test session detection (London, NY AM, NY PM, Asia)
✓ Test cycle start logic at session boundaries
✓ Test 90-minute cycle completion timing
✓ Test pivot updates during active cycles
✓ Test session transitions (end of one, start of next)
✓ Test history array accumulation
```

#### Phase 4 Tests: SMT Processing
```
✓ Test session cycle SMT detection
✓ Test label format: "London-C2", "NY AM-C3", etc.
✓ Test comparison: current cycle vs previous cycle
✓ Test line drawing for session SMTs
✓ Test invalidation for session SMTs
✓ Test independence: session SMTs don't affect global SMTs
```

#### Phase 5 Tests: Optimization
```
✓ Test input controls (enable/disable sessions)
✓ Test line limits for session SMTs
✓ Test debug mode features
✓ Test performance with all features enabled
✓ Test across different chart timeframes (1min, 5min, 15min)
```

### Phase 2: Integration Testing

#### Scenario 1: Full Day Cycle
```
Test Date: Typical Wednesday
Time Range: 00:00 - 23:59 (24 hours)
Expected:
- Asia session: 5 cycles detected (19:00-02:30)
- London session: 3 cycles detected (02:30-07:00)
- NY AM session: 3 cycles detected (07:00-11:30)
- NY PM session: 3 cycles detected (11:30-16:00)
- Total: 14 session cycles per day

Verify:
✓ All sessions start/end at correct times
✓ All cycles complete at correct intervals
✓ SMTs detected for eligible divergences
✓ No memory leaks (history cleared properly)
```

#### Scenario 2: Session Transition
```
Test Time: 06:55 - 07:05 (London → NY AM transition)
Expected:
- 06:55: London-C3 active
- 07:00: London-C3 completes, London session ends
- 07:00: NY AM session starts, NY AM-C1 begins
- 07:05: NY AM-C1 active

Verify:
✓ London manager.sessionActive = false
✓ NY AM manager.sessionActive = true
✓ London history preserved until next London session
✓ NY AM history cleared from previous day
✓ Correct cycle labels displayed
```

#### Scenario 3: Multiple Simultaneous SMTs
```
Test Scenario: High volatility period
Expected:
- Global 90-min SMT active
- Global 30-min SMT active
- Session "NY AM-C2" SMT active
- All three visible simultaneously

Verify:
✓ Lines don't overlap confusingly
✓ Labels are readable
✓ Colors differentiate global vs session
✓ Invalidation works independently for each
```

#### Scenario 4: Symbol Merging
```
Test Scenario: Multiple comparisons in same cycle
Time: London-C2 (04:00-05:30)
Expected:
- Detect SMT with BTC → Label: "SMT-BTC London-C2"
- Detect SMT with ES → Merge → Label: "SMT-BTC + ES London-C2"
- Detect SMT with NQ → Merge → Label: "SMT-BTC + ES + NQ London-C2"

Verify:
✓ Symbols merge in correct order
✓ " + " separator used
✓ Length calculation correct
✓ Line updates to longest version
✓ No duplicate symbols in label
```

### Phase 3: Regression Testing

```
After each phase completion, verify:
✓ All previous phases still work
✓ Global time cycles unaffected by session cycle additions
✓ Existing session SMT (if kept) still works
✓ Line drawing performance acceptable
✓ No unexpected errors in console
✓ Chart renders correctly on different timeframes
```

### Phase 4: Performance Testing

```
Test Configuration:
- Chart: 1-minute timeframe
- Bars Loaded: 20,000 bars (~14 days of minute data)
- All features enabled

Metrics to Monitor:
✓ Script compilation time < 2 seconds
✓ Bar execution time < 100ms per bar
✓ Memory usage < 500MB
✓ No "Script exceeded memory limit" errors
✓ No "Script execution timed out" errors
✓ Chart renders smoothly (no lag)

Optimization Targets:
- Minimize array.size() calls (cache results)
- Minimize array operations in loops
- Clear old history arrays promptly
- Limit total line objects on chart
```

### Phase 5: Edge Case Testing

#### Edge Case 1: Market Holidays
```
Test: US Market Holiday (e.g., July 4th)
Expected: No session activity, no errors
Verify:
✓ detectActiveSession() returns -1
✓ No cycles started
✓ No SMTs processed
✓ No errors logged
```

#### Edge Case 2: Daylight Saving Time
```
Test: DST transition weekend
Expected: Sessions adjust to new time
Verify:
✓ Session times still align with NYSE hours
✓ Cycle durations remain 90 minutes
✓ No duplicate or missed cycles
```

#### Edge Case 3: Low Liquidity Periods
```
Test: Asia session with minimal price movement
Expected: Cycles run but may not detect SMTs
Verify:
✓ Cycles still complete normally
✓ No errors if no pivots detected
✓ Empty history arrays handled gracefully
```

#### Edge Case 4: Rapid Invalidation
```
Test: Volatile market with quick reversals
Expected: Lines created and invalidated rapidly
Verify:
✓ Invalidation logic works correctly
✓ Lines removed promptly
✓ No orphaned line objects
✓ Memory usage stable
```

### Manual Testing Checklist

```
□ Load script on ES1! (S&P 500 E-mini futures)
□ Set comparison symbol 1 to "BTC1!"
□ Set comparison symbol 2 to "NQ1!"
□ Set timeframe to 5-minute chart
□ Load ~20 days of historical data
□ Enable all features
□ Observe:
  □ Global cycle SMTs appear (solid lines)
  □ Session cycle SMTs appear (dashed lines)
  □ Labels clearly distinguishable
  □ Real-time updates work during active cycles
  □ Invalidation removes broken SMTs
  □ No errors in Pine Logs
  □ No performance issues
□ Test input toggles (disable/enable features)
□ Test different symbol combinations
□ Test on different timeframes (1min, 15min, 1hour)
```


---

## Risk Mitigation

### Potential Risks & Solutions

#### Risk 1: Memory Overflow
**Problem**: Too many line objects and history arrays could exceed Pine Script memory limits.

**Solutions**:
1. **Implement strict line limits**: Default 20 for sessions, 30 for global cycles
2. **Clear history arrays**: Remove old session data when session restarts
3. **FIFO line removal**: Always remove oldest non-active lines first
4. **Configurable limits**: Let users adjust based on their needs

**Code Implementation**:
```pine
// Clear session history when session restarts
if sessionJustStarted
    array.clear(manager.history)  // Don't accumulate across days

// Aggressive FIFO cleanup
if array.size(manager.lines) >= maxLines
    // Remove oldest 3 lines at once to avoid frequent cleanup
    for i = 0 to 2
        if array.size(manager.lines) > 0
            oldestMeta = array.shift(manager.lines)  // Remove from start
            line.delete(oldestMeta.smtLine)
            label.delete(oldestMeta.smtLabel)
```

#### Risk 2: Performance Degradation
**Problem**: Too many loops and array operations per bar could slow execution.

**Solutions**:
1. **Cache array sizes**: Don't call `array.size()` repeatedly in loops
2. **Early exits**: Skip processing if session not active
3. **Limit pivot checks**: Only check pivots if session/cycle active
4. **Optimize invalidation**: Only check active lines, not all lines

**Code Implementation**:
```pine
// Cache frequently used values
var cached_s1_high = na
var cached_s1_low = na
if enableSessionSmt
    cached_s1_high := request.security(symbol1, timeframe.period, high)
    cached_s1_low := request.security(symbol1, timeframe.period, low)

// Early exit pattern
if not enableSessionCycles
    // Skip entire session cycle block

// Only check lines that could be invalidated
for i = array.size(manager.lines) - 1 to 0
    meta = array.get(manager.lines, i)
    if not meta.isForCurrentCycle
        continue  // Skip historical lines, they can't be invalidated by current price
```

#### Risk 3: Incorrect Session Timing
**Problem**: Session times might not align correctly across different timezones.

**Solutions**:
1. **Use explicit timezone**: Always use "America/New_York" for US sessions
2. **Document assumptions**: Clearly state that times are in ET
3. **Add debug markers**: Optional visual markers showing session boundaries
4. **Test across timezones**: Verify script works for users in different locations

**Code Implementation**:
```pine
// Always use explicit timezone
TIME_LONDON = "0230-0700:1234567"  // 02:30-07:00 ET, Mon-Sun
TIME_NY_AM = "0700-1130:1234567"   // 07:00-11:30 ET, Mon-Sun

// Add timezone parameter to time() function
inSession = time(timeframe.period, manager.sessionTime, "America/New_York")

// Debug markers
if debugMode and barstate.isrealtime
    if time == sessionStartTime
        label.new(bar_index, high, "▼ Session Start: " + sessionName)
```

#### Risk 4: Symbol Merging Conflicts
**Problem**: Symbols might merge incorrectly or create duplicate entries.

**Solutions**:
1. **Check for existing symbol**: Use `str.contains()` before adding
2. **Case-insensitive comparison**: Normalize symbol names
3. **Trim whitespace**: Clean symbol strings before comparison
4. **Limit merge count**: Maximum 5 symbols per line

**Code Implementation**:
```pine
// Normalize symbol for comparison
normalizedSymbol = str.upper(str.trim(symbolStr))

// Check if already present
if not str.contains(str.upper(existingMeta.cycleLabel), normalizedSymbol)
    // Safe to add
    newSymbol = existingMeta.cycleLabel + " + " + symbolStr
else
    // Already exists, don't add duplicate

// Limit merge count
symbolCount = str.length(newSymbol) - str.length(str.replace_all(newSymbol, " + ", "")) + 1
if symbolCount > 5
    // Don't merge, create new line instead
```

#### Risk 5: Pine Script Version Compatibility
**Problem**: Code written for v5 might have issues, or v6 features might not work.

**Solutions**:
1. **Stay on v5**: SMT-dev.pine is currently v5, keep it that way
2. **Avoid v6-only features**: Don't use `confirm` parameter or v6 syntax
3. **Test thoroughly**: Verify on TradingView before deployment
4. **Document version**: Clearly mark as v5 in comments

**Code Implementation**:
```pine
//@version=5
// This script requires Pine Script v5
// DO NOT upgrade to v6 without full testing

// Avoid v6-only features:
// ❌ input.symbol(..., confirm=false)  // v6 only
// ✓ input.symbol(...)                  // v5 compatible

// Use v5 syntax:
// ✓ timeframe.in_seconds              // v5 property
// ❌ timeframe.in_seconds(tf)         // v6 function
```

#### Risk 6: Data Request Limits
**Problem**: Too many `request.security()` calls could hit rate limits.

**Solutions**:
1. **Cache security data**: Request once per bar, reuse in all cycles
2. **Minimize calls**: Only request when needed
3. **Use barmerge options**: Optimize data fetching
4. **Error handling**: Gracefully handle na values

**Code Implementation**:
```pine
// Cache security requests at bar start
var float s1_high_cache = na
var float s1_low_cache = na
var float s2_high_cache = na
var float s2_low_cache = na

// Request once per bar
if barstate.isnew
    s1_high_cache := request.security(symbol1, timeframe.period, high, 
        barmerge.gaps_off, barmerge.lookahead_off)
    s1_low_cache := request.security(symbol1, timeframe.period, low,
        barmerge.gaps_off, barmerge.lookahead_off)
    // ... same for s2

// Reuse cached values in all cycle processing
updatePivots(cycleManager, mainHigh, mainLow, s1_high_cache, s1_low_cache, ...)
```

#### Risk 7: Historical Bar Calculation Issues
**Problem**: Historical bars might process differently than real-time bars.

**Solutions**:
1. **Consistent logic**: Ensure same code path for historical and real-time
2. **Avoid `barstate.isrealtime` checks**: Don't change behavior based on state
3. **Test on historical data**: Load many bars and verify correctness
4. **Use `barstate.isconfirmed`**: Wait for bar close when needed

**Code Implementation**:
```pine
// ❌ BAD: Different behavior for real-time vs historical
if barstate.isrealtime
    drawSMT(...)  // Real-time only

// ✓ GOOD: Same behavior for all bars
drawSMT(...)  // Always draw

// Use confirmed state for critical logic
if barstate.isconfirmed and shouldCompleteCycle()
    completeCycle()  // Only complete on confirmed bar close
```

---

## Implementation Timeline

### Conservative Estimate (Total: 15-20 hours)

```
Phase 1: V-3.0 Improvements (4-5 hours)
├─ Code changes: 2 hours
├─ Testing: 1.5 hours
└─ Bug fixes: 0.5-1.5 hours

Phase 2: Infrastructure (3-4 hours)
├─ Type definitions: 1 hour
├─ Initialization: 1 hour
├─ Testing: 1-2 hours

Phase 3: Detection Logic (5-6 hours)
├─ Session detection: 2 hours
├─ Cycle management: 2 hours
├─ Pivot updates: 1 hour
└─ Testing: 1-2 hours

Phase 4: SMT Processing (4-5 hours)
├─ Integration: 2 hours
├─ Line drawing: 1 hour
├─ Invalidation: 1 hour
└─ Testing: 1-2 hours

Phase 5: Optimization (2-3 hours)
├─ Input controls: 0.5 hour
├─ Debug mode: 0.5 hour
├─ Performance tuning: 1 hour
└─ Final testing: 0.5-1 hour

Total: 18-23 hours
```

### Aggressive Estimate (Total: 10-12 hours)

```
Phase 1: 2.5 hours
Phase 2: 2 hours
Phase 3: 3.5 hours
Phase 4: 3 hours
Phase 5: 1.5 hours

Total: 12.5 hours
```

**Note**: Aggressive estimate assumes:
- Minimal bugs encountered
- Clean implementation on first try
- Limited testing required
- No major refactoring needed

### Phased Rollout Strategy

**Option A: All-at-Once (Recommended for Clean Implementation)**
```
Week 1: Phases 1-2 (Infrastructure)
Week 2: Phases 3-4 (Core Logic)
Week 3: Phase 5 + Full Testing
Week 4: Production deployment
```

**Option B: Incremental (Safer, but more time)**
```
Week 1: Phase 1 only → Deploy → Test
Week 2: Phase 2 only → Deploy → Test
Week 3: Phase 3 only → Deploy → Test
Week 4: Phase 4 only → Deploy → Test
Week 5: Phase 5 → Final deployment
```

**Recommendation**: Use Option A for cleaner codebase and faster completion.

---

## Success Criteria

### Functional Requirements

✅ **Global Time Cycles**: Continue working exactly as before
- 90-min, 30-min, 10-min, 3-min cycles detect SMTs
- Lines draw correctly with proper labels
- Invalidation works correctly
- Symbol merging works with " + " separator

✅ **Session Cycle Detection**: New feature fully functional
- London: 3 cycles per session (02:30-07:00)
- NY AM: 3 cycles per session (07:00-11:30)
- NY PM: 3 cycles per session (11:30-16:00)
- Asia: 5 cycles per session (19:00-02:30)

✅ **Session Cycle SMTs**: Detected and displayed correctly
- Compares current cycle vs previous cycle within session
- Labels format: "SMT-BTC London-C2", etc.
- Dashed lines for visual distinction
- Independent from global cycle SMTs

✅ **Real-Time Updates**: Lines update as cycles progress
- Dynamic lines extend with new pivots
- Invalidation removes broken SMTs immediately
- Merging works correctly during active cycles

✅ **Independence**: Both systems work simultaneously
- Global cycles don't affect session cycles
- Session cycles don't affect global cycles
- Separate line management
- Independent invalidation

### Performance Requirements

✅ **Execution Speed**: Script executes efficiently
- Bar processing time < 100ms
- No timeout errors
- Smooth chart rendering

✅ **Memory Usage**: Within Pine Script limits
- No memory overflow errors
- Line limits enforced
- History arrays managed properly

✅ **Visual Quality**: Chart remains readable
- Lines don't overlap excessively
- Labels are clear and distinguishable
- Colors differentiate types effectively

### Code Quality Requirements

✅ **Maintainability**: Code is clean and documented
- Clear function names
- Commented complex logic
- Consistent naming conventions
- Modular structure

✅ **Testability**: Easy to test and debug
- Debug mode available
- Console logging for key events
- Visual markers for cycle boundaries

✅ **Extensibility**: Easy to add features later
- Modular session manager design
- Easy to add new sessions
- Easy to add new cycle durations
- Easy to add new SMT types

---

## Future Enhancements (Post-MVP)

### Enhancement 1: Multiple Cycle Durations per Session
**Description**: Add 30-min and 10-min cycles within sessions

**Example**:
```
London Session (270 min):
├─ 3x 90-min cycles (current)
├─ 9x 30-min cycles (new)
└─ 27x 10-min cycles (new)
```

**Benefit**: More granular session-based SMT detection

### Enhancement 2: Cross-Session SMT Detection
**Description**: Compare cycles across different sessions

**Example**:
```
Compare:
  London-C3 (last cycle of London)
  vs
  NY AM-C1 (first cycle of NY AM)

Detect SMT across session boundary
```

**Benefit**: Catch momentum shifts at key transition points

### Enhancement 3: Session Volume Profile
**Description**: Track volume distribution within session cycles

**Example**:
```
London-C2 Volume: High
London-C3 Volume: Low
→ Volume declining with price rising
→ Bearish divergence signal
```

**Benefit**: Add volume confirmation to SMTs

### Enhancement 4: Alert System
**Description**: Generate alerts for session cycle SMTs

**Example**:
```
Alert Trigger: New session cycle SMT detected
Alert Message: "Bearish SMT on BTC, NY AM-C2"
Alert Conditions:
  - Only for specific sessions
  - Only for specific symbols
  - Only for specific SMT types
```

**Benefit**: Real-time notifications for trading decisions

### Enhancement 5: Statistical Analysis
**Description**: Track success rate of session cycle SMTs

**Example**:
```
London-C2 Bearish SMTs:
  Total: 45
  Successful (price dropped): 32
  Success Rate: 71%

NY AM-C3 Bullish SMTs:
  Total: 38
  Successful (price rallied): 25
  Success Rate: 66%
```

**Benefit**: Identify highest probability setups

---

## Conclusion

This implementation plan provides a complete roadmap for upgrading `SMT-dev.pine` with dual-layer SMT detection. The architecture is designed to be:

- **Robust**: Handles edge cases and errors gracefully
- **Performant**: Optimized for real-time execution
- **Maintainable**: Clean, modular code structure
- **Extensible**: Easy to add features in the future
- **Tested**: Comprehensive testing strategy

By following this plan phase-by-phase, the implementation should take 15-20 hours of development time with a high confidence of success.

### Key Takeaways

1. **Two independent systems**: Global cycles + Session cycles
2. **Code reuse**: Same SMT detection logic for both
3. **Visual distinction**: Solid lines (global) vs dashed lines (sessions)
4. **Clear labels**: "90" vs "London-C2"
5. **Real-time updates**: Dynamic lines during active cycles
6. **Smart invalidation**: Each system tracks independently

### Next Steps

1. Review this plan thoroughly
2. Ask any clarifying questions
3. Begin Phase 1 implementation
4. Test each phase before moving to next
5. Deploy to production after Phase 5 complete

---

**Document Version**: 1.0  
**Created**: 2024  
**Last Updated**: 2024  
**Author**: Rovo Dev  
**Status**: Planning Complete - Ready for Implementation

