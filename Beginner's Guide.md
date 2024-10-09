# 开始你的第一个Backtrader策略

## Backtrader策略基本组件

一个最简单的Backtrader策略程序由三部分组成
- 策略引擎（Cerebro）
- 数据源（Data Feed）
- 策略（Strategy）

一个最简单的Backtrader策略程序如下:

```python
import backtrader as bt
from ffquant.feeds.MyFeed import MyFeed
import pytz

class SimpleStrategy(bt.Strategy):
    def __init__(self):
        super(SimpleStrategy, self).__init__()

    def next(self):
        dt = self.data.datetime.datetime(0).replace(tzinfo=pytz.utc).astimezone().strftime("%Y-%m-%d %H:%M:%S")
        print(f"dt: {dt}, close[0]: {self.data.close[0]}")

if __name__ == '__main__':
    cerebro = bt.Cerebro()
    my_data = MyFeed(symbol="CAPITALCOM:HK50", start_time="2024-10-07 10:00:00", end_time="2024-10-07 23:59:00")
    cerebro.adddata(my_data)
    cerebro.addstrategy(SimpleStrategy)
    cerebro.run()
```

针对每一根K线，SimpleStrategy的next方法都会被调用一次，在策略的next方法中可以访问市场行情、行情指标、账户余额、仓位等，我们在next方法中实现交易策略的逻辑。

## 基本概念 - 线（Lines）

数据源（Data Feeds）、指标（Indicators）和策略（Strategies）都有线（lines）。线Lines是由一系列连续的点组成的，当这些点连接在一起时就形成了线。比如市场行情的数据源（Data Feed）通常有以下的点的组合：

- 日期时间（DateTime）
- 开盘价（Open）
- 最高价（High）
- 最低价（Low）
- 收盘价（Close）
- 成交量（Volume）
- 成交金额（TurnOver）

例如，随时间变化的“开盘价（Open）”序列就是一条线。因此，一个市场行情的数据源通常有 7 条线。

## 基本概念 - 访问线（Lines）的值的索引（Index）说明

在访问一条线中的值时，当前值使用索引 0 进行访问，上一个值使用索引-1访问，上上一个值使用索引-2访问，上上上一个值使用索引-2访问，依次类推。

```python
self.data.close[0]  # 访问当前K线的close价格
```

```python
self.data.close[-1] # 访问上一根K线的close价格
```

可以把它现象为一个表示时间的x轴，原点0表示当前，原点左边（负值）表示历史，原点右边表示未来。

## 基本概念 - 访问数据源（Data Feeds）的语法糖

在一个策略Strategy中，一般有1个或者多个市场行情数据源，self.datas表示所有的市场行情数据源的列表，self.datas[0]表示第一个市场行情数据源，self.datas[1]表示第二个市场行情数据源，在策略开发中经常会用到以下的语法糖：

```python
self.data # 表示 self.datas[0]
```

```python
self.dataX # 表示 self.datas[X]
```

## 基本概念 - 访问策略（Strategy）的属性

- K线的数量
```python
class SimpleStrategy(bt.Strategy):
    def next(self):
        print(f"K candle count: {len(self)}")
```

- 市场行情

datas成员包含了所有加入到引擎中的市场行情数据源，通过datas[i]访问具体的数据源

- 券商Broker

通过broker成员访问，self.broker.getcash()返回余额，self.broker.getvalue()返回账户价值

- 仓位

self.position.size返回仓位信息，大于0表示多头仓位，小于0表示空头仓位

- 指标（Indicators）

在策略的构造函数中声明指标a，在next方法中通过self.a访问该指标，self.a[0]指标的当前值，self.a[-1]指标的上一个值

## 给策略加上指标（Indicators）

我们给上面的策略加上一个Trend指标，如下：
```python
import backtrader as bt
from ffquant.feeds.MyFeed import MyFeed
from ffquant.indicators.Trend import Trend
import pytz

class SimpleStrategy(bt.Strategy):
    def __init__(self):
        super(SimpleStrategy, self).__init__()
        self.trend = Trend(symbol="CAPITALCOM:HK50")

    def next(self):
        dt = self.data.datetime.datetime(0).replace(tzinfo=pytz.utc).astimezone().strftime("%Y-%m-%d %H:%M:%S")
        print(f"dt: {dt}, close[0]: {self.data.close[0]}, trend[0]: {self.trend[0]}")


if __name__ == '__main__':
    cerebro = bt.Cerebro()
    my_data = MyFeed(symbol="CAPITALCOM:HK50", start_time="2024-10-07 10:00:00", end_time="2024-10-07 23:59:00")
    cerebro.adddata(my_data)
    cerebro.addstrategy(SimpleStrategy)
    cerebro.run()

```

Trend指标值的说明，请查看本文档结尾的Reference部分

## 如何下单？

我们给策略加上下单的逻辑：

- 当trend为1（BULLISH）且当前close价格大于上一个close价格时，买入
- 当trend为-1（BEARISH）且当前close价格小于上一个close价格时，卖出

