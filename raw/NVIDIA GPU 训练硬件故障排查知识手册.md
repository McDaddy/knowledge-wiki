# NVIDIA GPU 训练硬件故障排查知识手册

## 知识点一：GPU 训练中最常见的硬件故障

在 NVIDIA GPU 大规模训练场景中，最常见的硬件故障主要集中在以下几个方面。

### 1. GPU 显存（HBM）错误 — 最高频

- **现象**：ECC（Error Correcting Code）不可纠正错误（Uncorrectable ECC errors），即 DBE（Double Bit Error）
- **影响**：直接导致训练进程崩溃（XID 48/63/64 等错误码）
- **背景**：HBM 堆叠工艺复杂，随运行时长增加，显存颗粒退化是最常见的硬件失效模式。A100/H100 上 HBM 相关故障在所有 GPU 硬件故障中占比最高

### 2. GPU 间互联故障（NVLink / NVSwitch）

- **现象**：NVLink CRC 错误、链路降速或断链；NVSwitch 上报 fatal error
- **影响**：多卡/多节点 AllReduce 通信失败或严重变慢，训练 hang 住或超时
- **特点**：在大规模集群（如数千卡）中，NVLink/NVSwitch 故障是第二大常见硬件问题。连接器物理接触不良、信号完整性退化是主要根因

### 3. GPU 过温降频 / 热故障

- **现象**：GPU 温度触发 thermal throttling（降频），严重时触发 thermal shutdown
- **影响**：训练吞吐骤降或 GPU 直接掉卡
- **根因**：散热系统失效（风扇故障、液冷泄漏/堵塞）、机房制冷异常、热界面材料老化

### 4. GPU 掉卡 / 不可见

- **现象**：PCIe 总线上 GPU 设备消失，nvidia-smi 无法识别，或报 "fallen off the bus"（XID 79）
- **根因**：PCIe 信号完整性问题、Riser 卡松动、GPU 供电异常、固件 bug 等

### 5. 电源相关故障

- **现象**：GPU 板卡供电不足或瞬时掉电，触发 power brake 或直接宕机
- **影响**：节点级训练中断

### 故障频率排序（业界经验）

| 排名 | 故障类型 | 典型表现 |
| --- | --- | --- |
| 1 | HBM ECC 错误 | XID 48/63/64，进程崩溃 |
| 2 | NVLink/NVSwitch 故障 | 通信超时、训练 hang |
| 3 | GPU 过温/降频 | 吞吐下降、热关机 |
| 4 | GPU 掉卡（PCIe） | 设备消失、XID 79 |
| 5 | 电源/供电异常 | 节点宕机 |

### 工程应对

- **定期巡检**：通过 DCGM / nvidia-smi 监控 ECC 计数、温度、NVLink 错误率，提前发现退化趋势
- **Checkpoint 机制**：大规模训练必须高频保存 checkpoint，出故障后快速恢复
- **冗余调度**：集群调度器自动隔离故障节点，将训练任务迁移到健康节点
- **行重置（Row Remapping）**：对可纠正 ECC 错误较多的显存行进行重映射，延缓 HBM 退化

---

## 知识点二：nvidia-bug-report.sh（NV Bugreport）

nvidia-bug-report.sh（通常简称 "nv bugreport"）是 NVIDIA 官方驱动自带的诊断日志收集脚本，安装驱动后就存在于系统中。

### 基本信息

| 项目 | 说明 |
| --- | --- |
| 脚本路径 | /usr/bin/nvidia-bug-report.sh |
| 输出文件 | 当前目录下生成 nvidia-bug-report.log.gz（压缩文本） |
| 运行方式 | sudo nvidia-bug-report.sh |
| 来源 | 随 NVIDIA 显卡驱动一起安装，属于驱动包的一部分 |

### 收集的内容

| 类别 | 具体内容 |
| --- | --- |
| 驱动信息 | 驱动版本、内核模块参数、加载状态 |
| GPU 硬件状态 | nvidia-smi 全量输出、GPU 温度/功耗/时钟/ECC 计数 |
| XID 错误 | 从 dmesg / journalctl 中提取的 NVRM XID 错误记录 |
| 内核日志 | 完整的 dmesg 输出（含 PCIe 错误、IOMMU 信息等） |
| PCIe 拓扑 | lspci -vvv 输出，包含 GPU 和 NVLink/NVSwitch 的 PCIe 信息 |
| 系统环境 | 内核版本、OS 发行版、CPU 信息、内存信息 |
| VBIOS / InfoROM | GPU 固件版本信息 |
| NVLink 状态 | NVLink 错误计数、链路状态 |
| CUDA 信息 | CUDA 版本、库路径 |
| X/Display 相关 | Xorg 日志（服务器场景通常不相关） |
| 进程信息 | 当前使用 GPU 的进程列表 |

