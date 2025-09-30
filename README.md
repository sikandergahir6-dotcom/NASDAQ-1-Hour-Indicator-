# NASDAQ-1-Hour-Indicator-
Here I have made a Nasdaq 1 hour timeframe indicator to test how my original strategy would perform with some personal tweaks. 
For this indicator it includes custom SL and TP for each trend flip 
Areas where not to trade will appear blue and state consolidation 
This was just a intital test I found that alot of the areas where consildating and this due to filters being to strict and not aligning but this is the base version which you are all happy to paste into PineScript and use.

Note: If you try this bot on another index, pair etc it will spam with labells asking you chnage to nasdaq. Furthermore, as this is just a draft and not the actual bot not all the trend flips have buy and sell labels with specific TP and SL's it's just being more specific

//@version=6
indicator("SG BOT — NASDAQ 1H (Cleaned, ADX/strings fixed)", overlay=true, max_labels_count=500)

// === SETTINGS (preset for NASDAQ / 1H) ===
useHeikinAshi       = input.bool(true,  "Use Heikin Ashi candles")
atrPeriod           = input.int(10,     "ATR Period", minval=1)
sensitivity         = input.float(1.0,  "Sensitivity (ATR multiplier for trailing)", step=0.1)
volumeMultiplier    = input.float(1.0,  "Volume Multiplier (x avg)", step=0.1)
bodyRatioLimit      = input.float(0.30, "Min Body % to Flip Trend (0-1)", minval=0.0, maxval=1.0)

rsiPeriod           = input.int(14,     "RSI Period")
rsiThresholdLong    = input.float(50,   "RSI Threshold Long", step=1)
rsiThresholdShort   = input.float(50,   "RSI Threshold Short", step=1)

adxPeriod           = input.int(14,     "ADX Period")
adxThreshold        = input.float(20,   "ADX Threshold (trend strength)")

bbLength            = input.int(20,     "BB Length")
bbStdDev            = input.float(2.0,  "BB StdDev")
bbWidthThresh       = input.float(0.045,"BB Width Threshold (squeeze)")

atrTPmult           = input.float(2.0,  "Take Profit (ATR x)")
atrSLmult           = input.float(1.0,  "Stop Loss (ATR x)")

require2BarConfirm  = input.bool(true,  "Require 2+ same-color candle confirmation")
suppressDuringConsol = input.bool(true, "Suppress entry labels during consolidation")

// === SYMBOL CHECK (soft warning if not NASDAQ-like) ===
sym = syminfo.tickerid
isLikelyNasdaq = str.contains(str.lower(sym), "nasdaq") or str.contains(str.lower(sym), "nas100") or str.contains(str.lower(sym), "ndx") or str.contains(str.lower(sym), "qqq")

if not isLikelyNasdaq
    // top-left small warning
    label.new(bar_index, high, "⚠️ Check ticker: not obviously NASDAQ", xloc.bar_index, yloc.price, color.new(color.orange, 90), label.style_label_left, color.white, size.small)

// === PRICE SOURCE (Heikin Ashi option) ===
haTicker = ticker.heikinashi(syminfo.tickerid)
haOpen   = request.security(haTicker, timeframe.period, open)
haClose  = request.security(haTicker, timeframe.period, close)
haHigh   = request.security(haTicker, timeframe.period, high)
haLow    = request.security(haTicker, timeframe.period, low)

srcOpen  = useHeikinAshi ? haOpen  : open
srcClose = useHeikinAshi ? haClose : close
srcHigh  = useHeikinAshi ? haHigh  : high
srcLow   = useHeikinAshi ? haLow   : low
src      = srcClose

// === BODY SIZE & VOLUME FILTER ===
body        = math.abs(srcClose - srcOpen)
candleRange = srcHigh - srcLow
bodyRatio   = candleRange > 0 ? body / candleRange : 0.0
avgVol      = ta.sma(volume, 20)
volOkay     = volume > (avgVol * volumeMultiplier)
bodyOkay    = bodyRatio >= bodyRatioLimit
confirmFlip = volOkay or bodyOkay

// === ATR TRAILING STOP (core UT Bot logic) ===
atrValue = ta.atr(atrPeriod)
nLoss    = sensitivity * atrValue

var float trailingStop = na
trailingStop := na(trailingStop[1]) ? src - nLoss :
     src > trailingStop[1] ? math.max(trailingStop[1], src - nLoss) :
     src < trailingStop[1] ? math.min(trailingStop[1], src + nLoss) :
     trailingStop[1]

// === RAW TREND SIGNALS ===
bullish = src > trailingStop
bearish = src < trailingStop

// === 2+ SAME-COLOR CANDLE CONFIRMATION ===
bullConfirm = bullish and bullish[1]
bearConfirm = bearish and bearish[1]

trendUp = bullConfirm and confirmFlip
trendDn = bearConfirm and confirmFlip

// === TREND STATE TRACKING ===
var string trend = "none"
trendPrev = trend

if (trendUp)
    trend := "bull"
else if (trendDn)
    trend := "bear"
else
    trend := trend

// === CONSOLIDATION / SQUEEZE DETECTION ===
// Bollinger Bands width
bbMid = ta.sma(src, bbLength)
bbStd = ta.stdev(src, bbLength)
bbUpper = bbMid + bbStdDev * bbStd
bbLower = bbMid - bbStdDev * bbStd
bbWidth = (bbUpper - bbLower) / math.max(1e-10, bbMid)  // relative width

