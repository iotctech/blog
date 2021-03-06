---
title: GPU集合训练性能优化
date: 2020-07-23 11:51:05
tags:
 - 深度学习
 - 分布式训练
 - 集合训练
 - GPU
 - 性能优化
categories:
 - [deeplearning]
---

分布式训练模式主要包括**参数服务器异步训练、参数服务器同步训练、集合训练（同步模式）、模型并行同步训练**四种，常用的模式有参数服务器异步训练和集合训练两种，特点如下：

| 模式 | 优点 | 缺点 |
|------|------|------|
| 参数服务器异步训练 | 计算和通信在时间上分散，通信等待时间少<br> 各worker独立训练，减少同步阻塞时间<br> 训练吞吐量大，训练速度快<br> worker间松耦合，容错性好 | 参数梯度失效<br> 无锁更新，临界区数据会引入参数脏数据<br> 难以确定S/C的比例 |
| 集合训练 | P2P模式，简单易于扩展<br> 参数梯度本地更新，通信带宽占用少 | 木桶效应，负载不均造成训练阻塞<br> 大Batch影响收敛<br> 工作组强耦合，容错性差 |

针对集合训练GPU性能优化方法如下：

### 1、常规优化
#### 1.1、梯度融合和多流通信
分布式性能的关键是如何优化通信环节的效率。训练梯度的产生时机依赖于它们各自在计算图中的位置，计算图中存在依赖关系的部分梯度决定了这一部分梯度被计算出来的先后顺序。在梯度同步的过程中，面临的一个优化问题是，我们是要收集到任意数量的梯度之后立刻对所有的梯度进行通信，还是选择某个更优化的组合方式来通信。

这里一个确定性结论是，对单个梯度进行单次通信，通信效率总是非常低下的，我们需要进行多个梯度的融合，然后再对融合后的更大粒度上进行通信，确保每次通信粒度尽量均匀，减小出现大幅波动的可能。

从通信库而言，单一的通信流效率比较低，多流通信会分配多个通信流进行梯度通信，每个流服务于切分出来的某个融合梯度，而后续切分的融合粒度并不依赖于当前切分的融合梯度。

#### 1.2、图依赖分析
![](1.png)

![](2.png)

如图所示，如果正常分层训练，顺序执行SYNC_op时间会形成GAP，影响训练效率，可以将op3算子放到op2运算之后，这样可以消除GAP，达到提升效率的目的。

#### 1.3、层次通信
常规参数同步需要不同卡依次同步，由于机器内NVLink带宽（300GB）远大于机器间网卡带宽（10GB），可以选择参数优先在单台机器内部完成同步，然后再进行机器间的参数同步，最好再将同步信息在机器内进行多卡广播，同步到各个卡上。这样可以缩短点对点的路径，充分利用NVLink高带宽的特点，加速训练。

#### 1.4、OP融合
单射OP融合，解决访存受限问题


### 2、自动混合精度（AMP）训练
针对精度敏感的信息使用FP32精度，其他使用FP16精度，混合双精度运算可以减少内存开销，加快训练速度。

具体实现，设置黑白灰名单，黑名单存储精度敏感op，如softmax，exp等；白名单是可以使用FP16的op，高加速比op；灰名单是根据前序op决定自己精度的op。

- 前向/反向/通信：FP16
- 参数更新（精度敏感）：FP32

在参数更新之前使用FP16数据进行参数同步。
Resnet50测试 AMP有效加速2.8倍


### 3、显存有限的大Batch训练组件
**Forward Recompute Backpropagation：GPU显存资源较少的情况下，无损对齐算法配置的推荐方式**

针对显存有限，但Batch Size比较大的训练组件，采用重计算反向网络方案进行优化。

深度学习训练过程中，在前向网络每个算子只对前一算子数据有依赖关系，对没有依赖关系的算子使用后直接释放显存。但是在反向网络中，依赖的算子资源比较多，占用的显存会急速上涨，如果batch size比较大的话，显存峰峰值会限制组件性能。然而真正GPU的使用率却没有达到最优。

针对上述问题，可以在方向网络中将初始算子参数进行保存，释放该算子下级依赖算子的参数，等到方向网络需要依赖算子的参数时，使用保存的算子参数，依次重新参与一遍运算，求出依赖算子的参数。

- 32G V100 & FP32，在多个不同模型下进行最大batch size测试，提升高达600%+
- 加入Forward Recompute模型运行的吞吐量会损失15%-35%，但在分布式情况下，吞吐量下降为0%-17%