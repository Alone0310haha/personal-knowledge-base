---
title: Transformer 如何成为 AI 模型的地基
bvid: BV1k6yWBEEmH
type: video-note
source: https://www.bilibili.com/video/BV1k6yWBEEmH/
transcript: file:///d:/code/self/tmp/BV1k6yWBEEmH_文字稿.md
tags:
  - ai
  - llm
  - transformer
  - encoder-decoder
  - decoder-only
  - encoder-only
  - gpt
  - bert
  - sampling
  - temperature
  - top-k
related:
  - "[[多头注意力 Multi-Head Attention 机制详解]]"
  - "[[Token 与 Embedding：LLM 与 RAG 的数据处理机制]]"
---

# Transformer 如何成为 AI 模型的地基

## 来源

- 视频：https://www.bilibili.com/video/BV1k6yWBEEmH/
- BV 号：BV1k6yWBEEmH

## 讨论视角

- 用“翻译任务”的原始 Transformer（Encoder-Decoder）当作母体结构，只看输入/输出与信息流，先建立宏观框架
- 把“模型参数是什么、训练在调什么”嵌入到结构解释里：每层重复结构相同但参数不同
- 以“生成 token 的逐步过程”解释为什么输出比输入更贵、为什么有 temperature / top-k 这类采样参数
- 用结构裁剪的方式串起主流 LLM：Encoder-Decoder → Decoder-only（GPT 系）/ Encoder-only（BERT 系）

## 核心论点与分论点

### 核心论点

- Transformer 的核心贡献不只是一套内部算子，更是一种可堆叠的通用结构范式：Encoder 负责把文本压缩为高层次语义表示，Decoder 负责在条件语义下逐 token 生成；主流大模型基本都在这一范式上裁剪/变体而来。

### 分论点 1：原始 Transformer = Encoder + Decoder（面向翻译）

- 任务：把源语言句子翻译成目标语言句子
- Encoder：把整句原文一次性编码成一个“高层次数字表示”（文字稿称“含义矩阵/语义表示”），不直接对应任何人类语言，但承载语义信息
- Decoder：在 Encoder 输出的条件下，按 token 逐步生成目标语言

### 分论点 2：堆叠（×N）意味着“结构相同、参数不同”，训练就是在调这些参数

- Encoder/Decoder 内部的块会重复 N 次（图中的 ×N）
- 每一层结构相同，但参数不同（作者用 \(y=Wx+b\) 类比：每层是同类运算但 W、b 不同）
- 大模型文件里巨量的权重，本质就是这些层中参数矩阵的数值；训练就是用数据反复调整这些数值，使输出符合预期

### 分论点 3：Decoder 的逐 token 生成解释了“输出更贵”与“采样参数”

- 生成过程：当前已生成的 token 序列 + 条件语义输入 → 输出“下一个 token 的概率分布”
- 每生成一个 token，都要完整跑一遍 Decoder（多层堆叠）
- 结论：输出越长，计算越多；因此 API 计费/耗时上输出通常比输入更贵
- 为了让结果不总是同一句话，需要在“从概率分布选 token”时引入随机性：
  - temperature：控制随机性强弱（越高越随机，0 时趋向总选最大概率）
  - top-k：只在概率最高的前 k 个 token 中采样

### 分论点 4：Transformer 的两大裁剪路线：GPT（Decoder-only）与 BERT（Encoder-only）

- Decoder-only（GPT 系）：把 Encoder-Decoder 的 Encoder 去掉/吸收，只保留适合“文字接龙/自回归生成”的 Decoder 结构（文字稿称 “Decoder-only Transformer”）
  - 训练更方便：可用海量任意文本做自监督（给定前文预测下一个 token），输入/输出可由规则自动构造
  - 产物：GPT、Claude、Gemini、DeepSeek、Kimi 等主流生成式大模型都属于此类变种
- Encoder-only（BERT 系）：保留 Encoder，用“掩码填空”式训练来学理解（mask middle token 让模型预测缺失词）
  - 更擅长：理解文章大意、信息抽取、实体识别、标注关系等
  - 产物：BERT 及其变体

## 关键数据 / 案例 / 结论

- 时间线（作者用来定位 Transformer 与大模型关系）：
  - 2017：提出 Transformer（Attention Is All You Need）
  - 2018：BERT 基于 Transformer 刷新多项榜单
  - 2019：GPT-2 让“大语言模型”概念进入大众视野
- 翻译示例：输入 “I am Wang” → Encoder 得到语义表示 → Decoder 从开始标记起逐 token 生成“我是王”直到结束标记
- token 概念：Decoder 输出的是词表中每个 token 的概率分布（文字稿举例 GPT-2 词表 50,257）
- 训练方式对比：
  - Encoder-Decoder 翻译训练需要成对语料（原文+译文），属于监督学习
  - Decoder-only 训练可用任意文本做 next-token 预测，属于自监督学习（输入前缀，输出下一个 token）
  - Encoder-only（BERT）用 mask 预测缺失词，偏理解型

## 思维导图（Markmap）

```markmap
# Transformer 成为 AI 模型地基
## 原始 Transformer（翻译）
### Encoder（编码器）
#### 输入：整句原文
#### 输出：高层次语义表示（人类不可直接读）
#### 结构：块重复 ×N（结构同、参数不同）
### Decoder（解码器）
#### 输入：语义表示 + 已生成 token
#### 输出：下一个 token 概率分布
#### 逐 token 生成直到结束标记
## 参数与训练
### 每层参数不同（类比 y=Wx+b）
### 训练=用数据调整参数数值
### 模型文件很大=存权重参数
## 生成为什么贵
### 每生成 1 个 token 都要完整跑一遍 Decoder
### 输出越长，计算越多 -> 输出更贵
## 采样策略
### temperature：随机性强弱
### top-k：只在前 k 概率 token 中采样
## 两条裁剪路线
### Decoder-only（GPT 系）
#### 适合自回归生成（文字接龙）
#### 训练：自监督 next-token
#### 代表：GPT/Claude/Gemini/DeepSeek/Kimi…
### Encoder-only（BERT 系）
#### 适合理解与信息抽取
#### 训练：mask 填空预测
#### 代表：BERT 及变体
```

## 文字稿

- file:///d:/code/self/tmp/BV1k6yWBEEmH_文字稿.md
