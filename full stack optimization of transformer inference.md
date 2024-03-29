 # Transformer 综述

---


## 主要工作
1. 分析和剖析现有 Transformer 架构中的瓶颈以及起以往卷积模型的异同
2. Transformer 架构在硬件上的影响，包括一些归一化层、softemax 和 GELU 非线性操作和线性操作对设计的影响
3. 优化一个固定 Transformer 架构的方法
4. 为 Transformer 模型找到正确的操作映射和调度挑战
5. 通过使用神经架构搜索来调整架构以优化 Transformer 的方法

---


## 介绍
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

---

## Transformer架构和性能瓶颈

一般来说，transformer模型由很多transformer模块组成，每个模块包含一个MHA模块和FFN模块。每种这样的模块最后会有一个LayerNorm操作和残差链接

### 非线性操作
一些类型：softmax,layernorm,GELU
虽然说这些非线性操作只占有很少一部分，但如果处理不当，可能会造成巨大的开销

由于它需要对所有输入值进行多次传递并将这些值保存在临时内存中，则非线性运算的挑战为：
1. 有效利用内存
2. 高效计算

对于softmax的操作包括三个部分
1. 指数操作
2. 将序列长度维度上的结果相加
3. 归一化输入（输入除以求和结果）

指数操作很容易会溢出，我们采用最大减法策略，引入最大值。但是，这样需要对输入进行额外的遍历，从而产生三次数字稳定实现

计算LayerNorm也需要在隐藏维度上对整个输入值进行多次遍历
1. 计算平均值
2. 计算标准差
3. 对输入值进行除法，进行标准化处理


    操作融合：通过将多个操作合并为单个操作来减少层间通信
	它一种方法，一个操作的输出直接作为后续操作的输入，而并不需要将其写入片外存储


由于LayNorm需要在运行的时候计算平均值和方差，所以，为了将此操作和前面的矩阵乘法操作融合，在输出结果之前，必须在reduction dimension也就是计算均值和方差的维度上累加上整个输出矩阵

但是这些会到导致不规则的tiling dimension和较低的数据利用率

由此，将这些操作和之前的层融合到一起，还是更好的利用tiling dimension来最大化冲利用，存在一个重要的权衡问题

### encoder和decoder架构
transformer最初引入encoder-decoder模型在机器翻译问题上

encoder是将整个原语言句子作为输入，通过多个transformer encoder blocks进行遍历，从而提取输入句子的高级特征

decoder通过encoder生成的原语言特征和之前的tokens来依次生成目标语言的tokens

#### encoder block
推理主要由矩阵乘法、元素加法和非线性操作构成

MHA和FFN模块中projection layers的开销和输入序列长度成线性关系

MHA中的act-to-act matmuls与输入序列长度成二次正比

序列长度短，projection layers占主导地位，encoder模块的总体复杂度为o(l),而对于较长的序列，act-to-act matmuls占主导地位，整体复杂度为o(l<sup>2</sup>)

#### decoder block
模块在预测句子中的token也是基于先前产生过的token

encoder模块对整个属于序列进行操作，而decoder依次只能推断一个token

操作随序列长度的增加而线性增加，也就是说，处理一个token在长时间步长中要比短时间步长所需要的计算更多。

目前常见的token生成技术是在后续迭代中缓存和重用先前产生的token的中间键和值


---

## 模型分析

### 工作负载分析

算术强度：从内存中加载的每个字节可以执行的浮点运算次数，也就是FLOPs/MOPs（访问的总字节数）

#### 端到端FLOPs和MOPs
由于在act-to-act matmuls中序列长度的二次复杂度，FLOPs和MOPs都是超线性的

#### 端到端算术强度
比MHA模块具有更高算术强度的FFN模块在小序列中占主导地位

在较大的序列长度中，这种趋势相反，这是由于MHA对于act-to-act matmuls的成本随着序列长度呈二次增长，导致端到端推理模型的算术强度降低

GPT-2展现出较低的算术强度。这是因为decoder仅由矩阵向量操作组成，这限制了数据重用的机会

每次矩阵向量操作的时候，大致只执行一次乘法和加法，而且load不能跨token共享

#### 每层FLOPs,MOPs和算术强度
对于每层来说，act-to-act matmuls运算强度要比FFN和MHA模块的projection layers要低，因为前者是d/l维，相对于后者的d和d<sub>FFN</sub>来说是很小的。

由此可见，对于MHA模块来说，小数量的head可以减少MOPs并增大算术强度

