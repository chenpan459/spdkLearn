# SPDK 基于 DPDK 的二次开发与封装

本文说明 **SPDK 在 DPDK 之上做了哪些封装、扩展与工程化工作**，侧重能力边界与代码落点。与仓库内 [SPDK调用DPDK流程分析文档](./SPDK调用DPDK流程分析文档.md)（初始化与调用链）互补：该文档讲怎么调，本文讲多做了什么、为什么。

---

## 1. 定位：DPDK 在 SPDK 中的角色

| 维度 | 说明 |
|------|------|
| **DPDK 承担** | EAL（进程/大页/亲和性）、PCI 总线与设备框架、部分内存 API（`rte_malloc` / memzone / mempool）、与版本相关的底层能力（如 `rte_power`、Vhost 用户态库等）。 |
| **SPDK 承担** | 面向存储的统一环境 API（`spdk/env.h`）、虚拟地址到物理/IOMMU 可 DMA 地址的登记与查询、SPDK 风格 PCI 驱动模型、热插拔与错误处理策略，以及 NVMe/块栈、Accel、Vhost 等业务层。 |

**不应混淆**：SPDK 的主线是 **用户态存储与 NVMe**，不是 DPDK 典型的网卡报文转发路径；DPDK 在这里更像高性能用户态运行时加设备/内存基础设施。

---

## 2. 架构分层（概念）

自上而下简述：

- **SPDK 业务与框架**：NVMe / Bdev / NVMf / iSCSI、`spdk_thread` / reactor、Accel、Vhost 等。
- **env_dpdk**：实现 `spdk/env.h`，内部包含 `mem_map` / `vtophys`、PCI 抽象、`spdk_env_init` 与 `post_init`。
- **DPDK**：`rte_eal_*`、`rte_pci`、堆与大页、`rte_power*`、`rte_vhost_*` 等。

业务代码优先调用 **SPDK env API**；需要密码学/压缩设备时再经 Accel 模块触及 **cryptodev/compressdev**；Vhost 路径使用 **rte_vhost** 做 GPA 到 VVA 等转换。

---

## 3. 核心二次开发：`lib/env_dpdk`

### 3.1 EAL 封装与启动策略

- **`spdk_env_init()`**（`spdk/lib/env_dpdk/init.c`）：根据 `struct spdk_env_opts` 拼装 DPDK EAL 命令行（core mask / `--lcores`、`--no-shconf`、大页、`env_context` 透传等），再调用 **`rte_eal_init()`**。
- **OpenSSL**：在同一初始化路径中完成 SSL 库初始化，便于依赖 TLS 的组件与 DPDK 共存。
- **`spdk_env_dpdk_post_init()`**：EAL 就绪后执行 SPDK 侧 PCI 环境、`mem_map`、`vtophys` 初始化；与 `spdk_env_init()` 拆开的目的是支持 **外部已调用 `rte_eal_init()`** 的场景（见 `spdk/include/spdk/env_dpdk.h` 中 `spdk_env_dpdk_post_init` / `spdk_env_dpdk_external_init`）。
- **析构优先级**：通过 `destructor(101)` 在适当时机调用 **`rte_eal_cleanup()`**，避免与 SPDK 其它析构顺序冲突。

关键代码参考（EAL 与 post_init 衔接）：

```797:831:spdk/lib/env_dpdk/init.c
	fflush(stdout);
	orig_optind = optind;
	optind = 1;
	rc = rte_eal_init(g_eal_cmdline_argcount, dpdk_args);
	optind = orig_optind;

	free(dpdk_args);

	if (rc < 0) {
		if (rte_errno == EALREADY) {
			SPDK_ERRLOG("DPDK already initialized\n");
		} else {
			SPDK_ERRLOG("Failed to initialize DPDK\n");
		}
		return -rte_errno;
	}

#ifdef __FreeBSD__
	legacy_mem = true;
#else
	legacy_mem = false;
	if (opts->no_huge || (opts->env_context && strstr(opts->env_context, "--legacy-mem") != NULL)) {
		legacy_mem = true;
	}
#endif

	rc = spdk_env_dpdk_post_init(legacy_mem);
	if (rc == 0) {
		g_external_init = false;
	}

	return rc;
}
```

