---
title: Freqtrade Weekend Lab：从规则策略到机器学习交易系统
draft: false
tags:
  - freqtrade
  - quantitative
  - algorithmic-trading
  - machine-learning
created: 2026-07-26 17:24
modified: 2026-07-26 18:44
---

本文记录一次基于 Freqtrade 的周末实验，重点是拆开交易系统中几个相互影响的环节，并观察它们如何共同影响结果。

先给结论：本次实验没有发现有效策略，但验证了入场、退出、止损、持仓占用、费用和执行环境之间的路径依赖。Freqtrade 降低了交易基础设施的实现成本，却没有降低研究者对数据、实验设计和风险边界的责任。

全文按照以下顺序展开：框架抽象 → 实验设计 → 规则策略结果 → 回测与 Dry-run 验证 → 机器学习扩展。

目标是理解以下问题：

1. Freqtrade 如何表达和执行交易策略；
2. 入场、退出、止损和持仓占用如何共同决定结果；
3. 回测、Dry-run 和真实交易分别验证什么；
4. 规则策略如何扩展到机器学习策略。

实验范围：

```text
Market:       Spot
Direction:    Long only
Pair:         BTC/USDT
Timeframe:    1h
Development:  2025-07-01 — 2026-04-01
Challenge:    2026-04-01 — 2026-07-01
```

共完成 12 个实验、10 个策略实现和一个持续运行的 Dry-run 实例。

---

## 1. Freqtrade 的核心抽象

Freqtrade 的策略通常实现为继承 `IStrategy` 的 Python 类。

主要接口包括：

```python
class MyStrategy(IStrategy):
    timeframe = "1h"
    stoploss = -0.08
    minimal_roi = {}

    def populate_indicators(self, dataframe, metadata):
        ...

    def populate_entry_trend(self, dataframe, metadata):
        ...

    def populate_exit_trend(self, dataframe, metadata):
        ...
```

其数据流可以概括为：

```text
OHLCV DataFrame
    ↓
Indicator / Feature Calculation
    ↓
enter_long / exit_long
    ↓
Position and Order Constraints
    ↓
Trade Lifecycle
```

策略类主要负责：

- 指标和特征；
- 入场条件；
- 退出条件；
- 策略级参数；
- 可选的仓位、止损和订单回调。

配置文件负责：

- 交易所；
- Pairlist；
- 资金和 stake；
- 最大同时持仓；
- 订单类型；
- Dry-run / Live；
- API 和 UI。

Freqtrade Engine 负责：

- 数据加载；
- 回测循环；
- 订单状态；
- Trade 持久化；
- 钱包状态；
- 实时调度。

Freqtrade 减少了交易基础设施的实现成本。数据完整性、前视偏差、费用假设、风控参数和策略有效性仍需由研究者验证。

---

## 2. 策略边界：时序与横截面

Freqtrade 可以同时处理多个交易对，但默认仍以单个交易对为计算单元：

```text
BTC/USDT DataFrame ─┐
ETH/USDT DataFrame ─┼→ Same Strategy → Independent Signals
SOL/USDT DataFrame ─┘
```

它自然表达的问题是：

```text
signal(pair, t) = f(history of pair before t)
```

例如：

- BTC 当前是否突破过去 20 小时高点；
- ETH 当前 RSI 是否进入超卖区；
- SOL 当前趋势是否反转。

这属于时序策略。

典型横截面策略则是：

```text
At time t:
    calculate factors for 200 assets
    normalize cross-sectionally
    rank assets
    select top and bottom groups
    assign target weights
    rebalance portfolio
```

其自然输出通常是目标权重：

```text
BTC:  +12%
ETH:   +8%
SOL:    0%
XRP:   -5%
Cash:  85%
```

Freqtrade 可以通过动态 Pairlist、informative pairs 或外部信号间接支持部分横截面逻辑，但并不原生提供：

- 横截面因子矩阵；
- 横截面标准化；
- 组合权重优化；
- 风险中性约束；
- 组合级再平衡。

在这组实验中，我把 Freqtrade 定位为：

> Freqtrade 是以单交易对时序信号为核心抽象的中低频加密交易框架。

---

## 3. 实验设计