非线性操作消耗的总体FLOPs很小，但是MOPs很大，对于越长的序列越是如此

### 分析

#### 时延细分
对于较短的时间序列，FFN模型占用的计算强度较大，而MHA大部分计算是projection layer

#### 端到端时延
由于解码器计算强度比较低，导致时延比编码大。但decoder推理是一个内存约束问题，而不是一个计算约束问题

---

## 硬件设计

### DNN加速器概述
DNN基本组成部件：
- 片外DRAM：保存整个网络的权重和activations
- 较小的片上内存：称为全局缓存，容纳足够容量的权重和activations，以允许数据重用并限制和片外DRAM之间的通信次数
- 一组PEs，每一个PE都有执行MAC操作的能力，还具一些RFs（寄存器文件）
- NoC

MAC操作三个参数：两个相乘的输入和当前部分的和
	从能耗角度来看，读写内存的开销要比MAC操作开销高几个数量级

数据流常常被分为时间数据流和空间数据流。前者是操作并行，后者数据可以在PE之间传输，并利用额外的数据重用机会
对于时间数据流，一般来说是对全局buffer中权重或者部分和进行重用的，PE之间并无通信（SIMD，SIMT）
对于空间数据流，数据在PE之间移动，其并无重复读取全局buffer。主要原因是因为PE是由RF的（FPGA，基于ASIC的加速器）
- 固定权重：直接将权重保存在PE的本地RF中并通过流来最小化权值矩阵所需的读取次数
- 固定输出：直接将输出保存在PE的本地RF中来减小能耗
- 啥都不固定：直接把RF的面积来去堆更大的全局buffer
- 行固定：在PE中保存一行固定的权重，输入和输出部分和来最大化部分和和权重的利用

### 将DNN加速器调整为Transformer
CNN和transformer加速器之间的一个区别是，每个内存层次的最优大小和带宽要求都不同

推理过程中如何计算非线性函数，也是一个很大的挑战。一般两种解决思路：在片上计算的时候提供特殊支持，或者直接丢给CPU

数据路径设计也需要考虑，这取决于加速器是为了MHA模块这几点还是为了端到端Transformer推理设计的。前者，将所有操作揉在一起，灵活度较低，但是有着较少的访问次数；后者通过在更通用的多engine中单独执行单个操作，其设计灵活度较高

对于MHA模块的优化，主要是从query × key,softmax,attention score × value这几个方面去优化

因为我们是针对特定数据流进行优化的，所以我们可以通过合理放置非线性处理单元来增加效率，但是会降低灵活性

一般来说，针对MHA模块的优化倾向于设计更厉害的数据路径来利用算子融合，而端到端的Transformer设计并不会去围绕着MHA模块中的graph-level数据流进行数据路径设计。

### 分析建模
假设计算时间和内存操作时间可以完全重叠，并且每个操作的总和是这二者的最大值。假定采用乒乓缓存

#### 时延细分和端到端时延
获取估计的延迟细分和端到端的运行时延

#### 非理想计算强度
非理想计算强度要比理想情况小不小，主要原因是tiling和在32为输出activation必须在非线性操作前加载和存储数据。当序列长度越长，后者的影响越明显

### 案例
CNN的FLOPs大多被用作计算matmuls和convolutions

一般CNN加速器不太支持非线性操作。在CNN加速器上跑BERT的时候，加速器本身不会支持LayerNorm这类操作，这类操作必须由cpu执行，会导致大量的执行开销。

想GELU和softmax这些也必须要给CPU进行浮点运算，导致能耗要比整数多几个数量级

matmuls是Transformer中相当大的开销，但是，最大程度优化matmuls性能，利用率无法达到1%以上。这是因为CPU卸载非线性开销导致的

Transformer执行的时候更多的时间花费在浮点非线性激活，规范化/量化和去量化上

为了缓解量化和去量化的时间，讲单纯的只有matmul被量化的BERT执行变成只有整数的I-BERT变体。本质上是用整型多项式近似代替浮点非线性运算

增加特殊的规范化单元和激活单元，去计算GELU，LayerNorm和Softmax


---

## 模型优化

### 量化
一般来说，推理并不需要很高的精度。量化可以对 DNN 模型进行压缩从而减小内存开销。模型尺寸也会因此大大减小。其次，量化激活进一步减少了中间部分内存带宽和存储。

优势：
1. 减少精确度的同时减少了内存上的开销
2. 进一步减少内存通信和中间部分的存储
3. 减少 ALU 和相应 PE 的大小、延迟和能耗

