# DPDK `lib/cryptodev` 代码分析

本文档分析目录 `spdk/dpdk/lib/cryptodev` 的核心代码，目标是说明：

- 这个库在 DPDK 里的定位；
- 控制面与数据面的分层方式；
- 会话（session）、队列、回调、统计与 trace 的实现关系；
- 应用如何与 PMD 驱动通过该库衔接。

---

## 1. 模块定位

`lib/cryptodev` 是 DPDK 的加密设备抽象层，作用与 `ethdev`/`compressdev` 类似：

- 向应用暴露统一 API（设备枚举、配置、队列、enqueue/dequeue、能力查询）；
- 向 PMD 提供 driver-facing 接口（`cryptodev_pmd.h`）；
- 统一管理 `crypto_op`、对称/非对称参数、会话对象与 fastpath 函数指针；
- 对外支持 telemetry 与 trace。

它本身不实现具体加密算法硬件流程，算法执行由各 `drivers/crypto/*` PMD 完成。

---

## 2. 编译规则（`meson.build`）

`lib/cryptodev/meson.build` 产物由三部分组成：

- 源文件：
  - `rte_cryptodev.c`（主体实现）
  - `cryptodev_pmd.c`（PMD 辅助层）
  - `cryptodev_trace_points.c`（trace 注册）
- 公开头：
  - `rte_cryptodev.h`
  - `rte_crypto.h`
  - `rte_crypto_sym.h`
  - `rte_crypto_asym.h`
  - `rte_cryptodev_trace_fp.h`
- 依赖：`kvargs`、`mbuf`、`rcu`、`telemetry`

可见这个库既有控制面实现，也把 fastpath trace 和 PMD glue 一并放在同一模块。

---

## 3. 文件职责总览

| 文件 | 主要职责 |
|---|---|
| `rte_cryptodev.c` | 设备全局管理、API实现、能力查询、启动停止、队列管理、回调、字符串映射、telemetry |
| `cryptodev_pmd.c` | PMD 初始化参数解析、设备创建/销毁、fastpath函数指针安装与重置 |
| `cryptodev_pmd.h` | PMD driver-facing 数据结构与回调接口定义 |
| `rte_cryptodev.h` | 应用接口、能力结构、队列/设备配置结构、统计结构、事件枚举 |
| `rte_crypto.h` | `rte_crypto_op` 通用结构、op池、sessionless/with-session 语义 |
| `rte_crypto_sym.h` | 对称算法/xform/operation 结构 |
| `rte_crypto_asym.h` | 非对称算法/xform/operation 结构 |
| `rte_cryptodev_core.h` | fastpath核心类型（enqueue/dequeue函数签名、qpdata） |
| `cryptodev_trace.h` / `rte_cryptodev_trace_fp.h` / `cryptodev_trace_points.c` | 控制面与数据面 tracepoint |

---

## 4. 核心架构：控制面与数据面分离

### 4.1 控制面（`rte_cryptodev.c`）

控制面负责：

- 设备实例表与全局状态管理；
- 设备配置、队列配置、启动停止、统计、能力查询；
- 回调注册/反注册；
- API 参数合法性检查与错误码统一；
- 与 PMD `dev_ops` 的对接。

### 4.2 数据面（fastpath）

数据面核心是函数指针与队列指针的“扁平缓存”：

- `rte_crypto_fp_ops[RTE_CRYPTO_MAX_DEVS]` 公开 fastpath 操作表；
- 每个设备启动后，`cryptodev_fp_ops_set()` 将 `enqueue_burst/dequeue_burst`、队列数组、回调数组等一次性拷贝到 FP ops；
- 未配置设备默认映射到 dummy enqueue/dequeue（返回 0 并置 `ENOTSUP`）。

这种设计减少了 fastpath 的间接层数，使 `rte_cryptodev_enqueue_burst()` 一类路径更接近直接 PMD 调用。

---

## 5. 关键数据结构

### 5.1 设备对象

在 `cryptodev_pmd.h` 中：

- `struct rte_cryptodev_data`：共享数据（`dev_id`、`name`、`dev_started`、`queue_pairs`、`dev_private`）。
- `struct rte_cryptodev`：运行态对象（PMD enqueue/dequeue 函数、`dev_ops`、feature flags、attached 状态、RCU 回调表等）。
- `struct rte_cryptodev_global`：全局设备数组与计数。

### 5.2 操作对象

