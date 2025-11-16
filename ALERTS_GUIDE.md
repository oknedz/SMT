# 🔔 SMT ALERTS - USER GUIDE

## ✅ ALERTS ADDED FOR 90 & 30 MINUTE CYCLES

Your **SMT_V2.pine** now includes smart alerts that notify you when SMT patterns are detected!

---

## 🎯 WHAT YOU GET

### **90-Minute Cycle Alerts**
- Fires when SMT detected in 90-min cycle
- Can be enabled/disabled independently
- Alert includes SMT type and symbol

### **30-Minute Cycle Alerts**
- Fires when SMT detected in 30-min cycle
- Can be enabled/disabled independently
- Alert includes SMT type and symbol

---

## 🔧 HOW TO ENABLE ALERTS

### Step 1: Enable in Indicator Settings
1. Open **SMT_V2.pine** in TradingView
2. Go to **Indicator Settings**
3. Find **"Alerts"** section
4. ✅ Enable **"90-Minute SMT Alerts"** (default: ON)
5. ✅ Enable **"30-Minute SMT Alerts"** (default: ON)
6. Click **OK**

### Step 2: Create Alert in TradingView
1. Right-click on chart
2. Select **"Add alert"**
3. **Condition:** Select your SMT indicator
4. **Alert name:** "SMT Detection"
5. **Message:** You can customize or use default
6. **Options:** Choose notification method (popup, email, webhook, etc.)
7. Click **Create**

---

## 📱 ALERT MESSAGE FORMAT

### Example Alert Messages:

**Single SMT Detected:**
```
90-Min Bearish SMT-BTC
```

**Multiple SMTs Detected:**
```
30-Min Bullish SMT-BTC, Inverse Bearish SMT-ES
```

**Message Components:**
- **Cycle:** 90-Min or 30-Min
- **Direction:** Bearish or Bullish
- **Type:** (empty for normal, "Inverse" for inverse SMT)
- **Symbol:** Abbreviated symbol name (e.g., BTC, ES)

---

## 🎯 ALERT TYPES EXPLAINED

### Normal Bearish SMT
```
90-Min Bearish SMT-BTC
```
**Meaning:** Main made higher high, BTC made lower high (divergence)

### Normal Bullish SMT
```
30-Min Bullish SMT-ES
```
**Meaning:** Main made lower low, ES made higher low (divergence)

### Inverse Bearish SMT
```
90-Min Inverse Bearish SMT-BTC
```
**Meaning:** BTC made higher high, Main made lower high (inverse divergence)

### Inverse Bullish SMT
```
30-Min Inverse Bullish SMT-ES
```
**Meaning:** ES made lower low, Main made higher low (inverse divergence)

---

## ⏰ WHEN ALERTS FIRE

### Alert Frequency
**Setting:** `alert.freq_once_per_bar_close`

**Meaning:**
- Alert fires **once per bar** when SMT is detected
- Only triggers at **bar close** (confirmed signal)
- Won't spam you with multiple alerts per bar

### Timeline Example:
```
00:00 - 90-min cycle starts
10:00 - Cycle updating...
20:00 - Cycle updating...
...
90:00 - Cycle completes, new cycle starts
        → Checks for SMT
        → If SMT found: 🔔 ALERT FIRES!
        
90:01 - New bar starts
        → Checks for SMT
        → If SMT found: 🔔 ALERT FIRES!
```

---

## 🎛️ CUSTOMIZATION OPTIONS

### Enable/Disable Alerts
Go to indicator settings:
- **90-Minute SMT Alerts:** ON/OFF
- **30-Minute SMT Alerts:** ON/OFF

**Use Cases:**
- Trading 90-min timeframe only? Disable 30-min alerts
- Want all alerts? Keep both enabled
- Testing phase? Disable all alerts temporarily

### Why No 10-Min and 3-Min Alerts?
- You requested only 90 and 30 min
- Too many alerts can be overwhelming
- Higher timeframes = more significant signals
- Can be added easily if needed!

---

## 📊 ALERT SCENARIOS

### Scenario 1: Single Symbol, Single SMT
```
Settings:
- Symbol1: BTC (enabled)
- Symbol2: ES (disabled)

Alert:
"90-Min Bearish SMT-BTC"
```

### Scenario 2: Two Symbols, Multiple SMTs
```
Settings:
- Symbol1: BTC (enabled)
- Symbol2: ES (enabled)

Alert:
"90-Min Bearish SMT-BTC, Bullish SMT-ES"
```

### Scenario 3: Only One Cycle Enabled
```
Settings:
- 90-Min Alerts: ON
- 30-Min Alerts: OFF

Result:
- Get alerts for 90-min SMTs only
- No alerts for 30-min SMTs
```

---

## 🔔 NOTIFICATION METHODS

TradingView allows multiple notification types:

### 1. **Popup Notification**
- Shows in TradingView interface
- Visual alert bubble

