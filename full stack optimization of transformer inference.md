# Transformer综述

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


## 模型优化

### 量化
一般来说，推理并不需要很高的精度。量化可以对DNN模型进行压缩从而减小内存开销。模型尺寸也会因此大大减小。其次，量化激活进一步减少了中间部分内存带宽和存储。