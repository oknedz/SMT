# Pine Script Debugging Guide

## Introduction

TradingView's close integration between the Pine Editor and the Supercharts interface enables efficient, interactive debugging of Pine Script® code. Pine scripts can create dynamic outputs in multiple locations, on and off the chart. Programmers can use these outputs to validate their scripts' behaviors and ensure everything works as expected.

Understanding the most effective tools and methods for inspecting a script helps programmers quickly find and fix potential problems in their code, which improves the overall coding experience. This page explains the script outputs that are the most useful for debugging, along with helpful tips and techniques.

> **Tip**: Effective debugging in the Pine Script environment requires an understanding of the Execution model, Time series structure, and Type system. We recommend reviewing these topics, along with string formatting, which the following techniques often use.

---

## Common Debug Outputs

Pine scripts can create outputs in several ways, each of which has different advantages. While programmers can use any of them to debug their code, some outputs are more optimal for debugging than others.

### 1. Pine Logs (`log.*` namespace)

The functions in the `log.*` namespace log interactive messages in the Pine Logs pane. These logging functions are the most convenient and flexible tools for debugging Pine code. Scripts can call `log.*()` functions on any execution from global or local scopes, enabling programmers to analyze historical and realtime script behaviors in depth with minimal code.

**Example:**
```pinescript
//@version=6
indicator("Common debug outputs - Pine Logs")

//@variable The natural logarithm of the current `high - low` range.
float logRange = math.log(high - low)

// Plot the `logRange`.
plot(logRange, "logRange")

if barstate.isconfirmed
    // Generate an "error" or "info" message on the confirmed bar, depending on whether `logRange` is defined.
    switch 
        na(logRange) => log.error("Undefined `logRange` value.")
        =>              log.info("`logRange` value: " + str.tostring(logRange))
else
    // Generate a "warning" message for unconfirmed values.
    log.warning("Unconfirmed `logRange` value: " + str.tostring(logRange))
```

### 2. Pine Drawings

Pine drawings display visuals in the main chart pane or the script's separate pane. Although they do not output results in other locations, such as the Data Window or Pine Logs pane, drawings provide convenient ways to visualize a script's data and logic within global or local scopes. Labels are the most flexible drawings for debugging, because they can display colored shapes with formatted text and tooltips at any available chart location.

**Example:**
```pinescript
//@version=6
indicator("Common debug outputs - Pine drawings", overlay = true)

//@variable Is `true` when a new bar opens on the "1D" timeframe.
bool newDailyBar = timeframe.change("1D")
//@variable The previous bar's `bar_index` from when `newDailyBar` last occurred.
int closedIndex = ta.valuewhen(newDailyBar, bar_index - 1, 0)
//@variable The previous bar's `close` from when `newDailyBar` last occurred.
float closedPrice = ta.valuewhen(newDailyBar, close[1], 0)

if newDailyBar
    // Draw a line from the previous `closedIndex` and `closedPrice` to the current values.
    line.new(closedIndex[1], closedPrice[1], closedIndex, closedPrice, width = 2)
    //@variable A string containing debug information to display in a label.
    string debugText = "'1D' bar closed at: \n(" + str.tostring(closedIndex) + ", " + str.tostring(closedPrice) + ")"
    //@variable Draws a label at the current `closedIndex` and `closedPrice`.
    label debugLabel = label.new(closedIndex, closedPrice, debugText, color = color.purple, textcolor = color.white)
```

### 3. Plot Functions (`plot*()`)

The `plot*()` functions can help to debug numeric values, conditions, and colors from a script's global scope. They can output results in up to four locations: the main chart pane or the script's pane, the status line, the price scale, and the Data Window. The display on the chart provides a quick view of the series' history, and the numbers in the other output locations show calculated information for specific bars.

**Example:**
```pinescript
//@version=6
indicator("Common debug outputs - Plots")

// Plot the `bar_index` in all available locations.
plot(bar_index, "bar_index", color.teal, 3)
```

