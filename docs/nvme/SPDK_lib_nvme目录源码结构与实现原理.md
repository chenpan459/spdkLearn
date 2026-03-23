# SPDK `lib/nvme` 目录：功能划分与实现原理

本文面向 **源码阅读**，说明 `spdk/lib/nvme` 下各文件职责、整体分层设计，以及 NVMe 用户态驱动中的关键机制。公开 API 定义见 `spdk/include/spdk/nvme.h`。与仓库内 [SPDK_NVMe库功能分析.md](./SPDK_NVMe库功能分析.md) 互补：该文档偏功能列表，本文偏 **架构与原理**。

---

## 1. 目录在 SPDK 中的位置

`lib/nvme` 实现 **SPDK NVMe 主机端驱动库**（`libnvme` / `spdk_nvme`）：向上通过 `spdk/nvme.h` 提供探测、控制器、命名空间、队列对、IO 与 Admin 命令等接口；向下依赖 `env`（大页、DMA、`spdk_vtophys`、PCI）、可选 RDMA/TCP 等传输。

**设计目标**：全用户态、以 **轮询完成队列** 为主路径、多传输统一抽象、可扩展多进程与热插拔。

---

## 2. 源文件与职责一览

| 源文件 | 主要职责 |
|--------|----------|
| **nvme.c** | 驱动全局状态：`g_spdk_nvme_driver`、探测/连接/分离的入口（`spdk_nvme_probe*`、`spdk_nvme_connect*`）、`trid` 解析与比较、transport 注册协调、多进程下控制器列表与异步 detach 聚合。 |
| **nvme_transport.c** | **传输层抽象**：`spdk_nvme_transport_register`、`nvme_transport_ctrlr_construct/scan/destruct` 等，按 `trid->trstring` 分发到具体 transport 的 `spdk_nvme_transport_ops`。 |
| **nvme_pcie.c** | **NVMe over PCIe**：PCI 枚举、控制器构造、MMIO/门铃、热插拔、SIGBUS 与寄存器重映射等（与 `nvme_pcie_common.c`、`nvme_pcie_internal.h` 配合）。 |
| **nvme_pcie_common.c** | PCIe 路径上共享逻辑（队列、请求、与 PCIe 相关的公共辅助）。 |
| **nvme_fabric.c** | **Fabrics 通用**：与 NVMe-oF 相关的 Property Get/Set、Connect 等 **与具体链路无关** 的命令封装（TCP/RDMA 等复用）。 |
| **nvme_tcp.c** | **NVMe/TCP** transport 实现（注册到 transport 表）。 |
| **nvme_rdma.c** | **NVMe/RDMA** transport（条件编译 `CONFIG_RDMA`）。 |
| **nvme_vfio_user.c** | **VFIO-user** 传输（条件编译 `CONFIG_VFIO_USER`），用于特定虚拟化/用户态设备后端场景。 |
| **nvme_ctrlr.c** | **控制器状态机**：寄存器读写经 `nvme_transport_ctrlr_get_reg_*`、CC/CSTS 初始化、Identify、命名空间树、AER、reset、keep-alive、features 等 **控制器级** 逻辑。 |
| **nvme_ctrlr_cmd.c** | Admin 命令封装（同步/异步辅助、与具体 opcode 相关的提交路径）。 |
| **nvme_ctrlr_ocssd_cmd.c** | Open-Channel SSD（OCSSD）相关 Admin 命令。 |
| **nvme_ns.c** | **命名空间**：容量、扇区大小、PI、多路径等 NS 级状态。 |
| **nvme_ns_cmd.c** | **IO 命令**：Read/Write/Flush/DSM 等构造与提交，与 `nvme_request`、payload（PRP/SGL）配合。 |
| **nvme_ns_ocssd_cmd.c** | OCSSD 命名空间 IO 命令。 |
| **nvme_qpair.c** | **队列对**：SQ/CQ、请求池、提交与完成轮询、`nvme_request` 生命周期、错误与重试、opcode 字符串调试等。 |
| **nvme_poll_group.c** | **Poll Group**：把多个 `qpair` 编组，批量 `process_completions`，可选 **accel_fn_table**（CRC32/拷贝链与序列回调），与 `fd_group` 配合中断模式。 |
| **nvme_discovery.c** | **发现服务**：从 discovery 控制器获取子系统/路径信息（Fabrics 场景）。 |
| **nvme_auth.c** | **NVMe over Fabrics 认证**（DH-CHAP 等与规范相关的流程）。 |
| **nvme_quirks.c** | **设备 quirk**：按 VID/DID 等匹配已知固件异常，在 `nvme_internal.h` 中以 `NVME_QUIRK_*` 标志使用。 |
| **nvme_zns.c** | **ZNS**（Zoned Namespace）相关逻辑。 |
| **nvme_opal.c** | **Opal/TCG 安全**相关（与 `nvme_opal_internal.h`）。 |
| **nvme_io_msg.c** + **nvme_io_msg.h** | 向 IO 路径投递 **外部消息**（`spdk_ring` + 专用 qpair），用于在控制面更新与 IO 线程间协调。 |
| **nvme_util.c** | 通用工具函数。 |
| **nvme_stubs.c** | 条件编译关闭时的桩函数，保证链接完整。 |
| **nvme_cuse.c** | **CUSE** 字符设备导出（`CONFIG_NVME_CUSE`），便于用户态把 NVMe 暴露为节点。 |
| **nvme_internal.h** | 内部类型：`nvme_request`、`spdk_nvme_ctrlr`/`qpair` 内部字段、quirk 常量、默认队列深度等。 |
| **spdk_nvme.map** | 导出符号版本控制。 |