### 使用场景

- 向 NVIDIA 提交 bug / RMA 工单时，这是他们要求的标准附件
- GPU 出现 XID 错误、掉卡、驱动崩溃等问题时，用它做第一步诊断
- 训练异常排查时，快速收集现场快照

### 实用命令

---

## 知识点三：lspci -vvv 输出解读

lspci -vvv 是 Linux 下查看 PCI/PCIe 设备详细信息的命令，-vvv 表示最高详细级别（very very verbose）。

### 输出结构

每个 PCI 设备会输出一个独立的段落，包含以下层次信息：

| 字段 | 说明 |
| --- | --- |
| BDF 地址 + 设备名 | 如 3b:00.0 3D controller: NVIDIA Corporation A100... |
| Subsystem | 子系统厂商和型号 |
| Control / Status | PCI 控制寄存器状态（Bus Master、Memory Space 等） |
| Latency / NUMA | 延迟、NUMA 节点 |
| Region (BAR) | 内存映射地址空间（BAR0~BAR5） |
| Capabilities | 最核心的部分，包括 PCIe 链路、电源管理、MSI-X、AER 等 |
| Kernel driver in use | 当前绑定的内核驱动 |
| Kernel modules | 可用的内核模块 |

### 典型输出样例（以 A100 GPU 为例）

### 关键字段解读（GPU 排障视角）

| 关注点 | 对应字段 | 健康值示例 |
| --- | --- | --- |
| PCIe 链路速率 | LnkSta: Speed | 16GT/s (ok) = Gen4 正常 |
| PCIe 链路宽度 | LnkSta: Width | x16 (ok) = 满宽正常 |
| AER 错误 | UESta / CESta | 全部为 - 表示无错误 |
| 电源状态 | Status: D0 | D0 = 正常工作状态 |
| 驱动绑定 | Kernel driver in use | nvidia 表示驱动正常加载 |
| BAR 地址空间 | Region 0/1/3 | 有正常内存地址映射 |

### 常见异常信号

- **Width x4 (downgraded)** — PCIe 降宽，可能是物理接触问题
- **Speed 2.5GT/s (downgraded)** — 降速到 Gen1，信号完整性问题
- **UESta 中出现 DLP+ / TLP+** — 数据链路/事务层出错，硬件故障信号
- **Kernel driver in use 为空** — 驱动未绑定，GPU 不可用
- **整个设备段消失** — GPU "掉卡"，PCIe 总线上已不可见

---

## 知识点四：ECC 错误详解

ECC（Error Correcting Code，纠错码）是一种内存数据完整性保护机制，通过在数据中附加校验位来检测和纠正比特翻转（bit flip）错误。在 GPU 语境下，主要保护的是 HBM 显存和 SRAM（L1/L2 Cache、寄存器文件等）。

### 比特翻转产生原因

- 宇宙射线 / 高能粒子（软错误，随机发生）
- 电气噪声 / 信号串扰
- 存储颗粒老化 / 制造缺陷（硬错误，持续发生）
- 温度过高导致电荷泄漏

### ECC 错误的两个级别

| 类型 | 全称 | 含义 | 影响 |
| --- | --- | --- | --- |
| SBE | Single Bit Error（可纠正错误） | 1 个 bit 翻转 | ECC 硬件自动纠正，业务无感知，但会计数累积 |
| DBE | Double Bit Error（不可纠正错误） | 2 个或更多 bit 翻转 | ECC 只能检测、无法纠正，数据已损坏 |

### 对训练的影响

- **SBE**：短期内无害，ECC 自动修复。但如果某个区域 SBE 持续累积，说明该存储单元正在退化，很可能最终演变为 DBE
- **DBE**：直接致命。GPU 驱动上报 XID 错误（如 XID 48/63/64），对应进程被 kill，训练中断

### 受 ECC 保护的 GPU 组件