### 4. Background and Bar Colors

The `bgcolor()` function displays colors in the background of the main chart pane or the script's pane. The `barcolor()` function colors the main chart's bars or candles. Although these outputs are less flexible than Pine Logs, drawings, and plots, they provide a quick way to inspect calculated colors and visualize conditions from the global scope.

**Example:**
```pinescript
//@version=6
indicator("Common debug outputs - Background and bar colors")

//@variable Is `true` if the `close` is rising over 2 bars.
bool risingPrice = ta.rising(close, 2)

// Highlight the chart background and color the main chart bars based on `risingPrice`.
bgcolor(risingPrice ? color.new(color.green, 70) : na, title= "`risingPrice` highlight")
barcolor(risingPrice ? color.aqua : chart.bg_color, title = "`risingPrice` bar color")
```

---

## Pine Logs

Pine Logs are interactive, user-defined messages that scripts can create from within global or local scopes at any point during code executions on the chart's dataset or requested datasets. They provide a simple, powerful way for programmers to inspect a script's calculations, logic, and execution flow with human-readable text. Using Pine Logs is the primary, most universal technique for debugging Pine Script code.

Pine Logs do not appear on the chart or in the Data Window. Instead, scripts print logged messages with prefixed date and time information in the dedicated Pine Logs pane. The inspection and filtering options in the Pine Logs pane help users analyze and navigate logs efficiently.

### Accessing Pine Logs

To access the pane, select "Pine Logs" from the Pine Editor's "More" menu or from the "More" menu in the status line of a script on the chart that uses the `log.*()` functions.

> **Notice**: Only personal scripts can generate Pine Logs. A published script cannot create logs, even if its source code contains `log.*()` function calls. Published libraries can export functions containing `log.*()` calls for use in personal scripts, but they cannot generate logs directly.

### Creating Logs

Scripts create Pine Logs by calling the functions in the `log.*` namespace: `log.info()`, `log.warning()`, or `log.error()`. All these logging functions have the following two signatures:

```pinescript
log.*(message) → void
log.*(formatString, arg0, arg1, ...) → void
```

Where:
- The first overload prints the specified "string" message in the Pine Logs pane.
- The second overload creates a formatted string based on its formatString and additional arguments, similar to `str.format()`, then displays the resulting text inside the pane.

#### Logging Levels

Each `log.*()` function has a different logging level, allowing programmers to categorize the messages shown in the Pine Logs pane:

- **`log.info()`**: Creates a message with the "info" level (gray text).
- **`log.warning()`**: Creates a message with the "warning" level (orange text).
- **`log.error()`**: Creates a message with the "error" level (red text).

**Example - Logging Levels:**
```pinescript
//@version=6
indicator("Logging levels demo", overlay = true)

// Display logs with all three logging levels in the Pine Logs pane on the first bar. 
if barstate.isfirst
    log.info("This is an 'info' message.")
    log.warning("This is a 'warning' message.")
    log.error("This is an 'error' message.")
```

### Historical vs Realtime Logs

- During **historical executions**, scripts log a new message once for each `log.*()` call on any bar.
- During **realtime executions**, scripts can call the `log.*()` functions to log messages for any available tick, regardless of whether the bar is confirmed.
- The logs created on realtime ticks are not subject to rollback.
- All logs remain available in the Pine Logs pane until the script restarts.

