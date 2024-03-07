# 主要工作
1. 分析和剖析现有 Transformer 架构中的瓶颈以及起以往卷积模型的异同
2. Transformer 架构在硬件上的影响，包括一些归一化层、softemax 和 GELU 非线性操作和线性操作对设计的影响
3. 优化一个固定 Transformer 架构的方法
4. 为 Transformer 模型找到正确的操作映射和调度挑战
5. 通过使用神经架构搜索来调整架构以优化 Transformer 的方法

# 介绍
CPU 和 GPU 主要用于通用计算, 但是深度学习模型常常是只有很少一部分不同的操作，并且由相当多的重复操作。GPU 和 CPU 的缺点之一就是无法利用大量数据

深度学习所用的硬件加速器具备三个特点：
1. 高效的计算
2. 少量不同操作的使用
3. 数据的重复利用

缺少对 Transformer 架构的 workload 的特征的理解

Transformer 主要由矩阵乘法和内存密集型非线性运算组成，还有很多种类操作节点，和数据流的拆分和串联。

分析的两个角度：
1. 分析 transformer 在运行时的数据特征和调研不同高效的 tranformer 推理模型
2. 通过在 full-stack DNN 加速器生成器 Gemmini 上应用我们所调研的模型来进行样例探究，来表征软件、硬件堆栈中的不同因素，从而优化 transformer 推理

	Gemmini 最初是为了 CNN 设计的，其并没有对 transformer 有很好的硬件加速支持

如果在 CNN 的 DSA 上运行 transformer 的主要瓶颈不一定会是线性运算上，可能更多的是浮点非线性运算以及量化和反量化运算所花费的时间

对于 transformer 加速器来说，accumulator 越大，scratchpad 越小越好。但是对于 CNN 加速器则是相反的情况更加理想

找到理想的矩阵乘法调度器对于 transformer 来说是一个挑战