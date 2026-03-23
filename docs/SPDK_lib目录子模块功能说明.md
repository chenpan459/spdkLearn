# SPDK `lib/` 目录：子模块功能说明

本文按 **子目录** 归纳 `spdk/lib` 下各静态库模块的职责。构建顺序与是否默认编译以 **`spdk/lib/Makefile`** 中 `DIRS-y` / `DIRS-$(CONFIG_*)` 为准（见文末说明）。

---

## 1. 应用框架与运行时

| 目录 | 生成库名 | 功能简述 |
|------|----------|----------|
| **event** | `event` | Reactor、应用主循环、`spdk_thread` 调度、定时器、poller 等 **SPDK 事件框架**（多数 app 的骨架）。 |
| **init** | `init` | **子系统初始化** 注册表（`SPDK_SUBSYSTEM`），按依赖顺序启动/关闭各模块。 |
| **thread** | `thread` | **`spdk_thread`**：用户态“线程”抽象、消息队列、与 reactor 绑定。 |

---

## 2. 环境、内存与系统抽象

| 目录 | 生成库名 | 功能简述 |
|------|----------|----------|
| **env_dpdk** | `env_dpdk` | 基于 **DPDK** 的默认实现：`spdk_env_init`、`spdk_malloc`、PCI、`spdk_mem_map`/`vtophys`、CPU 核等（见 [SPDK基于DPDK的二次开发与封装](./SPDK基于DPDK的二次开发与封装.md)）。 |
| **env_ocf** | `ocfenv`（注） | 与 **Open CAS Framework (OCF)** 集成时的 **env 适配层**（可选 `CONFIG_OCF`）。 |

注：`env_ocf/Makefile` 中 `LIBNAME := ocfenv`。

| 目录 | 生成库名 | 功能简述 |
|------|----------|----------|
| **dma** | `dma` | DMA 相关抽象与辅助（与 `spdk/dma.h` 等配合，服务 NVMe、bdev 等）。 |

---

## 3. 通用基础设施

| 目录 | 生成库名 | 功能简述 |
|------|----------|----------|
| **log** | `log` | 分级日志、`SPDK_LOG` 宏与后端。 |
| **util** | `util` | 字符串、位操作、文件、管道等 **通用工具** 函数。 |
| **conf** | `conf` | 配置文件解析（INI 风格等），供应用与 RPC 使用。 |
| **json** | `json` | 轻量 JSON 解析/生成（SPDK 内部 JSON 库）。 |
| **jsonrpc** | `jsonrpc` | JSON-RPC 协议解析与请求封装。 |
| **rpc** | `rpc` | 在 SPDK 中 **暴露 RPC 服务**、方法注册、与 jsonrpc 结合。 |
| **sock** | `sock` | **套接字抽象**（POSIX / SSL 等），供 NVMe/TCP、NVMf、iSCSI 等使用。 |
| **trace** | `trace` | 运行时 **trace 点** 记录基础设施。 |
| **trace_parser** | `trace_parser` | 解析 trace 数据（工具/库侧）。 |
| **keyring** | `keyring` | **密钥环**，服务 NVMe-oF 等场景的凭据管理。 |
| **notify** | `notify` | 外部 **通知** 机制（如 `spdk_notify`），与监控/事件联动。 |

---

## 4. 块设备与存储栈核心

| 目录 | 生成库名 | 功能简述 |
|------|----------|----------|
| **bdev** | `bdev` | **块设备抽象层**：队列、IO 通道、模块注册、各种 bdev 模块的枢纽。 |
| **blob** | `blob` | **Blobstore**：对象/blob 存储，是 lvol、部分 NVMe 应用的基础。 |
| **lvol** | `lvol` | 在 blobstore 上的 **逻辑卷（thin pool / lvol bdev）**。 |
| **nvme** | `nvme` | **NVMe 主机驱动库**（PCIe / TCP / RDMA 等传输），见 [lib/nvme 分析](./nvme/SPDK_lib_nvme目录源码结构与实现原理.md)。 |
| **nvmf** | `nvmf` | **NVMe-oF Target**：子系统、命名空间、Fabrics 连接与协议处理。 |
| **scsi** | `scsi` | **SCSI** 核心（命令、LUN 等），供 iSCSI 等使用。 |
| **iscsi** | `iscsi` | **iSCSI Target** 实现。 |
| **ftl** | `ftl` | **FTL（Flash Translation Layer）** bdev 库（Linux 上默认构建）。 |

---

## 5. 加速与硬件卸载

