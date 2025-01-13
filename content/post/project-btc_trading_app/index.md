---
title: "Bitcoin Trading App"
date: 2019-10-04T13:42:55+02:00
author: "Johannes Herrmann"
authorAvatar: "img/project_btc_trading_app/portrait.jpg"
tags: ["bitcoin", "crypto", "trading", "trading algorithms"]
categories: ["project"]
image: "img/project_btc_trading_app/btc-background.jpg"
draft: false
---
By anticipating market movements ahead of time it is possible to generate
profits. This application implements several trading strategies which aim
to do that automatically.
<!--more-->

# Content

1. <a href="#introduction">Introduction</a>
1. <a href="#chart">The Chart</a>
1. <a href="#traders">The Traders</a>
  1. <a href="#gen">General Trader Settings</a>
  1. <a href="#bnh">Buy and Hold</a>
  1. <a href="#ma">Moving Averages</a>
  1. <a href="#macd">MACD</a>
  1. <a href="#stochs">Stochastics</a>
  1. <a href="#rsi">Relative Strength Index</a>
  1. <a href="#stdev">Standard Deviation</a>

# <a id="#introduction"></a> Introduction

Since markets exist people have been trying to anticipate changes in the price
of an asset ahead of time, in order to generate profits. In theory it is simple:
buy when the price is low and sell when the price is high. But whether the price
is actually high or low depends on future prices, not past ones. A price is
considered high, if it is going to fall in the future, vice versa it is
considered low, if it is going to rise.
One of the tools
for predicting market movements is Technical Analysis (TA).  
This application implements some of the most promising techniques from TA
for trading crypto currencies (e.g. Bitcoin). These techniques are then used
by a "trader" to predict future price movement and generate Buy- or Sell-signals.  
This application also provides options to adjust parameters of techniques
and to combine them to create a trader that produces more accurate signals.

This blog post will explain how to read the candlestick chart which displays
the price and trader actions and how the implemented TA techniques work.


# <a id="#chart"></a> The Chart

<img src="/../../img/project_btc_trading_app/candle.jpg" width=80%>

A common way to display an assets price over time is a candlestick chart. Each
candle represents price action over a certain period of time, also called
"granularity". Often a value of one week, day or hour is chosen. A candle
consists of a thick body and two thin "wicks". The body shows the opening and
closing price in the selected time period. If the candle is red, the price fell
from the opening to the closing value and if the candle is green, the
price rose from open to close. The upper part of the thin wick represents the
maximum price over the period and the lower part represents the minimum.

The solid lines in the chart show the values of the trader accounts and can be
toggled on or off by (un-)checking the box "Show trader account value". The
green line represents trader one's account and the blue line trader two's
account.

Under the tab 'General Settings' you can choose the asset, the granularity (size
of the time interval covered by one candle) and whether to use historic or live
prices.
The historic prices are chosen by selecting a date range. The live prices are
selected by entering the number of candles (units of time) that should be
displayed.

# <a id="#traders"></a> The Trading Strategies

## <a id="#gen"></a> General Trader Settings

All traders regardless of their used strategy have several general settings:

### Stop Loss:

This percentage defines how much value the asset can loose before the trader
will sell it. This is a simple form of risk management.
For example: If the trader buys at 1000 USD per unit and stop loss is set to
10%, it will sell everything it bought when the price falls below 900 USD per
unit.


### Take Profit:

This percentage defines how much the price can rise before the trader secures
its profits. This is a sort of risk management, too, like stop loss.
Example: If the trader buys at 1000 USD per unit and take profit is set to 10%,
it will sell 100 USD worth of asset when the price hits 1100 USD per unit.  
Disclaimer: This mechanism is based on the premise that the USD is more stable
than the asset price, which is true for most crypto currencies.

### Fees and Budget:

Fees are specified in percent-tenths (0.1%). In practice, they can range from 0.5% to about 2%, depending on
the used crypto exchange.
The trader budget can be specified by a starting value, which the trader will
recieve right away. And a regularly added amount can be set, along with
a frequency defining how often the trader will recieve it.

## <a id="#bnh"></a> Buy and Hold

The "Buy and Hold" strategy is the most simple one. Whenever there are USD in
the account, use them to buy as much of the asset as possible and hold it.
With this strategy, bought assets are never sold.

## <a id="#ma"></a> Moving Average Crossover

This strategy is based on two moving averages. One takes a large sample size and
is called "slow", the other takes a smaller sample size and is called "fast".
In theory, the slow moving average is an estimation of the long-term price and
the fast moving average estimates the short-term price.  

A Buy-signal is triggered, when the fast moving average crosses over the slow
moving one. Vice versa a Sell-signal is triggered, if they cross the other way
around.

In the app you can choose between two "flavours" of moving averages: simple or
exponential. The exponential moving average puts more
weight on recent data points while the simple one assigns the same weight to all
of them.

## <a id="#macd"></a> MACD

This strategy is based on the MACD: moving average convergence-divergence. It is
calculated by taking the difference between a fast and a slow exponential moving
average (EMA, see <a href="#ma">Moving Averages</a>). Then, the average of this
difference is computed, which gives the MACD-signal.  
If this MACD-signal crosses above zero, a Buy-signal is triggered, if it crosses
below, a Sell-signal.

## <a id="#stochs"></a> Stochastic Oscillator

This oscillator's value is computed by first looking up the highest and lowest prices
in a defined time period. Then the difference between the current and lowest
price, as well as the difference between highest and lowest are calculated. The
ratio between those is then compared to a moving average of itself.  
If the ratio
crosses above the average, a Buy-signal is triggered and if it crosses below the
average, a Sell-signal is triggered.

## <a id="#rsi"></a> Relative Strength Index

To calculate the relative strength, upwards and downwards price movements are summed up
respectively and averaged over a given period.  
The relative strength index is the ratio between the average of upwards movements and
the sum of both averages.

There are many ways to interpret the RSI and there does not seem to be one
agreed upon interpretation.
For this app, the crossover of the RSI with an average of itself is considered.
If the RSI crosses above the average a Buy-signal is triggered, if it crosses
below the average, a Sell-signal is triggered.

## <a id="#stdev"></a> Standard Deviation

This strategy simply calculates the standard deviation of the price and a moving
average of the standard deviation. Contrary to the other strategies, this one
does not produce Buy- or Sell-signals. Instead, it signals whether a big price move
is imminent and thus if the trader should act or not.
If the standard deviation crosses above its average, a relatively big move is
expected and the trader can act. If it crosses below its average, no move is
expected and the trader can not act.