本次实验遵循以下约束：

- 每次只改变一个主要变量；
- 运行前记录预期；
- 不使用 Hyperopt；
- 不根据 Challenge 结果继续调参；
- 保留所有失败实验；
- 区分 observation、interpretation 和 conclusion；
- Dry-run 只验证系统路径，不评价盈利能力。

AI Agent 用于：

- 生成受控策略变体；
- 增加注释；
- 检查 diff；
- 执行指定回测；
- 分析日志；
- 汇总指标。

AI Agent 不被允许：

- 自动搜索参数；
- 根据收益继续迭代；
- 删除失败实验；
- 自动得出策略有效性结论。

---

## 4. EMA 趋势策略

基线规则：

```text
EMA10 crosses above EMA30 → Enter
EMA10 crosses below EMA30 → Exit
```

参数对照：

| Strategy |   EMA | Trades | Win Rate |   PF | Avg Duration | Return |
| -------- | ----: | -----: | -------: | ---: | -----------: | -----: |
| Fast     |  5/15 |    232 |    25.0% | 0.69 |          14h | -4.57% |
| Baseline | 10/30 |    114 |    21.9% | 0.53 |          27h | -5.77% |
| Slow     | 20/60 |     62 |    16.1% | 0.52 |          49h | -3.35% |

从这组结果可以直接看到：

```text
Longer EMA period
→ fewer crosses
→ fewer trades
→ longer holding period
```

但交易次数减少并未带来更高胜率或更高 PF。

Slow 版本总亏损较小，但每笔平均净结果并未明显改善。它降低了交易次数和市场暴露，单笔信号质量没有明显提升。

结论：

> 指标参数首先改变策略的频率和持仓结构，不应直接解释为信号质量变化。

---

## 5. 退出规则与路径依赖

在保持 EMA10/30 入场不变的情况下，将退出改为：

```text
minimal_roi = {"0": 0.05}
stoploss = -0.08
use_exit_signal = False
```

结果：

| Metric       | EMA Exit | Fixed ROI Exit |
| ------------ | -------: | -------------: |
| Trades       |      114 |             24 |
| Win Rate     |    21.9% |          45.8% |
| PF           |     0.53 |           0.48 |
| Avg Duration |      27h |             9d |
| Return       |   -5.77% |         -5.48% |

固定 ROI 版本的退出分布：

| Exit       | Count | Average Result |
| ---------- | ----: | -------------: |
| ROI        |    10 |         +5.00% |
| Stop loss  |    13 |         -8.18% |
| Force exit |     1 |         +1.45% |

只考虑 ROI 和止损两类退出时：

```text
Break-even win rate
= 8.18 / (8.18 + 5.00)
≈ 62.1%
```

实际胜率仅为 45.8%。

这组结果表明，胜率提高并未改善 PF。

退出规则延长了持仓时间，使大量后续入场信号无法执行。两个策略使用相同的入场条件，实际交易样本却已经不同。

策略结果具有路径依赖：

```text
Exit rule
→ holding duration
→ capital availability
→ executable future entries
→ complete trade distribution
```

> 退出规则会改变既有交易的后续机会集合。

---

## 6. RSI 均值回归

基线规则：

```text
RSI(14) crosses below 30 → Enter
RSI(14) crosses above 50 → Exit
```

该规则描述的是“超卖抢反弹”，属于短期动量衰减假设，不承担估值判断功能。

其假设是：

> 当短期负向动量达到极端值后，未来发生短期反弹或动量衰减的概率可能提高。

结果：

```text
Trades:       50
Win rate:     44.0%
Profit factor: 0.37
Max drawdown: 5.05%
Return:       -4.97%
Best trade:   +3.87%
```

退出分布：

| Exit      | Count |   Total PnL |
| --------- | ----: | ----------: |
| RSI exit  |    45 |  -8.99 USDT |
| Stop loss |     5 | -40.70 USDT |

5 笔止损的亏损金额相当于最终净亏损的约 82%。这个比例只针对最终净亏损，gross loss 需要单独计算；尾部交易对结果的影响仍然很明显。

---

## 7. 止损敏感性

在保持 RSI 入场和退出不变的情况下，仅修改止损：

