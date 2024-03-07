---
title: "How Quantization Works: From a Matrix Multiplication Perspective"
date: 2024-03-06
updated: 2024-03-07
lang: zh-CN
tags:
  - ml-system
  - llm
  - quantization
  - gemm
  - cuda-kernel
mathjax: true
---

## Introduction

量化是一种神经网络推理常用的加速技术。神经网络中主要的计算量来自卷积、线性层、Attention 等，这些算子在底层都是由 GEMM 等实现的。本文旨在**从矩阵乘法角度讨论量化的原理，并说明为什么有些量化是没有实用价值的**，也旨在从这个角度重新审视一些 LLM 量化方法。

我把**实用的量化**定义为：

1. 量化后仍然**可以使用 GEMM 完成运算**。这同时需要数学形式上的可行性和硬件的支持。它是能取得加速效果的基本要求；
2. 可以产生**实际加速效果**。加速可以来自更高的 INT8 硬件算力，也可以来自更小的 memory footprint 所节约的内存带宽。重要的是，加速收益必须大于量化开销。

## Let's do some math

设神经网络中的一个算子可以写成矩阵乘法形式：
$$\mathbf{Y}=\mathbf{X} \mathbf{W}^\top,$$
其中 $\mathbf{X} \in \mathbb{R}^{N \times C}$，$\mathbf{Y} \in \mathbb{R}^{N \times D}$，$\mathbf{W} \in \mathbb{R}^{D \times C}$，同时记量化后的版本为 $\hat{\mathbf{X}}$，$\hat{\mathbf{Y}}$，$\hat{\mathbf{W}}$。我们的目标是量化后仍然能用 GEMM 完成运算，即：
$$\hat{\mathbf{Y}}=\hat{\mathbf{X}} \hat{\mathbf{W}}^\top.$$

设 $\mathbf{X}$，$\mathbf{Y}$，$\mathbf{W}$ 的 per-element 量化函数分别为 $p_ {nc}(\cdot)$，$q_ {nd}(\cdot)$，$r_ {dc}(\cdot)$，即：
$$\begin{aligned}
    \hat{x}_ {nc} &= p_ {nc}(x_{nc}), \\\\
    \hat{y}_ {nd} &= q_ {nd}(y_{nd}), \\\\
    \hat{w}_ {dc} &= r_ {dc}(w_{dc}).
\end{aligned}$$
对应的反量化函数记为 $p_ {nc}^{-1}(\cdot)$，$q_ {nd}^{-1}(\cdot)$，$r_ {dc}^{-1}(\cdot)$。那么有：
$$\begin{aligned}
y_ {nd}
&= \sum_ {c=1}^{C} x_ {nc} w_ {dc}, \\\\
q_ {nd}^{-1}(\hat{y}_ {nd}) &= \sum_ {c=1}^{C} p_ {nc}^{-1}(\hat{x}_ {nc}) \cdot r_ {dc}^{-1}(\hat{w}_ {dc}).
\end{aligned}$$
上述公式即是**实用的量化**在数学上所需要满足的**基本约束条件**。

## Some basic quantization methods

有了这个基本约束条件，我们就能讨论几种基本的量化方法，包括 per-element, per-channel, per-token 和 per-tensor 量化。

### Per-element and Per-channel

上述基本约束条件中，左边的反量化函数 $q_ {nd}^{-1}(\cdot)$ 与 $c$ 无关，显然如果右侧的量化函数 $p_ {nc}^{-1}(\cdot)$ 和 $r_ {dc}^{-1}(\cdot)$ 与 $c$ 有关，**这个约束条件将不成立**。这意味着这两个条件无法同时满足：

1. 可以使用 GEMM 完成运算；
2. 在不同 $\mathbf{X}$ 和 $\mathbf{W}$ 的 channel 上有不同的量化函数。

换句话说，这说明了 **per-element 和 per-channel 量化不能用 GEMM 加速，他们不具备实用价值**。

### Per-token and per-tensor

由上述讨论可知，实用的量化至少需要满足：
$$\begin{aligned}
    p_ {n}(\cdot) &= p_ {nc} (\cdot), \quad \forall n, c, \\\\
    r_ {d}(\cdot) &= r_ {dc} (\cdot), \quad \forall d, c,
\end{aligned}$$
即**每个 channel 上的量化函数相同**，那么基本约束条件化为：
$$q_ {nd}^{-1}(\hat{y}_ {nd}) = \sum_ {c=1}^{C_ i} p_ {n}^{-1}(\hat{x}_ {nc}) \cdot r_ {d}^{-1}(\hat{w}_ {dc}),$$
如此，我们得到了 **per-token 量化**。如果我们进一步设：
$$\begin{aligned}
    p(\cdot) &= p_ {nc} (\cdot), \quad \forall n, c, \\\\
    r(\cdot) &= r_ {dc} (\cdot), \quad \forall d, c,
