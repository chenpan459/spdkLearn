# DPDK `lib/eventdev` 代码分析

本文档针对 `spdk/dpdk/lib/eventdev` 做代码层分析，重点是：

- eventdev 的架构定位与运行模型；
- 控制面与 fastpath 的数据结构；
- 各类 adapter（eth rx/tx、timer、crypto、dma、vector）的职责与软件/硬件两种路径；
- 典型调用链与工程实践注意点。

---

## 1. 模块定位

`eventdev` 是 DPDK 提供的“事件驱动调度框架”。  
相对传统 run-to-completion（应用直接轮询 Rx queue -> 处理 -> Tx），`eventdev` 引入“事件队列 + 调度器 + 事件端口”的中间层，把工作负载按 flow/priority/sched_type 调度到不同核心。

`rte_eventdev.h` 注释已经明确其模型：应用通过 event port 进行 dequeue/enqueue，底层由 event PMD 调度。

---

## 2. 编译规则（`meson.build`）

`lib/eventdev/meson.build` 的构成可以分三组：

1. **核心层**  
   `rte_eventdev.c`、`eventdev_private.c`、`rte_event_ring.c`

2. **adapter 层**  
   - `rte_event_eth_rx_adapter.c`
   - `rte_event_eth_tx_adapter.c`
   - `rte_event_timer_adapter.c`
   - `rte_event_crypto_adapter.c`
   - `rte_event_dma_adapter.c`
   - `rte_event_vector_adapter.c`

3. **可观测性**  
   `eventdev_trace_points.c`

依赖覆盖了 `ethdev/cryptodev/dmadev/timer/ring/mempool/mbuf`，说明 eventdev 的设计就是把多个子系统接到统一事件调度面。

---

## 3. 文件职责速览

| 文件 | 作用 |
|---|---|
| `rte_eventdev.h` | 应用侧 API、事件模型、配置结构、能力位、inline fastpath |
| `eventdev_pmd.h` | 驱动侧核心结构（`rte_eventdev`/`rte_eventdev_data`）与 `eventdev_ops` |
| `rte_eventdev.c` | 控制面主实现（设备生命周期、配置、queue/port link、caps 查询等） |
| `eventdev_private.c` | fastpath ops 安装/重置、dummy fallback |
| `rte_event_eth_rx_adapter.c` | eth Rx -> event 注入（轮询/中断/向量化） |
| `rte_event_eth_tx_adapter.c` | event -> eth Tx 输出 |
| `rte_event_timer_adapter.c` | timer event 注入（软实现 + PMD实现） |
| `rte_event_crypto_adapter.c` | event 与 cryptodev 的桥接 |
| `rte_event_dma_adapter.c` | event 与 dmadev 的桥接 |
| `rte_event_vector_adapter.c` | event vector 聚合与分发 |
| `rte_event_ring.c` | ring-based event device 支撑 |

---

## 4. 核心数据结构与分层

### 4.1 控制面对象

在 `eventdev_pmd.h`：

- `struct rte_eventdev_data`：共享配置/状态（`nb_queues`、`nb_ports`、`ports[]`、`queues_cfg[]`、`dev_conf`、`dev_started` 等）；
- `struct rte_eventdev`：运行态对象（`dev_ops`、enqueue/dequeue 指针、adapter 专用 enqueue 指针、preschedule/profile 钩子等）；
- `attached` 标志区分设备是否可用。

### 4.2 fastpath 扁平表

`rte_event_fp_ops[RTE_EVENT_MAX_DEVS]` 是 eventdev 的 fastpath 公共入口缓存：

- `enqueue_burst` / `enqueue_new_burst` / `enqueue_forward_burst`
- `dequeue_burst`
- `maintain`
- adapter 专用路径（`txa_enqueue`、`ca_enqueue`、`dma_enqueue`）
- `data = dev->data->ports`

由 `event_dev_fp_ops_set()` 安装，未配置时由 `event_dev_fp_ops_reset()` 指向 dummy 函数（直接报错或返回0），防止野调用。

---

## 5. 设备生命周期主链路

`rte_eventdev.c` 是 Northbound API 主体，常见流程：

1. 获取设备与能力：
   - `rte_event_dev_count()`
   - `rte_event_dev_get_dev_id()`
   - `rte_event_dev_info_get()`