**Example - Historical and Realtime Logging:**
```pinescript
//@version=6
indicator("Historical and realtime logs demo", "Average bar ratio")

//@variable The current bar's change from the `open` to `close`.
float numerator = close - open
//@variable The current bar's `low` to `high` range.
float denominator = high - low
//@variable The ratio of the bar's open-to-close change to its full range.
float ratio = numerator / denominator
//@variable The average `ratio` over 10 *non-na* values.
float average = ta.sma(ratio, 10)

// Plot the `average`.
plot(average, "average", color.purple, 3)

if barstate.isconfirmed
    switch denominator
        // Log an "error" message when the `denominator` is 0.
        0.0 => log.error("Division by 0 on confirmed bar!\nBar excluded from the average.")
        // Otherwise, log an "info" message containing a formatted representation of the variables' confirmed values.
        => log.info(
             "Values (Confirmed):
             \nnumerator: {0,number,#.########}
             \ndenominator: {1,number,#.########}
             \nratio: {2,number,#.########}
             \naverage: {3,number,#.########}",
             numerator, denominator, ratio, average
         )
else
    switch denominator
        // Log an "error" message for the unconfirmed bar when the `denominator` is 0.
        0.0 => log.error("Division by 0 on unconfirmed bar!")
        // Otherwise, log a "warning" message containing a formatted representation of the unconfirmed values.
        => log.warning(
             "Values (unconfirmed):
             \nnumerator: {0,number,#.########}
             \ndenominator: {1,number,#.########}
             \nratio: {2,number,#.########}
             \naverage: {3,number,#.########}",
             numerator, denominator, ratio, average
         )
```

### Key Notes

- **Confirmed bars only**: Programmers can use `barstate.isconfirmed` in the conditions that trigger `log.*()` calls to allow logs for any realtime bar only once, on its closing tick.
- **Pause realtime logs**: Users can pause realtime logs by selecting the "Disable logging" button at the top of the Pine Logs pane.
- **Volume management**: Allowing logging on any tick of an open bar can result in a large number of logged messages over time. Therefore, we recommend including unique information in the messages or using different logging levels for easy filtering from the Pine Logs pane.
- **Log limits**: The Pine Logs pane can display the most recent 10,000 logs for historical bars. If a programmer needs to view earlier logs, they can add logic in the code to filter specific `log.*()` calls.

### Inspecting Logs

When a script generates a log by calling any `log.*()` function call, the Pine Logs pane automatically prefixes the logged message with an **ISO 8601 timestamp** representing the log's assigned time, expressed in the chart's time zone:

- **Historical bars**: The timestamp represents the bar's opening time
- **Realtime logs**: The timestamp represents the system time of the log event

#### Log Features

Each log includes **"Source code"** and **"Scroll to bar"** options, which appear when hovering over the message in the Pine Logs pane:

- **"Source code"**: Opens the script in the Pine Editor and highlights the code line containing the specific `log.*()` call that triggered the log event
- **"Scroll to bar"**: Navigates the chart to the bar where the `log.*()` call occurred, then displays a temporary label above the bar, containing its date and time information

> **Note**: The label's time information depends on the chart's timeframe. For example, the label on a "1D" chart contains only the weekday and date, whereas the label on an intraday chart also includes the time of day.

#### Multiple Scripts

Every script on the chart that generates logs maintains an independent log history. The Pine Logs pane shows logs for only one script at a time. To inspect the logs from a specific script when multiple are on the chart, select its title from the dropdown menu at the top of the pane.

### Filtering Logs

The Pine Logs pane displays up to 10,000 logged messages from script executions on historical bars. It then appends a new log for each `log.*()` call executed on any realtime tick.

#### Built-in Filters

To help users navigate high volumes of logs efficiently, the pane includes filters that isolate logs based on:

- **Logging level** (info, warning, error)
- **Start date and time**
- **Search queries**

Users can apply these log filters individually or in combination to show only the messages that meet specific criteria. The filters are accessible from the icons below the "x" in the top-right portion of the pane.

#### Custom Code Filters

For custom filtering options, programmers can use conditional logic to activate specific `log.*()` calls selectively across a script's executions.

---

## Summary

Programmers can use any of these debugging outputs individually or in combination to debug their scripts, depending on the data types and structures that require inspection. The most effective approach often involves:

1. **Pine Logs** for detailed inspection of values and execution flow
2. **Drawings** for visual representation of data on charts
3. **Plots** for numeric value tracking across time series
4. **Colors** for quick visual feedback on conditions

Choose the debugging method that best fits your specific use case and debugging requirements.