\end{aligned}$$
即**量化函数对整个 $\mathbf{X}$ 和 $\mathbf{W}$ 中的元素都相同**，那么基本约束条件化为：
$$q_ {nd}^{-1}(\hat{y}_ {nd}) = q^{-1}(\hat{y}_ {nd}) = \sum_ {c=1}^{C_i} p^{-1}(\hat{x}_ {nc}) \cdot r^{-1}(\hat{w}_ {dc}),$$
我们就得到了 **per-tensor 量化**。这两种量化存在理论上的可行性，但实际使用还需要考虑硬件的支持（见下一节）。

方便起见，下文只讨论 per-token 量化，per-tensor 量化可以视为 per-token 量化的特例。实际中最常用的量化方式是**对称均匀量化**，它使用乘法实现取值范围的缩放：
$$\begin{aligned}
    \hat{x}_ {nc} &= p_ {n}(x_ {nc}) = p_ n x_ {nc}, \\\\
    \hat{w}_ {nd} &= r_ {d}(w_ {dc}) = r_ d w_ {dc}, \\\\
    \hat{y}_ {dc} &= q_ {nd}(y_ {nd}) = p_ n r_ d y_ {nd}.
\end{aligned}$$

我们可以把 per-token 对称均匀量化写成矩阵乘法的形式：
$$\begin{aligned}
    \hat{\mathbf{X}} &= \text{diag}(p_1,\cdots,p_ N)\cdot \mathbf{X} = \begin{pmatrix}
        p_ 1 & \cdots & p_ 1 \\\\
        \vdots & \ddots & \vdots \\\\
        p_ N & \cdots & p_ N
    \end{pmatrix} \otimes \mathbf{X}, \\\\
    \hat{\mathbf{W}} &= \text{diag}(r_1,\cdots,r_ D)\cdot \mathbf{W} = \begin{pmatrix}
        r_ 1 & \cdots & r_ D \\\\
        \vdots & \ddots & \vdots \\\\
        r_ 1 & \cdots & r_ D
    \end{pmatrix} \otimes \mathbf{W}, \\\\
    \hat{\mathbf{Y}} &= \text{diag}(p_1,\cdots,p_ N)\cdot \mathbf{Y} \cdot \text{diag}(r_1,\cdots,r_ D) = \begin{pmatrix}
        p_ 1 r_ 1 & \cdots & p_ 1 r_ D \\\\
        \vdots & \ddots & \vdots \\\\
        p_ N r_ 1 & \cdots & p_ N r_ D
    \end{pmatrix} \otimes \mathbf{Y},
\end{aligned}$$
其中 $\otimes$ 代表 element-wise matrix multiplication。可以看到这些量化和反量化都**可以用维度广播的 element-wise matrix multiplication 高效实现**，下图使用了一个例子展示了其计算过程：

![](quant_matrix.png)

## Hardware requirements

量化是否可以使用 GEMM，还需要考虑硬件的支持。例如，在 NVIDIA GPU 上，Tensor Core 支持 FP16 和 INT8 的矩阵乘，但不支持 FP16/INT8 混合精度矩阵乘。这意味着 W8A8 量化可以使用 Tensor Core 加速，但 W8A16 和 W16A8 量化缺少硬件加速的支持，在 NVIDIA GPU 上并不能取得加速效果。许多 W8A16 和 W16A8 量化方法实际上是将 INT8 反量化后使用 FP16 完成计算，它们的实际加速效果需要进一步讨论（见下文）。

## Performance analysis

以上讨论只说明了 per-token 量化可以使用 GEMM 计算，下文将讨论它是否具备实际加速效果。

我们比较如下三种情况：

1. 未量化，全部使用 FP16 存储和计算；
2. W8A8 量化，但输入输出的 activations 是 FP16 存储的，这是 `LLM.int8()` 等工作采用的方式。为了避免引入额外的 CUDA kernel launch 开销，我们假设量化和反量化操作都和 GEMM 进行了算子融合；
3. W8A16 量化，在内部把 weights 转化为 FP16 完成计算。同样使用了算子融合。

不失一般性，我们可以假设硬件的 INT8 吞吐量是 FP16 的两倍，即一次 INT8 运算的归一化操作数是 $1$，而 FP16 是 $2$。我们可以列出下表：