| Stop Loss | Trades | Win Rate |   PF | Worst Trade | Avg Duration | Return |
| --------- | -----: | -------: | ---: | ----------: | -----------: | -----: |
| -3%       |     63 |    44.4% | 0.45 |      -3.19% |       16h25m | -4.64% |
| -8%       |     50 |    44.0% | 0.37 |      -8.18% |          24h | -4.97% |
| -15%      |     48 |    43.8% | 0.37 |     -15.17% |       26h55m | -4.53% |

更紧的止损产生了三个效果：

```text
Earlier exit
→ smaller tail loss
→ earlier capital release
→ more executable future entries
```

63、50 和 48 笔交易来自不同的交易路径，不能视为同一组交易的不同退出结果。

不能仅根据退出数量差异推断：

- 哪些交易原本最终会盈利；
- 哪些止损交易原本会恢复；
- 某笔交易在另一版本中的反事实结果。

要完成严格比较，需要按原始信号时间匹配交易，或者使用独立的 event-based path analysis。

结论：

> 止损会重新塑造整个交易路径和收益分布，同时限制单笔最大亏损。

---

## 8. 趋势过滤器为何产生零交易

RSI 策略尝试了三个趋势过滤条件：

```text
close > EMA100
close > EMA20
EMA20 > EMA20.shift(5)
```

全部得到零交易。

目前能确认的是：

```text
RSI crosses below 30
AND
trend filter
```

在当前样本中的同一 candle 交集为零。

当前证据只支持“RSI 入场事件与趋势过滤器在同一 candle 没有交集”。以下结论仍缺乏证据：

- RSI 超卖和上升趋势在逻辑上互斥；
- BTC 在整个区间从未位于 EMA 之上；
- 趋势过滤器有效地删除了亏损交易。

正确的诊断方式应分别统计：

```text
A = RSI entry event count
B = filter true count
C = A ∩ B count
D = final entry count after all conditions
```

零交易首先反映信号空间为空，不能单独构成策略有效性证据。

---

## 9. Donchian 突破

Donchian 策略的基本结构是：

```text
Break above previous N-period high → Enter
Break below shorter exit channel → Exit
```

为避免使用当前 candle 自身高点，突破基准必须基于已完成 candle，例如：

```python
dataframe["entry_high"] = (
    dataframe["high"]
    .rolling(entry_period)
    .max()
    .shift(1)
)
```

成交量过滤器：

```text
volume > rolling_mean(volume, 20)
```

在该实验中只删除了约 5.8% 的交易，未显著改善结果。

这只能说明该简单过滤器在当前样本中没有产生明显区分能力，不能推广为“成交量无法过滤假突破”。

基线中没有交易触及 -8% 止损，说明较短退出通道主导了持仓生命周期。

但仅凭这一点无法证明退出截断了趋势盈利。还需要比较：

- 更宽的退出通道；
- 最大有利波动 MFE；
- 退出后的价格路径；
- 固定持有期退出。

---

## 10. Challenge 区间

三个基线策略在开发期和 Challenge 区间的排名发生反转。

开发期：

```text
Donchian loss < RSI loss < EMA loss
```

Challenge：

```text
EMA > RSI > Donchian
```

其中 EMA 在 Challenge 区间小幅盈利，但经济幅度很小、样本数量有限，正期望尚未得到证实。

这一结果说明：

> 单一市场区间中的策略排名不具有稳定的外推能力。

如果根据 Challenge 结果继续修改参数，则 Challenge 已转化为新的开发集。

---

## 11. 费用敏感性

费用从单边 0.1% 提高到 0.2% 后，策略结果明显恶化。

费用影响大致与换手相关：

```text
Higher trade count
→ more entry and exit fees
→ larger degradation in net return
```

但实验中曾出现最佳交易从 +7.82% 变为 +4.49% 的记录。

按每笔交易包含一次入场和一次退出计算，单边费用仅增加 0.1 个百分点时，完整交易的收益变化通常应接近额外 0.2 个百分点。3.33 个百分点的差异无法仅由费用解释。

该结果属于实验异常，优先检查：

- strategy class；
- config；
- timerange；
- result filename；
- cache；
- backtest result selection。

原则：