非均匀量化比均匀量化更能捕获原浮点数据的分布 

策略：混合精度量化
非均匀量化可以解决均匀量化异常值的问题，但也提出统一量化方案


### 稀疏
稀疏是一种通过移除冗余参数使得 DNN 模型变得稀疏的过程

裁减主要关注点是为了提升 NN 效率但是不牺牲其性能，什么权重应该被保留，什么会被删除

通常我们训练大模型然后通过裁减来对其进行压缩使其达到比从 scratch 中训练压缩模型更好的准确率。其原因是其能够使得损失情况更容易优化

一般来说，权重裁减分为两种类型：结构化裁减和非结构化裁减

非结构化裁减对硬件的充分有效利用是一件比较困难的事情。为了去更有效的存储非零数据，压缩内存形式是十分必要的，压缩单元甚至可以直接对压缩数据进行操作。

由于其对硬件设计不是十分友好，一般采用结构化裁减，但是结构化裁减的压缩率并没有比非结构化裁减高，反倒是低不少

还有一种裁减就是 activation 裁减，因为一句话中并不是所有单词对理解都是有用的，所以，可以删减掉一些不必要的词汇。
在硬件上，一般会需要检测逻辑来即时确定非零的位置。

#### 数据裁减
因为更小的数据对模型最终影响越小，所以，可以拿权重的绝对值作为衡量其重要性的标准

#### 运动裁减
在微调的过程中，考虑了权重的改变，随着微调的进行，对远离 0 的权重分配更大的重要性分数
它更加适用于使用预训练和微调方案训练的模型，随着微调的进行，其能够更好的捕捉哪些权重的重要

#### 一级裁减
其使用流入权重或一组权重的损失梯度作为评估模型准确率的重要参考标准。
这个方法认为梯度是在损失上参数归零影响的因素

相关的改进是，将权重大小和梯度乘积作为一个重要参考标准

#### 二级裁减
其使用损失权重的 Hession 矩阵作为评估模型准确率的重要参考标准
二阶信息能够更准确的去除权重的影响


裁减的一个主要的好处是减少了内存的访问。结构化裁减可以直接提高内存效率，这直接减少了矩阵乘法的大小和数量。非结构化裁减通常使用稀疏编码来压缩和存储数据

裁减还可以减小能耗和时延由于其能够削减不必要的计算。一般来说，结构化裁减实现这点相对简单，但是非结构化裁减需要一些特定的 trick 来识别并绕过空元素的计算

一些检测和跳过方法通过不设计 null 元素的操作来节约能耗。我们也可以为一些处于空闲状态中的 PE 分配不同的有效计算来减小时延

如果说，要通过非结构化稀疏 matmul 来保持 PE 利用率，必须还要执行负载均衡

由于不同 PE 工作强度不同，很有可能会出现 PE 在工作，其他的在等待的情况


### Transformer 特殊优化策略

#### 注意力加速
一种常见的途径是进行 token 裁减
DTA-Trans 在第一轮采用两级计划，这时决定哪个 token 应该被裁减。第二轮，根据剩余 tokens 的重要性来确定分配给每个剩余 token 的比特精度

另外一种途径是利用注意力评分激活的动态稀疏模式。一般需要专门的硬件逻辑来动态检测和加速这些动态稀疏模式

硬件支持对于加速注意力机制十分重要，因为其支持诸如 top-k, 角度近似, 聚类和多精度计算等操作，这些操作对于检测注意力分数的动态稀疏模式十分必要
#### 非线性操作
##### 函数近似
寻求近似非线性函数的精确值，以获得良好且计算效率高的近似值

很多工作的主要关注点是近似 Softmax 的指数运算，其他工作也利用了 log sum-exp 技巧来避免除法运算

##### 查找表
存储给定输入范围内的预先计算的输出值
现在主要的工作是，如何让查找表变得更小

#### 加速解码

一条减少推理时延的路是通过提前推出并跳过不必要的计算。该方法通过在中间层终止推理并使用中间隐藏状态进行预测来动态调整每个 token 生成的解码器深度，而不是等到结束层

状态迁移：退出到所跳过的层之前复制最后一层的激活

另外一种尝试是去使不同大小的模型协同工作。底层动机是大多数简单单词生成任务可以被放到一个更快但准确度更低的小模型中。这种方法不仅可以降低大模型执行频率，还可以实现其非自回归执行，因为它可以使用从小模型生成的所有 tokens 并并行处理他们，从而更有效的利用硬件