**构建**：见 `spdk/lib/nvme/Makefile`——`RDM`、`CUSE`、`VFIO_USER` 等为可选源文件与系统库。

---

## 3. 核心设计原理

### 3.1 传输层抽象（Transport ops）

所有传输实现同一套 **操作表** `spdk_nvme_transport_ops`（`nvme_transport.c` 中注册），包括：

- 控制器：`ctrlr_construct`、`ctrlr_scan`、`ctrlr_destruct`、`ctrlr_enable`、寄存器读写（同步/异步）等。
- 队列：`qpair_*`、提交与完成处理。

**原理**：NVMe 规范在 **PCIe** 与 **Fabrics** 上访问控制器寄存器、建队列的方式不同；Fabrics 还需 Property、Connect 等。抽象层让 `nvme_ctrlr.c` 里 **CC/CAP 等状态机** 用统一 `nvme_transport_ctrlr_get_reg_*` 调用，避免 `#ifdef TCP/RDMA` 散落。

注释（`nvme_transport.c`）说明：因 **PCIe 多进程** 限制，transport 指针不能随意挂在所有结构上，Admin 路径常需 **`nvme_get_transport(trstring)`** 再分发；IO 路径可在 `qpair` 上缓存 transport 指针以降低开销。

### 3.2 控制器与寄存器访问

`nvme_ctrlr.c` 通过宏将 **CAP/CC/CSTS** 等访问映射到 `nvme_transport_ctrlr_get_reg_*` / `set_reg_*`。这样 **PCIe MMIO** 与 **Fabrics Property 命令** 对上层呈现一致。

初始化顺序遵循 NVMe 规范：读 CAP、配置 Admin SQ/CQ、写 CC 使能、等待 RDY、Identify Controller/Namespace 等。

### 3.3 队列对、请求与轮询模型

- **sqpair**（`nvme_qpair.c`）管理 **提交项与完成项**、与 **门铃**（PCIe）或 **传输层等价机制**（Fabrics）交互。
- 每个 IO 对应 **`nvme_request`**（见 `nvme_internal.h`），包含命令、payload 类型（连续 PRP / SGL / iov）、完成回调。
- **默认路径**：CPU **主动轮询 CQ**（`spdk_nvme_qpair_process_completions`），避免内核中断上下文；可选 **中断 + fd_group**（poll group 等）。

**原理**：用户态 NVMe 的延迟优势主要来自 **无系统调用热路径 + 批处理完成**，与 SPDK 线程模型结合时，通常每个线程或 poller 固定处理若干 qpair。

### 3.4 Poll Group（`nvme_poll_group.c`）

将多个 `qpair` 归入一个 `spdk_nvme_poll_group`，**一次轮询批量处理完成**，减少 per-qpair 开销；可挂 **accel 回调**（CRC、拷贝、序列化）以在 **同一线程** 内做 IO 与简单计算流水线。

### 3.5 探测、连接与分离