> 无法由实验变量解释的数字，不应被纳入策略结论。

---

## 12. Lookahead 与递归指标检查

`lookahead-analysis` 未发现未来数据使用。

`recursive-analysis` 显示 EMA30 在不同启动长度下存在约 0.117% 的末端值差异。

这组结果说明：

> 不同 startup candle 长度下，指标值尚未完全一致。

是否发生交易信号变化仍需单独验证。

进一步验证应当：

1. 选择足够大的统一 `startup_candle_count`；
2. 重新运行 EMA 策略；
3. 比较交易数量和信号时间；
4. 判断递归误差是否具有交易层影响。

---

## 13. Backtest 与 Dry-run

Backtest 评估：

```text
historical OHLCV
+ candle execution assumptions
+ strategy logic
→ simulated trade result
```

Dry-run 评估：

```text
real-time data ingestion
+ scheduling
+ signal generation
+ internal order state
+ trade persistence
→ simulated live operation
```

Dry-run 覆盖信号生成、模拟订单状态和 Trade 持久化。以下执行因素属于它的验证范围之外：

- 实际滑点；
- 订单簿排队；
- 市场冲击；
- 真实成交竞争；
- 交易所写请求延迟。

模拟状态流转如下：

```text
Signal found
→ simulated order created
→ order state updated
→ Trade persisted
```

实验期间出现一次：

```text
Strategy analysis took 1403.41s
```

对于单交易对简单 EMA 策略，23 分钟分析时间属于严重异常，远超正常实时延迟。

应单独排查：

- CPU 和内存竞争；
- 容器阻塞；
- 网络请求是否进入策略路径；
- DataProvider 调用；
- 外部服务延迟；
- 系统时间和日志上下文；
- 与其他容器的资源竞争。

`RestartCount=0` 只反映容器重启状态；系统阻塞或延迟仍需结合资源、网络和日志检查。

---

## 14. 从规则策略到机器学习

规则策略：

```text
features
→ manually defined condition
→ signal
```

机器学习策略：

```text
features
→ model
→ prediction
→ deterministic trading rule
→ signal
```

例如预测未来 6 小时收益：

```python
future_return_6h = close.shift(-6) / close - 1
```

训练特征可以包括：

```text
return_1h
return_6h
realized_volatility_24h
RSI
ATR / close
volume z-score
distance to EMA
```

模型输出需要经过交易规则转换，才能形成交易信号。

交易规则仍需定义：

```text
predicted return
>
fees + expected slippage + uncertainty margin
→ enter
```

同时还需决定：

- 预测期限；
- 滚动训练窗口；
- 重训频率；
- 时间序列切分；
- 标签重叠；
- 预测阈值；
- 持仓期限；
- 退出和止损；
- 模型失效条件。

FreqAI 可以管理：

- 滚动训练；
- 模型保存；
- 历史回测中的定期重训；
- Dry-run / Live 预测；
- 模型更新和恢复。

研究者仍需决定：

- 预测目标是否有意义；
- 特征是否泄漏；
- 模型是否具有样本外信息；
- 预测能否覆盖交易成本。

第一阶段的 ML 研究应先评价预测能力，再评价交易收益：

```text
prediction quantile
→ realized future return
```

如果预测分组和实际未来收益不存在稳定关系，那么优化退出和止损通常只是在重新包装噪声。

---

## 15. 结论

本次实验没有发现有效策略，同时验证了几个重要结构：

1. 参数变化首先改变交易频率和持仓结构；
2. 胜率不能脱离平均盈亏和尾部损失解释；
3. 退出机制会改变后续可执行信号；
4. 止损会改变完整交易路径，并限制单笔最大亏损；
5. 零交易表示条件交集为空，过滤器是否有效仍需另行验证；
6. 开发期排名外推到新市场状态时缺乏稳定性；
7. Backtest、Dry-run 和 Live 分别覆盖不同风险；
8. 机器学习将部分规则替换为统计预测，策略设计仍然存在。

一个 Freqtrade 策略应同时包含：

```text
data
+ feature definition
+ entry logic
+ exit logic
+ position occupancy
+ fees
+ execution assumptions
+ market regime
```

这些部分共同决定最终结果。
