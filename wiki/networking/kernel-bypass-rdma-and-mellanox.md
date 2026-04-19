# 内核绕过、RDMA 与 Mellanox 网络栈

> Sources: Mira 技术问答整理, 2026-04-19
> Raw: [服务器运维与网络知识手册](../../raw/networking-systems/2026-04-19-server-operations-and-network-handbook.md)

## Overview

这篇文章把高性能网络里常见的几个关键词放回同一张图里：内核网络与用户态网络的差别、RDMA 的数据路径与控制路径、RoCE / InfiniBand 的实现区别，以及 Mellanox ConnectX、`mlx5_core` 和 OFED 在软件栈中的位置。它的核心不是背名词，而是建立“性能来自哪里，复杂性又来自哪里”的判断框架。

## 为什么会有内核绕过

内核网络的优点是通用、稳定、具备完整安全与治理能力，但系统调用、上下文切换、协议栈遍历和数据拷贝都会增加开销。用户态网络的思路是尽量让数据路径远离这些开销，把包处理、队列访问和内存交互前移到应用附近，因此能显著降低延迟并提高吞吐，但会付出更高的 CPU 占用和更弱的通用性。

## RDMA 的真正边界

RDMA 经常被简化成“完全绕过内核”，这只对数据路径成立。真正的连接建立、内存注册和地址解析仍要依赖内核 RDMA 子系统。换句话说，RDMA 是把高频数据搬运从内核里摘出来，而不是把整个网络控制面都取消掉。

## InfiniBand、RoCE 与部署取舍

InfiniBand 是专用高性能网络体系，性能和确定性最好；RoCEv2 则是在以太网上提供类似 RDMA 能力，更容易接入现有网络基础设施；iWARP 基于 TCP，容错更友好但性能上限较低。实际部署时，RoCE 能否跑好往往不取决于“有没有这块网卡”，而取决于无损网络、拥塞控制和流量隔离是否到位。

## Mellanox ConnectX 与 OFED 的角色

ConnectX 新几代网卡虽然型号在变，但内核侧主驱动大体统一在 `mlx5_core`，RDMA 与 IB 扩展能力则由 `mlx5_ib` 提供。OFED 则是围绕这一硬件生态的驱动、用户态库和工具集合，负责把网卡、verbs、测试工具、固件管理以及上层协议支持连成完整工作栈。升级网卡时，真正需要连带考虑的往往不是“模块名换没换”，而是 OFED 版本、固件和上层组件是否一起匹配。

## 对 GPU 训练与存储系统的意义

高性能训练和分布式存储并不是简单地“上 RDMA 就行”，而是需要把普通业务流量、管理流量和 RDMA 流量在网络规划上尽量隔离。这样既能减少拥塞与调优复杂度，也能避免让训练通信受日常业务噪声影响。

## See Also

- [分布式训练通信栈速览](../ai-llm/distributed-training-network-stack.md)
- [服务器管理接口与网络寻址基础](server-management-and-addressing-primitives.md)
- [GPU 排障信号与诊断工具](../gpu-ops/gpu-diagnostics-and-telemetry.md)