```python
import backtrader as bt
from ffquant.feeds.MyFeed import MyFeed
from ffquant.indicators.Trend import Trend
import pytz

class SimpleStrategy(bt.Strategy):
    def __init__(self):
        super(SimpleStrategy, self).__init__()
        self.trend = Trend(symbol="CAPITALCOM:HK50")

    def next(self):
        dt = self.data.datetime.datetime(0).replace(tzinfo=pytz.utc).astimezone().strftime("%Y-%m-%d %H:%M:%S")
        print(f"dt: {dt}, close[0]: {self.data.close[0]}, trend[0]: {self.trend[0]}")
        
        if self.trend[0] == 1 and self.data.close[0] > self.data.close[-1]:
            self.buy()
        elif self.trend[0] == -1 and self.data.close[0] < self.data.close[-1]:
            self.sell()


if __name__ == '__main__':
    cerebro = bt.Cerebro()
    my_data = MyFeed(symbol="CAPITALCOM:HK50", start_time="2024-10-07 10:00:00", end_time="2024-10-07 23:59:00")
    cerebro.adddata(my_data)
    cerebro.addstrategy(SimpleStrategy)
    cerebro.run()
```

buy()和sell()的参数说明如下:
|参数|类型|默认值|说明|
|---|---|---|---|
|exectype|int|bt.Order.Market|订单类型 bt.Order.Market或者bt.Order.Limit|
|price|float|None|订单价格，只在exectype为bt.Order.Limit有效|
|size|int|None|正整数，下单数量|

## 策略性能如何？

我们可以使用run_and_show_performance函数来图形化展示策略的性能，如下：
```python
import backtrader as bt
from ffquant.feeds.MyFeed import MyFeed
from ffquant.indicators.Trend import Trend
import pytz
from ffquant.plot.perf import run_and_show_performance

class SimpleStrategy(bt.Strategy):
    def __init__(self):
        super(SimpleStrategy, self).__init__()
        self.trend = Trend(symbol="CAPITALCOM:HK50")

    def next(self):
        dt = self.data.datetime.datetime(0).replace(tzinfo=pytz.utc).astimezone().strftime("%Y-%m-%d %H:%M:%S")
        print(f"dt: {dt}, close[0]: {self.data.close[0]}, trend[0]: {self.trend[0]}")
        
        if self.trend[0] == 1 and self.data.close[0] > self.data.close[-1]:
            self.buy()
        elif self.trend[0] == -1 and self.data.close[0] < self.data.close[-1]:
            self.sell()


if __name__ == '__main__':
    cerebro = bt.Cerebro()
    my_data = MyFeed(symbol="CAPITALCOM:HK50", start_time="2024-10-07 10:00:00", end_time="2024-10-07 23:59:00")
    cerebro.adddata(my_data)
    cerebro.addstrategy(SimpleStrategy)
    # cerebro.run() # 不要调用cerebro.run() 策略执行交给run_and_show_performance函数负责
    run_and_show_performance(cerebro)
```

恭喜你，你完成了你的第一个Backtrader策略。你可以继续查阅本文档的其他部分优化策略。

# 重要概念说明

- 指标（Indicators）的最小周期
- 概念2
- 概念3

# FAQ

- 策略回测完成后，如何改为实时策略并部署？
- 执行策略回测时，没有弹出策略性能显示tab？

# ffquant库Reference

## 交易品种symbol

- CAPITALCOM:HK50


## MyFeed

路径：ffquant.feeds.MyFeed

MyFeed类是对回测时的市场行情数据源的抽象，此数据源仅用于回测目的

**引入方式**

```python
from ffquant.feeds.MyFeed import MyFeed
```


**构造函数**

|参数名|类型|默认值|说明|
|-------|-------|-------|-------|
|start_time|string|无|用于回测的市场行情数据源的开始时间 格式为yyyy-mm-dd HH:MM:SS|
|end_time|string|无|用于回测的市场行情数据源的结束时间 格式为yyyy-mm-dd HH:MM:SS|
|symbol|string|无|交易品种 比如:CAPITALCOM:HK50|
|prefetch_size|string|60|用于性能优化，预取数据的长度|
|debug|boolean|False|是否打印debug日志|

**内部Lines**

|Line名称|类型|说明|
|------|------|------|
|datetime|float|距离1970年1月1日的天数，转换为字符串 self.data.datetime.datetime(0).replace(tzinfo=pytz.utc).astimezone().strftime("%Y-%m-%d %H:%M:%S")|
|open|float|open价格，在策略或者指标中的访问方式：self.data.open[0]|
|high|float|high价格，在策略或者指标中的访问方式：self.data.high[0]|
|low|float|low价格，在策略或者指标中的访问方式：self.data.low[0]|
|close|float|close价格，在策略或者指标中的访问方式：self.data.close[0]|
|volume|float|volume交易量，在策略或者指标中的访问方式：self.data.volume[0]|
|turnover|float|turnover交易额，在策略或者指标中的访问方式：self.data.turnover[0]|

**示例代码**

```python
cerebro = bt.Cerebro()
my_data = MyFeed(symbol="CAPITALCOM:HK50", start_time="2024-10-07 00:00:00", end_time="2024-10-07 12:00:00")
cerebro.adddata(my_data)
cerebro.run()
```