// ADX via DMI (returns +DI, -DI, ADX) — ta.dmi(len, smoothing)
[diplus, diminus, adx] = ta.dmi(adxPeriod, adxPeriod)

// ATR relative (normalize by price)
atrRel = atrValue / math.max(1e-10, bbMid)

// Consolidation criteria: narrow BB width AND low ADX AND low ATR relative
isSqueeze = bbWidth < bbWidthThresh
isLowAdx  = adx < adxThreshold
isLowAtr  = atrRel < 0.0045  // tuned for NASDAQ hourly; adjust if necessary

consolidation = isSqueeze and isLowAdx and isLowAtr

// Paint consolidation candles blue and label them
barcolor(consolidation ? color.new(color.blue, 10) : na)
if consolidation
    label.new(bar_index, low, "Consolidation", xloc.bar_index, yloc.belowbar, color.new(color.blue, 80), label.style_label_down, color.white, size.tiny)

// If consolidation begins, treat it as a pause (we suppress flips while it persists)
if consolidation
    trend := trend

// === MOMENTUM & STRENGTH FILTERS ===
rsi = ta.rsi(src, rsiPeriod)
bullMomentum = rsi > rsiThresholdLong
bearMomentum = rsi < rsiThresholdShort
strongTrend = adx >= adxThreshold

// === FINAL ENTRY FLIP CONDITIONS (suppress during consolidation if requested) ===
trendFlippedBull = (trendPrev != "bull") and (trend == "bull")
trendFlippedBear = (trendPrev != "bear") and (trend == "bear")

allowEntries = not consolidation or (not suppressDuringConsol)

entryLong = trendFlippedBull and bullMomentum and strongTrend and allowEntries
entryShort = trendFlippedBear and bearMomentum and strongTrend and allowEntries

// === ATR-BASED TAKE PROFIT & STOP LOSS SUGGESTIONS ===
takeProfitLong  = src + atrTPmult * atrValue
stopLossLong    = src - atrSLmult * atrValue
takeProfitShort = src - atrTPmult * atrValue
stopLossShort   = src + atrSLmult * atrValue

// === PLOTTING SIGNALS & TP/SL ===
plot(trailingStop, title="Trailing Stop", color=color.orange, linewidth=2)

// Plot TP/SL lines only when entries occur on that bar (so chart stays clean)
var line tpLine = na
var line slLine = na

if entryLong
    if not na(tpLine)
        line.delete(tpLine)
    if not na(slLine)
        line.delete(slLine)
    tpLine := line.new(bar_index, takeProfitLong, bar_index + 24, takeProfitLong, xloc.bar_index, extend.right, color.new(color.green, 60), line.style_dashed, 2)
    slLine := line.new(bar_index, stopLossLong,   bar_index + 24, stopLossLong,   xloc.bar_index, extend.right, color.new(color.red,   60), line.style_dashed, 2)
    label.new(bar_index, srcLow, "BUY\nTP:" + str.tostring((takeProfitLong - src) / syminfo.mintick, format.mintick) + "\nSL:" + str.tostring((src - stopLossLong) / syminfo.mintick, format.mintick),
      xloc.bar_index, yloc.belowbar, color.new(color.green, 0), label.style_label_up, color.white, size.small)

if entryShort
    if not na(tpLine)
        line.delete(tpLine)
    if not na(slLine)
        line.delete(slLine)
    tpLine := line.new(bar_index, takeProfitShort, bar_index + 24, takeProfitShort, xloc.bar_index, extend.right, color.new(color.green, 60), line.style_dashed, 2)
    slLine := line.new(bar_index, stopLossShort,  bar_index + 24, stopLossShort,  xloc.bar_index, extend.right, color.new(color.red,   60), line.style_dashed, 2)
    label.new(bar_index, srcHigh, "SELL\nTP:" + str.tostring((src - takeProfitShort) / syminfo.mintick, format.mintick) + "\nSL:" + str.tostring((stopLossShort - src) / syminfo.mintick, format.mintick),
      xloc.bar_index, yloc.abovebar, color.new(color.red, 0), label.style_label_down, color.white, size.small)

// Entry markers (only when allowed)
plotshape(entryLong, title="Buy Signal", location=location.belowbar, style=shape.triangleup, size=size.small, color=color.green, text="BUY")
plotshape(entryShort, title="Sell Signal", location=location.abovebar, style=shape.triangledown, size=size.small, color=color.red, text="SELL")

// Color candles for active trend (green/red) unless consolidation paints blue
barcolor(not consolidation and trend == "bull" ? color.new(color.green, 15) : na)
barcolor(not consolidation and trend == "bear" ? color.new(color.red, 15) : na)

// === ALERTS ===
alertcondition(entryLong,  title="UTB BUY - NASDAQ 1H", message="UT Bot Gold - BUY (NASDAQ 1H)")
alertcondition(entryShort, title="UTB SELL - NASDAQ 1H", message="UT Bot Gold - SELL (NASDAQ 1H)")
alertcondition(consolidation, title="UTB Consolidation", message="UT Bot Gold - CONSOLIDATION detected on NASDAQ 1H")

// === INFO (compact top-right table) ===
var table info = table.new(position.top_right, 1, 4)
if barstate.islast
    table.cell(info, 0, 0, "Trend: " + trend)
    table.cell(info, 0, 1, "Consol: " + str.tostring(consolidation))
    table.cell(info, 0, 2, "ADX: " + str.tostring(adx, format.mintick))
    table.cell(info, 0, 3, "RSI: " + str.tostring(rsi, format.mintick))