| 组件 | 说明 |
| --- | --- |
| HBM（显存） | 最大的存储区域，ECC 错误最高发 |
| L1 Cache | SM 内的一级缓存 |
| L2 Cache | GPU 全局二级缓存 |
| Register File | SM 内寄存器文件 |
| SRAM | 内部各类 SRAM 块 |

### 查看 ECC 状态

输出示例：

- **Volatile**：本次驱动加载以来的计数（重启清零）
- **Aggregate**：GPU 生命周期内的累计计数（持久存储在 InfoROM 中，不可清零）

### ECC 错误处置逻辑

1. SBE 偶发 → 正常，无需处理
2. SBE 持续累积（同一区域） → 触发 Row Remapping（行重映射）
3. Row Remapping 配额耗尽 → 该 GPU 应下线更换
4. 出现 DBE → 立即报 XID 48/63/64 → 进程崩溃 → GPU 应立即隔离

### Row Remapping（行重映射）

A100/H100 支持对 HBM 中反复出现 SBE 的行进行备用行替换，类似 SSD 的坏块重映射。查看命令：

---

## 知识点五：XID 与 SXid 错误码体系

### XID — GPU 自身的错误码

XID 是 NVIDIA GPU 驱动（nvidia.ko）上报的错误事件标识符。

### SXid — NVSwitch 的错误码

SXid 是专门针对 NVSwitch 的错误码体系。

### XID vs SXid 对比

| 维度 | XID | SXid |
| --- | --- | --- |
| 所属设备 | GPU 本身 | NVSwitch |
| 上报来源 | NVIDIA GPU 驱动（nvidia.ko） | NVSwitch 驱动（nvidia-fabricmanager） |
| 日志标识 | NVRM: Xid (PCI:xx:xx.x): 48, ... | nvidia-fabricmanager: SXid (PCI:xx:xx.x): 18, ... |
| 错误范围 | GPU 显存、PCIe、引擎、驱动层 | NVSwitch 端口、链路、路由、内部 SRAM |
| 出现场景 | 所有 GPU 服务器 | 仅多卡 NVSwitch 互联拓扑（如 HGX A100/H100 8卡） |

### 常见 SXid 错误码

| SXid | 含义 | 严重程度 |
| --- | --- | --- |
| SXid 18 | NVLink 链路 fatal error（不可恢复的链路错误） | 致命 |
| SXid 19 | NVLink 链路降级（link degraded） | 高 |
| SXid 20 | NVSwitch 内部 SRAM 不可纠正 ECC 错误 | 致命 |
| SXid 21 | NVSwitch 内部 SRAM 可纠正 ECC 错误 | 警告 |
| SXid 22 | NVLink 链路 CRC 错误（高频） | 中→高 |
| SXid 24 | NVSwitch ingress 请求错误 | 高 |
| SXid 31 | NVSwitch 端口 fatal error | 致命 |
| SXid 32 | NVSwitch 路由错误 | 高 |

### 日志示例

### 排查命令

---

## 知识点六：nvidia-smi 工具详解

nvidia-smi（NVIDIA System Management Interface）是 NVIDIA 驱动自带的 GPU 状态查看和管理命令行工具，相当于 GPU 的"任务管理器"。

### 核心功能

| 功能 | 说明 |
| --- | --- |
| 实时状态监控 | 查看 GPU 温度、功耗、时钟频率、显存使用率 |
| 进程查看 | 哪些进程在用 GPU、各占多少显存 |
| ECC 错误查看 | 查看显存/SRAM 的 ECC 错误计数 |
| 驱动/CUDA 版本 | 当前驱动版本和支持的 CUDA 版本 |
| GPU 管理操作 | 设置功耗上限、切换 ECC 模式、设置计算模式、GPU Reset 等 |

### 默认输出示例

### 字段含义

| 字段 | 含义 |
| --- | --- |
| GPU 0 / 1 | GPU 编号 |
| Persistence-M | 持久模式（On = 驱动常驻内存，推荐服务器开启） |
| Temp 34C | 当前温度 |
| Perf P0 | 性能状态（P0=最高性能，P8=空闲低功耗） |
| Pwr 62W / 400W | 当前功耗 / 功耗上限 |
| Memory 1234MiB / 81920MiB | 已用显存 / 总显存 |
| GPU-Util 95% | GPU 计算核心利用率 |
| Volatile Uncorr. ECC: 0 | 本次启动以来不可纠正 ECC 错误数 |
| Processes | 正在使用 GPU 的进程及其显存占用 |

### 常用命令

### 训练排障场景