| 目录 | 生成库名 | 功能简述 |
|------|----------|----------|
| **accel** | `accel` | **统一加速框架**：加密、压缩、CRC 等，可对接 DPDK cryptodev、软件实现等。 |
| **ioat** | `ioat` | Intel **I/OAT** DMA 拷贝引擎用户态驱动。 |
| **idxd** | `idxd` | Intel **DSA / IAA（IDXD）** 用户态接口（可选 `CONFIG_IDXD`）。 |
| **ae4dma** | `ae4dma` | **AMD AE4DMA** 相关用户态支持。 |
| **mlx5** | `mlx5` | 在特定 MLX5 RDMA 配置下 **`mlx5_dv`** 相关辅助（`CONFIG_RDMA_PROV=mlx5_dv`）。 |
| **rdma_provider** | `rdma_provider` | RDMA **提供者抽象**（`CONFIG_RDMA`）。 |
| **rdma_utils** | `rdma_utils` | RDMA **公共工具**（内存域、连接辅助等，`CONFIG_RDMA`）。 |

---

## 6. 虚拟化与前后端

| 目录 | 生成库名 | 功能简述 |
|------|----------|----------|
| **virtio** | `virtio` | **Virtio** 设备与传输（`CONFIG_VIRTIO`）。 |
| **vhost** | `vhost` | **vhost-user / vhost-blk** 等，与 QEMU 虚拟机对接块设备（`CONFIG_VHOST`）。 |
| **vfio_user** | `vfio_user` | 子目录 **host**：**VFIO-user** 主机侧库，用户态设备映射（Linux，`CONFIG` 控制）。 |
| **vfu_tgt** | `vfu_tgt` | **VFIO-user Target** 侧支持（`CONFIG_VFIO_USER`）。 |

---

## 7. 内核接口与杂项

| 目录 | 生成库名 | 功能简述 |
|------|----------|----------|
| **nbd** | `nbd` | **Linux NBD** 导出，把 bdev 暴露为内核 NBD 设备（Linux）。 |
| **ublk** | `ublk` | **Linux ublk** 相关集成（`CONFIG_UBLK`，Linux）。 |
| **fsdev** | `fsdev` | **文件系统设备**抽象（`CONFIG_FSDEV`）。 |
| **fuse_dispatcher** | `fuse_dispatcher` | 与 **FUSE** 分发配合（`CONFIG_FSDEV` 等场景）。 |

---

## 8. 平台与拓扑

| 目录 | 生成库名 | 功能简述 |
|------|----------|----------|
| **vmd** | `vmd` | Intel **VMD（Volume Management Device）** 下 NVMe 拓扑发现与控制（LED 等）。 |

---

## 9. 测试与桩

| 目录 | 生成库名 | 功能简述 |
|------|----------|----------|
| **ut** | `ut` | **单元测试** 框架与断言（`CONFIG_TESTS` 或 `CONFIG_UNIT_TESTS`）。 |
| **ut_mock** | `ut_mock` | 测试用 **mock** 实现。 |

---

## 10. `lib/Makefile` 构建分组（速查）

| 条件 | 额外加入的目录 |
|------|------------------|
| 默认 `DIRS-y` | `bdev` `blob` `conf` `dma` `accel` `event` `json` `jsonrpc` `log` `lvol` `rpc` `sock` `thread` `trace` `util` `nvme` `vmd` `nvmf` `scsi` `ioat` `ut_mock` `iscsi` `notify` `init` `trace_parser` `keyring` `ae4dma` |
| `OS=Linux` | `nbd` `ftl` `vfio_user` |
| `CONFIG_UBLK` | `ublk` |
| `CONFIG_TESTS` 或 `CONFIG_UNIT_TESTS` | `ut` |
| `CONFIG_OCF` | `env_ocf` |
| `CONFIG_IDXD` | `idxd` |
| `CONFIG_VHOST` | `vhost` |
| `CONFIG_VIRTIO` | `virtio` |
| `CONFIG_RDMA` | `rdma_provider` `rdma_utils` |
| `CONFIG_VFIO_USER` | `vfu_tgt` |
| `CONFIG_FSDEV` | `fsdev` `fuse_dispatcher` |
| `CONFIG_RDMA_PROV=mlx5_dv` | `mlx5` |
| `CONFIG_ENV` 指向 `lib/<name>` | 同名的 env 实现目录（如默认 `env_dpdk`） |

---

## 11. 依赖关系（读代码时的直觉方向）

- **上层应用**：通常链接 **event + init + bdev +（nvme 或 nvmf 等）+ env**，经 **thread** 跑在 reactor 上。
- **bdev**：依赖 **nvme**（或各 module 中的 bdev 后端）与 **blob/lvol** 等组合出完整存储栈。
- **RPC**：依赖 **json + jsonrpc + conf**，与 **bdev/nvmf** 等模块的 RPC 方法注册结合。

---

## 12. 小结

`spdk/lib` 将 SPDK 拆成 **可独立链接的静态库**：底层为 **env / dma / sock / thread**；中间为 **bdev、blob、nvme、nvmf、scsi、iscsi**；外围为 **virtio/vhost、ftl、accel、各类 DMA 引擎**；通用服务为 **log、util、rpc、trace、keyring**。若需某一目录的源码级分析，可在对应子目录下结合 `include/spdk/*.h` 与 `module/` 中同名后端一起阅读。