在 `rte_crypto.h`：

- `struct rte_crypto_op`：统一操作对象，包含 type/status/session-type 与对称/非对称 union。
- `enum rte_crypto_op_sess_type`：`WITH_SESSION`、`SESSIONLESS`、`SECURITY_SESSION`。
- `rte_crypto_op_pool_private`：mempool私有元数据（op类型、priv_size）。

在 `rte_crypto_sym.h` / `rte_crypto_asym.h`：

- 定义具体算法参数、xform 链、输入输出偏移、IV/摘要等细节。

---

## 6. 会话模型

`cryptodev` 支持两类常见模型：

1. **with-session**  
   预创建会话，运行时 op 只携带 session 句柄，适合高吞吐重复参数场景。

2. **sessionless**  
   每个 op 自带完整 xform/参数，灵活但开销更高。

`struct rte_cryptodev_sym_session` 中包含驱动私有区（`driver_priv_data[]`）与 IOVA，反映了“通用会话壳 + PMD私有上下文”的设计。

---

## 7. 设备生命周期主链路

典型流程（应用视角）：

1. 获取设备 ID / 枚举设备；
2. `rte_cryptodev_configure()`；
3. `rte_cryptodev_queue_pair_setup()`；
4. `rte_cryptodev_start()`；
5. enqueue/dequeue；
6. `rte_cryptodev_stop()`；
7. `rte_cryptodev_close()`。

对应实现上，控制面函数会先做：

- 设备有效性检查；
- 队列范围检查；
- `dev_ops` 是否存在与能力匹配；
- 再调用 PMD 的对应 `dev_ops->xxx`。

---

## 8. PMD 对接机制（`cryptodev_pmd.c`）

`cryptodev_pmd.c` 提供 PMD 常用工具：

- `rte_cryptodev_pmd_parse_input_args()`：解析 `name/max_nb_queue_pairs/socket_id`；
- `rte_cryptodev_pmd_create()`：分配 cryptodev + 分配 dev_private；
- `rte_cryptodev_pmd_destroy()`：释放设备与私有内存；
- `cryptodev_fp_ops_reset()`/`cryptodev_fp_ops_set()`：重置或安装 fastpath 函数表；
- `rte_cryptodev_pmd_probing_finish()`：secondary 进程补齐 FP ops。

这部分代码体现了 cryptodev 对多进程和 PMD 插拔场景的适配。

---

## 9. 能力与算法抽象

`rte_cryptodev.h` 将能力分为：

- symmetric capability；
- asymmetric capability；
- 设备 feature flags（例如是否支持 sessionless、raw datapath、cpu-crypto 等）。

同时提供大量能力检查辅助函数，如：

- `rte_cryptodev_sym_capability_check_cipher/auth/aead`
- `rte_cryptodev_asym_xform_capability_check_*`

以及算法名和枚举互转函数，便于 CLI/配置文件与内部枚举对齐。

---

## 10. 回调、统计与可观测性

### 10.1 回调

`rte_cryptodev.c` 内部维护 callback 链表与自旋锁，支持事件回调注册/注销，且有 active 标记避免执行期间被非法释放。

### 10.2 统计

统一统计结构 `rte_cryptodev_stats`，包括 enq/deq 总数和错误计数；实际统计可由 PMD实现，也可走库层聚合。

### 10.3 Trace

- 控制面 trace：设备配置、队列、启动停止；
- fastpath trace：enqueue/dequeue。

通过 `cryptodev_trace_points.c` 注册 tracepoint，便于定位瓶颈和异常路径。

---

## 11. 与 `bbdev/compressdev` 的共性与差异

共性：

- 都是“统一设备抽象 + PMD实现”的框架；
- 都采用队列模型与 burst 语义；
- 都有 pmd helper 层与 trace 支持。

差异（cryptodev 特有重点）：

- session 机制更复杂（with-session/sessionless/security session）；
- 对称/非对称两大 op 体系并存；
- 能力矩阵更大（cipher/auth/aead/asym/PQC 等）。

---

## 12. 一句话总结

`lib/cryptodev` 的本质是 **“高性能加密任务调度与能力抽象层”**：  
它在控制面统一设备/队列/能力/会话管理，在数据面把开销压缩到 PMD 函数指针调用，并通过标准化 op 结构把多种密码学场景（对称、非对称、PQC）纳入同一套 DPDK burst 模型。

