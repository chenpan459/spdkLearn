# DPDK `lib/ethdev` 代码分析

本文档面向 `spdk/dpdk/lib/ethdev`，给出 `ethdev` 的结构化解读：它如何管理网口设备、如何衔接 PMD、以及 Rx/Tx fastpath 为什么能做到低开销。

---

## 1. 模块定位

`ethdev` 是 DPDK 最核心的设备抽象层之一，职责是：

- 对应用暴露统一网卡 API（配置、队列、启动/停止、收发、统计、事件）；
- 对 PMD 暴露驱动接口（`eth_dev_ops`、设备分配/释放、回调框架）；
- 将控制面和数据面解耦：控制面走通用函数，数据面走扁平 fastpath 函数指针。

`rte_ethdev.h` 顶部注释已经明确：API 分为“application-oriented”和“driver-oriented”两部分。

---

## 2. 编译规则（`meson.build`）

`lib/ethdev/meson.build` 体现出 `ethdev` 是一个“平台 + 功能聚合”库：

- 核心源文件：
  - `rte_ethdev.c`、`ethdev_driver.c`、`ethdev_private.c`
  - `rte_flow.c`、`rte_mtr.c`、`rte_tm.c`
  - telemetry/profile/trace/sff 等配套文件
- Linux 额外包含：
  - `ethdev_linux_ethtool.c/.h`
- 依赖：
  - `net`、`kvargs`、`meter`、`telemetry`

这说明 `ethdev` 不仅是收发 API，还整合了 flow/meter/traffic manager 及监控入口。

---

## 3. 文件职责速览

| 文件 | 主要职责 |
|---|---|
| `rte_ethdev.h` | 应用侧 API 与大量 inline fastpath 封装 |
| `rte_ethdev.c` | 控制面主实现：设备管理、配置、队列、统计、xstats、offload 文本化、iterator |
| `ethdev_driver.h` | PMD 接口与核心内部结构（`rte_eth_dev`、`rte_eth_dev_data`、`eth_dev_ops`） |
| `ethdev_driver.c` | PMD 生命周期辅助：分配、attach secondary、release、事件回调 |
| `rte_ethdev_core.h` | fastpath核心类型与 `rte_eth_fp_ops` 扁平表 |
| `rte_ethdev_trace_fp.h` | Rx/Tx fastpath tracepoint |
| `rte_flow.c` / `rte_mtr.c` / `rte_tm.c` | Flow/Meter/TM 通用 API 转发层 |
| `rte_ethdev_telemetry.c` | telemetry 命令适配 |

---

## 4. 关键设计：控制面和数据面分离

### 4.1 控制面

控制面围绕 `struct rte_eth_dev` + `struct rte_eth_dev_data` 运作：

- `rte_eth_dev_data`：共享配置数据（队列数组、MAC、mtu、link、状态位等）；
- `rte_eth_dev`：每进程对象（Rx/Tx函数指针、`dev_ops`、intr、回调链表等）。

该分层让“设备配置共享、多进程可见”与“函数指针按进程绑定”可以同时成立。

### 4.2 数据面

`rte_eth_fp_ops[RTE_MAX_ETHPORTS]` 是扁平 fastpath 表，包含：

- Rx：`rx_pkt_burst`、queue data/callback data、descriptor接口；
- Tx：`tx_pkt_burst`、`tx_pkt_prepare`、queue data/callback data 等。

优势是减少层级和 cache miss，让 `rte_eth_rx_burst/rte_eth_tx_burst` 更贴近直接 PMD 调用。

---

## 5. 核心数据结构

### 5.1 `rte_eth_dev`（驱动层可见）

关键字段：

- `rx_pkt_burst` / `tx_pkt_burst`（最关键 fastpath 指针）；
- `dev_ops`（控制面操作集）；
- `data`（共享态）；
- 回调链表（link 事件回调、Rx/Tx callback 链）。

### 5.2 `rte_eth_dev_data`（共享态）

包含：