#### 选择更优的方案去使用

因此，在选择采用哪种优化技术的时候，必须用软硬件整体视角，并同时考虑底层硬件特性。
加速器是否在同一数据路径中支持 MHA 和 FFN 模块，还是每个模块包含单独的路径，这些会对进行的优化产生很大的影响

具有统一数据路径的加速器倾向于追求更通用的优化，这些优化既可以用于 MHA 和 FFN 模块，也可以至少应用于那些不需要改变数据路径以使其无法再计算其他模块的模式

如果 MHA 和 FFN 模块在单独数据路径中被分别计算或者 PE 可以重新配置，则可以单独进行更奇特的优化


---

## 将 Transformer 映射到硬件上

### 映射定义
用于在特定目标硬件体系结构上执行一组操作的一系列硬件指令

映射将列出完整的数据和内存指令序列，其最终目标是生成可以在硬件上执行的源代码或者编译的二进制文件

我们将映射决策的总空间以及其生成的映射称为映射空间

映射或者调度框架的目标是为给定的软件操作和所需的性能指标获得 Pareto-optimal 以及有效的映射

对于包括 Transformer 在内的 DNN 中核心计算算子，映射问题既是映射空间大带来的挑战，也是由于整体模型执行加速的潜在收益带来的回报

时空映射确定应该使用加速器的硬件资源并执行哪个循环

### 映射决策的关键

先向图像翻译成一系列张量操作，然后再把这些张量操作进行调度使其能够在硬件上运行

#### 图级层面
包括，改变图结构的决策，图中各个节点表示的张量运算的执行调度

##### 层融合或者操作融合

对于 CNN，将层间通信转变为层内通信，可以避免从内存中写/读；而对于 Transformer，此策略不可行‘

##### 张量的动态稀疏化
当根据激活映射作出裁减决策时，才会使用张量的动态稀疏化

常见的方法是对局部敏感做哈希，以将可能很小的点积归零

由于这种优化很大程度上依赖于数据，因此，对稀疏感知映射的结果相对较少，而这些结果主要涵盖给定书来嗯稀疏度的操作级映射的结果

##### 张量的静态稀疏化
其发生于裁减决策独立于激活的时候

#### 操作层面
1. 将操作切分成可适应内存层次结构不同层的 tiles
2. 决定计算的数据流，即 tile 的执行顺序以及保持静止或在处理器上移动的张量
3. 时空映射：决定哪些轴要并行化，哪些轴要串行运行
4. 决定如何交错通信和计算，以最大程度地减少时延
5. 将算术指令映射在硬件指令上

由于映射空间中点的选择会严重影响性能，所以硬件映射器的目的是在空间中寻找点可以最小化开销


映射决策的范围从简单数值或者分类参数到对正在运行的程序进行结构修改

### 寻找高性能映射
#### 映射策略
##### 图级调度器
融合 LayerNorm 并且从小 matmul 组合成大 matmul 有利于改变 GPU 性能

##### 操作级映射器
根据其做决策的方式，映射器被分为三种形式：暴力搜索、基于反馈的搜索和约束优化
###### 暴力搜索
要么是尽可能详细的三哦谱所，要么从地图中随机抽取大量点，一般也会采取启发式搜索去分割映射空间

坏处：越复杂的 workload 和硬件架构，暴力搜索越昂贵、对于任何 workload 或加速器结构的钢鞭，这个成本高昂的设计过程会重复进行，而无需利用任何先验知识

###### 基于反馈的搜索
适用于可大规模测量的现有硬件或者分析模型

###### 约束优化
其将调度问题表述为数值优化问题，以确定给定约束和目标函数集约束的变量复制

多面体编译充分利用基于约束优化的方法进行自动矢量化和循环 tiling。其主要侧重于测试 transform 的可行性，并提供信息来指导迭代搜索

### 映射的性能建模
Transformer 的性能建模的选择取决于映射器和目标工作负载

模型利用张量代数工作负载中已知的迭代空间便捷和静态可分析的数据访问模式来评估性能

还有一种模型引入了数据驱动的 ML 模型，其并非通过分析方式构建性能模型，而是通过统计技术以迭代方式将模型融合到随着时间收集的映射性能数据

先有模型的主要缺点时由于模型不能去硬件中实现的差异，其生成的映射无法在实际加速器上发挥最佳性能

生成指令流需要考虑大量的边缘情况

