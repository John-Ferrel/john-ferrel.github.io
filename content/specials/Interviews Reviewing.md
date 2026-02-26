---
title: Interviews Reviewing
draft: "False"
tags:
  - interview
created: 2026-02-25 19:45
modified: 2026-02-25 19:45
---
First, relax and treat the interview as a consultation where I aim to learn, understand, or delve deeper into a certain field.
The interviewer is usually a department head or key person in charge, with broader and deeper expertise in the professional area than I (and of course, a higher salary). This is a great opportunity to recognize your shortcomings and fill in the gaps.
Well, the word "interview" fits perfectly—it’s like saying, "瞅你咋啦" "瞅你咋地".

So I record some Q&As to modify my story.

## LSTM搭建情感分析模型

### 评估指标

二分类：准确率、F1-score、AUC-ROC
### 预处理

1. 文本清洗（去除HTML、特殊字符）
2. 分词
3. 构建词汇表（限制大小，如20000词）
4. 序列化（词→索引）
5. 填充/截断至固定长度

### 架构

输入 → 嵌入层 → LSTM层 → Dropout → 全连接层 → 输出

1. **输入层**: 清洗数据, 固定序列长度, 多则截断, 少则填充
2. **嵌入层**: 使用预训练词嵌入（Word2Vec）, 这里使用了知乎的一个预训练权重
3. **LSTM层**: 
4. **Dropout层**: 提高泛化性能
5. **全连接层**: 对其输出
6. **输出**: sigmoid二分类

### 超参数

1. **序列长度**：50-200个词（覆盖大多数评论）
2. **批次大小**：32-64
3. **学习率**：0.001（Adam优化器）
4. **训练轮次**：10-20（早停防止过拟合）
5. **正则化**：L2正则化 + Dropout

### Q&A
1. 数据怎么清洗: HTML标签, 特殊字符, 停用词, 分词
2. padding 怎么处理？pad到max_len, mask避免参与到loss
3. 用随机初始化 embedding 还是预训练? 用了预训练的, 提高收敛速度与泛化性能
4. 类别不平衡: 重采样
5. learning rate? 1e-3开始, 通常可以根据epoch设置milestone, 逐渐降低


## 合同识别中的印章识别与手写识别

### 项目架构

1. OCR 层（图像 → 文本 + 坐标）  
 2. 结构化解析层（段落 / 表格 / 条款识别）  
3. LLM 风险识别层（语义分析）  
4. RAG 法规增强层（知识校验）