### 3.2 内存：`spdk_malloc` 与 SPDK 独有映射层

- **基础分配**（`spdk/lib/env_dpdk/env.c`）：`spdk_malloc` / `spdk_zmalloc` / `spdk_free` 等映射到 **`rte_malloc_socket` / `rte_zmalloc_socket` / `rte_free`**，并处理 cache line 对齐、`enforce_numa` 回退等策略。
- **`spdk_mem_map` + `vtophys`**（`spdk/lib/env_dpdk/memory.c`）：这是 **SPDK 在 DPDK 之上的关键扩展**——维护多级页表式的虚拟地址翻译与注册状态（含 2MB/4KB 粒度、注册标志位），用于：
  - 将用户态缓冲区 **登记为对设备 DMA 可见**（并与 IOMMU/VFIO 路径协作，Linux 下可见 `rte_vfio`、`vfio_iommu_type1_dma_map` 等相关逻辑）。
  - 提供 **`spdk_vtophys`** 等路径，供 NVMe 等驱动构造 PRP/SGL 所需的物理地址。
- **与 DPDK 内存模型的关系**：底层仍依赖 DPDK 堆与大页；**翻译与注册语义**由 SPDK 统一抽象在 `spdk/memory.h` 体系，而非完全暴露 `rte_memseg` 细节给上层业务。

### 3.3 PCI：`spdk_pci_*` 抽象与 DPDK 绑定

- **统一设备模型**（`spdk/include/spdk/env.h`）：`spdk_pci_device`、`spdk_pci_driver_register`、`spdk_pci_enumerate` 等，业务驱动（如 NVMe）面向 SPDK 类型编程。
- **实现落点**（`spdk/lib/env_dpdk/pci.c` + `pci_dpdk*.c`）：内部将 SPDK 驱动翻译成 `struct rte_pci_driver` 并 **`rte_pci_register`**；BAR 映射、配置空间读写走 DPDK PCI 封装。
- **热插拔与线程**：使用 **`rte_eal_hotplug_remove`** 等，并针对 DPDK 行为做了 **重试**（如 `DPDK_HOTPLUG_RETRY_COUNT`）；配合 **`rte_alarm`** 等在安全线程上下文处理 detach。
- **错误处理**（`spdk/lib/env_dpdk/sigbus_handler.c`）：注册 **SIGBUS** 分发链，支持 **`spdk_pci_register_error_handler`**，用于 MMIO 异常等场景与上层恢复策略衔接。

### 3.4 线程与 CPU：`threads.c`

- `spdk_env_get_core_count`、`spdk_env_get_current_core`、`spdk_env_get_socket_id` 等直接映射 **`rte_lcore_*` / `rte_socket_*`**。
- 远程核启动使用 **`rte_eal_remote_launch` / `rte_eal_mp_wait_lcore`**。

说明：SPDK 的 **reactor / spdk_thread** 是在此之上的进一步框架，属于 SPDK event 子系统，但 **物理核枚举与亲和** 仍根植于 DPDK EAL。

### 3.5 DPDK PCI API 私有化后的兼容层

自 DPDK 22.11 起 PCI 部分 API 不再对外公开，SPDK 采用：

- **按版本拷贝/适配头文件与实现**：`spdk/lib/env_dpdk/` 下按 DPDK 版本分子目录（如 `22.11/`），由 **`pci_dpdk_2211.c`** 等与具体 ABI 对齐。
- **维护脚本**：`spdk/scripts/env_dpdk/README.md` 描述 **`check_dpdk_pci_api.sh`** 及版本目录、补丁目录约定。

这是典型的 **二次开发中的长期兼容工程**，而非单一功能点。

---

## 4. 其他模块中对 DPDK 的直接使用（功能级）

