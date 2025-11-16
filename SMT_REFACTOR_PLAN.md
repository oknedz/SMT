# SMT.pine - COMPREHENSIVE REFACTORING PLAN
## Code Improvement & Optimization Strategy

**Current State:** 578 lines | Multiple code duplications | Hard to maintain
**Target:** ~300-350 lines | DRY principles | Highly maintainable | More efficient

---

## 🔴 CRITICAL ISSUES IDENTIFIED

### 1. **MASSIVE CODE DUPLICATION** (PRIMARY ISSUE)
**Problem:** 4 nearly identical processSMT functions (90/30/10/3) - ~300 lines of duplicated code
- Lines 271-362: processSMT90 (92 lines)
- Lines 366-454: processSMT30 (89 lines) 
- Lines 458-515: processSMT10 (58 lines)
- Lines 518-574: processSMT3 (57 lines)

**Impact:** 
- Makes bug fixes 4x harder
- Maintenance nightmare
- Wasted memory/processing

### 2. **INEFFICIENT DATA STRUCTURES**
**Problem:** Separate cycle variables instead of unified structure
- Lines 66-74: 4 separate cycle instances + 4 separate arrays
- Lines 238-266: Repetitive cycle management code

**Impact:**
- Cannot easily iterate over cycles
- Leads to code duplication
- Hard to add new cycle types

### 3. **HARDCODED SMT TYPE INDICES**
**Problem:** Magic numbers (0-7) scattered throughout
- Lines 276, 285, 290, 296, 302, 311, 317, 323, 329
- Lines 350-351: Manual index checking with OR operators

**Impact:**
- Error-prone
- Hard to understand
- Difficult to extend

### 4. **INEFFICIENT SMT DETECTION LOGIC**
**Problem:** 8 separate if-statements for SMT detection per cycle
- Lines 284-334: Nested conditionals checking same cycle repeatedly
- Repeats for EVERY processSMT function

**Impact:**
- Poor performance
- Hard to read
- Maintenance burden

### 5. **REPETITIVE LABEL/ARRAY CREATION**
**Problem:** Same arrays created in every processSMT call
- Lines 344-345, 436-437, 502-503, 562-563
- Identical array.from() calls with hardcoded values

**Impact:**
- Wastes CPU cycles on every bar
- Memory inefficient

---

## ✅ PROPOSED SOLUTION ARCHITECTURE

### **PHASE 1: DATA STRUCTURE CONSOLIDATION**

#### 1.1 Create Unified Cycle Configuration Type
```pinescript
type CycleConfig
    int duration          // in minutes
    string label          // "90", "30", "10", "3"
    bool enabled          // enable90min, etc.
    bool requiresSpecialTF // for 3-min cycle
```

#### 1.2 Create Cycle Manager Type
```pinescript
type CycleManager
    CycleConfig config
    CycleData current
    array<CycleData> history
```

#### 1.3 Replace Individual Cycles with Array
**BEFORE:** (Lines 66-74)
```
var cycle90, cycle30, cycle10, cycle3 + 4 arrays
```

**AFTER:**
```pinescript
var array<CycleManager> cycleManagers = array.new<CycleManager>()
// Initialize with configs in setup
```

**Benefit:** 40+ lines → ~10 lines, easily extensible

---

### **PHASE 2: SMT TYPE ENUMERATION & METADATA**

#### 2.1 Create SMT Type Enum/Constants
```pinescript
// SMT Type identifiers
const int SMT_S1_BEARISH = 0
const int SMT_S1_BULLISH = 1
const int SMT_S2_BEARISH = 2
const int SMT_S2_BULLISH = 3
const int SMT_S1_INV_BEARISH = 4
const int SMT_S1_INV_BULLISH = 5
const int SMT_S2_INV_BEARISH = 6
const int SMT_S2_INV_BULLISH = 7
```

#### 2.2 Create SMT Configuration Type
```pinescript
type SMTConfig
    int typeId
    bool isSymbol1          // vs symbol2
    bool isHigh             // vs low
    bool isBearish          // vs bullish  
    bool isInverse          // inverse vs normal
    string labelPrefix      // "SMT-"
```

