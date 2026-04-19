# Knowledge Base Index

## gpu-ops

GPU 训练故障、健康信号和诊断工具的整理，偏硬件可靠性与运维视角。

| Article | Summary | Updated |
|---------|---------|---------|
| [GPU 训练硬件故障模式](gpu-ops/gpu-training-hardware-failures.md) | 总结训练场景中最常见的 GPU 硬故障类型、优先级和工程应对方式。 | 2026-04-19 |
| [GPU 排障信号与诊断工具](gpu-ops/gpu-diagnostics-and-telemetry.md) | 串联 `nvidia-smi`、XID/SXid、`lspci -vvv`、DCGM 和 AFID 的排障路径。 | 2026-04-19 |

## networking

服务器网络、带外管理与高性能通信相关知识，覆盖管理接口、寻址模型和 RDMA 栈。

| Article | Summary | Updated |
|---------|---------|---------|
| [服务器管理接口与网络寻址基础](networking/server-management-and-addressing-primitives.md) | 梳理 Redfish、BMC、OneKeyLog、Syslog、VPS、多 IP、VIP 与 EIP 的关系。 | 2026-04-19 |
| [内核绕过、RDMA 与 Mellanox 网络栈](networking/kernel-bypass-rdma-and-mellanox.md) | 解释内核网络、用户态网络、RDMA、RoCE、InfiniBand 与 OFED 的分层关系。 | 2026-04-19 |

## systems

操作系统基础概念，聚焦进程、地址空间和上下文切换。

| Article | Summary | Updated |
|---------|---------|---------|
| [进程、内存与上下文切换](systems/processes-memory-and-context-switching.md) | 解释程序与进程的区别，以及 PCB 如何支撑多进程安全切换。 | 2026-04-19 |
| [内核维护与 Linux 运行时观测](systems/kernel-maintenance-and-linux-runtime.md) | 总结 Livepatch、DKMS、内核升级、IRQ Off 与 epoll 的运维含义。 | 2026-04-19 |

## cloud-infra

云基础设施与硬件卸载相关概念，包括 DPU、FPGA 和弹性资源抽象。

| Article | Summary | Updated |
|---------|---------|---------|
| [DPU、FPGA 与弹性基础设施](cloud-infra/dpu-fpga-and-elastic-infra.md) | 梳理 DPU 与 FPGA 的关系，以及 EIP、EBM 代表的资源解耦思路。 | 2026-04-19 |

## service-mesh

服务治理与代理架构，覆盖 Service Mesh 的收益、代价和接入实践。

| Article | Summary | Updated |
|---------|---------|---------|
| [Service Mesh 与 Envoy 实战要点](service-mesh/service-mesh-and-envoy.md) | 概括 Mesh 架构价值，并解释监听地址导致的端口冲突根因。 | 2026-04-19 |

## ai-llm

大模型推理与分布式训练相关知识，偏资源瓶颈与通信栈理解。

| Article | Summary | Updated |
|---------|---------|---------|
| [KV Cache 与推理资源约束](ai-llm/kv-cache-and-inference-constraints.md) | 说明 KV Cache 如何以空间换时间，以及为什么推理常先受限于显存与带宽。 | 2026-04-19 |
| [分布式训练通信栈速览](ai-llm/distributed-training-network-stack.md) | 解释 NCCL、RDMA、IB、RoCE、NVLink 和 AllReduce 的分层关系。 | 2026-04-19 |

## crypto

加密货币基础概念，聚焦比特币的账本、共识和交易模型。

| Article | Summary | Updated |
|---------|---------|---------|
| [比特币运行机制基础](crypto/bitcoin-foundations.md) | 总结哈希、数字签名、工作量证明、最长链和 UTXO 的协同作用。 | 2026-04-19 |

## toolchains

编译器与工具链基础知识，聚焦现代编译框架的分层与生态。

| Article | Summary | Updated |
|---------|---------|---------|
| [LLVM 模块化编译器框架](toolchains/llvm-overview.md) | 解释 LLVM 的前端、中端、后端分层以及 Clang、LLD、LLDB 等生态组件。 | 2026-04-19 |
