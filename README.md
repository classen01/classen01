//@version=5
strategy("Forex One Minute Strategy", overlay=true)

// Define input variables
eventTime = input.time(timestamp("2023-07-31 08:49:30"), title="Event Time")
stopLossPips = input.int(100, title="Stop Loss (Pips)")
takeProfitPips = input.int(100, title="Take Profit (Pips)")
pendingOrderOffsetPips = input.int(200, title="Pending Order Offset (Pips)")
lotSize = input.float(1, title="Lot Size")

// Convert pips to price points
pipToPrice = syminfo.mintick * 10
stopLossPrice = stopLossPips * pipToPrice
takeProfitPrice = takeProfitPips * pipToPrice
pendingOrderOffset = pendingOrderOffsetPips * pipToPrice

// Calculate average market movement and draw horizontal lines
f_avg_movement() =>
    high4 = ta.highest(high, 4)
    low4 = ta.lowest(low, 4)
    avgPrice = (high4 + low4) / 2

    // Draw horizontal lines
    line.new(x1=bar_index - 3, y1=high4, x2=bar_index, y2=high4, color=color.red, width=2)
    line.new(x1=bar_index - 3, y1=low4, x2=bar_index, y2=low4, color=color.blue, width=2)
    line.new(x1=bar_index - 3, y1=avgPrice, x2=bar_index, y2=avgPrice, color=color.green, width=2)

    avgPrice

// Calculate average price on every bar
avgPrice = f_avg_movement()

// Set pending orders at event time
if (time == eventTime - 30000)  // 30 seconds before event
    buyStop = avgPrice + pendingOrderOffset
    sellStop = avgPrice - pendingOrderOffset

    // Set orders (use 'for' loop)
    for i = 0 to 4
        strategy.entry("Buy Stop " + str.tostring(i), strategy.long, stop=buyStop, qty=lotSize, comment="Buy Stop Order")
        strategy.entry("Sell Stop " + str.tostring(i), strategy.short, stop=sellStop, qty=lotSize, comment="Sell Stop Order")

// Manage trades and close non-triggered orders
if (strategy.opentrades > 0)
    // Loop through open trades (use 'for' loop)
    for i = 0 to strategy.opentrades - 1
        entryId = strategy.opentrades.entry_id(i) // Get the entry ID of the current trade
        if (strategy.opentrades > 0) // Check if any trade is open
            // Check if the trade is a long position 
            if (strategy.opentrades.entry_is_long(i))  // This is the correct method!
                strategy.exit("Take Profit/Stop Loss", from_entry=entryId, limit=strategy.opentrades.entry_price(i) + takeProfitPrice, stop=strategy.opentrades.entry_price(i) - stopLossPrice)
            else 
                strategy.exit("Take Profit/Stop Loss", from_entry=entryId, limit=strategy.opentrades.entry_price(i) - takeProfitPrice, stop=strategy.opentrades.entry_price(i) + stopLossPrice)

    // Close non-triggered orders
    if (time > eventTime + 60000)  // 1 minute after event
        strategy.cancel("Buy Stop")
        strategy.cancel("Sell Stop")