#### 2.3 Build SMT Metadata Array (ONE TIME)
```pinescript
var array<SMTConfig> smtConfigs = createSMTConfigs()

createSMTConfigs() =>
    configs = array.new<SMTConfig>()
    array.push(configs, SMTConfig.new(0, true, true, true, false, "SMT"))
    // ... 7 more
    configs
```

**Benefit:** Eliminates hardcoded indices, self-documenting

---

### **PHASE 3: UNIFIED SMT DETECTION ENGINE**

#### 3.1 Generic SMT Detector Function
**REPLACES:** 4 separate detect functions (Lines 143-153)

```pinescript
detectSMT(mainCurr, mainPrev, compCurr, compPrev, isBearish, isInverse) =>
    if not (not na(mainCurr) and not na(mainPrev) and not na(compCurr) and not na(compPrev))
        false
    else
        if isInverse
            isBearish ? (compCurr > compPrev and mainCurr < mainPrev) : 
                        (compCurr < compPrev and mainCurr > mainPrev)
        else
            isBearish ? (mainCurr > mainPrev and compCurr < compPrev) : 
                        (mainCurr < mainPrev and compCurr > compPrev)
```

**Benefit:** 4 functions → 1 function, easier to test

---

### **PHASE 4: UNIFIED SMT PROCESSING ENGINE**

#### 4.1 Single Master processSMT Function
**REPLACES:** All 4 processSMT functions (~300 lines → ~80 lines)

```pinescript
processSMT(cycleManager, smtConfigs) =>
    cycle = cycleManager.current
    prevCycles = cycleManager.history
    
    if not cycle.active or array.size(prevCycles) == 0
        false
    
    smtResults = array.new<CycleData>(8, na)
    alertMsg = ""
    hasAlert = false
    
    // Single loop through previous cycles
    for i = 0 to array.size(prevCycles) - 1
        prevCycle = array.get(prevCycles, i)
        cycleHasSMT = checkAllSMTTypes(cycle, prevCycle, smtResults, smtConfigs)
        
        if not cycleHasSMT
            markForRemoval(prevCycles, i)
    
    // Draw all detected SMTs
    drawSMTResults(cycle, smtResults, smtConfigs, cycleManager.config.label)
    
    hasAlert
```

#### 4.2 Helper: Check All SMT Types
```pinescript
checkAllSMTTypes(current, previous, results, configs) =>
    foundAny = false
    for i = 0 to array.size(configs) - 1
        config = array.get(configs, i)
        if shouldCheckType(config) and na(array.get(results, i))
            if detectSMTForType(current, previous, config)
                array.set(results, i, previous)
                foundAny := true
    foundAny
```

**Benefit:** 
- 300 lines → 80 lines (73% reduction)
- Single source of truth
- Bug fix once, fixes all cycles

---

### **PHASE 5: CYCLE MANAGEMENT OPTIMIZATION**

#### 5.1 Loop-Based Cycle Processing
**REPLACES:** Lines 238-266 (repetitive if/else)

```pinescript
if inSession()
    for i = 0 to array.size(cycleManagers) - 1
        manager = array.get(cycleManagers, i)
        
        if not manager.current.active and manager.config.enabled
            if not requiresSpecialCheck(manager) or meetsSpecialCheck(manager)
                startNewCycle(manager.current)
        
        if manager.config.enabled
            if shouldCompleteCycle(manager.current, getCycleDuration(manager.config.duration))
                completeCycle(manager)
            else
                updateCycle(manager.current)
```

**Benefit:** 
- 28 lines → ~12 lines
- Easy to add new cycles
- Consistent logic

---

### **PHASE 6: PERFORMANCE OPTIMIZATIONS**

#### 6.1 Cache Computed Values
```pinescript
// Compute once per bar, not per cycle
var symbolData1 = na
var symbolData2 = na

if barstate.isnew
    symbolData1 := SymbolData.new(s1_high, s1_low, enableSymbol1, getSymbolAbbr(symbol1))
    symbolData2 := SymbolData.new(s2_high, s2_low, enableSymbol2, getSymbolAbbr(symbol2))
```

#### 6.2 Early Exit Patterns
```pinescript
// Check session once
if not inSession()
    return

// Check array size once
if array.size(prevCycles) == 0
    return
```

#### 6.3 Reduce Array Operations
- Pre-allocate arrays with known sizes
- Use direct access instead of repeated get/set where possible

