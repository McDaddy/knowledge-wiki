# GPU 排障信号与诊断工具

> Sources: Unknown, Unknown; 小nono 知识库, 2026-03-26; Mira 技术问答整理, 2026-04-19
> Raw: [NVIDIA GPU 训练硬件故障排查知识手册](../../raw/NVIDIA GPU 训练硬件故障排查知识手册.md); [🗂️ 小nono 知识库](../../raw/🗂️ 小nono 知识库.md); [服务器运维与网络知识手册](../../raw/networking-systems/2026-04-19-server-operations-and-network-handbook.md)

## Overview

这篇文章把 GPU 训练排障中最常见的观测信号和诊断工具串成一条链路：先看 `nvidia-smi` 与 XID/SXid 识别问题形态，再用 `nvidia-bug-report.sh`、`lspci -vvv`、DCGM 等工具补足硬件、驱动、互联与历史趋势信息。目标不是堆命令，而是建立一套从症状到定位的排障顺序。

## 一线观测信号

最先应关注的是四类信号：XID 或 SXid 错误、ECC 计数、温度与频率、设备是否可见。XID 是 GPU 自身的错误事件，SXid 更偏向 NVSwitch 侧的故障；两者能快速告诉排障人员问题大致位于显存、PCIe、互联还是驱动层。`nvidia-smi` 则适合作为第一眼总览，检查显存占用、功耗、温度、不可纠正 ECC 和当前使用进程。

## 核心工具职责

### nvidia-smi

`nvidia-smi` 更适合作为实时健康面板。它适合回答“现在这张卡是否健康、是否可见、有没有在跑、温度和 ECC 是否异常”这类问题。训练现场排障时，它应当先于更重的日志收集工具使用。

### nvidia-bug-report.sh

`nvidia-bug-report.sh` 适合做现场快照归档，尤其是在向 NVIDIA 提交工单或需要完整保存驱动、内核日志、PCIe 拓扑、NVLink 状态时。它的价值不在于实时观测，而在于把故障时刻的上下文尽可能完整地冻结下来，方便后续复盘。

### lspci -vvv

`lspci -vvv` 用于定位 PCIe 侧异常，例如链路降宽、降速、AER 错误和驱动未绑定。对于“掉卡但原因不清”“设备还在但吞吐异常”“怀疑 riser 或主板通道问题”这类场景，它的价值很高。

### DCGM 与指标体系

DCGM 更适合大规模集群的持续监控，而不是单次手工排障。它把 ECC、温度、XID、功耗、NVLink 等指标转成可采集的长期序列，使运维能够发现逐步退化的卡，而不是等到任务直接崩溃才介入。新的资料进一步明确了它的组件分工：`nv-hostengine` 负责常驻采集，`dcgmi` 负责人工操作，`dcgm-exporter` 负责对接 Prometheus，Grafana 则承担可视化展示。

## 关键排障字段

ECC 相关字段里，SBE 表示可纠正错误，DBE 表示不可纠正错误。AFID、Bank、Address、Syndrome 这类字段则把显存错误进一步定位到物理位置，有助于识别是偶发软错误还是持续性硬错误。对于 PCIe 场景，`LnkSta` 的速率与宽度、`UESta/CESta` 是否报错，以及 `Kernel driver in use` 是否正常，是最先看的字段。AFID 尤其适合和 Bank、Address 一起看，用来区分“某个位置持续退化”与“整张卡大面积劣化”这两种完全不同的处置路径。

## 推荐排障顺序

一个稳定的顺序是：先用 `nvidia-smi` 和日志判断故障是否致命，再用 `nvidia-bug-report.sh` 固化现场，随后用 `lspci -vvv` 检查 PCIe 和设备绑定，最后回到 DCGM 或监控系统检查问题是否是长期趋势而非一次性抖动。这样能避免一上来就陷入海量日志，却遗漏最关键的硬故障信号。

## See Also

- [GPU 训练硬件故障模式](gpu-training-hardware-failures.md)
- [分布式训练通信栈速览](../ai-llm/distributed-training-network-stack.md)
