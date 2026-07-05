---
title: "理解注意力机制：从缩放点积到多头注意力"
date: 2026-07-05
draft: false
tags: ["LLM", "Reasoning"]
summary: "深入解析 Transformer 中的注意力机制——从 Query-Key-Value 的直觉出发，经过缩放点积注意力，到多头注意力，配合具体数值示例追踪数据流。"
ShowToc: true
math: true
---

注意力（Attention）是 Transformer 捕获长距离依赖的核心机制。这篇文章从数学推导到代码实现，逐步拆解，每一步都给出具体数值。

## 缩放点积注意力

给定 Query $Q$、Key $K$、Value $V$，注意力输出为：

$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right) V$$

用一个小型示例来追踪。设 $d_k = 4$：

```python
Q = [[1.0, 0.0, 0.5, 0.2]]   # shape (1, 4)
K = [[0.8, 0.1, 0.3, 0.6],   # shape (3, 4)
     [0.2, 0.9, 0.1, 0.4],
     [0.5, 0.5, 0.5, 0.5]]
V = [[1.0, 0.0],             # shape (3, 2)
     [0.0, 1.0],
     [0.5, 0.5]]
```

**Step 1**：计算 $QK^T$。对第一个 Key：$1.0 \times 0.8 + 0.0 \times 0.1 + 0.5 \times 0.3 + 0.2 \times 0.6 = 0.8 + 0 + 0.15 + 0.12 = \mathbf{1.07}$。

## 为什么要除以 √d_k？

随着 $d_k$ 增大，点积的方差也会增大。假设 $Q = [0.1, 0.2, 0.3, 0.4]$，$K = [0.1, 0.2, 0.3, 0.4]$，点积为 $0.01 + 0.04 + 0.09 + 0.16 = 0.30$。但当 $d_k = 1280$（如 GPT-3），点积的方差约为 1280，会把 softmax 推入梯度消失的区域。

> $\sqrt{d_k}$ 缩放使得无论维度多大，方差都保持在约 1，确保训练过程中 softmax 梯度稳定。

## 多头注意力

不是只做一个注意力函数，而是将 $Q, K, V$ 投影到 $h$ 个不同的子空间并行计算：

$$\text{MultiHead}(Q, K, V) = \text{Concat}(\text{head}_1, \dots, \text{head}_h) W^O$$

以 $d_{\text{model}} = 512$、$h = 8$ 为例，每个头处理 $d_k = 512 / 8 = 64$ 维。这允许模型同时关注来自不同表示子空间的信息。

### 为什么要多个头？

不同的头学习不同的模式：有的头捕获语法关系，有的关注语义关联。研究（Clark et al., 2019）表明某些头专门做共指消解，另一些头专注于关注前一个 token。

## 数值计算完整示例

继续上面的计算。$d_k = 4$，缩放因子 $\sqrt{4} = 2$：

```
scores = QK^T / 2 = [1.07, 0.82, 1.0] / 2 = [0.535, 0.410, 0.500]
weights = softmax(scores) = [0.349, 0.310, 0.341]
output  = weights @ V
        = 0.349*[1,0] + 0.310*[0,1] + 0.341*[0.5,0.5]
        = [0.349+0+0.171, 0+0.310+0.171]
        = [0.520, 0.481]
```

## 实现代码

```python
import torch
import torch.nn.functional as F

def scaled_dot_product_attention(Q, K, V, mask=None):
    """缩放点积注意力
    
    Args:
        Q: Query tensor, shape (..., seq_len_q, d_k)
        K: Key tensor,   shape (..., seq_len_k, d_k)
        V: Value tensor,  shape (..., seq_len_k, d_v)
        mask: 可选，用于 decoder 的因果遮罩
    
    Returns:
        output: 注意力输出, shape (..., seq_len_q, d_v)
        weights: 注意力权重, shape (..., seq_len_q, seq_len_k)
    """
    d_k = Q.size(-1)
    # Step 1: 计算注意力分数
    scores = torch.matmul(Q, K.transpose(-2, -1)) / (d_k ** 0.5)
    # Step 2: 应用遮罩（如果有）
    if mask is not None:
        scores = scores.masked_fill(mask == 0, -1e9)
    # Step 3: softmax 归一化
    weights = F.softmax(scores, dim=-1)
    # Step 4: 加权求和
    return torch.matmul(weights, V), weights
```
