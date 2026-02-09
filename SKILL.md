# SKILL: Quarter.pine Development

## 1. Overview
We are building a brand new indicator `Quarter.pine`.
**Goal**: Visualize "Ultra Accurate" time quarters with line fills and bottom boxes, and detect SMT live between the *Current Quarter* and the *Previous Completed Quarter*.

## 2. Requirements & Reference Logic

### A. Visuals
1.  **Day of the Week**:
    -   **Location**: Top of chart.
    -   **Reference**: `Time-Cycles-V4.pine` (lines 617-658).
    -   **Logic**: Use `plotshape` with `location.top`.
2.  **Quarter Line Fills**:
    -   **Location**: On chart, spanning the Quarter's duration and price range.
    -   **Reference**: `Time-Cycles-V4.pine` (lines 223-260 `draw_cycle_box`).
    -   **Style**: Semi-transparent fills distinguishing separate quarters.
3.  **Bottom Horizontal Quarters Table**:
    -   **Location**: Bottom of the chart/pane.
    -   **Style**: "Boxes" (likely `box.new` or `plotshape` at `location.bottom`).
    -   **Purpose**: Visual timeline of quarters.

### B. Timings & Logic (Confirmed)

#### 1. Session Quarters (90 Minutes)
Each 6-hour session is divided into 4 equal quarters of 90 minutes.
*   **ASIA** (18:00 - 00:00)
    *   Q1: 18:00 - 19:30
    *   Q2: 19:30 - 21:00 Asia Open
    *   Q3: 21:00 - 22:30
    *   Q4: 22:30 - 00:00
*   **LONDON** (00:00 - 06:00) TRUE DAY OPEN
    *   Q1: 00:00 - 01:30
    *   Q2: 01:30 - 03:00 London Open
    *   Q3: 03:00 - 04:30
    *   Q4: 04:30 - 06:00
*   **NY AM** (06:00 - 12:00)
    *   Q1: 06:00 - 07:30
    *   Q2: 07:30 - 09:00 NY AM Open
    *   Q3: 09:00 - 10:30
    *   Q4: 10:30 - 12:00
*   **NY PM** (12:00 - 18:00)
    *   Q1: 12:00 - 13:30
    *   Q2: 13:30 - 15:00 NY PM Open
    *   Q3: 15:00 - 16:30
    *   Q4: 16:30 - 18:00

#### 2. Daily Quarters (6 Hours)
The "True Open" day (18:00 - 18:00) is divided into 4 equal quarters of 6 hours.
*   **Q1**: 18:00 - 00:00 (Matches Asia Session)
*   **Q2**: 00:00 - 06:00 (Matches London Session)
*   **Q3**: 06:00 - 12:00 (Matches NY AM Session)
*   **Q4**: 12:00 - 18:00 (Matches NY PM Session)

#### 3. Weekly Quarters (Days)
*   **Concept**: The week is treated as a cycle with days as quarters.
*   **Q1**: Monday
*   **Q2**: Tuesday
*   **Q3**: Wednesday
*   **Q4**: Thursday
*   **Exclusion**: Friday is excluded from box highlighting/calculations.

#### 4. Monthly Quarters (Weeks)
*   **Concept**: A 4-week cycle makes up a month, where each week is a quarter.
*   **Q1**: Week 1
*   **Q2**: Week 2
*   **Q3**: Week 3
*   **Q4**: Week 4
*   **Logic**: Reset at start of new month? or rolling 4 weeks?
    *   *Assumption*: Standard calendar weeks within a month. If a month has >4 weeks, 5th week is ignored or part of Q4?
    *   *Correction based on "4 weeks"*: Likely usage of `weekofmonth` or specific weekly cycle logic.

### C. SMT Logic
-   **Reference**: `SMT_V-4.0.pine` (lines 647-655 `detectSMT`).
-   **New Logic**:
    -   **Comparison**: `Current Running Quarter` vs `Previous COMPLETED Quarter`.
    -   **Metrics**: High, Low, Open, Close? (Usually High/Low for SMT).
    -   **Live Detection**: Must update on every tick.
    -   *Example*: If Q2 is running, compare Q2 High/Low vs Q1 High/Low.

## 3. Plan
1.  **Setup**: Initialize script inputs and define Quarter Timings (once provided).
2.  **Visuals**: Implement Day of Week labels (Top) and Bottom Boxes.
3.  **State Management**: Track state of "Current Quarter" and store "Previous Quarter" data.
4.  **SMT Engine**: Implement the `detectSMT` function adapting `SMT_V-4.0` logic to use Quarter objects.
5.  **Rendering**: Draw fills and SMT lines/labels.

## 4. Implementation Details
-   **Inputs**:
    -   Toggles for displaying Session Quarters, Daily Quarters, and Weekly Quarters.
    -   Colors for Q1, Q2, Q3, Q4 boxes/fills.
-   **Visuals**:
    -   **Session Quarters**: Line fills (semi-transparent).
    -   **Weekly Quarters**: Boxes/Fills for Mon-Thu. Friday is blank.
    -   **Bottom Table**: Visual representation of the active quarter (Session/Day/Week).