|计算方式|FP16|W8A8 (FP16 activations I/O) |W8A16|
|:-:|:-:|:-:|:-:|
|GEMM OPs|$2NCD$|$NCD$|$2NCD$|
|GEMM Mem I/O|$2(NC+CD+ND)$|$2NC+CD+2N D$|$2NC+CD+2ND$|
|量化/反量化 OPs|$0$|$2NC+4ND$|$2CD$|
|量化/反量化 Mem I/O|$0$|$2(N+C_o)$|$2D$|
|总 OPs|$2NC D$|$NC D+2NC+4N D$|$2NCD+2CD$|
|总 Mem I/O|$2(NC+C D+N D)$|$2NC+C D+2N D+2(N+C_o)$|$2NC+CD+2ND+2D$|
|总算术强度 (OPs:I/O)|$\cfrac{1}{1/N+1/C+1/D}$|$\cfrac{1+2/D+4/C}{2/N+1/C+2/D+2/(NC)+2/(CD)}$|$\cfrac{1+2/N}{1/(2N)+1/C+1/D+1/(NC)}$|
|总算术强度（二阶近似）|$\cfrac{1}{1/N+1/C+1/D}$|$\cfrac{1}{2/N+1/C+2/D}$|$\cfrac{1}{1/(2N)+1/C+1/D}$|

分析上表，可以得出结论：

1. W8A8 量化（FP16 activations I/O）相对 FP16 减少了近一半计算量，但确降低了总算术强度。因此在 memory-bound 的场景 W8A8 量化可能无法获取 $2\times$ 吞吐量（ZeroQuant 改进了此问题，下文会介绍），但**在内存带宽足够的场景下会带来可观的吞吐量提升**；
2. W8A16 量化的计算量相对 FP16 基本保持不变，但总算术强度略有提升（当 $N$ 很大时提升更明显）。因此**在 memory-bound 的场景它也有实用价值**，特别是 LLM 中 activations 通常比 weights 更难量化。

## 一些 LLM 量化的例子

### `LLM.int8()`

`LLM.int8()` 实质上使用了选择性 per-token 量化。它使用 FP16 存储 weights 和 activations，然后对于不同的 token 采用不同的策略（如下图所示）：

![LLM.int8()](llm_int8.png)

- 对于适合量化的 token，它对 weights 和 activations 使用了 per-token 的 INT8 量化，并使用 INT8 GEMM 计算得到结果后反量化成 FP16；
- 对于包含离群值的 token，则直接使用 FP16 进行计算。

这两部分的结果即可合成最终结果。

### SmoothQuant

虽然 per-channel 量化并不实用，但对于 LLM activation 量化来说，最大的挑战来自于 activations，它在一些 channel 上会出现数量级较大的值（如下图）。

![](smooth_quant_motivation.png)

SmoothQuant 发现这些离群值出现的 channel 非常固定，但同时 weights 中很少会有离群值（因而容易量化），因此它提出在 activations 和 weights 之间平衡量化的难度，通过一个 per-channel 的缩放因子来实现：

![SmoothQuant](smooth_quant.png)

这种「平衡」可以用数学表述：
$$\begin{aligned}
    \mathbf{Y}
    &= \mathbf{X}\mathbf{W}^\top \\\\
    &= \mathbf{X} \cdot \text{diag}(s_ 1,\cdots,s_ C) \cdot \text{diag}(s_ 1,\cdots,s_ C)^{-1} \cdot \mathbf{W}^\top \\\\
    & = \left( \mathbf{X} \cdot \text{diag}(s_ 1,\cdots,s_ C) \right) \cdot \left( \mathbf{W}\cdot \text{diag}(s_ 1,\cdots,s_ C)^{-1} \right)^\top.
\end{aligned}$$
通过选取合适的缩放因子 $\text{diag}(s_ 1,\cdots,s_ C)$，我们就能达成平衡 activations 中的离群值的目的，随后我们就能对 $\mathbf{X} \cdot \text{diag}(s_ 1,\cdots,s_ C)$ 和 $\mathbf{W}\cdot \text{diag}(s_ 1,\cdots,s_ C)^{-1}$ 进行量化。下图是这个过程的一个例子：

![SmoothQuant example](smooth_quant_2.png)

**SmoothQuant 是 per-channel 量化的一种绝佳替代**，论文也展示了它在 LLM W8A8 量化中令人印象深刻的表现。

### ZeroQuant

在上述对 W8A8 的性能分析，我们发现如果使用 FP16 传递 activations，则量化后的总算数强度变小了，在 memory-bound 场景下可能无法达到预期的吞吐量提升。ZeroQuant 解决了这一问题，它把量化过程融合到前一个算子后，而反量化过程融合到 GEMM 之后，如下图所示。

![ZeroQuant](zero_quant.png)

如此，算子之间的传递的 activations 仍然是 INT8 的，这样可以将总的访存量降到 $NC+CD+ND+2(N+D)$，算术强度提升到和 FP16 接近的水平，充分发挥 INT8 高吞吐量的优势。

## 总结

本文从矩阵乘法的角度梳理了量化，并说明了实用的量化需要满足的一些基本条件，也说明了 per-channel 量化不实用的原因。本文还讨论了 per-token 量化的一些例子，包括 `LLM.int8()`，SmoothQuant 和 ZeroQuant。这些例子都是实用的量化方法，并且在实际中取得了可观的加速效果。