### 2. **Alert Notification Center**
- Stores in TradingView alert history
- Can review later

### 3. **Email**
- Sends email to your registered address
- Good for offline monitoring

### 4. **SMS** (Premium feature)
- Text message to phone
- Requires TradingView Premium

### 5. **Webhook**
- Integrates with external systems
- For automated trading bots
- Discord, Telegram, etc.

### 6. **Sound**
- Audio notification in browser
- Customizable sound

---

## ✅ ALERT CHECKLIST

Before expecting alerts:

- [ ] Indicator added to chart
- [ ] Alert settings enabled (90-min and/or 30-min)
- [ ] TradingView alert created for the indicator
- [ ] Notification method selected (popup, email, etc.)
- [ ] Comparison symbols enabled (Symbol1 or Symbol2)
- [ ] In session time (if session filter enabled)
- [ ] Actual SMT divergence exists in data

---

## 🧪 TESTING ALERTS

### How to Test:
1. **Enable alerts** in settings
2. **Create TradingView alert** for the indicator
3. **Wait for SMT detection** (need 2+ completed cycles)
4. **Verify alert fires** when SMT appears on chart

### Quick Test:
- Use **1-min chart** for faster cycle completion
- Enable **30-min cycle** (completes in 30 bars)
- Wait 60 minutes (2 cycles = 60 bars on 1-min chart)
- If SMT exists, alert should fire

---

## 💡 TIPS & BEST PRACTICES

### 1. **Don't Over-Alert**
- Too many alerts = alert fatigue
- Start with just 90-min enabled
- Add 30-min later if needed

### 2. **Confirm Before Trading**
- Alert is a notification, not a trade signal
- Always verify the SMT on chart
- Check market conditions

### 3. **Use Multiple Notification Methods**
- Popup + Email = redundancy
- Won't miss important signals

### 4. **Review Alert History**
- Check TradingView alert center
- Learn which SMTs are valid
- Improve your strategy

### 5. **Combine with Other Indicators**
- SMT is one piece of the puzzle
- Use with support/resistance, volume, etc.
- Multi-factor confirmation

---

## 🔍 TROUBLESHOOTING

### "Not Getting Any Alerts"

**Check:**
1. Alert enabled in indicator settings?
2. TradingView alert created?
3. Cycle has 2+ completed cycles?
4. Actual SMT divergence exists?
5. In session time?
6. Symbol data loading correctly?

### "Too Many Alerts"

**Solution:**
- Disable 30-min alerts, keep only 90-min
- Adjust notification settings
- Use "alert.freq_once_per_bar_close" (already set)

### "Alert Message Unclear"

**Example:**
```
90-Min Bearish SMT-BTC
```

**Breakdown:**
- **90-Min** = Cycle timeframe
- **Bearish** = Direction (expect downward movement)
- **SMT-BTC** = Bitcoin comparison showing divergence

---

## 📈 ALERT MESSAGE EXAMPLES

### All Possible Alert Types:

**90-Minute Cycle:**
```
90-Min Bearish SMT-BTC
90-Min Bullish SMT-BTC
90-Min Inverse Bearish SMT-BTC
90-Min Inverse Bullish SMT-BTC
90-Min Bearish SMT-ES
90-Min Bullish SMT-ES
90-Min Inverse Bearish SMT-ES
90-Min Inverse Bullish SMT-ES
```

**30-Minute Cycle:**
```
30-Min Bearish SMT-BTC
30-Min Bullish SMT-BTC
30-Min Inverse Bearish SMT-BTC
30-Min Inverse Bullish SMT-BTC
30-Min Bearish SMT-ES
30-Min Bullish SMT-ES
30-Min Inverse Bearish SMT-ES
30-Min Inverse Bullish SMT-ES
```

**Multiple SMTs:**
```
90-Min Bearish SMT-BTC, Bullish SMT-ES
30-Min Inverse Bearish SMT-BTC, Inverse Bullish SMT-ES
```

---

## 🎉 SUMMARY

**Alert Features:**
✅ 90-minute cycle alerts  
✅ 30-minute cycle alerts  
✅ Enable/disable independently  
✅ Smart message formatting  
✅ Multiple SMT types included  
✅ Once per bar (no spam)  
✅ Bar close confirmation  
✅ Easy to set up  

**Your Workflow:**
1. Enable alerts in indicator settings
2. Create TradingView alert
3. Choose notification method
4. Get notified when SMT detected
5. Review chart and trade accordingly

---

## 🚀 NEXT STEPS

1. **Copy SMT_V2.pine** to TradingView
2. **Enable alerts** in settings
3. **Create TradingView alert**
4. **Test and monitor**
5. **Adjust as needed**

**You'll now be notified instantly when significant SMT patterns appear! 🔔**

---

## ❓ Need More?

Want alerts for 10-min or 3-min cycles too? Just let me know!