---

## 📊 EXPECTED IMPROVEMENTS

### Code Metrics
| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| **Total Lines** | 578 | ~300-350 | 40-48% reduction |
| **ProcessSMT Lines** | ~300 | ~80 | 73% reduction |
| **Cycle Management** | 28 | ~12 | 57% reduction |
| **Functions** | 15 | ~12 | More reusable |
| **Duplicated Code** | High | Minimal | DRY compliant |

### Performance Metrics
- **CPU Usage:** ~20-30% reduction (fewer redundant checks)
- **Memory:** ~15-25% reduction (shared data structures)
- **Maintainability:** 10x easier to modify/extend

### Developer Experience
✅ Add new cycle: Change 1 line vs 50+ lines  
✅ Fix bug: Fix once vs 4 times  
✅ Add new SMT type: Add config vs scatter changes  
✅ Understand code: Clear structure vs hunt through duplicates  

---

## 🎯 IMPLEMENTATION STRATEGY

### Stage 1: Foundation (Low Risk)
1. Add constants for SMT types
2. Create new types (CycleConfig, CycleManager, SMTConfig)
3. Add helper functions alongside existing code
4. Test that nothing breaks

### Stage 2: Data Migration (Medium Risk)
1. Create cycleManagers array
2. Populate with existing data
3. Run in parallel with old system
4. Validate outputs match

### Stage 3: Logic Consolidation (High Impact)
1. Implement unified processSMT
2. Test exhaustively with one cycle type
3. Gradually replace old functions
4. Remove deprecated code

### Stage 4: Optimization (Polish)
1. Add caching
2. Optimize loops
3. Profile performance
4. Fine-tune

---

## 🔧 ADDITIONAL RECOMMENDATIONS

### Code Organization
```
// ═══ SECTION 1: TYPES & CONSTANTS ═══
// ═══ SECTION 2: CONFIGURATION ═══
// ═══ SECTION 3: CORE LOGIC ═══
// ═══ SECTION 4: SMT DETECTION ═══
// ═══ SECTION 5: VISUALIZATION ═══
// ═══ SECTION 6: EXECUTION ═══
```

### Function Naming Convention
- Verbs for actions: `detectSMT()`, `drawLine()`, `updateCycle()`
- Getters: `getCycleData()`, `getSMTConfig()`
- Checkers: `shouldAddLine()`, `isValidCycle()`

### Comments & Documentation
- Add brief function docstrings
- Explain complex logic inline
- Document type fields clearly

### Testing Strategy
- Create test cycle data
- Validate each phase separately
- Compare outputs before/after refactor
- Use debug plots to verify logic

---

## 🚀 QUICK WINS (Implement First)

1. **Extract constants** (5 mins, zero risk)
   - SMT type IDs, magic numbers
   
2. **Create SMTConfig type** (10 mins, low risk)
   - Consolidates metadata
   
3. **Unify SMT detection** (20 mins, medium risk)
   - 4 functions → 1 function
   
4. **Cache symbol data** (5 mins, low risk)
   - Immediate perf boost

---

## ⚠️ RISKS & MITIGATION

### Risk 1: Logic Changes During Refactor
**Mitigation:** 
- Refactor structure, not logic
- Keep same algorithms
- Test against known good outputs

### Risk 2: Performance Regression
**Mitigation:**
- Profile before/after
- Keep optimization as separate phase
- Can rollback if needed

### Risk 3: Breaking Existing Behavior
**Mitigation:**
- Extensive testing
- Parallel run old/new code
- Gradual migration

---

## 📝 SUMMARY

Your code works but suffers from extreme duplication making it hard to maintain. The refactor focuses on:

1. **DRY Principle:** Eliminate 300+ lines of duplication
2. **Data-Driven:** Use configurations instead of hardcoded logic
3. **Extensibility:** Easy to add cycles/symbols/SMT types
4. **Performance:** Reduce redundant operations
5. **Maintainability:** Clear structure, single source of truth

**ESTIMATED EFFORT:** 4-6 hours for full refactor
**ESTIMATED TIME SAVED:** 10x faster future modifications
**CODE QUALITY:** Professional-grade, maintainable, efficient

Would you like me to proceed with implementing any specific phase?