- **probe**：按 `trid` 调用对应 transport 的 `ctrlr_scan`，对用户设备回调 `attach_cb`。
- **connect**：直接按地址连接远程 Fabrics 控制器。
- **detach**：引用计数 + **异步销毁**（`nvme_ctrlr_detach_async` → `nvme_ctrlr_destruct_async`），poll 直到资源释放；支持 **批量 detach 上下文**（`spdk_nvme_detach_ctx`）。

PCIe 控制器在 **多进程** 下可进入共享列表（`nvme_ctrlr_shared`），与本地 `g_nvme_attached_ctrlrs` 区分。

### 3.6 Quirks（`nvme_quirks.c` + `nvme_internal.h`）

真实 SSD 固件与 QEMU 等与规范有偏差；通过 **VID/DID/子系统** 等匹配，设置 `NVME_QUIRK_*`（延迟 RDY、队列深度、SGL/DSM 行为等）。这是长期 **可维护性** 与 **兼容性** 的关键设计。

### 3.7 Fabrics 与发现、认证

- **nvme_fabric.c**：Property、Connect 等 **与链路无关** 的 Fabric 命令组装。
- **nvme_discovery.c**：Discovery Controller 流程。
- **nvme_auth.c**：DH-CHAP 等认证流程，与 NVMe-oF 安全规范对齐。

### 3.8 PCIe 特有问题（`nvme_pcie.c`）

- **热插拔**：与 `spdk_pci` 事件、允许列表、按 `trid` 查找控制器。
- **MMIO 故障**：`SIGBUS` 与寄存器 **匿名映射替换**（`nvme_sigbus_fault_sighandler` 等），避免访问已拔出设备的 BAR 直接崩溃。

### 3.9 IO 消息（`nvme_io_msg.c`）

通过 ring 将 **需要在 IO 线程执行的回调** 排队，与 `external_io_msgs_qpair` 配合，在 `nvme_io_msg_process` 中消费。用于 **控制面与 IO 路径解耦**（例如命名空间更新、多进程协调）。

---

## 4. 数据路径简图（概念）

```text
应用 / 上层 bdev
    spdk_nvme_ns_cmd_read / write / ...  (nvme_ns_cmd.c)
           ↓
    构造 nvme_request + PRP/SGL (payload)
           ↓
    提交到 spdk_nvme_qpair (nvme_qpair.c)
           ↓
    transport: nvme_pcie / nvme_tcp / nvme_rdma / ...
           ↓
    设备
           ↓
    轮询 CQ → 完成回调 → 释放 request
```

Admin 路径则经 `nvme_ctrlr_cmd.c` / `nvme_ctrlr.c` 与同一套 transport 寄存器或 Fabrics 命令。

---

## 5. 推荐阅读顺序（针对本目录）

1. `nvme_transport.c`：理解 ops 注册与分发。
2. `nvme.c`：`probe` / `connect` / `detach` 与 `trid`。
3. `nvme_ctrlr.c`：初始化状态机与 Identify 流程（可配合 `grep nvme_ctrlr_set_state`）。
4. `nvme_qpair.c`：提交与完成轮询。
5. `nvme_ns_cmd.c`：典型 IO 命令。
6. 按兴趣：`nvme_pcie.c`（本地）或 `nvme_fabric.c` + `nvme_tcp.c`/`nvme_rdma.c`（Fabrics）。

---

## 6. 相关文档与头文件

| 说明 | 路径 |
|------|------|
| 对外 API | `spdk/include/spdk/nvme.h` |
| NVMe 规范常量 | `spdk/include/spdk/nvme_spec.h` |
| 本库内部类型 | `spdk/lib/nvme/nvme_internal.h` |
| 功能向梳理 | [SPDK_NVMe库功能分析.md](./SPDK_NVMe库功能分析.md) |
| NVMe 对外接口与流程 | [SPDK_NVMe对外接口与调用流程分析.md](./SPDK_NVMe对外接口与调用流程分析.md) |

---

## 7. 小结

`lib/nvme` 的实现可概括为：**以 transport 抽象统一 PCIe 与 Fabrics**；**以 qpair + request 实现无锁化/少锁化 IO 路径**；**以 ctrlr 状态机贯彻规范**；**以 quirks、异步销毁、IO msg、SIGBUS 等处理工程现实**。阅读时抓住 **transport ops**、**ctrlr 初始化**、**qpair 轮询** 三条主链，即可快速建立整体图景。