- `rx_queues` / `tx_queues`、`nb_rx_queues` / `nb_tx_queues`；
- `dev_link`、`dev_conf`、`mtu`、MAC 地址集合；
- `dev_started`、`dev_configured`、queue state 等状态位；
- `flow_ops_mutex`、representor/backer 等高级特性字段。

---

## 6. 设备生命周期主链路

### 6.1 设备分配与注册（`ethdev_driver.c`）

- `rte_eth_dev_allocate(name)`：
  - 全局锁保护；
  - 找空闲 port；
  - 初始化 dummy fops（防止未配置路径被误用）；
  - 初始化共享 data（name、port_id、mtu、mutex）。

- secondary 进程通过 `rte_eth_dev_attach_secondary(name)` 挂接主进程已注册的端口，确保 port ID 一致。

### 6.2 probing 完成

`rte_eth_dev_probing_finish()`：

- secondary 时补齐 fastpath ops；
- 触发 `RTE_ETH_EVENT_NEW` 回调；
- 将状态置为 `ATTACHED`。

### 6.3 释放

`rte_eth_dev_release_port()`：

- 先发 `DESTROY` 事件；
- 重置 fastpath ops；
- 清空函数指针与设备对象；
- primary 释放共享资源（queues/mac/dev_private 等）。

---

## 7. 应用调用顺序与语义

`rte_ethdev.h` 推荐顺序：

1. `rte_eth_dev_configure()`
2. `rte_eth_tx_queue_setup()`
3. `rte_eth_rx_queue_setup()`
4. `rte_eth_dev_start()`
5. Rx/Tx burst
6. 变更配置前先 `rte_eth_dev_stop()`
7. 结束后 `rte_eth_dev_close()`

这是典型“先控制面收敛，再进入数据面”的 DPDK 设备模型。

---

## 8. 回调与并发模型

`ethdev` 有三类常见回调：

1. **设备事件回调**（link/new/remove 等）  
   由 `rte_eth_dev_callback_process()` 触发，带 active 标记和锁保护。

2. **Rx post-burst 回调**  
   挂在 `post_rx_burst_cbs[]`，在接收后执行。

3. **Tx pre-burst 回调**  
   挂在 `pre_tx_burst_cbs[]`，在发送前执行。

并发原则与注释一致：同一个队列的无锁 PMD 函数默认不应被多核并行调用，应用需自行保证。

---

## 9. 统计、xstats 与能力文本化

`rte_ethdev.c` 内置了：

- 基础 stats 名称映射（如 `rx_good_packets` 等）；
- queue 级统计映射；
- Rx/Tx offload 能力位到字符串映射；
- device capability 名称映射；
- RSS 算法名称映射。

这让 API/telemetry 输出更可读，也减少各 PMD 重复实现公共文本化逻辑。

---

## 10. 相关子系统：Flow / MTR / TM

`lib/ethdev` 内将三套高级管控接口并列提供：

- `rte_flow.*`（匹配动作规则）
- `rte_mtr.*`（流量计量）
- `rte_tm.*`（层级调度）

实现模式统一：按 `port_id` 找 `rte_eth_devices[port_id]`，校验后调用驱动对应 ops。  
这保证了高级特性与基础 `ethdev` 设备生命周期一致。

---

## 11. 多进程支持要点

- 共享数据通过 `eth_dev_shared_data` 管理；
- 端口创建/释放有全局锁同步；
- secondary 不重新创建设备，仅 attach 并复用端口信息；
- fastpath ops 在 secondary probing finish 时补齐，避免空指针。

这套机制和 `cryptodev/bbdev` 的多进程策略一致，但 `ethdev` 复杂度更高（队列、flow、representor、link事件都叠加）。

---

## 12. 一句话总结

`lib/ethdev` 本质是 DPDK 网络数据面的“总线枢纽”：  
它以 `rte_eth_dev`/`rte_eth_dev_data` 为核心，把 PMD 的差异封装到统一控制面 API，同时通过 `rte_eth_fp_ops` 保持 Rx/Tx fastpath 极简，从而在功能丰富（flow/mtr/tm/telemetry）的前提下维持高性能。

