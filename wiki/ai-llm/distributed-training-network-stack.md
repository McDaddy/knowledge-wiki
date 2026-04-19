# 分布式训练通信栈速览

> Sources: Unknown, Unknown; 小nono 知识库, 2026-03-26
> Raw: [NVIDIA GPU 训练硬件故障排查知识手册](../../raw/NVIDIA GPU 训练硬件故障排查知识手册.md); [🗂️ 小nono 知识库](../../raw/🗂️ 小nono 知识库.md)

## Overview

这篇文章梳理训练与推理在资源侧的不同瓶颈，并把 NCCL、RDMA、InfiniBand、RoCE、NVLink、NVSwitch 放回各自所在的层级。一个简化心智模型是：NCCL 负责调度通信，RDMA 和 TCP 是传输方式，IB 与 RoCE 是网络实现，NVLink 和 NVSwitch 则是机器内部 GPU 之间的硬件高速通道。

## 训练和推理为什么是两种系统

训练更依赖算力、显存容量和跨卡跨机通信；推理更依赖带宽、时延和并发效率。训练常常需要机内 NVLink 配合机间 InfiniBand 或 RoCE，而推理更多时候停留在单机多卡范围，瓶颈转移到 KV Cache、接入层和服务吞吐。

## 通信栈分层

NCCL 是软件库，不是网络硬件。它会根据拓扑自动选择走 NVLink、IB RDMA、RoCE 或必要时退回 TCP。RDMA 表示一种绕过 CPU 的远程内存访问范式；InfiniBand 是专用高性能网络体系；RoCE 则是在以太网上实现 RDMA。NVLink 与 NVSwitch 只解决同机 GPU 互联，不负责跨机网络。

## AllReduce 与数据并行

数据并行的核心前提是每张卡保存一份相同模型，各自对不同 batch 计算梯度，再通过 AllReduce 汇总平均梯度。因为每张卡用的是同一份平均结果更新参数，所以模型副本能够继续保持一致。问题在于 AllReduce 强同步，任何一张卡掉队，整体训练速度都会被拖慢。

## 训练中的木桶效应

当 GPU、链路或节点出现慢卡、丢包、链路降级时，训练系统不会只让一部分卡变慢，而是把等待传播到整个作业。因此硬件健康、互联稳定性和拓扑质量，和纯算力一样重要。很多“训练慢”的问题，本质上不是模型算不动，而是通信路径被最慢节点拖住。

## See Also

- [KV Cache 与推理资源约束](kv-cache-and-inference-constraints.md)
- [GPU 训练硬件故障模式](../gpu-ops/gpu-training-hardware-failures.md)
- [GPU 排障信号与诊断工具](../gpu-ops/gpu-diagnostics-and-telemetry.md)