2. 配置与建图：
   - `rte_event_dev_configure()`
   - `rte_event_queue_setup()`
   - `rte_event_port_setup()`
   - `rte_event_port_link()/unlink()`

3. 启动：
   - `rte_event_dev_start()`

4. 数据面循环：
   - `rte_event_dequeue_burst()`
   - `rte_event_enqueue_burst()`（NEW/FORWARD/RELEASE）

5. 停止与重配：
   - `rte_event_dev_stop()`
   - 重配后可再次 `start()`

整体遵循典型 DPDK 设备语义：先控制面收敛，后进入 burst fastpath。

---

## 6. 事件语义重点（`rte_eventdev.h`）

eventdev 模型里最关键的是事件元信息：

- `queue_id`、`flow_id`
- `sched_type`（ATOMIC/ORDERED/PARALLEL）
- `op`（NEW/FORWARD/RELEASE）

`FORWARD/RELEASE` 与“端口上下文”强相关，要求在同一 port 上维护事件生命周期，这也是调度一致性和有序性的基础。

---

## 7. Adapter 机制：软件桥接 + PMD能力查询

eventdev 的实践价值很大程度来自 adapter，把“非 event 接口”接入事件域。  
通用模式：

1. 先查 caps（`*_adapter_caps_get`）
2. PMD 没实现时回退 SW capability（`*_SW_CAP`）
3. 创建 adapter，必要时申请 service/core 资源
4. start 后在 service 函数里做批量搬运

### 7.1 Eth Rx Adapter

`rte_event_eth_rx_adapter.c` 体量最大，支持：

- 轮询 + 中断混合；
- 加权轮询（WRR）；
- 事件批量缓存；
- 向量事件（event vector）；
- 动态 timestamp mbuf field。

其本质是把 `ethdev Rx queue` 的包打包为 event 并投递到 event port。

### 7.2 Eth Tx Adapter

反向路径，把 event 还原为目的端口/队列的 Tx 发送动作。  
与 event vector、同目的地批处理优化耦合较深。

### 7.3 Timer Adapter

`rte_event_timer_adapter.c` 支持两种路径：

- PMD 提供 timer adapter ops（硬件/专用实现）；
- 无 PMD ops 时，使用默认软件实现（swtim_ops + service 线程）。

创建时会根据 caps 决定是否需要额外 event port（`INTERNAL_PORT` 能力位）。

### 7.4 Crypto / DMA / Vector Adapter

- `rte_event_crypto_adapter.c`：event 与 cryptodev 的桥接，内部有 circular buffer、批量阈值与 backpressure 处理。
- `rte_event_dma_adapter.c`：同理对接 dmadev。
- `rte_event_vector_adapter.c`：处理 vector 事件聚合/释放相关逻辑，与 Rx vector 场景配套。

---

## 8. 多进程与共享内存

eventdev 也沿用 DPDK 常见模式：

- 设备对象数组是全局静态；
- data 结构可放共享内存；
- adapter 常通过 memzone 管理实例数组或共享数据区；
- secondary 进程依赖主进程已建立的设备状态与 fastpath 指针安装。

---

## 9. trace 与可观测性

`rte_eventdev_trace_fp.h` 覆盖了关键 fastpath 观测点：

- enqueue/dequeue burst
- maintain
- profile switch/preschedule
- eth tx adapter enqueue
- crypto adapter enqueue
- timer arm/cancel burst

`eventdev_trace_points.c` 完成注册，便于线上问题定位（调度卡顿、adapter 回压、定时器抖动等）。

---

## 10. 与 `ethdev` 的关系（工程视角）

可以把 eventdev 看作在 `ethdev` 上再加一层“调度域”：

- 直接 `ethdev`：应用自己决定核间分发与流水；
- `eventdev`：把分发、顺序、原子流控交给 event PMD/adapter 框架。

当 pipeline 有多阶段、跨设备（eth+crypto+timer）协同时，eventdev 架构优势更明显；但控制复杂度和调优成本也更高。

---

## 11. 一句话总结

`lib/eventdev` 是 DPDK 的事件调度中枢：  
通过 `rte_eventdev` 核心 + 多种 adapter，将网络收包、加密、DMA、定时器等异构输入统一成事件流，在保持 burst fastpath 的同时提供有序/原子/并行调度语义，适合构建多阶段高并发数据面流水线。