## Trend

路径: ffquant.indicators.Trend

Trend类是对市场趋势的预测指标的抽象，预测方向为以下3个之一：

- 1 表示BULLISH 预测为牛市
- 0 表示无方向判断
- -1 表示BEARISH 预测为熊市

**引入方式**

```python
from ffquant.indicators.Trend import Trend
```


**构造函数**

|参数名|类型|默认值|说明|
|-------|-------|-------|-------|
|symbol|string|无|交易品种 比如:CAPITALCOM:HK50|
|max_retries|int|30|获取当前最新的信号时，重试的最大次数|
|prefetch_size|int|60|用于性能优化，预取数据的长度|
|debug|boolean|False|是否打印debug日志|

**内部Lines**

|Line名称|类型|说明|
|------|------|------|
|trend|float|1表示BULLISH，0表示无方向判断，-1表示BEARISH，float('-inf')表示无法获取该指标。在策略或者指标中的访问方式：self.trend[0]|

**最小周期**

该指标的最小周期为1

**示例代码**

```python
class SimpleStrategy(bt.Strategy):
    def __init__(self):
        super(SimpleStrategy, self).__init__()
        self.trend = Trend(symbol="CAPITALCOM:HK50")

    def next(self):
        dt = self.data.datetime.datetime(0).replace(tzinfo=pytz.utc).astimezone().strftime("%Y-%m-%d %H:%M:%S")
        print(f"dt: {dt}, trend[0]: {self.trend[0]}")

if __name__ == '__main__':
    cerebro = bt.Cerebro()
    my_data = MyFeed(symbol="CAPITALCOM:HK50", start_time="2024-10-07 10:00:00", end_time="2024-10-07 23:59:00")
    cerebro.adddata(my_data)
    cerebro.addstrategy(SimpleStrategy)
    cerebro.run()
```


## TurningPoint
路径: ffquant.indicators.TurningPoint

TurningPoint类是对市场趋势的预测指标的抽象，预测方向为以下3个之一：

- 1 表示turn up 预测为方向向上
- 0 表示无反向判断
- -1 表示turn down 预测为方向向下

**引入方式**

```python
from ffquant.indicators.TurningPoint import TurningPoint
```

**构造函数**

|参数名|类型|默认值|说明|
|-------|-------|-------|-------|
|symbol|string|无|交易品种 比如:CAPITALCOM:HK50|
|max_retries|int|30|获取当前最新的信号时，重试的最大次数|
|prefetch_size|int|60|用于性能优化，预取数据的长度|
|debug|boolean|False|是否打印debug日志|

**内部Lines**

|Line名称|类型|说明|
|------|------|------|
|tp|float|1表示turn up，0表示无方向判断，-1表示turn down，float('-inf')表示无法获取该指标。在策略或者指标中的访问方式：self.tp[0]|

**最小周期**

该指标的最小周期为1

**示例代码**

```python
class SimpleStrategy(bt.Strategy):
    def __init__(self):
        super(SimpleStrategy, self).__init__()
        self.tp = TurningPoint(symbol="CAPITALCOM:HK50")

    def next(self):
        dt = self.data.datetime.datetime(0).replace(tzinfo=pytz.utc).astimezone().strftime("%Y-%m-%d %H:%M:%S")
        print(f"dt: {dt}, tp[0]: {self.tp[0]}")

if __name__ == '__main__':
    cerebro = bt.Cerebro()
    my_data = MyFeed(symbol="CAPITALCOM:HK50", start_time="2024-10-07 10:00:00", end_time="2024-10-07 23:59:00")
    cerebro.adddata(my_data)
    cerebro.addstrategy(SimpleStrategy)
    cerebro.run()
```


## 显示回测性能

路径：ffquant.plot.perf.run_and_show_performance

run_and_show_performance函数用于显示回测策略的性能表现，如总收益、仓位变化、买卖点等。

**引入方式**

```python
from ffquant.plot.perf import run_and_show_performance
```

**参数说明**
|参数名|类型|默认值|说明|
|-------|-------|-------|-------|
|cerebro|bt.Cerebro|无|交易引擎Cerebro实例|
|riskfree_rate|float|0.01|无风险利率|
|use_local_dash_url|boolean|False|开发团队内部使用，在用IP地址访问策略开发平台时，传入此参数True显示策略性能界面|
|debug|boolean|False|是否打印debug日志|

**示例代码**

```python
class SimpleStrategy(bt.Strategy):
    def __init__(self):
        super(SimpleStrategy, self).__init__()

    def next(self):
        dt = self.data.datetime.datetime(0).replace(tzinfo=pytz.utc).astimezone().strftime("%Y-%m-%d %H:%M:%S")
        print(f"dt: {dt}, close[0]: {self.data.close[0]}")

if __name__ == '__main__':
    cerebro = bt.Cerebro()
    my_data = MyFeed(symbol="CAPITALCOM:HK50", start_time="2024-10-07 10:00:00", end_time="2024-10-07 23:59:00")
    cerebro.adddata(my_data)
    cerebro.addstrategy(SimpleStrategy)
    run_and_show_performance(cerebro)
```