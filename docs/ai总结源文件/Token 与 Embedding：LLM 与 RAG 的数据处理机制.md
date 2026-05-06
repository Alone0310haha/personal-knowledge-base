---
tags:
  - ai
  - llm
  - embedding
  - rag
  - 程序员
---

# Token 与 Embedding：LLM 与 RAG 的数据处理机制

## 来源

- [Token 与 Embedding：LLM 与 RAG 的数据处理机制](https://www.bilibili.com/video/BV1oNv8BPE2m/)

## 讨论视角

- 从“计算机不擅长直接处理文字”出发，解释为什么大模型入口先把文字变成 token（编号）
- 从“模型是连续函数”出发，解释为什么还需要 embedding：把离散编号映射到高维连续向量，才能表达“既近又远”的语义/语言关系并提高训练效率
- 对比澄清两类 embedding：
  - 大语言模型内部的 token embedding（逐 token）
  - RAG/检索用的文本 embedding（整段/整句）

## 核心论点与分论点

### 核心论点

- 大语言模型实际处理的是 token（数字编号）及其 embedding（高维向量），而不是原始文字；tokenizer 属于固定的数据预处理，而 embedding 是模型的一部分、需要训练得到。

### 分论点 1：Tokenization 的作用是把文字变成高效可计算的编号序列

- 计算机存储与计算天然更适合数值：文字在底层是离散字符序列，直接让模型从字符“读出概念”再处理会很慢
- 做法：构建词典/词表，用数字代表词或词片段；文本先经 tokenizer 变成 token 序列，再送入模型
- token 与“单词”不必一一对应：可把单词拆成词片段 token，以覆盖未见词（作者用 “oldest = old + EST” 类比）

### 分论点 2：Embedding 的作用是把“1 维编号”升维成“高维向量”，表达复杂相似性关系

- 作者给出的关键前提：大模型可近似看作一个复杂的连续函数；若输入数值更接近，输出更容易呈现相似性
- 但 token 只是一个数字（1 维），无法同时表达多种“相似/差异维度”（语言、类别、形状、颜色等）
- 解决：把 token 映射为多维向量（embedding），每个维度可承载某些抽象特征，从而表达“香蕉与苹果同为水果但不同形状/颜色”“苹果与 apple 同物不同语言”等“既近又远”的关系

### 分论点 3：Tokenizer 固定不变；Embedding 通过训练得到，是模型结构的一部分

- tokenization 属于数据预处理：训练时 tokenizer 固定不变，不在模型里更新
- embedding 不同：维度很高、要为每个 token 分配一组向量参数，无法手工设计，只能通过训练学出
- embedding 向量中每个维度的含义通常不可解释（作者强调：我们只知道这样做效果好，但难以解释每一维代表什么）

### 分论点 4：LLM embedding 与 RAG embedding 的相同点与根本差异

- 相同点：都把文本映射到向量空间，用向量距离/相似度表达语义关系
- 不同点（目的与训练方式）：
  - LLM：训练目标是“预测下一个 token”（文字接龙式 next-token prediction）
  - RAG embedding：训练目标是“概括文本含义并可检索”，常用对比学习（正样本更近、负样本更远）
- 不同点（架构细节）：作者提到 RAG/embedding 模型常为 encoder-only（类似 BERT）且一般不做 mask；而 LLM（decoder-only）为自回归生成需要 mask

## 关键数据 / 案例 / 结论

- 词表大小（vocab size / wordpiece size）影响效率与覆盖范围：
  - GPT-2 级别词表：约 50,257 个 token；中文常出现“一个汉字一个 token”，甚至生僻字需要多个 token，导致效率低与支持差
  - GPT-4o 的 tokenizer 词表：约 200,000，覆盖更多中文词，提升中文处理效率
- embedding 维度（作者称 d_model / n_embd）示例：
  - GPT-2 XL：约 1,600 维
  - DeepSeek-R1：约 7,168 维
  - 作者推测：更顶尖的模型 embedding 维度可能已上万（维度越高，代表“描述一个词的视角”越细腻）
- embedding 的一种实现方式（作者的解释路径）：
  - 把 token 先表示成 one-hot（长度=词表大小、几乎全 0）
  - 通过一个普通线性层（linear）把 one-hot 映射到 embedding 维度
  - 工程上可用查表/稀疏优化，但数学本质一致
- 输出端通常还有“反向映射”：
  - 模型内部处理的是 embedding
  - 最后通过线性层把 embedding 转回 token 空间，便于输出人类可读文字

## 思维导图（Markmap）

```markmap
# Token 与 Embedding（LLM & RAG）
## Token（为什么不是直接文字）
### 计算机擅长数值，不擅长直接处理离散字符
### Tokenizer：文字 -> token 编号序列
### token 不等于单词
#### 可拆成词片段（subword）
#### 覆盖未见词（例：old + EST）
### 词表大小（vocab size）影响效率
#### GPT-2 ~50,257：中文常一字一 token，效率低
#### GPT-4o ~200,000：覆盖更多中文词，效率更高
## Embedding（为什么还要再做一步）
### 模型可近似看作连续函数
### 1 维 token 无法表达多维相似性（语言/类别/形状/颜色…）
### Embedding：token -> 高维向量
#### 维度越高，描述视角越细腻
#### 示例：GPT-2 XL ~1600；DeepSeek-R1 ~7168；更大模型可能上万
## Tokenizer vs Embedding 的本质区别
### Tokenizer：预处理组件，训练时固定不变
### Embedding：模型的一部分，参数通过训练学得
### 向量每一维含义通常不可解释
## Embedding 的实现（直观版）
### one-hot（长度=词表大小）
### 线性层映射到 d_model 维
### 工程优化：查表/稀疏矩阵（本质一致）
## LLM embedding vs RAG embedding
### LLM
#### 目标：预测下一个 token（next-token）
#### 自回归：需要 mask（decoder-only）
### RAG embedding
#### 目标：整段语义向量用于检索
#### 训练：对比学习（正样本近、负样本远）
#### 常见：encoder-only（类似 BERT），通常不做 mask
```