| 能力 | 作用 | 主要代码位置 |
|------|------|----------------|
| **Accel CryptoDev** | 将 DPDK `cryptodev`（QAT、AES-NI、MLX5、UADK 等）接入 SPDK 统一 Accel 框架（session、mbuf、mempool、队列深度调优）。 | `spdk/module/accel/dpdk_cryptodev/` |
| **Accel CompressDev** | 同理，对接压缩设备。 | `spdk/module/accel/dpdk_compressdev/` |
| **调度 Governor** | 用 **`rte_power_*`** 查询/调节核心频率，对接 SPDK scheduler / governor 抽象。 | `spdk/module/scheduler/dpdk_governor/dpdk_governor.c` |
| **Vhost User** | guest GPA 到进程内 VVA 等通过 **`rte_vhost_va_from_guest_pa`** 等；字符设备、会话与 SPDK 线程模型结合。 | `spdk/lib/vhost/rte_vhost_user.c` 等 |

这些模块 **不替代 env_dpdk**，而是在 env 已就绪的前提下 **按需链接 DPDK 子库**（具体裁剪见 `spdk/dpdkbuild/Makefile`、`spdk/lib/env_dpdk/env.mk` 中的 `DPDK_LIB_LIST` 与条件编译）。

---

## 5. 与「纯 DPDK 应用」的能力对比（简表）

| 能力 | 纯 DPDK 应用常见用法 | SPDK 典型做法 |
|------|----------------------|----------------|
| 启动 | 自行解析 argv 调 `rte_eal_init` | `spdk_env_opts` 统一拼 EAL 加 `post_init` |
| DMA 缓冲区 | mbuf、mempool、dma 相关 API 分散使用 | **`spdk_mem_map` 注册 + `spdk_vtophys`** 与块设备 IO 模型对齐 |
| PCI 驱动 | 直接写 `rte_pci_driver` | **`spdk_pci_*`** 加多版本 compat |
| 存储协议 | 无 | NVMe/Bdev/NVMf 等 **SPDK 自有栈** |

---

## 6. 文档与代码索引

| 主题 | 路径 |
|------|------|
| EAL 与 post_init | `spdk/lib/env_dpdk/init.c` |
| 堆分配封装 | `spdk/lib/env_dpdk/env.c` |
| mem_map / vtophys / VFIO | `spdk/lib/env_dpdk/memory.c` |
| PCI 与热插拔 | `spdk/lib/env_dpdk/pci.c`, `pci_event.c` |
| DPDK PCI 版本适配 | `spdk/lib/env_dpdk/pci_dpdk_*.c`, `spdk/lib/env_dpdk/22.11/` 等 |
| SIGBUS | `spdk/lib/env_dpdk/sigbus_handler.c` |
| 线程封装 | `spdk/lib/env_dpdk/threads.c` |
| 对外 DPDK 专用 API | `spdk/include/spdk/env_dpdk.h` |
| 对外统一 env API | `spdk/include/spdk/env.h` |
| PCI 头同步脚本说明 | `spdk/scripts/env_dpdk/README.md` |
| 调用链详解 | [SPDK调用DPDK流程分析文档](./SPDK调用DPDK流程分析文档.md) |

---

## 7. 小结

SPDK 基于 DPDK 的「二次开发」可概括为三类：

1. **抽象与稳定接口**：用 `spdk/env.h` 屏蔽底层实现，便于测试与多环境（虽生产上以 env_dpdk 为主）。
2. **存储导向的增强**：以 **`mem_map` / `vtophys` / VFIO DMA** 为核心的 DMA 可见性与地址翻译，这是 NVMe 等路径的关键，超出普通 DPDK 示例应用的使用深度。
3. **工程与生态**：PCI API 版本适配、热插拔重试、SIGBUS、可选 Accel/电源/Vhost 集成，以及内嵌 DPDK 构建裁剪。

若需跟踪 **具体链接了哪些 DPDK 静态库**，请以 `spdk/dpdkbuild/Makefile` 与 `spdk/lib/env_dpdk/env.mk` 为准，并随 SPDK 版本更新核对。
