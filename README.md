//@version=5
indicator(title="NADARA SABRI", shorttitle="NADSAB", format=format.price, precision=2, timeframe="", timeframe_gaps=true)

ma(source, length, type) =>
    switch type
        "SMA" => ta.sma(source, length)
        "Bollinger Bands" => ta.sma(source, length)
        "EMA" => ta.ema(source, length)
        "SMMA (RMA)" => ta.rma(source, length)
        "WMA" => ta.wma(source, length)
        "VWMA" => ta.vwma(source, length)

rsiLengthInput = input.int(14, minval=1, title="RSI Length", group="RSI Settings")
rsiSourceInput = input.source(close, "Source", group="RSI Settings")
maTypeInput = input.string("SMA", title="MA Type", options=["SMA", "Bollinger Bands", "EMA", "SMMA (RMA)", "WMA", "VWMA"], group="MA Settings", display = display.data_window)
maLengthInput = input.int(14, title="MA Length", group="MA Settings", display = display.data_window)
bbMultInput = input.float(2.0, minval=0.001, maxval=50, title="BB StdDev", group="MA Settings", display = display.data_window)
showDivergence = input.bool(false, title="Show Divergence", group="RSI Settings", display = display.data_window)

up = ta.rma(math.max(ta.change(rsiSourceInput), 0), rsiLengthInput)
down = ta.rma(-math.min(ta.change(rsiSourceInput), 0), rsiLengthInput)
rsi = down == 0 ? 100 : up == 0 ? 0 : 100 - (100 / (1 + up / down))
rsiMA = ma(rsi, maLengthInput, maTypeInput)
isBB = maTypeInput == "Bollinger Bands"

rsiPlot = plot(rsi, "RSI", color=#7E57C2)
plot(rsiMA, "RSI-based MA", color=color.yellow)


//NADARA

h = input.float(8.,'Bandwidth', minval = 0)
mult = input.float(3., minval = 0)
src = input(close, 'Source')

repaint = input(true, 'Repainting Smoothing', tooltip = 'Repainting is an effect where the indicators historical output is subject to change over time. Disabling repainting will cause the indicator to output the endpoints of the calculations')

//Style
upCss = input.color(color.teal, 'Colors', inline = 'inline1', group = 'Style')
dnCss = input.color(color.red, '', inline = 'inline1', group = 'Style')

//-----------------------------------------------------------------------------}
//Functions
//-----------------------------------------------------------------------------{
//Gaussian window
gauss(x, h) => math.exp(-(math.pow(x, 2)/(h * h * 2)))

//-----------------------------------------------------------------------------}
//Append lines
//-----------------------------------------------------------------------------{
n = bar_index

var ln = array.new_line(0) 

if barstate.isfirst and repaint
    for i = 0 to 499
        array.push(ln,line.new(na,na,na,na))

//-----------------------------------------------------------------------------}
//End point method
//-----------------------------------------------------------------------------{
var coefs = array.new_float(0)
var den = 0.

if barstate.isfirst and not repaint
    for i = 0 to 499
        w = gauss(i, h)
        coefs.push(w)

    den := coefs.sum()

out = 0.
if not repaint
    for i = 0 to 499
        out += src[i] * coefs.get(i)
out /= den
mae = ta.sma(math.abs(src - out), 499) * mult

upper = out + mae
lower = out - mae
 
//-----------------------------------------------------------------------------}
//Compute and display NWE
//-----------------------------------------------------------------------------{
float y2 = na
float y1 = na

nwe = array.new<float>(0)
if barstate.islast and repaint
    sae = 0.
    //Compute and set NWE point 
    for i = 0 to math.min(499,n - 1)
        sum = 0.
        sumw = 0.
        //Compute weighted mean 
        for j = 0 to math.min(499,n - 1)
            w = gauss(i - j, h)
            sum += src[j] * w
            sumw += w

        y2 := sum / sumw
        sae += math.abs(src[i] - y2)
        nwe.push(y2)
    
    sae := sae / math.min(100,n - 1) * mult
    for i = 0 to math.min(100,n - 1)
        if i%2
            line.new(n-i+1, y1 + sae, n-i, nwe.get(i) + sae, color = upCss)
            line.new(n-i+1, y1 - sae, n-i, nwe.get(i) - sae, color = dnCss)
        
        if src[i] > nwe.get(i) + sae and src[i+1] < nwe.get(i) + sae
            label.new(n-i, src[i], '▼', color = color(na), style = label.style_label_down, textcolor = dnCss, textalign = text.align_center)
        if src[i] < nwe.get(i) - sae and src[i+1] > nwe.get(i) - sae
            label.new(n-i, src[i], '▲', color = color(na), style = label.style_label_up, textcolor = upCss, textalign = text.align_center)
        
        y1 := nwe.get(i)

//-----------------------------------------------------------------------------}
//Dashboard
//-----------------------------------------------------------------------------{
var tb = table.new(position.top_right, 1, 1
  , bgcolor = #1e222d
  , border_color = #373a46
  , border_width = 1
  , frame_color = #373a46
  , frame_width = 1)

if repaint
    tb.cell(0, 0, 'Repainting Mode Enabled', text_color = color.white, text_size = size.small)

//-----------------------------------------------------------------------------}
//Plot
//-----------------------------------------------------------------------------}
plot(repaint ? na : out + mae, 'Upper', upCss)
plot(repaint ? na : out - mae, 'Lower', dnCss)

//Crossing Arrows
plotshape(ta.crossunder(close, out - mae) ? low : na, "Crossunder", shape.labelup, location.absolute, color(na), 0 , text = '▲', textcolor = upCss, size = size.tiny)
plotshape(ta.crossover(close, out + mae) ? high : na, "Crossover", shape.labeldown, location.absolute, color(na), 0 , text = '▼', textcolor = dnCss, size = size.tiny)

//-----------------------------------------------------------------------------}