| 场景 | 命令 |
| --- | --- |
| 训练前检查 GPU 是否健康 | nvidia-smi -q 查看 ECC 错误、温度 |
| 训练中 GPU 利用率低 | nvidia-smi -l 1 观察 GPU-Util 是否正常 |
| 显存 OOM | 看 Memory-Usage 是否接近上限 |
| 怀疑 NVLink 问题 | nvidia-smi nvlink -e 查看链路错误计数 |
| GPU 掉卡 | nvidia-smi 看能否识别到所有卡 |
| 需要重置异常 GPU | nvidia-smi -r -i gpu_id |

---

## 知识点七：NVSwitch 错误上报机制

NVSwitch 是硬件芯片，其错误日志的上报依赖软件层。NVSwitch 有两种部署形态，上报机制不同。

### 两种 NVSwitch 形态

| 形态 | 位置 | 连接范围 | 代表平台 |
| --- | --- | --- | --- |
| 节点内 NVSwitch | 焊在 HGX 基板上 | 同一台机器内 8 张 GPU 互联 | HGX A100 / H100 |
| 节点间 NVSwitch（NVLink Switch） | 独立的 NVLink Switch Tray | 跨机器 GPU 互联（最多 256/576 GPU） | DGX SuperPOD（GB200 NVL72 等） |

### 节点内 NVSwitch — 由 Fabric Manager 上报

节点内的 NVSwitch 芯片通过 PCIe 挂在本机 CPU 下，对操作系统来说就是一个 PCIe 设备。

**上报链路**：NVSwitch 芯片（硬件寄存器记录错误） → NVSwitch 内核驱动（nvidia.ko 的一部分，轮询/中断读取寄存器） → nvidia-fabricmanager（用户态守护进程，解析并格式化错误） → 系统日志（journalctl / syslog） → 输出 SXid

**关键角色 nvidia-fabricmanager**：

| 项目 | 说明 |
| --- | --- |
| 全称 | NVIDIA Fabric Manager |
| 身份 | 用户态守护进程（systemd 服务） |
| 职责 | 管理节点内所有 NVSwitch 的初始化、NVLink 拓扑配置、健康监控、错误上报 |
| 日志 | journalctl -u nvidia-fabricmanager |
| 安装 | 随 NVIDIA Data Center GPU Manager 一起安装，nvidia-fabricmanager.service |

没有 Fabric Manager，NVSwitch 甚至无法正常工作——它负责在启动时配置 NVSwitch 的路由表，让 GPU 之间知道怎么通过 NVSwitch 互相访问。

### 节点间 NVSwitch（NVLink Switch Tray）— 由 UFM / Switch 自身 BMC 上报

在 GB200 NVL72 这类架构中，NVLink Switch Tray 是独立的硬件设备（类似网络交换机），不属于任何一台服务器。

**上报链路**：NVLink Switch Tray 硬件 → Switch 内置 BMC / 管理固件（自带管理处理器） → 带外管理网络（BMC 网口） → 集群管理软件（NVIDIA UFM / Base Command Manager / Redfish/IPMI 接口）

| 上报方式 | 说明 |
| --- | --- |
| Switch BMC | NVLink Switch Tray 自带 BMC，通过 Redfish API 暴露健康状态和错误日志 |
| NVIDIA UFM | 集群级 Fabric 管理软件，统一收集所有 NVLink Switch 的状态和错误 |
| 带内上报 | 连接到该 Switch 的各节点上的 Fabric Manager 也能检测到链路异常，上报 SXid |

### 总结

| 问题 | 答案 |
| --- | --- |
| NVSwitch 错误谁来感知？ | NVSwitch 芯片自身的硬件寄存器记录错误 |
| 节点内谁来读取和上报？ | nvidia-fabricmanager 守护进程（通过内核驱动访问寄存器） |
| 节点间 NVLink Switch 谁来上报？ | Switch 自身 BMC + NVIDIA UFM 集群管理软件 |
| 日志最终去哪？ | 节点内 → journalctl -u nvidia-fabricmanager；节点间 → UFM 控制台 / BMC Redfish 日志 |

核心结论：NVSwitch 虽然是"哑"硬件（没有跑操作系统），但它的错误通过"内核驱动读寄存器 → Fabric Manager 解析上报"这条链路变成了可见的 SXid 日志。在跨机场景下，则由 Switch 自带的 BMC 管理处理器独立上报。