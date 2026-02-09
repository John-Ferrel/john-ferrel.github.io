---
title: Forecasting Principles and Practice 3rd
draft: "false"
tags:
  - math
  - statistics
  - forecasting
  - reading
created: 2026-02-01 17:04
modified: 2026-02-01 17:04
---
## Reading Goal

我面临的问题是, 我对于销售预测(比如 M5 competition) 缺乏足够的认识.
实际的数据比M5 competition的更差:
1. SKU 长度不定: 有的长, 有的短
2. 0膨胀: 大概90%的序列, 0 都超过95%
3. 数值问题: 存在负值, 数据稀疏
4. 结构性断点: 很多对序列有冲击的事件, 没有记录
5. 长周期性: 这点可能初看问题不大, 但实际上, 由于很多零售行业会有很多短生命周期的产品, 导致长周期性是危险的.

显然, 上述问题导致了Tabular方法直接预测可能就不够适用. 

我希望得到一个相对通用的预测框架, 主要针对零售行业. 因此, 我将目光转向了时间序列方法.


## What can be forecast

对问题的充分定义, 以及合理地评估预测结果, 可能比如何预测更加重要. 



